# Quest 任务系统深度分析

## 1. 系统概述

**分析目标**：Quest 任务系统
**分析范围**：`src/server/game/Quests/`、`src/server/game/Handlers/QuestHandler.cpp`、`src/server/game/Entities/Player/PlayerQuest.cpp`
**分析重点**：任务生命周期、数据库表关系、状态流转

任务系统是 WoW 游戏的核心玩法之一，玩家通过完成任务获得经验、金币、物品、声望等奖励。任务系统涉及任务接取、进度追踪、完成提交、奖励发放等完整流程。

---

## 2. 核心数据结构

### 2.1 类图

```mermaid
classDiagram
    class Quest {
        +uint32 Id
        +uint32 Method
        +int32 ZoneOrSort
        +uint32 MinLevel
        +uint32 MaxLevel
        +int32 Level
        +uint32 Type
        +uint32 Flags
        +uint32 SpecialFlags
        +string Title
        +string Details
        +string Objectives
        +uint32 RequiredNpcOrGo[4]
        +uint32 RequiredNpcOrGoCount[4]
        +uint32 RequiredItemId[6]
        +uint32 RequiredItemCount[6]
        +uint32 RewardItemId[4]
        +uint32 RewardChoiceItemId[6]
        +GetQuestId()
        +IsRepeatable()
        +IsDaily()
        +IsWeekly()
        +IsAutoAccept()
        +IsAutoComplete()
        +XPValue()
    }
    
    class QuestStatusData {
        +QuestStatus Status
        +uint32 Timer
        +uint16 ItemCount[6]
        +uint16 CreatureOrGOCount[4]
        +uint16 PlayerCount
        +bool Explored
    }
    
    class Player {
        +QuestStatusMap m_QuestStatus
        +RewardedQuestSet m_RewardedQuests
        +CanTakeQuest()
        +CanAddQuest()
        +AddQuest()
        +CompleteQuest()
        +RewardQuest()
        +FailQuest()
        +AbandonQuest()
        +GetQuestStatus()
    }
    
    class ObjectMgr {
        +QuestMap mQuestTemplates
        +GetQuestTemplate()
        +LoadQuests()
    }
    
    ObjectMgr --> Quest : 管理
    Player --> QuestStatusData : 包含
    Player --> Quest : 操作
```

### 2.2 任务状态枚举

```cpp
enum QuestStatus : uint8
{
    QUEST_STATUS_NONE       = 0,  // 未接取
    QUEST_STATUS_COMPLETE   = 1,  // 已完成(可提交)
    QUEST_STATUS_INCOMPLETE = 3,  // 进行中
    QUEST_STATUS_FAILED     = 5,  // 失败
    QUEST_STATUS_REWARDED   = 6,  // 已领奖(仅内存使用)
};
```

### 2.3 任务标志

```cpp
enum QuestFlags
{
    QUEST_FLAGS_STAY_ALIVE          = 0x00000001,  // 完成任务期间必须存活
    QUEST_FLAGS_PARTY_ACCEPT        = 0x00000002,  // 队伍共享接受
    QUEST_FLAGS_SHARABLE            = 0x00000008,  // 可分享
    QUEST_FLAGS_RAID                = 0x00000040,  // 团队任务
    QUEST_FLAGS_HIDDEN_REWARDS      = 0x00000200,  // 隐藏奖励
    QUEST_FLAGS_TRACKING           = 0x00000400,  // 追踪任务(自动完成)
    QUEST_FLAGS_DAILY              = 0x00001000,  // 每日任务
    QUEST_FLAGS_FLAGS_PVP          = 0x00002000,  // 强制PVP
    QUEST_FLAGS_WEEKLY             = 0x00008000,  // 每周任务
    QUEST_FLAGS_AUTOCOMPLETE       = 0x00010000,  // 自动完成
    QUEST_FLAGS_AUTO_ACCEPT        = 0x00080000,  // 自动接受
};

enum QuestSpecialFlags
{
    QUEST_SPECIAL_FLAGS_REPEATABLE              = 0x0001,  // 可重复
    QUEST_SPECIAL_FLAGS_EXPLORATION_OR_EVENT    = 0x0002,  // 探索/事件
    QUEST_SPECIAL_FLAGS_AUTO_ACCEPT             = 0x0004,  // 自动接受
    QUEST_SPECIAL_FLAGS_DF_QUEST                = 0x0008,  // 随机副本任务
    QUEST_SPECIAL_FLAGS_MONTHLY                 = 0x0010,  // 每月任务
    QUEST_SPECIAL_FLAGS_CAST                    = 0x0020,  // 需要施法
    QUEST_SPECIAL_FLAGS_TIMED                   = 0x1000,  // 计时任务
};
```

