# AzerothCore Skills 模块详细分析

## 1. 模块概览

Skills 模块实现了魔兽世界中专业技能（Professional Skill）的三个核心扩展系统：**配方发现**（Skill Discovery）、**额外物品制造**（Extra Items）和**完美物品制造**（Perfect Items）。这三个子系统均为纯数据驱动，通过数据库表配置行为，运行时以内存映射表提供服务查询。模块不包含类定义，采用函数式接口 + 静态存储的设计。

**文件结构：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `SkillDiscovery.h` | 30 | 配方发现系统接口声明 |
| `SkillDiscovery.cpp` | 253 | 配方发现系统实现：数据加载、概率发现、显式发现、全部发现检查 |
| `SkillExtraItems.h` | 34 | 额外物品/完美物品系统接口声明 |
| `SkillExtraItems.cpp` | 250 | 额外物品/完美物品系统实现：数据加载、专精检测、概率判定 |

**模块位置：** `src/server/game/Skills/`

---

## 2. 子系统概览

Skills 模块包含三个逻辑独立的子系统，各自管理不同的专业机制：

```
┌──────────────────────────────────────────────────────────────┐
│                    Skills 模块                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────┐  ┌────────────────────┐             │
│  │  Skill Discovery   │  │  Skill Extra Items  │             │
│  │  (配方发现系统)     │  │  (制造增益系统)      │             │
│  │                    │  │                    │             │
│  │  - 机制发现        │  │  - 额外物品        │             │
│  │    (Alchemy)       │  │    (专精奖励)       │             │
│  │  - 显式发现        │  │  - 完美物品        │             │
│  │    (Research/Book) │  │    (珠宝完美切工)   │             │
│  └────────────────────┘  └────────────────────┘             │
│                                                              │
│  数据库表:                                                    │
│  skill_discovery_template  skill_extra_item_template          │
│                             skill_perfect_item_template        │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 子系统一：配方发现（Skill Discovery）

### 3.1 功能说明

配方发现系统控制专业技能中"领悟新配方"的机制。当玩家制作物品时，有概率随机发现一个尚未学会的配方。此系统支持两种发现模式：

| 模式 | 触发方式 | 典型应用 |
|------|----------|----------|
| **机制发现** (Mechanic Discovery) | 制作时自动触发（法术 Mechanic=DISCOVERY） | 炼金术：制作药水/药剂时有概率领悟新配方 |
| **显式发现** (Explicit Discovery) | 玩家主动使用发现技能 | 铭文研究（Northrend Inscription Research）、雕文精通之书 |

### 3.2 数据结构

```
SkillDiscoveryEntry
├── spellId: uint32          // 可被发现的法术 ID
├── reqSkillValue: uint32    // 所需技能等级
└── chance: float            // 发现概率（0~100）

SkillDiscoveryStore: unordered_map<int32, list<SkillDiscoveryEntry>>
    ├── key > 0 → 法术 ID（显式发现/机制发现，按触发法术索引）
    └── key < 0 → 技能线 ID 取反（机制发现，按技能线索引）
```

**key 编码规则：**
- `reqSpell > 0`：以触发法术 ID 为键，用于机制发现和显式发现
- `reqSpell == 0`：以技能线 ID 取反为键（`-int32(SkillLine)`），用于技能线关联的机制发现
- `reqSpell < 0`：非法值，加载时跳过

### 3.3 数据加载（`LoadSkillDiscoveryTable()` — `SkillDiscovery.cpp:46-156`）

```
LoadSkillDiscoveryTable()
    │
    ├── 查询: SELECT spellId, reqSpell, reqSkillValue, chance FROM skill_discovery_template
    │
    ├── 遍历每条记录:
    │   ├── if (chance <= 0) → 跳过并记录错误
    │   │
    │   ├── if (reqSpell > 0) → 法术模式:
    │   │   ├── 验证 reqSpell 法术存在
    │   │   ├── 验证 reqSpell 法术 Mechanic==DISCOVERY 或 IsExplicitDiscovery()
    │   │   └── SkillDiscoveryStore[reqSpell].emplace_back(spellId, reqSkillValue, chance)
    │   │
    │   ├── if (reqSpell == 0) → 技能线模式:
    │   │   ├── 查找 spellId 在 SkillLineAbility.dbc 中的技能线
    │   │   └── 对每个关联的技能线:
    │   │       SkillDiscoveryStore[-int32(SkillLine)].emplace_back(...)
    │   │
    │   └── if (reqSpell < 0) → 记录错误并跳过
    │
    └── 后置检查: 遍历所有法术，报告缺少数据的显式发现法术
