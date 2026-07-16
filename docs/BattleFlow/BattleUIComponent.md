# BattleUI — Turn Sequence Overlay

This document defines battle-time UI layering and the role of `BattleUI`.

Skill selection and action requests are **not** owned by `BattleUI`. Those live on
`ControlCharacterUI` and go through `SkillActionLogic`. See
`docs/SkillAction/SkillActionSystem.md`.

## Design Rules

```text
Persistent field UIs stay on screen during battle:
  ControlCharacterUI  -> skills / controlled actor
  Party UI            -> roster / quick switch (when opened by design)
  System UI           -> system menu (when opened by design)

Battle-only overlay:
  BattleUI            -> turn-based sequence presentation (turn order, phase cues, etc.)

Open/Close BattleUI overlay  -> BattleFlowLogic Client RPC -> BattleClientLogic -> UIManagerLogic
Battle phase / turn actor    -> BattleSystem @Sync (server writes, client reads)
Player skill confirm         -> ControlCharacterUI -> SkillActionLogic:RequestSkill
Battle session entity        -> Server spawn only (BattleFlowLogic); client never spawns it
```

Do **not** use `@Sync` to open or close `BattleUI`. Sync has latency and is meant for
ongoing display state, not one-shot shell commands.

Do **not** route skill / action requests from `BattleUI` into
`BattleSystem:RequestAction*`. That path is legacy and must not be used by UI.

## UI Layering During Battle

```text
[ BattleUI ]                 <- battle-only overlay (turn sequence)
[ ControlCharacterUI ]       <- stays available
[ Party UI / System UI ]     <- may stay available per product rules
[ World / actors ]
```

Rules:

- Entering battle **adds** `BattleUI` on top. It does not replace the persistent
  gameplay UIs.
- Leaving battle **closes** `BattleUI` only. It does not require closing
  `ControlCharacterUI`, Party UI, or System UI.
- `ControlCharacterUI` remains the skill browser and request sender in both field
  and battle.

## Responsibility Split

| Script / UI | Owns | Does not own |
|-------------|------|--------------|
| `BattleUI` / `BattleUIComponent` | Turn sequence display, battle-phase cues driven by sync or forwarded events | Skill slots, `RequestSkill`, formation edits, system menu |
| `ControlCharacterUIComponent` | Skill selection, confirm, description popup, controlled-actor skill list | Turn order board ownership, opening battle shell |
| `SkillActionLogic` | Public skill request gateway for field and battle | UI widget binding |
| `BattleSystem` | Phase, current actor, battle-only validation, resolution, presentation ack | UI open/close, skill button binding |
| `BattleClientLogic` | Enter/exit shell: camera, attach session refs, open/close `BattleUI` overlay | Damage calculation, legal target decisions |

## Script Locations

| Script | Path | ExecSpace |
|--------|------|-----------|
| `BattleFlowLogic` | `Logic/Battle/BattleFlowLogic.mlua` | Server orchestration + Client RPC shell |
| `BattleClientLogic` | `Logic/Battle/BattleClientLogic.mlua` | ClientOnly presentation shell |
| `BattleSystem` | `Battle/BattleSystem.mlua` | Server authority + `@Sync` display fields |
| `BattleUIComponent` | `UI/battle/BattleUIComponent.mlua` | ClientOnly turn-sequence overlay on `ui/BattleUI.ui` |
| `ControlCharacterUIComponent` | `UI/battle/ControlCharacterUIComponent.mlua` | ClientOnly persistent control HUD on `ui/ControlCharacterUI.ui` |
| `SkillActionLogic` | `Logic/Skill/SkillActionLogic.mlua` | Server request gateway + Client result callbacks |
| `UIManagerLogic` | `Logic/UI/UIManagerLogic.mlua` | ClientOnly UI registry + Enable |

## Enter Battle Flow

```mermaid
sequenceDiagram
    participant BFL as BattleFlowLogic (Server)
    participant BS as BattleSystem (Server entity)
    participant BCL as BattleClientLogic (Client)
    participant UIM as UIManagerLogic (Client)
    participant CC as ControlCharacterUI (Client)
    participant BUI as BattleUI (Client)

    BFL->>BS: StartBattle(payload)
    Note over BS: Sets @Sync battlePhase / currentActorId
    BFL->>BCL: EnterBattleClient(battleId, battleEntity, userId)
    Note over BFL: userId targets RPC; not in method signature
    Note over CC: Already open from World; stays on screen
    BCL->>UIM: OpenBattleUI()
    BCL->>CC: AttachBattleSession(battleEntity)
    BCL->>BUI: AttachBattleSession(battleEntity)
    Note over BUI: Turn-sequence overlay only
    BS-->>CC: @Sync via BattleSystem.OnSyncProperty -> BattleClientLogic
    BS-->>BUI: same forward path for sequence display
```

