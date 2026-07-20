# PlayerControlLogic

## Purpose

`PlayerControlLogic` owns local player operation locks.

It solves the problem where several systems may want to disable player control
at the same time. A single boolean is unsafe because one system can restore
control while another system still needs it locked.

## Runtime Owner

Path:

```text
RootDesk/MyDesk/Logic/PlayerControlLogic.mlua
```

`PlayerControlLogic` is client-only in practice because local input, local
movement stopping, and local UI presentation happen on the client.

## Responsibilities

- Store active control lock reasons.
- Disable the local `PlayerControllerComponent` when at least one reason is
  active.
- Stop residual movement when a lock is added.
- Restore the controller to the pre-lock state only after all reasons are
  removed.
- Expose query helpers for other client-side systems.

## Non-Responsibilities

- Do not decide when battle starts or ends.
- Do not open or close `BattleUI` (turn-sequence overlay) or `ControlCharacterUI`.
- Do not move the player between maps.
- Do not serialize player data.
- Do not own Actor HP, MP, stamina, or battle stats.

## Current API

```text
PushControlLock(reasonKey)
PopControlLock(reasonKey)
ClearControlLocks()
HasControlLock(reasonKey)
CanControl()
GetActiveLockCount()
```

Current / planned reason keys:

```text
Battle                 -- whole battle session (planned; align Enter/ExitBattle)
BattlePresentation     -- used today by BattleSkillPresentationComponent
Cutscene
Dialogue
SystemMenu
Interaction
```

## Battle Integration

Local movement locks are **not** the same as battle operation policy.

| Concern | Owner |
|---------|-------|
| Disable `PlayerController` while presenting / in battle modes that need it | `PlayerControlLogic` reason keys |
| May the player confirm a skill / edit party / switch member? | `BattleSystem` operation policy, enforced by Skill / Party / UI owners |

```text
EnterBattle / ExitBattle     -> Push/Pop "Battle" when product wants full move lock
Skill presentation           -> Push/Pop "BattlePresentation"
Party edit in battle         -> PartyLogic denies; do not invent a Party lock inside BattleUI
```

### Agent TODO

1. Align `BattleClientLogic` Enter/Exit with `PushControlLock("Battle")` /
   `PopControlLock("Battle")` if full field move lock during battle is required.
2. Keep presentation lock on `BattleSkillPresentationComponent` (`BattlePresentation`).
3. Do not put party-formation blocks into `PlayerControlLogic`; those stay on
   `PartyLogic` + battle `IsOperationAllowed`.

## Related

- `docs/BattleFlow/BattleSystem.md` — operation lock policy
- `docs/BattleFlow/BattleFlowLogic.md`
