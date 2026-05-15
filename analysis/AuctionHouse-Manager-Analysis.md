# AzerothCore 拍卖行系统完整分析报告

> 主要分析文件:
>
> - `src/server/game/AuctionHouse/AuctionHouseMgr.cpp` (核心管理器, \~643 行)
> - `src/server/game/AuctionHouse/AuctionHouseMgr.h` (头文件/数据结构)
> - `src/server/game/AuctionHouse/AuctionHouseSearcher.h` (搜索引擎)
> - `src/server/game/Handlers/AuctionHouseHandler.cpp` (网络包处理, \~790 行)
>   分析日期: 2026-05-02

***

## 1. 系统概述

AzerothCore 拍卖行系统由以下核心文件组成：

| 文件                           | 职责                             | 行数     |
| ---------------------------- | ------------------------------ | ------ |
| `AuctionHouseMgr.cpp`        | 核心管理器：物品缓存、邮件通知、过期处理、押金计算      | \~643  |
| `AuctionHouseHandler.cpp`    | 网络层：上架、竞拍、一口价、取消、搜索的 Opcode 处理 | \~790  |
| `AuctionHouseSearcher.h/cpp` | 多线程搜索引擎：独立线程处理搜索请求             | \~300+ |

系统实现的完整功能：

- 拍卖物品的加载与存储
- 拍卖邮件通知系统（竞拍成功、过期、出价被超、取消等）
- 三大阵营拍卖行（联盟/部落/中立）的管理
- 拍卖过期自动处理
- 押金与手续费计算
- 多线程搜索子系统

***

## 2. 架构设计

### 2.1 单例模式

```
AuctionHouseMgr 采用 Meyers 单例（线程安全的局部静态变量）:
  static AuctionHouseMgr instance;
  return &instance;

通过宏 #define sAuctionMgr AuctionHouseMgr::instance() 提供全局访问。
```

### 2.2 核心数据结构

| 成员                      | 类型                                              | 用途             |
| ----------------------- | ----------------------------------------------- | -------------- |
| `_hordeAuctions`        | `AuctionHouseObject`                            | 部落拍卖行实例        |
| `_allianceAuctions`     | `AuctionHouseObject`                            | 联盟拍卖行实例        |
| `_neutralAuctions`      | `AuctionHouseObject`                            | 中立拍卖行实例        |
| `_mAitems`              | `ItemMap` (unordered\_map\<ObjectGuid, Item\*>) | 所有拍卖物品的内存缓存    |
| `_auctionHouseSearcher` | `AuctionHouseSearcher*`                         | 搜索引擎（多线程）      |
| `_updateIntervalTimer`  | `IntervalTimer`                                 | 定时更新计时器（1分钟间隔） |

### 2.3 阵营映射逻辑

拍卖行的阵营分配通过 `GetAuctionsMap()` 和 `GetAuctionsMapByHouseId()` 两个方法实现：

- 若 `CONFIG_ALLOW_TWO_SIDE_INTERACTION_AUCTION` 为 true，所有拍卖统一使用中立拍卖行
- 否则根据 `FactionTemplateEntry` 的 `ourMask` 判断联盟/部落/中立
- `AuctionHouseId` 枚举: Alliance=2, Horde=6, Neutral=7

***

## 3. 核心功能详解

### 3.1 拍卖物品加载（LoadAuctionItems / LoadAuctions）

**加载流程：**

1. `LoadAuctionItems()` 先执行，从 `auctionhouse` + `item_instance` 表联查加载所有拍卖物品到 `_mAitems` 缓存
   - SQL: `SELECT ... FROM auctionhouse ah JOIN item_instance ii ON ah.itemguid = ii.guid`
2. `LoadAuctions()` 后执行，从 `auctionhouse` + `item_instance` 表联查加载拍卖条目，关联到对应阵营的 `AuctionHouseObject`
   - SQL: `SELECT ... FROM auctionhouse ah INNER JOIN item_instance ii ON ii.guid = ah.itemguid`

**关键细节：**

- 支持热重载（reload 时先清理旧数据）
- 无效物品模板会跳过并输出 ERROR 日志
- 加载失败的拍卖条目会被从数据库删除（`DeleteFromDB` → `DELETE FROM auctionhouse WHERE id=?`）
- 使用事务批量提交删除操作

### 3.2 拍卖邮件系统

这是文件中最复杂的部分，共 6 个邮件发送方法：

#### 3.2.1 SendAuctionWonMail（竞拍成功通知买家）

```
触发时机: 拍卖到期且有竞拍者
处理流程:
  1. 获取拍卖物品指针
  2. 查找竞拍者（在线/离线）
  3. 更新物品所有者为竞拍者（防止角色删除导致物品丢失）
     - UPDATE item_instance SET owner_guid = ? WHERE guid = ?
  4. 发送竞拍成功通知包（在线玩家）
  5. 更新成就统计
  6. 通过邮件发送物品
     - INSERT INTO mail (id, messageType, stationery, ...)
     - INSERT INTO mail_items (mail_id, item_guid, receiver)
  7. 若竞拍者不存在，直接删除物品
     - DELETE FROM item_instance WHERE guid = ?
```

#### 3.2.2 SendAuctionSalePendingMail（出售待处理通知卖家）

```
触发时机: 拍卖成交后，通知卖家款项将在延迟后到账
特殊处理:
  - 计算 deliveryDelay（邮件送达延迟）
  - 使用 PackedTime 编码预计到账时间
  - 邮件内容包含押金、手续费、净收入等详细信息
数据库操作:
  - INSERT INTO mail (id, messageType, stationery, ...) — 纯通知邮件，无附件
```

