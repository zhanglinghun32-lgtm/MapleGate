# BattleSystem Component

`BattleSystem` (`Battle/BattleSystem.mlua`) owns **one active turn-based battle
session**. Its job is narrow:

```text
1. Whose turn is it?          -> currentActorId + phase
2. Turn start / turn end      -> hooks, queue cleanup, next actor
3. Settlement                 -> apply accepted action outcomes to session state,
                                 then victory / defeat / escape / cancel
```

Everything else — skill casting, item use, skill/item UI, opening Party /
System / navigation UIs — belongs to other systems. Battle only **gates** those
systems while a session is active.

## Architectural Boundary

`BattleSystem` answers:

| Question | Owner |
|----------|-------|
| Is a battle active? | `BattleSystem` |
| Which phase / whose turn? | `BattleSystem` |
| May this actor submit an action *now*? | `BattleSystem` (permission) |
| What runs at turn start / turn end? | `BattleSystem` |
| Did the battle end, and with what result? | `BattleSystem` |
| Cast skill / use item / show those UIs? | **Not battle** — `SkillActionLogic`, inventory, `ControlCharacterUI`, etc. |
| Open Party / System / nav UI? | **Not battle** — `UIManagerLogic` + each UI owner |
| Block party formation change in battle? | Policy published by battle; enforced by `PartyLogic` / Party UI |

Battle constrains **when** shared systems may run. It does not become the owner
of those systems.

## Responsibilities

- Receive `StartBattle(payload)` from `BattleFlowLogic`.
- Register battle actors and own the turn queue.
- Own `@Sync battlePhase` / `@Sync currentActorId`.
- Run turn-start and turn-end hooks.
- Expose **operation lock / interaction policy** so other systems can ask
  whether an operation is allowed (skill confirm, item use, party edit, etc.).
- Accept *permission checks* from `SkillActionLogic` (and future item gateway):
  active session, correct phase, sponsor is current actor, owner user matches.
- Wait for shared action / presentation completion, then **settle**:
  death cleanup, victory/defeat/escape/cancel, advance turn or finish.
- Report final battle result to `BattleFlowLogic`.

## Non-Responsibilities

- Do **not** implement skill casting, targeting UI, range preview, or item use.
- Do **not** own skill/item button binding or description popups.
- Do **not** open or close Party / System / navigation UIs.
- Do **not** own formation edits, inventory writes, or field movement capability.
- Do **not** load encounter Config, save data, or post-battle scene flow.
- Do **not** act as the public skill request gateway — UI calls `SkillActionLogic`.
- Do **not** act as the public **battle start** gateway — callers use
  `BattleFlowLogic:RequestStartBattle` (dialogue / player attack / monster
  detect adapters). See `docs/BattleFlow/BattleFlowLogic.md`.
- Do **not** make movement or skills require an active battle session to exist
  as systems; battle only adds turn permission when a session is active.

## Operation Lock Policy

Battle must be able to **lock or allow** player operations without owning the
UI that performs them.

Recommended server query surface (name can vary in implementation):

```text
IsBattleActive(userId) -> boolean
GetBattlePhase(userId) -> string
GetCurrentActorId(userId) -> string
CanSubmitTurnAction(userId, actorId) -> boolean
IsOperationAllowed(userId, operationKey) -> boolean
```

Suggested `operationKey` values:

| operationKey | Default in battle | Enforced by |
|--------------|-------------------|-------------|
| `SubmitSkill` | Allowed only when `AwaitingInput` and actor is current; spends action points | `SkillActionLogic` + `BattleSystem` permission |
| `UseItem` | Same turn gate; **InventoryLogic** owns use/throw; spends action points | `InventoryLogic` + `BattleSystem` permission |
| `OpenPartyUI` | Allowed (UI may open) | `UIManagerLogic` / Party UI |
| `EditPartyFormation` | **Blocked** | `PartyLogic` / `PartyUIComponent` |
| `SwitchFieldMember` | **Blocked** | `PartyLogic` / ControlCharacter roster |
| `OpenSystemMenu` | Allowed | System menu owner |
| `SaveAndReturnToMenu` | Product policy (often allowed) | `PlayerDataLogic` / System menu |