---

## 3. 数据库表关系

### 3.1 数据库表关系图

```mermaid
erDiagram
    quest_template {
        uint ID PK "任务ID"
        string Title "任务标题"
        string Details "任务详情"
        string Objectives "任务目标"
        uint MinLevel "最低等级"
        int ZoneOrSort "区域/分类"
        uint Type "任务类型"
        uint Flags "任务标志"
        uint RewardXPDifficulty "经验奖励难度"
        int RewardMoney "金币奖励"
        uint RequiredNpcOrGo1 "目标NPC/GO 1"
        uint RequiredNpcOrGoCount1 "目标数量 1"
        uint RequiredItemId1 "需求物品 1"
        uint RequiredItemCount1 "物品数量 1"
        uint RewardItemId1 "奖励物品 1"
        uint RewardChoiceItemId1 "可选奖励 1"
    }
    
    quest_template_addon {
        uint ID PK "任务ID"
        uint MaxLevel "最高等级"
        uint RequiredClasses "职业限制"
        uint RequiredSkill "技能要求"
        uint PrevQuestId "前置任务"
        uint NextQuestId "后续任务"
        int ExclusiveGroup "互斥组"
        uint SpecialFlags "特殊标志"
    }
    
    creature_queststarter {
        uint id PK "NPC ID"
        uint quest PK "任务ID"
    }
    
    creature_questender {
        uint id PK "NPC ID"
        uint quest PK "任务ID"
    }
    
    gameobject_queststarter {
        uint id PK "GO ID"
        uint quest PK "任务ID"
    }
    
    gameobject_questender {
        uint id PK "GO ID"
        uint quest PK "任务ID"
    }
    
    character_queststatus {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
        uint status "任务状态"
        uint timer "计时器"
        uint itemCount1 "物品进度1"
        uint creatureCount1 "生物进度1"
        uint playerCount "玩家击杀数"
        tinyint explored "是否探索"
    }
    
    character_queststatus_daily {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
        uint time "完成时间"
    }
    
    character_queststatus_weekly {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
    }
    
    character_queststatus_monthly {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
    }
    
    character_queststatus_seasonal {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
        uint event "事件ID"
    }
    
    character_queststatus_rewarded {
        uint guid PK "角色GUID"
        uint quest PK "任务ID"
        bool active "是否激活"
    }
    
    quest_tracker {
        uint id PK "自增ID"
        uint quest_id "任务ID"
        uint character_guid "角色GUID"
        string core_hash "核心版本"
        string core_branch "分支"
        datetime quest_accept_time "接受时间"
        datetime quest_complete_time "完成时间"
        datetime quest_abandon_time "放弃时间"
    }
    
    quest_template ||--o{ quest_template_addon : "扩展"
    quest_template ||--o{ creature_queststarter : "发布者"
    quest_template ||--o{ creature_questender : "完成者"
    quest_template ||--o{ character_queststatus : "玩家进度"
    character_queststatus ||--o{ character_queststatus_daily : "每日"
    character_queststatus ||--o{ character_queststatus_weekly : "每周"
    character_queststatus ||--o{ character_queststatus_rewarded : "已完成"
```

### 3.2 数据库表说明

