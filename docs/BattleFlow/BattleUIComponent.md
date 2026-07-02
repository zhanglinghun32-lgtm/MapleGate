# BattleUIComponent — Client HUD + Sync + Input

This document describes how battle UI opens, reads server-authoritative state,
and sends player actions under the current project architecture.

## Design Rules

```text
Open/Close BattleUI shell  -> BattleFlowLogic Client RPC -> BattleClientLogic -> UIManagerLogic
Battle phase / turn actor  -> BattleSystem @Sync (server writes, client reads)
Player skill tap           -> BattleUIComponent local selection + description popup
Target confirm             -> BattleUIComponent -> BattleSystem RequestAction (Client->Server RPC)
Battle session entity      -> Server spawn only (BattleFlowLogic); client never spawns it
```

Do **not** use `@Sync` to open or close `BattleUI`. Sync has latency and is meant for
ongoing display state, not one-shot presentation commands.

## Script Locations

| Script | Path | ExecSpace |
|--------|------|-----------|
| `BattleFlowLogic` | `Logic/BattleFlowLogic.mlua` | Server orchestration + Client RPC shell |
| `BattleClientLogic` | `Logic/BattleClientLogic.mlua` | ClientOnly presentation shell |
| `BattleSystem` | `Battle/BattleSystem.mlua` | ServerOnly authority + `@Sync` display fields |
| `BattleUIComponent` | `UI/battle/BattleUIComponent.mlua` | ClientOnly HUD on `ui/BattleUI.ui` |
| `UIManagerLogic` | `Logic/UIManagerLogic.mlua` | ClientOnly UI registry + Enable |

## Enter Battle Flow

```mermaid
sequenceDiagram
    participant BFL as BattleFlowLogic (Server)
    participant BS as BattleSystem (Server entity)
    participant BCL as BattleClientLogic (Client)
    participant UIM as UIManagerLogic (Client)
    participant BUI as BattleUIComponent (Client)

    BFL->>BS: StartBattle(payload)
    Note over BS: Sets @Sync battlePhase / currentActorId
    BFL->>BCL: EnterBattleClient(battleId, battleEntity, userId)
    Note over BFL: userId targets RPC; not in method signature
    BCL->>UIM: OpenBattleUI()
    BCL->>BUI: AttachBattleSession(battleEntity)
    Note over BUI: Reads BattleSystem snapshot
    BS-->>BUI: @Sync replicate -> BattleSystem.OnSyncProperty -> BattleClientLogic -> BattleUIComponent.OnSyncProperty
```

### Step details

1. **Server** — `BattleFlowLogic:StartBattle(payload, battleEntity)` calls
   `BattleSystem:StartBattle(payload)`. If that fails, **do not** send client RPC.
2. **Server** — `NotifyBattleEntered(userId, battleId, battleEntity)` calls
   `EnterBattleClient(battleId, battleEntity, userId)`.
3. **Client** — `BattleClientLogic:EnterBattle(battleId, battleEntity)`:
   - Opens `BattleUI` through `UIManagerLogic` (not via Sync).
   - Stores `battleEntity` for the HUD.
   - Calls `BattleUIComponent:AttachBattleSession(battleEntity)`.
4. **Client** — `BattleUIComponent` caches `BattleSystem` from the battle entity and
   applies the current `@Sync` snapshot.

## @Sync Fields (BattleSystem)

| Property | Writer | Reader | Purpose |
|----------|--------|--------|---------|
| `battlePhase` | Server (`TransitionTo`) | Client UI | Enable/disable input, show turn state |
| `currentActorId` | Server (`BeginNextTurn`) | Client UI | Know whose turn it is |

`BattleSystem` implements `OnSyncProperty` on the **client** and forwards
`battlePhase` / `currentActorId` to `BattleClientLogic:DispatchBattleSyncToUI`,
which calls `BattleUIComponent:OnSyncProperty`.

