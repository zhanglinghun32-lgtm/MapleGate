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
| `jobs[]` | All jobs owned by the saved actor; each entry owns `jobType` and `level` |
| `activeJobIndex` | Selects which saved job becomes the runtime job |
| `constitution` / `dexterity` / `intelligence` / `will` / `perception` | 五大屬性（升級加點） |

### What is NOT persisted

| Not in Save | Why |
|---|---|
| `hp` / `mp` / `stamina` (**current**) | **Not remembered.** Each battle entry (and Create / fill into `BattleActorCom`) starts at full max (buffs may adjust). |
| `maxHp` / `maxMp` / `maxStamina` | Derived from five attributes (+ level / job); exact curves TBD |
| `baseDefense` / `speed` (and related) | Partly driven by attributes; full equations TBD |
| Cast-time `atk` | **Not an actor field** — `SkillActionWrapper` builds it per skill cast |
| `totalDefense` | Runtime after defense + buff (`RecalculateStats`) when needed |

```text
Save (slim)                          Runtime / BattleActorCom
configId, jobs[], activeJobIndex ->
five attributes               ->  PlayerDataLogic:BuildRuntimeActorState
                                       | attribute-driven maxHp / maxMp / maxStamina / …
                                       v
                                 hp/mp/stamina = max (full fill; buff TBD)
                                       v
                                 BattleActorCom:ImportSaveData(runtime)
                                 (Com still owns live current during battle)
                                       v
                                 SkillActionWrapper (on cast)
                                       five attrs + skillEffect / skill coeffs
                                       -> DamageRequest.atk  (cast-time only)
```

**Save never stores current resources.** `BattleActorCom` still has live
`hp` / `mp` / `stamina` for the session; Create / Import fills them to full
(max, optionally modified by buff).

**`BattleActorCom` never runs attribute→damage formulas and has no `atk`
field.** Capacities / mobility stay in `PlayerDataLogic`. Cast-time `atk` is
owned by `SkillActionWrapper`.

### Monster / NPC

Hardcoded in `actorConfig`: **five attributes + all derived columns**
(`maxHp`, `defense`, `speed`, `jumpForce`, …). Loaded by `ApplyConfig`, which
also seeds current resources to full max. **No attack / atk column** in Config.
No Save slim shape, no PlayerData formula.

### Cast-time `atk`（只在 Wrapper；Actor / Resolver 都沒有）

Design intent:

- Actor **沒有攻擊力**（無 `atk` / `baseAttack` / `totalAttack`）。
- **技能**提供基礎值與係數，並指定用哪個屬性，例如：

```text
atk = 50 + 0.5 * will
```

- 只有 `SkillActionWrapper` 在施放時計算 `atk`，寫入 `DamageRequest`。
- `BattleCalculatorLogic` 吃 `atk` 等純數值，輸出 `{ damage, ... }`。
- `SkillActionResolver` **只有 `damage`**（與 EffectRequest）；不讀 `atk`。

```text
skill base/coeff + attrs  --Wrapper-->  DamageRequest.atk
                          --Calculator-->  result.damage
                          --Resolver-->  ApplyDamage(damage)
```

五大屬性仍負責生存、機動、施法輔助、回復抗性、命中爆擊等；**不要**在
`BattleActorCom` 上快取全域攻擊力。

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
| `Actors[]` Save | **Slim** only: identity + `jobs[]` / `activeJobIndex` + five attrs (**no** current hp/mp/stamina) |
| `BuildRuntimeActorState` | Slim + derived + **current filled to full max** for Com import |
| `BattleActorCom` | Live: five attrs + derived + **session current** hp/mp/stamina; **no atk** |
| `ExportSaveData` | Slim only (drops current resources) |
| `ExportSnapshot` | Runtime full including current (not Save) |

### `actorConfig` column groups

| Group | Columns |
|---|---|
| Meta | `configId`, `displayName`, `actorType`, `jobType`, `level`, `naturalSkillKeys`, `defaultEquipmentKeys`, `description`, `enabled` |
| Five attrs | `constitution`, `dexterity`, `intelligence`, `will`, `perception` |
| Derived | `maxHp`, `maxMp`, `maxStamina`, `defense`, `speed`, `jumpForce`, `castRange`, `mpCostRate`, `recoveryRate`, `resistance`, `effectPotency`, `effectHitRate`, `criticalRate` |
| **Removed** | `baseAttack` / `atk` on actor — cast-time only via Wrapper |
| **Not in Save** | current `hp` / `mp` / `stamina` |

---

## Naming Map

| Meaning | Key | Player Save | Player load compute | Monster Config | On Com |
|---|---|---|---|---|---|
| Archetype / jobs / level | Save: `configId` / `jobs[]` / `activeJobIndex`; Config/runtime: `jobType` / `level` | yes | selects active job | one initial job | active job only |
| Five attrs | `constitution`…`perception` | yes | — | yes | yes |
| Current resources | `hp` / `mp` / `stamina` | **no** | fill **full** (= max, buff TBD) | seed = max | yes (session) |
| Derived capacities / combat helpers | `maxHp`, `defense`, `speed`, … | **no** | **yes** (`PlayerDataLogic`) | hardcoded | yes |
| Cast-time atk | `atk` on `DamageRequest` only | **no** | **no** (Wrapper on cast) | **no** | **no** |
| Final defense | `totalDefense` | **no** | from `defense` + buff later | from Config `defense` | yes |

---

## Resources

- **Current** (`hp` / `mp` / `stamina`): **not persisted**. On Create /
  `BuildRuntimeActorState` / battle entry fill, set to full `max*` (buffs may
  change the effective full value later). Live only on `BattleActorCom` during
  play / battle.
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

Cast-time `atk`: **`SkillActionWrapper` only** — never written by
`ComputeActorDerivedFromAttributes`, never a Config / Save / Com column.

Placeholder capacity curves live in `PlayerDataLogic`; replace when balance locks.

---

## Formula ownership

| Concern | Owner |
|---|---|
| Slim Save read/write (no currents) | `ExportSaveData` / SaveSlot |
| Attribute → capacities / mobility / resist / … (player) | `PlayerDataLogic` (TBD) |
| Fill current = full for Com Create / Import | `PlayerDataLogic:BuildRuntimeActorState` |
| Fill current = full on battle entry (player/ally) | `BattleFlowLogic:FillRosterResourcesToFull` → `BattleActorCom:FillResourcesToFull` |
| Skill base + coeff × attr → cast-time `atk` | `SkillActionWrapper` only |
| `atk` → `damage` (numbers only) | `BattleCalculatorLogic` |
| Apply `damage` / effects | `SkillActionResolver` (no `atk`) |
| Apply finished runtime to entity | `BattleActorCom:ImportSaveData` |
| Monster/NPC full load | `BattleActorCom:ApplyConfig` |

---

## Quick examples

| `configId` | Role | Persist five attrs? | Persist current hp? | Persist maxHp? |
|---|---|---|---|---|
| `playerWarrior` | Player template + Save seed | yes (Save) | **no** | no (compute) |
| `slime` | Monster Config only | in Config only | **no** (seed full on ApplyConfig) | in Config only |
