# GameplayFlowLogic

## Purpose

`GameplayFlowLogic` is the world-play flow coordinator. It sits above scene
routing and below feature-specific systems.

It should answer: "Is this user entering or leaving playable world flow?" It
should not answer battle turn order, UI button binding, save serialization, or
map-local gameplay details.

## Runtime Owner

Path:

```text
RootDesk/MyDesk/Logic/GameplayFlowLogic.mlua
```

`GameplayFlowLogic` is an `@Logic` because gameplay session flow survives map
transitions and is shared by multiple systems.

## Responsibilities

- Route a loaded user into the playable `World` scene.
- Route a user back to the `MainMenu` scene.
- Provide stable public entry points for future systems such as cutscenes,
  dialogue, field encounters, and battle entry.
- Keep `SceneLogic` as the low-level scene router.

## Non-Responsibilities

- Do not load or save player data directly.
- Do not bind UI buttons.
- Do not open or close specific UI roots directly.
- Do not own battle turn state or battle presentation.
- Do not manage map-local quest or encounter logic.
- Do not directly enable or disable `PlayerControllerComponent`.

## Current API

```text
EnterWorldForUser(userId, sourceKey)
ReturnToMainMenuForUser(userId, sourceKey)
```

`sourceKey` is a logging and future-policy key. Current callers include:

- `ContinueGame`
- `StartNewGame`
- `SaveAndReturnToMenu`

## Flow

```text
MainMenuUIComponent
  -> PlayerDataLogic:RequestContinueGame / RequestNewGame
  -> PlayerDataLogic loads or creates save data
  -> GameplayFlowLogic:EnterWorldForUser(...)
  -> SceneLogic:ChangeSceneForUser(userId, "World")
```

Returning to menu:

```text
SystemMenuUIComponent
  -> PlayerDataLogic:RequestSaveAndReturnToMenu
  -> PlayerDataLogic flushes save data
  -> GameplayFlowLogic:ReturnToMainMenuForUser(...)
  -> SceneLogic:ChangeSceneForUser(userId, "MainMenu")
```

## Future Extensions

- `RequestBattle(...)` can coordinate battle entry before delegating to
  `BattleFlowLogic`.
- `StartCutscene(cutsceneKey)` can coordinate player control locks and scene UI.
- `EnterDialogue(dialogueKey)` can coordinate UI and interaction locks.

Keep feature behavior in the feature owner. `GameplayFlowLogic` should remain
thin and boring.
