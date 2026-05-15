# AzerothCore Guilds 模块详细分析

## 1. 模块概览

Guilds 模块实现了魔兽世界中公会系统的完整功能，包括公会创建/解散、成员管理、等级权限、公会银行、事件日志等核心功能。模块采用面向对象的层次设计，将复杂的公会逻辑封装在 `Guild` 类及其嵌套类中。

**文件结构：**

| 文件 | 行数 | 职责 |
|------|------|------|
| `Guild.h` | 870 | 公会类声明，含所有嵌套类（Member、RankInfo、BankTab、LogEntry 等）和枚举定义 |
| `Guild.cpp` | 2963 | 完整实现，含 DB 操作、网络包发送、物品移动逻辑 |
| `GuildMgr.h` | 54 | 全局公会管理器单例声明 |
| `GuildMgr.cpp` | 415 | 公会集合管理、从数据库加载全部公会数据 |

**模块位置：** `src/server/game/Guilds/`

---

## 2. 枚举与常量

### 2.1 GuildMisc（`Guild.h:40-54`）

| 常量 | 值 | 含义 |
|------|---|------|
| `GUILD_BANK_MAX_TABS` | 6 | 公会银行最大标签页数 |
| `GUILD_BANK_MAX_SLOTS` | 98 | 每个标签页最大物品槽位数 |
| `GUILD_BANK_MONEY_LOGS_TAB` | 100 | 金币日志在 DB 中使用的 TabId |
| `GUILD_RANKS_MIN_COUNT` | 5 | 公会最少等级数 |
| `GUILD_RANKS_MAX_COUNT` | 10 | 公会最多等级数 |
| `GUILD_RANK_NONE` | 0xFF | 无效等级 ID 标记 |
| `GUILD_WITHDRAW_MONEY_UNLIMITED` | 0xFFFFFFFF | 无限金币提取 |
| `GUILD_WITHDRAW_SLOT_UNLIMITED` | 0xFFFFFFFF | 无限物品提取 |
| `GUILD_BANK_MONEY_LIMIT` | 0x7FFFFFFFFFFFF | 银行金币上限 |

### 2.2 GuildDefaultRanks（`Guild.h:62-72`）

```
0 = GR_GUILDMASTER   // 公会长 - 全部权限
1 = GR_OFFICER       // 官员 - 全部权限
2 = GR_VETERAN       // 老兵 - 公会聊天
3 = GR_MEMBER        // 会员 - 公会聊天
4 = GR_INITIATE      // 新人 - 公会聊天
// 晋升 = rank--  降级 = rank++
```

### 2.3 GuildRankRights（`Guild.h:74-95`）

权限位掩码，所有权限都包含 `GR_RIGHT_EMPTY (0x40)` 基位：

| 权限 | 值 | 含义 |
|------|---|------|
| `GR_RIGHT_GCHATLISTEN` | 0x41 | 公会频道收听 |
| `GR_RIGHT_GCHATSPEAK` | 0x42 | 公会频道发言 |
| `GR_RIGHT_OFFCHATLISTEN` | 0x44 | 官员频道收听 |
| `GR_RIGHT_OFFCHATSPEAK` | 0x48 | 官员频道发言 |
| `GR_RIGHT_INVITE` | 0x50 | 邀请入会 |
| `GR_RIGHT_REMOVE` | 0x60 | 移除成员 |
| `GR_RIGHT_PROMOTE` | 0xC0 | 晋升成员 |
| `GR_RIGHT_DEMOTE` | 0x140 | 降级成员 |
| `GR_RIGHT_SETMOTD` | 0x1040 | 设置公会信息 |
| `GR_RIGHT_EPNOTE` | 0x2040 | 编辑公开备注 |
| `GR_RIGHT_VIEWOFFNOTE` | 0x4040 | 查看官员备注 |
| `GR_RIGHT_EOFFNOTE` | 0x8040 | 编辑官员备注 |
| `GR_RIGHT_MODIFY_GUILD_INFO` | 0x10040 | 修改公会信息 |
| `GR_RIGHT_WITHDRAW_GOLD_LOCK` | 0x20000 | 锁定金币提取 |
| `GR_RIGHT_WITHDRAW_REPAIR` | 0x40000 | 修理提取 |
| `GR_RIGHT_WITHDRAW_GOLD` | 0x80000 | 提取金币 |
| `GR_RIGHT_CREATE_GUILD_EVENT` | 0x100000 | 创建公会事件 |
| `GR_RIGHT_ALL` | 0x001DF1FF | 全部权限 |

### 2.4 GuildBankEventLogTypes（`Guild.h:195-206`）

| 类型 | 值 | 含义 |
|------|---|------|
| `GUILD_BANK_LOG_DEPOSIT_ITEM` | 1 | 存入物品 |
| `GUILD_BANK_LOG_WITHDRAW_ITEM` | 2 | 取出物品 |
| `GUILD_BANK_LOG_MOVE_ITEM` | 3 | 移动物品 |
| `GUILD_BANK_LOG_DEPOSIT_MONEY` | 4 | 存入金币 |
| `GUILD_BANK_LOG_WITHDRAW_MONEY` | 5 | 取出金币 |
| `GUILD_BANK_LOG_REPAIR_MONEY` | 6 | 修理消耗 |
| `GUILD_BANK_LOG_MOVE_ITEM2` | 7 | 移动物品（方向2） |
| `GUILD_BANK_LOG_BUY_SLOT` | 9 | 购买标签页 |

### 2.5 GuildEventLogTypes（`Guild.h:208-216`）

| 类型 | 值 | 含义 |
|------|---|------|
| `GUILD_EVENT_LOG_INVITE_PLAYER` | 1 | 邀请玩家 |
| `GUILD_EVENT_LOG_JOIN_GUILD` | 2 | 加入公会 |
| `GUILD_EVENT_LOG_PROMOTE_PLAYER` | 3 | 晋升玩家 |
| `GUILD_EVENT_LOG_DEMOTE_PLAYER` | 4 | 降级玩家 |
| `GUILD_EVENT_LOG_UNINVITE_PLAYER` | 5 | 踢出玩家 |
| `GUILD_EVENT_LOG_LEAVE_GUILD` | 6 | 离开公会 |

### 2.6 其他枚举

