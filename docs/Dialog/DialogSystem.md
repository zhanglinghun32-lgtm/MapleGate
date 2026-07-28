# Dialog System

This document describes the current NPC dialog pipeline, the four Config tables,
their join rules, and the remaining implementation work.

## Scope

The dialog system currently owns:

- starting a dialog from a clicked world entity;
- choosing an entry node from player conditions;
- traversing text, option, and confirmation nodes;
- filtering choices and nodes by inventory or mission state;
- executing mission and shop actions on the server;
- displaying localized text and the actor's Config-defined thumbnail appearance in `Dialog.ui`.

Main files:

| Responsibility | File |
|---|---|
| Server authority, Config cache, session and traversal | `RootDesk/MyDesk/Logic/Dialog/DialogLogic.mlua` |
| World NPC click entry point | `RootDesk/MyDesk/NPC/NPCActorCom.mlua` |
| Client display and button callbacks | `RootDesk/MyDesk/UI/dialog/npcDialogUICom.mlua` |
| Shared actor thumbnail lookup | `RootDesk/MyDesk/Logic/ActorAppearanceLogic.mlua` |
| UI entity | `ui/Dialog.ui` |
| Config tables | `RootDesk/MyDesk/Data/Config/npcDialog/` |
| Localized strings | `RootDesk/MyDesk/Data/Localization/GameText.csv` |

## Current Flow

```text
NPCActorCom.OnActorClick()
    -> DialogLogic.RequestStartDialog(actorId)
    -> npcDialogEntry: choose first eligible entry by priority
    -> npcDialogNode: load node
    -> npcDialogChoice: filter and sort visible choices
    -> DialogLogic.OpenDialogClient(context)
    -> npcDialogUICom: localize text + query npcOutfit by actorId
    -> player presses Next / Option / Accept / Reject / Close
    -> DialogLogic.RequestResolveDialog(...)
    -> execute action
    -> jump to next node or close
```

`DialogLogic` keeps one in-memory session per user. The session stores the original
`actorId`, current `dialogKey`, and the server-filtered visible
choices. The client submits only an action and option index; the server resolves the
actual choice from its session.

## Implementation Status

### Completed

- [x] Cache all four Config tables once in `DialogLogic.OnBeginPlay`.
- [x] External start API accepts only `actorId`; world entities are not passed into Logic.
- [x] Choose an NPC entry by ascending `priority`.
- [x] Check both entry and target-node conditions.
- [x] Support `Text`, `Options`, and UI-side `Confirm` node presentation.
- [x] Sort choices by `displayOrder`.
- [x] Filter choices by choice condition and destination-node condition.
- [x] Limit visible choices to five, matching the five buttons in `Dialog.ui`.
- [x] Server-authoritative choice-index resolution.
- [x] Support node and choice actions:
  `Close`, `AcceptMission:<missionKey>`,
  `CompleteMission:<missionKey>`, and `OpenShop:<shopKey>`.
- [x] Support condition types:
  `AlwaysPass`, `ItemCount`, `MissionAccepted`, and `MissionCompleted`.
- [x] Support AND clauses inside one `groupKey` and OR between different groups.
- [x] Resolve dialog and option localization keys on the client.
- [x] Resolve dialog thumbnails through `npcOutfit` by `actorId`.
- [x] Use `type=Avatar` for CostumeManager slots and `type=Sprite` for `spriteRuid`.
- [x] Keep runtime model appearance authoritative; `npcOutfit` is display-only Config.
- [x] Close the UI and remove the server session on normal dialog completion.

### TODO / Blank Areas

- [ ] Validate interaction distance and that the clicked entity is still in the
  player's current map before starting and resolving a dialog.
- [ ] Add request throttling and duplicate-click protection.
- [ ] Clear sessions when a user disconnects, changes map, the talker is destroyed,
  or another modal flow interrupts the dialog.
- [ ] Define whether starting a second dialog replaces or rejects an active session.
- [ ] Add a Config validation pass for duplicate keys, missing node/condition links,
  unsupported values, dead nodes, and accidental infinite cycles.
