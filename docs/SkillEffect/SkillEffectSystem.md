# Skill Effect System

## Goal

Describe timed skill side-effects in Config so one skill can own zero or more
effect rows without packing geometry / buff / bonus fields into `skillConfig`.

**BeforeDamage** rows are evaluated in `SkillActionWrapper` and become **pure
numbers** on `DamageRequest` before `BattleCalculatorLogic`. **AfterDamage** /
**AfterAction** rows typically become `EffectRequest`s for
`SkillActionResolver`.

## Shared Key

`skillKey` is the join key shared with:

- `skillConfig.skillKey`
- `skillRange.keyName` (same string value)
- `skillPresentationConfig.skillKey`
- `skillPresentationStepConfig.skillKey`

One skill may have **multiple** `skillEffect` rows. Skills with no side-effects
have no rows.

## Config Path

| File | `GetTable` name |
|------|-----------------|
| `RootDesk/MyDesk/Data/Config/skill/skillEffect.csv` | `"skillEffect"` |
| `RootDesk/MyDesk/Data/Config/skill/skillEffect.userdataset` | (metadata; `name = skillEffect`) |

## Columns

| Column | Required | Meaning |
|--------|:--------:|---------|
| `skillKey` | yes | Same key as `skillConfig` |
| `timing` | yes | When the effect is considered |
| `effectType` | yes | Which handler interprets params |
| `param1`…`param4` | no | Effect-specific arguments (all strings in CSV) |
| `targetSide` | yes | Which targets this row may affect |

## `timing` and pipeline ownership

| Value | Owner | Role |
|-------|-------|------|
| `BeforeAction` | Wrapper (planned) | Pre-cast numeric / setup |
| `BeforeDamage` | **`SkillActionWrapper`** | Per-target: read live state → pure scalars on `DamageRequest` |
| `AfterDamage` | Resolver (via `EffectRequest`) | Apply buffs / follow-ups after damage |
| `AfterAction` | Resolver (via `EffectRequest`) | Post-cast cleanup / follow-ups |

### BeforeDamage template (Wrapper)

```text
foreach target:
  BuildBaseDamageRequest (atk = skillBase + coeff * attr, …)
  ApplyBeforeDamageEffects
    → load skillEffect rows (skillKey + timing=BeforeDamage)
    → if targetSide matches this target
    → convert target state to numeric bonus
    → fold into DamageRequest (e.g. damageMultiplier)
  send DamageRequest → BattleCalculatorLogic → { damage, … }
→ SkillActionResolver applies damage / After* EffectRequests
```

Wrapper never sends entities or `skillEffect` rows into Calculator — only
numbers.

## `effectType`

| Value | Timing (typical) | Param meaning |
|-------|------------------|---------------|
| `LOW_HP_DAMAGE_BONUS` | `BeforeDamage` | Target HP lower → higher damage. See below. |
| `ADD_BUFF` | `AfterDamage` / `AfterAction` | `param1` = buff key; `param2` = duration; `param3` = stack/intensity (optional) |

### `LOW_HP_DAMAGE_BONUS` (spearPower example)

Intent: **目標生命越低，造成傷害越高.**

| Param | Meaning |
|-------|---------|
| `param1` | **最高增傷幅度** (max bonus amplitude), e.g. `1.3` |
| `param2` | **吃滿最高增傷時的剩餘生命比例**, e.g. `0.7` (HP ≤ 70% → full bonus) |
| `param3` | reserved |
| `param4` | reserved |
| `targetSide` | Only matching targets get this bonus (e.g. `All` / `Enemy`) |

Planned scale (implement in `ComputeLowHpDamageBonus`):

```text
hpRatio = target.hp / target.maxHp
t = clamp( (1 - hpRatio) / (1 - param2), 0, 1 )   -- 0 at full HP, 1 at ratio ≤ param2
bonus = param1 * t
-- fold bonus into DamageRequest.damageMultiplier (exact fold rule TBD)
```

Example row:

```csv
spearPower,BeforeDamage,LOW_HP_DAMAGE_BONUS,1.3,0.7,,,All
```

## `targetSide`

| Value | Meaning |
|-------|---------|
| `Enemy` | Opposing side vs sponsor |
| `Ally` | Same side as sponsor |
| `Self` | Sponsor only |
| `Target` | Resolved hit-list targets |
| `All` | Every resolved damage target in this cast |
| `Sponsor` | Alias of sponsor (same as `Self` unless multi-sponsor) |

`DoesEffectApplyToTarget` in Wrapper enforces this before applying a
BeforeDamage numeric bonus.

## Authoring Rules

1. Join only by `skillKey`.
2. One logical trigger = one row.
3. BeforeDamage → numeric in Wrapper; After* → EffectRequest for Resolver.
4. Presentation VFX stays in `skillPresentation*`.

## Example Rows (current)

```csv
skillKey,timing,effectType,param1,param2,param3,param4,targetSide
spearPower,BeforeDamage,LOW_HP_DAMAGE_BONUS,1.3,0.7,,,All
venomEdge,AfterDamage,ADD_BUFF,Poison,5,,,Enemy
guardStrike,AfterAction,ADD_BUFF,Guard,1,,,Ally
```

## Related Docs

- [SkillAction/SkillCastPipeline.md](../SkillAction/SkillCastPipeline.md)
- [SkillAction/SkillActionSystem.md](../SkillAction/SkillActionSystem.md)
- [Calculator/BattleCalculator.md](../Calculator/BattleCalculator.md)
- [ScriptRules.md](../ScriptRules.md)
