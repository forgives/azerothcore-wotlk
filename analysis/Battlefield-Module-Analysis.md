# AzerothCore Battlefield 模块详细分析

## 1. 模块概览

Battlefield 模块实现了魔兽世界中的户外 PvP 战场系统，当前唯一实现是冬拥湖（Wintergrasp）。与传统 Battleground（战场）不同，Battlefield 是在主世界地图上运行的大规模 PvP 事件，无需排队匹配，区域内的玩家自动参与。模块采用模板方法模式（Template Method），基类 `Battlefield` 定义通用框架，子类 `BattlefieldWG` 实现具体逻辑。

**文件结构：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `Battlefield.h` | 486 | 基类声明：Battlefield、BfCapturePoint、BfGraveyard 及枚举定义 |
| `Battlefield.cpp` | 1092 | 基类完整实现：战斗流程、玩家管理、墓地、据点更新 |
| `BattlefieldMgr.h` | 79 | 全局管理器单例声明 |
| `BattlefieldMgr.cpp` | 132 | 管理器实现：初始化、玩家进出区域、定时更新 |
| `BattlefieldHandler.cpp` | 137 | 网络包处理：队列邀请、参战邀请、退出请求 |
| `Zones/BattlefieldWG.h` | 1486 | 冬拥湖声明：WG 特化类、建筑/工厂数据、NPC/GO 坐标常量 |
| `Zones/BattlefieldWG.cpp` | 1251 | 冬拥湖完整实现：战斗流程、泰坦遗迹、坚韧度、军衔晋升 |

**模块位置：** `src/server/game/Battlefield/`

---

## 2. 枚举与常量

### 2.1 BattlefieldTypes（`Battlefield.h:28-32`）

| 常量 | 值 | 含义 |
|------|---|------|
| `BATTLEFIELD_WG` | 0 | 冬拥湖 |
| `BATTLEFIELD_TB` | 1 | 托巴拉德（大灾变，未实现） |

### 2.2 BattlefieldIDs（`Battlefield.h:34-37`）

| 常量 | 值 | 含义 |
|------|---|------|
| `BATTLEFIELD_BATTLEID_WG` | 1 | 冬拥湖战斗 ID（用于网络包） |

### 2.3 BattlefieldObjectiveStates（`Battlefield.h:39-48`）

据点占领状态机，7 个状态描述从联盟/部落控制到中立的转换：

| 状态 | 值 | 含义 |
|------|---|------|
| `NEUTRAL` | 0 | 中立 |
| `ALLIANCE` | 1 | 联盟控制 |
| `HORDE` | 2 | 部落控制 |
| `NEUTRAL_ALLIANCE_CHALLENGE` | 3 | 中立→联盟争夺 |
| `NEUTRAL_HORDE_CHALLENGE` | 4 | 中立→部落争夺 |
| `ALLIANCE_HORDE_CHALLENGE` | 5 | 联盟→部落争夺 |
| `HORDE_ALLIANCE_CHALLENGE` | 6 | 部落→联盟争夺 |

### 2.4 BattlefieldSounds（`Battlefield.h:50-55`）

| 常量 | 值 | 含义 |
|------|---|------|
| `BF_HORDE_WINS` | 8454 | 部落胜利音效 |
| `BF_ALLIANCE_WINS` | 8455 | 联盟胜利音效 |
| `BF_START` | 3439 | 战斗开始音效 |

### 2.5 BattlefieldTimerGroups（`Battlefield.h:59-64`）

| 常量 | 值 | 含义 |
|------|---|------|
| `BATTLEFIELD_TIMER_GROUP_RESURRECT` | 1 | 复活定时器组 |
| `BATTLEFIELD_TIMER_GROUP_WAR` | 2 | 战斗专用定时器组 |
| `BATTLEFIELD_TIMER_GROUP_SAVE` | 3 | 状态保存定时器组 |

### 2.6 WintergraspSpells（`BattlefieldWG.h:47-90`）

WG 专有法术，分为四类：

**战斗军衔法术：**

| 法术 | ID | 含义 |
|------|---|------|
| `SPELL_RECRUIT` | 37795 | 新兵（5 击升下士） |
| `SPELL_CORPORAL` | 33280 | 下士（5 击升中尉） |
| `SPELL_LIEUTENANT` | 55629 | 中尉（可建造攻城车） |

**平衡/增益法术：**

| 法术 | ID | 含义 |
|------|---|------|
| `SPELL_TENACITY` | 58549 | 坚韧度（人数少方增益） |
| `SPELL_TENACITY_VEHICLE` | 59911 | 载具坚韧度 |
| `SPELL_TOWER_CONTROL` | 62064 | 塔楼控制增益 |

**奖励法术：**

| 法术 | ID | 含义 |
|------|---|------|
| `SPELL_VICTORY_REWARD` | 56902 | 胜利奖励荣誉 |
| `SPELL_DEFEAT_REWARD` | 58494 | 失败奖励荣誉 |
| `SPELL_DAMAGED_TOWER` | 59135 | 损毁塔楼奖励 |
| `SPELL_DESTROYED_TOWER` | 59136 | 摧毁塔楼奖励 |
| `SPELL_DAMAGED_BUILDING` | 59201 | 损坏建筑奖励 |
| `SPELL_INTACT_BUILDING` | 59203 | 完整建筑奖励 |

**相位法术：**

| 法术 | ID | 含义 |
|------|---|------|
| `SPELL_HORDE_CONTROLS_FACTORY_PHASE_SHIFT` | 56618 | 部落控制工厂相位（+Phase 16） |
| `SPELL_ALLIANCE_CONTROLS_FACTORY_PHASE_SHIFT` | 56617 | 联盟控制工厂相位（+Phase 32） |
| `SPELL_HORDE_CONTROL_PHASE_SHIFT` | 55773 | 部落控制全局相位（+Phase 64） |
| `SPELL_ALLIANCE_CONTROL_PHASE_SHIFT` | 55774 | 联盟控制全局相位（+Phase 128） |

