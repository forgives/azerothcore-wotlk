# AzerothCore Game Server 子模块功能分析

## 目录概述

`src/server/game` 目录下包含了魔兽世界私服的核心游戏逻辑实现。以下是各子模块的核心功能总结：

---

## 数据库说明

AzerothCore 使用三个数据库：
- **db_auth** - 登录数据库，存储账号、服务器列表等
- **db_characters** - 角色数据库，存储角色数据、物品、公会等
- **db_world** - 世界数据库，存储游戏配置、模板、脚本等

---

## 核心功能模块列表

| 序号 | 模块目录 | 核心功能描述 |
|------|---------|------------|
| 1 | **AI** | 生物人工智能系统。包含多种AI类型：CoreAI(战斗/守卫/被动/宠物AI)、ScriptedAI(脚本生物AI，支持对话和护送)、SmartScripts(可视化脚本AI系统)。负责NPC和怪物的行为逻辑、攻击判定、目标选择等。 |
| 2 | **Accounts** | 账户管理模块。处理账号创建、删除、密码验证、用户名修改、邮箱修改、权限等级查询等账户相关操作。 |
| 3 | **Achievements** | 成就系统。管理玩家成就进度、成就条件判断、成就奖励发放、离线玩家成就更新等。 |
| 4 | **Addons** | 插件管理模块。记录已安装插件信息、验证插件CRC、加载/禁用插件、管理被禁用的插件(Malicious addons)。 |
| 5 | **ArenaSpectator** | 竞技场观战系统。实现旁观者模式，允许玩家观战竞技场战斗并通过插件接收实时战斗数据。 |
| 6 | **AuctionHouse** | 拍卖行系统。处理物品拍卖、竞拍、出售、搜索、拍卖时间管理、拍卖邮件通知等。 |
| 7 | **Autobroadcast** | 自动广播系统。定时向全服发送公告消息，支持不同广播类型(世界公告/屏幕通知)，支持权重配置和多语言本地化。 |
| 8 | **Battlefield** | 大型世界战场系统。实现Wintergrasp(冬拥湖)等世界级战场，包括占领点控制、载具系统、战场事件触发等。 |
| 9 | **Battlegrounds** | 战场系统。实现多种战场模式(战歌/阿拉希/风暴之眼/副本战场等)、竞技场(Arena)、战场排队管理、赛季奖励分发。 |
| 10 | **Cache** | 内存缓存系统。CharacterCache缓存玩家角色信息(名称/等级/阵营/公会等)；WhoListCacheMgr缓存玩家列表以优化在线查询。 |
| 11 | **Calendar** | 游戏内日历系统。管理公会事件、团队活动邀请、事件创建/编辑/删除、日历邮件通知等。 |
| 12 | **Chat** | 聊天系统。处理所有聊天消息(私聊/频道/大喊/工会)、超链接解析、聊天命令系统、消息过滤、GM消息等。 |
| 13 | **Combat** | 战斗管理系统。包含ThreatManager(仇恨管理)、战斗状态追踪、伤害计算辅助等。 |
| 14 | **Conditions** | 条件判定系统。提供多种条件检查(Aura/物品/声望/任务/区域/技能等)用于触发器、NPC对话、掉落等逻辑判断；DisableMgr管理禁用内容。 |
| 15 | **DataStores** | 数据存储系统。加载DBC格式的游戏数据(技能/物品/地图/生物模板等)、M2模型数据、地图坐标转换、内容难度分级等。 |
| 16 | **DungeonFinding** | 地下城查找器(LFG/LFR)。实现随机副本排队、角色检查、队列匹配、奖励分配、副本浏览器等功能。 |
| 17 | **Entities** | 实体系统。最核心的模块，包含：Player(玩家)、Creature(NPC/怪物)、GameObject(游戏对象)、Item(物品)、Pet(宠物)、Unit(单位基类)、Vehicle(载具)、Corpse(尸体)、Transport(交通)等所有游戏实体的定义和实现。 |
| 18 | **Events** | 游戏事件管理器。管理周期性游戏事件(节日活动/世界事件)、事件触发条件、事件持续时间、假期日期计算等。 |
| 19 | **Globals** | 全局对象管理。ObjectAccessor提供全局对象访问接口；ObjectMgr管理所有游戏对象模板(NPC/物品/任务等)的加载和存储。 |
| 20 | **Grids** | 网格系统。游戏世界空间分区管理，包含Cell/Grid/CellImpl、GridNotifiers(通知器)、GridTerrainData(地形数据)、MapGrid等，用于优化空间查询和对象加载。 |
| 21 | **Groups** | 队伍系统。处理队伍创建/解散、队长转让、队伍成员管理、团队分配、队伍副本等功能。 |
| 22 | **Guilds** | 公会系统。公会创建/解散、成员管理、公会银行、公会等级/经验、通告、工会副本记录等。 |
| 23 | **Handlers** | 网络消息处理器目录。处理客户端发来的各类请求：角色/聊天/PVP/物品/交易/邮件/移动/施法等数十种协议处理程序。 |
| 24 | **Instances** | 副本系统。InstanceSaveMgr管理副本进度保存；InstanceScript提供副本脚本接口，用于副本内特殊逻辑。 |
| 25 | **Loot** | 战利品系统。管理掉落表、掉落规则、战利品存储、交易/开箱/怪物掉落等。 |
| 26 | **Mails** | 邮件系统。处理邮件发送/接收、附件、邮件过期、服务器邮件(GM发送)、邮件退款等。 |
| 27 | **Maps** | 地图系统。Map管理每个地图实例的更新/对象加载；MapInstanced处理副本地图；TransportMgr管理交通工具(船只/飞艇)；ZoneScript提供区域脚本接口。 |
| 28 | **Misc** | 杂项功能。BanMgr(封禁管理)、DynamicVisibility(动态可见性)、GameGraveyard(墓地系统)等。 |
| 29 | **Miscellaneous** | 辅助工具。包含经验/升级公式(Formulas)、语言常量定义等。 |
| 30 | **Modules** | 模块管理系统。提供插件式模块加载机制，允许动态加载/卸载功能模块。 |
| 31 | **Motd** | 每日消息系统。管理服务器登录时显示的公告信息(Message of the Day)。 |
| 32 | **Movement** | 移动系统。包含各种移动生成器(巡逻/追击/逃跑/跟随/路径点等)、Spline曲线移动、路径管理(PathGenerator)、运动主控器(MotionMaster)。 |
| 33 | **OutdoorPvP** | 户外PVP系统。实现世界PvP区域(如世界PvP事件)，区别于副本战场。 |
| 34 | **Petitions** | 请愿系统。处理公会创建请愿(签名收集)、竞技场队伍创建请愿等。 |
| 35 | **Pools** | 对象池系统。管理游戏对象的动态生成/回收，用于优化资源使用(如刷新对象池)。 |
| 36 | **Quests** | 任务系统。任务定义、任务进度跟踪、任务奖励、任务目标判定、任务链管理等。 |
| 37 | **Reputation** | 声望系统。阵营声望计算、声望等级、声望奖励、声望对立关系等。 |
| 38 | **Scripting** | 脚本系统。提供脚本扩展接口，支持模块化脚本注册(Achievement/Account/Map等脚本)。 |