| 数据库 | 表名 | 作用 |
|--------|------|------|
| db_world | `quest_template` | 任务模板主表，定义任务基本信息、目标、奖励 |
| db_world | `quest_template_addon` | 任务模板扩展表，定义前置任务、互斥组、特殊标志等 |
| db_world | `quest_template_locale` | 任务模板本地化表 |
| db_world | `quest_details` | 任务详情文本表 |
| db_world | `quest_offer_reward` | 任务奖励文本表 |
| db_world | `quest_request_items` | 任务请求物品文本表 |
| db_world | `quest_greeting` | 任务问候文本表 |
| db_world | `quest_poi` | 任务POI(兴趣点)表 |
| db_world | `quest_poi_points` | 任务POI坐标点表 |
| db_world | `creature_queststarter` | NPC任务发布关系表 |
| db_world | `creature_questender` | NPC任务完成关系表 |
| db_world | `gameobject_queststarter` | GO任务发布关系表 |
| db_world | `gameobject_questender` | GO任务完成关系表 |
| db_world | `creature_questitem` | NPC任务物品掉落表 |
| db_world | `gameobject_questitem` | GO任务物品掉落表 |
| db_characters | `character_queststatus` | 角色任务进度表 |
| db_characters | `character_queststatus_daily` | 每日任务完成记录 |
| db_characters | `character_queststatus_weekly` | 每周任务完成记录 |
| db_characters | `character_queststatus_monthly` | 每月任务完成记录 |
| db_characters | `character_queststatus_seasonal` | 季节任务完成记录 |
| db_characters | `character_queststatus_rewarded` | 已完成任务记录 |
| db_characters | `quest_tracker` | 任务追踪日志表 |

---

## 4. 任务生命周期

### 4.1 任务状态流转图

```mermaid
stateDiagram-v2
    [*] --> NONE: 任务未接取
    
    NONE --> INCOMPLETE: 接受任务(CanTakeQuest + AddQuest)
    
    INCOMPLETE --> COMPLETE: 完成目标(CanCompleteQuest + CompleteQuest)
    INCOMPLETE --> FAILED: 任务失败(FailQuest)
    INCOMPLETE --> NONE: 放弃任务(AbandonQuest)
    
    COMPLETE --> REWARDED: 领取奖励(RewardQuest)
    COMPLETE --> INCOMPLETE: 目标失效(IncompleteQuest)
    
    FAILED --> NONE: 放弃任务
    FAILED --> INCOMPLETE: 重新接受(可重复任务)
    
    REWARDED --> [*]: 任务完成
    
    note right of NONE
        任务日志中不存在
        m_QuestStatus中无记录
    end note
    
    note right of INCOMPLETE
        任务日志中存在
        状态: QUEST_STATUS_INCOMPLETE
        追踪进度: ItemCount/CreatureOrGOCount
    end note
    
    note right of COMPLETE
        目标已完成
        状态: QUEST_STATUS_COMPLETE
        可提交NPC领取奖励
    end note
    
    note right of REWARDED
        奖励已领取
        m_RewardedQuests中记录
        不可重复(除非可重复任务)
    end note
```

### 4.2 任务生命周期时序图

```mermaid
sequenceDiagram
    participant C as 客户端
    participant H as QuestHandler
    participant P as Player
    participant Q as Quest
    participant DB as 数据库
    
    Note over C,DB: 1. 任务接取流程
    C->>H: CMSG_QUESTGIVER_STATUS_QUERY
    H->>P: GetQuestDialogStatus()
    P-->>H: DIALOG_STATUS_AVAILABLE
    H-->>C: SMSG_QUESTGIVER_STATUS
    
    C->>H: CMSG_QUESTGIVER_HELLO
    H->>P: PrepareQuestMenu()
    H-->>C: SMSG_QUESTGIVER_QUEST_LIST
    
    C->>H: CMSG_QUESTGIVER_QUERY_QUEST
    H-->>C: SMSG_QUESTGIVER_QUEST_DETAILS
    
    C->>H: CMSG_QUESTGIVER_ACCEPT_QUEST
    H->>P: CanTakeQuest(quest)
    H->>P: CanAddQuest(quest)
    H->>P: AddQuest(quest)
    P->>DB: INSERT character_queststatus
    P-->>C: SendQuestUpdate()
    
    Note over C,DB: 2. 任务进度更新流程
    C->>P: 击杀NPC/收集物品
    P->>P: KilledMonster() / ItemAdded()
    P->>P: UpdateQuestObjective()
    P->>DB: UPDATE character_queststatus
    P-->>C: SendQuestUpdate()
    
    Note over C,DB: 3. 任务完成流程
    C->>H: CMSG_QUESTGIVER_COMPLETE_QUEST
    H->>P: CanCompleteQuest()
    H->>P: CompleteQuest()
    P->>DB: UPDATE character_queststatus (status=1)
    H-->>C: SMSG_QUESTGIVER_REQUEST_ITEMS
    
    Note over C,DB: 4. 奖励领取流程
    C->>H: CMSG_QUESTGIVER_CHOOSE_REWARD
    H->>P: CanRewardQuest()
    H->>P: RewardQuest()
    P->>P: GiveXP(), ModifyMoney(), StoreItem()
    P->>DB: DELETE character_queststatus
    P->>DB: INSERT character_queststatus_rewarded
    P-->>C: SendQuestReward()
```