### 2.7 WintergraspGameObjectBuildingType（`BattlefieldWG.h:463-471`）

| 类型 | 含义 |
|------|------|
| `BATTLEFIELD_WG_OBJECTTYPE_DOOR` | 要塞大门 |
| `BATTLEFIELD_WG_OBJECTTYPE_TITANRELIC` | 泰坦遗迹 |
| `BATTLEFIELD_WG_OBJECTTYPE_WALL` | 城墙 |
| `BATTLEFIELD_WG_OBJECTTYPE_DOOR_LAST` | 最后一道门（破坏后可点击泰坦遗迹） |
| `BATTLEFIELD_WG_OBJECTTYPE_KEEP_TOWER` | 要塞塔楼 |
| `BATTLEFIELD_WG_OBJECTTYPE_TOWER` | 攻击方塔楼（南方三塔） |

### 2.8 WintergraspData（`BattlefieldWG.h:92-102`）

WG 内部数据索引，用于 `Data32` 向量：

| 索引 | 含义 |
|------|------|
| `BATTLEFIELD_WG_DATA_INTACT_TOWER_ATT` | 攻击方完好塔数 |
| `BATTLEFIELD_WG_DATA_DAMAGED_TOWER_ATT` | 攻击方受损塔数 |
| `BATTLEFIELD_WG_DATA_BROKEN_TOWER_ATT` | 攻击方摧毁塔数 |
| `BATTLEFIELD_WG_DATA_MAX_VEHICLE_A/H` | 最大载具数（联盟/部落） |
| `BATTLEFIELD_WG_DATA_VEHICLE_A/H` | 当前载具数（联盟/部落） |

---

## 3. 类层次结构

```
ZoneScript (接口)
└── Battlefield (基类，核心框架)
    ├── BfCapturePoint (据点占领系统)
    │   └── WintergraspCapturePoint (WG 据点，关联工厂)
    ├── BfGraveyard (墓地系统)
    │   └── BfGraveyardWG (WG 墓地，含 GossipText)
    └── BattlefieldWG (冬拥湖实现)
        ├── BfWGGameObjectBuilding (可破坏建筑：墙/门/塔)
        └── WGWorkshop (工厂：控制载具上限和墓地)

BattlefieldMgr (全局单例 sBattlefieldMgr)
├── _battlefieldSet: vector<Battlefield*>       // 所有战场实例
└── _battlefieldMap: map<zoneId, Battlefield*>   // 区域→战场映射
```

---

## 4. 核心类详解

### 4.1 Battlefield 类（`Battlefield.h:207-483`）

```
Battlefield : ZoneScript
├── 核心状态:
│   ├── Timer: uint32                    // 全局计时器（战斗/非战斗共用）
│   ├── Enabled: bool                    // 是否启用
│   ├── Active: bool                     // 是否战斗中（IsWarTime()）
│   ├── DefenderTeam: TeamId             // 防守方阵营
│   ├── _scheduler: TaskScheduler        // 定时任务调度器
│   │
│   ├── 标识:
│   ├── TypeId: uint32                   // BattlefieldTypes 枚举
│   ├── BattleId: uint32                 // 战斗 ID（网络包用）
│   ├── ZoneId: uint32                   // 区域 ID（WG=4197）
│   ├── MapId: uint32                    // 地图 ID
│   └── BfMap: Map*                      // 地图实例指针
│
├── 玩家管理 (按阵营索引):
│   ├── Players[2]: GuidUnorderedSet              // 区域内所有玩家
│   ├── PlayersInQueue[2]: GuidUnorderedSet       // 队列中玩家
│   ├── PlayersInWar[2]: GuidUnorderedSet         // 参战玩家
│   ├── InvitedPlayers[2]: PlayerTimerMap          // 已邀请参战 (guid→过期时间)
│   ├── PlayersWillBeKick[2]: PlayerTimerMap       // 即将被踢 (guid→过期时间)
│   └── Groups[2]: GuidUnorderedSet               // 战场团队
│
├── 参数配置:
│   ├── MaxPlayer: uint32                // 每阵营最大玩家数
│   ├── MinPlayer: uint32               // 最低开战人数
│   ├── MinLevel: uint32                // 最低参与等级
│   ├── BattleTime: uint32              // 战斗持续时间 (ms)
│   ├── NoWarBattleTime: uint32         // 战斗间隔时间 (ms)
│   ├── RestartAfterCrash: uint32       // 崩溃后重启延迟 (ms)
│   ├── TimeForAcceptInvite: uint32     // 接受邀请超时 (秒)
│   ├── StartGroupingTimer: uint32      // 战前集合计时 (ms)
│   ├── StartGrouping: bool            // 是否已开始集合
│   └── KickPosition: WorldLocation     // 踢出传送位置
│
├── 据点与墓地:
│   ├── CapturePoints: BfCapturePointVector
│   └── GraveyardList: GraveyardVect
│
├── 通用数据:
│   ├── Data64: vector<uint64>
│   ├── Data32: vector<uint32>
│   └── StalkerGuid: ObjectGuid         // 消息广播用 NPC
│
├── 虚函数钩子 (Template Method):
│   ├── SetupBattlefield()              // 初始化（子类实现）
│   ├── OnBattleStart()                 // 战斗开始
│   ├── OnBattleEnd(endByTimer)         // 战斗结束
│   ├── OnStartGrouping()              // 开始集合
│   ├── OnPlayerJoinWar(player)         // 玩家参战
│   ├── OnPlayerLeaveWar(player)        // 玩家离战
│   ├── OnPlayerEnterZone(player)       // 玩家进入区域
│   ├── OnPlayerLeaveZone(player)       // 玩家离开区域
│   ├── SendInitWorldStatesToAll()      // 初始化世界状态
│   ├── FillInitialWorldStates(packet)  // 填充世界状态包
│   └── SendUpdateWorldStates(player)   // 更新世界状态
│
└── 模板辅助方法:
    ├── ForEachPlayerInZone(fn)          // 遍历区域内玩家
    ├── ForEachPlayerInQueue(fn)         // 遍历队列玩家
    ├── ForEachPlayerInWar(fn)           // 遍历参战玩家（双阵营）
    └── ForEachPlayerInWar(team, fn)     // 遍历参战玩家（单阵营）
```

