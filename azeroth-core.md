# AzerothCore 项目深度总结

## 项目概述

AzerothCore 是一个开源的《魔兽世界：巫妖王之怒》(World of Warcraft: Wrath of the Lich King, 3.3.5a版本) 私服模拟器，使用 C++ 编写。该项目基于 MaNGOS、TrinityCore 和 SunwellCore 发展而来，自2016年以来经历了大量开发改进，提供了稳定、高度模块化的游戏服务器实现。

### 核心特性

- **稳定性优先**：所有代码变更必须通过 CI 持续集成测试
- **仿官方内容**：追求与官方游戏一致的游戏体验
- **高度模块化**：支持动态模块加载，便于扩展和定制
- **社区驱动**：拥有活跃的开发者社区和贡献者

---

## 架构设计

### 1. 整体架构

```
AzerothCore
├── 客户端                    # 账号认证服务器
│   └── 处理登录、账号管理
│
├── 服务器                    # 游戏世界服务器
│   ├── 游戏逻辑核心
│   ├── 脚本系统
│   └── 模块系统
│
├── 数据提取工具
│   ├── dbcextract          # DBC 文件提取
│   ├── mapextractor        # 地图数据提取
│   └── vmap4_extractor     # 虚拟地图提取
│
└── 数据库
    ├── auth                # 认证数据库
    ├── characters          # 角色数据库
    └── world               # 世界数据库
```

### 2. 目录结构详解

#### `/apps/` - 应用程序和工具
- `compiler/` - 编译工具链
- `installer/` - 安装脚本和配置
- `test-framework/` - 测试框架
- `docker/` - Docker 容器化支持
- `startup-scripts/` - 服务器启动脚本
- `bash_shared/` - 共享的 Bash 脚本库

#### `/src/` - 源代码

##### `/src/server/game/` - 游戏逻辑核心

这是最核心的游戏逻辑实现目录，包含：

**实体系统 (`Entities/`)**
- `Player/` - 玩家对象（21个文件）：玩家属性、操作、社交、交易等
- `Creature/` - 生物/NPC系统（14个文件）：AI、生成、模板
- `GameObject/` - 游戏对象：宝箱、门、可交互物体
- `Unit/` - 单位基类：生物和玩家的共同基类
- `Item/` - 物品系统：物品属性、容器、装备
- `Pet/` - 宠物系统
- `Vehicle/` - 载具系统
- `Corpse/` - 尸体系统
- `Totem/` - 图腾系统
- `Transport/` - 运输工具（船、电梯等）
- `Object/` - 对象基类：所有游戏对象的根基

**网络处理 (`Handlers/`)**
- 处理客户端数据包（30+个处理器文件）：
  - `CharacterHandler.cpp` - 角色创建、删除、列表
  - `MovementHandler.cpp` - 移动同步
  - `SpellHandler.cpp` - 法术处理
  - `CombatHandler.cpp` - 战斗
  - `ItemHandler.cpp` - 物品操作
  - `ChatHandler.cpp` - 聊天系统
  - `QueryHandler.cpp` - 查询请求
  - `BattleGroundHandler.cpp` - 战场
  - `ArenaTeamHandler.cpp` - 竞技场
  - `GroupHandler.cpp` - 队伍系统
  - `GuildHandler.cpp` - 公会系统
  - `MailHandler.cpp` - 邮件系统
  - `AuctionHouseHandler.cpp` - 拍卖行
  - `LFGHandler.cpp` - 地下城查找器
  - `NPCHandler.cpp` - NPC交互
  - `PetHandler.cpp` - 宠物操作
  - `QuestHandler.cpp` - 任务系统

**核心系统**
- `Maps/` - 地图、网格、副本管理
- `Spells/` - 法术、光环、效果系统
- `AI/` - 人工智能系统
- `Scripting/` - 脚本系统接口
- `Combat/` - 战斗、威胁计算
- `Battlegrounds/` - 战场系统
- `Instances/` - 副本系统
- `Groups/` - 队伍系统
- `Guilds/` - 公会系统
- `Movement/` - 移动系统
- `Chat/` - 聊天系统
- `Loot/` - 战利品系统
- `Skills/` - 技能系统
- `Achievements/` - 成就系统
- `Quests/` - 任务系统
- `Conditions/` - 条件系统
- `Weather/` - 天气系统
- `Calendar/` - 日历系统
- `OutdoorPvP/` - 户外PvP
- `DungeonFinding/` - 地下城查找
- `AuctionHouse/` - 拍卖行
- `Reputation/` - 声望系统
- `DataStores/` - 数据缓存
- `Grids/` - 网格系统
- `Globals/` - 全局管理器
- `Server/` - 服务器管理
- `Warden/` - 反作弊系统

##### `/src/server/database/` - 数据库层
- `Database/` - 数据库连接和查询
- `Updater/` - 数据库更新和迁移
- `Logging/` - 数据库日志

##### `/src/server/shared/` - 共享组件
- `Network/` - 网络层实现
- `Packets/` - 数据包处理（ByteBuffer）
- `Configuration/` - 配置系统
- `DataStores/` - DBC 数据存储
- `Realms/` - 领域管理
- `Secrets/` - 密钥管理

##### `/src/tools/` - 工具程序
- 数据提取工具
- 数据库导入工具
- 地图生成工具

##### `/src/test/` - 单元测试
- Google Test 框架
- Mock 对象

#### `/modules/` - 模块系统

AzerothCore 的模块系统是其核心特性之一：

**模块类型**
- **静态链接**：模块代码直接编译进 worldserver
- **动态链接**：模块编译为独立的共享库(.so/.dll)，运行时加载