```

### 3.4 机制发现（`GetSkillDiscoverySpell()` — `SkillDiscovery.cpp:213-252`）

在玩家制作物品时，由 `Player::UpdateCraftSkill()` 调用（`PlayerUpdates.cpp:840`）：

```
GetSkillDiscoverySpell(skillId, spellId, player)
    │
    ├── 获取玩家技能等级: skillvalue = player->GetSkillValue(skillId)
    │
    ├── 第一步：按法术 ID 查找（reqSpell 模式）
    │   tab = SkillDiscoveryStore.find(int32(spellId))
    │   if (找到):
    │       遍历列表中每个条目:
    │           if (roll_chance_f(chance * RATE_SKILL_DISCOVERY)  // 受世界倍率影响
    │               && reqSkillValue <= skillvalue                 // 技能等级满足
    │               && !player->HasSpell(spellId))               // 尚未学会
    │               return spellId
    │       return 0  // 未发现
    │
    ├── 第二步：按技能线查找（reqSkill 模式）
    │   if (!skillId) → return 0
    │   tab = SkillDiscoveryStore.find(-(int32)skillId)
    │   if (找到):
    │       同上逻辑遍历并判定
    │
    └── return 0  // 无匹配数据
```

**调用链：**
```
Player::UpdateCraftSkill()                         // PlayerUpdates.cpp:840
    │
    ├── if (spellInfo->Mechanic == MECHANIC_DISCOVERY)
    │   └── GetSkillDiscoverySpell(SkillLine, spellId, this)
    │       └── return discoveredSpellId
    │
    └── if (discoveredSpellId)
        └── learnSpell(discoveredSpellId)          // 玩家学会新配方
```

### 3.5 显式发现（`GetExplicitDiscoverySpell()` — `SkillDiscovery.cpp:158-198`）

玩家主动使用发现技能时调用（如铭文研究、雕文精通之书），由脚本 `spell_gen_profession_research` 和 `spell_item_book_of_glyph_mastery` 触发：

```
GetExplicitDiscoverySpell(spellId, player)
    │
    ├── tab = SkillDiscoveryStore.find(int32(spellId))
    │   if (未找到) → return 0
    │
    ├── 获取玩家技能等级:
    │   skillvalue = player->GetSkillValue(SkillLine)
    │
    ├── 计算有效概率:
    │   full_chance = Σ(所有满足条件的条目的 chance)
    │   (条件: reqSkillValue <= skillvalue && !player->HasSpell(spellId))
    │
    ├── 加权随机选择:
    │   rate = full_chance / 100.0f
    │   roll = rand_chance() * rate     // roll 在 0..full_chance 范围内
    │   遍历条目:
    │       if (chance > roll) → 选中此条目
    │           if (spellId != 64323)  // 排除雕文精通之书
    │               player->UpdateGatherSkill(INSCRIPTION, ...)  // 更新铭文技能
    │           return spellId
    │       roll -= chance
    │
    └── return 0  // 本次未发现
```

**关键差异：**
- 显式发现**必定成功**（只要还有未发现的配方且技能等级满足），概率仅决定发现哪一个
- 机制发现**概率成功**，每次制作独立判定

### 3.6 全部发现检查（`HasDiscoveredAllSpells()` — `SkillDiscovery.cpp:200-211`）

```
HasDiscoveredAllSpells(spellId, player)
    │
    ├── tab = SkillDiscoveryStore.find(int32(spellId))
    │   if (未找到) → return true  (没有可发现的内容)
    │
    └── 遍历所有条目:
        if (!player->HasSpell(spellId)) → return false
        return true  // 玩家已学会所有可发现的法术