### 4.2 BfCapturePoint 类（`Battlefield.h:85-151`）

据点占领机制的核心实现，控制战场区域的旗帜争夺：

```
BfCapturePoint
├── ActivePlayers[2]: GuidUnorderedSet   // 据点内活跃玩家（按阵营）
├── Value: float                         // 占领进度值 (-MaxValue ~ +MaxValue)
├── Team: TeamId                         // 当前控制方
├── MaxValue: float                      // 占领进度上限
├── MinValue: float                      // 中立阈值 = MaxValue * neutralPercent / 100
├── MaxSpeed: float                      // 最大占领速度
├── OldState / State: BattlefieldObjectiveStates  // 状态机
├── NeutralValuePct: uint32             // 中立百分比
├── CapturePoint: ObjectGuid            // 关联的 GameObject
├── CapturePointEntry: uint32           // GameObject Entry
└── Bf: Battlefield*                    // 所属战场
```

**占领进度算法：**
- `factDiff = (alliancePlayers - hordePlayers) * diff / UPDATE_INTERVAL`
- 联盟人多 → Value 增加（正向），部落人多 → Value 减少（负向）
- 超过 `MinValue` → 联盟控制，低于 `-MinValue` → 部落控制
- 穿过零点时触发状态转换（NEUTRAL_X_CHALLENGE）

### 4.3 BfGraveyard 类（`Battlefield.h:153-205`）

```
BfGraveyard
├── ControlTeam: TeamId              // 控制方阵营
├── GraveyardId: uint32              // 墓地 ID
├── SpiritGuide[2]: ObjectGuid       // 双阵营灵魂医者
├── ResurrectQueue: GuidUnorderedSet // 复活等待队列
└── Bf: Battlefield*                 // 所属战场
```

**关键行为：**
- `GiveControlTo(team)`：变更控制方 + 重定位已死亡玩家到最近友方墓地
- `Resurrect()`：复活队列中所有玩家（由定时器每 RESURRECTION_INTERVAL 触发）
- `RelocateDeadPlayers()`：控制权变更时，将等待中的灵魂传送到新墓地

### 4.4 BattlefieldWG 类（`BattlefieldWG.h:252-454`）

```
BattlefieldWG : Battlefield
├── 战斗核心:
│   ├── IsRelicInteractible: bool         // 泰坦遗迹是否可交互
│   ├── TitansRelic: ObjectGuid          // 泰坦遗迹 GameObject
│   └── TenacityStack: int32             // 坚韧度叠加层数
│
├── 建筑与工厂:
│   ├── WorkshopsList: Workshop           // 6 个工厂
│   ├── BuildingsInZone: GameObjectBuilding  // 32 个可破坏建筑
│   ├── DefenderPortalList: GameObjectSet   // 防守方传送门
│   └── KeepGameObject[2]: GameObjectSet     // 要塞内 GameObject
│
├── 生物管理:
│   ├── Vehicles[2]: GuidUnorderedSet     // 战场载具
│   ├── CanonList: GuidUnorderedSet       // 塔防炮
│   ├── KeepCreature[2]: GuidUnorderedSet // 要塞内 NPC
│   ├── OutsideCreature[2]: GuidUnorderedSet  // 要塞外 NPC
│   └── UpdateTenacityList: GuidUnorderedSet  // 待更新坚韧度玩家
```

### 4.5 BfWGGameObjectBuilding 结构体（`BattlefieldWG.h:1047-1398`）

可破坏建筑（城墙、门、塔楼）的运行时管理：

```
BfWGGameObjectBuilding
├── m_Team: TeamId                    // 所属阵营
├── m_Build: ObjectGuid              // 关联 GameObject
├── m_Type: uint32                   // WintergraspGameObjectBuildingType
├── m_WorldState: uint32             // 客户端地图图标 WorldState
├── m_State: uint32                  // WintergraspGameObjectState
├── m_damagedText / m_destroyedText  // 警告文本 ID
├── m_GameObjectList[2]: GameObjectSet        // 关联 GameObject（旗帜等）
├── m_CreatureBottomList[2]: GuidUnorderedSet // 建筑底部 NPC
├── m_CreatureTopList[2]: GuidUnorderedSet    // 建筑顶部 NPC
├── m_TowerCannonBottomList: GuidUnorderedSet // 底部塔防炮
├── m_TurretTopList: GuidUnorderedSet        // 顶部炮塔
│
├── 核心方法:
│   ├── Init()          // 初始化：恢复建筑状态、生成关联 NPC/GO
│   ├── Rebuild()       // 重建：战斗开始时恢复所有建筑
│   ├── Damaged()       // 受损：更新 WorldState + 隐藏顶部 NPC
│   ├── Destroyed()     // 摧毁：最后一门被毁→开放泰坦遗迹
│   ├── UpdateCreatureAndGo()  // 根据阵营显示/隐藏 NPC/GO
│   ├── UpdateTurretAttack()   // 更新炮塔阵营和可见性
│   └── Save()         // 保存 WorldState 到数据库
```

**关键行为 — Destroyed()：**
- `BATTLEFIELD_WG_OBJECTTYPE_DOOR_LAST` 被摧毁 → `SetRelicInteractible(true)`，攻击方可点击泰坦遗迹
- `BATTLEFIELD_WG_OBJECTTYPE_TOWER` 被摧毁 → 从战斗时间扣除 10 分钟（3 塔全毁时）

