# 第4课 代码实战 — 基础版：二值信号量 + 计数信号量 + 互斥锁

> **目标：** 掌握 xSemaphoreCreateBinary / xSemaphoreCreateCounting / xSemaphoreCreateMutex 的基本用法，理解三种信号量的本质区别
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第3课队列通信已完成，理解任务阻塞和优先级抢占
> **Heap：** 建议 `configTOTAL_HEAP_SIZE` 设为 6144 (6KB)，因为本课创建了7个任务+3个信号量

---

## 代码流程图

```
main()
  │
  ├── HAL_Init() / SystemClock / GPIO      ← 硬件初始化
  │
  ├── xSemaphoreCreateBinary()             ← 二值信号量 (初始为空)
  ├── xSemaphoreCreateCounting(2, 2)       ← 计数信号量 (最大2, 初始2)
  ├── xSemaphoreCreateMutex()              ← 互斥锁 (初始可用)
  │
  ├── xTaskCreate(BinGiver,   pri=2)       ← Demo1: 每300ms给二值信号量
  ├── xTaskCreate(BinTaker,   pri=3)       ← Demo1: 等二值信号量→翻转LED0
  │
  ├── xTaskCreate(WorkerA,    pri=1)       ← Demo2: 每次获取一个"资源槽"
  ├── xTaskCreate(WorkerB,    pri=1)       ← Demo2: 获取→保持→释放
  ├── xTaskCreate(WorkerC,    pri=1)       ← Demo2: 最多2个Worker同时活跃
  │
  ├── xTaskCreate(MtxTaskA,   pri=2)       ← Demo3: 拿锁→快闪LED2→释放
  ├── xTaskCreate(MtxTaskB,   pri=2)       ← Demo3: 拿锁→慢闪LED2→释放
  │
  └── vTaskStartScheduler()                ← 🚀 永不返回

调度后并发世界:

  Demo1 (二值信号量 — 同步模式):
  BinGiver(pri2) ──Give──▶ [BinSem] ──Take──▶ BinTaker(pri3)
  每300ms                  初始=0             收到→Toggle LED0→Delay100ms

  Demo2 (计数信号量 — 资源池):
  WorkerA(pri1) ──Take──▶                    ┌── LED1=占用指示
  WorkerB(pri1) ──Take──▶ [CountSem(max=2)]──┤  只有2个Worker能同时持有
  WorkerC(pri1) ──Take──▶ (满则阻塞等待)     └── 第三个Take阻塞

  Demo3 (互斥锁 — 互斥访问):
  MtxTaskA(pri2) ──Take──▶ [Mutex] ──▶ 独占LED2 ──Give──▶ MtxTaskB(pri2)
  MtxTaskB(pri2) 等待A释放     → 快/慢闪交替 → 互斥无冲突

预期现象:
  LED0(PB13): 每300ms闪一次, 亮100ms (响应二值信号量)
  LED1(PA1):  大部分时间亮 (2个Worker持锁), 偶尔全灭 (全部释放)
  LED2(PB12): 快闪3次→灭→慢闪2次→灭→...交替 (互斥锁保证不重叠)
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 4 Basic: Binary Sem + Counting Sem + Mutex
  *
  *          Demonstrates three semaphore types on a single board:
  *
  *          Demo1 — Binary Semaphore (task synchronization):
  *            BinGiver periodically gives a binary semaphore;
  *            BinTaker blocks on it, toggles LED0 (PB13) when signalled.
  *            This is the classic "notification" pattern — no data, just a flag.
  *
  *          Demo2 — Counting Semaphore (resource pool):
  *            Three worker tasks compete for 2 slots managed by a counting
  *            semaphore (max=2, init=2). Only 2 workers can be "active"
  *            at any time; the third blocks until a slot is released.
  *            LED1 (PA1) indicates whether ANY worker holds a slot.
  *
  *          Demo3 — Mutex (mutual exclusion):
  *            Two tasks (MtxTaskA, MtxTaskB) share LED2 (PB12). Each takes
  *            the mutex, flashes LED2 with a distinct pattern, then releases.
  *            The mutex guarantees the patterns never overlap.
  *
  *          Key lesson: Mutex has OWNERSHIP — only the task that took it
  *          can give it back. Binary semaphore has NO ownership — any task
  *          can give/take. This makes mutex correct for mutual exclusion
  *          and binary semaphore correct for synchronization.
  *
  *          Architecture:
  *            BinGiver(pri2) ──Give──▶ [BinSem] ──Take──▶ BinTaker(pri3)
  *            WorkerA(pri1) ──Take──▶ [CountSem(2)] ──Give──▶ ...
  *            WorkerB(pri1) ──Take──▶                ──Give──▶ ...
  *            WorkerC(pri1) ──Take──▶ (blocks)       ──Give──▶ ...
  *            MtxTaskA(pri2) ──Take──▶ [Mutex] ──Give──▶ MtxTaskB(pri2)
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"     /* All semaphore APIs (Binary/Counting/Mutex/Recursive) */

/* Private typedef -----------------------------------------------------------*/

/**
  * @brief  Worker context passed to counting-semaphore worker tasks.
  *         Each worker has a unique ID and a work delay for visual variety.
  */
typedef struct {
    uint8_t  worker_id;      /* Worker identifier: 0=A, 1=B, 2=C               */
    uint16_t work_time_ms;   /* How long this worker "uses" the resource (ms)  */
    uint16_t rest_time_ms;   /* How long this worker rests between attempts    */
} WorkerCfg_t;

/* Private defines ------------------------------------------------------------*/

/* --- Counting Semaphore configuration --- */
#define COUNT_SEM_MAX       2U          /* Max simultaneous resource holders   */
#define COUNT_SEM_INIT      2U          /* Initial count (all slots available) */

/* --- Task priorities --- */
/* Binary semaphore: Taker higher than Giver → guaranteed prompt consumption  */
#define PRIO_BIN_TAKER      3U
#define PRIO_BIN_GIVER      2U

/* Counting semaphore: all workers equal priority, time-sliced if needed      */
#define PRIO_COUNT_WORKER   1U

/* Mutex: both tasks equal priority → time-sliced round-robin when both ready */
#define PRIO_MTX_A          2U
#define PRIO_MTX_B          2U

/* --- Task stack sizes (words) --- */
#define STACK_BIN_GIVER     50U         /* 200B — simple: give sem + delay     */
#define STACK_BIN_TAKER     80U         /* 320B — take sem + toggle + delay    */
#define STACK_COUNT_WORKER  80U         /* 320B — take/release sem + delay     */
#define STACK_MTX_A         100U        /* 400B — mutex + nested LED flashes   */
#define STACK_MTX_B         100U        /* 400B — mutex + nested LED flashes   */

/* --- Timing (ms) --- */
#define BIN_GIVE_PERIOD_MS  300U        /* Binary sem: give every 300ms        */
#define BIN_LED_ON_MS       100U        /* Binary sem: LED on time after take  */
#define MTX_TAKE_TIMEOUT_MS 200U        /* Mutex: max wait before giving up    */

/* --- LED pin definitions --- */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13     /* PB13 — on-board LED (bin sem)   */
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1      /* PA1  — ext LED (count sem)      */
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12     /* PB12 — ext LED (mutex)          */

/* --- LED helpers: active-low (set=off, reset=on) --- */
#define LED_ON(port, pin)   HAL_GPIO_WritePin((port), (pin), GPIO_PIN_RESET)
#define LED_OFF(port, pin)  HAL_GPIO_WritePin((port), (pin), GPIO_PIN_SET)

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — all semaphore handles in one struct.
  * @note   Per project convention: no scattered global variables.
  */
static struct {
    SemaphoreHandle_t xBinSem;      /* Binary semaphore — synchronization     */
    SemaphoreHandle_t xCountSem;    /* Counting semaphore — resource pool     */
    SemaphoreHandle_t xMutex;       /* Mutex — mutual exclusion               */
} g;                                /* g = globals */

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);

/* --- Demo 1: Binary Semaphore tasks --- */
static void BinGiver_Task(void *pvParameters);
static void BinTaker_Task(void *pvParameters);

/* --- Demo 2: Counting Semaphore tasks --- */
static void CountWorker_Task(void *pvParameters);

/* --- Demo 3: Mutex tasks --- */
static void MtxTaskA_Task(void *pvParameters);
static void MtxTaskB_Task(void *pvParameters);

/* --- Main ------------------------------------------------------------------*/

/**
  * @brief  Main program — creates 3 semaphores + 7 tasks, starts scheduler.
  * @note   vTaskStartScheduler() never returns; the while(1) below is a
  *         safety net that the compiler should never reach.
  */
int main(void)
{
    BaseType_t xReturned;

    /* --- Hardware init --- */
    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * --- Create Semaphores ---
     * All three types create SemaphoreHandle_t; their behavior differs
     * at the kernel level (ownership, priority inheritance, etc.).
     */

    /* Binary semaphore: created in "empty" state (0).
     * The Taker will block until the Giver gives it. */
    g.xBinSem = xSemaphoreCreateBinary();
    configASSERT(g.xBinSem != NULL);

    /* Counting semaphore: max=2, initial=2 (both slots available).
     * First two Take() calls succeed immediately; third blocks. */
    g.xCountSem = xSemaphoreCreateCounting(COUNT_SEM_MAX, COUNT_SEM_INIT);
    configASSERT(g.xCountSem != NULL);

    /* Mutex: created in "available" state. Includes priority inheritance.
     * Only the task that took it can give it back. */
    g.xMutex = xSemaphoreCreateMutex();
    configASSERT(g.xMutex != NULL);

    /*
     * --- Create Tasks ---
     */

    /* Demo 1: Binary semaphore pair (2 tasks) */
    xReturned = xTaskCreate(BinTaker_Task, "BinTaker", STACK_BIN_TAKER,
                            NULL, PRIO_BIN_TAKER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(BinGiver_Task, "BinGiver", STACK_BIN_GIVER,
                            NULL, PRIO_BIN_GIVER, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 2: Counting semaphore workers (3 tasks, 2 slots) */
    {
        static WorkerCfg_t cfgA = {0, 600, 1000};  /* Worker A: work 600ms, rest 1000ms */
        static WorkerCfg_t cfgB = {1, 800, 800};   /* Worker B: work 800ms, rest 800ms  */
        static WorkerCfg_t cfgC = {2, 400, 1200};  /* Worker C: work 400ms, rest 1200ms */

        xReturned = xTaskCreate(CountWorker_Task, "WorkerA", STACK_COUNT_WORKER,
                                &cfgA, PRIO_COUNT_WORKER, NULL);
        configASSERT(xReturned == pdPASS);

        xReturned = xTaskCreate(CountWorker_Task, "WorkerB", STACK_COUNT_WORKER,
                                &cfgB, PRIO_COUNT_WORKER, NULL);
        configASSERT(xReturned == pdPASS);

        xReturned = xTaskCreate(CountWorker_Task, "WorkerC", STACK_COUNT_WORKER,
                                &cfgC, PRIO_COUNT_WORKER, NULL);
        configASSERT(xReturned == pdPASS);
    }

    /* Demo 3: Mutex pair (2 tasks sharing LED2) */
    xReturned = xTaskCreate(MtxTaskA_Task, "MtxTaskA", STACK_MTX_A,
                            NULL, PRIO_MTX_A, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(MtxTaskB_Task, "MtxTaskB", STACK_MTX_B,
                            NULL, PRIO_MTX_B, NULL);
    configASSERT(xReturned == pdPASS);

    /* Launch the scheduler — never returns */
    vTaskStartScheduler();

    /* Safety net — should be unreachable */
    while (1) {}
}

/* ========================================================================== */
/*  Demo 1: Binary Semaphore — Task Synchronization                           */
/* ========================================================================== */

/**
  * @brief  Giver task: periodically gives the binary semaphore.
  * @note   xSemaphoreGive on a binary sem:
  *          - Sem is 0 → set to 1, unblock the highest-priority waiting task.
  *          - Sem is 1 → return err (binary sem already "full").
  *          Uses vTaskDelayUntil for precise period.
  */
static void BinGiver_Task(void *pvParameters)
{
    (void)pvParameters;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * Give the binary semaphore.
         * If BinTaker is blocked waiting, it wakes NOW (within this call).
         * Because BinTaker has higher priority (3 > 2), it preempts US
         * immediately and toggles LED0.
         */
        xSemaphoreGive(g.xBinSem);

        /* Wait until the next period boundary */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(BIN_GIVE_PERIOD_MS));
    }
}

/**
  * @brief  Taker task: blocks on binary semaphore, toggles LED0 when signalled.
  * @note   xSemaphoreTake with portMAX_DELAY:
  *          - Sem is 0 → task blocks (suspended, zero CPU)
  *          - Sem becomes 1 → task wakes, sem atomically reset to 0
  *          The "binary" nature means each Give corresponds to exactly one Take.
  */
static void BinTaker_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Block indefinitely until the Giver signals us.
         * This is ZERO-CPU waiting — the scheduler skips us entirely.
         */
        if (xSemaphoreTake(g.xBinSem, portMAX_DELAY) == pdPASS)
        {
            /* Signal received! Flash LED0 briefly to show we were woken up. */
            LED_ON(LED0_PORT, LED0_PIN);
            vTaskDelay(pdMS_TO_TICKS(BIN_LED_ON_MS));
            LED_OFF(LED0_PORT, LED0_PIN);
        }
        /* Loop back, wait for next signal */
    }
}

/* ========================================================================== */
/*  Demo 2: Counting Semaphore — Resource Pool                                */
/* ========================================================================== */

/**
  * @brief  Worker task: acquires a counting-semaphore "slot", works, releases.
  * @note   Pattern:
  *          1. xSemaphoreTake(g.xCountSem, portMAX_DELAY) → acquire a slot
  *          2. "Do work" (LED1 ON + delay) — simulate resource usage
  *          3. xSemaphoreGive(g.xCountSem) → release the slot
  *
  *          With max=2, init=2:
  *          - First 2 Take() calls → succeed immediately, count=0
  *          - Third Take() call → blocks until one of the first two releases
  *          - Each Give() → count++, unblocks one waiting worker (if any)
  *
  *          LED1 (PA1) is shared as a "resource busy" indicator:
  *          ON when any worker holds a slot, OFF when all slots free.
  *          Because multiple workers can share LED1, we use a simple rule:
  *          each worker turns LED1 ON while it holds a slot.
  */
static void CountWorker_Task(void *pvParameters)
{
    const WorkerCfg_t *cfg = (const WorkerCfg_t *)pvParameters;

    for (;;)
    {
        /*
         * Attempt to acquire a resource slot.
         * Blocks here if all slots are occupied (count == 0).
         */
        if (xSemaphoreTake(g.xCountSem, portMAX_DELAY) == pdPASS)
        {
            /* Slot acquired! Turn on the "busy" LED. */
            LED_ON(LED1_PORT, LED1_PIN);

            /* Simulate using the resource for work_time_ms */
            vTaskDelay(pdMS_TO_TICKS(cfg->work_time_ms));

            /* Done with the resource. Turn off LED, release slot. */
            LED_OFF(LED1_PORT, LED1_PIN);
            xSemaphoreGive(g.xCountSem);
        }

        /* Rest before next acquisition attempt */
        vTaskDelay(pdMS_TO_TICKS(cfg->rest_time_ms));
    }
}

/* ========================================================================== */
/*  Demo 3: Mutex — Mutual Exclusion                                          */
/* ========================================================================== */

/**
  * @brief  MtxTaskA: takes the mutex, flashes LED2 fast (3× @80ms), releases.
  * @note   Mutex vs Binary Semaphore — the critical difference:
  *          - Mutex: ONLY the task that took it can give it back (ownership).
  *          - Binary sem: ANY task can give it (no ownership concept).
  *          This ownership is essential for mutual exclusion correctness.
  *
  *          Timeout of MTX_TAKE_TIMEOUT_MS ensures we don't block forever
  *          if MtxTaskB holds the mutex for an unusually long time.
  */
static void MtxTaskA_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /* Try to acquire the mutex with a timeout */
        if (xSemaphoreTake(g.xMutex, pdMS_TO_TICKS(MTX_TAKE_TIMEOUT_MS)) == pdPASS)
        {
            /* Mutex acquired! Flash LED2 rapidly 3 times.
             * During this entire block, no other task can access LED2. */
            for (uint8_t i = 0; i < 3; i++)
            {
                LED_ON(LED2_PORT, LED2_PIN);
                vTaskDelay(pdMS_TO_TICKS(80));
                LED_OFF(LED2_PORT, LED2_PIN);
                vTaskDelay(pdMS_TO_TICKS(80));
            }

            /* Release the mutex — MtxTaskB can now acquire it */
            xSemaphoreGive(g.xMutex);

            /* Pause before next attempt, giving MtxTaskB a chance */
            vTaskDelay(pdMS_TO_TICKS(300));
        }
        else
        {
            /* Timeout: MtxTaskB held the mutex too long.
             * In production: log this event for diagnostics. */
        }
    }
}

/**
  * @brief  MtxTaskB: takes the mutex, flashes LED2 slow (2× @200ms), releases.
  * @note   Same priority as MtxTaskA (prio=2). With configUSE_TIME_SLICING=1,
  *          they will round-robin when both are ready.
  *
  *          Observable behavior: LED2 alternates between fast-3-blink (TaskA)
  *          and slow-2-blink (TaskB), NEVER overlapping — proof that the
  *          mutex correctly serializes access.
  */
static void MtxTaskB_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        if (xSemaphoreTake(g.xMutex, pdMS_TO_TICKS(MTX_TAKE_TIMEOUT_MS)) == pdPASS)
        {
            /* Mutex acquired! Flash LED2 slowly 2 times. */
            for (uint8_t i = 0; i < 2; i++)
            {
                LED_ON(LED2_PORT, LED2_PIN);
                vTaskDelay(pdMS_TO_TICKS(200));
                LED_OFF(LED2_PORT, LED2_PIN);
                vTaskDelay(pdMS_TO_TICKS(200));
            }

            xSemaphoreGive(g.xMutex);
            vTaskDelay(pdMS_TO_TICKS(500));
        }
        else
        {
            /* Timeout — MtxTaskA held the mutex too long */
        }
    }
}

/* ========================================================================== */
/*  Hardware Configuration                                                    */
/* ========================================================================== */

/**
  * @brief  Initialize 3 GPIO pins as push-pull outputs.
  *         PB13 = LED0 (binary sem), PA1 = LED1 (count sem), PB12 = LED2 (mutex)
  */
static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    /* PB13 — on-board LED (binary semaphore demo) */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    LED_OFF(LED0_PORT, LED0_PIN);   /* Start with LED off */

    /* PA1 — external LED (counting semaphore demo) */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    LED_OFF(LED1_PORT, LED1_PIN);

    /* PB12 — external LED (mutex demo) */
    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_PORT, &GPIO_InitStruct);
    LED_OFF(LED2_PORT, LED2_PIN);
}

/**
  * @brief  System clock: HSI 8MHz, no PLL.
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

/**
  * @brief  Stack overflow hook — called when configCHECK_FOR_STACK_OVERFLOW=2
  *         detects a corrupted stack canary at context switch.
  */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    /* Set breakpoint here — inspect xTask handle and pcTaskName in debugger */
    __disable_irq();
    for (;;) {}
}

/**
  * @brief  HAL fatal error handler.
  */
void APP_ErrorHandler(void)
{
    __disable_irq();
    for (;;) {}
}
```

