# AzerothCore WotLK — Account & Character 模块分析文档

> 基于项目源码的深度分析，适用于二次开发和模块定制。

---

## 目录

- [1. 项目架构概览](#1-项目架构概览)
- [2. Account 模块](#2-account-模块)
  - [2.1 模块结构](#21-模块结构)
  - [2.2 核心类](#22-核心类)
  - [2.3 数据库表](#23-数据库表)
  - [2.4 核心功能](#24-核心功能)
  - [2.5 RBAC 权限系统](#25-rbac-权限系统)
  - [2.6 脚本钩子](#26-脚本钩子)
- [3. Character 模块](#3-character-模块)
  - [3.1 模块结构](#31-模块结构)
  - [3.2 核心类与继承关系](#32-核心类与继承关系)
  - [3.3 数据库表](#33-数据库表)
  - [3.4 核心功能](#34-核心功能)
  - [3.5 物品与背包系统](#35-物品与背包系统)
  - [3.6 宠物系统](#36-宠物系统)
  - [3.7 任务系统](#37-任务系统)
  - [3.8 社交系统](#38-社交系统)
- [4. 模块间交互与数据流](#4-模块间交互与数据流)
  - [4.1 关键关系图](#41-关键关系图)
  - [4.2 账号登录全流程](#42-账号登录全流程)
  - [4.3 角色创建全流程](#43-角色创建全流程)
  - [4.4 角色进入世界全流程](#44-角色进入世界全流程)
  - [4.5 角色删除与账号删除](#45-角色删除与账号删除)
- [5. PlayerDump 系统 — 角色数据完整导出/导入](#5-playerdump-系统--角色数据完整导出导入)
  - [5.1 概述与 GM 命令](#51-概述与-gm-命令)
  - [5.2 涉及的 32 张数据库表全表分类](#52-涉及的-32-张数据库表全表分类)
  - [5.3 GUID 类型与重映射机制](#53-guid-类型与重映射机制)
  - [5.4 各表的 GUID 依赖关系详解](#54-各表的-guid-依赖关系详解)
  - [5.5 导出流程 (Writer)](#55-导出流程-writer)
  - [5.6 导入流程 (Reader)](#56-导入流程-reader)
  - [5.7 characters 表补充字段](#57-characters-表补充字段)
  - [5.8 邮件系统表](#58-邮件系统表)
  - [5.9 AtLoginFlags 登录标记](#5-9-atloginflags-登录标记)
  - [5.10 从 Dump 视角看角色数据完整性](#5-10-从-dump-视角看角色数据完整性)
  - [6. Update Fields 系统](#6-update-fields-系统)
- [7. 性能与安全设计](#7-性能与安全设计)
- [8. 扩展与定制指南](#8-扩展与定制指南)
- [9. 角色重置 — 将角色恢复到 1 级初始状态](#9-角色重置--将角色恢复到1级初始状态)

---

## 1. 项目架构概览

AzerothCore WotLK 采用**多数据库架构**，核心数据分布在三个数据库中：

| 数据库 | 用途 | 示例表 |
|--------|------|--------|
| `acore_auth` | 账号认证、权限、安全 | `account`, `account_access`, `account_banned`, `ip_banned`, `rbac_*` |
| `acore_characters` | 角色数据、物品、任务进度 | `characters`, `character_inventory`, `item_instance`, `character_spell` |
| `acore_world` | 游戏世界数据（静态） | `creature_template`, `item_template`, `quest_template` |

源码目录结构（核心部分）：

```
src/server/
├── game/
│   ├── Accounts/               # Account 模块核心
│   │   ├── AccountMgr.h/cpp    # 账号管理器
│   │   └── RBAC.h/cpp          # 基于角色的权限控制
│   ├── Entities/
│   │   ├── Player/             # Character 模块核心
│   │   │   ├── Player.h/cpp    # 玩家主类（3071 行头文件）
│   │   │   ├── PlayerStorage.cpp     # 数据库加载/保存
│   │   │   ├── PlayerQuest.cpp       # 任务系统
│   │   │   ├── PlayerUpdates.cpp     # 更新系统
│   │   │   ├── PlayerGossip.cpp      # NPC 交互
│   │   │   ├── PlayerMisc.cpp        # 杂项功能
│   │   │   ├── PlayerSettings.cpp/h  # 玩家设置
│   │   │   ├── PlayerTaxi.cpp/h      # 出租车/飞行系统
│   │   │   ├── SocialMgr.cpp/h       # 社交系统
│   │   │   ├── KillRewarder.cpp/h    # 击杀奖励
│   │   │   ├── RaceMgr.cpp/h         # 种族管理
│   │   │   ├── CinematicMgr.cpp/h    # 过场动画
│   │   │   └── TradeData.cpp/h       # 交易系统
│   │   ├── Unit/               # 单位基类（Player/Creature 的父类）
│   │   │   └── Unit.h/cpp     # 战斗、法术、属性系统
│   │   ├── Object/             # 对象基类
│   │   ├── Item/               # 物品系统
│   │   ├── Pet/                # 宠物系统
│   │   └── Creature/           # NPC 系统
│   ├── Server/
│   │   ├── WorldSession.h/cpp  # 客户端会话（Account 与 Character 的桥梁）
│   │   └── WorldSessionMgr.h/cpp  # 会话管理器
│   ├── Handlers/
│   │   ├── CharacterHandler.cpp     # 角色相关网络包处理
│   │   ├── MovementHandler.cpp
│   │   ├── CombatHandler.cpp
│   │   └── ...
│   ├── Cache/
│   │   └── CharacterCache.cpp/h     # 角色数据缓存
│   └── Scripting/
│       └── ScriptDefines/
│           └── AccountScript.h      # 账号脚本钩子定义
├── scripts/
│   └── Commands/
│       ├── cs_account.cpp           # 账号管理 GM 命令
│       ├── cs_character.cpp         # 角色管理 GM 命令
│       ├── cs_player.cpp            # 玩家操作 GM 命令
│       └── ...
└── database/Database/
    └── Implementation/
        └── CharacterDatabase.cpp/h  # 角色数据库实现
```

---

## 2. Account 模块

### 2.1 模块结构

Account 模块的核心职责是**账号的创建、认证、权限管理和安全控制**。它跨越两个层次：

- **Login Server 层**（认证服务器，本项目为内嵌实现）：处理 SRP6 协议认证、会话密钥协商
- **World Server 层**：处理账号权限、GM 等级、RBAC 权限、封禁/禁言管理

源文件清单：

| 文件路径 | 说明 |
|----------|------|
| `src/server/game/Accounts/AccountMgr.h/cpp` | 账号管理器，提供账号 CRUD、密码/邮箱修改、安全等级查询 |
| `src/server/game/Accounts/RBAC.h/cpp` | RBAC 权限系统，定义权限、角色、授权/拒绝/撤销机制 |
| `src/server/game/Server/WorldSession.h/cpp` | 客户端会话类，持有账号信息并管理 Player 实例 |
| `src/server/game/Server/WorldSessionMgr.h/cpp` | 会话管理器，管理所有在线连接 |
| `src/server/game/Scripting/ScriptDefines/AccountScript.h` | 账号脚本钩子接口 |
| `src/server/scripts/Commands/cs_account.cpp` | 账号管理 GM 命令实现 |

### 2.2 核心类

#### 2.2.1 AccountMgr（单例）

**文件**: `src/server/game/Accounts/AccountMgr.h:44`

```cpp
class AC_GAME_API AccountMgr
{
    // 账号生命周期管理
    AccountOpResult CreateAccount(std::string username, std::string password, std::string email = "");
    static AccountOpResult DeleteAccount(uint32 accountId);
    static AccountOpResult ChangeUsername(uint32 accountId, std::string newUsername, std::string newPassword);
    static AccountOpResult ChangePassword(uint32 accountId, std::string newPassword);
    static AccountOpResult ChangeEmail(uint32 accountId, std::string email);
    static bool CheckPassword(uint32 accountId, std::string password);

    // 查询接口
    static uint32 GetId(std::string const& username);           // 用户名 -> 账号 ID
    static uint32 GetSecurity(uint32 accountId);                // 获取 GM 等级
    static uint32 GetSecurity(uint32 accountId, int32 realmId); // 按 realm 获取
    static bool GetName(uint32 accountId, std::string& name);   // 账号 ID -> 用户名
    static uint32 GetCharactersCount(uint32 accountId);          // 角色数量

    // GM 等级判断
    static bool IsPlayerAccount(uint32 gmlevel);
    static bool IsGMAccount(uint32 gmlevel);
    static bool IsAdminAccount(uint32 gmlevel);
    static bool IsConsoleAccount(uint32 gmlevel);

    // RBAC
    static bool HasPermission(uint32 accountId, uint32 permission, uint32 realmId);
    void UpdateAccountAccess(rbac::RBACData* rbac, uint32 accountId, uint8 securityLevel, int32 realmId);
    void LoadRBAC();
};
```

**设计特点**:
- 单例模式：通过 `sAccountMgr` 宏访问
- 密码使用 **SRP6 (Secure Remote Password Protocol)** 加密，不存储明文密码
- 账号操作结果通过 `AccountOpResult` 枚举返回

**AccountOpResult 枚举** (`AccountMgr.h:23`):

```cpp
enum AccountOpResult
{
    AOR_OK,                  // 操作成功
    AOR_NAME_TOO_LONG,       // 用户名过长（最大 16 字符）
    AOR_PASS_TOO_LONG,       // 密码过长（最大 16 字符）
    AOR_EMAIL_TOO_LONG,      // 邮箱过长（最大 255 字符）
    AOR_NAME_ALREADY_EXIST,  // 用户名已存在
    AOR_NAME_NOT_EXIST,      // 用户名不存在
    AOR_DB_INTERNAL_ERROR    // 数据库内部错误
};
```

#### 2.2.2 WorldSession（客户端会话）

**文件**: `src/server/game/Server/WorldSession.h:383`

WorldSession 是 Account 模块与 Character 模块的**核心桥梁**，代表一个客户端连接。

```cpp
class WorldSession
{
    // === 账号信息 ===
    uint32 GetAccountId() const;              // 账号 ID
    AccountTypes GetSecurity() const;          // GM 安全等级
    std::string const& GetRemoteAddress();     // 客户端 IP
    LocaleConstant GetSessionDbcLocale() const;// 客户端语言

    // === 角色信息 ===
    Player* GetPlayer() const;                 // 当前登录的角色
    void SetPlayer(Player* player);
    bool PlayerLoading() const;               // 角色正在加载
    bool PlayerLogout() const;                // 角色正在登出

    // === 权限 ===
    rbac::RBACData* GetRBACData() const;       // RBAC 数据
    bool HasPermission(uint32 permissionId);   // 检查权限
    void LoadPermissions();                    // 加载权限

    // === 账号标记 ===
    uint32 GetAccountFlags() const;            // 账号标记（试用版、网游等）
    bool IsGMAccount() const;
    bool IsTrialAccount() const;

    // === 会话管理 ===
    void LogoutPlayer(bool save);              // 登出角色
    void KickPlayer(std::string const& reason); // 踢出玩家

    // === 数据包处理 ===
    void HandleCharEnumOpcode(WorldPacket&);   // 获取角色列表
    void HandleCharCreateOpcode(WorldPacket&); // 创建角色
    void HandleCharDeleteOpcode(WorldPacket&); // 删除角色
    void HandlePlayerLoginOpcode(WorldPacket&);// 进入世界
};
```

**WorldSession 与 Account/Player 的关系**:

```
WorldSocket (网络层)
    └── WorldSession (会话层，持有账号信息)
            ├── AccountId     (账号 ID)
            ├── Security      (GM 等级)
            ├── RBACData*     (RBAC 权限数据)
            ├── AccountData[] (账号全局数据)
            ├── Tutorials[]   (教程进度)
            └── Player*       (当前角色实例)
                    └── Player (游戏角色)
```

#### 2.2.3 辅助类

**CharacterCreateInfo** (`WorldSession.h:321`):
角色创建时传递的数据结构，包含用户指定的名称、种族、职业、性别、外观等。

**PacketFilter / MapSessionFilter / WorldSessionFilter** (`WorldSession.h:284-317`):
数据包过滤器，区分线程安全和不安全的数据包处理。

### 2.3 数据库表

Account 模块使用 `acore_auth` 数据库，核心表如下：

#### 2.3.1 `account` — 账号主表

**文件**: `data/sql/base/db_auth/account.sql`

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `int unsigned AUTO_INCREMENT` | 账号 ID（主键） |
| `username` | `varchar(32)` | 用户名（唯一索引） |
| `salt` | `binary(32)` | SRP6 密码盐值 |
| `verifier` | `binary(32)` | SRP6 密码验证器 |
| `session_key` | `binary(40)` | 会话密钥 |
| `totp_secret` | `varbinary(128)` | TOTP 两步验证密钥 |
| `email` | `varchar(255)` | 当前邮箱 |
| `reg_mail` | `varchar(255)` | 注册邮箱 |
| `joindate` | `timestamp` | 注册时间 |
| `last_ip` | `varchar(15)` | 最后登录 IP |
| `last_attempt_ip` | `varchar(15)` | 最后尝试登录 IP |
| `failed_logins` | `int unsigned` | 连续登录失败次数 |
| `locked` | `tinyint unsigned` | 是否锁定 |
| `lock_country` | `varchar(2)` | 国家锁定（2 字符国家码） |
| `last_login` | `timestamp` | 最后登录时间 |
| `online` | `int unsigned` | 在线状态 |
| `expansion` | `tinyint unsigned` | 资料片等级（2=WotLK） |
| `Flags` | `int unsigned` | 账号标记（试用版等） |
| `mutetime` | `bigint` | 禁言到期时间戳 |
| `mutereason` | `varchar(255)` | 禁言原因 |
| `muteby` | `varchar(50)` | 禁言操作者 |
| `locale` | `tinyint unsigned` | 客户端语言 |
| `os` | `varchar(3)` | 操作系统 |
| `recruiter` | `int unsigned` | 招募者账号 ID |
| `totaltime` | `int unsigned` | 总在线时间（秒） |

**安全设计要点**:
- 密码使用 **SRP6** 协议，只存储 `salt` 和 `verifier`，无法逆向获取密码
- 支持 **TOTP 两步验证**（`totp_secret`）
- 记录 `failed_logins` 和 `last_attempt_ip` 用于暴力破解防护
- 支持 `lock_country` 国家/地区锁定

#### 2.3.2 `account_access` — GM 等级

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `int unsigned` | 账号 ID（联合主键） |
| `gmlevel` | `tinyint unsigned` | GM 等级 |
| `RealmID` | `int` | 服务器 ID（-1 = 所有服务器） |
| `comment` | `varchar(255)` | 备注 |

**GM 等级**:
| 等级 | 常量 | 说明 |
|------|------|------|
| 0 | `SEC_PLAYER` | 普通玩家 |
| 1 | `SEC_MODERATOR` | 版主 |
| 2 | `SEC_GAMEMASTER` | 游戏管理员 |
| 3 | `SEC_ADMINISTRATOR` | 系统管理员 |
| 4 | `SEC_CONSOLE` | 控制台 |

#### 2.3.3 `account_banned` — 账号封禁

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | `int unsigned` | 账号 ID（联合主键） |
| `bandate` | `int unsigned` | 封禁时间（Unix 时间戳） |
| `unbandate` | `int unsigned` | 解封时间（0=永久） |
| `bannedby` | `varchar(50)` | 封禁操作者 |
| `banreason` | `varchar(255)` | 封禁原因 |
| `active` | `tinyint unsigned` | 是否生效 |

#### 2.3.4 `account_muted` — 账号禁言

| 字段 | 类型 | 说明 |
|------|------|------|
| `guid` | `int unsigned` | 角色 GUID（联合主键） |
| `mutedate` | `int unsigned` | 禁言时间 |
| `mutetime` | `int unsigned` | 禁言持续时间 |
| `mutedby` | `varchar(50)` | 操作者 |
| `mutereason` | `varchar(255)` | 原因 |

#### 2.3.5 `ip_banned` — IP 封禁

| 字段 | 类型 | 说明 |
|------|------|------|
| `ip` | `varchar(15)` | IP 地址 |
| `bandate` | `int unsigned` | 封禁时间 |
| `unbandate` | `int unsigned` | 解封时间 |
| `bannedby` | `varchar(50)` | 操作者 |
| `banreason` | `varchar(255)` | 原因 |

#### 2.3.6 `realmcharacters` — Realm 角色计数

| 字段 | 类型 | 说明 |
|------|------|------|
| `realmid` | `int unsigned` | 服务器 ID |
| `acctid` | `int unsigned` | 账号 ID |
| `numchars` | `tinyint unsigned` | 角色数量 |

用于跨 realm 的角色数量统计（限制每个账号在不同 realm 的总角色数）。

#### 2.3.7 RBAC 相关表（`acore_auth` 数据库）

| 表名 | 说明 |
|------|------|
| `rbac_permissions` | 权限定义（id, name） |
| `rbac_linked_permissions` | 权限关联（父权限 -> 子权限） |
| `rbac_default_permissions` | 默认权限分配（按 GM 等级和 realm） |
| `rbac_account_permissions` | 账号自定义权限（grant/deny/revoke） |
| `rbac_account_roles` | 账号角色分配 |

### 2.4 核心功能

#### 2.4.1 账号创建流程

```
AccountMgr::CreateAccount()
    ├── 1. 验证用户名长度 (≤16)
    ├── 2. 验证密码长度 (≤16)
    ├── 3. 验证邮箱长度 (≤255)
    ├── 4. 转大写处理
    ├── 5. 检查用户名是否已存在
    ├── 6. SRP6 生成 salt + verifier
    ├── 7. INSERT INTO account
    └── 8. 初始化 realmcharacters 计数记录
```

#### 2.4.2 账号删除流程

```
AccountMgr::DeleteAccount()
    ├── 1. 验证账号存在
    ├── 2. 触发脚本钩子 OnBeforeAccountDelete
    ├── 3. 获取账号下所有角色
    │   ├── 踢出在线角色 (KickPlayer + LogoutPlayer)
    │   └── Player::DeleteFromDB() (删除角色数据)
    ├── 4. 删除 character 相关数据
    │   ├── character_tutorial
    │   ├── character_account_data
    │   └── character_banned
    └── 5. 事务删除 auth 相关数据
        ├── account
        ├── account_access
        ├── realmcharacters
        ├── account_banned
        └── account_muted
```

#### 2.4.3 密码修改流程

```
AccountMgr::ChangePassword()
    ├── 1. 获取账号用户名
    ├── 2. 验证账号存在（不存在触发 OnFailedPasswordChange）
    ├── 3. 验证密码长度
    ├── 4. SRP6 重新生成 salt + verifier
    ├── 5. UPDATE account SET salt, verifier
    └── 6. 触发脚本钩子 OnPasswordChange
```

### 2.5 RBAC 权限系统

**文件**: `src/server/game/Accounts/RBAC.h`

RBAC (Role-Based Access Control) 是 AzerothCore 的细粒度权限系统。

#### 2.5.1 核心概念

```
Permission（权限）: 最小授权单位，定义一项具体操作
    例：RBAC_PERM_COMMAND_ACCOUNT_CREATE = 219

Role（角色）: 权限的集合
    例：RBAC_ROLE_GAMEMASTER = 197

Account -> Roles + Permissions
    账号可以同时拥有多个角色和权限
    每个角色/权限可以是 Grant(授权) 或 Deny(拒绝) 状态

最终权限计算:
    (Role Grants + User Grants) - (Role Denies - User Denies)
```

#### 2.5.2 权限分类

| 范围 | 权限 ID | 说明 |
|------|---------|------|
| 通用权限 | 1-53 | 即时登出、跳过队列、加入战场等 |
| 角色 | 196-199 | 管理员、GM、版主、玩家角色 |
| 命令权限 | 200+ | 每个可用 GM 命令对应一个权限 |
| | 200-216 | RBAC 命令 |
| | 217-266 | Account 命令 |
| | 267-299 | Cast/Cheat 命令 |
| | 371-376 | GM 模式命令 |
| | 388-399 | GameObject 命令 |
| | 437-460 | List/Lookup 命令 |
| | ... | （共 800+ 个权限） |

#### 2.5.3 Realm 级别权限

权限可以按 realm 粒度配置：
- `RealmID = -1`: 所有服务器生效（全局）
- `RealmID = 具体ID`: 仅在指定服务器生效

### 2.6 脚本钩子

**文件**: `src/server/game/Scripting/ScriptDefines/AccountScript.h`

```cpp
class AccountScript : public ScriptObject
{
    // 账号成功登录时
    virtual void OnAccountLogin(uint32 accountId);

    // 账号即将删除时
    virtual void OnBeforeAccountDelete(uint32 accountId);

    // IP 更新时
    virtual void OnLastIpUpdate(uint32 accountId, std::string ip);

    // 登录失败时
    virtual void OnFailedAccountLogin(uint32 accountId);

    // 邮箱变更（成功/失败）
    virtual void OnEmailChange(uint32 accountId);
    virtual void OnFailedEmailChange(uint32 accountId);

    // 密码变更（成功/失败）
    virtual void OnPasswordChange(uint32 accountId);
    virtual void OnFailedPasswordChange(uint32 accountId);

    // 角色创建前验证（可拦截创建）
    virtual bool CanAccountCreateCharacter(uint32 accountId, uint8 charRace, uint8 charClass);
};
```

---

## 3. Character 模块

### 3.1 模块结构

Character 模块是 AzerothCore 中**最大、最复杂的模块**，涉及约 19,494 行核心代码（仅 Player 类），管理 106 张数据库表。

源文件清单（核心部分）：

| 文件路径 | 大小 | 说明 |
|----------|------|------|
| `src/server/game/Entities/Player/Player.h` | 131K, 3071 行 | Player 类头文件 |
| `src/server/game/Entities/Player/Player.cpp` | 590K, 16423 行 | Player 主实现 |
| `src/server/game/Entities/Player/PlayerStorage.cpp` | 298K | 数据库加载/保存 |
| `src/server/game/Entities/Player/PlayerQuest.cpp` | 88K | 任务系统 |
| `src/server/game/Entities/Player/PlayerUpdates.cpp` | 80K | 更新系统 |
| `src/server/game/Entities/Player/PlayerGossip.cpp` | 17K | NPC 交互 |
| `src/server/game/Entities/Player/PlayerMisc.cpp` | 18K | 杂项功能 |
| `src/server/game/Entities/Player/PlayerSettings.cpp/h` | 6.1K/2.6K | 玩家设置 |
| `src/server/game/Entities/Player/PlayerTaxi.cpp/h` | 6.0K/3.0K | 飞行系统 |
| `src/server/game/Entities/Player/SocialMgr.cpp/h` | 11K/5.2K | 社交系统 |
| `src/server/game/Entities/Player/KillRewarder.cpp/h` | 13K/1.7K | 击杀奖励 |
| `src/server/game/Entities/Player/TradeData.cpp/h` | 3.6K/3.2K | 交易系统 |
| `src/server/game/Entities/Player/RaceMgr.cpp/h` | 1.9K/1.6K | 种族管理 |
| `src/server/game/Entities/Player/CinematicMgr.cpp/h` | 5.7K/1.9K | 过场动画 |
| `src/server/game/Entities/Unit/Unit.h/cpp` | 2353/17434 行 | 单位基类 |
| `src/server/game/Entities/Item/Item.h/cpp` | — | 物品系统 |
| `src/server/game/Entities/Item/Container/Bag.h/cpp` | — | 容器/背包 |
| `src/server/game/Entities/Pet/Pet.h/cpp` | — | 宠物系统 |
| `src/server/game/Handlers/CharacterHandler.cpp` | — | 角色网络包处理 |
| `src/server/game/Cache/CharacterCache.cpp/h` | — | 角色数据缓存 |

### 3.2 核心类与继承关系

```
Object                              # 所有游戏对象基类
├── WorldObject                     # 世界对象
│   ├── Unit                        # 活体单位基类
│   │   ├── Player                  # ★ 玩家角色（核心类）
│   │   ├── Creature                # NPC
│   │   │   ├── Guardian            # 守护者
│   │   │   │   └── Pet            # 宠物
│   │   │   ├── Minion              # 仆从
│   │   │   ├── Totem               # 图腾
│   │   │   └── TemporarySummon     # 临时召唤
│   │   └── ...
│   ├── GameObject                  # 游戏物体
│   ├── Corpse                      # 尸体
│   └── DynamicObject               # 动态对象
└── Item                            # 物品（不继承 WorldObject）
    └── Bag                         # 容器/背包
```

#### 3.2.1 Player 类

**文件**: `src/server/game/Entities/Player/Player.h`

`Player` 是 Character 模块的核心，继承自 `Unit` 和 `GridObject<Player>`。

**核心方法分组**:

**加载/保存**:
```cpp
bool LoadFromDB(ObjectGuid guid, CharacterDatabaseQueryHolder const& holder);
void SaveToDB(bool create, bool logout);
void SaveToDB(CharacterDatabaseTransaction trans, bool create, bool logout);
static void DeleteFromDB(uint32 lowguid, uint32 accountId, bool updateRealmChars, bool deleteNow);
```

**物品/背包**:
```cpp
InventoryResult CanStoreItem(uint8 bag, uint8 slot, ItemPosCountVec& dest, Item* pItem, bool swap = false) const;
InventoryResult CanEquipItem(uint8 slot, uint16& dest, Item* pItem, bool swap, bool not_loading = true) const;
Item* GetItemByGuid(ObjectGuid guid) const;
Item* GetItemByPos(uint16 pos) const;
Item* GetItemByPos(uint8 bag, uint8 slot) const;
Item* GetItemByEntry(uint32 entry) const;
void DestroyItem(uint8 bag, uint8 slot, bool update);
void EquipItem(uint16 pos, Item* pItem, bool update);
Item* StoreItem(ItemPosCountVec const& dest, Item* pItem, bool update);
```

**属性/战斗**:
```cpp
void Update(uint32 time);                   // 主更新循环
uint32 GetHealth() const;
uint32 GetMaxHealth() const;
uint32 GetLevel() const;
void GiveXP(uint32 xp, Unit* victim);
```

**法术/技能**:
```cpp
void LearnSpell(uint32 spell_id, bool fromCheat = false);
void RemoveSpell(uint32 spell_id, bool fromCheat = false);
bool HasSpell(uint32 spell_id) const;
uint16 GetSkillValue(uint32 skill) const;
void SetSkill(uint16 id, uint16 currVal, uint16 maxVal);
```

#### 3.2.2 Unit 类

**文件**: `src/server/game/Entities/Unit/Unit.h`

所有活体单位的基类（Player 和 Creature 都继承自 Unit），提供：

- 属性系统（力量、敏捷、耐力、智力、精神）
- 生命/能量系统（Health, Mana, Rage, Energy, Runic Power）
- 战斗系统（伤害计算、命中、暴击、闪避、招架）
- 法术系统（施法、Aura 效果）
- 威胁管理系统（ThreatManager）
- 移动系统

#### 3.2.3 Item 类

**文件**: `src/server/game/Entities/Item/Item.h`

```cpp
class Item : public Object
{
    static Item* CreateItem(uint32 item, uint32 count, Player const* player = nullptr, ...);
    virtual bool Create(ObjectGuid::LowType guidlow, uint32 itemid, Player const* owner);
    virtual void SaveToDB(CharacterDatabaseTransaction trans);
    virtual bool LoadFromDB(ObjectGuid::LowType guid, ObjectGuid owner_guid, Field* fields, uint32 entry);
    void SetEnchantment(EnchantmentSlot slot, uint32 id, uint32 duration, uint32 charges, ObjectGuid caster);
    bool IsSoulBound() const;
    bool IsBag() const;
    bool IsEquipped() const;
    uint32 GetCount() const;
};
```

#### 3.2.4 Pet 类

**文件**: `src/server/game/Entities/Pet/Pet.h`

```cpp
class Pet : public Guardian
{
    bool LoadPetFromDB(Player* owner, uint32 petEntry, uint32 petnumber, bool current, ...);
    void SavePetToDB(PetSaveMode mode);
    void GivePetXP(uint32 xp);
    void GivePetLevel(uint8 level);
    HappinessState GetHappinessState();
    void ToggleAutocast(SpellInfo const* spellInfo, bool apply);
};
```

### 3.3 数据库表

Character 模块使用 `acore_characters` 数据库，核心表如下：

#### 3.3.1 `characters` — 角色主表

**文件**: `data/sql/base/db_characters/characters.sql`

| 字段 | 类型 | 说明 |
|------|------|------|
| `guid` | `int unsigned` | 角色 GUID（全局唯一，主键） |
| `account` | `int unsigned` | 所属账号 ID（外键到 account.id） |
| `name` | `varchar(12)` | 角色名 |
| `race` | `tinyint unsigned` | 种族 |
| `class` | `tinyint unsigned` | 职业 |
| `gender` | `tinyint unsigned` | 性别 |
| `level` | `tinyint unsigned` | 等级 |
| `xp` | `int unsigned` | 当前经验值 |
| `money` | `int unsigned` | 金币（铜币） |
| `playerBytes` | `int unsigned` | 外观（皮肤、脸型、发型、发色） |
| `playerBytes2` | `int unsigned` | 外观（面部毛发、自定义） |
| `position_x/y/z` | `float` | 坐标 |
| `map` | `smallint unsigned` | 地图 ID |
| `orientation` | `float` | 朝向 |
| `zone` | `smallint unsigned` | 区域 ID |
| `online` | `tinyint unsigned` | 在线状态 |
| `totaltime` | `int unsigned` | 总在线时间 |
| `leveltime` | `int unsigned` | 当前等级在线时间 |
| `logout_time` | `int unsigned` | 登出时间 |
| `is_logout_resting` | `tinyint unsigned` | 登出时是否在休息区 |
| `restState` | `tinyint unsigned` | 休息状态 |
| `death_state` | `tinyint unsigned` | 死亡状态 |
| `taxi_path` | `text` | 飞行路线 |
| `talentGroupsCount` | `tinyint unsigned` | 天赋组数（双天赋=2） |
| `activeTalentGroup` | `tinyint unsigned` | 当前天赋组 |
| `exploredZones` | `longtext` | 已探索区域（128 位位掩码） |
| `equipmentCache` | `longtext` | 装备缓存 |
| `guildid` | `int unsigned` | 公会 ID |
| `arenaTeamId` | `int unsigned[3]` | 竞技场队伍 ID（2v2/3v3/5v5） |

#### 3.3.2 物品系统表

| 表名 | 说明 |
|------|------|
| `character_inventory` | 角色物品位置映射 (guid, bag, slot, item) |
| `item_instance` | 物品实例数据（属性、附魔、耐久度等） |

`character_inventory` 结构：
| 字段 | 说明 |
|------|------|
| `guid` | 角色 GUID |
| `bag` | 背包 GUID (0=主背包) |
| `slot` | 槽位编号 |
| `item` | 物品 GUID（唯一约束） |

#### 3.3.3 进阶系统表

| 表名 | 说明 |
|------|------|
| `character_spell` | 已学法术 (guid, spell, specMask) |
| `character_skills` | 技能等级 (guid, skill, value, max) |
| `character_aura` | 活跃的 Aura 效果 |
| `character_action` | 动作栏配置 (guid, spec, button, action, type) |
| `character_talent` | 天赋点分配 |
| `character_glyphs` | 雕文选择 |
| `character_equipmentsets` | 装备搭配方案 |
| `character_stats` | 角色属性快照 |

#### 3.3.4 任务系统表

| 表名 | 说明 |
|------|------|
| `character_queststatus` | 任务进度（目标计数、状态） |
| `character_queststatus_daily` | 每日任务重置时间 |
| `character_queststatus_weekly` | 每周任务重置时间 |
| `character_queststatus_monthly` | 每月任务重置时间 |
| `character_queststatus_seasonal` | 赛季任务重置时间 |
| `character_queststatus_rewarded` | 已完成任务 |

#### 3.3.5 声望/成就表

| 表名 | 说明 |
|------|------|
| `character_reputation` | 声望值 (guid, faction, standing, flags) |
| `character_achievement` | 成就完成记录 |
| `character_achievement_progress` | 成就进度 |

#### 3.3.6 宠物系统表

| 表名 | 说明 |
|------|------|
| `character_pet` | 宠物实例（等级、经验、HP/MP、技能数据） |
| `pet_aura` | 宠物 Aura |
| `pet_spell` | 宠物法术书 |
| `pet_spell_cooldown` | 宠物法术冷却 |

#### 3.3.7 社交/其他表

| 表名 | 说明 |
|------|------|
| `character_social` | 好友/忽略列表 (guid, friend, flags, note) |
| `character_homebind` | 炉石绑定点 |
| `character_instance` | 副本锁定 |
| `character_banned` | 角色封禁 |
| `character_declinedname` | 俄语变格名 |
| `account_data` | 账号全局数据 |
| `account_tutorial` | 教程进度 |
| `account_instance_times` | 副本重置时间 |

### 3.4 核心功能

#### 3.4.1 可选种族（WotLK）

| 常量 | 值 | 种族 | 阵营 |
|------|---|------|------|
| `RACE_HUMAN` | 1 | 人类 | 联盟 |
| `RACE_ORC` | 2 | 兽人 | 部落 |
| `RACE_DWARF` | 3 | 矮人 | 联盟 |
| `RACE_NIGHTELF` | 4 | 暗夜精灵 | 联盟 |
| `RACE_UNDEAD_PLAYER` | 5 | 亡灵 | 部落 |
| `RACE_TAUREN` | 6 | 牛头人 | 部落 |
| `RACE_GNOME` | 7 | 侏儒 | 联盟 |
| `RACE_TROLL` | 8 | 巨魔 | 部落 |
| `RACE_BLOODELF` | 10 | 血精灵 | 部落 |
| `RACE_DRAENEI` | 11 | 德莱尼 | 联盟 |

#### 3.4.2 可选职业（WotLK）

| 常量 | 值 | 职业 | 说明 |
|------|---|------|------|
| `CLASS_WARRIOR` | 1 | 战士 | — |
| `CLASS_PALADIN` | 2 | 圣骑士 | — |
| `CLASS_HUNTER` | 3 | 猎人 | — |
| `CLASS_ROGUE` | 4 | 盗贼 | — |
| `CLASS_PRIEST` | 5 | 牧师 | — |
| `CLASS_DEATH_KNIGHT` | 6 | 死亡骑士 | WotLK 新增 |
| `CLASS_SHAMAN` | 7 | 萨满 | — |
| `CLASS_MAGE` | 8 | 法师 | — |
| `CLASS_WARLOCK` | 9 | 术士 | — |
| `CLASS_DRUID` | 11 | 德鲁伊 | — |

#### 3.4.3 等级上限

- WotLK 最高等级: **80**
- 死亡骑士起始等级: **55**
- 支持双天赋系统（`talentGroupsCount = 2`）

#### 3.4.4 角色加载流程（Player::LoadFromDB）

```
Player::LoadFromDB()
    ├── 加载 characters 表基础数据
    ├── 加载 character_action (动作栏)
    ├── 加载 character_aura (活跃 Aura)
    ├── 加载 character_achievement
    ├── 加载 character_achievement_progress
    ├── 加载 character_battleground_random
    ├── 加载 character_declinedname
    ├── 加载 character_equipmentsets
    ├── 加载 character_gifts
    ├── 加载 character_glyphs
    ├── 加载 character_homebind
    ├── 加载 character_instance
    ├── 加载 character_inventory + item_instance
    ├── 加载 character_queststatus (所有类型)
    ├── 加载 character_reputation
    ├── 加载 character_skills
    ├── 加载 character_spell
    ├── 加载 character_spell_cooldown
    ├── 加载 character_stats
    ├── 加载 character_talent
    ├── 加载 account_data
    └── 初始化各种管理器（社会、交易、出租车等）
```

#### 3.4.5 角色保存流程（Player::SaveToDB）

```
Player::SaveToDB()
    ├── 开始数据库事务
    ├── UPDATE characters (基础数据)
    ├── UPDATE/INSERT item_instance (修改的物品)
    ├── UPDATE character_inventory (物品位置变更)
    ├── UPDATE character_spell (新学法术)
    ├── UPDATE character_skills (技能变化)
    ├── UPDATE character_reputation (声望变化)
    ├── UPDATE character_queststatus (任务进度)
    ├── UPDATE character_action (动作栏变更)
    ├── UPDATE character_talent (天赋变更)
    ├── UPDATE character_aura (Aura 变化)
    ├── DELETE + INSERT character_inventory (物品删除/新增)
    └── 提交事务
```

### 3.5 物品与背包系统

#### 3.5.1 背包布局

| 区域 | 背包槽位 | 物品槽位 |
|------|----------|----------|
| 装备栏 | — | 18 个槽位 |
| 主背包 | bag=0 | 16 个槽位（默认） |
| 附加背包 | bag=19~22 | 4 个背包，每个最多 36 格 |
| 银行主栏 | bag=0 (bank) | 6-7 格（可扩展） |
| 银行背包 | bag=67~73 | 最多 7 个银行背包 |
| 钥匙圈 | — | 32 格（WotLK 已废弃） |
| 货币 | — | 64 格 |

#### 3.5.2 装备槽位

| ID | 常量 | 名称 |
|----|------|------|
| 0 | EQUIPMENT_SLOT_HEAD | 头部 |
| 1 | EQUIPMENT_SLOT_NECK | 颈部 |
| 2 | EQUIPMENT_SLOT_SHOULDERS | 肩部 |
| 3 | EQUIPMENT_SLOT_BODY | 衬衣 |
| 4 | EQUIPMENT_SLOT_CHEST | 胸部 |
| 5 | EQUIPMENT_SLOT_WAIST | 腰部 |
| 6 | EQUIPMENT_SLOT_LEGS | 腿部 |
| 7 | EQUIPMENT_SLOT_FEET | 脚部 |
| 8 | EQUIPMENT_SLOT_WRISTS | 手腕 |
| 9 | EQUIPMENT_SLOT_HANDS | 手部 |
| 10 | EQUIPMENT_SLOT_FINGER1 | 戒指1 |
| 11 | EQUIPMENT_SLOT_FINGER2 | 戒指2 |
| 12 | EQUIPMENT_SLOT_TRINKET1 | 饰品1 |
| 13 | EQUIPMENT_SLOT_TRINKET2 | 饰品2 |
| 14 | EQUIPMENT_SLOT_BACK | 背部 |
| 15 | EQUIPMENT_SLOT_MAINHAND | 主手 |
| 16 | EQUIPMENT_SLOT_OFFHAND | 副手 |
| 17 | EQUIPMENT_SLOT_RANGED | 远程 |
| 18 | EQUIPMENT_SLOT_TABARD | 战袍 |

#### 3.5.3 物品操作流程

**物品拾取**:
```
LootItem
    ├── CanStoreItem()    # 检查背包空间
    ├── StoreItem()       # 放入背包
    ├── Item::CreateItem()  # 创建物品实例
    ├── INSERT item_instance
    ├── INSERT character_inventory
    └── 发送更新包给客户端
```

**装备物品**:
```
EquipItem
    ├── CanEquipItem()    # 检查是否可装备（等级/职业/种族需求）
    ├── 取消当前位置
    ├── 装备到目标槽位
    ├── 更新 character_inventory
    ├── 重新计算属性
    └── 发送更新包
```

### 3.6 宠物系统

#### 3.6.1 宠物类型

| 类型 | 说明 |
|------|------|
| Hunter Pet | 猎人宠物，永久保存 |
| Warlock/DK Summon | 术士/死亡骑士召唤物，临时 |
| Guardian | 守护者，临时保护 |
| Minion | 仆从，临时控制 |

#### 3.6.2 宠物功能

- 经验值和升级系统
- 快乐度系统（猎人宠物）
- 宠物天赋
- 自动施法管理
- 宠物稳定（存放闲置宠物）
- 宠物名字和变格名

### 3.7 任务系统

#### 3.7.1 任务状态追踪

`character_queststatus` 表中的 `status` 字段：
- 0: 未完成（进行中）
- 1: 已完成
- 3: 失败

#### 3.7.2 任务目标类型

- `mobcount1-4`: 击杀怪物（最多 4 种）
- `itemcount1-6`: 收集物品（最多 6 种）
- `playercount`: 击杀玩家（PvP 任务）
- `explored`: 探索区域

#### 3.7.3 任务冷却

| 类型 | 表名 | 重置周期 |
|------|------|----------|
| 每日 | `character_queststatus_daily` | 每日重置 |
| 每周 | `character_queststatus_weekly` | 每周重置 |
| 每月 | `character_queststatus_monthly` | 每月重置 |
| 赛季 | `character_queststatus_seasonal` | 赛季重置 |

### 3.8 社交系统

#### 3.8.1 好友/忽略列表

`character_social` 表:
- `flags` 字段标记类型：
  - `1`: 好友
  - `2`: 忽略
- `note` 字段：好友备注（最多 48 字符）

#### 3.8.2 管理器

**SocialMgr** (`src/server/game/Entities/Player/SocialMgr.cpp/h`):
- 管理好友/忽略列表
- 处理在线状态变更通知
- 处理社交消息

---

## 4. 模块间交互与数据流

### 4.1 关键关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        acore_auth 数据库                     │
│  ┌──────────┐  ┌───────────────┐  ┌──────────────┐         │
│  │  account  │  │ account_access│  │account_banned│  ...    │
│  │  (账号)   │  │  (GM 等级)    │  │  (封禁)      │         │
│  └─────┬────┘  └───────────────┘  └──────────────┘         │
│        │ 1:N                                                 │
└────────┼─────────────────────────────────────────────────────┘
         │
         │ account.id = characters.account
         │
┌────────┼─────────────────────────────────────────────────────┐
│        ▼            acore_characters 数据库                   │
│  ┌───────────┐    ┌──────────────────┐  ┌──────────────┐   │
│  │characters │◄───│character_inventory│  │character_pet │   │
│  │ (角色)    │    │    (物品位置)     │  │  (宠物)      │   │
│  └─────┬─────┘    └────────┬─────────┘  └──────────────┘   │
│        │ 1:N               │ 1:N                            │
│  ┌─────┴──────┐  ┌─────────┴──────┐  ┌────────────────┐   │
│  │character_  │  │ item_instance   │  │character_spell │   │
│  │queststatus │  │  (物品实例)     │  │character_skills│   │
│  │character_  │  │character_aura   │  │character_      │   │
│  │reputation  │  │character_social │  │talent          │   │
│  └────────────┘  └────────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**运行时对象关系**:

```
WorldSocket (TCP 连接)
    │
    ▼
WorldSession (会话)
    ├── AccountId = 123
    ├── Security = SEC_PLAYER
    ├── RBACData = { permissions: [...] }
    ├── AccountData[8] = { ... }
    ├── Tutorials[8] = { ... }
    │
    └── Player* = new Player()
            ├── guid = ObjectGuid(HighGuid::Player, 456)
            ├── class = CLASS_WARRIOR, race = RACE_HUMAN
            ├── level = 80
            ├── items[...] = { Item*, Item*, ... }
            ├── spells = { 123, 456, ... }
            ├── skills = { ... }
            ├── quests = { ... }
            ├── reputation = { ... }
            └── social = SocialMgr
```

### 4.2 账号登录全流程

```
[客户端]                          [LoginServer]                    [WorldServer]
    │                                   │                                │
    │  1. SRP6 认证挑战                   │                                │
    ├─────►                              │                                │
    │                                   │                                │
    │  2. SRP6 认证响应                   │                                │
    ├─────►                              │                                │
    │                                   ├── 查询 account 表               │
    │                                   ├── 验证 salt/verifier            │
    │                                   ├── 检查 account_banned           │
    │                                   ├── 检查 ip_banned               │
    │                                   └── 生成 session_key              │
    │                                   │                                │
    │  3. 认证成功 + realm 列表           │                                │
    │◄──────┤                            │                                │
    │                                   │                                │
    │  4. 连接 WorldServer               │                                │
    ├────────────────────────────────────┼───────────────────────────────►│
    │                                   │                         查询 account
    │                                   │                         验证 session_key
    │                                   │                         加载 account_access (GM)
    │                                   │                         加载 rbac 权限
    │                                   │                         创建 WorldSession
    │                                   │                                │
    │  5. AuthResponse (SMSG_AUTH_RESPONSE)                              │
    │◄──────────────────────────────────┼────────────────────────────────┤
    │                                   │                                │
    │  6. CharEnum (获取角色列表)                                          │
    ├───────────────────────────────────┼───────────────────────────────►│
    │                                   │                         查询 characters 表
    │                                   │                         按 account 过滤
    │                                   │                         计算总角色数
    │  7. 角色列表 SMSG_CHAR_ENUM                                        │
    │◄──────────────────────────────────┼────────────────────────────────┤
```

### 4.3 角色创建全流程

```
[客户端]                                    [WorldServer]
    │                                           │
    │  1. CMSG_CHAR_CREATE (name, race, class,  │
    │     gender, skin, face, hair, etc.)        │
    ├───────────────────────────────────────────►│
    │                                           │
    │                              ┌────────────┤
    │                              │ 验证阶段：  │
    │                              │ 1. 名称检查 │
    │                              │   - 长度限制 │
    │                              │   - 非法字符 │
    │                              │   - 保留名  │
    │                              │   - 重名检查 │
    │                              │ 2. 种族/职业│
    │                              │   - 组合合法性│
    │                              │   - DK 需 55+│
    │                              │ 3. 角色数量 │
    │                              │   - 账号上限│
    │                              │   - realm 上限│
    │                              │ 4. 账号状态  │
    │                              │   - RBAC 权限│
    │                              │   - 封禁检查 │
    │                              │ 5. 脚本钩子  │
    │                              │   - CanAccount│
    │                              │     CreateChar│
    │                              └────────────┤
    │                                           │
    │                              ┌────────────┤
    │                              │ 创建阶段：  │
    │                              │ 1. 生成 GUID│
    │                              │ 2. 初始化属性│
    │                              │   - playercreateinfo │
    │                              │   - player_class_stats│
    │                              │   - playercreateinfo_item│
    │                              │   - playercreateinfo_spell│
    │                              │   - playercreateinfo_action│
    │                              │   - playercreateinfo_skills│
    │                              │ 3. 设置位置  │
    │                              │   - 出生点坐标│
    │                              │ 4. 保存到 DB │
    │                              │   - INSERT characters│
    │                              │   - INSERT character_inventory│
    │                              │   - INSERT item_instance│
    │                              │   - INSERT character_action│
    │                              │   - INSERT character_spell│
    │                              │   - INSERT character_skills│
    │                              │   - INSERT character_homebind│
    │                              │ 5. 更新角色计数│
    │                              │   - UPDATE realmcharacters│
    │                              │   - 更新 CharacterCache│
    │                              └────────────┤
    │                                           │
    │  2. SMSG_CHAR_CREATE (成功/失败)           │
    │◄──────────────────────────────────────────┤
```

### 4.4 角色进入世界全流程

```
[客户端]                                    [WorldServer]
    │                                           │
    │  1. CMSG_PLAYER_LOGIN (guid)              │
    ├───────────────────────────────────────────►│
    │                                           │
    │                              ┌────────────┤
    │                              │ 预检查：    │
    │                              │ 1. 角色归属验证│
    │                              │   - guid 属于当前account│
    │                              │   - 查询 characters 表│
    │                              │ 2. 在线状态检查│
    │                              │   - 其他地方已登录│
    │                              │ 3. 封禁检查  │
    │                              └────────────┤
    │                                           │
    │                              ┌────────────┤
    │                              │ 异步加载：  │
    │                              │ LoginQueryHolder│
    │                              │   同时查询:  │
    │                              │   - characters 基础│
    │                              │   - character_inventory│
    │                              │   - item_instance│
    │                              │   - character_spell│
    │                              │   - character_skills│
    │                              │   - character_aura│
    │                              │   - character_queststatus│
    │                              │   - character_reputation│
    │                              │   - character_action│
    │                              │   - character_talent│
    │                              │   - character_glyphs│
    │                              │   - account_data│
    │                              │   - account_tutorial│
    │                              │   - guild member│
    │                              │   - arena team member│
    │                              │   - pet (异步) │
    │                              └────────────┤
    │                                           │
    │                              ┌────────────┤
    │                              │ 初始化：    │
    │                              │ 1. Player::LoadFromDB│
    │                              │   - 构建物品系统│
    │                              │   - 构建法术书  │
    │                              │   - 恢复任务进度│
    │                              │   - 恢复声望    │
    │                              │   - 恢复动作栏  │
    │                              │   - 恢复天赋    │
    │                              │ 2. 设置 Session│
    │                              │   - session->SetPlayer│
    │                              │   - player->SetSession│
    │                              │ 3. 加入世界    │
    │                              │   - Map::AddToMap│
    │                              │   - 加载周围对象│
    │                              │ 4. 加载宠物    │
    │                              │   - Pet::LoadPetFromDB│
    │                              │ 5. 更新在线状态│
    │                              │   - UPDATE characters SET online=1│
    │                              │ 6. 发送初始数据│
    │                              │   - SMSG_LOGIN_VERIFY_WORLD│
    │                              │   - SMSG_ACCOUNT_DATA_TIMES│
    │                              │   - SMSG_TUTORIAL_FLAGS│
    │                              │   - SMSG_INIT_WORLD_STATES│
    │                              │   - SMSG_BINDPOINTUPDATE│
    │                              │   - SMSG_TIME_SYNC_REQ│
    │                              │   - 角色周围对象│
    │                              └────────────┤
    │                                           │
    │  2. 进入游戏世界                           │
    │◄──────────────────────────────────────────┤
```

### 4.5 角色删除与账号删除

#### 角色删除

```
Player::DeleteFromDB()
    ├── 1. 更新 realmcharacters 角色计数
    ├── 2. 删除公会成员记录
    ├── 3. 删除竞技场队伍记录
    ├── 4. 删除角色数据（事务）
    │   ├── DELETE characters
    │   ├── DELETE character_account_data
    │   ├── DELETE character_achievement
    │   ├── DELETE character_achievement_progress
    │   ├── DELETE character_action
    │   ├── DELETE character_aura
    │   ├── DELETE character_banned
    │   ├── DELETE character_declinedname
    │   ├── DELETE character_equipmentsets
    │   ├── DELETE character_gifts
    │   ├── DELETE character_glyphs
    │   ├── DELETE character_homebind
    │   ├── DELETE character_instance
    │   ├── DELETE character_inventory
    │   ├── DELETE character_pet
    │   ├── DELETE character_pet_declinedname
    │   ├── DELETE character_queststatus (所有类型)
    │   ├── DELETE character_queststatus_rewarded
    │   ├── DELETE character_reputation
    │   ├── DELETE character_skills
    │   ├── DELETE character_social
    │   ├── DELETE character_spell
    │   ├── DELETE character_spell_cooldown
    │   ├── DELETE character_stats
    │   ├── DELETE character_talent
    │   ├── DELETE item_instance (WHERE owner_guid)
    │   ├── DELETE pet_aura
    │   ├── DELETE pet_spell
    │   └── DELETE pet_spell_cooldown
    ├── 5. 清理 CharacterCache
    └── 6. 如果有邮件，返回物品给发件人
```

#### 账号删除（级联）

见 [2.4.2 账号删除流程](#242-账号删除流程)，会先逐个删除所有角色。

---

## 5. PlayerDump 系统 — 角色数据完整导出/导入

PlayerDump 是 AzerothCore 内置的**角色数据导出/导入工具**，它能将一个角色的**全部关联数据**（涉及 32 张数据库表）完整序列化为 SQL 语句，并支持导入到另一个账号（甚至另一个服务器实例）。分析 PlayerDump 的代码是理解 Character 模块数据库表结构和表间依赖关系的最佳途径。

**核心源码**: `src/server/game/Tools/PlayerDump.h` / `PlayerDump.cpp`

### 5.1 概述与 GM 命令

PlayerDump 提供三个 GM 命令：

| 命令 | 说明 | RBAC 权限 |
|------|------|-----------|
| `.pdump write <file> <player>` | 将角色导出到文件 | `RBAC_PERM_COMMAND_PDUMP_WRITE` |
| `.pdump load <file> <account> [name] [guid]` | 从文件导入角色到指定账号 | `RBAC_PERM_COMMAND_PDUMP_LOAD` |
| `.pdump copy <player> <account> [name] [guid]` | 将在线角色直接复制到指定账号（write + load 一步完成） | `RBAC_PERM_COMMAND_PDUMP_COPY` |

**命令脚本**: `src/server/scripts/Commands/cs_character.cpp:43-47`

### 5.2 涉及的 32 张数据库表全表分类

PlayerDump 定义了一个有序的表数组 `DumpTables[]`（`PlayerDump.cpp:88-121`），严格按照依赖关系排序。这是理解角色数据全貌的最权威列表：

#### 5.2.1 按类型分组

**DTT_CHARACTER — 角色主表（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 1 | `characters` | 角色核心数据（名称、种族、职业、等级、位置、属性等） |

**DTT_CHAR_TABLE — 角色附属表，以 `guid` 为外键（16 张）**

| # | 表名 | 说明 |
|---|------|------|
| 2 | `character_account_data` | 账号级绑定数据（UI 布局、插件配置等） |
| 3 | `character_achievement` | 已完成成就 |
| 4 | `character_achievement_progress` | 成就进度（未完成的条件） |
| 5 | `character_action` | 动作栏配置（按钮对应技能/宏/物品） |
| 6 | `character_aura` | 登出时残留的 Aura 效果 |
| 7 | `character_declinedname` | 俄语变格名称（仅俄语客户端使用） |
| 8 | `character_glyphs` | 雕文选择（6 个雕文槽位） |
| 9 | `character_homebind` | 炉石绑定位置 |
| 10 | `character_queststatus` | 进行中/已完成/失败的 quest 进度 |
| 11 | `character_queststatus_daily` | 每日 quest 重置时间 |
| 12 | `character_queststatus_weekly` | 每周 quest 重置时间 |
| 13 | `character_queststatus_monthly` | 每月 quest 重置时间 |
| 14 | `character_queststatus_seasonal` | 赛季 quest 重置时间 |
| 15 | `character_queststatus_rewarded` | 历史上已完成的 quest 列表 |
| 16 | `character_reputation` | 各阵营声望值 |
| 17 | `character_skills` | 技能等级（专业、武器技能等） |
| 18 | `character_spell` | 已学法术列表（按天赋专精区分） |
| 19 | `character_spell_cooldown` | 法术冷却时间 |
| 20 | `character_talent` | 天赋点分配 |

**DTT_EQSET_TABLE — 装备搭配方案（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 8 | `character_equipmentsets` | 保存的装备搭配（可含 item0~item18 共 19 件装备引用） |

**DTT_INVENTORY — 背包物品映射（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 11 | `character_inventory` | 角色 → 背包/装备槽位 → 物品 GUID 的映射关系 |

**DTT_MAIL — 邮件（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 24 | `mail` | 角色的收件箱邮件 |

**DTT_MAIL_ITEM — 邮件附件（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 25 | `mail_items` | 邮件中的物品附件（关联邮件 ID 和物品 GUID） |

**DTT_ITEM — 物品实例（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 29 | `item_instance` | 物品实例数据（属性、附魔、耐久、宝石等） |

**DTT_ITEM_GIFT — 礼品（1 张）**

| # | 表名 | 说明 |
|---|------|------|
| 30 | `character_gifts` | 包装好的礼物物品 |

**DTT_PET — 宠物（2 张）**

| # | 表名 | 说明 |
|---|------|------|
| 12 | `character_pet` | 宠物实例（等级、经验、HP、技能、名称） |
| 13 | `character_pet_declinedname` | 宠物俄语变格名称 |

**DTT_PET_TABLE — 宠物附属表（3 张）**

| # | 表名 | 说明 |
|---|------|------|
| 26 | `pet_aura` | 宠物的 Aura 效果 |
| 27 | `pet_spell` | 宠物法术书 |
| 28 | `pet_spell_cooldown` | 宠物法术冷却 |

#### 5.2.2 完整导出顺序

表在 `DumpTables[]` 中的顺序**至关重要**——必须先导出父表，再导出子表，加载时才能正确解析 GUID 依赖：

```
序号  表名                            类型
───────────────────────────────────────────────────────
 1   characters                      DTT_CHARACTER        ← 角色主表
 2   character_account_data          DTT_CHAR_TABLE
 3   character_achievement           DTT_CHAR_TABLE
 4   character_achievement_progress  DTT_CHAR_TABLE
 5   character_action                DTT_CHAR_TABLE
 6   character_aura                  DTT_CHAR_TABLE
 7   character_declinedname          DTT_CHAR_TABLE
 8   character_equipmentsets         DTT_EQSET_TABLE
 9   character_glyphs                DTT_CHAR_TABLE
10   character_homebind              DTT_CHAR_TABLE
11   character_inventory             DTT_INVENTORY        ← 收集 item guids
12   character_pet                   DTT_PET              ← 收集 pet guids
13   character_pet_declinedname      DTT_PET
14   character_queststatus           DTT_CHAR_TABLE
15   character_queststatus_daily     DTT_CHAR_TABLE
16   character_queststatus_weekly    DTT_CHAR_TABLE
17   character_queststatus_monthly   DTT_CHAR_TABLE
18   character_queststatus_seasonal  DTT_CHAR_TABLE
19   character_queststatus_rewarded  DTT_CHAR_TABLE
20   character_reputation            DTT_CHAR_TABLE
21   character_skills                DTT_CHAR_TABLE
22   character_spell                 DTT_CHAR_TABLE
23   character_spell_cooldown        DTT_CHAR_TABLE
24   character_talent                DTT_CHAR_TABLE
25   mail                            DTT_MAIL             ← 收集 mail ids
26   mail_items                      DTT_MAIL_ITEM        ← 依赖 mail ids, 收集 item guids
27   pet_aura                        DTT_PET_TABLE       ← 依赖 pet guids
28   pet_spell                       DTT_PET_TABLE       ← 依赖 pet guids
29   pet_spell_cooldown              DTT_PET_TABLE       ← 依赖 pet guids
30   item_instance                   DTT_ITEM             ← 依赖 item guids
31   character_gifts                 DTT_ITEM_GIFT        ← 依赖 item guids
```

### 5.3 GUID 类型与重映射机制

PlayerDump 在导入时需要**重新映射所有 GUID**，以避免与目标服务器上的现有数据冲突。代码定义了以下 GUID 类型（`PlayerDump.cpp:32-46`）：

| GUID 类型 | 常量 | 位宽 | 说明 | 生成源 |
|-----------|------|------|------|--------|
| `GUID_TYPE_ACCOUNT` | 0 | 32 位 | 账号 ID | 导入时替换为目标账号 ID |
| `GUID_TYPE_CHAR` | 1 | 32 位 | 角色 GUID | 自动分配新的角色 GUID |
| `GUID_TYPE_PET` | 2 | 32 位 | 宠物 ID | `sObjectMgr->_hiPetNumber` 自增 |
| `GUID_TYPE_MAIL` | 3 | 32 位 | 邮件 ID | `sObjectMgr->_mailId` 自增 |
| `GUID_TYPE_ITEM` | 4 | 32 位 | 物品 GUID | `HighGuid::Item` Generator 自增 |
| `GUID_TYPE_EQUIPMENT_SET` | 5 | 64 位 | 装备搭配 ID | `sObjectMgr->_equipmentSetGuid` 自增 |
| `GUID_TYPE_NULL` | 6 | — | 设为 NULL | 用于软删除字段 |

**GUID 重映射流程**（`PlayerDump.cpp:806-816`）：

```cpp
// 导入前获取各类型 GUID 起始偏移量
ObjectGuid::LowType itemLowGuidOffset = sObjectMgr->GetGenerator<HighGuid::Item>().GetNextAfterMaxUsed();
ObjectGuid::LowType mailLowGuidOffset = sObjectMgr->_mailId;
ObjectGuid::LowType petLowGuidOffset = sObjectMgr->_hiPetNumber;
uint64 equipmentSetGuidOffset = sObjectMgr->_equipmentSetGuid;

// 导入时使用 map 保存 old_guid -> new_guid 的映射
std::map<ObjectGuid::LowType, ObjectGuid::LowType> items;
std::map<ObjectGuid::LowType, ObjectGuid::LowType> mails;
std::map<ObjectGuid::LowType, ObjectGuid::LowType> petIds;
std::map<uint64, uint64> equipmentSetIds;
```

### 5.4 各表的 GUID 依赖关系详解

`InitializeTables()` 函数（`PlayerDump.cpp:253-380`）在服务器启动时通过 `DESCRIBE` 查询动态获取每张表的所有列，并标记需要重映射的依赖字段。以下是各表的完整依赖关系：

#### characters 表

| 字段 | GUID 类型 | 导入处理 |
|------|-----------|----------|
| `guid` | `GUID_TYPE_CHAR` | 替换为新角色 GUID |
| `account` | `GUID_TYPE_ACCOUNT` | 替换为目标账号 ID |
| `deleteInfos_Account` | `GUID_TYPE_NULL` | 设为 NULL（清除软删除信息） |
| `deleteInfos_Name` | `GUID_TYPE_NULL` | 设为 NULL |
| `deleteDate` | `GUID_TYPE_NULL` | 设为 NULL |

#### character_equipmentsets 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_CHAR` | 所属角色 |
| `setguid` | `GUID_TYPE_EQUIPMENT_SET` | 装备搭配唯一 ID（64 位） |
| `item0` ~ `item18` | `GUID_TYPE_ITEM` | 搭配中的 19 件装备物品引用 |

#### character_inventory 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_CHAR` | 所属角色 |
| `bag` | `GUID_TYPE_ITEM` | 背包物品 GUID（0 表示主背包） |
| `item` | `GUID_TYPE_ITEM` | 物品 GUID |

> 注：`bag` 和 `item` 字段允许为 0（`allowZero=true`），表示空槽位或主背包根目录。

#### mail 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `id` | `GUID_TYPE_MAIL` | 邮件唯一 ID |
| `receiver` | `GUID_TYPE_CHAR` | 收件人角色 GUID |

#### mail_items 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `mail_id` | `GUID_TYPE_MAIL` | 所属邮件 ID |
| `item_guid` | `GUID_TYPE_ITEM` | 附件物品 GUID |
| `receiver` | `GUID_TYPE_CHAR` | 收件人角色 GUID |

#### item_instance 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_ITEM` | 物品唯一 GUID |
| `owner_guid` | `GUID_TYPE_CHAR` | 所有者角色 GUID |

#### character_gifts 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_CHAR` | 接收者角色 GUID |
| `item_guid` | `GUID_TYPE_ITEM` | 礼品物品 GUID |

#### character_pet / character_pet_declinedname 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `id` | `GUID_TYPE_PET` | 宠物唯一 ID |
| `owner` | `GUID_TYPE_CHAR` | 主人角色 GUID |

#### pet_aura / pet_spell / pet_spell_cooldown 表

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_PET` | 所属宠物 ID |

#### 其他 DTT_CHAR_TABLE 类型的表

所有 `character_achievement`, `character_action`, `character_aura`, `character_spell` 等表，只有一个依赖字段：

| 字段 | GUID 类型 | 说明 |
|------|-----------|------|
| `guid` | `GUID_TYPE_CHAR` | 所属角色 GUID |

### 5.5 导出流程 (Writer)

**类**: `PlayerDumpWriter`（`PlayerDump.h:74`）

```
PlayerDumpWriter::GetDump(guid)
    │
    ├── 1. PopulateGuids(guid)        ← 第一阶段：收集所有关联 GUID
    │       │
    │       └── 查询 4 张 BaseTable：
    │           ├── SELECT id FROM character_pet WHERE owner = guid     → _pets
    │           ├── SELECT id FROM mail WHERE receiver = guid           → _mails
    │           ├── SELECT guid FROM item_instance WHERE owner_guid = guid → _items
    │           └── SELECT setguid FROM character_equipmentsets WHERE guid = guid → _itemSets
    │
    └── 2. 循环遍历 31 张 DumpTables    ← 第二阶段：逐表生成 INSERT 语句
            │
            ├── DTT_CHARACTER:
            │   └── SELECT * FROM characters WHERE guid = guid
            │       额外检查: deleteInfos_Account 不为空则拒绝导出（已软删除的角色）
            │
            ├── DTT_CHAR_TABLE × 16:
            │   └── SELECT * FROM <table> WHERE guid = guid
            │
            ├── DTT_EQSET_TABLE:
            │   └── SELECT * FROM character_equipmentsets WHERE setguid IN (_itemSets)
            │
            ├── DTT_INVENTORY:
            │   └── SELECT * FROM character_inventory WHERE guid = guid
            │
            ├── DTT_PET × 2:
            │   └── SELECT * FROM <table> WHERE owner = guid
            │
            ├── DTT_MAIL:
            │   └── SELECT * FROM mail WHERE receiver = guid
            │
            ├── DTT_MAIL_ITEM:
            │   └── SELECT * FROM mail_items WHERE mail_id IN (_mails)
            │
            ├── DTT_PET_TABLE × 3:
            │   └── SELECT * FROM <table> WHERE guid IN (_pets)
            │
            ├── DTT_ITEM:
            │   └── SELECT * FROM item_instance WHERE guid IN (_items)
            │
            └── DTT_ITEM_GIFT:
                └── SELECT * FROM character_gifts WHERE item_guid IN (_items)
```

**导出文件格式示例**:
```sql
IMPORTANT NOTE: THIS DUMPFILE IS MADE FOR USE WITH THE 'PDUMP' COMMAND ONLY...
IMPORTANT NOTE: DO NOT apply it directly - it will irreversibly DAMAGE...

INSERT INTO `characters` (`guid`, `account`, `name`, `race`, `class`, ...) VALUES ('456', '1', 'MyChar', '1', '1', ...);
INSERT INTO `character_inventory` (`guid`, `bag`, `slot`, `item`) VALUES ('456', '0', '0', '78901');
INSERT INTO `character_inventory` (`guid`, `bag`, `slot`, `item`) VALUES ('456', '0', '1', '78902');
INSERT INTO `item_instance` (`guid`, `itemEntry`, `owner_guid`, ...) VALUES ('78901', '19019', '456', ...);
INSERT INTO `item_instance` (`guid`, `itemEntry`, `owner_guid`, ...) VALUES ('78902', '2586', '456', ...);
INSERT INTO `character_pet` (`id`, `entry`, `owner`, ...) VALUES ('1001', '14325', '456', ...);
...
```

### 5.6 导入流程 (Reader)

**类**: `PlayerDumpReader`（`PlayerDump.h:94`）

```
PlayerDumpReader::LoadDump(input, account, name, guid)
    │
    ├── 1. 预检查
    │   ├── 账号角色数 < 10（否则返回 DUMP_TOO_MANY_CHARS）
    │   ├── 确定目标角色 GUID（使用 guid 参数或自动分配下一个可用 GUID）
    │   ├── 验证角色名（如冲突则清空，导入时使用原名称 + 设置 AT_LOGIN_RENAME）
    │   └── 准备各类型 GUID 偏移量
    │
    ├── 2. 逐行解析 SQL
    │   └── 对每一行 INSERT INTO <table> ...:
    │       ├── 提取表名，匹配 DumpTableType
    │       ├── ValidateFields(): 验证列名是否与当前数据库结构一致
    │       ├── 遍历该表所有 IsDependentField 的列:
    │       │   ├── GUID_TYPE_ACCOUNT → 替换为目标账号 ID
    │       │   ├── GUID_TYPE_CHAR    → 替换为新角色 GUID
    │       │   ├── GUID_TYPE_PET     → RegisterNewGuid() 分配新宠物 ID
    │       │   ├── GUID_TYPE_MAIL    → RegisterNewGuid() 分配新邮件 ID
    │       │   ├── GUID_TYPE_ITEM    → RegisterNewGuid() 分配新物品 ID
    │       │   ├── GUID_TYPE_EQUIPMENT_SET → RegisterNewGuid() 分配新套装 ID
    │       │   └── GUID_TYPE_NULL    → 替换为 "NULL"
    │       ├── DTT_CHARACTER 额外处理:
    │       │   ├── 读取 race, class, gender, level
    │       │   ├── 如果名称冲突，拼接临时名称 + 设置 at_login = 1 (AT_LOGIN_RENAME)
    │       │   └── 否则使用指定名称
    │       ├── FixNULLfields(): 将 'NULL' 转为 NULL
    │       └── trans->Append(line): 加入数据库事务
    │
    ├── 3. 提交事务
    │   └── CharacterDatabase.CommitTransaction(trans)
    │
    └── 4. 后续处理
        ├── 更新 CharacterCache
        ├── 推进各 GUID 生成器:
        │   ├── Item Generator += items.size()
        │   ├── _mailId += mails.size()
        │   ├── _hiPetNumber += petIds.size()
        │   └── _equipmentSetGuid += equipmentSetIds.size()
        └── 更新 RealmCharCount
```

**RegisterNewGuid 算法**（`PlayerDump.cpp:493-502`）:

```cpp
// 使用 map 维护 old_guid -> new_guid 的映射
// 新 GUID = offset + 已映射数量
T newguid = guidOffset + T(guidMap.size());
guidMap.emplace(oldGuid, newguid);
```

这保证了**同一物品/宠物/邮件在不同表中的 GUID 引用保持一致**。

### 5.7 characters 表补充字段

PlayerDump 的源码揭示了 `characters` 表中一些未在基础文档中详述的重要字段：

#### 软删除字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `deleteInfos_Account` | `int unsigned` | 执行删除操作的账号 ID（0=未删除） |
| `deleteInfos_Name` | `varchar(12)` | 执行删除操作的角色名 |
| `deleteDate` | `int unsigned` | 删除时间戳（Unix 时间） |

角色删除时（非立即删除模式），这些字段被填充，角色保留在数据库中但标记为已删除。PlayerDump 拒绝导出已软删除的角色（`PlayerDump.cpp:681-686`）。

#### 登录标记字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `at_login` | `int unsigned` | 登录时需要执行的操作标记（见 5.9 节） |

### 5.8 邮件系统表

PlayerDump 揭示了邮件系统的表结构，这是角色数据的重要组成部分：

#### mail 表

| 字段 | 类型 | GUID 类型 | 说明 |
|------|------|-----------|------|
| `id` | `int unsigned` | `GUID_TYPE_MAIL` | 邮件 ID |
| `messageType` | `tinyint unsigned` | — | 邮件类型（普通/COD/AUCTION） |
| `stationery` | `tinyint unsigned` | — | 信纸类型 |
| `mailTemplateId` | `int unsigned` | — | 邮件模板 ID |
| `sender` | `int unsigned` | — | 发件人 GUID |
| `receiver` | `int unsigned` | `GUID_TYPE_CHAR` | 收件人 GUID |
| `subject` | `varchar(255)` | — | 主题 |
| `body` | `text` | — | 正文 |
| `has_items` | `tinyint unsigned` | — | 是否有附件 |
| `expire_time` | `int unsigned` | — | 过期时间 |
| `deliver_time` | `int unsigned` | — | 投递时间 |
| `money` | `int unsigned` | — | 附加金币 |
| `cod` | `int unsigned` | — | 货到付款金额 |
| `checked` | `tinyint unsigned` | — | 查看状态 |

#### mail_items 表

| 字段 | 类型 | GUID 类型 | 说明 |
|------|------|-----------|------|
| `mail_id` | `int unsigned` | `GUID_TYPE_MAIL` | 所属邮件 ID |
| `item_guid` | `int unsigned` | `GUID_TYPE_ITEM` | 附件物品 GUID |
| `receiver` | `int unsigned` | `GUID_TYPE_CHAR` | 收件人角色 GUID |

> **GUID 传递链**: `mail.receiver`(角色) → `mail_items.mail_id`(邮件) → `mail_items.item_guid`(物品) → `item_instance.guid`(物品数据)

### 5.9 AtLoginFlags 登录标记

`characters.at_login` 字段使用位掩码标记角色登录时需要执行的特殊操作：

**定义**: `src/server/game/Entities/Player/Player.h:583`

| 位标记 | 值 | 常量 | 说明 |
|--------|-----|------|------|
| bit 0 | `0x01` | `AT_LOGIN_RENAME` | 登录时强制重命名 |
| bit 1 | `0x02` | `AT_LOGIN_RESET_SPELLS` | 重置法术书 |
| bit 2 | `0x04` | `AT_LOGIN_RESET_TALENTS` | 重置天赋 |
| bit 3 | `0x08` | `AT_LOGIN_CUSTOMIZE` | 登录时打开外观定制 |
| bit 4 | `0x10` | `AT_LOGIN_RESET_PET_TALENTS` | 重置宠物天赋 |
| bit 5 | `0x20` | `AT_LOGIN_FIRST` | 首次登录标记 |
| bit 6 | `0x40` | `AT_LOGIN_CHANGE_FACTION` | 阵营变更 |
| bit 7 | `0x80` | `AT_LOGIN_CHANGE_RACE` | 种族变更 |
| bit 8 | `0x100` | `AT_LOGIN_RESET_AP` | 重置天赋点 |
| bit 9 | `0x200` | `AT_LOGIN_RESET_ARENA` | 重置竞技场积分 |
| bit 10 | `0x400` | `AT_LOGIN_CHECK_ACHIEVS` | 检查成就 |
| bit 11 | `0x800` | `AT_LOGIN_RESURRECT` | 复活 |

**PlayerDump 中的使用**: 导入角色时如果名称冲突，自动设置 `at_login = 1`（`AT_LOGIN_RENAME`），强制玩家首次登录时重命名。

### 5.10 从 Dump 视角看角色数据完整性

PlayerDump 的 32 张表覆盖了角色的**绝大部分持久化数据**，但也有一些数据**不在 Dump 范围内**：

**包含在 Dump 中**:

| 类别 | 涉及的表 |
|------|----------|
| 角色基础 | `characters`, `character_homebind`, `character_declinedname` |
| 物品装备 | `character_inventory`, `item_instance`, `character_equipmentsets`, `character_gifts` |
| 法术/技能/天赋 | `character_spell`, `character_spell_cooldown`, `character_skills`, `character_talent`, `character_glyphs` |
| 任务进度 | `character_queststatus` (6 张表) |
| 声望 | `character_reputation` |
| 成就 | `character_achievement`, `character_achievement_progress` |
| 宠物 | `character_pet`, `character_pet_declinedname`, `pet_aura`, `pet_spell`, `pet_spell_cooldown` |
| 邮件 | `mail`, `mail_items` |
| 动作栏 | `character_action` |
| 登出 Aura | `character_aura` |
| 账号数据 | `character_account_data` |

**不在 Dump 中**（源码注释 `/// @todo` 提到）:

| 类别 | 说明 | 存储位置 |
|------|------|----------|
| 副本锁定 | `character_instance` | 需要手动处理或重新绑定 |
| 公会成员 | `guild_member` | 需要重新加入公会 |
| 竞技场队伍 | `arena_team_member` | 需要重新加入队伍 |
| 好友/忽略列表 | `character_social` | 社交关系不随角色迁移 |
| 小队/团本进度 | 运行时数据 | 非持久化 |
| 教程进度 | `account_tutorial` | 属于账号级数据 |
| 银行物品 | 包含在 `character_inventory` 中 | bag 值在银行范围内即包含 |
| 拍卖行数据 | `auctionhouse` | 非角色直接关联 |

> 这意味着通过 pdump 迁移角色后，玩家将**丢失**副本锁定进度、公会关系和好友列表，需要重新获取。

---

## 6. Update Fields 系统

AzerothCore 使用 **Update Fields** 系统来同步服务端数据到客户端。每个实体类型有一组预定义的字段索引，通过位掩码标记哪些字段发生变化，增量发送给客户端。

### 6.1 核心字段分类

**Object 字段**:
- `OBJECT_FIELD_GUID`: 128 位全局唯一标识符
- `OBJECT_FIELD_ENTRY`: DBC 条目 ID
- `OBJECT_FIELD_SCALE_X`: 显示缩放

**Unit 字段**:
- `UNIT_FIELD_HEALTH` / `UNIT_FIELD_MAXHEALTH`: 生命值
- `UNIT_FIELD_POWER1-7`: 能量值（Mana, Rage, Focus, Energy, Happiness, Runes, Runic Power）
- `UNIT_FIELD_LEVEL`: 等级
- `UNIT_FIELD_FACTIONTEMPLATE`: 阵营模板
- `UNIT_FIELD_STAT0-4`: 基础属性（力量、敏捷、耐力、智力、精神）
- `UNIT_FIELD_BASEATTACKTIME`: 攻击速度

**Player 字段**:
- `PLAYER_FIELD_INV_SLOT_HEAD`: 装备栏（46 个字段）
- `PLAYER_FIELD_PACK_SLOT_1`: 背包栏（32 个字段）
- `PLAYER_FIELD_BANK_SLOT_1`: 银行栏（56 个字段）
- `PLAYER_FIELD_BANKBAG_SLOT_1`: 银行背包（14 个字段）
- `PLAYER_FIELD_CURRENCYTOKEN_SLOT_1`: 货币栏（64 个字段）
- `PLAYER_FIELD_COINAGE`: 金币
- `PLAYER_FIELD_KILLS`: PVP 击杀数
- `PLAYER_FIELD_LIFETIME_HONORABLE_KILLS`: 终生荣誉击杀
- `PLAYER_FIELD_DAILY_QUESTS_1`: 每日任务（25 个）
- `PLAYER_FIELD_GLYPH_SLOTS_1`: 雕文槽位（6 个）
- `PLAYER_FIELD_GLYPHS_1`: 雕文（6 个）
- `PLAYER_FIELD_ARENA_TEAM_INFO_1_1`: 竞技场队伍信息（21 个字段 × 3 队伍）

**Item 字段**:
- `ITEM_FIELD_OWNER`: 所有者 GUID
- `ITEM_FIELD_CONTAINED`: 所在容器 GUID
- `ITEM_FIELD_STACK_COUNT`: 堆叠数量
- `ITEM_FIELD_DURATION`: 持续时间
- `ITEM_FIELD_SPELL_CHARGES`: 法术充能
- `ITEM_FIELD_ENCHANTMENT`: 附魔数据（12 个槽位 × 3 个值）

---

## 7. 性能与安全设计

### 7.1 性能优化

| 策略 | 说明 |
|------|------|
| **CharacterCache** | 缓存角色基础数据（名称、等级、种族等），避免频繁查询 DB |
| **QueryHolder** | 角色加载时使用 `LoginQueryHolder` 合并多个查询，减少 DB 往返 |
| **Item Update Queue** | 物品变更不立即写入 DB，而是加入更新队列批量保存 |
| **Prepared Statements** | 所有数据库查询使用预处理语句，支持查询缓存 |
| **Periodic Save** | 角色数据定期自动保存，而非每次变更都写入 |
| **Selective Update** | Update Fields 系统只发送变化的部分，减少网络带宽 |
| **DBC Caching** | 静态数据（物品模板、法术信息等）从 DBC 文件加载并缓存在内存 |
| **RBAC In-Memory** | RBAC 权限数据加载到内存，权限检查不查数据库 |

### 7.2 安全机制

| 机制 | 说明 |
|------|------|
| **SRP6 密码** | 不存储明文密码，服务器不接触用户明文密码 |
| **TOTP 两步验证** | 支持 Google Authenticator 等两步验证 |
| **国家锁定** | 通过 `lock_country` 限制账号登录地区 |
| **IP 封禁** | `ip_banned` 表支持 IP 段封禁 |
| **账号封禁** | 支持临时和永久封禁 |
| **角色封禁** | 可单独封禁某个角色 |
| **禁言系统** | 支持临时禁言，记录原因和操作者 |
| **RBAC 权限** | 细粒度权限控制，800+ 个权限点 |
| **运动验证** | 客户端运动包经过验证（速度、传送检测） |
| **Warden** | 反作弊系统（`Warden.h`），检测客户端修改 |
| **数据包频率限制** | `PacketCounter` 限制数据包发送频率，防止洪水攻击 |
| **输入验证** | 聊天超链接验证、SQL 注入防护 |
| **角色归属验证** | 所有角色操作都验证角色属于当前账号 |
| **GM 等级分层** | 按 realm 粒度的 GM 等级控制 |

### 7.3 数据一致性

| 策略 | 说明 |
|------|------|
| **数据库事务** | 多表更新使用事务保证原子性 |
| **Logout Save** | 角色登出时强制保存所有未保存数据 |
| **Online Flag** | `characters.online` 字段防止角色被多次登录 |
| **realmcharacters** | 跨 realm 的角色数量统计确保一致性 |
| **GUID 全局唯一** | 角色 GUID 确保全局唯一，使用 `ObjectGuid` 类型 |

---

## 8. 扩展与定制指南

### 8.1 常见扩展点

#### 添加账号级别功能

```cpp
// 1. 在 AccountScript 中使用钩子
class MyAccountScript : public AccountScript
{
public:
    MyAccountScript() : AccountScript("MyAccountScript") {}

    void OnAccountLogin(uint32 accountId) override
    {
        // 账号登录时执行自定义逻辑
    }

    bool CanAccountCreateCharacter(uint32 accountId, uint8 race, uint8 class_) override
    {
        // 限制某些账号不能创建特定种族/职业
        return true; // 返回 false 阻止创建
    }
};

void AddSC_MyAccountScripts()
{
    new MyAccountScript();
}
```

#### 添加角色自定义字段

1. 在 `characters` 表添加列
2. 在 `Player` 类中添加成员变量
3. 在 `Player::LoadFromDB()` 中读取新字段
4. 在 `Player::SaveToDB()` 中保存新字段
5. 更新 `Player` 的 Update Fields（如需同步到客户端）

#### 添加 GM 命令

```cpp
// src/server/scripts/Commands/cs_my_command.cpp
class MyCommandScript : public CommandScript
{
public:
    ChatCommandTable GetCommands() const override
    {
        static ChatCommandTable myCommandTable =
        {
            { "mycommand", HandleMyCommand, RBAC_PERM_COMMAND_MY_COMMAND, Console::No },
        };
        static ChatCommandTable commandTable =
        {
            { "my", myCommandTable },
        };
        return commandTable;
    }

    static bool HandleMyCommand(ChatHandler* handler, const char* args)
    {
        // 命令实现
        return true;
    }
};
```

### 8.2 模块关键配置

角色创建相关配置（`worldserver.conf`）:
- `CharCreate.MinLevelForHeroicClass`: 创建英雄职业的最低等级
- `CharactersPerAccount`: 每账号最大角色数
- `CharactersPerRealm`: 每 realm 最大角色数
- `HeroicCharactersPerRealm`: 每 realm 最大英雄角色数

安全相关配置:
- `BoolCheckMovement`: 是否验证客户端运动
- `Battleground.CastDeserterOnAFKLeave`: BG 中 AFK 的惩罚
- `Warden.Enabled`: 是否启用反作弊

### 8.3 注意事项

1. **Player 类过于庞大**: `Player.h` 有 3071 行，`Player.cpp` 有 16423 行。任何修改都需要谨慎，避免引入 bug。
2. **数据库表间依赖**: 角色删除需要清理 30+ 张表，添加新表时需要同步更新 `DeleteFromDB`。
3. **缓存一致性**: 修改角色数据后，记得更新 `CharacterCache`。
4. **事务安全**: 涉及多表更新的操作必须使用数据库事务。
5. **RBAC 权限**: 添加新功能时，记得在 `rbac_permissions` 表中注册权限，并在 `RBAC.h` 中定义枚举。
6. **跨模块依赖**: `AccountMgr` 使用 `LoginDatabase`，`Player` 使用 `CharacterDatabase`，注意不要混淆数据库连接。

---

## 9. 角色重置 — 将角色恢复到 1 级初始状态

角色重置是指将现有角色**完全重置到刚创建时的状态**，包括等级、物品、任务、声望、技能等所有数据。这通常用于"洗号"或提供给玩家"重新开始"的功能。

### 9.1 角色创建时的初始数据来源

AzerothCore 使用 `acore_world` 数据库中的 `playercreateinfo*` 系列表定义每个种族/职业组合的初始数据：

| 表名 | 说明 | 关键字段 |
|------|------|---------|
| `playercreateinfo` | 出生点信息 | `race`, `class`, `map`, `zone`, `position_x/y/z`, `orientation` |
| `playercreateinfo_action` | 初始动作栏 | `race`, `class`, `button`, `action`, `type` |
| `playercreateinfo_item` | 初始物品（当前几乎为空） | `race`, `class`, `itemid`, `amount` |
| `playercreateinfo_skills` | 初始技能 | `race`, `class`, `skill`, `step`, `value`, `max` |
| `playercreateinfo_spell` | 初始法术（从 DBC） | `race`, `class`, `Spell`, `Note` |
| `playercreateinfo_spell_custom` | 自定义初始法术 | `race`, `class`, `Spell` |
| `CharStartOutfit` (DBC) | 起始装备配置 | `RaceID`, `ClassID`, `GenderID`, `ItemId[19]` |

**核心初始化逻辑**: `src/server/game/Entities/Player/Player.cpp:485-711`

```cpp
bool Player::Create(ObjectGuid::LowType guidlow, CharacterCreateInfo* createInfo)
{
    // 1. 获取 PlayerInfo (包含出生点、技能、法术、物品等)
    PlayerInfo const* info = sObjectMgr->GetPlayerInfo(createInfo->Race, createInfo->Class);

    // 2. 设置出生位置
    Relocate(info->positionX, info->positionY, info->positionZ, info->orientation);
    SetMap(sMapMgr->CreateMap(info->mapId, this));

    // 3. 初始化外观
    SetUInt32Value(PLAYER_BYTES, (createInfo->Skin | (createInfo->Face << 8) | ...));

    // 4. 初始化等级
    uint32 start_level = !IsClass(CLASS_DEATH_KNIGHT, CLASS_CONTEXT_INIT)
                             ? sWorld->getIntConfig(CONFIG_START_PLAYER_LEVEL)     // 普通职业 1 级
                             : sWorld->getIntConfig(CONFIG_START_HEROIC_PLAYER_LEVEL); // DK 55 级
    SetUInt32Value(UNIT_FIELD_LEVEL, start_level);

    // 5. 初始化属性
    InitStatsForLevel();
    InitTaxiNodesForLevel();
    InitGlyphsForLevel();
    InitTalentForLevel();
    InitPrimaryProfessions();

    // 6. 学习初始法术
    LearnDefaultSkills();     // 从 playercreateinfo_skills 表
    LearnCustomSpells();      // 从 playercreateinfo_spell_custom 表

    // 7. 添加初始动作栏
    for (PlayerCreateInfoActions::const_iterator action_itr = info->action.begin(); ...)
        addActionButton(action_itr->button, action_itr->action, action_itr->type);

    // 8. 添加初始装备
    if (CharStartOutfitEntry const* oEntry = GetCharStartOutfitEntry(...))
    {
        for (int j = 0; j < MAX_OUTFIT_ITEMS; ++j)
            StoreNewItemInBestSlots(oEntry->ItemId[j], 1);
    }

    // 9. 添加额外物品（playercreateinfo_item）
    for (PlayerCreateInfoItems::const_iterator item_id_itr = info->item.begin(); ...)
        StoreNewItemInBestSlots(item_id_itr->item_id, item_id_itr->item_amount);

    // 10. 设置初始金币、荣誉、竞技场积分
    SetUInt32Value(PLAYER_FIELD_COINAGE, !IsClass(CLASS_DEATH_KNIGHT, CLASS_CONTEXT_INIT)
                                             ? sWorld->getIntConfig(CONFIG_START_PLAYER_MONEY)
                                             : sWorld->getIntConfig(CONFIG_START_HEROIC_PLAYER_MONEY));
    SetHonorPoints(sWorld->getIntConfig(CONFIG_START_HONOR_POINTS));
    SetArenaPoints(sWorld->getIntConfig(CONFIG_START_ARENA_POINTS));

    // 11. 设置初始状态
    SetAtLoginFlag(AT_LOGIN_FIRST);  // 首次登录标记
    SetFullHealth();
    UpdateAllStats();

    return true;
}
```

### 9.2 需要重置的数据分类

完整重置角色到 1 级需要清理/重置以下数据：

#### 9.2.1 characters 表（角色主表）

| 字段 | 重置操作 | 说明 |
|------|----------|------|
| `level` | `UPDATE level = 1` (或 55 for DK) | 等级重置 |
| `xp` | `UPDATE xp = 0` | 经验值清零 |
| `money` | `UPDATE money = <初始金币>` | 普通职业 0，DK 初始金币（可配） |
| `totaltime`, `leveltime` | `UPDATE totaltime = 0, leveltime = 0` | 总在线时间重置 |
| `position_x/y/z`, `map`, `zone`, `orientation` | 查询 `playercreateinfo` 更新 | 传送回出生点 |
| `health` | 预留给 `SaveToDB` 处理 | 满血状态 |
| `power1` ~ `power7` | 预留给 `SaveToDB` 处理 | 各类能量满值 |
| `playerBytes`, `playerBytes2` | 保持不变 | 外观数据 |
| `at_login` | `UPDATE at_login = 0` | 清除登录标记 |
| `online` | 如在线需先踢出 | 确保离线 |

#### 9.2.2 物品相关表（核心）

| 表名 | 清理操作 | 说明 |
|------|----------|------|
| `character_inventory` | `DELETE WHERE guid = <角色GUID>` | 清空背包和装备栏 |
| `item_instance` | 通过 `character_inventory` 级联删除 | 所有物品实例被删除 |
| `character_equipmentsets` | `DELETE WHERE guid = <角色GUID>` | 删除保存的装备搭配 |
| `character_gifts` | `DELETE WHERE guid = <角色GUID>` | 删除礼物物品 |

#### 9.2.3 法术/技能/天赋相关表

| 表名 | 清理/重置操作 | 说明 |
|------|---------------|------|
| `character_spell` | `DELETE WHERE guid = <角色GUID>` | 清除已学法术（会由 `LearnDefaultSkills()` 重新添加） |
| `character_spell_cooldown` | `DELETE WHERE guid = <角色GUID>` | 清除法术冷却 |
| `character_skills` | `DELETE WHERE guid = <角色GUID>` | 清除技能等级 |
| `character_talent` | `DELETE WHERE guid = <角色GUID>` | 清除天赋点 |
| `character_glyphs` | `DELETE WHERE guid = <角色GUID>` | 清除雕文 |

#### 9.2.4 任务相关表

| 表名 | 清理/重置操作 | 说明 |
|------|---------------|------|
| `character_queststatus` | `DELETE WHERE guid = <角色GUID>` | 清除进行中/已完成/失败的 quest |
| `character_queststatus_rewarded` | `DELETE WHERE guid = <角色GUID>` | 清除历史完成记录 |
| `character_queststatus_daily` | `DELETE WHERE guid = <角色GUID>` | 清除每日任务计时 |
| `character_queststatus_weekly` | `DELETE WHERE guid = <角色GUID>` | 清除每周任务计时 |
| `character_queststatus_monthly` | `DELETE WHERE guid = <角色GUID>` | 清除每月任务计时 |
| `character_queststatus_seasonal` | `DELETE WHERE guid = <角色GUID>` | 清除赛季任务计时 |

#### 9.2.5 其他附属表

| 表名 | 清理/重置操作 | 说明 |
|------|---------------|------|
| `character_reputation` | `DELETE WHERE guid = <角色GUID>` | 清除声望值 |
| `character_achievement` | `DELETE WHERE guid = <角色GUID>` | 清除成就 |
| `character_achievement_progress` | `DELETE WHERE guid = <角色GUID>` | 清除成就进度 |
| `character_aura` | `DELETE WHERE guid = <角色GUID>` | 清除残留 Aura |
| `character_action` | 由 Player::Create() 重新添加 | 自动重置 |
| `character_social` | **保留** | 好友/忽略列表通常不重置 |
| `character_declinedname` | **保留** | 俄语变格名通常不重置 |

#### 9.2.6 宠物相关表

| 表名 | 清理/重置操作 | 说明 |
|------|---------------|------|
| `character_pet` | `DELETE WHERE owner = <角色GUID>` | 删除所有宠物实例 |
| `character_pet_declinedname` | 通过级联删除 | 宠物变格名 |
| `pet_aura` | 通过级联删除 | 宠物 Aura |
| `pet_spell` | 通过级联删除 | 宠物法术书 |
| `pet_spell_cooldown` | 通过级联删除 | 宠物法术冷却 |

#### 9.2.7 账号级数据（可选）

| 表名 | 是否重置 | 说明 |
|------|---------|------|
| `character_homebind` | **建议保留** | 炉石位置通常不重置 |
| `character_account_data` | **建议保留** | UI 布局、插件配置 |
| `account_tutorial` | **建议保留** | 教程进度 |

### 9.3 完整重置 SQL 脚本

以下是一个将角色重置到 1 级的完整 SQL 脚本，包含详细注释和操作步骤说明：

```sql
-- ========================================================================
-- AzerothCore 角色重置脚本
-- 功能：将指定角色重置到 1 级初始状态
-- 数据库：acore_characters
-- ========================================================================

-- === 步骤 0：变量定义 ===
-- 在实际使用时，将以下变量替换为实际值：
-- @char_guid      : 目标角色的 GUID（必填）
-- @account_id     : 账号 ID（可选，仅用于验证）
-- @target_race    : 目标角色的种族 ID（1=人类, 2=兽人, 3=矮人 等）
-- @target_class  : 目标角色的职业 ID（1=战士, 2=圣骑士, 3=猎人 等）

-- 示例变量赋值：
SET @char_guid = 456;        -- 要重置的角色 GUID
SET @target_race = 1;          -- 人类
SET @target_class = 1;         -- 战士

-- === 步骤 1：安全检查和准备工作 ===

-- 检查角色是否存在（如果不存在则提前终止）
SELECT COUNT(*) INTO @char_exists
FROM characters
WHERE guid = @char_guid;

-- 如果角色不存在，可以选择直接退出或报错
-- SELECT CONCAT('ERROR: Character with guid=', @char_guid, ' does not exist') AS Error;

-- 记录重置前的角色信息（用于日志/审计）
SELECT 
    name,
    race,
    class,
    level,
    xp,
    money,
    totaltime
INTO @old_name, @old_race, @old_class, @old_level, @old_xp, @old_money, @old_totaltime
FROM characters
WHERE guid = @char_guid;

-- 输出重置前信息（可选）
SELECT CONCAT('Resetting character: ', @old_name, 
              ' (Race:', @old_race, 
              ', Class:', @old_class, 
              ', Level:', @old_level, 
              ', XP:', @old_xp, 
              ', Money:', @old_money, 
              ', TotalTime:', @old_totaltime, ')') 
AS ResetInfo;

-- === 步骤 2：开始数据库事务 ===
-- 所有删除和更新操作都在一个事务中，确保原子性
START TRANSACTION;

-- === 步骤 3：确保玩家离线 ===
-- 说明：如果角色在线，标记为离线（WorldSession 会处理踢出）
-- 影响：避免数据不一致（正在修改时玩家在线）
UPDATE characters
SET online = 0
WHERE guid = @char_guid;

-- === 步骤 4：清理宠物系统 ===
-- 4.1 获取该角色的所有宠物 ID
CREATE TEMPORARY TABLE temp_pet_ids AS
SELECT id FROM character_pet WHERE owner = @char_guid;

-- 4.2 删除宠物相关数据（按照依赖顺序）
DELETE FROM pet_spell_cooldown      WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM pet_spell                 WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM pet_aura                  WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM character_pet_declinedname WHERE owner = @char_guid;
DELETE FROM character_pet             WHERE owner = @char_guid;

-- 4.3 清理临时表
DROP TEMPORARY TABLE IF EXISTS temp_pet_ids;

-- === 步骤 5：清理物品系统 ===
-- 说明：删除所有物品实例、装备搭配、背包、礼物
-- 依赖关系：character_inventory → item_instance（级联删除）
-- 注意：item_instance 表通过 character_inventory 的外键自动删除

-- 5.1 删除礼物物品
DELETE FROM character_gifts
WHERE guid = @char_guid;

-- 5.2 删除保存的装备搭配
DELETE FROM character_equipmentsets
WHERE guid = @char_guid;

-- 5.3 删除背包和装备栏中的所有物品
-- 说明：这会自动触发级联删除 item_instance 表中对应的记录
DELETE FROM character_inventory
WHERE guid = @char_guid;

-- 5.4 额外清理：显式删除可能遗留的物品实例（保险措施）
-- 说明：在某些数据库配置下，可能不会启用外键级联
DELETE FROM item_instance
WHERE owner_guid = @char_guid;

-- === 步骤 6：清理任务系统 ===
-- 说明：删除所有任务进度（进行中/已完成/失败/每日/每周/每月/赛季）
-- 影响：所有任务进度清零，下次登录时任务列表为空

DELETE FROM character_queststatus_rewarded
WHERE guid = @char_guid;

DELETE FROM character_queststatus_seasonal
WHERE guid = @char_guid;

DELETE FROM character_queststatus_monthly
WHERE guid = @char_guid;

DELETE FROM character_queststatus_weekly
WHERE guid = @char_guid;

DELETE FROM character_queststatus_daily
WHERE guid = @char_guid;

DELETE FROM character_queststatus
WHERE guid = @char_guid;

-- === 步骤 7：清理法术/技能/天赋/雕文系统 ===
-- 说明：删除所有已学法术、技能等级、天赋点分配、雕文选择
-- 后续：角色登录时会自动根据职业/等级重新学习基础法术和技能

-- 7.1 清理法术冷却
DELETE FROM character_spell_cooldown
WHERE guid = @char_guid;

-- 7.2 删除雕文
DELETE FROM character_glyphs
WHERE guid = @char_guid;

-- 7.3 删除天赋点
DELETE FROM character_talent
WHERE guid = @char_guid;

-- 7.4 删除技能
DELETE FROM character_skills
WHERE guid = @char_guid;

-- 7.5 删除已学法术
DELETE FROM character_spell
WHERE guid = @char_guid;

-- === 步骤 8：清理其他角色数据 ===
-- 说明：删除成就、声望、残留 Aura、动作栏

-- 8.1 删除成就进度
DELETE FROM character_achievement_progress
WHERE guid = @char_guid;

-- 8.2 删除已获得成就
DELETE FROM character_achievement
WHERE guid = @char_guid;

-- 8.3 删除声望值
DELETE FROM character_reputation
WHERE guid = @char_guid;

-- 8.4 删除登出时残留的 Aura
DELETE FROM character_aura
WHERE guid = @char_guid;

-- 8.5 删除动作栏配置
DELETE FROM character_action
WHERE guid = @char_guid;

-- 8.6 删除公会成员关系
DELETE FROM guild_member
WHERE guid = @char_guid;

-- 8.7 删除公会提取记录
DELETE FROM guild_member_withdraw
WHERE guid = @char_guid;

-- 8.8 删除竞技场队伍成员关系
DELETE FROM arena_team_member
WHERE guid = @char_guid;

-- === 步骤 9：重置 characters 表（角色主表） ===
-- 说明：重置等级、经验、金币、在线时间等核心属性
-- 位置：从 playercreateinfo 表获取种族/职业对应的出生点

-- 9.1 计算初始等级和金币
-- 普通职业起始等级：通常为 1
-- 死亡骑士起始等级：通常为 55
-- 初始金币：普通职业 0，DK 可配置

SET @start_level = CASE 
    WHEN @target_class = 6 THEN 55      -- 6 = CLASS_DEATH_KNIGHT
    ELSE 1
END;

SET @start_money = CASE 
    WHEN @target_class = 6 THEN 10000  -- DK 初始金币（可调整）
    ELSE 0
END;

-- 9.2 执行角色重置更新
UPDATE characters
SET 
    level = @start_level,          -- 重置等级到初始值
    xp = 0,                      -- 经验值清零
    money = @start_money,            -- 金币重置到初始值
    totaltime = 0,                 -- 总在线时间清零
    leveltime = 0,                 -- 当前等级在线时间清零
    position_x = (                -- 传送回出生点 X
        SELECT position_x 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    position_y = (                -- 传送回出生点 Y
        SELECT position_y 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    position_z = (                -- 传送回出生点 Z
        SELECT position_z 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    orientation = (               -- 传送回出生点朝向
        SELECT orientation 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    map = (                       -- 切换到出生点地图
        SELECT map 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    zone = (                      -- 设置为出生点区域
        SELECT zone 
        FROM playercreateinfo 
        WHERE race = @target_race AND class = @target_class
    ),
    at_login = 0,                 -- 清除登录标记（首次登录等）
    logout_time = 0,              -- 清除登出时间
    death_expire_time = 0,        -- 复活计时器清零
    talentGroupsCount = 1,        -- 重置为单天赋组
    activeTalentGroup = 0         -- 重置为第一个天赋组
WHERE guid = @char_guid;

-- === 步骤 10：可选 - 设置首次登录标记 ===
-- 说明：如果希望玩家下次登录时看到过场动画（cinematic），启用此行
-- 注意：这会触发 AT_LOGIN_FIRST 标记

-- UPDATE characters
-- SET at_login = 1
-- WHERE guid = @char_guid;

-- === 步骤 11：提交事务 ===
COMMIT;

-- === 步骤 12：验证结果 ===
-- 验证角色是否已成功重置
SELECT 
    c.guid,
    c.name,
    c.level AS new_level,
    c.xp AS new_xp,
    c.money AS new_money,
    c.map AS new_map,
    c.online AS online_status
FROM characters c
WHERE c.guid = @char_guid;

-- 输出成功信息
SELECT '========================================' AS Info;
SELECT 'Character reset completed successfully!' AS Info;
SELECT 'Next login will start at initial spawn point.' AS Info;
SELECT '========================================' AS Info;

-- === 错误处理建议 ===
-- 如果在执行过程中遇到错误：
-- 1. 检查数据库连接是否正常
-- 2. 检查是否有权限执行 DELETE/UPDATE 操作
-- 3. 检查 foreign key 约束是否阻止删除
-- 4. 如遇到外键错误，临时禁用外键检查：
--    SET FOREIGN_KEY_CHECKS = 0;
--    [执行删除操作]
--    SET FOREIGN_KEY_CHECKS = 1;

-- === 回滚方案 ===
-- 如果需要回滚（例如误操作）：
-- 1. 从备份恢复数据库
-- 2. 或使用 pdump 导出功能导出重置前的角色数据：
--    .pdump write backup.sql <playername>
--    .pdump load backup.sql <account> <newname>

-- === 脚本使用说明 ===
-- 1. 将脚本保存为 reset_character.sql
-- 2. 修改顶部的变量部分（@char_guid 等）
-- 3. 在 acore_characters 数据库中执行：
--    mysql -u root -p acore_characters < reset_character.sql
-- 4. 或通过客户端工具（如 HeidiSQL, Navicat）直接执行

-- === 注意事项 ===
-- 1. 执行前建议备份数据库
-- 2. 确保角色不在线（否则可能导致数据不一致）
-- 3. 此脚本会删除所有绑定数据，无法部分恢复
-- 4. 好友、公会、副本进度等数据不受影响
-- 5. 执行后清除 CharacterCache（重启服务器或使用命令）
```

#### 9.3.1 SQL 操作详细说明

| 步骤 | 表名 | 操作 | 说明 | 影响范围 |
|------|------|------|------|----------|
| **步骤 3** | `characters` | `UPDATE online = 0` | 确保玩家离线 | 单行 |
| **步骤 4.1** | `character_pet` | 读取宠物 ID | 为后续删除准备 | 可能 1-N 行 |
| **步骤 4.2** | `pet_spell_cooldown` | `DELETE` | 删除宠物法术冷却 | 可能 N 行 |
| **步骤 4.2** | `pet_spell` | `DELETE` | 删除宠物法术书 | 可能 N 行 |
| **步骤 4.2** | `pet_aura` | `DELETE` | 删除宠物 Aura | 可能 N 行 |
| **步骤 4.2** | `character_pet_declinedname` | `DELETE` | 删除宠物变格名 | 可能 1 行 |
| **步骤 4.2** | `character_pet` | `DELETE` | 删除宠物实例 | 1 行（触发级联） |
| **步骤 5.1** | `character_gifts` | `DELETE` | 删除礼物物品 | 可能 N 行 |
| **步骤 5.2** | `character_equipmentsets` | `DELETE` | 删除装备搭配 | 可能 N 行 |
| **步骤 5.3** | `character_inventory` | `DELETE` | 删除所有物品 | 通常 N 行 |
| **步骤 5.4** | `item_instance` | `DELETE` | 清理残留物品 | 可能 N 行 |
| **步骤 6** | `character_queststatus*` | `DELETE` × 6 | 删除所有任务进度 | 6 表，可能 N 行 |
| **步骤 7** | `character_spell/cooldown/skills/talent/glyphs` | `DELETE` × 5 | 删除法术/技能/天赋 | 5 表，可能 N 行 |
| **步骤 8** | `character_achievement*/reputation/aura/action` | `DELETE` × 4 | 删除成就/声望/Aura | 4 表，可能 N 行 |
| **步骤 9** | `characters` | `UPDATE` | 重置核心属性 | 单行（多字段） |

#### 9.3.2 性能优化建议

| 优化 | 方法 | 说明 |
|------|------|------|
| **索引检查** | 确保所有 `guid` 字段有索引 | 加速删除查询 |
| **批量删除** | 使用 `DELETE FROM table WHERE guid = ?` | 避免逐条删除 |
| **事务大小** | 如删除行数过多（>10000），考虑分批提交 | 防止事务过大 |
| **外键级联** | 确保外键约束正确，自动级联删除 | 避免手动多表删除 |
| **锁表** | 对于大表删除前临时加锁 `LOCK TABLES` | 减少并发影响 |
| **禁用外键** | 如需要，临时 `SET FOREIGN_KEY_CHECKS=0` | 提升大批量删除速度 |

#### 9.3.3 分步骤执行建议

如希望分步骤执行（更安全的回滚），可按以下顺序：

1. **第一阶段：准备和验证**
   ```sql
   -- 检查角色存在
   SELECT * FROM characters WHERE guid = @char_guid;
   
   -- 查看角色统计信息
   SELECT 
       COUNT(*) AS inventory_count
   FROM character_inventory WHERE guid = @char_guid;
   
   SELECT 
       COUNT(*) AS pet_count
   FROM character_pet WHERE owner = @char_guid;
   ```

2. **第二阶段：清理关联数据**
   ```sql
   -- 先执行 DELETE 操作，不更新 characters
   START TRANSACTION;
   
   -- 按依赖顺序删除宠物、物品、任务等
   [执行步骤 4-8 的所有 DELETE 语句]
   
   -- 提交第一阶段
   COMMIT;
   ```

3. **第三阶段：重置角色**
   ```sql
   -- 最后更新 characters 表
   START TRANSACTION;
   
   [执行步骤 9 的 UPDATE 语句]
   
   COMMIT;
   ```

#### 9.3.4 渐进式重置方案

如不需要完全重置，可以只重置部分数据：

**仅重置等级和位置**:
```sql
UPDATE characters
SET 
    level = 1,
    xp = 0,
    position_x = (SELECT position_x FROM playercreateinfo WHERE race = 1 AND class = 1),
    position_y = (SELECT position_y FROM playercreateinfo WHERE race = 1 AND class = 1),
    position_z = (SELECT position_z FROM playercreateinfo WHERE race = 1 AND class = 1),
    map = (SELECT map FROM playercreateinfo WHERE race = 1 AND class = 1),
    zone = (SELECT zone FROM playercreateinfo WHERE race = 1 AND class = 1)
WHERE guid = @char_guid;
```

**仅重置物品**:
```sql
DELETE FROM character_inventory WHERE guid = @char_guid;
DELETE FROM item_instance WHERE owner_guid = @char_guid;
```

**仅重置技能/天赋**:
```sql
DELETE FROM character_skills WHERE guid = @char_guid;
DELETE FROM character_talent WHERE guid = @char_guid;
-- 角色登录时会自动重新学习基础技能
```

### 9.4 重置流程代码实现

#### 9.4.1 WorldSession::HandlePlayerReset 命令

实现一个新的 GM 命令：

```cpp
// src/server/scripts/Commands/cs_character.cpp
static bool HandlePlayerResetCommand(ChatHandler* handler, std::string name, Optional<uint8> newLevelOpt)
{
    Player* target = nullptr;
    ObjectGuid targetGuid;

    // 查找目标角色
    if (Player* player = handler->GetSession()->GetPlayer())
    {
        if (name.empty() || player->GetName() == name)
        {
            target = player;
            targetGuid = target->GetGUID();
        }
    }

    if (!target)
    {
        uint32 accountId = sCharacterCache->GetCharacterAccountIdByName(name);
        if (!accountId)
        {
            handler->SendSysMessage(LANG_PLAYER_NOT_FOUND);
            return false;
        }

        targetGuid = sCharacterCache->GetCharacterGuidByName(name);
    }

    // 检查角色是否在线
    if (targetGuid && !target)
    {
        if (Player* onlinePlayer = ObjectAccessor::FindConnectedPlayer(targetGuid))
        {
            onlinePlayer->GetSession()->KickPlayer("Character reset");
        }
    }

    // 重置角色数据
    if (!Player::ResetCharacterToStartingState(targetGuid))
    {
        handler->SendSysMessage("Character reset failed.");
        return false;
    }

    handler->PSendSysMessage("Character '%s' has been reset to level 1.", name.c_str());
    return true;
}
```

#### 9.4.2 Player::ResetCharacterToStartingState

在 `Player` 类中添加静态方法：

```cpp
// src/server/game/Entities/Player/Player.h
class Player : public Unit, public GridObject<Player>
{
public:
    static bool ResetCharacterToStartingState(ObjectGuid::LowType guid);

private:
    // ... existing members
};
```

```cpp
// src/server/game/Entities/Player/Player.cpp
bool Player::ResetCharacterToStartingState(ObjectGuid::LowType guid)
{
    // 1. 获取角色基本信息
    uint8 race, playerClass;
    {
        CharacterDatabasePreparedStatement* stmt = CharacterDatabase.GetPreparedStatement(CHAR_SEL_CHARACTER_BASIC);
        stmt->SetData(0, guid);
        PreparedQueryResult result = CharacterDatabase.Query(stmt);
        if (!result)
            return false;

        race = (*result)[4].Get<uint8>();  // race
        playerClass = (*result)[3].Get<uint8>();  // class
    }

    // 2. 获取出生点信息
    float posX, posY, posZ, orientation;
    uint32 mapId, zoneId;
    {
        WorldDatabasePreparedStatement* stmt = WorldDatabase.GetPreparedStatement(WORLD_SEL_PLAYER_CREATE_INFO);
        stmt->SetData(0, race);
        stmt->SetData(1, playerClass);
        PreparedQueryResult result = WorldDatabase.Query(stmt);
        if (!result)
            return false;

        Field* fields = result->Fetch();
        mapId = fields[2].Get<uint16>();
        zoneId = fields[3].Get<uint32>();
        posX = fields[4].Get<float>();
        posY = fields[5].Get<float>();
        posZ = fields[6].Get<float>();
        orientation = fields[7].Get<float>();
    }

    // 3. 开始数据库事务
    CharacterDatabaseTransaction trans = CharacterDatabase.BeginTransaction();

    // 4. 清理宠物数据
    {
        // 获取宠物 GUID 列表
        CharacterDatabasePreparedStatement* stmt = CharacterDatabase.GetPreparedStatement(CHAR_SEL_PET_IDS_BY_OWNER);
        stmt->SetData(0, guid);
        PreparedQueryResult petResult = CharacterDatabase.Query(stmt);

        if (petResult)
        {
            std::vector<uint32> petGuids;
            do
            {
                petGuids.push_back((*petResult)[0].Get<uint32>());
            } while (petResult->NextRow());

            if (!petGuids.empty())
            {
                std::ostringstream petGuidStream;
                for (size_t i = 0; i < petGuids.size(); ++i)
                {
                    if (i > 0) petGuidStream << ',';
                    petGuidStream << petGuids[i];
                }

                // 删除宠物相关表
                stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_PET_SPELL_COOLDOWN_BY_PET);
                stmt->SetData(0, petGuidStream.str());
                trans->Append(stmt);

                stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_PET_SPELL_BY_PET);
                stmt->SetData(0, petGuidStream.str());
                trans->Append(stmt);

                stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_PET_AURA_BY_PET);
                stmt->SetData(0, petGuidStream.str());
                trans->Append(stmt);
            }
        }

        // 删除宠物主表
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_PET_BY_OWNER);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_PET_DECLINEDNAME_BY_OWNER);
        stmt->SetData(0, guid);
        trans->Append(stmt);
    }

    // 5. 清理物品系统
    {
        // 删除礼物
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_GIFT_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        // 删除装备搭配
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_EQUIPMENT_SET_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        // 删除 inventory（会自动级联删除 item_instance）
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_INVENTORY_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);
    }

    // 6. 清理法术/技能/天赋
    {
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_GLYPH_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_TALENT_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_SKILL_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_SPELL_COOLDOWN_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_SPELL_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);
    }

    // 7. 清理任务系统
    {
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_REWARDED_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_SEASONAL_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_MONTHLY_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_WEEKLY_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_DAILY_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_QUEST_STATUS_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);
    }

    // 8. 清理其他数据
    {
        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_ACHIEVEMENT_PROGRESS_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_ACHIEVEMENT_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_REPUTATION_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_AURA_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);

        stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_ACTION_BY_GUID);
        stmt->SetData(0, guid);
        trans->Append(stmt);
    }

    // 9. 重置 characters 主表
    bool isDeathKnight = (playerClass == CLASS_DEATH_KNIGHT);
    uint32 startLevel = isDeathKnight ? sWorld->getIntConfig(CONFIG_START_HEROIC_PLAYER_LEVEL)
                                      : sWorld->getIntConfig(CONFIG_START_PLAYER_LEVEL);
    uint32 startMoney = isDeathKnight ? sWorld->getIntConfig(CONFIG_START_HEROIC_PLAYER_MONEY)
                                      : sWorld->getIntConfig(CONFIG_START_PLAYER_MONEY);

    stmt = CharacterDatabase.GetPreparedStatement(CHAR_UPD_CHARACTER_RESET);
    stmt->SetData(0, startLevel);
    stmt->SetData(1, 0);  // xp
    stmt->SetData(2, startMoney);
    stmt->SetData(3, 0);  // totaltime
    stmt->SetData(4, 0);  // leveltime
    stmt->SetData(5, posX);
    stmt->SetData(6, posY);
    stmt->SetData(7, posZ);
    stmt->SetData(8, orientation);
    stmt->SetData(9, mapId);
    stmt->SetData(10, zoneId);
    stmt->SetData(11, 0);  // at_login
    stmt->SetData(12, guid);
    trans->Append(stmt);

    // 10. 提交事务
    CharacterDatabase.CommitTransaction(trans);

    // 11. 清理缓存
    sCharacterCache->RemoveCharacterCacheEntry(ObjectGuid(HighGuid::Player, guid));

    // 12. 如角色在线，强制登出（已在步骤 1 踢出）

    return true;
}
```

### 9.5 注意事项和最佳实践

#### 9.5.1 安全性考虑

| 问题 | 建议 |
|------|------|
| **在线状态** | 重置前必须踢出在线玩家或等待登出，避免数据不一致 |
| **事务安全** | 所有数据库操作必须在一个事务中，失败可回滚 |
| **外键约束** | 如启用外键，注意级联删除的顺序 |
| **GUID 唯一性** | 不要手动修改 GUID，使用系统提供的生成器 |

#### 9.5.2 数据保留建议

| 数据 | 建议处理 | 原因 |
|------|----------|------|
| **角色名称** | **保留** | 玩家身份标识 |
| **角色种族/职业** | **保留** | 无法更改，除非使用外观定制服务 |
| **外观数据** | **保留** | 玩家定制的外观 |
| **好友列表** | **可选保留** | 通常洗号玩家保留好友 |
| **炉石位置** | **可选保留** | 或重置回种族默认出生点 |

#### 9.5.3 性能优化

| 优化 | 说明 |
|------|------|
| **批量删除** | 使用 `DELETE FROM table WHERE guid IN (...)` 代替多次单条删除 |
| **级联删除** | 如数据库支持，利用外键级联删除自动清理关联表 |
| **缓存失效** | 重置后立即清理 `CharacterCache`，避免读取到旧数据 |
| **索引优化** | 确保 `guid` 字段有索引，加速删除查询 |

#### 9.5.4 与其他系统的兼容性

| 系统 | 影响 | 建议操作 |
|------|------|----------|
| **公会** | 重置不影响 | 角色仍保留在公会中 |
| **副本锁定** | 重置不影响 | 仍保留副本进度 |
| **竞技场队伍** | 重置不影响 | 队伍成员关系保留 |
| **邮件系统** | 需谨慎 | 邮件通常不删除，但如果发送者角色不存在可能导致问题 |

### 9.6 角色重置完整 SQL 汇总（可执行版本）

以下是将角色重置到 1 级初始状态的完整 SQL 操作汇总，可直接复制执行：

```sql
-- ========================================================================
-- AzerothCore 角色重置 SQL 汇总
-- 功能：将指定角色重置到 1 级初始状态
-- 数据库：acore_characters
-- 执行前请修改变量值
-- ========================================================================

-- === 变量配置（必须修改） ===
SET @char_guid = 456;        -- 要重置的角色 GUID
SET @target_race = 1;        -- 角色种族（1=人类, 2=兽人, 3=矮人 等）
SET @target_class = 1;       -- 角色职业（1=战士, 2=圣骑士, 3=猎人 等）

-- === 开始事务 ===
START TRANSACTION;

-- === 1. 确保角色离线 ===
UPDATE characters SET online = 0 WHERE guid = @char_guid;

-- === 2. 清理宠物系统 ===
CREATE TEMPORARY TABLE temp_pet_ids AS
SELECT id FROM character_pet WHERE owner = @char_guid;

DELETE FROM pet_spell_cooldown WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM pet_spell WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM pet_aura WHERE guid IN (SELECT id FROM temp_pet_ids);
DELETE FROM character_pet_declinedname WHERE owner = @char_guid;
DELETE FROM character_pet WHERE owner = @char_guid;
DROP TEMPORARY TABLE IF EXISTS temp_pet_ids;

-- === 3. 清理物品系统 ===
DELETE FROM character_gifts WHERE guid = @char_guid;
DELETE FROM character_equipmentsets WHERE guid = @char_guid;
DELETE FROM character_inventory WHERE guid = @char_guid;
DELETE FROM item_instance WHERE owner_guid = @char_guid;

-- === 4. 清理任务系统 ===
DELETE FROM character_queststatus_rewarded WHERE guid = @char_guid;
DELETE FROM character_queststatus_seasonal WHERE guid = @char_guid;
DELETE FROM character_queststatus_monthly WHERE guid = @char_guid;
DELETE FROM character_queststatus_weekly WHERE guid = @char_guid;
DELETE FROM character_queststatus_daily WHERE guid = @char_guid;
DELETE FROM character_queststatus WHERE guid = @char_guid;

-- === 5. 清理法术/技能/天赋/雕文 ===
DELETE FROM character_spell_cooldown WHERE guid = @char_guid;
DELETE FROM character_glyphs WHERE guid = @char_guid;
DELETE FROM character_talent WHERE guid = @char_guid;
DELETE FROM character_skills WHERE guid = @char_guid;
DELETE FROM character_spell WHERE guid = @char_guid;

-- === 6. 清理其他角色数据 ===
DELETE FROM character_achievement_progress WHERE guid = @char_guid;
DELETE FROM character_achievement WHERE guid = @char_guid;
DELETE FROM character_reputation WHERE guid = @char_guid;
DELETE FROM character_aura WHERE guid = @char_guid;
DELETE FROM character_action WHERE guid = @char_guid;

-- === 7. 清理公会和竞技场队伍关系 ===
DELETE FROM guild_member WHERE guid = @char_guid;
DELETE FROM guild_member_withdraw WHERE guid = @char_guid;
DELETE FROM arena_team_member WHERE guid = @char_guid;

-- === 8. 重置 characters 主表 ===
SET @start_level = CASE WHEN @target_class = 6 THEN 55 ELSE 1 END;
SET @start_money = CASE WHEN @target_class = 6 THEN 10000 ELSE 0 END;

UPDATE characters
SET 
    level = @start_level,
    xp = 0,
    money = @start_money,
    totaltime = 0,
    leveltime = 0,
    position_x = (SELECT position_x FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    position_y = (SELECT position_y FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    position_z = (SELECT position_z FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    orientation = (SELECT orientation FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    map = (SELECT map FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    zone = (SELECT zone FROM playercreateinfo WHERE race = @target_race AND class = @target_class),
    at_login = 0,
    logout_time = 0,
    death_expire_time = 0,
    talentGroupsCount = 1,
    activeTalentGroup = 0
WHERE guid = @char_guid;

-- === 提交事务 ===
COMMIT;

-- === 验证结果 ===
SELECT 
    guid,
    name,
    level AS new_level,
    xp AS new_xp,
    money AS new_money,
    map AS new_map,
    online AS online_status
FROM characters
WHERE guid = @char_guid;

SELECT 'Character reset completed successfully!' AS Result;
```

#### 9.6.1 SQL 操作清单汇总

| 序号 | 操作类型 | 表名 | SQL 语句 | 说明 |
|------|---------|------|----------|------|
| 1 | UPDATE | `characters` | `UPDATE characters SET online = 0 WHERE guid = @char_guid` | 确保角色离线 |
| 2 | DELETE | `pet_spell_cooldown` | `DELETE FROM pet_spell_cooldown WHERE guid IN (...)` | 删除宠物法术冷却 |
| 3 | DELETE | `pet_spell` | `DELETE FROM pet_spell WHERE guid IN (...)` | 删除宠物法术书 |
| 4 | DELETE | `pet_aura` | `DELETE FROM pet_aura WHERE guid IN (...)` | 删除宠物 Aura |
| 5 | DELETE | `character_pet_declinedname` | `DELETE FROM character_pet_declinedname WHERE owner = @char_guid` | 删除宠物变格名 |
| 6 | DELETE | `character_pet` | `DELETE FROM character_pet WHERE owner = @char_guid` | 删除宠物实例 |
| 7 | DELETE | `character_gifts` | `DELETE FROM character_gifts WHERE guid = @char_guid` | 删除礼物物品 |
| 8 | DELETE | `character_equipmentsets` | `DELETE FROM character_equipmentsets WHERE guid = @char_guid` | 删除装备搭配 |
| 9 | DELETE | `character_inventory` | `DELETE FROM character_inventory WHERE guid = @char_guid` | 删除所有物品 |
| 10 | DELETE | `item_instance` | `DELETE FROM item_instance WHERE owner_guid = @char_guid` | 清理残留物品 |
| 11 | DELETE | `character_queststatus_rewarded` | `DELETE FROM character_queststatus_rewarded WHERE guid = @char_guid` | 删除已完成任务记录 |
| 12 | DELETE | `character_queststatus_seasonal` | `DELETE FROM character_queststatus_seasonal WHERE guid = @char_guid` | 删除赛季任务进度 |
| 13 | DELETE | `character_queststatus_monthly` | `DELETE FROM character_queststatus_monthly WHERE guid = @char_guid` | 删除每月任务进度 |
| 14 | DELETE | `character_queststatus_weekly` | `DELETE FROM character_queststatus_weekly WHERE guid = @char_guid` | 删除每周任务进度 |
| 15 | DELETE | `character_queststatus_daily` | `DELETE FROM character_queststatus_daily WHERE guid = @char_guid` | 删除每日任务进度 |
| 16 | DELETE | `character_queststatus` | `DELETE FROM character_queststatus WHERE guid = @char_guid` | 删除任务进度 |
| 17 | DELETE | `character_spell_cooldown` | `DELETE FROM character_spell_cooldown WHERE guid = @char_guid` | 删除法术冷却 |
| 18 | DELETE | `character_glyphs` | `DELETE FROM character_glyphs WHERE guid = @char_guid` | 删除雕文 |
| 19 | DELETE | `character_talent` | `DELETE FROM character_talent WHERE guid = @char_guid` | 删除天赋点 |
| 20 | DELETE | `character_skills` | `DELETE FROM character_skills WHERE guid = @char_guid` | 删除技能 |
| 21 | DELETE | `character_spell` | `DELETE FROM character_spell WHERE guid = @char_guid` | 删除已学法术 |
| 22 | DELETE | `character_achievement_progress` | `DELETE FROM character_achievement_progress WHERE guid = @char_guid` | 删除成就进度 |
| 23 | DELETE | `character_achievement` | `DELETE FROM character_achievement WHERE guid = @char_guid` | 删除已获得成就 |
| 24 | DELETE | `character_reputation` | `DELETE FROM character_reputation WHERE guid = @char_guid` | 删除声望值 |
| 25 | DELETE | `character_aura` | `DELETE FROM character_aura WHERE guid = @char_guid` | 删除残留 Aura |
| 26 | DELETE | `character_action` | `DELETE FROM character_action WHERE guid = @char_guid` | 删除动作栏配置 |
| 27 | DELETE | `guild_member` | `DELETE FROM guild_member WHERE guid = @char_guid` | 删除公会成员关系 |
| 28 | DELETE | `guild_member_withdraw` | `DELETE FROM guild_member_withdraw WHERE guid = @char_guid` | 删除公会提取记录 |
| 29 | DELETE | `arena_team_member` | `DELETE FROM arena_team_member WHERE guid = @char_guid` | 删除竞技场队伍关系 |
| 30 | UPDATE | `characters` | `UPDATE characters SET level = ..., xp = 0, ... WHERE guid = @char_guid` | 重置角色核心属性 |

#### 9.6.2 执行统计

| 统计项 | 数量 |
|--------|------|
| **总操作数** | 30 个 SQL 操作 |
| **DELETE 操作** | 29 个 |
| **UPDATE 操作** | 1 个（最后重置 characters 表） |
| **涉及表数** | 29 张数据库表 |
| **事务数** | 1 个（所有操作在一个事务中） |
| **执行时间** | 通常 < 1 秒（取决于数据量） |

#### 9.6.3 使用说明

1. **修改变量**：将 `@char_guid`、`@target_race`、`@target_class` 替换为实际值
2. **备份数据**：执行前建议备份数据库或使用 `.pdump write` 导出角色数据
3. **确保离线**：角色必须不在线，否则可能导致数据不一致
4. **执行脚本**：在 `acore_characters` 数据库中执行
5. **验证结果**：检查最后的 SELECT 输出，确认重置成功

#### 9.6.4 快速执行命令

```bash
# 方式 1：直接执行 SQL 文件
mysql -u root -p acore_characters < reset_character.sql

# 方式 2：通过 MySQL 客户端执行
mysql -u root -p acore_characters
mysql> SET @char_guid = 456;
mysql> SET @target_race = 1;
mysql> SET @target_class = 1;
mysql> [粘贴上述 SQL 语句]

# 方式 3：使用 GM 命令（需要实现代码）
.character reset <playername>
```

---

> 本文档基于 AzerothCore WotLK 源码分析生成。如有功能更新或架构变更，请以最新源码为准。