### Step details

1. **Server** — `BattleFlowLogic:StartBattle(payload, battleEntity)` calls
   `BattleSystem:StartBattle(payload)`. If that fails, **do not** send client RPC.
2. **Server** — `NotifyBattleEntered(userId, battleId, battleEntity)` calls
   `EnterBattleClient(battleId, battleEntity, userId)`.
3. **Client** — `BattleClientLogic:EnterBattle(battleId, battleEntity)`:
   - Keeps `ControlCharacterUI` available (open if needed, then attach session).
   - Opens `BattleUI` overlay through `UIManagerLogic` (not via Sync).
   - Stores `battleEntity` for sync / presentation.
   - Attaches battle session refs to the UIs that need phase / actor display.
4. **Client** — Turn-sequence UI and control UI apply the current `@Sync` snapshot.

## @Sync Fields (BattleSystem)

| Property | Writer | Reader | Purpose |
|----------|--------|--------|---------|
| `battlePhase` | Server (`TransitionTo`) | Client UIs | Input gating, sequence / phase cues |
| `currentActorId` | Server (`BeginNextTurn`) | Client UIs | Whose turn / who may act |

`BattleSystem` implements `OnSyncProperty` on the **client** and forwards
`battlePhase` / `currentActorId` through `BattleClientLogic:DispatchBattleSyncToUI`.

Consumers:

- `ControlCharacterUIComponent` — input enable / controlled-actor presentation
- `BattleUIComponent` — turn-sequence display

Reason: MSW invokes `OnSyncProperty` on the component that **owns** the `@Sync`
property. UI scripts live on other entities, so they receive forwarded updates.

## Player Skill Input (SkillActionLogic)

Skill confirm belongs to `ControlCharacterUI`, not `BattleUI`.

```text
ControlCharacterUI skill confirm
  -> ControlCharacterUIComponent:RequestSkillAction(actionKey, actionContext)
  -> SkillActionLogic:RequestSkill(requestId, actionKey, actionContext)
  -> common validation (SkillLogic)
  -> if active battle: BattleSystem:RequestValidatedActionForUser(...)
  -> else: field execution path
  -> SkillActionLogic Accept/Reject client callback
```

Range helpers on the control UI may fill `TargetPoint` / `HasTargetPoint` in
`actionContext`. The server still owns final range expansion from `skillConfig`.

Battle-only server checks (inside `BattleSystem` after the gateway):

- battle is active
- phase is `AwaitingInput`
- sponsor is the current actor
- sender user matches `playerUserId`
- target rule / range resolution

UI must **not** call:

```text
BattleSystem:RequestAction
BattleSystem:RequestActionAtPosition
BattleSystem:RequestActionWithContext
```

Those APIs may remain temporarily as legacy server entry points, but new UI work
uses `SkillActionLogic` only.

## Exit Battle Flow

```text
BattleSystem:FinishBattle(result)
  -> BattleFlowLogic:HandleBattleFinished(userId, battleId, result)
  -> ExitBattleClient(battleId, result, userId)
  -> BattleClientLogic:ExitBattle
  -> UIManagerLogic:CloseBattleUI          (remove turn-sequence overlay)
  -> Detach battle session from ControlCharacterUI / BattleUI
  -> ControlCharacterUI / Party / System UI remain per field rules
```

## Maker Setup Checklist

1. `refresh` after pulling script changes.
2. Confirm `BattleUI.ui` root has `script.BattleUIComponent` attached.
3. Confirm `UIRegistry` row for `BattleUI` exists.
4. Confirm `ControlCharacterUI.ui` root has `script.ControlCharacterUIComponent`.
5. Confirm battle session `.model` has both `BattleSystem` and `BattleQueue`.
6. Battle entity must be spawned on the **server** with a non-nil map parent.

## BattleUI Scope

`BattleUI` should focus on turn-based sequence presentation, for example:

- turn order / queue strip
- current-phase cues
- battle-only banners or sequence markers

Out of scope for `BattleUI`:

- skill hotbar and confirm
- party formation editing
- system menu actions
- direct `BattleSystem` action RPCs

## Related Docs

- `docs/BattleFlow/BattleFlowLogic.md` — entry and finalization
- `docs/BattleFlow/BattleSystem.md` — session authority and action pipeline
- `docs/SkillAction/SkillActionSystem.md` — skill request gateway
- `docs/Party/PartySystem.md` — party UI and field control