Rules:

- Opening a UI is not the same as mutating party/battle state.
- Battle publishes policy; feature owners enforce it on their `Request*` paths.
- Client UI should disable illegal buttons using the same policy / synced phase,
  but server validation is authoritative.

## Turn Lifecycle

```text
Initializing
  -> TurnStart
     -> turn-start effects / cleanup
     -> if battle ended, Cleanup
  -> AwaitingInput
     -> only currentActorId may submit a turn action
     -> operation lock allows SubmitSkill / UseItem for that actor only
     -> skill and item actions spend that turn's action-point budget
  -> ResolvingAction / PresentingAction
     -> lock turn actions while shared systems execute + present
  -> TurnEnd
     -> settle deaths / end checks
  -> next TurnStart or Cleanup -> report result
```

## How Skills And Items Interact With Battle

```text
ControlCharacterUI / Item UI
  -> SkillActionLogic or Item gateway (NOT BattleSystem UI)
  -> common validation
  -> if in battle: ask BattleSystem permission (CanSubmitTurnAction / IsOperationAllowed)
  -> shared execution pipeline does the cast/use
  -> report completion to BattleSystem
  -> BattleSystem settles and advances the turn
```

`BattleSystem:RequestAction*` is legacy. New callers must not use it from UI.

## Synced Display State

| Property | Purpose |
|----------|---------|
| `@Sync battlePhase` | Phase cues + client-side input gating |
| `@Sync currentActorId` | Whose turn; who may confirm a turn action |

Do **not** use `@Sync` to open `BattleUI`. Overlay open/close stays on
`BattleFlowLogic` Client RPC -> `BattleClientLogic` -> `UIManagerLogic`.

Forward `OnSyncProperty` through `BattleClientLogic` to:

- `BattleUIComponent` — turn-sequence overlay only
- `ControlCharacterUIComponent` — enable/disable skill confirm from phase/actor
  (display only; authority stays on server)

## Settlement

Settlement means session-level consequences after an accepted turn action (or
turn-start/end hooks), not skill formula ownership:

- receive the already-applied `SkillExecutionResult` from
  `SkillExecutionLogic`
- query `IsDead()` on that component
- remove dead actors from the turn queue
- check victory / defeat / escape / cancel
- advance to the next turn or finish and notify `BattleFlowLogic`

`SkillExecutionLogic` calculates and applies mutations through
`BattleActorCom`. Battle **schedules** when that execution counts as a completed
turn and whether the battle ends. `BattleCalculatorLogic` only returns numeric
formula results.

## Actor Registry

Target shape:

```text
BattleSystem
- activeActorIds
- actorMap: actorId -> { EntityId, TeamId, ControlMode, ... }   -- roster only
- resolve: actorId -> Entity -> BattleActorCom                 -- live HP/MP/stats
- currentActorId
- battleQueue -> BattleQueue on the same entity
```

```text
UI HP bar  -> BattleActorCom.hp / maxHp on the entity
Death      -> BattleActorCom:IsDead()
Not        -> actorMap[actorId].Hp   (forbidden as authority)
```

Every registered unit has a live `EntityId`. No config-only phantom actors.

Resource mutations go through actor interfaces (`ApplyDamage`, `UseMp`, …), not
direct field writes on the session table.

### LEGACY — current code to rewrite

These paths still treat payload / `actorMap` tables as if they held live combat
state. Marked for change; do not extend them.

| Location | Why legacy |
|----------|------------|
| `BattleSystem.mlua` `RegisterActorsFromPayload` / `RegisterActorEntry` | Stores full payload row (including copied `hp`) into `actorMap` as the session record. Target: store roster ids + `EntityId` only; resolve `BattleActorCom` for resources |
| `BattleSystem.mlua` targeting helpers reading `actorMap[actorId]` position/type as if table were the actor | Prefer entity transform / component |
| `BattleSystem.mlua` alive / end checks that ignore `BattleActorCom:IsDead()` | Stub “everyone alive” / table fields — must query component |
| `BattleQueue.mlua` `RemoveDeadActors` placeholder | Comment says wait for `IsDead()` — implement via entity component |
| `BattleFlowLogic.mlua` `BuildFieldActorEntry` copying `hp`/`maxHp`/… into payload | LEGACY bootstrap; payload should be roster (`EntityId`) not resource dump |
| UI that would bind bars to `battleSystem.actorMap.*.hp` | Must bind entity `BattleActorCom` (`@Sync` or query) |

