# BattleUI — Turn Sequence Overlay

`BattleUI` is the **battle-only turn-sequence overlay**. It is not the owner of
skills, items, party edits, or system/navigation menus.

## Hard Boundary

```text
BattleUI shows:     turn order, phase cues, battle-only sequence chrome
BattleUI never:     casts skills, uses items, opens Party/System/nav as its job
BattleSystem locks: whether those other UIs may *complete* certain operations
HP / resource bars: always bind entity BattleActorCom (hp/maxHp), never payload copies
```

Persistent UIs stay on screen during battle:

| UI | Role in battle |
|----|----------------|
| `ControlCharacterUI` | Skill select / confirm (shared field+battle control HUD) |
| Party UI | May open; **formation / field-member switch blocked** by battle policy |
| System UI | May stay; individual actions follow product policy |
| `BattleUI` | Extra overlay for turn sequence |

```text
[ BattleUI ]              <- overlay: turn sequence only
[ ControlCharacterUI ]
[ Party / System UI ]
[ World ]
```

Enter battle **adds** `BattleUI`. Exit battle **removes** `BattleUI` only.

## Design Rules

```text
Open/Close BattleUI     -> BattleFlowLogic Client RPC -> BattleClientLogic -> UIManagerLogic
Phase / current actor   -> BattleSystem @Sync (forwarded; not used to open UI)
Skill / item confirm    -> ControlCharacterUI or Item UI -> SkillAction / Item gateway
Party / System open     -> their own owners via UIManagerLogic
Blocked ops in battle   -> PartyLogic / gateways query BattleSystem policy
```

Do **not** call `BattleSystem:RequestAction*` from any UI.

## Responsibility Split

| Owner | Owns | Does not own |
|-------|------|--------------|
| `BattleUIComponent` | Turn-sequence display from phase/actor sync | Skill slots, items, party edit, system menu |
| `ControlCharacterUIComponent` | Skill UI + `RequestSkill` | Turn board, battle open/close |
| `SkillActionLogic` | Skill request gateway | Turn scheduling UI |
| `BattleSystem` | Turn permission, start/end, settlement, **operation locks** | Opening UIs, skill widgets |
| `BattleClientLogic` | Overlay open/close, sync forward, camera | Skill/item casting |
| `PartyLogic` / Party UI | Roster UI; enforce blocked formation in battle | Turn order |
| System menu | System actions; enforce its own battle policy | Battle turn loop |

## Script / UI Locations

| Asset | Path |
|-------|------|
| Overlay UI | `ui/BattleUI.ui` |
| Overlay script | `RootDesk/MyDesk/UI/battle/BattleUIComponent.mlua` |
| Control HUD UI | `ui/ControlCharacterUI.ui` |
| Control HUD script | `RootDesk/MyDesk/UI/battle/ControlCharacterUIComponent.mlua` |
| Session | `RootDesk/MyDesk/Battle/BattleSystem.mlua` |
| Client shell | `RootDesk/MyDesk/Logic/Battle/BattleClientLogic.mlua` |

## Enter / Exit

```text
EnterBattleClient
  -> keep ControlCharacterUI / Party / System available
  -> OpenBattleUI + AttachBattleSession on sequence + control UIs as needed

ExitBattleClient
  -> CloseBattleUI
  -> Detach battle session refs
  -> leave persistent UIs alone
```

## Sync Consumers

| Field | BattleUI | ControlCharacterUI |
|-------|----------|--------------------|
| `battlePhase` | Sequence / phase cues | Enable/disable skill confirm chrome |
| `currentActorId` | Highlight whose turn | Gate confirm to current actor display |

Server still rejects illegal submits even if UI is wrong.

## Agent TODO — UI And Scripts

1. **`ui/BattleUI.ui`**
   - TODO: Author turn-order strip / phase markers only.
   - TODO: Remove or relocate skill grids, item buttons, party editors if they
     still live under this file from older layouts.

2. **`BattleUIComponent.mlua`**
   - TODO: Implement sequence refresh from forwarded `OnSyncProperty` only.
   - TODO: Delete skill select / `RequestBattleAction*` / description-popup
     ownership if still in this script (move leftovers to ControlCharacterUI
     or delete dead code).
   - TODO: No `BattleSystem:RequestAction*` calls.

3. **`BattleClientLogic.mlua` + `UIManagerLogic.mlua`**
   - TODO: `OpenBattleUI` / `CloseBattleUI` on enter/exit.
   - TODO: Register `BattleUI` in `UIRegistry` if missing.
   - TODO: Do not close ControlCharacterUI or SystemMenu when closing BattleUI.

4. **`ControlCharacterUI.ui` + `ControlCharacterUIComponent.mlua`**
   - TODO: Remain the skill (and later item) interaction surface in battle.
   - TODO: Disable confirm when battle policy denies `SubmitSkill`.
   - TODO: PartyMemberList: allow view; block switch/edit while battle active.

5. **Party / System UIs**
   - TODO: Opening allowed; `EditPartyFormation` / `SwitchFieldMember` blocked
     during battle (server + disabled controls).
   - TODO: Do not implement those blocks inside `BattleUIComponent`.

6. **Verification grep (for implementing agents)**
   - TODO: `RequestAction` / `RequestBattleAction` must not appear in UI scripts
     as the live path; only SkillAction / future item gateway.

## Related Docs

- `docs/BattleFlow/BattleSystem.md` — locks + settlement
- `docs/BattleFlow/BattleFlowLogic.md` — enter/exit
- `docs/SkillAction/SkillActionSystem.md`
- `docs/Party/PartySystem.md`