- **GuildCommandType** (`Guild.h:97-115`)：公会操作命令类型（创建/邀请/退出/晋升/降级等 16 种）
- **GuildCommandError** (`Guild.h:117-143`)：操作错误码（14 种）
- **GuildEvents** (`Guild.h:145-167`)：公会事件类型（20 种，用于客户端通知）
- **GuildBankRights** (`Guild.h:185-193`)：银行权限位（VIEW_TAB=0x01, PUT_ITEM=0x02, UPDATE_TEXT=0x04, FULL=0xFF）
- **GuildMemberFlags** (`Guild.h:228-235`)：成员状态标志（ONLINE/AFK/DND/MOBILE）
- **PetitionTurns** / **PetitionSigns** (`Guild.h:169-183`)：公会创建请愿书相关
- **GuildEmblemError** (`Guild.h:218-226`)：战袍设置错误码

---

## 3. 类层次结构

```
GuildMgr (全局单例 sGuildMgr)
└── GuildContainer: unordered_map<uint32, Guild*>

Guild
├── Member (嵌套类)
├── RankInfo (嵌套类)
├── BankTab (嵌套类)
├── LogEntry (嵌套基类)
│   ├── EventLogEntry
│   └── BankEventLogEntry
├── LogHolder<Entry> (模板类)
├── MoveItemData (嵌套基类 - 物品移动)
│   ├── PlayerMoveItemData
│   └── BankMoveItemData
└── EmblemInfo (独立类 - 战袍信息)
```

---

## 4. 核心类详解

### 4.1 Guild 类（`Guild.h:291-868`）

```
Guild
├── 核心字段:
│   ├── m_id: uint32                       // 公会 ID
│   ├── m_name: string                     // 公会名称
│   ├── m_leaderGuid: ObjectGuid           // 会长 GUID
│   ├── m_motd: string                     // 公会公告
│   ├── m_info: string                     // 公会信息
│   ├── m_createdDate: time_t              // 创建日期
│   ├── m_emblemInfo: EmblemInfo           // 战袍信息
│   ├── m_accountsNumber: uint32           // 不同账号数
│   └── m_bankMoney: uint64                // 银行金币
│
├── 集合:
│   ├── m_ranks: vector<RankInfo>          // 等级列表（索引即 rankId）
│   ├── m_members: unordered_map<uint32, Member>  // 成员（key=玩家lowGUID）
│   ├── m_bankTabs: vector<BankTab>        // 银行标签页
│   ├── m_eventLog: LogHolder<EventLogEntry>      // 事件日志
│   └── m_bankEventLog[7]: LogHolder<BankEventLogEntry>  // 银行日志（6标签+1金币）
│
├── 公共 API (Handle*):
│   ├── Create / Disband / HandleDisband
│   ├── HandleRoster / HandleQuery / SendInfo
│   ├── HandleSetMOTD / HandleSetInfo / HandleSetEmblem / SetName
│   ├── HandleSetLeader / HandleSetBankTabInfo / HandleSetMemberNote
│   ├── HandleSetRankInfo / HandleBuyBankTab
│   ├── HandleInviteMember / HandleAcceptMember
│   ├── HandleLeaveMember / HandleRemoveMember
│   ├── HandleUpdateMemberRank / HandleAddNewRank / HandleRemoveRank
│   ├── HandleMemberDepositMoney / HandleMemberWithdrawMoney
│   ├── HandleMemberLogout
│   ├── SwapItems / SwapItemsWithInventory / SetBankTabText
│   ├── UpdateMemberData / OnPlayerStatusChange
│   └── MassInviteToEvent / BroadcastToGuild / BroadcastPacket
│
└── 私有方法:
    ├── _CreateNewBankTab / _CreateDefaultGuildRanks / _CreateRank
    ├── _UpdateAccountsNumber / _IsLeader / _SetLeaderGUID
    ├── _ModifyBankMoney / _DeleteBankItems
    ├── _LogEvent / _LogBankEvent
    ├── _MoveItems / _DoItemsMove / _RemoveItem / _GetItem
    ├── _GetMemberRemainingSlots / _GetMemberRemainingMoney
    ├── _SendBankContent / _SendBankMoneyUpdate / _SendBankContentUpdate
    ├── _SendBankList / _BroadcastEvent
    └── _SetRankBankMoneyPerDay / _SetRankBankTabRightsAndSlots / ...
```

### 4.2 Member 嵌套类（`Guild.h:295-378`）

```
Member
├── m_guildId: uint32
├── m_guid: ObjectGuid
├── m_name: string
├── m_zoneId: uint32
├── m_level: uint8
├── m_class: uint8
├── m_gender: uint8
├── m_flags: uint8                   // GuildMemberFlags 位掩码
├── m_logoutTime: uint64
├── m_accountId: uint32
├── m_rankId: uint8
├── m_publicNote: string
├── m_officerNote: string
├── m_bankWithdraw[7]: array<int32>  // Tab0-5 + Money 的每日提取量
└── receiveGuildBankUpdatePackets: bool  // 是否订阅银行增量更新
```

**关键行为：**
- `GetBankWithdrawValue(tabId)`：会长始终返回 UNLIMITED
- `UpdateBankWithdrawValue()`：更新后立即写入 DB
- `ChangeRank()`：同步更新在线玩家的 rank 字段
- `FindPlayer()`：通过 `ObjectAccessor::FindConnectedPlayer()` 查找

### 4.3 RankInfo 嵌套类（`Guild.h:508-554`）

```
RankInfo
├── m_guildId: uint32
├── m_rankId: uint8
├── m_name: string
├── m_rights: uint32                  // GuildRankRights 位掩码
├── m_bankMoneyPerDay: uint32         // 每日金币提取限额
└── m_bankTabRightsAndSlots[6]: array<GuildBankRightsAndSlots>
    └── GuildBankRightsAndSlots { tabId, rights, slots }
```

**关键约束：** 会长（rankId=0）的权限永远为 `GR_RIGHT_ALL`，金币限额永远为 `UNLIMITED`，所有 Set* 方法都会强制覆盖。

### 4.4 BankTab 嵌套类（`Guild.h:556-586`）

```
BankTab
├── m_guildId: uint32
├── m_tabId: uint8
├── m_items[98]: array<Item*>        // 98 个物品槽位
├── m_name: string                   // 标签页名称
├── m_icon: string                   // 标签页图标
└── m_text: string                   // 标签页备注文本（最长 500 字符）
```

### 4.5 LogHolder 模板类（`Guild.h:482-506`）

```
LogHolder<Entry>
├── m_guildId: uint32
├── m_log: list<Entry>               // 有序日志列表（首元素最旧）
├── m_maxRecords: uint32             // 最大记录数（来自配置）
└── m_nextGUID: uint32               // 循环 GUID 计数器

行为:
- CanInsert(): m_log.size() < m_maxRecords
- AddEvent(): 超限时 pop_front() 移除最旧记录，emplace_back 新记录
- GetNextGUID(): 循环递增 (nextGUID + 1) % m_maxRecords
```

