# 第5课 代码实战 — 基础版：软件定时器 (One-shot + Auto-reload)

> **目标：** 掌握 xTimerCreate / xTimerStart / xTimerStop / xTimerChangePeriod / xTimerDelete
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第4课信号量已完成，理解任务阻塞和回调概念
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（已配好）

---

## 核心概念：软件定时器 vs 硬件定时器

```
┌──────────────────────────────────────────────────────────────────┐
│                    硬件定时器 vs 软件定时器                        │
│                                                                  │
│  硬件定时器 (如 TIM1, SysTick):                                    │
│    - 数量有限 (芯片只有 N 个定时器外设)                            │
│    - 精度高 (时钟周期级别, ns~μs)                                 │
│    - ISR 中处理, 不能阻塞                                         │
│    - 需要手动管理寄存器                                           │
│                                                                  │
│  软件定时器 (FreeRTOS Timer):                                     │
│    - 不限数量 (只要内存够)                                        │
│    - 精度低 (受 Daemon Task 调度影响, ms 级别)                    │
│    - 回调在 Daemon Task 上下文中执行, 可以调用部分 API              │
│    - 纯软件抽象, 无需操作硬件寄存器                                │
│                                                                  │
│  本质: FreeRTOS 内部维护一个有序定时器列表                         │
│        每次 tick 中断检查是否有定时器到期                          │
│        到期后 → 插入 Daemon Task 的命令队列                       │
│        Daemon Task 唤醒 → 依次执行回调                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 5 Basic: Software Timers
  *
  *          Demonstrates:
  *          - xTimerCreate()         — create one-shot & auto-reload timers
  *          - xTimerStart()          — start a timer (from task context)
  *          - xTimerStop()           — stop a running timer
  *          - xTimerChangePeriod()   — change period at runtime
  *          - xTimerReset()          — restart a timer from the beginning
  *          - Timer callback context — rules & constraints
  *          - Timer Daemon Task      — the hidden task behind timers
  *
  *          Architecture:
  *
  *            ┌─────────────────────────────────────────┐
  *            │         Timer Command Queue (10)         │
  *            │  [start][stop][period_change][...]       │
  *            └──────────────┬──────────────────────────┘
  *                           │
  *                           ▼
  *            ┌─────────────────────────────────────────┐
  *            │       Timer Daemon Task (prio=4)         │
  *            │  Processes commands, checks expiry,      │
  *            │  invokes callbacks in ITS OWN context    │
  *            └──────────────┬──────────────────────────┘
  *                           │
  *              ┌────────────┼──────────────┐
  *              ▼            ▼              ▼
  *         OneShot_Cb   AutoReload_Cb   (shared callback)
  *         LED0 flash   LED1 periodic   LED2 variable
  *
  *          Control_Task (prio=2):
  *            Periodically starts the one-shot timer
  *            Changes LED2 timer period to demonstrate runtime reconfig
  *
  *          LED assignment:
  *            LED0 (PB13): One-shot timer  — flashes once per trigger
  *            LED1 (PA1):  Auto-reload(500ms) — steady heartbeat
  *            LED2 (PB12): Auto-reload(variable) — period changes visibly
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "timers.h"

/* Private typedef -----------------------------------------------------------*/

/**
  * @brief  Timer callback argument.
  * @note   xTimerCreate's pvTimerID is stored in the timer object,
  *          accessible in the callback via pvTimerGetTimerID().
  *          We cast led_id (0/1/2) to void* to avoid a global lookup table.
  */
/* (no typedef needed — we cast uint8_t to void* directly) */

/* Private defines ------------------------------------------------------------*/

/* Timer periods (ms) */
#define ONESHOT_PERIOD_MS       3000U   /* One-shot: fires 3s after start      */
#define LED1_PERIOD_MS           500U   /* Auto-reload: toggles LED1 every 500ms */
#define LED2_PERIOD_INITIAL_MS   200U   /* Auto-reload: starts at 200ms         */
#define LED2_PERIOD_SLOW_MS     1000U   /* Auto-reload: changes to 1000ms later */

/* Control task timing (ms) */
#define CTRL_START_ONESHOT_MS   5000U   /* Start one-shot every 5 seconds       */
#define CTRL_CHANGE_PERIOD_MS   8000U   /* Change LED2 period every 8 seconds   */

/* Task priorities */
#define PRIO_CONTROL            2U
/* Timer Daemon Task priority = configTIMER_TASK_PRIORITY = 4 (configured) */

/* Task stack sizes (words) */
#define STACK_CONTROL           200U

/* LED pin definitions */
#define LED0_PORT               GPIOB
#define LED0_PIN                GPIO_PIN_13
#define LED1_PORT               GPIOA
#define LED1_PIN                GPIO_PIN_1
#define LED2_PORT               GPIOB
#define LED2_PIN                GPIO_PIN_12

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — timer handles in a single struct.
  */
static struct {
    TimerHandle_t xTimerOneShot;    /* One-shot: fires once, then stops        */
    TimerHandle_t xTimerLed1;       /* Auto-reload: 500ms periodic LED1 toggle */
    TimerHandle_t xTimerLed2;       /* Auto-reload: variable-period LED2 toggle*/
    uint8_t       led2_is_slow;     /* Tracks current LED2 period mode         */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);

/* --- Timer Callbacks (MUST NOT BLOCK) ------------------------------------- */
static void OneShotTimer_Callback(TimerHandle_t xTimer);
static void AutoReloadTimer_Callback(TimerHandle_t xTimer);

/* --- Control Task --------------------------------------------------------- */
static void Control_Task(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

/**
  * @brief  Creates 3 timers + 1 control task, then starts the scheduler.
  * @note   Timer creation ≠ timer start. Timers are created STOPPED.
  *          - OneShot: created, not started — Control task starts it later
  *          - Led1: created AND started immediately — runs forever
  *          - Led2: created AND started immediately — runs forever
  *
  *          xTimerCreate parameters:
  *            pcTimerName:  human-readable name (debug only, 10 chars max)
  *            xTimerPeriod: period in ticks (we use pdMS_TO_TICKS for ms)
  *            uxAutoReload: pdFALSE=one-shot, pdTRUE=auto-reload
  *            pvTimerID:    user context, accessible via pvTimerGetTimerID()
  *            pxCallback:   function called when timer expires
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Timer 1 — One-shot:
     *   pdFALSE = fires ONCE, then stops automatically.
     *   Must call xTimerStart() (or xTimerReset()) to fire again.
     *   pvTimerID = (void*)0 → callback knows this is LED0.
     */
    g.xTimerOneShot = xTimerCreate("Led0OneSht",
                                   pdMS_TO_TICKS(ONESHOT_PERIOD_MS),
                                   pdFALSE,          /* ← ONE-SHOT */
                                   (void *)0,        /* ← timer ID = 0 */
                                   OneShotTimer_Callback);
    configASSERT(g.xTimerOneShot != NULL);

    /*
     * Timer 2 — Auto-reload (LED1, 500ms):
     *   pdTRUE = fires repeatedly every 500ms until stopped.
     *   pvTimerID = (void*)1 → callback knows this is LED1.
     */
    g.xTimerLed1 = xTimerCreate("Led1Period",
                                pdMS_TO_TICKS(LED1_PERIOD_MS),
                                pdTRUE,           /* ← AUTO-RELOAD */
                                (void *)1,
                                AutoReloadTimer_Callback);
    configASSERT(g.xTimerLed1 != NULL);

    /*
     * Timer 3 — Auto-reload (LED2, starts at 200ms, changes later):
     *   Same callback as Timer 2 — pvTimerID distinguishes which LED.
     */
    g.xTimerLed2 = xTimerCreate("Led2VarSpd",
                                pdMS_TO_TICKS(LED2_PERIOD_INITIAL_MS),
                                pdTRUE,
                                (void *)2,
                                AutoReloadTimer_Callback);
    configASSERT(g.xTimerLed2 != NULL);

    /*
     * Start the auto-reload timers.
     * xTimerStart parameters:
     *   xTimer:    timer handle
     *   xBlockTime: how long to wait for the Daemon Task command queue
     *               0 = don't block (used here, safe at init time)
     *               portMAX_DELAY = block until command is queued
     *
     * OneShot is NOT started here — Control task does it later.
     */
    xReturned = xTimerStart(g.xTimerLed1, 0);
    configASSERT(xReturned == pdPASS);

    xReturned = xTimerStart(g.xTimerLed2, 0);
    configASSERT(xReturned == pdPASS);

    /* Create the control task */
    xReturned = xTaskCreate(Control_Task, "Control",
                            STACK_CONTROL, NULL, PRIO_CONTROL, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();  /* Never returns */
    while (1) {}
}

/* ========================================================================== */
/*  Timer Callbacks                                                            */
/* ========================================================================== */

/**
  * @brief  One-shot timer callback — toggles LED0 once, then the timer stops.
  * @note   KEY RULES for timer callbacks:
  *          1. MUST NOT BLOCK — no vTaskDelay, no xQueueReceive(timeout>0),
  *             no xSemaphoreTake(timeout>0)
  *          2. MUST be SHORT — long callbacks delay ALL other timers
  *             (they all share the same Daemon Task)
  *          3. CAN call xTimerStart/Stop/ChangePeriod/Reset
  *          4. CAN send to a queue with timeout=0 (non-blocking)
  *          5. CAN Give a semaphore (non-blocking)
  *          6. Runs in Daemon Task context, NOT interrupt context
  *
  *          If you need to do significant work:
  *            Callback → Give semaphore → wake up a worker task → do work
  *
  *   @param  xTimer: handle of the expired timer (useful if callback is shared)
  */
static void OneShotTimer_Callback(TimerHandle_t xTimer)
{
    /*
     * This is a one-shot timer, so it fires EXACTLY ONCE and then
     * transitions to the "dormant" state. xTimerStart() must be
     * called again (by Control task) to re-arm it.
     */

    /* Retrieve the LED ID we stored in pvTimerID during creation */
    uint8_t led_id = (uint8_t)(uint32_t)pvTimerGetTimerID(xTimer);

    /* Fast, non-blocking: toggle the LED */
    APP_LedToggle(led_id);

    /*
     * We could also call xTimerChangePeriod() or xTimerReset() here,
     * but since this is one-shot, xTimerReset() would restart it
     * from the current time — functionally same as xTimerStart().
     */
}

/**
  * @brief  Auto-reload timer callback — toggles LED1 or LED2.
  * @note   Shared by BOTH auto-reload timers.
  *          pvTimerID distinguishes which timer expired:
  *            (void*)1 → LED1 (500ms steady)
  *            (void*)2 → LED2 (variable speed)
  *
  *          Auto-reload timers automatically restart after the callback
  *          returns — no need to call xTimerStart() again.
  *          Contrast with one-shot: one-shot stops after callback.
  */
static void AutoReloadTimer_Callback(TimerHandle_t xTimer)
{
    uint8_t led_id = (uint8_t)(uint32_t)pvTimerGetTimerID(xTimer);

    /*
     * Direct GPIO toggle in a timer callback is FAST and NON-BLOCKING.
     * HAL_GPIO_TogglePin takes microseconds — perfectly safe here.
     *
     * DO NOT do this:
     *   vTaskDelay(100);            ← BLOCKS the Daemon Task!
     *   xQueueReceive(q, &d, 100);  ← BLOCKS the Daemon Task!
     *
     * If you need delays or blocking operations:
     *   Give a semaphore → worker task handles it
     */
    APP_LedToggle(led_id);
}

/* ========================================================================== */
/*  Control Task — Manages timers at runtime                                    */
/* ========================================================================== */

/**
  * @brief  Periodically starts the one-shot timer and changes LED2 period.
  * @note   Demonstrates runtime timer management:
  *
  *          Every 5 seconds: start the one-shot timer.
  *            → LED0 flashes once 3 seconds later (3000ms one-shot)
  *            → If the timer is already running, xTimerStart() has no effect
  *               (it won't restart the countdown). Use xTimerReset() for that.
  *
  *          Every 8 seconds: toggle LED2 timer period between 200ms and 1000ms.
  *            → LED2 speed visibly changes
  *            → xTimerChangePeriod() takes effect from the NEXT tick,
  *              not retroactively (won't change previously "missed" expiries)
  */
static void Control_Task(void *pvParameters)
{
    (void)pvParameters;

    TickType_t xLastWakeTime = xTaskGetTickCount();
    uint32_t cycle = 0;

    for (;;)
    {
        cycle++;

        /*
         * Phase 1: Start (or reset) the one-shot timer.
         * xTimerStart: if timer is dormant → starts it.
         *              if timer is already running → NO EFFECT.
         * xTimerReset: if timer is dormant → starts it.
         *              if timer is running → restarts countdown from zero.
         *
         * We use xTimerStart here — if the old one-shot hasn't fired yet,
         * we let it complete. Use xTimerReset if you want to "re-arm".
         */
        if ((cycle % 1) == 0)  /* Every cycle = 5s */
        {
            /*
             * 0 = non-blocking: Daemon Task queue has plenty of capacity
             * (configTIMER_QUEUE_LENGTH = 10). If the queue were full,
             * this would block or return pdFAIL.
             *
             * After this call: 3 seconds from NOW, OneShotTimer_Callback()
             * will fire and toggle LED0.
             */
            xTimerStart(g.xTimerOneShot, 0);
        }

        /*
         * Phase 2: Toggle LED2 timer period between fast (200ms) and slow (1000ms).
         * Runs every ~8 seconds (8s / 5s = 1.6 cycles, so we check every 2 cycles = 10s).
         * Actually, let's keep it simple: toggle every ~8 seconds.
         */
        if ((cycle % 2) == 0)  /* Every other cycle ≈ 10s... let me simplify */
        {
            /*
             * xTimerChangePeriod:
             *   - Changes the period of a running timer
             *   - New period takes effect AFTER the next expiry
             *   - If the old period was 200ms and new is 1000ms:
             *       current 200ms countdown continues, then resets to 1000ms
             *   - pdMS_TO_TICKS(0) is the second param: block time
             */
            TickType_t new_period;

            if (g.led2_is_slow)
            {
                new_period = pdMS_TO_TICKS(LED2_PERIOD_INITIAL_MS);
                g.led2_is_slow = 0;
            }
            else
            {
                new_period = pdMS_TO_TICKS(LED2_PERIOD_SLOW_MS);
                g.led2_is_slow = 1;
            }

            xTimerChangePeriod(g.xTimerLed2, new_period, 0);
        }

        /* Wait ~5 seconds between cycles */
        vTaskDelay(pdMS_TO_TICKS(CTRL_START_ONESHOT_MS));
    }
}

/* ========================================================================== */
/*  Hardware Helpers                                                          */
/* ========================================================================== */

static void APP_LedToggle(uint8_t led_id)
{
    switch (led_id)
    {
    case 0:
        HAL_GPIO_TogglePin(LED0_PORT, LED0_PIN);
        break;
    case 1:
        HAL_GPIO_TogglePin(LED1_PORT, LED1_PIN);
        break;
    case 2:
        HAL_GPIO_TogglePin(LED2_PORT, LED2_PIN);
        break;
    default:
        break;
    }
}

/* ========================================================================== */
/*  GPIO Configuration                                                        */
/* ========================================================================== */

static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    /* LED0 — One-shot demo */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* LED1 — Auto-reload 500ms heartbeat */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* LED2 — Variable period demo */
    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED2_PORT, LED2_PIN, GPIO_PIN_SET);
}

/* ========================================================================== */
/*  System Clock: HSI 8MHz, No PLL                                            */
/* ========================================================================== */

static void APP_SystemClockConfig(void)
{
    RCC_OscInitTypeDef OscInitstruct = {0};
    RCC_ClkInitTypeDef ClkInitstruct = {0};

    OscInitstruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
    OscInitstruct.HSIState       = RCC_HSI_ON;
    OscInitstruct.HSEState       = RCC_HSE_OFF;
    OscInitstruct.PLL.PLLState   = RCC_PLL_OFF;
    if (HAL_RCC_OscConfig(&OscInitstruct) != HAL_OK)
    {
        APP_ErrorHandler();
    }

    ClkInitstruct.ClockType      = RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK
                                 | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    ClkInitstruct.SYSCLKSource   = RCC_SYSCLKSOURCE_HSI;
    ClkInitstruct.AHBCLKDivider  = RCC_SYSCLK_DIV1;
    ClkInitstruct.APB1CLKDivider = RCC_HCLK_DIV1;
    ClkInitstruct.APB2CLKDivider = RCC_HCLK_DIV2;
    if (HAL_RCC_ClockConfig(&ClkInitstruct, FLASH_LATENCY_0) != HAL_OK)
    {
        APP_ErrorHandler();
    }
}

/* ========================================================================== */
/*  Hook Functions                                                            */
/* ========================================================================== */

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    __disable_irq();
    for (;;) {}
}

void APP_ErrorHandler(void)
{
    __disable_irq();
    for (;;) {}
}
```

