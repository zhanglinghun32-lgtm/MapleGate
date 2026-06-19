# BattleFlowLogic Design

This document defines the current responsibility split for entering, running,
and leaving turn-based battle mode.

## Current Direction

`BattleFlowLogic` is the battle entry and finalization layer.

`BattleSystem` is the server-authoritative battle session.

`BattleClientLogic` is the client-only presentation layer.

The important rule is:

```text
BattleFlowLogic starts and finalizes battle.
BattleSystem owns the active battle and sends battle client RPC directly.
BattleClientLogic only presents what the server battle session tells it.
```

## Responsibility Split

### BattleFlowLogic

Path:

```text
RootDesk/MyDesk/Logic/BattleFlowLogic.mlua
```

Responsibilities:

- Receive battle start requests from map components, NPCs, encounters, or scene flow.
- Validate whether the target player can enter battle.
- Read save/runtime data from domain Logic scripts.
- Resolve `encounterKey` into static encounter setup.
- Build the complete battle start payload.
- Spawn or find the `BattleSystem.model` battle session entity.
- Call `BattleSystem:StartBattle(payload)`.
- Receive battle-finished callbacks from `BattleSystem`.
- Apply rewards, save changes, mission progress, and scene flow after battle ends.

Non-responsibilities:

- Do not own the turn loop.
- Do not calculate turn order, damage, movement, or action results.
- Do not relay every battle event to the client.
- Do not bind battle UI buttons.
- Do not run battle animations, camera, or control state.
- Do not store permanent save data directly inside the battle session.

### BattleSystem

Path:

```text
RootDesk/MyDesk/Battle/BattleSystem.mlua
```

Runtime holder:

```text
RootDesk/MyDesk/Models/Battle/BattleSystem.model
```

Responsibilities:

- Receive `StartBattle(payload)` from `BattleFlowLogic`.
- Own one active server-authoritative battle session.
- Own actor ids, actor references, queue, phase, current turn, and battle result.
- Resolve turn order.
- Resolve actions, damage, resource changes, movement intent, death, victory, defeat, escape, and cancel.
- Apply battle state through battle actor interfaces.
- Send battle client RPC directly for battle presentation events.
- Report final battle result back to `BattleFlowLogic`.

Non-responsibilities:

- Do not load encounter Config by key.
- Do not read or write permanent save data directly.
- Do not decide post-battle scene transitions.
- Do not bind UI buttons.
- Do not own field monster chase AI.

### BattleClientLogic

Path:

```text
RootDesk/MyDesk/Logic/BattleClientLogic.mlua
```

Responsibilities:

- Handle client-only battle presentation.
- Open and close battle UI through `UIManagerLogic`.
- Disable and restore local player control.
- Stop local movement when entering battle.
- Apply and restore battle camera state.
- Play turn, damage, movement, skill, victory, defeat, and exit presentation.

Non-responsibilities:

- Do not calculate battle results.
- Do not decide valid targets or legal actions.
- Do not mutate server battle state.
- Do not read save data.

## Runtime Flow

Expected first-pass flow:

```text
1. Field / NPC / map trigger calls BattleFlowLogic:CreateBattle(...)
2. BattleFlowLogic validates sponsor, target, and encounterKey.
3. BattleFlowLogic reads domain data from existing Logic scripts.
4. BattleFlowLogic builds the battle start payload.
5. BattleFlowLogic spawns or finds BattleSystem.model.
6. BattleFlowLogic calls BattleSystem:StartBattle(payload).
7. BattleSystem initializes the server battle session and writes @Sync battlePhase / currentActorId.
8. BattleFlowLogic sends EnterBattleClient(battleId, battleEntity, userId) — Client RPC shell only.
9. BattleClientLogic opens BattleUI via UIManagerLogic and attaches BattleUIComponent to battleEntity.
10. BattleUIComponent reads @Sync through BattleSystem replication (forwarded via OnSyncProperty).
11. BattleUIComponent sends player actions through BattleSystem:RequestAction (Client->Server RPC).
12. BattleSystem owns turn order and action resolution.
13. BattleSystem detects battle end and reports to BattleFlowLogic.
14. BattleFlowLogic sends ExitBattleClient(battleId, result, userId).
15. BattleFlowLogic applies rewards, save changes, mission progress, and scene flow.
```

Recommended ownership diagram:

```text
BattleFlowLogic
  -> BattleSystem:StartBattle(payload)
  -> BattleClientLogic:EnterBattleClient / ExitBattleClient (UI shell RPC)

BattleSystem
  -> @Sync battlePhase / currentActorId (display state)
  -> BattleFlowLogic:HandleBattleFinished(...)
  -> action presentation RPC to BattleClientLogic
  -> waits in PresentingAction for client acknowledgement or timeout

BattleClientLogic
  -> UIManagerLogic Open/Close BattleUI
  -> BattleUIComponent Attach/Detach session

BattleUIComponent
  -> reads @Sync via BattleSystem + RequestAction Server RPC
```