**模块配置**
- `-DMODULES=static` - 静态链接所有模块
- `-DMODULES=dynamic` - 动态链接所有模块
- 可通过 CMakeLists.txt 配置单个模块的链接方式

**模块结构**
```
modules/
├── CMakeLists.txt           # 模块构建配置
├── ModulesLoader.cpp.in     # 加载器模板
├── create_module.sh         # 创建模块脚本
├── mod-eluna/               # Lua 脚本引擎模块
│   ├── src/                 # 源代码
│   ├── conf/                # 模块配置
│   └── CMakeLists.txt
└── [其他模块...]
```

**创建新模块**
```bash
cd modules
./create_module.sh  # 交互式创建
```

#### `/data/` - 数据文件

##### `/data/sql/` - SQL 数据
- `updates/` - 数据库更新文件
  - `pending_db_auth/` - 认证数据库更新
  - `pending_db_characters/` - 角色数据库更新
  - `pending_db_world/` - 世界数据库更新
- `archive/` - 归档的 SQL 文件
- `base/` - 基础数据库结构
- `create/` - 数据库创建脚本

#### `/conf/` - 配置文件
- `worldserver.conf.dist` - 世界服务器配置模板
- `authserver.conf.dist` - 认证服务器配置模板

#### `/deps/` - 第三方依赖
- MySQL 客户端库
- OpenSSL 加密库
- Boost 库
- JSON 处理库
- AzerothCore Bash 库

---

## 核心系统详解

### 1. 脚本系统

AzerothCore 支持多种脚本方式：

#### C++ 脚本
- 核心功能使用 C++ 实现
- 通过 `ScriptMgr` 管理所有脚本
- 脚本类型定义在 `ScriptDefines/` 目录
- 主要脚本接口：
  - `CreatureAI` - 生物 AI
  - `GameObjectAI` - 游戏对象 AI
  - `InstanceScript` - 副本脚本
  - `SpellScript` - 法术脚本
  - `AuraScript` - 光环脚本
  - `WorldScript` - 世界事件脚本
  - `PlayerScript` - 玩家事件脚本

#### mod-eluna (Lua 脚本引擎)
- **重要**：mod-eluna 是 AzerothCore 专用的 Lua 引擎，与原版 Eluna **不兼容**
- 支持 LuaJIT（推荐）、Lua 5.2、5.3、5.4
- 通过 Hook 系统扩展游戏功能
- 无需修改核心代码即可实现自定义功能

#### SmartAI (SAI)
- 数据库驱动的 AI 系统
- 通过 SQL 配置 NPC 行为
- 适用于简单的 NPC 逻辑

### 2. 网络架构

```
客户端
    ↓ (TCP)
WorldSocket/WorldSession
    ↓
WorldPacket 处理
    ↓
Handler 分发
    ↓
游戏逻辑处理
    ↓
响应 → 客户端
```

- 使用自定义协议（基于 WoW 协议）
- 数据包加密（ARC4/SRP6）
- ByteBuffer 统一处理序列化/反序列化

### 3. 地图系统

- **网格系统**：世界被划分为多个网格，按需加载
- **地图管理器**：管理所有地图实例
- **副本系统**：支持团队副本、英雄副本
- **地图类型**：
  - 普通地图
  - 副本地图
  - 战场地图
  - 竞技场地图

### 4. 数据库架构

#### 三数据库设计

**auth 数据库**
- `account` - 账号信息
- `account_access` - 权限管理
- `account_banned` - 封禁记录
- `realm` - 服务器列表

**characters 数据库**
- `characters` - 角色基础数据
- `character_inventory` - 物品栏
- `character_queststatus` - 任务状态
- `character_glyphs` - 雕文
- `guild` - 公会数据
- `arena_team` - 竞技场队伍

**world 数据库**
- `creature_template` - NPC 模板
- `gameobject_template` - 游戏对象模板
- `item_template` - 物品模板
- `quest_template` - 任务模板
- `spell_*` - 法术数据
- `creature_loot_template` - 战利品表

#### 更新系统
- SQL 文件按版本号命名
- 自动应用未执行的更新
- 支持回滚机制

### 5. 编译系统

#### CMake 配置
```cmake
cmake_minimum_required(VERSION 3.16...3.22)

# 关键选项
-DCMAKE_BUILD_TYPE=RelWithDebInfo  # 编译类型
-DSCRIPTS=static/dynamic           # 脚本链接方式
-DMODULES=static/dynamic           # 模块链接方式
-DLUA_VERSION=luajit               # Lua 版本
-DBUILD_TESTING=ON                 # 启用测试
```

#### 预编译头 (PCH)
- 加速编译过程
- `common.pch` - 通用代码
- `game.pch` - 游戏逻辑
- `script.pch` - 脚本代码

#### ccache 支持
- 默认启用，加速重复编译
- 可通过 `AC_CCACHE` 环境变量控制

---

## 开发工作流

### 1. 构建流程

```bash
# 方式一：使用交互式脚本（推荐）
./acore.sh init              # 首次初始化
./acore.sh compiler build    # 配置并编译
./acore.sh compiler all      # 清理+配置+编译

# 方式二：手动 CMake
mkdir build && cd build
cmake .. -DSCRIPTS=static -DMODULES=static
make -j$(nproc)

# 方式三：Docker
docker-compose up -d
```

### 2. 测试

```bash
# 单元测试（C++）
./acore.sh test core

# Bash 脚本测试
./acore.sh test bash
```

### 3. 配置管理

```bash
# 配置管理器
./acore.sh config

# 合并配置文件
# 配置文件在 conf/ 目录
# 模块配置在 modules/*/conf/ 目录
```

### 4. 模块管理

