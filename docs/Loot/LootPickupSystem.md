# Loot Pickup System

## Scope

MapleTile field drops are server-authoritative entities. A drop falls onto a
foothold, waits for a short pickup delay, and is collected when the currently
controlled field entity enters its pickup radius.

## Data

- Inventory key: sapphire
- Display name: 藍寶石
- Category: Material / Loot
- Effect: none
- Sprite/icon RUID: 9b1796a9a66d4248bcee8af6464dfb0e
- Maximum stack: 999

## Runtime Flow

T on controlled player -> LootTestInputComponent -> LootDropLogic
-> SpawnByModelId(SapphireDrop, parent=current map)
-> LootDropComponent checks controlled-player proximity on Server
-> InventoryLogic:AddItem(userId, sapphire, 1)
-> PlayerDataLogic:MarkDirty(userId) -> destroy ground drop.

The T binding is a development-only spawn entrance. Production monster/drop
tables should call LootDropLogic:SpawnDrop directly.