```

**用途：** 在施放显式发现技能前检查，若已全部发现则阻止施放并返回错误消息。

---

## 4. 子系统二：额外物品制造（Skill Extra Items）

### 4.1 功能说明

额外物品制造系统实现了专业技能的"专精奖励"机制。当玩家拥有特定专精法术时，制作物品有概率额外产出更多数量的成品。这是经典旧世到巫妖王之怒中炼金/锻造/制皮/裁缝等专业的核心机制。

**典型场景：**
- 炼金术：药水大师制作药水时，有概率产出 1~4 瓶额外药水
- 药剂大师制作药剂时，有概率产出额外药剂
- 元素大师转化元素时，有概率产出额外材料

### 4.2 数据结构

```
SkillExtraItemEntry
├── requiredSpecialization: uint32    // 所需专精法术 ID
├── additionalCreateChance: float     // 每次额外产出的概率
└── newMaxOrEntry: int32              // 额外物品最大数量

SkillExtraItemStore: map<uint32, SkillExtraItemEntry>
    └── key = 制造法术 ID (spellId)
```

### 4.3 数据加载（`LoadSkillExtraItemTable()` — `SkillExtraItems.cpp:138-200`）

```
LoadSkillExtraItemTable()
    │
    ├── 查询: SELECT spellId, requiredSpecialization, additionalCreateChance, additionalMaxNum
    │         FROM skill_extra_item_template
    │
    ├── 遍历每条记录:
    │   ├── 验证 spellId 法术存在
    │   ├── 验证 requiredSpecialization 法术存在
    │   ├── 验证 additionalCreateChance > 0
    │   ├── 验证 newMaxOrEntry != 0
    │   └── 存入 SkillExtraItemStore[spellId]
    │
    └── 日志: "Loaded N spell specialization definitions"
```

### 4.4 额外物品判定（`canCreateExtraItems()` — `SkillExtraItems.cpp:226-249`）

```
canCreateExtraItems(player, spellId, additionalChance, newMaxOrEntry)
    │
    ├── ret = SkillExtraItemStore.find(spellId)
    │   if (未找到) → return false
    │
    ├── if (!player->HasSpell(requiredSpecialization))
    │   → return false  // 没有所需专精
    │
    ├── 设置输出参数:
    │   additionalChance = specEntry->additionalCreateChance
    │   newMaxOrEntry = specEntry->newMaxOrEntry
    │
    └── return true  // 可以产生额外物品
```

### 4.5 制造流程中的额外物品判定（`SpellEffects.cpp:1718-1733`）

```
// SpellEffects.cpp: EffectCreateItem 处理中
int32 itemsCount = 1;
float additionalCreateChance = 0.0f;
int32 additionalMaxNum = 0;

if (canCreateExtraItems(player, m_spellInfo->Id, additionalCreateChance, additionalMaxNum))
{
    // 循环掷骰，直到失败或达到最大额外数
    while (roll_chance_f(additionalCreateChance) && itemsCount <= additionalMaxNum)
        ++itemsCount;
}

addNumber *= itemsCount;  // 最终制造数量 = 基础数量 × 额外倍率
```

**示例：** 炼金药水大师（requiredSpecialization=28677）制作法术 28550（法术ID）：
- additionalCreateChance = 14%
- additionalMaxNum = 4
- 最多产出 1（基础）+ 4（额外）= 5 瓶药水

---

## 5. 子系统三：完美物品制造（Skill Perfect Items）

### 5.1 功能说明

完美物品制造系统实现了珠宝加工的"完美切工"机制。当珠宝匠拥有特定专精时，切割宝石有概率产出完美品质的宝石（属性更高的版本）。与额外物品不同，完美物品**替换**原物品，而非额外产出。

### 5.2 数据结构

```
SkillPerfectItemEntry
├── requiredSpecialization: uint32    // 所需专精法术 ID
├── perfectCreateChance: float       // 完美产出概率
└── perfectItemType: uint32          // 完美物品的 Item ID

SkillPerfectItemStore: map<uint32, SkillPerfectItemEntry>
    └── key = 制造法术 ID (spellId)
```

### 5.3 数据加载（`LoadSkillPerfectItemTable()` — `SkillExtraItems.cpp:52-114`）

```
LoadSkillPerfectItemTable()
    │
    ├── 查询: SELECT spellId, requiredSpecialization, perfectCreateChance, perfectItemType
    │         FROM skill_perfect_item_template
    │
    ├── 遍历每条记录:
    │   ├── 验证 spellId 法术存在
    │   ├── 验证 requiredSpecialization 法术存在
    │   ├── 验证 perfectCreateChance > 0
    │   ├── 验证 perfectItemType 物品模板存在
    │   └── 存入 SkillPerfectItemStore[spellId]
    │
    └── 日志: "Loaded N spell perfection definitions"