---

## 5. 核心流程详解

### 5.1 任务接取流程

```mermaid
flowchart TB
    subgraph 接取检查
        A[CMSG_QUESTGIVER_ACCEPT_QUEST] --> B{CanTakeQuest}
        B --> C{SatisfyQuestStatus}
        C --> D{SatisfyQuestLevel}
        D --> E{SatisfyQuestClass}
        E --> F{SatisfyQuestRace}
        F --> G{SatisfyQuestSkill}
        G --> H{SatisfyQuestReputation}
        H --> I{SatisfyQuestPreviousQuest}
        I --> J{SatisfyQuestDay/Week/Month}
    end
    
    subgraph 添加任务
        J --> K{CanAddQuest}
        K --> L{SatisfyQuestLog}
        L --> M{检查背包空间}
        M --> N[AddQuest]
    end
    
    subgraph 初始化状态
        N --> O[创建QuestStatusData]
        O --> P[Status = INCOMPLETE]
        P --> Q[初始化ItemCount/CreatureOrGOCount]
        Q --> R[GiveQuestSourceItem]
        R --> S[写入数据库]
    end
    
    classDef entry fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef decision fill:#fff8e1,stroke:#ff8f00
    classDef process fill:#f5f5f5,stroke:#757575
    classDef data fill:#e8f5e9,stroke:#2e7d32
    
    class A entry
    class B,C,D,E,F,G,H,I,J,K,L,M decision
    class N,O,P,Q,R process
    class S data
```

**关键函数**：

| 函数 | 位置 | 说明 |
|------|------|------|
| `CanTakeQuest()` | PlayerQuest.cpp:251 | 检查是否可以接取任务 |
| `CanAddQuest()` | PlayerQuest.cpp:265 | 检查任务日志是否有空位 |
| `AddQuest()` | PlayerQuest.cpp:508 | 添加任务到日志 |
| `SatisfyQuestStatus()` | Player.cpp | 检查任务状态(是否已完成) |
| `SatisfyQuestLevel()` | PlayerQuest.cpp:968 | 检查等级要求 |
| `SatisfyQuestPreviousQuest()` | Player.cpp | 检查前置任务 |

### 5.2 任务进度更新流程

```mermaid
flowchart LR
    subgraph 触发源
        A[击杀NPC] --> D[UpdateQuestKill]
        B[收集物品] --> E[UpdateQuestItem]
        C[探索区域] --> F[UpdateQuestExplore]
        G[施放法术] --> H[UpdateQuestCast]
    end
    
    subgraph 进度更新
        D --> I{检查任务目标}
        E --> I
        F --> I
        H --> I
        I --> J[增加进度计数]
        J --> K{是否完成}
        K -->|是| L[CompleteQuest]
        K -->|否| M[SendQuestUpdate]
    end
    
    subgraph 数据持久化
        L --> N[更新数据库状态]
        M --> O[更新数据库进度]
    end
    
    classDef trigger fill:#f3e5f5,stroke:#7b1fa2
    classDef process fill:#f5f5f5,stroke:#757575
    classDef decision fill:#fff8e1,stroke:#ff8f00
    classDef data fill:#e8f5e9,stroke:#2e7d32
    
    class A,B,C,G trigger
    class D,E,F,H,I,J process
    class K decision
    class L,M,N,O data
```

**进度更新函数**：

| 函数 | 说明 |
|------|------|
| `KilledMonster()` | 击杀怪物更新进度 |
| `TalkedToCreature()` | 与NPC对话更新进度 |
| `ItemAdded()` | 物品数量变化更新进度 |
| `AreaExplored()` | 探索区域更新进度 |
| `CastSpell()` | 施放法术更新进度 |

### 5.3 任务完成流程