---

## 模块依赖关系简图

```
                    +-------------+
                    |  Handlers   | <-- 处理客户端请求
                    +------+------+
                           |
         +-----------------+------------------+
         |                 |                  |
         v                 v                  v
    +---------+      +---------+      +----------+
    |  Chat   |      | Combat  |      | Movement |
    +----+----+      +----+----+      +----+-----+
         |                |                 |
         +----------------+-----------------+
                          |
                          v
    +------------------------------------------------+
    |              Entities (核心)                   |
    |  Player / Creature / Unit / Item / GameObject |
    +---------------------+--------------------------+
                          |
    +---------------------+--------------------------+
    |                     |                          |
    v                     v                          v
 Maps/Instance      Battlegrounds            Quests/Reputation
    |                     |                          |
    +---------------------+--------------------------+
                          |
                          v
              +----------------------+
              |    AI / Scripting    |
              +----------------------+
```

---

## 模块详细说明

### 1. AI - 人工智能系统

**目录结构**：
- `CoreAI/` - 核心AI实现
- `ScriptedAI/` - 脚本化AI
- `SmartScripts/` - 智能脚本系统

**核心类**：
- `CreatureAI` - 生物AI基类，提供所有AI的公共接口
- `UnitAI` - 单位AI基类，CreatureAI的父类
- `AggressorAI` - 侵略型AI，主动攻击敌对目标
- `CombatAI` - 战斗AI，管理战斗中的技能使用
- `CasterAI` - 法师AI，管理施法行为
- `VehicleAI` - 载具AI，处理载具特殊逻辑
- `SmartAI` - 智能AI，通过数据库配置实现复杂行为

**主要功能**：
- **仇恨管理**：`UpdateVictim()`、`SetGazeOn()` 选择攻击目标
- **战斗事件**：`JustEngagedWith()`、`JustDied()`、`KilledUnit()` 等战斗回调
- **召唤管理**：`DoSummon()`、`JustSummoned()` 处理召唤物
- **法术响应**：`SpellHit()`、`SpellHitTarget()` 处理法术命中
- **路径点系统**：`WaypointStarted()`、`WaypointReached()` 处理巡逻路径
- **边界系统**：`CheckInRoom()`、`SetBoundary()` 限制活动范围
- **脱战机制**：`EnterEvadeMode()` 处理脱战逻辑

**SmartAI 特性**：
- 支持通过数据库配置实现复杂AI行为
- 支持护送任务(`SMART_ESCORT_ESCORTING`)
- 支持跟随、路径移动、距离控制
- 支持事件触发、法术施放、对话等

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `smart_scripts` | SmartAI脚本配置，定义AI行为触发条件和动作 |
| `waypoint_data` | 路径点数据，定义NPC巡逻路径坐标 |
| `waypoint_scripts` | 路径点脚本，定义到达路径点时执行的动作 |
| `script_waypoint` | 脚本路径点，用于脚本化AI的路径定义 |
| `creature_addon` | 生物附加数据，包含路径ID、挂载、光环等 |
| `creature_template_addon` | 生物模板附加数据，定义生物默认的AI相关属性 |
| `creature_formations` | 生物编队，定义NPC组队跟随关系 |
| `creature_summon_groups` | 生物召唤组，定义批量召唤的NPC |

---

### 2. Accounts - 账户管理

**核心命名空间**：`AccountMgr`

**主要功能**：
- **账号操作**：
  - `CreateAccount()` - 创建新账号
  - `DeleteAccount()` - 删除账号
  - `ChangeUsername()` - 修改用户名
  - `ChangePassword()` - 修改密码
  - `ChangeEmail()` - 修改邮箱

- **账号查询**：
  - `GetId()` - 根据用户名获取账号ID
  - `GetSecurity()` - 获取账号权限等级
  - `GetName()` - 根据ID获取用户名
  - `GetCharactersCount()` - 获取角色数量

- **权限检查**：
  - `IsPlayerAccount()` - 是否为普通玩家账号
  - `IsGMAccount()` - 是否为GM账号
  - `IsAdminAccount()` - 是否为管理员账号
  - `IsConsoleAccount()` - 是否为控制台账号

**返回枚举**：`AccountOpResult`
- `AOR_OK` - 操作成功
- `AOR_NAME_TOO_LONG` - 用户名过长
- `AOR_PASS_TOO_LONG` - 密码过长
- `AOR_NAME_ALREADY_EXIST` - 用户名已存在
- `AOR_NAME_NOT_EXIST` - 用户名不存在

**数据库表 (db_auth)**：
| 表名 | 作用 |
|------|------|
| `account` | 账号主表，存储用户名、密码哈希、邮箱、最后登录IP、权限等级等 |
| `account_access` | 账号权限表，存储账号在各服务器的GM权限等级 |
| `account_banned` | 账号封禁表，记录封禁状态和解封时间 |
| `account_muted` | 账号禁言表，记录禁言状态和解除时间 |
| `realmcharacters` | 服务器角色数量统计表 |
| `realmlist` | 服务器列表表，存储服务器名称、地址、类型等 |
| `ip_banned` | IP封禁表，记录被封禁的IP地址 |
| `uptime` | 服务器运行时间记录表 |
| `secret_digest` | 双因素认证密钥表 |

---

### 3. Achievements - 成就系统

**核心类**：
- `AchievementMgr` - 玩家成就管理器
- `AchievementGlobalMgr` - 全局成就管理器

**主要功能**：
- **成就进度追踪**：
  - `UpdateAchievementCriteria()` - 更新成就条件进度
  - `CompletedAchievement()` - 完成成就
  - `GetCriteriaProgress()` - 获取条件进度

- **成就条件类型**（部分）：
  - `ACHIEVEMENT_CRITERIA_DATA_TYPE_T_CREATURE` - 击杀特定生物
  - `ACHIEVEMENT_CRITERIA_DATA_TYPE_T_PLAYER_CLASS_RACE` - 特定职业/种族
  - `ACHIEVEMENT_CRITERIA_DATA_TYPE_S_AURA` - 获得特定光环
  - `ACHIEVEMENT_CRITERIA_DATA_TYPE_VALUE` - 数值条件
  - `ACHIEVEMENT_CRITERIA_DATA_TYPE_MAP_DIFFICULTY` - 特定副本难度

- **离线玩家支持**：
  - `CompletedAchievementForOfflinePlayer()` - 为离线玩家完成成就
  - `UpdateAchievementCriteriaForOfflinePlayer()` - 更新离线玩家成就进度

- **奖励系统**：
  - `AchievementReward` - 成就奖励结构(称号/物品/邮件)
  - `GetAchievementReward()` - 获取成就奖励

