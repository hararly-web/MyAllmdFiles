# 第7课 代码实战 — 基础版：任务通知 (Give/Take + Notify/Wait + 性能对比)

> **目标：** 掌握 xTaskNotifyGive/ulTaskNotifyTake（轻量同步）和 xTaskNotify/xTaskNotifyWait（带值通知），通过性能对比理解任务通知的速度优势
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第6课事件组已完成，理解阻塞等待和 IPC 选型
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（已配好）

---

## 核心概念：任务通知 = 每个任务内嵌的 32 位邮箱

```
┌──────────────────────────────────────────────────────────────────┐
│          任务通知 vs 信号量 vs 队列                                │
│                                                                  │
│  信号量 (Semaphore):                                              │
│    需创建内核对象 (SemaphoreHandle_t)                             │
│    Give = 释放, Take = 消耗                                      │
│    ~60B RAM 开销, 可多播                                         │
│                                                                  │
│  队列 (Queue):                                                   │
│    需创建内核对象 (QueueHandle_t)                                 │
│    Send → Queue → Receive, 数据按值拷贝                          │
│    ~100B+ RAM 开销, 有缓冲, 可多播                               │
│                                                                  │
│  任务通知 (Task Notification):   ← 本课                           │
│    无需创建! 每个任务的 TCB 自带了通知状态和 32 位通知值          │
│    0 额外 RAM 开销 — 发送方直接写入接收方 TCB 中                 │
│    xTaskNotifyGive/ulTaskNotifyTake = 轻量二值同步               │
│    xTaskNotify/xTaskNotifyWait      = 带 32 位数据的通知          │
│    限制: 单播、无缓冲、发送方不阻塞                               │
│                                                                  │
│    每个 TCB 内嵌:                                                 │
│    ┌────────────────────────────────────────────────┐            │
│    │ ulNotifiedValue  (uint32_t) — 通知值          │            │
│    │ ucNotifyState    (enum)    — Not/Pending/Waiting│          │
│    └────────────────────────────────────────────────┘            │
│                                                                  │
│  关键特性:                                                        │
│    - 最快 IPC 机制 (约 1.7× 快于信号量, 3× 快于队列)              │
│    - 通知值是 32-bit, 可打包任意小型数据                          │
│    - 5 种 eAction 模式: NoAction/SetBits/Increment/Overwrite/    │
│      WithoutOverwrite                                            │
│    - 发送方永远不阻塞 (立即返回)                                  │
│    - ISR 可以发送通知, 不能接收通知                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 程序架构

```
Demo 1: 轻量同步 (Give/Take)
  NotifyGiveTaker_Task  — 自通知自取，验证 Pending 机制

Demo 2: 带值通知 (Notify/Wait)
  NotifySender_Task     — 打包 LED 命令到 32-bit → Notify
  NotifyReceiver_Task   — Wait 通知 → 解包 → 执行 LED 闪烁

Demo 3: 性能对比
  PerfCompare_Task      — 10000 次 Give/Take 对比 Semaphore vs TaskNotify
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 7 Basic: Task Notifications
  *
  *          Demonstrates:
  *          - xTaskNotifyGive() / ulTaskNotifyTake()  — lightweight sync
  *          - xTaskNotify() / xTaskNotifyWait()       — value-carrying notify
  *          - Performance comparison: TaskNotify vs Semaphore vs Queue
  *          - Efficient 32-bit value packing for LED commands
  *
  *          Architecture:
  *
  *            [Demo 1: Give/Take — Lightweight Binary Sync]
  *              NotifyGiveTaker sends notification to itself,
  *              then attempts 3 Takes — only the 1st succeeds.
  *              Proves: notification is NOT buffered.
  *
  *            [Demo 2: Notify/Wait — Value-Carrying Notification]
  *              Sender packs {led_id, delay_ms} into uint32_t,
  *              sends via xTaskNotify(eSetValueWithOverwrite).
  *              Receiver blocks in xTaskNotifyWait, unpacks value,
  *              and executes LED toggle sequences.
  *
  *            [Demo 3: PerfCompare]
  *              Cycles 10000 Give/Take pairs through Semaphore
  *              and TaskNotify. LED1 blink rate indicates semaphore
  *              performance; LED2 indicates TaskNotify performance.
  *
  *          LED assignment:
  *            LED0 (PB13): Demo1 Give/Take — toggle on successful notification
  *            LED1 (PA1):  Demo2 Receiver — toggle on received LED command
  *            LED2 (PB12): PerfCompare — fast blink = TaskNotify test running
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"        /* Only for PerfCompare demo — not needed otherwise */