### 4.6 MoveItemData 类层次（`Guild.h:588-673`）

物品移动的抽象层，统一处理银行→银行、银行→背包、背包→银行的物品操作：

```
MoveItemData (抽象基类)
├── m_pGuild, m_pPlayer, m_container, m_slotId
├── m_pItem, m_pClonedItem, m_vec (ItemPosCountVec)
├── 纯虚函数: IsBank, InitItem, RemoveItem, StoreItem, LogBankEvent
├── 虚函数: HasStoreRights, HasWithdrawRights
│
├── PlayerMoveItemData (玩家背包)
│   ├── IsBank() → false
│   └── 无额外权限检查（玩家自己的背包）
│
└── BankMoveItemData (公会银行)
    ├── IsBank() → true
    ├── HasStoreRights() → 检查 rank 的 GUILD_BANK_RIGHT_PUT_ITEM
    ├── HasWithdrawRights() → 检查剩余提取次数和权限
    └── 私有: _StoreItem, _ReserveSpace, CanStoreItemInTab
```

---

## 5. 数据库表

所有公会相关表存储在 **acore_characters** 数据库中。

### 5.1 guild（公会主表）

加载步骤 1，加载时自动清理无成员的孤儿公会：
```sql
DELETE g FROM guild g LEFT JOIN guild_member gm ON g.guildid = gm.guildid WHERE gm.guildid IS NULL
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildid` | int unsigned | 公会 ID（主键） |
| `name` | varchar(24) | 公会名称 |
| `leaderguid` | int unsigned | 会长角色 GUID |
| `EmblemStyle` | tinyint unsigned | 战袍样式 |
| `EmblemColor` | tinyint unsigned | 战袍颜色 |
| `BorderStyle` | tinyint unsigned | 边框样式 |
| `BorderColor` | tinyint unsigned | 边框颜色 |
| `BackgroundColor` | tinyint unsigned | 背景颜色 |
| `info` | text | 公会信息 |
| `motd` | varchar(128) | 公会公告 |
| `createdate` | int unsigned | 创建时间戳 |
| `BankMoney` | bigint unsigned | 银行金币 |

**加载查询：**
```sql
SELECT g.guildid, g.name, g.leaderguid, g.EmblemStyle, g.EmblemColor,
       g.BorderStyle, g.BorderColor, g.BackgroundColor, g.info, g.motd,
       g.createdate, g.BankMoney, COUNT(gbt.guildid)
FROM guild g LEFT JOIN guild_bank_tab gbt ON g.guildid = gbt.guildid
GROUP BY g.guildid ORDER BY g.guildid ASC
```

### 5.2 guild_rank（等级表）

加载步骤 2，清理孤儿记录：
```sql
DELETE gr FROM guild_rank gr LEFT JOIN guild g ON gr.guildId = g.guildId WHERE g.guildId IS NULL
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildId` | int unsigned | 公会 ID |
| `rid` | tinyint unsigned | 等级 ID（0=会长） |
| `rname` | varchar(24) | 等级名称 |
| `rights` | int unsigned | 权限位掩码 |
| `BankMoneyPerDay` | int unsigned | 每日金币提取限额 |

### 5.3 guild_member（成员表）

加载步骤 3，联合查询 characters 表和 guild_member_withdraw 表：
```sql
SELECT guildid, gm.guid, `rank`, pnote, offnote,
       w.tab0, w.tab1, w.tab2, w.tab3, w.tab4, w.tab5, w.money,
       c.name, c.level, c.class, c.gender, c.zone, c.account, c.logout_time
FROM guild_member gm
LEFT JOIN guild_member_withdraw w ON gm.guid = w.guid
LEFT JOIN characters c ON c.guid = gm.guid
ORDER BY guildid ASC
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildId` | int unsigned | 公会 ID |
| `guid` | int unsigned | 角色 GUID |
| `rank` | tinyint unsigned | 等级 ID |
| `pnote` | varchar(32) | 公开备注 |
| `offnote` | varchar(32) | 官员备注 |

### 5.4 guild_member_withdraw（成员每日提取记录表）

| 列 | 类型 | 含义 |
|----|------|------|
| `guid` | int unsigned | 角色 GUID（主键） |
| `tab0` ~ `tab5` | int unsigned | Tab0-5 的今日已提取物品数 |
| `money` | int unsigned | 今日已提取金币数 |

**重置：** 每日维护时 `TRUNCATE guild_member_withdraw`

### 5.5 guild_bank_right（银行标签页权限表）

加载步骤 4，清理孤儿记录：
```sql
DELETE gbr FROM guild_bank_right gbr LEFT JOIN guild g ON gbr.guildId = g.guildId WHERE g.guildId IS NULL
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildId` | int unsigned | 公会 ID |
| `TabId` | tinyint unsigned | 标签页 ID |
| `rid` | tinyint unsigned | 等级 ID |
| `gbright` | tinyint unsigned | 银行权限 |
| `SlotPerDay` | int unsigned | 每日可提取物品数 |

### 5.6 guild_eventlog（事件日志表）

加载步骤 5，清理超限记录：
```sql
DELETE FROM guild_eventlog WHERE LogGuid > {CONFIG_GUILD_EVENT_LOG_COUNT}
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildid` | int unsigned | 公会 ID |
| `LogGuid` | int unsigned | 日志 GUID（循环） |
| `EventType` | tinyint unsigned | GuildEventLogTypes |
| `PlayerGuid1` | int unsigned | 玩家1 GUID |
| `PlayerGuid2` | int unsigned | 玩家2 GUID |
| `NewRank` | tinyint unsigned | 新等级 |
| `TimeStamp` | int unsigned | 时间戳 |

### 5.7 guild_bank_eventlog（银行事件日志表）

加载步骤 6，清理超限记录：
```sql
DELETE FROM guild_bank_eventlog WHERE LogGuid > {CONFIG_GUILD_BANK_EVENT_LOG_COUNT}
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildid` | int unsigned | 公会 ID |
| `TabId` | tinyint unsigned | 标签页 ID（100=金币日志） |
| `LogGuid` | int unsigned | 日志 GUID（循环） |
| `EventType` | tinyint unsigned | GuildBankEventLogTypes |
| `PlayerGuid` | int unsigned | 玩家 GUID |
| `ItemOrMoney` | int unsigned | 物品ID或金币数 |
| `ItemStackCount` | smallint unsigned | 物品堆叠数 |
| `DestTabId` | tinyint unsigned | 目标标签页 |
| `TimeStamp` | int unsigned | 时间戳 |