---

## FreeRTOSConfig.h 额外配置

本课需要确认以下宏在 `FreeRTOSConfig.h` 中已启用 (当前项目已配好):

```c
#define configUSE_MUTEXES                  1   /* xSemaphoreCreateMutex       */
#define configUSE_RECURSIVE_MUTEXES        1   /* xSemaphoreCreateRecursiveMutex */
#define configUSE_COUNTING_SEMAPHORES      1   /* xSemaphoreCreateCounting     */
#define configSUPPORT_DYNAMIC_ALLOCATION   1   /* Dynamic creation needs heap  */

/* ❗ 重要: 本课创建 7个任务+3个信号量, 建议增大堆 */
#define configTOTAL_HEAP_SIZE              6144  /* 从4096增加到6144 (6KB)     */
```

---

## 内存预算

```
configTOTAL_HEAP_SIZE = 6144 bytes (recommended)

Item                    │ Size   │ Qty │ Subtotal
────────────────────────┼────────┼─────┼─────────
Idle Task               │   760B │  1  │   760B
Timer Service Task      │   760B │  1  │   760B
BinTaker (80W stack)    │   440B │  1  │   440B
BinGiver (50W stack)    │   320B │  1  │   320B
WorkerA (80W stack)     │   440B │  1  │   440B
WorkerB (80W stack)     │   440B │  1  │   440B
WorkerC (80W stack)     │   440B │  1  │   440B
MtxTaskA (100W stack)   │   520B │  1  │   520B
MtxTaskB (100W stack)   │   520B │  1  │   520B
Binary Semaphore        │    80B │  1  │    80B
Counting Semaphore      │    80B │  1  │    80B
Mutex                   │    80B │  1  │    80B
────────────────────────┼────────┼─────┼─────────
TOTAL                   │        │     │  4800B / 6144B
FREE                    │        │     │  1344B

✅ 4800B used, 1344B free — healthy margin for heap fragmentation overhead.
```