```mermaid
flowchart TB
    A[CMSG_QUESTGIVER_COMPLETE_QUEST] --> B{CanCompleteQuest}
    
    B --> C{检查物品目标}
    C --> D{检查击杀目标}
    D --> E{检查探索目标}
    E --> F{检查计时器}
    F --> G{检查声望要求}
    
    G -->|全部满足| H[CompleteQuest]
    G -->|不满足| I[返回失败]
    
    H --> J[Status = COMPLETE]
    J --> K[SetQuestSlotState]
    K --> L{IsTrackingQuest}
    L -->|是| M[RewardQuest 自动领奖]
    L -->|否| N[等待玩家提交]
    
    classDef entry fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef decision fill:#fff8e1,stroke:#ff8f00
    classDef process fill:#f5f5f5,stroke:#757575
    classDef data fill:#e8f5e9,stroke:#2e7d32
    
    class A entry
    class B,C,D,E,F,G,L decision
    class H,J,K,M process
    class N data
```

### 5.4 奖励领取流程

```mermaid
flowchart TB
    A[CMSG_QUESTGIVER_CHOOSE_REWARD] --> B{CanRewardQuest}
    
    B --> C{检查任务状态}
    C --> D{检查每日/每周限制}
    D --> E{检查物品需求}
    E --> F{检查背包空间}
    
    F -->|通过| G[RewardQuest]
    F -->|失败| H[SendQuestGiverOfferReward]
    
    subgraph 奖励发放
        G --> I[销毁任务物品]
        I --> J[发放可选奖励]
        J --> K[发放固定奖励]
        K --> L[发放经验]
        L --> M[发放金币]
        M --> N[发放声望]
        N --> O[发放称号]
        O --> P[施放奖励法术]
    end
    
    subgraph 状态更新
        P --> Q[RemoveActiveQuest]
        Q --> R[SetRewardedQuest]
        R --> S[更新每日/每周状态]
        S --> T[SaveToDB]
    end
    
    classDef entry fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef decision fill:#fff8e1,stroke:#ff8f00
    classDef process fill:#f5f5f5,stroke:#757575
    classDef data fill:#e8f5e9,stroke:#2e7d32
    
    class A entry
    class B,C,D,E,F decision
    class G,I,J,K,L,M,N,O,P,Q,R,S process
    class T data
```

---

## 6. 网络消息处理

### 6.1 消息类型

| 消息 | 方向 | 说明 |
|------|------|------|
| `CMSG_QUESTGIVER_STATUS_QUERY` | C->S | 查询NPC任务状态 |
| `SMSG_QUESTGIVER_STATUS` | S->C | 返回NPC任务状态 |
| `CMSG_QUESTGIVER_HELLO` | C->S | 与NPC对话 |
| `SMSG_QUESTGIVER_QUEST_LIST` | S->C | 返回任务列表 |
| `CMSG_QUESTGIVER_QUERY_QUEST` | C->S | 查询任务详情 |
| `SMSG_QUESTGIVER_QUEST_DETAILS` | S->C | 返回任务详情 |
| `CMSG_QUESTGIVER_ACCEPT_QUEST` | C->S | 接受任务 |
| `CMSG_QUESTGIVER_COMPLETE_QUEST` | C->S | 完成任务 |
| `SMSG_QUESTGIVER_REQUEST_ITEMS` | S->C | 请求任务物品 |
| `CMSG_QUESTGIVER_CHOOSE_REWARD` | C->S | 选择奖励 |
| `SMSG_QUESTGIVER_OFFER_REWARD` | S->C | 显示奖励 |
| `CMSG_QUESTLOG_REMOVE_QUEST` | C->S | 放弃任务 |
| `CMSG_PUSHQUESTTOPARTY` | C->S | 分享任务给队伍 |
| `CMSG_QUEST_QUERY` | C->S | 查询任务信息 |
| `SMSG_QUEST_QUERY_RESPONSE` | S->C | 返回任务信息 |

### 6.2 Handler 函数列表

