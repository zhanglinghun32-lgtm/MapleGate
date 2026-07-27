# Shared Progression

## Concept

Each SaveSlot owns one shared level and one shared experience bar.

- Experience gained by any owned actor is credited once to the SaveSlot.
- Party membership, active formation, field control, and battle participation do
  not decide who receives experience.
- `Progression.Level` is the total earned level budget shared by the SaveSlot.
- Every `jobs[]` entry persists its own `level`; those values are the player's
  custom distribution of the shared total.
- Actor-specific attribute growth is additionally represented by saved custom
  attribute allocations.

```text
monster reward
  -> PlayerDataLogic:AddSharedExperience(userId, amount)
  -> SaveSlot.Progression.Experience
  -> future level formula
  -> PlayerDataLogic:SetSharedLevelAndExperience(userId, level, remainder)
```

## Save Shape

```lua
Progression = {
    Level = 1,
    Experience = 0
}
```

Path: `slotData.Progression`

| Key | Type | Default | Meaning |
|---|---|---:|---|
| `Level` | integer | `1` | SaveSlot-wide total earned level budget. |
| `Experience` | integer | `0` | Current shared experience value. |

The experience threshold/curve is not finalized. `AddSharedExperience` records
the reward and marks the slot dirty; the future formula owner must call
`SetSharedLevelAndExperience` after calculating level-ups and remainder.

## Job-Level Distribution

Each actor job persists:

```lua
jobs = {
    { jobType = "Warrior", level = 6 },
    { jobType = "Mage", level = 4 }
}
```

The allocated sum across every owned actor/job must not exceed
`Progression.Level`. The difference is an unallocated level budget:

```text
unallocated = Progression.Level - sum(Actors[].jobs[].level)
```

`PlayerDataLogic:SetActorJobLevel` validates this invariant and
`GetUnallocatedJobLevelCount` exposes the remainder. The active job's saved
`level` becomes runtime `BattleActorCom.level`; the root total is not copied into
that field.

## Actor Attribute Customization

Each `Actors[]` entry persists only points allocated above its immutable
`actorConfig` base attributes:

```lua
attributeAllocations = {
    constitution = 0,
    dexterity = 0,
    intelligence = 0,
    will = 0,
    perception = 0
}
```

Runtime total attribute:

```text
actorConfig base + Actors[].attributeAllocations = BattleActorCom attribute
```

This retains different attribute builds in addition to the custom job-level
distribution. Per-job experience, derived stats, and current resources are not
saved.

## Migration

Schema v7 migrates old saves as follows:

1. Shared `Level` becomes the sum of saved actor job levels. If actor jobs are
   absent, legacy party-member levels are used as fallback.
2. Shared `Experience` becomes the maximum legacy `Party.Members[].Exp`.
   Values are not summed because old member rows may duplicate one reward.
3. `jobs[].level` is normalized and retained; legacy actor-level `level` is
   migrated into the first job. Party-member `Level`/`Exp` are removed.
4. Legacy saved attribute totals are converted to non-negative custom points by
   subtracting the matching `actorConfig` base attributes.

The migrated slot is marked dirty and is rewritten only through the existing
explicit SaveSlot action.
