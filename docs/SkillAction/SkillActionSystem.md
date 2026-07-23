# Skill Action System

## Goal

One server-authoritative skill request entry for **field and battle**. UI must
not depend on `BattleSystem` for casting. Battle only answers: may this actor
submit a turn action *right now*?

Cast math and apply use the shared pipeline in
[SkillCastPipeline.md](SkillCastPipeline.md).

## Ownership

| System | Owns |
|--------|------|
| `SkillActionLogic` | Public gateway, sender identity, routing, `SkillCastContext`, accept/reject callbacks, pipeline orchestration |
| `SkillActionWrapper` | `atk` + BeforeDamage effects → pure `DamageRequest`; After* → `EffectRequest` |
| `BattleCalculatorLogic` | `atk` (+ scalars) → `damage`; no attrs / Config |
| `SkillActionResolver` | Apply `damage` / buff / move / anim — **never `atk`** |
| `SkillLogic` | Learned skills, config, resources, cooldown rules |
| `ControlCharacterUIComponent` | Skill selection / confirm / range preview UI |
| `BattleSystem` | Turn permission + settlement after completion — **not** skill UI or formulas |
| `BattleUI` | Turn-sequence overlay only — **never** skill requests |

## Cast Pipeline (summary)

```text
SkillActionLogic
  -> SkillCastContext
SkillActionWrapper
  -> DamageRequest / EffectRequest
BattleCalculatorLogic
  -> calculation results
SkillActionResolver
  -> apply damage / buff / move / anim events
```

Details and table shapes: [SkillCastPipeline.md](SkillCastPipeline.md).

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
- Execution: resolver / pipeline result (fields TODO); presentation may follow
  accept on the client for Field mode.

## Validation Split

Common (always):

- user / action / entity
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
  -> BuildCastContext + ExecuteCastPipeline
       (Wrapper -> Calculator -> Resolver)
  -> notify accept / BattleSystem completion
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

1. Implement `BuildCastContext` fields (sponsor, targets, skillEffect rows).
2. Implement `SkillActionWrapper:BuildRequests` (equip / buff / skillRange).
3. Implement `BattleCalculatorLogic:CalculateDamage` stages per
   [BattleCalculator.md](../Calculator/BattleCalculator.md).
4. Implement `SkillActionResolver:Resolve` (apply + presentation hooks).
5. Wire `ExecuteCastPipeline` into Field and Battle paths inside
   `RequestSkill` / `RouteBattleSkill`.
6. Shrink `BattleSystem` embedded skill resolution to permission + settle only.

## Related Docs

- [SkillCastPipeline.md](SkillCastPipeline.md)
- [SkillEffect/SkillEffectSystem.md](../SkillEffect/SkillEffectSystem.md)
- [Calculator/BattleCalculator.md](../Calculator/BattleCalculator.md)
- [BattleFlow/BattleSystem.md](../BattleFlow/BattleSystem.md)