### 5.8 guild_bank_tab（银行标签页表）

加载步骤 7，清理孤儿记录：
```sql
DELETE gbt FROM guild_bank_tab gbt LEFT JOIN guild g ON gbt.guildId = g.guildId WHERE g.guildId IS NULL
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildid` | int unsigned | 公会 ID |
| `TabId` | tinyint unsigned | 标签页 ID |
| `TabName` | varchar(16) | 标签页名称 |
| `TabIcon` | varchar(64) | 标签页图标 |
| `TabText` | text | 标签页备注 |

### 5.9 guild_bank_item（银行物品表）

加载步骤 8，联合 item_instance 表加载物品详情：
```sql
SELECT creatorGuid, giftCreatorGuid, count, duration, charges, flags,
       enchantments, randomPropertyId, durability, playedTime, text,
       guildid, TabId, SlotId, item_guid, itemEntry
FROM guild_bank_item gbi INNER JOIN item_instance ii ON gbi.item_guid = ii.guid
```

| 列 | 类型 | 含义 |
|----|------|------|
| `guildid` | int unsigned | 公会 ID |
| `TabId` | tinyint unsigned | 标签页 ID |
| `SlotId` | tinyint unsigned | 槽位 ID |
| `item_guid` | int unsigned | 物品实例 GUID |

### 5.10 petition（公会创建请愿书表）

公会创建流程中的请愿书记录，在玩家从公会注册员 NPC 购买请愿书时创建。

| 列 | 类型 | 含义 |
|----|------|------|
| `ownerguid` | int unsigned | 请愿书所有者（发起创建公会的玩家）GUID |
| `petitionguid` | int unsigned | 请愿书物品 GUID |
| `petition_id` | int unsigned | 请愿书 ID（自增） |
| `name` | varchar(24) | 拟创建的公会名称 |
| `type` | tinyint unsigned | 类型（0=公会，1=竞技场战队） |

**索引：**
- 主键: `(ownerguid, type)`
- 唯一索引: `(ownerguid, petitionguid)`
- 索引: `(petition_id)`

### 5.11 petition_sign（请愿书签名表）

其他玩家对请愿书的签名记录，当签名数达到要求（默认9人）后可提交创建公会。

| 列 | 类型 | 含义 |
|----|------|------|
| `ownerguid` | int unsigned | 请愿书所有者 GUID |
| `petitionguid` | int unsigned | 请愿书物品 GUID |
| `petition_id` | int unsigned | 请愿书 ID |
| `playerguid` | int unsigned | 签名玩家 GUID |
| `player_account` | int unsigned | 签名玩家账号 ID |
| `type` | tinyint unsigned | 类型（0=公会，1=竞技场战队） |

**索引：**
- 主键: `(petitionguid, playerguid)`
- 索引: `(playerguid)`
- 索引: `(ownerguid)`
- 索引: `(petition_id, playerguid)`

**约束：** 同一账号下的多个角色不能重复签名（通过 `player_account` 检查）。

### 5.12 表间关系图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              acore_characters 数据库                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌──────────────┐       ┌──────────────┐       ┌──────────────────┐           │
│   │    guild     │──────<│  guild_rank  │       │ guild_bank_right │           │
│   │  (公会主表)   │       │  (等级表)     │       │  (银行权限表)     │           │
│   └──────┬───────┘       └──────────────┘       └────────┬─────────┘           │
│          │                                               │                     │
│          │ guildid                                       │ guildId, rid        │
│          │                                               │                     │
│          ├───────────────────────────────────────────────┤                     │
│          │                                               │                     │
│          ▼                                               ▼                     │
│   ┌──────────────┐       ┌──────────────────┐    ┌──────────────┐             │
│   │ guild_member │──────<│guild_member_     │    │ guild_bank_  │             │
│   │  (成员表)     │       │    withdraw      │    │    right     │             │
│   └──────┬───────┘       │ (每日提取记录)    │    └──────────────┘             │
│          │               └──────────────────┘                                  │
│          │                                                                      │
│          │ guid ──────────────────────────────┐                                │
│          │                                    │                                │
│          ▼                                    ▼                                │
│   ┌──────────────┐                     ┌──────────────┐                        │
│   │  characters  │                     │   petition   │                        │
│   │  (角色表)     │                     │ (请愿书表)    │                        │
│   └──────────────┘                     └──────┬───────┘                        │
│                                               │                                │
│                                               │ petitionguid                   │
│                                               ▼                                │
│                                        ┌──────────────┐                        │
│                                        │petition_sign │                        │
│                                        │ (签名表)      │                        │
│                                        └──────────────┘                        │
│                                                                                 │
│   ┌──────────────┐       ┌────────────────────┐    ┌───────────────────┐       │
│   │guild_bank_tab│──────<│  guild_bank_item   │───>│  item_instance    │       │
│   │ (标签页表)    │       │   (银行物品表)      │    │  (物品实例表)      │       │
│   └──────────────┘       └────────────────────┘    └───────────────────┘       │
│                                                                                 │
│   ┌──────────────┐       ┌────────────────────┐                                │
│   │guild_eventlog│       │guild_bank_eventlog │                                │
│   │ (事件日志)    │       │  (银行事件日志)     │                                │
│   └──────────────┘       └────────────────────┘                                │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

关系说明:
──────────
guild.guildid ──────────────> guild_rank.guildId           (1:N)
guild.guildid ──────────────> guild_member.guildId         (1:N)
guild.guildid ──────────────> guild_bank_tab.guildid       (1:N)
guild.guildid ──────────────> guild_bank_item.guildid      (1:N)
guild.guildid ──────────────> guild_bank_right.guildId     (1:N)
guild.guildid ──────────────> guild_eventlog.guildid       (1:N)
guild.guildid ──────────────> guild_bank_eventlog.guildid  (1:N)

guild_member.guid ──────────> characters.guid              (1:1)
guild_member.guid ──────────> guild_member_withdraw.guid   (1:1)

guild_rank.(guildId,rid) ──> guild_bank_right.(guildId,rid) (1:N)
guild_bank_tab.(guildid,TabId) ──> guild_bank_item.(guildid,TabId) (1:N)
guild_bank_item.item_guid ─> item_instance.guid            (N:1)

