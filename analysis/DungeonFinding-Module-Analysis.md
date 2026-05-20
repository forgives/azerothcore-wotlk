# AzerothCore DungeonFinding (LFG) 模块详细分析

## 1. 模块概览

DungeonFinding 模块（又称 LFG — Looking For Group）实现了魔兽世界的地下城查找器（Dungeon Finder）和团队浏览器（Raid Browser）系统。模块负责从玩家加入队列、角色检查、匹配组队、提案确认到进入副本的完整生命周期管理，以及投票踢人和副本完成奖励等辅助功能。

**文件结构：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `LFG.h` | 504 | 枚举定义、Lfg5Guids 固定大小容器、类型别名 |
| `LFG.cpp` | 40 | 工具函数（ConcatenateDungeons、GetRolesString、GetStateString） |
| `LFGPlayerData.h/cpp` | 85/55 | 玩家 LFG 数据封装（状态、锁定副本、角色、注释） |
| `LFGGroupData.h/cpp` | 82/105 | 团队 LFG 数据封装（状态、成员、副本、踢人计数） |
| `LFGQueue.h/cpp` | 125/380 | 匹配队列核心：加入/移除队列、兼容性检查、组队匹配 |
| `LFGMgr.h/cpp` | 661/2500+ | 全局管理器单例：完整生命周期管理、数据库交互、网络包发送 |
| `LFGScripts.h/cpp` | 50/200 | 钩子脚本：玩家/团队事件驱动的 LFG 状态同步 |

**模块位置：** `src/server/game/DungeonFinding/`

---

## 2. 枚举与常量

### 2.1 LFGEnum（`LFG.h:30-35`）

| 常量 | 值 | 含义 |
|------|---|------|
| `LFG_TANKS_NEEDED` | 1 | 每组所需坦克数 |
| `LFG_HEALERS_NEEDED` | 1 | 每组所需治疗数 |
| `LFG_DPS_NEEDED` | 3 | 每组所需输出数 |

### 2.2 LfgRoles（`LFG.h:37-44`）

| 角色 | 值 | 含义 |
|------|---|------|
| `PLAYER_ROLE_NONE` | 0x00 | 无角色 |
| `PLAYER_ROLE_LEADER` | 0x01 | 队长 |
| `PLAYER_ROLE_TANK` | 0x02 | 坦克 |
| `PLAYER_ROLE_HEALER` | 0x04 | 治疗 |
| `PLAYER_ROLE_DAMAGE` | 0x08 | 输出 |

### 2.3 LfgState（`LFG.h:66-76`）

LFG 系统的状态机，描述玩家/团队在 LFG 流程中的当前阶段：

| 状态 | 含义 |
|------|------|
| `LFG_STATE_NONE` | 未使用 LFG/LFR |
| `LFG_STATE_ROLECHECK` | 角色检查进行中 |
| `LFG_STATE_QUEUED` | 排队中 |
| `LFG_STATE_PROPOSAL` | 提案确认中（有人已匹配） |
| `LFG_STATE_BOOT` | 投票踢人进行中 |
| `LFG_STATE_DUNGEON` | 在 LFG 团队中，副本进行中 |
| `LFG_STATE_FINISHED_DUNGEON` | 在 LFG 团队中，副本已完成 |
| `LFG_STATE_RAIDBROWSER` | 使用团队浏览器 |

### 2.4 LfgLockStatusType（`LFG.h:79-93`）

玩家无法进入某副本的锁定原因：

| 锁定类型 | 值 | 含义 |
|---------|---|------|
| `INSUFFICIENT_EXPANSION` | 1 | 资料片不足 |
| `TOO_LOW_LEVEL` | 2 | 等级过低 |
| `TOO_HIGH_LEVEL` | 3 | 等级过高 |
| `TOO_LOW_GEAR_SCORE` | 4 | 装等过低 |
| `TOO_HIGH_GEAR_SCORE` | 5 | 装等过高 |
| `RAID_LOCKED` | 6 | 副本锁定（ID 已绑定） |
| `ATTUNEMENT_TOO_LOW_LEVEL` | 1001 | 钥匙等级过低 |
| `QUEST_NOT_COMPLETED` | 1022 | 前置任务未完成 |
| `MISSING_ITEM` | 1025 | 缺少所需物品 |
| `NOT_IN_SEASON` | 1031 | 不在活动季节 |
| `MISSING_ACHIEVEMENT` | 1034 | 缺少所需成就 |

### 2.5 LfgJoinResult（`LFGMgr.h:99-119`）

加入队列的结果码：

| 结果 | 值 | 含义 |
|------|---|------|
| `LFG_JOIN_OK` | 0 | 加入成功 |
| `LFG_JOIN_FAILED` | 1 | 角色检查失败 |
| `LFG_JOIN_GROUPFULL` | 2 | 队伍已满 |
| `LFG_JOIN_DESERTER` | 12 | 有逃亡者 Debuff |
| `LFG_JOIN_RANDOM_COOLDOWN` | 14 | 随机副本冷却中 |
| `LFG_JOIN_TOO_MUCH_MEMBERS` | 16 | 队伍超过5人 |

