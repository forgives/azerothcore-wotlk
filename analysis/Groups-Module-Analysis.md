# Groups 模块深度分析文档

> 分析范围: `src/server/game/Groups/` (7 个文件)
> 关联处理器: `src/server/game/Handlers/GroupHandler.cpp`, `src/server/game/Handlers/LFGHandler.cpp`
> 关联数据库: `acore_characters` (characters 数据库)

---

## 一、模块概述

Groups 模块是 AzerothCore 中负责**玩家组队系统**的核心模块，管理从 5 人小队 (Party) 到 40 人团队 (Raid) 的所有组队逻辑。它涵盖以下核心职能:

| 职能 | 说明 |
|------|------|
| **组队生命周期** | 创建、邀请、接受、拒绝、移除、解散 |
| **团队管理** | 小队转团队、子分组管理、团队标记 |
| **掉落系统** | 队伍拾取、需求贪婪、队长分配、Roll 点 |
| **副本管理** | 难度设置、副本重置、实例绑定 |
| **战场集成** | 战场/战场组队、队列验证 |
| **LFG 集成** | 随机副本组队、投票踢人、角色分配 |
| **通信广播** | 组队信息同步、就位检查、小地图标记 |

---

## 二、文件结构与职责

```
src/server/game/Groups/
├── Group.h              # Group 类声明、所有枚举/常量、Roll 类、MemberSlot 结构体
├── Group.cpp            # Group 类完整实现 (~2567 行)
├── GroupMgr.h           # GroupMgr 单例声明
├── GroupMgr.cpp         # 组队数据加载、ID 管理、清理逻辑
├── GroupReference.h     # GroupReference 链表引用节点声明
├── GroupReference.cpp   # GroupReference 链接/断开回调实现
└── GroupRefMgr.h        # GroupRefMgr 链表管理器声明 (纯头文件)
```

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `Group.h` | ~365 | 所有类型定义、Group 类接口、Roll 类、MemberSlot 结构体 |
| `Group.cpp` | ~2567 | Group 全部业务逻辑实现 |
| `GroupMgr.h` | ~50 | GroupMgr 单例接口 |
| `GroupMgr.cpp` | ~170 | 服务器启动时加载所有组队数据，ID 生成，数据清理 |
| `GroupReference.h` | ~40 | 在线玩家与 Group 的双向链接节点 |
| `GroupReference.cpp` | ~30 | 链接建立/销毁回调 |
| `GroupRefMgr.h` | ~20 | 在线成员链表管理器 |

---

## 三、核心类架构

### 3.1 类关系总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         GroupMgr (单例)                          │
│  GroupStore: map<uint32, Group*>                                │
│  _groupIds: vector<bool>  (ID 位图)                              │
│  _nextGroupId: LowType                                          │
│                                                                 │
│  + LoadGroups()        启动时加载所有组队                          │
│  + GenerateGroupId()   分配新组队 ID                              │
│  + AddGroup() / RemoveGroup()                                   │
│  + GetGroupByGUID()                                             │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 拥有
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Group                                    │
│                                                                 │
│  m_memberSlots: list<MemberSlot>     [全部成员, 含离线, DB 持久化] │
│  m_memberMgr: GroupRefMgr            [仅在线成员, 内存链表]        │
│  m_invitees: set<Player*>            [待接受邀请的玩家]            │
│  RollId: vector<Roll*>               [活跃的 Roll 点]             │
│                                                                 │
│  m_leaderGuid, m_leaderName                                      │
│  m_groupType, m_lootMethod, m_lootThreshold                      │
│  m_dungeonDifficulty, m_raidDifficulty                           │
│  m_bgGroup, m_bfGroup                                            │
│  m_targetIcons[8], m_lfgGroupFlags                               │
└──────────────────────────────┬──────────────────────────────────┘
                               │ 包含
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  MemberSlot  │  │ GroupRefMgr  │  │    Roll      │
    │  (嵌套结构体) │  │  (链表管理器) │  │  (Roll 点)   │
    │              │  │              │  │              │
    │ guid         │  │ getFirst()   │  │ itemGUID     │
    │ name         │  │    ↓         │  │ playerVote   │
    │ group (子组) │  │ GroupRef...  │  │ totalNeed    │
    │ flags        │  │    ↓         │  │ totalGreed   │
    │ roles (LFG)  │  │ nullptr      │  │ totalPass    │
    └──────────────┘  └──────┬───────┘  └──────────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ GroupReference   │
                    │ (继承 Reference) │
                    │                  │
                    │ iSubGroup        │
                    │ next()           │
                    └────────┬─────────┘
                             │ 双向链接
                             ▼
                    ┌──────────────────┐
                    │     Player       │
                    └──────────────────┘
```

### 3.2 Group 类 — 核心实体

Group 是整个模块的核心，约 2567 行实现代码，承担组队的所有业务逻辑。

**嵌套结构体 MemberSlot** (持久化成员数据):
```cpp
struct MemberSlot {
    ObjectGuid  guid;      // 角色 GUID
    std::string name;      // 角色名
    uint8       group;     // 子分组索引 (0-7, 团队模式)
    uint8       flags;     // GroupMemberFlags 位掩码
    uint8       roles;     // LFG 角色 (坦克/治疗/DPS)
};
```

**关键成员变量**:

| 类型 | 变量名 | 说明 |
|------|--------|------|
| `MemberSlotList` | `m_memberSlots` | 全部成员列表 (含离线), DB 持久化 |
| `GroupRefMgr` | `m_memberMgr` | 在线成员链表, 内存双向引用 |
| `InvitesList` | `m_invitees` | 待接受邀请集合 |
| `ObjectGuid` | `m_leaderGuid` | 队长 GUID |
| `GroupType` | `m_groupType` | 组队类型位掩码 |
| `Difficulty` | `m_dungeonDifficulty` | 地城难度 |
| `Difficulty` | `m_raidDifficulty` | 团队难度 |
| `LootMethod` | `m_lootMethod` | 分配方式 |
| `ItemQualities` | `m_lootThreshold` | Roll 点品质阈值 |
| `Rolls` | `RollId` | 活跃 Roll 点列表 |
| `uint8*` | `m_subGroupsCounts` | 各子分组人数计数 (大小 8) |
| `ObjectGuid[8]` | `m_targetIcons` | 团队标记 (星星/月亮等) |
| `Battleground*` | `m_bgGroup` | 战场指针 |
| `Battlefield*` | `m_bfGroup` | 战场区域指针 |
| `uint8` | `m_lfgGroupFlags` | LFG 特殊标志 |
| `DataMap` | `CustomData` | 脚本自定义数据存储 |

### 3.3 GroupReference 与 GroupRefMgr — 在线成员引用系统

这是一个**双向链表引用系统**，用于高效遍历在线成员:

```
GroupRefMgr (Group::m_memberMgr)
    │
    ├── GroupReference ←→ Player A  (iSubGroup=0)
    ├── GroupReference ←→ Player B  (iSubGroup=1)
    ├── GroupReference ←→ Player C  (iSubGroup=0)
    └── nullptr (链表尾)