/* Private defines ------------------------------------------------------------*/

/* LED pin definitions */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12

/*
 * Pack/unpack macros for value-carrying notification.
 * Upper 16-bit = LED id (0=LED0, 1=LED1, 2=LED2)
 * Lower 16-bit = delay in ms (50 ~ 999)
 */
#define NOTIFY_PACK(led_id, delay_ms)  (((uint32_t)((led_id) & 0xFF) << 16) | \
                                        ((uint32_t)(delay_ms) & 0xFFFF))
#define NOTIFY_GET_LED(val)            ((uint8_t)(((val) >> 16) & 0xFF))
#define NOTIFY_GET_DELAY(val)          ((uint16_t)((val) & 0xFFFF))

/* Timing (ms) */
#define GIVETAKE_PERIOD_MS   1000U    /* Demo1: send notify every 1s          */
#define TAKE_TIMEOUT_MS      200U     /* Demo1: take timeout 200ms            */
#define SENDER_PERIOD_MS     600U     /* Demo2: sender interval               */
#define PERF_PERIOD_MS       5000U    /* Demo3: benchmark interval            */
#define PERF_ITERATIONS      10000UL  /* Demo3: iterations per benchmark      */

/* Task priorities */
#define PRIO_RECEIVER        3U       /* Receiver processes notifications ASAP */
#define PRIO_SENDER          2U       /* Sender at medium priority             */
#define PRIO_GIVETAKER       2U       /* GiveTaker at medium priority          */
#define PRIO_PERF            1U       /* Benchmark at low priority             */

/* Task stack sizes (words) */
#define STACK_RECEIVER       200U
#define STACK_SENDER         150U
#define STACK_GIVETAKER      150U
#define STACK_PERF           250U

/* LED command sequence for Demo 2 (8 commands, cycled) */
#define CMD_SEQ_LEN          8U

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — all shared state in one struct.
  */
static struct {
    TaskHandle_t xReceiverHandle;  /* Target for value-carrying notifications */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);
static void APP_LedOn(uint8_t led_id);
static void APP_LedOff(uint8_t led_id);

/* --- Demo 1: Give/Take Lightweight Sync ----------------------------------- */
static void NotifyGiveTaker_Task(void *pvParameters);

/* --- Demo 2: Value-Carrying Notify/Wait ----------------------------------- */
static void NotifySender_Task(void *pvParameters);
static void NotifyReceiver_Task(void *pvParameters);

