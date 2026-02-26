# Computer Mode (NPC auto-play) Implementation Plan

## Scope
This plan adds **Computer Mode** for games started by host with 1–3 humans.
It is designed for the current Firebase-driven room model (`rooms/{roomCode}`), with host-authoritative NPC orchestration.

---

## 1) Computer Mode activation + NPC creation

### Trigger
On host `Start`:
- Count seated human participants (`tables/*` entries where `isNpc !== true`).
- If count is `1..3`, enable computer mode and fill seats to 5 players.
- If count is `>=4`, normal mode (no NPC fill).

### Data model additions
At room root:
- `computerMode: { enabled: boolean, totalPlayers: number, hostUid: string, startedAt: number }`

At table seat:
- `isNpc: true` for NPC seats.
- `npcProfile: { nameSeed: string, strategy: "default" }`

### Color + name generation
- Build `usedColors` from human seats.
- Candidate colors: `SEAT_COLORS` minus `usedColors`.
- Shuffle candidate colors, assign to NPC seats.
- Name source list (Western-style):
  `Alice, Benjamin, Charlotte, Daniel, Emma, Felix, Grace, Henry, Isaac, Julia, Kevin, Liam, Mason, Nora, Oliver, Peter, Quinn, Ruby, Sophia, Thomas, Victor, William, Zoe`.
- Pick without duplication in a game (fallback: append numeric suffix).

### Pseudocode
```js
async function ensureComputerModeSeats(roomRef, roomData){
  const human = listSeatedHumans(roomData.tables);
  if (human.length < 1 || human.length > 3) return roomData;

  const target = 5;
  const missing = target - human.length;
  if (missing <= 0) return roomData;

  const openSeats = allSeatIndexes().filter(i => !roomData.tables?.[i]);
  const freeColors = shuffled(SEAT_COLORS.filter(c => !humanColors.has(c)));
  const names = shuffled(WESTERN_NAMES);

  for (let i = 0; i < missing; i++){
    const seat = openSeats[i];
    const uid = `npc_${Date.now()}_${i}_${Math.random().toString(36).slice(2,8)}`;
    roomData.tables[seat] = {
      playerId: uid,
      playerName: takeUniqueName(names),
      color: freeColors[i],
      isNpc: true,
      joinedAt: Date.now(),
      npcProfile: { nameSeed: "western", strategy: "default" }
    };
  }

  roomData.computerMode = {
    enabled: true,
    totalPlayers: 5,
    hostUid: state.userId,
    startedAt: Date.now()
  };
  return roomData;
}
```

---

## 2) Host-authoritative NPC turn step scheduler (2-second pacing)

### Why scheduler
To satisfy:
- strict no-overlap execution,
- deterministic pacing,
- anti-double-attack guarantees.

### State guard
```ts
NpcTurnGuard = {
  npcTurnId: string,
  actionLock: boolean,
  hasAttackedThisTurn: boolean,
  isResolvingTurnStep: boolean,
  currentStep: "idle"|"move"|"roomAction"|"attack"|"end",
  cancelled: boolean
}
```

### Step contract
Every step is an awaited task:
1. movement
2. room effect/action (at least one action)
3. mandatory attack if in range
4. end turn

Between major steps: `await waitMs(2000)`.

### Pseudocode
```js
async function runNpcTurn(turnUid){
  const turnId = `${turnUid}:${turn.updatedAt}`;
  if (npcGuard.actionLock) return;
  npcGuard = newGuard(turnId);

  try {
    await runStep(turnId, "move", async () => {
      await npcMoveStep(turnUid);
    });

    await waitMs(2000);

    await runStep(turnId, "roomAction", async () => {
      await npcMandatoryRoomAction(turnUid);
    });

    await waitMs(2000);

    await runStep(turnId, "attack", async () => {
      if (!npcGuard.hasAttackedThisTurn){
        await npcAttackStepIfTargetExists(turnUid);
        npcGuard.hasAttackedThisTurn = true;
      }
    });

    await waitMs(2000);

    await runStep(turnId, "end", async () => {
      await endNpcTurn(turnUid);
    });
  } finally {
    npcGuard.actionLock = false;
  }
}
```

`runStep` validates `turnId` and guard flags so stale timers never continue after turn transition.

---

## 3) Room action selection logic (mandatory action)

### Rule
Each turn must execute at least one room-available action.

### Policy
- room1/room2: always draw (100%).
- room3/room4: invoke room special once if available; otherwise fallback draw if allowed by rule.
- room5: draw green.
- room6: random draw from `[green, white, black]`.
- room7: no draw requirement itself; mandatory action can be attack if available, otherwise fallback movement-derived room effect before end.

### Pseudocode
```js
async function npcMandatoryRoomAction(uid){
  const rid = roomIdOf(uid);
  switch (rid){
    case 1:
    case 2:
      return drawForActor(uid, randomPick(["white","black"]));
    case 3:
      if (canUseRoom3Special(uid)) return useRoom3Special(uid);
      return drawForActor(uid, "white");
    case 4:
      if (canUseRoom4Special(uid)) return useRoom4Special(uid);
      return drawForActor(uid, "black");
    case 5:
      return drawForActor(uid, "green");
    case 6:
      return drawForActor(uid, randomPick(["green","white","black"]));
    default:
      return markMandatoryActionDone(uid); // explicit no-op only when truly no legal action
  }
}
```

---