---

## 验证清单

```
□ 1. 编译 0 Error, 0 Warning (需要 #include "semphr.h")
□ 2. 检查 configUSE_COUNTING_SEMAPHORES=1, configUSE_MUTEXES=1
□ 3. configTOTAL_HEAP_SIZE ≥ 6144 (否则 tasks创建可能失败)
□ 4. 下载运行后, 观察LED行为:

   Demo1 (二值信号量):
   □ PB13 每300ms亮100ms — BinGiver给信号量 → BinTaker消费 → LED闪烁

   Demo2 (计数信号量):
   □ PA1 大部分时间亮 — 因为3个Worker竞争2个槽, 槽大部分时间被占用
   □ PA1 偶尔短暂全灭 — 所有槽释放的瞬间 (观察用示波器更准确)

   Demo3 (互斥锁):
   □ PB12 快闪3次(80ms间隔) → 灭约300ms → 慢闪2次(200ms间隔) → 灭约500ms → 循环
   □ 快闪和慢闪永不重叠 — 证明互斥锁生效

□ 5. 在 xSemaphoreTake 处设断点, 观察不同信号量的阻塞/唤醒行为
```

---

## 观察要点

| 现象 | 说明 |
|------|------|
| LED0 每300ms亮一次 | BinGiver 每300ms给二值信号量 → BinTaker取到→亮100ms→灭 |
| LED1 常亮, 偶尔灭 | 3个Worker争2个槽, LED1做"占用指示", 所有槽释放瞬间全灭 |
| LED2 快慢交替, 不重叠 | MtxTaskA(快3闪) 和 MtxTaskB(慢2闪) 通过互斥锁串行访问LED2 |
| 二值信号量不累计 | Give时如果信号量已是"满", 第二次Give返回 err (二值信号量只有0/1) |
| 计数信号量可累计 | Give时如果计数未达max, 计数+1; Take时计数-1, 计数=0则阻塞 |
| 互斥锁有所有权 | MtxTaskA拿了锁, MtxTaskB不能Give(释放) — 只有拥有者才能释放 |

