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
- Do not open or close BattleUI.
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

Current reason keys:

```text
Battle
```

Future reason keys:

```text
Cutscene
Dialogue
SystemMenu
Interaction
```

## Battle Integration

`BattleClientLogic` no longer directly stores or restores
`PlayerControllerComponent.Enable`. Instead:

```text
EnterBattle -> PlayerControlLogic:PushControlLock("Battle")
ExitBattle  -> PlayerControlLogic:PopControlLock("Battle")
```

This allows future systems to stack locks safely.