### 2.6 LFGMgrEnum（`LFGMgr.h:47-56`）

| 常量 | 值 | 含义 |
|------|---|------|
| `LFG_TIME_ROLECHECK` | 45s | 角色检查超时 |
| `LFG_TIME_BOOT` | 120s | 投票踢人超时 |
| `LFG_TIME_PROPOSAL` | 40s | 提案确认超时 |
| `LFG_QUEUEUPDATE_INTERVAL` | 8s | 队列更新间隔 |
| `LFG_SPELL_DUNGEON_COOLDOWN` | 71328 | 随机副本冷却法术 |
| `LFG_SPELL_DUNGEON_DESERTER` | 71041 | 逃亡者 Debuff 法术 |
| `LFG_SPELL_LUCK_OF_THE_DRAW` | 72221 | 幸运抽奖增益法术 |
| `LFG_GROUP_KICK_VOTES_NEEDED` | 3 | 踢人所需投票数 |

### 2.7 LfgType（`LFGMgr.h:67-75`）

| 类型 | 值 | 含义 |
|------|---|------|
| `LFG_TYPE_DUNGEON` | 1 | 普通地下城 |
| `LFG_TYPE_RAID` | 2 | 团队副本 |
| `LFG_TYPE_HEROIC` | 5 | 英雄地下城 |
| `LFG_TYPE_RANDOM` | 6 | 随机地下城 |

---

## 3. 类层次结构

```
LFGMgr (全局单例 sLFGMgr)
├── LfgPlayerDataContainer PlayersStore     // guid → LfgPlayerData
├── LfgGroupDataContainer GroupsStore       // gguid → LfgGroupData
├── LfgQueueContainer QueuesStore           // groupType → LFGQueue
├── LfgRoleCheckContainer RoleChecksStore   // gguid → LfgRoleCheck
├── LfgProposalContainer ProposalsStore     // proposalId → LfgProposal
├── LfgPlayerBootContainer BootsStore       // gguid → LfgPlayerBoot
├── LfgRewardContainer RewardMapStore       // (dungeonId,maxLevel) → LfgReward*
├── LFGDungeonContainer LfgDungeonStore     // dungeonId → LFGDungeonData
├── LfgCachedDungeonContainer CachedDungeonMapStore  // groupType → LfgDungeonSet
├── LfgDungeonCooldownContainer DungeonCooldownStore // playerGuid → {dungeonId → time}
└── RBStoreMap RaidBrowserStore[2]          // faction → {dungeonId → {playerGuid → RBEntryInfo}}

LFGQueue (每阵营一个实例)
├── LfgQueueDataContainer QueueDataStore    // guid → LfgQueueData
├── LfgCompatibleContainer CompatibleList   // 兼容组合列表
├── LfgWaitTimesContainer waitTimesAvg/Tank/Healer/DpsStore
└── LfgGuidList newToQueueStore             // 待加入队列的 GUID 列表

LfgPlayerData (每个玩家一份)
LfgGroupData (每个团队一份)
```

---

## 4. 核心类详解

### 4.1 Lfg5Guids（`LFG.h:122-495`）

固定大小为 5 的有序 GUID 数组，专为 LFG 5 人组优化。避免了 `std::vector` 的堆分配开销，所有操作（insert/remove/has/size）都通过展开的 if-else 实现：

```
Lfg5Guids
├── guids: array<ObjectGuid, 5>    // 按 GUID 值排序存储
├── roles: LfgRolesMap*             // 可选的角色映射（深拷贝语义）
│
├── insert(g)       // 按 GUID 值有序插入，手动移动元素
├── remove(g)       // 找到后前移后续元素
├── hasGuid(g)      // 5 次比较
├── size()          // 从 index[2] 开始分支判断，避免循环
├── operator<       // 逐位字典序比较，用于 std::set 排序
└── operator==      // 5 个 GUID 全等比较
```

### 4.2 LfgPlayerData（`LFGPlayerData.h:31-80`）

```
LfgPlayerData
├── m_State: LfgState               // 当前 LFG 状态
├── m_OldState: LfgState            // 前一状态（角色检查/提案失败时恢复）
├── m_canOverrideRBState: bool      // 是否可覆盖团队浏览器状态
├── m_LockedDungeons: LfgLockMap    // 锁定副本及原因 {dungeonId → lockStatus}
├── m_TeamId: TeamId                // 阵营（决定进入哪个队列）
├── m_Group: ObjectGuid             // 加入 LFG 时的原始团队 GUID
├── m_randomPlayers: uint8          // 随机队伍中的玩家数
├── m_Roles: uint8                  // 选择的角色（LfgRoles 位掩码）
├── m_Comment: string               // 玩家注释（LFR 中使用）
└── m_SelectedDungeons: LfgDungeonSet  // 选择的副本集合
```

**状态恢复机制：** `RestoreState()` 将 `m_State` 恢复为 `m_OldState`，用于角色检查失败或提案被拒绝时恢复到排队前状态。

