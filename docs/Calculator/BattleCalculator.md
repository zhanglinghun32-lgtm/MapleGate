# BattleCalculator

`BattleCalculator` is the single place that turns fully prepared numeric inputs
into combat results (damage, defense reduction, level curves, and other
formulas).

It is a **pure calculation** system: given inputs, produce numbers. It does not
read live entity state, mutate resources, play effects, or decide turn order.

## File Placement

- Logic script: `RootDesk/MyDesk/Logic/Battle/BattleCalculatorLogic.mlua`
  (`@Logic`, world-session singleton, stateless calculation gateway).
- Called from the skill cast pipeline by `SkillActionLogic` /
  `SkillActionWrapper` (Field and Battle). See
  [SkillCastPipeline.md](../SkillAction/SkillCastPipeline.md).
- The calculator does not read Config. Its caller must resolve Config rules and
  pass the final numeric values.

This document records responsibility boundaries and the calling contract first.
The concrete pipeline stages and the exact return structure are filled in later.

## Purpose

- Own all battle math formulas in one location: damage, defense reduction,
  critical, hit/evade, level/exp curves, and future combat formulas.
- Receive already-converged numeric values from `SkillActionWrapper`. It does
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

## Cast-time `atk` → Calculator → `damage`

Actors have **no 攻擊力**. Skills author base + coeff against an attribute;
Wrapper evaluates that into `atk` before calling Calculator.

```text
-- Wrapper only (example)
atk = 50 + 0.5 * will

-- Calculator
damage = f(atk, damageMultiplier, finalDamageMultiplier, masteryMultiplier, defense, …)

-- Resolver
ApplyDamage(damage)   -- never receives atk
```

| Concept | Owner | Notes |
|---------|-------|-------|
| Five attributes | Actor / Save | No `atk` on actor |
| Skill base + coeff | Skill Config | e.g. base `50`, coeff `0.5`, attr `will` |
| Cast-time `atk` | `SkillActionWrapper` only | e.g. `50 + 0.5 * will` |
| `damage` | `BattleCalculatorLogic` out | Only value Resolver uses for HP |

Do **not** pass skill base/coeff or attributes into Calculator — only finished
`atk` and other pure scalars. Do **not** cache `atk` on the actor.

## Confirmed Damage Formula (shape)

Calculator starts from Wrapper-supplied `atk` (already includes skill base/coeff):

```text
輸出 ≈ f(atk, damage%, 最終傷害, 熟練度, defense, …)
```

| Factor | Context field | Notes |
|--------|---------------|-------|
| 施放攻擊力 | `atk` | From Wrapper only. Not on actor. Not passed to Resolver. |
| 傷害% | `damageMultiplier` | Converged outside as `1 + Σ damageRate` |
| 最終傷害 | `finalDamageMultiplier` | Converged outside as `Π (1 + finalDamage)` |
| 熟練度 | `masteryMultiplier` | Normalized outside; range currently `0.3 .. 1.0` |

Aggregation difference (important):

```text
傷害%   : damageMultiplier = 1 + (r1 + r2 + r3 + ...)      -- 加法
最終傷害 : finalDamageMultiplier = (1 + f1) * (1 + f2) * ... -- 乘法
```

Defense subtraction happens **after** attack output (later stage).

## Calling Contract — 方案一：Context Table

Every formula entry takes **one context table** instead of a long positional
parameter list. This avoids "parameter explosion" as the game grows.

```lua
local result = _BattleCalculatorLogic:CalculateDamage({
    atk = 85,                  -- e.g. Wrapper: 50 + 0.5 * will
    damageMultiplier = 1.35,
    finalDamageMultiplier = 1.32,
    masteryMultiplier = 0.85,

    -- Defense input shape remains TODO.
    totalDefense = 40
})
-- result.damage is what SkillActionResolver applies
```

Rules for the context table:

- **Numbers only.** No entities, five attributes, skill base/coeff rows, or
  decision flags.
- Wrapper must supply finished `atk` (skill formula already evaluated).
- Calculator does not infer defaults or re-read skill Config.
- Invalid / missing required input is a Wrapper contract failure.
- Resolver must not be given this context — only `{ damage, ... }` results.

## Internal Implementation — 方案四：Damage Pipeline

Internally `CalculateDamage` runs the context through an **ordered list of
stages**. Each stage is a small function `(context, acc) -> acc` that reads the
context and updates a running accumulator. This keeps each formula factor
isolated, testable, and reorderable.

```lua
-- conceptual shape (stages are data, not one giant expression)
local PIPELINE = {
    Stage_Atk,             -- acc = atk  (already = skill base + coeff * attr)
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
| 1 | `Stage_Atk` | `atk` | `acc = atk` | ✅ atk built in Wrapper |
| 2 | `Stage_DamagePercent` | `damageMultiplier` | `acc = acc * damageMultiplier` | ✅ aggregation outside |
| 3 | `Stage_FinalDamage` | `finalDamageMultiplier` | `acc = acc * finalDamageMultiplier` | ✅ aggregation outside |
| 4 | `Stage_Mastery` | `masteryMultiplier` | `acc = acc * masteryMultiplier` | ✅ shape; normalize outside |
| 5 | `Stage_Defense` | input shape TODO | apply the fixed defense formula | ⏳ formula TODO |
| 6 | `Stage_Clamp` | no gameplay classification | apply the fixed floor/cap/round rule | ⏳ TODO |

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

### D1. Skill base/coeff → atk — **DECIDED: wrapper owns it**

`SkillActionWrapper` evaluates skill formulas such as `50 + 0.5 * will` into
`atk`, and converges other multipliers. Calculator never sees attributes, skill
base, or skill coeff columns. Resolver never sees `atk`.

### D2. 熟練度 (mastery) 算法

Shape confirmed: `SkillActionWrapper` supplies a normalized `0.3 .. 1.0`
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
  -> SkillActionWrapper: atk = skillBase + skillCoeff * attr
  -> DamageRequest { atk, … }
  -> BattleCalculatorLogic:CalculateDamage → { damage, … }
  -> SkillActionResolver:ApplyDamage(damage)   -- no atk
  -> @Sync / OnSyncProperty refresh UI bars
```

`BattleCalculator` sits between finished `atk` and apply-time `damage`. It never
crosses into attribute lookup, skill Config, or mutation.

## Agent TODO

1. Implement `CalculateDamage` numeric pipeline from `atk` seed.
2. Keep DamagePercent(加法) / FinalDamage(乘法) / Mastery as scalar stages.
3. Treat missing `atk` as Wrapper contract failure.
4. Fill in `Stage_Defense` and structured return fields once decided.
5. Keep Resolver API free of `atk` / `DamageRequest`.

## Related Docs

- `docs/Actor/ActorVariableExplain.md` — no actor 攻擊力; Wrapper owns `atk`.
- `docs/SkillAction/SkillCastPipeline.md` — value lifetime table.
- `docs/BattleFlow/BattleSystem.md` — turn permission / settlement.
- `docs/SkillAction/SkillActionSystem.md` — request gateway.