#### 3.2.3 SendAuctionSuccessfulMail（出售成功通知卖家）

```
触发时机: 拍卖到期且有竞拍者
利润计算: profit = bid + deposit - GetAuctionCut()
成就更新:
  - ACHIEVEMENT_CRITERIA_TYPE_GOLD_EARNED_BY_AUCTIONS
  - ACHIEVEMENT_CRITERIA_TYPE_HIGHEST_AUCTION_SOLD
大额交易日志:
  - 当竞拍价 >= 500 金币时，记录到 log_money 表
  - 包含买卖双方信息、IP地址、物品ID等
数据库操作:
  - INSERT INTO mail (id, messageType, stationery, ...) — 含金币附件
  - INSERT INTO log_money VALUES(...) — 竞拍价 >= 500G 时（直接拼接SQL，非预编译）
```

#### 3.2.4 SendAuctionExpiredMail（过期退回物品给卖家）

```
触发时机: 拍卖到期且无竞拍者
处理: 将物品通过邮件退回卖家
注意: 不退还押金（这是游戏设计意图）
数据库操作:
  - INSERT INTO mail (id, messageType, stationery, ...) — 含物品附件
  - INSERT INTO mail_items (mail_id, item_guid, receiver)
```

#### 3.2.5 SendAuctionOutbiddedMail（出价被超通知前竞拍者）

```
触发时机: 有人出更高价
处理:
  - 通知旧竞拍者
  - 退还旧竞拍者的出价金额
  - 计算超出金额: CalculateAuctionOutBid(bid) = max(bid * 5%, 1铜)
数据库操作:
  - INSERT INTO mail (id, messageType, stationery, ...) — 含退还金币
```

#### 3.2.6 SendAuctionCancelledToBidderMail（卖家取消通知竞拍者）

```
触发时机: 卖家取消拍卖
处理: 退还竞拍者的出价金额
数据库操作:
  - INSERT INTO mail (id, messageType, stationery, ...) — 含退还金币
```

### 3.3 邮件系统的 Hook 机制

每个邮件方法都通过 `sScriptMgr` 提供了 Before 钩子：

| 钩子                                                        | 用途     |
| --------------------------------------------------------- | ------ |
| `OnBeforeAuctionHouseMgrSendAuctionWonMail`               | 竞拍成功前  |
| `OnBeforeAuctionHouseMgrSendAuctionSalePendingMail`       | 出售待处理前 |
| `OnBeforeAuctionHouseMgrSendAuctionSuccessfulMail`        | 出售成功前  |
| `OnBeforeAuctionHouseMgrSendAuctionExpiredMail`           | 过期前    |
| `OnBeforeAuctionHouseMgrSendAuctionOutbiddedMail`         | 出价被超前  |
| `OnBeforeAuctionHouseMgrSendAuctionCancelledToBidderMail` | 取消前    |

钩子参数包含 `sendNotification`、`updateAchievementCriteria`、`sendMail` 等布尔标志，允许模块修改默认行为。

### 3.4 押金计算（GetAuctionDeposit）

```cpp
公式推导:
  MSV = 物品售价 (SellPrice)
  multiplier = depositPercent / 100 (从 DBC 读取)
  timeHr = 时间段数 (以 12 小时为单位)
  deposit = (multiplier * MSV * count / 3) * timeHr * 3 * RATE_AUCTION_DEPOSIT

简化: deposit = multiplier * MSV * count * timeHr * RATE_AUCTION_DEPOSIT

最低押金: 100 铜 * RATE_AUCTION_DEPOSIT
```

### 3.5 定时更新（Update）

```cpp
void AuctionHouseMgr::Update(uint32 const diff)
{
    _updateIntervalTimer.Update(diff);  // 每分钟触发一次
    if (_updateIntervalTimer.Passed())
    {
        sScriptMgr->OnBeforeAuctionHouseMgrUpdate();
        _hordeAuctions.Update();      // 处理部落拍卖过期
        _allianceAuctions.Update();   // 处理联盟拍卖过期
        _neutralAuctions.Update();    // 处理中立拍卖过期
        _updateIntervalTimer.Reset();
    }
    _auctionHouseSearcher->Update();  // 处理搜索结果
}
```

### 3.6 AuctionHouseObject::Update()（拍卖过期处理）

```cpp
核心逻辑:
  checkTime = 当前游戏时间 + 60秒
  遍历所有拍卖:
    if (expire_time > checkTime) → 跳过
    if (无竞拍者) → SendAuctionExpiredMail（退回物品）
    else → SendAuctionSuccessfulMail + SendAuctionWonMail（完成交易）
    删除拍卖记录 + 物品缓存
```

**注意事项：**

- 使用 `checkTime = now + 60` 作为提前量，避免边界条件
- 先发卖家邮件再发买家邮件（保证卖家先收到通知）
- 使用数据库事务批量处理，减少 I/O

***

## 4. AuctionEntry 结构分析

### 4.1 字段说明

