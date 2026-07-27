# Mission System

## Scope

This document defines the target mission architecture for this project. It
extends the existing `MissionLogic`, `PlayerDataLogic`, Inventory, Progression,
and NPC Dialog systems instead of importing a second quest/save framework.

The project has **one-time missions only**:

- no daily missions;
- no weekly missions;
- no reset cycle;
- no repeatable acceptance;
- a completed `missionKey` can never be accepted again in the same SaveSlot.

The official `quest-achievement-package` is a useful reference for its
ActionEvent / condition pattern, but is not integrated directly because its
player components and DataStorage ownership overlap the existing
`PlayerDataLogic` and `MissionLogic`.

## Ownership

| Concern | Owner |
|---|---|
| Mission definitions | Four UserDataSets under `Data/Config/mission/` |
| Active/completed state and progress | `MissionLogic` |
| Save serialization | `PlayerDataLogic` via `slotData.Mission` |
| Item grants, checks, and consumption | `InventoryLogic` |
| Shared experience reward | `PlayerDataLogic:AddSharedExperience` |
| Acceptance/turn-in dialogue | `DialogLogic` |
| Acceptance predicates shared by Dialog/Mission | Planned `ConditionLogic` |
| Accept/complete side effects | Planned `MissionActionResolver` |
| Map-local NPC visibility | NPC / map component, driven by mission state or player flag |
| Mission UI | Client-only UI fed by server snapshots |

Static Config is never mutated at runtime. Mission progress changes the
in-memory SaveSlot cache and calls `PlayerDataLogic:MarkDirty(userId)`;
DataStorage writes remain part of the existing explicit SaveSlot flow.

## State Model

```text
Unavailable
    ↓ acceptance conditions pass
Available
    ↓ TryAcceptMission succeeds
Active
    ↓ all objectives are satisfied
ReadyToTurnIn
    ↓ TryTurnInMission succeeds
Completed
```

`Available` and `ReadyToTurnIn` are computed states. Do not persist them
separately because they can be derived from Config, relation state, inventory,
and objective progress.

`Completed[missionKey] = true` is terminal. There is no reset or repeat path.

## Runtime Flow

```text
Dialog / Battle / Inventory / Map trigger
    ↓ ReportAction(userId, actionType, targetKey, amount)
MissionLogic
    ├─ validates mission state
    ├─ updates objective progress
    ├─ computes ReadyToTurnIn
    └─ marks SaveSlot dirty

Dialog AcceptMission / CompleteMission
    ↓
MissionLogic:TryAcceptMission / TryTurnInMission
    ├─ validates source NPC + shared conditions
    ├─ pre-validates every mutation
    ├─ changes mission state exactly once
    └─ calls MissionActionResolver / reward owners
```

Only authoritative server systems report progress:

- confirmed monster death → `Kill`;
- inventory mutation → recompute `Collect` / `Deliver`;
- validated dialogue action → `Talk`;
- server-side map trigger → `Reach`.

The client must never submit arbitrary progress such as “I killed five
monsters.”

## The Four Mission Config Tables

All four tables are `.csv` + `.userdataset` pairs under:

`RootDesk/MyDesk/Data/Config/mission/`

Runtime lookup uses each UserDataSet's `name`, not its folder path.

### 1. `missionConfig`

One row per mission. This table owns identity, display keys, acceptance/turn-in
actors, and high-level lifecycle policy.

Target columns:

| Column | Required | Meaning |
|---|:---:|---|
| `missionKey` | yes | Stable unique mission key. |
| `nameKey` | yes | LocaleDataSet key for mission name. |
| `descriptionKey` | yes | LocaleDataSet key for mission description. |
| `categoryKey` | no | UI grouping key such as `Main` or `Side`. |
| `acceptActorKey` | no | NPC actor key allowed to accept the mission. Blank for non-dialog acceptance. |
| `reportActorKey` | no | NPC actor key allowed to turn in. Blank for automatic completion. |
| `acceptConditionKey` | no | Shared condition-group key evaluated before acceptance. |
| `completionMode` | yes | `Report` or `Auto`. |
| `autoAccept` | yes | If `true`, acceptance begins as soon as conditions pass. |
| `canAbandon` | yes | Whether an active mission may be abandoned. |
| `enabled` | yes | Disabled missions are not loaded. |
| `notes` | no | Author-only notes. |

