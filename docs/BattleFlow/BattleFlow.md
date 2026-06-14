# Battle Flow Design

This document records the current direction for the turn-based battle flow.
No runtime code is implemented yet.

## Goal

Design `BattleFlowLogic.mlua` as the first battle-flow entry point.

Use `BattleSystem.model` as the runtime holder for one battle session.

The battle flow is turn-based, so monster chase / field AI is outside this system.

## Planned Files

### `RootDesk/MyDesk/Flow/BattleFlowLogic.mlua`

World-level battle entry point.

Expected responsibilities:

- Receive battle start requests from map components, NPCs, encounters, or scene flow.
- Validate whether the player can enter battle.
- Expose the server-side battle creation entry point.
- Prepare player-side battle data.
- Prepare enemy-side battle data.
- Create a `BattleSystem.model` instance for the battle session.
- Call `StartBattle(payload)` on the spawned battle system component.
- Own high-level battle lifecycle callbacks such as battle start, battle end, victory, defeat, and escape.

Non-responsibilities:

- Do not calculate every attack result directly.
- Do not bind battle UI buttons.
- Do not own monster field movement or chase AI.
- Do not store permanent player save data directly.
- Do not run the turn loop after `BattleSystem` has started.

Planned entry point:

```lua
method Entity CreateBattle(Entity sponsorEntity, Entity targetEntity, string encounterKey)
```

Parameter meanings:

- `sponsorEntity`: the entity that starts or sponsors the battle. This may be a field monster, NPC, map event entity, trigger zone, or map component owner.
- `targetEntity`: the player or party leader entity entering the battle.
- `encounterKey`: a stable key used to look up `EncounterConfig` data. It is not the config data itself.

`BattleFlowLogic` should use the sponsor and target as runtime context, then use `encounterKey` to resolve static battle setup.

Recommended flow:

1. Validate `sponsorEntity` and `targetEntity`.
2. Read `currentMap` from the sponsor or target context.
3. Load `EncounterConfig` by `encounterKey`.
4. Build player battle data from the target player / party.
5. Build enemy battle data from `EncounterConfig`, optionally merged with sponsor runtime data.
6. Build the battle start payload.
7. Spawn `BattleSystem.model`.
8. Call `BattleSystemComponent:StartBattle(payload)`.

When spawning the model at runtime, `BattleFlowLogic` must pass a non-nil map
entity as the spawn parent. The first version can use the current map from the
sponsor or target context.

## BattleSystem Model

`BattleSystem.model` is the runtime holder for a single battle session.

Recommended model component:

```text
script.BattleSystemComponent
```

Expected responsibilities:

- Receive a complete battle start payload.
- Own the active battle session state.
- Own the turn loop after `StartBattle(payload)`.
- Own actor ids and `BattleActorComponent` references for the battle.
- Own active battle effects or delegate them to the future `EffectSystem`.
- Resolve player and enemy actions through battle calculators.
- Report battle end result back to `BattleFlowLogic`.

Non-responsibilities:

- Do not load encounter Config by key.
- Do not read permanent player save data directly.
- Do not decide scene transition after battle ends.
- Do not own field monster AI.

Planned entry point:

```lua
method void StartBattle(table payload)
```

`StartBattle(payload)` should assume that `BattleFlowLogic` already validated
the sponsor, target, encounter key, and required battle data.

## Battle Start Payload

The first version can use a simple table-like payload.

Suggested shape:

```lua
{
    battleId = "",
    sceneKey = "",
    mapName = "",
    sponsorEntityId = "",
    playerUserId = "",
    playerParty = {},
    enemyParty = {},
    encounterKey = "",
    rewardKey = ""
}
```

## Encounter Key And Config

`encounterKey` is a string identifier, not the config object itself.

Example values:

- `"Field_Slime_01"`
- `"Training_Dummy_01"`
- `"Boss_Mushmom_01"`
- `"Quest_IntroBattle_01"`

The key should point to an `EncounterConfig` row or table entry under Config data.

Conceptually:

```text
encounterKey -> EncounterConfig -> enemy party, rewards, battle rules, UI/theme data
```

Suggested `EncounterConfig` fields:

```lua
{
    encounterKey = "Training_Dummy_01",
    enemyGroupKey = "TrainingDummyGroup",
    rewardKey = "TrainingDummyReward",
    battleRuleKey = "DefaultTurnBattle",
    battleSceneKey = "",
    battleBgmKey = "",
    canEscape = true,
    maxTurnCount = 0
}
```

Why use `encounterKey` instead of passing the whole config:

- The caller stays simple. A monster, NPC, or map trigger only needs to know which encounter it represents.
- Battle balancing stays in Config data instead of scattered scripts.
- Multiple field entities can reuse the same encounter setup.
- `BattleFlowLogic` can validate missing or invalid encounters in one place.
- Later, save data only needs to remember keys, not duplicate static config data.

`sponsorEntity` can still contribute runtime context. For example, a field monster can provide its current level, position, or special state, while `encounterKey` decides the base enemy group and reward table.

## Actor Data

Each battle actor should be registered as an entity with `BattleActorComponent`.
The first version should keep `BattleActorComponent` as the actor state authority.

Actor type examples:

- `"Player"`
- `"Companion"`
- `"Monster"`
- `"Boss"`

For concrete actor properties and ownership rules, see
`docs/Actor/BattleActorComponent.md`.

`BattleSystemComponent` should store actor ids and component references, then
apply state changes through `BattleActorComponent` interfaces.

## Battle Lifecycle

Expected first-pass flow:

1. Field system calls `BattleFlowLogic:CreateBattle(sponsorEntity, targetEntity, encounterKey)`.
2. `BattleFlowLogic` validates the sponsor, target, and encounter key.
3. `BattleFlowLogic` loads `EncounterConfig` by `encounterKey`.
4. `BattleFlowLogic` builds the battle start payload.
5. `BattleFlowLogic` spawns `BattleSystem.model`.
6. `BattleFlowLogic` calls `BattleSystemComponent:StartBattle(payload)`.
7. `BattleSystemComponent` initializes turn order.
8. Player selects an action.
9. `BattleSystemComponent` resolves the action.
10. Enemy turn resolves through turn-based rules.
11. Battle ends with victory, defeat, escape, or cancel.
12. `BattleSystemComponent` reports the result back to `BattleFlowLogic`.
13. `BattleFlowLogic` applies rewards, save changes, and scene flow decisions.

## Open Questions

- Should battle happen in the same map, or transition to a battle scene?
- Where should `BattleSystem.model` be spawned when battle happens in a separate battle scene?
- Should battle UI be opened by `BattleFlowLogic`, by `BattleSystemComponent`, or by a battle UI controller through `UIManagerLogic`?
- Should enemy data come from a static Config DataSet, a monster entity on the field, or both?
- Should player party data be copied from save data or from live runtime components?
- Should action resolution use MSW `AttackComponent` / `HitComponent`, or a pure turn-based calculation layer first?

## Current Direction

Use `BattleFlowLogic` as the battle entry point and session factory.

Use `BattleSystem.model` as the first battle runtime holder.

Design rule:

```text
BattleFlowLogic:CreateBattle(...)
  -> validate and prepare data
  -> spawn BattleSystem.model
  -> BattleSystemComponent:StartBattle(payload)
  -> BattleSystemComponent owns the active turn battle
```

Keep persistent player save data separate from battle runtime state.

Keep battle UI separate from battle logic until the battle flow shape is stable.