| 字段                  | 类型                        | 说明              |
| ------------------- | ------------------------- | --------------- |
| `Id`                | uint32                    | 拍卖唯一 ID         |
| `houseId`           | AuctionHouseId            | 所属拍卖行（联盟/部落/中立） |
| `item_guid`         | ObjectGuid                | 拍卖物品的 GUID      |
| `item_template`     | uint32                    | 物品模板 ID         |
| `itemCount`         | uint32                    | 物品数量（堆叠）        |
| `owner`             | ObjectGuid                | 卖家角色 GUID       |
| `startbid`          | uint32                    | 起拍价             |
| `bid`               | uint32                    | 当前最高出价          |
| `buyout`            | uint32                    | 一口价             |
| `expire_time`       | time\_t                   | 过期时间戳           |
| `bidder`            | ObjectGuid                | 当前最高出价者 GUID    |
| `deposit`           | uint32                    | 押金（创建时计算）       |
| `auctionHouseEntry` | AuctionHouseEntry const\* | DBC 数据指针        |

### 4.2 数据库操作

| 操作                   | 预编译语句                    | SQL                                                                                                                                                         | 涉及表                              |
| -------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **SaveToDB**         | `CHAR_INS_AUCTION`       | `INSERT INTO auctionhouse (id, houseid, itemguid, itemowner, buyoutprice, time, buyguid, lastbid, startbid, deposit) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)` | `auctionhouse`                   |
| **DeleteFromDB**     | `CHAR_DEL_AUCTION`       | `DELETE FROM auctionhouse WHERE id = ?`                                                                                                                     | `auctionhouse`                   |
| **UpdateBid**        | `CHAR_UPD_AUCTION_BID`   | `UPDATE auctionhouse SET buyguid = ?, lastbid = ? WHERE id = ?`                                                                                             | `auctionhouse`                   |
| **LoadAuctionItems** | `CHAR_SEL_AUCTION_ITEMS` | `SELECT ... FROM auctionhouse ah JOIN item_instance ii ON ah.itemguid = ii.guid`                                                                            | `auctionhouse` + `item_instance` |
| **LoadAuctions**     | `CHAR_SEL_AUCTIONS`      | `SELECT ... FROM auctionhouse ah INNER JOIN item_instance ii ON ii.guid = ah.itemguid`                                                                      | `auctionhouse` + `item_instance` |

### 4.3 邮件构建

**邮件主题格式:** `{item_template}:0:{response}:{auction_id}:{item_count}`

**邮件正文格式:** `{owner_guid_hex}:{bid}:{buyout}:{deposit}:{cut}:{moneyDelay}:{eta}`

***

## 5. 搜索子系统集成

`AuctionHouseSearcher` 是独立的多线程搜索引擎，与 `AuctionHouseMgr` 的交互点：

| 操作   | 触发时机                                | 方法                                 |
| ---- | ----------------------------------- | ---------------------------------- |
| 添加索引 | `AuctionHouseObject::AddAuction`    | `Searcher->AddAuction(auction)`    |
| 移除索引 | `AuctionHouseObject::RemoveAuction` | `Searcher->RemoveAuction(auction)` |
| 结果消费 | `AuctionHouseMgr::Update`           | `Searcher->Update()`               |

搜索子系统使用生产者-消费者队列（`ProducerConsumerQueue` / `MPSCQueue`）实现线程安全的请求/响应传递。

***

## 6. 脚本钩子（ScriptMgr）汇总

| 钩子                                        | 触发时机                       |
| ----------------------------------------- | -------------------------- |
| `OnBeforeAuctionHouseMgrUpdate`           | 每分钟 Update 循环开始前           |
| `OnAuctionAdd`                            | 拍卖添加到 AuctionHouseObject 时 |
| `OnAuctionRemove`                         | 拍卖从 AuctionHouseObject 移除时 |
| `OnAuctionExpire`                         | 拍卖过期（无竞拍者）                 |
| `OnAuctionSuccessful`                     | 拍卖成交                       |
| `OnBeforeAuctionHouseMgrSendAuction*Mail` | 各类邮件发送前（6个）                |

***

## 7. 关键常量与配置

| 常量/配置                        | 值      | 说明             |
| ---------------------------- | ------ | -------------- |
| `AH_MINIMUM_DEPOSIT`         | 100 铜  | 最低押金           |
| `MIN_AUCTION_TIME`           | 12 小时  | 最短拍卖时间         |
| `MAX_AUCTION_ITEMS`          | 160    | 单角色最大拍卖数       |
| `MAX_AUCTIONS_PER_PAGE`      | 50     | 每页显示拍卖数        |
| `AUCTION_SEARCH_DELAY`       | 300 ms | 搜索防抖延迟         |
| `MAX_GETALL_RETURN`          | 55000  | getAll 模式最大返回数 |
| `RATE_AUCTION_DEPOSIT`       | 服务器配置  | 押金倍率           |
| `RATE_AUCTION_CUT`           | 服务器配置  | 手续费倍率          |
| `CONFIG_MAIL_DELIVERY_DELAY` | 服务器配置  | 邮件送达延迟         |

***

## 8. 代码质量分析

### 8.1 优点

- **事务安全**: 所有数据库操作使用 `CharacterDatabaseTransaction`，保证原子性
- **Hook 完善**: 每个关键操作都有 Before 钩子，便于模块扩展
- **错误处理**: 物品/角色不存在时有合理的降级处理
- **日志完善**: 使用分级日志系统（LOG\_DEBUG/INFO/WARN/ERROR）
- **缓存策略**: `_mAitems` 提供 O(1) 的物品查找

### 8.2 潜在问题

