# AzerothCore `.gobject info` 命令深度分析

## 目录

1. [概述](#概述)
2. [前置知识](#前置知识)
3. [命令入口与注册](#命令入口与注册)
4. [核心处理函数详解](#核心处理函数详解)
5. [数据获取流程](#数据获取流程)
6. [关键数据结构](#关键数据结构)
7. [辅助函数分析](#辅助函数分析)
8. [完整执行流程图](#完整执行流程图)
9. [数据库表关系](#数据库表关系)
10. [实战示例](#实战示例)
11. [扩展阅读](#扩展阅读)

---

## 概述

### 什么是 `.gobject info` 命令？

`.gobject info` 是 AzerothCore 中用于查询游戏对象（GameObject）详细信息的 GM 命令。它可以显示：

- **模板信息**：对象类型、显示ID、名称等静态数据
- **实例信息**：位置、旋转、刷新时间等动态数据
- **运行时状态**：当前状态、战利品状态、相位掩码等

### 使用方式

```
.gobject info <entry>           # 通过模板ID查询
.gobject info guid <spawnId>    # 通过实例ID查询
```

**示例**：
。
```
.gobject info 12345        # 查询 entry 为 12345 的游戏对象模板信息
.gobject info guid 67890   # 查询 spawnId 为 67890 的游戏对象实例信息
```

---

## 前置知识

### 核心概念

在深入分析之前，需要理解以下核心概念：

#### 1. GameObject（游戏对象）

游戏对象是魔兽世界中非 NPC 的可交互实体，包括：

| 类型 | 说明 | 示例 |
|------|------|------|
| 门 (DOOR) | 可开关的门 | 副本大门 |
| 箱子 (CHEST) | 可打开的容器 | 宝箱、矿脉 |
| 任务给予者 (QUESTGIVER) | 任务相关对象 | 任务物品 |
| 陷阱 (TRAP) | 触发效果的对象 | 陷阱 |
| 椅子 (CHAIR) | 可坐下的对象 | 椅子、凳子 |
| 传送门 (TRANSPORT) | 传送装置 | 飞艇、船只 |
| 钓鱼点 (FISHINGHOLE) | 钓鱼区域 | 钓鱼点 |

#### 2. Entry 与 SpawnId 的区别

这是理解 `.gobject info` 命令的关键：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Entry (模板ID)                           │
│                                                                 │
│  来源: gameobject_template 表                                   │
│  含义: 定义对象的"类型"，如"铁矿"这个类型的模板                  │
│  特点: 同一种对象只有一个 entry                                 │
│  示例: 铁矿的 entry = 12345                                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 一个 entry 可以有多个实例
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SpawnId (实例ID/GUID)                      │
│                                                                 │
│  来源: gameobject 表                                            │
│  含义: 定义对象的"实例"，如"艾尔文森林的铁矿A"                  │
│  特点: 同一种对象可以有无数个 spawnId                           │
│  示例: 艾尔文森林铁矿A 的 spawnId = 67890                       │
│        艾尔文森林铁矿B 的 spawnId = 67891                       │
│        (两者 entry 相同，但 spawnId 不同)                       │
└─────────────────────────────────────────────────────────────────┘
```

**类比理解**：
- `entry` 相当于"汽车型号"（如：丰田卡罗拉）
- `spawnId` 相当于"车牌号"（如：京A12345、京A67890）

#### 3. 三层数据结构

`.gobject info` 命令涉及三层数据：

```
┌─────────────────────────────────────────────────────────────────┐
│  第一层: GameObjectTemplate (模板数据)                          │
│  ─────────────────────────────────────                          │
│  来源: gameobject_template 数据库表                             │
│  存储: ObjectMgr::_gameObjectTemplateStore                      │
│  内容: entry, type, displayId, name, 类型特定数据               │
│  特点: 静态数据，所有实例共享                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 一个模板对应多个实例
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  第二层: GameObjectData (实例数据)                              │
│  ─────────────────────────────────────                          │
│  来源: gameobject 数据库表                                      │
│  存储: ObjectMgr::_gameObjectDataStore                          │
│  内容: spawnId, id(entry), position, rotation, spawntimesecs    │
│  特点: 持久化数据，保存在数据库中                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 一个实例数据对应一个运行时对象（如果已加载）
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  第三层: GameObject (运行时对象)                                │
│  ─────────────────────────────────────                          │
│  来源: 从数据库加载到内存                                       │
│  存储: Map 系统的 Grid 中                                       │
│  内容: 动态状态、战利品信息、当前状态                           │
│  特点: 只有玩家附近的对象才会加载到内存                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 命令入口与注册

### 文件位置

```
src/server/scripts/Commands/cs_gobject.cpp
```

### 命令注册

命令通过 `GetCommands()` 方法注册到命令系统：

```cpp
// cs_gobject.cpp:44-67
ChatCommandTable GetCommands() const override
{
    static ChatCommandTable gobjectCommandTable =
    {
        // ... 其他命令
        { "info", HandleGameObjectInfoCommand, SEC_MODERATOR, Console::No },
        // ... 其他命令
    };

    static ChatCommandTable commandTable =
    {
        { "gobject", gobjectCommandTable }
    };

    return commandTable;
}
```

**参数说明**：

| 参数 | 值 | 含义 |
|------|-----|------|
| 命令名 | "info" | 子命令名称 |
| 处理函数 | HandleGameObjectInfoCommand | 命令处理函数 |
| 权限级别 | SEC_MODERATOR | 需要协调员及以上权限 |
| 控制台 | Console::No | 不允许控制台执行 |

### 命令解析流程

```
用户输入: ".gobject info 12345"
              │
              ▼
┌─────────────────────────────────────┐
│ 1. ChatHandler::ParseCommands()     │
│    检查前缀 "."                     │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 2. TryExecuteCommand()              │
│    解析: "gobject info 12345"       │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 3. 命令树查找                       │
│    "gobject" → 找到命令节点         │
│    "info" → 找到子命令节点          │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 4. 权限检查                         │
│    SEC_MODERATOR (1) <= 用户权限?   │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 5. 调用 HandleGameObjectInfoCommand │
│    参数: handler, "12345"           │
└─────────────────────────────────────┘
```

---

## 核心处理函数详解

### 函数签名

```cpp
// cs_gobject.cpp:581
static bool HandleGameObjectInfoCommand(
    ChatHandler* handler,
    Optional<EXACT_SEQUENCE("guid")> isGuid,
    Variant<Hyperlink<gameobject_entry>, Hyperlink<gameobject>, uint32> data
);
```

### 参数解析

#### 参数 1: `handler`

- 类型: `ChatHandler*`
- 含义: 命令处理上下文，包含执行者信息、会话等

#### 参数 2: `isGuid`

- 类型: `Optional<EXACT_SEQUENCE("guid")>`
- 含义: 可选参数，如果存在则表示按 spawnId 查询
- 示例: `.gobject info guid 12345` 中 `isGuid` 为 "guid"

#### 参数 3: `data`

- 类型: `Variant<Hyperlink<gameobject_entry>, Hyperlink<gameobject>, uint32>`
- 含义: 查询目标，可以是：
  - `Hyperlink<gameobject_entry>`: 游戏内链接（entry）
  - `Hyperlink<gameobject>`: 游戏内链接（spawnId）
  - `uint32`: 直接输入的数字

### 完整函数代码

```cpp
// cs_gobject.cpp:581-642
static bool HandleGameObjectInfoCommand(
    ChatHandler* handler,
    Optional<EXACT_SEQUENCE("guid")> isGuid,
    Variant<Hyperlink<gameobject_entry>, Hyperlink<gameobject>, uint32> data)
{
    // ========== 第一部分: 变量声明 ==========
    uint32 entry = 0;           // 模板ID
    uint32 type = 0;            // 对象类型
    uint32 displayId = 0;       // 显示ID
    std::string name;           // 对象名称
    uint32 lootId = 0;          // 战利品ID
    GameObject* gameObject = nullptr;  // 运行时对象

    // ========== 第二部分: 解析参数，确定查询方式 ==========

    ObjectGuid::LowType spawnId = 0;

    // 方式一: 通过 spawnId 查询
    // 条件: 使用了 "guid" 关键字，或者传入的是 gameobject 链接
    if (isGuid || data.holds_alternative<Hyperlink<gameobject>>())
    {
        spawnId = *data;  // 获取 spawnId

        // 从 ObjectMgr 获取实例数据
        GameObjectData const* spawnData = sObjectMgr->GetGameObjectData(spawnId);
        if (!spawnData)
        {
            handler->SendErrorMessage(LANG_COMMAND_OBJNOTFOUND, spawnId);
            return false;
        }

        // 从实例数据获取 entry
        entry = spawnData->id;

        // 尝试获取运行时对象（可能不存在，如果对象不在玩家附近）
        gameObject = handler->GetObjectFromPlayerMapByDbGuid(spawnId);
    }
    // 方式二: 通过 entry 查询
    else
    {
        entry = *data;  // 直接使用 entry
    }

    // ========== 第三部分: 获取模板信息 ==========

    GameObjectTemplate const* gameObjectInfo = sObjectMgr->GetGameObjectTemplate(entry);
    if (!gameObjectInfo)
    {
        handler->SendErrorMessage(LANG_GAMEOBJECT_NOT_EXIST, entry);
        return false;
    }

    // 提取模板信息
    type = gameObjectInfo->type;
    displayId = gameObjectInfo->displayId;
    name = gameObjectInfo->name;

    // 根据类型获取战利品ID
    if (type == GAMEOBJECT_TYPE_CHEST)
        lootId = gameObjectInfo->chest.lootId;
    else if (type == GAMEOBJECT_TYPE_FISHINGHOLE)
        lootId = gameObjectInfo->fishinghole.lootId;

    // ========== 第四部分: 输出信息 ==========

    // 基础模板信息
    handler->PSendSysMessage(LANG_GOINFO_ENTRY, entry);

    // 运行时对象信息（仅当对象存在于当前地图时）
    if (gameObject)
        handler->PSendSysMessage("GUID: {}", gameObject->GetGUID().ToString());

    handler->PSendSysMessage(LANG_GOINFO_TYPE, type);
    handler->PSendSysMessage(LANG_GOINFO_LOOTID, lootId);
    handler->PSendSysMessage(LANG_GOINFO_DISPLAYID, displayId);

    // 运行时状态信息（仅当对象存在时）
    if (gameObject)
    {
        handler->PSendSysMessage("LootMode: {}", gameObject->GetLootMode());
        handler->PSendSysMessage("LootState: {}", gameObject->getLootState());
        handler->PSendSysMessage("GOState: {}", gameObject->GetGoState());
        handler->PSendSysMessage("PhaseMask: {}", gameObject->GetPhaseMask());
        handler->PSendSysMessage("IsLootEmpty: {}", gameObject->loot.empty());
        handler->PSendSysMessage("IsLootLooted: {}", gameObject->loot.isLooted());
    }

    handler->PSendSysMessage(LANG_GOINFO_NAME, name);

    return true;
}
```

---

## 数据获取流程

### 流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                     .gobject info 命令                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  参数解析: entry 或 spawnId (guid)                               │
│                                                                 │
│  .gobject info 12345        → entry = 12345                     │
│  .gobject info guid 67890   → spawnId = 67890                   │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┴───────────────────┐
          │                                       │
          ▼                                       ▼
┌──────────────────────┐              ┌──────────────────────┐
│   通过 entry 查询    │              │  通过 spawnId 查询   │
│                      │              │                      │
│  entry = *data       │              │  spawnId = *data     │
└──────────┬───────────┘              │  ↓                   │
           │                          │  GetGameObjectData() │
           │                          │  ↓                   │
           │                          │  entry = spawnData->id│
           │                          └──────────┬───────────┘
           │                                     │
           └──────────────────┬──────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  获取 GameObjectTemplate (模板数据)                             │
│                                                                 │
│  sObjectMgr->GetGameObjectTemplate(entry)                       │
│  ↓                                                              │
│  从 _gameObjectTemplateStore 内存缓存中查找                      │
│  ↓                                                              │
│  返回: entry, type, displayId, name, 类型特定数据               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  尝试获取 GameObject (运行时对象)                               │
│                                                                 │
│  handler->GetObjectFromPlayerMapByDbGuid(spawnId)               │
│  ↓                                                              │
│  从玩家当前地图的 Grid 中查找                                    │
│  ↓                                                              │
│  返回: GameObject* 或 nullptr                                   │
│  (只有玩家附近的对象才会被加载到内存)                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  输出信息                                                       │
│                                                                 │
│  - 模板信息: entry, type, displayId, name, lootId              │
│  - 运行时信息: GUID, LootState, GOState, PhaseMask 等          │
└─────────────────────────────────────────────────────────────────┘
```

### 数据来源详解

#### 1. GetGameObjectTemplate - 获取模板数据

```cpp
// ObjectMgr.h:761
GameObjectTemplate const* GetGameObjectTemplate(uint32 entry);
```

**实现逻辑**：

```cpp
// ObjectMgr.cpp (简化版)
GameObjectTemplate const* ObjectMgr::GetGameObjectTemplate(uint32 entry)
{
    // 从内存缓存中查找
    auto it = _gameObjectTemplateStore.find(entry);
    if (it == _gameObjectTemplateStore.end())
        return nullptr;

    return &it->second;
}
```

**数据来源**：数据库表 `gameobject_template`

**加载时机**：服务器启动时，`ObjectMgr::LoadGameObjectTemplates()` 从数据库加载所有模板到内存

#### 2. GetGameObjectData - 获取实例数据

```cpp
// ObjectMgr.h:1267-1272
GameObjectData const* GetGameObjectData(ObjectGuid::LowType spawnId) const
{
    GameObjectDataContainer::const_iterator itr = _gameObjectDataStore.find(spawnId);
    if (itr == _gameObjectDataStore.end())
        return nullptr;
    return &itr->second;
}
```

**数据来源**：数据库表 `gameobject`

**加载时机**：服务器启动时，`ObjectMgr::LoadGameobjects()` 从数据库加载所有实例数据到内存

#### 3. GetObjectFromPlayerMapByDbGuid - 获取运行时对象

```cpp
// Chat.cpp:597-607
GameObject* ChatHandler::GetObjectFromPlayerMapByDbGuid(ObjectGuid::LowType lowguid)
{
    if (!m_session)
        return nullptr;

    // 从玩家当前地图的 GameObject 存储中查找
    auto bounds = GetPlayer()->GetMap()->GetGameObjectBySpawnIdStore().equal_range(lowguid);

    if (bounds.first != bounds.second)
        return bounds.first->second;

    return nullptr;
}
```

**关键点**：
- 只能获取玩家当前地图的对象
- 只有玩家附近（Grid 范围内）的对象才会被加载
- 如果对象不在玩家附近，返回 `nullptr`

---

## 关键数据结构

### 1. GameObjectTemplate - 模板数据结构

**文件位置**: `src/server/game/Entities/GameObject/GameObjectData.h:31-663`

```cpp
struct GameObjectTemplate
{
    uint32  entry;          // 模板ID
    uint32  type;           // 对象类型 (GAMEOBJECT_TYPE_*)
    uint32  displayId;      // 显示模型ID
    std::string name;       // 对象名称
    std::string IconName;   // 图标名称
    std::string castBarCaption;  // 施法条标题
    std::string unk1;       // 未知字段
    float   size;           // 对象大小

    // 类型特定数据（使用 union 节省内存）
    union
    {
        // GAMEOBJECT_TYPE_DOOR (0)
        struct {
            uint32 startOpen;
            uint32 lockId;
            uint32 autoCloseTime;
            // ...
        } door;

        // GAMEOBJECT_TYPE_CHEST (3)
        struct {
            uint32 lockId;
            uint32 lootId;          // 战利品ID
            uint32 chestRestockTime;
            uint32 consumable;
            // ...
        } chest;

        // GAMEOBJECT_TYPE_FISHINGHOLE (25)
        struct {
            uint32 radius;
            uint32 lootId;          // 战利品ID
            // ...
        } fishinghole;

        // ... 其他类型

        // 原始数据访问
        struct {
            uint32 data[MAX_GAMEOBJECT_DATA];
        } raw;
    };

    std::string AIName;     // AI 名称
    uint32 ScriptId;        // 脚本ID
    bool IsForQuests;       // 是否用于任务

    // 辅助方法
    [[nodiscard]] uint32 GetLootId() const;
    [[nodiscard]] uint32 GetLockId() const;
    [[nodiscard]] bool IsDespawnAtAction() const;
    // ...
};
```

**设计说明**：
- 使用 `union` 存储类型特定数据，不同类型的对象有不同的数据字段
- 例如：箱子（CHEST）有 `lootId`，门（DOOR）有 `autoCloseTime`

### 2. GameObjectData - 实例数据结构

**文件位置**: `src/server/game/Entities/GameObject/GameObjectData.h:714-723`

```cpp
struct GameObjectData : public SpawnData
{
    GameObjectData() : SpawnData(SPAWN_TYPE_GAMEOBJECT) {}

    uint32 id{0};               // 对应的 entry (gameobject_template.entry)
    G3D::Quat rotation;         // 四元数旋转
    int32 spawntimesecs{0};     // 刷新时间（秒）
    uint32 animprogress{0};     // 动画进度
    GOState go_state{GO_STATE_ACTIVE};  // 初始状态
    uint8 artKit{0};            // 外观套件
};
```

**继承的 SpawnData 字段**：

```cpp
struct SpawnData
{
    uint32 mapid{0};            // 地图ID
    float posX{0.0f};           // X坐标
    float posY{0.0f};           // Y坐标
    float posZ{0.0f};           // Z坐标
    float orientation{0.0f};    // 朝向
    uint32 phaseMask{0};        // 相位掩码
    // ...
};
```

### 3. GameObject - 运行时对象类

**文件位置**: `src/server/game/Entities/GameObject/GameObject.h`

```cpp
class GameObject : public WorldObject, public GridObject<GameObject>
{
public:
    // 获取模板信息
    [[nodiscard]] GameObjectTemplate const* GetGOInfo() const { return m_goInfo; }
    [[nodiscard]] GameObjectData const* GetGameObjectData() const { return m_goData; }
    [[nodiscard]] ObjectGuid::LowType GetSpawnId() const { return m_spawnId; }

    // 获取运行时状态
    [[nodiscard]] GOState GetGoState() const;
    [[nodiscard]] LootState getLootState() const;
    [[nodiscard]] uint16 GetLootMode() const;
    [[nodiscard]] uint32 GetPhaseMask() const;

    // 战利品相关
    Loot loot;  // 战利品对象

    // ... 其他方法和成员
private:
    GameObjectTemplate const* m_goInfo;  // 模板数据指针
    GameObjectData const* m_goData;      // 实例数据指针
    ObjectGuid::LowType m_spawnId;       // 实例ID
    // ...
};
```

---

## 辅助函数分析

### GetObjectFromPlayerMapByDbGuid

**文件位置**: `src/server/game/Chat/Chat.cpp:597-607`

```cpp
GameObject* ChatHandler::GetObjectFromPlayerMapByDbGuid(ObjectGuid::LowType lowguid)
{
    // 检查是否有会话（控制台没有会话）
    if (!m_session)
        return nullptr;

    // 从玩家当前地图获取 GameObject 存储
    // GetGameObjectBySpawnIdStore() 返回一个 multimap
    // key: spawnId, value: GameObject*
    auto bounds = GetPlayer()->GetMap()->GetGameObjectBySpawnIdStore().equal_range(lowguid);

    // 如果找到，返回第一个匹配的对象
    if (bounds.first != bounds.second)
        return bounds.first->second;

    return nullptr;
}
```

**为什么使用 multimap？**

因为理论上同一个 spawnId 可能对应多个 GameObject（虽然通常只有一个），使用 `equal_range` 可以获取所有匹配的对象。

### Map 的 GameObject 存储

```cpp
// Map 内部存储结构（简化）
class Map
{
    // 通过 spawnId 查找 GameObject
    std::unordered_multimap<ObjectGuid::LowType, GameObject*> m_gameObjectBySpawnId;

    // 通过 GUID 查找 GameObject
    std::unordered_map<ObjectGuid, GameObject*> m_gameObjects;

public:
    auto& GetGameObjectBySpawnIdStore() { return m_gameObjectBySpawnId; }
};
```

---

## 完整执行流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户输入命令                                       │
│                                                                             │
│   游戏内: .gobject info 12345                                               │
│   或:    .gobject info guid 67890                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        1. 命令解析阶段                                       │
│                                                                             │
│   ChatHandler::ParseCommands()                                              │
│   ├── 检查命令前缀 (.)                                                      │
│   ├── 去掉前缀得到: "gobject info 12345"                                    │
│   └── 调用 TryExecuteCommand()                                              │
│                                                                             │
│   TryExecuteCommand()                                                       │
│   ├── 分词: ["gobject", "info", "12345"]                                   │
│   ├── 查找命令树: COMMAND_MAP["gobject"] → ChatCommandNode                  │
│   ├── 查找子命令: node._subCommands["info"] → ChatCommandNode               │
│   ├── 权限检查: IsInvokerVisible()                                          │
│   │   ├── 检查是否允许控制台执行 (Console::No → 控制台不可)                 │
│   │   └── 检查权限级别 (SEC_MODERATOR ≤ 用户权限)                           │
│   └── 调用命令处理函数                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     2. HandleGameObjectInfoCommand 执行                      │
│                                                                             │
│   参数解析:                                                                  │
│   ├── isGuid = nullopt (没有 "guid" 关键字)                                 │
│   └── data = 12345 (uint32)                                                 │
│                                                                             │
│   确定查询方式:                                                              │
│   └── isGuid 为 false，使用 entry 方式                                      │
│       └── entry = 12345                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     3. 获取模板数据                                          │
│                                                                             │
│   sObjectMgr->GetGameObjectTemplate(12345)                                  │
│   │                                                                         │
│   ├── 查找 _gameObjectTemplateStore (内存缓存)                              │
│   │   └── 数据来源: gameobject_template 数据库表                            │
│   │                                                                         │
│   └── 返回 GameObjectTemplate*                                              │
│       ├── entry = 12345                                                     │
│       ├── type = 3 (GAMEOBJECT_TYPE_CHEST)                                  │
│       ├── displayId = 456                                                   │
│       ├── name = "富瑟银矿"                                                 │
│       └── chest.lootId = 789                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     4. 尝试获取运行时对象                                    │
│                                                                             │
│   handler->GetObjectFromPlayerMapByDbGuid(spawnId)                          │
│   │                                                                         │
│   ├── spawnId = 0 (entry 模式没有 spawnId)                                  │
│   │                                                                         │
│   └── 返回 nullptr (无法获取运行时对象)                                      │
│                                                                             │
│   注: 如果使用 "guid" 模式，且对象在玩家附近，则返回 GameObject*            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     5. 输出信息                                              │
│                                                                             │
│   模板信息:                                                                  │
│   ├── Entry: 12345                                                          │
│   ├── Type: 3 (CHEST)                                                       │
│   ├── LootId: 789                                                           │
│   ├── DisplayId: 456                                                        │
│   └── Name: 富瑟银矿                                                        │
│                                                                             │
│   运行时信息 (仅当 gameObject 存在时):                                       │
│   ├── GUID: Creature:0x0                                                    │
│   ├── LootMode: 1                                                           │
│   ├── LootState: GO_LOOTSTATE_READY                                        │
│   ├── GOState: GO_STATE_READY                                              │
│   ├── PhaseMask: 1                                                          │
│   ├── IsLootEmpty: true                                                     │
│   └── IsLootLooted: false                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 数据库表关系

### 表结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    gameobject_template 表                       │
│                    (对象模板定义)                                │
├─────────────────────────────────────────────────────────────────┤
│ entry        │ uint32   │ 主键，模板ID                          │
│ type         │ uint32   │ 对象类型                              │
│ displayId    │ uint32   │ 显示模型ID                            │
│ name         │ varchar  │ 对象名称                              │
│ IconName     │ varchar  │ 图标名称                              │
│ castBarCaption│ varchar │ 施法条标题                            │
│ size         │ float    │ 对象大小                              │
│ data0-data23 │ uint32   │ 类型特定数据                          │
│ AIName       │ varchar  │ AI名称                                │
│ ScriptName   │ varchar  │ 脚本名称                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 1:N
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       gameobject 表                             │
│                    (对象实例定义)                                │
├─────────────────────────────────────────────────────────────────┤
│ guid         │ uint32   │ 主键，实例ID (spawnId)                │
│ id           │ uint32   │ 外键 → gameobject_template.entry     │
│ map          │ uint32   │ 地图ID                                │
│ zoneId       │ uint32   │ 区域ID                                │
│ areaId       │ uint32   │ 子区域ID                              │
│ phaseMask    │ uint32   │ 相位掩码                              │
│ position_x   │ float    │ X坐标                                 │
│ position_y   │ float    │ Y坐标                                 │
│ position_z   │ float    │ Z坐标                                 │
│ orientation  │ float    │ 朝向                                  │
│ rotation0-3  │ float    │ 四元数旋转                            │
│ spawntimesecs│ int32    │ 刷新时间(秒)                          │
│ animprogress │ uint32   │ 动画进度                              │
│ state        │ uint8    │ 初始状态                              │
└─────────────────────────────────────────────────────────────────┘
```

### 查询示例

```sql
-- 查询所有铁矿的模板信息
SELECT entry, name, type, data1 as lootId
FROM gameobject_template
WHERE name LIKE '%铁矿%';

-- 查询特定模板的所有实例
SELECT g.guid, g.id, g.position_x, g.position_y, g.position_z, g.map
FROM gameobject g
WHERE g.id = 12345;  -- 替换为实际的 entry

-- 查询特定实例的详细信息（关联查询）
SELECT
    g.guid as spawnId,
    g.id as entry,
    gt.name,
    gt.type,
    g.position_x, g.position_y, g.position_z,
    g.map,
    g.spawntimesecs
FROM gameobject g
JOIN gameobject_template gt ON g.id = gt.entry
WHERE g.guid = 67890;  -- 替换为实际的 spawnId
```

---

## 实战示例

### 示例 1: 查询矿脉信息

```
输入: .gobject info 12345

输出:
Entry: 12345
Type: 3 (CHEST)
LootId: 789
DisplayId: 456
Name: 富瑟银矿
```

**分析**：
- 这是通过 entry 查询的例子
- 只显示了模板信息，没有运行时状态
- 因为没有指定 spawnId，无法获取运行时对象

### 示例 2: 查询特定矿脉实例

```
输入: .gobject info guid 67890

输出 (如果对象在玩家附近):
Entry: 12345
GUID: Creature:0x123456789
Type: 3 (CHEST)
LootId: 789
DisplayId: 456
LootMode: 1
LootState: 0 (READY)
GOState: 1 (READY)
PhaseMask: 1
IsLootEmpty: true
IsLootLooted: false
Name: 富瑟银矿
```

**分析**：
- 这是通过 spawnId 查询的例子
- 显示了完整的模板信息和运行时状态
- 因为对象在玩家附近，所以能获取到运行时对象

### 示例 3: 对象不在玩家附近

```
输入: .gobject info guid 99999

输出:
Entry: 12345
Type: 3 (CHEST)
LootId: 789
DisplayId: 456
Name: 富瑟银矿
```

**分析**：
- 虽然使用了 spawnId 查询
- 但对象不在玩家附近（未加载到内存）
- 所以只显示模板信息，没有运行时状态

---

## 扩展阅读

### 相关命令

| 命令 | 说明 |
|------|------|
| `.gobject target` | 查找最近的游戏对象 |
| `.gobject near` | 列出附近的游戏对象 |
| `.gobject add` | 添加新的游戏对象 |
| `.gobject delete` | 删除游戏对象 |
| `.gobject move` | 移动游戏对象 |
| `.gobject respawn` | 重生游戏对象 |

### 相关源文件

| 文件 | 说明 |
|------|------|
| `cs_gobject.cpp` | gobject 命令实现 |
| `GameObject.h/cpp` | GameObject 类定义 |
| `GameObjectData.h` | 数据结构定义 |
| `ObjectMgr.h/cpp` | 对象管理器 |
| `Chat.h/cpp` | 命令处理框架 |

### 对象类型枚举

```cpp
enum GameObjectTypes
{
    GAMEOBJECT_TYPE_DOOR                = 0,
    GAMEOBJECT_TYPE_BUTTON              = 1,
    GAMEOBJECT_TYPE_QUESTGIVER          = 2,
    GAMEOBJECT_TYPE_CHEST               = 3,
    GAMEOBJECT_TYPE_BINDER              = 4,
    GAMEOBJECT_TYPE_GENERIC             = 5,
    GAMEOBJECT_TYPE_TRAP                = 6,
    GAMEOBJECT_TYPE_CHAIR               = 7,
    GAMEOBJECT_TYPE_SPELL_FOCUS         = 8,
    GAMEOBJECT_TYPE_TEXT                = 9,
    GAMEOBJECT_TYPE_GOOBER              = 10,
    GAMEOBJECT_TYPE_TRANSPORT           = 11,
    GAMEOBJECT_TYPE_AREADAMAGE          = 12,
    GAMEOBJECT_TYPE_CAMERA              = 13,
    GAMEOBJECT_TYPE_MAPOBJECT           = 14,
    GAMEOBJECT_TYPE_MO_TRANSPORT        = 15,
    GAMEOBJECT_TYPE_DUELFLAG            = 16,
    GAMEOBJECT_TYPE_FISHINGNODE         = 17,
    GAMEOBJECT_TYPE_SUMMONING_RITUAL    = 18,
    GAMEOBJECT_TYPE_MAILBOX             = 19,
    GAMEOBJECT_TYPE_DONOTUSE            = 20,
    GAMEOBJECT_TYPE_GUARDPOST           = 21,
    GAMEOBJECT_TYPE_SPELLCASTER         = 22,
    GAMEOBJECT_TYPE_MEETINGSTONE        = 23,
    GAMEOBJECT_TYPE_FLAGSTAND           = 24,
    GAMEOBJECT_TYPE_FISHINGHOLE         = 25,
    GAMEOBJECT_TYPE_FLAGDROP            = 26,
    GAMEOBJECT_TYPE_MINI_GAME           = 27,
    GAMEOBJECT_TYPE_UNKNOWN             = 28,
    GAMEOBJECT_TYPE_CAPTURE_POINT       = 29,
    GAMEOBJECT_TYPE_AURA_GENERATOR      = 30,
    GAMEOBJECT_TYPE_DUNGEON_DIFFICULTY  = 31,
    GAMEOBJECT_TYPE_BARBER_CHAIR        = 32,
    GAMEOBJECT_TYPE_DESTRUCTIBLE_BUILDING = 33,
    GAMEOBJECT_TYPE_GUILDBANK           = 34,
    GAMEOBJECT_TYPE_TRAPDOOR            = 35
};
```

### 状态枚举

```cpp
// GOState - 对象状态
enum GOState
{
    GO_STATE_ACTIVE             = 0,  // 激活状态（门打开）
    GO_STATE_READY              = 1,  // 准备状态（门关闭）
    GO_STATE_ACTIVE_ALTERNATIVE = 2   // 备用激活状态
};

// LootState - 战利品状态
enum LootState
{
    GO_LOOTSTATE_READY          = 0,  // 可以被打开
    GO_LOOTSTATE_ACTIVATED      = 1,  // 正在被打开
    GO_LOOTSTATE_JUST_DEACTIVATED = 2, // 刚刚被打开
    GO_LOOTSTATE_DEACTIVATED    = 3   // 已被打开，等待刷新
};
```

---

## 总结

### 核心要点

1. **两种查询方式**：
   - 通过 `entry` 查询：只能获取模板信息
   - 通过 `spawnId` 查询：可以获取模板信息 + 运行时状态（如果对象已加载）

2. **三层数据结构**：
   - `GameObjectTemplate`：模板数据，所有实例共享
   - `GameObjectData`：实例数据，保存在数据库
   - `GameObject`：运行时对象，只有玩家附近才会加载

3. **数据来源**：
   - 模板数据：`gameobject_template` 表 → `_gameObjectTemplateStore`
   - 实例数据：`gameobject` 表 → `_gameObjectDataStore`
   - 运行时对象：Map 的 Grid 系统

4. **权限要求**：
   - 需要 `SEC_MODERATOR` 及以上权限
   - 不允许控制台执行

---

*文档版本: 1.0*
*最后更新: 2024*
*适用于: AzerothCore WotLK*