```

- **MemberSlot**: 持久化存储，包含所有成员（含离线），对应 `group_member` 表
- **GroupReference**: 运行时存储，仅包含在线玩家，通过 `Player::SetGroup()` 建立链接
- **两者必须同步**: `ChangeMembersGroup()` 同时更新 `MemberSlot.group` 和 `GroupReference.iSubGroup`

**遍历模式**:
- 遍历全部成员（含离线）: 迭代 `m_memberSlots`
- 仅遍历在线成员: 通过 `GetFirstMember()->next()` 链表遍历

### 3.4 GroupMgr — 全局组队管理器

单例模式 (`sGroupMgr`)，负责:

1. **启动加载** (`LoadGroups()`): 从数据库加载所有组队和成员数据
2. **ID 管理** (`GenerateGroupId()`): 使用位图分配唯一组队 ID
3. **注册/注销** (`AddGroup()` / `RemoveGroup()`): 维护内存中的组队映射

### 3.5 Roll — Roll 点系统

继承自 `LootValidatorRef`，表示一次物品 Roll 点:

| 字段 | 类型 | 说明 |
|------|------|------|
| `itemGUID` | `ObjectGuid` | 物品 GUID |
| `itemid` | `uint32` | 物品模板 ID |
| `playerVote` | `map<ObjectGuid, RollVote>` | 每个玩家的投票 |
| `totalPlayersRolling` | `uint8` | 参与 Roll 的总人数 |
| `totalNeed` / `totalGreed` / `totalPass` | `uint8` | 各选项计数 |
| `rollVoteMask` | `uint8` | 允许的 Roll 类型位掩码 |

---

## 四、枚举与常量详解

### 4.1 组队规模常量

| 常量 | 值 | 说明 |
|------|----|------|
| `MAXGROUPSIZE` | 5 | 小队最大人数 |
| `MAXRAIDSIZE` | 40 | 团队最大人数 |
| `MAX_RAID_SUBGROUPS` | 8 | 团队子分组数 (40/5) |
| `TARGETICONCOUNT` | 8 | 团队标记数量 |

### 4.2 GroupType — 组队类型 (位掩码)

| 枚举 | 值 | 说明 |
|------|----|------|
| `GROUPTYPE_NORMAL` | 0x00 | 普通小队 |
| `GROUPTYPE_BG` | 0x01 | 战场组队 |
| `GROUPTYPE_RAID` | 0x02 | 团队 |
| `GROUPTYPE_BGRAID` | 0x03 | 战场团队 (BG\|RAID) |
| `GROUPTYPE_LFG_RESTRICTED` | 0x04 | LFG 限制模式 |
| `GROUPTYPE_LFG` | 0x08 | LFG 组队 |

### 4.3 RollVote — Roll 点投票选项

| 枚举 | 值 | 说明 |
|------|----|------|
| `PASS` | 0 | 放弃 |
| `NEED` | 1 | 需求 |
| `GREED` | 2 | 贪婪 |
| `DISENCHANT` | 3 | 分解 |
| `NOT_EMITED_YET` | 4 | 尚未投票 |
| `NOT_VALID` | 5 | 无效 |

### 4.4 GroupMemberOnlineStatus — 成员在线状态 (位掩码)

| 枚举 | 值 | 说明 |
|------|----|------|
| `MEMBER_STATUS_OFFLINE` | 0x00 | 离线 |
| `MEMBER_STATUS_ONLINE` | 0x01 | 在线 |
| `MEMBER_STATUS_PVP` | 0x02 | PvP 状态 |
| `MEMBER_STATUS_DEAD` | 0x04 | 死亡 |
| `MEMBER_STATUS_GHOST` | 0x08 | 灵魂状态 |
| `MEMBER_STATUS_PVP_FFA` | 0x10 | 自由 PvP |
| `MEMBER_STATUS_UNK3` | 0x20 | 位置相关 (Lua) |
| `MEMBER_STATUS_AFK` | 0x40 | 暂离 |
| `MEMBER_STATUS_DND` | 0x80 | 勿扰 |

### 4.5 GroupMemberFlags — 成员标志 (位掩码)

| 枚举 | 值 | 说明 |
|------|----|------|
| `MEMBER_FLAG_ASSISTANT` | 0x01 | 助理 |
| `MEMBER_FLAG_MAINTANK` | 0x02 | 主坦克 |
| `MEMBER_FLAG_MAINASSIST` | 0x04 | 主助理 |

### 4.6 GroupUpdateFlags — 成员状态更新标志 (位掩码)

用于 `SMSG_PARTY_MEMBER_STATS` 包，标识哪些字段已变更:

| 标志 | 值 | 传输数据 |
|------|----|----------|
| `GROUP_UPDATE_FLAG_STATUS` | 0x00000001 | uint16 状态标志 |
| `GROUP_UPDATE_FLAG_CUR_HP` | 0x00000002 | uint32 当前生命值 |
| `GROUP_UPDATE_FLAG_MAX_HP` | 0x00000004 | uint32 最大生命值 |
| `GROUP_UPDATE_FLAG_POWER_TYPE` | 0x00000008 | uint8 能量类型 |
| `GROUP_UPDATE_FLAG_CUR_POWER` | 0x00000010 | uint16 当前能量 |
| `GROUP_UPDATE_FLAG_MAX_POWER` | 0x00000020 | uint16 最大能量 |
| `GROUP_UPDATE_FLAG_LEVEL` | 0x00000040 | uint16 等级 |
| `GROUP_UPDATE_FLAG_ZONE` | 0x00000080 | uint16 区域 ID |
| `GROUP_UPDATE_FLAG_POSITION` | 0x00000100 | uint16 + uint16 坐标 |
| `GROUP_UPDATE_FLAG_AURAS` | 0x00000200 | uint64 光环掩码 + 逐位数据 |
| `GROUP_UPDATE_FLAG_PET_GUID` | 0x00000400 | uint64 宠物 GUID |
| `GROUP_UPDATE_FLAG_PET_NAME` | 0x00000800 | null-terminated 字符串 |
| `GROUP_UPDATE_FLAG_PET_MODEL_ID` | 0x00001000 | uint16 宠物模型 |
| `GROUP_UPDATE_FLAG_PET_CUR_HP` | 0x00002000 | uint32 |
| `GROUP_UPDATE_FLAG_PET_MAX_HP` | 0x00004000 | uint32 |
| `GROUP_UPDATE_FLAG_PET_POWER_TYPE` | 0x00008000 | uint8 |
| `GROUP_UPDATE_FLAG_PET_CUR_POWER` | 0x00010000 | uint16 |
| `GROUP_UPDATE_FLAG_PET_MAX_POWER` | 0x00020000 | uint16 |
| `GROUP_UPDATE_FLAG_PET_AURAS` | 0x00040000 | uint64 光环掩码 + 逐位数据 |
| `GROUP_UPDATE_FLAG_VEHICLE_SEAT` | 0x00080000 | uint32 载具座位 ID |

### 4.7 lfgGroupFlags — LFG 组队特殊标志

| 枚举 | 值 | 说明 |
|------|----|------|
| `GROUP_LFG_FLAG_APPLY_RANDOM_BUFF` | 0x001 | 应用随机副本 Buff |
| `GROUP_LFG_FLAG_IS_RANDOM_INSTANCE` | 0x002 | 随机副本组队 |
| `GROUP_LFG_FLAG_IS_HEROIC` | 0x004 | 英雄难度 |

### 4.8 DifficultyPreventionChangeType — 难度变更阻止类型

| 枚举 | 值 | 说明 |
|------|----|------|
| `DIFFICULTY_PREVENTION_CHANGE_NONE` | 0 | 无阻止 |
| `DIFFICULTY_PREVENTION_CHANGE_RECENTLY_CHANGED` | 1 | 最近已变更 (60 秒冷却) |
| `DIFFICULTY_PREVENTION_CHANGE_BOSS_KILLED` | 2 | 已击杀 Boss |

---

## 五、数据库表结构

Groups 模块直接使用 3 张 `acore_characters` 数据库表:

### 5.1 `groups` 表 — 组队主表

```sql
CREATE TABLE `groups` (
  `guid`              int unsigned NOT NULL,           -- 组队唯一 ID (PK)
  `leaderGuid`        int unsigned NOT NULL,           -- 队长角色 GUID
  `lootMethod`        tinyint unsigned NOT NULL,       -- 分配方式 (0=自由, 1=轮流, 2=队长, 3=需求贪婪, 4=队长分配)
  `looterGuid`        int unsigned NOT NULL,           -- 轮流拾取者 GUID
  `lootThreshold`     tinyint unsigned NOT NULL,       -- Roll 点品质阈值 (2=绿色, 3=蓝色, 4=紫色)
  `icon1`             bigint unsigned NOT NULL,        -- 团队标记 1 (星星) 目标 GUID
  `icon2`             bigint unsigned NOT NULL,        -- 团队标记 2 (月亮) 目标 GUID
  `icon3`             bigint unsigned NOT NULL,        -- 团队标记 3 (方块) 目标 GUID
  `icon4`             bigint unsigned NOT NULL,        -- 团队标记 4 (三角) 目标 GUID
  `icon5`             bigint unsigned NOT NULL,        -- 团队标记 5 (骷髅) 目标 GUID
  `icon6`             bigint unsigned NOT NULL,        -- 团队标记 6 (叉号) 目标 GUID
  `icon7`             bigint unsigned NOT NULL,        -- 团队标记 7 (心形) 目标 GUID
  `icon8`             bigint unsigned NOT NULL,        -- 团队标记 8 (皇冠) 目标 GUID
  `groupType`         tinyint unsigned NOT NULL,       -- 组队类型 (GroupType 位掩码)
  `difficulty`        tinyint unsigned NOT NULL DEFAULT '0',  -- 地城难度
  `raidDifficulty`    tinyint unsigned NOT NULL DEFAULT '0',  -- 团队难度
  `masterLooterGuid`  int unsigned NOT NULL,           -- 物品分配者 GUID
  PRIMARY KEY (`guid`),
  KEY `leaderGuid` (`leaderGuid`)
) ENGINE=InnoDB;
```

**SQL 操作**:

| 操作 | Prepared Statement | 触发时机 |
|------|--------------------|----------|
| INSERT | `CHAR_INS_GROUP` | `Group::Create()` |
| DELETE | `CHAR_DEL_GROUP` | `Group::Disband()`, `Group::LoadGroupFromDB()` (清理孤立数据) |
| UPDATE leaderGuid | `CHAR_UPD_GROUP_LEADER` | `Group::ChangeLeader()` |
| UPDATE groupType | `CHAR_UPD_GROUP_TYPE` | `Group::ConvertToLFG()`, `Group::ConvertToRaid()` |
| UPDATE difficulty | `CHAR_UPD_GROUP_DIFFICULTY` | `Group::SetDungeonDifficulty()` |
| UPDATE raidDifficulty | `CHAR_UPD_GROUP_RAID_DIFFICULTY` | `Group::SetRaidDifficulty()` |

### 5.2 `group_member` 表 — 组队成员表

```sql
CREATE TABLE `group_member` (
  `guid`         int unsigned NOT NULL,              -- 组队 ID (FK → groups.guid)
  `memberGuid`   int unsigned NOT NULL,              -- 角色 GUID (PK)
  `memberFlags`  tinyint unsigned NOT NULL DEFAULT '0',  -- 成员标志 (GroupMemberFlags)
  `subgroup`     tinyint unsigned NOT NULL DEFAULT '0',  -- 子分组索引 (0-7)
  `roles`        tinyint unsigned NOT NULL DEFAULT '0',  -- LFG 角色
  PRIMARY KEY (`memberGuid`)
) ENGINE=InnoDB;
```

**SQL 操作**:

| 操作 | Prepared Statement | 触发时机 |
|------|--------------------|----------|
| REPLACE INTO | `CHAR_REP_GROUP_MEMBER` | `Group::AddMember()` |
| DELETE (单人) | `CHAR_DEL_GROUP_MEMBER` | `Group::RemoveMember()` |
| DELETE (全组) | `CHAR_DEL_GROUP_MEMBER_ALL` | `Group::Disband()` |
| UPDATE subgroup | `CHAR_UPD_GROUP_MEMBER_SUBGROUP` | `Group::ChangeMembersGroup()` |
| UPDATE memberFlags | `CHAR_UPD_GROUP_MEMBER_FLAG` | `Group::SetGroupMemberFlag()` |

### 5.3 `lfg_data` 表 — LFG 组队数据

```sql
CREATE TABLE `lfg_data` (
  `guid`     int unsigned NOT NULL DEFAULT '0',  -- 组队 ID (FK → groups.guid, PK)
  `dungeon`  int unsigned NOT NULL DEFAULT '0',  -- 副本 ID
  `state`    tinyint unsigned NOT NULL DEFAULT '0',  -- LFG 状态
  PRIMARY KEY (`guid`)
) ENGINE=InnoDB;
```

**SQL 操作**:

| 操作 | Prepared Statement | 触发时机 |
|------|--------------------|----------|
| REPLACE INTO | `CHAR_REP_LFG_DATA` | LFG 状态变更 |
| DELETE | `CHAR_DEL_LFG_DATA` | `Group::Disband()` |

### 5.4 数据库关系图

```
┌──────────────────────┐       ┌──────────────────────┐       ┌──────────────────────┐
│      characters      │       │       groups          │       │     group_member      │
│                      │       │                       │       │                       │
│  guid (PK)           │◄──┐   │  guid (PK)            │◄──────│  guid (FK)            │
│  name                │   │   │  leaderGuid (FK) ─────┼───┘   │  memberGuid (PK,FK)───┼──→ characters.guid
│  ...                 │   │   │  lootMethod           │       │  memberFlags          │
│                      │   ┌───│  looterGuid           │       │  subgroup             │
└──────────────────────┘   │   │  lootThreshold        │       │  roles                │
                           │   │  icon1..icon8         │       └──────────────────────┘
                           │   │  groupType            │
                           │   │  difficulty           │       ┌──────────────────────┐
                           │   │  raidDifficulty       │       │      lfg_data         │
                           │   │  masterLooterGuid     │       │                       │
                           │   └───────────────────────│───────│  guid (PK, FK)        │
                           │                           │       │  dungeon              │
                           └───────────────────────────┘       │  state                │
                                                               └──────────────────────┘
