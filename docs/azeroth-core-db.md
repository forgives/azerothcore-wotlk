# AzerothCore 数据库架构深度分析

## 目录

1. [数据库架构概览](#数据库架构概览)
2. [auth 数据库（认证数据库）](#auth-数据库认证数据库)
3. [characters 数据库（角色数据库）](#characters-数据库角色数据库)
4. [world 数据库（世界数据库）](#world-数据库世界数据库)
5. [数据库连接池架构](#数据库连接池架构)
6. [表关系和依赖](#表关系和依赖)
7. [索引和性能优化](#索引和性能优化)

---

## 数据库架构概览

### 三数据库设计

AzerothCore 使用 MySQL 8.0+ 数据库，采用**三数据库分离设计**：

```
┌─────────────────────────────────────────────────────────────┐
│                    AzerothCore 数据库系统                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────────┐  ┌─────────────┐  │
│  │   auth       │  │   characters     │  │   world     │  │
│  │  认证数据库    │  │    角色数据库      │  │  世界数据库   │  │
│  │              │  │                  │  │             │  │
│  │ - 账户管理    │  │ - 角色数据        │  │ - 游戏内容   │  │
│  │ - 权限控制    │  │ - 物品系统        │  │ - NPC配置    │  │
│  │ - 服务器列表  │  │ - 公会系统        │  │ - 任务数据   │  │
│  │ - 封禁管理    │  │ - 邮件系统        │  │ - 法术数据   │  │
│  └──────────────┘  └──────────────────┘  └─────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 数据库目录结构

```
data/sql/
├── base/                      # 基础数据库结构（最新版本）
│   ├── db_auth/              # auth 数据库表定义
│   │   ├── account.sql
│   │   ├── account_access.sql
│   │   ├── account_banned.sql
│   │   ├── realmlist.sql
│   │   ├── ip_banned.sql
│   │   └── ...
│   ├── db_characters/        # characters 数据库表定义
│   │   ├── characters.sql
│   │   ├── character_inventory.sql
│   │   ├── character_spell.sql
│   │   ├── guild.sql
│   │   └── ...
│   └── db_world/             # world 数据库表定义
│       ├── creature_template.sql
│       ├── item_template.sql
│       ├── quest_template.sql
│       └── ...
├── create/                    # 数据库创建脚本
│   ├── create_mysql.sql
│   └── drop_mysql.sql
├── updates/                   # 数据库更新文件
│   ├── pending_db_auth/
│   ├── pending_db_characters/
│   └── pending_db_world/
├── archive/                   # 归档的历史更新
└── custom/                    # 自定义数据
```

### 数据库命名规范

**命名标准（来自代码）：**

```
{DB}_{SEL/INS/UPD/DEL/REP}_{Summary of data changed}
```

| 前缀 | 说明 | 示例 |
|------|------|------|
| `LOGIN_SEL` | auth 数据库查询 | `LOGIN_SEL_REALMLIST` |
| `CHAR_SEL` | characters 数据库查询 | `CHAR_SEL_CHARACTER` |
| `CHAR_INS` | characters 数据库插入 | `CHAR_INS_CHARACTER_BAN` |
| `CHAR_UPD` | characters 数据库更新 | `CHAR_UPD_CHARACTER_BAN` |
| `CHAR_DEL` | characters 数据库删除 | `CHAR_DEL_QUEST_STATUS_DAILY` |

---

## auth 数据库（认证数据库）

**数据库名称：** `acore_auth`

**作用：** 管理用户账户、认证、权限和服务器列表

### 表结构详解

#### 1. account - 账户表

**作用：** 存储所有用户账户信息和认证数据

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 | 注释 |
|------|------|------|--------|------|
| `id` | int unsigned | 账户唯一标识符 | AUTO_INCREMENT | Identifier |
| `username` | varchar(32) | 用户名 | '' | 唯一索引 |
| `salt` | binary(32) | 密码盐值 | - | SRP6 认证 |
| `verifier` | binary(32) | 密码验证器 | - | SRP6 认证 |
| `session_key` | binary(40) | 会话密钥 | NULL | 登录后生成 |
| `totp_secret` | varbinary(128) | 双因素认证密钥 | NULL | 可选 |
| `email` | varchar(255) | 邮箱地址 | '' | 当前邮箱 |
| `reg_mail` | varchar(255) | 注册邮箱 | '' | 注册时邮箱 |
| `joindate` | timestamp | 注册时间 | CURRENT_TIMESTAMP | |
| `last_ip` | varchar(15) | 最后登录IP | '127.0.0.1' | |
| `last_attempt_ip` | varchar(15) | 最后尝试登录IP | '127.0.0.1' | |
| `failed_logins` | int unsigned | 失败登录次数 | 0 | 安全机制 |
| `locked` | tinyint unsigned | 账户锁定状态 | 0 | 0=未锁定 |
| `lock_country` | varchar(2) | 锁定国家代码 | '00' | |
| `last_login` | timestamp | 最后登录时间 | NULL | |
| `online` | int unsigned | 在线状态 | 0 | |
| `expansion` | tinyint unsigned | 资料片版本 | 2 | 2=WotLK |
| `Flags` | int unsigned | 账户标志 | 0 | 各种设置 |
| `mutetime` | bigint | 禁言到期时间 | 0 | Unix时间戳 |
| `mutereason` | varchar(255) | 禁言原因 | '' | |
| `muteby` | varchar(50) | 禁言执行者 | '' | |
| `locale` | tinyint unsigned | 语言区域 | 0 | 客户端语言 |
| `os` | varchar(3) | 操作系统 | '' | Win/Mac |
| `recruiter` | int unsigned | 推荐人ID | 0 | 推荐系统 |
| `totaltime` | int unsigned | 总游戏时间(秒) | 0 | 统计数据 |

**索引：**
- PRIMARY KEY (`id`)
- UNIQUE KEY `idx_username` (`username`)

**代码引用：** `LOGIN_INS_ACCOUNT`, `LOGIN_SEL_ACCOUNT_BY_ID`, `LOGIN_UPD_EXPANSION`

---

#### 2. realmlist - 服务器列表表

**作用：** 定义可连接的游戏服务器列表

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 服务器唯一ID | AUTO_INCREMENT |
| `name` | varchar(32) | 服务器名称 | '' |
| `address` | varchar(255) | 外部连接地址 | '127.0.0.1' |
| `localAddress` | varchar(255) | 内部连接地址 | '127.0.0.1' |
| `localSubnetMask` | varchar(255) | 子网掩码 | '255.255.255.0' |
| `port` | smallint unsigned | 端口号 | 8085 |
| `icon` | tinyint unsigned | 图标类型 | 0 |
| `flag` | tinyint unsigned | 服务器标志 | 2 |
| `timezone` | tinyint unsigned | 时区 | 0 |
| `allowedSecurityLevel` | tinyint unsigned | 允许的安全级别 | 0 |
| `population` | float | 人口数量 | 0 |
| `gamebuild` | int unsigned | 客户端版本 | 12340 |

**索引：**
- PRIMARY KEY (`id`)
- UNIQUE KEY `idx_name` (`name`)

**默认数据：**
```sql
(1,'AzerothCore','127.0.0.1','127.0.0.1','255.255.255.0',8085,0,0,1,0,0,12340)
```

**代码引用：** `LOGIN_SEL_REALMLIST`, `LOGIN_SEL_REALMLIST_SECURITY_LEVEL`

---

#### 3. account_access - 账户权限表

**作用：** 定义账户在不同服务器上的管理权限级别

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 自增ID | AUTO_INCREMENT |
| `gmlevel` | tinyint unsigned | GM等级 | 0 |
| `RealmID` | int signed | 服务器ID (-1=全部) | -1 |

**GM等级说明：**
- `0` - 普通玩家
- `1` - 版主
- `2` - 游戏管理员
- `3` - 高级管理员
- `4` - 开发者/拥有者

**代码引用：** `LOGIN_SEL_ACCOUNT_ACCESS`, `LOGIN_GET_ACCOUNT_ACCESS_GMLEVEL`

---

#### 4. account_banned - 账户封禁表

**作用：** 记录账户封禁信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 自增ID | AUTO_INCREMENT |
| `bandate` | int unsigned | 封禁开始时间 | 0 |
| `unbandate` | int unsigned | 封禁结束时间 | 0 |
| `bannedby` | varchar(50) | 封禁执行者 | '' |
| `banreason` | varchar(255) | 封禁原因 | '' |
| `active` | tinyint unsigned | 是否激活 | 1 |

**代码引用：** `LOGIN_SEL_ACCOUNT_BANNED`, `LOGIN_DEL_ACCOUNT_BANNED`, `LOGIN_UPD_EXPIRED_ACCOUNT_BANS`

---

#### 5. ip_banned - IP封禁表

**作用：** 记录IP地址封禁信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ip` | varchar(15) | IP地址 | '' |
| `bandate` | int unsigned | 封禁开始时间 | 0 |
| `unbandate` | int unsigned | 封禁结束时间 | 0 |
| `bannedby` | varchar(50) | 封禁执行者 | '' |
| `banreason` | varchar(255) | 封禁原因 | '' |

**代码引用：** `LOGIN_SEL_IP_BANNED`, `LOGIN_INS_IP_BANNED`, `LOGIN_DEL_IP_NOT_BANNED`

---

#### 6. uptime - 服务器运行时间表

**作用：** 记录服务器运行状态和统计

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `realmid` | int unsigned | 服务器ID | 0 |
| `starttime` | int unsigned | 启动时间 | 0 |
| `uptime` | int unsigned | 运行时长(秒) | 0 |
| `maxplayers` | smallint unsigned | 最大在线人数 | 0 |
| `revision` | varchar(255) | 服务器版本 | '' |

---

#### 7. build_info - 客户端版本信息表

**作用：** 支持的客户端版本配置

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `build` | int unsigned | 客户端版本号 |
| `majorVersion` | int unsigned | 主版本号 |
| `minorVersion` | int unsigned | 次版本号 |
| `bugfixVersion` | int unsigned | 修复版本号 |
| `hotfixVersion` | int unsigned | 热修版本号 |
| `winAuthSeed` | varchar(32) | Windows认证种子 |
| `macAuthSeed` | varchar(32) | Mac认证种子 |

---

#### 8. logs - 操作日志表

**作用：** 记录GM操作日志

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int unsigned | 自增ID |
| `time` | int unsigned | 时间戳 |
| `realm` | int unsigned | 服务器ID |
| `type` | tinyint unsigned | 日志类型 |
| `account` | int unsigned | 账户ID |
| `character` | int unsigned | 角色GUID |
| `logstring` | text | 日志内容 |

---

#### 9. autobroadcast - 自动广播表

**作用：** 服务器自动公告配置

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int unsigned | 自增ID |
| `text` | text | 广播内容 |
| `next` | int unsigned | 下次发送时间 |
| `realmid` | int unsigned | 服务器ID (-1=全部) |

---

### auth 数据库表关系图

```
account (1) ──┬── (N) account_access
              │
              ├── (N) account_banned
              │
              ├── (N) realmcharacters
              │
              └── (N) logs

realmlist (1) ── (N) realmcharacters
```

---

## characters 数据库（角色数据库）

**数据库名称：** `acore_characters`

**作用：** 存储所有玩家角色数据、社交、公会、物品等

### 核心角色表

#### 1. characters - 角色主表

**作用：** 存储所有角色基本信息

**完整字段结构（核心字段）：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色唯一标识符 | 0 |
| `account` | int unsigned | 账户ID | 0 |
| `name` | varchar(12) | 角色名 | - |
| `race` | tinyint unsigned | 种族 | 0 |
| `class` | tinyint unsigned | 职业 | 0 |
| `gender` | tinyint unsigned | 性别 | 0 |
| `level` | tinyint unsigned | 等级 | 0 |
| `xp` | int unsigned | 经验值 | 0 |
| `money` | int unsigned | 金币(铜币) | 0 |
| `skin` | tinyint unsigned | 皮肤 | 0 |
| `face` | tinyint unsigned | 脸型 | 0 |
| `hairStyle` | tinyint unsigned | 发型 | 0 |
| `hairColor` | tinyint unsigned | 发色 | 0 |
| `facialStyle` | tinyint unsigned | 面部特征 | 0 |
| `bankSlots` | tinyint unsigned | 银行槽位数 | 0 |
| `restState` | tinyint unsigned | 休息状态 | 0 |
| `playerFlags` | int unsigned | 玩家标志 | 0 |
| `position_x` | float | X坐标 | 0 |
| `position_y` | float | Y坐标 | 0 |
| `position_z` | float | Z坐标 | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `instance_id` | int unsigned | 副本ID | 0 |
| `orientation` | float | 面向方向 | 0 |
| `taximask` | text | 飞行点掩码 | NULL |
| `online` | tinyint unsigned | 在线状态 | 0 |
| `totaltime` | int unsigned | 总游戏时间 | 0 |
| `leveltime` | int unsigned | 当前等级游戏时间 | 0 |
| `logout_time` | int unsigned | 登出时间 | 0 |
| `rest_bonus` | float | 休息奖励 | 0 |
| `health` | int unsigned | 生命值 | 0 |
| `power1` | int unsigned | 能量1(法力) | 0 |
| `power2` | int unsigned | 能量2(怒气) | 0 |
| `power3` | int unsigned | 能量3(焦点) | 0 |
| `power4` | int unsigned | 能量4(能量) | 0 |
| `power5` | int unsigned | 能量5(符文) | 0 |
| `power6` | int unsigned | 能量6(符文能量) | 0 |
| `power7` | int unsigned | 能量7(灵魂碎片) | 0 |
| `arenaPoints` | int unsigned | 竞技场点数 | 0 |
| `totalHonorPoints` | int unsigned | 总荣誉点数 | 0 |
| `todayKills` | smallint unsigned | 今日击杀 | 0 |
| `totalKills` | int unsigned | 总击杀数 | 0 |
| `chosenTitle` | int unsigned | 选择头衔 | 0 |
| `exploredZones` | longtext | 已探索区域 | NULL |
| `equipmentCache` | longtext | 装备缓存 | NULL |
| `knownTitles` | longtext | 已知头衔 | NULL |
| `creation_date` | timestamp | 创建时间 | CURRENT_TIMESTAMP |

**索引：**
- PRIMARY KEY (`guid`)
- KEY `idx_account` (`account`)
- KEY `idx_online` (`online`)
- KEY `idx_name` (`name`)

**代码引用：**
- `CHAR_SEL_CHARACTER` - 加载角色
- `CHAR_INS_CHARACTER` - 创建角色
- `CHAR_UPD_CHARACTER` - 更新角色
- `CHAR_DEL_CHARACTER` - 删除角色

---

#### 2. character_stats - 角色属性表

**作用：** 存储角色详细属性值

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `guid` | int unsigned | 角色GUID |
| `maxhealth` | int unsigned | 最大生命值 |
| `maxpower1` | int unsigned | 最大法力值 |
| `maxpower2` | int unsigned | 最大怒气 |
| `maxpower3` | int unsigned | 最大焦点 |
| `maxpower4` | int unsigned | 最大能量 |
| `maxpower5` | int unsigned | 最大符文 |
| `maxpower6` | int unsigned | 最大符文能量 |
| `maxpower7` | int unsigned | 最大灵魂碎片 |
| `armor` | int unsigned | 护甲值 |
| `strength` | int unsigned | 力量 |
| `agility` | int unsigned | 敏捷 |
| `stamina` | int unsigned | 耐力 |
| `intellect` | int unsigned | 智力 |
| `spirit` | int unsigned | 精神 |

---

#### 3. character_inventory - 角色背包表

**作用：** 存储角色物品栏和背包内容

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `bag` | int unsigned | 背包GUID | 0 |
| `slot` | tinyint unsigned | 槽位 | 0 |
| `item` | int unsigned | 物品GUID | 0 |

**槽位说明：**
- `slot 0-18`: 装备栏
- `slot 19-22`: 背包1
- `slot 23-26`: 背包2
- `slot 27-30`: 背包3
- `slot 31-34`: 背包4
- `slot 35-63`: 主背包
- `bag=0`: 主背包/装备栏
- `bag>0`: 具体背包ID

**代码引用：** `CHAR_SEL_CHARACTER_INVENTORY`

---

#### 4. item_instance - 物品实例表

**作用：** 存储所有物品的详细数据

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 物品唯一GUID | 0 |
| `itemEntry` | int unsigned | 物品模板ID | 0 |
| `owner_guid` | int unsigned | 所有者GUID | 0 |
| `creatorGuid` | int unsigned | 创建者GUID | 0 |
| `giftCreatorGuid` | int unsigned | 礼物创建者 | 0 |
| `count` | int unsigned | 数量 | 1 |
| `duration` | int | 持续时间 | 0 |
| `charges` | tinytext | 充能次数 | NULL |
| `flags` | int unsigned | 物品标志 | 0 |
| `enchantments` | text | 附魔数据 | '' |
| `randomPropertyId` | smallint | 随机属性ID | 0 |
| `durability` | smallint unsigned | 耐久度 | 0 |
| `playedTime` | int unsigned | 使用时间 | 0 |
| `text` | text | 物品文本 | NULL |

**索引：**
- PRIMARY KEY (`guid`)
- KEY `idx_owner_guid` (`owner_guid`)

---

### 法术和技能表

#### 5. character_spell - 角色法术表

**作用：** 记录角色已学会的法术

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `spell` | int unsigned | 法术ID | 0 |
| `specMask` | tinyint unsigned | 天赋专精掩码 | 1 |

**specMask 说明：**
- `1` (0b00000001): 专精1
- `2` (0b00000010): 专精2

**代码引用：** `CHAR_SEL_CHARACTER_SPELL`

---

#### 6. character_skills - 角色技能表

**作用：** 记录专业技能等级

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `skill` | smallint unsigned | 技能ID | 0 |
| `value` | smallint unsigned | 当前等级 | 0 |
| `max` | smallint unsigned | 最大等级 | 0 |

**代码引用：** `CHAR_SEL_CHARACTER_SKILLS`

---

#### 7. character_aura - 角色光环表

**作用：** 记录角色的持续性效果

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `guid` | int unsigned | 角色GUID |
| `casterGuid` | bigint unsigned | 施法者GUID |
| `itemGuid` | bigint unsigned | 物品GUID |
| `spell` | int unsigned | 法术ID |
| `effectMask` | int unsigned | 效果掩码 |
| `recalculateMask` | int unsigned | 重新计算掩码 |
| `stackAmount` | tinyint unsigned | 堆叠层数 |
| `amount0` | int | 效果1数值 |
| `amount1` | int | 效果2数值 |
| `amount2` | int | 效果3数值 |
| `baseAmount0` | int | 基础数值1 |
| `baseAmount1` | int | 基础数值2 |
| `baseAmount2` | int | 基础数值3 |
| `maxduration` | int | 最大持续时间 |
| `duration` | int | 剩余持续时间 |
| `charges` | tinyint unsigned | 充能次数 |

---

### 任务系统表

#### 8. character_queststatus - 角色任务状态表

**作用：** 记录任务进度

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `quest` | int unsigned | 任务ID | 0 |
| `status` | tinyint unsigned | 状态 | 0 |
| `explored` | tinyint unsigned | 是否探索 | 0 |
| `timer` | int unsigned | 计时器 | 0 |
| `mobcount1` | tinyint unsigned | 杀怪计数1 | 0 |
| `mobcount2` | tinyint unsigned | 杀怪计数2 | 0 |
| `mobcount3` | tinyint unsigned | 杀怪计数3 | 0 |
| `mobcount4` | tinyint unsigned | 杀怪计数4 | 0 |
| `itemcount1` | smallint unsigned | 物品计数1 | 0 |
| `itemcount2` | smallint unsigned | 物品计数2 | 0 |
| `itemcount3` | smallint unsigned | 物品计数3 | 0 |
| `itemcount4` | smallint unsigned | 物品计数4 | 0 |
| `playercount` | smallint unsigned | 玩家计数 | 0 |

**状态说明：**
- `0` - 未接受
- `1` - 进行中
- `2` - 已完成
- `3` - 已失败

**代码引用：**
- `CHAR_SEL_CHARACTER_QUESTSTATUS`
- `CHAR_DEL_QUEST_STATUS_DAILY`
- `CHAR_DEL_QUEST_STATUS_WEEKLY`

---

### 社交系统表

#### 9. character_social - 角色社交表

**作用：** 好友和忽略列表

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `friend` | int unsigned | 好友GUID | 0 |
| `note` | varchar(48) | 备注 | '' |
| `flags` | tinyint unsigned | 标志 | 0 |

**flags 说明：**
- `1` - 好友
- `2` - 忽略

---

### 公会系统表

#### 10. guild - 公会主表

**作用：** 公会基本信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `name` | varchar(24) | 公会名称 | '' |
| `leaderguid` | int unsigned | 会长GUID | 0 |
| `EmblemStyle` | tinyint unsigned | 会徽样式 | 0 |
| `EmblemColor` | tinyint unsigned | 会徽颜色 | 0 |
| `BorderStyle` | tinyint unsigned | 边框样式 | 0 |
| `BorderColor` | tinyint unsigned | 边框颜色 | 0 |
| `BackgroundColor` | tinyint unsigned | 背景颜色 | 0 |
| `info` | varchar(500) | 公会信息 | '' |
| `motd` | varchar(128) | 会明日公告 | '' |
| `createdate` | int unsigned | 创建日期 | 0 |
| `BankMoney` | bigint unsigned | 公会银行金币 | 0 |

---

#### 11. guild_member - 公会成员表

**作用：** 公会成员信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `guid` | int unsigned | 角色GUID | 0 |
| `rank` | tinyint unsigned | 职位等级 | 0 |
| `pnote` | varchar(31) | 个人备注 | '' |
| `offnote` | varchar(31) | 官方备注 | '' |

---

#### 12. guild_rank - 公会职位表

**作用：** 公会职位配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `rid` | tinyint unsigned | 职位ID | 0 |
| `rname` | varchar(16) | 职位名称 | '' |
| `rights` | int unsigned | 权限 | 0 |
| `BankMoneyPerDay` | int unsigned | 每日取款限额 | 0 |

---

### 邮件系统表

#### 13. mail - 邮件主表

**作用：** 邮件系统数据

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 邮件ID | 0 |
| `messageType` | tinyint unsigned | 邮件类型 | 0 |
| `sender` | int unsigned | 发送者GUID | 0 |
| `receiver` | int unsigned | 接收者GUID | 0 |
| `subject` | varchar(255) | 主题 | '' |
| `body` | text | 内容 | NULL |
| `has_items` | tinyint unsigned | 有物品 | 0 |
| `expire_time` | int unsigned | 过期时间 | 0 |
| `deliver_time` | int unsigned | 送达时间 | 0 |
| `money` | int unsigned | 金币 | 0 |
| `cod` | int unsigned | 货到付款 | 0 |
| `checked` | tinyint unsigned | 读取状态 | 0 |

---

### 副本系统表

#### 14. instance - 副本表

**作用：** 副本进度记录

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 实例ID | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `resettime` | int unsigned | 重置时间 | 0 |
| `difficulty` | tinyint unsigned | 难度 | 0 |
| `completedEncounters` | int unsigned | 已完成BOSS | 0 |
| `data` | tinytext | 实例数据 | NULL |

---

#### 15. group_instance - 队伍副本表

**作用：** 队伍副本绑定

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `leaderGuid` | int unsigned | 队长GUID | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `difficulty` | tinyint unsigned | 难度 | 0 |
| `instance` | int unsigned | 实例ID | 0 |

---

### characters 数据库表关系图

```
characters (1) ──┬── (N) character_inventory ──┬── (1) item_instance
                 │                            │
                 ├── (N) character_spell       │
                 │                            │
                 ├── (N) character_skills     │
                 │                            │
                 ├── (N) character_aura       │
                 │                            │
                 ├── (N) character_queststatus│
                 │                            │
                 ├── (N) character_social ────┼── (1) characters (friend)
                 │                            │
                 ├── (N) mail ────────────────┘
                 │
                 ├── (1) guild_member ─── (1) guild
                 │                            │
                 │                            ├── (N) guild_rank
                 │                            └── (N) guild_bank_tab
                 │
                 └── (1) group_member ─── (1) groups
```

---

## world 数据库（世界数据库）

**数据库名称：** `acore_world`

**作用：** 存储所有游戏内容数据，包括NPC、物品、任务、法术等

### 生物系统表

#### 1. creature - 生物实例表

**作用：** 定义世界中的生物及其位置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | bigint unsigned | 生物唯一GUID | 0 |
| `id` | int unsigned | 生物模板ID | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `zoneId` | smallint unsigned | 区域ID | 0 |
| `areaId` | smallint unsigned | 区域ID | 0 |
| `spawnMask` | tinyint unsigned | 生成掩码 | 1 |
| `phaseMask` | int unsigned | 相位掩码 | 1 |
| `equipment_id` | int unsigned | 装备ID | 0 |
| `position_x` | float | X坐标 | 0 |
| `position_y` | float | Y坐标 | 0 |
| `position_z` | float | Z坐标 | 0 |
| `orientation` | float | 面向方向 | 0 |
| `spawntimesecs` | int unsigned | 重生时间(秒) | 0 |
| `wander_distance` | float | 游荡距离 | 0 |
| `currentwaypoint` | int unsigned | 当前路径点 | 0 |
| `curhealth` | int unsigned | 当前生命值 | 0 |
| `curmana` | int unsigned | 当前法力值 | 0 |
| `MovementType` | tinyint unsigned | 移动类型 | 0 |
| `npcflag` | int unsigned | NPC标志 | 0 |
| `unit_flags` | int unsigned | 单位标志 | 0 |
| `dynamicflags` | int unsigned | 动态标志 | 0 |
| `ScriptName` | char(64) | 脚本名称 | '' |
| `VerifiedBuild` | int | 客户端版本 | 0 |

**MovementType 说明：**
- `0` - 不移动
- `1` - 随机游荡
- `2` - 路径移动

---

#### 2. creature_template - 生物模板表

**作用：** 定义生物的属性和外观

**完整字段结构（核心字段）：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `entry` | int unsigned | 模板ID | 0 |
| `difficulty_entry_1` | int unsigned | 10人普通 | 0 |
| `difficulty_entry_2` | int unsigned | 25人普通 | 0 |
| `difficulty_entry_3` | int unsigned | 10人英雄 | 0 |
| `KillCredit1` | int unsigned | 击杀积分1 | 0 |
| `KillCredit2` | int unsigned | 击杀积分2 | 0 |
| `name` | char(100) | 名称 | '0' |
| `subname` | char(100) | 副标题 | NULL |
| `IconName` | char(100) | 图标名称 | NULL |
| `gossip_menu_id` | int unsigned | 对话菜单 | 0 |
| `minlevel` | tinyint unsigned | 最小等级 | 1 |
| `maxlevel` | tinyint unsigned | 最大等级 | 1 |
| `exp` | smallint | 经验倍率 | 0 |
| `faction` | smallint unsigned | 阵营 | 0 |
| `npcflag` | int unsigned | NPC标志 | 0 |
| `speed_walk` | float | 行走速度 | 1 |
| `speed_run` | float | 跑步速度 | 1.14286 |
| `scale` | float | 缩放比例 | 1 |
| `rank` | tinyint unsigned | 等级 | 0 |
| `dmgschool` | tinyint | 伤害类型 | 0 |
| `DamageModifier` | float | 伤害修正 | 1 |
| `BaseAttackTime` | int unsigned | 基础攻击速度 | 0 |
| `RangeAttackTime` | int unsigned | 远程攻击速度 | 0 |
| `unit_class` | tinyint unsigned | 单位类型 | 0 |
| `unit_flags` | int unsigned | 单位标志 | 0 |
| `unit_flags2` | int unsigned | 单位标志2 | 0 |
| `type` | tinyint unsigned | 生物类型 | 0 |
| `type_flags` | int unsigned | 类型标志 | 0 |
| `family` | tinyint | 家族(宠物) | 0 |
| `lootid` | int unsigned | 战利品ID | 0 |
| `HealthModifier` | float | 生命修正 | 1 |
| `ManaModifier` | float | 法力修正 | 1 |
| `ArmorModifier` | float | 护甲修正 | 1 |
| `RacialLeader` | tinyint unsigned | 种族领袖 | 0 |
| `RegenHealth` | tinyint unsigned | 生命再生 | 1 |
| `mechanic_immune_mask` | int unsigned | 免疫机制 | 0 |
| `AIName` | char(64) | AI名称 | '' |
| `ScriptName` | char(64) | 脚本名称 | '' |

**npcflag 说明：**
- `1` - 售货商人
- `2` - 任务发布者
- `4` - 说话
- `16` - 专业训练师
- `128` - 旅店老板
- `4096` - 飞行管理员

---

#### 3. creature_template_addon - 生物扩展表

**作用：** 生物额外的视觉效果和行为

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `entry` | int unsigned | 模板ID | 0 |
| `mount` | int unsigned | 坐骑ID | 0 |
| `bytes1` | int unsigned | 字节1 | 0 |
| `bytes2` | int unsigned | 字节2 | 0 |
| `emote` | int unsigned | 表情 | 0 |
| `visibilityDistanceType` | tinyint unsigned | 可见距离 | 0 |
| `auras` | varchar(500) | 光环列表 | '' |

---

### 物品系统表

#### 4. item_template - 物品模板表

**作用：** 定义所有物品的属性

**完整字段结构（核心字段）：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `entry` | int unsigned | 物品ID | 0 |
| `class` | tinyint unsigned | 物品大类 | 0 |
| `subclass` | tinyint unsigned | 物品小类 | 0 |
| `name` | varchar(255) | 物品名称 | '' |
| `displayid` | int unsigned | 显示ID | 0 |
| `Quality` | tinyint unsigned | 品质 | 0 |
| `Flags` | int unsigned | 标志 | 0 |
| `BuyPrice` | bigint | 购买价格 | 0 |
| `SellPrice` | int | 出售价格 | 0 |
| `InventoryType` | tinyint unsigned | 装备位置 | 0 |
| `AllowableClass` | int | 允许职业 | -1 |
| `AllowableRace` | int | 允许种族 | -1 |
| `ItemLevel` | smallint unsigned | 物品等级 | 0 |
| `RequiredLevel` | tinyint unsigned | 需求等级 | 0 |
| `RequiredSkill` | smallint unsigned | 需求技能 | 0 |
| `RequiredSkillRank` | smallint unsigned | 技能等级 | 0 |
| `maxcount` | int | 最大数量 | 0 |
| `stackable` | int | 可堆叠 | 1 |
| `ContainerSlots` | tinyint unsigned | 容器槽位 | 0 |
| `stat_type1-10` | tinyint unsigned | 属性类型 | 0 |
| `stat_value1-10` | int | 属性值 | 0 |
| `dmg_min1` | float | 最小伤害1 | 0 |
| `dmg_max1` | float | 最大伤害1 | 0 |
| `dmg_type1` | tinyint unsigned | 伤害类型1 | 0 |
| `armor` | int unsigned | 护甲 | 0 |
| `holy_res` | smallint | 神圣抗性 | 0 |
| `fire_res` | smallint | 火焰抗性 | 0 |
| `nature_res` | smallint | 自然抗性 | 0 |
| `frost_res` | smallint | 冰霜抗性 | 0 |
| `shadow_res` | smallint | 暗影抗性 | 0 |
| `arcane_res` | smallint | 奥术抗性 | 0 |
| `delay` | int unsigned | 攻击速度 | 0 |
| `ammotype` | smallint unsigned | 弹药类型 | 0 |
| `RangedModRange` | float | 射程修正 | 0 |
| `spellid_1` | int unsigned | 法术ID1 | 0 |
| `spelltrigger_1` | tinyint unsigned | 触发类型1 | 0 |
| `spellcharges_1` | smallint | 充能次数1 | 0 |
| `spellppmRate_1` | float | 每分钟频率1 | 0 |
| `spellcooldown_1` | int | 冷却时间1 | -1 |
| `spellcategory_1` | int unsigned | 法术类别1 | 0 |
| `spellcategorycooldown_1` | int | 类别冷却1 | -1 |
| `bonding` | tinyint unsigned | 绑定类型 | 0 |
| `description` | varchar(255) | 描述 | '' |
| `PageText` | int unsigned | 页面文本 | 0 |
| `LanguageID` | int unsigned | 语言ID | 0 |
| `PageMaterial` | int unsigned | 页面材质 | 0 |
| `startquest` | int unsigned | 开始任务 | 0 |
| `lockid` | int unsigned | 锁ID | 0 |
| `Material` | int unsigned | 材质 | 0 |
| `Sheath` | tinyint unsigned | 收起位置 | 0 |
| `RandomProperty` | int unsigned | 随机属性 | 0 |
| `RandomSuffix` | int unsigned | 随机后缀 | 0 |
| `itemset` | int unsigned | 套装ID | 0 |
| `MaxDurability` | int unsigned | 最大耐久 | 0 |
| `area` | int unsigned | 区域限制 | 0 |
| `Map` | int unsigned | 地图限制 | 0 |
| `BagFamily` | int unsigned | 背包类型 | 0 |
| `TotemCategory` | int unsigned | 图腾类别 | 0 |
| `socketColor_1` | tinyint unsigned | 插槽颜色1 | 0 |
| `socketContent_1` | int unsigned | 插槽内容1 | 0 |
| `socketBonus` | int unsigned | 插槽奖励 | 0 |
| `GemProperties` | int unsigned | 宝石属性 | 0 |
| `RequiredDisenchantSkill` | int unsigned | 需求附魔 | 0 |
| `ArmorDamageModifier` | float | 护甲伤害修正 | 0 |
| `duration` | int unsigned | 持续时间 | 0 |
| `ItemLimitCategoryId` | int unsigned | 限制类别 | 0 |
| `HolidayId` | int unsigned | 节日ID | 0 |

**物品品质 Quality：**
- `0` - 灰色（垃圾）
- `1` - 白色（普通）
- `2` - 绿色（优秀）
- `3` - 蓝色（精良）
- `4` - 紫色（史诗）
- `5` - 橙色（传说）
- `6` - 红色（神器）

---

### 任务系统表

#### 5. quest_template - 任务模板表

**作用：** 定义任务属性和奖励

**完整字段结构（核心字段）：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ID` | int unsigned | 任务ID | 0 |
| `QuestType` | tinyint unsigned | 任务类型 | 2 |
| `QuestLevel` | smallint | 任务等级 | 1 |
| `MinLevel` | tinyint unsigned | 最低等级 | 0 |
| `QuestSortID` | smallint | 排序ID | 0 |
| `QuestInfoID` | smallint unsigned | 任务分类 | 0 |
| `SuggestedGroupNum` | tinyint unsigned | 建议人数 | 0 |
| `RequiredFactionId1` | smallint unsigned | 需求阵营1 | 0 |
| `RequiredFactionId2` | smallint unsigned | 需求阵营2 | 0 |
| `RequiredFactionValue1` | int | 需求声望1 | 0 |
| `RequiredFactionValue2` | int | 需求声望2 | 0 |
| `RewardXPDifficulty` | tinyint unsigned | 经验奖励等级 | 0 |
| `RewardMoney` | int | 金币奖励 | 0 |
| `RewardMoneyDifficulty` | int unsigned | 金币奖励等级 | 0 |
| `RewardDisplaySpell` | int unsigned | 显示法术 | 0 |
| `RewardSpell` | int | 法术奖励 | 0 |
| `RewardHonor` | int | 荣誉奖励 | 0 |
| `RewardKillHonor` | float | 击杀荣誉 | 0 |
| `StartItem` | int unsigned | 开始物品 | 0 |
| `Flags` | int unsigned | 标志 | 0 |
| `RequiredPlayerKills` | tinyint unsigned | 需求击杀 | 0 |
| `RewardItem1-4` | int unsigned | 物品奖励 | 0 |
| `RewardAmount1-4` | smallint unsigned | 物品数量 | 0 |
| `RewardChoiceItemID1-6` | int unsigned | 选择物品 | 0 |
| `RewardChoiceItemQuantity1-6` | smallint unsigned | 选择数量 | 0 |
| `RewardTitle` | tinyint unsigned | 头衔奖励 | 0 |
| `RewardTalents` | tinyint unsigned | 天赋点奖励 | 0 |
| `RewardArenaPoints` | smallint unsigned | 竞技场点数 | 0 |
| `ReqCreatureOrGOId1-4` | int unsigned | 需求生物/GO | 0 |
| `ReqCreatureOrGOCount1-4` | smallint unsigned | 需求数量 | 0 |
| `ReqSpellCast1-4` | int unsigned | 需求施法 | 0 |
| `RewChoiceItemId1-6` | int unsigned | 选择物品ID | 0 |
| `RewChoiceItemCount1-6` | smallint unsigned | 选择数量 | 0 |
| `RewItemId1-4` | int unsigned | 物品奖励ID | 0 |
| `RewItemCount1-4` | smallint unsigned | 物品奖励数量 | 0 |
| `RewRepFaction1-5` | smallint unsigned | 声望阵营 | 0 |
| `RewRepValue1-5` | int | 声望值 | 0 |
| `RewRepLimit1-5` | int | 声望上限 | 0 |
| `QuestDescription` | text | 任务描述 | NULL |
| `LogDescription` | text | 日志描述 | NULL |
| `QuestCompletionLog` | text | 完成文本 | NULL |
| `RequiredItemId1-6` | int unsigned | 需求物品ID | 0 |
| `RequiredItemCount1-6` | smallint unsigned | 需求物品数量 | 0 |
| `SourceSpellId` | int unsigned | 源法术ID | 0 |
| `PrevQuestId` | int unsigned | 前置任务 | 0 |
| `NextQuestId` | int unsigned | 后续任务 | 0 |
| `ExclusiveGroup` | int unsigned | 互斥组 | 0 |
| `NextQuestInChain` | int unsigned | 链中下一个任务 | 0 |
| `AllowableRaces` | int | 允许种族 | -1 |
| `SoundAccept` | int unsigned | 接受音效 | 0 |
| `SoundTurnIn` | int unsigned | 提交音效 | 0 |

---

#### 6. creature_queststarter - 任务起始者表

**作用：** 定义哪些NPC给予哪些任务

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 生物模板ID | 0 |
| `quest` | int unsigned | 任务ID | 0 |

---

#### 7. creature_questender - 任务结束者表

**作用：** 定义哪些NPC接受哪些任务

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 生物模板ID | 0 |
| `quest` | int unsigned | 任务ID | 0 |

---

### 游戏对象表

#### 8. gameobject - 游戏对象实例表

**作用：** 定义世界中的物体（门、箱子、矿点等）

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | bigint unsigned | 对象GUID | 0 |
| `id` | int unsigned | 对象模板ID | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `zoneId` | smallint unsigned | 区域ID | 0 |
| `areaId` | smallint unsigned | 区域ID | 0 |
| `spawnMask` | tinyint unsigned | 生成掩码 | 1 |
| `phaseMask` | int unsigned | 相位掩码 | 1 |
| `position_x` | float | X坐标 | 0 |
| `position_y` | float | Y坐标 | 0 |
| `position_z` | float | Z坐标 | 0 |
| `orientation` | float | 面向方向 | 0 |
| `rotation0` | float | 旋转0 | 0 |
| `rotation1` | float | 旋转1 | 0 |
| `rotation2` | float | 旋转2 | 0 |
| `rotation3` | float | 旋转3 | 0 |
| `spawntimesecs` | int unsigned | 重生时间 | 0 |
| `animprogress` | int unsigned | 动画进度 | 0 |
| `state` | tinyint unsigned | 状态 | 0 |
| `ScriptName` | char(64) | 脚本名称 | '' |

---

### SmartAI 脚本表

#### 9. smart_scripts - 聪能脚本表

**作用：** 数据库驱动的AI行为配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `entryorguid` | int | 条目ID或GUID | 0 |
| `source_type` | tinyint unsigned | 源类型 | 0 |
| `id` | smallint unsigned | 脚本ID | 0 |
| `link` | smallint unsigned | 链接ID | 0 |
| `event_type` | tinyint unsigned | 事件类型 | 0 |
| `event_phase_mask` | smallint unsigned | 阶段掩码 | 0 |
| `event_chance` | tinyint unsigned | 触发概率 | 100 |
| `event_flags` | smallint unsigned | 事件标志 | 0 |
| `event_param1` | int unsigned | 事件参数1 | 0 |
| `event_param2` | int unsigned | 事件参数2 | 0 |
| `event_param3` | int unsigned | 事件参数3 | 0 |
| `event_param4` | int unsigned | 事件参数4 | 0 |
| `event_param5` | int unsigned | 事件参数5 | 0 |
| `event_param6` | int unsigned | 事件参数6 | 0 |
| `action_type` | tinyint unsigned | 动作类型 | 0 |
| `action_param1` | int unsigned | 动作参数1 | 0 |
| `action_param2` | int unsigned | 动作参数2 | 0 |
| `action_param3` | int unsigned | 动作参数3 | 0 |
| `action_param4` | int unsigned | 动作参数4 | 0 |
| `action_param5` | int unsigned | 动作参数5 | 0 |
| `action_param6` | int unsigned | 动作参数6 | 0 |
| `target_type` | tinyint unsigned | 目标类型 | 0 |
| `target_param1` | int unsigned | 目标参数1 | 0 |
| `target_param2` | int unsigned | 目标参数2 | 0 |
| `target_param3` | int unsigned | 目标参数3 | 0 |
| `target_param4` | int unsigned | 目标参数4 | 0 |
| `target_x` | float | 目标X坐标 | 0 |
| `target_y` | float | 目标Y坐标 | 0 |
| `target_z` | float | 目标Z坐标 | 0 |
| `target_o` | float | 目标面向 | 0 |
| `comment` | text | 注释 | '' |

**source_type 说明：**
- `0` - 生物
- `1` - 游戏对象
- `2` - 物品
- `9` - 玩家

**事件类型 event_type 说明：**
- `0` - UPDATE
- `1` - HP
- `2` - MANA
- `3` - AGGRO
- `4` - DEATH
- `6` - KILL
- `10` - SPELLHIT
- `19` - OOC_LOS

**动作类型 action_type 说明：**
- `1` - TALK
- `11` - CAST
- `12` - SUMMON
- `15` - ADD_AURA
- `20` - SET_FACTION
- `75` - ATTACK_START

---

### 战利品表

#### 10. creature_loot_template - 生物战利品表

**作用：** 定义生物死亡后的掉落

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `Entry` | int unsigned | 生物模板ID | 0 |
| `Item` | int unsigned | 物品ID | 0 |
| `Reference` | int unsigned | 引用ID | 0 |
| `Chance` | float | 掉落概率 | 100 |
| `QuestRequired` | tinyint unsigned | 需求任务 | 0 |
| `LootMode` | tinyint unsigned | 掉落模式 | 1 |
| `GroupId` | tinyint unsigned | 分组ID | 0 |
| `MinCount` | tinyint unsigned | 最小数量 | 1 |
| `MaxCount` | tinyint unsigned | 最大数量 | 1 |
| `Comment` | varchar(255) | 注释 | '' |

---

### 游戏事件表

#### 11. game_event - 游戏事件表

**作用：** 世界事件配置（节日、活动）

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `eventEntry` | int unsigned | 事件ID | 0 |
| `start_time` | timestamp | 开始时间 | '0000-00-00' |
| `end_time` | timestamp | 结束时间 | '0000-00-00' |
| `occurence` | bigint | 重复间隔 | 0 |
| `length` | int unsigned | 持续时长(分钟) | 0 |
| `holiday` | int unsigned | 节日ID | 0 |
| `holidayStage` | tinyint unsigned | 节日阶段 | 0 |
| `description` | varchar(255) | 描述 | '' |
| `world_event` | tinyint unsigned | 世界事件 | 0 |
| `announce` | tinyint unsigned | 公告 | 0 |

---

### 其他重要表

#### 12. command - GM命令表

**作用：** 定义GM命令权限

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 命令ID | 0 |
| `name` | varchar(50) | 命令名称 | '' |
| `permission` | tinyint unsigned | 需要权限 | 0 |
| `help` | text | 帮助文本 | NULL |

---

#### 13. conditions - 条件表

**作用：** 定义各种条件检查

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `SourceTypeOrReferenceId` | int unsigned | 源类型 | 0 |
| `SourceGroup` | int unsigned | 源组 | 0 |
| `SourceEntry` | int unsigned | 源条目 | 0 |
| `SourceId` | int unsigned | 源ID | 0 |
| `ElseGroup` | int unsigned | 否则组 | 0 |
| `ConditionTypeOrReference` | int unsigned | 条件类型 | 0 |
| `ConditionTarget` | tinyint unsigned | 条件目标 | 0 |
| `ConditionValue1` | int unsigned | 条件值1 | 0 |
| `ConditionValue2` | int unsigned | 条件值2 | 0 |
| `ConditionValue3` | int unsigned | 条件值3 | 0 |
| `NegativeCondition` | tinyint unsigned | 反向条件 | 0 |
| `ErrorType` | int unsigned | 错误类型 | 0 |
| `ErrorTextId` | int unsigned | 错误文本 | 0 |
| `ScriptName` | varchar(255) | 脚本名称 | '' |
| `Comment` | varchar(255) | 注释 | '' |

---

### world 数据库表关系图

```
creature_template (1) ──┬── (N) creature
                        │
                        ├── (N) creature_loot_template ──┬── (1) item_template
                        │                                │
                        ├── (N) creature_queststarter ──┼── (1) quest_template
                        │                                │
                        ├── (N) creature_questender ────┘
                        │
                        └── (N) smart_scripts

gameobject_template (1) ── (N) gameobject

item_template (1) ──┬── (1) item_enchantment_template
                   │
                   ├── (N) item_loot_template
                   │
                   └── (N) npc_vendor

quest_template (1) ──┬── (N) quest_request_items
                     │
                     ├── (N) quest_offer_reward
                     │
                     ├── (N) quest_details
                     │
                     └── (N) game_event_quest
```

---

## 数据库连接池架构

### DatabaseWorkerPool 模板类

**位置：** `src/server/database/Database/DatabaseWorkerPool.h`

```cpp
template <class T>
class DatabaseWorkerPool
{
private:
    enum InternalIndex
    {
        IDX_ASYNC,  // 异步操作线程池
        IDX_SYNCH,  // 同步操作线程池
        IDX_SIZE
    };

    MySQLConnection* m_connections[IDX_SIZE];  // 连接池
    ProducerConsumerQueue<SQLOperation*>* m_queues[IDX_SIZE];  // 工作队列
    uint8 m_workerThreads[IDX_SIZE];  // 工作线程数量
```

### 三种数据库连接池

```cpp
// LoginDatabase - auth 数据库
typedef DatabaseWorkerPool<LoginDatabaseConnection> LoginDatabase;

// CharacterDatabase - characters 数据库
typedef DatabaseWorkerPool<CharacterDatabaseConnection> CharacterDatabase;

// WorldDatabase - world 数据库
typedef DatabaseWorkerPool<WorldDatabaseConnection> WorldDatabase;
```

### 数据库操作模式

#### 1. 异步操作（非阻塞）

```cpp
// 用于一次性操作，不影响主线程
LoginDatabase.Execute("UPDATE account SET online = 0");
CharacterDatabase.Execute("DELETE FROM mail WHERE expire_time < UNIX_TIMESTAMP()");
```

#### 2. 同步操作（阻塞）

```cpp
// 查询操作，需要立即获取结果
QueryResult result = CharacterDatabase.Query("SELECT name FROM characters");
if (result)
{
    do
    {
        Field* fields = result->Fetch();
        std::string name = fields[0].Get<std::string>();
    } while (result->NextRow());
}
```

#### 3. 预处理语句（推荐）

```cpp
// 性能更高，防止SQL注入
PreparedStatement* stmt = LoginDatabase.GetPreparedStatement(LOGIN_SEL_ACCOUNT_BY_ID);
stmt->setInt32(0, accountId);
PreparedQueryResult result = LoginDatabase.Query(stmt);
```

---

## 表关系和依赖

### 核心关系图

```
account (auth) ── (1:N) ── characters ── (1:1) ── character_stats
                                    │
                                    ├── (1:N) ── character_inventory ── (N:1) ── item_instance
                                    │
                                    ├── (1:N) ── character_spell
                                    │
                                    ├── (1:N) ── character_queststatus ── (N:1) ── quest_template (world)
                                    │
                                    ├── (1:1) ── guild_member ── (N:1) ── guild
                                    │
                                    └── (1:N) ── mail

creature_template (world) ── (1:N) ── creature
                                    │
                                    ├── (1:N) ── creature_loot_template ── (N:1) ── item_template
                                    │
                                    ├── (1:N) ── creature_queststarter ── (N:1) ── quest_template
                                    │
                                    └── (1:N) ── smart_scripts

item_template (world) ── (1:N) ── item_instance (characters)
```

### 外键关系（逻辑）

虽然没有显式的外键约束，但以下关系在代码中维护：

| 从表 | 从表字段 | 主表 | 主表字段 |
|------|----------|------|----------|
| characters | account | account | id |
| character_inventory | guid | characters | guid |
| character_inventory | item | item_instance | guid |
| item_instance | owner_guid | characters | guid |
| character_spell | guid | characters | guid |
| guild_member | guildid | guild | guildid |
| guild_member | guid | characters | guid |
| mail | receiver | characters | guid |
| mail | sender | characters | guid |
| creature | id | creature_template | entry |
| gameobject | id | gameobject_template | entry |
| character_queststatus | quest | quest_template | ID |

---

## 索引和性能优化

### 索引策略

#### auth 数据库

```sql
-- account 表
PRIMARY KEY (id)
UNIQUE KEY idx_username (username)

-- realmlist 表
PRIMARY KEY (id)
UNIQUE KEY idx_name (name)

-- account_banned 表
KEY idx_account_id (id)
```

#### characters 数据库

```sql
-- characters 表
PRIMARY KEY (guid)
KEY idx_account (account)
KEY idx_online (online)
KEY idx_name (name)

-- character_inventory 表
PRIMARY KEY (item)
UNIQUE KEY guid (guid, bag, slot)
KEY idx_guid (guid)

-- item_instance 表
PRIMARY KEY (guid)
KEY idx_owner_guid (owner_guid)

-- guild 表
PRIMARY KEY (guildid)
```

#### world 数据库

```sql
-- creature 表
PRIMARY KEY (guid)
KEY idx_id (id)
KEY idx_map (map)

-- creature_template 表
PRIMARY KEY (entry)

-- item_template 表
PRIMARY KEY (entry)

-- quest_template 表
PRIMARY KEY (ID)

-- smart_scripts 表
PRIMARY KEY (entryorguid, source_type, id, link)
```

### 性能优化技巧

1. **预处理语句缓存**：预编译语句被缓存并复用
2. **批量操作**：使用事务批量提交减少数据库往返
3. **异步查询**：非关键数据使用异步操作
4. **连接池**：复用数据库连接避免频繁建立
5. **读写分离**：支持主从复制（代码层面支持）

---

## 附录：数据库更新系统

### 更新文件命名规范

```
{YYYY}_{MM}_{DD}_{HH}.sql

示例：
2026_01_23_01.sql
```

### 更新流程

1. 将 SQL 文件放入 `data/sql/updates/pending_db_*/`
2. 服务器启动时自动检测并应用未执行的更新
3. 更新记录在 `updates` 表中标记
4. 已应用的更新移动到 `archive/`

### 更新表结构

```sql
CREATE TABLE `updates` (
    `name` varchar(200) NOT NULL COMMENT 'Filename',
    `hash` char(40) DEFAULT NULL COMMENT 'SHA-1 hash',
    `state` enum('RELEASED','ARCHIVED') NOT NULL DEFAULT 'RELEASED',
    `timestamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `speed` int unsigned DEFAULT NULL COMMENT 'Update application speed in ms',
    PRIMARY KEY (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='Database update tracking';
```

---

### 其他重要表

#### 14. game_graveyard - 墓地/复活点表

**作用：** 定义所有地图的玩家复活点位置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ID` | int | 墓地唯一ID | 0 |
| `Map` | int | 地图ID | 0 |
| `x` | float | X坐标 | 0 |
| `y` | float | Y坐标 | 0 |
| `z` | float | Z坐标 | 0 |
| `Comment` | varchar(255) | 注释/位置名 | NULL |

**示例数据：**
```
(1,0,-9115,423,96,'Stormwind')
(33,1,1357.1,-4412.01,28.3841,'Orgrimmar')
(90,1,10054.3,2117.12,1329.63,'Darnassus')
(102,0,-5049.45,-809.697,495.127,'Ironforge')
(96,0,1822.61,214.674,60.1402,'Undercity')
```

---

#### 15. game_tele - 传送点表

**作用：** GM命令和游戏内传送点的位置定义

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 传送点ID | 0 |
| `position_x` | float | X坐标 | 0 |
| `position_y` | float | Y坐标 | 0 |
| `position_z` | float | Z坐标 | 0 |
| `orientation` | float | 面向方向 | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `name` | varchar(100) | 传送点名称 | '' |

**用途：** GM `.tele` 命令使用

---

#### 16. npc_vendor - 商人出售表

**作用：** 定义NPC商人出售的物品

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `entry` | int unsigned | 生物模板ID | 0 |
| `item` | int unsigned | 物品ID | 0 |
| `maxcount` | smallint unsigned | 最大库存数量 | 0 |
| `incrtime` | int unsigned | 补货时间(秒) | 0 |
| `ExtendedCost` | int unsigned | 扩展成本(英雄点/竞技场点数) | 0 |

**maxcount 说明：**
- `0` - 无限数量
- `>0` - 有限数量，售完后需等待 incrtime 秒补货

---

#### 17. battleground_template - 战场模板表

**作用：** 战场配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ID` | int unsigned | 战场ID | 0 |
| `MinPlayers` | smallint unsigned | 最少玩家 | 0 |
| `MaxPlayers` | smallint unsigned | 最多玩家 | 0 |
| `MinLvl` | tinyint unsigned | 最低等级 | 0 |
| `MaxLvl` | tinyint unsigned | 最高等级 | 0 |
| `AllianceStartLoc` | int unsigned | 联盟起始位置 | 0 |
| `AllianceStartO` | float | 联盟起始面向 | 0 |
| `HordeStartLoc` | int unsigned | 部落起始位置 | 0 |
| `HordeStartO` | float | 部落起始面向 | 0 |

**战场类型：**
- `1` - 战歌峡谷
- `2` - 阿拉希盆地
- `3` - 伐木场
- `30` - 奥特兰克山谷
- `489` - 战歌峡谷
- `529` - 阿拉希盆地
- `566` - 刀锋山竞技场
- `607` - 远征滩
- `628` - 寒冰皇冠

---

#### 18. areatrigger_teleport - 区域传送表

**作用：** 定义进入区域后的传送效果

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 触发器ID | 0 |
| `name` | char(100) | 名称 | '' |
| `target_map` | int unsigned | 目标地图 | 0 |
| `target_position_x` | float | 目标X | 0 |
| `target_position_y` | float | 目标Y | 0 |
| `target_position_z` | float | 目标Z | 0 |
| `target_orientation` | float | 目标面向 | 0 |

---

#### 19. spell_dbc - 法术表

**作用：** 存储所有法术的基础数据（DBC格式）

**完整字段结构（部分）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `Id` | int unsigned | 法术ID |
| `Category` | int unsigned | 法术类别 |
| `Dispel` | int unsigned | 驱散类型 |
| `Mechanic` | int unsigned | 机制类型 |
| `Attributes` | int unsigned | 法术属性 |
| `SpellEffects` | text | 法术效果列表 |
| `Duration` | int | 持续时间 |
| `PowerType` | tinyint unsigned | 能量类型 |
| `ManaCost` | int unsigned | 消耗法力 |
| `Range` | int unsigned | 施法范围 |

---

#### 20. faction_dbc - 阵营表

**作用：** 游戏阵营定义

**完整字段结构：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `ID` | int unsigned | 阵营ID |
| `Name` | varchar(100) | 阵营名称 |
| `Description` | varchar(200) | 阵营描述 |
| `ReputationRaceMask` | int | 种族掩码 |
| `ReputationClassMask` | int | 职业掩码 |
| `ReputationBase` | int | 初始声望值 |
| `ParentFactionID` | int | 父阵营ID |

**重要阵营ID：**
- `72` - 暴风城
- `76` - 铁炉堡
- `69` - 达纳苏斯
- `530` - 诺莫瑞根
- `577` - 联盟
- `68` - 幽暗城
- `529` - 部落
- `911` - 达拉然

---

#### 21. achievement_reward - 成就奖励表

**作用：** 成就完成后的奖励

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ID` | int unsigned | 成就ID | 0 |
| `TitleA` | int unsigned | 联盟头衔 | 0 |
| `TitleH` | int unsigned | 部落头衔 | 0 |
| `Item` | int unsigned | 物品奖励 | 0 |
| `Sender` | int unsigned | 邮件发送者 | 0 |
| `Subject` | varchar(255) | 邮件主题 | '' |
| `Body` | text | 邮件内容 | '' |
| `MailTemplateId` | int unsigned | 邮件模板 | 0 |

---

#### 22. command - GM命令表

**作用：** 定义GM命令及其权限要求

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 命令ID | 0 |
| `name` | varchar(50) | 命令名称 | '' |
| `permission` | tinyint unsigned | 需要权限 | 0 |
| `help` | text | 帮助文本 | NULL |

**常用命令：**
| 命令 | 权限 | 说明 |
|------|------|------|
| `.announce` | 1 | 发送系统公告 |
| `.appear` | 2 | 传送到玩家 |
| `.summon` | 2 | 召唤玩家 |
| `.go` | 2 | 传送 |
| `.gps` | 2 | 显示位置信息 |
| `.modify` | 3 | 修改角色属性 |
| `.reload` | 3 | 重载表格 |
| `.debug` | 3 | 调试命令 |
| `.server` | 4 | 服务器管理 |

---

#### 23. disables - 禁用表

**作用：** 禁用特定内容（法术、地图、生物等）

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `sourceType` | tinyint unsigned | 源类型 | 0 |
| `entry` | int unsigned | 条目ID | 0 |
| `flags` | smallint unsigned | 标志 | 0 |
| `comment` | varchar(255) | 注释 | '' |

**sourceType 说明：**
- `0` - 禁用法术
- `1` - 禁用地图
- `2` - 禁用生物
- `3` - 禁用游戏对象
- `4` - 禁用物品
- `7` - 禁用任务
- `8` - 禁用区域

---

#### 24. creature_classlevelstats - 生物等级属性表

**作用：** 定义不同等级生物的基础属性

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `level` | tinyint unsigned | 等级 | 1 |
| `class` | tinyint unsigned | 单位类型 | 0 |
| `basehp0` | int unsigned | 基础生命值0 | 1 |
| `basehp1` | int unsigned | 基础生命值1 | 1 |
| `basemana` | int unsigned | 基础法力值 | 0 |
| `basearmor` | int unsigned | 基础护甲 | 0 |
| `attackpower` | int unsigned | 攻击强度 | 0 |
| `rangedattackpower` | int unsigned | 远程攻击强度 | 0 |

**class 说明：**
- `0` - 无效
- `1` - 世界生物
- `2` - 末日活动
- `4` - 精英
- `8` - 稀有

---

### characters 数据库补充表

#### 邮件系统（续）

#### mail_items - 邮件物品表

**作用：** 邮件中附带的物品

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `mail_id` | int unsigned | 邮件ID | 0 |
| `item_guid` | int unsigned | 物品GUID | 0 |
| `item_template` | int unsigned | 物品模板ID | 0 |

---

#### 拍卖行系统

#### auctionhouse - 拍卖行表

**作用：** 拍卖物品数据

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 拍卖ID | 0 |
| `houseid` | tinyint unsigned | 拍卖行ID | 7 |
| `itemguid` | int unsigned | 物品GUID | 0 |
| `itemowner` | int unsigned | 物品所有者 | 0 |
| `buyoutprice` | int unsigned | 一口价 | 0 |
| `time` | int unsigned | 剩余时间 | 0 |
| `buyguid` | int unsigned | 竞拍者 | 0 |
| `lastbid` | int unsigned | 当前最高出价 | 0 |
| `startbid` | int unsigned | 起始价 | 0 |
| `deposit` | int unsigned | 押金 | 0 |

**houseid 说明：**
- `2` - 联盟拍卖行
- `6` - 部落拍卖行
- `7` - 中立拍卖行

---

#### 竞技场系统

#### arena_team - 竞技队列表

**作用：** 竞技场队伍信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `arenaTeamId` | int unsigned | 竞技队ID | 0 |
| `name` | varchar(24) | 队伍名称 | '' |
| `captainGuid` | int unsigned | 队长GUID | 0 |
| `type` | tinyint unsigned | 队伍类型 | 0 |
| `rating` | smallint unsigned | 等级分 | 0 |
| `seasonGames` | smallint unsigned | 赛季场数 | 0 |
| `seasonWins` | smallint unsigned | 赛季胜场 | 0 |
| `weekGames` | smallint unsigned | 周场数 | 0 |
| `weekWins` | smallint unsigned | 周胜场 | 0 |
| `rank` | int unsigned | 排名 | 0 |
| `backgroundColor` | int unsigned | 背景色 | 0 |
| `emblemStyle` | tinyint unsigned | 徽章样式 | 0 |
| `emblemColor` | int unsigned | 徽章颜色 | 0 |
| `borderStyle` | tinyint unsigned | 边框样式 | 0 |
| `borderColor` | int unsigned | 边框颜色 | 0 |

**type 说明：**
- `2` - 2v2 竞技场
- `3` - 3v3 竞技场
- `5` - 5v5 竞技场

---

#### arena_team_member - 竞技队成员表

**作用：** 竞技队成员信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `arenaTeamId` | int unsigned | 竞技队ID | 0 |
| `guid` | int unsigned | 角色GUID | 0 |
| `weekGames` | smallint unsigned | 周场数 | 0 |
| `weekWins` | smallint unsigned | 周胜场 | 0 |
| `seasonGames` | smallint unsigned | 赛季场数 | 0 |
| `seasonWins` | smallint unsigned | 赛季胜场 | 0 |
| `personalRating` | smallint unsigned | 个人等级分 | 0 |

---

#### 队伍系统

#### groups - 队伍列表

**作用：** 队伍信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 队伍GUID | 0 |
| `leaderGuid` | int unsigned | 队长GUID | 0 |
| `lootMethod` | tinyint unsigned | 分配方式 | 0 |
| `looterGuid` | int unsigned | 分配者 | 0 |
| `lootThreshold` | tinyint unsigned | 分配阈值 | 0 |
| `icon1` | bigint unsigned | 目标标记1 | 0 |
| `icon2-8` | bigint unsigned | 目标标记2-8 | 0 |
| `groupType` | tinyint unsigned | 队伍类型 | 0 |
| `difficulty` | tinyint unsigned | 地下城难度 | 0 |
| `raidDifficulty` | tinyint unsigned | 团队副本难度 | 0 |
| `masterLooterGuid` | int unsigned | 分配者 | 0 |

**lootMethod 说明：**
- `0` - 自由拾取
- `1` - round robin (轮流)
- `2` - master looter (队长分配)
- `3` - group loot (队伍分配)
- `4` - need before greed (需求优先)

---

#### group_member - 队伍成员表

**作用：** 队伍成员信息

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 队伍GUID | 0 |
| `memberGuid` | int unsigned | 成员GUID | 0 |
| `flags` | tinyint unsigned | 标志 | 0 |
| `subgroup` | tinyint unsigned | 子组 | 0 |
| `roles` | tinyint unsigned | 角色 | 0 |

---

#### 宠物系统

#### character_pet - 猎人/术士宠物表

**作用：** 宠物数据

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `id` | int unsigned | 宠物唯一ID | 0 |
| `entry` | int unsigned | 生物模板ID | 0 |
| `owner` | int unsigned | 所有者GUID | 0 |
| `modelid` | int unsigned | 模型ID | 0 |
| `CreatedBySpell` | int unsigned | 创建法术 | 0 |
| `PetType` | tinyint unsigned | 宠物类型 | 0 |
| `level` | smallint unsigned | 等级 | 1 |
| `exp` | int unsigned | 经验值 | 0 |
| `Reactstate` | tinyint unsigned | 反应状态 | 0 |
| `name` | varchar(21) | 宠物名称 | 'Pet' |
| `renamed` | tinyint unsigned | 是否已改名 | 0 |
| `slot` | tinyint unsigned | 槽位 | 0 |
| `curhealth` | int unsigned | 当前生命值 | 1 |
| `curmana` | int unsigned | 当前法力值 | 0 |
| `curhappiness` | int unsigned | 当前快乐度 | 0 |
| `savetime` | int unsigned | 保存时间 | 0 |
| `abdata` | text | 法术数据 | NULL |

**PetType 说明：**
- `0` - 猎人宠物
- `1` - 术士宠物
- `2` - 死亡骑士食尸鬼
- `3` - 巫师法师的水元素

**Reactstate 说明：**
- `0` - 被动
- `1` - 防御
- `2` - 主动

---

#### 成就系统

#### character_achievement - 角色成就表

**作用：** 角色已完成的成就

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `achievement` | smallint unsigned | 成就ID | 0 |
| `date` | int unsigned | 完成时间 | 0 |

---

#### character_achievement_progress - 成就进度表

**作用：** 成就完成进度

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `criteria` | int unsigned | 条件ID | 0 |
| `counter` | int unsigned | 计数器 | 0 |
| `date` | int unsigned | 完成时间 | 0 |

---

#### 公会银行系统

#### guild_bank_tab - 公会银行标签

**作用：** 公会银行标签配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `TabId` | tinyint unsigned | 标签ID | 0 |
| `TabName` | varchar(16) | 标签名称 | '' |
| `TabIcon` | varchar(100) | 标签图标 | '' |
| `TabText` | varchar(500) | 标签描述 | '' |

---

#### guild_bank_item - 公会银行物品

**作用：** 公会银行中的物品

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `TabId` | tinyint unsigned | 标签ID | 0 |
| `ItemId` | int unsigned | 物品ID | 0 |
| `itemGuid` | int unsigned | 物品GUID | 0 |

---

#### guild_bank_right - 公会银行权限

**作用：** 公会银行标签权限配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guildid` | int unsigned | 公会ID | 0 |
| `TabId` | tinyint unsigned | 标签ID | 0 |
| `rid` | tinyint unsigned | 职位ID | 0 |
| `gbright` | tinyint unsigned | 权限 | 0 |
| `SlotPerDay` | int unsigned | 每日操作次数 | 0 |

---

#### 副本系统（续）

#### character_instance - 角色副本绑定

**作用：** 角色与副本的绑定关系

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `instance` | int unsigned | 副本ID | 0 |
| `permanent` | tinyint unsigned | 永久绑定 | 0 |

---

#### instance_reset - 副本重置

**作用：** 副本重置时间记录

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `mapid` | smallint unsigned | 地图ID | 0 |
| `difficulty` | tinyint unsigned | 难度 | 0 |
| `resettime` | int unsigned | 重置时间 | 0 |
| `instanceid` | int unsigned | 实例ID | 0 |

---

#### 其他系统表

#### character_glyphs - 角色雕文表

**作用：** 角色装备的雕文

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `talentGroup` | tinyint unsigned | 天赋组 | 0 |
| `glyphId` | smallint unsigned | 雕文ID | 0 |

---

#### character_talent - 角色天赋表

**作用：** 角色天赋配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `talentGroup` | tinyint unsigned | 天赋组 | 0 |
| `talentId` | smallint unsigned | 天赋ID | 0 |
| `rank` | smallint unsigned | 等级 | 0 |

---

#### character_action - 角色动作条表

**作用：** 动作栏技能配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 角色GUID | 0 |
| `spec` | tinyint unsigned | 天赋专精 | 0 |
| `button` | int unsigned | 按钮位置 | 0 |
| `action` | int unsigned | 动作类型 | 0 |
| `type` | tinyint unsigned | 类型 | 0 |

---

#### channels - 频道表

**作用：** 自定义频道配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `m_name` | varchar(128) | 频道名称 | '' |
| `m_team` | int unsigned | 阵营 | 0 |
| `m_announce` | tinyint unsigned | 公告 | 0 |
| `m_moderate` | tinyint unsigned | 监管 | 0 |
| `m_password` | varchar(32) | 密码 | '' |

---

#### gm_ticket - GM工单表

**作用：** 玩家提交的GM工单

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ticketId` | int unsigned | 工单ID | 0 |
| `guid` | int unsigned | 角色GUID | 0 |
| `name` | varchar(12) | 角色名 | '' |
| `createtime` | int unsigned | 创建时间 | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `posX` | float | X坐标 | 0 |
| `posY` | float | Y坐标 | 0 |
| `posZ` | float | Z坐标 | 0 |
| `message` | text | 工单内容 | NULL |
| `createTime` | int unsigned | 创建时间 | 0 |
| `completed` | tinyint unsigned | 已完成 | 0 |
| `assignedTo` | int unsigned | 分配给 | 0 |
| `comment` | text | 备注 | NULL |

---

#### corpse - 尸体表

**作用：** 玩家尸体位置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `guid` | int unsigned | 尸体GUID | 0 |
| `posX` | float | X坐标 | 0 |
| `posY` | float | Y坐标 | 0 |
| `posZ` | float | Z坐标 | 0 |
| `data` | int unsigned | 额外数据 | 0 |
| `map` | smallint unsigned | 地图ID | 0 |
| `time` | int unsigned | 死亡时间 | 0 |

---

#### game_event_condition - 游戏事件条件表

**作用：** 游戏事件触发条件

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `eventEntry` | int unsigned | 事件ID | 0 |
| `conditionId` | int unsigned | 条件ID | 0 |
| `num` | tinyint unsigned | 数量 | 0 |

---

#### creature_text - 生物对话表

**作用：** NPC对话文本配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `CreatureID` | int unsigned | 生物ID | 0 |
| `GroupID` | tinyint unsigned | 对话组 | 0 |
| `ID` | int unsigned | 文本ID | 0 |
| `Text` | varchar(255) | 对话文本 | '' |
| `Type` | tinyint unsigned | 类型 | 0 |
| `Language` | tinyint unsigned | 语言 | 0 |
| `Probability` | tinyint unsigned | 概率 | 0 |
| `Emote` | smallint unsigned | 表情 | 0 |
| `Duration` | int unsigned | 持续时间 | 0 |
| `Sound` | int unsigned | 音效 | 0 |
| `comment` | varchar(255) | 注释 | '' |

**Type 说明：**
- `0` - 说话（say）
- `1` - 喊话（yell）
- `2` - 低语（emote）
- `12` - 区域文字

---

#### npc_text - NPC对话页表

**作用：** NPC对话页面配置

**完整字段结构：**

| 字段 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `ID` | int unsigned | 对话页ID | 0 |
| `prob0` | float | 概率0 | 0 |
| `text0_0` | longtext | 文本0_0 | NULL |
| `text0_1` | longtext | 文本0_1 | NULL |
| `lang0` | tinyint unsigned | 语言0 | 0 |
| `em0_0` | smallint unsigned | 表情0_0 | 0 |
| `em0_1` | smallint unsigned | 表情0_1 | 0 |
| `em0_2` | smallint unsigned | 表情0_2 | 0 |
| `em0_3` | smallint unsigned | 表情0_3 | 0 |
| `em0_4` | smallint unsigned | 表情0_4 | 0 |
| `em0_5` | smallint unsigned | 表情0_5 | 0 |
| `em0_1_delay_0` | int unsigned | 延迟 | 0 |

---

## 完整数据库表统计

基于深入探索，AzerothCore 数据库包含以下完整表统计：

### auth 数据库（18 个表）

1. account - 账户表
2. account_access - 账户权限
3. account_banned - 账户封禁
4. account_muted - 账户禁言
5. autobroadcast - 自动广播
6. autobroadcast_locale - 广播本地化
7. build_info - 客户端版本
8. ip_banned - IP封禁
9. logs - 操作日志
10. logs_ip_actions - IP操作日志
11. motd - 每日提示
12. motd_localized - 提示本地化
13. realmcharacters - 服务器角色数
14. realmlist - 服务器列表
15. secret_digest - 密钥摘要
16. updates - 更新记录
17. updates_include - 更新包含
18. uptime - 运行时间统计

### characters 数据库（74 个表）

**核心表：**
1. characters - 角色主表
2. character_stats - 角色属性
3. character_inventory - 物品栏
4. character_spell - 法术
5. character_skills - 技能
6. character_aura - 光环
7. character_queststatus - 任务状态
8. character_social - 社交
9. character_pet - 宠物
10. item_instance - 物品实例

**任务系统：**
11. character_queststatus_daily - 日常任务
12. character_queststatus_weekly - 周常任务
13. character_queststatus_monthly - 月常任务
14. character_queststatus_seasonal - 节日任务
15. character_queststatus_rewarded - 已完成任务

**成就系统：**
16. character_achievement - 成就
17. character_achievement_progress - 成就进度
18. character_achievement_offline_updates - 离线更新

**公会系统：**
19. guild - 公会主表
20. guild_member - 公会成员
21. guild_rank - 公会职位
22. guild_bank_eventlog - 银行日志
23. guild_bank_item - 银行物品
24. guild_bank_right - 银行权限
25. guild_bank_tab - 银行标签
26. guild_eventlog - 公会日志

**竞技场系统：**
27. arena_team - 竞技队
28. arena_team_member - 队员

**邮件系统：**
29. mail - 邮件
30. mail_items - 邮件物品

**队伍系统：**
31. groups - 队伍
32. group_member - 成员

**副本系统：**
33. instance - 副本
34. instance_reset - 重置
35. character_instance - 角色副本绑定
36. group_instance - 队伍副本绑定

**拍卖行系统：**
37. auctionhouse - 拍卖行

**战场系统：**
38. battleground_deserters - 逃跑者
39. character_battleground_random - 随机战场

**其他系统：**
40. character_action - 动作条
41. character_aura - 光环
42. character_banned - 角色封禁
43. character_declinedname - 变格名称
44. character_equipmentsets - 装备方案
45. character_glyphs - 雕文
46. character_homebind - 绑定位置
47. character_talent - 天赋
48. channels - 频道
49. channels_bans - 频道封禁
50. channels_rights - 频道权限
51. corpse - 尸体
52. creature_respawn - 生物重生
53. game_event_save - 事件保存
54. game_event_condition_save - 事件条件保存
55. gameobject_respawn - 对象重生
56. gm_survey - GM调查
57. gm_subsurvey - GM子调查
58. gm_ticket - GM工单
59. lag_reports - 延迟报告
60. lfg_data - 地下城查找器
61. log_arena_fights - 竞技场战斗日志
62. log_arena_memberstats - 竞技场统计
63. addon - 插件
64. banned_addons - 禁用插件

### world 数据库（302 个表）

**核心游戏内容（50+ 表）：**

1. creature - 生物实例
2. creature_template - 生物模板
3. creature_addon - 生物扩展
4. creature_equip_template - 生物装备
5. creature_loot_template - 战利品
6. creature_onkill_reputation - 击杀声望
7. creature_queststarter - 任务开始
8. creature_questender - 任务结束
9. creature_text - 对话文本
10. creature_model_info - 模型信息
11. gameobject - 游戏对象
12. gameobject_template - 游戏对象模板
13. item_template - 物品模板
14. item_enchantment_table - 附魔
15. quest_template - 任务模板
16. quest_offer_reward - 任务奖励
17. quest_request_items - 任务需求
18. quest_details - 任务详情
19. smart_scripts - SmartAI脚本
20. npc_vendor - 商人出售
21. npc_trainer - 训练师
22. spell_dbc - 法术数据

**地形和传送（20+ 表）：**
23. game_graveyard - 墓地
24. game_tele - 传送点
25. areatrigger_teleport - 区域传送
26. areatrigger_tavern - 旅店区域
27. areatrigger_scripts - 触发器脚本

**阵营和声望（10+ 表）：**
28. faction_dbc - 阵营
29. factiontemplate_dbc - 阵营模板
30. reputation_reward_rate - 声望倍率

**战场和竞技场（15+ 表）：**
31. battleground_template - 战场模板
32. battleground_template_bc - 战场模板BC
33. arena_season_reward - 赛季奖励
34. arena_season_reward_group - 奖励组

**事件系统（15+ 表）：**
35. game_event - 游戏事件
36. game_event_condition - 事件条件
37. game_event_creature - 事件生物
38. game_event_gameobject - 事件对象
39. game_event_npc_vendor - 事件商人
40. game_event_quest_condition - 任务条件
41. game_event_creature_quest - 生物任务
42. game_event_gameobject_quest - 对象任务
43. game_event_battleground_holiday - 节日战场

**DBC 数据文件（150+ 表）：**
- *_dbc.sql - 从客户端提取的 DBC 文件

**系统配置（20+ 表）：**
44. command - GM命令
45. disables - 禁用配置
46. conditions - 条件系统
47. acore_string - 服务器字符串
48. antidos_opcode_policies - 反作弊策略

**技能和法术（10+ 表）：**
49. skill_discovery_template - 技能发现
50. skill_extra_item_template - 额外物品
51. skill_fishing_base_level - 钓鱼基础等级
52. spell_target_position - 法术目标位置

**副本系统（10+ 表）：**
53. instance_template - 副本模板
54. dungeon_access_template - 副本访问
55. dungeon_access_requirements - 副本需求
56. dungeonencounter_dbc - 副本BOSS

**战利品系统（20+ 表）：**
57. reference_loot_template - 引用战利品
58. fishing_loot_template - 钓鱼战利品
59. disenchant_loot_template - 分解战利品
60. skinning_loot_template - 剥皮战利品
61. mail_loot_template - 邮件战利品
62. item_loot_template - 物品战利品
63. prospecting_loot_template - 采矿战利品
64. milling_loot_template - 制草战利品

---

## 总结

AzerothCore 使用三数据库分离架构，将认证、角色数据和游戏内容完全分离：

| 数据库 | 表数量 | 主要作用 | 核心表 |
|--------|--------|----------|--------|
| **auth** | 18 | 账户认证和权限 | account, realmlist, account_access |
| **characters** | 74 | 角色数据和社交 | characters, character_inventory, guild, mail, arena_team |
| **world** | 302+ | 游戏内容数据 | creature_template, item_template, quest_template, spell_dbc |

**总计：394+ 个表**

**设计优势：**
1. **可扩展性**：可以单独扩展任何数据库
2. **可维护性**：数据分离便于备份和管理
3. **性能**：可以针对不同数据库优化查询
4. **灵活性**：支持多服务器共享 world 数据库
5. **模块化**：表之间职责明确，便于理解和修改

**代码集成：**
- 使用 `DatabaseWorkerPool` 模板类管理连接
- 预处理语句提供高性能和安全性
- 支持异步和同步两种操作模式
- 完整的更新系统确保数据库版本一致