### 4.6 WGWorkshop 结构体（`BattlefieldWG.h:1400-1483`）

工厂控制权管理，决定载具上限和墓地归属：

```
WGWorkshop
├── bf: BattlefieldWG*        // 所属战场
├── workshopId: uint8         // 工厂 ID（WintergraspWorkshopIds）
├── teamControl: TeamId       // 控制方阵营
├── state: uint32             // WorldState 显示值
│
├── GiveControlTo(team, init) // 变更控制权
│   ├── 更新 WorldState 图标
│   ├── 发送区域警告
│   ├── 关联墓地控制权变更
│   ├── 更新载具上限 (UpdateCounterVehicle)
│   └── 触发区域依赖光环更新
│
└── Save()                    // 保存到数据库
```

**工厂与载具上限关系：** 每个被控制的工厂增加 4 个最大载具数。6 个工厂中有 4 个可争夺（NE/NW/SE/SW），2 个属于要塞（KEEP_WEST/KEEP_EAST）始终归防守方。

---

## 5. 数据库表

Battlefield 模块的持久化完全依赖 **`world_state`** 表（存储在 **acore_characters** 数据库中），这是一个通用的键值存储系统，所有战斗状态（防守方、计时器、建筑状态、工厂控制权）都通过 WorldState ID 索引存储。

### 5.1 world_state 表

```sql
CREATE TABLE `world_state` (
  `Id` int unsigned NOT NULL COMMENT 'Internal save ID',
  `Data` longtext,
  PRIMARY KEY (`Id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='WorldState save system';
```

| 列 | 类型 | 含义 |
|----|------|------|
| `Id` | int unsigned | 内部保存 ID（对应 WorldStateSaveIds 枚举，WG 使用 ID=20） |
| `Data` | longtext | 序列化状态数据（空格分隔的键值对字符串） |

**存储方式：** 所有 WG 相关的 WorldState 值被序列化到单行记录中（Id=20），格式为空格分隔的数值序列。每个 WorldState ID 对应一个数值，通过 `sWorldState->setWorldState()` / `sWorldState->getWorldState()` 读写。

### 5.2 Battlefield 使用的 WorldState ID 列表

| WorldState ID | 常量名 | 含义 |
|---------------|--------|------|
| 3801 | `WORLD_STATE_BATTLEFIELD_WG_ACTIVE` | 战斗是否激活 |
| 3802 | `WORLD_STATE_BATTLEFIELD_WG_DEFENDER` | 防守方阵营（0=联盟,1=部落） |
| 3781 | `WORLD_STATE_BATTLEFIELD_WG_CLOCK` | 战斗时钟 |
| 4354 | `WORLD_STATE_BATTLEFIELD_WG_CLOCK_TEXTS` | 时钟文本 |
| 3803 | `WORLD_STATE_BATTLEFIELD_WG_ATTACKER` | 攻击方阵营 |
| 3804 | `WORLD_STATE_BATTLEFIELD_WG_CONTROL` | 控制方显示（1=部落,2=联盟） |
| 3710 | `WORLD_STATE_BATTLEFIELD_WG_SHOW` | 是否显示战斗 UI |
| 4375 | `WORLD_STATE_BATTLEFIELD_WG_ICON_ACTIVE` | 冰晶图标是否激活 |
| 3490 | `WORLD_STATE_BATTLEFIELD_WG_VEHICLE_H` | 部落当前载具数 |
| 3491 | `WORLD_STATE_BATTLEFIELD_WG_MAX_VEHICLE_H` | 部落最大载具数 |
| 3680 | `WORLD_STATE_BATTLEFIELD_WG_VEHICLE_A` | 联盟当前载具数 |
| 3681 | `WORLD_STATE_BATTLEFIELD_WG_MAX_VEHICLE_A` | 联盟最大载具数 |
| 3700 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_NW` | 西北工厂状态 |
| 3701 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_NE` | 东北工厂状态 |
| 3702 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_SW` | 西南工厂状态 |
| 3703 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_SE` | 东南工厂状态 |
| 3698 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_K_W` | 要塞西工厂状态 |
| 3699 | `WORLD_STATE_BATTLEFIELD_WG_WORKSHOP_K_E` | 要塞东工厂状态 |
| 4022 | `WORLD_STATE_BATTLEFIELD_WG_HORDE_KEEP_CAPTURED` | 部落攻占次数 |
| 4023 | `WORLD_STATE_BATTLEFIELD_WG_ALLIANCE_KEEP_CAPTURED` | 联盟攻占次数 |
| 4024 | `WORLD_STATE_BATTLEFIELD_WG_HORDE_KEEP_DEFENDED` | 部落防守次数 |
| 4025 | `WORLD_STATE_BATTLEFIELD_WG_ALLIANCE_KEEP_DEFENDED` | 联盟防守次数 |
| 3749-3773 | (建筑 WorldState) | 32 个建筑的状态（完好/受损/摧毁） |

### 5.3 数据保存时机

```
定时保存（每 60 秒）:
    _scheduler.Schedule(60s, BATTLEFIELD_TIMER_GROUP_SAVE, ...)
        ├── sWorldState->setWorldState(WORLD_STATE_BATTLEFIELD_WG_ACTIVE, Active)
        ├── sWorldState->setWorldState(WORLD_STATE_BATTLEFIELD_WG_DEFENDER, DefenderTeam)
        └── sWorldState->setWorldState(ClockWorldState[0], Timer)

战斗结束时（OnBattleEnd）:
    ├── 每个建筑 building->Save() → setWorldState(worldState, state)
    └── 每个工厂 workshop->Save() → setWorldState(worldState, state)

战斗结束统计:
    ├── sWorldState->setWorldState(KEEP_CAPTURED, getWorldState(KEEP_CAPTURED) + 1)
    └── sWorldState->setWorldState(KEEP_DEFENDED, getWorldState(KEEP_DEFENDED) + 1)
```

