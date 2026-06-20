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
| Serialization | `_UtilLogic:TableToString(slotData)` |
| Authority | Server only: `PlayerDataLogic` |

Field names are case-sensitive. `Actors`, `actors`, and `Actor` are different
keys. Public save-data keys use PascalCase exactly as written below.

## Root Fields

| Exact key | Type | Required | Owner | Notes |
|---|---|---:|---|---|
| `Version` | integer | Yes | `PlayerDataLogic` | Current schema version is `1`. |
| `Profile` | table | Yes | `PlayerDataLogic` | Slot metadata. |
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

## Actors

Path: `slotData.Actors[index]`

| Exact key | Type | Default for `Actors[1]` | Source / meaning |
|---|---|---|---|
| `ConfigId` | string | `"playerWarrior"` | Matches `actorConfig.actorKey`. |
| `Level` | integer | `1` | Persistent actor level. |
| `Hp` | integer | `120` | Current HP. |
| `MaxHp` | integer | `120` | Current maximum HP after persistent growth. |
| `Mp` | integer | `20` | Current MP. |
| `MaxMp` | integer | `20` | Current maximum MP after persistent growth. |
| `Stamina` | integer | `100` | Current stamina. |
| `MaxStamina` | integer | `100` | Current maximum stamina. |
| `BaseAttack` | integer | `12` | Persistent attack before temporary modifiers. |
| `BaseDefense` | integer | `5` | Persistent defense before temporary modifiers. |
| `Speed` | number | `3` | Persistent speed value. |

Not saved because they are derived runtime caches:

- `totalAttack`
- `totalDefense`

`BattleActorCom:ImportSaveData()` loads `ConfigId` defaults first, restores the
saved values above, then recalculates derived totals.

## Party

Path: `slotData.Party`

| Exact key | Type | Notes |
|---|---|---|
| `Members` | array<table> | Party actor records. |
| `Formation` | array<string> | Actor keys in formation order. |

Path: `slotData.Party.Members[index]`

| Exact key | Type | Default member |
|---|---|---|
| `ActorKey` | string | `"playerWarrior"` |
| `Level` | integer | `1` |
| `Exp` | integer | `0` |
| `Hp` | integer | From `actorConfig.maxHp` |
| `Mp` | integer | From `actorConfig.maxMp` |

### Known Duplicate Authority

`Party.Members[index]` currently duplicates `Level`, `Hp`, and `Mp` from
`Actors[index]`. Do not add new writes to both locations. Before implementing
party-member persistence, choose one source of truth. Recommended direction:

- `Actors` owns persistent actor identity, progression, and resources.
- `Party.Members` stores actor references only, or is removed.
- `Party.Formation` stores the ordered actor references used by the party.

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
    Version = 1,
    Profile = {
        DisplayName = "Player",
        PlayTimeSeconds = 0,
        LastSceneKey = "World"
    },
    Actors = {
        {
            ConfigId = "playerWarrior",
            Level = 1,
            Hp = 120,
            MaxHp = 120,
            Mp = 20,
            MaxMp = 20,
            Stamina = 100,
            MaxStamina = 100,
            BaseAttack = 12,
            BaseDefense = 5,
            Speed = 3
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
3. Use PascalCase for every public save key.
4. Update both the producer (`CreateDefaultData` / `ExportSaveData`) and consumer
   (`LoadUserData` / `ImportSaveData`).
5. Increment `Version` when compatibility or migration logic is required.
6. Update this document in the same change.
7. Verify one save and one reload through Maker logs.

