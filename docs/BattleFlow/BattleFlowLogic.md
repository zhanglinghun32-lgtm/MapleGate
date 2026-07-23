# BattleFlowLogic Design

This document defines entering, leaving, and finalizing turn-based battle mode.

## Current Direction

```text
BattleFlowLogic   -> start payload, spawn session, enter/exit client shell, post-battle rewards
BattleSystem      -> turn permission, turn start/end, settlement, finish result
BattleClientLogic -> BattleUI overlay + camera + sync forward; not skill/item UI
```

Shared gameplay (skills, items, movement, Party/System UI) exists **with or
without** battle. Battle only adds turn permission and operation locks.

```text
Battle owns:          current actor, turn start/end, settlement
Battle does not own:  skill/item cast + their UI, opening Party/System/nav UIs
Battle must provide:  ability to lock operations (e.g. block party edit in battle)
```

## Responsibility Split

### BattleFlowLogic

Path: `RootDesk/MyDesk/Logic/Battle/BattleFlowLogic.mlua`

Responsibilities:

- Own the **single public battle-entry API** used by all trigger sources.
- Receive battle start requests from dialogue, player attack, monster aggro, or
  any other system that needs to start a battle.
- Validate the player can enter battle (not already in battle, valid map, etc.).
- Resolve trigger context into a start payload (including whole-party snapshot
  when Party is ready).
- Spawn / find `BattleSystem.model` and call `StartBattle(payload)`.
- Send `EnterBattleClient` / `ExitBattleClient` shell RPCs.
- On finish: rewards, save touches, mission progress, scene flow.

Non-responsibilities:

- Do not own the turn loop or current-actor permission.
- Do not cast skills, use items, or bind those UIs.
- Do not open Party / System / navigation UIs (except via normal UI managers if
  a post-battle flow explicitly needs it).
- Do not own NPC dialogue trees, monster aggro AI, or field attack hit detection.
- Do not relay every turn event to the client.

### BattleSystem

Path: `RootDesk/MyDesk/Battle/BattleSystem.mlua`

Responsibilities:

- Turn permission (`battlePhase`, `currentActorId`).
- Turn start / turn end.
- Settlement after accepted turn actions; victory / defeat / escape / cancel.
- Publish operation lock policy for other systems.
- Report finish to `BattleFlowLogic`.

Non-responsibilities:

- Skill/item execution and display.
- Opening system or navigation UIs.
- See `docs/BattleFlow/BattleSystem.md` for the full boundary.

### BattleClientLogic

Path: `RootDesk/MyDesk/Logic/Battle/BattleClientLogic.mlua`

Responsibilities:

- Open/close **BattleUI** turn-sequence overlay only.
- Keep persistent UIs available: ControlCharacterUI, Party, System (per product).
- Attach/detach battle session refs for sync consumers.
- Battle camera.
- Forward `@Sync` phase/actor updates.
- Acknowledge action presentation completion to `BattleSystem` when the session
  waits on presentation.

Non-responsibilities:

- Skill hotbar, item UI, party formation edits.
- Deciding legal targets or mutating server battle state.
- Becoming the owner of `PlayerController` beyond requesting locks through
  `PlayerControlLogic`.

## Battle Entry Triggers

Field content may start a battle in **three** product ways. All of them must call
**one** public entry on `BattleFlowLogic`. Callers never spawn `BattleSystem`
themselves and never talk to `BattleClientLogic` for shell open.

### The three triggers

| # | Trigger | Typical caller | Who initiates |
|---|---------|----------------|---------------|
| 1 | After NPC dialogue | Dialogue / NPC script after a choice or end line | Story / NPC system |
| 2 | Player actively attacks a unit (NPC / monster) | `FieldNormalAttack` (or future field skill hit) on confirmed hit | Player action |
| 3 | Monster discovers the player | Monster aggro / sight / chase AI | Monster / field AI |