See also `docs/BattleFlow/BattleUIComponent.md` for the UI + Sync + input split.

## Client RPC Direction

Battle **shell** events (open/close UI, control lock, camera) are emitted by
`BattleFlowLogic` after the server session starts or finishes.

Battle **display state** during combat (`battlePhase`, `currentActorId`) uses
`@Sync` on `BattleSystem`. The HUD reads sync through `BattleUIComponent`; do not
use `@Sync` to open `BattleUI`.

One-shot **presentation** events are emitted by `BattleSystem` through
`@ExecSpace("Client")` RPC. Action resolution enters `PresentingAction`, and the
server advances to `TurnEnd` only after `BattleClientLogic` acknowledges the
matching presentation id or the server timeout expires.

Example future RPC categories:

```text
TurnStartedClient(battleId, actorId)
ActorMovedClient(battleId, actorId, x, y)
ActorDamagedClient(battleId, actorId, damage, hpAfter)
ActionResolvedClient(battleId, actionResult)
```

These methods should be declared on `BattleSystem` with
`@ExecSpace("Client")`, then call `_BattleClientLogic` locally on the target
client.

Targeting rule:

```lua
self:ActorDamagedClient(battleId, actorId, damage, hpAfter, playerUserId)
```

`playerUserId` is the last call-site argument used to target the client. It is
not declared in the RPC method signature.

`BattleFlowLogic` should not be used as a relay for every turn/action event.
Keep it focused on entry, UI shell, and finalization.

## Battle Start Payload

The first version can use a plain table payload.

Suggested shape:

```lua
{
    battleId = "",
    sceneKey = "",
    mapName = "",
    sponsorEntityId = "",
    playerUserId = "",
    playerParty = {},
    enemyParty = {},
    encounterKey = "",
    rewardKey = "",
    battleRuleKey = "",
    canEscape = true,
    maxTurnCount = 0
}
```

`BattleFlowLogic` is responsible for validating and building this payload before
calling `BattleSystem:StartBattle(payload)`.

`BattleSystem` may trust that required static setup has already been resolved,
but it should still validate the minimum runtime fields it needs to run safely.

## Data Sources

`BattleFlowLogic` reads domain state from existing Logic scripts. Examples:

```text
PartyLogic
InventoryLogic
SkillLogic
MissionLogic
PlayerDataLogic
```

`BattleSystem` receives prepared battle data and should not call DataStorage or
load permanent save data directly.

Static encounter and balance data should live under flat Config data:

```text
RootDesk/MyDesk/Data/Config/
```

Examples:

```text
encounterConfig
actorConfig
skillConfig
equipmentConfig
```

Skill icons stay in Config columns such as `skillConfig.iconRuid`; do not create
a separate RUID catalog.

## Encounter Key

`encounterKey` is a stable string identifier, not the config object itself.

Example values:

```text
Field_Slime_01
Training_Dummy_01
Boss_Mushmom_01
Quest_IntroBattle_01
```

Conceptually:

```text
encounterKey -> EncounterConfig -> enemy party, rewards, battle rules, UI/theme data
```

Why use `encounterKey`:

- The caller stays simple.
- Multiple field entities can reuse one encounter setup.
- Battle balancing stays in Config data.
- `BattleFlowLogic` validates missing or invalid encounters in one place.
- Save data can remember keys instead of duplicated static config.

`sponsorEntity` can still contribute runtime context, such as current level,
field position, or special state.

## BattleSystem Runtime Entity

Use `BattleSystem.model` as the runtime holder for one battle session.

When spawning it, pass a non-nil map entity as parent.

```text
SpawnByModelId("battlesystem", "BattleSystemSession", position, mapEntity)
```

The battle session entity should not be destroyed immediately after the battle
result is calculated. Send final client RPC first, then clean up after the client
has had a chance to receive the exit/end presentation event.

## Actor Data

Each battle actor should be represented by battle actor data and, when needed, a
`BattleActorComponent`.

Actor type examples:

```text
Player
Companion
Monster
Boss
```

For concrete actor ownership rules, see:

```text
docs/Actor/BattleActorComponent.md
```

`BattleSystem` should store actor ids and actor references, then apply state
changes through actor-facing interfaces instead of assigning HP/MP/stamina fields
directly.

## Open Questions

- Should battle happen in the same map, or transition to a battle scene?
- Where should `BattleSystem.model` be spawned when battle happens in a separate battle scene?
- Should `BattleSystem` send all presentation events as immediate RPC, or should some public state also use `@Sync`?
  - **Current answer:** `@Sync` for `battlePhase` and `currentActorId`; Client RPC for UI shell open/close and one-shot animations; client acknowledgement for action presentation completion.
- How much of action result presentation should be payload-driven versus inferred by `BattleClientLogic`?
- Should action resolution use MSW `AttackComponent` / `HitComponent`, or a pure turn-based calculation layer first?

