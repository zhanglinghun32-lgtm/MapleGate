# BattleSystem Component

The battle session is owned by `BattleSystem` (`Battle/BattleSystem.mlua`).
The component owns the battle session flow; actor state remains inside each actor's
`BattleActorComponent` (`Battle/BattleActorComponent.mlua`).

If the session is spawned at runtime, use a `BattleSystem.model` template only
as the spawn vehicle — the script file does not need to live beside the model.

Config data (e.g. `skillConfig`, planned `encounterConfig`) lives in flat
`Data/Config/`. Skill icons use `skillConfig.iconRuid` — there is no RUID Catalog.

## Actor Registry

The current design uses actor entities with `BattleActorComponent`.

Recommended registry:

```text
BattleSystem
- activeActorIds
- actorMap: actorId -> actor entry / BattleActorComponent
- currentActorId
- battleQueue -> BattleQueue component on the same entity

BattleQueue
- turnQueue
- actorMap (read-only copy for speed / alive checks)
```

State changes should go through `BattleActorComponent`:

```text
targetActor:ApplyDamage(value)
sponsorActor:UseMp(cost)
sponsorActor:UseStamina(cost)
```

`BattleSystem` should not directly assign HP, MP, stamina, or stat
values.

## Battle Keys

Battle system enum-like strings should be centralized in
`RootDesk/MyDesk/Logic/BattleKeys.mlua`.

Use `_BattleKeys` instead of writing magic strings such as `"Damage"`,
`"TotalAttack"`, or `"Enemy"` directly in battle scripts.

Examples:

```lua
entry.entryType = _BattleKeys.entryDamage
modifier.statKey = _BattleKeys.statTotalAttack
action.targetRule = _BattleKeys.targetEnemy
```

Content keys still belong in Config data, not `BattleKeys`.

Examples:

- `NormalAttack`
- `PoisonSlash`
- `IronSword`
- `AttackUp`
- `Poison`

## Responsibilities

- Receive `StartBattle(payload)` from `BattleFlowLogic`.
- Register all battle actors.
- Own active actor ids and turn queue.
- Decide whose turn it is.
- Receive action requests.
- Validate that the sponsor can act and the target is valid.
- Ask `BattleCalculator` to calculate action results.
- Apply action results through `BattleActorComponent` interfaces.
- Send battle presentation RPC directly to `BattleClientLogic`.
- Wait in `PresentingAction` until the client acknowledges the presentation or the server timeout expires.
- Remove dead actors from the active turn queue.
- Check victory, defeat, escape, and cancel conditions.
- Report battle result back to `BattleFlowLogic`.

## Non-Responsibilities

- Do not load encounter Config by key.
- Do not read or write permanent save data directly.
- Do not decide scene transitions after battle ends.
- Do not bind battle UI buttons.
- Do not own field monster movement or chase AI.
- Do not directly mutate actor resources without `BattleActorComponent`.
- Do not route every battle event through `BattleFlowLogic`.

## Action Flow

```text
RequestAction(sponsorId, actionKey, targetIds/actionContext)
  -> validate battle is active
  -> validate sponsorId is current turn actor
  -> resolve click/bounds targeting into authoritative target ids
  -> validate sponsor and targets exist
  -> ask BattleCalculator for result
  -> apply result entries through BattleActorComponent
  -> transition to PresentingAction
  -> send action presentation RPC to BattleClientLogic
  -> wait for client acknowledgement (or server timeout fallback)
  -> transition to TurnEnd
  -> process deaths
  -> remove dead actors from turnQueue
  -> check battle end
  -> advance turn
```

## Synced Display State

`BattleSystem` exposes server-authoritative display fields to clients:

| Property | Purpose |
|----------|---------|
| `@Sync battlePhase` | Turn phase for enabling HUD input |
| `@Sync currentActorId` | Current turn actor id |

Server-only fields such as `battleId`, `playerUserId`, and `isActive` stay
non-synced unless a future HUD requirement needs them.

Do **not** use `@Sync` to open `BattleUI`. UI shell open/close stays on
`BattleFlowLogic` Client RPC -> `BattleClientLogic` -> `UIManagerLogic`.

On the client, `BattleSystem:OnSyncProperty` forwards phase/actor updates to
`BattleUIComponent` through `BattleClientLogic`.

See `docs/BattleFlow/BattleUIComponent.md`.

## Client Presentation RPC

`BattleSystem` owns the active battle events, so it is also the owner of battle
presentation RPC.

The action flow uses a presentation sequence id so stale or duplicate client
acknowledgements cannot advance a newer turn:

```text
ResolvingAction
  -> PresentingAction
  -> PlayActionPresentationClient(...)
  -> BattleClientLogic:PlayActionPresentation(...)
  -> NotifyActionPresentationFinished(battleId, presentationId)
  -> TurnEnd
```

