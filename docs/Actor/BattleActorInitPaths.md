# BattleActorCom Init Paths

This document defines how `BattleActorCom` chooses its **stats authority** at
startup. One component serves every combatant; the branch is
`statsSource`.

| `statsSource` | Who | Live stats authority | Startup entry |
|---|---|---|---|
| `"Config"` (default) | Monster / NPC | `actorConfig` CSV row | `OnBeginPlay` → `ApplyConfig(configId)` |
| `"Save"` | Player (party actors) | `SaveSlot.Actors[]` via `PlayerDataLogic` | `ImportSaveData` (not `OnBeginPlay`) |

`configId` is only an **archetype key** (`actorConfig.configId`, e.g.
`playerWarrior`, `slime`). It is not “always reload grown stats from CSV”.
Combat field names are **lowerCamelCase** across Config / Actors[] / Com.

Player Save is **slim** (per-actor custom attribute allocations + `jobs[]` /
`activeJobIndex`), while root `Progression` owns one shared level/experience
bar. The live `BattleActorCom.jobType` is the selected active job and its
`level` comes from that job's saved `jobs[].level`; `Progression.Level` is the
shared total allocation budget. Current
`hp`/`mp`/`stamina` are **not** persisted; `BuildRuntimeActorState` fills them
to full max for `ImportSaveData` / battle entry. `maxHp` / def / speed are
computed in `PlayerDataLogic` at load. Monster/NPC stay full Config literals
(and seed currents to max on `ApplyConfig`). Details: `ActorVariableExplain.md`.

Script: `RootDesk/MyDesk/Battle/BattleActorCom.mlua`  
Related: `docs/Actor/BattleActorComponent.md`, `docs/Actor/ActorVariableExplain.md`, `docs/PlayerData/SaveSlotSchema.md`

---

## Decision Rule

```text
Is this entity a player-owned combatant whose level / maxHp / attributes grow
and persist in SaveSlot?

  YES -> statsSource = "Save"
         PlayerDataLogic loads SaveSlot.Actors[] -> ImportSaveData
         OnBeginPlay must NOT ApplyConfig

  NO  -> statsSource = "Config"  (default)
         Maker / spawn sets configId
         OnBeginPlay -> ApplyConfig(configId)
```

---

## Path A — Monster / NPC (`statsSource = "Config"`)

```text
Maker or spawn
  -> attach BattleActorCom
  -> set configId = actorConfig.configId   (e.g. "slime")
  -> leave statsSource = "Config" (default)

OnBeginPlay
  -> ApplyConfig(configId)
  -> fill level / maxHp / hp / five attrs / ... from actorConfig row
  -> RecalculateStats + ClampResources
```

Rules:

- Static design data only. Do not write grown values back into `actorConfig`.
- Do not call `ImportSaveData` / `ExportSaveData` for these entities.
- Missing `configId` on this path is an authoring error.

---

## Path B — Player (`statsSource = "Save"`)

```text
Load / NewGame / Continue
  -> PlayerDataLogic applies SaveSlot
  -> ApplyPrimaryActorToPlayer(userId, slotData.Actors)
       1. Ensure BattleActorCom on DefaultPlayer
       2. battleActor.statsSource = "Save"
       3. runtime = BuildRuntimeActorState(Actors[1])
          -- Config base + attribute allocations + active job level
       4. battleActor:ImportSaveData(runtime)
            a. ApplyConfig(configId)     -- archetype seed
            b. Overlay slim Save + computed runtime fields (incl. full currents)
            c. RecalculateStats + Clamp

    OnBeginPlay (if it runs with statsSource already "Save")
  -> skip ApplyConfig
  -> wait for / already applied ImportSaveData
```

Rules:

- Save stores shared progression plus slim actor customization; capacities and
  currents are never authoritative in Save.
- New-game seed: `CreateDefaultActorList()` writes slim Actors only (no currents).
- Flush: `ExportSaveData()` → `PlayerDataLogic` writes runtime level into the
  active job, converts attribute totals to allocations, and writes slim
  `Actors[1]`.

Flush on save:

```text
PlayerDataLogic:CollectPlayerActors
  -> BattleActorCom:ExportSaveData()   -- runtime identity/attribute totals
  -> PlayerDataLogic converts totals to attributeAllocations
  -> PlayerDataLogic updates jobs[activeJobIndex].level
  -> slotData.Actors[1]
```

---

## What Lives Where

| Data | Config (`actorConfig`) | Save (`Actors[]`) | Runtime (`BattleActorCom`) |
|---|---|---|---|
| Archetype / job / level | one initial job/level | jobs with allocated levels + shared total at root | active job + its level |
| Five attributes | Monster literals; player base template | custom allocations only | base + allocation |
| Current `hp` / `mp` / `stamina` | seed = max | **no** (fill full on import) | yes (session) |
| Derived (`maxHp`, `defense`, `speed`, …) | Monster literals in Config | **no** | yes (PlayerDataLogic → Import) |
| `atk` / 攻擊力 | **not in Config** | **no** | **no** (Wrapper cast-time only) |
| `totalDefense` | from Config `defense` | **no** | yes (`RecalculateStats`) |

---

## Maker Checklist

| Entity | `statsSource` | `configId` | Who fills stats |
|---|---|---|---|
| Slime / monster model | `"Config"` (default) | required (`slime`, …) | `OnBeginPlay` |
| Village NPC | `"Config"` (default) | required | `OnBeginPlay` |
| DefaultPlayer | `"Save"` | empty until Import, or set only as archetype hint | `PlayerDataLogic` |

---

## Anti-Patterns

- Do not use `OnBeginPlay` + `ApplyConfig` as the player’s final stats source.
- Do not store a second authoritative HP on `BattleSystem.actorMap` or payload.
- Do not grow player `maxHp` by editing `actorConfig.csv` at runtime.
- Do not invent a second `PlayerActorCom` that also owns HP.

---

## Code Touchpoints

| Step | File / API |
|---|---|
| Branch at begin play | `BattleActorCom:OnBeginPlay` |
| Config fill | `BattleActorCom:ApplyConfig` |
| Save fill | `BattleActorCom:ImportSaveData` / `ExportSaveData` |
| Player apply | `PlayerDataLogic:ApplyPrimaryActorToPlayer` |
| Player collect | `PlayerDataLogic:CollectPlayerActors` |
| Schema | `docs/PlayerData/SaveSlotSchema.md` → `Actors[]` |
