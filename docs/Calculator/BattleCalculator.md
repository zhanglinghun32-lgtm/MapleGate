# BattleCalculator

`BattleCalculator` is the single place that turns fully prepared numeric inputs
into combat results (damage, defense reduction, level curves, and other
formulas).

It is a **pure calculation** system: given inputs, produce numbers. It does not
read live entity state, mutate resources, play effects, or decide turn order.

## File Placement

- Logic script: `RootDesk/MyDesk/Logic/BattleCalculatorLogic.mlua`
  (`@Logic`, world-session singleton, stateless calculation gateway).
- The calculator does not read Config. Its caller must resolve Config rules and
  pass the final numeric values.

This document records responsibility boundaries and the calling contract first.
The concrete pipeline stages and the exact return structure are filled in later.

## Purpose

- Own all battle math formulas in one location: damage, defense reduction,
  critical, hit/evade, level/exp curves, and future combat formulas.
- Receive already-converged numeric values from `SkillExecutionLogic`. It does
  not know where those values came from.
- Return a structured result that callers apply through the proper authority
  (e.g. `BattleActorCom:ApplyDamage`).

## Responsibilities

- Compute damage from complete numeric inputs.
- Execute arithmetic stages in a fixed order.
- Own balance formula shape (which factors multiply, which add).
- Return a structured, inspectable result.

## Non-Responsibilities

- Do not read or write `hp` / `mp` / `stamina` (that is `BattleActorCom`).
- Do not aggregate stats, modifier lists, or Config rows.
- Do not look up `skillKey`, `skillType`, `targetType`, or `targetCount`.
- Do not decide single/multi-target amplification, fallback values, or clamps
  that belong to input preparation.
- Do not resolve entities/components or validate targets.
- Do not apply results, play hit effects, or emit events.
- Do not decide turn permission or targeting.
- Do not hold per-battle state.

## Confirmed Damage Formula

The attack-output formula (before defense subtraction) is:

```text
輸出 = 總攻擊力 * 技能倍率 * 傷害% * 最終傷害 * 熟練度
damage = totalAttack * skillPower * damageMultiplier
       * finalDamageMultiplier * masteryMultiplier
```

| Factor | Context field | Aggregation | Notes |
|--------|---------------|-------------|-------|
| 總攻擊力 | `totalAttack` | — | Already aggregated **outside** (base + equip + buff). Calculator just reads it. |
| 技能倍率 | `skillPower` | — | Final coefficient after skill/target rules. |
| 傷害% | `damageMultiplier` | — | Already converged as `1 + Σ damageRate`. |
| 最終傷害 | `finalDamageMultiplier` | — | Already converged as `Π (1 + finalDamage)`. |
| 熟練度 | `masteryMultiplier` | — | Already normalized; expected range currently `0.3 .. 1.0`. |

Aggregation difference (important):

```text
傷害%   : damageMultiplier = 1 + (r1 + r2 + r3 + ...)      -- 加法
最終傷害 : finalDamageMultiplier = (1 + f1) * (1 + f2) * ... -- 乘法
```

The equations above describe how `SkillExecutionLogic` prepares the two
multipliers. `BattleCalculatorLogic` receives only their resulting numbers.
Defense subtraction happens **after** this attack output (kept as a later stage).

## Calling Contract — 方案一：Context Table

Every formula entry takes **one context table** instead of a long positional
parameter list. This avoids "parameter explosion" as the game grows.

```lua
local result = _BattleCalculatorLogic:CalculateDamage({
    totalAttack = 120,
    skillPower = 1.20,
    damageMultiplier = 1.35,
    finalDamageMultiplier = 1.32,
    masteryMultiplier = 0.85,

    -- Defense input shape remains TODO.
    totalDefense = 40
})
```

Rules for the context table:

- The context contains **numbers only**. No entities, Config rows/keys, modifier
  lists, target classifications, or decision flags.
- `SkillExecutionLogic` must supply every required value.
- The calculator does not infer missing values or choose neutral defaults.
- The calculator does not clamp/normalize prepared multipliers. Invalid input is
  a caller contract violation.
- Adding a new formula factor requires the wrapper to converge it first, then
  add one numeric context field and one arithmetic pipeline stage.

## Internal Implementation — 方案四：Damage Pipeline

Internally `CalculateDamage` runs the context through an **ordered list of
stages**. Each stage is a small function `(context, acc) -> acc` that reads the
context and updates a running accumulator. This keeps each formula factor
isolated, testable, and reorderable.

```lua
-- conceptual shape (stages are data, not one giant expression)
local PIPELINE = {
    Stage_TotalAttack,     -- acc = totalAttack
    Stage_SkillPower,      -- acc = acc * skillPower
    Stage_DamagePercent,   -- acc = acc * damageMultiplier
    Stage_FinalDamage,     -- acc = acc * finalDamageMultiplier
    Stage_Mastery,         -- acc = acc * masteryMultiplier
    -- ---- attack output done; below are post-output (TODO / open) ----
    Stage_Defense,         -- defense arithmetic; exact input/formula TODO
    Stage_Clamp,           -- floor (min 1?), cap, round to integer
}

function BattleCalculatorLogic:CalculateDamage(context)
    local acc = self:NewAccumulator(context)
    for _, stage in ipairs(self.pipeline) do
        acc = stage(context, acc)
    end
    return self:BuildResult(context, acc)
end
```