`ResolveAction` returns one `actionResultPayload` for both client cache updates
and presentation. Its contract is `{ Revision, Changes }`; `Changes` contains
every authoritative mutation caused by the action (damage, healing, resources,
buff add/remove, cooldown, movement, death, and other actor patches). Send this
table once rather than emitting one RPC per changed field.

`presentationTimeoutSeconds` is a server-side fallback. It prevents a missing,
disconnected, or failed client presentation from permanently locking the battle.

Examples:

```text
BattleStartedClient(battleId, snapshot)
TurnStartedClient(battleId, actorId)
ActorMovedClient(battleId, actorId, x, y)
ActorDamagedClient(battleId, actorId, damage, hpAfter)
ActionResolvedClient(battleId, actionResult)
BattleEndedClient(battleId, result)
```

These methods should use `@ExecSpace("Client")` and call `_BattleClientLogic`
on the target client.

Targeting rule:

```lua
self:ActorDamagedClient(battleId, actorId, damage, hpAfter, playerUserId)
```

The final `playerUserId` is the RPC target at the call site. It is not declared
in the client RPC method signature.

`BattleFlowLogic` remains the entry and finalization layer. It should receive
the final battle result for rewards, save updates, mission progress, and scene
flow, but it should not relay every turn/action presentation event.

## BattleCalculator Input

`BattleCalculator` should receive already-prepared input. It should not query UI,
scene flow, save data, or raw Config by itself.

A `BattleSkill` component in `Battle/BattleSkill.mlua` loads one `skillConfig`
row by `skillKey`, then exposes the prepared `actionConfig` table through
`BuildActionConfig()`.

Recommended input shape:

```text
BattleActionInput
- battleId
- actionKey
- sponsorId
- targetIds
- actionContext
- sponsorSnapshot
- targetSnapshots
- actionConfig
- effectModifiers
- randomSeed
```

Snapshot values should contain only what the calculator needs for this action:

```text
ActorBattleSnapshot
- actorId
- teamId
- level
- hp
- maxHp
- mp
- maxMp
- stamina
- maxStamina
- totalAttack
- totalDefense
- speed
- activeEffectKeys
```

This keeps `BattleCalculator` deterministic and easy to test.

`actionContext` is the normalized targeting payload. Legacy callers may still
send a plain `targetIds` array. Point/range callers send:

```text
ActionContext
- TargetIds: optional explicit actor ids
- TargetPoint: optional world-space Vector2 from click/touch
- Bounds: optional explicit world-space bounds table
- ResolvedTargetIds: server-authoritative ids after range expansion
- RangeShape / RangeRadius / RangeWidth / RangeHeight: copied from skillConfig
```

The server reads `skillConfig.targetShape`, `targetRadius`, `targetWidth`,
`targetHeight`, and `targetMaxCount` by `actionKey` before resolving the range.

## BattleCalculator Result

One action can produce multiple effects, so the result should be a list of
entries instead of a single damage number.

Recommended result shape:

```text
BattleActionResult
- battleId
- sponsorId
- actionKey
- entries
- logs
```

Each entry describes one effect to apply:

```text
BattleResultEntry
- entryType
- sourceActorId
- targetActorId
- value
- resourceKey
- effectKey
- duration
- isHit
- isCritical
- reason
```

Recommended first `entryType` values from `_BattleKeys`:

- `_BattleKeys.entryDamage`
- `_BattleKeys.entryHeal`
- `_BattleKeys.entryMpChange`
- `_BattleKeys.entryStaminaChange`
- `_BattleKeys.entryAddEffect`
- `_BattleKeys.entryRemoveEffect`
- `_BattleKeys.entryRevive`
- `_BattleKeys.entryGuard`
- `_BattleKeys.entryMiss`

Example action result:

```text
BattleActionResult
- sponsorId = "Player01"
- actionKey = "SlashWithPoison"
- entries:
  - Damage target=Monster01 value=30
  - AddEffect target=Monster01 effectKey=Poison duration=3
  - StaminaChange target=Player01 value=-10
```

`BattleSystem` applies each entry. `BattleCalculator` only describes
what should happen.

## Applying Results

Recommended application rules:

- `Damage` calls `targetActor:ApplyDamage(value)`.
- `Heal` calls `targetActor:Heal(value)`.
- `MpChange` calls `RecoverMp()` or `UseMp()` depending on sign.
- `StaminaChange` calls `RecoverStamina()` or `UseStamina()` depending on sign.
- `AddEffect` delegates to `EffectSystem`.
- `RemoveEffect` delegates to `EffectSystem`.
- `Revive` delegates to a future actor revive interface.
- `Guard` and `Miss` produce presentation/UI feedback but do not mutate HP.

After all entries are applied, `BattleSystem` checks each affected
actor for death and removes dead actors from `turnQueue`.