1. **内存管理**: `_mAitems` 使用裸指针 + 手动 delete，析构函数中有明确的清理循环。但 `RemoveAItem` 的 `deleteFromDB` 参数需要外部传入事务指针，调用不当可能泄漏。
2. **大额交易日志 SQL 注入风险** (第 226-228 行):
   ```cpp
   CharacterDatabase.Execute("INSERT INTO log_money VALUES({}, {}, \"{}\", ...)",
       gpd->AccountId, ..., gpd->Name, ...);
   ```
   虽然使用了 `Execute` 的格式化，但 `gpd->Name` 来自数据库，理论上已可信。若需要更安全，应使用预编译语句。
3. **格式化字符串拼接** (第 226-228 行):
   使用 `Execute` 直接拼接 SQL 而非预编译语句，不符合项目其他地方的最佳实践。
4. **\_next 迭代器未使用**:
   `AuctionHouseObject` 的 `_next` 成员在 `Update()` 中未被使用（直接遍历整个 map），可能是历史遗留的优化意图未完成。
5. **Update() 中的删除模式**:
   ```cpp
   for (AuctionEntryMap::iterator itr, iter = _auctionsMap.begin(); iter != _auctionsMap.end(); )
   {
       itr = iter++;
       // ... 可能删除 itr 指向的元素
   }
   ```
   使用 `iter++` 先推进再操作的经典模式，正确处理了迭代器失效。

***

## 9. 调用关系图

```
World::Update()
  └─ AuctionHouseMgr::Update()
       ├─ AuctionHouseObject::Update() × 3 (horde/alliance/neutral)
       │    ├─ SendAuctionExpiredMail()        [无竞拍者]
       │    ├─ SendAuctionSuccessfulMail()      [有竞拍者]
       │    ├─ SendAuctionWonMail()             [有竞拍者]
       │    ├─ AuctionEntry::DeleteFromDB()
       │    ├─ RemoveAItem()
       │    └─ RemoveAuction()
       └─ AuctionHouseSearcher::Update()

服务器启动
  └─ LoadAuctionItems()    → 加载物品到 _mAitems
  └─ LoadAuctions()        → 加载拍卖到 AuctionHouseObject

玩家操作（在 AuctionHouseHandler.cpp 中）
  ├─ PlaceBid
  │    ├─ SendAuctionOutbiddedMail()     [通知前竞拍者]
  │    └─ Searcher->UpdateBid()
  ├─ BuyAuction
  │    ├─ SendAuctionSuccessfulMail()
  │    ├─ SendAuctionWonMail()
  │    └─ Searcher->RemoveAuction()
  ├─ SellItem
  │    ├─ AddAItem()
  │    ├─ AuctionHouseObject::AddAuction()
  │    │    └─ Searcher->AddAuction()
  │    └─ AuctionEntry::SaveToDB()
  └─ CancelAuction
       ├─ SendAuctionCancelledToBidderMail()
       ├─ SendAuctionExpiredMail()
       └─ RemoveAuction()
```

***

## 10. 线程安全考量

- **主线程**: `AuctionHouseMgr::Update()`、所有邮件方法、`LoadAuctions/LoadAuctionItems` 均在主线程执行
- **搜索线程**: `AuctionHouseWorkerThread` 在独立线程运行，通过队列与主线程通信
- **线程边界**: `AddAuction/RemoveAuction` 在主线程调用，通过 `NotifyAllWorkers` 将更新推送到搜索线程的 `_auctionUpdatesQueue`
- **潜在风险**: `_mAitems` 的读写均在主线程，搜索线程通过 `SearchableAuctionEntry` 持有物品数据的副本，避免了竞态

***

## 11. 总结

`AuctionHouseMgr.cpp` 是一个成熟、功能完整的拍卖行管理模块，约 643 行代码。它采用经典的单例管理器模式，通过邮件系统实现异步通知，通过 Hook 系统支持模块扩展。主要设计决策包括：

1. **阵营隔离**: 三套独立的 `AuctionHouseObject`，支持跨阵营交互配置
2. **异步邮件**: 所有通知通过游戏邮件系统，支持离线玩家
3. **事务批处理**: 过期处理使用数据库事务，减少 I/O 开销
4. **搜索解耦**: 搜索引擎独立线程运行，通过队列通信

代码质量整体良好，遵循 AzerothCore 的编码规范，适合作为学习游戏服务器拍卖行系统实现的参考。

***

## 12. AuctionHouseHandler.cpp 网络包处理分析

> 分析文件: `src/server/game/Handlers/AuctionHouseHandler.cpp` (\~790 行)

### 12.1 Opcode 处理函数清单

| 函数                              | Opcode                            | 功能                 |
| ------------------------------- | --------------------------------- | ------------------ |
| `HandleAuctionHelloOpcode`      | `CMSG_AUCTION_HELLO`              | 与拍卖师 NPC 对话，打开拍卖窗口 |
| `HandleAuctionSellItem`         | `CMSG_AUCTION_SELL_ITEM`          | 上架物品               |
| `HandleAuctionPlaceBid`         | `CMSG_AUCTION_PLACE_BID`          | 竞拍或一口价购买           |
| `HandleAuctionRemoveItem`       | `CMSG_AUCTION_REMOVE_ITEM`        | 取消拍卖               |
| `HandleAuctionListBidderItems`  | `CMSG_AUCTION_LIST_BIDDER_ITEMS`  | 查看已竞拍物品列表          |
| `HandleAuctionListOwnerItems`   | `CMSG_AUCTION_LIST_OWNER_ITEMS`   | 查看已上架物品列表          |
| `HandleAuctionListItems`        | `CMSG_AUCTION_LIST_ITEMS`         | 搜索拍卖物品             |
| `HandleAuctionListPendingSales` | `CMSG_AUCTION_LIST_PENDING_SALES` | 查看待处理销售（未实现，返回空）   |

