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
| [BattleFlow/BattleFlow.md](BattleFlow/BattleFlow.md) | Turn-based battle entry (`BattleFlowLogic`) |
| [BattleFlow/BattleSystem.md](BattleFlow/BattleSystem.md) | Battle session component (`BattleSystemComponent`) |
| [Actor/BattleActorComponent.md](Actor/BattleActorComponent.md) | Per-actor HP / MP / stats authority |

## Adding A New System

1. Create `docs/{System}/{System}.md` (or `docs/Design/{System}.md`) with scope, Config columns, Logic API, and component list.
2. Add Config tables under `Data/Config/` (flat).
3. Add `@Logic` under `Logic/`.
4. Add `@Component` under the matching domain folder (`UI/`, `Battle/`, `MapScene/`, …) and bind on the entity in Maker.
5. Add UI in flat `ui/` if the system needs new screens; put UI scripts in `UI/` (subfolders per screen OK).