petition.ownerguid ─────────> petition_sign.ownerguid      (1:N)
petition.petitionguid ──────> petition_sign.petitionguid   (1:N)
```

### 5.13 各表加载顺序与清理策略

| 步骤 | 表名 | 加载顺序 | 孤儿清理 SQL |
|------|------|----------|-------------|
| 1 | `guild` | 最先 | `DELETE g FROM guild g LEFT JOIN guild_member gm ON g.guildid = gm.guildid WHERE gm.guildid IS NULL` |
| 2 | `guild_rank` | 依赖 guild | `DELETE gr FROM guild_rank gr LEFT JOIN guild g ON gr.guildId = g.guildId WHERE g.guildId IS NULL` |
| 3 | `guild_member` | 依赖 guild + characters | 联合 guild_member_withdraw 和 characters 加载 |
| 4 | `guild_bank_right` | 依赖 guild | `DELETE gbr FROM guild_bank_right gbr LEFT JOIN guild g ON gbr.guildId = g.guildId WHERE g.guildId IS NULL` |
| 5 | `guild_eventlog` | 依赖 guild | `DELETE FROM guild_eventlog WHERE LogGuid > {CONFIG_COUNT}` |
| 6 | `guild_bank_eventlog` | 依赖 guild | `DELETE FROM guild_bank_eventlog WHERE LogGuid > {CONFIG_COUNT}` |
| 7 | `guild_bank_tab` | 依赖 guild | `DELETE gbt FROM guild_bank_tab gbt LEFT JOIN guild g ON gbt.guildId = g.guildId WHERE g.guildId IS NULL` |
| 8 | `guild_bank_item` | 依赖 guild_bank_tab + item_instance | 清理无关联物品 |

**注意：** `petition` 和 `petition_sign` 表不在此加载流程中，它们仅在公会创建流程中使用（购买请愿书→收集签名→提交创建），公会创建完成后相关记录会被删除。

---

## 6. 数据加载流程（`GuildMgr::LoadGuilds()`）

```
GuildMgr::LoadGuilds()                              // GuildMgr.cpp:92-405
    │
    ├── Step 1: 加载公会基本信息（guild 表）
    │   └── 清理无成员的孤儿公会 → 创建 Guild 对象 → AddGuild()
    │
    ├── Step 2: 加载等级数据（guild_rank 表）
    │   └── 清理孤儿等级 → guild->LoadRankFromDB()
    │
    ├── Step 3: 加载成员数据（guild_member + guild_member_withdraw + characters）
    │   └── 清理孤儿成员和提取记录 → guild->LoadMemberFromDB()
    │
    ├── Step 4: 加载银行权限（guild_bank_right 表）
    │   └── 清理孤儿权限 → guild->LoadBankRightFromDB()
    │
    ├── Step 5: 加载事件日志（guild_eventlog 表）
    │   └── 清理超限日志 → guild->LoadEventLogFromDB()
    │
    ├── Step 6: 加载银行事件日志（guild_bank_eventlog 表）
    │   └── 清理超限日志 → guild->LoadBankEventLogFromDB()
    │
    ├── Step 7: 加载银行标签页（guild_bank_tab 表）
    │   └── 清理孤儿标签页 → guild->LoadBankTabFromDB()
    │
    ├── Step 8: 加载银行物品（guild_bank_item + item_instance）
    │   └── 清理孤儿物品 → guild->LoadBankItemFromDB()
    │
    └── Step 9: 验证数据完整性
        └── guild->Validate() → 失败则 delete guild
```

**加载顺序的重要性：** 每步依赖前一步的数据。公会必须先加载才能关联等级、成员等。所有清理操作（孤儿记录删除）在加载前执行。

---

## 7. 核心流程详解

### 7.1 公会创建（`Guild::Create()` — `Guild.cpp:1057-1113`）

```
Create(leader, name)
    │
    ├── 检查名称唯一性: sGuildMgr->GetGuildByName(name)
    ├── 生成 ID: sGuildMgr->GenerateGuildId()
    ├── 设置默认值: motd = "No message set.", bankMoney = 0
    ├── 写入 guild 表 (CHAR_INS_GUILD)
    ├── _CreateDefaultGuildRanks(locale)
    │   └── 创建 5 个默认等级: 会长(ALL), 官员(ALL), 老兵(聊天), 会员(聊天), 新人(聊天)
    ├── AddMember(leaderGuid, GR_GUILDMASTER)
    │   ├── 创建 Member 对象
    │   ├── player->SetInGuild(id) / SetRank(0)
    │   ├── member.SaveToDB() → CHAR_INS_GUILD_MEMBER
    │   ├── _UpdateAccountsNumber()
    │   ├── _LogEvent(JOIN_GUILD)
    │   ├── _BroadcastEvent(GE_JOINED)
    │   └── sScriptMgr->OnGuildAddMember()
    ├── 根据 CONFIG_GUILD_BANK_INITIAL_TABS 创建初始银行标签页
    └── sScriptMgr->OnGuildCreate()
```

### 7.2 公会解散（`Guild::Disband()` — `Guild.cpp:1116-1164`）

```
Disband()
    │
    ├── sScriptMgr->OnGuildDisband()
    ├── _BroadcastEvent(GE_DISBANDED)
    ├── 循环 DeleteMember() 移除所有成员
    │   └── 每个成员: 清除玩家公会ID, 删除请愿书, 从DB删除
    ├── 删除所有数据库记录:
    │   ├── CHAR_DEL_GUILD
    │   ├── CHAR_DEL_GUILD_RANKS
    │   ├── CHAR_DEL_GUILD_BANK_TABS
    │   ├── _DeleteBankItems(trans, true)  // 删除物品实例
    │   ├── CHAR_DEL_GUILD_BANK_ITEMS
    │   ├── CHAR_DEL_GUILD_BANK_RIGHTS
    │   ├── CHAR_DEL_GUILD_BANK_EVENTLOGS
    │   ├── CHAR_DEL_GUILD_EVENTLOGS
    │   └── CHAR_DEL_GUILD_MEMBERS
    ├── sGuildMgr->RemoveGuild(m_id)
    └── 通知日历管理器删除公会事件