### 12.2 发送辅助函数

| 函数                              | Opcode                             | 功能               |
| ------------------------------- | ---------------------------------- | ---------------- |
| `SendAuctionHello`              | `MSG_AUCTION_HELLO`                | 打开拍卖窗口响应（含拍卖行ID） |
| `SendAuctionCommandResult`      | `SMSG_AUCTION_COMMAND_RESULT`      | 上架/竞拍/取消操作结果反馈   |
| `SendAuctionBidderNotification` | `SMSG_AUCTION_BIDDER_NOTIFICATION` | 通知竞拍者被超价         |
| `SendAuctionOwnerNotification`  | `SMSG_AUCTION_OWNER_NOTIFICATION`  | 通知卖家已售出          |

### 12.3 上架流程 (HandleAuctionSellItem) - 详细分析

**协议解析:**

```
客户端发送: auctioneer_guid + itemsCount + [item_guid(16B) + count(4B)] * N + bid(4B) + buyout(4B) + etime(4B)
```

**校验层次（从外到内）:**

1. **数量校验**: `itemsCount <= MAX_AUCTION_ITEMS(160)`
2. **单物品校验**: GUID 非零, count > 0 且 <= 1000
3. **价格校验**: bid 和 etime 非零, 不超过 `MAX_MONEY_AMOUNT`
4. **NPC 校验**: `GetNPCIfCanInteractWith(auctioneer, UNIT_NPC_FLAG_AUCTIONEER)`
5. **拍卖行校验**: 从 NPC 派系获取 `AuctionHouseEntry`
6. **时间校验**: 只允许 12h / 24h / 48h 三种时长
7. **物品存在性**: 每个 itemGUID 在背包中存在
8. **物品合法性**:
   - 不在拍卖缓存中（未重复上架）
   - `CanBeTraded()` 为 true
   - 非空背包 (`IsNotEmptyBag`)
   - 非创建物品 (`ITEM_FLAG_CONJURED`)
   - 无持续时间限制 (`ITEM_FIELD_DURATION == 0`)
   - 数量足够 (`item->GetCount() >= count[i]`)
   - 所有物品为同一模板 (`itemEntry` 一致)
9. **防重复 GUID**: O(n^2) 双重循环检测相同 GUID（防作弊）
10. **堆叠上限**: `item->GetMaxStackCount() >= finalCount`

**上架两种模式:**

**模式 A - 整堆上架** (`itemsCount == 1 && item->GetCount() == count[i]`):

```
1. 直接移动原物品到拍卖缓存 (AddAItem)
2. 创建 AuctionEntry 并添加到拍卖行 (AddAuction → Searcher->AddAuction)
3. 从玩家背包移除 (MoveItemFromInventory)
4. 事务: DeleteFromInventoryDB + SaveToDB + SaveAuction + SaveInventoryAndGold
   - DELETE FROM character_inventory WHERE item = ?     (从背包移除)
   - INSERT INTO auctionhouse (...) VALUES (...)         (保存拍卖)
   - UPDATE characters SET ... WHERE guid = ?            (保存金币)
5. 返回成功 + 更新成就
```

**模式 B - 拆分/合并上架**:

```
1. CloneItem 创建新物品（finalCount 数量）
2. 添加新物品到拍卖缓存和拍卖行
3. 遍历原物品:
   - 若堆叠数 == 需要数: 从背包移除 + 删除
   - 若堆叠数 > 需要数: 减少数量 + 标记 ITEM_CHANGED
4. 事务: SaveNewItem + SaveAuction + SaveInventoryAndGold
   - REPLACE INTO item_instance (...) VALUES (...)       (保存新物品/更新物品)
   - DELETE FROM character_inventory WHERE item = ?      (从背包移除)
   - INSERT INTO auctionhouse (...) VALUES (...)          (保存拍卖)
   - UPDATE characters SET ... WHERE guid = ?             (保存金币)
5. 返回成功 + 更新成就
```

### 12.4 竞拍/一口价流程 (HandleAuctionPlaceBid) - 详细分析

**协议解析:**

```
客户端发送: auctioneer_guid(16B) + auctionId(4B) + price(4B)
```

**校验层次:**

1. auctionId 和 price 非零
2. NPC 可交互
3. 脚本钩子 `OnPlayerCanPlaceAuctionBid` 检查
4. 拍卖存在且不是自己的
5. 同账号其他角色检测（防小号自拍自买）
6. 价格校验: `price > bid` 且 `price >= startbid`
7. 最小加价: 若非一口价，`price >= bid + CalculateAuctionOutBid(bid)` (即 bid + bid\*5%)
8. 金币充足

**分支 A - 普通竞拍** (`price < buyout || buyout == 0`):

```
三种扣款情况:
  ├─ 无竞拍者 → 扣除全款 price
  ├─ 竞拍者是自己（加价）→ 只扣差额 (price - auction->bid)
  └─ 竞拍者是他人 → SendAuctionOutbiddedMail + 扣全款

更新: auction->bidder = player, auction->bid = price
通知: Searcher->UpdateBid
数据库: CHAR_UPD_AUCTION_BID (UPDATE auctionhouse SET buyguid = ?, lastbid = ? WHERE id = ?)
成就: ACHIEVEMENT_CRITERIA_TYPE_HIGHEST_AUCTION_BID
```

**分支 B - 一口价购买** (`price >= buyout && buyout > 0`):

