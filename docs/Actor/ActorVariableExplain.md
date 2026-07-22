# Actor Variable Explain

Glossary of **actor numeric / identity fields**: what each value means, where it
lives, and how formulas should treat it.

Use this file when:

- Recording balance meaning of a field
- Writing or reviewing attribute → resource / damage formulas
- Mapping `actorConfig` ↔ slim `SaveSlot.Actors[]` ↔ `BattleActorCom`

Related:

- `docs/Actor/BattleActorComponent.md` — ownership / authority
- `docs/Actor/BattleActorInitPaths.md` — Config vs Save init
- `docs/PlayerData/SaveSlotSchema.md` — exact Save keys
- `docs/Calculator/BattleCalculator.md` — battle math (reads converged numbers only)

Runtime authority on the live entity is always **`BattleActorCom`**.

Out of scope here: owned skills / inventory and other play systems also persist
in SaveSlot, but are not expanded in this document.

---

## Core Model

### What is persisted (player `Actors[]`) — slim only

`actorConfig` is a **design table** (full columns for monsters/NPCs and player
templates). Player Save does **not** mirror the whole Config row.

| Persist in `Actors[]` | Why |
|---|---|
| `configId` | Archetype / template link |
| `jobType` | Job |
| `level` | Level |
| `constitution` / `dexterity` / `intelligence` / `will` / `perception` | 五大屬性（升級加點） |
| `hp` / `mp` / `stamina` | **Current** resources only |

### What is NOT persisted (recomputed / system-driven at runtime)

| Not in Save | Why |
|---|---|
| `maxHp` / `maxMp` / `maxStamina` | Derived from five attributes (+ level / job); exact curves TBD |
| `baseDefense` / `speed` (and related) | Partly driven by attributes; full equations TBD |
| Attack / magic attack | **Not** from five attributes — skill / equip / buff (see below) |
| `totalAttack` / `totalDefense` | Runtime after equip / buff (`RecalculateStats`) |

```text
Save (slim)                          Runtime
configId, jobType, level      ->
five attributes               ->  PlayerDataLogic:BuildRuntimeActorState
hp, mp, stamina (current)     ->       | attribute-driven capacities / mobility / …
                                       v
                                 maxHp, maxMp, maxStamina, defense, speed, …
                                       v
                                 BattleActorCom:ImportSaveData(runtime)
                                 (+ equip/skill/buff for attack & totals)
```

**`BattleActorCom` never runs attribute formulas.** It only receives finished
runtime numbers from `PlayerDataLogic` (and later equip/buff pipelines).

### Monster / NPC

Hardcoded in `actorConfig`: **five attributes + all derived columns**
(`maxHp`, `defense`, `speed`, `jumpForce`, …). Loaded by `ApplyConfig`.
**No attack column** in Config (attack comes from skill/equip/buff at runtime,
same rule as players). No Save slim shape, no PlayerData formula.

### Attack = magic attack（不受五大屬性）

Design intent (formulas still TBD in code):

- **物理攻擊力**與**魔法攻擊力**視為同一套攻擊力（同一數值語意 / 同一來源管線）。
- 攻擊力**一般不受五大屬性影響**。
- 攻擊力主要來自：**技能 / 裝備 / buff**（以及後續職業或專武規則，若有）。

五大屬性負責生存、機動、施法輔助、回復抗性、命中爆擊等；**不要**把
`baseAttack` / `totalAttack` 寫成「體質 + 靈巧」這類屬性加總（舊 placeholder
已廢止）。

### Five primary attributes（五大屬性）— 簡介

Player: 升級加點，存入 Save。  
Monster/NPC: 僅 Config 寫死，無加點 UI。

| Key | 中文 | 影響（設計意圖） |
|---|---|---|
| `constitution` | 體質 | 防禦力、血量上限 |
| `dexterity` | 靈巧 | 速度、跳躍力、耐力 |
| `intelligence` | 智力 | 魔力上限、施法距離、魔力消耗量 |
| `will` | 意志 | 回復量、抗性、效果增幅 |
| `perception` | 感知 | 效果命中率、爆擊率 |

#### `constitution`（體質）

影響：

- 防禦力（defense）
- 血量上限（`maxHp`）

#### `dexterity`（靈巧）

影響：

- 速度（`speed` / 行動或移動相關）
- 跳躍力（jump；場上移動系統，細節 TBD）
- 耐力（`maxStamina` / stamina 相關）