```

### 7.3 成员移除与会长自动继承（`Guild::DeleteMember()` — `Guild.cpp:2286-2345`）

当会长被移除时，自动将公会转让给下一位最高等级的在线成员：

```
DeleteMember(guid, isDisbanding, isKicked, canDeleteGuild)
    │
    ├── if m_leaderGuid == guid && !isDisbanding:
    │   ├── 查找继承者: 遍历所有成员
    │   │   ├── 优先: 在线 + 非会长的最低 rankId
    │   │   └── 次选: 离线 + 非会长的最低 rankId
    │   ├── 找到继承者 → _SetLeaderGUID(newLeader)
    │   └── 未找到 && canDeleteGuild → Disband()
    │
    ├── 从 m_members 中移除
    ├── 更新玩家/角色缓存: guildId = 0
    ├── 从银行物品中移除该成员的物品（如果被踢出）
    ├── _DeleteMemberFromDB()
    ├── _UpdateAccountsNumber()
    ├── _LogEvent(UNINVITE/LEAVE)
    ├── _BroadcastEvent(GE_REMOVED/GE_LEFT)
    └── sScriptMgr->OnGuildRemoveMember()
```

### 7.4 银行物品移动（`Guild::_MoveItems()` — `Guild.cpp:2678-2729`）

这是公会银行最复杂的逻辑，支持 4 种操作模式：

```
_MoveItems(pSrc, pDest, splitedAmount)
    │
    ├── 1. 初始化源物品: pSrc->InitItem()
    ├── 2. 验证源物品: pSrc->CheckItem(splitedAmount)
    ├── 3. 检查目标存放权限: pDest->HasStoreRights(pSrc)
    ├── 4. 检查源取出权限: pSrc->HasWithdrawRights(pDest)
    │
    ├── [splitedAmount > 0] 分堆模式:
    │   ├── 克隆物品: pSrc->CloneItem()
    │   └── 移动分堆物品到目标
    │
    └── [splitedAmount == 0] 整堆模式:
        ├── 尝试合并到目标空位: _DoItemsMove(merge)
        │   └── 成功 → 完成
        └── 合并失败 → 尝试交换:
            ├── 初始化目标物品: pDest->InitItem()
            ├── 检查反向存放权限
            ├── 检查反向取出权限
            └── 交换两个物品: _DoItemsMove(swap)

_DoItemsMove(pSrc, pDest, sendError, splitedAmount)
    ├── CanStore 检查（双向）
    ├── 记录 GM 日志
    ├── 记录银行事件日志
    ├── RemoveItem（双向）
    ├── StoreItem（双向）
    └── 更新成员提取计数
```

**权限检查矩阵：**

| 操作 | BankMoveItemData::HasStoreRights | BankMoveItemData::HasWithdrawRights |
|------|----------------------------------|-------------------------------------|
| 存入银行 | 检查 rank 的 PUT_ITEM 权限 | 无需检查（是自己的物品） |
| 从银行取出 | 无需检查（是自己的背包） | 检查 rank 权限 + 剩余提取次数 |
| 银行内移动 | 检查目标 tab 的 PUT_ITEM | 检查源 tab 的权限 + 剩余次数 |

### 7.5 验证流程（`Guild::Validate()` — `Guild.cpp:2069-2138`）

加载后的数据完整性验证：

```
Validate()
    │
    ├── 验证等级:
    │   ├── 数量 5~10?
    │   ├── ID 连续无间隔?
    │   └── 每个等级的银行权限完整?
    │   └── 不合法 → 重建默认等级
    │
    ├── 验证成员:
    │   └── rankId > ranksSize → 降级到最低等级
    │
    ├── 验证会长:
    │   ├── 会长不存在 → 自动删除 + 继承
    │   ├── 会长不是 rankId=0 → 修复
    │   └── 多会长检测 (Guild.AllowMultipleGuildMaster 配置)
    │
    └── _UpdateAccountsNumber()
```

### 7.6 每日重置（`GuildMgr::ResetTimes()` — `GuildMgr.cpp:407-414`）

```
ResetTimes()
    ├── 遍历所有公会: guild->ResetTimes()
    │   └── 每个成员: member.ResetValues()  // m_bankWithdraw 全部归零
    └── TRUNCATE guild_member_withdraw      // 清空提取记录表
```

---

## 8. 公会银行系统

### 8.1 标签页价格

每个标签页的价格独立配置（`Guild.cpp:94-113`）：

| TabId | 配置键 | 默认价格 |
|-------|--------|----------|
| 0 | `Guild.Bank.TabCost.0` | 100 金 |
| 1 | `Guild.Bank.TabCost.1` | 200 金 |
| 2 | `Guild.Bank.TabCost.2` | 300 金 |
| 3 | `Guild.Bank.TabCost.3` | 500 金 |
| 4 | `Guild.Bank.TabCost.4` | 1000 金 |
| 5 | `Guild.Bank.TabCost.5` | 2000 金 |

### 8.2 标签页购买流程（`Guild::HandleBuyBankTab()` — `Guild.cpp:1411-1439`）

```
HandleBuyBankTab(session, tabId)
    │
    ├── 检查 tabId == _GetPurchasedTabsSize()（必须是下一个待购页）
    ├── 检查玩家是否在线且有权限
    ├── 检查玩家金币 >= _GetGuildBankTabPrice(tabId)
    ├── 扣除玩家金币
    ├── _CreateNewBankTab()
    │   ├── m_bankTabs.emplace_back(m_id, tabId)
    │   ├── 写入 DB: CHAR_DEL_GUILD_BANK_TAB + CHAR_INS_GUILD_BANK_TAB
    │   └── 为所有等级创建该 tab 的默认权限
    ├── 更新银行内容给所有订阅成员
    └── _BroadcastEvent(GE_BANK_TAB_PURCHASED)