- **数据存储**：
  - `CriteriaProgressMap` - 条件进度映射
  - `CompletedAchievementMap` - 已完成成就映射

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `achievement_dbc` | 成就定义表，存储成就ID、名称、描述、点数等 |
| db_world | `achievement_criteria_data` | 成就条件数据表，定义成就完成条件 |
| db_world | `achievement_reward` | 成就奖励表，定义成就完成后的奖励(称号/物品) |
| db_world | `achievement_reward_locale` | 成就奖励本地化表 |
| db_world | `achievement_category_dbc` | 成就分类表 |
| db_world | `achievement_criteria_dbc` | 成就条件定义表 |
| db_characters | `character_achievement` | 角色成就表，记录玩家已完成的成就 |
| db_characters | `character_achievement_progress` | 角色成就进度表，记录成就条件进度 |
| db_characters | `character_achievement_offline_updates` | 离线成就更新队列表 |

---

### 4. Addons - 插件管理

**核心数据结构**：
- `AddonInfo` - 插件信息(名称/启用状态/CRC/状态)
- `SavedAddon` - 已保存的插件信息
- `BannedAddon` - 被封禁的插件信息(MD5/时间戳)

**主要功能**：
- `LoadFromDB()` - 从数据库加载插件信息
- `SaveAddon()` - 保存插件信息到数据库
- `GetAddonInfo()` - 获取插件信息
- `GetBannedAddons()` - 获取被封禁的插件列表

**安全机制**：
- CRC校验验证插件完整性
- MD5哈希识别恶意插件
- 支持封禁特定版本的插件

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `addons` | 插件信息表，存储玩家使用的插件名称、CRC校验值等 |
| `banned_addons` | 被封禁插件表，存储被禁用的插件MD5哈希和封禁时间 |

---

### 5. ArenaSpectator - 竞技场观战

**核心功能**：
- **观战命令**：
  - `HandleSpectatorSpectateCommand()` - 处理观战命令
  - `HandleSpectatorWatchCommand()` - 处理观看命令
  - `HandleResetCommand()` - 重置观战状态

- **数据传输**：
  - `SendCommand_String()` - 发送字符串数据
  - `SendCommand_UInt32Value()` - 发送数值数据
  - `SendCommand_Spell()` - 发送法术信息
  - `SendCommand_Cooldown()` - 发送冷却信息
  - `SendCommand_Aura()` - 发送光环信息

- **常量定义**：
  - `SPECTATOR_ADDON_VERSION` - 插件版本号
  - `SPECTATOR_COOLDOWN_MIN/MAX` - 冷却时间范围
  - `SPECTATOR_SPELL_BINDSIGHT` - 绑定视野法术

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `arena_team` | 竞技场队伍表，存储队伍名称、等级、统计等 |
| `arena_team_member` | 竞技场队伍成员表 |
| `character_arena_stats` | 角色竞技场统计表，存储各赛季积分 |
| `log_arena_fights` | 竞技场战斗日志表 |
| `log_arena_memberstats` | 竞技场成员统计日志表 |
| `pvpstats_battlegrounds` | PVP战场统计表 |
| `pvpstats_players` | PVP玩家统计表 |

---

### 6. AuctionHouse - 拍卖行

**核心类**：
- `AuctionHouseMgr` - 拍卖行管理器
- `AuctionHouseObject` - 拍卖行对象(联盟/部落/中立)
- `AuctionEntry` - 拍卖条目

**拍卖行阵营**：
- `AuctionHouseFaction::Alliance` - 联盟拍卖行
- `AuctionHouseFaction::Horde` - 部落拍卖行
- `AuctionHouseFaction::Neutral` - 中立拍卖行

**主要功能**：
- **拍卖操作**：
  - `AddAuction()` - 添加拍卖
  - `RemoveAuction()` - 移除拍卖
  - `Update()` - 更新拍卖状态

- **邮件通知**：
  - `SendAuctionWonMail()` - 发送竞拍成功邮件
  - `SendAuctionSuccessfulMail()` - 发送拍卖成功邮件
  - `SendAuctionExpiredMail()` - 发送拍卖过期邮件
  - `SendAuctionOutbiddedMail()` - 发送被超价邮件

- **常量定义**：
  - `MIN_AUCTION_TIME` - 最小拍卖时间(12小时)
  - `MAX_AUCTION_ITEMS` - 最大拍卖物品数(160)
  - `MAX_AUCTIONS_PER_PAGE` - 每页最大拍卖数(50)

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_characters | `auctionhouse` | 拍卖行表，存储拍卖物品、价格、时间、买家卖家等 |
| db_world | `auctionhouse_dbc` | 拍卖行定义表，定义拍卖行ID和阵营 |

---

### 7. Autobroadcast - 自动广播

**核心类**：`AutobroadcastMgr`

**广播类型**：`AnnounceType`
- `World` - 世界公告
- `Notification` - 屏幕通知
- `Both` - 两者同时

**主要功能**：
- `LoadAutobroadcasts()` - 加载广播消息
- `LoadAutobroadcastsLocalized()` - 加载本地化消息
- `SendAutobroadcasts()` - 发送广播消息

**数据结构**：
- `AutobroadcastsMap` - 广播消息映射(ID -> 消息列表)
- `AutobroadcastsWeightMap` - 消息权重映射(用于随机选择)

**数据库表 (db_auth)**：
| 表名 | 作用 |
|------|------|
| `autobroadcast` | 自动广播消息表，存储广播文本、权重、广播类型等 |
| `autobroadcast_locale` | 自动广播本地化表，支持多语言广播 |

---

### 8. Battlefield - 世界战场

**核心类**：
- `Battlefield` - 战场基类
- `BfCapturePoint` - 占领点
- `BfGraveyard` - 墓地

**战场类型**：
- `BATTLEFIELD_WG` - 冬拥湖
- `BATTLEFIELD_TB` - 托巴拉德(Cataclysm)

**主要功能**：
- **战斗流程**：
  - `StartBattle()` - 开始战斗
  - `EndBattle()` - 结束战斗
  - `IsWarTime()` - 是否战斗中

- **玩家管理**：
  - `HandlePlayerEnterZone()` - 玩家进入区域
  - `HandlePlayerLeaveZone()` - 玩家离开区域
  - `KickPlayerFromBattlefield()` - 踢出玩家

- **占领点系统**：
  - `HandlePlayerEnter()` - 玩家进入占领点
  - `HandlePlayerLeave()` - 玩家离开占领点
  - `Update()` - 更新占领状态
  - `ChangeTeam()` - 改变控制方

- **墓地系统**：
  - `GiveControlTo()` - 给予控制权
  - `Resurrect()` - 复活玩家
  - `AddPlayer()` - 添加玩家到复活队列

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `game_graveyard` | 墓地定义表，存储墓地ID、位置、所属区域等 |
| `graveyard_zone` | 墓地区域关联表，定义各区域的默认复活点 |
| `outdoorpvp_template` | 户外PVP模板表，定义世界PVP区域配置 |

---

### 9. Battlegrounds - 战场/竞技场

**核心类**：
- `Battleground` - 战场基类
- `BattlegroundMgr` - 战场管理器
- `BattlegroundQueue` - 战场队列
- `Arena` - 竞技场

**战场类型**：
- `BattlegroundWS` - 战歌峡谷(夺旗)
- `BattlegroundAB` - 阿拉希盆地(资源点)
- `BattlegroundAV` - 奥特兰克山谷(大型战场)
- `BattlegroundEY` - 风暴之眼(混合模式)
- `BattlegroundSA` - 远滩海滩(攻防战)
- `BattlegroundIC` - 岛屿征服(大型战场)