Example:

```csv
missionKey,nameKey,descriptionKey,categoryKey,acceptActorKey,reportActorKey,acceptConditionKey,completionMode,autoAccept,canAbandon,enabled,notes
missionWelcome,mission_welcome_name,mission_welcome_desc,Main,npcElder,npcGuard,conditionWelcomeAvailable,Report,false,false,true,初次差事
```

Rules:

1. Player-facing text belongs in `GameText.csv`; this table stores localization
   keys only.
2. There is no `repeatable`, cycle, reset, daily, or weekly column.
3. Do not store `nextMissionKey`. The next mission references the previous
   mission through its acceptance condition, so the chain has one owner.
4. `acceptActorKey` and `reportActorKey` use actor identity, never map
   `entityId`.
5. `completionMode=Auto` requires a blank `reportActorKey`.

### 2. `missionObjectiveConfig`

One mission may have multiple rows. Each row describes one independently tracked
objective.

Target columns:

| Column | Required | Meaning |
|---|:---:|---|
| `missionKey` | yes | Join to `missionConfig.missionKey`. |
| `objectiveKey` | yes | Unique within the mission; Save progress uses this key. |
| `displayOrder` | yes | Stable objective display order. |
| `objectiveType` | yes | `Talk`, `Kill`, `Collect`, `Deliver`, `Reach`, or `Custom`. |
| `targetKey` | yes | Actor, monster, item, location, or custom target key. |
| `requiredCount` | yes | Required amount, minimum `1`. |
| `progressMode` | yes | `Accumulate` or `InventorySnapshot`. |
| `consumeOnTurnIn` | yes | For Deliver objectives, consume required items on successful turn-in. |
| `textKey` | yes | LocaleDataSet key for objective text. |
| `enabled` | yes | Disabled rows are ignored. |
| `notes` | no | Author-only notes. |

Example:

```csv
missionKey,objectiveKey,displayOrder,objectiveType,targetKey,requiredCount,progressMode,consumeOnTurnIn,textKey,enabled,notes
missionWelcome,deliverLetter,1,Deliver,sealedLetter,1,InventorySnapshot,true,mission_obj_deliver_letter,true,交付密封信
missionSlimeHunt,killSlime,1,Kill,slime,5,Accumulate,false,mission_obj_kill_slime,true,擊倒史萊姆
```

Progress rules:

- `Kill`, `Talk`, and `Reach` normally use `Accumulate`.
- `Collect` and `Deliver` use `InventorySnapshot`: read the authoritative
  Inventory count instead of permanently incrementing a counter.
- Progress is clamped to `0 .. requiredCount`.
- Every enabled objective must pass before the mission is ready.
- Deliver items are consumed only inside a successful turn-in transaction.

### 3. `missionRewardConfig`

One mission may have multiple reward rows. Rewards are separated from side
effects so the Mission UI can display them without interpreting world actions.

Target columns:

| Column | Required | Meaning |
|---|:---:|---|
| `missionKey` | yes | Join to `missionConfig.missionKey`. |
| `rewardGroupKey` | yes | Reward group unique within the mission, such as `Guaranteed` or `StarterWeapon`. |
| `selectionMode` | yes | `All` grants every row in the group; `ChooseOne` requires one selected `optionKey`. |
| `optionKey` | conditional | Stable option identity within a `ChooseOne` group. Blank for `All`. Multiple rows may share one option to form a reward bundle. |
| `rewardOrder` | yes | Stable grant/presentation order. |
| `rewardType` | yes | `Item`, `Currency`, `SharedExperience`, `Skill`, or `Actor`. |
| `targetKey` | conditional | Item/currency/skill/actor key. Blank for shared experience. |
| `amount` | yes | Positive grant amount. |
| `targetActor` | yes | `ActiveActor`, `PrimaryActor`, or another documented target mode. |
| `enabled` | yes | Disabled rows are ignored. |
| `notes` | no | Author-only notes. |

Example:

