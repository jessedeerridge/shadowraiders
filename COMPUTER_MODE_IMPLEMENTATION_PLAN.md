# Computer Mode (NPC Auto-play) Implementation Plan

## Scope and assumptions
- Activate only when host presses **Start** and human participants are **1–3**.
- Force total participants to **5** by adding NPC seats.
- Keep all turn progression in a **single scheduler** to avoid overlapping timers.
- Let host client be the **authoritative NPC executor** (writes all NPC actions to DB).

---

## 1) NPC creation (colors/names)

### Data model additions
```ts
// rooms/{roomCode}
{
  computerMode: {
    enabled: boolean,
    hostUid: string,
    totalPlayers: 5,
    createdAt: number,
  },
  npcs: {
    [npcUid: string]: {
      uid: string,
      name: string,
      color: string,
      seatIndex: number,
      isNpc: true,
      createdAt: number,
    }
  },
  tables: {
    [seatIndex: string]: {
      playerId: string,
      playerName: string,
      color: string,
      isNpc?: boolean,
    }
  }
}
```

### Name pool
```ts
const WESTERN_NAMES = [
  "Oliver", "Sophia", "Emma", "Noah", "Lucas", "Mia", "Liam", "Ethan", "Chloe", "Ava",
  "Mason", "Logan", "Grace", "Hannah", "Ella", "Nora", "Alice", "Jack", "Leo", "Henry",
  "Ruby", "Scarlett", "Amelia", "Aria", "Julian", "Theo", "Daniel", "Clara", "Iris", "Violet"
];
```

### Creation flow pseudocode
```ts
async function maybeEnableComputerModeOnStart(room) {
  const humans = getHumanParticipants(room.tables); // 1..3 to enable
  if (humans.length < 1 || humans.length > 3) return { enabled:false };

  const targetTotal = 5;
  const missing = targetTotal - humans.length;
  if (missing <= 0) return { enabled:false };

  const usedColors = new Set(humans.map(h => h.color));
  const availableColors = SEAT_COLORS.filter(c => !usedColors.has(c));
  const pickedColors = shuffle(availableColors).slice(0, missing);

  const usedNames = new Set(humans.map(h => h.playerName.toLowerCase()));
  const npcNames = pickUniqueNames(WESTERN_NAMES, missing, usedNames);

  const emptySeats = getEmptySeatIndexes(room.tables).slice(0, missing);

  const updates = {};
  const npcs = {};
  for (let i = 0; i < missing; i++) {
    const npcUid = `npc_${Date.now()}_${i}_${randomId(6)}`;
    const seat = emptySeats[i];
    const name = npcNames[i];
    const color = pickedColors[i];

    updates[`tables/${seat}`] = {
      playerId: npcUid,
      playerName: name,
      color,
      isNpc: true,
      updatedAt: Date.now(),
    };
    npcs[npcUid] = { uid:npcUid, name, color, seatIndex:seat, isNpc:true, createdAt:Date.now() };
  }

  updates[`computerMode`] = {
    enabled: true,
    hostUid: room.hostId,
    totalPlayers: 5,
    createdAt: Date.now(),
  };
  updates[`npcs`] = npcs;

  await update(roomRef, updates);
  return { enabled:true, npcUids:Object.keys(npcs) };
}
```

---

## 2) Turn step scheduler (strict sequencing, 2s pacing)

### Required guards
```ts
let npcTurnId = "";
let actionLock = false;
let isResolvingTurnStep = false;
let hasAttackedThisTurn = false;
```

### Scheduler contract
- Steps execute in order and **await completion**.
- Use one async loop instead of many `setTimeout`s.
- `sleep(2000)` between major steps.