### 5.4 数据加载流程（`BattlefieldWG::SetupBattlefield()`）

```
SetupBattlefield()
    │
    ├── 读取 WorldState 判断是否首次运行:
    │   if (!getWorldState(ACTIVE) && !getWorldState(DEFENDER) && !getWorldState(CLOCK))
    │       ├── setWorldState(ACTIVE, false)           // 首次：随机防守方
    │       ├── setWorldState(DEFENDER, urand(0,1))
    │       └── setWorldState(CLOCK, NoWarBattleTime)
    │
    ├── 恢复战斗状态:
    │   ├── Active = getWorldState(ACTIVE)
    │   ├── DefenderTeam = getWorldState(DEFENDER)
    │   └── Timer = getWorldState(CLOCK)
    │
    ├── 崩溃恢复:
    │   if (Active)
    │       ├── Active = false
    │       └── Timer = RestartAfterCrash    // 设置重启延迟
    │
    ├── 初始化墓地 (7 个)
    ├── 初始化工厂 (6 个)
    ├── 生成要塞 NPC (45 个，双阵营)
    ├── 生成外部 NPC (14 个)
    ├── 生成塔防炮 (16 个)
    ├── 生成建筑 (32 个，读取 WorldState 恢复状态)
    ├── 生成传送门 (12 个)
    │
    └── 启动定时器:
        ├── 复活定时器 (RESURRECT_INTERVAL)
        └── 状态保存定时器 (60s)
```

---

## 6. 核心流程详解

### 6.1 战场生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Battlefield 生命周期                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────┐    Timer <= StartGroupingTimer            │
│  │   非战斗期            │ ──────────────────────────>              │
│  │   NoWarBattleTime     │    InvitePlayersInZoneToQueue()           │
│  │   (默认 150 分钟)     │    OnStartGrouping()                      │
│  └──────────────────────┘                                           │
│          │ Timer <= 0                                               │
│          ▼                                                          │
│  ┌──────────────────────┐                                           │
│  │   战斗期              │  BattleTime (默认 30 分钟)                │
│  │   Active = true       │  - 据点争夺                              │
│  │   据点更新            │  - 建筑破坏                              │
│  │   踢 AFK 玩家         │  - 载具战斗                              │
│  └──────────────────────┘                                           │
│          │ Timer <= 0 (超时)  或  泰坦遗迹被点击 (提前结束)          │
│          ▼                                                          │
│  ┌──────────────────────┐                                           │
│  │   战斗结束            │  EndBattle(endByTimer)                   │
│  │   - 发放奖励          │  - 重建建筑                              │
│  │   - 保存状态          │  - 更新 NPC                              │
│  │   - 防守方变更        │  - 清理载具                              │
│  └──────────────────────┘                                           │
│          │                                                          │
│          └──────────────────> 回到非战斗期                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 战斗开始（`Battlefield::StartBattle()` — `Battlefield.cpp:299-345`）

```
StartBattle()
    │
    ├── 清空参战玩家列表和团队列表
    ├── Timer = BattleTime
    ├── Active = true
    │
    ├── 注册战斗期定时任务:
    │   ├── 每 20s 踢 AFK 玩家 (BATTLEFIELD_TIMER_GROUP_WAR)
    │   └── 每 5s 处理邀请超时 + 区域内玩家邀请 (BATTLEFIELD_TIMER_GROUP_WAR)
    │
    ├── InvitePlayersInZoneToWar()       // 邀请区域内玩家参战
    ├── InvitePlayersInQueueToWar()      // 邀请队列中玩家参战
    ├── DoPlaySoundToAll(BF_START)       // 播放战斗开始音效
    │
    ├── OnBattleStart()                  // 子类实现（WG：生成泰坦遗迹、重建建筑等）
    └── SendUpdateWorldStates()          // 更新客户端世界状态
```

### 6.3 战斗结束（`Battlefield::EndBattle()` — `Battlefield.cpp:347-372`）

```
EndBattle(endByTimer)
    │
    ├── Active = false
    ├── _scheduler.CancelGroup(BATTLEFIELD_TIMER_GROUP_WAR)   // 取消战斗期定时器
    ├── StartGrouping = false
    │
    ├── if (!endByTimer)                                    // 攻击方胜利（点击泰坦遗迹）
    │   └── SetDefenderTeam(GetAttackerTeam())              // 攻击方变为新防守方
    │
    ├── 播放胜利音效 (BF_ALLIANCE_WINS / BF_HORDE_WINS)
    ├── OnBattleEnd(endByTimer)                             // 子类实现（WG：奖励、重建等）
    │
    ├── Timer = NoWarBattleTime                             // 重置为非战斗期计时器
    ├── SendInitWorldStatesToAll()
    └── SendUpdateWorldStates()
```

### 6.4 玩家进入区域（`Battlefield::HandlePlayerEnterZone()` — `Battlefield.cpp:77-110`）

```
HandlePlayerEnterZone(player, zone)
    │
    ├── sScriptMgr->OnBattlefieldPlayerEnterZone(this, player)
    │
    ├── if (player->IsInFlight()) → 跳过邀请（飞行中）
    │
    ├── if (IsWarTime()):
    │   ├── if (HasWarVacancy(team)) → InvitePlayerToWar()
    │   └── else → PlayersWillBeKick[team][guid] = now + 10s
    │                InvitePlayerToQueue()   // 战场已满，加入队列
    │
    ├── else (非战斗期):
    │   └── if (Timer <= StartGroupingTimer) → InvitePlayerToQueue()  // 战前 15 分钟开始排队
    │
    ├── Players[team].insert(guid)
    └── OnPlayerEnterZone(player)
```

### 6.5 玩家参战邀请流程

