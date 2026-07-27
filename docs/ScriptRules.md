# MSW Project Architecture & Script Rules

This document is the project convention for folder layout, new-system workflow,
naming, and Logic / Component responsibility boundaries.

## New System Workflow

Build each game system in this order:

```text
1. Design doc   docs/{System}/ or docs/Design/{System}/
2. Config       RootDesk/MyDesk/Data/Config/ (or Config/skill/ for skill presentation tables)
3. Logic        RootDesk/MyDesk/Logic/{System}Logic.mlua
4. Component    RootDesk/MyDesk/{Domain}/.../*.mlua — bind on the target entity in Maker
```

Do not skip ahead. Config keys and Logic entry points should exist in the design
doc before implementation starts.

Example (battle):

```text
docs/BattleFlow/BattleFlowLogic.md
  -> encounterConfig.csv (planned)
  -> Logic/BattleFlowLogic.mlua
  -> Battle/BattleSystem.mlua, Battle/BattleSkill.mlua (planned)
  -> ui/BattleUI.ui + UI bindings (existing)
```

## Workspace Layout

```text
c:\github\
├── map/                              # .map files (Maker scan)
├── ui/                               # .ui files — flat, no subfolders
│   ├── mainMenu.ui
│   ├── BattleUI.ui
│   └── ...
├── docs/                             # Design & architecture (Git only)
└── RootDesk/MyDesk/
    ├── Logic/                        # All @Logic scripts
    ├── MapScene/                     # Map-root @Component scripts
    ├── UI/                           # UI @Component scripts (may use subfolders per screen)
    ├── Battle/                       # Battle @Component scripts
    ├── NPC/                          # NPC / dialogue @Component scripts
    ├── Models/{Category}/            # .model templates — reuse / runtime spawn only
    └── Data/
        ├── Config/                   # Static design tables (default flat)
        │   ├── skillConfig.csv       # Skill balance stays at Config root
        │   ├── skill/                # skillRange / skillEffect / presentation DataSets
        │   └── skillAnimation/       # Skill animation JSON assets
        └── PlayerData/               # Save slot schemas
```

## Folder Rules

### Config — flat by default; documented system tables may use a folder

Most static game tables live directly in `Data/Config/` as `.csv` + `.userdataset`
pairs. Skill presentation tables live under `Data/Config/skill/`; the four
Mission tables live under `Data/Config/mission/`. Do not invent other category
subfolders unless documented here.

| Table | Path | `GetTable` name | Purpose |
|-------|------|-----------------|---------|
| `actorConfig` | `Config/` | `"actorConfig"` | Actor base stats |
| `skillConfig` | `Config/` | `"skillConfig"` | Skills (`iconRuid`, balance); no range geometry |
| `skillRange` | `Config/skill/` | `"skillRange"` | Skill range shape/size (`keyName` == `skillKey`) |
| `skillEffect` | `Config/skill/` | `"skillEffect"` | Timed skill logic effects (`skillKey` join) |
| `skillPresentationConfig` | `Config/skill/` | `"skillPresentationConfig"` | Skill presentation header |
| `skillPresentationStepConfig` | `Config/skill/` | `"skillPresentationStepConfig"` | Skill presentation timeline steps |
| `consumableConfig` | `Config/` | `"consumableConfig"` | Consumables |
| `equipmentConfig` | `Config/` | `"equipmentConfig"` | Equipment |
| `inventoryConfig` | `Config/` | `"inventoryConfig"` | Inventory items |
| `missionConfig` | `Config/mission/` | `"missionConfig"` | One-time mission identity and lifecycle |
| `missionObjectiveConfig` | `Config/mission/` | `"missionObjectiveConfig"` | Mission objective rows |
| `missionRewardConfig` | `Config/mission/` | `"missionRewardConfig"` | Mission reward rows |
| `missionActionConfig` | `Config/mission/` | `"missionActionConfig"` | Accept / complete side effects |
| `UIRegistry` | `Config/` | `"UIRegistry"` | UI key → entity lookup |

Runtime lookup uses the `name` field inside each `.userdataset`, not the file path.

### No Catalog

Do not maintain a separate sprite / RUID catalog DataSet.

- **Runtime / game-data images** (skill icon, item icon): store RUID in the
  relevant Config column (e.g. `skillConfig.iconRuid`).
- **UI layout images** (frames, backgrounds, placeholders): set directly in
  Maker on `SpriteGUIRendererComponent` for live preview; persisted in `.ui`.

### UI — flat only

All `.ui` files live directly under `ui/`. Do not create subfolders such as
`ui/Battle/` or `ui/MainMenu/`.

UI runtime scripts (button binding, panel logic) stay in `RootDesk/MyDesk/UI/`.
Subfolders per screen are allowed (e.g. `UI/mainMenu/`). Bind the script on the
UI entity in Maker; register the UI root in `UIRegistry`.

### Logic — centralized

All `@Logic` scripts live in `RootDesk/MyDesk/Logic/`.

Examples: `SceneLogic`, `UIManagerLogic`, `StartController`, `BattleKeys`,
`DataUtil`, `UIToast`, `UIPopup`, planned `BattleFlowLogic`.

Logic survives map transitions. Do not put world/session managers in feature
subfolders.

`DataUtil` holds shared pure helpers (table / DataSet row readers) callable as
`_DataUtil:GetTableString(...)`, `_DataUtil:GetRowString(...)`, etc.

### Component — domain folders

`@Component` scripts live in the folder that matches their domain. Most entities
in this project are single-use (one UI screen, one map hook, one battle session)
and are not reused — **do not require a `.model` just to host a component script**.