### Pseudocode
```ts
async function runNpcTurnScheduler(turn, npcUid) {
  if (actionLock) return;
  actionLock = true;

  npcTurnId = `${npcUid}:${turn.updatedAt}:${Date.now()}`;
  hasAttackedThisTurn = false;

  try {
    // Step 1: move (or choice move from room7)
    await runStep(npcTurnId, async () => await npcMoveStep(npcUid));
    await sleep(2000);

    // Step 2: mandatory room action/effect resolution
    await runStep(npcTurnId, async () => await npcRoomActionStep(npcUid));
    await sleep(2000);

    // Step 3: mandatory attack if valid target exists (at most once)
    await runStep(npcTurnId, async () => {
      if (hasAttackedThisTurn) return;
      const target = pickAttackTargetInRange(npcUid);
      if (!target) return;
      hasAttackedThisTurn = true;
      await npcAttackStep(npcUid, target);
    });

    // Step 4: end turn
    await runStep(npcTurnId, async () => await endNpcTurn(npcUid));
  } finally {
    actionLock = false;
    isResolvingTurnStep = false;
  }
}

async function runStep(expectedTurnId, fn) {
  if (npcTurnId !== expectedTurnId) return;
  if (isResolvingTurnStep) return;
  isResolvingTurnStep = true;
  try { await fn(); } finally { isResolvingTurnStep = false; }
}
```

---

## 3) Room action selection logic (mandatory action)

### Rules
- Every turn must perform at least one available room action.
- room1: white draw **100%** (NPC never skips)
- room2: black draw **100%**
- room5: green draw
- room6: random from green/white/black
- room3/room4: include special action logic

### Pseudocode
```ts
async function npcRoomActionStep(uid) {
  const roomId = markerRoomId(markerCache[uid]);

  switch (roomId) {
    case 1:
      await atomicCardFlow(uid, "white");
      return;
    case 2:
      await atomicCardFlow(uid, "black");
      return;
    case 3:
      await runRoom3SpecialOrFallbackDraw(uid);
      return;
    case 4:
      await runRoom4SpecialOrFallbackDraw(uid);
      return;
    case 5:
      await atomicCardFlow(uid, "green");
      return;
    case 6: {
      const type = randomPick(["green", "white", "black"]);
      await atomicCardFlow(uid, type);
      return;
    }
    default:
      // room7 and others: perform any valid draw/effect per rule availability
      await runFallbackMandatoryAction(uid);
  }
}

async function atomicCardFlow(uid, type) {
  // draw -> render/show -> apply effects -> grant equipment -> close/continue
  const card = await drawCardFromDeckAtomic(type);
  await publishCardUiEvent({ uid, type, card });
  await applyCardEffectAtomic({ uid, type, card });
  await grantEquipmentIfAnyAtomic({ uid, card, type });
  await closeCardUiIfNeeded({ uid, card, type });

  if (type === "green") {
    await autoGreenSendReceiveAnswerClose(uid, card);
  }
}
```

---

## 4) Auto green card flow (room5/room6)

### Requirements mapping
- Auto handle `send → receive → answer → close` for NPC.
- NPC answer uses faction alignment.

### Faction-aligned answer heuristic
```ts
function chooseGreenAnswerByFaction(receiverUid, payload) {
  const faction = factionForUid(receiverUid); // シチズン/レイダー/シャドウ
  const targetUid = payload.targetUid;

  // Example heuristic:
  // - same faction likely "YES" (support)
  // - opposing faction likely "NO" (mislead)
  const sameFaction = factionForUid(targetUid) === faction;

  if (faction === "シチズン") return sameFaction ? "はい" : "いいえ";
  if (faction === "レイダー") return sameFaction ? "はい" : "いいえ";
  if (faction === "シャドウ")  return sameFaction ? "はい" : "いいえ";
  return Math.random() < 0.5 ? "はい" : "いいえ";
}
```

### Flow pseudocode
```ts
async function autoGreenSendReceiveAnswerClose(senderUid, greenCard) {
  const receiverUid = pickGreenReceiver(senderUid);
  await createGreenRequest({ senderUid, receiverUid, greenCard, at:Date.now() });

  const isReceiverNpc = isNpc(receiverUid);
  if (isReceiverNpc) {
    const answer = chooseGreenAnswerByFaction(receiverUid, { targetUid: senderUid, greenCard });
    await writeGreenAnswer({ receiverUid, answer, at:Date.now() });
  }

  await finalizeGreenRequest({ senderUid, receiverUid });
  await closeGreenModalForAll({ requestId: greenCard.requestId });
}
```