## Battle Keys

Enum-like strings live in `Logic/Battle/BattleKeys.mlua` (`_BattleKeys`).
Content keys (`normalAttack`, item keys, …) stay in Config.

## Agent TODO — Scripts And UI To Change Later

Do **not** expand battle ownership while doing these. Prefer moving logic *out*
of `BattleSystem` when unsure.

### High priority — align code with this boundary

1. **`RootDesk/MyDesk/Battle/BattleSystem.mlua`**
   - Keep: phase, current actor, turn start/end, permission checks, settlement,
     finish callback.
   - Extract / stop growing: skill target-range expansion, `ResolveAction` skill
     formulas/mutations, skill presentation ownership. Route shared execution
     through `SkillExecutionLogic`.
   - Add: explicit `IsOperationAllowed` / `CanSubmitTurnAction` (or equivalent)
     used by Skill / Party / Item gateways.
   - Mark `RequestAction*` as legacy; ensure no new UI callers.

2. **`RootDesk/MyDesk/Logic/Skill/SkillActionLogic.mlua`**
   - Remain the only skill request gateway from UI.
   - Query battle permission, then call `SkillExecutionLogic`; do not embed turn
     scheduling or formula work.

3. **`RootDesk/MyDesk/UI/battle/ControlCharacterUIComponent.mlua`**
   - Keep skill select / confirm / range preview here (not in BattleUI).
   - Gate confirm with battle phase/actor sync + server reject path.
   - Remove any remaining direct `BattleSystem:RequestAction*` calls if present.

4. **`RootDesk/MyDesk/UI/battle/BattleUIComponent.mlua`** + **`ui/BattleUI.ui`**
   - Strip skill hotbar / confirm / description ownership if still present.
   - Keep only turn-sequence widgets (queue strip, phase cues, banners).
   - Wire open/close from `BattleClientLogic` (overlay only).

5. **`RootDesk/MyDesk/Logic/Battle/BattleClientLogic.mlua`**
   - Open/Close `BattleUI` overlay on enter/exit.
   - Forward sync to BattleUI + ControlCharacterUI.
   - Do not own skill casting UI.
   - Align operation locks with `PlayerControlLogic` (see that doc’s TODO).

### Party / System UI — battle must not own open/close; must block some ops

6. **`PartyLogic` / future `PartyUIComponent` / `ui/PartyUI.ui` (or ControlCharacter PartyMemberList)**
   - Allow opening party UI during battle if product wants it.
   - **Block** `EditPartyFormation` / `SwitchFieldMember` while battle active
     via `BattleSystem` policy (or `PartyLogic:CanEditParty` reading battle state).

7. **`UI/SystemMenuUI` scripts + `ui` system menu**
   - May stay open during battle.
   - Enforce any battle-blocked menu actions on the System / PlayerData side,
     not inside `BattleSystem` UI code.

### Follow-ups

8. **Item use gateway** (new Logic, mirror `SkillActionLogic`) — turn action +
   battle permission; UI not under `BattleUI`.
9. **`BattleActorCom.mlua`** — add `ApplyDamage`, `IsDead`, …; keep `hp`/`maxHp`
   (current vs max) as the only resource authority UI and battle both use.
10. **LEGACY cleanup** — rewrite `RegisterActorEntry` / `BuildFieldActorEntry` /
    `BattleQueue` death checks so they no longer treat payload `Hp` as live state.
11. **Remove legacy** UI→`BattleSystem.RequestAction*` paths after grep shows
    zero callers.

## Related Docs

- `docs/BattleFlow/BattleFlowLogic.md` — enter / exit / rewards
- `docs/BattleFlow/BattleUIComponent.md` — overlay layering
- `docs/SkillAction/SkillActionSystem.md` — skill gateway
- `docs/Party/PartySystem.md` — party ops blocked in battle
- `docs/PlayerControl/PlayerControlLogic.md` — local control locks