**竞技场类型**：`ArenaType`
- `ARENA_TYPE_2v2` - 2v2竞技场
- `ARENA_TYPE_3v3` - 3v3竞技场
- `ARENA_TYPE_5v5` - 5v5竞技场

**战场状态**：`BattlegroundStatus`
- `STATUS_NONE` - 无状态
- `STATUS_WAIT_QUEUE` - 等待排队
- `STATUS_WAIT_JOIN` - 等待加入
- `STATUS_IN_PROGRESS` - 进行中
- `STATUS_WAIT_LEAVE` - 等待离开

**主要功能**：
- `AddPlayer()` - 添加玩家
- `RemovePlayerAtLeave()` - 移除玩家
- `EndBattleground()` - 结束战场
- `UpdatePlayerScore()` - 更新玩家分数
- `SendPacketToAll()` - 向所有玩家发送包

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `battleground_template` | 战场模板表，定义战场ID、类型、名称、人数限制等 |
| db_world | `battlemaster_entry` | 战场管理员表，定义NPC与战场的对应关系 |
| db_world | `battlemasterlist_dbc` | 战场管理员列表定义表 |
| db_world | `pvpdifficulty_dbc` | PVP难度定义表 |
| db_world | `game_event_battleground_holiday` | 战场假期事件表，定义战场节假日 |
| db_world | `arena_season_reward` | 竞技场赛季奖励表 |
| db_world | `arena_season_reward_group` | 竞技场赛季奖励组表 |
| db_characters | `character_battleground_random` | 角色随机战场状态表 |
| db_characters | `battleground_deserters` | 战场逃亡者记录表 |
| db_characters | `pvpstats_battlegrounds` | PVP战场统计表 |
| db_characters | `pvpstats_players` | PVP玩家统计表 |

---

### 10. Cache - 缓存系统

**核心类**：
- `WhoListCacheMgr` - Who列表缓存管理器
- `CharacterCache` - 角色缓存

**WhoListPlayerInfo 结构**：
- 玩家GUID、团队、权限等级
- 等级、职业、种族、区域、性别
- 是否可见、玩家名称、公会名称

**主要功能**：
- `Update()` - 更新缓存(每5秒)
- `GetWhoList()` - 获取Who列表

**性能优化**：
- 避免每次查询都遍历所有在线玩家
- 缓存结果供多个查询复用

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_characters | `characters` | 角色主表，缓存从中读取角色名称、等级、种族、职业等 |
| db_characters | `guild_member` | 公会成员表，缓存从中读取玩家公会信息 |

---

### 11. Calendar - 日历系统

**核心类**：`CalendarMgr`

**事件类型**：`CalendarEventType`
- `CALENDAR_TYPE_RAID` - 团队副本
- `CALENDAR_TYPE_DUNGEON` - 地下城
- `CALENDAR_TYPE_PVP` - PVP活动
- `CALENDAR_TYPE_MEETING` - 会议
- `CALENDAR_TYPE_OTHER` - 其他

**邀请状态**：`CalendarInviteStatus`
- `CALENDAR_STATUS_INVITED` - 已邀请
- `CALENDAR_STATUS_ACCEPTED` - 已接受
- `CALENDAR_STATUS_DECLINED` - 已拒绝
- `CALENDAR_STATUS_CONFIRMED` - 已确认
- `CALENDAR_STATUS_STANDBY` - 待命

**主要功能**：
- **事件管理**：
  - `AddEvent()` - 添加事件
  - `RemoveEvent()` - 移除事件
  - `UpdateEvent()` - 更新事件

- **邀请管理**：
  - `AddInvite()` - 添加邀请
  - `RemoveInvite()` - 移除邀请
  - `UpdateInvite()` - 更新邀请

- **查询功能**：
  - `GetPlayerEvents()` - 获取玩家事件
  - `GetGuildEvents()` - 获取公会事件
  - `GetPlayerInvites()` - 获取玩家邀请

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `calendar_events` | 日历事件表，存储事件ID、标题、时间、类型、创建者等 |
| `calendar_invites` | 日历邀请表，存储邀请ID、事件ID、被邀请者、状态等 |

---

### 12. Chat - 聊天系统

**核心类**：
- `ChatHandler` - 聊天处理器
- `CliHandler` - 控制台命令处理器
- `AddonChannelCommandHandler` - 插件频道命令处理器

**主要功能**：
- **消息构建**：
  - `BuildChatPacket()` - 构建聊天数据包

- **消息发送**：
  - `SendNotification()` - 发送通知
  - `SendGMText()` - 发送GM文本
  - `SendWorldText()` - 发送世界文本
  - `SendSysMessage()` - 发送系统消息

- **命令处理**：
  - `ParseCommands()` - 解析命令
  - `extractKeyFromLink()` - 从链接提取关键字
  - `extractSpellIdFromLink()` - 从链接提取法术ID

- **目标选择**：
  - `getSelectedPlayer()` - 获取选中玩家
  - `getSelectedCreature()` - 获取选中生物
  - `getSelectedUnit()` - 获取选中单位

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `command` | GM命令表，定义命令名称、权限等级、帮助文本等 |
| db_world | `acore_string` | 核心字符串表，存储系统消息文本 |
| db_world | `broadcast_text` | 广播文本表，存储NPC对话和系统消息 |
| db_world | `broadcast_text_locale` | 广播文本本地化表 |
| db_characters | `channels` | 自定义频道表，存储频道名称、密码、权限等 |
| db_characters | `channels_bans` | 频道封禁表 |
| db_characters | `channels_rights` | 频道权限表 |

---

### 13. Combat - 战斗系统

**核心类**：
- `ThreatManager` - 仇恨管理器
- `ThreatReference` - 仇恨引用
- `CombatManager` - 战斗管理器

**仇恨状态**：`OnlineState`
- `ONLINE_STATE_ONLINE` - 在线(正常目标)
- `ONLINE_STATE_SUPPRESSED` - 抑制(免疫目标)
- `ONLINE_STATE_OFFLINE` - 离线(无效目标)

**嘲讽状态**：`TauntState`
- `TAUNT_STATE_TAUNT` - 被嘲讽
- `TAUNT_STATE_NONE` - 正常
- `TAUNT_STATE_DETAUNT` - 被降仇恨

**主要功能**：
- **仇恨操作**：
  - `AddThreat()` - 添加仇恨
  - `ScaleThreat()` - 缩放仇恨
  - `ResetThreat()` - 重置仇恨
  - `ClearAllThreat()` - 清除所有仇恨

- **目标选择**：
  - `GetCurrentVictim()` - 获取当前目标
  - `GetLastVictim()` - 获取上一个目标
  - `UpdateVictim()` - 更新目标

