# Party System

This document defines party membership, field control, formation editing, and
battle-entry rules. It also records how `PartyLogic` interacts with existing
world, battle, save, and UI systems.

Read `docs/ScriptRules.md` first for the project workflow:

```text
Design doc -> Config -> Logic -> Component
```

## Purpose

The party system answers:

```text
Who is in the player's party?
In what order do they fight?
Which party member currently moves on the overworld?
When battle starts, does the whole party enter together?
```

It does **not** answer turn order inside battle, damage calculation, inventory
rules, or UI shell open/close policy.

## Core Rules

These rules are fixed for the current design.

| Rule | Description |
|------|-------------|
| Party size cap | A party has at most **4** members. |
| Whole-party battle entry | Field encounters always start battle for the **entire current formation**. A single member cannot enter battle alone while others stay outside. |
| Single field mover | On the overworld, only **one** party member receives player movement input at a time. |
| Switch anytime | The player may switch the active field member anytime **outside** blocked states (battle, cutscene, dialogue, etc.). |
| Formation editing | The player may change party composition/order while **not** blocked. **Blocked during battle.** |
| Battle UI policy | Battle does **not** own Party UI open/close. Opening Party UI in battle may be allowed; mutating formation / switching field member must be denied via battle operation locks. |
| Server authority | Membership, formation order, active field slot, and battle snapshot are server-owned. |
| Save on flush | Persistent party data is written only through `PlayerDataLogic` save/flush. |

## Responsibility Matrix

| System | Role in party feature |
|--------|----------------------|
| `PartyLogic` | Server authority for membership, formation, active field slot, validation, save export/import, battle snapshot |
| `PartyClientLogic` | Client coordinator for field avatar switching, companion presentation, and party UI refresh triggers |
| `PartyUIComponent` | Party roster / formation editing UI; sends change requests to server |
| `PlayerDataLogic` | Loads and flushes `slotData.Party`; does not own party mutation rules |
| `PlayerControlLogic` | Locks local `PlayerControllerComponent` during battle and other blocked states |
| `BattleFlowLogic` | Reads party snapshot from `PartyLogic` and builds full `playerParty` payload |
| `BattleSystem` | Turn permission / settlement only; supplies operation locks (`EditPartyFormation`, `SwitchFieldMember` blocked). Does not open Party UI. |
| `SkillActionLogic` | Skill gateway; uses battle permission when a session is active |
| `ControlCharacterUIComponent` | Shows skills for the controlled actor; does not own party membership |
| `BattleActorCom` | Owns one actor's HP / MP / stats; one instance per battle-capable actor |
| `UIManagerLogic` | Opens/closes `Party` UI shell by key |

## PartyLogic

Path:

```text
RootDesk/MyDesk/Logic/Party/PartyLogic.mlua
```

`PartyLogic` is the single server authority for party state.

### Responsibilities

- Store per-user runtime party data in `userParties[userId]`.
- Enforce the **4-member cap**.
- Own `Formation`: ordered actor references used for battle turn registration.
- Own `ActiveFieldSlot`: which formation slot currently controls overworld movement.
- Validate party edits:
  - actor exists in the user's actor roster
  - actor is not duplicated in formation
  - slot index is within `1..4`
  - edit is allowed in the current world state
- Export and import `slotData.Party` through `ExportUserData()` / `LoadUserData()`.
- Provide `GetBattlePartySnapshot(userId)` for `BattleFlowLogic`.
- Provide `GetActiveFieldActorKey(userId)` / `GetActiveFieldSlot(userId)` for field systems.
- Emit client notifications after successful party mutations.

### Non-Responsibilities

- Do not bind UI buttons or read `.ui` UUIDs.
- Do not move entities directly.
- Do not enable or disable `PlayerControllerComponent`.
- Do not start or resolve battle turns.
- Do not write `UserDataStorage` directly.
- Do not duplicate persistent actor progression in `Party.Members`.

### Recommended Public API

```text
LoadUserData(userId, partyData)
ExportUserData(userId)
CreateDefaultData()

GetMemberCount(userId)
GetFormation(userId)
GetActiveFieldSlot(userId)
GetActiveFieldActorKey(userId)
GetBattlePartySnapshot(userId)

RequestSetFormation(userId, formationActorKeys)
RequestAddMember(userId, actorKey, slotIndex)
RequestRemoveMember(userId, slotIndex)
RequestSwapMembers(userId, slotA, slotB)
RequestSwitchActiveFieldSlot(userId, slotIndex)
CanEditParty(userId)
CanSwitchFieldMember(userId)
```

