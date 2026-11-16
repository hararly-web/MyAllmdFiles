# 第5课 深入专题 — Timer Daemon Task 精讲

> **目标：** 理解 Daemon Task 的创建、生命周期、命令队列机制，掌握定时器回调约束的根因
> **前置：** 第5课基础版软件定时器已完成
> **关联：** 本章内容适用于所有 xxxFromISR / Deferred Interrupt Handling 等 Deferred-Execution 模式

---

## 0. 为什么需要 Daemon Task？

先思考一个设计问题：

```
┌──────────────────────────────────────────────────────────────────┐
│  FreeRTOS 的设计约束:                                              │
│                                                                  │
│  问题1: Tick ISR 必须极快 (< 几μs)                                │
│     → 不能在 ISR 中执行用户的定时器回调 (回调可能做任何事情)         │
│                                                                  │
│  问题2: 定时器操作 (Start/Stop/Create) 可能被多任务并发调用          │
│     → 需要串行化所有对定时器列表的修改 (命令队列 = 天然互斥)        │
│                                                                  │
│  问题3: 需要一种 "延迟执行" 机制                                   │
│     → ISR 只记录 "定时器X到期了", 由某个任务稍后执行回调            │
│                                                                  │
│  解决方案: 创建一个专用的 Task, 所有定时器操作都通过它串行处理       │
│           这就是 Daemon Task                                      │
│                                                                  │
│  ┌─────────────┐     ┌──────────────┐     ┌─────────────┐       │
│  │ Tick ISR    │────→│  命令队列     │────→│ Daemon Task │       │
│  │ (快速标记)   │     │ (缓冲 & 串行) │     │ (执行回调)   │       │
│  └─────────────┘     └──────────────┘     └─────────────┘       │
│                                                                  │
│  这个模式叫 Deferred Execution (延迟执行)                          │
│  硬件 ISR → 只做最少工作 → 信号/命令 → 软件 Task 处理              │
│  第5课 Daemon Task = 第8课中断延迟处理的预热!                      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Daemon Task 是什么？

```
┌──────────────────────────────────────────────────────────────────┐
│  Daemon Task 不是魔法, 它就是一个普通的 FreeRTOS Task             │
│                                                                  │
│  FreeRTOS 在 vTaskStartScheduler() 内部自动创建它                  │
│  (当 configUSE_TIMERS=1 时)                                      │
│                                                                  │
│  等价伪代码:                                                      │
│                                                                  │
│  // FreeRTOS/timers.c 内部的启动逻辑                               │
│  xTimerCreateTimerTask()                                         │
│  {                                                               │
│      xReturn = xTaskCreate(                                       │
│          prvTimerTask,                  // ← 入口函数              │
│          "Tmr Svc",                     // ← 任务名               │
│          configTIMER_TASK_STACK_DEPTH,  // ← 栈大小 (128 words)   │
│          NULL,                          // ← 无参数               │
│          configTIMER_TASK_PRIORITY,     // ← 优先级 (4)           │
│          &xTimerTaskHandle              // ← 保存句柄             │
│      );                                                          │
│  }                                                               │
│                                                                  │
│  就像一个你创建的 xTaskCreate(MyTask, "MyTask", ...) 一样,         │
│  Daemon Task 有: TCB / 栈 / 优先级 / 状态 / 名字                  │
│                                                                  │
│  你可以在 Keil 调试器中看到它:                                     │
│    → RTOS Viewer → Tasks → "Tmr Svc" (或类似名字)                 │
│    → vTaskList() 输出 "Tmr Svc    R   4    ..."                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. 命令队列 — 一切的枢纽

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                   Timer 命令队列 = Daemon Task 的唯一天地                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  所有定时器操作都不是"直接"的:                                                 │
│    xTimerStart()  ────┐                                                      │
│    xTimerStop()   ────┤                                                      │
│    xTimerReset()  ────┼──→ 写入命令 ──→ [命令队列] ──→ Daemon Task 处理       │
│    xTimerChangePeriod()─┤                   ▲               │                │
│    Tick ISR (到期)  ────┘                   │               ▼                │
│                                    configTIMER_QUEUE_LENGTH  实际执行操作     │
│                                    (默认10, 队列深度)                         │
│                                                                              │
│  命令类型 (tmrCOMMAND_xxx):                                                    │
│    tmrCOMMAND_START              启动定时器                                    │
│    tmrCOMMAND_STOP               停止定时器                                    │
│    tmrCOMMAND_CHANGE_PERIOD      改变周期                                     │
│    tmrCOMMAND_RESET              重置                                        │
│    tmrCOMMAND_START_DONT_TRACE   启动 (不产生 trace 事件)                      │
│    tmrCOMMAND_EXECUTE_CALLBACK   执行回调 (由 Tick ISR 写入!)                   │
│                                                                              │
│  每个命令的结构 (DaemonTaskMessage_t):                                         │
│    ┌─────────────────────────────────────┐                                   │
│    │ TimerHandle_t xTimer;               │ ← 哪个定时器                        │
│    │ BaseType_t     xCommandID;          │ ← 什么操作                          │
│    │ TickType_t     xOptionalValue;      │ ← 附加数据 (如新周期)               │
│    │ TickType_t     xMessageID;          │ ← 消息序号 (排序用)                 │
│    └─────────────────────────────────────┘                                   │
│                                                                              │
│  关键: 这个队列的大小是 configTIMER_QUEUE_LENGTH                               │
│  队列满了 → xTimerStart(timeout) 会阻塞或返回失败!                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 一条命令的完整旅程