```
三种扣款情况:
  ├─ 竞拍者是自己 → 扣差额 (buyout - bid)
  ├─ 有其他竞拍者 → 扣全款 buyout + SendAuctionOutbiddedMail
  └─ 无竞拍者 → 扣全款 buyout

立即完成交易:
  1. SendAuctionSalePendingMail (通知卖家待处理)
     - INSERT INTO mail (...)
  2. SendAuctionSuccessfulMail (通知卖家已售出 + 金币)
     - INSERT INTO mail (...) — 含金币
     - INSERT INTO log_money VALUES(...) — 竞拍价 >= 500G 时
  3. SendAuctionWonMail (发送物品给买家)
     - UPDATE item_instance SET owner_guid = ? WHERE guid = ?
     - INSERT INTO mail (...) — 含物品
     - INSERT INTO mail_items (mail_id, item_guid, receiver)
  4. OnAuctionSuccessful 钩子
  5. auction->DeleteFromDB
     - DELETE FROM auctionhouse WHERE id = ?
  6. RemoveAItem + RemoveAuction

关键: 一口价时，三封邮件 + 删除操作全部在同一事务中提交
```

### 12.5 取消拍卖流程 (HandleAuctionRemoveItem) - 详细分析

```
1. NPC 交互校验
2. 获取拍卖条目
3. 验证玩家是 owner

4. 若有竞拍者:
   ├─ 计算手续费: auctionCut = GetAuctionCut() (bid * cutPercent%)
   ├─ 检查卖家有足够金币付手续费
   ├─ SendAuctionCancelledToBidderMail (退还竞拍者资金)
   └─ 扣除卖家手续费

5. 退回物品: MailDraft(AUCTION_CANCELED).AddItem(pItem).SendMailTo(卖家)

6. 清理:
   ├─ SaveInventoryAndGoldToDB
   ├─ auction->DeleteFromDB
   ├─ RemoveAItem
   └─ RemoveAuction

注意: 取消时押金不退还（与过期相同），这是游戏设计
```

### 12.6 搜索流程 (HandleAuctionListItems) - 详细分析

```
协议解析:
  guid + listfrom + searchedname + levelmin + levelmax +
  auctionSlotID + auctionMainCategory + auctionSubCategory +
  quality + usable + getAll + sortOrderCount + [sortMode + isDesc] * N

处理:
  1. 搜索名 UTF-8 → wstring → 转小写
  2. 确定拍卖行阵营
  3. 构建 AuctionHouseSearchInfo
  4. 若 usable=1:
     ├─ 收集 classMask, raceMask, level
     ├─ 遍历 SkillStatusMap 获取技能
     └─ 遍历 PlayerSpellMap 获取当前天赋法术
  5. 提交到 AuctionHouseSearcher 异步处理

搜索特性:
  - 支持 getAll 模式 (返回最多 55000 条, MAX_GETALL_RETURN)
  - 支持 11 种排序方式 (AUCTION_SORT_*)
  - 支持按等级、品质、装备栏位、物品类型过滤
  - 分页支持 (listfrom 参数, 每页 50 条)
```

***

## 13. 完整拍卖生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    拍卖行完整生命周期                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [创建] HandleAuctionSellItem                                   │
│    ├─ 校验物品 + 扣押金                                          │
│    ├─ 创建 AuctionEntry                                         │
│    ├─ sAuctionMgr->AddAItem(item)                               │
│    ├─ auctionHouse->AddAuction(AH)                              │
│    │    └─ Searcher->AddAuction() [索引]                         │
│    ├─ 从玩家背包移除物品                                          │
│    └─ 事务: DeleteInventory + SaveItem + SaveAuction             │
│         ├─ DELETE FROM character_inventory WHERE item = ?        │
│         ├─ REPLACE INTO item_instance (...) VALUES (...)         │
│         ├─ INSERT INTO auctionhouse (...) VALUES (...)           │
│         └─ UPDATE characters SET ... WHERE guid = ?              │
│                                                                 │
│  [竞拍] HandleAuctionPlaceBid (price < buyout)                  │
│    ├─ 价格校验 + 扣款                                            │
│    ├─ 退还前竞拍者 (SendAuctionOutbiddedMail)                    │
│    │    └─ INSERT INTO mail (...) — 含退还金币                    │
│    ├─ 更新 auction->bid / bidder                                │
│    ├─ Searcher->UpdateBid()                                     │
│    └─ 数据库: CHAR_UPD_AUCTION_BID                              │
│         └─ UPDATE auctionhouse SET buyguid=?, lastbid=? WHERE id=? │
│                                                                 │
│  [一口价] HandleAuctionPlaceBid (price >= buyout)               │
│    ├─ 扣款 + 退还前竞拍者                                        │
│    ├─ SendAuctionSalePendingMail → 卖家                          │
│    │    └─ INSERT INTO mail (...)                                │
│    ├─ SendAuctionSuccessfulMail → 卖家 (含金币)                   │
│    │    ├─ INSERT INTO mail (...) — 含金币                        │
│    │    └─ INSERT INTO log_money VALUES(...) — >= 500G            │
│    ├─ SendAuctionWonMail → 买家 (含物品)                          │
│    │    ├─ UPDATE item_instance SET owner_guid=? WHERE guid=?     │
│    │    ├─ INSERT INTO mail (...) — 含物品                        │
│    │    └─ INSERT INTO mail_items (mail_id, item_guid, receiver)  │
│    ├─ auction->DeleteFromDB()                                   │
│    │    └─ DELETE FROM auctionhouse WHERE id = ?                  │
│    ├─ RemoveAItem() + RemoveAuction()                           │
│    └─ OnAuctionSuccessful 钩子                                  │
│                                                                 │
│  [取消] HandleAuctionRemoveItem                                 │
│    ├─ 若有竞拍者: 退还资金 + 扣卖家手续费                          │
│    │    ├─ INSERT INTO mail (...) — 退还竞拍者金币                 │
│    │    └─ INSERT INTO mail_items (mail_id, item_guid, receiver)  │
│    ├─ 邮件退回物品给卖家 (AUCTION_CANCELED)                       │
│    │    ├─ INSERT INTO mail (...) — 含物品                        │
│    │    └─ INSERT INTO mail_items (mail_id, item_guid, receiver)  │
│    ├─ auction->DeleteFromDB()                                   │
│    │    └─ DELETE FROM auctionhouse WHERE id = ?                  │
│    └─ RemoveAItem() + RemoveAuction()                           │
│                                                                 │
│  [过期] AuctionHouseObject::Update (每分钟定时)                   │
│    ├─ 无竞拍者 → SendAuctionExpiredMail (退回物品)                 │
│    │    ├─ INSERT INTO mail (...) — 含物品                        │
│    │    └─ INSERT INTO mail_items (mail_id, item_guid, receiver)  │
│    ├─ 有竞拍者 → SendAuctionSuccessfulMail + SendAuctionWonMail  │
│    │    ├─ INSERT INTO mail (...) — 卖家金币                      │
│    │    ├─ INSERT INTO log_money VALUES(...) — >= 500G            │
│    │    ├─ UPDATE item_instance SET owner_guid=? WHERE guid=?     │
│    │    ├─ INSERT INTO mail (...) — 买家物品                      │
│    │    └─ INSERT INTO mail_items (mail_id, item_guid, receiver)  │
│    ├─ auction->DeleteFromDB()                                   │
│    │    └─ DELETE FROM auctionhouse WHERE id = ?                  │
│    └─ RemoveAItem() + RemoveAuction()                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

