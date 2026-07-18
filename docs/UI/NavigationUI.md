# NavigationUI

## Scope

`NavigationUI` is a persistent, horizontal shortcut bar positioned beside the
existing top-left `SystemMenuUI`. It owns only navigation interactions; inventory,
party, and character screen contents remain separate systems.

## Layout

- A collapse/expand button remains visible at all times.
- The expanded panel contains three 88 x 88 buttons in this order:
  Inventory, Party, Character.
- The panel starts collapsed and uses the supplied sprite RUIDs for each feature.

## Public UI API

- `SetExpanded(expanded)`
- `OpenInventoryUI()` -> `_UIManagerLogic:OpenUI("Inventory")`
- `OpenPartyUI()` -> `_UIManagerLogic:OpenUI("Party")`
- `OpenCharacterUI()` -> `_UIManagerLogic:OpenUI("Character")`

The three methods are the boundary for future screen-specific integration.