Same downstream path after the unified API accepts the request:

```text
any trigger
  -> BattleFlowLogic public entry
  -> validate + build payload
  -> spawn session + BattleSystem:StartBattle
  -> EnterBattleClient
```

### Unified public interface

**Owner:** `BattleFlowLogic` only.

```text
RequestStartBattle(request) -> boolean
```

Rules:

- This is the **only** supported way for other systems to start a battle.
- Dialogue, field attack, and monster aggro are thin adapters that fill a
  `request`, then call `RequestStartBattle`.
- Internal helpers such as today's `StartFieldBattle` / `StartBattle` should be
  folded under or behind this API; external scripts must not keep adding new
  public start methods per trigger type.
- Return `false` when ignored (already in battle, invalid request, spawn failed).
  Callers must tolerate a rejected start without corrupting their own state.

### Two-layer contract

```text
Caller fills BattleStartRequest  (EntityId refs + trigger)
  -> BattleFlowLogic resolves entities, ensures BattleActorCom on each
  -> BattleSystem receives BattleStartPayload (roster of EntityIds + session meta)
```

**HP / MP / stamina / stats live on the entity `BattleActorCom`, not in the
payload as a second source of truth.**

UI (bars, numbers, death cues) must read the **same** actor component the
server mutates. There is **no** “config-only monster” path: every battle unit
is a live map entity with `BattleActorCom` already carrying:

```text
maxHp  — capacity / 一般狀態上限
hp     — current / 當前狀態
```

(and the same pattern for `maxMp`/`mp`, `maxStamina`/`stamina`).

Neutral civilians / non-hostile NPCs are **never** listed in any battle party.
They stay on the field outside the session until something makes them hostile;
only then may a later request or mid-battle event pull them in as Enemy (or
Ally, if designed that way).

Inventory / items are **not** part of battle entry. Item use and throw go through
`InventoryLogic` (same as outside battle). In battle, `BattleSystem` only gates
the turn and whether the action is allowed; using or throwing an item spends
**action points** for that turn.

---

## BattleStartRequest (external input)

What adapters pass into `RequestStartBattle`. PascalCase public keys.

### Top-level fields

| Key | Type | Required | Who fills | Notes |
|-----|------|:--------:|-----------|-------|
| `TriggerType` | string | Yes | Adapter | `"Dialogue"` \| `"PlayerAttack"` \| `"MonsterDetect"` \| future |
| `PlayerUserId` | string | Yes* | Adapter or derived | *If omitted, Flow derives from `PlayerEntity` / `senderUserId` |
| `PlayerEntityId` | string | Yes* | Adapter | Field entity id of the player-side sponsor (active field avatar). *Required unless Flow can resolve from `PlayerUserId` alone |
| `MapEntityId` | string | No | Adapter / Flow | Defaults to player `CurrentMap` |
| `EncounterKey` | string | No | Adapter | **Type / rule key only** (rewards, battle rules, analytics). Not the enemy roster when `EnemyRefs` is present |
| `RewardKey` | string | No | Adapter / encounter | Empty = resolve from `EncounterKey` Config if any |
| `BattleRuleKey` | string | No | Adapter / encounter | Escape rules, turn caps, ambush policy, etc. |
| `CanEscape` | boolean | No | Adapter / encounter | Default from `BattleRuleKey` / encounter |
| `OpeningActionKey` | string | No | PlayerAttack | e.g. `"normalAttack"` already landed |
| `SourceKey` | string | No | Adapter | Logging / mission / analytics |
| `DialogueKey` | string | No | Dialogue | Conversation that caused the fight |
| `EnemyRefs` | array | Yes | Adapter | **Authoritative enemy roster seed.** At least one primary foe. Flow expands with pull-in radius |
| `AllyRefs` | array | No | Adapter | Optional **友軍** (AI-only friendlies) |
| `Seed` | integer | No | Adapter / Flow | Optional RNG seed for deterministic tests |

