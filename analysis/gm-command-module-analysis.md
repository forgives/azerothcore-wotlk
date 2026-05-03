# AzerothCore GM 命令模块深度分析

## 目录

1. [概述](#概述)
2. [架构设计](#架构设计)
3. [核心组件详解](#核心组件详解)
4. [命令注册机制](#命令注册机制)
5. [命令解析流程](#命令解析流程)
6. [权限控制系统](#权限控制系统)
7. [参数解析系统](#参数解析系统)
8. [实战示例分析](#实战示例分析)
9. [如何添加新命令](#如何添加新命令)
10. [常见问题与调试](#常见问题与调试)

---

## 概述

### 什么是 GM 命令系统？

GM（Game Master）命令系统是 AzerothCore 中用于服务器管理的核心功能。它允许具有特定权限的账户通过聊天框或控制台执行各种管理操作，如：

- 传送玩家
- 修改角色属性
- 生成/删除 NPC 和物品
- 管理账户和角色
- 重载配置
- 服务器维护

### 命令格式

游戏内命令以 `.` 或 `!` 开头：

```
.gm on          # 开启 GM 模式
.npc add 12345  # 生成 ID 为 12345 的 NPC
.tele orgrimmar # 传送到奥格瑞玛
```

控制台命令不需要前缀：

```
server shutdown 60    # 60秒后关闭服务器
account create test test123  # 创建账户
```

---

## 架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户输入层                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  游戏内聊天   │    │  控制台输入   │    │  远程管理(RA) │      │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
└─────────┼───────────────────┼───────────────────┼───────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                     ChatHandler 层                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ChatHandler (游戏内) / CliHandler (控制台)               │   │
│  │  - ParseCommands(): 解析命令入口                          │   │
│  │  - SendSysMessage(): 发送系统消息                         │   │
│  │  - GetPlayer(): 获取当前玩家                              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   命令解析层 (ChatCommand)                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  ChatCommandNode                                         │   │
│  │  - TryExecuteCommand(): 查找并执行命令                    │   │
│  │  - LoadCommandMap(): 加载命令映射表                       │   │
│  │  - SendCommandHelp(): 显示帮助信息                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    命令脚本层 (CommandScript)                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │cs_gm.cpp   │ │cs_npc.cpp  │ │cs_go.cpp   │ │cs_*.cpp    │   │
│  │GM相关命令  │ │NPC命令     │ │传送命令    │ │其他命令    │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 目录结构

```
src/server/
├── game/                              # 核心游戏逻辑
│   ├── Chat/                          # 聊天系统核心
│   │   ├── Chat.h/cpp                 # ChatHandler 类定义
│   │   └── ChatCommands/              # 命令解析框架
│   │       ├── ChatCommand.h/cpp      # 命令节点与执行
│   │       ├── ChatCommandArgs.h/cpp  # 参数解析器
│   │       ├── ChatCommandTags.h/cpp  # 参数标签类型
│   │       └── ChatCommandHelpers.h   # 辅助函数
│   └── Scripting/ScriptDefines/       # 脚本基类
│       └── CommandScript.h/cpp        # 命令脚本基类
│
└── scripts/Commands/                  # 具体命令实现
    ├── cs_script_loader.cpp           # 脚本加载器
    ├── cs_gm.cpp                      # GM 模式命令
    ├── cs_npc.cpp                     # NPC 相关命令
    ├── cs_go.cpp                      # 传送命令
    ├── cs_character.cpp               # 角色管理命令
    ├── cs_account.cpp                 # 账户管理命令
    ├── cs_reload.cpp                  # 重载命令
    └── ...                            # 其他命令文件
```

---

## 核心组件详解

### 1. ChatHandler 类

`ChatHandler` 是命令处理的核心类，负责命令解析、消息发送和权限检查。

**文件位置**: `src/server/game/Chat/Chat.h`

```cpp
class ChatHandler
{
public:
    explicit ChatHandler(WorldSession* session);

    // 命令解析入口
    bool ParseCommands(std::string_view text);
    bool _ParseCommands(std::string_view text);

    // 消息发送
    void SendSysMessage(std::string_view str, bool escapeCharacters = false);
    void SendNotification(std::string_view str);
    void SendErrorMessage(uint32 entry);
    void SendGlobalGMSysMessage(const char* str);

    // 权限检查
    bool HasLowerSecurity(Player* target, ObjectGuid guid = ObjectGuid::Empty, bool strong = false);
    bool IsAvailable(uint32 securityLevel) const;

    // 获取目标
    Player* GetPlayer() const;
    Player* getSelectedPlayer() const;
    Creature* getSelectedCreature() const;
    Unit* getSelectedUnit() const;

    // 工具方法
    bool IsConsole() const { return (m_session == nullptr); }
    std::string GetNameLink(Player* chr) const;

protected:
    WorldSession* m_session;    // 玩家会话（控制台为 nullptr）
    bool sentErrorMessage;      // 是否已发送错误消息
};
```

**关键方法解析**:

#### ParseCommands - 命令解析入口

```cpp
bool ChatHandler::ParseCommands(std::string_view text)
{
    ASSERT(!text.empty());

    // 检查命令前缀（必须是 . 或 ! 开头）
    if ((text[0] != '!') && (text[0] != '.'))
        return false;

    // 忽略单个 . 或 !
    if (text.length() < 2)
        return false;

    // 忽略连续的点（如 ...）
    if (text[1] == text[0])
        return false;

    // 忽略分隔符开头
    if (text[1] == Acore::Impl::ChatCommands::COMMAND_DELIMITER)
        return false;

    // 去掉前缀后解析
    return _ParseCommands(text.substr(1));
}
```

#### _ParseCommands - 实际命令执行

```cpp
bool ChatHandler::_ParseCommands(std::string_view text)
{
    // 尝试执行命令
    if (Acore::ChatCommands::TryExecuteCommand(*this, text))
        return true;

    // 对普通玩家隐藏命令不存在的信息
    if (m_session && AccountMgr::IsPlayerAccount(m_session->GetSecurity()) 
        && !sWorld->getBoolConfig(CONFIG_ALLOW_PLAYER_COMMANDS))
        return false;

    // 对 GM 显示错误
    SendErrorMessage(LANG_CMD_INVALID, text);
    return true;
}
```

### 2. CliHandler 类

`CliHandler` 继承自 `ChatHandler`，专门用于控制台命令处理。

```cpp
class CliHandler : public ChatHandler
{
public:
    using Print = void(void*, std::string_view);
    explicit CliHandler(void* callbackArg, Print* zprint);

    // 重写的方法
    std::string GetAcoreString(uint32 entry) const override;
    void SendSysMessage(std::string_view, bool escapeCharacters) override;
    bool ParseCommands(std::string_view str) override;

    // 控制台总是可以输出
    bool HasSession() const override;

private:
    void* m_callbackArg;
    Print* m_print;    // 输出回调函数
};
```

### 3. CommandScript 类

`CommandScript` 是所有命令脚本的基类。

**文件位置**: `src/server/game/Scripting/ScriptDefines/CommandScript.h`

```cpp
class CommandScript : public ScriptObject
{
protected:
    CommandScript(const char* name);

public:
    // 必须实现：返回命令表
    [[nodiscard]] virtual std::vector<Acore::ChatCommands::ChatCommandBuilder> GetCommands() const = 0;
};
```

**实现原理**:

```cpp
// CommandScript.cpp
CommandScript::CommandScript(const char* name)
    : ScriptObject(name)
{
    // 自动注册到脚本系统
    ScriptRegistry<CommandScript>::AddScript(this);
}

// ScriptMgr 获取所有命令
Acore::ChatCommands::ChatCommandTable ScriptMgr::GetChatCommands()
{
    Acore::ChatCommands::ChatCommandTable table;

    // 遍历所有已注册的命令脚本
    for (auto const& [scriptID, script] : ScriptRegistry<CommandScript>::ScriptPointerList)
    {
        Acore::ChatCommands::ChatCommandTable cmds = script->GetCommands();
        std::move(cmds.begin(), cmds.end(), std::back_inserter(table));
    }

    return table;
}
```

### 4. ChatCommandNode 类

`ChatCommandNode` 是命令树的核心数据结构，每个节点代表一个命令或子命令。

**文件位置**: `src/server/game/Chat/ChatCommands/ChatCommand.h`

```cpp
class ChatCommandNode
{
public:
    static void LoadCommandMap();           // 加载命令映射
    static bool TryExecuteCommand(ChatHandler& handler, std::string_view cmd);  // 执行命令
    static void SendCommandHelpFor(ChatHandler& handler, std::string_view cmd); // 显示帮助

private:
    std::string _name;                      // 命令名称
    CommandInvoker _invoker;                // 命令执行器
    CommandPermissions _permission;         // 权限配置
    std::variant<std::monostate, AcoreStrings, std::string> _help;  // 帮助文本
    std::map<std::string_view, ChatCommandNode, StringCompareLessI_T> _subCommands;  // 子命令
};
```

---

## 命令注册机制

### 注册流程

```
┌──────────────────────────────────────────────────────────────┐
│ 1. 服务器启动                                                 │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. AddCommandsScripts() 被调用                                │
│    (定义在 cs_script_loader.cpp)                             │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. 各 AddSC_xxx_commandscript() 创建命令脚本实例              │
│    new gm_commandscript();                                   │
│    new npc_commandscript();                                  │
│    ...                                                       │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. 构造函数自动注册到 ScriptRegistry<CommandScript>           │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. ChatCommandNode::LoadCommandMap() 被调用                   │
│    - 从 ScriptMgr 获取所有命令                                │
│    - 构建命令树                                               │
│    - 从数据库加载权限覆盖和帮助文本                           │
└──────────────────────────────────────────────────────────────┘
```

### 命令脚本加载器

**文件**: `src/server/scripts/Commands/cs_script_loader.cpp`

```cpp
// 声明所有命令脚本的注册函数
void AddSC_account_commandscript();
void AddSC_achievement_commandscript();
void AddSC_gm_commandscript();
// ... 更多声明

// 统一注册函数
void AddCommandsScripts()
{
    AddSC_account_commandscript();
    AddSC_achievement_commandscript();
    AddSC_gm_commandscript();
    // ... 调用所有注册函数
}
```

### 命令表定义

每个命令脚本通过 `GetCommands()` 返回命令表：

```cpp
// 使用 using 声明简化代码
using namespace Acore::ChatCommands;

class gm_commandscript : public CommandScript
{
public:
    gm_commandscript() : CommandScript("gm_commandscript") { }

    ChatCommandTable GetCommands() const override
    {
        // 定义子命令表
        static ChatCommandTable gmCommandTable =
        {
            // 格式: { "子命令名", 处理函数, 权限级别, 是否允许控制台 }
            { "chat",      HandleGMChatCommand,       SEC_GAMEMASTER,     Console::No  },
            { "fly",       HandleGMFlyCommand,        SEC_GAMEMASTER,     Console::No  },
            { "ingame",    HandleGMListIngameCommand, SEC_PLAYER,         Console::Yes },
            { "list",      HandleGMListFullCommand,   SEC_ADMINISTRATOR,  Console::Yes },
            { "visible",   HandleGMVisibleCommand,    SEC_GAMEMASTER,     Console::No  },
            { "on",        HandleGMOnCommand,         SEC_MODERATOR,      Console::No  },
            { "off",       HandleGMOffCommand,        SEC_MODERATOR,      Console::No  },
            { "spectator", HandleGMSpectatorCommand,  SEC_GAMEMASTER,     Console::No  },
        };

        // 定义顶层命令表
        static ChatCommandTable commandTable =
        {
            { "gm", gmCommandTable }  // "gm" 命令包含上述子命令
        };

        return commandTable;
    }

    // 命令处理函数...
};
```

### ChatCommandBuilder 结构

`ChatCommandBuilder` 是构建命令的核心结构：

```cpp
struct ChatCommandBuilder
{
    // 带处理函数的命令
    template <typename TypedHandler>
    ChatCommandBuilder(char const* name, 
                       TypedHandler& handler, 
                       AcoreStrings help, 
                       uint32 securityLevel, 
                       Console allowConsole);

    // 只有子命令的命令（容器命令）
    ChatCommandBuilder(char const* name, 
                       std::vector<ChatCommandBuilder> const& subCommands);
};
```

---

## 命令解析流程

### 命令执行流程图

```
用户输入: ".gm on"
         │
         ▼
┌─────────────────────────────────────┐
│ 1. ChatHandler::ParseCommands()     │
│    检查前缀，去掉 "."               │
│    剩余: "gm on"                    │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 2. TryExecuteCommand("gm on")       │
│    在命令树中查找                   │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 3. 分词处理                         │
│    token1 = "gm"                    │
│    token2 = "on"                    │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 4. 遍历命令树                       │
│    COMMAND_MAP["gm"] → ChatCommandNode │
│    node._subCommands["on"] → 子节点 │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 5. 权限检查                         │
│    IsInvokerVisible() 检查:         │
│    - 是否允许控制台执行             │
│    - 账户权限是否足够               │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 6. 执行命令处理函数                 │
│    _invoker(&handler, "")           │
│    调用 HandleGMOnCommand()         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ 7. 记录日志                         │
│    LogCommandUsage() 记录 GM 操作   │
└─────────────────────────────────────┘
```

### TryExecuteCommand 核心代码

```cpp
bool ChatCommandNode::TryExecuteCommand(ChatHandler& handler, std::string_view cmdStr)
{
    ChatCommandNode const* cmd = nullptr;
    ChatSubCommandMap const* map = &GetTopLevelMap();

    // 去除首尾分隔符
    while (!cmdStr.empty() && (cmdStr.front() == COMMAND_DELIMITER))
        cmdStr.remove_prefix(1);
    while (!cmdStr.empty() && (cmdStr.back() == COMMAND_DELIMITER))
        cmdStr.remove_suffix(1);

    std::string_view oldTail = cmdStr;
    
    // 逐级查找命令
    while (!oldTail.empty())
    {
        auto [token, newTail] = tokenize(oldTail);
        ASSERT(!token.empty());

        // 查找匹配的命令
        FilteredCommandListIterator it1(*map, handler, token);
        if (!it1)
            break;  // 没有找到匹配的子命令

        // 检查是否唯一匹配
        if (!StringEqualI(it1->first, token))
        {
            auto it2 = it1;
            ++it2;
            if (it2)
            {
                // 多个匹配，显示所有可能性
                handler.PSendSysMessage(LANG_CMD_AMBIGUOUS, token);
                // ... 显示匹配列表
                return true;
            }
        }

        // 进入下一级
        cmd = &it1->second;
        map = &cmd->_subCommands;
        oldTail = newTail;
    }

    // 执行命令
    if (cmd)
    {
        handler.SetSentErrorMessage(false);
        if (cmd->IsInvokerVisible(handler) && cmd->_invoker(&handler, oldTail))
        {
            // 执行成功，记录日志
            if (!handler.IsConsole())
                LogCommandUsage(*handler.GetSession(), cmdStr);
        }
        else if (!handler.HasSentErrorMessage())
        {
            // 执行失败，显示帮助
            cmd->SendCommandHelp(handler);
            handler.SetSentErrorMessage(true);
        }
        return true;
    }

    return false;
}
```

---

## 权限控制系统

### 权限级别定义

**文件**: `src/common/Common.h`

```cpp
enum AccountTypes
{
    SEC_PLAYER         = 0,  // 普通玩家
    SEC_MODERATOR      = 1,  // 协调员
    SEC_GAMEMASTER     = 2,  // 游戏管理员
    SEC_ADMINISTRATOR  = 3,  // 管理员
    SEC_CONSOLE        = 4   // 控制台（最高权限）
};
```

### 权限检查流程

```cpp
bool ChatCommandNode::IsInvokerVisible(ChatHandler const& who) const
{
    if (!_invoker)
        return false;

    // 脚本钩子（可被模块覆盖）
    if (!sScriptMgr->OnBeforeIsInvokerVisible(_name, _permission, who))
        return true;

    // 控制台检查
    if (who.IsConsole() && (_permission.AllowConsole == Console::No))
        return false;

    if (who.IsConsole() && (_permission.AllowConsole == Console::Yes))
        return true;

    // 游戏内权限检查
    return !who.IsConsole() && who.IsAvailable(_permission.RequiredLevel);
}
```

### 权限配置来源

权限可以从两个来源配置：

1. **代码定义**（默认值）

```cpp
{ "on", HandleGMOnCommand, SEC_MODERATOR, Console::No }
```

2. **数据库覆盖**（`command` 表）

```sql
-- 查看命令权限配置
SELECT name, security, help FROM command;

-- 修改命令权限
UPDATE command SET security = 3 WHERE name = 'gm on';
```

### 数据库权限加载

```cpp
void ChatCommandNode::LoadCommandMap()
{
    // ... 加载代码定义的命令

    // 从数据库加载权限覆盖
    if (PreparedQueryResult result = WorldDatabase.Query(
        WorldDatabase.GetPreparedStatement(WORLD_SEL_COMMANDS)))
    {
        do
        {
            Field* fields = result->Fetch();
            std::string_view name = fields[0].Get<std::string_view>();
            std::string_view help = fields[2].Get<std::string_view>();
            uint32 secLevel = fields[1].Get<uint8>();

            // 查找命令节点
            // 覆盖权限级别
            if (cmd->_invoker && (cmd->_permission.RequiredLevel != secLevel))
            {
                LOG_WARN("sql.sql", 
                    "Table `command` has permission {} for '{}' which does not match the core ({}). Overriding.",
                    secLevel, name, cmd->_permission.RequiredLevel);
                cmd->_permission.RequiredLevel = secLevel;
            }

            // 设置帮助文本
            if (std::holds_alternative<std::monostate>(cmd->_help))
                cmd->_help.emplace<std::string>(help);

        } while (result->NextRow());
    }
}
```

---

## 参数解析系统

AzerothCore 使用模板元编程实现类型安全的参数解析。

### 参数解析器架构

```cpp
// 参数信息模板（需要在 ArgInfo 中特化）
template <typename T, typename = void>
struct ArgInfo 
{ 
    static ChatCommandResult TryConsume(T& val, ChatHandler const* handler, std::string_view args);
};
```

### 支持的参数类型

#### 1. 基本数值类型

```cpp
template <typename T>
struct ArgInfo<T, std::enable_if_t<std::is_integral_v<T> || std::is_floating_point_v<T>>>
{
    static ChatCommandResult TryConsume(T& val, ChatHandler const* handler, std::string_view args)
    {
        auto [token, tail] = tokenize(args);
        if (token.empty())
            return std::nullopt;

        // 字符串转数值
        if (Optional<T> v = StringTo<T>(token, 0))
            val = *v;
        else
            return FormatAcoreString(handler, LANG_CMDPARSER_STRING_VALUE_INVALID, 
                                     token, GetTypeName<T>());

        return tail;
    }
};
```

#### 2. 字符串类型

```cpp
template <>
struct ArgInfo<std::string, void>
{
    static ChatCommandResult TryConsume(std::string& val, ChatHandler const* handler, 
                                        std::string_view args)
    {
        std::string_view view;
        ChatCommandResult next = ArgInfo<std::string_view>::TryConsume(view, handler, args);
        if (next)
            val.assign(view);
        return next;
    }
};
```

#### 3. 枚举类型

```cpp
template <typename T>
struct ArgInfo<T, std::enable_if_t<std::is_enum_v<T>>>
{
    static ChatCommandResult TryConsume(T& val, ChatHandler const* handler, std::string_view args)
    {
        std::string_view strVal;
        ChatCommandResult next = ArgInfo<std::string_view>::TryConsume(strVal, handler, args);
        
        if (next)
        {
            // 尝试按名称匹配
            if (T const* match = Match(strVal))
            {
                val = *match;
                return next;
            }
        }

        // 尝试按数值匹配
        using U = std::underlying_type_t<T>;
        U uVal = 0;
        if (ChatCommandResult next2 = ArgInfo<U>::TryConsume(uVal, handler, args))
        {
            if (EnumUtils::IsValid<T>(uVal))
            {
                val = static_cast<T>(uVal);
                return next2;
            }
        }
        // ...
    }
};
```

#### 4. Optional 类型

```cpp
template <typename Tuple, typename NestedNextType, std::size_t offset>
struct MultiConsumer<Tuple, Optional<NestedNextType>, offset>
{
    static ChatCommandResult TryConsumeTo(Tuple& tuple, ChatHandler const* handler, 
                                          std::string_view args)
    {
        auto& myArg = std::get<offset>(tuple);
        myArg.emplace();

        // 尝试解析参数
        ChatCommandResult result1 = ArgInfo<NestedNextType>::TryConsume(
            myArg.value(), handler, args);
        if (result1)
            if ((result1 = ConsumeFromOffset<Tuple, offset + 1>(tuple, handler, *result1)))
                return result1;

        // 尝试省略参数
        myArg = std::nullopt;
        ChatCommandResult result2 = ConsumeFromOffset<Tuple, offset + 1>(tuple, handler, args);
        // ...
    }
};
```

### 命令处理函数签名示例

```cpp
// 无参数
static bool HandleGMOnCommand(ChatHandler* handler);

// 必需参数
static bool HandleModifyHPCommand(ChatHandler* handler, int32 healthPoints);

// 可选参数
static bool HandleGMFlyCommand(ChatHandler* handler, Optional<bool> enable);

// 多参数
static bool HandleGoXYZCommand(ChatHandler* handler, float x, float y, 
                               Optional<float> z, Optional<uint32> mapId);

// 链接参数（从游戏内链接提取）
static bool HandleNpcAddCommand(ChatHandler* handler, 
                                Variant<Hyperlink<creature_entry>, uint32> id);

// 枚举参数
static bool HandleModifyGenderCommand(ChatHandler* handler, Gender gender);
```

---

## 实战示例分析

### 示例 1: `.gm on` 命令

```cpp
static bool HandleGMOnCommand(ChatHandler* handler)
{
    // 1. 获取玩家对象
    handler->GetPlayer()->SetGameMaster(true);
    
    // 2. 更新可见性
    handler->GetPlayer()->UpdateTriggerVisibility();
    
    // 3. 发送通知
    handler->SendNotification(LANG_GM_ON);
    
    return true;
}
```

**执行流程**:
1. 玩家输入 `.gm on`
2. 系统解析出命令路径 `gm` → `on`
3. 检查权限 `SEC_MODERATOR`
4. 调用 `HandleGMOnCommand`
5. 设置玩家的 GM 标志
6. 发送成功消息

### 示例 2: `.modify hp 1000` 命令

```cpp
static bool HandleModifyHPCommand(ChatHandler* handler, int32 healthPoints)
{
    // 1. 获取目标玩家
    Player* target = handler->getSelectedPlayer();

    // 2. 参数验证
    if (!CheckModifyInt32(handler, target, healthPoints))
        return false;

    // 3. 发送消息给执行者
    handler->PSendSysMessage(LANG_YOU_CHANGE_HP, 
                             handler->GetNameLink(target), 
                             healthPoints, healthPoints);

    // 4. 发送消息给目标
    if (handler->needReportToTarget(target))
    {
        ChatHandler(target->GetSession()).PSendSysMessage(
            LANG_YOURS_HP_CHANGED, handler->GetNameLink(), 
            healthPoints, healthPoints);
    }

    // 5. 执行修改
    target->SetMaxHealth(healthPoints);
    target->SetHealth(healthPoints);

    return true;
}
```

### 示例 3: `.npc add 12345` 命令

```cpp
// 命令定义
static ChatCommandTable npcAddCommandTable =
{
    { "formation", HandleNpcAddFormationCommand, SEC_ADMINISTRATOR, Console::No },
    { "item",      HandleNpcAddVendorItemCommand, SEC_ADMINISTRATOR, Console::No },
    { "move",      HandleNpcAddMoveCommand,       SEC_ADMINISTRATOR, Console::No },
    { "temp",      HandleNpcAddTempSpawnCommand,  SEC_ADMINISTRATOR, Console::No },
    { "",          HandleNpcAddCommand,           SEC_ADMINISTRATOR, Console::No }  // 默认
};

// 处理函数
static bool HandleNpcAddCommand(ChatHandler* handler, 
                                Variant<Hyperlink<creature_entry>, uint32> id)
{
    // id 可以是:
    // 1. 游戏内链接点击（Hyperlink<creature_entry>）
    // 2. 直接输入的数字（uint32）

    uint32 creatureId = 0;
    if (std::holds_alternative<uint32>(id))
        creatureId = std::get<uint32>(id);
    else
        creatureId = std::get<Hyperlink<creature_entry>>(id);

    // 创建 NPC...
}
```

### 示例 4: 带子命令的命令组

```cpp
ChatCommandTable GetCommands() const override
{
    static ChatCommandTable modifyCommandTable =
    {
        { "hp",         HandleModifyHPCommand,      SEC_GAMEMASTER, Console::No },
        { "mana",       HandleModifyManaCommand,    SEC_GAMEMASTER, Console::No },
        { "money",      HandleModifyMoneyCommand,   SEC_GAMEMASTER, Console::No },
        { "speed",      modifyspeedCommandTable },  // 嵌套子命令表
    };

    static ChatCommandTable modifyspeedCommandTable =
    {
        { "fly",    HandleModifyFlyCommand,  SEC_GAMEMASTER, Console::No },
        { "walk",   HandleModifySpeedCommand, SEC_GAMEMASTER, Console::No },
        { "swim",   HandleModifySwimCommand,  SEC_GAMEMASTER, Console::No },
    };

    static ChatCommandTable commandTable =
    {
        { "modify", modifyCommandTable }
    };

    return commandTable;
}
```

**命令结构**:
```
.modify hp <值>
.modify mana <值>
.modify money <值>
.modify speed fly <值>
.modify speed walk <值>
.modify speed swim <值>
```

---

## 如何添加新命令

### 步骤 1: 创建命令脚本文件

在 `src/server/scripts/Commands/` 目录下创建新文件，如 `cs_mycommand.cpp`：

```cpp
/*
 * This file is part of the AzerothCore Project. See AUTHORS file for Copyright information
 * ... 许可证头 ...
 */

#include "Chat.h"
#include "CommandScript.h"
#include "Language.h"
#include "Player.h"

using namespace Acore::ChatCommands;

class mycommand_commandscript : public CommandScript
{
public:
    mycommand_commandscript() : CommandScript("mycommand_commandscript") { }

    ChatCommandTable GetCommands() const override
    {
        static ChatCommandTable mycommandCommandTable =
        {
            { "hello",   HandleMyCommandHello,   SEC_PLAYER,       Console::No },
            { "info",    HandleMyCommandInfo,    SEC_GAMEMASTER,   Console::Yes },
            { "set",     HandleMyCommandSet,     SEC_ADMINISTRATOR, Console::No },
        };

        static ChatCommandTable commandTable =
        {
            { "mycommand", mycommandCommandTable }
        };

        return commandTable;
    }

    static bool HandleMyCommandHello(ChatHandler* handler, Optional<std::string> name)
    {
        if (name)
            handler->PSendSysMessage("Hello, {}!", *name);
        else
            handler->SendSysMessage("Hello, World!");

        return true;
    }

    static bool HandleMyCommandInfo(ChatHandler* handler)
    {
        Player* player = handler->GetSession()->GetPlayer();
        handler->PSendSysMessage("Player: {}, Level: {}, Map: {}",
            player->GetName(), player->GetLevel(), player->GetMapId());
        return true;
    }

    static bool HandleMyCommandSet(ChatHandler* handler, uint32 value, Optional<bool> announce)
    {
        // 执行设置操作...
        handler->PSendSysMessage("Value set to: {}", value);

        if (announce.value_or(false))
            handler->SendGlobalSysMessage("A new value has been set!");

        return true;
    }
};

void AddSC_mycommand_commandscript()
{
    new mycommand_commandscript();
}
```

### 步骤 2: 注册脚本

编辑 `src/server/scripts/Commands/cs_script_loader.cpp`：

```cpp
// 添加声明
void AddSC_mycommand_commandscript();

// 在 AddCommandsScripts() 中添加调用
void AddCommandsScripts()
{
    // ... 其他脚本
    AddSC_mycommand_commandscript();
}
```

### 步骤 3: 添加语言字符串（可选）

编辑语言文件或数据库 `acore_string` 表：

```sql
INSERT INTO acore_string (entry, content_default) 
VALUES (15000, 'My custom message: %s');
```

在代码中使用：

```cpp
handler->SendNotification(15000, "test");
```

### 步骤 4: 添加数据库帮助文本（可选）

```sql
INSERT INTO command (name, security, help) 
VALUES 
('mycommand', 0, 'Syntax: .mycommand <subcommand>\nMy custom command group.'),
('mycommand hello', 0, 'Syntax: .mycommand hello [name]\nSay hello!'),
('mycommand info', 2, 'Syntax: .mycommand info\nShow player info.'),
('mycommand set', 3, 'Syntax: .mycommand set <value> [announce]\nSet a value.');
```

### 步骤 5: 编译测试

```bash
# 重新编译
cd build
make -j$(nproc)

# 重启服务器测试
```

---

## 常见问题与调试

### 问题 1: 命令不显示或无法执行

**可能原因**:
- 权限不足
- 命令未正确注册
- 控制台限制

**调试方法**:

```cpp
// 在命令处理函数开头添加日志
static bool HandleMyCommand(ChatHandler* handler)
{
    LOG_INFO("module.mycommand", "MyCommand executed by {}", 
             handler->GetSession()->GetPlayer()->GetName());
    // ...
}
```

### 问题 2: 参数解析失败

**检查参数类型是否支持**:

```cpp
// 查看支持的类型
static_assert(std::is_integral_v<int32>);  // OK
static_assert(std::is_enum_v<MyEnum>);     // 需要特化 ArgInfo
```

**自定义类型支持**:

```cpp
// 为自定义类型特化 ArgInfo
template <>
struct ArgInfo<MyCustomType, void>
{
    static ChatCommandResult TryConsume(MyCustomType& val, ChatHandler const* handler, 
                                        std::string_view args)
    {
        std::string str;
        ChatCommandResult next = ArgInfo<std::string>::TryConsume(str, handler, args);
        if (next)
        {
            if (ParseMyCustomType(str, val))
                return next;
            return "Invalid custom type format";
        }
        return std::nullopt;
    }
};
```

### 问题 3: 命令权限覆盖不生效

**检查数据库表**:

```sql
-- 查看命令配置
SELECT * FROM command WHERE name LIKE 'gm%';

-- 确保命令名称正确（包含完整路径）
UPDATE command SET security = 3 WHERE name = 'gm on';
```

### 问题 4: 控制台命令无法执行

**检查 Console 设置**:

```cpp
// Console::Yes 允许控制台执行
{ "list", HandleGMListCommand, SEC_ADMINISTRATOR, Console::Yes },

// Console::No 禁止控制台执行
{ "on", HandleGMOnCommand, SEC_MODERATOR, Console::No },
```

### 调试技巧

1. **启用命令日志**:

```cpp
// 在 worldserver.conf 中
LogFilterGMCommands = 0  // 记录所有 GM 命令
```

2. **使用 `.help` 命令**:

```
.help gm          # 查看 gm 命令帮助
.help gm on       # 查看 gm on 的详细帮助
```

3. **检查命令是否加载**:

```cpp
// 在 LoadCommandMap() 后添加
for (auto& [name, cmd] : COMMAND_MAP)
{
    LOG_INFO("commands", "Loaded command: {}", name);
}
```

---

## 附录

### A. 所有命令文件列表

| 文件 | 功能 |
|------|------|
| cs_account.cpp | 账户管理 |
| cs_achievement.cpp | 成就管理 |
| cs_arena.cpp | 竞技场管理 |
| cs_autobroadcast.cpp | 自动广播 |
| cs_bag.cpp | 背包管理 |
| cs_ban.cpp | 封禁管理 |
| cs_bf.cpp | 战场管理 |
| cs_cache.cpp | 缓存操作 |
| cs_cast.cpp | 施法相关 |
| cs_character.cpp | 角色管理 |
| cs_cheat.cpp | 作弊命令 |
| cs_debug.cpp | 调试工具 |
| cs_deserter.cpp | 逃兵系统 |
| cs_disable.cpp | 禁用功能 |
| cs_event.cpp | 事件管理 |
| cs_gear.cpp | 装备相关 |
| cs_gm.cpp | GM 模式 |
| cs_go.cpp | 传送命令 |
| cs_gobject.cpp | 游戏对象 |
| cs_group.cpp | 组队管理 |
| cs_guild.cpp | 公会管理 |
| cs_honor.cpp | 荣誉系统 |
| cs_instance.cpp | 副本管理 |
| cs_inventory.cpp | 物品栏 |
| cs_item.cpp | 物品管理 |
| cs_learn.cpp | 学习技能 |
| cs_lfg.cpp | 随机副本 |
| cs_list.cpp | 列表查询 |
| cs_lookup.cpp | 查找功能 |
| cs_mail.cpp | 邮件系统 |
| cs_message.cpp | 消息广播 |
| cs_misc.cpp | 杂项命令 |
| cs_mmaps.cpp | 寻路系统 |
| cs_modify.cpp | 属性修改 |
| cs_npc.cpp | NPC 管理 |
| cs_pet.cpp | 宠物管理 |
| cs_player.cpp | 玩家命令 |
| cs_player_settings.cpp | 玩家设置 |
| cs_pooltools.cpp | 池工具 |
| cs_quest.cpp | 任务管理 |
| cs_reload.cpp | 重载配置 |
| cs_reset.cpp | 重置功能 |
| cs_script_loader.cpp | 脚本加载 |
| cs_send.cpp | 发送消息 |
| cs_server.cpp | 服务器管理 |
| cs_spectator.cpp | 观战模式 |
| cs_spellinfo.cpp | 法术信息 |
| cs_tele.cpp | 传送点 |
| cs_ticket.cpp | 工单系统 |
| cs_titles.cpp | 称号管理 |
| cs_worldstate.cpp | 世界状态 |
| cs_wp.cpp | 路径点 |

### B. 权限级别对照表

| 级别 | 名称 | 说明 |
|------|------|------|
| 0 | SEC_PLAYER | 普通玩家，可执行基本命令 |
| 1 | SEC_MODERATOR | 协调员，可执行基础管理 |
| 2 | SEC_GAMEMASTER | GM，可执行大部分管理命令 |
| 3 | SEC_ADMINISTRATOR | 管理员，可执行所有命令 |
| 4 | SEC_CONSOLE | 控制台，最高权限 |

### C. 常用消息函数

| 函数 | 用途 |
|------|------|
| SendSysMessage() | 发送系统消息 |
| SendNotification() | 发送通知（屏幕中央） |
| SendErrorMessage() | 发送错误消息 |
| PSendSysMessage() | 格式化发送系统消息 |
| SendGlobalGMSysMessage() | 发送给所有 GM |
| SendGlobalSysMessage() | 发送给所有人 |

### D. 参考链接

- [AzerothCore Wiki](https://www.azerothcore.org/wiki/)
- [AzerothCore GitHub](https://github.com/azerothcore/azerothcore)
- [TrinityCore 命令文档](https://trinitycore.info/en/home)

---

*文档版本: 1.0*
*最后更新: 2024*
*适用于: AzerothCore WotLK*
