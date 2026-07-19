# Skill Action System

## Goal

Provide one server-authoritative skill request entry point that works both inside
and outside a turn-based battle. UI code must not depend on `BattleSystem`.

## Ownership

- `SkillActionLogic`: public request gateway, sender identity, common validation,
  context routing, and accepted/rejected client callbacks.
- `SkillLogic`: learned-skill, config, actor-state, resource, and cooldown rules.
- `BattleSystem`: battle membership, phase, current actor, turn permission,
  turn-start/turn-end scheduling, and completion waiting.
- Shared action execution: authoritative targeting, resource consumption,
  gameplay mutations, and reusable presentation payloads regardless of context.
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
- battle turn permission; battle-specific target restrictions may be supplied as
  policy, but reusable target resolution does not belong to the turn scheduler

Field-only validation:

- same-map and target validity (initial implementation)
- field range/collision rules (TODO)
- resource consumption and gameplay effects (TODO)

## Context Independence

Movement, avatar actions, and skill presentation do not require an active battle.
`SkillActionLogic` selects the execution context after common validation:

- Field: execute immediately under field rules.
- Battle: ask `BattleSystem` whether the sponsor may act this turn, then execute
  through the same shared action capability and report completion to the session.

Do not add a reusable skill behavior only inside `BattleSystem`. Battle-specific
code should describe timing or policy; the action itself belongs to the shared
skill/action system.

## UI Callers

Current UI path:

```text
ControlCharacterUIComponent
  -> SkillActionLogic:RequestSkill
```

`BattleUI` does **not** send skill requests. It is the turn-sequence overlay only.
See `docs/BattleFlow/BattleUIComponent.md`.

## Migration

1. ~~Route UI through `SkillActionLogic`.~~ Done for `ControlCharacterUI`.
2. Keep legacy `BattleSystem.RequestAction*` APIs temporarily for compatibility.
3. Battle requests already enter through `RequestValidatedActionForUser`.
4. Implement authoritative field effects and resource consumption.
5. Remove legacy direct UI-to-`BattleSystem` request APIs once no callers remain.
   Do not add new UI callers to `RequestAction*`.