```csv
missionKey,rewardGroupKey,selectionMode,optionKey,rewardOrder,rewardType,targetKey,amount,targetActor,enabled,notes
missionWelcome,Guaranteed,All,,1,SharedExperience,,20,PrimaryActor,true,共享經驗
missionWelcome,Guaranteed,All,,2,Currency,gold,10,ActiveActor,true,金幣沿用 Inventory item
missionWelcome,StarterWeapon,ChooseOne,Sword,10,Item,ironSword,1,ActiveActor,true,武器擇一
missionWelcome,StarterWeapon,ChooseOne,Staff,10,Item,oakStaff,1,ActiveActor,true,武器擇一
```

Ownership rules:

- Every `All` group is granted automatically.
- Every `ChooseOne` group requires exactly one valid `optionKey`.
- Rows sharing `(missionKey, rewardGroupKey, optionKey)` form one selectable
  bundle and are granted together.
- The Dialog choice supplies only `rewardGroupKey` and `optionKey`; reward type,
  item key, and amount are resolved from the server-side Config cache.
- `SharedExperience` calls `PlayerDataLogic:AddSharedExperience`; do not write
  legacy `Party.Members.Exp`.
- `Currency` uses the same Inventory item key consumed by Shop
  `currencyItemKey`; do not create a second gold balance.
- Item rewards go through `InventoryLogic`.
- Every reward must be pre-validated before any reward is applied.

### 4. `missionActionConfig`

Defines non-reward mutations at acceptance or completion. The CSV stores
declarative action types, never raw Lua function or callback names.

Target columns:

| Column | Required | Meaning |
|---|:---:|---|
| `missionKey` | yes | Join to `missionConfig.missionKey`. |
| `timing` | yes | `OnAccept`, `OnComplete`, or `OnAbandon`. |
| `actionOrder` | yes | Stable execution order. |
| `actionType` | yes | Registered `MissionActionResolver` action type. |
| `targetKey` | conditional | Item, mission, flag, actor, or map-state key. |
| `amount` | no | Numeric argument; default `0`. |
| `param1` | no | Action-specific string argument. |
| `param2` | no | Action-specific string argument. |
| `failurePolicy` | yes | `Abort` or `Continue`. Use `Abort` for authoritative mutations. |
| `enabled` | yes | Disabled rows are ignored. |
| `notes` | no | Author-only notes. |

Example:

```csv
missionKey,timing,actionOrder,actionType,targetKey,amount,param1,param2,failurePolicy,enabled,notes
missionWelcome,OnAccept,1,AddItem,sealedLetter,1,,,Abort,true,接任務時取得密封信
missionWelcome,OnComplete,1,SetPlayerFlag,npcElderGone,1,,,Abort,true,玩家個人世界狀態
```

Initial registered action types:

- `AddItem`
- `RemoveItem`
- `SetPlayerFlag`
- `SetNpcVisibility`
- `StartMission` only if explicit automatic chaining is later desired
- `Custom` only through an allow-listed resolver handler

Do not store values such as `SomeLogic:SomeMethod` in Config. Resolver handlers
are registered in code by `actionType`, return `{ Success, Reason }`, and are
server-only.

## Acceptance Conditions

`missionConfig.acceptConditionKey` references the same reusable condition model
used by NPC Dialog. The current evaluator is embedded in `DialogLogic`; before
Mission implementation it should be extracted into a shared `ConditionLogic` so
Dialog and Mission cannot disagree about the same condition.

Existing condition types remain usable:

- `ItemCount`
- `MissionAccepted`
- `MissionCompleted`
- `AlwaysPass`

Planned mission-related types:

- `MissionAvailable`
- `MissionReadyToTurnIn`
- `RelationRank`

### Relation threshold conditions

Mission acceptance may require **Neutral or above** or **Friendly or above**.
Represent this as a normal ordered comparison, not a special boolean.

Canonical relation tiers:

| Tier | Rank |
|---|---:|
| `Hostile` | 0 |
| `Unfriendly` | 1 |
| `Neutral` | 2 |
| `Friendly` | 3 |
| `Trusted` | 4 |

Example shared-condition rows:

```csv
conditionKey,groupKey,displayOrder,conditionType,targetKey,comparison,requiredValue,enabled,notes
conditionNeutralOrAbove,main,1,RelationRank,village,GreaterOrEqual,Neutral,true,村莊關係需中立以上
conditionFriendlyOrAbove,main,1,RelationRank,npcElder,GreaterOrEqual,Friendly,true,與村長需友好以上
```