- **仇恨列表**：
  - `GetSortedThreatList()` - 获取排序的仇恨列表
  - `GetUnsortedThreatList()` - 获取未排序的仇恨列表
  - `GetThreatListSize()` - 获取仇恨列表大小

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `creature_classlevelstats` | 生物等级属性表，定义各等级生物的基础属性 |
| `creature_template` | 生物模板表，包含伤害、攻击速度等战斗相关字段 |
| `spell_threat` | 法术威胁值表，定义法术产生的仇恨值 |
| `spell_bonus_data` | 法术奖励数据表，定义法术伤害/治疗加成系数 |

---

### 14. Conditions - 条件系统

**核心类**：`ConditionMgr`

**条件类型**（部分）：`ConditionTypes`
- `CONDITION_AURA` - 光环条件
- `CONDITION_ITEM` - 物品条件
- `CONDITION_ZONEID` - 区域条件
- `CONDITION_REPUTATION_RANK` - 声望条件
- `CONDITION_SKILL` - 技能条件
- `CONDITION_QUESTREWARDED` - 任务奖励条件
- `CONDITION_CLASS` - 职业条件
- `CONDITION_RACE` - 种族条件
- `CONDITION_LEVEL` - 等级条件
- `CONDITION_NEAR_CREATURE` - 附近生物条件

**条件源类型**：`ConditionSourceType`
- `CONDITION_SOURCE_TYPE_CREATURE_LOOT_TEMPLATE` - 生物掉落
- `CONDITION_SOURCE_TYPE_SPELL` - 法术条件
- `CONDITION_SOURCE_TYPE_GOSSIP_MENU` - 菜单条件
- `CONDITION_SOURCE_TYPE_QUEST_AVAILABLE` - 任务可用条件

**主要功能**：
- 条件检查用于控制游戏逻辑分支
- 支持复杂的嵌套条件组合
- 可配置在数据库中动态加载

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `conditions` | 条件表，定义各类条件判断逻辑，用于控制游戏流程分支 |
| `disables` | 禁用表，定义被禁用的游戏内容(物品/任务/副本等) |

---

### 15. DataStores - 数据存储

**核心功能**：
- 加载DBC(Database Client)文件
- 解析游戏数据到内存结构

**主要数据类型**：
- 技能数据(Spell.dbc)
- 物品数据(Item.dbc)
- 生物模板(CreatureDisplayInfo.dbc)
- 地图数据(Map.dbc)
- 区域数据(AreaTable.dbc)
- 模型数据(M2文件)

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `*_dbc` | DBC数据表，存储从客户端DBC文件导入的游戏基础数据 |
| `spell_dbc` | 法术定义表 |
| `item_dbc` | 物品定义表 |
| `creature*_dbc` | 生物相关定义表 |
| `map_dbc` | 地图定义表 |
| `areatable_dbc` | 区域定义表 |
| `faction_dbc` | 阵营定义表 |
| `chrclasses_dbc` | 职业定义表 |
| `chrraces_dbc` | 种族定义表 |

---

### 16. DungeonFinding - 地下城查找器

**核心类**：`LFGMgr`

**加入结果**：`LfgJoinResult`
- `LFG_JOIN_OK` - 加入成功
- `LFG_JOIN_FAILED` - 加入失败
- `LFG_JOIN_GROUPFULL` - 队伍已满
- `LFG_JOIN_DESERTER` - 有逃亡者debuff
- `LFG_JOIN_RANDOM_COOLDOWN` - 随机副本冷却中

**角色检查状态**：`LfgRoleCheckState`
- `LFG_ROLECHECK_FINISHED` - 角色检查完成
- `LFG_ROLECHECK_MISSING_ROLE` - 缺少角色
- `LFG_ROLECHECK_WRONG_ROLES` - 角色错误

**主要功能**：
- 随机副本排队
- 角色检查(坦克/治疗/输出)
- 队列匹配
- 奖励分配
- 副本浏览器

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `lfg_dungeon_template` | LFG副本模板表，定义副本队列配置 |
| db_world | `lfg_dungeon_rewards` | LFG奖励表，定义随机副本奖励 |
| db_world | `lfgdungeons_dbc` | LFG副本定义表 |
| db_world | `dungeon_access_template` | 副本访问模板表 |
| db_world | `dungeon_access_requirements` | 副本访问需求表 |
| db_characters | `lfg_data` | LFG数据表，存储玩家LFG状态 |

---

### 17. Entities - 实体系统

**核心类层次**：
```
Object
  +-- Unit
  |     +-- Player
  |     +-- Creature
  |     |     +-- Pet
  |     |     +-- Totem
  |     |     +-- TempSummon
  |     +-- Vehicle
  +-- GameObject
  +-- Item
  |     +-- Bag
  +-- Corpse
  +-- DynamicObject
```

**Unit 类主要功能**：
- **属性系统**：生命值、法力值、属性值、抗性
- **战斗系统**：攻击、伤害、命中、暴击
- **光环系统**：Buff/Debuff管理
- **移动系统**：位置、朝向、速度
- **仇恨系统**：ThreatManager集成

**Player 类主要功能**：
- **角色数据**：等级、经验、天赋、技能
- **物品系统**：背包、装备、银行
- **任务系统**：任务日志、进度追踪
- **社交系统**：好友、公会、队伍
- **PVP系统**：荣誉、战场、竞技场

**Creature 类主要功能**：
- AI行为控制
- 刷新/死亡管理
- 掉落/拾取
- 商店/训练

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_characters | `characters` | 角色主表，存储角色基础数据(等级/经验/位置/金钱等) |
| db_characters | `character_stats` | 角色属性表，存储当前属性值 |
| db_characters | `character_inventory` | 角色背包表，存储物品位置信息 |
| db_characters | `item_instance` | 物品实例表，存储物品详细属性 |
| db_characters | `character_aura` | 角色光环表，存储Buff/Debuff |
| db_characters | `character_spell` | 角色法术表，存储已学法术 |
| db_characters | `character_talent` | 角色天赋表 |
| db_characters | `character_skills` | 角色技能表 |
| db_characters | `character_action` | 角色动作条表 |
| db_characters | `character_glyphs` | 角色雕文表 |
| db_characters | `character_pet` | 角色宠物表 |
| db_characters | `corpse` | 尸体表 |
| db_world | `creature` | 生物刷新表，定义生物刷新位置 |
| db_world | `creature_template` | 生物模板表，定义生物属性 |
| db_world | `gameobject` | 游戏对象刷新表 |
| db_world | `gameobject_template` | 游戏对象模板表 |
| db_world | `item_template` | 物品模板表 |
| db_world | `playercreateinfo` | 玩家创建信息表，定义新角色初始数据 |

---

### 18. Events - 游戏事件

**核心类**：
- `GameEventMgr` - 游戏事件管理器
- `HolidayDateCalculator` - 假期日期计算器