### 4.3 LfgGroupData（`LFGGroupData.h:28-78`）

```
LfgGroupData
├── m_State: LfgState               // 团队 LFG 状态
├── m_OldState: LfgState            // 前一状态
├── m_Leader: ObjectGuid            // 队长 GUID
├── m_Players: LfgGuidSet           // 团队成员集合
├── m_RandomQueuedPlayers: LfgGuidSet  // 排队随机副本的成员
├── m_Dungeon: uint32               // 副本 Entry
├── _isLFGGroup: bool               // 是否是 LFG 创建的团队
└── m_KicksLeft: uint8              // 剩余踢人次数
```

### 4.4 LFGQueue（`LFGQueue.h:72-121`）

```
LFGQueue
├── QueueDataStore: map<guid, LfgQueueData>  // 队列中的玩家/团队数据
├── CompatibleList: list<Lfg5Guids>          // 已发现的兼容组合
├── CompatibleTempList: list<Lfg5Guids>      // 迭代时临时存储新兼容组合
├── waitTimesAvg/Tank/Healer/DpsStore       // 按角色分类的平均等待时间
├── newToQueueStore: LfgGuidList             // 新加入队列的 GUID 列表
├── restoredAfterProposal: LfgGuidList       // 提案失败后恢复的 GUID
│
├── AddToQueue(guid, failedProposal)         // 加入队列
├── RemoveFromQueue(guid, partial)           // 移出队列
├── FindGroups() → uint8                     // 核心匹配算法
├── FindNewGroups(newGuid)                   // 为新加入者寻找兼容组
├── CheckCompatibility(...)                  // 兼容性检查
├── FindBestCompatibleInQueue(itrQueue)      // 寻找最佳兼容组合
└── UpdateQueueTimers(diff)                  // 更新等待时间并广播
```

**LfgQueueData 结构：**
```
LfgQueueData
├── joinTime: time_t                 // 加入队列时间
├── lastRefreshTime: time_t          // 上次刷新时间
├── tanks/healers/dps: uint8         // 所需角色数
├── dungeons: LfgDungeonSet          // 选择的副本
├── roles: LfgRolesMap               // 角色映射
└── bestCompatible: Lfg5Guids        // 当前最佳兼容组合
```

**LfgCompatibility 枚举：**
| 兼容性 | 含义 |
|--------|------|
| `LFG_COMPATIBILITY_PENDING` | 待检查 |
| `LFG_INCOMPATIBLES_WRONG_GROUP_SIZE` | 队伍人数错误 |
| `LFG_INCOMPATIBLES_TOO_MUCH_PLAYERS` | 总人数超过5 |
| `LFG_INCOMPATIBLES_MULTIPLE_LFG_GROUPS` | 多个 LFG 团队 |
| `LFG_INCOMPATIBLES_HAS_IGNORES` | 玩家互相屏蔽 |
| `LFG_INCOMPATIBLES_NO_ROLES` | 角色不满足（缺坦克/治疗） |
| `LFG_INCOMPATIBLES_NO_DUNGEONS` | 无共同副本 |
| `LFG_COMPATIBLES_WITH_LESS_PLAYERS` | 兼容但人未满 |
| `LFG_COMPATIBLES_MATCH` | 完全匹配（5人齐） |

### 4.5 LFGMgr（`LFGMgr.h:419-651`）

全局管理器，协调所有 LFG 子系统的运行：

```
LFGMgr (sLFGMgr)
├── 核心存储:
│   ├── PlayersStore: map<ObjectGuid, LfgPlayerData>       // 所有玩家 LFG 数据
│   ├── GroupsStore: map<ObjectGuid, LfgGroupData>         // 所有团队 LFG 数据
│   ├── QueuesStore: map<uint8, LFGQueue>                   // 按阵营分的队列
│   ├── RoleChecksStore: map<ObjectGuid, LfgRoleCheck>      // 进行中的角色检查
│   ├── ProposalsStore: map<uint32, LfgProposal>            // 进行中的提案
│   ├── BootsStore: map<ObjectGuid, LfgPlayerBoot>          // 进行中的踢人投票
│   └── RewardMapStore: multimap<uint32, LfgReward*>        // 随机副本奖励
│
├── 副本数据:
│   ├── LfgDungeonStore: map<uint32, LFGDungeonData>        // 副本数据（来自 DBC + DB）
│   └── CachedDungeonMapStore: map<uint8, LfgDungeonSet>    // 按 groupType 缓存的副本集合
│
├── 副本冷却:
│   └── DungeonCooldownStore: map<playerGuid, {dungeonId → time}>  // 防止连续排到同一副本
│
├── 团队浏览器 (Raid Browser):
│   ├── RaidBrowserStore[2]: RBStoreMap                      // 按阵营分的浏览器数据
│   ├── RBSearchersStore[2]: RBSearchersMap                  // 搜索中的玩家
│   ├── RBCacheStore[2]: RBCacheMap                          // 缓存的数据包
│   ├── RBInternalInfoStorePrev/Curr[2]: RBInternalInfoMapMap // 增量更新的内部数据
│   └── RBUsedDungeonsStore[2]: RBUsedDungeonsSet            // 使用的副本 ID 集合
│
├── 核心 API:
│   ├── JoinLfg(player, roles, dungeons, comment)            // 加入 LFG
│   ├── LeaveLfg(guid)                                       // 离开 LFG
│   ├── UpdateRoleCheck(gguid, guid, roles)                   // 更新角色检查
│   ├── UpdateProposal(proposalId, guid, accept)              // 更新提案
│   ├── InitBoot(gguid, kicker, victim, reason)              // 发起踢人
│   ├── UpdateBoot(guid, accept)                              // 更新踢人投票
│   ├── FinishDungeon(gguid, dungeonId, currMap)              // 完成副本
│   └── TeleportPlayer(player, out, teleportLocation)         // 传送玩家
│
└── Update(diff, task)                                        // 分任务定时更新
```