```bash
# 列出可用模块
./acore.sh module list

# 安装模块
./acore.sh module install <模块名>

# 更新模块
./acore.sh module update <模块名>

# 移除模块
./acore.sh module remove <模块名>
```

### 5. 服务器运行

```bash
# 启动世界服务器（带自动重启）
./acore.sh run-worldserver

# 启动认证服务器
./acore.sh run-authserver

# 使用服务管理器（后台运行）
./acore.sh service-manager
```

---

## 提交规范

### 提交消息格式

```
### TITLE
## Type(Scope/Subscope): 简短描述（最多50字符）

### DESCRIPTION
## 详细说明变更内容和原因（每行最多72字符）

## 引用链接：issue、commit、PR或参考资料
```

### 类型 (Type)
- `feat` - 新功能
- `fix` - Bug修复
- `refactor` - 重构
- `style` - 代码格式
- `docs` - 文档
- `test` - 测试
- `chore` - 构建/工具

### 范围 (Scope)
- `CORE` - 核心代码
- `DB` - 数据库

### 子范围示例
- `DB/SAI` - SmartAI
- `CORE/Raid` - 团队副本
- `CORE/Commands` - GM命令
- `DB/Quest` - 任务
- `DB/Item` - 物品

---

## 配置系统

### 配置策略

支持四种严重性级别（`doc/ConfigPolicy.md`）：
- `skip` - 忽略并继续
- `warn` - 记录警告并继续
- `error` - 记录错误并继续
- `fatal` - 记录致命错误并中止

### 预设策略

```bash
# 默认策略
AC_CONFIG_POLICY_PRESET_DEFAULT
# 缺失文件=错误, 缺失选项=警告, 关键选项=致命

# 零配置策略
AC_CONFIG_POLICY_PRESET_ZERO_CONF
# 依赖默认值和环境变量

# 严格策略
AC_CONFIG_POLICY_PRESET_STRICT
# 所有问题都报错，适合CI
```

### 配置方式优先级
1. CLI 参数 `--config-policy`
2. 环境变量 `AC_CONFIG_POLICY`
3. 配置文件 `conf/dist/config.sh`

---

## 深入技术细节

### 1. 服务器启动流程和初始化

**主入口文件：** `src/server/apps/worldserver/Main.cpp`

```cpp
// 层次化初始化顺序
1. 配置系统初始化    - sConfigMgr->LoadAppConfigs()
2. 日志系统初始化    - sLog->Initialize()
3. OpenSSL 加密初始化 - OpenSSLCrypto::threadsSetup()
4. 线程池创建        - Boost Asio IoContext
5. 数据库连接        - DatabaseLoader
6. 脚本系统初始化    - sScriptMgr->Initialize()
7. 世界设置          - sWorld->SetInitialWorldSettings()
8. 网络启动          - sWorldSocketMgr.StartWorldNetwork()
```

**关键实现特性：**
- 使用 RAII 模式管理资源生命周期
- 支持多线程 Boost Asio 网络架构
- 采用智能指针管理资源
- 支持冻结检测器防止服务器卡死（`FreezeDetector`）

### 2. 网络层架构

**核心类：** `SocketMgr<WorldSocket>` 模板类

```
客户端
    ↓ (TCP + 加密)
AsyncAcceptor (异步接受器)
    ↓
NetworkThread[] (网络线程池)
    ↓
WorldSocket (连接处理器)
    ↓
WorldSession (会话管理)
    ↓
WorldPacket (数据包处理)
    ↓
Handler 分发 (OpcodeTable)
    ↓
游戏逻辑处理
```

**网络线程模型：**
```cpp
class NetworkThread
{
    void Start();                          // 启动线程
    void Stop();                           // 停止线程
    void AddSocket(std::shared_ptr<SocketType> sock);  // 添加连接
    void Process();                        // 处理网络事件

private:
    std::mutex _mutex;                     // 线程锁
    std::vector<std::shared_ptr<SocketType>> _sockets;  // 连接列表
    std::shared_ptr<Acceptor> _acceptor;   // 接受器
};
```

**连接负载均衡：**
- 自动选择连接数最少的线程处理新连接
- 支持 `SELECT_DELAY` 配置处理延迟
- 采用 Nagle 算法优化（`_tcpNoDelay` 配置）

**数据包加密：**
- ARC4 流加密
- SRP6 认证协议
- Session Key 密钥交换

### 3. 线程模型和并发处理

**线程池管理：**
```cpp
// 主线程池（世界逻辑）
std::shared_ptr<std::vector<std::thread>> threadPool;
for (int i = 0; i < numThreads; ++i) {
    threadPool->push_back(std::thread([ioContext]() {
        ioContext->run();  // Boost Asio 事件循环
    }));
}

// 网络线程池
NetworkThread<SocketType>[] _threads;
```

**线程类型：**
1. **主线程** - 核心游戏逻辑
2. **网络线程池** - 处理客户端连接
3. **数据库工作线程** - 异步 SQL 执行
4. **地图更新线程** - Grid 系统更新

**同步机制：**
```cpp
// 读写锁 - 保护共享数据
std::shared_mutex _mutex;

// 原子操作 - 无锁计数
std::atomic<uint32> _scriptCount;

// 事件驱动 - 减少锁竞争
ioContext->post([]() { /* 异步任务 */ });
```

### 4. 内存管理机制

**智能指针使用策略：**
```cpp
// 共享所有权 - 游戏实体
std::shared_ptr<Unit> unit;

// 独占所有权 - 网络连接
std::unique_ptr<WorldSocket> socket;

// 弱引用 - 防止循环引用
std::weak_ptr<Player> playerRef;
```

**对象池模式：**
- `ByteBuffer` 预分配缓冲区池
- `PacketBuffer` 复用机制
- `GridRefManager` 网格引用管理