- [ ] Define behavior when an `Options` or `Confirm` node has no eligible choices.
- [ ] Add actual `Confirm` rows and document the required two-choice convention:
  visible choice 1 = Accept, visible choice 2 = Reject.
- [ ] Decide whether choices beyond five should be rejected by validation or rendered
  through a dynamic scroll list.
- [ ] Add localization formatting arguments through
  `_LocalizationService:GetTextFormat`.
- [ ] Separate `speakerKey` identity from speaker display-name localization if actor
  IDs should not be used directly as text keys.
- [ ] Add more condition types when needed, such as actor ownership, active job,
  level, currency, map state, flags, and compound numeric comparisons.
- [ ] Replace the hard-coded action parser with a documented action registry if the
  action list grows.
- [ ] Handle action failure explicitly. Mission or shop failure should be able to
  keep the dialog open and route to an error node instead of continuing blindly.
- [ ] Support mission choose-one rewards. Dialog resolves the authoritative
  choice row, passes only `rewardGroupKey` / `optionKey` to `MissionLogic`, and
  advances only after turn-in succeeds; see `docs/Mission/MissionSystem.md`.
- [ ] Decide whether any one-time dialog decisions or conversation progress belong
  in `PlayerDataLogic`; current dialog sessions and traversal progress are not saved.
- [ ] Replace fixed UI entity/component UUID defaults in `npcDialogUICom` with Maker
  scene bindings, following the project clone/binding rule.
- [ ] Add click/hover sound effects and accessibility/keyboard navigation.
- [ ] Add automated traversal tests covering every entry, condition group, choice,
  action, and destination node.

## Config Tables

Every table is a UserDataSet pair:

```text
TableName.csv          <- row data
TableName.userdataset  <- Maker metadata and runtime table name
```

Edit rows in Maker's Data Editor or edit the UTF-8 CSV, then stop Play, Refresh the
workspace, and start Play again. Runtime lookup uses the UserDataSet `name`, which
currently matches the names below.

### `npcOutfit`

Purpose: display-only actor thumbnails for dialog, shop, and similar UI. Runtime
world entities keep the appearance from their `.model`; this table never overwrites
the world entity.

Runtime table name: `npcOutfit`

- `actorId`: lookup key shared with dialog and shop actor identity.
- `type`: `Avatar` or `Sprite`.
- `spriteRuid`: required only for `Sprite`.
- Avatar slot columns: `body`, `cap`, `cape`, `coat`, `earAccessory`,
  `eyeAccessory`, `faceAccessory`, `face`, `glove`, `hair`, `longcoat`,
  `oneHandWeapon`, `pants`, `shoes`, `subWeapon`, `twoHandWeapon`.

`ActorAppearanceLogic` caches this client-visible table. UI code selects
`SpriteGUIRendererComponent.ImageRUID` or `CostumeManagerComponent` by `type`.

### 1. `npcDialogEntry`

Purpose: choose the first node when a specific actor is clicked.

Runtime table name: `npcDialogEntry`

| Column | Meaning |
|---|---|
| `npcKey` | Actor identity passed to `RequestStartDialog`; joins the clicked NPC to its entry candidates. |
| `entryKey` | Unique descriptive key for this entry rule; currently used for authoring/debug identity. |
| `priority` | Ascending priority. Smaller numbers are tested first. |
| `conditionKey` | Optional join to `dialogCondition.conditionKey`. Blank means pass. |
| `entryDialogKey` | Required join to `npcDialogNode.dialogKey`. |
| `enabled` | `true` loads the row; disabled rows are ignored. |
| `notes` | Author-only note. |

Usage rules:

1. Add one or more rows for the same `npcKey`.
2. Put specific states first with small priorities.
3. Add a high-priority-number fallback row with blank `conditionKey`.
4. `DialogLogic` selects the first row whose entry condition and destination node
   condition both pass.

Current `npcGuard` order:

```text
priority 10   completed mission
priority 20   accepted mission
priority 30   owns sapphire
priority 1000 default options
```

### 2. `npcDialogNode`

Purpose: define every dialog node and its default continuation.

Runtime table name: `npcDialogNode`

