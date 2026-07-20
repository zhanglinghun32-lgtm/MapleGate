# Skill Action System

## Goal

One server-authoritative skill request entry for **field and battle**. UI must
not depend on `BattleSystem` for casting. Battle only answers: may this actor
submit a turn action *right now*?

## Ownership

| System | Owns |
|--------|------|
| `SkillActionLogic` | Public gateway, sender identity, routing, accept/reject callbacks |
| `SkillLogic` | Learned skills, config, resources, cooldown rules |
| Shared execution | Targeting, mutations, presentation payloads (field or battle) |
| `ControlCharacterUIComponent` | Skill selection / confirm / range preview UI |
| `BattleSystem` | Turn permission + settlement after completion — **not** skill UI or formulas |
| `BattleUI` | Turn-sequence overlay only — **never** skill requests |

## Request Contract

```lua
_SkillActionLogic:RequestSkill(requestId, actionKey, {
    TargetIds = {},
    TargetPoint = nil,
    HasTargetPoint = false
})
```

Server derives user/sponsor from `senderUserId`. Client battle claims are not trusted.

## Result Contract

- Accepted: `OnSkillRequestAccepted(requestId, actionKey, executionMode)`
- Rejected: `OnSkillRequestRejected(requestId, actionKey, reason)`

## Validation Split

Common (always):

- user / action / entity / `BattleActorCom`
- learned skill + `skillConfig`
- alive, MP/stamina, cooldown (TODO)

Battle-only permission (ask `BattleSystem`, do not reimplement turn loop here):

- active battle for this user
- `AwaitingInput`
- sponsor is `currentActorId`
- `IsOperationAllowed(userId, "SubmitSkill")` when that API exists

Field-only:

- same-map / target validity (initial)
- range / effects (TODO)

## Context Independence

Skills exist without battle. In battle:

```text
RequestSkill
  -> common validation
  -> BattleSystem permission
  -> shared execute
  -> notify BattleSystem completion
  -> BattleSystem settles turn
```

Do not add reusable skill behavior only inside `BattleSystem`.

## UI Callers

```text
ControlCharacterUIComponent -> SkillActionLogic:RequestSkill
```

`BattleUI` must not send skill requests.

Inventory use/throw is owned by `InventoryLogic`, not SkillAction and not battle
payload. In battle, Inventory asks `BattleSystem` for turn permission and spends
action points — same gate pattern as skills.

Legacy: `BattleSystem.RequestAction*` — no new UI callers.

## Agent TODO

1. **`SkillActionLogic.mlua`** — call battle `CanSubmitTurnAction` /
   `IsOperationAllowed("SubmitSkill")` + action-point spend; keep execution
   outside BattleSystem.
2. **`ControlCharacterUIComponent.mlua`** — disable confirm when phase/actor/policy
   denies; no direct `RequestAction*`.
3. **`BattleSystem.mlua`** — shrink embedded skill resolution; permission + settle only.
4. **`InventoryLogic`** — battle use/throw live path; permission key `UseItem` +
   action points; no bag copy in `BattleStartPayload`.
5. Grep UI for `RequestAction` / `RequestBattleAction` and clear live paths.

## Related Docs

- `docs/BattleFlow/BattleSystem.md`
- `docs/BattleFlow/BattleUIComponent.md`
