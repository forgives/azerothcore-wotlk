# AzerothCore Warden 反作弊模块详细分析

## 1. 模块概览

Warden 是暴雪的反作弊系统，AzerothCore 实现了其服务端组件。模块负责向客户端下发反作弊检查指令、验证返回结果、并在检测到作弊时执行惩罚（日志/踢出/封禁）。同时提供了 `WardenPayloadMgr` 子系统，允许通过 Warden 通道向客户端发送自定义 Lua 脚本，实现无补丁服务端定制。

**文件结构：**

| 文件 | 职责 |
|------|------|
| `Warden.h` / `Warden.cpp` | 基类，提供加密/解密、模块传输、惩罚执行、Session 处理分发 |
| `WardenWin.h` / `WardenWin.cpp` | Windows 客户端实现（完整功能，含检查构建与结果验证） |
| `WardenMac.h` / `WardenMac.cpp` | macOS 客户端实现（功能受限，当前已禁用） |
| `WardenCheckMgr.h` / `WardenCheckMgr.cpp` | 全局单例，管理反作弊检查定义的加载与查询 |
| `WardenPayloadMgr.h` / `WardenPayloadMgr.cpp` | 自定义 Lua Payload 管理（注册/缓存/排队） |
| `enuminfo_WardenCheckMgr.cpp` | WardenActions 枚举的 EnumUtils 自动生成代码 |
| `Modules/WardenModuleWin.h` | Windows 客户端 Warden 模块二进制数据（18756 字节） |
| `Modules/WardenModuleMac.h` | macOS 客户端 Warden 模块二进制数据（9318 字节） |

**模块位置：** `src/server/game/Warden/`

---

## 2. 类层次结构

```
Warden (抽象基类)
├── WardenWin (Windows 客户端，完整实现)
└── WardenMac (macOS 客户端，当前已禁用)

WardenCheckMgr (全局单例 sWardenCheckMgr)
WardenPayloadMgr (每个 Warden 实例内嵌一个)
```

### 2.1 Warden 基类（`Warden.h:102-153`）

```
Warden
├── 构造/析构: Warden() / ~Warden()
│
├── 纯虚函数（平台相关）:
│   ├── Init(WorldSession*, SessionKey const&)     // 初始化密钥和模块
│   ├── GetModuleForClient()                       // 获取客户端模块
│   ├── InitializeModule()                         // 发送模块初始化命令
│   ├── RequestHash()                              // 请求哈希验证
│   ├── HandleHashResult(ByteBuffer&)               // 处理哈希结果
│   ├── IsCheckInProgress()                        // 检查是否有检查进行中
│   ├── ForceChecks()                              // 强制立即发送检查
│   ├── RequestChecks()                            // 构建并发送检查请求
│   └── HandleData(ByteBuffer&)                    // 处理检查结果
│
├── 公共方法:
│   ├── IsInitialized()                            // 是否已完成初始化
│   ├── ProcessLuaCheckResponse(msg)               // 处理 Lua 检查的 Addon 回调
│   ├── SendModuleToClient()                       // 分块发送模块到客户端
│   ├── RequestModule()                            // 请求客户端加载模块
│   ├── Update(diff)                               // 主更新循环
│   ├── DecryptData(buf, len)                      // RC4 解密
│   ├── EncryptData(buf, len)                      // RC4 加密
│   ├── IsValidCheckSum(checksum, data, len)       // 校验和验证
│   ├── BuildChecksum(data, len)                   // SHA1 基校验和构建（静态）
│   ├── ApplyPenalty(checkId, reason)              // 执行惩罚
│   └── GetPayloadMgr()                            // 获取 Payload 管理器
│
└── 私有成员:
    ├── _session: WorldSession*
    ├── _payloadMgr: WardenPayloadMgr
    ├── _inputKey[16] / _outputKey[16]              // RC4 密钥对
    ├── _seed[16]                                   // 哈希种子
    ├── _inputCrypto / _outputCrypto: ARC4           // RC4 加密器
    ├── _checkTimer: uint32                          // 下次检查倒计时（初始 10s）
    ├── _clientResponseTimer: uint32                 // 客户端响应超时计时器
    ├── _dataSent: bool                              // 是否已发送数据等待响应
    ├── _module: ClientWardenModule*                 // 客户端模块数据
    ├── _initialized: bool                           // 是否完成握手
    ├── _interrupted: bool                           // 是否被 ForceChecks 打断
    ├── _checkInProgress: bool                       // 检查是否进行中
    └── _interruptCounter: uint32                    // 打断嵌套计数
```

### 2.2 WardenWin（`WardenWin.h:70-92`）

在基类之上增加的成员：

| 成员 | 类型 | 用途 |
|------|------|------|
| `_serverTicks` | `uint32` | 请求检查时的服务器时间戳（用于 TIMING_CHECK） |
| `_ChecksTodo[MAX_WARDEN_CHECK_TYPES]` | `std::list<uint16>[3]` | 各类型待执行检查 ID 池 |
| `_CurrentChecks` | `std::list<uint16>` | 当前批次要发送的检查 ID |
| `_PendingChecks` | `std::list<uint16>` | 因包大小限制延后到下一批的检查 ID |

### 2.3 WardenCheckMgr（`WardenCheckMgr.h:63-87`）

全局单例（`#define sWardenCheckMgr WardenCheckMgr::instance()`），负责从数据库加载和管理所有检查定义。