/* --- Demo 3: Performance Comparison --------------------------------------- */
static void PerfCompare_Task(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

/**
  * @brief  Creates 4 tasks demonstrating 3 task-notification patterns.
  * @note   No kernel objects are created for Demo 1 and Demo 2 —
  *          task notifications use zero extra heap memory.
  *          Only Demo 3 creates a temporary semaphore for comparison.
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Demo 2: Create receiver FIRST so sender can reference its handle.
     * Save the handle in g.xReceiverHandle for the sender to use.
     */
    xReturned = xTaskCreate(NotifyReceiver_Task, "Receiver",
                            STACK_RECEIVER, NULL, PRIO_RECEIVER,
                            &g.xReceiverHandle);
    configASSERT(xReturned == pdPASS);

    /* Demo 2: Create sender — uses g.xReceiverHandle to send notifications */
    xReturned = xTaskCreate(NotifySender_Task, "Sender",
                            STACK_SENDER, NULL, PRIO_SENDER, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 1: Self-notify with Give/Take */
    xReturned = xTaskCreate(NotifyGiveTaker_Task, "GiveTaker",
                            STACK_GIVETAKER, NULL, PRIO_GIVETAKER, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 3: Performance comparison */
    xReturned = xTaskCreate(PerfCompare_Task, "PerfCmp",
                            STACK_PERF, NULL, PRIO_PERF, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();
    while (1) {}
}

/* ========================================================================== */
/*  Demo 1: Give/Take — Lightweight Binary Sync                                 */
/* ========================================================================== */

/**
  * @brief  Sends notification to itself, then attempts 3 Takes.
  *
  * @note   Key learning points:
  *         - xTaskNotifyGive() increments ulNotifiedValue and sets state=Pending.
  *         - ulTaskNotifyTake(pdTRUE, timeout):
  *             pdTRUE  = reset count to 0 on exit (binary semaphore behavior)
  *             pdFALSE = decrement count by 1  (counting semaphore behavior)
  *           Returns: the count BEFORE decrement (not 0/1 but the actual count).
  *
  *         - Only ONE notification is sent, so only the 1st Take succeeds.
  *           2nd and 3rd Takes timeout (return 0).
  *         - This proves: notifications are NOT buffered — unlike a queue,
  *           there's no "queue of notifications" waiting.
  *
  *         Expected LED0 behavior:
  *           Every 1s: 1 fast toggle (100ms on, 900ms off) = notification received.
  *           No extra toggles from timeout Takes.
  */
static void NotifyGiveTaker_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Step 1: Send one notification to self.
         * Internally: ulNotifiedValue++, ucNotifyState = eNotified.
         */
        xTaskNotifyGive(xTaskGetCurrentTaskHandle());

        /*
         * Step 2: Try 3 Takes — only the 1st will get the notification.
         * The 2nd and 3rd will time out after 200ms each.
         */
        for (uint8_t i = 0; i < 3; i++)
        {
            /*
             * ulTaskNotifyTake(pdTRUE, timeout):
             *   pdTRUE: clear the notification value to 0 on exit.
             *   Returns: the task's notification value BEFORE clearing.
             *
             *   If a notification arrived → returns N (count before clear).
             *   If timeout → returns 0.
             */
            uint32_t ulCount = ulTaskNotifyTake(pdTRUE,
                                                pdMS_TO_TICKS(TAKE_TIMEOUT_MS));

            if (ulCount > 0)
            {
                /* Notification received! Toggle LED0 once. */
                APP_LedToggle(0);
            }
            /*
             * else: timeout — no notification. LED0 stays unchanged.
             * This happens on the 2nd and 3rd iterations of the inner loop.
             */
        }

        /* Wait before next round */
        vTaskDelay(pdMS_TO_TICKS(GIVETAKE_PERIOD_MS));
    }
}

/* ========================================================================== */
/*  Demo 2: Value-Carrying Notify/Wait                                          */
/* ========================================================================== */

/**
  * @brief  Sends packed LED commands to the Receiver via xTaskNotify.
  *
  * @note   xTaskNotify(xTaskToNotify, ulValue, eAction, pulPreviousNotifyValue):
  *         - xTaskToNotify:  handle of the receiving task
  *         - ulValue:        32-bit value to send (application-defined packing)
  *         - eAction:        eSetValueWithOverwrite = always update the value
  *         - pulPreviousNotifyValue: optional pointer to receive previous value
  *                                 (pass NULL to ignore)
  *
  *         The Sender packs {led_id, delay_ms} into a single uint32_t:
  *            bits 31..16 = led_id (0~2)
  *            bits 15..0  = delay_ms (50~999)
  *
  *         The Sender cycles through 8 predefined commands and sends one
  *         every SENDER_PERIOD_MS. If the Receiver hasn't processed the
  *         previous notification yet, eSetValueWithOverwrite silently
  *         replaces it.
  */
static void NotifySender_Task(void *pvParameters)
{
    (void)pvParameters;

    /*
     * Predefined command sequence: {led_id, delay_ms}
     * Each entry produces a distinct toggle pattern.
     */
    static const struct {
        uint8_t  led_id;
        uint16_t delay_ms;
    } cmd_seq[CMD_SEQ_LEN] = {
        {0, 100},   /* LED0 toggle, 100ms → fast blink, 5Hz         */
        {1, 100},   /* LED1 toggle, 100ms → fast blink, 5Hz         */
        {0, 300},   /* LED0 toggle, 300ms → medium blink, ~1.7Hz    */
        {1, 300},   /* LED1 toggle, 300ms → medium blink, ~1.7Hz    */
        {0, 100},   /* LED0 toggle, 100ms → back to fast            */
        {2, 100},   /* LED2 toggle, 100ms → fast blink, 5Hz         */
        {2, 500},   /* LED2 toggle, 500ms → slow blink, 1Hz         */
        {2, 100},   /* LED2 toggle, 100ms → back to fast            */
    };

    uint8_t idx = 0;

    for (;;)
    {
        uint32_t ulNotifyValue;

        /* Pack LED id and delay into 32-bit notification value */
        ulNotifyValue = NOTIFY_PACK(cmd_seq[idx].led_id,
                                    cmd_seq[idx].delay_ms);

        /*
         * Send notification to the Receiver task.
         * eSetValueWithOverwrite: always update — if Receiver hasn't
         * consumed the previous notification, it gets overwritten.
         * Returns immediately (never blocks).
         *
         * For loss-sensitive data, check if previous value was lost:
         *   uint32_t ulPrev;
         *   BaseType_t xResult = xTaskNotify(g.xReceiverHandle,
         *       ulNotifyValue, eSetValueWithOverwrite, &ulPrev);
         *   if (xResult == pdPASS && ulPrev != 0) { /* overwrote old */ }
         */
        (void)xTaskNotify(g.xReceiverHandle,
                          ulNotifyValue,
                          eSetValueWithOverwrite,
                          NULL);

        /* Advance index, wrapping around */
        idx = (idx + 1) % CMD_SEQ_LEN;

        vTaskDelay(pdMS_TO_TICKS(SENDER_PERIOD_MS));
    }
}

