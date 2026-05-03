# World Timer Array Analysis

## 概述

本文档只描述当前代码中已经核实的 `World.cpp` 定时器数组实现，不包含未经验证的推测性表结构说明。

分析范围主要包括：

- `src/server/game/World/World.h`
- `src/server/game/World/World.cpp`
- `src/common/Utilities/Timer.h`

目标是讲清楚三件事：

1. `_timers[WUPDATE_COUNT]` 的执行原理
2. 每个 `WorldTimers` 枚举项在当前代码里的实际用途
3. 哪些地方容易被误解，哪些地方当前实现本身有明显问题

---

## 1. `WorldTimers` 枚举

当前定义在 [World.h](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.h)：

```cpp
enum WorldTimers
{
    WUPDATE_UPTIME,
    WUPDATE_EVENTS,
    WUPDATE_CLEANDB,
    WUPDATE_AUTOBROADCAST,
    WUPDATE_MAILBOXQUEUE,
    WUPDATE_PINGDB,
    WUPDATE_5_SECS,
    WUPDATE_WHO_LIST,
    WUPDATE_COUNT
};
```

这里的含义不是“线程列表”，而是：

- `World` 对象有一个 `_timers[WUPDATE_COUNT]` 数组
- 数组中的每一格都是一个 `IntervalTimer`
- 枚举值只是数组下标的语义名字

也就是说：

- `_timers[WUPDATE_UPTIME]` 是 uptime 定时器
- `_timers[WUPDATE_WHO_LIST]` 是 who list 定时器
- `_timers[WUPDATE_5_SECS]` 是一个共享的 5 秒 timer

---

## 2. 定时器底层原理

### 2.1 `IntervalTimer` 结构