```
WardenCheckMgr
├── CheckStore: vector<WardenCheck>               // 按 ID 线性索引的检查定义
├── CheckResultStore: map<uint32, WardenCheckResult> // MEM_CHECK / MPQ_CHECK 的预期结果
├── CheckIdPool[3]: vector<uint16>                // 各类型检查 ID 池（用于轮询）
│   ├── [WARDEN_CHECK_MEM_TYPE]    MEM_CHECK, MODULE_CHECK
│   ├── [WARDEN_CHECK_LUA_TYPE]    LUA_EVAL_CHECK
│   └── [WARDEN_CHECK_OTHER_TYPE]  PAGE_CHECK, DRIVER_CHECK 等
├── LoadWardenChecks()                          // 从 warden_checks 表加载
├── LoadWardenOverrides()                       // 从 warden_action 表加载惩罚覆盖
├── GetWardenDataById(id)                       // 按 ID 获取检查定义
├── GetWardenResultById(id)                     // 按 ID 获取预期结果
└── GetMaxValidCheckId()                        // 返回 CheckStore.size()
```

### 2.4 WardenPayloadMgr（`WardenPayloadMgr.h:35-136`）

每个 Warden 实例内嵌一个，管理自定义 Lua Payload 的生命周期。

```
WardenPayloadMgr
├── CachedChecks: map<uint16, WardenCheck>       // 注册的 Payload 缓存（ID → 检查定义）
├── QueuedPayloads: list<uint16>                 // 等待发送的 Payload ID 队列
├── RegisterPayload(payload) → uint16            // 自动分配 ID 注册
├── RegisterPayload(payload, id, replace) → bool // 指定 ID 注册
├── UnregisterPayload(id) → bool                 // 注销
├── GetPayloadById(id) → WardenCheck*            // 查询
├── QueuePayload(id, pushToFront)                // 加入发送队列
├── DequeuePayload(id) → bool                    // 从队列移除
└── 常量:
    ├── WardenPayloadOffsetMin = 5000            // 自定义 ID 最小值
    ├── WardenPayloadOffsetMax = 9999            // 自定义 ID 最大值
    └── WardenPayloadCheckType = 139             // 使用 LUA_EVAL_CHECK 类型
```

---

## 3. 核心数据结构

### 3.1 WardenCheck（`WardenCheckMgr.h:43-54`）

每条反作弊检查的完整定义：

```cpp
struct WardenCheck
{
    uint8 Type;                  // WardenCheckType 枚举值
    BigNumber Data;              // 二进制数据（PAGE_CHECK/DRIVER_CHECK 的哈希种子）
    uint32 Address;              // 内存地址（MEM_CHECK/PAGE_CHECK/PROC_CHECK）
    uint8 Length;                // 读取长度（MEM_CHECK/PAGE_CHECK/PROC_CHECK）
    std::string Str;             // 字符串（LUA 代码/MPQ 文件名/驱动名/模块名）
    std::string Comment;         // 人类可读描述
    uint16 CheckId;              // 检查 ID（对应数据库 id）
    std::array<char, 4> IdStr;   // 4 字符的 ID 字符串（LUA_EVAL_CHECK 专用）
    uint32 Action;               // 惩罚动作（WARDEN_ACTION_LOG/KICK/BAN）
};
```

### 3.2 WardenCheckResult（`WardenCheckMgr.h:58-61`）

```cpp
struct WardenCheckResult
{
    BigNumber Result;            // MEM_CHECK/MPQ_CHECK 的预期内存/哈希值
};
```

### 3.3 ClientWardenModule（`Warden.h:92-98`）

```cpp
struct ClientWardenModule
{
    std::array<uint8, 16> Id{};            // MD5 模块指纹
    std::array<uint8, 16> Key{};           // 模块密钥
    uint32 CompressedSize{};               // 压缩数据大小
    uint8* CompressedData{};               // 压缩模块二进制
};
```

### 3.4 网络协议结构体（`Warden.h:65-84`，1 字节对齐）

```cpp
struct WardenModuleUse        // 请求客户端加载模块
{
    uint8 Command;             // WARDEN_SMSG_MODULE_USE (0)
    uint8 ModuleId[16];        // MD5 模块指纹
    uint8 ModuleKey[16];       // 模块密钥
    uint32 Size;               // 模块压缩大小（大端序）
};  // 共 37 字节

struct WardenModuleTransfer   // 分块传输模块数据
{
    uint8 Command;             // WARDEN_SMSG_MODULE_CACHE (1)
    uint16 DataSize;           // 本块大小（最大 500）
    uint8 Data[500];           // 数据内容
};  // 共 503 字节

struct WardenHashRequest      // 请求哈希验证
{
    uint8 Command;             // WARDEN_SMSG_HASH_REQUEST (5)
    uint8 Seed[16];            // 随机种子
};  // 共 17 字节
```

### 3.5 WardenInitModuleRequest（`WardenWin.h:31-59`）

Windows 客户端模块初始化命令（3 个子命令合并在一个包中）：