```

### 5.4 完美物品判定（`CanCreatePerfectItem()` — `SkillExtraItems.cpp:202-224`）

```
CanCreatePerfectItem(player, spellId, perfectCreateChance, perfectItemType)
    │
    ├── ret = SkillPerfectItemStore.find(spellId)
    │   if (未找到) → return false
    │
    ├── if (!player->HasSpell(requiredSpecialization))
    │   → return false  // 没有所需专精
    │
    ├── 设置输出参数:
    │   perfectCreateChance = thisEntry->perfectCreateChance
    │   perfectItemType = thisEntry->perfectItemType
    │
    └── return true  // 可以产生完美物品
```

### 5.5 制造流程中的完美物品判定（`SpellEffects.cpp:1703-1714`）

```
// SpellEffects.cpp: EffectCreateItem 处理中（在额外物品之前）
float perfectCreateChance = 0.0f;
uint32 perfectItemType = itemId;  // 默认为普通物品

if (CanCreatePerfectItem(player, m_spellInfo->Id, perfectCreateChance, perfectItemType))
    if (roll_chance_f(perfectCreateChance))
        newitemid = perfectItemType;  // 完美物品替换普通物品

// 然后进入额外物品判定...
```

---

## 6. 数据库表

所有 Skills 相关表存储在 **acore_world** 数据库中。

### 6.1 skill_discovery_template（配方发现模板表）

```sql
CREATE TABLE `skill_discovery_template` (
  `spellId` int unsigned NOT NULL DEFAULT '0' COMMENT 'SpellId of the discoverable spell',
  `reqSpell` int unsigned NOT NULL DEFAULT '0' COMMENT 'spell requirement',
  `reqSkillValue` smallint unsigned NOT NULL DEFAULT '0' COMMENT 'skill points requirement',
  `chance` float NOT NULL DEFAULT '0' COMMENT 'chance to discover',
  PRIMARY KEY (`spellId`,`reqSpell`)
) ENGINE=InnoDB COMMENT='Skill Discovery System';
```

| 列 | 类型 | 含义 |
|----|------|------|
| `spellId` | int unsigned | 可被发现的法术 ID（新配方） |
| `reqSpell` | int unsigned | 触发条件法术 ID（0=按技能线匹配，>0=按法术匹配） |
| `reqSkillValue` | smallint unsigned | 所需技能等级（0=无限制） |
| `chance` | float | 发现概率百分比（0~100） |

**索引：** 主键 `(spellId, reqSpell)`

**数据特征分析：**

| reqSpell 值 | 含义 | 数据量 | 示例 |
|-------------|------|--------|------|
| 0 | 炼金机制发现（按技能线） | 12 条 | spellId=28580~28591, chance=0.1% |
| 28571~28576 | 炼金专精发现 | 5 条 | 炼金大师分支配方 |
| 60350 | 炼金北裂发现 | 12 条 | 巫妖王炼金新配方, chance=50% |
| 60893 | 炼金研究 | 8 条 | 北裂炼金研究, chance=100%, reqSkillValue=400 |
| 61177 | 铭文北裂研究 | ~70 条 | 北裂铭文研究, chance=100%, reqSkillValue=385 |
| 61288 | 铭文小研究 | ~30 条 | 铭文小研究, chance=100%, reqSkillValue=75~350 |
| 61756 | 铭文研究（无技能要求） | ~70 条 | 与 61177 配对, chance=100%, reqSkillValue=0 |
| 64323 | 雕文精通之书 | ~40 条 | 使用物品发现, chance=100% |

### 6.2 skill_extra_item_template（额外物品模板表）

```sql
CREATE TABLE `skill_extra_item_template` (
  `spellId` int unsigned NOT NULL DEFAULT '0' COMMENT 'SpellId of the item creation spell',
  `requiredSpecialization` int unsigned NOT NULL DEFAULT '0' COMMENT 'Specialization spell id',
  `additionalCreateChance` float NOT NULL DEFAULT '0' COMMENT 'chance to create add',
  `additionalMaxNum` tinyint NOT NULL DEFAULT '0',
  PRIMARY KEY (`spellId`)
) ENGINE=InnoDB COMMENT='Skill Specialization System';
```

| 列 | 类型 | 含义 |
|----|------|------|
| `spellId` | int unsigned | 制造法术 ID |
| `requiredSpecialization` | int unsigned | 所需专精法术 ID |
| `additionalCreateChance` | float | 每次额外产出的概率百分比 |
| `additionalMaxNum` | tinyint | 最大额外产出数量 |

**索引：** 主键 `(spellId)`

**专精法术 ID 对照：**

| 专精法术 ID | 名称 | 职业/专业 | additionalCreateChance | additionalMaxNum |
|------------|------|----------|----------------------|-----------------|
| 28675 | 药剂大师 (Elixir Master) | 炼金 | 14~16% | 4 |
| 28677 | 药水大师 (Potion Master) | 炼金 | 14~17% | 4~8 |
| 28672 | 转化大师 (Transmutation Master) | 炼金 | 16% | 3~5 |
| 26797 | 大地之血转化 | 炼金（特殊转化） | 100% | 1 |
| 26798 | 空气转化 | 炼金（特殊转化） | 100% | 1 |
| 26801 | 水之转化 | 炼金（特殊转化） | 100% | 1 |

### 6.3 skill_perfect_item_template（完美物品模板表）

```sql
CREATE TABLE `skill_perfect_item_template` (
  `spellId` int unsigned NOT NULL DEFAULT '0' COMMENT 'SpellId of the item creation spell',
  `requiredSpecialization` int unsigned NOT NULL DEFAULT '0' COMMENT 'Specialization spell id',
  `perfectCreateChance` float NOT NULL DEFAULT '0' COMMENT 'chance to create the perfect item instead',
  `perfectItemType` int unsigned NOT NULL DEFAULT '0' COMMENT 'perfect item type to create instead',
  PRIMARY KEY (`spellId`)
) ENGINE=InnoDB COMMENT='Crafting Perfection System';
```

| 列 | 类型 | 含义 |
|----|------|------|
| `spellId` | int unsigned | 制造法术 ID（宝石切割） |
| `requiredSpecialization` | int unsigned | 所需专精法术 ID |
| `perfectCreateChance` | float | 完美产出概率百分比 |
| `perfectItemType` | int unsigned | 完美品质物品 ID |

**索引：** 主键 `(spellId)`

**数据特征：**
- 所有 70 条记录的 `requiredSpecialization = 55534`（珠宝加工专精）
- 所有记录的 `perfectCreateChance = 20`（20% 概率）
- `spellId` 范围：53831~54017（巫妖王之怒宝石切割法术）
- `perfectItemType` 范围：41429~41502（完美品质宝石物品 ID）

### 6.4 表间关系图

```
┌───────────────────────────────────────────────────────────────────────────┐
│                           acore_world 数据库                              │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────┐                                          │
│  │  skill_discovery_template   │  ← 配方发现系统                           │
│  │  (spellId, reqSpell,        │                                          │
│  │   reqSkillValue, chance)    │     reqSpell > 0 ──> spell.dbc 法术       │
│  └─────────────────────────────┘     reqSpell = 0 ──> SkillLineAbility.dbc│
│                                                                           │
│  ┌─────────────────────────────┐                                          │
│  │  skill_extra_item_template  │  ← 额外物品系统                           │
│  │  (spellId, requiredSpec,    │                                          │
│  │   additionalChance, maxNum) │     requiredSpec ──> spell.dbc 专精法术   │
│  └─────────────────────────────┘     spellId ──────> spell.dbc 制造法术   │
│                                                                           │
│  ┌─────────────────────────────┐                                          │
│  │ skill_perfect_item_template │  ← 完美物品系统                           │
│  │  (spellId, requiredSpec,    │                                          │
│  │   perfectChance, itemtype) │     requiredSpec ──> spell.dbc 专精法术   │
│  └─────────────────────────────┘     perfectItemType ──> item_template    │
│                                                           (物品模板表)    │
└───────────────────────────────────────────────────────────────────────────┘
```

### 6.5 各表加载顺序

| 步骤 | 表名 | 加载函数 | 依赖 |
|------|------|----------|------|
| 1 | `skill_discovery_template` | `LoadSkillDiscoveryTable()` | spell.dbc, SkillLineAbility.dbc |
| 2 | `skill_extra_item_template` | `LoadSkillExtraItemTable()` | spell.dbc |
| 3 | `skill_perfect_item_template` | `LoadSkillPerfectItemTable()` | spell.dbc, item_template |

所有加载在 `World::SetInitialWorldSettings()` 中按顺序执行（`World.cpp:718-724`）。

---

## 7. 制造流程中的完整调用链

```
玩家施放制造法术
    │
    ▼
