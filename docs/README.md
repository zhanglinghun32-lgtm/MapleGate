# Project Documentation

Design and architecture docs for this MSW project. Maker does not scan this
folder — it is for humans and AI collaborators only.

## Conventions

Read **`ScriptRules.md`** first. It defines:

- New-system workflow: **Design doc → Config → Logic → Component (domain folder)**
- Flat `Data/Config/` and flat `ui/`
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
| [Party/PartySystem.md](Party/PartySystem.md) | Party membership, field control, formation, whole-party battle entry |
| [PlayerData/SaveSlotSchema.md](PlayerData/SaveSlotSchema.md) | Canonical SaveSlot keys, types, casing, and ownership |
| [UI/SystemMenuUI.md](UI/SystemMenuUI.md) | Persistent gameplay system menu and save actions |
| [GameplayFlow/GameplayFlowLogic.md](GameplayFlow/GameplayFlowLogic.md) | Playable world entry/exit coordination |
| [PlayerControl/PlayerControlLogic.md](PlayerControl/PlayerControlLogic.md) | Local player operation lock ownership |
| [SkillAction/SkillActionSystem.md](SkillAction/SkillActionSystem.md) | Shared skill requests in field and battle contexts |
| [SkillPresentation/SkillPresentationSystem.md](SkillPresentation/SkillPresentationSystem.md) | Data-driven avatar action and effect timelines |

## Adding A New System

1. Create `docs/{System}/{System}.md` (or `docs/Design/{System}.md`) with scope, Config columns, Logic API, and component list.
2. Add Config tables under `Data/Config/` (flat).
3. Add `@Logic` under `Logic/`.
4. Add `@Component` under the matching domain folder (`UI/`, `Battle/`, `MapScene/`, …) and bind on the entity in Maker.
5. Add UI in flat `ui/` if the system needs new screens; put UI scripts in `UI/` (subfolders per screen OK).