---

## 调试技巧

### 1. 观察信号量值

```c
/* 在 Watch 窗口添加: */
uxSemaphoreGetCount(g.xBinSem);     /* Binary sem: 0 or 1            */
uxSemaphoreGetCount(g.xCountSem);   /* Count sem: 0, 1, or 2        */
/* Mutex: use uxQueueMessagesWaiting() — mutex is queue-based       */
uxQueueMessagesWaiting((QueueHandle_t)g.xMutex); /* 0=locked, 1=free */
```

### 2. 验证计数信号量的"第3个阻塞"

```
在 WorkerC 的 xSemaphoreTake 行设断点:
  → 如果此时 WorkerA和WorkerB都持有着槽, WorkerC会停在这一行不返回
  → 当 WorkerA 或 WorkerB 调用 xSemaphoreGive 后, WorkerC立即恢复执行
  → 这就是"信号量从0变1 → 唤醒最高优先级等待者"的实时性
```

### 3. 验证互斥锁的"不重叠"

```
用逻辑分析仪接 PB12:
  → 快闪组(3个脉冲, 80ms间隔) 和慢闪组(2个脉冲, 200ms间隔) 边界清晰
  → 快闪和慢闪之间一定有一段灭的时间 (一个任务释放锁, 另一个还没拿到)
  → 绝不会看到快闪脉冲插入慢闪脉冲中间
```