```
[服务端] InvitePlayerToWar(player)
    │
    ├── 检查: 飞行中? 在战场中? 等级不够?
    ├── 检查: IsPlayerInWarOrInvited? (已在战或已邀请)
    ├── sScriptMgr->OnBattlefieldBeforeInvitePlayerToWar()
    ├── 移出 PlayersWillBeKick
    ├── InvitedPlayers[team][guid] = now + TimeForAcceptInvite (20s)
    └── SendBfInvitePlayerToWar(BattleId, ZoneId, time)  ──────> [客户端]
                                                                    │
                                                            玩家选择:
                                                            ├── 接受 → HandleBfEntryInviteResponse
                                                            │         → PlayerAcceptInviteToWar()
                                                            │         ├── AddOrSetPlayerToCorrectBfGroup()
                                                            │         ├── PlayersInWar[team].insert(guid)
                                                            │         ├── InvitedPlayers[team].erase(guid)
                                                            │         ├── if AFK → ToggleAFK()
                                                            │         └── OnPlayerJoinWar(player)
                                                            │
                                                            └── 拒绝 → KickPlayerFromBattlefield()
```

### 6.6 WG 战斗结束奖励（`BattlefieldWG::OnBattleEnd()` — `BattlefieldWG.cpp:348-504`）

```
OnBattleEnd(endByTimer)
    │
    ├── 移除泰坦遗迹
    ├── 隐藏所有塔防炮
    ├── 切换要塞 NPC（隐藏攻击方、显示防守方）
    ├── 切换外部 NPC
    ├── 重置所有墓地（非战斗期归防守方）
    ├── 更新传送门阵营
    ├── 重建所有建筑 + Save()
    ├── 重置工厂 + Save()
    ├── 清除所有载具
    │
    ├── 发放奖励:
    │   ├── 防守方:
    │   │   ├── AreaExploredOrEventHappens(胜利任务)
    │   │   ├── SPELL_ESSENCE_OF_WINTERGRASP (冬拥湖精华)
    │   │   ├── SPELL_VICTORY_REWARD (胜利荣誉)
    │   │   └── 按损毁塔数: SPELL_DAMAGED_TOWER / SPELL_DESTROYED_TOWER
    │   │
    │   └── 攻击方:
    │       ├── SPELL_DEFEAT_REWARD (失败荣誉)
    │       └── 按损毁建筑数: SPELL_DAMAGED_BUILDING / SPELL_INTACT_BUILDING
    │
    ├── 更新相位法术（攻击方胜利时切换）
    ├── 清空参战玩家列表
    │
    └── 更新统计 WorldState (KEEP_CAPTURED / KEEP_DEFENDED)
```

### 6.7 坚韧度系统（`BattlefieldWG::UpdateTenacity()` — `BattlefieldWG.cpp:1156-1231`）

坚韧度是 WG 的阵营平衡机制，人数少的一方获得增益：

```
UpdateTenacity()
    │
    ├── 计算新叠加层数:
    │   newStack = ((horde/alliance) - 1.0) * 4.0  (如果联盟人数少 → 正值，加给联盟)
    │   newStack = (1.0 - (alliance/horde)) * 4.0  (如果部落人数少 → 负值，加给部落)
    │
    ├── if (newStack == TenacityStack): 只更新新加入玩家
    │
    ├── 移除旧增益:
    │   从原受益阵营玩家/载具移除 SPELL_TENACITY / SPELL_TENACITY_VEHICLE
    │
    ├── 施加新增益:
    │   └── 对新受益阵营施加 SPELL_TENACITY (最多 20 层)
    │       + 根据层数施加荣誉增益:
    │       ├── 5-9 层: SPELL_GREAT_HONOR
    │       ├── 10-14 层: SPELL_GREATER_HONOR
    │       └── 15+ 层: SPELL_GREATEST_HONOR
    │
    └── TenacityStack = newStack
```

### 6.8 军衔晋升系统（`BattlefieldWG::PromotePlayer()` — `BattlefieldWG.cpp:799-830`）

```
PromotePlayer(killer)
    │
    ├── if (有 SPELL_RECRUIT 光环):
    │   ├── 击杀叠加 < 5 → CastSpell(SPELL_RECRUIT) 增加层数
    │   └── 击杀叠加 >= 5 → RemoveAura(RECRUIT) + CastSpell(CORPORAL)
    │                        SendWarning(FIRSTRANK)  // "你已被晋升为下士"
    │
    └── if (有 SPELL_CORPORAL 光环):
        ├── 击杀叠加 < 5 → CastSpell(SPELL_CORPORAL) 增加层数
        └── 击杀叠加 >= 5 → RemoveAura(CORPORAL) + CastSpell(LIEUTENANT)
                             SendWarning(SECONDRANK)  // "你已被晋升为中尉"
```

军衔决定玩家可使用的载具类型：
- **新兵**：只能使用投石车
- **下士**：可使用破碎者
- **中尉**：可使用攻城引擎

---

## 7. 网络协议

### 7.1 服务端发送的关键数据包

| 数据包 | Opcode | 用途 |
|--------|--------|------|
| `SMSG_BATTLEFIELD_MGR_QUEUE_INVITE` | 邀请加入队列 | 战前 15 分钟，邀请区域内玩家排队 |
| `SMSG_BATTLEFIELD_MGR_QUEUE_REQUEST_RESPONSE` | 队列响应 | 告知玩家是否成功加入队列 |
| `SMSG_BATTLEFIELD_MGR_ENTRY_INVITE` | 邀请参战 | 战斗开始时，邀请玩家参战 |
| `SMSG_BATTLEFIELD_MGR_ENTERED` | 已参战 | 确认玩家已进入战场 |
| `SMSG_BATTLEFIELD_MGR_EJECTED` | 离开战场 | 踢出战场/离开队列通知 |
| `SMSG_AREA_SPIRIT_HEALER_TIME` | 复活倒计时 | 灵魂医者复活时间 |

### 7.2 客户端发送的数据包