以 `xTimerStart(g.xTimerOneShot, pdMS_TO_TICKS(100))` 为例：

```
时间线: 你的任务调用 → 命令入队 → Daemon Task 处理 → 定时器激活

┌──────────────────────────────────────────────────────────────────────────────┐
│ 阶段1: 你的任务上下文 (如 Control_Task, prio=2)                                │
│                                                                              │
│   Control_Task:                                                              │
│     xTimerStart(g.xTimerOneShot, pdMS_TO_TICKS(100));                        │
│       │                                                                      │
│       ├── 构造 DaemonTaskMessage_t:                                          │
│       │     .xTimer       = g.xTimerOneShot                                 │
│       │     .xCommandID   = tmrCOMMAND_START                                │
│       │     .xMessageID   = xTaskGetTickCount() (用于排序)                   │
│       │                                                                      │
│       └── xQueueSend(daemon_cmd_q, &msg, pdMS_TO_TICKS(100));               │
│             │                                                                │
│             ├── 队列有空位 → 立即写入, 返回 pdPASS                             │
│             │   你的任务继续运行                                               │
│             │                                                                │
│             └── 队列满 → 你的任务阻塞在这里 (最多 100ms)                        │
│                 - 100ms内 Daemon 处理了一条命令 → 空位出现 → 写入 → pdPASS     │
│                 - 100ms后仍满 → 返回 errQUEUE_FULL → 你决定怎么处理            │
│                                                                              │
│  注意: xTimerStart 返回 ≠ 定时器已经启动了!                                    │
│        它只是命令入队了。真正启动要等 Daemon Task 处理这条命令                   │
├──────────────────────────────────────────────────────────────────────────────┤
│ 阶段2: Daemon Task 上下文 (prio=4)                                            │
│                                                                              │
│   prvTimerTask() 循环体 (简化):                                               │
│                                                                              │
│   for (;;)                                                                   │
│   {                                                                          │
│       /* 读取下一条命令, 如果有定时器将到期则最多等待到那个时刻 */               │
│       xQueueReceive(daemon_cmd_q, &msg, xNextExpireTime);                    │
│                                                                              │
│       /* 处理到期定时器 (如果有的话) */                                         │
│       prvProcessReceivedCommands();                                          │
│                                                                              │
│       /* 处理命令 */                                                          │
│       switch (msg.xCommandID)                                                │
│       {                                                                      │
│           case tmrCOMMAND_START:                                             │
│               /* ① 把 myTimer 加入 active_timer_list */                      │
│               /* ② 计算到期时间 = now + period */                             │
│               /* ③ 如果到期时间 < 当前最早的到期时间 → 更新 Daemon 的等待超时  */ │
│               break;                                                         │
│                                                                              │
│           case tmrCOMMAND_EXECUTE_CALLBACK:     /* ← Tick ISR 写入的 */       │
│               /* ① 调用 myTimer->pxCallbackFunction(myTimer) */               │
│               /* ② 如果是 Auto-reload → 重新计算到期时间, 保持 active */       │
│               /* ③ 如果是 One-shot   → 移到 dormant_list */                  │
│               break;                                                         │
│       }                                                                      │
│   }                                                                          │
│                                                                              │
│   Daemon Task 的执行循环优先级分析:                                            │
│     Daemon(prio4) > Control_Task(prio2)                                      │
│     → Control_Task 调用 xTimerStart → 入队 →                                 │
│       如果 Daemon 在等队列 → Daemon 被唤醒 (prio4 > prio2)                    │
│       → Daemon 立即抢占 Control_Task → 处理 START 命令                       │
│       → Daemon 处理完回到 xQueueReceive                                     │
│       → Control_Task 恢复运行 (xTimerStart 返回)                             │
│                                                                              │
│   看到了吗? 一个 xTimerStart 可能触发两次任务切换!                              │
│     ① Control → Daemon (命令入队唤醒)                                         │
│     ② Daemon → Control (命令处理完毕)                                        │
└──────────────────────────────────────────────────────────────────────────────┘
│ 阶段3: Tick ISR 触发到期                                                      │
│                                                                              │
│   SysTick_Handler (每 10ms, 因为 configTICK_RATE_HZ=100)                      │
│     → xPortSysTickHandler()                                                  │
│       → xTaskIncrementTick()                                                 │
│         → prvCheckTimersInActiveList()                                       │
│           │                                                                  │
│           │  /* 检查 active 定时器列表中是否有到期的 */                         │
│           │  myTimer = active_list[0];  ← 按到期时间排序, 最早的在第一个        │
│           │  if (current_tick >= myTimer->expiry_time)                        │
│           │  {                                                               │
│           │      /* 构造 EXECUTE_CALLBACK 命令 */                              │
│           │      msg.xCommandID = tmrCOMMAND_EXECUTE_CALLBACK;               │
│           │      msg.xTimer     = myTimer;                                   │
│           │      /* 写入 Daemon 命令队列 */                                    │
│           │      xQueueSendFromISR(daemon_cmd_q, &msg, &flag);               │
│           │      /* 如果 Daemon 在等队列 → 标记 Daemon 为 Ready */            │
│           │  }                                                               │
│           │                                                                  │
│           │  关键: Tick ISR 不直接调用 myTimer 的回调!                          │
│           │        它写入 EXECUTE_CALLBACK 命令后就返回了                       │
│           │        真实的回调执行发生在上面的 阶段2 中                           │
│                                                                              │
│   这解释了为什么 "定时器精度 = tick 精度 + Daemon 调度延迟":                      │
│     tick N:   定时器到期, ISR 写入 EXECUTE_CALLBACK                           │
│     tick N:   xPortSysTickHandler 可能触发 PendSV (任务切换)                   │
│     tick N+?: Daemon 被调度 → xQueueReceive 返回 → 执行回调                    │
│     → "?" 取决于: 当前运行的任务优先级是否 > Daemon                             │
│               Daemon 命令队列中有多少条命令排在前面                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 回调约束的根因

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  为什么定时器回调 "不能阻塞"？                                                  │
│                                                                              │
│  根因: 你的回调运行在 Daemon Task 的栈上                                        │
│                                                                              │
│  ┌──────────────────────────────────────┐                                    │
│  │        Daemon Task 栈 (128 words)     │                                    │
│  │                                      │                                    │
│  │  ┌──────────────────────────────┐   │                                    │
│  │  │ prvProcessReceivedCommands() │   │                                    │
│  │  │   │                          │   │                                    │
│  │  │   └→ switch(cmd)            │   │                                    │
│  │  │        case EXECUTE:        │   │                                    │
│  │  │          │                  │   │                                    │
│  │  │          └→ YourCallback()  │   │  ← 你的回调就在这里!                  │
│  │  │               │             │   │    运行在 Daemon Task 的栈帧上!        │
│  │  │               │             │   │                                    │
│  │  │    如果你 vTaskDelay(1000):  │   │                                    │
│  │  │      Daemon Task 阻塞       │   │  Daemon Task 被移出 Ready List      │
│  │  │      1秒内: 没有任务处理    │   │                                    │
│  │  │      定时器命令队列          │   │  ALL 其他定时器过期事件堆积在队列里!   │
│  │  │      你的回调去度假了        │   │  你的回调去度假了                     │
│  │  │      所有其他定时器: 💀      │   │  所有其他定时器: 💀                   │
│  │  └──────────────────────────────┘   │                                    │
│  └──────────────────────────────────────┘                                    │
│                                                                              │
│  栈溢出风险:                                                                  │
│    你的回调 + Daemon 自己的栈帧 + 可能的嵌套函数调用                            │
│    → 如果回调里定义了大的局部数组 → 栈溢出                                     │
│    → configTIMER_TASK_STACK_DEPTH 默认 128 words = 512 bytes                 │
│    → 不够用! 需要增大                                                        │
│                                                                              │
│  正确模式: 回调 → 信号量 → Worker Task                                        │
│  ┌──────────────────────────────────────────────────────┐                   │
│  │ 回调函数 (Daemon Task 上下文中, 几 μs)                │                   │
│  │   xSemaphoreGive(xWorkerSem);  // 非阻塞, μs 级       │                   │
│  │                                                      │                   │
│  │ Worker Task (独立栈, 可以阻塞)                        │                   │
│  │   xSemaphoreTake(xWorkerSem, portMAX_DELAY);          │                   │
│  │   // 做重活: vTaskDelay, 计算, printf, ...            │                   │
│  └──────────────────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 命令队列溢出场景

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  什么时候命令队列会满？                                                        │
│                                                                              │
│  场景: 你快速连续调用 xTimerStart, 但 Daemon 来不及处理                         │
│                                                                              │
│  for (int i = 0; i < 20; i++)     // ← 循环 20 次                            │
│  {                                                                           │
│      xTimerStart(timers[i], 0);   // ← 每次写一条 START 命令                 │
│  }                                                                           │
│                                                                              │
│  如果 configTIMER_QUEUE_LENGTH = 10:                                          │
│    前 10 次 → pdPASS (入队成功)                                                │
│    第 11 次 → 取决于 xBlockTime 参数:                                         │
│      xBlockTime = 0:             → 立即返回 errQUEUE_FULL                     │
│      xBlockTime = portMAX_DELAY: → 阻塞直到 Daemon 处理了某条命令, 腾出空位     │
│      xBlockTime = 100ms:         → 等最多 100ms, 超时返回 errQUEUE_FULL       │
│                                                                              │
│  生产环境的守卫做法:                                                           │
│                                                                              │
│  BaseType_t result = xTimerStart(timer, pdMS_TO_TICKS(50));                  │
│  if (result != pdPASS)                                                       │
│  {                                                                           │
│      // 命令队列长期满 → Daemon Task 被饿死了!                                │
│      // 根因: 有更高优先级任务占着 CPU, Daemon 得不到执行                      │
│      // 方案: 提升 configTIMER_TASK_PRIORITY 或降低阻塞任务的优先级             │
│      error_count++;                                                          │
│  }                                                                           │
│                                                                              │
│  经验公式:                                                                    │
│    configTIMER_QUEUE_LENGTH ≥ MAX(定时器数量, 预期的瞬时操作次数)               │
│    默认值 10 对于大多数场景足够了                                              │
│    如果你有 20 个定时器且可能在短时间内全部 Start → 设为 20+                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 调试 Daemon Task

### 6.1 用 vTaskList 确认 Daemon Task 存在

```c
/* 在 Control_Task 中调用 */
static char task_list_buf[512];
vTaskList(task_list_buf);
/*
  输出示例:
  Name          State  Prio  Stack  Num
  Tmr Svc       B      4     89     3    ← Daemon Task! (名字因版本而异)
  Control       R      2     156    5
  IDLE          R      0     45     1
  ...
*/
```

### 6.2 用 uxTaskGetStackHighWaterMark 检查 Daemon 栈

```c
/* FreeRTOSConfig.h 需要: INCLUDE_xTaskGetIdleTaskHandle = 1 */
extern TaskHandle_t xTimerTaskHandle;  /* FreeRTOS 内部变量, timers.c 中定义 */

