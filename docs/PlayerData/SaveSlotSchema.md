# SaveSlot Schema

This document is the canonical field registry for player SaveSlot data.
Update it whenever a saved field is added, renamed, moved, or removed.

## Storage Address

| Item | Value |
|---|---|
| Storage | `UserDataStorage` |
| Owner | The current MSW `userId` |
| Key format | `Slot{slotIndex}` |
| Current keys | `Slot1`, `Slot2`, `Slot3` |
| Serialization | `_HttpService:JSONEncode(slotData)` / `_HttpService:JSONDecode(rawData)` |
| Authority | Server only: `PlayerDataLogic` |

Field names are case-sensitive. `Actors`, `actors`, and `Actor` are different
keys. Root / subsystem table keys use PascalCase (`Profile`, `Actors`,
`Inventory`, …). **Actor combat fields inside `Actors[]` use lowerCamelCase**
to match `actorConfig` and `BattleActorCom` (see `docs/Actor/ActorVariableExplain.md`).

Legacy slots written with `_UtilLogic:TableToString` are still accepted on load,
then rewritten as JSON on the next save. JSON is required because
`TableToString` does not round-trip nested tables correctly.
Before JSON encoding, empty Lua tables receive an internal `__MSWEmptyTable`
sentinel in a serialization-only copy because MSW cannot infer whether an empty
table is a JSON array or object. The sentinel is removed immediately after decode.

## Root Fields

| Exact key | Type | Required | Owner | Notes |
|---|---|---:|---|---|
| `Version` | integer | Yes | `PlayerDataLogic` | Current schema version is `3`. v2 added `PlayerPosition` + JSON; v3 Actors keys are slim lowerCamelCase (`configId`, five attrs, current `hp`/`mp`/`stamina`). |
| `Profile` | table | Yes | `PlayerDataLogic` | Slot metadata. |
| `PlayerPosition` | table | Yes | `PlayerDataLogic` | Last world position captured when the slot is saved. |
| `Actors` | array<table> | Yes | `PlayerDataLogic` / `BattleActorCom` | `Actors[1]` is currently applied to DefaultPlayer. |
| `Party` | table | Yes | `PartyLogic` | Party membership and formation. |
| `Inventory` | table | Yes | `InventoryLogic` | Item stacks and equipment keys. |
| `Skill` | table | Yes | `SkillLogic` | Learned skills and hotkeys. |
| `Mission` | table | Yes | `MissionLogic` | Active and completed missions. |

## Profile

Path: `slotData.Profile`

| Exact key | Type | Default | Notes |
|---|---|---|---|
| `DisplayName` | string | `"Player"` | Slot display name. |
| `PlayTimeSeconds` | integer | `0` | Accumulated play time; update logic is not implemented yet. |
| `LastSceneKey` | string | `"World"` | Public scene key, not a map name. |

## PlayerPosition

Path: `slotData.PlayerPosition`

| Exact key | Type | Default | Source / meaning |
|---|---|---:|---|
| `X` | number | `0` | `TransformComponent.WorldPosition.x` at save time. |
| `Y` | number | `0` | `TransformComponent.WorldPosition.y` at save time. |
| `Z` | number | `0` | `TransformComponent.WorldPosition.z` at save time. |

`ContinueGame` converts `X` and `Y` to a `Vector2` and passes it through
`GameplayFlowLogic` to `SceneLogic`, which uses it as the target of
`PlayerComponent:MoveToMapPosition`. `Z` is retained for schema completeness but
the current 2D map-transition API consumes only `X` and `Y`.

## Actors

Path: `slotData.Actors[index]`

**Slim persist only** (lowerCamelCase). `actorConfig` may hold more design
columns; Save does not copy them. See `docs/Actor/ActorVariableExplain.md`.

| Exact key | Type | Default for `Actors[1]` | Source / meaning |
|---|---|---|---|
| `configId` | string | `"playerWarrior"` | Archetype link to `actorConfig.configId`. |
| `jobType` | string | `"Warrior"` | Job. |
| `level` | integer | `1` | Level. |
| `constitution` | integer | `10` | 體質 — allocatable. |
| `dexterity` | integer | `6` | 靈巧 — allocatable. |
| `intelligence` | integer | `4` | 智力 — allocatable. |
| `will` | integer | `6` | 意志 — allocatable. |
| `perception` | integer | `5` | 感知 — allocatable. |
| `hp` | integer | (full after formula) | **Current** HP only. |
| `mp` | integer | (full after formula) | **Current** MP only. |
| `stamina` | integer | (full after formula) | **Current** stamina only. |

**Do not persist** (recomputed in `PlayerDataLogic:BuildRuntimeActorState`):

- Derived: `maxHp` / `maxMp` / `maxStamina` / `defense` / `speed` / `jumpForce` /
  `castRange` / `mpCostRate` / `recoveryRate` / `resistance` / `effectPotency` /
  `effectHitRate` / `criticalRate`
- Attack: `attack` / `totalAttack` (equip / skill / buff at runtime)
- `totalDefense`