| Domain | Script folder | Typical bind target |
|--------|---------------|---------------------|
| UI | `UI/` (subfolders OK) | Entity on `.ui` |
| Map | `MapScene/` | Map root entity in `.map` |
| Battle | `Battle/` | Battle UI entity, spawned session entity, or inline entity |
| NPC | `NPC/` | NPC entity in `.map` or `.ui` |

Rules:

- Keep the `.mlua` near the feature it serves (`Battle/BattleSkill.mlua`, not a generic `Components/` tree).
- Attach the script on the target entity in Maker (`.ui`, `.map`, or `.model`).
- Use `.model` only when the entity composition is **reused** (≥2 map instances) or **spawned at runtime** via `SpawnByModelId`. A `.model` is not the default place for component scripts.

### Entity / Component property binding — no fixed entity IDs

Do not assign a scene entity UUID directly as the default value of an `Entity` or
component-reference property in `.mlua`.

```lua
-- Correct: leave the reference unassigned in source, then bind it in Maker.
property Entity selectedBackground = nil
property ButtonComponent slotButton = nil

-- Wrong: clones keep pointing back to this original prefab / dummy entity.
property Entity selectedBackground = "7f59b0c7-e7e5-4f14-b3cd-4a46e44bce25"
```

Binding rules:

- Bind `Entity`, `Component`, and UI child references by dragging the target from
  the current scene, UI, map, or model hierarchy into the script property in Maker.
- Cloneable entities, dummy UI items, prefabs, runtime-spawned models, and their
  child components must never contain hard-coded source entity UUIDs in `.mlua`.
  Each instance must resolve to its own bound children after cloning.
- Do not use UI Builder binding injection to place UUID defaults into scripts used
  by cloneable UI items. The `.ui` / `.model` instance owns those property values.
- Fixed IDs are allowed only for documented world-wide registry data such as
  `UIRegistry`; they must not be used as a substitute for a local component
  property binding.
- When reviewing a clone-related display or click bug, first check whether a script
  property points back to the prefab / dummy entity instead of the cloned instance.

### Models — reuse and spawn only

`Models/{Category}/` holds `.model` templates for entities that appear multiple
times or are created at runtime. Single-use UI or map entities do not need a
matching `.model`.

## RUID And Images

| Use case | Where to store |
|----------|----------------|
| Skill / item / equipment icon used at runtime | Config column (`iconRuid`) |
| Battle UI slot frame, panel background | Maker → `.ui` |
| World entity sprite | `.model` → `SpriteRendererComponent.SpriteRUID` |

Fill `iconRuid` in Config after uploading the asset in Maker. Scripts read Config;
they do not maintain a separate RUID registry.

## Naming

- Script file names use PascalCase: `SceneLogic.mlua`, `UIManagerLogic.mlua`, `MainMenuUIComponent.mlua`.
- Script declarations match the file name exactly: `script SceneLogic extends Logic`.
- Public scene/UI/data keys use PascalCase strings: `"MainMenu"`, `"World"`, `"CharacterStatus"`, `"Party"`.
- Config row keys use camelCase or PascalCase consistently per table (`skillKey`, `encounterKey`).
- Actor combat fields (`configId`, `maxHp`, five attrs, …) use **lowerCamelCase**
  in `actorConfig`, `BattleActorCom`, and snapshots. SaveSlot uses root
  `Progression` for shared total level/experience, `Actors[].jobs[].level` for
  custom job-level distribution, and lowerCamelCase
  `Actors[].attributeAllocations` for actor-specific points — see
  `docs/Actor/ActorVariableExplain.md`. Actors have no `atk` / 攻擊力.
- Properties, local variables, and parameters use lowerCamelCase: `mainMenuSceneKey`, `userId`, `entryPosition`.
- Methods use PascalCase: `ChangeScene`, `OpenUI`, `GetEntryPosition`.
- Engine lifecycle methods keep the engine spelling: `OnBeginPlay`, `OnEndPlay`, `OnUpdate`.
- Event handler methods use PascalCase and a clear event suffix: `OnNewGameClicked`, `OnSettingClicked`.

## Responsibility Boundaries

- `@Logic` is for world/session-wide managers that survive map transitions.
- Map-specific behavior belongs in a map root `@Component` (`MapScene/`).
- UI interaction behavior belongs in a UI `@Component` (`UI/`).
- Battle, NPC, and other feature behavior belongs in that feature's `@Component` folder (`Battle/`, `NPC/`, …).
- UI opening, closing, duplicate prevention, and key lookup belong in `UIManagerLogic`.
- UI keys and entity lookup data belong in `UIRegistry` DataSet, not scattered UUID literals in scripts.
- Scene selection and top-level map/UI pairing belong in `SceneLogic`.
- Player/map entry positions should be provided by the target map component when possible.
- Static balance data belongs in Config; enum-like system strings belong in `BattleKeys` Logic.

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

## Related Docs

- `docs/BattleFlow/BattleFlowLogic.md` — battle entry and finalization
- `docs/BattleFlow/BattleSystem.md` — turn permission, turn start/end, settlement, operation locks
- `docs/BattleFlow/BattleUIComponent.md` — BattleUI turn-sequence overlay; skill/item/system UIs stay outside battle ownership
- `docs/Actor/BattleActorComponent.md` — per-actor battle state design
- `docs/Actor/BattleActorInitPaths.md` — Player Save vs Monster/NPC Config init
- `docs/Actor/ActorVariableExplain.md` — Actor field meanings / formula review glossary