定义在 [Timer.h](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/common/Utilities/Timer.h#L147)：

```cpp
struct IntervalTimer
{
public:
    void Update(time_t diff)
    {
        _current += diff;
        if (_current < 0)
            _current = 0;
    }

    bool Passed()
    {
        return _current >= _interval;
    }

    void Reset()
    {
        if (_current >= _interval)
            _current %= _interval;
    }

    void SetCurrent(time_t current) { _current = current; }
    void SetInterval(time_t interval) { _interval = interval; }
    [[nodiscard]] time_t GetInterval() const { return _interval; }
    [[nodiscard]] time_t GetCurrent() const { return _current; }

private:
    time_t _interval{0};
    time_t _current{0};
};
```

只有两个核心状态：

| 字段 | 作用 |
|------|------|
| `_interval` | 目标间隔，单位毫秒 |
| `_current` | 当前已累计时间，单位毫秒 |

你可以把它理解成一个非常简单的累计计时器：

- `SetInterval(5000)`：目标间隔设为 5000 ms
- `Update(diff)`：把本帧经过的时间加进去
- `Passed()`：判断累计时间是否达到间隔
- `Reset()`：达到后保留余数，进入下一轮

### 2.2 `diff` 是什么

`World::Update(uint32 diff)` 的 `diff` 表示：

- 本次世界循环距离上次世界循环经过了多少毫秒

这意味着：

- timer 不是独立线程
- timer 不是系统级定时器
- timer 的推进完全依赖主循环每帧传进来的 `diff`

如果主线程某一帧卡住了，那么下一帧的 `diff` 会变大，timer 也会一起“跳过去”。

### 2.3 `_timers[]` 是如何推进的

推进位置在 [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1106)：

```cpp
for (int i = 0; i < WUPDATE_COUNT; ++i)
{
    if (_timers[i].GetCurrent() >= 0)
        _timers[i].Update(diff);
    else
        _timers[i].SetCurrent(0);
}
```

这段代码的真实含义是：

1. 世界主循环每帧运行一次
2. 每帧都遍历整个 `_timers[]`
3. 给每个 timer 累加本帧的 `diff`

然后每个业务分支再单独判断：

```cpp
if (_timers[XXX].Passed())
{
    _timers[XXX].Reset();
    // 执行业务逻辑
}
```

### 2.4 `Passed()` 和 `Reset()` 的实际意义

`Passed()`：

```cpp
return _current >= _interval;
```

表示“这个 timer 到时间了”。

`Reset()`：

```cpp
_current %= _interval;
```

不是简单清零，而是保留超过周期的余数。

例子：

- interval = 5000
- current = 5378
- Reset 后 current = 378

这么做的好处是：

- 不会因为帧时间不稳定而不断累积误差
- 长期平均频率更稳定

### 2.5 一个完整例子

假设某个 timer：

- `_interval = 5000`
- `_current = 0`

然后每帧推进：

| 帧 | `diff` | 更新后 `_current` | `Passed()` |
|----|--------|-------------------|------------|
| 1 | 100 | 100 | false |
| 2 | 120 | 220 | false |
| ... | ... | ... | ... |
| 49 | 70 | 5050 | true |

当 `Passed()` 为真时，就会进入业务分支：

```cpp
if (_timers[WUPDATE_5_SECS].Passed())
{
    _timers[WUPDATE_5_SECS].Reset();
    // 执行任务
}
```

Reset 之后：

- `_current = 50`

下一轮就从 50 ms 继续累计。

### 2.6 timer 本身不会“主动执行任务”

这是最容易误解的点。

`IntervalTimer` 只负责：

- 记录时间
- 判断是否到点

它并不会主动回调任何函数。

真正决定“到点后做什么”的，是 `World::Update()` 里的 `if (_timers[...].Passed())` 分支。

所以完整模型是：

1. `SetInterval()` 设定周期
2. 每帧 `Update(diff)` 推进 timer
3. `Passed()` 判断是否到点
4. 到点后执行当前帧里的业务逻辑
5. `Reset()` 进入下一轮

### 2.7 为什么会有“定期卡一下”的体感

因为这些定时任务都运行在世界主线程路径上。

timer 只负责“决定哪一帧执行”，真正造成卡顿的是：

- 到点后执行的那段业务逻辑本身很重

例如：

- 全量遍历在线玩家
- 全量扫描拍卖
- 同步查库处理大量邮件

因此：

- timer 不是卡顿根因
- timer 是“把某段重逻辑固定安排在某个节拍执行”的机制

---

## 3. 当前代码中的 timer 初始化

在 [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp) 的初始化阶段，当前可见的设置如下：

```cpp
_timers[WUPDATE_UPTIME].SetInterval(getIntConfig(CONFIG_UPTIME_UPDATE) * MINUTE * IN_MILLISECONDS);
_timers[WUPDATE_CLEANDB].SetInterval(getIntConfig(CONFIG_LOGDB_CLEARINTERVAL) * MINUTE * IN_MILLISECONDS);
_timers[WUPDATE_AUTOBROADCAST].SetInterval(getIntConfig(CONFIG_AUTOBROADCAST_INTERVAL));
_timers[WUPDATE_PINGDB].SetInterval(getIntConfig(CONFIG_DB_PING_INTERVAL) * MINUTE * IN_MILLISECONDS);
_timers[WUPDATE_5_SECS].SetInterval(5 * IN_MILLISECONDS);
_timers[WUPDATE_WHO_LIST].SetInterval(15 * IN_MILLISECONDS);
_timers[WUPDATE_EVENTS].SetInterval(nextGameEvent);
```

这里需要注意两点：

1. `WUPDATE_EVENTS` 不是固定值，而是动态间隔
2. `WUPDATE_WHO_LIST` 在当前代码中已经是 15 秒，不再是 5 秒

---

## 4. 当前代码中每个 timer 的实际用途

这一节只写当前代码里明确能确认的用途。

### 4.1 `WUPDATE_UPTIME`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1258)

作用：

- 周期性更新 `uptime` 表中的本次启动运行时长和峰值在线人数

特点：

- 周期来自 `UpdateUptimeInterval`
- 属于异步数据库写

### 4.2 `WUPDATE_EVENTS`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1273)

作用：

- 驱动 `sGameEventMgr->Update()`
- 动态计算下一次事件检查时间

特点：

- 不是固定周期
- 每次执行后会重新设置 interval

### 4.3 `WUPDATE_CLEANDB`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1192)

作用：

- 周期性清理登录库旧日志

特点：

- 周期来自 `LogDB.Opt.ClearInterval`
- 只有 `CONFIG_LOGDB_CLEARTIME > 0` 时才执行

### 4.4 `WUPDATE_AUTOBROADCAST`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1218)

作用：

- 周期性调用 `sAutobroadcastMgr->SendAutobroadcasts()`

特点：

- 周期来自 `AutoBroadcast.Timer`
- 前置开关是 `CONFIG_AUTOBROADCAST`

### 4.5 `WUPDATE_MAILBOXQUEUE`

当前状态：

- 枚举中存在
- 当前 `World.cpp` 中没有看到对应的 `SetInterval()` 和 `Passed()` 分支

结论：

- 当前版本中这个 timer 没有实际启用

### 4.6 `WUPDATE_PINGDB`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1283)

作用：

- 调用 `CharacterDatabase.KeepAlive()`
- 调用 `LoginDatabase.KeepAlive()`
- 调用 `WorldDatabase.KeepAlive()`

特点：

- 作用是数据库连接保活
- 不属于业务数据处理

### 4.7 `WUPDATE_5_SECS`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1115)

当前作用：

- 每 5 秒执行一次到期角色封禁清理 SQL

对应逻辑：

```cpp
if (_timers[WUPDATE_5_SECS].Passed())
{
    _timers[WUPDATE_5_SECS].Reset();

    CharacterDatabasePreparedStatement* stmt = CharacterDatabase.GetPreparedStatement(CHAR_DEL_EXPIRED_BANS);
    CharacterDatabase.Execute(stmt);
}
```

注意：

- 这个 timer 当前不是“通用维护任务容器”的抽象概念
- 但从命名上看，它仍然是一个共享节拍 timer，而不是“角色封禁专用 timer”

这也是后续想拆分角色封禁清理逻辑时，需要考虑单独建 timer 的原因。

### 4.8 `WUPDATE_WHO_LIST`

执行位置：

- [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1125)

当前作用：

- 周期性调用 `sWhoListCacheMgr->Update()`
- 重建 `/who` 查询缓存

当前周期：

- 15 秒

特点：

- 在主线程中全量重建 who 缓存
- 周期越短，CPU 压力越高
- 周期越长，`/who` 结果越不实时

---

## 5. `_timers[]` 之外的“按时间点触发”任务

`World.cpp` 里不是所有周期逻辑都通过 `_timers[]` 数组驱动。

还有一类逻辑是：

- 用 `_nextDailyQuestReset`
- `_nextWeeklyQuestReset`
- `_nextMonthlyQuestReset`
- `_nextRandomBGReset`
- `_nextCalendarOldEventsDeletionTime`
- `_nextGuildReset`
- `_mail_expire_check_timer`

这些并不是 `IntervalTimer`，而是直接保存一个“绝对时间点”，然后每帧比较：

```cpp
if (currentGameTime > _nextDailyQuestReset)
{
    ResetDailyQuests();
}
```

所以在理解 `World.cpp` 调度逻辑时，要把两类机制分开：

1. `_timers[]`：固定间隔或动态间隔的累计型 timer
2. `_nextXXX`：按绝对时间点触发的比较型 timer

---

## 6. 当前实现中已确认的两个问题

### 6.1 `GetNextWhoListUpdateDelaySecs()` 读取了错误的 timer

函数在 [World.cpp](E:/games/wow-study/AzerothCore_SRC/azerothcore-wotlk/src/server/game/World/World.cpp#L1816)：

```cpp
uint32 World::GetNextWhoListUpdateDelaySecs()
{
    if (_timers[WUPDATE_5_SECS].Passed())
        return 1;

    uint32 t = _timers[WUPDATE_5_SECS].GetInterval() - _timers[WUPDATE_5_SECS].GetCurrent();
    t = std::min(t, (uint32)_timers[WUPDATE_5_SECS].GetInterval());

    return uint32(std::ceil(t / 1000.0f));
}
```

从函数名字看，它应该返回：

- `WUPDATE_WHO_LIST` 下次触发还有多久

但当前代码实际读取的是：

- `WUPDATE_5_SECS`

所以这个函数的语义当前是错误的。

### 6.2 `WUPDATE_5_SECS` 的命名和用途耦合不清晰

目前 `WUPDATE_5_SECS` 名字表达的是“一个 5 秒节拍”，  
但业务上它实际被用来执行“过期角色封禁清理”。

这带来的问题是：

- 如果以后只想调整角色封禁清理频率
- 直接修改 `WUPDATE_5_SECS` 可能误伤依赖它的其他代码

因此从维护性上讲：

- 更清晰的做法是给“过期角色封禁清理”单独建 timer

---

## 7. 理解 timer 行为时最容易踩的坑

### 7.1 `SetInterval()` 不会马上执行任务

`SetInterval(5000)` 只是把目标间隔设成 5000 ms，  
并不会立刻执行任务。

真正触发任务还要等：

1. 主循环不断 `Update(diff)`
2. `_current` 累加到 `_interval`
3. `Passed()` 为真

### 7.2 改某个 timer 的 interval，不等于改了“所有 5 秒逻辑”

只有引用同一个 timer 槽位的逻辑才会受到影响。

例如：

- 改 `WUPDATE_WHO_LIST`
- 不会自动改 `WUPDATE_5_SECS`

但如果某个辅助函数错误地引用了别的 timer，就会出现“看起来无关，实际被误伤”的情况。

### 7.3 timer 到点不代表“后台执行”

当前模型里：

- 到点后的业务逻辑是在世界主线程执行

所以 timer 只是调度器，不是异步执行器。

---

## 8. 一句话总结

`World.cpp` 的 `_timers[]` 不是线程，也不是系统级定时器，而是一个在世界主循环中逐帧推进的 `IntervalTimer` 数组；每个 timer 通过“累计 `diff` -> `Passed()` 判断 -> 当前帧执行逻辑 -> `Reset()` 保留余数”的方式工作。

---

## 9. 当前文档不再包含的内容说明

为了去掉不准确内容，本文档刻意不再展开以下部分：

- 未逐项核实的数据库表字段结构
- 推测性的业务说明
- 与 `World timer array` 关系不大的旁支模块细节

如果后续需要，可以再拆成两份单独文档：

1. `World timer array` 原理文档
2. `World.cpp` 周期任务业务梳理文档

