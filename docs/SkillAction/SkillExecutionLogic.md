# SkillExecutionLogic

`SkillExecutionLogic` is the server-authoritative execution wrapper between
`SkillActionLogic` and `BattleCalculatorLogic`.

It converts a confirmed skill request containing entity/config identities into
fully prepared numeric calculation contexts, calculates every target, applies
the results through the owning components, and returns one structured execution
result.

## File Placement

- Planned script: `RootDesk/MyDesk/Logic/SkillExecutionLogic.mlua`
- Script type: `@Logic`
- Execution: server-only internal methods

The Logic is shared by field and battle skill execution. It does not hold
per-battle mutable state.

## Position In The Flow

```text
SkillActor / ControlCharacterUI
  -> SkillActionLogic:RequestSkill(requestId, skillKey, actionContext)
  -> SkillActionLogic validates sender / learned skill / battle permission
  -> SkillExecutionLogic:ExecuteSkill(executionRequest)
       -> resolve sponsor + skill config + targets
       -> build prepared snapshots
       -> build one prepared calculator context per target
       -> BattleCalculatorLogic:CalculateDamage(context)
       -> collect targetResults[]
       -> apply targetResults[] through BattleActorCom
  -> return SkillExecutionResult
  -> presentation / BattleSystem settlement
```

## Responsibilities

- Resolve the authoritative sponsor entity from trusted server context.
- Load the skill row by `SkillKey`.
- Resolve, validate, deduplicate, and freeze the final target list.
- Read `BattleActorCom` values and build immutable sponsor/target snapshots.
- Apply skill-type, single/multi-target, and target-count rules from Config.
- Aggregate additive damage-rate sources into one `DamageMultiplier`.
- Aggregate multiplicative final-damage sources into one
  `FinalDamageMultiplier`.
- Normalize mastery into one `MasteryMultiplier`.
- Build a complete numeric calculator context for each target.
- Calculate all target results before mutating any target.
- Apply approved results through `BattleActorCom` authority.
- Return one structured result containing `TargetResults`.

## Non-Responsibilities

- Do not accept authoritative stats from the client.
- Do not own learned-skill, cooldown, or resource records (`SkillLogic` owns
  those rules).
- Do not own turn permission or settlement (`BattleSystem`).
- Do not implement arithmetic formula stages (`BattleCalculatorLogic`).
- Do not mutate `hp` directly; call `BattleActorCom:ApplyDamage`.
- Do not play animations, effects, sounds, or update UI.

## Public Input Contract

`SkillActionLogic` builds the request after validating sender identity and
execution permission:

```lua
local executionResult = _SkillExecutionLogic:ExecuteSkill({
    RequestId = requestId,
    ExecutionMode = "Battle", -- "Field" | "Battle"
    SponsorEntityId = sponsorEntity.Id,
    SkillKey = actionKey,
    RequestedTargetIds = actionContext.TargetIds,
    TargetPoint = actionContext.TargetPoint,
    HasTargetPoint = actionContext.HasTargetPoint
})
```

Rules:

- `SponsorEntityId` is derived by the server, never trusted from client input.
- Requested target data is untrusted intent. The wrapper resolves, validates,
  deduplicates, and freezes the authoritative final target list.
- Runtime stats, skill power, multipliers, and defense values are not accepted
  in this request. The wrapper resolves them from authoritative sources.

## Snapshot Phase

Snapshots freeze all values used by this one execution so target iteration
order cannot change the formula.

### SponsorSnapshot

```lua
{
    entityId = "...",
    totalAttack = 120,
    masteryMultiplier = 0.85,
    damageRateSources = { 0.10, 0.25 },
    finalDamageSources = { 0.20, 0.10 }
}
```

`totalAttack` is read from the cached value owned by `BattleActorCom`.
The wrapper does not rebuild equipment/buff stats during each target loop.

### SkillSnapshot

```lua
{
    skillKey = "FireBall",
    skillType = "Attack",
    targetType = "Multi",
    baseSkillPower = 1.50
}
```

The wrapper applies skill-type and target-count Config rules and produces the
final `skillPower` used by every target context.

### TargetSnapshot

```lua
{
    entityId = "...",
    totalDefense = 40
}
```

Target-specific values are captured before any result is applied.