| Column | Meaning |
|---|---|
| `dialogKey` | Unique node key. Referenced by Entry, Choice, and other nodes. |
| `speakerKey` | Sent to UI as `nameKey`; currently localized client-side. |
| `nodeType` | `Text`, `Options`, or `Confirm`. Values are case-sensitive. |
| `textKey` | LocaleDataSet key for the main dialog text. |
| `nextDialogKey` | Optional destination used when a `Text` node receives `Next`. |
| `actionKey` | Optional action executed before the node's next jump. |
| `conditionKey` | Optional condition required to enter/show this node. |
| `enabled` | `true` loads the node. |
| `notes` | Author-only note. |

Node behavior:

- `Text`: UI shows the Next button. Next executes `actionKey`, then follows
  `nextDialogKey`; blank destination closes the dialog.
- `Options`: UI shows eligible rows from `npcDialogChoice` with the same
  `dialogKey`.
- `Confirm`: UI shows Accept and Reject. It still requires Choice rows; Accept maps
  to visible choice 1 and Reject maps to visible choice 2.

Do not place literal player-facing text in `textKey`. Add the text to
`GameText.csv` and store only its localization key here.

### 3. `npcDialogChoice`

Purpose: define buttons belonging to an `Options` or `Confirm` node.

Runtime table name: `npcDialogChoice`

| Column | Meaning |
|---|---|
| `choiceKey` | Unique descriptive identifier for the choice. |
| `dialogKey` | Parent join to `npcDialogNode.dialogKey`. |
| `displayOrder` | Ascending display order after filtering. |
| `textKey` | LocaleDataSet key displayed on the choice button. |
| `nextDialogKey` | Optional destination node after selection. |
| `actionKey` | Optional server action executed before the jump. |
| `conditionKey` | Optional condition controlling whether this choice is visible. |
| `enabled` | `true` loads the row. |
| `notes` | Author-only note. |

Selection order:

1. Filter by `enabled`.
2. Sort by `displayOrder`.
3. Check the choice's `conditionKey`.
4. If `nextDialogKey` is present, check that the destination node exists and its
   condition passes.
5. Keep the first five eligible choices.
6. On click, execute `actionKey`, then jump to `nextDialogKey`; blank destination
   closes the dialog.

Supported action forms:

| Value | Behavior |
|---|---|
| blank | No action. |
| `Close` | No extra action; a blank destination then closes normally. |
| `AcceptMission:missionWelcome` | Calls `MissionLogic:StartMission`. |
| `CompleteMission:missionWelcome` | Calls `MissionLogic:CompleteMission`. |
| `OpenShop:generalStore01` | Calls `ShopLogic:OpenShopForUser`. |

Planned choose-one reward action forms:

| Value | Target behavior |
|---|---|
| `CompleteMission:missionWelcome:StarterWeapon:Sword` | Complete the mission with `StarterWeapon = Sword`. |
| `SelectMissionReward:missionWelcome:StarterWeapon:Sword` | Store an authoritative selection in the server Dialog session for a later completion action. |

The client still submits only a visible choice index. `DialogLogic` resolves the
configured group and option from its session; it never accepts reward item keys,
amounts, or reward types from the client. The complete transaction and
`missionRewardConfig` schema are defined in
`docs/Mission/MissionSystem.md`.

### 4. `dialogCondition`

Purpose: define reusable predicates for Entry, Node, or Choice rows.

Runtime table name: `dialogCondition`

| Column | Meaning |
|---|---|
| `conditionKey` | Reusable condition identity referenced by the other three tables. |
| `groupKey` | Clauses with the same group are AND. Different groups are OR. Blank becomes `main`. |
| `displayOrder` | Clause order for stable authoring/debug output. It does not change boolean meaning. |
| `conditionType` | Evaluator type listed below. |
| `targetKey` | Item or mission key, depending on type. |
| `comparison` | Numeric or boolean comparison. |
| `requiredValue` | Required number or boolean string. |
| `enabled` | `true` loads the clause. |
| `notes` | Author-only note. |

Supported condition types:

