# AzerothCore AI 系统深入讲解与配置实践

本文档是 `AI_System_Analysis.md` 的补充材料，深入讲解各类 AI 类型的使用场景、配置方法和最佳实践。

---

## 目录

1. [各类 AI 类型深入对比](#1-各类-ai-类型深入对比)
2. [PassiveAI 深入讲解](#2-passiveai-深入讲解)
3. [CombatAI/CasterAI 深入讲解](#3-combataicasterai-深入讲解)
4. [PetAI 深入讲解](#4-petai-深入讲解)
5. [GuardAI 深入讲解](#5-guardai-深入讲解)
6. [BossAI 深入讲解](#6-bossai-深入讲解)
7. [SmartAI 深入讲解](#7-smartai-深入讲解)
8. [AI 配置实践案例](#8-ai-配置实践案例)
9. [AI 系统调试与问题排查](#9-ai-系统调试与问题排查)
10. [AI 系统性能优化](#10-ai-系统性能优化)

---

## 1. 各类 AI 类型深入对比

### 1.1 AI 类型选择决策树

```
开始配置 AI
    │
    ├─ 是否需要复杂的事件-条件-动作逻辑？
    │   ├─ 是 → 使用 SmartAI（数据库配置）
    │   └─ 否 ↓
    │
    ├─ 是否是副本首领？
    │   ├─ 是 → 使用 BossAI（C++ 脚本）
    │   └─ 否 ↓
    │
    ├─ 是否是宠物？
    │   ├─ 是 → 自动使用 PetAI
    │   └─ 否 ↓
    │
    ├─ 是否需要主动攻击玩家？
    │   ├─ 是 → CombatAI 或 CasterAI
    │   └─ 否 ↓
    │
    ├─ 是否只在被攻击时反击？
    │   ├─ 是 → ReactorAI
    │   └─ 否 ↓
    │
    ├─ 是否是城镇守卫？
    │   ├─ 是 → GuardAI
    │   └─ 否 ↓
    │
    └─ 被动 NPC → PassiveAI
```

### 1.2 AI 类型特性对比表

| AI 类型 | 配置方式 | 复杂度 | 性能开销 | 适用场景 | 是否需要 C++ | 是否支持数据库配置 |
|---------|---------|--------|---------|-----------|-------------|------------------|
| **PassiveAI** | AIName | 低 | 极低 | 商人、任务 NPC | 否 | 是 |
| **ReactorAI** | AIName | 低 | 低 | 被动怪物 | 否 | 是 |
| **AggressorAI** | AIName | 低 | 低 | 主动怪物 | 否 | 是 |
| **CombatAI** | AIName + 法术列表 | 中 | 中 | 普通战斗怪物 | 否 | 是 |
| **CasterAI** | AIName + 法术列表 | 中 | 中 | 施法怪物 | 否 | 是 |
| **GuardAI** | AIName | 中 | 中 | 城镇守卫 | 否 | 是 |
| **PetAI** | 自动选择 | 高 | 中 | 所有宠物 | 否 | 否 |
| **BossAI** | C++ 脚本 | 高 | 高 | 副本首领 | 是 | 部分 |
| **SmartAI** | 数据库脚本 | 高 | 中高 | 复杂行为 NPC | 否 | 是 |
| **npc_escortAI** | C++ 脚本 | 中 | 中 | 护送任务 | 是 | 部分 |
| **FollowerAI** | C++ 脚本 | 中 | 中 | 跟随 NPC | 是 | 部分 |

### 1.3 权限优先级详解

```cpp
enum Permitions : int32
{
    PERMIT_BASE_NO                 = -1,   // 不允许（NullCreatureAI）
    PERMIT_BASE_IDLE              = 1,    // 空闲 AI（PassiveAI, CritterAI）
    PERMIT_BASE_REACTIVE         = 100,   // 反应型 AI（ReactorAI, AggressorAI）
    PERMIT_BASE_PROACTIVE        = 200,   // 主动型 AI（CombatAI, CasterAI）
    PERMIT_BASE_FACTION_SPECIFIC = 400,   // 阵营特定 AI（GuardAI）
    PERMIT_BASE_SPECIAL          = 800    // 特殊 AI（PetAI, SmartAI, TotemAI）
};
```

**选择逻辑**：
1. 系统遍历所有注册的 AI 工厂
2. 调用每个工厂的 `Permit(creature)` 方法
3. 选择返回值最高的 AI
4. 如果多个 AI 返回值相同，后注册的 AI 优先

**自定义 AI 示例**：

```cpp
// 自定义 AI 类
class MyCustomAI : public CreatureAI
{
public:
    explicit MyCustomAI(Creature* c) : CreatureAI(c) {}
    
    // 静态方法：返回优先级
    static int32 Permissible(Creature const* creature)
    {
        // 示例：只对 entry=12345 的生物使用此 AI
        if (creature->GetEntry() == 12345)
            return PERMIT_BASE_FACTION_SPECIFIC + 50;  // 高于 GuardAI
        
        return PERMIT_BASE_NO;
    }
    
    void UpdateAI(uint32 diff) override
    {
        // 自定义 AI 逻辑
    }
};

// 注册 AI
class MyCustomAIFactory : public CreatureAIFactory<MyCustomAI>
{
public:
    MyCustomAIFactory() : CreatureAIFactory<MyCustomAI>("MyCustomAI") {}
};

// 在 AIRegistry::Initialize() 中注册
(new MyCustomAIFactory())->RegisterSelf();
```

---

## 2. PassiveAI 深入讲解

### 2.1 适用场景

PassiveAI 适用于以下类型的 NPC：
- **商人**（Vendor）：只提供买卖功能
- **任务发布者**（Quest Giver）：只提供任务对话
- **训练师**（Trainer）：提供技能训练
- **银行职员**、**旅店老板**等
- **剧情 NPC**：只参与对话，不参与战斗

### 2.2 配置方法

#### 方法 1：通过数据库 AIName 字段

```sql
-- 设置 NPC 使用 PassiveAI
UPDATE `creature_template` 
SET `AIName` = 'PassiveAI' 
WHERE `entry` = 12345;
```

#### 方法 2：通过 C++ 脚本指定

```cpp
class MyNPCAI : public PassiveAI
{
public:
    explicit MyNPCAI(Creature* c) : PassiveAI(c) {}
    
    // 可以重写方法添加自定义行为
    void Reset() override
    {
        // 初始化逻辑
    }
};

// 在脚本注册时使用
CreatureAI* GetAI(Creature* creature) override
{
    return new MyNPCAI(creature);
}
```

### 2.3 派生类详解

#### 2.3.1 CritterAI - 小动物 AI

```cpp
class CritterAI : public PassiveAI
{
public:
    explicit CritterAI(Creature* c) : PassiveAI(c) {}
    
    // 被攻击时逃跑
    void JustEngagedWith(Unit* who) override
    {
        me->GetMotionMaster()->MoveFleeing(who);
    }
    
    // 脱战后清理逃跑状态
    void EnterEvadeMode(EvadeReason why) override
    {
        me->ClearUnitState(UNIT_STATE_FLEEING);
        PassiveAI::EnterEvadeMode(why);
    }
    
    // 逃跑完成后的回调
    void MovementInform(uint32 type, uint32 id) override;
    
    // 更新逻辑（空实现）
    void UpdateAI(uint32 diff) override {}
    
    static int32 Permissible(Creature const* creature);
};
```

**行为特点**：
1. **被动触发**：继承自 PassiveAI，不会主动攻击
2. **受惊逃跑**：被攻击时立即逃跑（不是看到敌人时逃跑）
3. **自动返回**：脱战后清除逃跑状态，返回原位置

**配置示例**：

```sql
-- 小动物使用 CritterAI
UPDATE `creature_template` 
SET `AIName` = 'CritterAI' 
WHERE `entry` IN (998, 999, 1000);  -- 松鼠、兔子等
```

#### 2.3.2 NullCreatureAI - 空 AI

```cpp
class NullCreatureAI : public CreatureAI
{
public:
    explicit NullCreatureAI(Creature* c) : CreatureAI(c) {}
    
    // 所有方法都是空的
    void UpdateAI(uint32) override {}
    void OnCharmed(bool) override {}
};
```

**用途**：完全禁用 AI（如：场景中装饰性的生物）

### 2.4 最佳实践

1. **商人 NPCs**：使用 PassiveAI + `npcflag` 设置
2. **剧情 NPCs**：使用 PassiveAI + SmartAI 组合（对话用 SmartScripts）
3. **避免使用 NullCreatureAI**：除非确实需要完全禁用 AI

---

## 3. CombatAI/CasterAI 深入讲解

### 3.1 CombatAI 详解

CombatAI 从数据库读取法术列表，自动在战斗中施放。

#### 3.1.1 法术列表配置

**重要**: CombatAI 本身不自动加载法术，需要配合以下方式使用：

##### 方式 1：使用 SmartScripts（推荐）

```sql
-- 为生物添加战斗法术
INSERT INTO smart_scripts 
(entryorguid, source_type, id, event_type, event_param1, event_param2, 
 action_type, action_param1, target_type, comment)
VALUES
(12345, 0, 0, 0, 5000, 8000,  -- 事件：每 5-8 秒
 11, 12345, 2,  -- 动作：施放法术 12345，目标：当前受害者
 'CombatAI - In Combat - Cast Spell');
```

##### 方式 2：继承 CombatAI 并手动添加法术

```cpp
class MyCombatAI : public CombatAI
{
public:
    MyCombatAI(Creature* c) : CombatAI(c) 
    {
        // 手动添加法术
        spells.push_back(SPELL_FIREBALL);
        spells.push_back(SPELL_FROST_NOVA);
    }
    
    void JustEngagedWith(Unit* who) override
    {
        // 调度法术事件
        events.ScheduleEvent(0, 5s);  // 第一个法术
        events.ScheduleEvent(1, 10s); // 第二个法术
    }
    
    void ExecuteEvent(uint32 eventId) override
    {
        if (eventId < spells.size())
        {
            DoCastVictim(spells[eventId]);
            events.ScheduleEvent(eventId, 8s);  // 重新调度
        }
    }
};
```

##### 方式 3：使用 CasterAI（自动保持距离）

```cpp
class MyCasterAI : public CasterAI
{
public:
    MyCasterAI(Creature* c) : CasterAI(c) 
    {
        // CasterAI 会自动保持距离
        spells.push_back(SPELL_FIREBALL);
    }
};
```

**注意**: `creature_template_addon.auras` 字段用于添加被动光环，不是用于 CombatAI 的法术列表。

#### 3.1.2 EventMap 使用

CombatAI 使用 EventMap 调度法术施放：

```cpp
void CombatAI::InitializeAI()
{
    // 从数据库或脚本读取法术列表
    LoadSpellsFromDB();
    
    // 调度法术事件
    for (uint32 i = 0; i < spells.size(); ++i)
    {
        events.ScheduleEvent(i, spellCooldown[i] * IN_MILLISECONDS);
    }
}

void CombatAI::UpdateAI(uint32 diff)
{
    events.Update(diff);
    
    if (!UpdateVictim())
        return;
    
    // 执行到期的法术事件
    if (uint32 eventId = events.ExecuteEvent())
    {
        DoCastVictim(spells[eventId]);
        events.ScheduleEvent(eventId, spellCooldown[eventId] * IN_MILLISECONDS);
    }
    
    DoMeleeAttackIfReady();
}
```

### 3.2 CasterAI 详解

CasterAI 继承自 CombatAI，增加了保持距离的逻辑。

#### 3.2.1 保持距离机制

CasterAI 使用 `AttackStartCaster` 方法保持距离：

```cpp
class CasterAI : public CombatAI
{
public:
    explicit CasterAI(Creature* c) : CombatAI(c) 
    { 
        m_attackDist = MELEE_RANGE;  // 默认为近战范围
    }
    
    void InitializeAI() override
    {
        CombatAI::InitializeAI();
        // 根据法术范围调整攻击距离
        for (uint32 spellId : spells)
        {
            SpellInfo const* spellInfo = sSpellMgr->GetSpellInfo(spellId);
            if (spellInfo)
            {
                float maxRange = spellInfo->GetMaxRange();
                if (maxRange > m_attackDist)
                    m_attackDist = maxRange;
            }
        }
    }
    
    void AttackStart(Unit* victim) override 
    { 
        AttackStartCaster(victim, m_attackDist);  // 保持指定距离
    }
    
private:
    float m_attackDist;
};
```

**工作原理**：
1. 初始化时，根据法术列表计算最大射程
2. 攻击时，调用 `AttackStartCaster` 保持距离
3. 如果目标太近，会自动后退到射程内

#### 3.2.2 配置示例

```sql
-- 配置一个法师怪物
UPDATE `creature_template` 
SET `AIName` = 'CasterAI',
    `unit_flags` = `unit_flags` | 32768  -- 施法者标志
WHERE `entry` = 12346;

-- 添加法术到 creature_template_addon（不推荐，建议用 SmartScripts）
-- 或者使用 SmartScripts 配置法术施放逻辑
```

### 3.3 ArcherAI 详解

ArcherAI 用于远程物理攻击的怪物（弓箭手、枪手等）。

```cpp
void ArcherAI::AttackStart(Unit* target)
{
    // 保持距离攻击
    if (me->GetDistance(target) < 15.0f)
    {
        me->GetMotionMaster()->MoveChase(target, 20.0f);
    }
    else
    {
        me->GetMotionMaster()->MoveChase(target, 5.0f);
    }
}
```

### 3.4 最佳实践

1. **普通怪物**：使用 CombatAI（近战+法术）
2. **施法怪物**：使用 CasterAI（自动保持距离）
3. **远程怪物**：使用 ArcherAI（弓箭手、枪手）
4. **固定位置攻击**：使用 TurretAI（炮塔、防御塔）
5. **法术配置**：优先使用 SmartScripts，避免使用废弃的 `auras` 字段

---

## 4. PetAI 深入讲解

### 4.1 PetAI 强制选择机制

**重要**: PetAI 是强制选择的，**无法被覆盖**。

#### 4.1.1 选择逻辑

```cpp
CreatureAI* SelectAI(Creature* creature)
{
    // 宠物强制使用 PetAI，忽略所有其他设置
    if (creature->IsPet())
        return sCreatureAIRegistry->GetRegistryItem("PetAI")->Create(creature);
    
    // ... 其他逻辑
}
```

#### 4.1.2 实际行为

| 设置 | 是否生效 | 说明 |
|------|---------|------|
| AIName = 'SmartAI' | ❌ 不生效 | 被强制覆盖为 PetAI |
| ScriptName = 'custom_pet_ai' | ❌ 不生效 | 被强制覆盖为 PetAI |
| AIName = '' | ✅ 生效 | 仍然使用 PetAI |
| 不设置任何字段 | ✅ 生效 | 仍然使用 PetAI |

#### 4.1.3 为什么这样设计？

宠物有特殊的行为模式（跟随主人、自动攻击、法术施放），必须使用 PetAI 才能正确工作。如果允许自定义 AI，会破坏宠物的基本功能。

**如果需要自定义宠物行为**：
- 继承 PetAI 类
- 在子类中重写特定方法
- 但不要改变基本的攻击/跟随逻辑

### 4.2 PetAI 行为模式

#### 4.2.1 攻击模式

宠物有四种攻击模式（通过 `PetAction` 设置）：

```cpp
enum PetAction
{
    PET_ACTION_STAY    = 0,  // 停留
    PET_ACTION_FOLLOW  = 1,  // 跟随
    PET_ACTION_ATTACK  = 2,  // 攻击
    PET_ACTION_DISMISS = 3   // 解散
};
```

#### 4.2.2 PetAI 更新逻辑

```cpp
void PetAI::UpdateAI(uint32 diff)
{
    // 1. 更新计时器
    i_tracker.Update(diff);
    
    // 2. 如果没有目标，跟随主人
    if (!me->GetVictim())
    {
        if (me->GetOwner() && !me->IsInRange(me->GetOwner(), 0.0f, 5.0f))
        {
            me->GetMotionMaster()->MoveFollow(me->GetOwner(), PET_FOLLOW_DIST, me->GetFollowAngle());
        }
        return;
    }
    
    // 3. 如果在战斗中，执行攻击
    if (me->IsEngaged())
    {
        DoMeleeAttack();
        
        // 施放宠物法术（如小鬼的火焰箭）
        if (i_tracker.Passed())
        {
            DoCastPetSpell();
            i_tracker.Reset(3000);  // 3秒冷却
        }
    }
}
```

### 4.3 宠物法术系统

宠物法术通过 `PetSpell` 系统管理：

```cpp
struct PetSpell
{
    uint32 spellId;       // 法术 ID
    bool active;          // 是否激活
    bool autoCast;        // 是否自动施放
};

// 宠物自动施放法术的逻辑
void PetAI::DoCastPetSpell()
{
    for (PetSpell const& petSpell : me->GetPetSpells())
    {
        if (petSpell.active && petSpell.autoCast)
        {
            // 检查法术施放条件
            if (CanCastPetSpell(petSpell.spellId))
            {
                DoCastVictim(petSpell.spellId);
                break;
            }
        }
    }
}
```

### 4.4 配置宠物

#### 方法 1：通过 `creature_template` 配置

```sql
-- 宠物模板配置
INSERT INTO `creature_template` 
(`entry`, `name`, `AIName`, `ScriptName`, ...)
VALUES
(17252, 'Imp', '', '', ...);  -- AIName 为空，PetAI 会自动选择
```

#### 方法 2：通过 C++ 脚本配置

```cpp
class impAI : public PetAI
{
public:
    explicit impAI(Creature* c) : PetAI(c) {}
    
    void UpdateAI(uint32 diff) override
    {
        // 小鬼的自定义逻辑
        PetAI::UpdateAI(diff);
    }
};
```

### 4.5 最佳实践

1. **不要手动设置 AIName**：PetAI 会自动选择
2. **宠物法术**：通过 `PetSpell` 系统配置，而不是 AI
3. **宠物行为**：通过 `PetAction` 控制，而不是修改 AI
4. **自定义宠物**：继承自 PetAI，但不要改变基本的攻击/跟随逻辑

---

## 5. GuardAI 深入讲解

### 5.1 GuardAI 工作原理

GuardAI 用于城镇守卫，具有以下特点：
1. **保护城镇**：攻击犯罪玩家（PvP 触发）
2. **响应求助**：玩家发出求助表情时，前往援助
3. **群体协作**：附近的守卫会一起加入战斗
4. **返回岗位**：战斗结束后返回原始位置

### 5.2 GuardAI 核心方法

```cpp
void GuardAI::MoveInLineOfSight(Unit* who)
{
    // 1. 检查是否是犯罪玩家
    if (me->IsHostileTo(who) && who->IsPlayer())
    {
        Player* player = who->ToPlayer();
        
        // 2. 检查玩家是否有犯罪行为（PvP 标志、红名等）
        if (player->HasFlag(PLAYER_FLAGS, PLAYER_FLAGS_CONTESTED_PVP))
        {
            // 3. 攻击罪犯
            AttackStart(who);
        }
    }
}

void GuardAI::ReceiveEmote(Player* player, uint32 textEmote)
{
    // 1. 检查是否是求助表情
    if (textEmote == TEXT_EMOTE_HELPME)
    {
        // 2. 前往玩家位置
        me->GetMotionMaster()->MovePoint(0, player->GetPosition());
    }
}

void GuardAI::JustDied(Unit* killer)
{
    // 1. 通知其他守卫
    me->CallForHelp(50.0f);
    
    // 2. 调用基类方法
    CreatureAI::JustDied(killer);
}
```

### 5.3 配置 GuardAI

```sql
-- 配置城镇守卫
UPDATE `creature_template` 
SET `AIName` = 'GuardAI',
    `faction` = 35,  -- 联盟守卫阵营
    `unit_flags` = `unit_flags` | 512  -- 守卫标志
WHERE `entry` = 68;  -- 暴风城守卫
```

### 5.4 自定义守卫行为

```cpp
class CustomGuardAI : public GuardAI
{
public:
    explicit CustomGuardAI(Creature* c) : GuardAI(c) {}
    
    void JustEngagedWith(Unit* who) override
    {
        // 自定义：呼叫增援
        me->SummonCreature(68, me->GetPosition(), TEMPSUMMON_TIMED_DESPAWN, 60000);
        
        GuardAI::JustEngagedWith(who);
    }
};
```

### 5.5 最佳实践

1. **使用 GuardAI**：所有城镇守卫都应使用 GuardAI
2. **阵营设置**：确保守卫的阵营正确（联盟=35，部落=36）
3. **求助表情**：GuardAI 默认响应 `TEXT_EMOTE_HELPME`
4. **自定义行为**：继承自 GuardAI，不要直接修改 GuardAI

---

## 6. BossAI 深入讲解

### 6.1 BossAI 核心功能

BossAI 是专门为副本首领设计的 AI 基类，提供以下核心功能：
1. **事件调度**：使用 EventMap 调度技能
2. **阶段管理**：支持多阶段战斗
3. **召唤物管理**：使用 SummonList 管理召唤物
4. **脱战处理**：自动处理脱战逻辑
5. **难度适配**：支持 10/25 人、普通/英雄难度

### 6.2 BossAI 完整示例

```cpp
// 首领 Entry ID
const uint32 BOSS_ENTRY = 12345;

// 事件 ID
enum Events
{
    EVENT_CAST_FIREBALL      = 1,
    EVENT_SUMMON_ADDS        = 2,
    EVENT_PHASE_TRANSITION   = 3,
    EVENT_ENRAGE             = 4
};

// 阶段 ID
enum Phases
{
    PHASE_ONE   = 1,
    PHASE_TWO   = 2,
    PHASE_THREE = 3
};

class BossAI_Example : public BossAI
{
public:
    BossAI_Example(Creature* creature) : BossAI(creature, BOSS_ENTRY) {}
    
    void JustEngagedWith(Unit* who) override
    {
        // 1. 调用基类方法
        BossAI::JustEngagedWith(who);
        
        // 2. 调度第一阶段技能
        events.ScheduleEvent(EVENT_CAST_FIREBALL, 5s);
        events.ScheduleEvent(EVENT_SUMMON_ADDS, 20s);
        
        // 3. 调度阶段转换（血量 50% 时进入第二阶段）
        ScheduleHealthCheckEvent(50, [this]() {
            EnterPhase(PHASE_TWO);
        });
        
        // 4. 调度狂暴（10 分钟后狂暴）
        ScheduleEnrageTimer(12346, 10min);
    }
    
    void EnterPhase(uint32 phase)
    {
        // 1. 设置阶段
        SetPhase(phase);
        
        // 2. 根据阶段调度技能
        switch (phase)
        {
            case PHASE_TWO:
                events.CancelEvent(EVENT_SUMMON_ADDS);
                events.ScheduleEvent(EVENT_CAST_FIREBALL, 3s);  -- 更快的火球
                events.ScheduleEvent(EVENT_PHASE_TRANSITION, 30s);  -- 进入第三阶段
                break;
            case PHASE_THREE:
                events.ScheduleEvent(EVENT_ENRAGE, 0s);  -- 立即狂暴
                break;
        }
    }
    
    void ExecuteEvent(uint32 eventId) override
    {
        switch (eventId)
        {
            case EVENT_CAST_FIREBALL:
                DoCastVictim(SPELL_FIREBALL);
                events.ScheduleEvent(EVENT_CAST_FIREBALL, 8s);  -- 8 秒后再次施放
                break;
                
            case EVENT_SUMMON_ADDS:
                // 召唤小怪
                for (uint32 i = 0; i < 5; ++i)
                {
                    if (Creature* add = DoSummon(ADD_ENTRY, me, 5.0f))
                    {
                        summons.Summon(add);  -- 添加到召唤物列表
                    }
                }
                events.ScheduleEvent(EVENT_SUMMON_ADDS, 30s);
                break;
                
            case EVENT_PHASE_TRANSITION:
                EnterPhase(PHASE_THREE);
                break;
                
            case EVENT_ENRAGE:
                DoCastSelf(SPELL_ENRAGE);
                break;
        }
    }
    
    void JustDied(Unit* killer) override
    {
        // 1. 调用基类方法
        BossAI::JustDied(killer);
        
        // 2. 自定义死亡逻辑（如：触发剧情）
        Talk(EMOTE_DEATH);
    }
    
    void UpdateAI(uint32 diff) override
    {
        // 1. 更新事件调度器
        events.Update(diff);
        
        // 2. 如果不在战斗中，返回
        if (!UpdateVictim())
            return;
        
        // 3. 执行到期的事件
        ExecuteEvent(events.ExecuteEvent());
        
        // 4. 处理生命值检查
        ProcessHealthCheck();
        
        // 5. 更新召唤物
        UpdateSummons(diff);
        
        // 6. 近战攻击
        DoMeleeAttackIfReady();
    }
    
private:
    void UpdateSummons(uint32 diff)
    {
        // 更新所有召唤物（如：小怪跟随主人）
        for (Creature* summon : summons)
        {
            if (summon->IsAlive())
            {
                // 自定义召唤物更新逻辑
            }
        }
    }
};
```

### 6.3 EventMap 高级用法

#### 6.3.1 事件分组

```cpp
// 定义事件组
const uint32 GROUP_FIRE_SPELLS = 1;
const uint32 GROUP_ICE_SPELLS  = 2;

void BossAI_Example::JustEngagedWith(Unit* who)
{
    // 调度事件组
    events.ScheduleEvent(EVENT_CAST_FIREBALL, 5s, GROUP_FIRE_SPELLS);
    events.ScheduleEvent(EVENT_CAST_FIRE_BLAST, 10s, GROUP_FIRE_SPELLS);
    events.ScheduleEvent(EVENT_CAST_ICE_LANCE, 5s, GROUP_ICE_SPELLS);
}

void BossAI_Example::EnterPhase(uint32 phase)
{
    // 取消特定组的事件
    if (phase == PHASE_TWO)
    {
        events.CancelEventGroup(GROUP_FIRE_SPELLS);  -- 取消所有火系法术
    }
}
```

#### 6.3.2 随机时间调度

```cpp
void BossAI_Example::JustEngagedWith(Unit* who)
{
    // 调度随机时间事件（20-30 秒后执行）
    events.ScheduleEvent(EVENT_SUMMON_ADDS, 20s, 30s);
}
```

### 6.4 SummonList 高级用法

```cpp
// 检查召唤物状态
void BossAI_Example::UpdateAI(uint32 diff)
{
    // 如果所有小怪都死了，执行某个动作
    if (summons.empty())
    {
        Talk(EMOTE_ALL_ADDS_DEAD);
    }
    
    // 如果任何小怪在战斗中，将玩家带入战斗
    if (summons.IsAnyCreatureInCombat())
    {
        DoZoneInCombat();
    }
}

// 操作所有召唤物
void BossAI_Example::JustDied(Unit* killer)
{
    // 杀死所有召唤物
    summons.DespawnAll();
    
    // 或者对召唤物执行某个动作
    summons.DoForAllSummons([](WorldObject* summon) {
        if (Creature* creature = summon->ToCreature())
        {
            creature->CastSpell(creature, SPELL_EXPLODE, true);
        }
    });
}
```

### 6.5 难度适配

BossAI 提供多种难度适配方法：

```cpp
void BossAI_Example::JustEngagedWith(Unit* who)
{
    // 方法 1：使用 RAID_MODE 宏
    uint32 fireballDamage = RAID_MODE(1000, 2000, 1500, 3000);  -- 普通10/普通25/英雄10/英雄25
    
    // 方法 2：检查难度
    if (Is25ManRaid())
    {
        // 25 人难度逻辑
    }
    
    if (IsHeroic())
    {
        // 英雄难度逻辑
    }
    
    // 方法 3：使用 DUNGEON_MODE 宏（5 人副本）
    uint32 trashCount = DUNGEON_MODE(3, 5);  -- 普通/英雄
}
```

### 6.6 最佳实践

1. **使用 EventMap**：不要使用 `UpdateAI` 中的计时器，使用 EventMap
2. **管理召唤物**：使用 SummonList 管理所有召唤物
3. **阶段转换**：使用 `ScheduleHealthCheckEvent` 处理阶段转换
4. **难度适配**：使用 `RAID_MODE` 和 `DUNGEON_MODE` 宏
5. **脱战处理**：BossAI 自动处理脱战，不要手动调用 `EnterEvadeMode`
6. **调试**：使用 `Talk` 输出调试信息

---

## 7. SmartAI 深入讲解

SmartAI 是 AzerothCore 最强大的 AI 系统，使用事件-条件-动作模型，完全通过数据库配置。

### 7.1 SmartScripts 事件-条件-动作模型

```
事件 (Event) → 条件 (Condition) → 动作 (Action)
                ↓ (条件不满足)
               不执行动作
```

### 7.2 常用事件类型详解

#### 7.2.1 SMART_EVENT_AGGRO (1) - 获得仇恨

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 0, 100, 0,  -- 事件：获得仇恨
 0, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0, 0, 0,  -- 动作：说话（GroupID=0）
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On Aggro - Say Line 0');
```

**事件参数**：
- `event_param1-6`：通常不使用（取决于事件类型）

#### 7.2.2 SMART_EVENT_HEALTH_PCT (7) - 血量百分比

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 1, 0, 7, 0, 100, 0,  -- 事件：血量百分比
 50, 100, 0, 0, 0, 0,  -- event_param1=50（血量 <= 50% 时触发）
 11, 12345, 0, 0, 0, 0, 0,  -- 动作：施放法术 12345
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On 50% HP - Cast Spell');
```

**事件参数**：
- `event_param1`：血量百分比（<= 此值时触发）
- `event_param2`：比较类型（0=小于等于，1=大于等于）

#### 7.2.3 SMART_EVENT_RANGE (6) - 范围检测

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 2, 0, 6, 0, 100, 0,  -- 事件：范围检测
 5, 20, 0, 0, 0, 0,  -- event_param1=5（最小距离）, event_param2=20（最大距离）
 11, 12345, 0, 0, 0, 0, 0,  -- 动作：施放法术
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On Range 5-20yd - Cast Spell');
```

#### 7.2.4 SMART_EVENT_OOC_LOS (11) - 脱离战斗时视线内

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 3, 0, 11, 0, 100, 0,  -- 事件：脱离战斗时视线内有玩家
 0, 20, 0, 0, 0, 0,  -- event_param1=0（重复触发器）, event_param2=20（最大范围）
 1, 0, 0, 0, 0, 0, 0,  -- 动作：说话
 7, 0, 0, 0, 0,  -- 目标：玩家
 0, 0, 0, 0, 0, 'OOC - Say Greeting');
```

### 7.3 常用动作类型详解

#### 7.3.1 SMART_ACTION_TALK (1) - 说话

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 0, 100, 0,  -- 事件：获得仇恨
 0, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0, 0, 0,  -- 动作：说话（action_param1=GroupID）
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On Aggro - Say Line 0');
```

**配置 creature_text 表**：

```sql
INSERT INTO `creature_text` 
VALUES
(12345, 0, 0, 'I will destroy you!', 12, 0, 100, 0, 0, 0, 0, 0, 'Aggro Line 1'),
(12345, 0, 1, 'You are doomed!', 12, 0, 100, 0, 0, 0, 0, 0, 'Aggro Line 2');
-- GroupID=0 有两条文本，随机选择一条
```

#### 7.3.2 SMART_ACTION_CAST (11) - 施放法术

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 0, 100, 0,  -- 事件：获得仇恨
 0, 0, 0, 0, 0, 0,
 11, 12345, 0, 0, 0, 0, 0,  -- 动作：施放法术（action_param1=法术 ID）
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On Aggro - Cast Spell');
```

**动作参数**：
- `action_param1`：法术 ID
- `action_param2`：施法标志（0=正常，1=触发，2=电击）

#### 7.3.3 SMART_ACTION_SUMMON (13) - 召唤生物

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 0, 100, 0,  -- 事件：获得仇恨
 0, 0, 0, 0, 0, 0,
 13, 54321, 0, 0, 0, 0, 0,  -- 动作：召唤生物（action_param1=生物 Entry）
 0, 0, 0, 0, 0,  -- 目标：自己
 0, 0, 0, 0, 0, 'On Aggro - Summon Add');
```

#### 7.3.4 SMART_ACTION_SET_PHASE (22) - 设置阶段

```sql
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 7, 0, 100, 0,  -- 事件：血量 50%
 50, 0, 0, 0, 0, 0,
 22, 2, 0, 0, 0, 0, 0,  -- 动作：设置阶段 2
 0, 0, 0, 0, 0,  -- 目标：自己
 0, 0, 0, 0, 0, 'On 50% HP - Enter Phase 2');
```

### 7.4 常用目标类型详解

#### 7.4.1 SMART_TARGET_VICTIM (1) - 当前受害者

```sql
-- 对当前受害者执行动作
`target_type` = 1  -- SMART_TARGET_VICTIM
```

#### 7.4.2 SMART_TARGET_HOSTILE_RANDOM (4) - 随机敌对目标

```sql
-- 对随机敌对目标执行动作
`target_type` = 4  -- SMART_TARGET_HOSTILE_RANDOM
`target_param1` = 0  -- 不包括当前受害者（0=否，1=是）
```

#### 7.4.3 SMART_TARGET_CLOSEST_PLAYER (20) - 最近玩家

```sql
-- 对最近的玩家执行动作
`target_type` = 20  -- SMART_TARGET_CLOSEST_PLAYER
`target_param1` = 50  -- 最大距离（码）
```

### 7.5 条件系统（conditions 表）

SmartScripts 支持通过 `conditions` 表添加条件判断。

**示例**：只有当玩家有 Buff 时才施放法术

```sql
-- 1. 配置 SmartScripts 事件
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 0, 100, 0,  -- 事件：获得仇恨
 0, 0, 0, 0, 0, 0,
 11, 12345, 0, 0, 0, 0, 0,  -- 动作：施放法术
 1, 0, 0, 0, 0,  -- 目标：当前受害者
 0, 0, 0, 0, 0, 'On Aggro - Cast Spell (Conditional)');

-- 2. 配置条件（只有当目标有 Buff 54321 时才执行）
INSERT INTO `conditions` 
VALUES
('SmartScripts', 12345, 0, 0, 0,  -- SourceType=13 (SmartScripts), 对应 smart_scripts 的 entryorguid
 16, 0, 54321, 0, 0, 0, 0, 0, 0, 0, 0,  -- ConditionType=16 (Target Has Aura), AuraID=54321
 '', 'Target has Aura 54321');
```

**ConditionType 常用值**：
- `0`：无（总是满足）
- `1`：法术施放（Aura）
- `16`：目标有光环
- `24`：玩家有任务
- `28`：玩家种族

### 7.6 高级技巧

#### 7.6.1 链接事件（Link）

```sql
-- 事件 A 执行后，自动执行事件 B（通过 link 字段）
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 1, 1, 0, 100, 0,  -- 事件 0：获得仇恨，链接到事件 1
 0, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0, 0, 0,  -- 动作：说话
 1, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'On Aggro - Say Line 0'),

(-12345, 0, 1, 0, 0, 0, 100, 0,  -- 事件 1：被事件 0 链接调用
 0, 0, 0, 0, 0, 0,
 11, 12345, 0, 0, 0, 0, 0,  -- 动作：施放法术
 1, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'Linked - Cast Spell');
```

#### 7.6.2 阶段系统（Phase）

```sql
-- 只有当前阶段匹配时，事件才会触发
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 1, 1, 100, 0,  -- 事件：获得仇恨，只在阶段 1 触发（event_phase_mask=1）
 0, 0, 0, 0, 0, 0,
 11, 12345, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'Phase 1 - Cast Spell'),

(-12345, 0, 1, 0, 1, 2, 100, 0,  -- 事件：获得仇恨，只在阶段 2 触发（event_phase_mask=2）
 0, 0, 0, 0, 0, 0,
 11, 54321, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'Phase 2 - Cast Spell');
```

#### 7.6.3 计时器事件（SMART_EVENT_UPDATE）

```sql
-- 定时执行（每 5 秒）
INSERT INTO `smart_scripts` 
VALUES
(-12345, 0, 0, 0, 0, 0, 100, 0,  -- 事件：UPDATE（定时）
 5000, 5000, 0, 0, 0, 0,  -- event_param1=5000（初始延迟 5 秒）, event_param2=5000（重复间隔 5 秒）
 11, 12345, 0, 0, 0, 0, 0,
 1, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'Every 5s - Cast Spell');
```

### 7.7 最佳实践

1. **优先使用 SmartAI**：大多数 NPC 行为都可以通过 SmartScripts 实现
2. **事件设计**：一个事件只做一件事，使用 `link` 连接多个动作
3. **阶段管理**：使用阶段系统管理复杂行为
4. **条件判断**：使用 `conditions` 表添加复杂条件
5. **调试**：使用 `SMART_ACTION_TALK` 输出调试信息
6. **性能**：避免在 `SMART_EVENT_UPDATE` 中使用高频率（< 1s）

---

## 8. AI 配置实践案例

### 8.1 案例 1：简单的巡逻守卫

**需求**：创建一个在固定路径巡逻的守卫，发现敌人时攻击，战斗结束后返回巡逻路径。

**实现步骤**：

#### 步骤 1：创建巡逻路径

```sql
-- 创建路径 ID=1001 的巡逻路径
INSERT INTO `waypoints` 
(`entry`, `pointid`, `position_x`, `position_y`, `position_z`, `orientation`, `delay`, `comment`)
VALUES
(1001, 1, -10616.7, -1150.73, 28.0361, NULL, 5000, 'Guard Patrol - Point 1'),
(1001, 2, -10609.4, -1154.94, 28.2175, NULL, 0, 'Guard Patrol - Point 2'),
(1001, 3, -10605.3, -1157.31, 30.007, NULL, 3000, 'Guard Patrol - Point 3'),
(1001, 4, -10600.3, -1159.58, 30.0602, NULL, 0, 'Guard Patrol - Point 4');
```

#### 步骤 2：配置生物

```sql
-- 配置守卫生物
UPDATE `creature_template` 
SET `AIName` = 'SmartAI',  -- 使用 SmartAI
    `MovementType` = 2,     -- 路径移动
    `faction` = 35          -- 联盟阵营
WHERE `entry` = 12345;

-- 配置生物的附加数据（路径 ID）
INSERT INTO `creature_addon` 
(`guid`, `path_id`, `mount`, `bytes1`, `bytes2`, `emote`, `visibilityDistanceType`, `auras`)
VALUES
(123456, 1001, 0, 0, 0, 0, 0, NULL);  -- guid=123456 的生物使用路径 1001
```

#### 步骤 3：配置 SmartScripts

```sql
-- 配置守卫的 AI 行为
INSERT INTO `smart_scripts` 
(`entryorguid`, `source_type`, `id`, `link`, `event_type`, `event_phase_mask`, `event_chance`, `event_flags`, 
 `event_param1`, `event_param2`, `event_param3`, `event_param4`, `event_param5`, `event_param6`,
 `action_type`, `action_param1`, `action_param2`, `action_param3`, `action_param4`, `action_param5`, `action_param6`,
 `target_type`, `target_param1`, `target_param2`, `target_param3`, `target_param4`,
 `target_x`, `target_y`, `target_z`, `target_o`, `comment`)
VALUES
-- 事件 1：获得仇恨时，停止路径移动，开始攻击
(-12345, 0, 0, 0, 1, 0, 100, 0,
 0, 0, 0, 0, 0, 0,
 20, 0, 0, 0, 0, 0, 0,  -- 动作：停止路径移动
 0, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'On Aggro - Stop Path'),

-- 事件 2：脱战时，恢复路径移动
(-12345, 0, 1, 0, 4, 0, 100, 0,
 0, 0, 0, 0, 0, 0,
 21, 0, 0, 0, 0, 0, 0,  -- 动作：恢复路径移动
 0, 0, 0, 0, 0,
 0, 0, 0, 0, 0, 'On Evade - Resume Path');
```

### 8.2 案例 2：多阶段首领战斗

**需求**：创建一个多阶段首领，血量 75% 和 50% 时进入新相位，使用不同技能。

**实现步骤**：

#### 步骤 1：配置首领模板

```sql
UPDATE `creature_template` 
SET `AIName` = '',  -- 不使用 SmartAI，使用 C++ 脚本
    `ScriptName` = 'boss_example'  -- C++ 脚本名
WHERE `entry` = 12345;
```

#### 步骤 2：编写 C++ 脚本

```cpp
class boss_example : public CreatureScript
{
public:
    boss_example() : CreatureScript("boss_example") {}
    
    CreatureAI* GetCreatureAI(Creature* creature) const override
    {
        return new boss_exampleAI(creature);
    }
    
    struct boss_exampleAI : public BossAI
    {
        boss_exampleAI(Creature* creature) : BossAI(creature, BOSS_ENTRY) {}
        
        void JustEngagedWith(Unit* who) override
        {
            BossAI::JustEngagedWith(who);
            
            // 第一阶段技能
            events.ScheduleEvent(EVENT_CAST_FIREBALL, 5s);
            events.ScheduleEvent(EVENT_SUMMON_ADDS, 20s);
            
            // 阶段转换
            ScheduleHealthCheckEvent(75, [this]() { EnterPhase(2); });
            ScheduleHealthCheckEvent(50, [this]() { EnterPhase(3); });
        }
        
        void EnterPhase(uint32 phase)
        {
            SetPhase(phase);
            
            switch (phase)
            {
                case 2:
                    Talk(EMOTE_PHASE_2);
                    events.CancelEvent(EVENT_SUMMON_ADDS);
                    events.ScheduleEvent(EVENT_CAST_FROSTBOLT, 3s);
                    break;
                case 3:
                    Talk(EMOTE_PHASE_3);
                    events.ScheduleEvent(EVENT_ENRAGE, 0s);
                    break;
            }
        }
        
        void ExecuteEvent(uint32 eventId) override
        {
            switch (eventId)
            {
                case EVENT_CAST_FIREBALL:
                    DoCastVictim(SPELL_FIREBALL);
                    events.ScheduleEvent(EVENT_CAST_FIREBALL, 8s);
                    break;
                case EVENT_CAST_FROSTBOLT:
                    DoCastVictim(SPELL_FROSTBOLT);
                    events.ScheduleEvent(EVENT_CAST_FROSTBOLT, 6s);
                    break;
                case EVENT_SUMMON_ADDS:
                    DoSummon(ADD_ENTRY, me, 5.0f);
                    events.ScheduleEvent(EVENT_SUMMON_ADDS, 30s);
                    break;
                case EVENT_ENRAGE:
                    DoCastSelf(SPELL_ENRAGE);
                    break;
            }
        }
    };
};
```

### 8.3 案例 3：复杂的任务 NPC

**需求**：创建一个任务 NPC，玩家接受任务后，NPC 开始跟随玩家，到达目的地后完成任务。

**实现步骤**：

#### 步骤 1：配置 NPC

```sql
UPDATE `creature_template` 
SET `AIName` = 'SmartAI',
    `ScriptName` = ''  -- 不使用 C++ 脚本
WHERE `entry` = 12345;
```

#### 步骤 2：配置 SmartScripts

```sql
INSERT INTO `smart_scripts` 
VALUES
-- 事件 1：玩家接受任务时，开始跟随玩家
(-12345, 0, 0, 0, 16, 0, 100, 0,  -- 事件：接受任务（event_type=16）
 12345, 0, 0, 0, 0, 0,  -- event_param1=任务 ID
 25, 0, 5.0, 0, 0, 0, 0,  -- 动作：跟随玩家（action_type=25, action_param1=距离）
 7, 0, 0, 0, 0,  -- 目标：动作调用者（玩家）
 0, 0, 0, 0, 0, 'On Accept Quest - Start Follow'),

-- 事件 2：玩家到达目的地时，完成任务
(-12345, 0, 1, 0, 11, 0, 100, 0,  -- 事件：脱离战斗时视线内有玩家（用于检测玩家位置）
 0, 10, 0, 0, 0, 0,  -- event_param2=10（最大距离 10 码）
 17, 0, 0, 0, 0, 0, 0,  -- 动作：完成任务（action_type=17）
 7, 0, 0, 0, 0,  -- 目标：动作调用者（玩家）
 100.5, 200.3, 30.0, 0, 0, 'On Reach Destination - Complete Quest');
```

**注意**：此案例需要配合 `conditions` 表，确保只在玩家有特定任务时才触发。

---

## 9. AI 系统调试与问题排查

### 9.1 常用调试方法

#### 9.1.1 使用 Talk 输出调试信息

```cpp
void MyAI::UpdateAI(uint32 diff)
{
    // 输出调试信息（只在调试版本）
#ifdef DEBUG
    me->Talk(0);  -- 播放 creature_text 中 GroupID=0 的文本
#endif
    
    // 正常逻辑
    // ...
}
```

#### 9.1.2 使用 LOG_DEBUG 输出日志

```cpp
#include "Log.h"

void MyAI::UpdateAI(uint32 diff)
{
    // 输出调试日志（需要启用 debug 日志级别）
    LOG_DEBUG("scripts", "MyAI::UpdateAI - diff=%u, victim=%s", 
              diff, me->GetVictim() ? me->GetVictim()->GetName() : "nullptr");
    
    // 正常逻辑
    // ...
}
```

#### 9.1.3 使用 GM 命令调试

```
# 查看生物 AI 状态
.npc ai info

# 查看生物的威胁列表
.threat list

# 查看生物的当前状态
.npc info
```

### 9.2 常见问题排查

#### 问题 1：AI 不工作

**可能原因**：
1. `AIName` 或 `ScriptName` 配置错误
2. C++ 脚本没有注册
3. AI 选择逻辑错误

**排查步骤**：

```sql
-- 1. 检查 AIName 和 ScriptName
SELECT `entry`, `name`, `AIName`, `ScriptName` 
FROM `creature_template` 
WHERE `entry` = 12345;

-- 2. 检查生物是否生成
SELECT `guid`, `id`, `position_x`, `position_y`, `position_z` 
FROM `creature` 
WHERE `id` = 12345;
```

```cpp
// 3. 在 C++ 代码中添加日志
CreatureAI* FactorySelector::SelectAI(Creature* creature)
{
    LOG_DEBUG("scripts", "SelectAI for creature entry=%u, AIName=%s, ScriptName=%s",
              creature->GetEntry(), creature->GetAIName().c_str(), creature->GetScriptName().c_str());
    
    // ... 选择逻辑
}
```

#### 问题 2：SmartScripts 不触发

**可能原因**：
1. `smart_scripts` 表配置错误
2. 事件参数配置错误
3. 条件不满足

**排查步骤**：

```sql
-- 1. 检查 smart_scripts 配置
SELECT * FROM `smart_scripts` 
WHERE `entryorguid` = -12345 AND `source_type` = 0;

-- 2. 检查条件配置
SELECT * FROM `conditions` 
WHERE `SourceTypeOrReferenceId` = 13 AND `SourceGroup` = 12345;

-- 3. 启用 SmartScripts 调试日志
-- 在 worldserver.conf 中设置：
-- App.Logger.scripts = Debug
```

#### 问题 3：生物不移动

**可能原因**：
1. `MovementType` 配置错误
2. 路径点配置错误
3. 运动主逻辑错误

**排查步骤**：

```sql
-- 1. 检查 MovementType
SELECT `entry`, `name`, `MovementType` 
FROM `creature_template` 
WHERE `entry` = 12345;

-- 2. 检查路径点配置
SELECT * FROM `waypoints` 
WHERE `entry` = (SELECT `path_id` FROM `creature_addon` WHERE `guid` = 123456);

-- 3. 检查生物附加数据
SELECT * FROM `creature_addon` 
WHERE `guid` = 123456;
```

### 9.3 性能优化建议

1. **避免使用高频率的 SMART_EVENT_UPDATE**（< 1s）
2. **减少不必要的 Talk 调用**（会播放声音）
3. **使用 EventMap 而不是手动计时器**
4. **在 UpdateAI 中避免复杂计算**
5. **使用 SummonList 管理召唤物，避免遍历**

---

## 10. AI 系统性能优化

### 10.1 事件调度优化

#### 10.1.1 使用 EventMap 而不是手动计时器

**不推荐**：

```cpp
class MyAI : public CreatureAI
{
    uint32 _timer;
    
    void UpdateAI(uint32 diff) override
    {
        _timer += diff;
        if (_timer >= 5000)  // 5 秒
        {
            DoCastVictim(SPELL_FIREBALL);
            _timer = 0;
        }
    }
};
```

**推荐**：

```cpp
class MyAI : public CreatureAI
{
    EventMap events;
    
    void UpdateAI(uint32 diff) override
    {
        events.Update(diff);
        
        if (uint32 eventId = events.ExecuteEvent())
        {
            DoCastVictim(SPELL_FIREBALL);
            events.ScheduleEvent(eventId, 5s);
        }
    }
};
```

#### 10.1.2 使用事件组管理相关事件

```cpp
const uint32 GROUP_FIRE_SPELLS = 1;

void MyAI::JustEngagedWith(Unit* who)
{
    // 调度事件组
    events.ScheduleEvent(EVENT_CAST_FIREBALL, 5s, GROUP_FIRE_SPELLS);
    events.ScheduleEvent(EVENT_CAST_FIRE_BLAST, 10s, GROUP_FIRE_SPELLS);
}

void MyAI::EnterPhase(uint32 phase)
{
    // 取消整个事件组
    if (phase == PHASE_TWO)
    {
        events.CancelEventGroup(GROUP_FIRE_SPELLS);
    }
}
```

### 10.2 目标选择优化

#### 10.2.1 使用 SelectTarget 而不是遍历威胁列表

**不推荐**：

```cpp
void MyAI::UpdateAI(uint32 diff)
{
    // 遍历威胁列表
    ThreatList const& threatList = me->GetThreatMgr().GetSortedThreatList();
    for (auto itr : threatList)
    {
        if (itr->getUnit()->GetHealthPct() < 50)
        {
            DoCast(itr->getUnit(), SPELL_HEAL);
            break;
        }
    }
}
```

**推荐**：

```cpp
void MyAI::UpdateAI(uint32 diff)
{
    // 使用 SelectTarget
    Unit* target = SelectTarget(SelectTargetMethod::Random, 0, [this](Unit const* target) {
        return target->GetHealthPct() < 50;
    });
    
    if (target)
    {
        DoCast(target, SPELL_HEAL);
    }
}
```

### 10.3 内存优化

#### 10.3.1 使用对象池

对于频繁创建/销毁的对象（如召唤物），使用对象池：

```cpp
// 不推荐：频繁创建/销毁
void MyAI::ExecuteEvent(uint32 eventId)
{
    if (eventId == EVENT_SUMMON_ADDS)
    {
        Creature* add = new Creature();  // 创建新对象
        // ...
    }
}

// 推荐：使用对象池（AzerothCore 内部已实现）
void MyAI::ExecuteEvent(uint32 eventId)
{
    if (eventId == EVENT_SUMMON_ADDS)
    {
        Creature* add = DoSummon(ADD_ENTRY, me, 5.0f);  // 使用对象池
        // ...
    }
}
```

### 10.4 网络优化

#### 10.4.1 减少不必要的网络包发送

```cpp
void MyAI::UpdateAI(uint32 diff)
{
    // 不推荐：每帧都发送移动包
    me->GetMotionMaster()->MovePoint(0, position);
    
    // 推荐：只在必要时发送
    if (!me->IsInRange(position, 0.0f, 5.0f))
    {
        me->GetMotionMaster()->MovePoint(0, position);
    }
}
```

---

## 总结

本文档深入讲解了 AzerothCore AI 系统的各类 AI 类型、配置方法和最佳实践。通过使用合适的 AI 类型、遵循最佳实践、进行有效调试和性能优化，可以创建出高效、稳定的游戏 AI。

**关键要点**：
1. **选择合适的 AI 类型**：根据需求选择 PassiveAI、CombatAI、BossAI 或 SmartAI
2. **使用 EventMap**：避免使用手动计时器
3. **优先使用 SmartScripts**：大多数行为可以通过数据库配置实现
4. **有效调试**：使用 Talk、LOG_DEBUG 和 GM 命令
5. **性能优化**：避免高频率事件、减少不必要的计算

---

**文档版本**: 1.0  
**最后更新**: 2025-05-09  
**作者**: AzerothCore 分析工具