```

### 5.5 启动时数据清理逻辑

`GroupMgr::LoadGroups()` 在加载前执行三步清理:

```sql
-- 1. 删除队长角色已不存在的组队
DELETE FROM `groups` WHERE leaderGuid NOT IN (SELECT guid FROM characters);

-- 2. 删除成员不足 2 人的组队
DELETE FROM `groups` WHERE guid NOT IN (
    SELECT guid FROM group_member GROUP BY guid HAVING COUNT(guid) > 1
);

-- 3. 删除孤立的 LFG 数据 (组队不存在或非 LFG 类型)
DELETE lfg_data FROM lfg_data
LEFT JOIN `groups` ON lfg_data.guid = groups.guid
WHERE groups.guid IS NULL OR groups.groupType <> 12;
```

成员加载前也执行清理:

```sql
-- 删除组队已不存在的成员记录
DELETE FROM group_member WHERE guid NOT IN (SELECT guid FROM `groups`);

-- 删除角色已不存在的成员记录
DELETE FROM group_member WHERE memberGuid NOT IN (SELECT guid FROM characters);
```

---

## 六、核心业务流程

### 6.1 组队创建流程

```
Player A 邀请 Player B
    │
    ▼
HandleGroupInviteOpcode()
    │  验证: 目标在线、不在战斗、未被屏蔽、未满员
    │  调用 Group::AddInvite(player)
    │  发送 SMSG_GROUP_INVITE 给目标
    ▼