**RAII 模式应用：**
```cpp
class Transaction {
    Transaction(Database& db) : _db(db) { _db.BeginTransaction(); }
    ~Transaction() { if(_active) _db.Rollback(); }
    void Commit() { _db.Commit(); _active = false; }
};
```

### 5. 游戏实体系统层次结构

**Object 基类设计：**

```
Object (抽象基类)
│
├── WorldObject (世界对象)
│   │
│   ├── Unit (战斗单位)
│   │   ├── Creature (生物/NPC)
│   │   │   ├── TempSummon (临时召唤)
│   │   │   └── Pet (宠物)
│   │   │
│   │   └── Player (玩家)
│   │
│   ├── GameObject (游戏对象)
│   │   ├── GameObject (可交互物体)
│   │   └── Transport (运输工具)
│   │
│   ├── Corpse (尸体)
│   └── DynamicObject (动态对象)
│
└── Item (物品)
    ├── Item (普通物品)
    └── Container (容器)
```

**Object 核心功能接口：**
```cpp
class Object {
protected:
    ObjectGuid guid;                  // 全局唯一标识符
    uint32 entry;                     // 模板ID
    uint32 typeId;                    // 类型ID
    UpdateMask updateValues;          // 动态属性更新掩码

    virtual void Update(uint32 diff) = 0;  // 纯虚函数
public:
    virtual void AddToWorld();        // 添加到世界
    virtual void RemoveFromWorld();   // 从世界移除
    void BuildUpdateData(UpdateDataMapType&);  // 构建更新数据
};
```

**Unit 战斗单位实现：**
```cpp
class Unit : public WorldObject {
    // 属性系统
    uint32 health, maxHealth;
    Powers powerType;
    uint32 power[MAX_POWERS];

    // 状态系统
    uint32 mechanicMask;              // 机制掩码（昏迷、沉默等）

    // 战斗系统
    Unit* getVictim();                // 当前目标
    void Attack(Unit* victim);        // 攻击
    void CastSpell(SpellInfo const* spellInfo);  // 施法

    // AI 接口
    CreatureAI* i_AI;                 // AI 引用
};
```

**Player 玩家实现：**
```cpp
class Player : public Unit {
    // 会话管理
    WorldSession* m_session;

    // 背包系统
    Item* m_items[MAX_PLAYER_SLOTS];
    uint8 m_inventory;

    // 社交系统
    Guild* guild;
    Group* group;

    // 进度系统
    QuestStatusMap mQuestStatus;
    AchievementMgr achievementMgr;
};
```

### 6. 地图和世界管理

**Map 系统架构：**

```
Map (抽象基类)
│
├── WorldMap (普通世界地图)
│   └── 单一世界实例
│
├── MapInstanced (多实例地图)
│   ├── InstanceMap[] (副本实例数组)
│   └── 实例管理器
│
├── InstanceMap (副本地图)
│   ├── 团队副本
│   ├── 地下城
│   └── 场景
│
└── BattlegroundMap (战场地图)
    ├── 战场实例
    └── 竞技场
```

**Grid 网格系统：**
```cpp
// 网格常量
#define GRID_SIZE      33          // 每个网格 33 码
#define NUMBER_OF_GRIDS 64        // 每个地图 64x64 网格

class Map {
    // 网格管理
    GridMap* GridMaps[MAX_GRID_LEN][MAX_GRID_LEN];

    // 对象追踪
    TypeUnorderedMap<AllObjectType, ObjectGuid> m_objectsStore;

    // 更新管理
    void Update(uint32 diff) override;
};
```

**网格加载优化：**
```cpp
class NGrid {
    bool i_gridState;              // 网格状态
    GridInfo* GridInfo;
    time_t i_unloadTime;           // 卸载时间戳

    // 延迟加载
    bool isGridObjectDataLoaded() const;
    void SetGridObjectDataLoaded(bool on);
};
```

**视野管理机制：**
```cpp
class VisibilityData {
    void AddVisibility(Unit* viewer);
    void RemoveVisibility(Unit* viewer);

    // 视野更新
    void UpdateVisibilityOf(Unit* target);
    void BuildUpdateData(UpdateDataMapType&);
};
```

### 7. 战斗系统

**威胁系统实现：**

```
ThreatMgr (威胁管理器)
│
├── ThreatContainer (威胁容器)
│   ├── HostileReference[] (敌对引用列表)
│   └── 排序的威胁列表
│
└── 当前受害者
```

```cpp
class ThreatMgr {
    ThreatContainer i_threatContainer;     // 威胁容器
    ThreatContainer i_sortedThreatList;    // 排序后的列表
    float i_totalThreat;                    // 总威胁值
    Unit* i_currentVictim;                  // 当前目标

    // 威胁计算
    float AddThreat(Unit* target, float threat);
    void ModifyThreatPercent(Unit* target, int32 percent);

    // 目标选择
    Unit* getHostileTarget();
};

class HostileReference : public Reference<Unit, ThreatMgr> {
    float iThreat;              // 威胁值
    float iTempThreatModifier;  // 临时修正
    int32 iTempThreatReduction; // 临时减少
    bool iOnline;               // 在线状态
};
```

**AI 系统架构：**

```
CreatureAI (抽象基类)
│
├── ScriptedAI (脚本化AI)
│   └── C++ 实现的复杂AI
│
├── SmartAI (智能AI)
│   └── 数据驱动的可配置AI
│
├── NullAI (空AI)
│   └── 静态生物（训练假人等）
│
├── PetAI (宠物AI)
│   └── 宠物专用行为
│
└── GuardAI (卫兵AI)
    └── 城市卫兵行为
```