```cpp
struct WardenInitModuleRequest
{
    // 子命令 1: 注册 4 个文件操作函数
    uint8 Command1;                    // WARDEN_SMSG_MODULE_INITIALIZE (3)
    uint16 Size1;                      // 20
    uint32 CheckSumm1;
    uint8 Unk1, Unk2;
    uint8 Type;                        // 1
    uint8 String_library1;
    uint32 Function1[4];               // SFileOpenFile, GetFileSize, ReadFile, CloseFile

    // 子命令 2: 注册 Lua 执行函数
    uint8 Command2;
    uint16 Size2;                      // 8
    uint32 CheckSumm2;
    uint8 Unk3, Unk4;
    uint8 String_library2;
    uint32 Function2;                  // FrameScript::Execute (0x00419210)
    uint8 Function2_set;

    // 子命令 3: 注册计时器函数
    uint8 Command3;
    uint16 Size3;                      // 8
    uint32 CheckSumm3;
    uint8 Unk5, Unk6;
    uint8 String_library3;
    uint32 Function3;                  // PerformanceCounter (0x0046AE20)
    uint8 Function3_set;
};
```

---

## 4. 协议定义

### 4.1 Warden 内部操作码（`Warden.h:27-44`）

所有 Warden 数据通过 `CMSG_WARDEN_DATA` / `SMSG_WARDEN_DATA` 传输，内部第一个字节为 Warden 操作码。

**客户端 → 服务端：**

| 操作码 | 值 | 含义 |
|--------|---|------|
| `WARDEN_CMSG_MODULE_MISSING` | 0 | 客户端没有缓存该模块，请求传输 |
| `WARDEN_CMSG_MODULE_OK` | 1 | 模块加载成功 |
| `WARDEN_CMSG_CHEAT_CHECKS_RESULT` | 2 | 反作弊检查结果 |
| `WARDEN_CMSG_MEM_CHECKS_RESULT` | 3 | 内存检查结果（不匹配时） |
| `WARDEN_CMSG_HASH_RESULT` | 4 | 哈希验证结果 |
| `WARDEN_CMSG_MODULE_FAILED` | 5 | 模块加载失败（缓存问题） |

**服务端 → 客户端：**

| 操作码 | 值 | 含义 |
|--------|---|------|
| `WARDEN_SMSG_MODULE_USE` | 0 | 请求客户端加载指定模块 |
| `WARDEN_SMSG_MODULE_CACHE` | 1 | 传输模块数据（分块，每块最大 500 字节） |
| `WARDEN_SMSG_CHEAT_CHECKS_REQUEST` | 2 | 反作弊检查请求 |
| `WARDEN_SMSG_MODULE_INITIALIZE` | 3 | 模块初始化命令 |
| `WARDEN_SMSG_MEM_CHECKS_REQUEST` | 4 | 内存检查请求 |
| `WARDEN_SMSG_HASH_REQUEST` | 5 | 哈希验证请求 |

### 4.2 世界服务器操作码

| 操作码 | 值 | 方向 | 处理 |
|--------|---|------|------|
| `SMSG_WARDEN_DATA` | 0x2E6 | S→C | 服务端→客户端 Warden 封包 |
| `CMSG_WARDEN_DATA` | 0x2E7 | C→S | `WorldSession::HandleWardenDataOpcode`，**绕过 opcode 队列**直接处理 |

> **重要：** `CMSG_WARDEN_DATA` 在 `WorldSocket.cpp:489` 处被特殊处理，绕过了常规的 opcode 队列机制（`packetToQueue->GetOpcode() != CMSG_WARDEN_DATA`），确保 Warden 响应包不被延迟。

---

## 5. 检查类型详解（`Warden.h:46-57`）

### 5.1 WardenCheckType 枚举

| 类型 | 值 | 数据载荷 | 用途 |
|------|----|---------|------|
| `MEM_CHECK` | 0xF3 (243) | moduleNameIndex(1) + Address(4) + Len(1) | 读取指定内存地址，验证未被篡改 |
| `PAGE_CHECK_A` | 0xB2 (178) | Seed(4) + SHA1(20) + Addr(4) + Len(1) | 扫描所有内存页查找指定哈希 |
| `PAGE_CHECK_B` | 0xBF (191) | Seed(4) + SHA1(20) + Addr(4) + Len(1) | 仅扫描 MZ+PE 头的内存页 |
| `MPQ_CHECK` | 0x98 (152) | fileNameIndex(1) | 验证 MPQ 文件完整性（SHA1 哈希） |
| `LUA_EVAL_CHECK` | 139 | Lua 代码字符串 | 在客户端执行 Lua 脚本，检测外挂 |
| `DRIVER_CHECK` | 0x71 (113) | Seed(4) + SHA1(20) + driverNameIndex(1) | 检测非法内核驱动是否已加载 |
| `TIMING_CHECK` | 0x57 (87) | 空 | 检测 `GetTickCount()` 是否被 Hook |
| `PROC_CHECK` | 0x7E (126) | Seed(4) + SHA1(20) + moduleIdx + procIdx + Addr + Len | 检测函数是否被 Hook |
| `MODULE_CHECK` | 0xD9 (217) | Seed(4) + SHA1(20) | 检测非法 DLL 是否被注入 |

### 5.2 检查类型分类（`WardenCheckMgr.h:34-41`）

```cpp
enum WardenCheckTypes
{
    WARDEN_CHECK_MEM_TYPE   = 0,   // MEM_CHECK(243), MODULE_CHECK(217)
    WARDEN_CHECK_LUA_TYPE   = 1,   // LUA_EVAL_CHECK(139)
    WARDEN_CHECK_OTHER_TYPE = 2,   // PAGE_CHECK_A(178), PAGE_CHECK_B(191),
                                   // DRIVER_CHECK(113), PROC_CHECK(126)
};
```

---

