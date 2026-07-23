# BattleActorComponent

`BattleActorComponent` is the core state component for any actor that can
participate in battle.

## File Placement

Keep `BattleActorComponent.mlua` in `RootDesk/MyDesk/Battle/`. Bind it on the
battle actor entity in Maker (inline on `.map`, on a spawned model, or on a
battle UI entity — whichever holds that actor in your setup).

Single-use battle actors do not need a dedicated `.model` unless you spawn them
at runtime or reuse the same composition in multiple places. See
`docs/ScriptRules.md`.

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

- **No actor 攻擊力** (`atk` / `baseAttack` / `totalAttack`). Skills supply
  formulas such as `50 + 0.5 * will`; only `SkillActionWrapper` evaluates that
  into cast-time `DamageRequest.atk`. Resolver applies `damage` only.
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

`BattleActorCom` (documented historically as `BattleActorComponent`) is the
**only** place that stores real combat resources for a unit.

Every battle unit is a **live entity** with this component. There is no
config-only / payload-only HP path.

### Current vs max (required shape)

| Property | Meaning |
|----------|---------|
| `maxHp` | 一般狀態 — current maximum capacity |
| `hp` | 當前狀態 — current points, always `0 .. maxHp` |
| `maxMp` / `mp` | Same pattern |
| `maxStamina` / `stamina` | Same pattern |

UI must display these component fields (via `@Sync` or direct read on the client
copy of the entity component). Do **not** display a separate number copied from
`BattleStartPayload` or `BattleSystem.actorMap`.

Other systems may:

- Read resources from `BattleActorCom`.
- Request resource changes through `BattleActorCom` (`ApplyDamage`, `Heal`, …).
- Keep display-only caches **refreshed from the component** (e.g. `OnSyncProperty`).

Other systems must not:

- Store separate authoritative HP / MP / stamina on battle session tables.
- Treat payload `Hp` / `MaxHp` as live state.
- Decide death without `BattleActorCom:IsDead()` (or equivalent on the component).

## Resource vs Stat Shape

Resources and stats should be modeled differently.

Resources usually need a current value and a max value:

```text
hp / maxHp
mp / maxMp
stamina / maxStamina
```

Defense and similar stats may use a base / total pair:

```text
baseDefense / totalDefense
```

**攻擊力 is not a Com property.** Wrapper builds cast-time `atk` from the skill
formula; Calculator turns it into `damage`; Resolver never sees `atk`.

Do not force every property into one generic shape:

- HP / MP / stamina change during play and are clamped by max.
- `atk` exists only inside `DamageRequest` for one cast; never on the actor.

Recommended first implementation:

- Use explicit properties such as `hp`, `maxHp`, `mp`, `maxMp`,
  `stamina`, `maxStamina`, five attributes, and defense / speed totals.
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
EquipmentConfig["IronSword"] = { ... bonuses that Wrapper / RecalculateStats can read }
```

Equipment may still contribute to **cast-time** damage, but that contribution is
folded by `SkillActionWrapper` when building `DamageRequest.atk` (and related
scalars), not stored as actor `totalAttack`.

`totalDefense` and similar non-attack totals should be recalculated from sources
whenever actor stats are initialized or equipment changes — never manually
incremented once and treated as a new base.

Recommended first implementation:

- Store equipped keys explicitly on `BattleActorComponent`.
- Load equipment effects from Config during `RecalculateStats()` for defense /
  speed / resource modifiers as needed.
- Treat `total*` defense-like values as runtime caches, not Save data.
- Leave skill damage attack power to `SkillActionWrapper` → Calculator.
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
- `SkillActionWrapper` evaluates skill base + coeff × attribute into `atk`.
- `BattleCalculatorLogic` turns `atk` (+ scalars) into `damage`.
- `SkillActionResolver` applies `damage` only — no `atk` on the apply path.

Do not store permanent 攻擊力 on the actor.

Recommended attack flow:

```text
SkillActionLogic validates the request
  -> SkillActionWrapper: atk = 50 + 0.5 * will (example)
  -> BattleCalculatorLogic:CalculateDamage({ atk, … }) → { damage, … }
  -> SkillActionResolver: ApplyDamage(damage)
```

Recommended first EffectSystem interface:

```text
AddEffect(targetActorId, effectKey, value, duration)
RemoveEffect(effectId)
TickTurn(actorId)
GetStatRate(actorId, statKey)
GetStatFlat(actorId, statKey)
```

Recommended modifier targets:

- `DamageDealt` / cast-time atk inputs (consumed by Wrapper, not stored as atk)
- `DamageTaken`
- `Defense`
- `Speed`
- `MpCost`
- `StaminaCost`
- Five-attribute modifiers when effects change attrs temporarily

Even after the effect system is added, keep equipment and attribute sources
independent and deterministic. Effects should feed Wrapper convergence or
`RecalculateStats()` for defense-like totals — not invent a permanent actor
`atk` field.

## Component Interaction

Recommended direction:

- Shared skill / inventory systems request mutations; they do not own HP.
- `BattleActorCom` applies damage/heal and owns death flag from HP.
- `BattleSystem` settlement asks `IsDead()` and updates turn queue / victory.
- UI reads the same `BattleActorCom` the server mutates.

Example conceptual flow:

```text
SkillActionResolver / Inventory
  -> BattleActorCom:ApplyDamage()
  -> BattleActorCom updates hp (clamped to maxHp)
  -> @Sync / OnSyncProperty refreshes UI bars on that entity
  -> BattleSystem settlement: IsDead() -> queue / win-lose
```

### LEGACY notes for implementers

| File | Issue |
|------|-------|
| `Battle/BattleActorCom.mlua` | Has `hp`/`maxHp` `@Sync` but missing `ApplyDamage` / `IsDead` — add these |
| `Battle/BattleSystem.mlua` | Registers payload tables into `actorMap` including copied resources — **LEGACY** |
| `Battle/BattleQueue.mlua` | Death removal stub — **LEGACY** until it uses `BattleActorCom:IsDead()` |
| `Logic/Battle/BattleFlowLogic.mlua` `BuildFieldActorEntry` | Copies hp into payload — **LEGACY**; prefer EntityId roster only |

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

- `docs/Actor/BattleActorInitPaths.md` — **Player (`statsSource=Save`) vs Monster/NPC (`statsSource=Config`)** startup paths
- `docs/Actor/ActorVariableExplain.md` — field meanings for balance notes and formula review

Future actor documents can live beside this file:

- `QuestActor.md`
- `DialogueActor.md`
- `ShopActor.md`
- `PartyActor.md`