Player B 接受邀请
    │
    ▼
HandleGroupAcceptOpcode()
    │  如果 Player A 还没有组队:
    │    → Group::Create(leader)    ← 创建新组队, INSERT INTO groups
    │    → Group::AddMember(A)      ← REPlACE INTO group_member
    │  Group::AddMember(B)          ← REPLACE INTO group_member
    │  Group::SendUpdate()          ← SMSG_GROUP_LIST 发给所有人
    ▼
组队建立完成
```

### 6.2 组队解散流程

```
Group::Disband(hideDestroy)
    │
    ├─ 遍历 m_memberSlots:
    │   ├─ 如果在战场: 从战场移除
    │   ├─ 如果在副本: 处理实例绑定
    │   ├─ 从所有活跃 Roll 中移除
    │   └─ player->SetGroup(nullptr)  ← 断开 GroupReference 链接
    │
    ├─ 删除所有邀请
    │
    ├─ 数据库清理:
    │   ├─ DELETE FROM groups WHERE guid = ?
    │   ├─ DELETE FROM group_member WHERE guid = ?
    │   └─ DELETE FROM lfg_data WHERE guid = ?
    │
    ├─ 发送 SMSG_GROUP_DESTROYED 给所有成员
    │
    ├─ sGroupMgr->RemoveGroup(this)
    │
    └─ delete this  ← 自我销毁
