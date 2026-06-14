# BattleActorComponent

`BattleActorComponent` is the core state component for any actor that can
participate in battle.

This document records responsibility boundaries first. The concrete `.mlua`
interface can be finalized later when `BattleActorComponent.mlua` is created.

## Purpose

- Represent one battle-capable actor.
- Own battle state that must have a single source of truth.
- Provide the future interface used by battle flow controllers, skills, UI, and effects.

## Responsibilities

- Own battle resources such as `hp`, `mp`, and `stamina`.
- Own core battle stats such as base attack and total attack.
- Provide battle-state queries such as `IsDead()`.
- Provide controlled mutation points such as `ApplyDamage()`, `Heal()`,
  `UseMp()`, and `UseStamina()`.
- Emit or trigger resource/stat change notifications for other components.
- Keep death-state decisions close to HP changes.

## Non-Responsibilities

- Do not bind UI widgets.
- Do not decide player input or turn order.
- Do not own skill definitions.
- Do not own equipment definitions.
- Do not own buff definitions.
- Do not directly manage quest progress.

## Core Properties

`BattleActorComponent` should own battle properties that define the actor's
current combat state.

Resource values:

- `hp` / `maxHp`: survival resource. Reaching 0 usually means dead or defeated.
- `mp` / `maxMp`: skill resource. Usually consumed by magic or special skills.
- `stamina` / `maxStamina`: action resource. Used for actions such as attack,
  dodge, guard, dash, or other turn/action costs.

Stat values:

- `baseAttack`: actor's attack before equipment, buffs, debuffs, and temporary effects.
- `totalAttack`: final attack value after all modifiers are applied.
- `baseDefense`: actor's defense before modifiers.
- `totalDefense`: final defense value after modifiers.
- `speed`: turn order or action timing value.

Future stat candidates:

- `criticalRate`
- `criticalDamageRate`
- `hitRate`
- `evasionRate`
- `elementPower`
- `elementResistance`
- `statusResistance`

Equipment keys:

- `weaponKey`: equipped weapon config key.
- `armorKey`: equipped armor config key.
- `accessoryKey`: equipped accessory config key.
- Additional slot keys can be added later when equipment slots are finalized.

Skill keys:

- `skillKeys`: keys for skills available to this battle actor.
- Skill definitions and effects should still live in Config data.

## Resource Authority Rule

`BattleActorComponent` is the only component that should store real battle
resources such as HP, MP, and stamina.

Other components may:

- Read resources from `BattleActorComponent`.
- Request resource changes through `BattleActorComponent`.
- Keep display-only cached values that are refreshed from change events.

Other components should not:

- Store separate authoritative HP, MP, or stamina values.
- Subtract or restore resources directly.
- Spend MP or stamina by assigning values directly.
- Decide death without checking `BattleActorComponent`.

## Resource vs Stat Shape

Resources and stats should be modeled differently.

Resources usually need a current value and a max value:

```text
hp / maxHp
mp / maxMp
stamina / maxStamina
```

Stats usually need a base value and a calculated total value:

```text
baseAttack / totalAttack
baseDefense / totalDefense
```

Do not force every property into one generic shape just because some values have
pairs. `hp` and `attack` look similar as numbers, but they behave differently:

- HP changes during battle and is clamped by `maxHp`.
- MP changes during battle and is clamped by `maxMp`.
- Stamina changes during actions and is clamped by `maxStamina`.
- Base attack rarely changes during one action.
- Total attack is recalculated from base stats, equipment, buffs, debuffs, and battle rules.

Recommended first implementation:

- Use explicit properties such as `hp`, `maxHp`, `mp`, `maxMp`,
  `stamina`, `maxStamina`, `baseAttack`, and `totalAttack`.
- Add helper methods to avoid duplicate logic.
- Consider a shared data structure later only if resource handling becomes repetitive.

Potential helper methods:

- `ClampResources()`
- `GetHpRate()`
- `GetMpRate()`
- `GetStaminaRate()`
- `CanUseMp(cost)`
- `CanUseStamina(cost)`
- `RecalculateStats()`

## Equipment And Stat Calculation

`BattleActorComponent` may record equipped item keys, but it should not own full
equipment definitions.

For the current design, equipment has no customization. This means an equipment
key points directly to static Config data, and that Config row provides the final
equipment effect.

Example:

```text
weaponKey = "IronSword"
EquipmentConfig["IronSword"].attackBonus = 10
```

`totalAttack` should not be manually incremented once and then treated as the new
base value. Instead, it should be recalculated from sources whenever actor stats
are initialized or equipment changes.

Recommended first-version formula:

```text
totalAttack = baseAttack + equipmentAttackBonus
```

For a sword with `+10 attack`:

```text
baseAttack = 20
weaponKey = "IronSword"
equipmentAttackBonus = 10
totalAttack = 30
```

Important rule:

