# MSW Script Rules

This project uses these naming and responsibility rules for all `.mlua` scripts.

## Naming

- Script file names use PascalCase: `SceneLogic.mlua`, `UIManagerLogic.mlua`, `MainMenuUIComponent.mlua`.
- Script declarations match the file name exactly: `script SceneLogic extends Logic`.
- Public scene/UI/data keys use PascalCase strings: `"MainMenu"`, `"World"`, `"CharacterStatus"`, `"Party"`.
- Properties, local variables, and parameters use lowerCamelCase: `mainMenuSceneKey`, `userId`, `entryPosition`.
- Methods use PascalCase: `ChangeScene`, `OpenUI`, `GetEntryPosition`.
- Engine lifecycle methods keep the engine spelling: `OnBeginPlay`, `OnEndPlay`, `OnUpdate`.
- Event handler methods use PascalCase and a clear event suffix: `OnNewGameClicked`, `OnSettingClicked`.

## Responsibility Boundaries

- `@Logic` is for world/session-wide managers that survive map transitions.
- Map-specific behavior belongs in a map root `@Component`.
- UI interaction behavior belongs in a UI root `@Component`.
- UI opening, closing, duplicate prevention, and key lookup belong in `UIManagerLogic`.
- UI keys and entity lookup data belong in `UIRegistry` DataSet, not scattered UUID literals in scripts.
- Scene selection and top-level map/UI pairing belong in `SceneLogic`.
- Player/map entry positions should be provided by the target map component when possible.

## SceneLogic

`SceneLogic` is the top-level scene router.

Responsibilities:

- Expose `ChangeScene(sceneKey)` as the public scene-change interface.
- Resolve a scene key such as `"MainMenu"` or `"World"` to an entry map.
- Move the user to the scene entry map.
- Ask the target map component for an entry position when that component exists.
- Tell `UIManagerLogic` which scene UI should open.
- Run the previous scene close callback before applying the next scene UI.

Non-responsibilities:

- Do not bind UI buttons.
- Do not store UI entity UUIDs.
- Do not directly manage UI entity enable state.
- Do not implement map-local gameplay logic.
- Do not decide detailed player spawn rules beyond asking the map component.

Current scene keys:

- `"MainMenu"`: map `MainMenu`, UI key `MainMenu`.
- `"World"`: map `map01`, no scene UI yet.