**主要功能**：
- 节日活动(冬幕节/情人节等)
- 世界事件(暗月马戏团等)
- 周期性事件触发
- 事件持续时间管理

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `game_event` | 游戏事件表，定义事件ID、开始时间、结束时间、周期等 |
| `game_event_prerequisite` | 事件前提条件表 |
| `game_event_condition` | 事件条件表 |
| `game_event_creature` | 事件生物表，定义事件期间刷新的NPC |
| `game_event_gameobject` | 事件游戏对象表 |
| `game_event_quest_condition` | 事件任务条件表 |
| `game_event_creature_quest` | 事件NPC任务表 |
| `game_event_gameobject_quest` | 事件对象任务表 |
| `game_event_pool` | 事件池表 |
| `game_event_model_equip` | 事件模型装备表 |
| `game_event_npc_vendor` | 事件NPC商人表 |
| `game_event_npcflag` | 事件NPC标志表 |
| `game_event_arena_seasons` | 事件竞技场赛季表 |
| `game_event_battleground_holiday` | 事件战场假期表 |
| `game_event_seasonal_questrelation` | 事件季节任务关系表 |
| `holiday_dates` | 假期日期表 |
| `holidays_dbc` | 假期定义表 |
| db_characters | `game_event_save` | 事件保存表，记录事件状态 |
| db_characters | `game_event_condition_save` | 事件条件保存表 |

---

### 19. Globals - 全局对象管理

**核心类**：
- `ObjectAccessor` - 对象访问器
- `ObjectMgr` - 对象管理器

**ObjectAccessor 主要功能**：
- `FindPlayer()` - 查找玩家
- `FindCreature()` - 查找生物
- `FindGameObject()` - 查找游戏对象
- `GetPlayers()` - 获取所有玩家

**ObjectMgr 主要功能**：
- 加载所有模板数据
- 管理NPC/物品/任务等模板
- 提供模板查询接口

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `creature_template` | 生物模板表 |
| db_world | `gameobject_template` | 游戏对象模板表 |
| db_world | `item_template` | 物品模板表 |
| db_world | `quest_template` | 任务模板表 |
| db_world | `npc_trainer` | NPC训练师表 |
| db_world | `npc_vendor` | NPC商人表 |
| db_world | `gossip_menu` | 对话菜单表 |

---

### 20. Grids - 网格系统

**核心概念**：
- 将地图划分为网格(Grid)
- 每个网格包含多个单元格(Cell)
- 按需加载/卸载网格

**主要类**：
- `MapGrid` - 地图网格
- `Cell` - 单元格
- `GridNotifiers` - 网格通知器

**优化效果**：
- 减少内存占用
- 提高查询效率
- 支持动态加载

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `map_dbc` | 地图定义表，定义地图ID、名称、类型等 |
| `mapdifficulty_dbc` | 地图难度定义表 |
| `areatable_dbc` | 区域定义表，定义区域与地图的关系 |

---

### 21. Groups - 队伍系统

**核心类**：`Group`

**队伍类型**：`GroupType`
- `GROUPTYPE_NORMAL` - 普通队伍(5人)
- `GROUPTYPE_RAID` - 团队(最多40人)
- `GROUPTYPE_BG` - 战场队伍
- `GROUPTYPE_LFG` - 随机副本队伍

**分配方式**：`LootMethod`
- `FREE_FOR_ALL` - 自由拾取
- `ROUND_ROBIN` - 轮流拾取
- `MASTER_LOOT` - 队长分配
- `GROUP_LOOT` - 队伍分配
- `NEED_BEFORE_GREED` - 需求优先

**主要功能**：
- 队伍创建/解散
- 成员管理/邀请
- 队长转让
- 分配设置
- 副本进度绑定

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `groups` | 队伍表，存储队伍ID、队长、类型、分配方式等 |
| `group_member` | 队伍成员表，存储成员角色、子组等 |

---

### 22. Guilds - 公会系统

**核心类**：`Guild`

**公会等级**：`GuildDefaultRanks`
- `GR_GUILDMASTER` - 公会会长
- `GR_OFFICER` - 官员
- `GR_VETERAN` - 资深会员
- `GR_MEMBER` - 普通会员
- `GR_INITIATE` - 新会员

**公会权限**：`GuildRankRights`
- `GR_RIGHT_GCHATLISTEN` - 公会频道监听
- `GR_RIGHT_GCHATSPEAK` - 公会频道发言
- `GR_RIGHT_INVITE` - 邀请成员
- `GR_RIGHT_REMOVE` - 移除成员
- `GR_RIGHT_WITHDRAW_GOLD` - 提取金币

**主要功能**：
- 公会创建/解散
- 成员管理
- 公会银行
- 公会等级/经验
- 公会事件日志

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `guild` | 公会表，存储公会名称、会长、创建日期、信息等 |
| `guild_member` | 公会成员表，存储成员等级、备注、贡献等 |
| `guild_rank` | 公会等级表，定义公会等级名称和权限 |
| `guild_bank_tab` | 公会银行标签页表 |
| `guild_bank_item` | 公会银行物品表 |
| `guild_bank_right` | 公会银行权限表 |
| `guild_bank_eventlog` | 公会银行事件日志表 |
| `guild_eventlog` | 公会事件日志表 |
| `guild_member_withdraw` | 公会成员提取记录表 |

---

### 23. Handlers - 网络消息处理

**主要处理器**：
- `AuthHandler` - 认证处理
- `CharacterHandler` - 角色处理
- `ChatHandler` - 聊天处理
- `CombatHandler` - 战斗处理
- `MovementHandler` - 移动处理
- `SpellHandler` - 法术处理
- `ItemHandler` - 物品处理
- `QuestHandler` - 任务处理
- `GuildHandler` - 公会处理
- `BattleGroundHandler` - 战场处理

**消息类型**：
- `CMSG_*` - 客户端消息
- `SMSG_*` - 服务器消息

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_auth | `logs` | 登录日志表 |
| db_auth | `logs_ip_actions` | IP操作日志表 |
| db_characters | `log_arena_fights` | 竞技场战斗日志 |
| db_characters | `log_encounter` | Boss战斗日志 |
| db_characters | `log_money` | 金钱交易日志 |

---

### 24. Instances - 副本系统

**核心类**：
- `InstanceSaveMgr` - 副本进度保存管理器
- `InstanceScript` - 副本脚本基类

**主要功能**：
- 副本进度保存
- 重置时间管理
- Boss击杀记录
- 副本锁定

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `instance_template` | 副本模板表，定义副本ID、最大玩家数、重置时间等 |
| db_world | `instance_encounters` | 副本遭遇战表，定义Boss信息 |
| db_world | `mapdifficulty_dbc` | 地图难度定义表 |
| db_characters | `instance` | 副本实例表，存储副本进度 |
| db_characters | `instance_reset` | 副本重置时间表 |
| db_characters | `character_instance` | 角色副本进度表 |
| db_characters | `instance_saved_go_state_data` | 副本保存的游戏对象状态数据 |

---

### 25. Loot - 战利品系统

> **详细分析文档**: [Loot-System-Analysis.md](./Loot-System-Analysis.md)

**核心类**：
- `LootStore` - 掉落存储管理器，管理所有掉落模板
- `LootTemplate` - 掉落模板，定义一组掉落规则
- `LootGroup` - 掉落分组，组内物品共享概率池
- `LootStoreItem` - 掉落条目定义（物品ID/概率/数量等）
- `LootItem` - 实际掉落物品实例
- `Loot` - 掉落对象，包含一次掉落的所有物品
- `LootView` - 掉落视图，处理玩家可见性