Do **not** pass `Environment` on the request. Flow reads weather / terrain /
map objects from the **current map defaults** (nothing invented out of thin air).

Do **not** pass inventory / item stacks. Items stay owned by `InventoryLogic`.

`EnemyRefs` / `AllyRefs` entry shape (`BattleUnitRef`):

| Key | Type | Required | Notes |
|-----|------|:--------:|-------|
| `EntityId` | string | **Yes** | Live field entity. Every battle unit must already exist on the map |
| `ActorId` | string | No | Stable battle id override; Flow assigns if empty |
| `TeamHint` | string | No | Rare override; default from which list the ref sits in |
| `Position` | table | No | Rare override; default from entity `TransformComponent` |

`configId` is **not** a substitute for a missing entity. Config is read by
`BattleActorCom` on that entity (`ApplyConfig` / save import), not by spawning
a phantom unit from the request alone.

```text
EnemyRefs = seed hostile **entities** (Dialogue / attack target / spotting monster)
AllyRefs  = 友軍 entities — AI controlled; not player party members
Player party is NEVER passed as refs here — Flow always pulls Formation entities from PartyLogic
```

### Enemy pull-in radius (PlayerAttack and MonsterDetect)

After the adapter supplies the **primary** foe(s) in `EnemyRefs`, Flow expands
the enemy list:

```text
For each primary EnemyRef with a live EntityId:
  read that monster's Config pull-in radius
  collect every battle-capable monster entity within that radius on the same map
  merge into EnemyParty (dedupe by EntityId)
```

Rules:

- Same radius rule for **player attacking a monster** and **monster discovering
  the player** — the primary monster's Config radius decides who else is pulled in.
- Civilians / neutrals in the radius are **not** pulled in.
- No soft cap on enemy / ally / map-object counts.
- Dialogue adapters should already build the intended `EnemyRefs` (including any
  scripted extras). Radius expansion still applies to live monster primaries
  unless a future battle-rule flag disables it (TODO if needed).

Config column name TBD (working name: `battlePullRadius` on monster /
`actorConfig`). Marked TODO in agent list until the Config column is added.

### What callers do **not** pass

