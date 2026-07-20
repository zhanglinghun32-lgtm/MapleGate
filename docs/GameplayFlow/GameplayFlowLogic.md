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
- Route a continuing user into the playable `World` scene at the loaded slot position.
- Route a user back to the `MainMenu` scene.
- Provide stable public entry points for future systems such as cutscenes,
  dialogue, and field encounters. Battle starts still go through
  `BattleFlowLogic:RequestStartBattle` (see Battle Entry Triggers), not a
  parallel starter on this Logic.
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
EnterWorldForUserAtPosition(userId, sourceKey, entryPosition)
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
  -> ContinueGame: GameplayFlowLogic:EnterWorldForUserAtPosition(..., savedPosition)
  -> NewGame: GameplayFlowLogic:EnterWorldForUser(...)
  -> SceneLogic moves the player to map01 with the selected entry position
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

- `RequestBattle(...)` may exist only as a thin router to
  `BattleFlowLogic:RequestStartBattle` — do not duplicate encounter validation
  or payload building here. See `docs/BattleFlow/BattleFlowLogic.md` § Battle
  Entry Triggers.
- `StartCutscene(cutsceneKey)` can coordinate player control locks and scene UI.
- `EnterDialogue(dialogueKey)` can coordinate UI and interaction locks.
  Dialogue-driven combat still ends in `BattleFlowLogic:RequestStartBattle`,
  not a second battle starter on this Logic.

Keep feature behavior in the feature owner. `GameplayFlowLogic` should remain
thin and boring.