Why a pipeline (vs one big expression):

- Each factor is one small, named stage — easy to read, unit-test, and tweak.
- Order is explicit and adjustable (e.g. crit before or after defense).
- New mechanics插入一個 stage，不動既有 stage。
- The pipeline contains arithmetic only; gameplay variants must arrive as
  already-converged numeric inputs.

### Pipeline Stages

Confirmed stages (1–5) implement the attack-output formula. Stages 6+ are the
post-output steps that are still open (see Boundary Decisions).

| # | Stage | Reads from context | Effect on accumulator | Status |
|---|-------|--------------------|-----------------------|--------|
| 1 | `Stage_TotalAttack` | `totalAttack` | `acc = totalAttack` | ✅ confirmed |
| 2 | `Stage_SkillPower` | `skillPower` | `acc = acc * skillPower` | ✅ confirmed |
| 3 | `Stage_DamagePercent` | `damageMultiplier` | `acc = acc * damageMultiplier` | ✅ confirmed; aggregation is outside |
| 4 | `Stage_FinalDamage` | `finalDamageMultiplier` | `acc = acc * finalDamageMultiplier` | ✅ confirmed; aggregation is outside |
| 5 | `Stage_Mastery` | `masteryMultiplier` | `acc = acc * masteryMultiplier` | ✅ shape confirmed; normalization is outside |
| 6 | `Stage_Defense` | input shape TODO | apply the fixed defense formula | ⏳ formula TODO |
| 7 | `Stage_Clamp` | no gameplay classification | apply the fixed floor/cap/round rule | ⏳ TODO |

> Additional stages (critical / element / damage-taken) are only added once the
> open decisions below are settled — insert them as new stages without touching
> the confirmed ones.

## Return Value — Structured Result (TODO — 結構之後補上)

`CalculateDamage` returns a **structured table**, not a bare integer, so callers
and UI can show breakdowns (crit, blocked, element) without recomputing.

```lua
-- placeholder shape; final fields defined later
{
    damage      = 0,      -- final integer damage to apply
    isCritical  = false,
    -- TODO: more fields (rawDamage, blockedByDefense, elementMultiplier,
    --       stageBreakdown, ...) to be finalized
}
```

Rules for the result:

- The result is **data only**. The caller decides what to do with it
  (`BattleActorCom:ApplyDamage(result.damage)`, show floating text, etc.).
- Keep a stable `damage` field so early callers work before the full breakdown
  is finalized.

## Boundary Decisions

### D1. Skill/target amplification — **DECIDED: wrapper owns convergence**

`SkillExecutionLogic` reads `skillType`, `targetType`, `targetCount`, and Config,
then supplies final `skillPower` and `damageMultiplier`. The calculator never
sees those classifications.

### D2. 熟練度 (mastery) 算法

Shape confirmed: `SkillExecutionLogic` supplies a normalized `0.3 .. 1.0`
`masteryMultiplier`; `Stage_Mastery` only multiplies it. The source algorithm
(fixed per skill level? grows with use? config curve?) remains TODO.

### D3. 爆擊 / 屬性 / 承傷 是否納入此公式？

Not in the confirmed 5-factor chain. When needed, the wrapper first supplies
their final numeric factors, then the calculator adds arithmetic stages without
classification/config branches.

## Other Formula Entries

Same numeric-context + (optional) pipeline pattern applies as formulas are added:

- `CalculateDefenseReduction(context)` — TODO
- `CalculateLevelCurve(context)` — exp/level growth, TODO
- `CalculateHitChance(context)` — hit vs evade, TODO

Each returns its own structured result. Shared helpers (clamp, round, rng) live
in the calculator.

## Interaction Flow

```text
SkillActionLogic validates the confirmed request
  -> SkillExecutionLogic resolves sponsor / skill / targets
  -> SkillExecutionLogic converges all source values
  -> build numeric-only context table (方案一)
  -> BattleCalculatorLogic:CalculateDamage(context)   (方案四 pipeline inside)
  -> returns structured result
  -> SkillExecutionLogic collects all target results
  -> BattleActorCom:ApplyDamage(result.damage)        (authority applies it)
  -> @Sync / OnSyncProperty refresh UI bars
```

`BattleCalculator` sits strictly between "all numeric values are converged" and
"the wrapper applies results". It never crosses into lookup, classification,
aggregation, validation, or mutation.

## Agent TODO

1. **`BattleCalculatorLogic.mlua`** — `@Logic`; implement `CalculateDamage(context)`
   with a numeric-only pipeline and no Config/entity lookups.
2. Implement confirmed stages 1–5 (`TotalAttack → SkillPower → DamagePercent(加法)
   → FinalDamage(乘法) → Mastery`) using scalar multiplier inputs only.
3. Treat missing/malformed required input as a wrapper contract failure; do not
   add gameplay fallback decisions inside Calculator.
4. Fill in `Stage_Defense` input and formula once decided.
5. Finalize the structured return fields once presentation needs are known.

## Related Docs

- `docs/Actor/BattleActorComponent.md` — provides prepared `total*` values.
- `docs/SkillAction/SkillExecutionLogic.md` — resolves and converges all inputs.
- `docs/BattleFlow/BattleSystem.md` — turn permission / settlement.
- `docs/SkillAction/SkillActionSystem.md` — where skill inputs originate.
