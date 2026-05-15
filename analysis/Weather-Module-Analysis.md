# AzerothCore Weather 模块详细分析

## 1. 模块概览

Weather 模块负责模拟艾泽拉斯世界中各区域的天气系统。它实现了基于概率的天气变化机制，支持季节性天气概率差异，并通过网络协议向客户端广播天气状态变更。

**文件结构：**

| 文件 | 职责 |
|------|------|
| `Weather.h` / `Weather.cpp` | 单区域天气实例，负责天气生成、更新、网络包发送 |
| `WeatherMgr.h` / `WeatherMgr.cpp` | 天气数据管理器，负责从数据库加载天气配置数据 |

**模块位置：** `src/server/game/Weather/`

---

## 2. 数据结构

### 2.1 WeatherSeasonChances（`Weather.h:32-37`）

定义单个季节中各种天气类型的发生概率：

```cpp
struct WeatherSeasonChances
{
    uint32 rainChance;    // 雨天概率（0-100）
    uint32 snowChance;    // 雪天概率（0-100）
    uint32 stormChance;   // 风暴概率（0-100）
};
```

### 2.2 WeatherData（`Weather.h:39-43`）

包含所有四个季节的天气概率和关联脚本：

```cpp
struct WeatherData
{
    WeatherSeasonChances data[WEATHER_SEASONS]; // 春/夏/秋/冬
    uint32 ScriptId;                             // 关联的 WeatherScript ID
};
```

### 2.3 WeatherState 枚举（`Weather.h:45-61`）

客户端可见的天气状态，用于网络协议传输：

| 枚举值 | 数值 | 含义 |
|--------|------|------|
| `WEATHER_STATE_FINE` | 0 | 晴天 |
| `WEATHER_STATE_FOG` | 1 | 雾 |
| `WEATHER_STATE_LIGHT_RAIN` | 3 | 小雨 |
| `WEATHER_STATE_MEDIUM_RAIN` | 4 | 中雨 |
| `WEATHER_STATE_HEAVY_RAIN` | 5 | 大雨 |
| `WEATHER_STATE_LIGHT_SNOW` | 6 | 小雪 |
| `WEATHER_STATE_MEDIUM_SNOW` | 7 | 中雪 |
| `WEATHER_STATE_HEAVY_SNOW` | 8 | 大雪 |
| `WEATHER_STATE_LIGHT_SANDSTORM` | 22 | 小沙尘暴 |
| `WEATHER_STATE_MEDIUM_SANDSTORM` | 41 | 中沙尘暴 |
| `WEATHER_STATE_HEAVY_SANDSTORM` | 42 | 大沙尘暴 |
| `WEATHER_STATE_THUNDERS` | 86 | 雷暴 |
| `WEATHER_STATE_BLACKRAIN` | 90 | 黑暗之雨 |
| `WEATHER_STATE_BLACKSNOW` | 106 | 黑暗之雪 |

### 2.4 WeatherType 枚举（`SharedDefines.h:3369-3379`）

服务端内部使用的天气类型，比 WeatherState 更粗粒度：

```cpp
enum WeatherType
{
    WEATHER_TYPE_FINE       = 0,   // 晴天
    WEATHER_TYPE_RAIN       = 1,   // 雨
    WEATHER_TYPE_SNOW       = 2,   // 雪
    WEATHER_TYPE_STORM      = 3,   // 风暴
    WEATHER_TYPE_THUNDERS   = 86,  // 雷
    WEATHER_TYPE_BLACKRAIN  = 90   // 黑雨
};

#define MAX_WEATHER_TYPE 4  // 基础天气类型数量（不含特殊类型）
```

---

## 3. 类设计

### 3.1 WeatherMgr 命名空间（`WeatherMgr.h`）

提供全局的天气数据管理功能，使用匿名命名空间中的 `std::unordered_map<uint32, WeatherData>` 存储所有区域的天氣配置。

```
WeatherMgr
├── LoadWeatherData()           // 启动时从数据库加载
└── GetWeatherData(zone_id)     // 按区域 ID 查询天气配置
```

#### 3.1.1 数据加载流程（`WeatherMgr.cpp:44-105`）

```
LoadWeatherData()
    │
    ├── 查询 WorldDatabase: game_weather 表
    │
    └── 遍历结果集：
        ├── 读取 zone_id
        ├── 循环 4 个季节 (WEATHER_SEASONS)：
        │   ├── rainChance  = fields[season * 3 + 1]
        │   ├── snowChance  = fields[season * 3 + 2]
        │   ├── stormChance = fields[season * 3 + 3]
        │   └── 验证每个概率 <= 100，超限则重置为 25 并记录错误
        └── ScriptId = sObjectMgr->GetScriptId(ScriptName)
```