| `conditionType` | Actual value | Typical fields |
|---|---|---|
| `AlwaysPass` | Always `true` | Other fields may be blank. |
| `ItemCount` | `_InventoryLogic:GetItemCount(userId, targetKey)` | Numeric `requiredValue`; e.g. `GreaterOrEqual`, `1`. |
| `MissionAccepted` | Active mission exists | Boolean `requiredValue`; normally `Equal`, `true`. |
| `MissionCompleted` | Mission completion state | Boolean `requiredValue`; e.g. `Equal`, `false`. |

Numeric comparisons currently supported by `DialogLogic`:

```text
Equal
NotEqual
Greater
GreaterOrEqual
Less
LessOrEqual
```

Boolean comparisons use `Equal` or `NotEqual`. A blank comparison defaults to
equality.

Grouping example:

```csv
conditionKey,groupKey,displayOrder,conditionType,targetKey,comparison,requiredValue
conditionA,main,1,ItemCount,sealedLetter,GreaterOrEqual,1
conditionA,main,2,MissionAccepted,missionWelcome,Equal,true
```

Both rows must pass because both are in `main`.

```csv
conditionKey,groupKey,displayOrder,conditionType,targetKey,comparison,requiredValue
conditionB,active,1,MissionAccepted,missionWelcome,Equal,true
conditionB,completed,1,MissionCompleted,missionWelcome,Equal,true
```

Either group may pass, so this means mission accepted OR mission completed.

## Authoring A New Conversation

Example: create a two-step conversation for `npcExample`.

1. Add localization rows to `GameText.csv`:

```csv
Key,Source,Note,zh-TW,en
dialog_npcExample_hello,你好。,NPC greeting,你好。,Hello.
dialog_npcExample_question,要詢問消息嗎？,Choice text,要詢問消息嗎？,Ask for news?
dialog_npcExample_reply,最近很平靜。,NPC reply,最近很平靜。,It has been quiet lately.
```

2. Add nodes:

```csv
dialogKey,speakerKey,nodeType,textKey,nextDialogKey,actionKey,conditionKey,enabled,notes
npcExample_Options_01,npcExample,Options,dialog_npcExample_hello,,,,true,default options
npcExample_Text_01,npcExample,Text,dialog_npcExample_reply,,Close,,true,reply
```

3. Add a choice:

```csv
choiceKey,dialogKey,displayOrder,textKey,nextDialogKey,actionKey,conditionKey,enabled,notes
exampleAsk,npcExample_Options_01,1,dialog_npcExample_question,npcExample_Text_01,,,true,ask
```

4. Add the entry:

```csv
npcKey,entryKey,priority,conditionKey,entryDialogKey,enabled,notes
npcExample,exampleDefault,1000,,npcExample_Options_01,true,default entry
```

5. Attach `NPCActorCom` to the world entity and set `actorId = "npcExample"` in
   Maker, or ensure its `ActorCenterCom.actorId` resolves to the same value.
6. Add the same `actorId` to `npcOutfit` with `type=Avatar` and costume slots, or
   `type=Sprite` and a `spriteRuid`.
7. Stop Play, Refresh the workspace, then Play and click the entity.

## Current Example: `npcGuard`

When the guard is clicked:

1. `npcDialogEntry` tests completed, accepted, sapphire, then default.
2. The selected entry opens its `npcDialogNode`.
3. Special text nodes may continue to `npcGuard_Option_01`.
4. `npcGuard_Option_01` loads its choices from `npcDialogChoice`.
5. Choosing village status jumps to `npcGuard_Normal_01`.
6. Choosing the mission runs `AcceptMission:missionWelcome`.
7. Choosing the shop runs `OpenShop:generalStore01`.
8. Choosing leave closes because it has no next node.

## Maintenance Rules

- Keys and enum-like strings are case-sensitive.
- Keep all player-facing text in LocaleDataSet, not the four dialog Config tables.
- A blank `conditionKey` passes.
- A nonblank missing condition fails closed and logs a warning.
- A missing entry or destination node does not silently create content.
- Keep a default Entry row unless intentionally making the NPC unavailable.
- Keep `displayOrder` values stable to prevent choices from changing position.
- Do not trust client-provided actor identity or option content; the server session is
  authoritative.
- Update this document whenever a node type, condition type, action type, or CSV
  column changes.