```

### 6.3 成员移除流程

```
Group::RemoveMember(guid, method, kicker, reason)
    │
    ├─ 如果是 LFG 限制组且非自愿离开:
    │   └─ 委托给 sLFGMgr->InitBoot()  ← 投票踢人流程
    │
    ├─ 从 m_memberSlots 中移除
    ├─ 从 Roll 中移除 (RemovePlayerFromRolls)
    ├─ DELETE FROM group_member WHERE memberGuid = ?
    │
    ├─ 如果被移除的是队长:
    │   └─ 自动选举新队长 (第一个在线成员)
    │
    ├─ 如果只剩 1 人且非 LFG/BG:
    │   └─ 自动解散
    │
    ├─ 发送 SMSG_GROUP_UNINVITE 给被移除者
    ├─ 发送 SMSG_GROUP_LIST 更新给剩余成员
    │
    └─ 触发脚本钩子 OnGroupRemoveMember
```

### 6.4 小队转团队流程

```
Group::ConvertToRaid()
    │
    ├─ m_groupType |= GROUPTYPE_RAID
    ├─ _initRaidSubGroupsCounter()  ← 初始化 8 个子分组计数器
    ├─ UPDATE groups SET groupType = ? WHERE guid = ?
    ├─ SendUpdate()  ← SMSG_GROUP_LIST 更新
    └─ 触发脚本钩子
```

### 6.5 Roll 点流程

```
Group::GroupLoot(loot, pLootedObject)
    │
    ├─ 遍历掉落物品中品质 >= m_lootThreshold 的物品:
    │   └─ 创建 Roll 对象, 加入 RollId
    │
    ├─ 对每个 Roll:
    │   ├─ SendLootStartRoll()  ← SMSG_LOOT_START_ROLL
    │   └─ 设置 60 秒倒计时
    │
    ▼
玩家投票 → CountRollVote(playerGUID, itemGUID, choice)
    │
    ├─ 记录投票: roll->playerVote[guid] = choice
    ├─ 更新计数器 (totalNeed/totalGreed/totalPass)
    ├─ SendLootRoll()  ← SMSG_LOOT_ROLL 广播
    │
    ├─ 如果所有玩家已投票:
    │   └─ CountTheRoll()  ← 结算
    │
    ▼
CountTheRoll(roll, allowedMap)
    │
    ├─ 需求 > 贪婪 > 分解 > 放弃 (优先级)
    │
    ├─ 如果有人需求:
    │   └─ 随机选择一个需求者 → SendLootRollWon()
    │
    ├─ 如果无人需求但有人贪婪:
    │   └─ 随机选择一个贪婪者 → SendLootRollWon()
    │
    ├─ 如果全部放弃:
    │   └─ SendLootAllPassed()
    │
    └─ 从 RollId 中移除该 Roll
```

### 6.6 难度变更流程

```
Group::SetDungeonDifficulty(difficulty)
    │
    ├─ 验证: 阻止冷却是否过期
    │   └─ 如果 _difficultyChangePreventionTime > 当前时间:
    │       ├─ DIFFICULTY_PREVENTION_CHANGE_RECENTLY_CHANGED (60 秒内)
    │       └─ DIFFICULTY_PREVENTION_CHANGE_BOSS_KILLED
    │
    ├─ m_dungeonDifficulty = difficulty
    ├─ UPDATE groups SET difficulty = ? WHERE guid = ?
    │
    ├─ 遍历在线成员:
    │   ├─ player->SetDungeonDifficulty(difficulty)
    │   └─ 发送 SMSG_SET_DUNGEON_DIFFICULTY
    │
    └─ SetDifficultyChangePrevention(RECENTLY_CHANGED)  ← 设置 60 秒冷却