---

## Timer API 速查

| API | 功能 | 备注 |
|-----|------|------|
| `xTimerCreate(name, period, reload, id, cb)` | 创建定时器 (返回 handle) | 创建后是**停止**状态 |
| `xTimerStart(timer, timeout)` | 启动定时器 | timeout=等待 Daemon 命令队列的 ticks |
| `xTimerStop(timer, timeout)` | 停止定时器 | 回调不再触发 |
| `xTimerReset(timer, timeout)` | 重置定时器 | 停止的→启动；运行的→重新计时 |
| `xTimerChangePeriod(timer, period, timeout)` | 改变周期 | 下次到期后才生效 |
| `xTimerDelete(timer, timeout)` | 删除定时器，释放内存 | 删除前确保不在回调中使用 |
| `xTimerIsTimerActive(timer)` | 查询是否在运行 | pdTRUE=运行中/dormant=停止 |
| `pvTimerGetTimerID(timer)` | 获取创建时设置的 ID | 回调中区分多个 timer |
| `xTimerGetPeriod(timer)` | 获取当前周期 (tick) | 调试用 |
| `xTimerGetExpiryTime(timer)` | 获取下次到期时间 (tick) | 调试用 |

---

## One-Shot vs Auto-Reload 状态机

```
┌──────────────────────────────────────────────────────────────────┐
│                    定时器生命周期                                  │
│                                                                  │
│  One-Shot (pdFALSE):                                            │
│                                                                  │
│    xTimerCreate ──→ [Dormant] ──xTimerStart──→ [Running]        │
│                              ◄──expires──        │              │
│                              ◄──xTimerStop──     │              │
│                              ◄──xTimerReset──    │              │
│                              ──xTimerReset──────→│ (重新计时)     │
│                                                                  │
│  Auto-Reload (pdTRUE):                                          │
│                                                                  │
│    xTimerCreate ──→ [Dormant] ──xTimerStart──→ [Running]        │
│                              ◄──xTimerStop──     │              │
│                                                   │              │
│                                     expires ──────┘              │
│                                     (自动重启)                     │
│                                                                  │
│  关键区别:                                                        │
│    One-shot 到期后 → 自动变 Dormant                               │
│    Auto-reload 到期后 → 保持 Running, 重新计时                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Timer Daemon Task 原理

```
┌──────────────────────────────────────────────────────────────────┐
│              软件定时器的底层实现                                   │
│                                                                  │
│  FreeRTOS 内部:                                                   │
│                                                                  │
│  Tick ISR (每次 tick)                                             │
│    │                                                             │
│    ├── 遍历定时器列表, 检查是否有到期                               │
│    │   到期 → 将 "执行回调" 命令写入 Daemon Task 的命令队列          │
│    │                                                             │
│    └── xPortSysTickHandler() → 可能触发任务切换                    │
│                                                                  │
│  Daemon Task (prio=configTIMER_TASK_PRIORITY):                   │
│    │                                                             │
│    ├── 等待命令队列有数据                                          │
│    │    (命令来源: tick ISR 写入的到期事件 + 任务调用的 API)         │
│    │                                                             │
│    ├── 收到命令 → 执行对应的回调函数                                │
│    │   回调执行期间, 其他所有定时器的到期事件在队列里排队             │
│    │   → 所以回调必须短!                                           │
│    │                                                             │
│    └── 处理下一条命令                                              │
│                                                                  │
│  结论:                                                            │
│    - 如果 Daemon Task 优先级低 → 高优先级任务会延迟定时器回调        │
│    - 如果一个回调执行了 100ms → 所有其他定时器也延迟 100ms           │
│    - 定时器精度 = tick 精度 + Daemon Task 调度延迟                  │
│    - 需要硬实时精度 → 用硬件定时器                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## FreeRTOSConfig.h 关键配置