`targetKey` may be a faction key such as `village` or an actor key such as
`npcElder`; the Relation system owns that distinction. `ConditionLogic` converts
both the actual and required tier to numeric rank and applies the standard
comparison.

Relation state is not implemented yet. The required owner contract is:

```text
RelationLogic:GetRelationRank(userId, targetKey) -> integer
```

Until that Logic and its SaveSlot section exist, a nonblank `RelationRank`
condition must fail closed and log a warning. Do not silently treat a missing
relation as Neutral.

## Accept Transaction

```text
TryAcceptMission(userId, missionKey, sourceActorKey)
  1. Config exists and enabled
  2. mission is not Active or Completed
  3. acceptConditionKey passes
  4. sourceActorKey matches acceptActorKey when configured
  5. pre-validate every OnAccept action
  6. create Active state
  7. execute OnAccept actions in actionOrder
  8. MarkDirty once
  9. publish mission snapshot
 10. return { Success, Reason }
```

The method needs a per-user/per-mission resolving lock so two clicks cannot
accept the same mission twice.

## Turn-In Transaction

```text
TryTurnInMission(userId, missionKey, sourceActorKey, rewardSelections)
  1. mission is Active
  2. every objective is satisfied
  3. sourceActorKey matches reportActorKey
  4. Deliver items are still present
  5. every ChooseOne group has exactly one valid selected option
  6. pre-validate consumption, selected rewards, and OnComplete actions
  7. acquire resolving lock
  8. consume Deliver items
  9. grant every All group and the selected ChooseOne bundles
 10. execute OnComplete actions in actionOrder
 11. remove Active state; set Completed[missionKey] = true
 12. MarkDirty once
 13. publish mission snapshot
 14. return { Success, Reason }
```

No mutation should be capable of failing after pre-validation. If an
authoritative `Abort` action fails, keep the mission Active and return failure;
Dialog must not advance to the success node.

`rewardSelections` is a table keyed by reward group:

```lua
{
    StarterWeapon = "Sword"
}
```

It is a request parameter, not persisted mission progress. `MissionLogic`
accepts only group and option keys, looks up the actual reward rows in its
server-side cache, and rejects missing, unknown, disabled, or extra selections.

## Dialog Integration

Existing action strings remain the adapter:

- `AcceptMission:<missionKey>`
- `CompleteMission:<missionKey>`

Target changes:

1. `DialogLogic` passes its validated `session.ActorId` as `sourceActorKey`.
2. `AcceptMission` calls `TryAcceptMission`.
3. `CompleteMission` calls `TryTurnInMission`.
4. `ExecuteAction` returns `{ Success, Reason }`.
5. Dialog advances only when the action succeeds; failure keeps the current node
   open or routes to a configured error node.

### Dialog-driven choose-one rewards

Reward selection is presented through normal `npcDialogNode` and
`npcDialogChoice` rows. The client still submits only the visible option index;
`DialogLogic` resolves the authoritative choice row from its server session and
passes the configured group/option keys to `MissionLogic`.

For a mission with one choose-one group, each reward option may complete the
mission directly:

```csv
choiceKey,dialogKey,displayOrder,textKey,nextDialogKey,actionKey,conditionKey,enabled,notes
chooseSword,npcGuard_Reward_01,1,reward_choose_sword,,CompleteMission:missionWelcome:StarterWeapon:Sword,,true,選擇劍
chooseStaff,npcGuard_Reward_01,2,reward_choose_staff,,CompleteMission:missionWelcome:StarterWeapon:Staff,,true,選擇法杖
```

Target parser result:

```text
CompleteMission:<missionKey>:<rewardGroupKey>:<optionKey>
    -> rewardSelections[rewardGroupKey] = optionKey
    -> MissionLogic:TryTurnInMission(
           userId,
           missionKey,
           session.ActorId,
           rewardSelections
       )
```