```cpp
class CreatureAI {
    // 生命周期钩子
    virtual void Reset() = 0;
    virtual void EnterCombat(Unit* who);
    virtual void JustDied(Unit* killer);
    virtual void JustRespawned();

    // 更新循环
    virtual void UpdateAI(uint32 diff) = 0;

    // 移动相关
    virtual void MovementInform(uint32 type, uint32 id);

    // 施法相关
    void DoCast(uint32 spellId);
    void DoCast(Unit* target, uint32 spellId);
    void DoCastVictim(uint32 spellId);

    // 调度系统
    EventMap events;  // 事件调度器
};
```

### 8. 法术系统

**Spell 法术执行流程：**

```
Spell::prepare()
    ↓
Spell::CheckCast() - 施法检查
    ├── 目标检查
    ├── 范围检查
    ├── 资源检查（法力/怒气）
    ├── 冷却检查
    └── 状态检查（沉默/昏迷）
    ↓
Spell::cast() - 执行施法
    ↓
Spell::handle_immediate() - 立即效果
    ↓
Spell::handle_effects() - 处理效果
    ├── SpellEffect::EffectDamage()
    ├── SpellEffect::EffectHeal()
    ├── SpellEffect::EffectBuff()
    └── ... (100+ 效果类型)
    ↓
Spell::finish() - 施法完成
```

**Aura 光环系统：**

```
Aura (光环)
│
├── AuraEffect[] (效果数组)
│   ├── AuraEffect 0 (效果1)
│   ├── AuraEffect 1 (效果2)
│   └── AuraEffect 2 (效果3)
│
└── AuraApplication[] (应用实例)
    ├── 应用到目标1
    ├── 应用到目标2
    └── 应用到目标N
```

### 9. 数据库层

**DatabaseWorkerPool 连接池架构：**

```cpp
template <class T>
class DatabaseWorkerPool {
    enum InternalIndex {
        IDX_ASYNC,          // 异步连接池索引
        IDX_SYNCH,          // 同步连接池索引
        IDX_SIZE            // 索引总数
    };

    // 连接池
    MySQLConnection* m_connections[IDX_SIZE];

    // 工作队列
    ProducerConsumerQueue<SQLOperation*>* m_queues[IDX_SIZE];

    // 线程配置
    uint8 m_workerThreads[IDX_SIZE];

public:
    // 异步操作（非阻塞）
    void Execute(std::string_view sql);
    void Execute(PreparedStatement<T>* stmt);
    QueryResultFuture AsyncQuery(std::string_view sql);

    // 同步操作（阻塞）
    void DirectExecute(std::string_view sql);
    QueryResult Query(std::string_view sql);
    PreparedQueryResult Query(PreparedStatement<T>* stmt);

    // 事务操作
    Transaction<T> BeginTransaction();
};
```

### 10. 关键设计模式

| 设计模式 | 应用场景 | 示例类 |
|---------|---------|--------|
| **单例模式** | 全局管理器 | `sWorld`, `sLog`, `sConfigMgr` |
| **工厂模式** | 对象创建 | `CreatureAISelector`, `GameObjectAI` |
| **观察者模式** | 事件系统 | `ThreatMgr`, `UpdateData` |
| **模板方法** | 算法框架 | `Spell::cast()`, `CreatureAI::UpdateAI()` |
| **策略模式** | 算法选择 | `ThreatCalcHelper`, `MovementGenerator` |
| **适配器模式** | 接口转换 | `Socket<T>`, `DatabaseWorkerPool<T>` |
| **责任链模式** | 事件处理 | `ScriptMgr` Hook 链, `SpellEffect` 处理 |
| **对象池模式** | 资源复用 | `ByteBuffer` Pool, `PacketBuffer` Pool |
| **命令模式** | 操作封装 | `ChatCommand`, `GMCommand` |
| **状态模式** | 状态管理 | `Unit::States`, `Spell::CastState` |

### 11. 代码约定

**命名规范：**
- 类名：`PascalCase` - `Player`, `GameObject`
- 成员变量：`m_camelCase` 或 `camelCase` - `m_health`, `maxPower`
- 方法名：`PascalCase` - `Update()`, `HandleLogin()`
- 局部变量：`camelCase` - `targetCount`, `damage`
- 常量：`UPPER_SNAKE_CASE` - `MAX_PLAYERS`, `GRID_SIZE`
- 宏定义：`UPPER_SNAKE_CASE` - `LOG_FILTER_NETWORK`
- 文件名：对应类名 - `Player.cpp`, `GameObject.h`

### 12. Hook 系统

**ScriptMgr Hook 列表（部分）：**

```cpp
// 服务器生命周期
OnStartup()
OnShutdown()
OnBeforeConfigLoad(bool reload)

// 玩家事件
OnLogin(Player* player)
OnLogout(Player* player)
OnChat(Player* player, uint32 type, std::string& msg)
OnLevelChanged(Player* player, uint8 oldLevel)

// 生物事件
OnCreatureCreate(Creature* creature)
OnCreatureUpdate(Creature* creature, uint32 diff)
OnCreatureDeath(Creature* creature, Unit* killer)

// 法术事件
OnSpellCast(Unit* caster, Spell* spell)
OnSpellFailed(Unit* caster, Spell* spell)
OnAuraApply(Unit* target, Unit* caster, Aura* aura)
```

### 13. Eluna Lua 引擎深度集成

**核心组件：**

```
mod-eluna/
├── src/
│   ├── LuaEngine.cpp           # Lua 引擎核心
│   ├── ElunaQuery.cpp          # 数据库查询封装
│   ├── ElunaEventMgr.cpp       # 事件管理器
│   ├── ElunaTemplate.cpp       # 模板绑定
│   └── Bindings/
│       ├── GlobalFunctions.cpp # 全局函数绑定
│       ├── UnitMethods.cpp     # Unit 方法绑定
│       ├── PlayerMethods.cpp   # Player 方法绑定
│       └── GameObjectMethods.cpp # GameObject 方法绑定
└── LuaEngine.h
```