#### `intelligence`（智力）

影響：

- 魔力上限（`maxMp`）
- 施法距離（cast / skill range 相關修正，細節 TBD）
- 魔力消耗量（MP cost 相關修正，細節 TBD）

#### `will`（意志）

影響：

- 回復量（heal / recover 相關）
- 抗性（status / element resistance 相關）
- 效果增幅（buff / effect potency 相關）

#### `perception`（感知）

影響：

- 效果命中率（effect hit / apply chance）
- 爆擊率（critical rate）

> 影響對照已同步到 `actorConfig` 衍伸欄、`BattleActorCom`、以及
> `PlayerDataLogic:ComputeActorDerivedFromAttributes`（係數仍為 placeholder）。

---

## Naming Canon (lowerCamelCase)

Same key spelling wherever a field appears. Do not mix `MaxHp` / `maxHp`.

| Layer | Contents |
|---|---|
| `actorConfig.csv` | Meta + five attrs + **derived** columns (no attack) |
| `Actors[]` Save | **Slim** only (no derived, no attack) |
| `BuildRuntimeActorState` | Slim + PlayerData-computed derived |
| `BattleActorCom` | Live: five attrs + derived + attack (from equip/skill/buff) |
| `ExportSaveData` | Slim only |
| `ExportSnapshot` | Runtime full (not Save) |

### `actorConfig` column groups

| Group | Columns |
|---|---|
| Meta | `configId`, `displayName`, `actorType`, `jobType`, `level`, `naturalSkillKeys`, `defaultEquipmentKeys`, `description`, `enabled` |
| Five attrs | `constitution`, `dexterity`, `intelligence`, `will`, `perception` |
| Derived | `maxHp`, `maxMp`, `maxStamina`, `defense`, `speed`, `jumpForce`, `castRange`, `mpCostRate`, `recoveryRate`, `resistance`, `effectPotency`, `effectHitRate`, `criticalRate` |
| **Removed** | `baseAttack` / attack — not attribute-derived |

---

## Naming Map

| Meaning | Key | Player Save | Player load compute | Monster Config | On Com |
|---|---|---|---|---|---|
| Archetype / job / level | `configId` / `jobType` / `level` | yes | — | yes | yes |
| Five attrs | `constitution`…`perception` | yes | — | yes | yes |
| Current resources | `hp` / `mp` / `stamina` | yes | clamp | seed = max | yes |
| Derived capacities / combat helpers | `maxHp`, `defense`, `speed`, … | **no** | **yes** (`PlayerDataLogic`) | hardcoded | yes |
| Attack (= magic attack) | `attack` / `totalAttack` | **no** | **no** | **no** | yes (equip/skill/buff) |
| Final defense | `totalDefense` | **no** | from `defense` + buff later | from Config `defense` | yes |

---

## Resources

- **Current** (`hp` / `mp` / `stamina`): persist for players; clamp after load.
- **Max**: player = attribute-driven formulas (TBD); monster = Config. Never
  store player max in Save.

---

## Attribute → derived keys

| Attribute | Derived keys on Config / Com |
|---|---|
| `constitution` | `defense`, `maxHp` |
| `dexterity` | `speed`, `jumpForce`, `maxStamina` |
| `intelligence` | `maxMp`, `castRange`, `mpCostRate` |
| `will` | `recoveryRate`, `resistance`, `effectPotency` |
| `perception` | `effectHitRate`, `criticalRate` |

Attack / magic attack: **skill / equipment / buff** only — never written by
`ComputeActorDerivedFromAttributes`, never a Config column.

Placeholder curves live in `PlayerDataLogic`; replace when balance locks.

---

## Formula ownership

| Concern | Owner |
|---|---|
| Slim Save read/write | `ExportSaveData` / SaveSlot |
| Attribute → capacities / mobility / resist / … (player) | `PlayerDataLogic` (TBD) |
| Attack / magic attack from equip/skill/buff | Equip / Skill / Effect systems (TBD) |
| Apply finished runtime to entity | `BattleActorCom:ImportSaveData` |
| Monster/NPC full load | `BattleActorCom:ApplyConfig` |
| Damage math | `BattleCalculatorLogic` |

---

## Quick examples

| `configId` | Role | Persist five attrs? | Persist maxHp? |
|---|---|---|---|
| `playerWarrior` | Player template + Save seed | yes (Save) | no (compute) |
| `slime` | Monster Config only | in Config only | in Config only |