Spell::EffectCreateItem()                           // SpellEffects.cpp:~1700
    │
    ├── [完美物品判定] ─────────────────────────────────────────
    │   CanCreatePerfectItem(player, spellId, chance, itemType)
    │   if (返回 true):
    │       if (roll_chance_f(chance))
    │           newitemid = itemType     // 完美物品替换原物品
    │   ─────────────────────────────────────────────────────────
    │
    ├── [额外物品判定] ─────────────────────────────────────────
    │   canCreateExtraItems(player, spellId, chance, maxNum)
    │   if (返回 true):
    │       itemsCount = 1
    │       while (roll_chance_f(chance) && itemsCount <= maxNum)
    │           ++itemsCount
    │       addNumber *= itemsCount     // 数量乘以额外倍率
    │   ─────────────────────────────────────────────────────────
    │
    ├── 执行物品创建: player->StoreNewItem(dest, newitemid, addNumber)
    └── ...

玩家制作完成后:
Player::UpdateCraftSkill()                          // PlayerUpdates.cpp:~840
    │
    ├── [机制发现判定] ─────────────────────────────────────────
    │   if (spellInfo->Mechanic == MECHANIC_DISCOVERY):
    │       discoveredSpell = GetSkillDiscoverySpell(SkillLine, spellId, player)
    │       if (discoveredSpell):
    │           learnSpell(discoveredSpell)  // 学会发现的新配方
    │   ─────────────────────────────────────────────────────────
    │
    └── 更新专业技能等级