**掉落类型**：`LootType`
- `LOOT_CORPSE` - 尸体掉落（怪物击杀）
- `LOOT_PICKPOCKETING` - 偷窃（盗贼技能）
- `LOOT_FISHING` - 钓鱼
- `LOOT_DISENCHANTING` - 分解（附魔）
- `LOOT_SKINNING` - 剥皮
- `LOOT_PROSPECTING` - 选矿（珠宝）
- `LOOT_MILLING` - 研磨（铭文）

**分配方式**：`LootMethod`
- `FREE_FOR_ALL` - 自由拾取
- `ROUND_ROBIN` - 轮流拾取
- `MASTER_LOOT` - 队长分配
- `GROUP_LOOT` - 队伍分配（掷骰）
- `NEED_BEFORE_GREED` - 需求优先

**掷骰类型**：`RollType`
- `ROLL_PASS` - 放弃
- `ROLL_NEED` - 需要
- `ROLL_GREED` - 贪婪
- `ROLL_DISENCHANT` - 分解

**核心流程**：
1. **模板加载**: `LoadLootTables()` → `LoadLootTemplate_*()` → `LootStore::LoadLootTable()`
2. **掉落生成**: `FillLoot()` → `LootTemplate::Process()` → `LootGroup::Roll()` → `AddItem()`
3. **权限检查**: `GetPermission()` → 检查分配方式/阈值/队伍状态
4. **物品拾取**: `StoreLootItem()` → `CanStoreNewItem()` → `StoreNewItem()`

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `creature_loot_template` | 生物掉落表，定义怪物掉落物品 |
| `gameobject_loot_template` | 游戏对象掉落表 |
| `item_loot_template` | 物品掉落表，定义开箱/袋子掉落 |
| `fishing_loot_template` | 钓鱼掉落表 |
| `skinning_loot_template` | 剥皮掉落表 |
| `pickpocketing_loot_template` | 偷窃掉落表 |
| `disenchant_loot_template` | 分解掉落表 |
| `mail_loot_template` | 邮件掉落表 |
| `spell_loot_template` | 法术掉落表 |
| `reference_loot_template` | 参考掉落表，定义可复用的掉落模板 |
| `prospecting_loot_template` | 选矿掉落表 |
| `milling_loot_template` | 研磨掉落表 |
| `player_loot_template` | 玩家掉落表(PVP) |

---

### 26. Mails - 邮件系统

**核心功能**：
- 邮件发送/接收
- 附件处理
- 邮件过期
- 服务器邮件(GM发送)
- 拍卖行邮件

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `mail` | 邮件表，存储邮件发送者、接收者、主题、正文、时间等 |
| `mail_items` | 邮件物品表，存储邮件附件 |
| `mail_server_template` | 服务器邮件模板表 |
| `mail_server_template_items` | 服务器邮件模板物品表 |
| `mail_server_template_conditions` | 服务器邮件模板条件表 |
| `mail_server_character` | 服务器邮件角色表 |
| db_world | `mail_level_reward` | 邮件等级奖励表，定义升级奖励邮件 |
| db_world | `mailtemplate_dbc` | 邮件模板定义表 |

---

### 27. Maps - 地图系统

**核心类**：
- `Map` - 地图基类
- `MapInstanced` - 副本地图
- `BattlegroundMap` - 战场地图
- `InstanceMap` - 副本地图

**主要功能**：
- 地图加载/卸载
- 对象管理
- 可见性控制
- 天气系统
- 地形数据

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `map_dbc` | 地图定义表，定义地图ID、名称、类型、目录等 |
| `mapdifficulty_dbc` | 地图难度定义表 |
| `game_weather` | 天气表，定义各地图的天气概率 |
| `transports` | 交通工具表，定义船只/飞艇等 |
| `transportanimation_dbc` | 交通动画表 |
| `transportrotation_dbc` | 交通旋转表 |
| `areatrigger` | 区域触发器表 |
| `areatrigger_teleport` | 区域触发器传送表 |
| `areatrigger_tavern` | 区域触发器旅馆表 |
| `areatrigger_scripts` | 区域触发器脚本表 |

---

### 28. Misc - 杂项功能

**BanMgr 主要功能**：
- 账号封禁
- IP封禁
- 角色封禁

**GameGraveyard 主要功能**：
- 墓地位置管理
- 复活点分配

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_auth | `account_banned` | 账号封禁表 |
| db_auth | `ip_banned` | IP封禁表 |
| db_characters | `character_banned` | 角色封禁表 |
| db_world | `game_graveyard` | 墓地定义表 |
| db_world | `graveyard_zone` | 墓地区域关联表 |

---

### 29. Miscellaneous - 辅助工具

**Formulas 主要功能**：
- 经验计算公式
- 升级所需经验
- 荣誉计算

**Language 主要功能**：
- 语言常量定义
- 消息ID定义

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `player_xp_for_level` | 升级经验表，定义各等级所需经验 |
| `exploration_basexp` | 探索经验表，定义探索区域基础经验 |
| `player_class_stats` | 玩家职业属性表 |
| `player_race_stats` | 玩家种族属性表 |

---

### 30. Modules - 模块系统

**核心类**：`ModuleMgr`

**主要功能**：
- 动态模块加载
- 模块依赖管理
- 模块配置

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `module_string` | 模块字符串表，存储模块自定义文本 |
| `module_string_locale` | 模块字符串本地化表 |

---

### 31. Motd - 每日消息

**核心类**：`MotdMgr`

**主要功能**：
- 管理登录公告
- 支持动态更新

**数据库表 (db_auth)**：
| 表名 | 作用 |
|------|------|
| `motd` | 每日消息表，存储服务器登录公告 |
| `motd_localized` | 每日消息本地化表 |

---

### 32. Movement - 移动系统

**移动生成器类型**：`MovementGeneratorType`
- `IDLE_MOTION_TYPE` - 空闲
- `RANDOM_MOTION_TYPE` - 随机移动
- `WAYPOINT_MOTION_TYPE` - 路径点移动
- `CHASE_MOTION_TYPE` - 追击
- `HOME_MOTION_TYPE` - 返回
- `FLEEING_MOTION_TYPE` - 逃跑
- `FOLLOW_MOTION_TYPE` - 跟随
- `FORMATION_MOTION_TYPE` - 编队

**核心类**：
- `MotionMaster` - 移动主控器
- `PathGenerator` - 路径生成器
- `MoveSpline` - 移动曲线

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `waypoint_data` | 路径点数据表，定义NPC巡逻路径 |
| `waypoint_scripts` | 路径点脚本表 |
| `script_waypoint` | 脚本路径点表 |
| `creature_addon` | 生物附加数据，包含路径ID |
| `creature_template_movement` | 生物模板移动表，定义移动类型和速度 |
| `creature_movement_override` | 生物移动覆盖表 |

---

### 33. OutdoorPvP - 户外PVP

**核心类**：
- `OutdoorPvP` - 户外PVP基类
- `OutdoorPvPMgr` - 户外PVP管理器

**主要功能**：
- 世界PVP区域控制
- PVP目标争夺
- 奖励发放

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `outdoorpvp_template` | 户外PVP模板表，定义世界PVP区域配置 |

---

