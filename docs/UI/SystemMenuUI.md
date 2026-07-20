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

## Battle Policy

Battle does **not** own SystemMenu open/close. The overlay may remain during
battle.

Individual actions (save, return to menu, …) follow product rules and must be
enforced in SystemMenu / `PlayerDataLogic`, not inside `BattleUI` or by making
`BattleSystem` open the menu.

### Agent TODO

1. Decide which SystemMenu actions stay enabled in battle.
2. If any action must be blocked, gate it with battle activity /
   `IsOperationAllowed`, not by hiding the entire SystemMenu from BattleUI code.
3. Keep SystemMenu as a persistent overlay relative to `BattleUI` layering
   (`docs/BattleFlow/BattleUIComponent.md`).