/**
  * @brief  Blocks waiting for notifications, unpacks and executes LED commands.
  *
  * @note   xTaskNotifyWait(ulBitsToClearOnEntry, ulBitsToClearOnExit,
  *                         pulNotificationValue, xTicksToWait):
  *
  *         - ulBitsToClearOnEntry: bits to clear BEFORE waiting (usually 0)
  *         - ulBitsToClearOnExit:  bits to clear AFTER receiving (usually 0)
  *         - pulNotificationValue: receives the notification value
  *         - xTicksToWait: timeout (portMAX_DELAY = wait forever)
  *
  *         Returns: pdTRUE if notification received, pdFALSE if timeout.
  *
  *         This is the most flexible notification receive API —
  *         it can receive values sent by xTaskNotify() with any eAction.
  *
  *         In this demo, we use the value to determine:
  *         - Which LED to toggle (NOTIFY_GET_LED)
  *         - How long to delay between toggles (NOTIFY_GET_DELAY)
  */
static void NotifyReceiver_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        uint32_t ulNotifiedValue;

        /*
         * Block indefinitely until a notification arrives.
         * On return, ulNotifiedValue holds the value sent by the Sender.
         */
        BaseType_t xResult = xTaskNotifyWait(
                                 0UL,              /* No bits cleared on entry         */
                                 0UL,              /* No bits cleared on exit          */
                                 &ulNotifiedValue, /* Receives the 32-bit notify value */
                                 portMAX_DELAY);   /* Block forever                   */

        if (xResult == pdPASS)
        {
            /* Unpack notification value into LED id and delay */
            uint8_t  led_id   = NOTIFY_GET_LED(ulNotifiedValue);
            uint16_t delay_ms = NOTIFY_GET_DELAY(ulNotifiedValue);

            /*
             * Execute the LED toggle.
             * During vTaskDelay, the Receiver cannot accept new notifications.
             * The Sender may overwrite the notification value — this is
             * a design trade-off: simplicity vs. guaranteed delivery.
             */
            APP_LedToggle(led_id);
            vTaskDelay(pdMS_TO_TICKS(delay_ms));
        }
    }
}

/* ========================================================================== */
/*  Demo 3: Performance Comparison — TaskNotify vs Semaphore                    */
/* ========================================================================== */