- `baseAttack` remains the actor's own attack.
- `weaponKey` remembers which equipment is equipped.
- Config data says what the weapon provides.
- `totalAttack` is a derived cache produced by `RecalculateStats()`.

This prevents double-counting. For example, if the actor equips a `+10` sword,
then later equips a `+15` sword, the implementation should recalculate from
`baseAttack`, not subtract and add directly against the previous `totalAttack`.

Recommended first implementation:

- Store equipped keys explicitly on `BattleActorComponent`.
- Load equipment effects from Config during `RecalculateStats()`.
- Recalculate all `total*` stats from base values every time equipment changes.
- Treat `totalAttack` and other `total*` values as runtime cached results, not
  persistent save data.
- Do not introduce `EquipmentActor` while equipment has no customization or
  runtime instance state.

## Effect System And Modifiers

Buffs and debuffs are runtime effects, not equipment.

Equipment is currently static: an equipment key points to Config and Config
provides the final equipment value. Effects are different because two effects
with the same effect key may carry different runtime values, such as attack up
10% or attack up 11%.

`EffectSystem` should own active effect instances for the current battle.

Effect Config is static:

```text
EffectConfig["AttackUp"] = {
  targetStat = "TotalAttack",
  operator = "RateAdd",
  defaultValue = 0.10
}
```

Effect instance is runtime state:

```text
effectId = "battle01_actorA_effect003"
effectKey = "AttackUp"
sourceActorId = "Player01"
targetActorId = "Player01"
value = 0.11
remainingTurn = 3
stackCount = 1
```

Recommended responsibilities:

- `EffectSystem` owns active effect instances.
- `EffectSystem` manages duration, stack count, source actor, target actor, and
  concrete modifier value.
- `EffectSystem` exposes modifier query methods.
- `BattleActorComponent` asks `EffectSystem` for modifiers during
  `RecalculateStats()`.
- `BattleActorComponent` still owns the final `total*` stat cache.
- `BattleCalculator` receives prepared values and does not care how they were
  calculated.

Do not let effects directly mutate `totalAttack`, equipment keys, or Config
data. Effects should provide modifiers; `BattleActorComponent` should combine
those modifiers into final values.

Recommended attack flow:

```text
BattleActorComponent:Attack(target)
  -> RecalculateStats()
  -> attackValue = totalAttack
  -> BattleCalculator:ResolveAttack(sponsor, target, attackValue)
```

`BattleCalculator` should only receive already-prepared inputs such as sponsor,
target, and value. It should not know whether the value came from base stats,
equipment, buffs, debuffs, skills, or battle rules.

Recommended first EffectSystem interface:

```text
AddEffect(targetActorId, effectKey, value, duration)
RemoveEffect(effectId)
TickTurn(actorId)
GetStatRate(actorId, statKey)
GetStatFlat(actorId, statKey)
```

Recommended modifier targets:

- `BaseAttack`
- `EquipmentAttackBonus`
- `TotalAttack`
- `DamageDealt`
- `DamageTaken`
- `Defense`
- `Speed`
- `MpCost`
- `StaminaCost`

Example future formula:

```text
totalAttack = baseAttack + equipmentAttackBonus * equipmentAttackBonusRate
```

For a debuff that reduces equipment attack bonus by 50%:

```text
baseAttack = 20
equipmentAttackBonus = 10
equipmentAttackBonusRate = 0.5
totalAttack = 25
```

Even after the effect system is added, keep the raw equipment calculation
independent and deterministic. Effects should wrap or modify that result during
`RecalculateStats()`, not rewrite equipment data.

## Component Interaction

Recommended direction:

- `BattleController` calls battle actions at the correct turn timing.
- Skill components calculate or request effects, then call `BattleActorComponent`.
- Equipment components or Config readers provide stat bonuses, not resource storage.
- Buff components provide temporary modifiers, not resource storage.
- UI components listen for changes and display HP, MP, stamina, and stats.

Example conceptual flow:

```text
BattleController
  -> SkillComponent
  -> BattleActorComponent.ApplyDamage()
  -> BattleActorComponent updates HP
  -> resource bars / animation / battle flow react to the change
```

## Future Interface Notes

Potential methods:

- `ApplyDamage(damage)`
- `Heal(amount)`
- `UseMp(cost)`
- `RecoverMp(amount)`
- `UseStamina(cost)`
- `RecoverStamina(amount)`
- `IsDead()`
- `GetHpRate()`
- `GetMpRate()`
- `GetStaminaRate()`
- `RecalculateStats()`
- `EquipItem(slotKey, itemKey)`
- `UnequipItem(slotKey)`
- `GetEquippedItemKey(slotKey)`

Potential events:

- `BattleActorResourceChangedEvent`
- `BattleActorStatChangedEvent`
- `BattleActorDeadEvent`
- `BattleActorRevivedEvent`

These names are placeholders until the implementation starts.

## Related Actor Documents

Future actor documents can live beside this file:

- `QuestActor.md`
- `DialogueActor.md`
- `ShopActor.md`
- `PartyActor.md`