---

## 5. 数据库表

### 5.1 lfg_data（acore_characters）

运行时 LFG 状态持久化，用于服务器重启后恢复 LFG 团队的副本状态：

```sql
CREATE TABLE `lfg_data` (
  `guid` int unsigned NOT NULL DEFAULT '0' COMMENT 'Global Unique Identifier',
  `dungeon` int unsigned NOT NULL DEFAULT '0',
  `state` tinyint unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`guid`)
) ENGINE=InnoDB COMMENT='LFG Data';
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guid` | int unsigned | 团队 GUID（仅存储团队，不存储玩家） |
| `dungeon` | int unsigned | 当前副本 Entry |
| `state` | tinyint unsigned | LFG 状态（LFG_STATE_DUNGEON=5 或 LFG_STATE_FINISHED_DUNGEON=6） |

**保存逻辑（`_SaveToDB`）：** 使用 `CHAR_REP_LFG_DATA`（REPLACE INTO）在副本状态变更时保存。仅在 `state` 为 `LFG_STATE_DUNGEON` 或 `LFG_STATE_FINISHED_DUNGEON` 时加载。

### 5.2 lfg_dungeon_rewards（acore_world）

随机副本完成奖励的等级分段配置：

```sql
CREATE TABLE `lfg_dungeon_rewards` (
  `dungeonId` int unsigned NOT NULL DEFAULT '0' COMMENT 'Dungeon entry from dbc',
  `maxLevel` tinyint unsigned NOT NULL DEFAULT '0' COMMENT 'Max level at which this reward is rewarded',
  `firstQuestId` int unsigned NOT NULL DEFAULT '0' COMMENT 'Quest id with rewards for first dungeon this day',
  `otherQuestId` int unsigned NOT NULL DEFAULT '0' COMMENT 'Quest id with rewards for Nth dungeon this day',
  PRIMARY KEY (`dungeonId`,`maxLevel`)
) ENGINE=InnoDB;
```

| 列 | 类型 | 含义 |
|----|------|------|
| `dungeonId` | int unsigned | 随机副本 ID（如 259=TBC 随机，261=WotLK 随机） |
| `maxLevel` | tinyint unsigned | 此奖励适用的最大等级 |
| `firstQuestId` | int unsigned | 每日首次完成的奖励任务 ID |
| `otherQuestId` | int unsigned | 每日后续完成的奖励任务 ID（0=无额外奖励） |

**奖励等级分段示例（dungeonId=258，经典随机副本）：**

| maxLevel | firstQuestId | otherQuestId | 适用等级范围 |
|----------|-------------|-------------|-------------|
| 15 | 24881 | 24889 | 1~15 |
| 25 | 24882 | 24890 | 16~25 |
| 34 | 24883 | 24891 | 26~34 |
| 45 | 24884 | 24892 | 35~45 |
| 55 | 24885 | 24893 | 46~55 |
| 60 | 24886 | 24894 | 56~60 |

**奖励查询逻辑：** `GetRandomDungeonReward(dungeon, level)` 遍历 `RewardMapStore` 中 `dungeonId` 匹配的条目（按 maxLevel 升序排列），找到第一个 `maxLevel >= playerLevel` 的条目。`firstQuestId` 用于每日首次，`otherQuestId` 用于后续。

### 5.3 lfg_dungeon_template（acore_world）

副本传送坐标覆盖，补充 DBC 中缺少的入口位置：

```sql
CREATE TABLE `lfg_dungeon_template` (
  `dungeonId` int unsigned NOT NULL DEFAULT '0' COMMENT 'Unique id from LFGDungeons.dbc',
  `name` varchar(255),
  `position_x` float NOT NULL DEFAULT '0',
  `position_y` float NOT NULL DEFAULT '0',
  `position_z` float NOT NULL DEFAULT '0',
  `orientation` float NOT NULL DEFAULT '0',
  `VerifiedBuild` int DEFAULT NULL,
  PRIMARY KEY (`dungeonId`)
) ENGINE=InnoDB;
```

