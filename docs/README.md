# Project Documentation

Design and architecture docs for this MSW project. Maker does not scan this
folder — it is for humans and AI collaborators only.

## Conventions

Read **`ScriptRules.md`** first. It defines:

- New-system workflow: **Design doc → Config → Logic → Component (domain folder)**
- `Data/Config/` flat by default (`skill/` for range / effect / presentation); flat `ui/`
- No RUID Catalog — use Config columns or Maker `.ui` preview
- `@Logic` in `Logic/`; `@Component` in domain folders (`UI/`, `Battle/`, …)

## Index

| Path | Topic |
|------|-------|
| [ScriptRules.md](ScriptRules.md) | Architecture, folders, naming, responsibilities |
| [BattleFlow/BattleFlowLogic.md](BattleFlow/BattleFlowLogic.md) | `RequestStartBattle` request/payload schema; three triggers; enter/exit |
| [BattleFlow/BattleSystem.md](BattleFlow/BattleSystem.md) | Turn permission, turn start/end, settlement, operation locks |
| [BattleFlow/BattleUIComponent.md](BattleFlow/BattleUIComponent.md) | BattleUI turn-sequence overlay only; persistent UIs stay |
| [Actor/BattleActorComponent.md](Actor/BattleActorComponent.md) | Per-actor HP / MP / stats authority |
| [Actor/BattleActorInitPaths.md](Actor/BattleActorInitPaths.md) | Player Save vs Monster/NPC Config init paths (`statsSource`) |
| [Actor/ActorVariableExplain.md](Actor/ActorVariableExplain.md) | Actor field meanings for balance notes and formula review |
| [Calculator/BattleCalculator.md](Calculator/BattleCalculator.md) | Numeric-only combat formula pipeline; accepts fully converged scalar values |
| [Dialog/DialogSystem.md](Dialog/DialogSystem.md) | Dialog flow, current status, four Config tables, actions, conditions, and TODOs |
| [Mission/MissionSystem.md](Mission/MissionSystem.md) | One-time mission architecture, four Config tables, relation-gated acceptance, rewards, and actions |
| [Party/PartySystem.md](Party/PartySystem.md) | Party membership, field control, formation, whole-party battle entry |
| [PlayerData/SaveSlotSchema.md](PlayerData/SaveSlotSchema.md) | Canonical SaveSlot keys, types, casing, and ownership |
| [Progression/SharedProgression.md](Progression/SharedProgression.md) | Shared total level/experience, job-level distribution, and actor attribute allocations |
| [Shop/ShopSystem.md](Shop/ShopSystem.md) | Planned server transactions and per-player shop-state persistence |
| [Localization/Localization.md](Localization/Localization.md) | LocaleDataSet location, CSV editing workflow, and runtime boundary |
| [UI/SystemMenuUI.md](UI/SystemMenuUI.md) | Persistent gameplay system menu and save actions |
| [GameplayFlow/GameplayFlowLogic.md](GameplayFlow/GameplayFlowLogic.md) | Playable world entry/exit coordination |
| [PlayerControl/PlayerControlLogic.md](PlayerControl/PlayerControlLogic.md) | Local player operation lock ownership |
| [SkillAction/SkillActionSystem.md](SkillAction/SkillActionSystem.md) | Shared skill requests in field and battle contexts |
| [SkillAction/SkillCastPipeline.md](SkillAction/SkillCastPipeline.md) | Cast pipeline: Context → Wrapper → Calculator → Resolver |
| [SkillPresentation/SkillPresentationSystem.md](SkillPresentation/SkillPresentationSystem.md) | Data-driven avatar action and effect timelines |
| [SkillEffect/SkillEffectSystem.md](SkillEffect/SkillEffectSystem.md) | Timed skill logic effects (`skillEffect` Config schema) |

## Adding A New System

1. Create `docs/{System}/{System}.md` (or `docs/Design/{System}.md`) with scope, Config columns, Logic API, and component list.
2. Add Config tables under `Data/Config/` (skill range / effect / presentation under `Config/skill/`).
3. Add `@Logic` under `Logic/`.
4. Add `@Component` under the matching domain folder (`UI/`, `Battle/`, `MapScene/`, …) and bind on the entity in Maker.
5. Add UI in flat `ui/` if the system needs new screens; put UI scripts in `UI/` (subfolders per screen OK).
