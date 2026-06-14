# BattleSystem Model

`BattleSystem.model` is the runtime holder for one active turn-based battle.

The model should have a `script.BattleSystemComponent` component. The component
owns the battle session flow, while actor state remains inside each actor's
`BattleActorComponent`.

## Actor Registry

The current design uses actor entities with `BattleActorComponent`.

Recommended registry:

```text
BattleSystemComponent
- activeActorIds
- turnQueue
- actorMap: actorId -> BattleActorComponent
- currentActorId
```

State changes should go through `BattleActorComponent`:

```text
targetActor:ApplyDamage(value)
sponsorActor:UseMp(cost)
sponsorActor:UseStamina(cost)
```

`BattleSystemComponent` should not directly assign HP, MP, stamina, or stat
values.

## Battle Keys

Battle system enum-like strings should be centralized in
`RootDesk/MyDesk/Flow/BattleKeys.mlua`.

Use `_BattleKeys` instead of writing magic strings such as `"Damage"`,
`"TotalAttack"`, or `"Enemy"` directly in battle scripts.

Examples:

```lua
entry.entryType = _BattleKeys.entryDamage
modifier.statKey = _BattleKeys.statTotalAttack
action.targetRule = _BattleKeys.targetEnemy
```

Content keys still belong in Config data, not `BattleKeys`.

Examples:

- `NormalAttack`
- `PoisonSlash`
- `IronSword`
- `AttackUp`
- `Poison`

## Responsibilities

- Receive `StartBattle(payload)` from `BattleFlowLogic`.
- Register all battle actors.
- Own active actor ids and turn queue.
- Decide whose turn it is.
- Receive action requests.
- Validate that the sponsor can act and the target is valid.
- Ask `BattleCalculator` to calculate action results.
- Apply action results through `BattleActorComponent` interfaces.
- Remove dead actors from the active turn queue.
- Check victory, defeat, escape, and cancel conditions.
- Report battle result back to `BattleFlowLogic`.

## Non-Responsibilities

- Do not load encounter Config by key.
- Do not read or write permanent save data directly.
- Do not decide scene transitions after battle ends.
- Do not bind battle UI buttons.
- Do not own field monster movement or chase AI.
- Do not directly mutate actor resources without `BattleActorComponent`.

## Action Flow

```text
RequestAction(sponsorId, actionKey, targetIds)
  -> validate battle is active
  -> validate sponsorId is current turn actor
  -> validate sponsor and targets exist
  -> ask BattleCalculator for result
  -> apply result entries through BattleActorComponent
  -> process deaths
  -> remove dead actors from turnQueue
  -> check battle end
  -> advance turn
```

## BattleCalculator Input

`BattleCalculator` should receive already-prepared input. It should not query UI,
scene flow, save data, or raw Config by itself.

Skill action config should come from `skillConfig` DataSet rows. A runtime
`BattleSkill` component can load one row by `skillKey`, then expose the prepared
`actionConfig` table through `BuildActionConfig()`.

Recommended input shape:

```text
BattleActionInput
- battleId
- actionKey
- sponsorId
- targetIds
- sponsorSnapshot
- targetSnapshots
- actionConfig
- effectModifiers
- randomSeed
```

Snapshot values should contain only what the calculator needs for this action:

```text
ActorBattleSnapshot
- actorId
- teamId
- level
- hp
- maxHp
- mp
- maxMp
- stamina
- maxStamina
- totalAttack
- totalDefense
- speed
- activeEffectKeys
```

This keeps `BattleCalculator` deterministic and easy to test.

## BattleCalculator Result

One action can produce multiple effects, so the result should be a list of
entries instead of a single damage number.

Recommended result shape:

```text
BattleActionResult
- battleId
- sponsorId
- actionKey
- entries
- logs
```

Each entry describes one effect to apply:

```text
BattleResultEntry
- entryType
- sourceActorId
- targetActorId
- value
- resourceKey
- effectKey
- duration
- isHit
- isCritical
- reason
```

Recommended first `entryType` values from `_BattleKeys`:

- `_BattleKeys.entryDamage`
- `_BattleKeys.entryHeal`
- `_BattleKeys.entryMpChange`
- `_BattleKeys.entryStaminaChange`
- `_BattleKeys.entryAddEffect`
- `_BattleKeys.entryRemoveEffect`
- `_BattleKeys.entryRevive`
- `_BattleKeys.entryGuard`
- `_BattleKeys.entryMiss`

Example action result:

```text
BattleActionResult
- sponsorId = "Player01"
- actionKey = "SlashWithPoison"
- entries:
  - Damage target=Monster01 value=30
  - AddEffect target=Monster01 effectKey=Poison duration=3
  - StaminaChange target=Player01 value=-10
```

`BattleSystemComponent` applies each entry. `BattleCalculator` only describes
what should happen.

## Applying Results

Recommended application rules:

- `Damage` calls `targetActor:ApplyDamage(value)`.
- `Heal` calls `targetActor:Heal(value)`.
- `MpChange` calls `RecoverMp()` or `UseMp()` depending on sign.
- `StaminaChange` calls `RecoverStamina()` or `UseStamina()` depending on sign.
- `AddEffect` delegates to `EffectSystem`.
- `RemoveEffect` delegates to `EffectSystem`.
- `Revive` delegates to a future actor revive interface.
- `Guard` and `Miss` produce presentation/UI feedback but do not mutate HP.

After all entries are applied, `BattleSystemComponent` checks each affected
actor for death and removes dead actors from `turnQueue`.