```

### 8.3 金币操作

**存入金币（`HandleMemberDepositMoney()`）：**
- 无权限检查，任何成员都可以存入
- 从玩家背包扣除金币
- `_ModifyBankMoney(trans, amount, true)` 增加银行金币
- 记录 GUILD_BANK_LOG_DEPOSIT_MONEY 日志

**提取金币（`HandleMemberWithdrawMoney()`）：**
- 检查 GR_RIGHT_WITHDRAW_GOLD 或 GR_RIGHT_WITHDRAW_REPAIR 权限
- 检查每日剩余提取额度（`_GetMemberRemainingMoney()`）
- 会长无限制
- 记录 GUILD_BANK_LOG_WITHDRAW_MONEY 或 GUILD_BANK_LOG_REPAIR_MONEY

---

## 9. 公会创建请愿书流程（`PetitionsHandler.cpp`）

公会的创建不是直接创建，而是通过请愿书（Petition）流程：

**步骤 1：购买请愿书（`HandlePetitionBuyOpcode` — line 32）**
- 从公会注册员 NPC 购买请愿书物品
- charterid = GUILD_CHARTER, cost = `Guild.CharterCost`（默认 1000 铜币 = 10 银币）
- 验证：玩家未加入其他公会、公会名称未被占用
- 扣除金币，在玩家背包创建请愿书物品
- 生成请愿书 ID，存入 DB `petition` 表

**步骤 2：收集签名（`HandlePetitionSignOpcode` — line 394）**
- 其他玩家对请愿书签名
- 验证：签名者未加入公会、非请愿书主人、未重复签名、非同账号

**步骤 3：提交请愿书（`HandleTurnInPetitionOpcode` — line 645）**
- 将签好的请愿书交回 NPC
- 验证：签名数 >= `MinPetitionSigns`（默认 9）
- 创建公会：`Guild* guild = new Guild; guild->Create(player, name);`
- 注册：`sGuildMgr->AddGuild(guild);`
- 所有签名者自动加入公会：`guild->AddMember(signerGuid);`
- 删除请愿书 DB 记录

相关枚举：
- `PETITION_TURN_OK (0)` — 创建成功
- `PETITION_TURN_ALREADY_IN_GUILD (2)` — 签名者已在其他公会
- `PETITION_TURN_NEED_MORE_SIGNATURES (4)` — 签名不足
- `PETITION_SIGN_OK (0)` — 签名成功
- `PETITION_SIGN_ALREADY_SIGNED (1)` — 已签名
- `PETITION_SIGN_ALREADY_IN_GUILD (2)` — 已在公会
- `PETITION_SIGN_CANT_SIGN_OWN (3)` — 不能签自己的
- `PETITION_SIGN_NOT_SERVER (4)` — 不在同一服务器

---

## 10. 网络协议

公会模块使用 `WorldPackets::Guild` 命名空间下的数据包（定义在 `GuildPackets.h` 中）：

### 10.1 服务端发送的关键数据包

| 数据包 | 用途 |
|--------|------|
| `GuildCommandResult` | 操作结果通知 |
| `PlayerSaveGuildEmblem` | 战袍保存结果 |
| `GuildEventLogQueryResults` | 事件日志查询结果 |
| `GuildBankLogQueryResults` | 银行日志查询结果 |
| `GuildBankTextQueryResult` | 银行标签页文本 |
| `GuildBankListQueryResult` | 银行内容列表 |
| SMSG_GUILD_EVENT | 公会事件广播 |

### 10.2 登录时发送的信息

```
SendLoginInfo(session)                              // Guild.cpp:1905-1926
    ├── 公会信息（名称、会长、等级数等）
    ├── 成员名单（Roster）
    ├── 公会公告
    └── 银行权限信息
```

---

## 11. 脚本系统集成

### 11.1 脚本钩子（完整列表 — `GuildScript.h`）

GuildScript 不是数据库绑定的（`IsDatabaseBound() = false`），所有公会共享同一脚本实例。

| 钩子 | 触发位置 | 参数 | 说明 |
|------|----------|------|------|
| `OnAddMember` | `AddMember()` | Guild*, Player*, uint8& plRank | 可修改 rank |
| `OnRemoveMember` | `DeleteMember()` | Guild*, Player*, bool isDisbanding, bool isKicked | |
| `OnMOTDChanged` | `HandleSetMOTD()` | Guild*, const string& newMotd | |
| `OnInfoChanged` | `HandleSetInfo()` | Guild*, const string& newInfo | |
| `OnCreate` | `Create()` | Guild*, Player* leader, const string& name | |
| `OnDisband` | `Disband()` | Guild* | |
| `OnMemberWitdrawMoney` | `HandleMemberWithdrawMoney()` | Guild*, Player*, uint32& amount, bool isRepair | 可修改金额 |
| `OnMemberDepositMoney` | `HandleMemberDepositMoney()` | Guild*, Player*, uint32& amount | 可修改金额 |
| `OnItemMove` | `_MoveItems()` | Guild*, Player*, Item*, isSrcBank, srcContainer, srcSlotId, isDestBank, destContainer, destSlotId | |
| `OnEvent` | `_LogEvent()` | Guild*, uint8 eventType, playerGuid1, playerGuid2, newRank | |
| `OnBankEvent` | `_LogBankEvent()` | Guild*, uint8 eventType, tabId, playerGuid, itemOrMoney, itemStackCount, destTabId | |
| `CanGuildSendBankList` | `_SendBankList()` | Guild const*, WorldSession*, tabId, sendAllSlots | 返回 bool，可阻止发送 |

### 11.2 GuildScript 基类（`GuildScript.h:42-82`）

```cpp
class GuildScript : public ScriptObject
{
    bool IsDatabaseBound() const override { return false; }