```

---

## 七、网络协议 (Opcode)

### 7.1 客户端 → 服务端 (CMSG)

所有 CMSG 处理器均注册为 `STATUS_LOGGEDIN` + `PROCESS_THREADUNSAFE`（在主线程执行）:

| Opcode | 值 | 处理器 | 说明 |
|--------|----|--------|------|
| `CMSG_GROUP_INVITE` | 0x06E | `HandleGroupInviteOpcode` | 邀请玩家 |
| `CMSG_GROUP_CANCEL` | 0x070 | `Handle_NULL` | 取消邀请 (无操作) |
| `CMSG_GROUP_ACCEPT` | 0x072 | `HandleGroupAcceptOpcode` | 接受邀请 |
| `CMSG_GROUP_DECLINE` | 0x073 | `HandleGroupDeclineOpcode` | 拒绝邀请 |
| `CMSG_GROUP_UNINVITE` | 0x075 | `HandleGroupUninviteOpcode` | 按名字踢人 |
| `CMSG_GROUP_UNINVITE_GUID` | 0x076 | `HandleGroupUninviteGuidOpcode` | 按 GUID 踢人 |
| `CMSG_GROUP_SET_LEADER` | 0x078 | `HandleGroupSetLeaderOpcode` | 转让队长 |
| `CMSG_GROUP_DISBAND` | 0x07B | `HandleGroupDisbandOpcode` | 离开组队 |
| `CMSG_GROUP_CHANGE_SUB_GROUP` | 0x27E | `HandleGroupChangeSubGroupOpcode` | 移动成员到子分组 |
| `CMSG_GROUP_SWAP_SUB_GROUP` | 0x280 | `HandleGroupSwapSubGroupOpcode` | 交换两个成员的子分组 |
| `CMSG_GROUP_RAID_CONVERT` | 0x28E | `HandleGroupRaidConvertOpcode` | 小队转团队 |
| `CMSG_GROUP_ASSISTANT_LEADER` | 0x28F | `HandleGroupAssistantLeaderOpcode` | 设置/取消助理 |

### 7.2 服务端 → 客户端 (SMSG)

| Opcode | 值 | 说明 |
|--------|----|------|
| `SMSG_GROUP_INVITE` | 0x06F | 发送邀请给目标 |
| `SMSG_GROUP_CANCEL` | 0x071 | 取消邀请通知 |
| `SMSG_GROUP_DECLINE` | 0x074 | 拒绝邀请通知 |
| `SMSG_GROUP_UNINVITE` | 0x077 | 踢人通知 |
| `SMSG_GROUP_SET_LEADER` | 0x079 | 队长变更通知 |
| `SMSG_GROUP_DESTROYED` | 0x07C | 组队解散通知 |
| `SMSG_GROUP_LIST` | 0x07D | 完整组队信息 (最核心的包) |
| `SMSG_GROUP_JOINED_BATTLEGROUND` | 0x2E8 | 加入战场通知 |
| `SMSG_GROUPACTION_THROTTLED` | 0x411 | 操作频率限制 |

### 7.3 `SMSG_GROUP_LIST` 包结构

这是最核心的组队信息包，由 `Group::SendUpdateToPlayer()` 构建:

```
[SMSG_GROUP_LIST]
├── uint8   groupType           -- 组队类型标志
├── uint8   playerSubGroup      -- 接收者所在子分组
├── uint8   playerFlags         -- 接收者成员标志
├── uint8   playerRoles         -- 接收者 LFG 角色
├── [如果 LFG 组队]:
│   ├── uint8  lfgState         -- LFG 状态 (0=进行中, 2=已完成)
│   └── uint32 lfgDungeon       -- LFG 副本 ID
├── uint64  groupGUID           -- 组队 GUID
├── uint32  counter             -- 序列计数器
├── uint8   memberCount         -- 其他成员数量
├── [对每个其他成员]:
│   ├── string memberName       -- 成员名字
│   ├── uint64 memberGUID       -- 成员 GUID
│   ├── uint8  onlineStatus     -- 在线状态 (0x01=在线)
│   ├── uint8  memberSubGroup   -- 子分组
│   ├── uint8  memberFlags      -- 成员标志
│   └── uint8  memberRoles      -- LFG 角色
├── uint64  leaderGUID          -- 队长 GUID
├── uint8   lootMethod          -- 分配方式
├── uint64  looterGUID          -- 拾取者 GUID
├── uint8   lootThreshold       -- 品质阈值
├── uint8   dungeonDifficulty   -- 地城难度
└── uint8   raidDifficulty      -- 团队难度
```

---

## 八、脚本钩子 (Script Hooks)

Groups 模块在关键生命周期节点调用 `sScriptMgr` 钩子，允许模块/脚本扩展行为:

| 钩子 | 触发位置 | 说明 |
|------|----------|------|
| `OnConstructGroup` | Group 构造函数 | 组队对象创建 |
| `OnDestructGroup` | Group 析构函数 | 组队对象销毁 |
| `OnCreate` | `Group::Create()` | 组队正式创建 |
| `OnGroupInviteMember` | `Group::AddInvite()` | 玩家被邀请 |
| `OnGroupAddMember` | `Group::AddMember()` | 玩家加入组队 |
| `OnGroupRemoveMember` | `Group::RemoveMember()` | 玩家离开/被踢 |
| `OnGroupChangeLeader` | `Group::ChangeLeader()` | 队长变更 |
| `OnGroupDisband` | `Group::Disband()` | 组队解散 |
| `CanGroupJoinBattlegroundQueue` | `Group::CanJoinBattlegroundQueue()` | 验证能否加入战场队列 |
| `OnPlayerGroupRollRewardItem` | `Group::CountTheRoll()` | Roll 点获得物品 |

---

## 九、外部系统交互

```
                        ┌─────────────────┐
                        │    Groups 模块    │
                        └────────┬────────┘
                                 │
        ┌────────────┬───────────┼───────────┬────────────┐
        ▼            ▼           ▼           ▼            ▼
  ┌───────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌──────────┐
  │ LFG 系统  │ │ 战场系统 │ │副本系统│ │脚本系统│ │角色缓存  │
  │ (LFGMgr) │ │(BG/BF)  │ │(Inst)  │ │(Script)│ │(CharCache)│
  └───────────┘ └──────────┘ └────────┘ └────────┘ └──────────┘