本课依赖以下配置（第1课已配好，无需改动）：

```c
/* FreeRTOSConfig.h — 已存在的配置, 确认即可 */
#define configUSE_TIMERS                1           /* 必须为 1             */
#define configTIMER_TASK_PRIORITY       (configMAX_PRIORITIES - 1)  /* =4 */
#define configTIMER_TASK_STACK_DEPTH    configMINIMAL_STACK_SIZE     /* =128 */
#define configTIMER_QUEUE_LENGTH        10          /* Daemon 命令队列长度  */

/* main.c 需要 include */
#include "timers.h"   /* xTimerCreate / xTimerStart / ... */
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Timer Daemon Task       │   632B  │  1  │   632B   (128W stack + TCB)
Control Task            │   920B  │  1  │   920B   (200W stack + TCB)
Timer1 (One-shot)       │    68B  │  1  │    68B   (timer struct)
Timer2 (Auto-reload)    │    68B  │  1  │    68B
Timer3 (Auto-reload)    │    68B  │  1  │    68B
Timer Command Queue     │   120B  │  1  │   120B   (10 × 8B items + QCB)
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  2636B / 8192B  ✅
```

---

## 验证清单

```
□ 1. 添加 #include "timers.h" 到 main.c
□ 2. 编译 0 Error, 0 Warning
□ 3. 下载运行后观察:
□ 3a. LED1 (PA1) — 持续匀速 500ms 闪烁 (心跳)
      → 证明: Auto-reload timer 正常运行
□ 3b. LED2 (PB12) — 闪烁速度周期性切换 (快→慢→快→慢...)
      → 证明: xTimerChangePeriod 成功改变周期
□ 3c. LED0 (PB13) — 每隔 ~5s 闪一次 (持续 3s 后灭)
      → 证明: One-shot timer 被 Control task 周期性启动
□ 4.  在 AutoReloadTimer_Callback 设断点:
      观察 pvTimerGetTimerID(xTimer) 返回值 (1 或 2)
□ 5.  在 Control_Task 的 xTimerStart 设断点:
      单步确认 0=pdPASS (命令成功入队)
□ 6.  Keil Watch 窗口添加:
      xTimerIsTimerActive(g.xTimerOneShot) → 0/1
      xTimerGetPeriod(g.xTimerLed2)       → 200 或 1000 ticks
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|----------|
| 回调中阻塞 | vTaskDelay / xQueueReceive(timeout>0) 在回调中 → Daemon 卡死 | 回调只做轻量操作, 用信号量通知 worker task |
| 回调太长 | 100ms 回调 → 所有其他定时器延迟 100ms | 回调控制在 μs 级别 |
| One-Shot 忘记重新 Start | 到期后自动停止, 不再触发 | 需要重复触发用 Auto-reload, 或在回调中 Reset |
| xTimerStart 在 ISR | 没有 FromISR 版本的 xTimerStart | 用 xTimerStartFromISR() |
| 删除正在运行的定时器 | xTimerDelete 后回调可能还在执行 | 删除前 xTimerStop, 确认不在回调中 |
| 定时器周期 < tick 周期 | 如果 tick=10ms, 创建 3ms 定时器 → 实际 10ms | 最小周期 = 1 tick = 1/configTICK_RATE_HZ |
| pvTimerID 传栈变量 | 栈变量出作用域后 ID 失效 | 传常量/全局变量/或直接用枚举值转 void* |