/**
  * @brief  Benchmarks TaskNotify vs Semaphore by running 10,000 Give/Take pairs.
  *
  * @note   Methodology:
  *         1. Create a temporary binary semaphore.
  *         2. Run 10000 Give/Take pairs on the semaphore.
  *         3. Delete the semaphore.
  *         4. Run 10000 Give/Take pairs with TaskNotify.
  *         5. Toggle LEDs to provide visual comparison:
  *            - Semaphore test: LED1 flashes (slower cadence)
  *            - TaskNotify test: LED2 flashes (faster cadence)
  *
  *         In a real production benchmark you would use a hardware timer
  *         or DWT cycle counter for precise measurement. Here we use
  *         LED blink frequency as a rough visual proxy.
  *
  *         Expected result: TaskNotify runs noticeably faster.
  *
  *         Why TaskNotify is faster:
  *         - No kernel object creation/destruction overhead
  *         - No critical section entry/exit (notification operates
  *           directly on TCB fields)
  *         - No queue data copy (memcpy)
  *         - No separate object validity checks
  */
static void PerfCompare_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Phase 1: Benchmark binary semaphore.
         * Create a temporary semaphore for this iteration.
         */
        SemaphoreHandle_t xSemBench = xSemaphoreCreateBinary();
        if (xSemBench == NULL)
        {
            /* Heap exhausted — skip this round */
            vTaskDelay(pdMS_TO_TICKS(PERF_PERIOD_MS));
            continue;
        }

        APP_LedOn(1);                     /* LED1 ON  = sem test starting  */

        for (uint32_t i = 0; i < PERF_ITERATIONS; i++)
        {
            /*
             * For fair comparison: the "consumer" is this task itself.
             * Since semaphore Give/Take doesn't cross task boundaries
             * when done on same task, the comparison measures raw API
             * overhead — not IPC latency (which would require 2 tasks).
             *
             * Production note: a real IPC benchmark uses 2 tasks —
             * one producer, one consumer — and measures round-trip time.
             */
            xSemaphoreGive(xSemBench);
            xSemaphoreTake(xSemBench, portMAX_DELAY);

            /* Occasional LED toggle for visual feedback */
            if ((i & 0x1FF) == 0)          /* every 512 iterations */
            {
                APP_LedToggle(1);
            }
        }

        APP_LedOff(1);                    /* LED1 OFF = sem test done     */
        vSemaphoreDelete(xSemBench);

        /*
         * Phase 2: Benchmark TaskNotify.
         * Uses the task's built-in notification — no external object.
         */
        APP_LedOn(2);                     /* LED2 ON  = notify test starting */

        /*
         * Clear any stale notification before starting.
         * ulTaskNotifyTake(pdTRUE, 0): take with zero timeout,
         * returns immediately — if pending notification exists, consume it.
         */
        (void)ulTaskNotifyTake(pdTRUE, 0);

        for (uint32_t i = 0; i < PERF_ITERATIONS; i++)
        {
            /*
             * Give notification to self, then take it back.
             * Same "single-task" methodology as the semaphore phase.
             *
             * xTaskNotifyGive(xTaskGetCurrentTaskHandle()):
             *   Increments ulNotifiedValue, sets state=eNotified.
             *   Internally calls xTaskGenericNotify(..., eIncrement, ...).
             *
             * ulTaskNotifyTake(pdTRUE, portMAX_DELAY):
             *   pdTRUE: reset count to 0 after reading.
             *   Returns the count before clearing.
             */
            xTaskNotifyGive(xTaskGetCurrentTaskHandle());
            (void)ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

            /* Occasional LED toggle for visual feedback */
            if ((i & 0x1FF) == 0)          /* every 512 iterations */
            {
                APP_LedToggle(2);
            }
        }

        APP_LedOff(2);                    /* LED2 OFF = notify test done  */

        /*
         * Visual indicator: LED0 long flash = benchmark round complete.
         * Compare LED1 and LED2 blink speeds during the test phases —
         * LED2 (TaskNotify) should blink faster as the API is leaner.
         */
        APP_LedOn(0);
        vTaskDelay(pdMS_TO_TICKS(200));
        APP_LedOff(0);

        /* Wait before next benchmark round */
        vTaskDelay(pdMS_TO_TICKS(PERF_PERIOD_MS));
    }
}

/* ========================================================================== */
/*  Hardware Helpers                                                          */
/* ========================================================================== */

/**
  * @brief  Toggle LED by index.
  * @param  led_id: 0=LED0(PB13), 1=LED1(PA1), 2=LED2(PB12)
  */