**数据库表结构 (`game_weather`)：**

| 列名 | 含义 |
|------|------|
| `zone` | 区域 ID |
| `spring_rain/snow/storm_chance` | 春季雨/雪/暴概率 |
| `summer_rain/snow/storm_chance` | 夏季雨/雪/暴概率 |
| `fall_rain/snow/storm_chance` | 秋季雨/雪/暴概率 |
| `winter_rain/snow/storm_chance` | 冬季雨/雪/暴概率 |
| `ScriptName` | 关联的脚本名称 |

### 3.2 Weather 类（`Weather.h:64-90`）

代表一个区域（zone）的天气实例，管理天气状态的生命周期。

```
Weather
├── 构造: Weather(Map*, zone, WeatherData const*)
├── Update(uint32 diff)                    // 主循环 tick
├── ReGenerate()                           // 随机生成新天气
├── UpdateWeather()                        // 广播天气变更
├── SendWeatherUpdateToPlayer(Player*)     // 单玩家发送
├── SendFineWeatherUpdateToPlayer(Player*) // 静态：发送晴天
├── SetWeather(type, grade)                // 手动设置天气
├── GetZone() / GetScriptId()              // 访问器
└── GetWeatherState()                      // 内部转换：Type+Grade -> State
```

**成员变量：**

| 变量 | 类型 | 用途 |
|------|------|------|
| `m_map` | `Map*` | 所属地图引用 |
| `m_zone` | `uint32` | 区域 ID |
| `m_type` | `WeatherType` | 当前天气类型 |
| `m_grade` | `float` | 天气强度 (0.0 ~ 1.0) |
| `m_timer` | `IntervalTimer` | 天气变化计时器 |
| `m_weatherChances` | `WeatherData const*` | 天气概率配置（来自数据库） |

---

## 4. 核心流程分析

### 4.1 系统启动

`WeatherMgr::LoadWeatherData()` 在 `World::SetInitialWorldSettings()` 中被调用（`World.cpp:605`），位于加载生物链接复活点之后、加载任务之前。

```
World::SetInitialWorldSettings()                       // World.cpp:605
    └── WeatherMgr::LoadWeatherData()
            └── 从 game_weather 表加载所有区域的天气概率配置
```

### 4.2 Weather 实例的创建与销毁

Weather 对象**不是全局预创建的**，而是按需创建、延迟销毁：

```
Map::GetOrGenerateZoneDefaultWeather(zoneId)     // Map.cpp:3191
    │
    ├── 查询 WeatherMgr::GetWeatherData(zoneId)
    │   └── 无数据 → return nullptr
    │
    └── 查找 _zoneDynamicInfo[zoneId].DefaultWeather
        ├── 已存在 → 直接返回
        └── 不存在 → 创建 Weather(map, zoneId, weatherData)
            ├── ReGenerate()     // 初始生成天气
            └── UpdateWeather()  // 广播给当前区域玩家
```

**销毁机制：** 当 `Weather::Update()` 返回 `false`（区域无玩家）时，`Map::UpdateWeather()` 会将 `DefaultWeather` unique_ptr reset，从而销毁 Weather 对象。

```
Map::UpdateWeather(diff)                          // Map.cpp:3155
    │
    ├── _weatherUpdateTimer.Update(diff)
    ├── 未到时间 → return
    └── 遍历所有 zoneDynamicInfo：
        ├── DefaultWeather->Update(interval)
        │   └── 返回 false → zoneInfo.DefaultWeather.reset()  // 销毁！
        └── _weatherUpdateTimer.Reset()
```

### 4.3 主更新循环（三级计时器架构）

天气系统采用**三级计时器**架构，实现精确的更新频率控制：

```
Level 1: Map::Update(t_diff)                         // 每帧调用 (Map.cpp:518)
    │
    └── Level 2: Map::UpdateWeather(t_diff)           // 每 1 秒触发 (Map.cpp:3155)
            │   _weatherUpdateTimer.SetInterval(1s)    // Map.cpp:81
            │
            └── Level 3: Weather::Update(interval)    // 每 10 分钟触发 (Weather.cpp:42)
                    │   m_timer = CONFIG_INTERVAL_CHANGEWEATHER (默认 600000ms)
                    │
                    ├── m_timer.Passed()?
                    │   ├── m_timer.Reset()
                    │   ├── ReGenerate()      // 生成新天气
                    │   │   └── 天气有变化?
                    │   │       └── UpdateWeather()  // 广播
                    │   │           └── 返回 false → 销毁 Weather
                    │   └── sScriptMgr->OnWeatherUpdate(this, diff)
                    └── return true (继续存活)
```