---

## 5) Dice UI + speech bubble sequencing

### Movement bubble timing
```ts
async function npcMoveStep(uid) {
  await showPositionBubble(uid, "ふります！", { color:"white", durationMs:1500 });
  await showMoveDiceUi(uid); // after bubble disappears

  const result = rollMoveDice();
  await publishMoveDiceEvent({ uid, ...result });
  await waitDiceFinalize("move", uid);

  // Critical rule: movement visual 0.2s after finalization
  await sleep(200);
  await applyNpcMapMovement(uid, result.sum);
}
```

### Attack bubble timing
```ts
async function npcAttackStep(attackerUid, targetUid) {
  const tName = playerById(targetUid)?.name || "対象";
  await showPositionBubble(attackerUid, `${tName}さんを攻撃！`, { durationMs:1500 });

  const result = rollAttackDice(attackerUid);
  await publishAttackDiceEvent({ uid:attackerUid, targetUid, ...result });
  await waitDiceFinalize("attack", attackerUid);

  // remove bubble immediately after finalize
  await hidePositionBubble(attackerUid);

  // Critical rule: HP-board icon movement 0.2s after finalize
  await sleep(200);
  await applyNpcAttackDamageAndHpMovement(attackerUid, targetUid, result);
}
```

---

## 6) Strict anti-double-attack guard

### Guard design
- `hasAttackedThisTurn` hard stop inside scheduler.
- `npcTurnId` snapshot check before every step.
- `actionLock` blocks re-entry from duplicate listeners.
- `isResolvingTurnStep` prevents simultaneous step execution.

### Pseudocode
```ts
function canNpcAttackNow(uid) {
  if (actionLock) return false;
  if (isResolvingTurnStep) return false;
  if (hasAttackedThisTurn) return false;
  if (!isCurrentTurn(uid)) return false;
  if (turnState().attackDone) return false;
  return hasValidAttackTarget(uid);
}

async function tryNpcAttack(uid) {
  if (!canNpcAttackNow(uid)) return;
  hasAttackedThisTurn = true;
  await npcAttackStep(uid, pickAttackTargetInRange(uid));
  await markTurnAttackDone(uid);
}
```

---

## 7) 0.2s post-finalization timing rule (map + HP board)

### Contract
- Dice finalize event is the zero point.
- Any icon movement happens at `finalizeTs + 200ms`.

### Pseudocode utility
```ts
async function applyAfterDiceFinalize(kind, actorUid, fn) {
  const finalizeTs = await waitDiceFinalize(kind, actorUid);
  const wait = Math.max(0, finalizeTs + 200 - Date.now());
  if (wait > 0) await sleep(wait);
  await fn();
}

// movement
await applyAfterDiceFinalize("move", npcUid, async () => {
  await applyNpcMapMovement(npcUid, sum);
});

// attack/hp
await applyAfterDiceFinalize("attack", npcUid, async () => {
  await applyNpcAttackDamageAndHpMovement(attackerUid, targetUid, result);
});
```

---

## Integration points (recommended)
1. **Start button handler**: call `maybeEnableComputerModeOnStart(roomData)` before dealing identities.
2. **Turn subscription**: host-only watcher checks current player; if NPC and `computerMode.enabled`, run scheduler.
3. **Dice event pipeline**: ensure NPC-triggered events still render same animation as human events.
4. **Green flow handlers**: add NPC auto-answer branch where receiver has `isNpc=true`.
5. **Card draw pipeline**: refactor to atomic helper for white/black reliability.

---

## Test checklist
- Start with 1, 2, 3 humans → total becomes 5 with NPC seats/colors/names.
- NPC turn order executes exactly once per turn.
- room1/2 NPC always draws.
- room6 random draw path works; green path fully auto-closes.
- NPC movement/attack/card dice all show UI animation.
- Speech bubble timing: 1.5s before dice roll.
- No double attack in one turn under rapid turn updates.
- Map/HP icon move only after dice finalize + 0.2s.