Reason: MSW invokes `OnSyncProperty` on the component that **owns** the `@Sync`
property. The HUD script lives on the UI entity, so it receives forwarded updates
instead of relying on Sync to open the UI.

## Player Input (RequestAction)

When the synced phase is `AwaitingInput` and `currentActorId` is set, clicking a
skill first updates local selection state and opens `BattleSkillDescriptionPopup`.
It does not send an action yet. After target selection and confirmation:

```text
BattleUIComponent button click
  -> BattleUIComponent:RequestBattleAction(actionKey, targetIds)
  -> BattleSystem:RequestAction(sponsorId, actionKey, targetIds)   [@ExecSpace("Server")]
  -> BattleSystem:RequestActionForUser(...)                         [ServerOnly validation]
```

For range skills, UI code can call
`RequestBattleActionAtWorldPosition(actionKey, targetPoint)` or
`RequestBattleActionAtScreenPosition(actionKey, screenPoint)`. The screen-point
variant converts through `_UILogic:ScreenToWorldPosition`; the server still owns
the final range expansion from `skillConfig`.

The server validates:

- battle is active
- phase is `AwaitingInput`
- `sponsorId == currentActorId`
- sender user matches `playerUserId`

## Exit Battle Flow

```text
BattleSystem:FinishBattle(result)
  -> BattleFlowLogic:HandleBattleFinished(userId, battleId, result)
  -> ExitBattleClient(battleId, result, userId)
  -> BattleClientLogic:ExitBattle
  -> UIManagerLogic:CloseBattleUI
  -> BattleUIComponent:DetachBattleSession
```

## Maker Setup Checklist

1. `refresh` after pulling script changes.
2. Confirm `BattleUI.ui` root has `script.BattleUIComponent` attached.
3. Confirm `UIRegistry` row for `BattleUI` exists.
4. Confirm battle session `.model` has both `BattleSystem` and `BattleQueue`.
5. Battle entity must be spawned on the **server** with a non-nil map parent.

## Skill Selection Flow

```text
Skill slot click
  -> BattleUIComponent:OnSkillSlotClicked(event, skillIndex)
  -> resolve skillKeysByIndex[skillIndex] + slot entity
  -> BattleUIComponent:SelectSkill(skillKey, slotEntity)
  -> cache selectedSkillKey + sponsor actorData
  -> BattleSkillDescriptionPopup:ShowSkill(skillKey, sponsorActorData, slotEntity)
  -> popup calculates selection overlay X/Y and enables it
```

`BattleSkillDescriptionPopup` owns its config lookup and close button. Its display
interface accepts the skill key, sponsor actorData, and selected slot entity; the
caller does not calculate popup/overlay position or pass preformatted UI strings.

## Skill Slot Binding

`BattleUIComponent` scans sequential `SkillSlot_N` children under `SkillGrid` and
connects every button to the shared `OnSkillSlotClicked(event, skillIndex)` handler.
The current UI keeps the legacy first-slot name `SkillSlot_01`; later slots use
`SkillSlot_2`, `SkillSlot_3`, and so on. Adding the next continuous slot therefore
does not require another property, handler field, or click method.

Use `SetSkillSlotKey(skillIndex, skillKey)` when another UI update path assigns a
skill to a slot. A slot is interactable whenever its skill key is populated, so
local browsing and the description popup still work before action submission is
legal. `inputEnabled` is enforced later by the RequestAction confirmation path.

## Current MVP Scope

- Skill slots use one loop-based binding and one indexed click handler.
- First skill slot (`SkillSlot_01`) currently maps to `normalAttack` and opens its description.
- Selection overlay is a transparent placeholder until the final animation entity exists.
- Target selection and confirmation are follow-up work; skill tap does not call RequestAction.
- HP bars, skill icons, and population of the remaining slot keys are follow-up tasks.

## Related Docs

- `docs/BattleFlow/BattleFlowLogic.md` — entry and finalization
- `docs/BattleFlow/BattleSystem.md` — session authority and action pipeline