| 列 | 类型 | 含义 |
|----|------|------|
| `dungeonId` | int unsigned | 副本 ID（对应 LFGDungeons.dbc） |
| `name` | varchar(255) | 副本名称（仅用于 DB 查看） |
| `position_x/y/z` | float | 传送目标坐标 |
| `orientation` | float | 传送目标朝向 |
| `VerifiedBuild` | int | 验证构建号 |

**坐标加载优先级：**
1. `lfg_dungeon_template` 表中的坐标（DB 覆盖）
2. `areatrigger_teleport` 表中的地图入口触发器
3. 若两者都无 → 记录错误日志

### 5.4 表间关系图

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           数据库关系                                          │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  acore_characters                        acore_world                          │
│  ───────────────                         ───────────                          │
│                                                                               │
│  ┌──────────────┐                        ┌──────────────────────┐             │
│  │  lfg_data    │                        │ lfg_dungeon_template │             │
│  │  (guid,      │                        │  (dungeonId,         │             │
│  │   dungeon,   │                        │   name, x/y/z/o)    │             │
│  │   state)     │                        └──────────┬───────────┘             │
│  └──────┬───────┘                                   │                         │
│         │                                           │ dungeonId               │
│         │ guid=groupGuid                            │                         │
│         │                                           ▼                         │
│         │                          ┌──────────────────────────────┐            │
│         │                          │  lfg_dungeon_rewards         │            │
│         │                          │  (dungeonId, maxLevel,       │            │
│         │                          │   firstQuestId, otherQuestId)│            │
│         │                          └──────────────────────────────┘            │
│         │                                                                      │
│         │ dungeon → LFGDungeons.dbc (副本定义)                                  │
│         │ state → LfgState 枚举                                                 │
│         │ firstQuestId/otherQuestId → quest_template (任务模板)                  │
│                                                                               │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │  LFGDungeons.dbc (只读)                                      │               │
│  │  ID, Name, MapID, TypeID, ExpansionLevel, GroupID,           │               │
│  │  MinLevel, MaxLevel, Difficulty, Flags                       │               │
│  └──────────────────────────────────────────────────────────────┘               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 核心流程详解

### 6.1 LFG 生命周期状态机

```
                    ┌─────────────────┐
                    │  LFG_STATE_NONE │◄──────────────────────────────────┐
                    └────────┬────────┘                                   │
                             │ JoinLfg()                                  │
                             ▼                                            │
                    ┌─────────────────┐                                   │
              ┌────►│LFG_STATE_       │                                    │
              │     │  ROLECHECK      │                                    │
              │     └────────┬────────┘                                   │
              │              │ 角色检查完成                                  │
              │              ▼                                            │
              │     ┌─────────────────┐    提案失败/拒绝                    │
              │     │ LFG_STATE_      │──────────────────┐                  │
              │     │  QUEUED         │                  │                  │
              │     └────────┬────────┘                  │                  │
              │              │ 匹配成功                     │ RestoreState()  │
              │              ▼                            │                  │
              │     ┌─────────────────┐                  │                  │
              │     │ LFG_STATE_      │── 超时/拒绝 ─────┤                  │
              │     │  PROPOSAL       │                  │                  │
              │     └────────┬────────┘                  │                  │
              │              │ 全部接受                     │                  │
              │              ▼                            │                  │
              │     ┌─────────────────┐                   │                  │
              │     │ LFG_STATE_      │◄── 踢人超时 ──┐   │                  │
              │     │  DUNGEON        │               │   │                  │
              │     └────────┬────────┘               │   │                  │
              │              │ 副本完成                  │   │                  │
              │              ▼                          │   │                  │
              │     ┌─────────────────┐      ┌─────────┴──┐│                  │
              │     │ LFG_STATE_      │      │LFG_STATE_  ││                  │
              │     │  FINISHED_      │      │  BOOT       ││                  │
              │     │  DUNGEON        │      └────────────┘│                  │
              │     └────────┬────────┘                    │                  │
              │              │ 离开/解散                     │                  │
              │              └─────────────────────────────┴──────────────────┘
              │
              │ 角色检查失败/超时/中止
              └──── RestoreState() → LFG_STATE_NONE
```

### 6.2 加入 LFG（`LFGMgr::JoinLfg()` — `LFGMgr.cpp:598`）

```
JoinLfg(player, roles, dungeons, comment)
    │
    ├── 前置检查:
    │   ├── 已在 RaidBrowser → 重新发送加入包
    │   ├── 已在角色检查 → 中止旧检查
    │   ├── 已在队列 → 先移出队列
    │   ├── 已在提案 → 返回 INTERNAL_ERROR
    │   └── 脚本拦截: OnPlayerCanJoinLfg()
    │
    ├── 副本验证:
    │   ├── 类型检查: 不能混合副本/团队/随机
    │   ├── 随机副本只允许选1个
    │   └── 无效副本类型 → DUNGEON_INVALID
    │
    ├── 玩家/团队限制检查:
    │   ├── 无 RBAC 权限 → NOT_MEET_REQS
    │   ├── 在战场 → USING_BG_SYSTEM
    │   ├── 有逃亡者 Debuff → DESERTER
    │   ├── 随机副本冷却中 → RANDOM_COOLDOWN
    │   ├── 团队超过5人 → TOO_MUCH_MEMBERS
    │   └── 成员离线/未确认 → DISCONNECTED
    │
    ├── 生成锁定副本映射 (GetCompatibleDungeons)
    │   └── 对每个副本检查每个成员的锁定状态
    │
    ├── if (有队伍):
    │   ├── 启动角色检查 (UpdateRoleCheck)
    │   └── 设置状态为 ROLECHECK
    │
    └── if (单人):
        ├── 直接设置角色和副本
        ├── 加入队列 (GetQueue().AddToQueue)
        └── 设置状态为 QUEUED
```