`Request*` methods should be `@ExecSpace("Server")` entry points. Internal
validation stays `ServerOnly`.

### Battle Snapshot Shape

`GetBattlePartySnapshot(userId)` returns one prepared actor entry per formation
slot, in formation order. Empty slots are omitted.

Suggested entry:

```lua
{
    actorId = "Party_{userId}_{slotIndex}",
    actorType = "Player" | "Companion",
    actorKey = "",
    slotIndex = 1,
    entityId = "",          -- field entity id when spawned; may be empty before field spawn exists
    configId = "",
    level = 1,
    hp = 0,
    maxHp = 0,
    mp = 0,
    maxMp = 0,
    stamina = 0,
    maxStamina = 0,
    totalAttack = 0,
    totalDefense = 0,
    speed = 0,
    skillKeys = {}
}
```

`BattleFlowLogic` converts this snapshot into `payload.playerParty` without
reading save data directly.

## PartyClientLogic

Path:

```text
RootDesk/MyDesk/Logic/Party/PartyClientLogic.mlua
```

`PartyClientLogic` is the client-only coordinator for field presentation.

### Responsibilities

- Apply server-approved active field member switches on the local client.
- Decide which party entity receives `PlayerControllerComponent` input.
- Refresh `ControlCharacterUIComponent` controlled actor and skill list after a switch.
- Update companion field presentation:
  - follow active leader
  - hide/show non-active members
  - play switch presentation if needed
- Forward party-state changes to `PartyUIComponent`.
- Rebuild field party entities after load, save continue, or map enter when required.

### Non-Responsibilities

- Do not change authoritative membership or formation.
- Do not start battle.
- Do not calculate damage or turn order.
- Do not save party data.

### Recommended Public API

```text
OnPartyStateChanged(userId, partyRevision)
ApplyActiveFieldSlot(slotIndex)
RefreshPartyPresentation()
NotifyPartyUI()
```

Server should call a `@ExecSpace("Client")` RPC on `PartyLogic` or
`PartyClientLogic` after successful party mutation.

## PartyUIComponent

Path:

```text
RootDesk/MyDesk/UI/party/PartyUIComponent.mlua
```

UI root:

```text
ui/PartyUI.ui
```

Register the UI key in `UIRegistry` as `"Party"`.

### Responsibilities

- Display current party members and formation slots `1..4`.
- Let the player rearrange formation order.
- Let the player add/remove members from the owned actor roster.
- Let the player switch the active field member from the party screen.
- Disable editing while `PartyLogic:CanEditParty()` is false.
- Send `RequestSetFormation`, `RequestAddMember`, `RequestRemoveMember`,
  `RequestSwapMembers`, and `RequestSwitchActiveFieldSlot` to `PartyLogic`.

### Non-Responsibilities

- Do not store authoritative party data.
- Do not open or close itself directly; use `UIManagerLogic`.
- Do not spawn field entities.
- Do not build battle payloads.

### Relation To ControlCharacterUI

`ui/ControlCharacterUI.ui` already contains `PartyMemberList`. That area is for
**quick field switching and compact roster display** during gameplay.

Recommended split:

| UI | Purpose |
|----|---------|
| `PartyUI` | Full formation editing, add/remove, detailed roster management |
| `ControlCharacterUI.PartyMemberList` | Quick active-member switch and compact status while exploring |

Both UIs call `PartyLogic` requests. Neither owns party authority.

## PlayerDataLogic

Path:

```text
RootDesk/MyDesk/Logic/Actor/PlayerDataLogic.mlua
```

### Responsibilities

- On load: call `_PartyLogic:LoadUserData(userId, slotData.Party)`.
- On flush: write `slotData.Party = _PartyLogic:ExportUserData(userId)`.
- Continue to own `slotData.Actors` as persistent actor records.

### Non-Responsibilities

- Do not validate formation size or battle entry rules.
- Do not decide which actor is currently controlled in the field.

## Data Ownership

Canonical save schema lives in `docs/PlayerData/SaveSlotSchema.md`.

### Recommended Source Of Truth

| Information | Owner |
|-------------|-------|
| Persistent actor identity, level, HP, MP, stamina, base stats | `slotData.Actors` via `BattleActorCom` / `PlayerDataLogic` |
| Party membership references | `slotData.Party.Formation` via `PartyLogic` |
| Active field slot | `slotData.Party.ActiveFieldSlot` via `PartyLogic` |
| Derived battle totals | runtime only on `BattleActorCom` |