玩家主动使用发现技能:
spell_gen_profession_research / spell_item_book_of_glyph_mastery
    │
    ├── CheckRequirement():
    │   if (HasDiscoveredAllSpells(spellId, player))
    │       → SPELL_FAILED_CUSTOM_ERROR  // 阻止施放
    │
    └── HandleScript():
        discoveredSpell = GetExplicitDiscoverySpell(spellId, player)
        if (discoveredSpell):
            learnSpell(discoveredSpell)
        UpdateCraftSkill(spellId)
```

---

## 8. 配置项汇总

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `Rate.Skill.Discovery` | 1.0 | 机制发现概率倍率（仅影响 `GetSkillDiscoverySpell`，不影响显式发现） |

**注意：** 显式发现（`GetExplicitDiscoverySpell`）不受 `Rate.Skill.Discovery` 影响，因为显式发现的设计是"只要还有未发现的配方就必定成功"。倍率仅应用于制作时的随机机制发现。

---

## 9. 数据验证与错误处理

### 9.1 skill_discovery_template 验证

| 检查项 | 行为 |
|--------|------|
| `chance <= 0` | 跳过，记录到错误流 |
| `reqSpell > 0` 且 reqSpell 法术不存在 | 跳过，LOG_ERROR（每个 reqSpell 仅报告一次） |
| `reqSpell > 0` 但法术既非 DISCOVERY 也非 ExplicitDiscovery | 跳过，LOG_ERROR |
| `reqSpell == 0` 但 spellId 不在 SkillLineAbility.dbc | 跳过，LOG_ERROR |
| `reqSpell < 0` | 跳过，LOG_ERROR |
| 显式发现法术在表中无对应数据 | LOG_ERROR（后置检查） |

### 9.2 skill_extra_item_template 验证

| 检查项 | 行为 |
|--------|------|
| spellId 法术不存在 | 跳过，LOG_ERROR |
| requiredSpecialization 法术不存在 | 跳过，LOG_ERROR |
| additionalCreateChance <= 0 | 跳过，LOG_ERROR |
| additionalMaxNum == 0 | 跳过，LOG_ERROR |

### 9.3 skill_perfect_item_template 验证

| 检查项 | 行为 |
|--------|------|
| spellId 法术不存在 | 跳过，LOG_ERROR |
| requiredSpecialization 法术不存在 | 跳过，LOG_ERROR |
| perfectCreateChance <= 0 | 跳过，LOG_ERROR |
| perfectItemType 物品模板不存在 | 跳过，LOG_ERROR |

---

## 10. 关键设计特征

### 10.1 纯数据驱动

三个子系统均完全由数据库表驱动，核心逻辑不包含任何硬编码的专业/法术 ID。添加新的发现配方、额外物品或完美物品只需修改数据库，无需修改代码。

### 10.2 双索引发现机制

SkillDiscoveryStore 使用正/负键的双索引设计：
- 正键（法术 ID）：精确匹配触发法术，用于显式发现和按法术触发的机制发现
- 负键（-技能线 ID）：按技能线匹配，用于"制作任意该技能线物品都有概率发现"的机制

这种设计使得同一个可发现配方可以同时被多个触发源关联。

### 10.3 专精检查前置

额外物品和完美物品系统都采用"先检查专精再返回参数"的模式。调用者先通过 `canCreateExtraItems` / `CanCreatePerfectItem` 获取概率和参数，然后自行掷骰。这种设计让调用者（SpellEffects）拥有完整的控制权，可以在制造流程中灵活组合两种效果。

### 10.4 完美物品与额外物品的互斥优先级

在 `SpellEffects.cpp` 的制造流程中：
1. **先判定完美物品**（替换原物品类型）
2. **再判定额外物品**（增加产出数量）

这意味着完美物品可以和额外物品同时生效：先替换为完美品质，再计算额外数量。

### 10.5 显式发现的保底机制

`GetExplicitDiscoverySpell` 的概率算法确保只要还有未发现的配方，就一定会发现一个：
- 计算所有未发现配方的概率总和 `full_chance`
- 将随机 roll 缩放到 `0..full_chance` 范围内
- 遍历列表时用递减 roll 选择，保证必选中一个

与之对比，`HasDiscoveredAllSpells` 在施放前检查，避免玩家浪费材料在无意义的发现上。

### 10.6 雕文精通之书的特殊处理

在 `GetExplicitDiscoverySpell` 中，当 `spellId == 64323`（雕文精通之书）时，跳过技能更新调用：
```cpp
if (spellId != 64323)
    player->UpdateGatherSkill(SKILL_INSCRIPTION, ...);