Load: slim `Actors[]` → `BuildRuntimeActorState` →
`BattleActorCom:ImportSaveData(runtime)`.  
Export: `ExportSaveData()` writes slim keys only (legacy fat keys drop on next save).

## Party

Path: `slotData.Party`

| Exact key | Type | Notes |
|---|---|---|
| `Members` | array<table> | Legacy party actor records. Prefer `Formation` + `Actors[]`. |
| `Formation` | array<string> | Actor keys in formation order. Max length 4. |
| `ActiveFieldSlot` | integer | Formation slot that currently controls overworld movement. Default `1`. |

Path: `slotData.Party.Members[index]`

| Exact key | Type | Default member |
|---|---|---|
| `ActorKey` | string | `"playerWarrior"` |
| `Level` | integer | `1` |
| `Exp` | integer | `0` |
| `Hp` | integer | From `actorConfig.maxHp` |
| `Mp` | integer | From `actorConfig.maxMp` |

### Deprecation Direction

`slotData.Party.Members` currently duplicates actor stats from `Actors`. New work
must **not** add more duplicate writes.

Recommended migration:

```text
Party.Formation = ordered actor keys
Party.ActiveFieldSlot = current field controller slot
Actors[] = persistent actor state
```

If `Party.Members` remains temporarily for compatibility, treat it as read-only
legacy data until a migration removes it.

See `docs/Party/PartySystem.md` for the full party responsibility split.

## Inventory

Path: `slotData.Inventory`

| Exact key | Type | Notes |
|---|---|---|
| `Items` | array<table> | Item stacks by slot index. |
| `Equipped` | table | Equipment slot key to item key; currently empty by default. |

Path: `slotData.Inventory.Items[index]`

| Exact key | Type | Notes |
|---|---|---|
| `ItemKey` | string | Matches an inventory/config item key. |
| `Count` | integer | Stack count. |

Default stacks:

| Index | `ItemKey` | `Count` |
|---:|---|---:|
| 1 | `hpPotionSmall` | 5 |
| 2 | `mpPotionSmall` | 3 |
| 3 | `sealedLetter` | 1 |

## Skill

Path: `slotData.Skill`

| Exact key | Type | Notes |
|---|---|---|
| `Learned` | array<table> | Learned skill records. |
| `Hotkeys` | table<string, string> | Hotkey key to skill key. |

Path: `slotData.Skill.Learned[index]`

| Exact key | Type | Default learned skill |
|---|---|---|
| `SkillKey` | string | `"normalAttack"` |
| `Level` | integer | `1` |

Default hotkey: `slotData.Skill.Hotkeys.Slot1 = "normalAttack"`.

## Mission

Path: `slotData.Mission`

| Exact key | Type | Notes |
|---|---|---|
| `Active` | array<table> | Active mission states. |
| `Completed` | table<string, boolean> | Mission key to completion state. |

Path: `slotData.Mission.Active[index]`

| Exact key | Type | Notes |
|---|---|---|
| `MissionKey` | string | Matches `missionConfig.missionKey`. |
| `Progress` | table<string, integer> | Objective key to current amount. |

Completed example: `slotData.Mission.Completed[missionKey] = true`.

## Current Shape Example

```lua
{
    Version = 2,
    Profile = {
        DisplayName = "Player",
        PlayTimeSeconds = 0,
        LastSceneKey = "World"
    },
    PlayerPosition = {
        X = 0,
        Y = 0,
        Z = 0
    },
    Actors = {
        {
            configId = "playerWarrior",
            jobType = "Warrior",
            level = 1,
            constitution = 10,
            dexterity = 6,
            intelligence = 4,
            will = 6,
            perception = 5,
            hp = 125,
            mp = 32,
            stamina = 78
        }
    },
    Party = {
        Members = {
            {
                ActorKey = "playerWarrior",
                Level = 1,
                Exp = 0,
                Hp = 120,
                Mp = 20
            }
        },
        Formation = { "playerWarrior" }
    },
    Inventory = {
        Items = {
            { ItemKey = "hpPotionSmall", Count = 5 },
            { ItemKey = "mpPotionSmall", Count = 3 },
            { ItemKey = "sealedLetter", Count = 1 }
        },
        Equipped = {}
    },
    Skill = {
        Learned = {
            { SkillKey = "normalAttack", Level = 1 }
        },
        Hotkeys = {
            Slot1 = "normalAttack"
        }
    },
    Mission = {
        Active = {},
        Completed = {}
    }
}
```

## Change Checklist

Before adding or changing a saved value:

1. Search this document for the exact key and intended path.
2. Confirm another section does not already own the same information.
3. Use PascalCase for root / subsystem keys; use lowerCamelCase for `Actors[]`
   combat fields (same as `BattleActorCom` / `actorConfig`).
4. Update both the producer (`CreateDefaultData` / `ExportSaveData`) and consumer
   (`LoadUserData` / `ImportSaveData`).
5. Increment `Version` when compatibility or migration logic is required.
6. Update this document in the same change.
7. Verify one save and one reload through Maker logs.