### Save Shape Update

Add this field to `slotData.Party`:

| Exact key | Type | Default | Notes |
|-----------|------|---------|-------|
| `ActiveFieldSlot` | integer | `1` | Formation slot that currently controls overworld movement. |

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

## BattleFlowLogic

Path:

```text
RootDesk/MyDesk/Logic/Battle/BattleFlowLogic.mlua
```

### Responsibilities

- Before `StartBattle(payload)`, call `_PartyLogic:GetBattlePartySnapshot(userId)`.
- Put the full snapshot into `payload.playerParty`.
- Reject battle entry when the party has zero valid battle actors.
- Keep encounter resolution, reward application, and client shell RPC ownership.

### Non-Responsibilities

- Do not store formation order long-term.
- Do not let the triggering entity alone become the only `playerParty` member.

### Required Behavior Change

Current field battle entry builds `playerParty` from only the local player's
`BattleActorCom`. That is a temporary MVP path.

Target behavior:

```text
any trigger adapter
  -> BattleFlowLogic:RequestStartBattle(request)
  -> expand PlayerParty / EnemyParty / AllyParty / SharedInventory / Environment
  -> BattleSystem:StartBattle(payload)
```

See field tables in `docs/BattleFlow/BattleFlowLogic.md` (§ BattleStartRequest /
§ BattleStartPayload). Neutrals are never included unless they become hostile.

Important rule:

```text
The entity that triggered the encounter may be the active field avatar,
but battle entry always includes the whole party.
```

## BattleSystem

Path:

```text
RootDesk/MyDesk/Battle/BattleSystem.mlua
```

### Responsibilities

- Consume `payload.playerParty` as the player-side actor list.
- Register every supplied party actor into turn order / actor map.
- Apply HP, MP, stamina, and death state through `BattleActorCom` interfaces.

### Non-Responsibilities

- Do not decide which actors belong to the party.
- Do not load `slotData.Party` or `slotData.Actors` directly.

## SkillActionLogic And Field Control

### Responsibilities

- In overworld, treat the **active field party member** as the skill sponsor.
- Resolve sponsor entity through `PartyLogic` + `PartyClientLogic`, not by
  assuming `DefaultPlayer` is always the sponsor.
- Keep battle validation unchanged: sponsor must be the current battle actor.

### Non-Responsibilities

- Do not edit party formation.
- Do not switch field control by itself.

## PlayerControlLogic

### Responsibilities

- Continue to own control-lock stacking (`Battle`, `Cutscene`, `Dialogue`, etc.).
- Party switching must be blocked while any lock that forbids field control is active.

### Non-Responsibilities

- Do not store active field slot.
- Do not decide party membership.

Recommended new lock consumers:

```text
Battle      -> already exists
Cutscene    -> future
Dialogue    -> future
PartyEdit   -> optional if formation modal needs a separate lock
```

## Field Runtime Model

### Single Active Mover

Only one party member is the controlled field avatar at a time.

```text
PartyLogic.ActiveFieldSlot = 2
  -> PartyClientLogic enables PlayerController on slot-2 entity
  -> other party entities do not receive movement input
  -> ControlCharacterUI shows slot-2 actor skills and resources
```

### Companion Presentation

Non-active members should still exist in the overworld as party followers or
hidden companions. Presentation is client-side; authority remains server-side.

Recommended first version:

```text
Active member   -> full avatar, PlayerController enabled
Other members   -> follow active member at fixed offsets, controller disabled
```

Alternative later version:

```text
Only active member visible in field
Other members appear only in battle
```

Pick one presentation mode in implementation; do not split authority across
both.

### Switch Flow

```text
Player taps member in ControlCharacterUI or PartyUI
  -> PartyLogic:RequestSwitchActiveFieldSlot(userId, slotIndex)
  -> server validates CanSwitchFieldMember
  -> server updates ActiveFieldSlot
  -> PartyLogic sends client RPC
  -> PartyClientLogic applies controller swap and presentation
  -> ControlCharacterUIComponent updates controlled actor and skill list
```

## Battle Entry Flow

All field/story starts go through one Flow API (see
`docs/BattleFlow/BattleFlowLogic.md` § Battle Entry Triggers):