UBaseType_t daemon_stack_free = uxTaskGetStackHighWaterMark(xTimerTaskHandle);
/*
  如果 daemon_stack_free < 20 (words), 说明栈快不够用了
  → 增大 configTIMER_TASK_STACK_DEPTH
  → 或者在回调中减少局部变量
*/
```

### 6.3 查看命令队列积压

```c
/* Daemon 命令队列也是普通队列, 可以用队列 API 查询 */
extern QueueHandle_t xTimerQueue;  /* FreeRTOS 内部变量 (非公开API, 调试用) */

UBaseType_t pending_cmds = uxQueueMessagesWaiting(xTimerQueue);
/*
  pending_cmds = 0/1 → 正常
  pending_cmds ≥ 5  → Daemon 处理不过来了!
  pending_cmds ≥ configTIMER_QUEUE_LENGTH → 队列满, 新命令会失败
*/
```

> ⚠️ `xTimerTaskHandle` 和 `xTimerQueue` 是 FreeRTOS 内部变量（`timers.c` 中定义），不是公开 API。调试时可以通过 `extern` 声明访问，生产代码中应避免依赖内部实现。

---

## 7. 配置建议对照表

| 场景 | configTIMER_TASK_PRIORITY | configTIMER_TASK_STACK_DEPTH | configTIMER_QUEUE_LENGTH |
|------|--------------------------|------------------------------|--------------------------|
| 简单 LED 闪烁 (≤3个定时器) | configMAX_PRIORITIES - 1 (=4) | 128 (默认) | 10 (默认) |
| 回调中有复杂运算 | 不变 | **256** (回调局部变量多) | 不变 |
| 10+ 个定时器 | **configMAX_PRIORITIES - 1 或更高** | 不变 | **定时器数量 + 5** |
| 高优先级任务经常抢占 | **高于所有任务** | 不变 | 不变 |
| 回调通过信号量通知 Worker | 不变 | 不变 (回调极轻量) | 不变 |
| 定时器用于超时检测 (要求精准) | **最高优先级** | 不变 | 不变 |

---

## 8. 一句话总结

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   Timer Daemon Task = 一个你看不见的 xTaskCreate() 出来的普通任务               │
│                                                                              │
│   它的工作:                                                                    │
│     ① 从命令队列收消息 (xQueueReceive)                                         │
│     ② 按消息类型操作定时器列表                                                  │
│     ③ 到期时调用你的回调函数 (注意: 在你的回调里, 你是在 Daemon 的栈上运行!)     │
│                                                                              │
│   它的约束 → 你的约束:                                                         │
│     → Daemon 只有一个 (所有定时器共享它)                                       │
│     → 所以: 你的回调不能阻塞, 不能耗时                                         │
│     → 所以: Daemon 优先级要高于使用定时器的任务                                 │
│     → 所以: 命令队列深度要够 (>= 瞬时最大操作数)                                │
│                                                                              │
│   这本质上是 "Deferred Execution" 模式:                                        │
│     ISR → 只做最少的事 → 命令入队 → Daemon Task → 执行用户代码                 │
│     第8课中断管理中会再次见到这个模式!                                          │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

> 📎 **基础版:** 见 `Lesson5_Timer_Basic.md` — One-shot + Auto-reload 定时器代码实战