| 数据包 | 处理函数 | 用途 |
|--------|----------|------|
| `HandleBfQueueInviteResponse` | `BattlefieldHandler.cpp:89` | 玩家接受/拒绝排队邀请 |
| `HandleBfEntryInviteResponse` | `BattlefieldHandler.cpp:105` | 玩家接受/拒绝参战邀请 |
| `HandleBfExitRequest` | `BattlefieldHandler.cpp:125` | 玩家主动退出队列 |

---

## 8. 脚本系统集成

### 8.1 脚本钩子

| 钩子 | 触发位置 | 说明 |
|------|----------|------|
| `OnBattlefieldPlayerEnterZone` | `HandlePlayerEnterZone()` | 玩家进入区域，可修改阵营 |
| `OnBattlefieldPlayerLeaveZone` | `HandlePlayerLeaveZone()` | 玩家离开区域，可恢复阵营 |
| `OnBattlefieldPlayerLeaveWar` | `HandlePlayerLeaveZone()` | 玩家离开战斗 |
| `OnBattlefieldPlayerJoinWar` | `PlayerAcceptInviteToWar()` | 玩家加入战斗 |
| `OnBattlefieldBeforeInvitePlayerToWar` | `InvitePlayerToWar()` | 邀请参战前 |

---

## 9. 配置项汇总

### 9.1 世界配置（`WorldConfig.cpp:589-598`）

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `Wintergrasp.Enable` | 1 | 启用冬拥湖（0=禁用,1=启用,2=完全禁用） |
| `Wintergrasp.PlayerMax` | 120 | 每阵营最大玩家数 |
| `Wintergrasp.PlayerMin` | 0 | 每阵营最少开战人数 |
| `Wintergrasp.PlayerMinLvl` | 75 | 最低参与等级 |
| `Wintergrasp.BattleTimer` | 30 | 战斗持续时间（分钟） |
| `Wintergrasp.NoBattleTimer` | 150 | 战斗间隔时间（分钟） |
| `Wintergrasp.CrashRestartTimer` | 10 | 崩溃后重启延迟（分钟） |
| `Wintergrasp.SkipBattleSessionCount` | 3500 | 跳过战斗的会话数阈值 |

### 9.2 关键常量

| 常量 | 值 | 含义 |
|------|---|------|
| `BATTLEFIELD_OBJECTIVE_UPDATE_INTERVAL` | 1000 | 据点更新间隔 (ms) |
| `TimeForAcceptInvite` | 20 | 参战邀请超时 (秒) |
| `StartGroupingTimer` | 15 * MINUTE * IN_MILLISECONDS | 战前集合时间 (15 分钟) |
| `WG_MAX_OBJ` | 32 | WG 最大建筑数 |
| `WG_MAX_TURRET` | 16 | WG 最大炮塔数 |
| `WG_MAX_KEEP_NPC` | 45 | WG 要塞 NPC 数 |
| `WG_MAX_WORKSHOP` | 6 | WG 工厂数 |
| `WG_MAX_ATTACKTOWERS` | 3 | WG 攻击方塔楼数（南方三塔） |
| `WG_MAX_TELEPORTER` | 12 | WG 传送门数 |

---

## 10. BattlefieldMgr 单例（`BattlefieldMgr.h`）

```
BattlefieldMgr (sBattlefieldMgr)
├── _battlefieldSet: vector<Battlefield*>      // 所有战场实例
├── _battlefieldMap: map<zoneId, Battlefield*>  // 区域 ID → 战场映射
├── _updateTimer: uint32                        // 更新间隔计时器
│
├── InitBattlefield()                           // 初始化（创建 BattlefieldWG 实例）
├── HandlePlayerEnterZone(player, zoneId)       // 玩家进入区域
├── HandlePlayerLeaveZone(player, zoneId)       // 玩家离开区域
├── HandlePlayerResurrects(player, zoneId)      // 玩家复活
├── GetBattlefieldToZoneId(zoneId) → Battlefield*
├── GetBattlefieldByBattleId(battleId) → Battlefield*
├── GetZoneScript(zoneId) → ZoneScript*
├── AddZone(zoneId, Battlefield*)               // 注册区域
├── Update(diff)                                // 定时更新（1s 间隔）
├── HandleGossipOption(player, guid, gossipId)
├── CanTalkTo(player, creature, gso)
└── HandleDropFlag(player, spellId)
```

**初始化逻辑（`BattlefieldMgr::InitBattlefield()` — `BattlefieldMgr.cpp:38-60`）：**
- 仅当 `CONFIG_WINTERGRASP_ENABLE != 2` 时初始化
- 创建 `BattlefieldWG` 实例并调用 `SetupBattlefield()`
- 初始化失败则删除实例

---

## 11. WG 静态数据汇总

### 11.1 墓地坐标（7 个）

| ID | 名称 | 坐标 (X, Y, Z) | 墓地 ID | 初始控制 |
|----|------|-----------------|---------|----------|
| 0 | 东北工厂 | 5104.75, 2300.94, 368.58 | 1329 | 中立→防守方 |
| 1 | 西北工厂 | 5099.12, 3466.04, 368.48 | 1330 | 中立→防守方 |
| 2 | 东南工厂 | 4314.65, 2408.52, 392.64 | 1333 | 中立→攻击方 |
| 3 | 西南工厂 | 4331.72, 3235.70, 390.25 | 1334 | 中立→攻击方 |
| 4 | 要塞 | 5537.99, 2897.49, 517.06 | 1285 | 中立→防守方 |
| 5 | 部落 | 5032.45, 3711.38, 372.47 | 1331 | 部落 |
| 6 | 联盟 | 5140.79, 2179.12, 390.95 | 1332 | 联盟 |

### 11.2 工厂与载具上限

| 工厂 | 可争夺 | 初始控制方 | 载具贡献 |
|------|--------|-----------|----------|
| 东北 (NE) | 是 | 防守方 | +4 |
| 西北 (NW) | 是 | 防守方 | +4 |
| 东南 (SE) | 是 | 攻击方 | +4 |
| 西南 (SW) | 是 | 攻击方 | +4 |
| 要塞西 (KEEP_WEST) | 否 | 防守方 | +4 |
| 要塞东 (KEEP_EAST) | 否 | 防守方 | +4 |

