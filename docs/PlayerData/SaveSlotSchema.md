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
| `Version` | integer | Yes | `PlayerDataLogic` | Current schema version is `6`. v2 added `PlayerPosition` + JSON; v3 Actors slim lowerCamelCase; v4 drops persisted current `hp`/`mp`/`stamina`; v5 moves inventory ownership into each `Actors[].inventory`; **v6 replaces the saved actor-level `jobType` / `level` pair with `jobs[]` + `activeJobIndex`**. |
| `Profile` | table | Yes | `PlayerDataLogic` | Slot metadata. |
| `PlayerPosition` | table | Yes | `PlayerDataLogic` | Last world position captured when the slot is saved. |
| `Actors` | array<table> | Yes | `PlayerDataLogic` / `BattleActorCom` | `Actors[1]` is currently applied to DefaultPlayer. |
| `Party` | table | Yes | `PartyLogic` | Party membership and formation. |
| `Inventory` | table | Yes | `InventoryLogic` | Legacy compatibility shell. On v4/older load, its contents migrate once into `Actors[1].inventory`; v5 saves it empty. |
| `Skill` | table | Yes | `SkillLogic` | Learned skills and hotkeys. |
| `Mission` | table | Yes | `MissionLogic` | Active and completed missions. |
| `ShopState` | table | Planned | `ShopLogic` | **TODO:** Per-player vendor state. Missing shops are seeded from Config once; initialized shops load from PlayerData thereafter. |

## ShopState (Planned)

TODO — transaction and persistence are not implemented yet.

- `ShopUICom` will send buy/sell intent to the server after confirmation; it must
  never change inventory, currency, or vendor stock locally.
- `ShopLogic` will validate the request and perform the authoritative currency,
  player inventory, and vendor stock transfer before returning refreshed snapshots.
- `shopConfig` and `shopProductConfig` are initialization templates only. A shop is
  read from Config when that player has no saved state for it, such as the first
  open in a new slot.
- After initialization, opening the shop and loading a save use
  `slotData.ShopState`.
- Successful transactions update the in-memory SaveSlot section and mark the user
  dirty through `PlayerDataLogic`; persistence still occurs through the normal
  explicit SaveSlot flow.

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
| `jobs` | array<table> | `{ { jobType = "Warrior", level = 1 } }` | Every job acquired by this saved actor. The template's initial entry is seeded from the single `actorConfig.jobType` / `actorConfig.level` pair. |
| `activeJobIndex` | integer | `1` | One-based index into `jobs`; selects the job imported into the live `BattleActorCom`. |
| `constitution` | integer | `10` | 體質 — allocatable. |
| `dexterity` | integer | `6` | 靈巧 — allocatable. |
| `intelligence` | integer | `4` | 智力 — allocatable. |
| `will` | integer | `6` | 意志 — allocatable. |
| `perception` | integer | `5` | 感知 — allocatable. |
| `inventory` | table | See Inventory | Per-character inventory. Every owned actor has one, whether or not it appears in `Party.Formation`. |

### Multi-Job Shape

`actorConfig.csv` remains a design-template table and therefore contains only
one `jobType` and one `level` per row. When a new player actor is created,
`PlayerDataLogic:CreateDefaultActorList()` converts those two columns into the
first saved `jobs[]` entry. Additional jobs exist only in PlayerData.

```lua
jobs = {
    { jobType = "Warrior", level = 12 },
    { jobType = "Thief", level = 5 }
},
activeJobIndex = 1
```

Path: `slotData.Actors[actorIndex].jobs[jobIndex]`

| Exact key | Type | Notes |
|---|---|---|
| `jobType` | string | Canonical job key. Its spelling follows the job keys used by gameplay/config. |
| `level` | integer | Level owned by this specific job; minimum `1`. |

- Array order is stable presentation order. `activeJobIndex` explicitly selects
  the currently equipped job; callers must not assume index `1` is always active.
- The runtime `BattleActorCom.jobType` / `BattleActorCom.level` pair represents
  only the active job. It is not the authoritative collection.
- On save, `CollectPlayerActors()` updates the active `jobs[]` entry from the
  runtime pair while preserving every inactive job.

### v5 and older migration

When an actor has no valid `jobs[]`, `NormalizeActorJobs()` migrates:

```text
actor.jobType + actor.level
        ↓
actor.jobs[1].jobType + actor.jobs[1].level
actor.activeJobIndex = 1
```

If the legacy pair is also missing, the initial job is recovered from the
actor's `actorConfig` row. Legacy top-level `jobType`, `JobType`, `level`, and
`Level` keys are removed from the in-memory save and are not written by the next
explicit save.

**Do not persist** (filled or recomputed at runtime):

- **Current** `hp` / `mp` / `stamina` — not remembered; `BuildRuntimeActorState`
  / battle entry fills **full** (`= max*`, buff TBD) into `BattleActorCom`
- Derived: `maxHp` / `maxMp` / `maxStamina` / `defense` / `speed` / `jumpForce` /
  `castRange` / `mpCostRate` / `recoveryRate` / `resistance` / `effectPotency` /
  `effectHitRate` / `criticalRate`
- Attack / `atk`: never persisted (cast-time only in Wrapper)
- `totalDefense`

Load: slim `Actors[]` → `BuildRuntimeActorState` (currents = full) →
`BattleActorCom:ImportSaveData(runtime)`.  
Export: `BattleActorCom:ExportSaveData()` exposes the active runtime pair;
`PlayerDataLogic:CollectPlayerActors()` merges it into `jobs[activeJobIndex]`,
preserves inactive jobs, and writes the slim v6 actor shape.

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

Authoritative path in schema v5: `slotData.Actors[index].inventory`

| Exact key | Type | Notes |
|---|---|---|
| `Items` | array<table> | Item stacks by slot index. |
| `Equipped` | table | Equipment slot key to item key; currently empty by default. |

Path: `slotData.Actors[actorIndex].inventory.Items[index]`

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

Only the default primary actor receives these starter stacks. Newly acquired
reserve actors start with an empty inventory unless their acquisition flow
explicitly supplies items.

### Legacy root Inventory migration

For slots written before schema v5:

1. If `Actors[1].inventory` is absent and root `Inventory` exists, the entire
   root payload is assigned to `Actors[1].inventory`.
2. This preserves previously collected items such as `sapphire`.
3. The next explicit save writes every actor inventory under `Actors[]` and
   leaves root `Inventory` empty.
4. Inventory UI snapshots enumerate all `Actors[]`; `Party.Formation` only
   selects which actor receives a newly picked-up field item.

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
    Version = 6,
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
            jobs = {
                { jobType = "Warrior", level = 1 }
            },
            activeJobIndex = 1,
            constitution = 10,
            dexterity = 6,
            intelligence = 4,
            will = 6,
            perception = 5,
            inventory = {
                Items = {
                    { ItemKey = "hpPotionSmall", Count = 5 },
                    { ItemKey = "mpPotionSmall", Count = 3 },
                    { ItemKey = "sealedLetter", Count = 1 }
                },
                Equipped = {}
            }
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
    Inventory = { Items = {}, Equipped = {} },
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