    virtual void OnAddMember(Guild*, Player*, uint8& plRank) { }
    virtual void OnRemoveMember(Guild*, Player*, bool isDisbanding, bool isKicked) { }
    virtual void OnMOTDChanged(Guild*, const string& newMotd) { }
    virtual void OnInfoChanged(Guild*, const string& newInfo) { }
    virtual void OnCreate(Guild*, Player* leader, const string& name) { }
    virtual void OnDisband(Guild*) { }
    virtual void OnMemberWitdrawMoney(Guild*, Player*, uint32& amount, bool isRepair) { }
    virtual void OnMemberDepositMoney(Guild*, Player*, uint32& amount) { }
    virtual void OnItemMove(Guild*, Player*, Item*, bool isSrcBank, uint8 srcContainer,
                           uint8 srcSlotId, bool isDestBank, uint8 destContainer, uint8 destSlotId) { }
    virtual void OnEvent(Guild*, uint8 eventType, ObjectGuid::LowType playerGuid1,
                        ObjectGuid::LowType playerGuid2, uint8 newRank) { }
    virtual void OnBankEvent(Guild*, uint8 eventType, uint8 tabId, ObjectGuid::LowType playerGuid,
                            uint32 itemOrMoney, uint16 itemStackCount, uint8 destTabId) { }
    virtual bool CanGuildSendBankList(Guild const*, WorldSession*, uint8 tabId, bool sendAllSlots) { return true; }
};
```

---

## 12. 配置项汇总

### 12.1 世界配置

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `Guild.BankTabCost0` | 1000000 (100g) | 第1个银行标签页价格（铜币） |
| `Guild.BankTabCost1` | 2500000 (250g) | 第2个标签页价格 |
| `Guild.BankTabCost2` | 5000000 (500g) | 第3个标签页价格 |
| `Guild.BankTabCost3` | 10000000 (1000g) | 第4个标签页价格 |
| `Guild.BankTabCost4` | 25000000 (2500g) | 第5个标签页价格 |
| `Guild.BankTabCost5` | 50000000 (5000g) | 第6个标签页价格 |
| `Guild.EventLogRecordsCount` | GUILD_EVENTLOG_MAX_RECORDS | 事件日志最大记录数（可热重载） |
| `Guild.BankEventLogRecordsCount` | GUILD_BANKLOG_MAX_RECORDS | 银行日志最大记录数（可热重载） |
| `Guild.BankInitialTabs` | 0 | 新公会初始银行标签页数 |
| `Guild.ResetHour` | 6 | 每日重置时间（小时，可热重载） |
| `Guild.MemberLimit` | 0 | 公会成员上限（0=无限制） |
| `Guild.CharterCost` | 1000 (10银) | 公会请愿书价格（铜币） |
| `MinPetitionSigns` | 9 | 创建公会最少签名数 |
| `AllowTwoSide.Interaction.Guild` | false | 是否允许阵营间公会交互 |

### 12.2 其他配置

| 配置键 | 默认值 | 含义 |
|--------|--------|------|
| `Guild.AllowMultipleGuildMaster` | 0 | 是否允许多个 rankId=0 的成员 |

### 12.3 常量

| 常量 | 值 | 含义 |
|------|---|------|
| `EMBLEM_PRICE` | 10 金 | 战袍设计费用 |
| `MAX_GUILD_BANK_TAB_TEXT_LEN` | 500 | 标签页备注最大长度 |

---

## 13. GM 命令

通过 `cs_guild.cpp` 提供的命令，所有命令均支持控制台执行（Console::Yes）：

| 命令 | 权限 | 用途 |
|------|------|------|
| `.guild create [leader] [guildname]` | RBAC_PERM_COMMAND_GUILD_CREATE | 以指定玩家为会长创建公会 |
| `.guild delete [guildname]` | RBAC_PERM_COMMAND_GUILD_DELETE | 解散公会 |
| `.guild invite [playername] [guildname]` | RBAC_PERM_COMMAND_GUILD_INVITE | 邀请玩家加入公会 |
| `.guild uninvite [playername]` | RBAC_PERM_COMMAND_GUILD_UNINVITE | 踢出公会 |
| `.guild rank [playername] [rank]` | RBAC_PERM_COMMAND_GUILD_RANK | 设置成员等级 |
| `.guild rename [oldname] [newname]` | RBAC_PERM_COMMAND_GUILD_RENAME | 重命名公会 |
| `.guild info [guildid|guildname]` | RBAC_PERM_COMMAND_GUILD_INFO | 显示公会详情（名称/会长/日期/成员数/金币/MOTD/等级列表） |

---

## 14. GuildMgr 单例（`GuildMgr.h`）

```
GuildMgr
├── GuildStore: unordered_map<uint32, Guild*>    // guildId → Guild*
├── NextGuildId: uint32                          // 自增 ID 生成器
│
├── LoadGuilds()                                 // 启动时 9 步加载
├── AddGuild(guild) / RemoveGuild(id)
├── GetGuildById(id) → Guild*
├── GetGuildByName(name) → Guild*               // 大小写不敏感
├── GetGuildByLeader(guid) → Guild*             // 线性扫描
├── GetGuildNameById(id) → string
├── GenerateGuildId() → uint32                  // 自增，溢出时关闭服务器
└── ResetTimes()                                // 每日重置
```

---

## 15. 关键设计特征

### 15.1 数据一致性保障

- **孤儿记录清理：** 每次加载前使用 LEFT JOIN DELETE 清理所有无关联的子表记录
- **事务操作：** 银行物品移动使用 `CharacterDatabaseTransaction` 保证原子性
- **即时持久化：** 大多数修改操作立即写入 DB（不等待保存周期）
- **Validate() 修复：** 加载后验证并修复不一致数据

### 15.2 会长权限保护

所有修改 RankInfo 的方法都包含会长保护：
```cpp
if (m_rankId == GR_GUILDMASTER)
    rights = GR_RIGHT_ALL;           // 强制全部权限
    money = GUILD_WITHDRAW_MONEY_UNLIMITED;  // 强制无限金币
    rightsAndSlots.SetGuildMasterValues();    // 强制全部银行权限
```

### 15.3 银行更新订阅机制

Member 有 `receiveGuildBankUpdatePackets` 标志：
- 玩家打开银行时订阅
- 银行内容变化时仅向订阅者发送增量更新
- 关闭银行时取消订阅
- 减少不必要的网络流量

### 15.4 循环日志系统

`LogHolder` 使用循环 GUID 机制（`GetNextGUID()`），日志记录数受 `m_maxRecords` 限制。新增记录时如果超出限制，最旧的记录从内存列表移除（但 DB 中通过 DELETE+INSERT 覆盖），实现固定大小的日志窗口。

### 15.5 账号数追踪

`_UpdateAccountsNumber()` 统计公会中不同账号的数量，用于限制同一账号的多个角色加入不同公会。通过遍历所有成员的 accountId，使用 set 去重。

---

## 16. 数据流总览

```
[启动时]
    GuildMgr::LoadGuilds()
    ├── Step 1: guild 表 → Guild 对象创建
    ├── Step 2: guild_rank → RankInfo 加载
    ├── Step 3: guild_member + guild_member_withdraw + characters → Member 加载
    ├── Step 4: guild_bank_right → 银行权限加载
    ├── Step 5: guild_eventlog → 事件日志加载
    ├── Step 6: guild_bank_eventlog → 银行日志加载
    ├── Step 7: guild_bank_tab → BankTab 加载
    ├── Step 8: guild_bank_item + item_instance → 物品实例加载
    └── Step 9: Validate() → 数据完整性验证

[运行时 - 玩家操作]
    WorldSession → HandleGuild*Opcode
        └── Guild::HandleXxx(session, ...)
            ├── 权限检查: _HasRankRight(player, right)
            ├── 业务逻辑执行
            ├── DB 操作 (PreparedStatement)
            ├── 日志记录: _LogEvent() / _LogBankEvent()
            ├── 网络通知: _BroadcastEvent() / _SendBankList()
            └── 脚本回调: sScriptMgr->OnGuild*()

[每日重置]
    GuildMgr::ResetTimes()
    ├── Member::ResetValues() → m_bankWithdraw 全部归零
    └── TRUNCATE guild_member_withdraw

[公会解散]
    Guild::Disband()
    ├── 逐个删除成员
    ├── 删除所有关联 DB 记录（8 张表）
    ├── 删除银行物品实例
    └── sGuildMgr->RemoveGuild()
```