## 4) Room5/Room6 green-card full auto flow (send→receive→answer→close)

### Requirements mapping
- If NPC draws green in room5/6: full flow runs automatically.
- NPC responder answers by faction goal alignment.
- After answer: auto close UI/modal.

### Answer logic (faction-aligned)
- `シチズン`: tends to deny shadow-aligned prompts, accept citizen-beneficial outcomes.
- `レイダー`: prioritize raider survival and eliminating shadow pressure.
- `シャドウ`: hide identity unless forced; choose options that increase chaos/damage.

(Implement as deterministic heuristic table + fallback random tie-break.)

### Pseudocode
```js
async function runAutoGreenFlow(senderUid, card){
  const receiverUid = chooseReceiverInRangeOrAny(senderUid);
  const requestId = createGreenRequest(senderUid, receiverUid, card);

  await writeGreenInbox(receiverUid, { from: senderUid, requestId, card });
  await waitMs(600);

  const receiverFaction = factionForUid(receiverUid);
  const options = greenAnswerOptionsForUid(receiverUid, card);
  const selected = chooseGreenAnswerByFaction(receiverFaction, options, card);

  await writeGreenResponse(senderUid, {
    from: senderUid,
    to: receiverUid,
    requestId,
    answerId: selected.id,
    answerLabel: selected.label
  });

  await waitMs(600);
  await closeGreenFlowFor(senderUid, receiverUid, requestId); // close dock/modal + signal
}
```

For white/black draw flow, enforce atomic sequence:
`draw -> render/show -> apply effects -> grant equipment -> close/continue`.

---

## 5) NPC dice UI + speech bubble sequencing

### Movement bubble
- Before movement roll: show white positionline bubble `「ふります！」` for 1.5s.
- After bubble disappears, trigger movement dice event/UI.

### Attack bubble
- Before attack roll: show `「(PlayerName)さんを攻撃！」` for 1.5s.
- Then trigger attack dice event/UI.
- Remove bubble after dice finalization.

### Pseudocode
```js
async function npcMoveStep(uid){
  await showPositionBubble(uid, "ふります！", { white:true, ms:1500 });
  await emitMoveDiceEvent(uid); // existing diceEvents/move path
  await waitForDiceFinalize("move", uid);
  await waitMs(200); // strict post-finalization delay
  await applyMoveByDice(uid);
}

async function npcAttackStepIfTargetExists(uid){
  const target = chooseAttackTargetInRange(uid);
  if (!target) return;

  await showPositionBubble(uid, `${playerName(target)}さんを攻撃！`, { ms:1500 });
  await emitAttackDiceEvent(uid, target);
  await waitForDiceFinalize("attack", uid);
  hidePositionBubble(uid);
  await waitMs(200); // strict post-finalization delay
  await applyAttackOutcome(uid, target);
}
```

---

## 6) Strict anti-double-attack guarding

### Guard rules
- Attack step can run once per `npcTurnId` only.
- Any room-effect path that can invoke attack must call `requestNpcAttackStep(turnId)` which checks:
  - `actionLock`
  - `isResolvingTurnStep`
  - `hasAttackedThisTurn`
  - `currentStep === "attack"`

### Pseudocode
```js
function canExecuteNpcAttack(turnId){
  return npcGuard.npcTurnId === turnId
    && !npcGuard.cancelled
    && npcGuard.actionLock
    && !npcGuard.isResolvingTurnStep
    && !npcGuard.hasAttackedThisTurn
    && npcGuard.currentStep === "attack";
}

async function guardedNpcAttack(turnId, fn){
  if (!canExecuteNpcAttack(turnId)) return false;
  npcGuard.isResolvingTurnStep = true;
  try {
    await fn();
    npcGuard.hasAttackedThisTurn = true;
    return true;
  } finally {
    npcGuard.isResolvingTurnStep = false;
  }
}
```

---

## 7) 0.2s post-finalization movement timing (strict)

### Required points
Apply icon movement only **after** dice finalization + 200ms for:
- map marker movement,
- HP board movement triggered by NPC attack effects.

### Timing hook
In dice animation completion callback (`onDiceFinalized`):
```js
await waitMs(200);
applyVisualMovement();
```

Never call `applyVisualMovement()` directly from roll-start or mid-animation paths.

---

## 8) Integration checklist

1. Extend `dealIdentityCards` start path:
   - call `ensureComputerModeSeats` before role dealing.
2. Extend `subscribePlayers` parsing:
   - include `isNpc` and `npcProfile` in `latestPlayers` cache.
3. Add host-only watcher in room subscription:
   - `maybeRunComputerModeTurn()` if current turn owner is NPC.
4. Add `NpcTurnScheduler` module-level singleton.
5. Add actor-parameterized helpers:
   - `drawForActor(uid, type)`
   - `applyMoveByDice(uid)`
   - `applyAttackOutcome(uid, targetUid)`
   - `open/close green flow for actor`.
6. Add safe cancellation:
   - cancel running NPC sequence when turn owner changes, room resets, or game ends.

---

## 9) Failure prevention notes

- **No overlapping timers:** all waits awaited in one async chain, no parallel `setTimeout` orchestration for steps.
- **Atomic card flow:** card pipeline wrapped in single guarded async transaction + UI promise chain.
- **Re-entrancy safety:** each async step validates `npcTurnId` at entry and after await boundaries.
- **Host authority only:** only host executes NPC mutations; non-host clients render events only.