static void APP_LedToggle(uint8_t led_id)
{
    switch (led_id)
    {
    case 0: HAL_GPIO_TogglePin(LED0_PORT, LED0_PIN); break;
    case 1: HAL_GPIO_TogglePin(LED1_PORT, LED1_PIN); break;
    case 2: HAL_GPIO_TogglePin(LED2_PORT, LED2_PIN); break;
    default: break;
    }
}

/**
  * @brief  Turn LED ON by index.
  * @note   LED is active-low on PY32F403-STK: GPIO_PIN_RESET = ON.
  * @param  led_id: 0=LED0(PB13), 1=LED1(PA1), 2=LED2(PB12)
  */
static void APP_LedOn(uint8_t led_id)
{
    switch (led_id)
    {
    case 0: HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_RESET); break;
    case 1: HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_RESET); break;
    case 2: HAL_GPIO_WritePin(LED2_PORT, LED2_PIN, GPIO_PIN_RESET); break;
    default: break;
    }
}

/**
  * @brief  Turn LED OFF by index.
  * @note   LED is active-low on PY32F403-STK: GPIO_PIN_SET = OFF.
  * @param  led_id: 0=LED0(PB13), 1=LED1(PA1), 2=LED2(PB12)
  */
static void APP_LedOff(uint8_t led_id)
{
    switch (led_id)
    {
    case 0: HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET); break;
    case 1: HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET); break;
    case 2: HAL_GPIO_WritePin(LED2_PORT, LED2_PIN, GPIO_PIN_SET); break;
    default: break;
    }
}

/* ========================================================================== */
/*  GPIO Configuration                                                        */
/* ========================================================================== */

/**
  * @brief  Initialize 3 LEDs as push-pull outputs with pull-up.
  * @note   LED pin mapping:
  *           LED0 = PB13, LED1 = PA1, LED2 = PB12
  *         All LEDs are active-low: write 0 = ON, write 1 = OFF.
  *         Initial state: all OFF (GPIO_PIN_SET).
  */
static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    /* LED0 — Demo1 Give/Take indicator */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* LED1 — Demo2 Receiver / Demo3 Semaphore */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* LED2 — Demo3 TaskNotify */
    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED2_PORT, LED2_PIN, GPIO_PIN_SET);
}

/* ========================================================================== */
/*  System Clock: HSI 8MHz, No PLL                                            */
/* ========================================================================== */

/**
  * @brief  System Clock Configuration — HSI 8MHz.
  * @note   Must match configCPU_CLOCK_HZ in FreeRTOSConfig.h.
  */
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

## Task Notify API 速查

| API | 功能 | 关键参数 / 返回值 |
|-----|------|-------------------|
| `xTaskNotifyGive(xTask)` | 通知值+1 (轻量Give) | 任务句柄; 返回 pdPASS |
| `vTaskNotifyGiveFromISR(xTask, pxFlag)` | ISR中通知值+1 | `pxHigherPriorityTaskWoken` |
| `ulTaskNotifyTake(xClear, timeout)` | 取通知值 (调用者自身) | `pdTRUE`=清零, `pdFALSE`=-1; 返回清除前的计数值 |
| `xTaskNotify(xTask, val, eAction, pPrev)` | 通用通知发送 | 5种 eAction; pPrev可获取被覆盖的旧值(NULL=忽略) |
| `xTaskNotifyFromISR(...)` | ISR中通用通知发送 | 多一个 `pxHigherPriorityTaskWoken` 参数 |
| `xTaskNotifyWait(clearIn, clearOut, pVal, timeout)` | 通用通知等待 | pVal 接收通知值; 可指定进入/退出时的位清除 |
| `xTaskNotifyStateClear(xTask)` | 清除通知状态 (不取通知值) | 将状态从 Pending 改为 Not Pending; 返回值: pdPASS/pdFAIL |
| `xTaskNotifyAndQuery(xTask, val, eAction, pPrev)` | 发送通知+查询旧值 | 多一个 `pulPreviousNotifyValue` 参数 vs xTaskNotify |

---

## ulTaskNotifyTake 的 xClearOnExit 参数详解