### 6.3 角色检查（`LFGMgr::UpdateRoleCheck()` — `LFGMgr.cpp:1502`）

```
UpdateRoleCheck(gguid, guid, roles)
    │
    ├── 查找 RoleChecksStore[gguid]
    │
    ├── if (guid == Empty): 中止角色检查
    │   └── state = ABORTED, 所有人 RestoreState
    │
    ├── 更新指定玩家的角色选择
    │
    ├── 检查角色是否满足需求 (CheckGroupRoles):
    │   ├── 至少1坦克 + 1治疗 + 输出
    │   └── 不满足 → state = WRONG_ROLES
    │
    ├── 检查是否所有人都选择了角色:
    │   └── 有人选 NONE → state = NO_ROLE
    │
    ├── if (角色检查完成):
    │   ├── state = FINISHED
    │   ├── 设置状态为 QUEUED
    │   ├── 加入队列
    │   └── 广播更新
    │
    └── 设置取消时间: cancelTime = now + LFG_TIME_ROLECHECK (45s)
```

### 6.4 队列匹配（`LFGQueue::FindGroups()` — `LFGQueue.cpp`）

```
FindGroups() → uint8
    │
    ├── 处理 newToQueueStore 中的新加入者:
    │   └── 对每个新 GUID:
    │       ├── FindNewGroups(newGuid)      // 寻找兼容组合
    │       └── AddToCompatibles(...)       // 添加到兼容列表
    │
    ├── 检查兼容列表中是否有完整5人组:
    │   └── 对每个 Lfg5Guids:
    │       if (bestCompatible.size() == 5):
    │           ├── 创建 LfgProposal
    │           ├── 设置所有玩家的角色和团队
    │           ├── 从队列中移除相关 GUID
    │           └── return 1 (表示找到了新组)
    │
    └── return 0 (未找到完整组)
```

**兼容性检查（`CheckCompatibility()`）：**
```
CheckCompatibility(checkWith, newGuid, foundMask, foundCount, currentCompatibles)
    │
    ├── 检查1: 人数不超过5
    ├── 检查2: 没有多个 LFG 团队
    ├── 检查3: 没有互相屏蔽 (HasIgnore)
    ├── 检查4: 角色满足需求 (CheckGroupRoles)
    ├── 检查5: 有共同副本 (dungeons 交集不为空)
    │
    └── 返回 LfgCompatibility 值
```

### 6.5 提案确认（`LFGMgr::UpdateProposal()` — `LFGMgr.cpp:1920`）

```
UpdateProposal(proposalId, guid, accept)
    │
    ├── 查找 ProposalsStore[proposalId]
    ├── 更新该玩家的接受状态:
    │   ├── LFG_ANSWER_AGREE (1)  = 接受
    │   ├── LFG_ANSWER_DENY (0)   = 拒绝
    │   └── LFG_ANSWER_PENDING (-1) = 未响应
    │
    ├── if (有人拒绝):
    │   ├── state = LFG_PROPOSAL_FAILED
    │   ├── 接受者恢复到队列 (AddToQueue(failedProposal=true))
    │   ├── 拒绝者离开 LFG (LeaveLfg)
    │   └── 移除提案
    │
    ├── if (所有人接受):
    │   ├── state = LFG_PROPOSAL_SUCCESS
    │   └── MakeNewGroup(proposal)   // 创建新团队
    │
    └── if (超时未全员响应):
        └── RemoveProposal → LFG_UPDATETYPE_PROPOSAL_FAILED
```

### 6.6 创建团队（`LFGMgr::MakeNewGroup()` — `LFGMgr.cpp:1690`）

```
MakeNewGroup(proposal)
    │
    ├── 确定目标团队:
    │   ├── if (proposal.isNew) → 创建新 Group
    │   └── if (!proposal.isNew) → 使用已有 Group
    │
    ├── 将所有玩家加入团队:
    │   ├── 非原团队成员 → group->AddMember(player)
    │   └── 原团队成员 → player->SetBattlegroundOrBattlefieldRaid(group)
    │
    ├── 传送玩家到副本:
    │   └── TeleportPlayer(player, false)
    │
    ├── 设置团队 LFG 状态:
    │   ├── SetDungeon(gguid, proposal.dungeonId)
    │   ├── SetState(gguid, LFG_STATE_DUNGEON)
    │   ├── 所有玩家 SetState(guid, LFG_STATE_DUNGEON)
    │   └── 施加 LFG_SPELL_LUCK_OF_THE_DRAW 增益
    │
    ├── 设置队长
    └── 保存到数据库 (_SaveToDB)
```