## 6. 完整生命周期流程

### 6.1 系统初始化

```
World::SetInitialWorldSettings()                       // World.cpp:961-966
    ├── sWardenCheckMgr->LoadWardenChecks()            // 从 acore_world.warden_checks 加载
    └── sWardenCheckMgr->LoadWardenOverrides()         // 从 acore_characters.warden_action 加载
```

### 6.2 会话级 Warden 初始化

```
WorldSocket::HandleAuthSession()                       // WorldSocket.cpp:604-718
    │
    ├── 检查 Warden 是否启用
    ├── 若启用且 OS 不是 Win/OSX → 拒绝连接           // WorldSocket.cpp:605-611
    │
    └── WorldSession 创建后：
        └── _worldSession->InitWarden(sessionKey, OS)  // WorldSocket.cpp:717
                │
                ├── os == "Win":
                │   └── _warden = make_unique<WardenWin>()
                │       └── WardenWin::Init(session, key)
                │
                └── os == "OSX":
                    └── 已注释禁用（会导致客户端崩溃）
```

### 6.3 WardenWin::Init 密钥协商（`WardenWin.cpp:113-136`）

```
Init(session, K)
    │
    ├── 1. 密钥派生
    │   SessionKeyGenerator<SHA1> WK(K)
    │   ├── 将 SessionKey 分为两半，各自 SHA1 哈希
    │   └── Generate(_inputKey, 16)  // C→S 密钥
    │       Generate(_outputKey, 16) // S→C 密钥
    │
    ├── 2. 初始化 RC4
    │   _inputCrypto.Init(_inputKey)
    │   _outputCrypto.Init(_outputKey)
    │
    ├── 3. 设置种子
    │   memcpy(_seed, Module.Seed, 16)
    │
    ├── 4. 准备模块
    │   _module = GetModuleForClient()
    │       ├── 从 WardenModuleWin.h 读取编译时嵌入的模块二进制
    │       ├── 计算模块 MD5 作为 ID
    │       └── 使用硬编码的 ModuleKey
    │
    └── 5. 请求客户端加载模块
        RequestModule() → 发送 WardenModuleUse（加密后）
```

### 6.4 模块握手流程

```
服务端                              客户端
  │                                    │
  │── WardenModuleUse ──────────────→  │  (SMSG, 命令 0)
  │                                    │
  │   (客户端检查缓存)                   │
  │                                    │
  │←── MODULE_MISSING (0) ──────────  │  或 MODULE_OK (1)
  │                                    │
  │── WardenModuleTransfer × N ─────→  │  (SMSG, 命令 1, 每块≤500字节)
  │                                    │  (客户端解压并加载模块)
  │                                    │
  │←── MODULE_OK (1) ──────────────  │
  │                                    │
  │── WardenHashRequest ────────────→  │  (SMSG, 命令 5, 携带 Seed)
  │                                    │  (客户端用 Seed + ModuleKey 计算 SHA1)
  │                                    │
  │←── HASH_RESULT (4) ────────────  │  (20 字节 SHA1 哈希)
  │                                    │
  │   验证哈希 (constant-time compare)  │
  │   ├── 失败 → ApplyPenalty()        │
  │   └── 成功:                        │
  │       ├── 密钥切换                 │
  │       │   _inputKey  = ClientKeySeed
  │       │   _outputKey = ServerKeySeed
  │       ├── 重新初始化 RC4           │
  │       ├── _initialized = true      │
  │       └── InitializeModule()       │  → 发送函数注册命令
  │                                    │
  │   === 握手完成，进入检查循环 ===    │
```

### 6.5 密钥切换机制（`WardenWin.cpp:230-252`）

握手成功后，RC4 密钥从派生密钥切换到模块内嵌的硬编码密钥：

```
输入密钥:  Module.ClientKeySeed    (7F96EEFD...)
输出密钥:  Module.ServerKeySeed    (C2B7ADED...)
验证哈希:  Module.ClientKeySeedHash (568C054C...)

验证: CRYPTO_memcmp(clientHash, ClientKeySeedHash, 20) == 0
```

这一步确保客户端确实加载了正确的 Warden 模块（而非伪造模块），因为只有真正的模块才知道如何用正确的种子计算哈希。

---

## 7. 检查请求构建与发送（`WardenWin::RequestChecks()` — `WardenWin.cpp:277-561`）

这是 Warden 最复杂的方法，负责构建一个包含多种检查类型的加密数据包。

### 7.1 检查选取策略

```
RequestChecks()
    │
    ├── 1. 补充检查池
    │   for each checkType:
    │       if _ChecksTodo[type].empty():
    │           从 sWardenCheckMgr->CheckIdPool[type] 重新填充
    │
    ├── 2. 清理无效的 PendingChecks（移除已被删除的检查 ID）
    │
    ├── 3a. 无 PendingChecks（正常情况）：
    │   for each checkType:
    │       for i = 0..CONFIG_WARDEN_NUM_*_CHECKS:
    │           ├── 优先处理 QueuedPayloads（自定义 Lua Payload）
    │           │   └── push_front 到 _CurrentChecks
    │           ├── 从 _ChecksTodo 尾部取出检查 ID
    │           ├── LUA 类型 → push_front（Lua 检查必须在最前面）
    │           └── 其他类型 → push_back
    │
    ├── 3b. 有 PendingChecks（上一批超限延后的检查）：
    │   ├── 将 PendingChecks 加入 CurrentChecks
    │   └── 确保至少包含一些 Lua 检查
    │
    ├── 4. 包大小过滤（客户端限制 512 字节）
    │   expectedSize 从 4 开始（命令字节 + 长度 + 校验和预留）
    │   遍历 CurrentChecks:
    │       if expectedSize + checkSize > 500:
    │           移入 _PendingChecks（下一批发送）
    │
    └── 5. 构建数据包（双区结构）
```