## Value-Convergence Rules

The wrapper owns all classification, lookup, fallback, and aggregation.
`BattleCalculatorLogic` receives no source lists and makes no gameplay
decisions.

```text
damageMultiplier =
    1 + sum(all additive DamageRate sources after skill/target rules)

finalDamageMultiplier =
    product(1 + each FinalDamage source after skill/target rules)

masteryMultiplier =
    normalized mastery value in the allowed range (currently 0.3 .. 1.0)

skillPower =
    base skill power after skill-type / target-type / target-count rules
```

Config ownership (values remain TODO):

```text
BattleCalcConfig.SkillTypeAmplify
BattleCalcConfig.TargetCountFalloff
```

These Config rows are read by `SkillExecutionLogic`, not by
`BattleCalculatorLogic`.

## Per-Target Calculator Context

After convergence, each target receives a complete numeric-only context:

```lua
{
    totalAttack = 120,
    skillPower = 1.20,
    damageMultiplier = 1.35,
    finalDamageMultiplier = 1.32,
    masteryMultiplier = 0.85,

    -- Defense input shape remains TODO until the defense formula is selected.
    totalDefense = 40
}
```

The context must not contain:

- `SkillKey`, `SkillType`, `TargetType`, or `TargetCount`
- entity/component references
- Config rows or Config keys
- modifier source lists
- flags requiring Calculator branching

## Two-Phase Execution

### Phase 1 — Calculate all targets

```text
freeze sponsorSnapshot
freeze skillSnapshot
freeze targetSnapshots[]
foreach targetSnapshot:
  converge values
  build numeric context
  calculate result
  append targetResult
```

No HP/resource mutation is allowed in this phase.

### Phase 2 — Apply all targets

```text
foreach targetResult:
  revalidate target entity/component existence
  BattleActorCom:ApplyDamage(targetResult.Calculation.damage)
  record Applied / failure reason
```

This prevents earlier targets from changing the stats used by later targets and
keeps multi-target skills deterministic.

## Result Contract

The wrapper returns the outer execution result. Each target contains the inner
structured result returned by `BattleCalculatorLogic`.

```lua
{
    Success = true,
    Reason = "",
    RequestId = 1,
    ExecutionMode = "Battle",
    SponsorEntityId = "...",
    SkillKey = "FireBall",
    TargetCount = 2,
    TargetResults = {
        {
            TargetEntityId = "...",
            Calculation = {
                damage = 120
            },
            Applied = true,
            ApplyReason = ""
        }
    }
}
```

The final return fields can be expanded later, but `TargetResults` is always an
ordered array matching the frozen target order.

## Failure Policy

- Validation/snapshot failure before calculation: return `Success = false` and
  apply nothing.
- Prepared-context validation failure before calling Calculator: return failure
  and apply nothing.
- Target invalidated between calculate/apply: record `Applied = false` for that
  target; the policy for rolling back other targets remains TODO.
- Log request ID, skill key, sponsor ID, target count, and result count at
  critical checkpoints.

## Planned Interface

```text
ExecuteSkill(executionRequest) -> SkillExecutionResult
ResolveSponsorSnapshot(executionRequest) -> SponsorSnapshot
ResolveSkillSnapshot(skillKey, targetCount) -> SkillSnapshot
ResolveTargetSnapshots(executionRequest, skillSnapshot) -> TargetSnapshot[]
BuildCalculatorContext(sponsorSnapshot, skillSnapshot, targetSnapshot) -> table
ApplyTargetResults(targetResults) -> void
```

## Agent TODO

1. Implement `SkillExecutionLogic.mlua` only after the Config columns and result
   structure are finalized.
2. Route both field and battle execution through `ExecuteSkill`.
3. Replace `BattleSystem:ResolveAction` formula/mutation work with wrapper
   invocation; keep BattleSystem responsible for permission and settlement.
4. Add deterministic tests for target ordering and two-phase calculation.
5. Decide rollback policy when a target disappears between calculate/apply.

## Related Docs

- `docs/SkillAction/SkillActionSystem.md`
- `docs/Calculator/BattleCalculator.md`
- `docs/Actor/BattleActorComponent.md`
- `docs/BattleFlow/BattleSystem.md`
