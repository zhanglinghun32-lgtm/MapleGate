# Skill Action System

## Goal

Provide one server-authoritative skill request entry point that works both inside
and outside a turn-based battle. UI code must not depend on `BattleSystem`.

## Ownership

- `SkillActionLogic`: public request gateway, sender identity, common validation,
  context routing, and accepted/rejected client callbacks.
- `SkillLogic`: learned-skill, config, actor-state, resource, and cooldown rules.
- `BattleSystem`: battle membership, phase, current actor, battle targeting,
  turn mutation, and presentation acknowledgement.
- Field execution: world target validation, resource consumption, gameplay
  mutations, and field presentation. Phase 1 implements validation and
  presentation; authoritative field effects remain TODO.
- `ControlCharacterUIComponent`: selection and request presentation only.

## Request Contract

```lua
_SkillActionLogic:RequestSkill(requestId, actionKey, {
    TargetIds = {},
    TargetPoint = nil,
    HasTargetPoint = false
})
```

The server derives the user and sponsor entity from `senderUserId`. Client
sponsor ids and client claims about battle state are never trusted.

## Result Contract

- Accepted: `OnSkillRequestAccepted(requestId, actionKey, executionMode)`
- Rejected: `OnSkillRequestRejected(requestId, actionKey, reason)`

Client requests are asynchronous. Dispatch success is not action acceptance.

## Validation Split

Common validation:

- non-empty user/action
- user entity and `BattleActorCom`
- learned skill
- `skillConfig` row
- alive actor
- sufficient MP/stamina
- cooldown (TODO)

Battle-only validation:

- active battle and matching owner
- `AwaitingInput` phase
- sponsor is the current actor
- battle target rules

Field-only validation:

- same-map and target validity (initial implementation)
- field range/collision rules (TODO)
- resource consumption and gameplay effects (TODO)

## Migration

1. Route UI through `SkillActionLogic`.
2. Keep legacy `BattleSystem.RequestAction*` APIs temporarily.
3. Route battle requests through `RequestValidatedActionForUser`.
4. Implement authoritative field effects and resource consumption.
5. Remove legacy direct UI-to-BattleSystem request APIs after all callers migrate.