```
┌──────────────────────────────────────────────────────────────────┐
│  ulTaskNotifyTake(pdTRUE, timeout)   — "二值信号量模式"           │
│    进入时: 通知值 = N                                              │
│    返回:   N (进入时的值)                                         │
│    退出时: 通知值 = 0  (清零!)                                    │
│                                                                  │
│    相当于: count = sem->count; sem->count = 0; return count;     │
│    用途:   "你通知了我多少次?" — 取走所有通知，状态复位            │
│                                                                  │
│  ulTaskNotifyTake(pdFALSE, timeout)  — "计数信号量模式"           │
│    进入时: 通知值 = N                                              │
│    返回:   N > 0 ? pdPASS : pdFAIL                                │
│    退出时: 通知值 = N - 1  (减1!)                                 │
│                                                                  │
│    相当于: if(sem->count > 0) { sem->count--; return pdPASS; }   │
│    用途:   "消耗一个通知" — 类似 xSemaphoreTake                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ ⚠️ 常见误解:                                              │    │
│  │                                                         │    │
│  │ ulTaskNotifyTake() 的返回值 ≠ 成功/失败                   │    │
│  │                                                         │    │
│  │ 返回的是: 进入函数时通知值的值 (清除前的计数值)          │    │
│  │                                                         │    │
│  │  返回值 > 0  → 有通知                                    │    │
│  │  返回值 = 0  → 无通知 (超时)                              │    │
│  │                                                         │    │
│  │ 判断方法:                                                │    │
│  │  if (ulTaskNotifyTake(pdTRUE, timeout) > 0) { ... }     │    │
│  │  ✓ 正确 — 计数>0 表示收到了通知                          │    │
│  │                                                         │    │
│  │  if (ulTaskNotifyTake(pdTRUE, timeout) == pdPASS) {     │    │
│  │  ✗ 可能导致误判 — 如果通知值为2, 返回2, 2≠1(pdPASS)    │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

---

## eAction 快速参考

```
┌─────────────────────────┬────────────────────────────────────────────┐
│ eAction                 │ 行为                                       │
├─────────────────────────┼────────────────────────────────────────────┤
│ eNoAction               │ 仅通知状态→Pending，不修改通知值            │
│ eSetBits                │ 通知值 |= ulValue (OR accumulating)        │
│ eIncrement              │ 通知值++ (count mode, same as Give)        │
│ eSetValueWithOverwrite  │ 通知值 = ulValue (always overwrite)        │
│ eSetValueWithoutOverwrite│ 通知值 = ulValue ONLY if not pending      │
│                         │ (returns pdFAIL if already pending)        │
└─────────────────────────┴────────────────────────────────────────────┘

使用建议:
  • 纯同步(无数据)  → xTaskNotifyGive() / ulTaskNotifyTake()
  • 单值覆盖(最新)  → eSetValueWithOverwrite (如传感器最新值)
  • 位事件累积      → eSetBits (如多状态标志)
  • 计数通知        → eIncrement (如按钮按下次数)
  • 仅通知不写值    → eNoAction (如"唤醒接收方但不传递数据")
```

---

## FreeRTOSConfig.h 配置

```c
/*
 * Task notifications are ALWAYS available — no config macro needed!
 * Unlike configUSE_TIMERS, configUSE_EVENT_GROUPS, etc.,
 * task notification is part of the core task module (tasks.c).
 *
 * The only relevant config is for the FromISR helper:
 */
#define configUSE_TASK_NOTIFICATIONS    1   /* Default = 1, always on */

/*
 * If you want to disable all task notification APIs to save code size,
 * set this to 0. Not recommended for normal use.
 */
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Daemon Task             │   632B  │  1  │   632B
Receiver Task           │   920B  │  1  │   920B   (200W stack + TCB)
Sender Task             │   720B  │  1  │   720B   (150W stack + TCB)
GiveTaker Task          │   720B  │  1  │   720B   (150W stack + TCB)
PerfCompare Task        │  1120B  │  1  │  1120B   (250W stack + TCB)
Semaphore (temporary)   │    60B  │  1  │    60B   (perf test only)
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  4932B / 8192B  ✅
FREE                    │         │     │  3260B

Note: Zero additional heap for notifications themselves —
      they use embedded TCB fields at no extra RAM cost.