### 4. 模拟计数信号量满

```c
/* 临时把 COUNT_SEM_MAX 改为 1:
 * 三选一 — 每次只有一个Worker能持有槽, 另外两个排队 *
 * 这时 LED1 的闪烁变成单个Worker的节奏 */
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|----------|
| 二值信号量做互斥 | Binary sem没有所有权概念, 任务B可以Give任务A的信号量 | 用 Mutex, 有所有权检查 |
| Mutex 在 ISR 中用 | Mutex 不能在中断中 Take/Give (涉及优先级继承逻辑) | ISR中用二值/计数信号量的 FromISR 版本 |
| Mutex 递归死锁 | 同一任务两次 Take 同一个 Mutex → 死锁 | 用 RecursiveMutex, 或用信号量 |
| 忘记 `#include "semphr.h"` | 编译报隐式声明 — semphr.h是纯宏, 但宏展开需要FreeRTOS.h先include | `#include "FreeRTOS.h"` 必须在前 |
| Take后忘记Give | 资源永久泄漏, 其他任务永久阻塞 | Take-Give成对出现, 用do-while(0)或RAII模式 |
| SemaphoreHandle_t 当 QueueHandle_t 用 | 编译可能通过但行为不正确 | 只用 semphr.h 的宏来操作信号量 |
| 优先级反转 | 低优先任务持有锁, 中优先任务抢占, 高优先任务等锁 | 用 Mutex (有优先级继承) |

---

## 最佳实践清单

| # | 实践 | 说明 |
|---|------|------|
| 1 | 同步用二值信号量, 互斥用Mutex | 不同目的用不同工具, 不混用 |
| 2 | Mutex Take/Give 在同一任务中成对出现 | 所有权原则 — 谁拿锁谁释放 |
| 3 | 计数信号量 max = 实际资源数 | 不虚报资源数, 否则可能过度分配 |
| 4 | `configASSERT` 检查所有信号量创建 | xSemaphoreCreateXxx 返回 NULL → 立即停止 |
| 5 | Take 时用 portMAX_DELAY 或合理超时 | 避免"黑洞等待" (无限阻塞无日志) |
| 6 | ISR 中只用 GiveFromISR / TakeFromISR | 且只用二值/计数信号量, 不用Mutex |
| 7 | 全局变量用结构体封装 | 所有信号量句柄集中管理 |

---

> **📎 进阶版:** 见 `Lesson4_Semaphore_Advanced.md` — 优先级继承完整演示 + ISR信号量 + 递归互斥锁