### 34. Petitions - 请愿系统

**核心类**：`PetitionMgr`

**主要功能**：
- 公会创建请愿
- 签名收集
- 竞技场队伍创建请愿

**数据库表 (db_characters)**：
| 表名 | 作用 |
|------|------|
| `petition` | 请愿表，存储请愿ID、类型、名称、创建者等 |
| `petition_sign` | 请愿签名表，存储签名玩家信息 |

---

### 35. Pools - 对象池

**核心类**：`PoolMgr`

**主要功能**：
- 对象池管理
- 动态刷新
- 资源复用

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `pool_template` | 对象池模板表，定义池ID、最大刷新数、刷新间隔等 |
| `pool_creature` | 生物池表，定义生物池成员 |
| `pool_gameobject` | 游戏对象池表 |
| `pool_quest` | 任务池表 |
| `pool_pool` | 池嵌套表，定义子池 |
| db_characters | `pool_quest_save` | 任务池保存表 |

---

### 36. Quests - 任务系统

> **详细分析文档**: [Quest-System-Analysis.md](./Quest-System-Analysis.md)

**核心类**：
- `Quest` - 任务定义类 (QuestDef.h)
- `QuestStatusData` - 任务状态数据结构
- `Player` - 玩家任务操作 (PlayerQuest.cpp)

**任务状态**：`QuestStatus`
- `QUEST_STATUS_NONE` - 无状态（未接取）
- `QUEST_STATUS_INCOMPLETE` - 进行中
- `QUEST_STATUS_COMPLETE` - 已完成（可提交）
- `QUEST_STATUS_FAILED` - 失败
- `QUEST_STATUS_REWARDED` - 已领奖（仅内存使用）

**任务标志**：`QuestFlags`
- `QUEST_FLAGS_DAILY` - 每日任务
- `QUEST_FLAGS_WEEKLY` - 每周任务
- `QUEST_FLAGS_SHARABLE` - 可分享
- `QUEST_FLAGS_TRACKING` - 追踪任务（自动完成）
- `QUEST_FLAGS_AUTO_ACCEPT` - 自动接受

**特殊标志**：`QuestSpecialFlags`
- `QUEST_SPECIAL_FLAGS_REPEATABLE` - 可重复
- `QUEST_SPECIAL_FLAGS_TIMED` - 计时任务
- `QUEST_SPECIAL_FLAGS_EXPLORATION_OR_EVENT` - 探索/事件
- `QUEST_SPECIAL_FLAGS_MONTHLY` - 每月任务

**主要功能**：
- 任务接取/完成/放弃
- 目标追踪（击杀/收集/探索/施法）
- 奖励发放（经验/金币/物品/声望/称号）
- 任务链管理（前置/后续/互斥）

**核心流程**：
1. **接取流程**: `CanTakeQuest()` → `CanAddQuest()` → `AddQuest()`
2. **进度更新**: `KilledMonster()` / `ItemAdded()` → `UpdateQuestObjective()`
3. **完成流程**: `CanCompleteQuest()` → `CompleteQuest()`
4. **领奖流程**: `CanRewardQuest()` → `RewardQuest()`

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `quest_template` | 任务模板表，定义任务ID、名称、目标、奖励等 |
| db_world | `quest_template_addon` | 任务模板附加表，定义任务链、前置任务等 |
| db_world | `quest_template_locale` | 任务模板本地化表 |
| db_world | `quest_details` | 任务详情表，定义任务描述文本 |
| db_world | `quest_offer_reward` | 任务奖励文本表 |
| db_world | `quest_request_items` | 任务请求物品文本表 |
| db_world | `quest_greeting` | 任务问候文本表 |
| db_world | `quest_poi` | 任务POI表，定义任务目标地图标记 |
| db_world | `quest_poi_points` | 任务POI点表 |
| db_world | `quest_money_reward` | 任务金钱奖励表 |
| db_world | `creature_queststarter` | NPC任务开始表，定义任务发布者 |
| db_world | `creature_questender` | NPC任务结束表 |
| db_world | `gameobject_queststarter` | 对象任务开始表 |
| db_world | `gameobject_questender` | 对象任务结束表 |
| db_world | `creature_questitem` | NPC任务物品表 |
| db_world | `gameobject_questitem` | 对象任务物品表 |
| db_characters | `character_queststatus` | 角色任务状态表 |
| db_characters | `character_queststatus_rewarded` | 角色已完成任务表 |
| db_characters | `character_queststatus_daily` | 每日任务状态表 |
| db_characters | `character_queststatus_weekly` | 每周任务状态表 |
| db_characters | `character_queststatus_monthly` | 每月任务状态表 |
| db_characters | `character_queststatus_seasonal` | 季节任务状态表 |
| db_characters | `quest_tracker` | 任务追踪表，记录任务完成日志 |

---

### 37. Reputation - 声望系统

**核心类**：`ReputationMgr`

**主要功能**：
- 阵营声望计算
- 声望等级(仇恨/敌对/中立/友善/尊敬/崇敬/崇拜)
- 声望奖励
- 声望对立关系

**数据库表**：
| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `faction_dbc` | 阵营定义表，定义阵营ID、名称、对立关系等 |
| db_world | `factiontemplate_dbc` | 阵营模板表 |
| db_world | `reputation_reward_rate` | 声望奖励率表 |
| db_world | `reputation_spillover_template` | 声望溢出模板表 |
| db_world | `creature_onkill_reputation` | 生物击杀声望表 |
| db_world | `player_factionchange_reputations` | 玩家阵营变更声望表 |
| db_characters | `character_reputation` | 角色声望表，存储玩家各阵营声望值 |

---

### 38. Scripting - 脚本系统

**脚本类型**：
- `AccountScript` - 账号脚本
- `AchievementScript` - 成就脚本
- `MapScript` - 地图脚本
- `PlayerScript` - 玩家脚本
- `CreatureScript` - 生物脚本
- `ItemScript` - 物品脚本
- `SpellScript` - 法术脚本

**主要功能**：
- 提供脚本扩展接口
- 支持模块化脚本注册
- 支持热重载

**数据库表 (db_world)**：
| 表名 | 作用 |
|------|------|
| `smart_scripts` | SmartAI脚本表，定义AI行为脚本 |
| `spell_scripts` | 法术脚本表 |
| `spell_script_names` | 法术脚本名称表 |
| `event_scripts` | 事件脚本表 |
| `waypoint_scripts` | 路径点脚本表 |
| `areatrigger_scripts` | 区域触发器脚本表 |

---

## 总结

AzerothCore的game服务器采用了高度模块化的架构设计：

1. **网络层(Handlers)**: 负责网络通信和协议解析，处理客户端请求
2. **基础业务层(Chat/Combat/Movement)**: 提供游戏基础功能
3. **实体层(Entities)**: 所有游戏对象的基类定义(Player/Creature/Item/GameObject)
4. **游戏系统层**: 包含战斗/任务/声望/公会等游戏玩法
5. **AI层**: 负责NPC行为逻辑

这种架构使得各模块职责清晰，便于维护和扩展。

---

*本文档基于 AzerothCore WotLK 源代码分析生成*