```

| 外部系统 | 交互方式 | 说明 |
|----------|----------|------|
| **LFG 系统** (`sLFGMgr`) | 深度集成 | LFG 状态管理、投票踢人、副本分配 |
| **战场系统** (`Battleground`/`Battlefield`) | 指针引用 | BG 组队管理、队列验证 |
| **实例保存管理器** (`sInstanceSaveMgr`) | 数据查询 | 队长变更时复制实例绑定 |
| **竞技场系统** (`sArenaTeamMgr`) | 验证逻辑 | 评级竞技场队列验证 |
| **角色缓存** (`sCharacterCache`) | 名字查询 | 成员名字与 GUID 映射 |
| **物品管理器** (`sObjectMgr`) | 模板查询 | Roll 点时获取物品信息 |
| **地图管理器** (`sMapMgr`) | 副本重置 | `ResetInstances()` 操作 |
| **脚本管理器** (`sScriptMgr`) | 钩子回调 | 所有生命周期事件 |

---

## 十、关键设计模式与注意事项

### 10.1 双存储架构

Groups 模块使用**双存储**设计:

- **MemberSlot** (`m_memberSlots`): 持久化存储，包含所有成员（含离线），对应数据库 `group_member` 表
- **GroupReference** (`m_memberMgr`): 运行时存储，仅包含在线玩家，双向链表

两者必须保持同步。`ChangeMembersGroup()` 等方法同时更新两者。

### 10.2 线程安全

所有组队 CMSG 处理器注册为 `PROCESS_THREADUNSAFE`，意味着在主线程（地图线程）中执行，避免并发修改组队状态。

### 10.3 自我销毁模式

`Group::Disband()` 在最后调用 `delete this`，这意味着:
- 调用 `Disband()` 后不能再访问 Group 对象
- 所有成员必须在 `delete this` 之前断开链接

### 10.4 ID 分配策略

`GroupMgr::GenerateGroupId()` 使用位图 (`vector<bool>`) 从 `_nextGroupId` 开始线性扫描第一个可用 ID。理论上 ID 空间为 `uint32`，溢出时会关闭服务器。

### 10.5 LFG 深度集成

LFG 逻辑深度嵌入 Group 类本身:
- `ConvertToLFG()` 修改组队类型并强制设置需求贪婪分配
- `RemoveMember()` 对 LFG 组队委托给 `sLFGMgr->InitBoot()` (投票踢人)
- `SendUpdateToPlayer()` 为 LFG 组队追加副本信息
- `CanJoinBattlegroundQueue()` 阻止 LFG 组队加入战场

### 10.6 成员查找复杂度

`_getMemberCSlot()` / `_getMemberWSlot()` 对 `m_memberSlots` (std::list) 进行线性查找，复杂度 O(n)。对于 5 人小队影响不大，但 40 人团队中频繁调用可能成为性能瓶颈。

### 10.7 DataMap 自定义数据

Group 类包含一个 `DataMap CustomData` 公共成员，允许脚本/模块在 Group 对象上附加自定义数据，无需修改核心代码。

---

## 十一、Group 类方法分类索引

### 组队操作
`Create`, `LoadGroupFromDB`, `LoadMemberFromDB`, `AddInvite`, `RemoveInvite`, `RemoveAllInvites`, `AddLeaderInvite`, `AddMember`, `RemoveMember`, `ChangeLeader`, `Disband`

### 属性访问
`IsFull`, `isLFGGroup`, `isRaidGroup`, `isBFGroup`, `isBGGroup`, `IsCreated`, `GetGroupType`, `GetLeaderGUID`, `GetLeader`, `GetGUID`, `GetLeaderName`

### 成员查询
`IsMember`, `IsLeader`, `GetMemberGUID`, `IsAssistant`, `GetInvited`, `SameSubGroup`, `HasFreeSlotSubGroup`, `GetMemberSlots`, `GetFirstMember`, `GetMembersCount`, `GetInviteeCount`, `GetMemberGroup`

### 类型转换
`ConvertToLFG`, `ConvertToRaid`, `CheckLevelForRaid`

### 掉落系统
`SetLootMethod`, `SetLooterGuid`, `SetMasterLooterGuid`, `UpdateLooterGuid`, `SetLootThreshold`, `GroupLoot`, `NeedBeforeGreed`, `MasterLoot`, `CountTheRoll`, `CountRollVote`, `EndRoll`, `RemovePlayerFromRolls`, `SendLootStartRoll`, `SendLootRoll`, `SendLootRollWon`, `SendLootAllPassed`, `SendLooter`

### 子分组管理
`ChangeMembersGroup`, `SetTargetIcon`, `SetGroupMemberFlag`, `RemoveUniqueGroupMemberFlag`

### 难度管理
`GetDifficulty`, `GetDungeonDifficulty`, `GetRaidDifficulty`, `SetDungeonDifficulty`, `SetRaidDifficulty`, `ResetInstances`, `SetDifficultyChangePrevention`

### 战场集成
`SetBattlegroundGroup`, `SetBattlefieldGroup`, `CanJoinBattlegroundQueue`

### 通信广播
`DoMinimapPing`, `SendTargetIconList`, `SendUpdate`, `SendUpdateToPlayer`, `UpdatePlayerOutOfRange`, `BroadcastPacket`, `BroadcastReadyCheck`, `OfflineReadyCheck`

### LFG 相关
`SetLfgRoles`, `AddLfgBuffFlag`, `AddLfgRandomInstanceFlag`, `AddLfgHeroicFlag`, `IsLfgWithBuff`, `IsLfgRandomInstance`, `IsLfgHeroic`

---

## 十二、与其他模块的对比

| 特性 | Groups | Guilds |
|------|--------|--------|
| 生命周期 | 临时 (服务器重启后可能清理) | 永久 |
| 最大规模 | 40 人 | 无硬限制 |
| 数据库表 | groups + group_member + lfg_data | guild + guild_member + guild_rank 等 |
| 管理器 | GroupMgr (单例) | GuildMgr (单例) |
| 在线引用 | GroupReference 链表 | 类似机制 |
| 脚本钩子 | 10 个 | 更多 |
| 战场/LFG 集成 | 深度集成 | 无 |

---

## 十三、GM 命令

Groups 模块提供了一组 GM 命令用于管理玩家组队。所有命令均通过 RBAC 权限系统控制，要求 **SEC_GAMEMASTER (等级 2)** 或更高权限。

### 13.1 核心组队命令 (`.group`)

定义在 `src/server/scripts/Commands/cs_group.cpp`，注册为 `.group` 子命令:

| 命令 | 语法 | 权限 ID | 控制台可用 | 说明 |
|------|------|---------|-----------|------|
| `.group list` | `.group list [player]` | 477 | 是 | 列出目标玩家组队的所有成员，显示 Party/Raid 类型、人数、成员标志 (Assistant/MainTank/MainAssist)。支持离线玩家查询。 |
| `.group join` | `.group join <player_in_group> <player_name>` | 476 | 否 | 将第二个玩家加入第一个玩家所在的组队。要求源玩家已在组队中、目标玩家未在组队中、组队未满员。 |
| `.group remove` | `.group remove [player]` | 475 | 否 | 将目标玩家从其当前组队中移除。**支持离线玩家**，通过角色缓存查找组队 GUID。 |
| `.group disband` | `.group disband [player]` | 474 | 否 | 完全解散目标玩家的整个组队。调用 `group->Disband()`。要求目标在线。 |
| `.group revive` | `.group revive [player]` | 868 | 否 | 复活目标玩家组队中的所有成员。移除"救赎之魂"光环 (spell 27827)，恢复生命值 (GM 100%，普通玩家 50%)，生成尸体骨骼并保存到数据库。要求目标在组队中且在线。 |
| `.group leader` | `.group leader [player]` | 473 | 否 | 将组队队长转让给目标玩家。调用 `group->ChangeLeader(guid)` 和 `group->SendUpdate()`。如果目标已是队长则不执行。 |

### 13.2 组队召唤命令 (`.groupsummon`)

定义在 `src/server/scripts/Commands/cs_misc.cpp`，为顶级命令:

| 命令 | 语法 | 权限 ID | 控制台可用 | 说明 |
|------|------|---------|-----------|------|
| `.groupsummon` | `.groupsummon [player]` | 478 | 否 | 将目标玩家的整个组队传送到 GM 当前位置。**副本特殊处理**: 如果 GM 在副本内，GM 必须是组队队长才能召唤。遍历所有成员 (跳过 GM 自身)，停止飞行中的玩家，保存回城点，然后传送到 GM 的地图/坐标。通过 `HasLowerSecurity` 安全检查防止召唤更高权限的玩家。 |

### 13.3 组队传送命令 (`.teleport group`)

定义在 `src/server/scripts/Commands/cs_tele.cpp`，为 `.teleport` 子命令:

| 命令 | 语法 | 权限 ID | 控制台可用 | 说明 |
|------|------|---------|-----------|------|
| `.teleport group` | `.teleport group <name>` | 741 | 否 | 将选中玩家的整个组队传送到已保存的 `game_tele` 位置。要求选中一个玩家。不能传送到战场或竞技场。遍历所有组队成员，检查安全等级后逐一传送，停止飞行中的玩家。 |

### 13.4 LFG 组队信息命令 (`.lfg group`)

定义在 `src/server/scripts/Commands/cs_lfg.cpp`，为 `.lfg` 子命令:

| 命令 | 语法 | 权限 ID | 控制台可用 | 说明 |
|------|------|---------|-----------|------|
| `.lfg group` | `.lfg group [player]` | 432 | 否 | 显示目标玩家组队的 LFG 信息。包括: 是否为 LFG 组队、当前 LFG 状态、LFG 副本 ID。遍历每个组队成员显示各自的 LFG 信息。要求目标在线且在组队中。 |

### 13.5 调试命令 (`.debug send`)

定义在 `src/server/scripts/Commands/cs_debug.cpp`:

| 命令 | 语法 | 权限 ID | 控制台可用 | 说明 |
|------|------|---------|-----------|------|
| `.debug send qpartymsg` | `.debug send qpartymsg <msg>` | (DEBUG) | 否 | 调试命令，向玩家发送任务共享响应消息。调用 `SendPushToPartyResponse`，用于测试与队伍成员的任务共享行为。`<msg>` 为 `QuestShareMessages` 枚举值。 |

### 13.6 已定义但未实现的命令权限

以下 RBAC 权限已在系统中定义并分配给 GM 角色，但**没有对应的命令处理器**，属于预留权限:

| 权限名 | ID | 说明 |
|--------|----|------|
| `RBAC_PERM_COMMAND_GROUP_SET` | 861 | 预留: `.group set` |
| `RBAC_PERM_COMMAND_GROUP_ASSISTANT` | 862 | 预留: `.group set assistant` |
| `RBAC_PERM_COMMAND_GROUP_MAINTANK` | 863 | 预留: `.group set maintank` |
| `RBAC_PERM_COMMAND_GROUP_MAINASSIST` | 864 | 预留: `.group set mainassist` |

这些权限定义在 `src/server/game/Accounts/RBAC.h` (行 633-636)，通过 `data/sql/updates/db_auth/2026_05_01_00.sql` 分配给角色 197 (GM Commands)。如需实现这些命令，需要在 `cs_group.cpp` 中添加对应的处理函数和命令注册。

### 13.7 权限层级说明

```
Role 192 (SEC_ADMINISTRATOR, 等级 3)
    └── Role 193 (SEC_GAMEMASTER, 等级 2)
            ├── Role 197 (Gamemaster Commands)  ← 所有 .group 命令在此
            └── Role 194 (SEC_MODERATOR, 等级 1)
                    └── Role 195 (Player Commands)  ← 不包含组队命令