> **注意：** `Weather::Update()` 接收的 `diff` 参数是 `_weatherUpdateTimer.GetInterval()`（即 1 秒），而不是真正的帧间隔。这意味着 `Weather::m_timer` 每 1 秒累加一次。

### 4.4 天气生成算法（`Weather::ReGenerate()` — `Weather.cpp:67-185`）

这是天气系统最核心的算法，分为多个阶段：

**阶段 1：决定是否变化（概率表）**

```
urand(0, 99) → u
├── u < 30  → 不变化，return false（30% 概率保持现状）
├── u < 60  → 天气好转或改变（30%）
├── u < 90  → 天气恶化（30%）
└── u < 100 → 激烈变化（10%）
```

**阶段 2：根据当前状态执行变化**

```
保存 old_type, old_grade

计算当前季节：
  season = ((DayInYear - 78 + 365) / 91) % 4
  // 春分(3/20)起，每91天一个季节

[u < 60] && [grade < 0.333]:
    → 设为晴天 (type=FINE, grade=0)

[u < 60] && [type != FINE]:
    → grade -= 0.3333（强度降低），return true

[u < 90] && [type != FINE]:
    → grade += 0.3333（强度增加），return true

[type != FINE]（即 u >= 90，激烈变化）:
    ├── grade < 0.333 → grade = 0.9999（轻→暴）
    ├── grade > 0.667 → 50%: grade -= 0.667（暴→轻）
    │                 → 50%: 清空（→阶段3）
    └── 中等强度 → 清空（→阶段3）
```

**阶段 3：选择新天气类型（当前为 FINE 时）**

```
从 WeatherData 中获取当前季节的概率：
  chance1 = rainChance
  chance2 = rainChance + snowChance
  chance3 = chance2 + stormChance

rnd = urand(1, 100)
├── rnd <= chance1     → RAIN
├── rnd <= chance2     → SNOW
├── rnd <= chance3     → STORM
└── rnd > chance3      → FINE
```

**阶段 4：确定强度**

```
[type == FINE]:
    → grade = 0.0

[u < 90]（普通变化）:
    → grade = rand_norm() * 0.3333   // 轻度（0 ~ 0.3333）

[u >= 90]（激烈变化）:
    → 50%: grade = rand_norm() * 0.3333 + 0.3334  // 中度
    → 50%: grade = rand_norm() * 0.3333 + 0.6667  // 重度
```

**返回值：** `m_type != old_type || m_grade != old_grade`（仅当天气真正发生变化时返回 true）

### 4.5 WeatherType → WeatherState 转换（`Weather::GetWeatherState()` — `Weather.cpp:279-315`）

将内部天气类型+强度映射为客户端显示状态：

```
grade < 0.27 → FINE（任何类型，强度极低都视为晴天）

RAIN:
  ├── grade < 0.40 → LIGHT_RAIN
  ├── grade < 0.70 → MEDIUM_RAIN
  └── grade >= 0.70 → HEAVY_RAIN

SNOW:
  ├── grade < 0.40 → LIGHT_SNOW
  ├── grade < 0.70 → MEDIUM_SNOW
  └── grade >= 0.70 → HEAVY_SNOW

STORM:
  ├── grade < 0.40 → LIGHT_SANDSTORM
  ├── grade < 0.70 → MEDIUM_SANDSTORM
  └── grade >= 0.70 → HEAVY_SANDSTORM

BLACKRAIN  → BLACKRAIN（固定状态）
THUNDERS   → THUNDERS（固定状态）
FINE       → FINE
default    → FINE
```

### 4.6 天气广播流程（`Weather::UpdateWeather()` — `Weather.cpp:200-265`）

```
UpdateWeather()
    │
    ├── 限制 grade 范围: [0.0001, 0.9999]
    ├── 获取 WeatherState state = GetWeatherState()
    ├── 构建 Weather 包: WeatherPackets::Misc::Weather(state, grade)
    ├── Map::SendZoneMessage(zone, packet)
    │   ├── 遍历地图中所有玩家
    │   ├── 过滤：同一 zone && IsInWorld() && (可选: teamId 过滤)
    │   ├── player->SendDirectMessage(packet)
    │   └── 无玩家 → return false
    ├── 记录日志
    └── sScriptMgr->OnWeatherChange(this, state, grade)
```

---