**最大载具数** = 联盟控制工厂数 × 4（联盟） / 部落控制工厂数 × 4（部落）

### 11.3 建筑类型分布（32 个）

| 类型 | 数量 | 说明 |
|------|------|------|
| WALL | 23 | 城墙（含带通道的 3 个） |
| KEEP_TOWER | 4 | 要塞塔楼 |
| TOWER | 3 | 南方攻击方塔楼 |
| DOOR | 1 | 要塞大门 |
| DOOR_LAST | 1 | 最后一道门（保护泰坦遗迹） |

---

## 12. 关键设计特征

### 12.1 模板方法模式

`Battlefield` 基类定义了完整的战斗生命周期框架（开始→更新→结束），通过虚函数钩子（`OnBattleStart`、`OnBattleEnd`、`OnPlayerJoinWar` 等）让子类特化行为。`BattlefieldWG` 只需实现这些钩子，无需重写核心流程。

### 12.2 基于WorldState的持久化

所有战斗状态（建筑损坏、工厂控制、防守方阵营、计时器）通过 WorldState 系统持久化到单一的 `world_state` 表中。这种设计的优势是：
- 服务器崩溃后可从数据库恢复战斗状态
- 建筑损坏/工厂占领状态跨服务器重启保持
- 无需额外的专用数据库表

### 12.3 TaskScheduler 定时管理

使用 `TaskScheduler` 管理三类定时任务（复活、战斗、保存），通过分组机制实现战斗期定时器的批量取消：
- `BATTLEFIELD_TIMER_GROUP_RESURRECT`：始终运行
- `BATTLEFIELD_TIMER_GROUP_WAR`：仅战斗期运行，结束时取消
- `BATTLEFIELD_TIMER_GROUP_SAVE`：始终运行

### 12.4 崩溃恢复

`SetupBattlefield()` 中检测到上一次 `Active = true` 时，将战斗状态重置为非战斗期并设置 `RestartAfterCrash` 延迟，避免崩溃后立即开战。

### 12.5 人数平衡机制

- **坚韧度 (Tenacity)**：人数少方获得增益，最多 20 层
- **最大玩家限制**：`HasWarVacancy()` 检查每阵营参战+邀请人数
- **会话数跳过**：超过 `SkipBattleSessionCount` 时自动结束战斗
- **AFK 踢出**：战斗期每 20 秒检查并踢出 AFK 玩家

### 12.6 NPC 阵营切换

要塞内外的 NPC 分为双阵营版本（`entryHorde` / `entryAlliance`），战斗结束时根据新的防守方切换 NPC 可见性（`HideNpc` / `ShowNpc`），通过相位掩码（PhaseMask）控制。

---

## 13. 数据流总览

```
[启动时]
    BattlefieldMgr::InitBattlefield()
        └── new BattlefieldWG → SetupBattlefield()
            ├── 从 world_state 表加载: Active, DefenderTeam, Timer
            ├── 崩溃恢复: if (Active) → Active=false, Timer=RestartAfterCrash
            ├── 初始化墓地/工厂/建筑 (从 WorldState 恢复建筑状态)
            ├── 生成 NPC/GO (双阵营版本，按防守方显示/隐藏)
            ├── 注册区域: RegisterZone(ZoneId)
            └── 启动定时器: 复活(周期) + 保存(60s)

[运行时 - 主循环]
    BattlefieldMgr::Update(diff)
        └── 每 1000ms → Battlefield::Update(diff)
            ├── Timer 倒计时
            ├── Timer <= 0:
            │   ├── 战斗中 → EndBattle(true)
            │   └── 非战斗 → StartBattle()
            ├── 战前集合: Timer <= StartGroupingTimer → InvitePlayersInZoneToQueue()
            ├── _scheduler.Update(diff)
            │   ├── 复活定时器 (始终运行)
            │   ├── 保存定时器 (始终运行, 60s)
            │   ├── 踢AFK定时器 (战斗期, 20s)
            │   ├── 邀请超时定时器 (战斗期, 5s)
            │   └── 坚韧度更新 (战斗期, 10s)
            └── 战斗期: 据点更新 BfCapturePoint::Update(diff)

[玩家进入区域]
    Player::UpdateZone() → BattlefieldMgr::HandlePlayerEnterZone()
        └── Battlefield::HandlePlayerEnterZone()
            ├── 脚本钩子: OnBattlefieldPlayerEnterZone
            ├── 战斗中 → InvitePlayerToWar (有人数限制) / InvitePlayerToQueue
            ├── 非战斗 → 战前 15 分钟排队邀请
            └── Players[team].insert(guid)

[战斗开始]
    Battlefield::StartBattle()
        ├── 清空旧数据 → Timer=BattleTime → Active=true
        ├── 注册战斗期定时器 (AFK/邀请/坚韧度)
        ├── InvitePlayersInZoneToWar() + InvitePlayersInQueueToWar()
        ├── OnBattleStart() [WG: 生成泰坦遗迹, 重建建筑, 生成炮塔, 重置据点]
        └── SendUpdateWorldStates()

[战斗结束]
    Battlefield::EndBattle(endByTimer)
        ├── Active=false → 取消战斗期定时器
        ├── if (!endByTimer) → SetDefenderTeam(AttackerTeam)
        ├── OnBattleEnd() [WG: 移除遗迹, 切换NPC, 重建建筑, 奖励, 保存]
        ├── Timer=NoWarBattleTime
        └── SendInitWorldStatesToAll() + SendUpdateWorldStates()

[定时保存]
    每 60 秒 → sWorldState->setWorldState(ACTIVE/DEFENDER/CLOCK)
    战斗结束 → building->Save() + workshop->Save() + 统计计数
```