**Lua 绑定机制：**

```cpp
// 方法注册宏
RegisterMethod(Unit, GetHealth);
RegisterMethod(Unit, SetHealth);
RegisterMethod(Unit, GetMaxHealth);

// Lua 状态管理
class Eluna {
    lua_State* L;                    # Lua 状态机
    EventMgr* eventMgr;              # 事件管理器
    void ExecuteCall(uint8 funcRef); # 执行 Lua 函数
};
```

**Eluna 性能优化：**
- **字节码缓存**：预编译 Lua 脚本为字节码
- **延迟加载**：按需加载 Lua 模块
- **内存管理**：Lua 垃圾回收优化
- **绑定缓存**：C++ 对象绑定缓存

### 14. SmartAI 系统

**数据库驱动 AI 配置：**

```sql
-- Smart AI 脚本表
smart_scripts:
- entryorguid      # 生物 GUID 或模板 ID
- source_type      # 类型 (0/生物, 1/游戏对象, 9/玩家)
- id               # 动作 ID
- link             # 链接到下一个动作
- event_type       # 事件类型 (更新/OOS/血量等)
- event_phase_mask # 阶段掩码
- event_chance     # 触发概率
- event_flags      # 事件标志
- event_param1/2/3/4  # 事件参数
- action_type      # 动作类型
- action_param1/2/3/4/5/6  # 动作参数
- target_type      # 目标类型
- target_param1/2/3  # 目标参数
```

**SmartAI 事件类型：**

| 事件类型 | 说明 | 示例参数 |
|---------|------|---------|
| SMART_EVENT_UPDATE | 定时更新 | 初始延迟, 重复间隔 |
| SMART_EVENT_HP | HP百分比 | HP最小值, HP最大值 |
| SMART_EVENT_MANA | 法力百分比 | 法力最小值, 法力最大值 |
| SMART_EVENT_AGGRO | 进入战斗 | 无 |
| SMART_EVENT_DEATH | 死亡时 | 无 |
| SMART_EVENT_EVADE | 逃离时 | 无 |
| SMART_EVENT_SPELLHIT | 被法术击中 | 法术ID, 学校 |
| SMART_EVENT_RANGE | 进入范围 | 距离最小值, 最大值 |
| SMART_EVENT_OOC_LOS | 脱战且视野内 | 距离最大值 |
| SMART_EVENT_RESPAWN | 重生时 | 无 |
| SMART_EVENT_TARGET_BUFFED | 目标获得光环 | 光环ID |

**SmartAI 动作类型：**

| 动作类型 | 说明 | 参数 |
|---------|------|-----|
| SMART_ACTION_TALK | 说话/喊话 | 文本组ID |
| SMART_ACTION_SET_FACTION | 设置阵营 | 阵营ID |
| SMART_ACTION_MORPH_TO_ENTRY | 变身 | 生物模板ID |
| SMART_ACTION_SOUND | 播放音效 | 音效ID |
| SMART_ACTION_PLAY_EMOTE | 播放表情 | 表情ID |
| SMART_ACTION_CAST | 施放法术 | 法术ID, 目标类型 |
| SMART_ACTION_SUMMON_CREATURE | 召唤生物 | 生物ID, 类型, 时间 |
| SMART_ACTION_ADD_AURA | 添加光环 | 光环ID, 目标 |
| SMART_ACTION_ATTACK_START | 开始攻击 | 无 |
| SMART_ACTION_STORE_TARGET_LIST | 存储目标列表 | 最大数量 |

### 15. 性能优化技术详解

**编译时优化：**

```cmake
# 预编译头文件 (PCH)
target_precompile_headers(compiler_settings
    INTERFACE
        <common/SharedDefines.h>
        <openssl/opensslv.h>
)

# ccache 配置
find_program(CCACHE_PROGRAM ccache)
set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE_PROGRAM})
```

**运行时优化：**

```cpp
// 1. 空间分区 - Grid 系统
class Map {
    GridMap* GridMaps[MAX_GRID_LEN][MAX_GRID_LEN];
    // 只更新玩家所在网格及其周边网格
    void Update(uint32 diff) {
        for (auto& grid : activeGrids) {
            grid->Update(diff);
        }
    }
};

// 2. 对象池 - 复用 ByteBuffer
class ByteBufferPool {
    std::vector<ByteBuffer*> pool;
    ByteBuffer* Allocate() {
        if (!pool.empty()) {
            auto buf = pool.back();
            pool.pop_back();
            return buf;
        }
        return new ByteBuffer();
    }
    void Reclaim(ByteBuffer* buf) {
        buf->clear();
        pool.push_back(buf);
    }
};

// 3. 批量更新 - 减少网络包
class UpdateData {
    std::vector<Unit*> units;
    void BuildPacket(WorldPacket& packet) {
        for (auto unit : units) {
            unit->BuildValuesUpdate(packet);
        }
    }
};

// 4. 惰性加载 - 按需加载
class GridMap {
    bool _loaded;
    void EnsureLoaded() {
        if (!_loaded) {
            LoadFromDB();
            _loaded = true;
        }
    }
};
```

**缓存策略：**

```cpp
// DBC 数据缓存
class DBCStorage {
    std::vector<T> _data;
    std::unordered_map<uint32, uint32> _index;
    T const* LookupEntry(uint32 id) const {
        auto it = _index.find(id);
        return it != _index.end() ? &_data[it->second] : nullptr;
    }
};

// 查询结果缓存
class ObjectMgr {
    CachedQueryResult _creatureCache;
    CreatureTemplate const* GetCreatureTemplate(uint32 id) {
        return _creatureCache.Lookup(id);
    }
};
```