```text
Dialogue | PlayerAttack | MonsterDetect | other
  -> BattleFlowLogic:RequestStartBattle(request)
  -> PartyLogic:GetBattlePartySnapshot(userId)
  -> payload.playerParty = full formation
  -> BattleSystem:StartBattle(payload)
```

```text
1. Any trigger adapter builds a BattleStartRequest (schema TBD)
2. BattleFlowLogic validates playerUserId and encounter context
3. BattleFlowLogic reads PartyLogic:GetBattlePartySnapshot(userId)
4. BattleFlowLogic builds payload.playerParty with all formation members
5. BattleFlowLogic starts BattleSystem
6. BattleClientLogic locks field control through PlayerControlLogic as needed
7. Every party member enters the battle session together
8. BattleSystem resolves turns among all registered player-side actors
9. On battle end, BattleFlowLogic writes results back to actor owners and exits
```

Important rule:

```text
The entity that triggered the encounter may be the active field avatar or a
detecting monster, but battle entry always includes the whole party on the
player side.
```

### Battle Exit Rule

When battle ends, actor state changes must map back to the correct persistent
actor record in `Actors[]`. `PartyLogic` only keeps references and slot order.

## UI Open Flow

```text
Gameplay UI button / menu action
  -> UIManagerLogic:OpenUI("Party")
  -> PartyUIComponent:RefreshFromPartyLogic()
  -> player edits formation
  -> PartyUIComponent sends Request* calls
  -> PartyLogic validates and mutates server state
  -> PartyLogic notifies PartyClientLogic and PartyUIComponent
```

Party UI should be unavailable during battle.

## Config

No new Config table is required for the first party version.

Party composition references existing actor keys from:

```text
RootDesk/MyDesk/Data/Config/actorConfig.csv
```

Future recruit rules may add:

```text
partyUnlockConfig
companionJoinConfig
```

## Implementation Order

Build in this order:

```text
1. Finalize save shape: Party.Formation + Party.ActiveFieldSlot
2. Expand PartyLogic validation and Request* APIs
3. Add PartyClientLogic field avatar switching
4. Wire BattleFlowLogic to GetBattlePartySnapshot()
5. Add PartyUIComponent for full formation editing
6. Wire ControlCharacterUI.PartyMemberList for quick switching
7. Battle operation locks: CanEditParty / CanSwitchFieldMember false while battle active
8. Verify save/load, continue game, field switch, and whole-party battle entry
```

### Agent TODO — Battle Interaction (Party Must Enforce, Battle Must Not Own UI)

1. **`PartyLogic.mlua`**
   - TODO: `CanEditParty` / `CanSwitchFieldMember` return false when
     `BattleFlowLogic` / `BattleSystem` reports an active battle for that user.
   - TODO: `RequestSetFormation` / `RequestSwitchActiveFieldSlot` reject with a
     clear reason while battle-locked.

2. **`PartyUI` / `ControlCharacterUI` PartyMemberList**
   - TODO: Opening roster may stay allowed; disable edit/switch controls in battle.
   - TODO: Do not implement this disable logic inside `BattleUIComponent`.

3. **`docs/BattleFlow/BattleSystem.md`**
   - Operation keys: `EditPartyFormation`, `SwitchFieldMember` default blocked.

## Verification Checklist

- New game creates a one-member party in slot 1.
- Party cannot exceed 4 members.
- Switching active field member changes controlled avatar and skill sponsor.
- Field encounter enters battle with all formation members, not only the active avatar.
- Party edits are blocked during battle.
- Save and continue restore `Formation` and `ActiveFieldSlot`.
- `Actors[]` remains the only persistent owner of actor stats.

## Open Questions

- Should non-active field members follow the leader immediately, or stay at last anchored positions until battle?
- Should dead actors remain in formation but be skipped in battle entry, or block battle start entirely?
- Do companions each need their own field entity at load, or are they spawned on demand from `Actors[]`?
- Should `ActiveFieldSlot` reset to `1` after battle, or remember the pre-battle slot?
- Does `PartyUI` allow empty slots in formation, or must members occupy the lowest slots first?

## Related Docs

- `docs/ScriptRules.md`
- `docs/PlayerData/SaveSlotSchema.md`
- `docs/BattleFlow/BattleFlowLogic.md`
- `docs/BattleFlow/BattleSystem.md`
- `docs/Actor/BattleActorComponent.md`
- `docs/PlayerControl/PlayerControlLogic.md`
- `docs/SkillAction/SkillActionSystem.md`
