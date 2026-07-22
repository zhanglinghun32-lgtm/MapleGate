# Skill Presentation System

## Goal

Play ordinary skill presentations from DataSet timelines without adding one
method or one action-key branch per skill.

## Shared Key

`skillKey` is the only join key. `skillConfig`, `skillPresentationConfig`, and
`skillPresentationStepConfig` use the exact same value (for example
`spearPower`). There is no separate `presentationKey` column.

## Config

Tables live under `RootDesk/MyDesk/Data/Config/skill/` (`.csv` + `.userdataset`
pairs). `skillConfig` stays at `Data/Config/` and joins only by `skillKey`.

- `skill/skillPresentationConfig.csv`: one row per skill presentation. Avatar
  action, play rate, and total duration.
- `skill/skillPresentationStepConfig.csv`: one row per timed presentation event.
  Effects choose Sponsor or Target attachment and carry local offset/scale data.

## Runtime

`BattleSkillPresentationComponent` loads the config row and matching steps,
sorts them by absolute `atSeconds`, then dispatches each event through a generic
event runner. Absolute timeline waits prevent frame-rounding error from
accumulating across steps.

## Manual Adjustment

AnimationClip pivots and canvas bounds differ, so local offset and scale remain
authored values. They belong in Config, not in skill-specific mLua methods.

## Extensions

Ordinary AvatarAction and Effect events are data-driven. Skills requiring dash,
teleport, projectile simulation, branching, or sustained input may later use a
named custom handler while retaining the shared `skillKey`.