## 5. 网络协议

### 5.1 Weather 数据包（`MiscPackets.h:31-42`，`MiscPackets.cpp:20-32`）

```cpp
class Weather final : public ServerPacket
{
public:
    Weather();
    Weather(WeatherState weatherID, float intensity = 0.0f, bool abrupt = false);
    WorldPacket const* Write() override;

    bool Abrupt = false;           // 是否立即切换（无过渡）
    float Intensity = 0.0f;        // 天气强度
    WeatherState WeatherID = WeatherState(0);  // 天气状态 ID
};
```

**网络包详情：**

- **Opcode:** `SMSG_WEATHER`
- **载荷大小:** 4 + 4 + 1 = 9 字节
- **字段:**
  - `uint32 WeatherID` — 天气状态枚举值
  - `float Intensity` — 天气强度 (0.0 ~ 1.0)
  - `uint8 Abrupt` — 是否跳过过渡动画直接切换

### 5.2 玩家进入区域时的天气同步

```
Player 进入新区域 (Player::UpdateZone)
    └── Map::SendZoneDynamicInfo(zoneId, player)    // Map.cpp:3112
            └── SendZoneWeather(zoneDynamicInfo, player)  // Map.cpp:3142
                    ├── 有 WeatherId 覆盖 → 发送覆盖天气
                    ├── 有 DefaultWeather → 发送默认天气
                    └── 都没有 → 发送晴天
```

---

## 6. 脚本系统集成

### 6.1 WeatherScript 基类（`WeatherScript.h:23-33`）

```cpp
class WeatherScript : public ScriptObject, public UpdatableScript<Weather>
{
protected:
    WeatherScript(const char* name);

public:
    [[nodiscard]] bool IsDatabaseBound() const override { return true; }

    // 天气变化时回调
    virtual void OnChange(Weather* weather, WeatherState state, float grade) { }
    // 每 tick 更新回调（继承自 UpdatableScript<Weather>）
    virtual void Update(Weather* weather, uint32 diff) { }
};
```

- `IsDatabaseBound() = true`：脚本通过 `game_weather.ScriptName` 字段与区域绑定
- 继承 `UpdatableScript<Weather>`，提供 `Update(Weather*, uint32 diff)` 钩子，每次 Weather tick 都会调用

### 6.2 脚本回调调用链

**OnWeatherChange（天气变更时）：**
```
ScriptMgr::OnWeatherChange(weather, state, grade)     // WeatherScript.cpp:23
    ├── ExecuteScript<ALEScript>(...)                  // ALE Lua 引擎回调
    │   └── ALEScript::OnWeatherChange(weather, state, grade)  // ALEScript.h:36
    └── ScriptRegistry<WeatherScript>::GetScriptById(weather->GetScriptId())
        └── tempScript->OnChange(weather, state, grade)  // DB 绑定脚本回调
```

**OnWeatherUpdate（每次 Weather tick）：**
```
ScriptMgr::OnWeatherUpdate(weather, diff)              // WeatherScript.cpp:38
    └── ScriptRegistry<WeatherScript>::GetScriptById(weather->GetScriptId())
        └── tempScript->Update(weather, diff)            // 通过 UpdatableScript 调用
```

> **注意：** `OnWeatherUpdate` 不会分发给 ALEScript，仅分发给数据库绑定的 WeatherScript。而 `OnWeatherChange` 会同时分发给两者。

---

## 7. 与其他系统的交互

### 7.1 与 Map 系统的关系

```
Map
├── _zoneDynamicInfo: unordered_map<uint32, ZoneDynamicInfo>
│   └── ZoneDynamicInfo
│       ├── DefaultWeather: unique_ptr<Weather>   // 自动生成的天气
│       ├── WeatherId: WeatherState                // 手动覆盖的天气状态
│       └── WeatherGrade: float                    // 手动覆盖的强度
├── _weatherUpdateTimer: IntervalTimer             // 天气更新节拍器
├── UpdateWeather(diff)                            // 驱动所有区域天气更新
├── GetOrGenerateZoneDefaultWeather(zoneId)        // 获取/创建天气实例
├── SetZoneWeather(zoneId, state, grade)           // 手动设置覆盖天气
├── SendZoneWeather(zoneId, player)                // 向玩家发送天气
└── SendZoneDynamicInfo(zoneId, player)            // 发送完整区域信息（含天气）
```

### 7.2 GM 命令

通过 `.weather` 命令可手动控制天气：

