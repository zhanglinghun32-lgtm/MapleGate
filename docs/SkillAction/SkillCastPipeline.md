# Skill Cast Pipeline

Context-independent skill cast flow for **Field (open world)** and **Battle**.
Only the formula step reuses the battle calculator; request routing, request
expansion, and result application must not live inside `BattleSystem`.

## Pipeline

```text
SkillActionLogic.mlua
    ↓ build SkillCastContext for this cast
SkillActionWrapper.mlua
    ↓ foreach target:
    ↓   atk = skill base + coeff * attr
    ↓   ApplyBeforeDamageEffects (e.g. LOW_HP_DAMAGE_BONUS → damageMultiplier)
    ↓   DamageRequest { atk, damageMultiplier, … }
BattleCalculatorLogic.mlua
    ↓ numbers in → { damage, … } out
SkillActionResolver.mlua
    ↓ apply damage / After* EffectRequests — never atk
```

`BeforeDamage` skillEffect rows are **Wrapper-only** templates: read target
state (HP ratio, …) → pure scalar → Calculator. See
[SkillEffectSystem.md](../SkillEffect/SkillEffectSystem.md) (`spearPower`).

## Value lifetime

| Value | Actor | Wrapper | Calculator in | Calculator out / Resolver |
|-------|:-----:|:-------:|:-------------:|:-------------------------:|
| Five attributes | yes | read | no | no |
| Skill base + coeff | no (skill Config) | read → build `atk` | no | no |
| `atk` | **no** | **produce** | yes (`DamageRequest`) | **no** |
| `damage` | no | no | produce | **apply** |

Actors have **no 攻擊力**. Skills author formulas like `50 + 0.5 * will`; Wrapper
evaluates that into `atk` once per cast. After Calculator, only `damage` remains
on the apply path.

## Scripts

| Script | Path | Type | Role |
|--------|------|------|------|
| `SkillActionLogic` | `Logic/Skill/SkillActionLogic.mlua` | `@Logic` | Gateway; orchestrate pipeline |
| `SkillActionWrapper` | `Logic/Skill/SkillActionWrapper.mlua` | `@Logic` | `atk` + `ApplyBeforeDamageEffects` → `DamageRequest` |
| `BattleCalculatorLogic` | `Logic/Battle/BattleCalculatorLogic.mlua` | `@Logic` | `atk` (+ scalars) → `damage` |
| `SkillActionResolver` | `Logic/Skill/SkillActionResolver.mlua` | `@Logic` | Apply `damage` / effects / move / anim |

Access: `_SkillActionLogic`, `_SkillActionWrapper`, `_BattleCalculatorLogic`,
`_SkillActionResolver`.

## Independence Rule

| Layer | May depend on BattleSystem? | Notes |
|-------|:---------------------------:|-------|
| `SkillActionLogic` | permission gate only | Ask battle “may this actor act now?” |
| `SkillActionWrapper` | no | Only owner of cast-time `atk` |
| `BattleCalculatorLogic` | no | Numbers in → numbers out |
| `SkillActionResolver` | no | `damage` only for HP apply; no `atk` |

## Data Shapes (scaffold)

### SkillCastContext

```lua
{
    RequestId = 1,
    UserId = "...",
    SkillKey = "spearPower",
    ExecutionMode = "Field", -- or "Battle"
    ActionContext = {
        TargetIds = {},
        TargetPoint = nil,
        HasTargetPoint = false
    },
}
```

### Wrapper → DamageRequest

Skill Config / skillEffect supplies **base** and **coeff** (and which attribute).
Wrapper evaluates, for example:

```text
atk = 50 + 0.5 * will
```

```lua
{
    DamageRequests = {
        -- per target; Calculator input only
        -- {
        --   atk = 50 + 0.5 * will,
        --   damageMultiplier = 1 + beforeDamageBonuses,  -- e.g. LOW_HP_DAMAGE_BONUS
        --   finalDamageMultiplier = ...,
        --   masteryMultiplier = ...,
        --   totalDefense = ...,
        --   targetEntityId = "..."
        -- }
    },
    EffectRequests = {
        -- AfterDamage / AfterAction for Resolver (not BeforeDamage)
        -- { skillKey, timing, effectType, param1..4, targetSide, targetIds }
    }
}
```

Do **not** send attributes, HP, or `skillEffect` rows into Calculator — only
finished scalars. `spearPower` BeforeDamage example: `param1`=max bonus,
`param2`=HP ratio for full bonus, `targetSide` filters targets.

### Calculator → Resolve

```lua
-- Calculator out
{ damage = 0, isCritical = false, ... }

-- Resolver in (no atk)
_SkillActionResolver:Resolve(castContext, calculationResults, effectRequests)
-- uses calculationResults[i].damage only for ApplyDamage
```

## Orchestration Stub

```lua
local castContext = _SkillActionLogic:BuildCastContext(...)
local requestBundle = _SkillActionWrapper:BuildRequests(castContext)
local calculationResults = {}
for _, damageRequest in ipairs(requestBundle.DamageRequests) do
    table.insert(calculationResults, _BattleCalculatorLogic:CalculateDamage(damageRequest))
end
return _SkillActionResolver:Resolve(
    castContext, calculationResults, requestBundle.EffectRequests
)
```

## Related Docs

- [SkillActionSystem.md](SkillActionSystem.md)
- [Calculator/BattleCalculator.md](../Calculator/BattleCalculator.md)
- [Actor/ActorVariableExplain.md](../Actor/ActorVariableExplain.md)
- [SkillEffect/SkillEffectSystem.md](../SkillEffect/SkillEffectSystem.md)