```

---

## 验证清单

```
□ 1. 确认 #include "FreeRTOS.h" 和 #include "task.h" (无需额外头文件!)
□ 2. 编译 0 Error, 0 Warning
□ 3. 下载运行后观察:
□ 3a. LED0 (PB13) — 每1秒短暂闪烁1次（仅第1次Take成功）
      → 证明: Give/Take 正常，通知无缓冲（只有1个）
□ 3b. LED1 (PA1)  — 不定间隔闪烁（受通知值中的delay_ms控制）
      → 证明: xTaskNotify/xTaskNotifyWait 带值通知正常工作
□ 3c. LED2 (PB12) — 在性能测试阶段快闪（比LED1在测试阶段的闪烁更快）
      → 证明: TaskNotify 确实比 Semaphore 快
□ 4.  在 NotifyGiveTaker 的 ulTaskNotifyTake 处设断点:
      观察 ulCount — 第1次=1, 第2次=0, 第3次=0
□ 5.  在 NotifyReceiver 的 xTaskNotifyWait 处设断点:
      检查 ulNotifiedValue — 高位=led_id, 低位=delay_ms
□ 6.  在 PerfCompare 中分别设断点观察两个循环:
      肉眼观察 LED1 vs LED2 闪烁频率差异
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|---------|
| 忘记通知无缓冲 | 连续给同一任务发多个通知 → 后面的会覆盖/丢失 | 用队列（Queue）替代 |
| ulTaskNotifyTake 返回值误判 | 返回值 > 0 才表示有通知，不是 == pdPASS | `if (ulTaskNotifyTake(...) > 0)` |
| eSetValueWithoutOverwrite 丢弃 | 接收方未取走上次通知 → 新的通知被丢弃 | 检查返回值，或用 eSetValueWithOverwrite |
| 多播需求用通知 | 一个通知只能给一个任务 | 用事件组或信号量 |
| ISR 中 xTaskNotifyWait | ISR 不能阻塞！ | 只在任务中调用 xTaskNotifyWait |
| 误用通知替代队列 | 需要缓冲多条消息 → 通知做不到 | 通知: 单值; 队列: 缓冲 |
| pdTRUE/pdFALSE 搞反 | ulTaskNotifyTake(pdTRUE) 清零, pdFALSE 减1 | 记: TRUE=清零(二值), FALSE=减1(计数) |

---

## 最佳实践清单

| # | 实践 | 说明 |
|---|------|------|
| 1 | 纯同步用 Give/Take | 零开销, 0 RAM, 最快路径 |
| 2 | 带数据用 Notify/Wait | 32位可打包小型命令/状态, 省去队列开销 |
| 3 | `notify_value > 0` 判断 | ulTaskNotifyTake 的返回值不是 pdPASS/pdFAIL |
| 4 | 最新值优先 → eSetValueWithOverwrite | 传感器、状态更新等场景 |
| 5 | 需要缓冲 → 改用队列 | 不硬上通知, 数据丢失代价大 |
| 6 | 需要多播 → 改用信号量/事件组 | 通知只能单播 |
| 7 | 打包通知值用宏 | `NOTIFY_PACK()/NOTIFY_GET_*()` 清晰且无运行时开销 |
| 8 | 全局变量用结构体封装 | 任务句柄统一管理在 `g` 结构体中 |

---

## 通知 vs 队列 vs 信号量 最终选型决策树

```
  需要传递数据?
    ├── Yes → 数据需要缓冲?
    │         ├── Yes → ✅ 队列 (Queue)
    │         └── No  → ✅ xTaskNotify() — 带值通知
    │
    └── No, 纯同步 → 需要通知多个任务?
                    ├── Yes → ✅ 信号量 (Semaphore) / 事件组 (Event Group)
                    └── No  → ✅ xTaskNotifyGive/ulTaskNotifyTake
                              (最快, 0 额外 RAM, 唯一限制: 单播)

  决策规则:
    • 只有一个接收方 → 优先考虑任务通知 (性能/内存最优)
    • 多个接收方     → 信号量或事件组
    • 需要缓冲消息   → 队列
    • 不确定         → 从队列开始 (最灵活), 后续根据性能数据优化
```
