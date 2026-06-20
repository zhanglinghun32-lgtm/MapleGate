# SystemMenuUI

## Purpose

`SystemMenuUI` is the persistent system overlay shown after a save slot enters the
`World` scene. Battle, dialogue, enhancement, and other gameplay UI may open or
close without replacing this overlay.

## UI Structure

| Entity | Responsibility |
|---|---|
| `settingToggle` | Gear button that expands or collapses the system actions. |
| `menuPanel` | Collapsible container below the gear button. |
| `storeData` | Requests an immediate save of the currently loaded slot. |
| `backToMenu` | Saves the loaded slot, then returns to `MainMenu`. |

The UI root is managed as an overlay by `UIManagerLogic`, so it is not stored in
the single `currentOpenUI` slot used by normal scene and gameplay screens.

## Runtime Flow

1. `SceneLogic.ApplySceneUI` enables the overlay only for the `World` scene.
2. `SystemMenuUIComponent` owns button event wiring and collapsed state.
3. `storeData` calls `PlayerDataLogic.RequestSaveCurrentSlot`.
4. `backToMenu` calls `PlayerDataLogic.RequestSaveAndReturnToMenu`.
5. SaveSlot serialization remains server-authoritative in `PlayerDataLogic`.

## Keys

- UI key: `SystemMenuUI`
- Gameplay scene key: `World`
- Menu scene key: `MainMenu`