```
.cs_misc.cpp → HandleWeatherCommand
    └── player->GetMap()->SetZoneWeather(zoneId, WeatherState, grade)
            └── 设置 WeatherId + WeatherGrade 覆盖
            └── SendZoneMessage 广播
```

### 7.3 脚本中的使用示例

**祖阿曼 - 阿基尔松（鹰神）：**
```
boss_akilzon.cpp:172-174
  SetWeather(WEATHER_STATE_HEAVY_RAIN, 0.9999f)  // 战斗中暴雨
  → me->GetMap()->SetZoneWeather(me->GetZoneId(), state, grade)

  SetWeather(WEATHER_STATE_FINE, 0.0f)           // 战斗结束/死亡时恢复
```

**天灾入侵事件：**
```
scourge_invasion.cpp:70,72
  weather->SetWeather(WEATHER_TYPE_STORM, 0.25f)  // 事件开始：风暴
  weather->SetWeather(WEATHER_TYPE_RAIN, 0.0f)    // 事件结束：晴天
```

---

### 8.1 世界配置项

| 配置键 | 默认值 | 含义 | 位置 |
|--------|--------|------|------|
| `ChangeWeatherInterval` | `600000` (10分钟) | 天气自动变化间隔（毫秒） | `WorldConfig.cpp:175` |

### 8.2 IntervalTimer 工具类（`Timer.h:147-197`）

天气系统中大量使用的简单累加型周期计时器：

```cpp
struct IntervalTimer
{
    void Update(time_t diff) { _current += diff; if (_current < 0) _current = 0; }
    bool Passed() { return _current >= _interval; }
    void Reset() { if (_current >= _interval) _current %= _interval; }
    void SetCurrent(time_t current) { _current = current; }
    void SetInterval(time_t interval) { _interval = interval; }
    time_t GetInterval() const { return _interval; }
    time_t GetCurrent() const { return _current; }

private:
    time_t _interval{0};
    time_t _current{0};
};
```

**天气系统中的使用位置：**

| 计时器 | 所在类 | 间隔 | 用途 |
|--------|--------|------|------|
| `m_timer` | `Weather` | 600000ms (10分钟) | 控制天气重新生成的触发频率 |
| `_weatherUpdateTimer` | `Map` | 1000ms (1秒) | 控制对区域天气对象的轮询频率 |

---

## 9. 关键设计特征

### 9.1 按需创建/自动销毁

Weather 对象不会在服务器启动时为每个区域预创建。只有当有玩家处于某个区域时，通过 `GetOrGenerateZoneDefaultWeather()` 延迟创建。当区域无玩家时，`UpdateWeather()` 广播失败导致 Weather 对象被销毁。这是一种资源优化策略。

### 9.2 双层天气系统

- **DefaultWeather**：基于概率自动变化的动态天气
- **WeatherId 覆盖**：GM 命令或脚本设置的手动覆盖天气

当存在覆盖天气时，DefaultWeather 仍然存在但不会主动广播。当覆盖被清除后，DefaultWeather 恢复自动更新。

### 9.3 季节感知

天气生成使用 `Acore::Time::GetDayInYear()` 计算当前季节，从春分（3月20日，一年第78天）开始，每91天一个季节。不同季节使用不同的雨/雪/暴概率配置，使天气系统与现实世界的季节变化相呼应。

### 9.4 强度分级

天气强度 `grade` (0.0-1.0) 决定了 `WeatherState` 的显示等级：
- `< 0.27` → 晴天（阈值以下不显示天气效果）
- `0.27 - 0.40` → 轻度
- `0.40 - 0.70` → 中度
- `0.70+` → 重度

### 9.5 Markov 链式变化

天气变化不是完全随机的，而是基于当前状态进行概率转移：
- 30% 概率维持现状
- 30% 概率好转（强度降低）
- 30% 概率恶化（强度增加）
- 10% 概率剧烈变化

这种设计使天气变化更自然，避免频繁跳变。

---

## 10. 数据流总结

```
[game_weather DB 表]
        │
        ▼ (启动时)
WeatherMgr::LoadWeatherData()
        │
        ▼
_weatherData (unordered_map<zone_id, WeatherData>)
        │
        ▼ (按需)
Map::GetOrGenerateZoneDefaultWeather(zoneId)
        │
        ▼
Weather 对象创建
        │
        ▼ (每 ChangeWeatherInterval 毫秒)
Weather::Update()
    ├── ReGenerate()  → 基于概率+季节生成新天气
    └── UpdateWeather() → WeatherState 转换
        │
        ▼
Map::SendZoneMessage() → 玩家客户端
        │
        ▼
WeatherPackets::Misc::Weather → SMSG 包
```