```
这是因为雕文精通之书是消耗品物品，使用它不应该增加铭文技能等级（与铭文研究法术不同）。

---

## 11. 数据流总览

```
[服务器启动]
    World::SetInitialWorldSettings()
        ├── LoadSkillDiscoveryTable()
        │   └── SELECT FROM skill_discovery_template → SkillDiscoveryStore
        ├── LoadSkillExtraItemTable()
        │   └── SELECT FROM skill_extra_item_template → SkillExtraItemStore
        └── LoadSkillPerfectItemTable()
            └── SELECT FROM skill_perfect_item_template → SkillPerfectItemStore

[玩家制作物品]
    Spell::EffectCreateItem()
        ├── CanCreatePerfectItem() → SkillPerfectItemStore 查询
        │   └── if 通过: roll_chance → 替换为完美物品
        ├── canCreateExtraItems() → SkillExtraItemStore 查询
        │   └── if 通过: 循环 roll_chance → 增加产出数量
        └── 创建物品

[玩家制作完成 - 技能更新]
    Player::UpdateCraftSkill()
        └── if MECHANIC_DISCOVERY:
            └── GetSkillDiscoverySpell() → SkillDiscoveryStore 查询
                └── if 发现: learnSpell(discoveredSpell)

[玩家使用发现技能/物品]
    spell_gen_profession_research / spell_item_book_of_glyph_mastery
        ├── CheckRequirement: HasDiscoveredAllSpells() → SkillDiscoveryStore 查询
        │   └── if 全部发现 → SPELL_FAILED_CUSTOM_ERROR
        └── HandleScript: GetExplicitDiscoverySpell() → SkillDiscoveryStore 查询
            └── 加权随机选择 → learnSpell(discoveredSpell)
```