| 函数 | 文件位置 | 处理消息 |
|------|---------|---------|
| `HandleQuestgiverStatusQueryOpcode` | QuestHandler.cpp:35 | CMSG_QUESTGIVER_STATUS_QUERY |
| `HandleQuestgiverHelloOpcode` | QuestHandler.cpp:78 | CMSG_QUESTGIVER_HELLO |
| `HandleQuestgiverAcceptQuestOpcode` | QuestHandler.cpp:114 | CMSG_QUESTGIVER_ACCEPT_QUEST |
| `HandleQuestgiverQueryQuestOpcode` | QuestHandler.cpp:204 | CMSG_QUESTGIVER_QUERY_QUEST |
| `HandleQuestQueryOpcode` | QuestHandler.cpp:238 | CMSG_QUEST_QUERY |
| `HandleQuestgiverChooseRewardOpcode` | QuestHandler.cpp:251 | CMSG_QUESTGIVER_CHOOSE_REWARD |
| `HandleQuestgiverRequestRewardOpcode` | QuestHandler.cpp:356 | CMSG_QUESTGIVER_REQUEST_REWARD |
| `HandleQuestLogRemoveQuest` | QuestHandler.cpp:400 | CMSG_QUESTLOG_REMOVE_QUEST |
| `HandleQuestgiverCompleteQuest` | QuestHandler.cpp:488 | CMSG_QUESTGIVER_COMPLETE_QUEST |
| `HandlePushQuestToParty` | QuestHandler.cpp:539 | CMSG_PUSHQUESTTOPARTY |

---

## 7. 特殊任务类型

### 7.1 每日任务

```mermaid
flowchart LR
    A[接取每日任务] --> B{检查每日限制}
    B -->|已满25个| C[拒绝接取]
    B -->|未满| D[接取成功]
    D --> E[完成任务]
    E --> F[记录到character_queststatus_daily]
    F --> G[每日重置]
    G --> A
```

**特点**：
- 每天最多完成 25 个每日任务
- 每日服务器时间 03:00 重置
- 存储在 `character_queststatus_daily` 表

### 7.2 每周任务

**特点**：
- 每周重置一次
- 存储在 `character_queststatus_weekly` 表
- 通常与团队副本相关

### 7.3 计时任务

```mermaid
flowchart TB
    A[接取计时任务] --> B[启动计时器]
    B --> C{时间内完成?}
    C -->|是| D[正常完成]
    C -->|否| E[FailQuest]
    E --> F[任务失败]
    
    D --> G[停止计时器]
    G --> H[领取奖励]
```

**特点**：
- 有时间限制（`TimeAllowed` 字段）
- 超时自动失败
- 显示倒计时 UI

### 7.4 任务链

```mermaid
flowchart LR
    A[任务A] -->|PrevQuestId| B[任务B]
    B -->|PrevQuestId| C[任务C]
    C -->|NextQuestInChain| D[任务D]
    
    E[任务X] -.->|ExclusiveGroup| F[任务Y]
    F -.->|ExclusiveGroup| G[任务Z]
    
    style A fill:#e8f5e9
    style B fill:#e8f5e9
    style C fill:#e8f5e9
    style D fill:#e8f5e9
    style E fill:#fff8e1
    style F fill:#fff8e1
    style G fill:#fff8e1
```

**关系类型**：
- **前置任务**：`PrevQuestId`，必须先完成前置任务
- **后续任务**：`NextQuestInChain`，完成后自动接取
- **互斥任务**：`ExclusiveGroup`，同组任务只能完成一个

---

## 8. 关键代码片段

### 8.1 接受任务

```cpp
// PlayerQuest.cpp:508
void Player::AddQuest(Quest const* quest, Object* questGiver)
{
    uint16 log_slot = FindQuestSlot(0);
    if (log_slot >= MAX_QUEST_LOG_SIZE)
        return;

    uint32 quest_id = quest->GetQuestId();
    
    // 创建任务状态数据
    QuestStatusData& questStatusData = m_QuestStatus[quest_id];
    questStatusData.Status = QUEST_STATUS_INCOMPLETE;
    questStatusData.Explored = false;
    
    // 初始化物品目标计数
    if (quest->HasSpecialFlag(QUEST_SPECIAL_FLAGS_DELIVER))
        for (uint8 i = 0; i < QUEST_ITEM_OBJECTIVES_COUNT; ++i)
            questStatusData.ItemCount[i] = 0;
    
    // 初始化击杀目标计数
    if (quest->HasSpecialFlag(QUEST_SPECIAL_FLAGS_KILL | QUEST_SPECIAL_FLAGS_CAST | QUEST_SPECIAL_FLAGS_SPEAKTO))
        for (uint8 i = 0; i < QUEST_OBJECTIVES_COUNT; ++i)
            questStatusData.CreatureOrGOCount[i] = 0;
    
    // 给予任务起始物品
    GiveQuestSourceItem(quest);
    
    // 设置任务日志槽位
    SetQuestSlot(log_slot, quest_id, qtime);
    
    // 标记需要保存
    m_QuestStatusSave[quest_id] = true;
}
```

