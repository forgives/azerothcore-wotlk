# Spells 法术系统 - 完整分析

> AzerothCore WotLK 3.3.5a
> 模块路径: `src/server/game/Spells/`
> 总代码量: ~43,781 行 (16 个文件)

---

## 目录

1. [模块概览](#1-模块概览)
2. [文件结构与职责](#2-文件结构与职责)
3. [核心类体系](#3-核心类体系)
4. [Spell 类 - 法术施放核心](#4-spell-类---法术施放核心)
5. [SpellInfo 类 - 法术原型数据](#5-spellinfo-类---法术原型数据)
6. [SpellEffectInfo - 法术效果信息](#6-spelleffectinfo---法术效果信息)
7. [法术效果系统 (SpellEffects)](#7-法术效果系统-spelleffects)
8. [法术目标选择系统](#8-法术目标选择系统)
9. [光环系统 (Auras)](#9-光环系统-auras)
10. [AuraType 完整枚举](#10-auratype-完整枚举)
11. [SpellScript 脚本系统](#11-spellscript-脚本系统)
12. [AuraScript 光环脚本系统](#12-aurascript-光环脚本系统)
13. [SpellMgr 法术管理器](#13-spellmgr-法术管理器)
14. [SpellInfoCorrections 法术修正](#14-spellinfocorrections-法术修正)
15. [数据库依赖](#15-数据库依赖)
16. [DBC 客户端数据依赖](#16-dbc-客户端数据依赖)
17. [关键枚举与常量定义](#17-关键枚举与常量定义)
18. [施法生命周期](#18-施法生命周期)
19. [Proc 触发系统](#19-proc-触发系统)
20. [与其他模块的关系](#20-与其他模块的关系)

---

## 1. 模块概览

Spells 模块是 AzerothCore 游戏服务器中最庞大、最核心的系统之一，负责处理游戏中所有法术的施放、效果执行、光环管理、目标选择、伤害计算和脚本扩展。

### 核心职责

| 职责 | 描述 | 核心文件 |
|------|------|----------|
| 法术施放 | 管理从开始施法到完成的完整生命周期 | `Spell.cpp/h` |
| 法术原型 | 存储法术的静态属性数据 (从 DBC 加载) | `SpellInfo.cpp/h` |
| 法术效果 | 实现 100+ 种法术效果的执行逻辑 | `SpellEffects.cpp` |
| 光环系统 | 管理持续性的法术效果 (增益/减益) | `Auras/SpellAuras.cpp/h` |
| 光环效果 | 实现近 300 种光环类型的处理器 | `Auras/SpellAuraEffects.cpp/h` |
| 法术管理 | 全局法术注册表、查询和初始化 | `SpellMgr.cpp/h` |
| 脚本系统 | 提供可编程的钩子系统扩展法术行为 | `SpellScript.cpp/h` |
| 法术修正 | 对 DBC 数据进行运行时修正 | `SpellInfoCorrections.cpp` |

### 文件规模统计

| 文件 | 行数 | 大小 | 职责 |
|------|------|------|------|
| `Spell.cpp` | 9,164 | 378KB | 法术施放核心实现 |
| `Auras/SpellAuraEffects.cpp` | 7,076 | 307KB | 光环效果实现 |
| `SpellEffects.cpp` | 6,365 | 245KB | 法术效果实现 |
| `SpellInfoCorrections.cpp` | 5,373 | 185KB | 法术数据修正 |
| `SpellMgr.cpp` | 3,782 | 146KB | 法术管理器 |
| `SpellInfo.cpp` | 3,281 | 147KB | 法术原型实现 |
| `Auras/SpellAuras.cpp` | 2,954 | 118KB | 光环系统实现 |
| `SpellScript.cpp` | 1,213 | 45KB | 脚本系统实现 |
| `SpellScript.h` | 964 | 49KB | 脚本系统定义 |
| `Spell.h` | 875 | 36KB | 法术类定义 |
| `SpellMgr.h` | 838 | 38KB | 法术管理器定义 |
| `SpellInfo.h` | 585 | 23KB | 法术原型定义 |
| `Auras/SpellAuraDefines.h` | 399 | 26KB | 光环枚举定义 |
| `SpellDefines.h` | 184 | 9KB | 法术枚举定义 |
| `Auras/SpellAuraEffects.h` | 405 | 23KB | 光环效果定义 |
| `Auras/SpellAuras.h` | 323 | 17KB | 光环类定义 |

---

## 2. 文件结构与职责

```
src/server/game/Spells/
├── Spell.h                   # Spell 类及相关辅助类型定义
├── Spell.cpp                 # 法术施放全流程实现 (核心)
├── SpellInfo.h               # SpellInfo 类、SpellEffectInfo、目标类型枚举
├── SpellInfo.cpp             # 法术原型数据计算和验证逻辑
├── SpellInfoCorrections.cpp  # 运行时修正 DBC 中的错误数据
├── SpellDefines.h            # 中断标志、施法修饰符、触发标志等枚举
├── SpellEffects.cpp          # 100+ 种法术效果的具体实现
├── SpellMgr.h                # SpellMgr 法术管理器定义、Proc 类型
├── SpellMgr.cpp              # 全局法术加载、数据库读取、查询接口
├── SpellScript.h             # SpellScript/AuraScript 脚本框架定义
├── SpellScript.cpp           # 脚本框架实现
└── Auras/                    # 光环子系统
    ├── SpellAuraDefines.h    # AuraType 枚举 (~300种)、处理模式
    ├── SpellAuras.h          # Aura 类定义
    ├── SpellAuras.cpp        # 光环应用/移除/叠加逻辑
    ├── SpellAuraEffects.h    # AuraEffect 类定义
    └── SpellAuraEffects.cpp  # 每种光环效果的处理实现
```

---

## 3. 核心类体系

```
SpellInfo (不可变原型数据, 从 DBC 构建)
    ├── SpellEffectInfo[3] (每个法术最多3个效果)
    │   ├── SpellImplicitTargetInfo (目标信息)
    │   └── SpellRadiusEntry* (半径引用)
    ├── SpellChainNode* (法术等级链)
    ├── SpellDurationEntry* (持续时间引用)
    ├── SpellRangeEntry* (范围引用)
    └── SpellCategoryEntry* (分类引用)

Spell (可变的法术施放实例)
    ├── SpellCastTargets (目标数据)
    ├── SpellValue* (运行时数值覆盖)
    ├── m_uniqueTargetInfo (单位目标列表)
    ├── m_uniqueGOTargetInfo (游戏对象目标列表)
    ├── m_loadedScripts (已加载的 SpellScript 列表)
    └── m_hitTriggerSpells (命中触发的法术)

Aura (光环实例, 关联施法者和目标)
    └── AuraEffect[3] (光环效果)
        └── AuraScript 挂钩点

SpellMgr (全局单例, 法术注册中心)
    ├── SpellInfo* 注册表 (spellId -> SpellInfo*)
    ├── SpellChainNode* 链表
    ├── SpellProcEntryMap (Proc 数据)
    ├── SpellBonusDataMap (法术加成数据)
    └── 各种查询容器

SpellScript (脚本扩展)
    └── 注册钩子: BeforeCast, OnHit, OnEffectHitTarget, 等

AuraScript (光环脚本扩展)
    └── 注册钩子: OnEffectApply, OnEffectPeriodic, DoCheckProc, 等
```

---

## 4. Spell 类 - 法术施放核心

**文件**: `Spell.h` (875行), `Spell.cpp` (9,164行)

Spell 类是法术系统的**中央工作引擎**，是一个有状态的多阶段命令对象，封装了从开始施法到完成执行的全部逻辑。

### 4.1 关键辅助类型

#### SpellCastTargets
存储法术的目标信息:
- `m_unitTarget` (Unit*): 单位目标
- `m_itemTarget` (Item*): 物品目标
- `m_GOTarget` (GameObject*): 游戏对象目标
- `m_srcPos` / `m_dstPos` (WorldLocation): 来源/目标位置

#### SpellValue
存储运行时法术数值覆盖:
- `EffectBasePoints[3]`: 覆盖效果基础值
- `RadiusMod`: 半径修正
- `MaxTargets`: 最大目标数
- `AuraStack`: 光环层数
- `AuraDuration`: 光环持续时间

#### TargetInfo
单个目标的信息:
- `targetGUID`: 目标 GUID
- `missCondition`: 未命中原因 (SPELL_MISS_*)
- `effectMask`: 受影响的效果掩码
- `damage`: 造成的伤害
- `crit`: 是否暴击
- `reflectResult`: 反射结果

### 4.2 Spell 类成员变量

#### 公开成员

| 变量 | 类型 | 描述 |
|------|------|------|
| `m_spellInfo` | `SpellInfo const* const` | 不可变的法术原型数据 |
| `m_CastItem` | `Item*` | 施法使用的物品 |
| `m_weaponItem` | `Item*` | 涉及的武器 |
| `m_targets` | `SpellCastTargets` | 目标信息 |
| `m_customError` | `SpellCustomErrors` | 自定义错误码 |
| `m_comboTarget` | `Unit*` | 连击点目标 |
| `m_comboPointGain` | `int8` | 连击点获取量 |
| `m_appliedMods` | `std::set<Aura*>` | 影响此法术的光环集合 |

#### 关键受保护成员

| 变量 | 类型 | 描述 |
|------|------|------|
| `m_caster` | `Unit* const` | 施法者 |
| `m_spellValue` | `SpellValue* const` | 可修改的法术值 |
| `m_originalCasterGUID` | `ObjectGuid` | 真正的施法者 (如光环施放者) |
| `m_spellSchoolMask` | `SpellSchoolMask` | 法术学派 |
| `m_powerCost` | `int32` | 计算后的法力/能量消耗 |
| `m_casttime` | `int32` | 计算后的施法时间 |
| `m_canReflect` | `bool` | 是否可被反射 |
| `m_spellState` | `uint32` | 当前法术状态 |
| `m_UniqueTargetInfo` | `std::list<TargetInfo>` | 单位目标列表 |
| `m_UniqueGOTargetInfo` | `std::list<GOTargetInfo>` | GO 目标列表 |
| `m_UniqueItemInfo` | `std::list<ItemTargetInfo>` | 物品目标列表 |
| `m_destTargets[3]` | `SpellDestination[]` | 每效果的目标位置 |
| `m_loadedScripts` | `std::list<SpellScript*>` | 已加载脚本 |
| `m_damage` | `int32` | 累计伤害 |
| `m_healing` | `int32` | 累计治疗 |
| `m_procAttacker` | `uint32` | 攻击者 Proc 标志 |
| `m_procVictim` | `uint32` | 受害者 Proc 标志 |

### 4.3 法术状态 (SpellState)

```cpp
SPELL_STATE_NULL        = 0  // 未初始化
SPELL_STATE_PREPARING   = 1  // 准备中 (验证阶段)
SPELL_STATE_CASTING     = 2  // 施法中 (读条)
SPELL_STATE_FINISHED    = 3  // 已完成
SPELL_STATE_IDLE        = 4  // 空闲/等待
SPELL_STATE_DELAYED     = 5  // 延迟执行
```

### 4.4 效果处理模式 (SpellEffectHandleMode)

```cpp
SPELL_EFFECT_HANDLE_LAUNCH         // 发射阶段 (SMSG_SPELL_START)
SPELL_EFFECT_HANDLE_LAUNCH_TARGET  // 发射阶段, 每目标
SPELL_EFFECT_HANDLE_HIT            // 命中阶段 (SMSG_SPELL_GO)
SPELL_EFFECT_HANDLE_HIT_TARGET     // 命中阶段, 每目标
```

### 4.5 法术施放标志 (SpellCastFlags - 网络包)

```cpp
CAST_FLAG_PENDING              = 0x0001  // 有施法进度
CAST_FLAG_UNKNOWN_2            = 0x0002
CAST_FLAG_AMMO                 = 0x0004  // 包含弹药信息
CAST_FLAG_TRAJECTORY           = 0x0008  // 抛物线轨迹
CAST_FLAG_ADJUST_MISSILE       = 0x0010  // 调整飞行物
CAST_FLAG_NO_GCD               = 0x0004  // 无公共冷却
CAST_FLAG_UNKNOWN_10           = 0x0200
CAST_FLAG_VISUAL_CHAIN         = 0x0400  // 闪电链视觉效果
CAST_FLAG_UNKNOWN_13           = 0x0800
CAST_FLAG_RUNE_LIST            = 0x1000  // 符文消耗列表
CAST_FLAG_PROJECTILE           = 0x2000  // 投射物
CAST_FLAG_IMMUNITY             = 0x4000  // 免疫相关
CAST_FLAG_UNKNOWN_17           = 0x8000
CAST_FLAG_HEAL_PREDICTION      = 0x40000000 // 治疗预测
```

### 4.6 核心方法分组

#### 生命周期方法

```
prepare(targets, triggeredByAura)  // 准备法术, 返回 SpellCastResult
cast(skipCheck)                   // 公开施法入口
_cast(skipCheck)                  // 内部施法实现
finish(ok)                        // 完成法术
cancel(bySelf)                    // 取消施法
update(difftime)                  // 每帧更新
```

#### 验证方法

```
CheckCast(strict, param1, param2)  // 主验证入口
CheckPetCast(target)               // 宠物施法验证
CheckItems(param1, param2)         // 物品需求验证
CheckRange(strict)                 // 距离验证
CheckPower()                       // 法力验证
CheckRuneCost(RuneCostID)          // 符文消耗验证
CheckCasterAuras(preventionOnly)   // 施法者光环限制
CheckSpellFocus()                  // 法术焦点验证
```

#### 目标选择方法

```
InitExplicitTargets(targets)       // 初始化显式目标
SelectExplicitTargets()            // 解析显式目标
SelectSpellTargets()               // 主目标选择入口
SelectEffectImplicitTargets(...)   // 单效果隐式目标
SelectImplicitNearbyTargets(...)   // 近距离目标
SelectImplicitConeTargets(...)     // 锥形区域目标
SelectImplicitAreaTargets(...)     // 区域目标
SelectImplicitChainTargets(...)    // 链式目标
SelectImplicitTrajTargets(...)     // 抛物线目标
SearchAreaTargets(...)             // 区域搜索
SearchChainTargets(...)            // 链式搜索
```

#### 效果调度

```
HandleEffects(pUnit, pItem, pGO, i, mode)  // 分发效果处理
DoAllEffectOnTarget(TargetInfo*)            // 对目标执行所有效果
DoSpellHitOnUnit(unit, effectMask, scale)  // 对单位执行命中
```

#### 网络包方法

```
SendSpellStart()     // CMSG_SPELL_START
SendSpellGo()        // SMSG_SPELL_GO
SendCastResult()     // 发送施法结果
SendChannelUpdate()  // 通道更新
SendChannelStart()   // 通道开始
SendLogExecute()     // 执行日志
SendInterrupted()    // 中断通知
```

---

## 5. SpellInfo 类 - 法术原型数据

**文件**: `SpellInfo.h` (585行), `SpellInfo.cpp` (3,281行)

SpellInfo 是法术的**不可变原型数据**，在服务器启动时从 DBC (Spell.dbc) 构建，运行期间不会改变。

### 5.1 DBC 派生字段

| 字段 | 类型 | DBC 映射 | 描述 |
|------|------|---------|------|
| `Id` | `uint32` | ID | 法术 ID |
| `SchoolMask` | `uint32` | SchoolMask | 法术学派掩码 |
| `Dispel` | `uint32` | Dispel | 驱散类型 |
| `Mechanic` | `uint32` | Mechanic | 机制类型 |
| `Attributes` | `uint32` | Attributes | 主属性标志 |
| `AttributesEx`~`AttributesEx7` | `uint32` ×8 | 属性扩展 1~7 | 扩展属性 |
| `AttributesCu` | `uint32` | (计算得出) | 服务端自定义属性 |
| `Stances` / `StancesNot` | `uint32` | 姿态 | 变形形态要求/排除 |
| `Targets` | `uint32` | Targets | 法术目标类型 |
| `CasterAuraState` | `uint32` | CasterAuraState | 施法者光环状态要求 |
| `TargetAuraState` | `uint32` | TargetAuraState | 目标光环状态要求 |
| `CastTimeEntry` | `SpellCastTimesEntry*` | CastingTimeIndex | 施法时间 |
| `RecoveryTime` | `uint32` | RecoveryTime | 冷却时间 (ms) |
| `CategoryRecoveryTime` | `uint32` | CategoryRecoveryTime | 分类冷却 (ms) |
| `StartRecoveryCategory` | `uint32` | StartRecoveryCategory | GCD 分类 |
| `StartRecoveryTime` | `uint32` | StartRecoveryTime | GCD 时长 |
| `InterruptFlags` | `uint32` | InterruptFlags | 中断标志 |
| `AuraInterruptFlags` | `uint32` | AuraInterruptFlags | 光环中断标志 |
| `ChannelInterruptFlags` | `uint32` | ChannelInterruptFlags | 引导中断标志 |
| `ProcFlags` | `uint32` | ProcFlags | 触发标志 |
| `ProcChance` | `uint32` | ProcChance | 触发几率 |
| `ProcCharges` | `uint32` | ProcCharges | 触发充能 |
| `DurationEntry` | `SpellDurationEntry*` | DurationIndex | 持续时间 |
| `PowerType` | `uint32` | PowerType | 能量类型 |
| `ManaCost` | `uint32` | ManaCost | 基础消耗 |
| `ManaCostPercentage` | `uint32` | ManaCostPct | 消耗百分比 |
| `RuneCostID` | `uint32` | RuneCostID | 符文消耗 |
| `RangeEntry` | `SpellRangeEntry*` | RangeIndex | 范围 |
| `Speed` | `float` | Speed | 飞行物速度 |
| `StackAmount` | `uint32` | StackAmount | 最大层数 |
| `Reagent[8]` | `int32[]` | Reagent[0..7] | 材料需求 |
| `ReagentCount[8]` | `uint32[]` | ReagentCount[0..7] | 材料数量 |
| `SpellFamilyName` | `uint32` | SpellFamilyName | 法术家族 |
| `SpellFamilyFlags` | `flag96` | SpellFamilyFlags[0..2] | 家族标志 |
| `DmgClass` | `uint32` | DmgClass | 伤害类别 |
| `PreventionType` | `uint32` | PreventionType | 防止类型 |
| `EquippedItemClass` | `int32` | EquippedItemClass | 装备物品类别 |
| `Effects[3]` | `SpellEffectInfo[3]` | - | 3 个法术效果 |
| `ExplicitTargetMask` | `uint32` | (计算得出) | 预计算目标掩码 |
| `ChainEntry` | `SpellChainNode*` | (加载) | 法术等级链信息 |

### 5.2 计算得出字段

| 字段 | 类型 | 描述 |
|------|------|------|
| `_auraState` | `AuraStateType` | 计算得出的光环状态 |
| `_spellSpecific` | `SpellSpecificType` | 特殊法术类型 |
| `_isStackableWithRanks` | `bool` | 不同等级可叠加 |
| `_isSpellValid` | `bool` | 法术是否有效 |
| `_isCritCapable` | `bool` | 是否可暴击 |
| `JumpDistance` | `float` | 跳跃距离 |
| `_diminishInfoNonTriggered` | `SpellDiminishInfo` | 非触发 DR 信息 |
| `_diminishInfoTriggered` | `SpellDiminishInfo` | 触发 DR 信息 |
| `_immunityInfo[3]` | `ImmunityInfo` | 每效果免疫信息 |

### 5.3 关键方法

#### 属性检查
- `HasAttribute(SpellAttr0~7)`: 检查 8 种属性标志
- `HasAttribute(SpellCustomAttributes)`: 检查自定义属性

#### 效果/光环查询
- `HasEffect(SpellEffects)`: 是否包含指定效果
- `HasAura(AuraType)`: 是否应用指定光环
- `HasAnyAura()`: 是否有任何光环
- `HasAreaAuraEffect()`: 是否有区域光环效果
- `HasOnlyDamageEffects()`: 是否仅有伤害效果

#### 法术分类
- `IsPassive()` / `IsChanneled()` / `IsAutocastable()`
- `IsPositive()` / `IsPositiveEffect(uint8)`
- `IsProfession()` / `IsPrimaryProfession()`
- `IsAffectingArea()` / `IsTargetingArea()`
- `IsBreakingStealth()` / `IsRangedWeaponSpell()`

#### 等级链
- `IsRanked()` / `GetRank()` / `GetFirstRankSpell()`
- `GetNextRankSpell()` / `GetPrevRankSpell()`
- `IsRankOf(SpellInfo*)` / `IsDifferentRankOf(SpellInfo*)`

#### 数值计算
- `GetDuration()` / `GetMaxDuration()` / `GetMaxTicks()`
- `CalcCastTime(Unit*, Spell*)`: 计算施法时间
- `CalcPowerCost(Unit*, SpellSchoolMask, Spell*)`: 计算能量消耗
- `GetMinRange()` / `GetMaxRange()`: 获取距离

### 5.4 自定义属性 (SPELL_ATTR0_CU_*)

服务端计算的自定义属性, 不存储在 DBC 中:

```cpp
SPELL_ATTR0_CU_ENCHANT_PROC              // 附魔触发效果
SPELL_ATTR0_CU_CONE_BACK                 // 锥形法术向后
SPELL_ATTR0_CU_SHARE_DAMAGE              // 伤害分摊
SPELL_ATTR0_CU_NO_INITIAL_THREAT         // 无初始仇恨
SPELL_ATTR0_CU_AURA_CC                   // 光环是控制效果
SPELL_ATTR0_CU_DONT_BREAK_STEALTH        // 不打破潜行
SPELL_ATTR0_CU_NO_PVP_FLAG               // 不触发 PVP
SPELL_ATTR0_CU_DIRECT_DAMAGE             // 直接伤害
SPELL_ATTR0_CU_CHARGE                    // 冲锋法术
SPELL_ATTR0_CU_BINARY_SPELL              // 二元法术 (完全抵抗/命中)
SPELL_ATTR0_CU_NEGATIVE_EFF0/1/2         // 效果 0/1/2 为负面
SPELL_ATTR0_CU_POSITIVE_EFF0/1/2         // 效果 0/1/2 为正面
SPELL_ATTR0_CU_IGNORE_ARMOR              // 忽略护甲
SPELL_ATTR0_CU_ENCOUNTER_REWARD          // 副本奖励法术
SPELL_ATTR0_CU_BYPASS_MECHANIC_IMMUNITY  // 绕过机制免疫
SPELL_ATTR0_CU_AURA_CANNOT_BE_SAVED      // 光环不保存
SPELL_ATTR0_CU_FORCE_AURA_SAVING         // 强制保存光环
SPELL_ATTR0_CU_ONLY_ONE_AREA_AURA        // 仅允许一个区域光环
SPELL_ATTR0_CU_SINGLE_AURA_STACK         // 单光环叠加行为
SPELL_ATTR0_CU_ALLOW_INFLIGHT_TARGET     // 允许飞行中的目标
SPELL_ATTR0_CU_PICKPOCKET                // 扒窃法术
SPELL_ATTR0_CU_IGNORE_EVADE              // 忽略脱战
SPELL_ATTR0_CU_REQ_TARGET_FACING_CASTER  // 目标必须面向施法者
SPELL_ATTR0_CU_REQ_CASTER_BEHIND_TARGET  // 施法者必须在目标背后
SPELL_ATTR0_CU_NEEDS_AMMO_DATA           // 需要弹药数据
SPELL_ATTR0_CU_SCHOOLMASK_NORMAL_WITH_MAGIC  // 学派含物理+魔法
SPELL_ATTR0_CU_NO_POSITIVE_TAKEN_BONUS   // 无正面获得加成
SPELL_ATTR0_CU_POSITIVE / _NEGATIVE      // 复合标志
```

---

## 6. SpellEffectInfo - 法术效果信息

**定义位置**: `SpellInfo.h`

每个法术最多包含 3 个效果 (`MAX_SPELL_EFFECTS = 3`)，每个效果由 SpellEffectInfo 描述。

### 6.1 字段列表

| 字段 | 类型 | 描述 |
|------|------|------|
| `EffectIndex` | `uint8` | 效果索引 (0/1/2) |
| `Effect` | `uint32` | 效果类型 (SPELL_EFFECT_*) |
| `ApplyAuraName` | `AuraType` | 光环类型 (SPELL_AURA_*) |
| `Amplitude` | `uint32` | 周期效果间隔 (ms) |
| `DieSides` | `int32` | 随机伤害骰子面数 |
| `RealPointsPerLevel` | `float` | 每等级点数缩放 |
| `BasePoints` | `int32` | 基础效果值 |
| `PointsPerComboPoint` | `float` | 每连击点附加值 |
| `ValueMultiplier` | `float` | 值乘数 |
| `DamageMultiplier` | `float` | 伤害乘数 |
| `BonusMultiplier` | `float` | 加成乘数 |
| `MiscValue` | `int32` | 杂项值 (生物 ID, 物品 ID 等) |
| `MiscValueB` | `int32` | 第二杂项值 |
| `Mechanic` | `Mechanics` | 机制类型 |
| `TargetA` | `SpellImplicitTargetInfo` | 主目标 |
| `TargetB` | `SpellImplicitTargetInfo` | 副目标 |
| `RadiusEntry` | `SpellRadiusEntry const*` | 半径 (从 DBC) |
| `ChainTarget` | `uint32` | 链式跳跃数 |
| `ItemType` | `uint32` | 物品类型 |
| `TriggerSpell` | `uint32` | 触发法术 |
| `SpellClassMask` | `flag96` | 法术家族掩码 |

### 6.2 关键方法

```
IsEffect(SpellEffects)              // 检查是否为指定效果
IsAura(AuraType)                    // 检查是否为指定光环
CalcValue(Unit*, ...)               // 计算效果数值
CalcRadius(Unit*, Spell*)           // 计算半径
GetProvidedTargetMask()             // 提供的目标标志
GetImplicitTargetType()             // 隐式目标类型
IsTargetingArea()                   // 是否目标区域
IsAreaAuraEffect()                  // 是否区域光环
```

---

## 7. 法术效果系统 (SpellEffects)

**文件**: `SpellEffects.cpp` (6,365行)

法术效果系统包含 100+ 种不同的效果处理器，每种对应一个 `SpellEffects` 枚举值。

### 7.1 完整效果方法列表

| 方法 | 效果类型 | 描述 |
|------|----------|------|
| `EffectNULL` | 0 | 无效果 |
| `EffectUnused` | 1 | 未使用 |
| `EffectResurrectNew` | - | 新复活机制 |
| `EffectInstaKill` | - | 即时击杀 |
| `EffectEnvironmentalDMG` | - | 环境伤害 |
| `EffectSchoolDMG` | - | 学派伤害 (火/冰/奥/等) |
| `EffectDummy` | - | 虚拟效果 (脚本处理) |
| `EffectTriggerSpell` | - | 触发另一个法术 |
| `EffectTriggerMissileSpell` | - | 触发飞行物法术 |
| `EffectForceCast` | - | 强制施放 |
| `EffectTriggerRitualOfSummoning` | - | 召唤仪式 |
| `EffectJump` | - | 跳跃 (施法者->目标) |
| `EffectJumpDest` | - | 跳跃到目标位置 |
| `EffectTeleportUnits` | - | 传送单位 |
| `EffectTeleUnitsFaceCaster` | - | 传送并面向施法者 |
| `EffectApplyAura` | - | 施加光环 |
| `EffectApplyAreaAura` | - | 施加区域光环 |
| `EffectUnlearnSpecialization` | - | 忘记专精 |
| `EffectPowerDrain` | - | 抽取能量 |
| `EffectSendEvent` | - | 发送事件 |
| `EffectPowerBurn` | - | 燃烧能量 |
| `EffectHeal` | - | 治疗 |
| `EffectHealPct` | - | 百分比治疗 |
| `EffectHealMechanical` | - | 机械治疗 |
| `EffectHealMaxHealth` | - | 最大生命值治疗 |
| `EffectHealthLeech` | - | 生命汲取 |
| `EffectCreateItem` | - | 创建物品 |
| `EffectCreateItem2` | - | 创建物品 (从模板) |
| `EffectCreateRandomItem` | - | 创建随机物品 |
| `EffectPersistentAA` | - | 持续区域光环 (地面效果) |
| `EffectEnergize` | - | 充能 |
| `EffectEnergizePct` | - | 百分比充能 |
| `EffectOpenLock` | - | 开锁 |
| `EffectSummonChangeItem` | - | 召唤切换物品 |
| `EffectProficiency` | - | 熟练度检查 |
| `EffectSummonType` | - | 召唤类型 |
| `EffectLearnSpell` | - | 学习法术 |
| `EffectDispel` | - | 驱散 |
| `EffectDualWield` | - | 双持 |
| `EffectPull` | - | 拉拽 |
| `EffectDistract` | - | 分心 |
| `EffectPickPocket` | - | 扒窃 |
| `EffectAddFarsight` | - | 添加远景视角 |
| `EffectUntrainTalents` | - | 重置天赋 |
| `EffectLearnSkill` | - | 学习技能 |
| `EffectAddHonor` | - | 增加荣誉 |
| `EffectTradeSkill` | - | 专业技能 |
| `EffectEnchantItemPerm` | - | 永久附魔物品 |
| `EffectEnchantItemPrismatic` | - | 棱彩附魔 |
| `EffectEnchantItemTmp` | - | 临时附魔 |
| `EffectTameCreature` | - | 驯服生物 |
| `EffectSummonPet` | - | 召唤宠物 |
| `EffectLearnPetSpell` | - | 学习宠物技能 |
| `EffectTaunt` | - | 嘲讽 |
| `EffectWeaponDmg` | - | 武器伤害 + 效果 |
| `EffectThreat` | - | 仇恨 |
| `EffectInterruptCast` | - | 打断施法 |
| `EffectSummonObjectWild` | - | 召唤野生游戏对象 |
| `EffectScriptEffect` | - | 脚本效果 |
| `EffectSanctuary` | - | 庇护所 |
| `EffectAddComboPoints` | - | 添加连击点 |
| `EffectDuel` | - | 决斗 |
| `EffectStuck` | - | 卡死释放 |
| `EffectSummonPlayer` | - | 召唤玩家 |
| `EffectActivateObject` | - | 激活游戏对象 |
| `EffectApplyGlyph` | - | 雕文 |
| `EffectEnchantHeldItem` | - | 附魔手持物品 |
| `EffectDisEnchant` | - | 分解 |
| `EffectInebriate` | - | 醉酒 |
| `EffectFeedPet` | - | 喂养宠物 |
| `EffectDismissPet` | - | 解散宠物 |
| `EffectSummonObject` | - | 召唤游戏对象 |
| `EffectResurrect` | - | 复活 |
| `EffectAddExtraAttacks` | - | 额外攻击 |
| `EffectParry` | - | 招架 |
| `EffectBlock` | - | 格挡 |
| `EffectLeap` | - | 跳跃 |
| `EffectReputation` | - | 声望 |
| `EffectQuestComplete` | - | 完成任务 |
| `EffectForceDeselect` | - | 强制取消选择 |
| `EffectSelfResurrect` | - | 自我复活 |
| `EffectSkinning` | - | 剥皮 |
| `EffectCharge` | - | 冲锋 |
| `EffectChargeDest` | - | 冲锋到位置 |
| `EffectKnockBack` | - | 击退 |
| `EffectLeapBack` | - | 后跃 |
| `EffectQuestClear` | - | 清除任务 |
| `EffectSendTaxi` | - | 乘坐飞行 |
| `EffectPullTowards` | - | 拉向施法者 |
| `EffectDispelMechanic` | - | 按机制驱散 |
| `EffectResurrectPet` | - | 复活宠物 |
| `EffectDestroyAllTotems` | - | 销毁所有图腾 |
| `EffectDurabilityDamage` | - | 耐久度伤害 |
| `EffectDurabilityDamagePCT` | - | 百分比耐久度伤害 |
| `EffectModifyThreatPercent` | - | 修改仇恨百分比 |
| `EffectTransmitted` | - | 传输 (工程学物品) |
| `EffectProspecting` | - | 采矿 |
| `EffectMilling` | - | 研磨 |
| `EffectSkill` | - | 技能 |
| `EffectSpiritHeal` | - | 灵魂治疗 (战场) |
| `EffectSkinPlayerCorpse` | - | 剥玩家尸体 |
| `EffectStealBeneficialBuff` | - | 偷取有益增益 |
| `EffectKillCreditPersonal` | - | 个人击杀积分 |
| `EffectKillCredit` | - | 击杀积分 |
| `EffectQuestFail` | - | 任务失败 |
| `EffectQuestStart` | - | 开始任务 |
| `EffectActivateRune` | - | 激活符文 |
| `EffectCreateTamedPet` | - | 创建被驯服宠物 |
| `EffectDiscoverTaxi` | - | 发现飞行路线 |
| `EffectTitanGrip` | - | 泰坦之握 |
| `EffectRedirectThreat` | - | 仇恨转移 |
| `EffectGameObjectDamage` | - | 游戏对象伤害 |
| `EffectGameObjectRepair` | - | 修复游戏对象 |
| `EffectGameObjectSetDestructionState` | - | 设置 GO 毁灭状态 |
| `EffectRenamePet` | - | 重命名宠物 |
| `EffectPlayMusic` | - | 播放音乐 |
| `EffectSpecCount` | - | 专精计数 |
| `EffectActivateSpec` | - | 激活专精 (双天赋) |
| `EffectPlaySound` | - | 播放声音 |
| `EffectRemoveAura` | - | 移除光环 |
| `EffectCastButtons` | - | 施法按钮 |
| `EffectRechargeManaGem` | - | 充能法力宝石 |
| `EffectBind` | - | 绑定 |
| `EffectSummonRaFFriend` | - | 召唤随机副本队友 |

---

## 8. 法术目标选择系统

法术目标选择是 Spell 类中最复杂的子系统，决定法术影响哪些目标。

### 8.1 目标选择分类 (SpellTargetSelectionCategories)

| 值 | 名称 | 描述 |
|---|------|------|
| 0 | NYI | 未实现 |
| 1 | DEFAULT | 默认单体目标 |
| 2 | CHANNEL | 通道 (光束) 目标 |
| 3 | NEARBY | 附近目标 |
| 4 | CONE | 锥形区域 |
| 5 | AREA | 区域选择 |
| 6 | TRAJ | 抛物线路径 |

### 8.2 目标引用类型 (SpellTargetReferenceTypes)

| 值 | 名称 | 描述 |
|---|------|------|
| 0 | NONE | 无参考 |
| 1 | CASTER | 施法者位置 |
| 2 | TARGET | 当前目标 |
| 3 | LAST | 上一个目标 |
| 4 | SRC | 来源位置 |
| 5 | DEST | 目标位置 |

### 8.3 目标对象类型 (SpellTargetObjectTypes)

| 值 | 名称 | 描述 |
|---|------|------|
| 0 | NONE | 无对象 |
| 1 | SRC | 来源 (位置) |
| 2 | DEST | 目标 (位置) |
| 3 | UNIT | 单位 (玩家/NPC) |
| 4 | UNIT_AND_DEST | 单位 + 目标位置 |
| 5 | GOBJ | 游戏对象 |
| 6 | GOBJ_ITEM | 游戏对象物品 |
| 7 | ITEM | 物品 |
| 8 | CORPSE | 尸体 |
| 9 | CORPSE_ENEMY | 敌方尸体 (效果用) |
| 10 | CORPSE_ALLY | 友方尸体 (效果用) |

### 8.4 目标检查类型 (SpellTargetCheckTypes)

| 值 | 名称 | 描述 |
|---|------|------|
| 0 | DEFAULT | 无特殊检查 |
| 1 | ENTRY | 匹配特定生物 |
| 2 | ENEMY | 必须敌对 |
| 3 | ALLY | 必须友好 |
| 4 | PARTY | 必须在小队 |
| 5 | RAID | 必须在团队 |
| 6 | RAID_CLASS | 团队中特定职业 |
| 7 | PASSENGER | 必须是载具乘客 |
| 8 | CORPSE | 必须是尸体 |

### 8.5 目标方向类型 (SpellTargetDirectionTypes)

| 值 | 名称 | 描述 |
|---|------|------|
| 0 | NONE | 无方向限制 |
| 1 | FRONT | 正前方 |
| 2 | BACK | 正后方 |
| 3 | RIGHT | 右侧 |
| 4 | LEFT | 左侧 |
| 5 | FRONT_RIGHT | 右前方 |
| 6 | BACK_RIGHT | 右后方 |
| 7 | BACK_LEFT | 左后方 |
| 8 | FRONT_LEFT | 左前方 |
| 9 | RANDOM | 随机方向 |
| 10 | ENTRY | 基于条目数据 |

### 8.6 显式目标标志 (SpellCastTargetFlags)

客户端发送的目标标志:

| 标志 | 值 | 描述 |
|------|---|------|
| `TARGET_FLAG_NONE` | 0x0 | 无目标 |
| `TARGET_FLAG_UNIT` | 0x2 | 单位 |
| `TARGET_FLAG_UNIT_RAID` | 0x4 | 团队成员 |
| `TARGET_FLAG_UNIT_PARTY` | 0x8 | 小队成员 |
| `TARGET_FLAG_ITEM` | 0x10 | 物品 |
| `TARGET_FLAG_SOURCE_LOCATION` | 0x20 | 来源位置 |
| `TARGET_FLAG_DEST_LOCATION` | 0x40 | 目标位置 |
| `TARGET_FLAG_UNIT_ENEMY` | 0x80 | 敌方单位 |
| `TARGET_FLAG_UNIT_ALLY` | 0x100 | 友方单位 |
| `TARGET_FLAG_CORPSE_ENEMY` | 0x200 | 敌方尸体 |
| `TARGET_FLAG_UNIT_DEAD` | 0x400 | 死亡单位 |
| `TARGET_FLAG_GAMEOBJECT` | 0x800 | 游戏对象 |
| `TARGET_FLAG_CORPSE_ALLY` | 0x8000 | 友方尸体 |
| `TARGET_FLAG_UNIT_MINIPET` | 0x10000 | 战斗宠物 |
| `TARGET_FLAG_GLYPH_SLOT` | 0x20000 | 雕文槽 |
| `TARGET_FLAG_DEST_TARGET` | 0x40000 | 目标即目的地 |

---

## 9. 光环系统 (Auras)

**目录**: `src/server/game/Spells/Auras/`
**代码量**: 11,157 行 (5 个文件)

光环系统管理所有持续性的法术效果, 包括增益 (buff) 和减益 (debuff)。

### 9.1 类层次结构

```
Aura (光环实例)
├── 施法者相关: m_casterGuid, m_castItemEntry, m_maxDuration, m spellInfo
├── 目标管理: m_applications (目标映射)
├── 效果管理: AuraEffect[3]
└── 脚本: AuraScript 列表

AuraApplication (光环应用到特定目标)
├── 关联: GetTarget(), GetBase()
├── 标志: m_removeMode, m_slot, m_effectMask
└── 状态: m_isNeedPreRemove

AuraEffect (单个光环效果)
├── 数值: m_amount, m_baseAmount, m_periodicTimer
├── 属性: m_tickNumber, m_canRecalculateAmount
└── 类型: GetAuraType(), GetEffIndex(), GetMiscValue()
```

### 9.2 Aura 类核心成员

```cpp
class Aura {
    // 核心数据
    ObjectGuid const    m_casterGuid;           // 施法者 GUID
    uint32 const        m_castItemEntry;        // 施法物品
    SpellInfo const*    m_spellInfo;            // 法术原型
    WorldObject*        m_owner;                // 光环拥有者

    // 效果数组
    AuraEffect*         m_effects[MAX_SPELL_EFFECTS];

    // 应用管理
    AuraApplicationMap  m_applications;         // 目标 -> 应用映射

    // 时间
    int32               m_maxDuration;          // 最大持续时间
    int32               m_duration;             // 当前持续时间
    int32               m_timeCla;              // 上次更新时间
    uint8               m_charges;              // 充能数
    uint8               m_stackAmount;          // 叠加层数

    // 脚本
    std::list<AuraScript*> m_loadedScripts;
};
```

### 9.3 AuraEffect 类核心成员

```cpp
class AuraEffect {
    AuraType const          m_auraType;           // 光环类型
    uint8 const             m_effIndex;           // 效果索引
    int32                   m_amount;             // 效果数值
    int32                   m_baseAmount;         // 基础数值
    int32                   m_periodicTimer;      // 周期计时器
    uint32                  m_tickNumber;         // 已触发次数
    Aura*                   m_base;               // 所属 Aura
    SpellModifier*          m_spellmod;           // 法术修正器
    std::list<AuraScript*>  m_auraScripts;        // 挂载的脚本
    int32                   m_period;             // 周期间隔
    int32                   m_amplitude;          // DBC 周期值
};
```

### 9.4 光环应用/移除流程

#### 应用流程
1. 创建 Aura 对象
2. 创建 AuraEffect[3]
3. 对每个目标创建 AuraApplication
4. 按效果顺序调用 `AuraEffect::HandleApply`
5. 发送 `SMSG_AURA_UPDATE` 客户端包

#### 移除流程
1. 设置移除模式 (`AURA_REMOVE_BY_*`)
2. 按效果逆序调用 `AuraEffect::HandleRemove`
3. 清理 AuraApplication
4. 删除 Aura 对象

### 9.5 光环移除模式 (AuraRemoveMode)

```
AURA_REMOVE_NONE              // 无 (默认)
AURA_REMOVE_BY_DEFAULT        // 自然过期
AURA_REMOVE_BY_CANCEL         // 被取消
AURA_REMOVE_BY_ENEMY_SPELL    // 被敌方法术驱散
AURA_REMOVE_BY_EXPIRE         // 持续时间结束
AURA_REMOVE_BY_DEATH          // 死亡
AURA_REMOVE_BY_SHIELD_BREAK   // 护盾破碎
```

### 9.6 AuraEffectHandleModes

```cpp
AURA_EFFECT_HANDLE_DEFAULT              = 0x00 // 默认
AURA_EFFECT_HANDLE_REAL                 = 0x01 // 实际应用/移除
AURA_EFFECT_HANDLE_SEND_FOR_CLIENT      = 0x02 // 发送给客户端
AURA_EFFECT_HANDLE_CHANGE_AMOUNT        = 0x04 // 数量变化
AURA_EFFECT_HANDLE_REAPPLY              = 0x08 // 重新应用
AURA_EFFECT_HANDLE_STAT                 = 0x10 // 属性重算
AURA_EFFECT_HANDLE_SKILL                = 0x20 // 技能重算
```

---

## 10. AuraType 完整枚举

**定义**: `Auras/SpellAuraDefines.h`

共约 300 种光环类型, 按功能分类:

### 10.1 控制/限制类

| 值 | 名称 | 描述 |
|---|------|------|
| 2 | MOD_POSSESS | 精神控制 |
| 5 | MOD_CONFUSE | 混乱 |
| 6 | MOD_CHARM | 驯服/魅惑 |
| 7 | MOD_FEAR | 恐惧 |
| 12 | MOD_STUN | 眩晕 |
| 25 | MOD_PACIFY | 镇静 (不可攻击) |
| 26 | MOD_ROOT | 定身 |
| 27 | MOD_SILENCE | 沉默 |
| 33 | MOD_DECREASE_SPEED | 减速 |
| 60 | MOD_PACIFY_SILENCE | 镇静+沉默 |
| 66 | FEIGN_DEATH | 假死 |
| 67 | MOD_DISARM | 缴械 |
| 93 | MOD_UNATTACKABLE | 不可攻击 |
| 236 | CONTROL_VEHICLE | 控制载具 |

### 10.2 属性/数值修改类

| 值 | 名称 | 描述 |
|---|------|------|
| 9 | MOD_ATTACKSPEED | 攻击速度 |
| 13 | MOD_DAMAGE_DONE | 伤害输出 |
| 14 | MOD_DAMAGE_TAKEN | 受到伤害 |
| 22 | MOD_RESISTANCE | 抗性 |
| 29 | MOD_STAT | 属性修改 |
| 30 | MOD_SKILL | 技能修改 |
| 31 | MOD_INCREASE_SPEED | 加速 |
| 34 | MOD_INCREASE_HEALTH | 增加生命 |
| 35 | MOD_INCREASE_ENERGY | 增加能量 |
| 47 | MOD_PARRY_PERCENT | 招架百分比 |
| 49 | MOD_DODGE_PERCENT | 躲闪百分比 |
| 50 | MOD_CRITICAL_HEALING_AMOUNT | 暴击治疗量 |
| 51 | MOD_BLOCK_PERCENT | 格挡百分比 |
| 52 | MOD_WEAPON_CRIT_PERCENT | 武器暴击百分比 |
| 54 | MOD_HIT_CHANCE | 命中几率 |
| 55 | MOD_SPELL_HIT_CHANCE | 法术命中几率 |
| 57 | MOD_SPELL_CRIT_CHANCE | 法术暴击 |
| 61 | MOD_SCALE | 缩放 |
| 79 | MOD_DAMAGE_PERCENT_DONE | 伤害百分比 |
| 80 | MOD_PERCENT_STAT | 属性百分比 |
| 99 | MOD_ATTACK_POWER | 攻击强度 |
| 107 | ADD_FLAT_MODIFIER | 平坦修正 |
| 108 | ADD_PCT_MODIFIER | 百分比修正 |
| 115 | MOD_HEALING | 治疗量 |
| 118 | MOD_HEALING_PCT | 治疗百分比 |
| 124 | MOD_RANGED_ATTACK_POWER | 远程攻击强度 |
| 128 | MOD_POSSESS_PET | 控制宠物 |
| 137 | MOD_TOTAL_STAT_PERCENTAGE | 总属性百分比 |
| 138 | MOD_MELEE_HASTE | 近战急速 |
| 166 | MOD_ATTACK_POWER_PCT | 攻击强度百分比 |
| 189 | MOD_RATING | 评级 |
| 240 | MOD_EXPERTISE | 精准 |

### 10.3 伤害/吸收类

| 值 | 名称 | 描述 |
|---|------|------|
| 3 | PERIODIC_DAMAGE | 周期伤害 (DOT) |
| 8 | PERIODIC_HEAL | 周期治疗 (HOT) |
| 15 | DAMAGE_SHIELD | 伤害护盾 |
| 24 | PERIODIC_ENERGIZE | 周期能量恢复 |
| 53 | PERIODIC_LEECH | 周期汲取 |
| 64 | PERIODIC_MANA_LEECH | 周期法力汲取 |
| 69 | SCHOOL_ABSORB | 学派吸收 |
| 81 | SPLIT_DAMAGE_PCT | 分摊伤害 |
| 97 | MANA_SHIELD | 法力护盾 |
| 153 | SPLIT_DAMAGE_FLAT | 固定分摊伤害 |
| 162 | POWER_BURN | 燃烧能量 |
| 194 | MOD_TARGET_ABSORB_SCHOOL | 目标学派吸收 |

### 10.4 免疫/保护类

| 值 | 名称 | 描述 |
|---|------|------|
| 37 | EFFECT_IMMUNITY | 效果免疫 |
| 38 | STATE_IMMUNITY | 状态免疫 |
| 39 | SCHOOL_IMMUNITY | 学派免疫 |
| 40 | DAMAGE_IMMUNITY | 伤害免疫 |
| 41 | DISPEL_IMMUNITY | 驱散免疫 |
| 77 | MECHANIC_IMMUNITY | 机制免疫 |
| 147 | MECHANIC_IMMUNITY_MASK | 机制免疫掩码 |
| 267 | MOD_IMMUNE_AURA_APPLY_SCHOOL | 免疫光环学派 |

### 10.5 形态/视觉类

| 值 | 名称 | 描述 |
|---|------|------|
| 16 | MOD_STEALTH | 潜行 |
| 17 | MOD_STEALTH_DETECT | 潜行探测 |
| 18 | MOD_INVISIBILITY | 隐身 |
| 19 | MOD_INVISIBILITY_DETECT | 隐身探测 |
| 36 | MOD_SHAPESHIFT | 变形 |
| 56 | TRANSFORM | 变身 |
| 61 | MOD_SCALE | 缩放 |
| 239 | MOD_SCALE_2 | 缩放 2 |
| 273 | X_RAY | 透视 |
| 247 | CLONE_CASTER | 克隆施法者 |

### 10.6 移动/位移类

| 值 | 名称 | 描述 |
|---|------|------|
| 31 | MOD_INCREASE_SPEED | 加速 |
| 32 | MOD_INCREASE_MOUNTED_SPEED | 坐骑加速 |
| 33 | MOD_DECREASE_SPEED | 减速 |
| 58 | MOD_INCREASE_SWIM_SPEED | 游泳加速 |
| 82 | WATER_BREATHING | 水下呼吸 |
| 104 | WATER_WALK | 水上行走 |
| 105 | FEATHER_FALL | 缓落 |
| 106 | HOVER | 悬浮 |
| 129 | MOD_SPEED_ALWAYS | 始终加速 |
| 130 | MOD_MOUNTED_SPEED_ALWAYS | 始终坐骑加速 |
| 201 | FLY | 飞行 |
| 205-211 | MOD_INCREASE_FLIGHT_* | 飞行速度系列 |
| 241 | FORCE_MOVE_FORWARD | 强制向前移动 |

### 10.7 Proc/触发类

| 值 | 名称 | 描述 |
|---|------|------|
| 23 | PERIODIC_TRIGGER_SPELL | 周期触发法术 |
| 42 | PROC_TRIGGER_SPELL | 触发法术 |
| 43 | PROC_TRIGGER_DAMAGE | 触发伤害 |
| 48 | PERIODIC_TRIGGER_SPELL_FROM_CLIENT | 客户端周期触发 |
| 109 | ADD_TARGET_TRIGGER | 目标触发 |
| 111 | ADD_CASTER_HIT_TRIGGER | 施法者命中触发 |
| 148 | RETAIN_COMBO_POINTS | 保留连击点 |
| 223 | RAID_PROC_FROM_CHARGE | 团队充能触发 |
| 225 | RAID_PROC_FROM_CHARGE_WITH_VALUE | 团队充能触发(带值) |
| 226 | PERIODIC_DUMMY | 周期虚拟 |
| 227 | PERIODIC_TRIGGER_SPELL_WITH_VALUE | 周期触发(带值) |
| 231 | PROC_TRIGGER_SPELL_WITH_VALUE | 触发法术(带值) |
| 286 | ABILITY_PERIODIC_CRIT | 周期暴击 |

### 10.8 特殊机制类

| 值 | 名称 | 描述 |
|---|------|------|
| 4 | DUMMY | 虚拟 (脚本处理) |
| 10 | MOD_THREAT | 仇恨修改 |
| 11 | MOD_TAUNT | 嘲讽 |
| 28 | REFLECT_SPELLS | 法术反射 |
| 44 | TRACK_CREATURES | 追踪生物 |
| 45 | TRACK_RESOURCES | 追踪资源 |
| 70 | EXTRA_ATTACKS | 额外攻击 |
| 78 | MOUNTED | 骑乘 |
| 86 | CHANNEL_DEATH_ITEM | 引导死亡物品 |
| 95 | GHOST | 鬼魂状态 |
| 96 | SPELL_MAGNET | 法术磁铁 |
| 112 | OVERRIDE_CLASS_SCRIPTS | 覆盖职业脚本 |
| 139 | FORCE_REACTION | 强制声望 |
| 145 | MOD_PET_TALENT_POINTS | 宠物天赋点 |
| 146 | ALLOW_TAME_PET_TYPE | 允许驯服类型 |
| 160 | MOD_AOE_AVOIDANCE | AoE 闪避 |
| 196 | MOD_COOLDOWN | 冷却修改 |
| 215 | ARENA_PREPARATION | 竞技场准备 |
| 216 | HASTE_SPELLS | 法术急速 |
| 221 | MOD_DETAUNT | 降低仇恨 |
| 249 | CONVERT_RUNE | 转换符文 |
| 256 | NO_REAGENT_USE | 无需材料 |
| 261 | PHASE | 相位 |
| 262 | ABILITY_IGNORE_AURASTATE | 忽略光环状态 |
| 263 | ALLOW_ONLY_ABILITY | 仅允许特定技能 |
| 280 | MOD_ARMOR_PENETRATION_PCT | 护甲穿透百分比 |

---

## 11. SpellScript 脚本系统

**文件**: `SpellScript.h` (964行), `SpellScript.cpp` (1,213行)

SpellScript 提供了一套钩子系统，允许通过 C++ 脚本扩展或修改任何法术的行为。

### 11.1 脚本生命周期

```
SPELL_SCRIPT_STATE_NONE          // 未初始化
SPELL_SCRIPT_STATE_REGISTRATION  // Register() 调用中
SPELL_SCRIPT_STATE_LOADING       // Load() 调用中
SPELL_SCRIPT_STATE_UNLOADING     // Unload() 调用中
```

### 11.2 SpellScript 钩子执行顺序

```
 1. BeforeCast              - 施法条满，施法处理前
 2. OnCheckCast             - 覆盖 CheckCast 结果
 3a. OnObjectAreaTargetSelect  - 过滤区域目标
 3b. OnObjectTargetSelect      - 过滤单体目标
 3c. OnDestinationTargetSelect - 过滤目标位置
 4. OnCast                  - 法术发射/执行前
 5. AfterCast               - 飞行物发射后
 6. OnEffectLaunch          - 发射阶段效果处理前
 7. OnEffectLaunchTarget    - 发射阶段每目标
 8. OnEffectHit             - 命中阶段效果处理前
 9. BeforeHit               - 法术命中目标前
10. OnEffectHitTarget       - 命中阶段效果处理前(每目标)
11. OnHit                   - 伤害/Proc 前(每目标)
12. AfterHit                - 所有处理完成后(每目标)
```

### 11.3 SpellScript 钩子注册

#### 施法钩子

| 钩子 | 签名 | 注册宏 | 触发时机 |
|------|------|--------|---------|
| `BeforeCast` | `void()` | `SpellCastFn(F)` | 施法条满，施法处理前 |
| `OnCast` | `void()` | `SpellCastFn(F)` | 法术发射/执行前 |
| `AfterCast` | `void()` | `SpellCastFn(F)` | 飞行物发射+即时操作后 |

#### 检查钩子

| 钩子 | 签名 | 注册宏 | 触发时机 |
|------|------|--------|---------|
| `OnCheckCast` | `SpellCastResult()` | `SpellCheckCastFn(F)` | 覆盖施法合法性检查 |

#### 效果钩子

| 钩子 | 签名 | 注册宏 | 触发时机 |
|------|------|--------|---------|
| `OnEffectLaunch` | `void(SpellEffIndex)` | `SpellEffectFn(F,I,N)` | 发射阶段，每效果 |
| `OnEffectLaunchTarget` | `void(SpellEffIndex)` | `SpellEffectFn(F,I,N)` | 发射阶段，每效果每目标 |
| `OnEffectHit` | `void(SpellEffIndex)` | `SpellEffectFn(F,I,N)` | 命中阶段，每效果 |
| `OnEffectHitTarget` | `void(SpellEffIndex)` | `SpellEffectFn(F,I,N)` | 命中阶段，每效果每目标 |

#### 命中钩子

| 钩子 | 签名 | 注册宏 | 触发时机 |
|------|------|--------|---------|
| `BeforeHit` | `void(SpellMissInfo)` | `BeforeSpellHitFn(F)` | 法术命中目标前 |
| `OnHit` | `void()` | `SpellHitFn(F)` | 伤害/Proc 应用前 |
| `AfterHit` | `void()` | `SpellHitFn(F)` | 目标所有操作完成后 |

#### 目标选择钩子

| 钩子 | 签名 | 注册宏 | 触发时机 |
|------|------|--------|---------|
| `OnObjectAreaTargetSelect` | `void(std::list<WorldObject*>&)` | `SpellObjectAreaTargetSelectFn(F,I,N)` | 过滤区域目标列表 |
| `OnObjectTargetSelect` | `void(WorldObject*&)` | `SpellObjectTargetSelectFn(F,I,N)` | 过滤单体目标 |
| `OnDestinationTargetSelect` | `void(SpellDestination&)` | `SpellDestinationTargetSelectFn(F,I,N)` | 过滤目标位置 |

### 11.4 SpellScript 交互方法

#### 始终可用
```cpp
GetCaster()                    // Unit* 施法者
GetOriginalCaster()            // Unit* 原始施法者
GetSpellInfo()                 // SpellInfo const*
GetSpellValue()                // SpellValue const*
```

#### 施法准备后 - 显式目标
```cpp
GetExplTargetDest()            // WorldLocation*
GetExplTargetWorldObject()     // WorldObject*
GetExplTargetUnit()            // Unit*
GetExplTargetGObj()            // GameObject*
GetExplTargetItem()            // Item*
```

#### 命中阶段 - 目标数据
```cpp
GetHitUnit()                   // Unit*
GetHitCreature()               // Creature*
GetHitPlayer()                 // Player*
GetHitDest()                   // WorldLocation*
GetHitDamage() / SetHitDamage() / PreventHitDamage()
GetHitHeal() / SetHitHeal() / PreventHitHeal()
GetHitAura() / PreventHitAura()
PreventHitEffect(SpellEffIndex)
PreventHitDefaultEffect(SpellEffIndex)
GetEffectValue() / SetEffectValue()
GetCastItem()                  // Item*
GetTriggeringSpell()           // SpellInfo const*
```

### 11.5 注册宏

```cpp
// 准备宏 (在脚本类顶部使用)
PrepareSpellScript(script_class_name)
PrepareAuraScript(script_class_name)

// 注册宏
RegisterSpellScript(script_class_name)
RegisterSpellAndAuraScriptPair(spell_script, aura_script)
```

---

## 12. AuraScript 光环脚本系统

### 12.1 所有钩子方法

#### 区域目标检查

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `DoCheckAreaTarget` | `bool(Unit*)` | `AuraCheckAreaTargetFn(F)` |

#### 驱散钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `OnDispel` | `void(DispelInfo*)` | `AuraDispelFn(F)` |
| `AfterDispel` | `void(DispelInfo*)` | `AuraDispelFn(F)` |

#### 效果应用/移除钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `OnEffectApply` | `void(AuraEffect const*, AuraEffectHandleModes)` | `AuraEffectApplyFn(F,I,N,M)` |
| `AfterEffectApply` | `void(AuraEffect const*, AuraEffectHandleModes)` | `AuraEffectApplyFn(F,I,N,M)` |
| `OnEffectRemove` | `void(AuraEffect const*, AuraEffectHandleModes)` | `AuraEffectRemoveFn(F,I,N,M)` |
| `AfterEffectRemove` | `void(AuraEffect const*, AuraEffectHandleModes)` | `AuraEffectRemoveFn(F,I,N,M)` |

#### 周期钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `OnEffectPeriodic` | `void(AuraEffect const*)` | `AuraEffectPeriodicFn(F,I,N)` |
| `OnEffectUpdatePeriodic` | `void(AuraEffect*)` | `AuraEffectUpdatePeriodicFn(F,I,N)` |

#### 计算钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `DoEffectCalcAmount` | `void(AuraEffect const*, int32&, bool&)` | `AuraEffectCalcAmountFn(F,I,N)` |
| `DoEffectCalcPeriodic` | `void(AuraEffect const*, bool&, int32&)` | `AuraEffectCalcPeriodicFn(F,I,N)` |
| `DoEffectCalcSpellMod` | `void(AuraEffect const*, SpellModifier*&)` | `AuraEffectCalcSpellModFn(F,I,N)` |

#### 吸收/护盾/分摊钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `OnEffectAbsorb` | `void(AuraEffect*, DamageInfo&, uint32&)` | `AuraEffectAbsorbFn(F,I)` |
| `AfterEffectAbsorb` | `void(AuraEffect*, DamageInfo&, uint32&)` | - |
| `OnEffectManaShield` | `void(AuraEffect*, DamageInfo&, uint32&)` | `AuraEffectManaShieldFn(F,I)` |
| `AfterEffectManaShield` | `void(AuraEffect*, DamageInfo&, uint32&)` | - |
| `OnEffectSplit` | `void(AuraEffect*, DamageInfo&, uint32&)` | `AuraEffectSplitFn(F,I)` |

#### Proc 钩子

| 钩子 | 签名 | 宏 |
|------|------|-----|
| `DoCheckProc` | `bool(ProcEventInfo&)` | `AuraCheckProcFn(F)` |
| `DoCheckEffectProc` | `bool(AuraEffect const*, ProcEventInfo&)` | `AuraCheckEffectProcFn(F,I,N)` |
| `DoAfterCheckProc` | `bool(ProcEventInfo&, bool)` | `AuraAfterCheckProcFn(F)` |
| `DoPrepareProc` | `void(ProcEventInfo&)` | `AuraProcFn(F)` |
| `OnProc` | `void(ProcEventInfo&)` | `AuraProcFn(F)` |
| `AfterProc` | `void(ProcEventInfo&)` | `AuraProcFn(F)` |
| `OnEffectProc` | `void(AuraEffect const*, ProcEventInfo&)` | `AuraEffectProcFn(F,I,N)` |
| `AfterEffectProc` | `void(AuraEffect const*, ProcEventInfo&)` | `AuraEffectProcFn(F,I,N)` |

### 12.2 AuraScript 交互方法

#### 基础访问
```cpp
GetSpellInfo() / GetId()
GetCasterGUID() / GetCaster()
GetOwner() / GetUnitOwner()
GetAura() / GetType()
```

#### 持续时间
```cpp
GetDuration() / SetDuration()
RefreshDuration()
GetApplyTime() / GetMaxDuration()
SetMaxDuration() / CalcMaxDuration()
IsExpired() / IsPermanent()
```

#### 充能/层数
```cpp
GetCharges() / SetCharges()
CalcMaxCharges() / ModCharges() / DropCharge()
GetStackAmount() / SetStackAmount() / ModStackAmount()
```

#### 目标 (仅在带有 AuraApplication 的钩子中有效)
```cpp
GetTarget()                    // 当前光环目标 Unit*
GetTargetApplication()         // AuraApplication*
```

#### 控制
```cpp
PreventDefaultAction()         // 阻止默认行为
Remove(AuraRemoveMode)         // 移除光环
```

---

## 13. SpellMgr 法术管理器

**文件**: `SpellMgr.h` (838行), `SpellMgr.cpp` (3,782行)

SpellMgr 是全局单例，作为法术系统的注册中心和查询接口。

### 13.1 核心容器

```cpp
// 法术注册表
SpellInfoMap                    mSpellInfoMap;           // spellId -> SpellInfo*
SpellChainMap                   mSpellChains;            // first_spell -> SpellChainNode

// Proc 数据
SpellProcEntryMap               mSpellProcEventMap;      // spellId -> SpellProcEntry
SpellProcEventMap               mSpellProcEventMap;      // 兼容容器

// 延伸数据
SpellBonusDataMap               mSpellBonusData;         // 法术加成数据
SpellThreatMap                  mSpellThreatMap;         // 法术仇恨数据
SpellMixologyMap                mSpellMixologyMap;       // 调配学数据
SpellPetAuraMap                 mSpellPetAuraMap;        // 宠物光环
SpellEnchantProcEventMap        mSpellEnchantProcEventMap; // 附魔触发
SpellLinkedMap                  mSpellLinkedStore;       // 关联法术
SpellAreaMap                    mSpellAreaMap;           // 区域法术
SpellCooldownOverrideMap        mSpellCooldownOverrideMap; // 冷却覆盖
SpellGroupMap                   mSpellGroupMap;          // 法术组
SpellGroupStackMap              mSpellGroupStackMap;     // 法术组叠加规则

// 其他
SpellTargetPositionMap          mSpellTargetPositionMap; // 法术目标位置
SpellRequiredMap                mSpellRequiredMap;       // 法术前置
CreatureImmunityMap             mCreatureImmunityMap;    // 生物免疫
SpellJumpDistanceMap            mSpellJumpDistanceMap;   // 跳跃距离
```

### 13.2 关键方法

#### 初始化
```cpp
InitializeSpellInfoPrecomputedFlags()   // 预计算标志
LoadSpellRanks()                        // 加载法术等级
LoadSpellRequired()                     // 加载前置法术
LoadSpellLearnSkills()                  // 加载法术学习技能
LoadSpellTargetPositions()              // 加载目标位置
LoadSpellGroups()                       // 加载法术组
LoadSpellGroupStackRules()              // 加载叠加规则
LoadSpellProcEvents()                   // 加载 Proc 事件
LoadSpellBonusData()                    // 加载加成数据
LoadSpellThreats()                      // 加载仇恨数据
LoadSpellMixology()                     // 加载调配学
LoadSpellPetAuras()                     // 加载宠物光环
LoadSpellEnchantProcData()              // 加载附魔触发
LoadSpellLinked()                       // 加载关联法术
LoadSpellArea()                         // 加载区域法术
LoadSpellCooldownOverrides()            // 加载冷却覆盖
LoadSpellJumpDistances()                // 加载跳跃距离
LoadSpellCustomAttr()                   // 加载自定义属性
```

#### 查询方法
```cpp
GetSpellInfo(spellId)                   // 获取法术原型
GetSpellChainNode(firstSpellId)         // 获取等级链节点
GetSpellInfoForDifficulty(spellId, difficultyId) // 难度缩放
GetSpellBonusData(spellId)              // 获取加成数据
GetSpellThreat(spellId)                 // 获取仇恨修正
GetSpellArea(spellId)                   // 获取区域法术数据
IsSpellProcEventCanTriggeredBy(procEntry, eventInfo) // Proc 触发检查
CanAurasStack(spellInfo1, spellInfo2)   // 光环叠加检查
```

### 13.3 Proc 数据结构 (SpellProcEntry)

```cpp
struct SpellProcEntry {
    uint32  SchoolMask;           // 学派掩码
    uint32  SpellFamilyName;      // 法术家族
    flag96  SpellFamilyMask;      // 家族标志
    uint32  ProcFlags;            // 触发条件
    uint32  SpellTypeMask;        // 法术类型掩码
    uint32  SpellPhaseMask;       // 法术阶段掩码
    uint32  HitMask;              // 命中掩码
    uint32  AttributesMask;       // 属性掩码
    uint32  DisableEffectsMask;   // 禁用效果掩码
    float   ProcsPerMinute;       // 每分钟触发次数
    float   Chance;               // 触发几率
    uint32  Cooldown;             // 内置冷却
    uint32  Charges;              // 触发充能数
};
```

---

## 14. SpellInfoCorrections 法术修正

**文件**: `SpellInfoCorrections.cpp` (5,373行)

此文件包含对 DBC 数据的运行时修正，修复客户端数据中的错误或不一致。

### 修正类型

1. **属性覆盖**: 修改 `AttributesCu` 自定义属性
2. **效果目标修正**: 修改效果的 `TargetA`/`TargetB`
3. **范围修正**: 修改法术距离
4. **持续时间修正**: 修改效果持续时间
5. **类别修正**: 修改法术分类
6. **机制修正**: 修改效果机制类型
7. **区域/地图限制**: 修改法术可用区域

### 修正模式

```cpp
// 在 ApplySpellFixes 中按 Spell ID 逐一修正
switch (spellInfo->Id)
{
    case 12345:  // 具体法术 ID
        spellInfo->AttributesCu |= SPELL_ATTR0_CU_XXX;
        spellInfo->Effects[EFFECT_0].TargetA = Targets::TARGET_XXX;
        break;
}
```

---

## 15. 数据库依赖

### 15.1 World 数据库表 (17张)

| 表名 | 字段 | 加载方法 | 描述 |
|------|------|---------|------|
| `creature_immunities` | ID, SchoolMask, DispelTypeMask, MechanicsMask, Effects, Auras, ImmuneAoE, ImmuneChain | `LoadCreatureImmunities()` | 生物免疫定义 |
| `spell_ranks` | first_spell_id, spell_id, rank | `LoadSpellRanks()` | 法术等级链 |
| `spell_required` | spell_id, req_spell | `LoadSpellRequired()` | 法术前置要求 |
| `spell_target_position` | ID, EffectIndex, MapID, PositionX, PositionY, PositionZ, Orientation | `LoadSpellTargetPositions()` | 法术目标坐标 |
| `spell_group` | id, spell_id | `LoadSpellGroups()` | 法术分组 |
| `spell_group_stack_rules` | group_id, stack_rule | `LoadSpellGroupStackRules()` | 分组叠加规则 |
| `spell_proc` | SpellId, SchoolMask, SpellFamilyName, SpellFamilyMask0~2, ProcFlags, SpellTypeMask, SpellPhaseMask, HitMask, AttributesMask, DisableEffectsMask, ProcsPerMinute, Chance, Cooldown, Charges | `LoadSpellProcEvents()` | 法术触发条件 |
| `spell_bonus_data` | entry, direct_bonus, dot_bonus, ap_bonus, ap_dot_bonus | `LoadSpellBonusData()` | 法术加成系数 |
| `spell_threat` | entry, flatMod, pctMod, apPctMod | `LoadSpellThreats()` | 法术仇恨修正 |
| `spell_mixology` | entry, pctMod | `LoadSpellMixology()` | 调配学效果 |
| `spell_pet_auras` | spell, effectId, pet, aura | `LoadSpellPetAuras()` | 宠物光环 |
| `spell_enchant_proc_data` | entry, customChance, PPMChance, procEx, attributeMask | `LoadSpellEnchantProcData()` | 附魔触发数据 |
| `spell_linked_spell` | spell_trigger, spell_effect, type | `LoadSpellLinked()` | 法术关联触发 |
| `spell_area` | spell, area, quest_start, quest_start_status, quest_end_status, quest_end, aura_spell, racemask, gender, autocast | `LoadSpellArea()` | 区域法术 |
| `spell_cooldown_overrides` | Id, RecoveryTime, CategoryRecoveryTime, StartRecoveryTime, StartRecoveryCategory | `LoadSpellCooldownOverrides()` | 冷却覆盖 |
| `spell_jump_distance` | ID, JumpDistance | `LoadSpellJumpDistances()` | 跳跃距离 |
| `spell_custom_attr` | spell_id, attributes | `LoadSpellCustomAttr()` | 自定义属性 |

### 15.2 Character 数据库表 (间接引用)

| 表名 | 描述 |
|------|------|
| `character_spell` | 角色法术列表 (清理无效法术) |
| `character_talent` | 角色天赋 (清理无效天赋) |

### 15.3 脚本相关表

| 表名 | 描述 | 使用方 |
|------|------|--------|
| `spell_script_names` | 法术脚本名称映射 | 脚本系统外部加载 |

---

## 16. DBC 客户端数据依赖

### 16.1 直接使用的 DBC 存储 (12个)

| DBC 存储 | DBC 文件 | 用途 |
|----------|---------|------|
| `sSpellStore` | Spell.dbc | 核心法术数据 (SpellEntry) |
| `sSpellCastTimesStore` | SpellCastTimes.dbc | 施法时间 |
| `sSpellDurationStore` | SpellDuration.dbc | 法术持续时间 |
| `sSpellRangeStore` | SpellRange.dbc | 法术范围 |
| `sSpellRadiusStore` | SpellRadius.dbc | 效果半径 |
| `sSpellCategoryStore` | SpellCategory.dbc | 法术分类/冷却分组 |
| `sSpellDifficultyStore` | SpellDifficulty.dbc | 难度缩放法术 |
| `sSpellVisualStore` | SpellVisual.dbc | 法术视觉效果 |
| `sSpellRuneCostStore` | SpellRuneCost.dbc | 死骑符文消耗 |
| `sSpellShapeshiftFormStore` | SpellShapeshiftForm.dbc | 变形形态 |
| `sSpellItemEnchantmentStore` | SpellItemEnchantment.dbc | 物品附魔 |
| `sSkillLineAbilityStore` | SkillLineAbility.dbc | 技能线-法术映射 |

### 16.2 间接引用的 DBC 文件

| DBC 文件 | 引用位置 | 用途 |
|----------|---------|------|
| SpellFocusObject.dbc | Spell.cpp:4522 | 法术焦点对象 |
| AreaTable.dbc | Spell.cpp:4524 | 区域检查 |
| Map.dbc | SpellMgr.cpp:1512 | 地图验证 |
| SkillLine.dbc | Spell.cpp:4608 | 技能线查询 |
| SoundEntries.dbc | SpellEffects.cpp:203,204 | 播放声音 |
| ScreenEffect.dbc | SpellAuraEffects.cpp:325 | 屏幕效果 |
| summonproperties.dbc | SpellEffects.cpp:2371 | 召唤属性 |

---

## 17. 关键枚举与常量定义

### 17.1 施法中断标志 (SpellInterruptFlags)

```cpp
SPELL_INTERRUPT_FLAG_MOVEMENT            = 0x01  // 移动中断
SPELL_INTERRUPT_FLAG_PUSH_BACK           = 0x02  // 伤害推后施法
SPELL_INTERRUPT_FLAG_UNK3                = 0x04  // 未知
SPELL_INTERRUPT_FLAG_INTERRUPT           = 0x08  // 伤害完全打断
SPELL_INTERRUPT_FLAG_ABORT_ON_DMG        = 0x10  // 直伤中断
SPELL_INTERRUPT_UNK                      = 0x20  // 未知 (雕文常见)
```

### 17.2 引导中断标志 (SpellChannelInterruptFlags)

```cpp
CHANNEL_INTERRUPT_FLAG_INTERRUPT         = 0x08  // 伤害打断引导
CHANNEL_FLAG_DELAY                       = 0x4000 // 引导有延迟
```

### 17.3 光环中断标志 (SpellAuraInterruptFlags) - 部分

```cpp
AURA_INTERRUPT_FLAG_HITBYSPELL           = 0x00000001  // 被负面法术命中
AURA_INTERRUPT_FLAG_TAKE_DAMAGE          = 0x00000002  // 受到任何伤害
AURA_INTERRUPT_FLAG_CAST                 = 0x00000004  // 施放任何法术
AURA_INTERRUPT_FLAG_MOVE                 = 0x00000008  // 任何移动
AURA_INTERRUPT_FLAG_TURNING              = 0x00000010  // 转向
AURA_INTERRUPT_FLAG_JUMP                 = 0x00000020  // 进入战斗
AURA_INTERRUPT_FLAG_NOT_MOUNTED          = 0x00000040  // 下马
AURA_INTERRUPT_FLAG_NOT_ABOVEWATER       = 0x00000080  // 入水
AURA_INTERRUPT_FLAG_NOT_UNDERWATER       = 0x00000100  // 离开水
AURA_INTERRUPT_FLAG_NOT_SHEATHED         = 0x00000200  // 拔武器
AURA_INTERRUPT_FLAG_TALK                 = 0x00000400  // 与NPC交谈/拾取
AURA_INTERRUPT_FLAG_USE                  = 0x00000800  // 使用游戏对象
AURA_INTERRUPT_FLAG_MELEE_ATTACK         = 0x00001000  // 近战攻击
AURA_INTERRUPT_FLAG_TRANSFORM            = 0x00008000  // 变形
AURA_INTERRUPT_FLAG_MOUNT                = 0x00020000  // 上马
AURA_INTERRUPT_FLAG_NOT_SEATED           = 0x00040000  // 站起
AURA_INTERRUPT_FLAG_CHANGE_MAP           = 0x00080000  // 离开地图/传送
AURA_INTERRUPT_FLAG_TELEPORTED           = 0x00400000  // 被传送
AURA_INTERRUPT_FLAG_ENTER_PVP_COMBAT     = 0x00800000  // 进入PVP
AURA_INTERRUPT_FLAG_DIRECT_DAMAGE        = 0x01000000  // 直接伤害
AURA_INTERRUPT_FLAG_LANDING              = 0x02000000  // 落地
AURA_INTERRUPT_FLAG_LEAVE_COMBAT         = 0x80000000  // 离开战斗
```

### 17.4 法术修正操作 (SpellModOp)

```cpp
SPELLMOD_DAMAGE                  = 0   // 伤害
SPELLMOD_DURATION                = 1   // 持续时间
SPELLMOD_THREAT                  = 2   // 仇恨
SPELLMOD_EFFECT1                 = 3   // 效果1基础值
SPELLMOD_CHARGES                 = 4   // 充能
SPELLMOD_RANGE                   = 5   // 范围
SPELLMOD_RADIUS                  = 6   // 半径
SPELLMOD_CRITICAL_CHANCE         = 7   // 暴击几率
SPELLMOD_ALL_EFFECTS             = 8   // 所有效果
SPELLMOD_NOT_LOSE_CASTING_TIME   = 9   // 防止施法推后
SPELLMOD_CASTING_TIME            = 10  // 施法时间
SPELLMOD_COOLDOWN                = 11  // 冷却
SPELLMOD_EFFECT2                 = 12  // 效果2基础值
SPELLMOD_IGNORE_ARMOR            = 13  // 忽略护甲
SPELLMOD_COST                    = 14  // 消耗
SPELLMOD_CRIT_DAMAGE_BONUS       = 15  // 暴击伤害加成
SPELLMOD_RESIST_MISS_CHANCE      = 16  // 抵抗/未命中几率
SPELLMOD_JUMP_TARGETS            = 17  // 链式跳跃目标
SPELLMOD_CHANCE_OF_SUCCESS       = 18  // 成功几率
SPELLMOD_ACTIVATION_TIME         = 19  // 激活时间
SPELLMOD_DAMAGE_MULTIPLIER       = 20  // 伤害乘数
SPELLMOD_GLOBAL_COOLDOWN         = 21  // GCD
SPELLMOD_DOT                     = 22  // DOT 伤害
SPELLMOD_EFFECT3                 = 23  // 效果3基础值
SPELLMOD_BONUS_MULTIPLIER        = 24  // 加成乘数
SPELLMOD_PROC_PER_MINUTE         = 26  // PPM 速率
SPELLMOD_VALUE_MULTIPLIER        = 27  // 值乘数
SPELLMOD_RESIST_DISPEL_CHANCE    = 28  // 驱散抵抗
SPELLMOD_SPELL_COST_REFUND       = 30  // 失败消耗返还
```

### 17.5 法术值修正 (SpellValueMod)

```cpp
SPELLVALUE_BASE_POINT0          = 0   // 效果0基础值
SPELLVALUE_BASE_POINT1          = 1   // 效果1基础值
SPELLVALUE_BASE_POINT2          = 2   // 效果2基础值
SPELLVALUE_RADIUS_MOD           = 3   // 半径修正
SPELLVALUE_MAX_TARGETS          = 4   // 最大目标数
SPELLVALUE_AURA_STACK           = 5   // 光环层数
SPELLVALUE_AURA_DURATION        = 6   // 光环持续时间
SPELLVALUE_FORCED_CRIT_RESULT   = 7   // 强制暴击
SPELLVALUE_MISCVALUE0           = 8   // 杂项值0
SPELLVALUE_MISCVALUE1           = 9   // 杂项值1
SPELLVALUE_MISCVALUE2           = 10  // 杂项值2
```

### 17.6 触发施法标志 (TriggerCastFlags)

```cpp
TRIGGERED_NONE                               = 0x00000000
TRIGGERED_IGNORE_GCD                         = 0x00000001  // 忽略GCD
TRIGGERED_IGNORE_SPELL_AND_CATEGORY_CD       = 0x00000002  // 忽略冷却
TRIGGERED_IGNORE_POWER_AND_REAGENT_COST      = 0x00000004  // 忽略消耗
TRIGGERED_IGNORE_CAST_ITEM                   = 0x00000008  // 不消耗施法物品
TRIGGERED_IGNORE_AURA_SCALING                = 0x00000010  // 忽略光环缩放
TRIGGERED_IGNORE_CAST_IN_PROGRESS            = 0x00000020  // 允许在施法中施放
TRIGGERED_IGNORE_COMBO_POINTS                = 0x00000040  // 忽略连击点
TRIGGERED_CAST_DIRECTLY                      = 0x00000080  // 直接施放
TRIGGERED_IGNORE_AURA_INTERRUPT_FLAGS        = 0x00000100  // 忽略光环中断
TRIGGERED_IGNORE_SET_FACING                  = 0x00000200  // 不调整朝向
TRIGGERED_IGNORE_SHAPESHIFT                  = 0x00000400  // 忽略变形形态
TRIGGERED_IGNORE_CASTER_AURASTATE            = 0x00000800  // 忽略光环状态
TRIGGERED_IGNORE_CASTER_MOUNTED_OR_ON_VEHICLE = 0x00002000  // 忽略坐骑/载具
TRIGGERED_IGNORE_CASTER_AURAS                = 0x00010000  // 忽略施法者光环
TRIGGERED_DISALLOW_PROC_EVENTS               = 0x00020000  // 禁止Proc (默认)
TRIGGERED_DONT_REPORT_CAST_ERROR             = 0x00040000  // 不报告错误
TRIGGERED_IGNORE_EQUIPPED_ITEM_REQUIREMENT   = 0x00080000  // 忽略装备要求
TRIGGERED_NO_PERIODIC_RESET                  = 0x00100000  // 不重置周期
TRIGGERED_IGNORE_EFFECTS                     = 0x00200000  // 忽略效果
TRIGGERED_FULL_MASK                          = 0x0007FFFF
TRIGGERED_FULL_DEBUG_MASK                    = 0xFFFFFFFF
```

---

## 18. 施法生命周期

### 18.1 完整流程

```
客户端发送 CMSG_CAST_SPELL
    │
    ▼
Unit::CastSpell / Spell::prepare()
    │
    ├─ LoadScripts() - 加载 SpellScript
    ├─ InitExplicitTargets() - 初始化目标
    ├─ CheckCast() - 施法验证
    │   ├─ CheckCasterAuras() - 沉默/眩晕检查
    │   ├─ CheckRange() - 距离检查
    │   ├─ CheckPower() - 能量检查
    │   ├─ CheckItems() - 物品/材料检查
    │   ├─ CheckRuneCost() - 符文消耗检查
    │   ├─ CheckSpellFocus() - 法术焦点检查
    │   ├─ CallScriptCheckCastHandlers() - 脚本检查
    │   └─ spell_area 条件检查
    │
    ├─ SelectSpellTargets() - 目标选择
    │   ├─ SelectExplicitTargets() - 显式目标
    │   ├─ SelectImplicitNearbyTargets() - 近距离
    │   ├─ SelectImplicitConeTargets() - 锥形
    │   ├─ SelectImplicitAreaTargets() - 区域
    │   ├─ SelectImplicitChainTargets() - 链式
    │   ├─ SelectImplicitTrajTargets() - 抛物线
    │   └─ 脚本目标过滤
    │
    ├─ TakePower() - 消耗能量
    ├─ TakeReagents() - 消耗材料
    ├─ TakeCastItem() - 消耗施法物品
    │
    ├─ SendSpellStart() - 发送施法开始包
    │
    ├─ [读条阶段] update() 循环
    │   ├─ m_timer -= difftime
    │   ├─ 推后处理 (受伤)
    │   └─ m_timer <= 0 时完成读条
    │
    ├─ CallScriptBeforeCastHandlers() - 脚本 BeforeCast
    ├─ CallScriptOnCastHandlers() - 脚本 OnCast
    │
    ├─ HandleLaunchPhase() - 发射阶段
    │   ├─ 对每个目标: DoAllEffectOnLaunchTarget()
    │   │   ├─ CallScriptOnEffectLaunchTarget()
    │   │   └─ HandleEffects(..., LAUNCH_TARGET)
    │   │       └─ EffectXxx() (发射阶段处理)
    │   └─ SendSpellGo() - 发送法术执行包
    │
    ├─ [飞行物阶段] handle_delayed()
    │   └─ 等待飞行物到达目标
    │
    ├─ HandleThreatSpells() - 处理仇恨
    │
    ├─ 命中阶段: DoAllEffectOnTarget()
    │   ├─ DoSpellHitOnUnit()
    │   │   ├─ 抵抗/免疫检查
    │   │   ├─ CallScriptBeforeHitHandlers()
    │   │   ├─ CallScriptOnHitHandlers()
    │   │   ├─ HandleEffects(..., HIT_TARGET)
    │   │   │   └─ EffectXxx() (命中阶段处理)
    │   │   │       ├─ EffectSchoolDMG() - 伤害计算
    │   │   │       ├─ EffectHeal() - 治疗
    │   │   │       ├─ EffectApplyAura() - 施加光环
    │   │   │       └─ ... 其他效果
    │   │   ├─ DoTriggersOnSpellHit() - 触发命中法术
    │   │   ├─ CallScriptAfterHitHandlers()
    │   │   └─ Proc 处理
    │   └─ 对 GO/Item 目标同样处理
    │
    ├─ CallScriptAfterCastHandlers() - 脚本 AfterCast
    │
    └─ finish(ok) - 完成清理
        ├─ 移除施法状态
        ├─ TriggerGlobalCooldown() - GCD
        └─ 发送冷却事件
```

### 18.2 伤害计算流程

```
EffectSchoolDMG()
    │
    ├─ CalculateSpellDamage(effIndex, target)  // 计算基础伤害
    │   └─ BasePoints + Random(DieSides) + caster scaling
    │
    ├─ ApplyDamageMultipliers()                // 应用伤害乘数
    │   ├─ Effect.DamageMultiplier
    │   ├─ Spell modifiers (SPELLMOD_DAMAGE)
    │   └─ Target taken modifiers
    │
    ├─ CalcAbsorbResistance()                  // 吸收和抵抗
    │   ├─ Armor reduction (物理)
    │   ├─ School resistance (魔法)
    │   └─ Absorb auras
    │
    ├─ DealDamage()                            // 造成伤害
    │   ├─ Unit::DealDamage()
    │   ├─ DamageInfo creation
    │   └─ Kill check
    │
    └─ Threat generation                       // 生成仇恨
```

### 18.3 光环应用流程

```
EffectApplyAura()
    │
    ▼
Aura::_CreateForCaster() / Aura::TryCreateAfterCasting()
    │
    ├─ Unit::_AddAura()
    │   ├─ 检查现有光环 (叠加/替换)
    │   ├─ 创建 Aura 对象
    │   ├─ 创建 AuraEffect[3]
    │   ├─ ApplyAllEffect(aura, target)
    │   │   └─ 对每个效果:
    │   │       ├─ AuraEffect::HandleApply(AURA_EFFECT_HANDLE_REAL)
    │   │       │   └─ SpellAuraEffects.cpp 中的具体处理
    │   │       └─ 脚本 OnEffectApply / AfterEffectApply
    │   └─ 发送 SMSG_AURA_UPDATE
    │
    └─ Aura 存储到 Unit 的光环列表中
```

---

## 19. Proc 触发系统

### 19.1 Proc 事件分类

Proc 系统是连接法术系统和光环系统的桥梁, 定义了哪些事件可以触发光环效果。

#### 攻击者完成 Proc 标志 (部分)

```
PROC_FLAG_DONE_SPELL_MELEE_DMG_CLASS     // 近战伤害完成
PROC_FLAG_DONE_SPELL_RANGED_DMG_CLASS    // 远程伤害完成
PROC_FLAG_DONE_SPELL_MAGIC_DMG_CLASS     // 法术伤害完成
PROC_FLAG_DONE_SPELL_NONE_DMG_CLASS_POS  // 正面无法术伤害完成
PROC_FLAG_DONE_HIT_MELEE                 // 近战命中
PROC_FLAG_DONE_HIT_RANGED                // 远程命中
PROC_FLAG_DONE_HIT_SPELL                 // 法术命中
PROC_FLAG_DONE_CRIT_HIT_MELEE            // 近战暴击
PROC_FLAG_DONE_CRIT_HIT_RANGED           // 远程暴击
PROC_FLAG_DONE_CRIT_HIT_SPELL            // 法术暴击
```

#### 受害者 Proc 标志 (部分)

```
PROC_FLAG_TAKEN_SPELL_MELEE_DMG_CLASS    // 受到近战伤害
PROC_FLAG_TAKEN_SPELL_RANGED_DMG_CLASS   // 受到远程伤害
PROC_FLAG_TAKEN_SPELL_MAGIC_DMG_CLASS    // 受到法术伤害
PROC_FLAG_TAKEN_HIT_MELEE                // 被近战命中
PROC_FLAG_TAKEN_HIT_RANGED               // 被远程命中
PROC_FLAG_TAKEN_HIT_SPELL                // 被法术命中
PROC_FLAG_TAKEN_CRIT_HIT_MELEE           // 被近战暴击
PROC_FLAG_TAKEN_CRIT_HIT_RANGED          // 被远程暴击
PROC_FLAG_TAKEN_CRIT_HIT_SPELL           // 被法术暴击
```

### 19.2 扩展 Proc 标志 (ProcFlagsEx)

```cpp
PROC_EX_NORMAL_HIT          = 0x0000001   // 普通命中
PROC_EX_CRITICAL_HIT        = 0x0000002   // 暴击
PROC_EX_MISS                = 0x0000004   // 未命中
PROC_EX_DODGE               = 0x0000008   // 闪避
PROC_EX_PARRY               = 0x0000010   // 招架
PROC_EX_BLOCK               = 0x0000020   // 格挡
PROC_EX_EVADE               = 0x0000040   // 脱战
PROC_EX_IMMUNE              = 0x0000080   // 免疫
PROC_EX_DEFLECT             = 0x0000100   // 偏转
PROC_EX_ABSORB              = 0x0000200   // 吸收
PROC_EX_REFLECT             = 0x0000400   // 反射
PROC_EX_INTERRUPT           = 0x0000800   // 打断
PROC_EX_FULL_RESIST         = 0x0001000   // 完全抵抗
PROC_EX_PARTIAL_RESIST      = 0x0002000   // 部分抵抗
PROC_EX_HIT_TRIGGER_A       = 0x0004000   // 命中触发 A
PROC_EX_HIT_TRIGGER_B       = 0x0008000   // 命中触发 B
PROC_EX_NOT_ACTIVE_SPELL    = 0x0010000   // 非活跃法术
```

### 19.3 Proc 触发流程

```
伤害/治疗/命中事件发生
    │
    ▼
Unit::ProcDamageAndSpell(ProcEventInfo&)
    │
    ├─ 遍历 Unit 的所有 Aura
    │   ├─ 检查 ProcEntry 条件
    │   ├─ DoCheckProc() 脚本检查
    │   ├─ DoCheckEffectProc() 效果检查
    │   ├─ PPM 速率计算
    │   ├─ 几率检查
    │   ├─ 内置冷却检查
    │   │
    │   ├─ DoPrepareProc() 脚本
    │   ├─ 消耗充能 (mod charges)
    │   │
    │   └─ TriggerProc()
    │       ├─ OnProc() / AfterProc() 脚本
    │       ├─ OnEffectProc() / AfterEffectProc() 脚本
    │       └─ 执行触发:
    │           ├─ PROC_TRIGGER_SPELL -> CastSpell()
    │           ├─ PROC_TRIGGER_DAMAGE -> DealDamage()
    │           └─ 其他触发类型
    │
    └─ spell_linked_spell 关联触发
```

---

## 20. 与其他模块的关系

### 20.1 上游依赖 (被 Spells 使用)

| 模块 | 使用方式 |
|------|---------|
| `Entities/` | Unit, Player, Creature, GameObject, Item |
| `DataStores/` | DBC 数据存储 (sSpellStore 等) |
| `Database/` | WorldDatabase, CharacterDatabase |
| `Maps/` | Map, Cell, 地图管理 |
| `Grids/` | ObjectAccessor, 对象查找 |
| `Movement/` | PathGenerator, 移动系统 |
| `Packets/` | 网络包定义和发送 |
| `Scripting/` | 脚本加载器和宏 |
| `Conditions/` | ConditionMgr, 条件系统 |
| `Gossip/` | Gossip 系统 (部分法术交互) |
| `Loot/` | Loot 系统 (EffectCreateItem) |
| `Quests/` | Quest 系统 (任务相关效果) |
| `Trade/` | 交易系统 |
| `Chat/` | 聊天系统 (法术发送消息) |
| `Battlegrounds/` | 战场系统 |
| `Arenas/` | 竞技场系统 |
| `Groups/` | 队伍/团队系统 |
| `Pets/` | 宠物系统 |
| `Vehicles/` | 载具系统 |
| `Achievements/` | 成就系统 |

### 20.2 下游依赖 (使用 Spells 的模块)

几乎所有游戏模块都依赖 Spells:
- `Combat/` - 战斗系统使用法术进行攻击/伤害
- `AI/` - AI 系统使用法术进行施法决策
- `MovementHandlers/` - 移动处理器处理施法中断
- `SpellHandler.cpp` - 客户端法术包处理
- `PetHandler.cpp` - 宠物法术包处理
- `CharacterHandler.cpp` - 角色登录时加载法术
- `ItemHandler.cpp` - 物品使用触发法术
- `GameObjectHandler.cpp` - 游戏对象使用触发法术

### 20.3 数据流图

```
Spell.dbc (客户端)
    │
    ▼
DBCStore (启动加载)
    │
    ▼
SpellInfo (不可变原型)
    │
    │ cast spell
    ▼
Spell (可变施放实例)
    ├── 目标选择
    ├── 效果执行
    │   ├── EffectSchoolDMG -> 伤害
    │   ├── EffectHeal -> 治疗
    │   ├── EffectApplyAura -> Aura 创建
    │   └── EffectXxx -> 各种效果
    │
    ▼
Aura (持续效果)
    ├── AuraEffect (效果处理器)
    │   ├── 周期触发 (DOT/HOT)
    │   ├── 属性修改
    │   └── 状态限制
    │
    └── Proc 触发 -> 新 Spell

World DB (spell_proc, spell_bonus_data, etc.)
    │
    ▼
SpellMgr (全局注册表 + 扩展数据)
    └── 查询接口 -> Spell/SpellInfo/Aura

SpellScript / AuraScript (C++ 脚本)
    └── 钩子回调 -> 修改默认行为
```

---

## 附录: 文件路径索引

| 文件 | 绝对路径 |
|------|---------|
| Spell.h | `src/server/game/Spells/Spell.h` |
| Spell.cpp | `src/server/game/Spells/Spell.cpp` |
| SpellInfo.h | `src/server/game/Spells/SpellInfo.h` |
| SpellInfo.cpp | `src/server/game/Spells/SpellInfo.cpp` |
| SpellInfoCorrections.cpp | `src/server/game/Spells/SpellInfoCorrections.cpp` |
| SpellDefines.h | `src/server/game/Spells/SpellDefines.h` |
| SpellEffects.cpp | `src/server/game/Spells/SpellEffects.cpp` |
| SpellMgr.h | `src/server/game/Spells/SpellMgr.h` |
| SpellMgr.cpp | `src/server/game/Spells/SpellMgr.cpp` |
| SpellScript.h | `src/server/game/Spells/SpellScript.h` |
| SpellScript.cpp | `src/server/game/Spells/SpellScript.cpp` |
| SpellAuras.h | `src/server/game/Spells/Auras/SpellAuras.h` |
| SpellAuras.cpp | `src/server/game/Spells/Auras/SpellAuras.cpp` |
| SpellAuraEffects.h | `src/server/game/Spells/Auras/SpellAuraEffects.h` |
| SpellAuraEffects.cpp | `src/server/game/Spells/Auras/SpellAuraEffects.cpp` |
| SpellAuraDefines.h | `src/server/game/Spells/Auras/SpellAuraDefines.h` |