***

## 14. 邮件系统完整流程

```
邮件类型               触发条件              接收者    内容
─────────────────────────────────────────────────────────────────
AUCTION_WON           拍卖到期/一口价       买家      物品
AUCTION_SUCCESSFUL    拍卖到期/一口价       卖家      金币 (bid + deposit - cut)
AUCTION_SALE_PENDING  一口价立即            卖家      通知 (含预计到账时间)
AUCTION_EXPIRED       拍卖到期无竞拍        卖家      物品 (押金不退)
AUCTION_OUTBIDDED     有更高出价            前竞拍者  退还的金币
AUCTION_CANCELLED_TO_BIDDER  卖家取消       竞拍者    退还的金币
AUCTION_CANCELED      卖家取消              卖家      物品 (扣手续费)
```

***

## 15. 脚本钩子完整列表

| 钩子                                                        | 触发位置                  | 用途           |
| --------------------------------------------------------- | --------------------- | ------------ |
| `CanSendAuctionHello`                                     | SendAuctionHello      | 控制是否允许打开拍卖窗口 |
| `OnPlayerCanPlaceAuctionBid`                              | HandleAuctionPlaceBid | 控制是否允许竞拍     |
| `OnBeforeAuctionHouseMgrUpdate`                           | Update (每分钟)          | 定时更新前        |
| `OnAuctionAdd`                                            | AddAuction            | 拍卖添加后        |
| `OnAuctionRemove`                                         | RemoveAuction         | 拍卖移除后        |
| `OnAuctionExpire`                                         | Update (过期)           | 拍卖过期（无竞拍）    |
| `OnAuctionSuccessful`                                     | Update/一口价            | 拍卖成交         |
| `OnBeforeAuctionHouseMgrSendAuctionWonMail`               | 邮件发送前                 | 控制竞拍成功邮件     |
| `OnBeforeAuctionHouseMgrSendAuctionSalePendingMail`       | 邮件发送前                 | 控制待处理邮件      |
| `OnBeforeAuctionHouseMgrSendAuctionSuccessfulMail`        | 邮件发送前                 | 控制售出邮件       |
| `OnBeforeAuctionHouseMgrSendAuctionExpiredMail`           | 邮件发送前                 | 控制过期邮件       |
| `OnBeforeAuctionHouseMgrSendAuctionOutbiddedMail`         | 邮件发送前                 | 控制出价被超邮件     |
| `OnBeforeAuctionHouseMgrSendAuctionCancelledToBidderMail` | 邮件发送前                 | 控制取消邮件       |

***

## 16. 数据流向图

```
客户端 (WoW 3.3.5a)
  │
  │ CMSG_AUCTION_* (网络包)
  ▼
WorldSession::HandleAuction*Opcode
  │
  ├─ 校验 NPC 交互 + 脚本钩子
  │
  ├─ 上架/竞拍/取消 ──────────────────────────────┐
  │                                                │
  ▼                                                ▼
AuctionHouseMgr                              AuctionHouseSearcher
  ├─ _mAitems (物品缓存)                      ├─ WorkerThread (独立线程)
  ├─ _hordeAuctions                           ├─ SearchableAuctionEntry
  ├─ _allianceAuctions                        ├─ 请求队列 (PCQueue)
  └─ _neutralAuctions                         └─ 响应队列 (MPSCQueue)
       │                                                │
       │ AuctionEntry                                    │
       ├─ SaveToDB()                                     │
       ├─ DeleteFromDB()                                 │
       └─ LoadFromDB()                                   │
              │                                          │
              ▼                                          ▼
       CharacterDatabase                          WorldSession
       ┌─ auctionhouse (拍卖条目)                  (SMSG_AUCTION_* 响应包)
       ├─ item_instance (物品实例)
       ├─ character_inventory (角色背包)
       ├─ characters (角色金币)
       ├─ mail (邮件)
       ├─ mail_items (邮件物品)
       └─ log_money (大额交易日志)
```