**并发优化：**

```cpp
// 无锁队列
template<typename T>
class LockFreeQueue {
    struct Node {
        T data;
        std::atomic<Node*> next;
    };
    std::atomic<Node*> head;
    std::atomic<Node*> tail;
    void Push(T value);
    bool Pop(T& value);
};

// 读写锁 - 允许多读单写
class EntityMap {
    mutable std::shared_mutex _mutex;
    std::unordered_map<ObjectGuid, Entity*> _map;

    Entity* Get(ObjectGuid guid) const {
        std::shared_lock lock(_mutex);
        return _map[guid];
    }
    void Insert(Entity* entity) {
        std::unique_lock lock(_mutex);
        _map[entity->GetGUID()] = entity;
    }
};
```

---

## 依赖项

### 核心依赖
- **CMake** 3.16-3.22 - 构建系统
- **C++17** 编译器（GCC, Clang, MSVC）
- **MySQL** 5.7+/8.0+ - 数据库
- **OpenSSL** - 加密
- **Boost** - C++ 库集合

### 可选依赖
- **ccache** - 编译缓存
- **LuaJIT/Lua** - 脚本支持
- **Readline** - 命令行编辑

### macOS 特殊要求
```bash
# 安装依赖
brew install mysql openssl@3 readline boost

# 编译时特殊路径（自动处理）
-DMYSQL_ADD_INCLUDE_PATH=/usr/local/include
-DMYSQL_LIBRARY=/usr/local/lib/libmysqlclient.dylib
```

---

## 项目特点总结

### 优势
1. **模块化设计**：易于扩展，无需修改核心代码
2. **稳定可靠**：CI/CD 保证代码质量
3. **社区活跃**：持续维护和更新
4. **文档完善**：Wiki、Doxygen、GitHub文档齐全
5. **工具丰富**：从编译到部署的全套工具链

### 适用场景
- 学习 C++ 服务器开发
- 学习 MMORPG 架构设计
- 学习网络编程和数据库设计
- 创建自定义 WoW 私服
- 游戏服务器研究和开发

### 社区资源
- 官网：https://www.azerothcore.org/
- Wiki：https://www.azerothcore.org/wiki
- Discord：https://discord.gg/gkt4y2x
- GitHub：https://github.com/azerothcore/azerothcore-wotlk
- 模块目录：https://www.azerothcore.org/catalogue

---

## 功能修改快速索引

本章节帮助开发者快速定位需要修改的文件位置。

| 要修改的功能 | 修改位置 | 关键文件 | 备注 |
|------------|---------|---------|------|
| 修改玩家属性/等级/经验 | `src/server/game/Entities/Player/` | Player.cpp, Player.h | 玩家核心逻辑 |
| 修改NPC AI/行为 | `src/server/game/AI/` 或 `src/server/scripts/` | SmartAI.cpp, ScriptedCreature.cpp | 简单用SAI，复杂用C++脚本 |
| 修改法术效果 | `src/server/game/Spells/` | SpellEffects.cpp, Spell.cpp | 法术核心逻辑 |
| 修改副本机制 | `src/server/game/Instances/` + `src/server/scripts/` | InstanceScript.h, 对应副本脚本 | 副本脚本在地图目录下 |
| 添加/修改GM命令 | `src/server/scripts/Commands/` | cs_*.cpp 文件 | 命令脚本统一在这里 |
| 修改物品属性/掉落 | `data/sql/` + `src/server/game/Entities/Item/` | item_template 表, Item.cpp | 主要是数据库数据 |
| 修改任务系统 | `src/server/game/Quests/` + DB | QuestMgr.cpp, quest_template 表 | |
| 修改聊天系统 | `src/server/game/Chat/` | Chat.cpp, ChatHandler.cpp | |
| 修改公会系统 | `src/server/game/Guilds/` | Guild.cpp, GuildMgr.cpp | |
| 修改战场/竞技场 | `src/server/game/Battlegrounds/` | Battleground.cpp, 对应BG脚本 | |
| 修改网络协议/封包 | `src/server/game/Handlers/` | *Handler.cpp | 客户端消息处理 |
| 修改登录/认证 | `src/server/authserver/` | AuthSession.cpp | authserver |
| 修改配置项 | `src/server/shared/Configuration/` + `conf/` | Config.cpp, worldserver.conf.dist | |

---

## C++ 脚本系统详解

### 脚本目录结构

```
src/server/scripts/
├── Commands/          # GM命令脚本 (cs_*.cpp)
├── EasternKingdoms/   # 东部王国副本/区域脚本
├── Kalimdor/          # 卡利姆多脚本
├── Northrend/         # 诺森德脚本
├── Outland/           # 外域脚本
├── Spells/            # 法术脚本
├── World/             # 世界事件脚本
├── Custom/            # 自定义脚本
└── OutdoorPvP/        # 野外PvP脚本
```

### 脚本注册方式

- 脚本通过 `ScriptLoader.cpp.in.cmake` 自动生成加载器
- 每个脚本使用 `void AddSC_XXX()` 函数注册
- CMake 在构建时自动收集所有脚本，无需手动修改

### 主要脚本接口

| 脚本类型 | 基类 | 用途 |
|---------|------|------|
| 生物AI | `CreatureAI` | NPC行为逻辑 |
| 游戏对象AI | `GameObjectAI` | 可交互物体逻辑 |
| 副本脚本 | `InstanceScript` | 副本机制和进度 |
| 法术脚本 | `SpellScript` / `AuraScript` | 法术和光环效果 |
| 玩家脚本 | `PlayerScript` | 玩家事件钩子 |
| 世界脚本 | `WorldScript` | 全局事件钩子 |