```

所有组队 GM 命令的 RBAC 权限 (473-478, 432, 741, 868) 均归属于 **Role 197**，因此需要 **SEC_GAMEMASTER (等级 2)** 或更高权限才能使用。管理员 (SEC_MODERATOR, 等级 1) 和普通玩家无法使用这些命令。

### 13.8 命令实现细节

#### 辅助函数

多个组队命令共用 `ChatHandler::GetPlayerGroupAndGUIDByName()` 辅助函数 (定义在 `src/server/game/Chat/Chat.cpp` 行 926):
- 根据玩家名或选中目标解析出 Group 对象和角色 GUID
- 支持 `offline` 参数，允许对离线玩家操作
- 被 `.group remove`、`.group disband`、`.group leader`、`.group revive` 等命令使用

#### 命令脚本注册

`group_commandscript` 类 (行 27) 通过 `AddSC_group_commandscript()` (行 297) 注册，加载路径:
```
cs_group.cpp → AddSC_group_commandscript() → cs_script_loader.cpp → 命令系统
```

---

## 十四、总结

Groups 模块是 AzerothCore 中一个功能完整、设计成熟的组队系统实现。其核心特点:

1. **双存储架构**: MemberSlot (持久化) + GroupReference (运行时) 确保数据一致性和查询效率
2. **深度 LFG 集成**: LFG 逻辑直接嵌入 Group 类，而非独立模块
3. **完整的掉落系统**: 支持自由拾取、轮流拾取、队长分配、需求贪婪、队长分配五种模式
4. **丰富的脚本钩子**: 10 个生命周期钩子支持模块扩展
5. **线程安全设计**: 所有操作在主线程执行，避免竞态条件
6. **自我销毁模式**: `Disband()` 使用 `delete this`，需注意调用后安全性