### 8.2 完成任务

```cpp
// PlayerQuest.cpp:599
void Player::CompleteQuest(uint32 quest_id)
{
    if (!quest_id)
        return;

    // 设置状态为完成
    SetQuestStatus(quest_id, QUEST_STATUS_COMPLETE);
    
    // 更新任务日志槽位状态
    auto log_slot = FindQuestSlot(quest_id);
    if (log_slot < MAX_QUEST_LOG_SIZE)
        SetQuestSlotState(log_slot, QUEST_STATE_COMPLETE);
    
    // 追踪任务自动领奖
    Quest const* qInfo = sObjectMgr->GetQuestTemplate(quest_id);
    if (qInfo && qInfo->HasFlag(QUEST_FLAGS_TRACKING))
        RewardQuest(qInfo, 0, this, false);
}
```

### 8.3 领取奖励

```cpp
// PlayerQuest.cpp:660
void Player::RewardQuest(Quest const* quest, uint32 reward, Object* questGiver, bool announce, bool isLFGReward)
{
    uint32 quest_id = quest->GetQuestId();
    
    // 销毁任务物品
    for (uint8 i = 0; i < QUEST_ITEM_OBJECTIVES_COUNT; ++i)
        DestroyItemCount(quest->RequiredItemId[i], quest->RequiredItemCount[i], true);
    
    // 发放可选奖励
    if (quest->GetRewChoiceItemsCount() > 0)
        if (uint32 itemId = quest->RewardChoiceItemId[reward])
            StoreNewItem(dest, itemId, true);
    
    // 发放固定奖励
    if (quest->GetRewItemsCount() > 0)
        for (uint32 i = 0; i < quest->GetRewItemsCount(); ++i)
            if (uint32 itemId = quest->RewardItemId[i])
                StoreNewItem(dest, itemId, true);
    
    // 发放声望
    RewardReputation(quest);
    
    // 发放经验
    uint32 XP = CalculateQuestRewardXP(quest);
    GiveXP(XP, nullptr, 1.0f, isLFGReward);
    
    // 发放金币
    if (int32 rewOrReqMoney = quest->GetRewOrReqMoney(GetLevel()))
        ModifyMoney(rewOrReqMoney);
    
    // 更新每日/每周状态
    if (quest->IsDaily())
        SetDailyQuestStatus(quest_id);
    else if (quest->IsWeekly())
        SetWeeklyQuestStatus(quest_id);
    
    // 移除活动任务，标记为已完成
    RemoveActiveQuest(quest_id, false);
    SetRewardedQuest(quest_id);
}
```

---

## 9. 总结

### 9.1 系统特点

1. **模块化设计**：任务定义、状态管理、网络处理分离
2. **灵活的目标类型**：支持击杀、收集、探索、施法等多种目标
3. **丰富的任务类型**：支持普通、每日、每周、每月、季节性任务
4. **完善的任务链**：支持前置任务、后续任务、互斥任务
5. **自动完成机制**：支持追踪任务、自动接受、自动完成

### 9.2 数据流向

```
quest_template (定义) 
    ↓
ObjectMgr (加载到内存)
    ↓
Player::m_QuestStatus (运行时状态)
    ↓
character_queststatus (持久化)
    ↓
character_queststatus_rewarded (完成记录)
```

### 9.3 关键文件

| 文件 | 说明 |
|------|------|
| `QuestDef.h/cpp` | 任务定义类 |
| `QuestHandler.cpp` | 网络消息处理 |
| `PlayerQuest.cpp` | 玩家任务操作 |
| `GossipDef.cpp` | 任务对话界面 |

---

*本文档基于 AzerothCore WotLK 源代码分析生成*