***

## 17. Handler 层代码质量分析

### 17.1 优点

- **多层防御**: 输入校验从协议层→NPC层→物品层→经济层逐级过滤
- **事务一致**: 口价购买时，三封邮件 + 删除操作在同一事务中提交，防止数据丢失
- **防作弊**: 重复 GUID 检测、同账号跨角色检测、价格合法性校验
- **成就集成**: 上架/竞拍/售出均有对应的成就更新

### 17.2 潜在问题

1. **HandleAuctionRemoveItem 事务泄漏** (第 550-555 行):
   ```cpp
   CharacterDatabaseTransaction trans = CharacterDatabase.BeginTransaction();
   if (auction && auction->owner == player->GetGUID())
   {
       Item* pItem = sAuctionMgr->GetAItem(auction->item_guid);
       if (!pItem)
           return;  // ← 事务 trans 未提交也未回滚，且已 BeginTransaction
   ```
   当 `pItem` 为 nullptr 时直接 return，事务对象泄漏。虽然后续析构可能自动回滚，但这不是规范做法。
2. **HandleAuctionSellItem 外层循环多余** (第 250 行):
   ```cpp
   for (uint32 i = 0; i < itemsCount; ++i)  // 外层循环
   {
       Item* item = items[i];
       // ... 每次循环都创建新的 auctionTime、deposit、AuctionEntry
       // 但根据前面校验，所有物品是同一模板，所以循环体内的结果相同
       // 实际上 itemsCount > 1 时只处理最后一件物品
   }
   ```
   当 `itemsCount > 1` 时，外层循环会为每个物品创建 AuctionEntry 并扣除押金，但只有第一个物品的拍卖会被正确返回（return 在循环内）。这可能导致多余的押金扣除和内存泄漏。
3. **HandleAuctionSellItem 模式 B 中的重复事务** (第 354-376 行):
   遍历物品删除时，每个需要完全删除的物品都单独开启事务，效率不高。可以统一到一个事务中。
4. **HandleAuctionListPendingSales** 未实现:
   返回空数据，注释中有待完成的代码框架。
5. **搜索参数 loc\_idx / locdbc\_idx 的双重 locale**:
   `ahPlayerInfo.loc_idx` 和 `locdbc_idx` 分别是数据库 locale 和 DBC locale，区分使用确保多语言支持正确。

***

## 18. 总结

AzerothCore 拍卖行系统由 `AuctionHouseMgr.cpp`（核心管理器，\~643行）和 `AuctionHouseHandler.cpp`（网络处理，\~790行）两个文件组成，合计约 1430 行代码。

**架构特点:**

- 单例管理器 + 三阵营隔离的 AuctionHouseObject 容器
- 邮件系统实现异步通知，支持离线玩家
- 多线程搜索解耦，通过生产者-消费者队列通信
- 完善的 ScriptMgr 钩子系统支持模块扩展

**数据流:** 客户端 Opcode → Handler 校验 → Mgr 物品/拍卖管理 → Searcher 索引更新 → DB 事务持久化 → 邮件通知

**代码质量:** 整体良好，遵循 AzerothCore 编码规范。主要关注点是 Handler 层的事务管理和循环结构存在少量不规范之处。

***

## 19. 数据库表汇总

拍卖行系统涉及以下 CharacterDatabase 表：

| 表名                    | 用途       | 操作类型                               | 触发场景                    |
| --------------------- | -------- | ---------------------------------- | ----------------------- |
| `auctionhouse`        | 拍卖条目存储   | SELECT / INSERT / UPDATE / DELETE  | 加载、上架、竞拍、一口价、取消、过期      |
| `item_instance`       | 物品实例数据   | SELECT / REPLACE / UPDATE / DELETE | 加载、上架、竞拍成功（转移所有权）、过期退回  |
| `character_inventory` | 角色背包物品   | DELETE                             | 上架（从背包移除）               |
| `characters`          | 角色数据（金币） | UPDATE                             | 上架（扣押金）、竞拍（扣款）、取消（扣手续费） |
| `mail`                | 邮件记录     | INSERT                             | 所有邮件通知（6种类型）            |
| `mail_items`          | 邮件物品附件   | INSERT                             | 含物品的邮件（竞拍成功、过期退回、取消退回）  |
| `log_money`           | 大额交易日志   | INSERT                             | 竞拍价 >= 500G 的成交记录       |

### 表关系图

```
auctionhouse (拍卖条目)
  │
  ├─ itemguid ──→ item_instance (物品实例)
  │                  │
  │                  ├─ owner_guid ──→ characters (角色)
  │                  │
  │                  └─ guid ──→ mail_items (邮件物品附件)
  │                               │
  │                               └─ mail_id ──→ mail (邮件)
  │
  ├─ itemowner ──→ characters (卖家)
  │
  └─ buyguid ──→ characters (买家)

character_inventory (角色背包)
  │
  └─ item ──→ item_instance (物品实例)

log_money (大额交易日志)
  │
  └─ 独立记录，无外键关联
```