### 7.2 数据包双区结构

Warden 检查包被组织为**两个区域**：

**区域 A — 字符串池（LUA/MPQ/DRIVER 的字符串数据）：**

```
uint8  WARDEN_SMSG_CHEAT_CHECKS_REQUEST (2)

for each check in _CurrentChecks:
    switch (check->Type):
        LUA_EVAL_CHECK:
            uint8  length (前缀+Lua代码+中缀+ID+后缀 的总长)
            bytes  _luaEvalPrefix + Lua代码 + _luaEvalMidfix + IdStr + _luaEvalPostfix
        MPQ_CHECK / DRIVER_CHECK:
            uint8  length
            bytes  字符串内容
```

**区域 B — Xor 编码的检查指令：**

```
uint8  0x00                                    // 固定分隔符
uint8  TIMING_CHECK ^ xorByte                  // XOR 编码的检查类型
uint8  index = 1                               // 字符串索引计数器

for each check in _CurrentChecks:
    uint8  check->Type ^ xorByte               // XOR 编码的检查类型

    switch (check->Type):
        MEM_CHECK:
            uint8  0x00
            uint32 Address
            uint8  Length

        PAGE_CHECK_A / PAGE_CHECK_B:
            bytes  Data (24 字节哈希种子)
            uint32 Address
            uint8  Length

        MPQ_CHECK / LUA_EVAL_CHECK:
            uint8  index++

        DRIVER_CHECK:
            bytes  Data (24 字节哈希种子)
            uint8  index++

        MODULE_CHECK:
            bytes  randomSeed (4 字节)
            bytes  HMAC_SHA1(seed, Str) (20 字节)

uint8  xorByte                                // 结束标记
```

### 7.3 Lua 检查的特殊编码

Lua 检查利用客户端 Addon 消息通道将结果回传：

```
模板:
  local S,T,R=SendAddonMessage,function()
  [Lua检查代码]                               // 执行并捕获结果
  end R=S and T()if R then S('_TW',
  [4位ID字符串],                              // 如 "0788"
  'GUILD')end

工作原理:
  1. 客户端在受限 Lua 环境中执行检查代码
  2. 如果检测到外挂特征，通过 SendAddonMessage 发送 '_TW\t[检查ID]'
  3. 服务端在 WorldSession::HandleAddonMessageOpcode 中拦截
  4. 调用 Warden::ProcessLuaCheckResponse() 解析并执行惩罚

总长度限制: sizeof(prefix)-1 + sizeof(midfix)-1 + sizeof(postfix)-1 + 170 = 255 字节
```

---

## 8. 检查结果处理（`WardenWin::HandleData()` — `WardenWin.cpp:563-772`）

### 8.1 数据包验证

```
HandleData(buff)
    │
    ├── 读取 Length(2) + Checksum(4)
    ├── 验证 Length 与缓冲区剩余大小匹配
    │   └── 不匹配 → ApplyPenalty("Failed size checks")
    ├── 验证 Checksum（SHA1-based BuildChecksum）
    │   └── 不匹配 → ApplyPenalty("Failed checksum")
    │
    └── 如果被 ForceChecks 打断 → 忽略所有结果
```

### 8.2 TIMING_CHECK 处理

```
读取 result(1) + newClientTicks(4)

if result == 0x00:
    → 可能 GetTickCount() 被 Hook（已注释，因误报率高）

计算 tick 差值:
    ticksNow = GameTime::GetGameTimeMS()
    ourTicks = newClientTicks + (ticksNow - _serverTicks)
    diff = ourTicks - newClientTicks
```

### 8.3 各类型检查结果验证

**MEM_CHECK (243)：**
```
读取 Mem_Result(1)
├── Mem_Result != 0 → 失败（内存不可读/被保护）
└── Mem_Result == 0:
    读取 bytes[Length]
    └── 与 CheckResultStore[id].Result 比较 (CRYPTO_memcmp)
        └── 不匹配 → 失败（内存被篡改）
```

**PAGE_CHECK_A (178) / PAGE_CHECK_B (191) / DRIVER_CHECK (113) / MODULE_CHECK (217)：**
```
读取 byte(1)
└── 与 0xE9 比较
    └── 不等于 0xE9 → 失败
        // 0xE9 是 x86 JMP 指令的操作码
        // 表示检测的目标地址被修改（注入了跳转指令）
```

**LUA_EVAL_CHECK (139)：**
```
读取 result(1)
├── result == 0: 正常（Lua 代码返回 nil/false）
│   └── 丢弃附加的字符串数据
└── result != 0: 由 Addon 消息通道处理（见 ProcessLuaCheckResponse）
```

**MPQ_CHECK (152)：**
```
读取 Mpq_Result(1)
├── Mpq_Result != 0 → 失败（文件打开失败）
└── Mpq_Result == 0:
    读取 SHA1[20]
    └── 与 CheckResultStore[id].Result 比较
        └── 不匹配 → 失败（MPQ 文件被篡改）
```