### 6.7 完成副本（`LFGMgr::FinishDungeon()` — `LFGMgr.cpp:2317`）

```
FinishDungeon(gguid, dungeonId, currMap)
    │
    ├── 获取副本数据中的 rDungeonId (随机副本 ID)
    │
    ├── 设置状态:
    │   ├── SetState(gguid, LFG_STATE_FINISHED_DUNGEON)
    │   └── 所有玩家 SetState(guid, LFG_STATE_FINISHED_DUNGEON)
    │
    ├── 添加副本冷却 (AddDungeonCooldown)
    │
    ├── 发放随机副本奖励:
    │   └── 对每个排队随机副本的玩家:
    │       ├── GetRandomDungeonReward(rDungeonId, level)
    │       ├── if (firstQuestId): 玩家可完成每日首杀任务
    │       └── if (otherQuestId): 玩家可完成每日额外任务
    │
    ├── 更新等待时间统计
    └── 保存到数据库 (_SaveToDB)
```

### 6.8 投票踢人（`LFGMgr::InitBoot()` — `LFGMgr.cpp:2132`）

```
InitBoot(gguid, kicker, victim, reason)
    │
    ├── 前置检查:
    │   ├── 团队必须是 LFG 团队 (IsLfgGroup)
    │   ├── 副本进行中 (state == DUNGEON)
    │   ├── 不在踢人进行中
    │   ├── 还有踢人次数 (KicksLeft > 0)
    │   └── 防踢保护时间内 (LFG.KickPreventionTimer)
    │
    ├── 创建 LfgPlayerBoot:
    │   ├── cancelTime = now + LFG_TIME_BOOT (120s)
    │   ├── inProgress = true
    │   ├── victim = 被踢者
    │   ├── reason = 踢人原因
    │   └── votes: 发起者自动投赞成票
    │
    ├── 设置所有玩家状态为 BOOT
    └── 广播踢人提案给所有成员
```

---

## 7. 定时更新机制

`LFGMgr::Update(diff, task)` 将更新分为 3 个任务，由 `World::Update()` 轮流调用：

| task | 执行内容 | 频率 |
|------|---------|------|
| 0 | 清理过期的角色检查/提案/踢人/副本冷却 | 每次轮到 |
| 1 | 队列匹配（`FindGroups`）或等待时间更新 | 每次轮到 |
| 2 | 新提案通知 + 团队浏览器更新 | 每次轮到 |

---

## 8. 团队浏览器（Raid Browser）

团队浏览器是 LFR（Looking For Raid）的浏览界面，与 Dungeon Finder 的自动匹配不同，RB 让玩家浏览其他玩家/团队的列表并手动邀请。

**数据结构：**
- `RaidBrowserStore[2]`：按阵营存储每个副本下的玩家条目（角色+注释）
- `RBInternalInfoStoreCurr/Prev[2]`：增量更新所需的内部信息快照
- `RBCacheStore[2]`：缓存的全量数据包（首次请求时发送）

**更新机制（`UpdateRaidBrowser`）：**
1. 定时（10s）轮询一个副本 ID
2. 对该副本构建当前所有条目的内部信息
3. 与上一次快照比较，生成差量更新（新增/删除/修改）
4. 将差量包发送给所有正在浏览该副本的玩家
5. 全量请求直接发送缓存包

---

## 9. 配置项汇总

### 9.1 世界配置

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `DungeonFinder.OptionsMask` | 5 | LFG 选项位掩码（1=DF, 2=RB, 4=Seasonal） |
| `DungeonFinder.CastDeserter` | true | 离开副本时施加逃亡者 Debuff |
| `DungeonFinder.AllowCompleted` | true | 允许已完成副本的团队继续排队 |
| `DungeonFinder.DungeonSelectionCooldown` | 0 | 副本选择冷却时间（分钟，0=禁用） |
| `LFG.Location.All` | false | LFG 传送是否忽略副本位置限制 |
| `LFG.MaxKickCount` | 2 | 每个副本内最大踢人次数（<=3） |
| `LFG.KickPreventionTimer` | 900 | 副本开始后防踢保护时间（秒，<=15min） |
| `Debug.LFG` | false | 调试模式（允许单人排5人副本） |

### 9.2 OptionsMask 位定义

| 位 | 值 | 含义 |
|----|---|------|
| `LFG_OPTION_ENABLE_DUNGEON_FINDER` | 0x01 | 启用地下城查找器 |
| `LFG_OPTION_ENABLE_RAID_BROWSER` | 0x02 | 启用团队浏览器 |
| `LFG_OPTION_ENABLE_SEASONAL_BOSSES` | 0x04 | 启用季节性 Boss |

默认值 5 = 0x01 | 0x04 = Dungeon Finder + Seasonal Bosses（不含 Raid Browser）。