If a mission later has multiple `ChooseOne` groups, intermediate choices use
`SelectMissionReward:<missionKey>:<rewardGroupKey>:<optionKey>` to accumulate
selections in the server-side Dialog session. The final
`CompleteMission:<missionKey>` action forwards the whole selection table.
Selections are cleared when the Dialog session closes or turn-in succeeds.

Security and transaction rules:

1. Never pass `rewardType`, `targetKey`, or `amount` from the client or Dialog
   action.
2. The selected `optionKey` must belong to the requested mission and group.
3. Exactly one option is required for each enabled `ChooseOne` group.
4. `All` groups reject supplied selections and are always granted.
5. The reward preview is client-side presentation only; Config remains server
   authority.
6. Failed validation keeps the mission Active and does not consume items, grant
   partial rewards, set Completed, or close the success branch.
7. Duplicate clicks are blocked by the same per-user/per-mission resolving lock
   used by normal completion.

Recommended NPC entry priority:

```text
10    MissionReadyToTurnIn
20    MissionAccepted
30    MissionAvailable
40    MissionCompleted
1000  fallback
```

Authoring must keep NPC identity consistent. The current sample says
`missionWelcome` is accepted from `npcElder`, while the current
`npcGuard` choice executes `AcceptMission:missionWelcome`; one side must be
changed before source-actor validation is enabled.

## Save Shape

The existing one-time mission shape remains valid:

```lua
Mission = {
    Active = {
        {
            MissionKey = "missionSlimeHunt",
            Progress = {
                killSlime = 3
            }
        }
    },
    Completed = {
        missionWelcome = true
    }
}
```

Rules:

- `Progress` keys are `missionObjectiveConfig.objectiveKey`.
- InventorySnapshot objectives may be recomputed and need not duplicate
  inventory counts permanently.
- `Completed` stays `table<string, boolean>` because missions never repeat.
- Mission progress updates only the in-memory cache and calls `MarkDirty`.
- Do not call DataStorage per kill, item pickup, or objective update.

## NPC Visibility and World Effects

Do not save or resolve a permanent NPC effect by map `entityId`.

- `actorKey` identifies the NPC design.
- `entityId` identifies one loaded map instance only.
- Destroying a shared server entity would affect every player.

For a per-player disappearance:

1. derive visibility from `MissionCompleted` or set a player flag;
2. server validation rejects interaction for that player;
3. a targeted client update hides that NPC's local presentation and touch
   interaction;
4. map entry reapplies the condition.

For a true world-wide disappearance, use map/world state owned by a map-scoped
component. This is a separate feature from a player's mission completion.

## Config Validation

Mission cache loading should fail closed and log all of the following:

- duplicate `missionKey`;
- duplicate `(missionKey, objectiveKey)`;
- missing joins between the four tables;
- enabled mission with no enabled objective;
- unknown objective/reward/action/condition type;
- invalid `completionMode`;
- `Auto` mission with `reportActorKey`;
- `Report` mission without `reportActorKey`;
- non-positive counts or reward amounts;
- duplicate `(missionKey, rewardGroupKey, optionKey, rewardOrder)`;
- unknown `selectionMode`;
- blank `optionKey` in a `ChooseOne` group;
- nonblank `optionKey` in an `All` group;
- a `ChooseOne` reward option that is not reachable from an eligible Dialog
  choice;
- Deliver objective not using `InventorySnapshot`;
- `consumeOnTurnIn=true` on a non-Deliver objective;
- unknown relation tier;
- `acceptActorKey` / Dialog action mismatch;
- localization keys missing from `GameText`.

## Implementation Order

1. Lock this document and the four CSV schemas.
2. Extract shared condition evaluation from `DialogLogic` into
   `ConditionLogic`; add `RelationRank` as fail-closed until Relation is
   implemented.
3. Preload and validate all four Mission DataSets in `MissionLogic.OnBeginPlay`.
4. Implement `TryAcceptMission`, objective evaluation, and
   `TryTurnInMission`.
5. Add Inventory removal/pre-validation and reward adapters.
6. Implement `MissionActionResolver`.
7. Make `DialogLogic.ExecuteAction` result-aware.
8. Connect authoritative Kill / Inventory / Talk / Reach progress sources.
9. Add server snapshots and Mission UI.
10. Verify accept → save/reload → progress → turn-in → reward → completed
    dialogue and NPC visibility.