### 8.4 惩罚执行（`Warden::ApplyPenalty()` — `Warden.cpp:199-277`）

```
ApplyPenalty(checkId, reason)
    │
    ├── 确定 action:
    │   ├── checkId > 0 且检查存在 → action = check->Action
    │   └── 否则 → action = CONFIG_WARDEN_CLIENT_FAIL_ACTION (默认)
    │
    ├── 执行 action:
    │   ├── WARDEN_ACTION_LOG (0):
    │   │   └── 仅记录日志
    │   ├── WARDEN_ACTION_KICK (1):
    │   │   └── _session->KickPlayer(causeMsg)
    │   └── WARDEN_ACTION_BAN (2):
    │       ├── 获取账号名
    │       └── sBan->BanAccount(accountName, duration, causeMsg, "Server")
    │           // duration = CONFIG_WARDEN_CLIENT_BAN_DURATION (默认 86400 秒 = 1 天)
    │
    └── 记录详细报告日志:
        "Warden: Player {name} (guid {guid}, account id: {id}) failed warden
         {checkId} check ({comment}). Action: {action}"
```

---

## 9. 更新循环（`Warden::Update()` — `Warden.cpp:98-131`）

```
WorldSession::Update(diff)                            // WorldSession.cpp:569-582
    └── if (_warden && m_Socket->IsOpen())
            └── _warden->Update(diff)

Warden::Update(diff)
    │
    ├── !_initialized → return（握手未完成则不发送检查）
    │
    ├── _dataSent == true（等待客户端响应）:
    │   ├── _clientResponseTimer += diff
    │   └── if timer > CONFIG_WARDEN_CLIENT_RESPONSE_DELAY * 1000:
    │       └── KickPlayer("Warden: clientResponseTimer > maxClientResponseDelay")
    │           // 客户端响应超时 = 可能被修改/绕过
    │
    └── _dataSent == false（可以发送新检查）:
        ├── _checkTimer -= diff
        └── if _checkTimer <= 0:
            ├── RequestChecks()                     // 构建并发送检查
            └── HandleData() 中会设置:
                _checkTimer = CONFIG_WARDEN_CLIENT_CHECK_HOLDOFF * 1000
                // 默认 30 秒后发送下一批检查
```

---

## 10. 密码学基础设施

### 10.1 密钥派生 — SessionKeyGenerator（`SessionKeyGenerator.h`）

```
SessionKeyGenerator<SHA1>(SessionKey K)
    │
    ├── K 分为两半: left = K[0..15], right = K[16..31]
    ├── o1 = SHA1(left)
    ├── o2 = SHA1(right)
    └── o0 = SHA1(o1 || o2)
        │
        Generate(buf, sz):
        │   从 o0 digest 中逐字节流出
        │   流完后重新计算 o0 = SHA1(o1 || o2)
        └── 重复直到 sz 字节填满
```

### 10.2 RC4 加密 — ARC4（`common/Cryptography/ARC4.h`）

基于 OpenSSL EVP 接口的 RC4 流密码实现：

```cpp
class ARC4 {
    void Init(uint8 const* seed, size_t len);    // 设置密钥
    void UpdateData(uint8* data, size_t len);     // 原地加密/解密
};
```

Warden 使用两个独立的 RC4 实例：
- `_outputCrypto`：加密 S→C 数据（用 `_outputKey`）
- `_inputCrypto`：解密 C→S 数据（用 `_inputKey`）

RC4 是流密码，加密和解密使用相同的操作，但方向不同需要不同密钥。

### 10.3 校验和构建 — BuildChecksum（`Warden.cpp:170-182`）

```cpp
SHA1(data) → hash (20 bytes)
将 20 字节按 uint32 分为 5 个 dword
checksum = dword[0] ^ dword[1] ^ dword[2] ^ dword[3] ^ dword[4]
```

### 10.4 常量时间比较

哈希验证使用 `CRYPTO_memcmp()`（OpenSSL）而非 `memcmp()`，防止时序攻击。

---

## 11. 数据库表

### 11.1 warden_checks（acore_world 数据库）

```sql
CREATE TABLE `warden_checks` (
  `id` smallint unsigned NOT NULL AUTO_INCREMENT,
  `type` tinyint unsigned DEFAULT NULL,
  `data` varchar(48) DEFAULT NULL,
  `str` varchar(170) DEFAULT NULL,
  `address` int unsigned DEFAULT NULL,
  `length` tinyint unsigned DEFAULT NULL,
  `result` varchar(24) DEFAULT NULL,
  `comment` varchar(50) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
```

| 列 | 类型 | 用途 |
|----|------|------|
| `id` | smallint unsigned | 检查 ID（主键，线性索引用于 vector） |
| `type` | tinyint unsigned | WardenCheckType 枚举值 |
| `data` | varchar(48) | 十六进制数据（PAGE/DRIVER 的哈希种子） |
| `str` | varchar(170) | 字符串数据（Lua 代码/文件名/驱动名） |
| `address` | int unsigned | 内存地址 |
| `length` | tinyint unsigned | 读取长度 |
| `result` | varchar(24) | 预期结果十六进制（MEM/MPQ 的哈希值） |
| `comment` | varchar(50) | 检查描述 |