### 9.3 关键法术

| 法术 ID | 名称 | 含义 |
|---------|------|------|
| 71328 | Dungeon Cooldown | 随机副本完成后冷却 |
| 71041 | Dungeon Deserter | 逃亡者 Debuff（30 分钟无法排本） |
| 72221 | Luck of the Draw | 幸运抽奖增益（随机副本额外属性） |

---

## 10. LFGScripts 钩子

LFGScripts 负责将玩家/团队事件（登录/登出/加入团队/离开团队/换地图等）同步到 LFGMgr：

| 钩子 | 触发场景 | 处理逻辑 |
|------|---------|---------|
| `OnPlayerLogin` | 玩家登录 | 初始化锁定副本、设置阵营 |
| `OnPlayerLogout` | 玩家登出 | 从 LFG 系统移除 |
| `OnPlayerJoinGroup` | 加入团队 | 设置团队 GUID、更新成员列表 |
| `OnPlayerLeaveGroup` | 离开团队 | 清除团队数据、移出队列 |
| `OnGroupDisband` | 团队解散 | 清除所有成员的 LFG 状态 |
| `OnLeaderChange` | 队长变更 | 更新 LFG 队长 |
| `OnPlayerChangeArea` | 换区域 | 更新团队浏览器区域信息 |

---

## 11. 关键设计特征

### 11.1 三阶段匹配

LFG 匹配分为角色检查→排队→提案三个阶段，每阶段都有超时机制：
- 角色检查：45 秒超时
- 提案确认：40 秒超时
- 投票踢人：120 秒超时

### 11.2 增量兼容性更新

`LFGQueue` 使用 `CompatibleList` + `CompatibleTempList` 的双缓冲设计。迭代 `CompatibleList` 时新发现的兼容组合存入 `CompatibleTempList`，避免迭代中修改容器。迭代完成后合并。

### 11.3 副本冷却防重复

`DungeonCooldownStore` 记录玩家最近完成的副本 ID 和时间，在下次匹配时过滤掉冷却中的副本。若所有副本都在冷却中则跳过过滤，避免玩家无法排队。

### 11.4 分任务更新

`LFGMgr::Update()` 使用 task 参数将不同类型的工作分散到不同调用周期，避免单次更新耗时过长影响服务器性能。匹配计算（task=1）和提案通知（task=2）分离执行。

### 11.5 Lfg5Guids 性能优化

使用固定大小数组（5个元素）+ 手动展开的 if-else 替代 `std::vector`，消除堆分配。排序插入保证 `operator<` 的字典序比较一致性，使 `Lfg5Guids` 可作为 `std::set` 的键。

### 11.6 团队浏览器增量更新

RB 系统存储当前和前一次快照，通过 `PlayerSameAs()` 比较字段差异，仅发送变更部分。全量数据包缓存后仅在首次请求时发送。

---

## 12. 数据流总览

```
[服务器启动]
    LFGMgr 初始化: m_options = DungeonFinder.OptionsMask
    ├── LoadLFGDungeons()
    │   ├── LFGDungeons.dbc → LfgDungeonStore (副本定义)
    │   ├── lfg_dungeon_template → 传送坐标覆盖
    │   └── areatrigger_teleport → 兜底传送坐标
    └── LoadRewards()
        └── lfg_dungeon_rewards → RewardMapStore

[玩家登录]
    LFGScripts::OnPlayerLogin
        └── InitializeLockedDungeons(player)
            └── 检查等级/装等/资料片/绑定/前置 → LfgLockMap

[玩家加入 LFG]
    JoinLfg(player, roles, dungeons, comment)
        ├── 单人: 直接加入队列 → LFG_STATE_QUEUED
        └── 组队: 启动角色检查 → LFG_STATE_ROLECHECK
            └── UpdateRoleCheck() → 全员确认角色 → LFG_STATE_QUEUED

[队列匹配]
    LFGMgr::Update(task=1)
        └── LFGQueue::FindGroups()
            ├── 处理新加入者 → FindNewGroups()
            ├── 兼容性检查 → CheckCompatibility()
            └── 找到5人组 → 创建 LfgProposal → ProposalsStore

[提案通知]
    LFGMgr::Update(task=2)
        └── 检测新提案 → 发送 SendLfgUpdateProposal 给所有玩家
            └── 玩家选择接受/拒绝 → UpdateProposal()

[提案成功]
    UpdateProposal() → LFG_PROPOSAL_SUCCESS
        └── MakeNewGroup()
            ├── 创建/重组团队
            ├── 传送玩家到副本入口
            ├── 施加 Luck of the Draw 增益
            └── 保存 lfg_data 表

[副本完成]
    FinishDungeon(gguid, dungeonId, currMap)
        ├── 设置状态为 FINISHED_DUNGEON
        ├── 添加副本冷却
        ├── 发放随机副本奖励（首杀/日常任务）
        └── 保存 lfg_data 表

[投票踢人]
    InitBoot() → 创建 LfgPlayerBoot
        └── UpdateBoot() → 达到3票 → 踢出玩家
```