- HP / MP / stamina / attribute numbers (those live on each entity's `BattleActorCom`; no actor `atk`)
- Full skill tables as authority (optional display hints only; prefer reading
  from actor / SkillLogic via the entity)
- Inventory / item stacks (owned by `InventoryLogic`; not battle payload)
- Environment inventing weather/terrain (Flow reads **map defaults**)
- Neutral / civilian entity ids (ignored if sent; not battle units)
- UI open commands
- Config-only foe rows without an `EntityId`

### Minimal examples per trigger

**Dialogue**

Hostile NPC / dialogue flow builds `EnemyRefs` before calling start.
`EncounterKey` is only a type/rule key, not the enemy list source.

```lua
{
    TriggerType = "Dialogue",
    PlayerUserId = userId,
    PlayerEntityId = playerEntity.Id,
    DialogueKey = "Quest_Intro_FightOffer",
    EncounterKey = "Quest_IntroBattle_01",  -- type / rewards / rules only
    EnemyRefs = {
        { EntityId = hostileNpc.Id }  -- adapter prepared; authoritative roster seed
    },
    AllyRefs = {}  -- optional escort: { EntityId = escort.Id }
}
```

**PlayerAttack**

```lua
{
    TriggerType = "PlayerAttack",
    PlayerUserId = userId,
    PlayerEntityId = playerEntity.Id,
    OpeningActionKey = "normalAttack",
    EnemyRefs = {
        { EntityId = targetMonster.Id }  -- primary; Flow pulls others by Config radius
    }
}
```

**MonsterDetect**

```lua
{
    TriggerType = "MonsterDetect",
    PlayerUserId = userId,
    PlayerEntityId = playerEntity.Id,
    EnemyRefs = {
        { EntityId = spottingMonster.Id }  -- primary; Flow pulls others by Config radius
    }
}
```

---

## BattleStartPayload (session input to BattleSystem)

Built only by `BattleFlowLogic`. `BattleSystem:StartBattle(payload)` consumes this.

### Top-level

| Key | Type | Notes |
|-----|------|-------|
| `BattleId` | string | Flow-generated unique id |
| `TriggerType` | string | Copied from request |
| `PlayerUserId` | string | Session owner |
| `SponsorEntityId` | string | Field avatar that triggered / was targeted |
| `SceneKey` | string | e.g. `"World"` |
| `MapName` | string | Map entity name |
| `EncounterKey` | string | May be empty for pure entity fights |
| `RewardKey` | string | |
| `BattleRuleKey` | string | |
| `CanEscape` | boolean | |
| `MaxTurnCount` | integer | `0` = unlimited |
| `OpeningActionKey` | string | Optional |
| `SourceKey` | string | Optional |
| `DialogueKey` | string | Optional |
| `PlayerParty` | array | 我方 — player-controlled / party formation |
| `EnemyParty` | array | 敵方 (after radius expansion) |
| `AllyParty` | array | 友軍 — AI only; empty if none |
| `Environment` | table | Copied from **current map defaults** |
| `Seed` | integer | Optional |

### Team semantics

| List | Control | Starts as | Friendly-fire / betrayal |
|------|---------|-----------|--------------------------|
| `PlayerParty` | Player (turn input when current) | Player team | N/A |
| `AllyParty` | **AI only** | Friendly to player | **TBD — not locked.** Preliminary concept only: e.g. betray after 2 mistaken hits, or immediately if the damaging skill is tagged to force betrayal. Leave policy unimplemented until design settles |
| `EnemyParty` | AI (or future special rules) | Hostile | — |

```text
Civilians / neutral NPCs: not in any of these lists at start.
They are not battle units until hostility is established.
No soft caps on party / enemy / ally / map-object counts.
```

### Actor HP authority (locked)

```text
Entity + BattleActorCom
  maxHp / maxMp / maxStamina     = 一般狀態（上限）
  hp / mp / stamina              = 當前狀態
  base* / total*                 = 攻防等（total 為衍生快取）

UI 顯示  ----reads---->  同一顆 BattleActorCom（@Sync 或查詢）
扣血死亡 ----writes---->  同一顆 BattleActorCom（ApplyDamage / IsDead）
BattleSystem ----------->  只持有 ActorId ↔ EntityId 名冊與回合權限
```

Rules:

1. Every unit in `PlayerParty` / `EnemyParty` / `AllyParty` **must** have a
   non-empty `EntityId` pointing at a valid map entity with `BattleActorCom`.
2. There is **no config-only / phantom monster** in battle. Config initializes
   the component on the entity before or as the unit appears on the field.
3. Payload must **not** become a second HP store. Optional numeric copies in
   payload (if any) are debug/bootstrap only and are **legacy to remove**.
4. Client HUD binds to the entity actor (or its synced component fields), not to
   `BattleSystem.actorMap[actorId].Hp`.

### BattleActorEntry (roster row — identity, not resource authority)

| Key | Type | Required | Notes |
|-----|------|:--------:|-------|
| `ActorId` | string | Yes | Unique in this battle |
| `EntityId` | string | **Yes** | Live entity; required for all teams |
| `ActorType` | string | Yes | `_BattleKeys` actor type |
| `TeamId` | string | Yes | `"Player"` \| `"Ally"` \| `"Enemy"` at start |
| `ControlMode` | string | Yes | `"Player"` \| `"AI"` — AllyParty always `"AI"` |
| `configId` | string | No | Convenience copy of `BattleActorCom.configId` for logs/UI keys |
| `SlotIndex` | integer | No | Party formation slot; `0` if N/A |
| `CanBetray` | boolean | No | Ally only; betrayal rules TBD |

**Removed from authoritative payload design** (do not treat as live HP):

- `hp` / `maxHp` / `mp` / `maxMp` / `stamina` / `maxStamina`
- `baseDefense` / `totalDefense` / `speed` (no `atk` / `totalAttack` on actor)
- Copied `SkillKeys` / `Buffs` as the only skill/buff store

Skills and buffs for gameplay also prefer live actor / effect systems on the
entity. Entry-time buff application still runs on the component (field-carried +
on-enter-battle passives).

**Buffs at entry** (applied on `BattleActorCom` / EffectSystem, not as payload HP):

- Carry **field-already-active** effects on the actor.
- Character **passives** may run an “on battle enter” hook and add opening buffs
  on the same actor.

### Items and inventory (not in payload)

Battle does **not** carry a bag snapshot.

```text
InventoryLogic owns use / throw requests (field and battle)
  -> in battle: BattleSystem only checks turn permission + action-point budget
  -> InventoryLogic mutates the real shared party inventory
```

Using a consumable or throwing a consumable both go through inventory “use”
paths. The battle distinction is **action-point cost** on the current turn
(limited per turn), not a separate battle inventory copy.

Same pattern as skills: display may follow the controlled entity's
`BattleActorCom` / learned list; inventory is queried live from
`InventoryLogic` when the player confirms an item action.

### Environment

Read from the **current map’s defaults**. Do not invent weather, terrain, or
props that are not already on / configured for that map.

| Key | Type | Notes |
|-----|------|-------|
| `WeatherKey` | string | From map default / map component |
| `TerrainKey` | string | Global terrain profile from map |
| `TerrainCells` | array\<table\> | Optional per-cell data already authored on the map |
| `MapObjects` | array\<table\> | Field map objects eligible for this battle (cover, etc.) |

**MapObject entry:**

| Key | Type | Notes |
|-----|------|-------|
| `ObjectId` | string | Battle-local id |
| `ObjectKey` | string | Config key |
| `EntityId` | string | Optional field entity |
| `X` / `Y` | number | |
| `Hp` / `MaxHp` | integer | Optional if destructible |
| `Tags` | array\<string\> | e.g. `"Cover"`, `"Flammable"` |

### Full payload example (roster shape)

```lua
{
    BattleId = "Field_user_123",
    TriggerType = "PlayerAttack",
    PlayerUserId = "user",
    SponsorEntityId = "...",
    SceneKey = "World",
    MapName = "map01",
    EncounterKey = "",
    RewardKey = "",
    BattleRuleKey = "FieldEncounter",
    CanEscape = true,
    MaxTurnCount = 0,
    OpeningActionKey = "normalAttack",
    PlayerParty = {
        {
            ActorId = "Party_user_1",
            EntityId = "player-entity-id",
            ActorType = "Player",
            TeamId = "Player",
            ControlMode = "Player",
            configId = "playerWarrior",
            SlotIndex = 1,
            CanBetray = false
        }
    },
    EnemyParty = {
        {
            ActorId = "Enemy_1",
            EntityId = "monster-entity-id",
            ActorType = "Monster",
            TeamId = "Enemy",
            ControlMode = "AI",
            configId = "slime",
            SlotIndex = 0,
            CanBetray = false
        }
    },
    AllyParty = {},
    Environment = {
        WeatherKey = "",
        TerrainKey = "",
        TerrainCells = {},
        MapObjects = {}
    }
}
```

HP bars read `EntityId` → `BattleActorCom.hp` / `maxHp`, never a payload `Hp` field.

---

## Expansion duties inside BattleFlowLogic

| Payload field | Primary source |
|---------------|----------------|
| `PlayerParty` | Party formation → each member's **field entity** + require `BattleActorCom` |
| `EnemyParty` | `EnemyRefs` EntityIds + **Config pull-in radius** (other monster **entities**) |
| `AllyParty` | `AllyRefs` EntityIds |
| `Environment` | **Current map defaults** only |
| Resources (HP…) | **Never copied as authority** — already on each `BattleActorCom` |
| Inventory | **Not in payload** — live `InventoryLogic` |
| Neutrals | **Excluded** |

```text
TODO(code): Builders resolve EntityId → BattleActorCom; reject missing Com.
Do not write entry.Hp into actorMap as live HP.
```

### Resolved design decisions

| Topic | Decision |
|-------|----------|
| Ally betrayal | **Leave open.** Early idea only: 2 mistaken hits → betray, or immediate betray if the skill is tagged. Do not implement yet |
| Shared inventory in payload | **No.** InventorySys / `InventoryLogic` owns use & throw; battle locks turn + spends action points |
| Entry buffs | Carry **field-active** buffs; passives may apply **on-enter-battle** opening buffs |
| Environment | Read **map defaults**; never invent |
| Pull-in on detect / attack | Primary monster Config **radius** pulls all monsters in range (both triggers) |
| Soft caps | **None** |
| Dialogue enemies | Adapter builds `EnemyRefs`; **`EnemyRefs` wins**. `EncounterKey` is type/rules only |
| Resource authority | **`BattleActorCom` on each entity** (`hp`/`maxHp` …). UI matches entity. No config-only units |
| Payload role | Roster + session meta (`ActorId`↔`EntityId`). Not a live HP duplicate |

---

### Trigger adapters (who calls the unified API)

```text
1) Dialogue
   NPC / Dialogue system finishes a battle-flagged line or choice
     -> fill request.triggerType = "Dialogue"
     -> BattleFlowLogic:RequestStartBattle(request)

2) PlayerAttack
   FieldNormalAttack (or field skill) confirms hit on battle-capable target
     -> fill request.triggerType = "PlayerAttack"
     -> BattleFlowLogic:RequestStartBattle(request)

3) MonsterDetect
   Monster aggro / detection component decides the player is spotted
     -> fill request.triggerType = "MonsterDetect"
     -> BattleFlowLogic:RequestStartBattle(request)
```

Adapter non-responsibilities:

- Do not spawn `BattleSystem.model`.
- Do not open `BattleUI` or lock party UI themselves (shell stays on Flow/Client).
- Do not build turn order or enemy AI inside the adapter.
- Do not invent a second public start API on `GameplayFlowLogic` /
  `BattleSystem` / map scripts. `GameplayFlowLogic` may later *route* to
  `RequestStartBattle`, but must not duplicate payload rules.

### Trigger-specific notes

**Dialogue**

- Dialogue owns conversation UI and when the fight is offered.
- Adapter **must** prepare `EnemyRefs` before `RequestStartBattle`.
- `EncounterKey` is a type/rules key only; enemy roster follows `EnemyRefs`.
- TODO: whether dialogue UI closes before BattleUI overlay opens.

**PlayerAttack**

- Current MVP path: `FieldNormalAttack` -> (today) `StartFieldBattle`.
- Target must be a battle-capable unit (NPC or monster with actor data).
- Miss / no `BattleActorCom` = no battle.
- Adapter passes primary target in `EnemyRefs`; Flow expands by Config
  `battlePullRadius` (name TBD).
- TODO: migrate to `RequestStartBattle`; keep hit detection outside Flow.

**MonsterDetect**

- Monster / field AI owns sight, leash, and “discovered” timing.
- Adapter passes spotting monster as primary `EnemyRefs`; Flow expands by the
  same Config pull-in radius as PlayerAttack.
- TODO: ambush / surprise turn rules (who acts first) are battle-rule policy,
  not a separate start API.

## Runtime Flow

```text
1. Trigger adapter -> BattleFlowLogic:RequestStartBattle(request)
2. Validate + expand (party, enemy radius pull-in, map Environment, field buffs)
3. Spawn BattleSystem.model -> StartBattle(payload)
4. @Sync battlePhase / currentActorId
5. EnterBattleClient
6. BattleClientLogic: open BattleUI overlay; persistent UIs stay
7. Skills -> SkillActionLogic; items use/throw -> InventoryLogic
8. Both ask BattleSystem for turn permission + action-point budget
9. BattleSystem settles turn / checks end
10. HandleBattleFinished -> ExitBattleClient
11. Close BattleUI overlay only; apply rewards / save / scene
```

### Legacy note — field normal attack

```text
A key -> FieldNormalAttack hit -> (today) StartFieldBattle(...)
                                 (target) RequestStartBattle(PlayerAttack)
```

`FieldNormalAttack` remains field hit detection only, not a BattleSystem skill UI.

## Ownership Diagram

```text
Dialogue / FieldNormalAttack / MonsterDetect / other systems
  -> BattleFlowLogic:RequestStartBattle(request)     -- unified entry only

BattleFlowLogic
  -> BattleSystem:StartBattle / HandleBattleFinished
  -> EnterBattleClient / ExitBattleClient

BattleSystem
  -> turn permission + start/end + settlement
  -> operation lock policy for Skill / Party / Item / UI owners
  -> @Sync phase / currentActorId

BattleClientLogic
  -> BattleUI overlay open/close
  -> sync forward to BattleUI + ControlCharacterUI

ControlCharacterUI / Item UI / Party UI / System UI
  -> own their screens and Request* calls
  -> skills via SkillActionLogic; items via InventoryLogic
  -> respect battle operation locks + action points
```

## Client Shell Rules

- Open/close **BattleUI** via Client RPC, never via `@Sync`.
- Do not close ControlCharacterUI / Party / System solely because battle ended.
- Skill confirm never calls `BattleSystem:RequestAction*` from UI.

## Data Sources

`BattleFlowLogic` may read: `PartyLogic`, `SkillLogic`, `MissionLogic`,
`PlayerDataLogic`, live map/actor components, monster Config (pull radius),
map environment defaults.

`InventoryLogic` is **not** snapshotted into the start payload. Battle item
actions call Inventory live, gated by battle turn / action points.

`BattleSystem` receives `BattleStartPayload` only. It must not call DataStorage
or rebuild party from save during the session start path.

## Agent TODO — For Later Implementation (Code Left Blank)

### Unified battle entry (do this before adding more start helpers)

0. **`BattleFlowLogic.mlua`**
   - TODO: Implement `RequestStartBattle(request)` (`BattleStartRequest` →
     expand → `BattleStartPayload` **roster**).
   - TODO: Builders: `BuildPlayerParty`, `BuildEnemyParty` (pull-radius),
     `BuildAllyParty`, `BuildEnvironmentFromMap` — each row **requires EntityId**
     + valid `BattleActorCom`.
   - TODO: **Do not** copy hp/maxHp into payload as authority; remove LEGACY
     `BuildFieldActorEntry` resource dump.
   - TODO: **Do not** build inventory into payload.
   - TODO: Reject empty `EnemyRefs` or refs without `BattleActorCom`.
   - TODO: Ignore / strip neutral entity ids.
   - TODO: Keep `StartFieldBattle` as a temporary adapter until
     `FieldNormalAttack` migrates.
   - TODO: Centralize “already in battle” / map / user validation.
   - TODO: Enter-battle passive buff hook timing (Flow vs TurnStart) — pick one.

1. **Dialogue adapter**
   - TODO: Build `EnemyRefs` before `RequestStartBattle`; set `EncounterKey` only
     as type/rules if needed.
   - TODO: Do not spawn battle session or open BattleUI from dialogue scripts.

2. **PlayerAttack adapter**
   - TODO: `FieldNormalAttack.mlua` → `RequestStartBattle` with primary
     `EnemyRefs` only; Flow expands radius.
   - TODO: Keep hit detection on the field component.

3. **MonsterDetect adapter**
   - TODO: Aggro / detect component → `RequestStartBattle` with spotting monster
     as primary `EnemyRefs`; Flow expands radius.
   - TODO: AI owns sight/leash; Flow owns session start only.

4. **Config**
   - TODO: Add monster pull-in radius column (working name `battlePullRadius`)
     under flat `Data/Config/` (`actorConfig` or monster table).

### Battle shell

5. **`BattleClientLogic.mlua`**
   - TODO: `OpenBattleUI` / `CloseBattleUI` via `UIManagerLogic` on enter/exit.
   - TODO: attach/detach `BattleUIComponent` session; keep ControlCharacterUI up.
   - TODO: push/pop battle-related control locks through `PlayerControlLogic`
     consistently with presentation locks.

6. **`UIManagerLogic.mlua`**
   - TODO: ensure `BattleUI` registry key + open/close helpers exist and do not
     stomp persistent overlays (SystemMenu / ControlCharacter).

7. **`ui/BattleUI.ui` + `BattleUIComponent.mlua`**
   - TODO: layout = turn sequence only; remove skill/item chrome if still in file.
   - TODO: consume forwarded `battlePhase` / `currentActorId` for sequence UI.

### Gateways and locks (battle does not own UI)

8. **`BattleSystem.mlua`**
   - TODO: expose `IsOperationAllowed` / `CanSubmitTurnAction` / action-point
     budget checks (skill + item).
   - TODO: **LEGACY** rewrite `RegisterActorEntry` — roster EntityId only; resolve
     `BattleActorCom` for resources / death.
   - TODO: **LEGACY** `CountAliveActorsByTeam` must use `IsDead()` on components.
   - TODO: shrink skill-resolution ownership into shared skill/action pipeline.
   - TODO: Ally betrayal policy — leave stubbed until design locks (2 hits vs
     skill-tagged instant betray).

9. **`SkillActionLogic.mlua` + `ControlCharacterUIComponent.mlua`**
   - TODO: confirm-only path through SkillAction; disable confirm when battle
     lock denies `SubmitSkill`.
   - TODO: grep and remove UI→`RequestAction*` leftovers.

10. **`InventoryLogic` (+ item UI)**
    - TODO: battle use/throw still call Inventory; query battle permission +
      spend action points; no battle bag copy.

11. **Party**
    - TODO: `PartyLogic:CanEditParty` / `CanSwitchFieldMember` return false while
      battle active.
    - TODO: Party UI / `ControlCharacterUI` PartyMemberList disable those controls
      in battle; opening the panel may still be allowed.

12. **System menu**
    - TODO: document and enforce which SystemMenu actions stay allowed in battle;
      do not put that logic inside `BattleUI`.

### Payload / settlement follow-ups

13. **`BattleFlowLogic.mlua`**
    - TODO: `PlayerParty` from party snapshot (whole party enters).
    - TODO: real reward / save / mission finalization after `HandleBattleFinished`.
    - TODO: write-back actor resources after battle end (inventory stays on
      InventoryLogic continuously).

14. **`BattleActorCom.mlua`**
    - TODO: add `ApplyDamage` / `Heal` / `IsDead`; `hp`/`maxHp` remain the only
      resource authority for UI and settlement (`@Sync` already present).

15. **`BattleQueue.mlua`**
    - TODO: **LEGACY** `RemoveDeadActors` / `GetActorSpeed` — use entity
      `BattleActorCom`, not payload table fields.

16. **`GameplayFlowLogic.mlua`** (optional router only)
    - TODO: if world-flow needs a battle hook, call
      `BattleFlowLogic:RequestStartBattle` — do not duplicate entry validation.

## Related Docs

- `docs/BattleFlow/BattleSystem.md`
- `docs/BattleFlow/BattleUIComponent.md`
- `docs/SkillAction/SkillActionSystem.md`
- `docs/Party/PartySystem.md`