**加载策略（`WardenCheckMgr::LoadWardenChecks()`）：**
1. 先查询 `MAX(id)` 确定 vector 大小
2. 按 `id` 顺序加载，存入 `CheckStore[id]`（O(1) 访问）
3. 同时按类型分入 `CheckIdPool[WARDEN_CHECK_MEM/LUA/OTHER_TYPE]`
4. 每个 Lua 检查生成 4 字符的 `IdStr`（如 `0788`）
5. Lua 检查 ID 限制 ≤ 9999（因为 IdStr 为 4 位）
6. Lua 检查 str 长度限制 ≤ 170 字节（`WARDEN_MAX_LUA_CHECK_LENGTH`）
7. 每个 Action 默认取 `CONFIG_WARDEN_CLIENT_FAIL_ACTION`

**种子数据：** 基础 SQL 文件包含 794 条检查记录（ID 1-811，有间隔），覆盖：
- 内存完整性检查（防飞、防穿墙、防无坠伤等）
- 页面哈希检查（扫描代码注入）
- 驱动检测（已知恶意驱动）
- DLL 注入检测（RPE、WPE PRO、tMorph、BlackMagic 等）
- Lua 脚本检测（PQR、EWT、WoWPlus、Lua Unlocker 等）

### 11.2 warden_action（acore_characters 数据库）

```sql
CREATE TABLE `warden_action` (
  `wardenId` smallint unsigned NOT NULL,
  `action` tinyint unsigned DEFAULT NULL,
  PRIMARY KEY (`wardenId`)
) ENGINE=InnoDB;
```

| 列 | 类型 | 用途 |
|----|------|------|
| `wardenId` | smallint unsigned | 对应 warden_checks.id |
| `action` | tinyint unsigned | 0=日志, 1=踢出, 2=封禁 |

**加载策略（`WardenCheckMgr::LoadWardenOverrides()`）：**
- 覆盖 `CheckStore[wardenId].Action` 的值
- 支持热重载（`.reload warden_action` GM 命令）
- 存储在 characters 数据库中（按服务器/阵营/角色数据）

---

## 12. 配置项汇总

| 配置键 | 默认值 | 类型 | 含义 |
|--------|--------|------|------|
| `Warden.Enabled` | `true` | bool | 是否启用 Warden 系统 |
| `Warden.NumMemChecks` | `3` | uint32 | 每批发送的内存/DLL 检查数量 |
| `Warden.NumLuaChecks` | `1` | uint32 | 每批发送的 Lua 检查数量 |
| `Warden.NumOtherChecks` | `7` | uint32 | 每批发送的页面/驱动等检查数量 |
| `Warden.BanDuration` | `86400` | uint32 | 封禁持续时间（秒），默认 1 天 |
| `Warden.ClientCheckHoldOff` | `30` | uint32 | 两批检查之间的最小间隔（秒） |
| `Warden.ClientCheckFailAction` | `0` | uint32 | 检查失败默认动作（0=日志, 1=踢, 2=封禁） |
| `Warden.ClientResponseDelay` | `600` | uint32 | 客户端响应超时（秒），超时则踢出 |

**定义位置：** `WorldConfig.cpp:554-561`

---

## 13. GM 命令

| 命令 | 用途 |
|------|------|
| `.reload warden_action` | 热重载 `warden_action` 表，允许运行时调整检查的惩罚动作 |

实现位置：`cs_reload.cpp:752-761`，需要 `RBAC_PERM_COMMAND_RELOAD` 权限。

---

## 14. WardenPayloadMgr 深度分析

### 14.1 设计目标

WardenPayloadMgr 是一个**利用 Warden Lua 检查通道发送自定义 Lua 脚本**的子系统。它允许服务端开发者：

- 在客户端 UI 中创建自定义框架
- 读取客户端 CVars
- 调用被保护的 Lua 函数
- 实现无需客户端补丁的服务端功能

### 14.2 Payload ID 分配

```
标准检查 ID: 0 ~ 4999        (数据库 warden_checks)
自定义 Payload ID: 5000 ~ 9999 (WardenPayloadOffsetMin ~ Max)
Lua 检查 ID 上限: 9999       (4 字符 IdStr 的十进制上限)
```

建议使用 9000-9999 范围避免与数据库 ID 冲突。

### 14.3 与标准检查的区别

| 特性 | 标准检查 (warden_checks) | 自定义 Payload |
|------|--------------------------|---------------|
| 来源 | 数据库 | 运行时代码注册 |
| ID 范围 | 0-4999 | 5000-9999 |
| 包编码 | 带 Lua 前缀/中缀/后缀 | 直接发送 Lua 代码（无包装） |
| 缓存位置 | sWardenCheckMgr->CheckStore | WardenPayloadMgr->CachedChecks |
| 发送方式 | 从 CheckIdPool 轮询 | 通过 QueuedPayloads 队列优先插入 |

### 14.4 发送流程

```
1. 代码注册:
   payloadMgr->RegisterPayload("myLuaCode")  → 自动分配 ID（如 5000）
   payloadMgr->QueuePayload(5000)

2. RequestChecks() 时:
   当处理 WARDEN_CHECK_LUA_TYPE 时:
   if !_payloadMgr.QueuedPayloads.empty():
       取出队列头部 → push_front 到 _CurrentChecks
       替代标准的数据库检查

3. 包构建时:
   if checkId >= WardenPayloadOffsetMin:
       // 无前缀/中缀/后缀，直接发送
       buff << uint8(check->Str.size())
       buff.append(check->Str.data(), check->Str.size())
```

---

## 15. ForceChecks 机制

`ForceChecks()` 用于在检查结果尚未返回时强制发送新的检查请求：