---

## 典型开发流程示例

### 案例1：修复某个NPC的法术

1. **查找 NPC entry**
   - 在游戏中用 GM 命令：`.npc info`
   - 或查询数据库：`SELECT entry, name FROM creature_template WHERE name LIKE '%NPC名%';`

2. **查看 AI 类型**
   - 检查 `creature_template.AIName` 字段
   - 如果是 `SmartAI`：修改 `smart_scripts` 数据库表
   - 如果是 `EventAI` 或有特定脚本名：查找对应的 C++ 脚本

3. **定位脚本文件**
   - 根据 NPC 所在区域查找：`src/server/scripts/{地图}/{区域}/`
   - 例如：诺森德的冰冠冰川 → `src/server/scripts/Northrend/IcecrownCitadel/`

4. **修改并测试**
   - 修改脚本后重新编译 worldserver
   - 重启服务器或用 `.reload scripts` 热重载（如支持）

---

### 案例2：修改玩家升级经验

1. **核心逻辑位置**
   - 主文件：`src/server/game/Entities/Player/Player.cpp`
   - 查找函数：`GiveXP()` 或 `LevelUp()`

2. **经验公式**
   - 可能在 `Player::CalculateLevelUpXP()` 或相关函数
   - 也可能在 `XpForLevel` DBC 数据中

3. **数据库数据**
   - `player_levelstats` 表：各等级属性
   - `xp_for_level` 表（如有）：升级所需经验

---

### 案例3：添加新的GM命令

1. **选择命令文件**
   - 在 `src/server/scripts/Commands/` 目录下
   - 选择相关的 `cs_*.cpp` 文件，如 `cs_misc.cpp` 或创建新文件

2. **实现命令处理函数**
   ```cpp
   static bool HandleMyCommand(ChatHandler* handler, const char* args)
   {
       // 命令逻辑
       handler->SendSysMessage("命令执行成功！");
       return true;
   }
   ```

3. **注册命令**
   - 在同一文件的 `AddSC_XXX()` 函数中添加：
   ```cpp
   handler->RegisterCommand("mycmd", HandleMyCommand, SEC_GAMEMASTER, false);
   ```

4. **重新编译**

---

## 关键文件速查表

### 核心管理类（单例）

| 管理器 | 头文件 | 调用方式 | 主要功能 |
|--------|--------|---------|---------|
| 世界管理器 | World.h | `sWorld` | 全局配置、更新循环、在线玩家 |
| 对象管理器 | ObjectMgr.h | `sObjectMgr` | NPC/物品/任务模板加载、缓存 |
| 法术管理器 | SpellMgr.h | `sSpellMgr` | 法术信息、法术书籍 |
| 脚本管理器 | ScriptMgr.h | `sScriptMgr` | 脚本注册、事件钩子调度 |
| 地图管理器 | MapMgr.h | `sMapMgr` | 地图实例创建和管理 |
| 配置管理器 | Config.h | `sConfigMgr` | 配置文件读取、访问配置项 |
| 日志管理器 | Log.h | `sLog` | 日志系统、日志输出 |
| 生物AI选择器 | CreatureAISelector.h | - | 根据配置选择AI类型 |

### 数据库相关

| 组件 | 位置 | 说明 |
|------|------|------|
| 认证数据库操作 | `src/server/database/Database/LoginDatabase.*` | auth 数据库 |
| 角色数据库操作 | `src/server/database/Database/CharacterDatabase.*` | characters 数据库 |
| 世界数据库操作 | `src/server/database/Database/WorldDatabase.*` | world 数据库 |
| SQL 准备语句 | `src/server/database/PreparedStatements/` | 预编译的 SQL 查询 |

---

## 调试技巧

### 日志配置

在 `worldserver.conf` 中配置日志：

```ini
# 控制台输出
Appender.Console=1,5,6

# 文件输出
Appender.File=2,5,7,Server.log,w

# Root logger
Logger.root=4,Console File

# 特定模块日志
Logger.entities.player=6,Console
Logger.spells=5,Console
```

**日志级别：** 0=Disabled, 1=Fatal, 2=Error, 3=Warn, 4=Info, 5=Debug, 6=Trace

**代码中添加日志：**
```cpp
LOG_INFO("entities.player", "玩家 {} 登录，等级: {}", playerName, level);
LOG_DEBUG("spells", "施法 {}，目标: {}", spellId, targetName);
```

### 常用GM调试命令

| 命令 | 功能 |
|------|------|
| `.npc info` | 查看选中NPC的详细信息 |
| `.go creature #id` | 传送到指定NPC |
| `.go object #id` | 传送到指定游戏对象 |
| `.additem #id` | 添加物品到背包 |
| `.learn #spellid` | 学习法术 |
| `.unlearn #spellid` | 遗忘法术 |
| `.levelup #n` | 提升n级 |
| `.debug play all` | 开启所有调试信息 |
| `.reload scripts` | 重新加载脚本 |
| `.reload smart_scripts` | 重新加载SmartAI |

---

## 许可证

- GNU General Public License v2.0
- 非 Blizzard 官方产品
- 仅用于学习和测试目的

---

## 结语

AzerothCore 是一个成熟、稳定、高度模块化的 WoW 服务器模拟器。其架构设计体现了现代 C++ 服务器的最佳实践，包括：
- 清晰的分层架构
- 灵活的模块系统
- 完善的脚本支持
- 强大的配置管理
- 全面的测试覆盖

无论是学习服务器开发、游戏架构，还是创建自定义游戏服务器，AzerothCore 都是一个优秀的参考和基础平台。