```
ForceChecks()
    ├── if _dataSent:
    │   ├── _interrupted = true
    │   └── _interruptCounter++    // 支持嵌套打断
    │
    └── RequestChecks()            // 立即构建新请求

HandleData() 中:
    ├── if _interrupted:
    │   ├── 忽略所有检查结果（旧数据已过时）
    │   ├── _interruptCounter--
    │   └── if counter == 0: _interrupted = false
    └── 只有未被中断时才执行惩罚
```

**使用场景：** 当游戏状态突然变化（如进入副本、切换地图），旧的检查结果可能因内存布局变化而产生误报，此时需要强制刷新。

---

## 16. WardenMac 实现差异

`WardenMac` 是 macOS 客户端的 Warden 实现，但**当前已被禁用**（`WorldSession.cpp:1351` 被注释）。

主要差异：
- 使用硬编码的模块种子（`0x4D808D2C77D905C41A6380EC08586AFE`）
- 模块 ID 为 `0DBBF209A27B1E279A9FEC5C168A15F7`（MD5）
- 哈希验证使用不同的密钥派生算法（XOR/加减/乘法而非预存密钥）
- `RequestChecks()` 只发送固定测试字符串 `"Test string!"`
- `HandleData()` 验证 SHA1 和 MD5 后直接踢出玩家
- **不继承 IsCheckInProgress() 和 ForceChecks()**（无中断机制）
- **不实现完整的检查构建/结果验证**

---

## 17. 安全设计分析

### 17.1 通信安全

- 所有 Warden 通信使用 RC4 加密
- 密钥通过 SessionKey 派生（SHA1 基哈希链）
- 握手后切换到模块内嵌密钥（双层密钥体系）
- 检查指令使用 XOR 混淆（`check->Type ^ xorByte`）
- 哈希验证使用常量时间比较（防时序攻击）

### 17.2 反绕过设计

- **响应超时检测：** 客户端必须在 `ClientResponseDelay` 秒内响应，否则踢出
- **校验和验证：** 响应包必须通过 SHA1-based checksum
- **TIMING_CHECK：** 检测 `GetTickCount()` 是否被 Hook
- **操作系统限制：** 仅允许 Win 和 OSX 客户端（`WorldSocket.cpp:605`）
- **CMSG 绕过队列：** Warden 响应包不走常规 opcode 队列，防止被延迟

### 17.3 已知限制

- RC4 加密已被认为不安全（但受限于客户端实现）
- WardenMac 未启用（macOS 客户端没有反作弊保护）
- TIMING_CHECK 被注释（`result == 0x00` 的检查因误报率高已禁用）
- PROC_CHECK 未实现（代码中有注释掉的框架）
- `ForceChecks` 期间所有检查结果被忽略（可能被滥用）
- WardenPayloadMgr 可被用于执行任意 Lua（安全双刃剑）

---

## 18. 数据流总览

```
[启动时]
    acore_world.warden_checks ──→ WardenCheckMgr::CheckStore
    acore_characters.warden_action ──→ CheckStore[].Action 覆盖

[会话建立]
    WorldSocket::HandleAuthSession()
        │
        ├── SessionKeyGenerator<SHA1>(SessionKey)
        │   → _inputKey, _outputKey → ARC4 初始化
        │
        ├── 从 WardenModuleWin.h 加载模块二进制
        │   → RequestModule() → MODULE_USE 包
        │
        ├── 客户端 MODULE_MISSING → SendModuleToClient() × N
        ├── 客户端 MODULE_OK → RequestHash()
        ├── 客户端 HASH_RESULT → HandleHashResult()
        │   → 验证模块 → 密钥切换 → _initialized = true
        └── InitializeModule() → 注册函数地址

[运行时循环]
    WorldSession::Update(diff)
        └── Warden::Update(diff)
            ├── _dataSent == false:
            │   └── _checkTimer到期 → RequestChecks()
            │       ├── 从 CheckIdPool 取出检查 ID
            │       ├── 包大小过滤 (≤500 字节)
            │       ├── 构建 XOR 编码数据包
            │       ├── RC4 加密 → SMSG_WARDEN_DATA
            │       └── _dataSent = true
            │
            └── _dataSent == true:
                ├── _clientResponseTimer 累加
                └── 超时 → KickPlayer

[客户端响应]
    CMSG_WARDEN_DATA (绕过队列直接处理)
        └── HandleWardenDataOpcode()
            └── WardenWin::HandleData()
                ├── 验证 Length + Checksum
                ├── 处理 TIMING_CHECK
                ├── 逐个验证检查结果
                │   ├── MEM_CHECK: 内存比较
                │   ├── PAGE/DRIVER/MODULE: 0xE9 JMP 检测
                │   ├── LUA_EVAL: 由 Addon 通道处理
                │   └── MPQ_CHECK: SHA1 哈希比较
                ├── 检查失败 → ApplyPenalty()
                │   ├── LOG: 仅日志
                │   ├── KICK: 踢出
                │   └── BAN: BanAccount(duration)
                └── 设置 _checkTimer = HoldOff

[Lua 检查结果]
    客户端 SendAddonMessage('_TW\t[检查ID]')
        └── ProcessLuaCheckResponse()
            ├── 验证检查 ID 有效且为 LUA_EVAL_CHECK
            ├── 有效 → ApplyPenalty(id)
            └── 无效/伪造 → ApplyPenalty(0, "Sent bogus Lua check response")
```
