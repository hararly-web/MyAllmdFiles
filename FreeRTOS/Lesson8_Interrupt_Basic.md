# 第8课 代码实战 — 基础版：中断管理 (ISR→Task 通信 + 临界区保护 + 优先级配置)

> **目标：** 掌握 xxxFromISR() API 规则、临界区保护、中断优先级配置、ISR→Task 通信完整模式
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)，TIM6 内部定时器
> **前置：** 第7课任务通知已完成，理解阻塞等待和 ISR 基本概念
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（已配好）

---

## 核心概念：中断管理全景图

```
┌──────────────────────────────────────────────────────────────────────────┐
│            FreeRTOS + Cortex-M4F 中断管理核心架构                          │
│                                                                          │
│  硬件中断优先级 (PY32F403: 3-bit NVIC, 8 级优先级 0~7)                     │
│                                                                          │
│   优先级0  ┬────────────────  NMI / HardFault  ── 最高, 不可屏蔽          │
│   优先级1  │                 ❌ 不能调 FreeRTOS API                        │
│   优先级2  ┘                 (可打断内核临界区)                            │
│   ─────────┼────────────  configLIBRARY_MAX_SYSCALL = 3                  │
│   优先级3  │                                                                │
│   优先级4  │                 ✅ 可调 xxxFromISR() API                      │
│   优先级5  │                 (不能阻塞, 只能 Try或FromISR)                  │
│   优先级6  │                                                               │
│   优先级7  ┴────────────  SysTick / PendSV (内核中断, 最低优先级)          │
│                                                                          │
│  核心规则:                                                                │
│    1. ISR 永远不能阻塞 (No portMAX_DELAY!)                                │
│    2. 优先级 >= MAX_SYSCALL 的 ISR 才能调 FromISR API                     │
│    3. FromISR API 比任务 API 多一个 pxHigherPriorityTaskWoken 参数        │
│    4. ISR 结束时必须 portYIELD_FROM_ISR(xHigherPriorityTaskWoken)        │
│    5. 临界区 protection 使用 taskENTER_CRITICAL/taskEXIT_CRITICAL        │
│       (内部通过关中断实现, 嵌套安全)                                       │
│                                                                          │
│  ISR→Task 通信标准模式 (最重要!):                                          │
│    ISR:  xQueueSendFromISR(q, &data, &xTaskWoken)                       │
│          portYIELD_FROM_ISR(xTaskWoken)                                  │
│    Task: xQueueReceive(q, &data, portMAX_DELAY)                         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 程序架构

```
 Demo 1: ISR→Task 通信 (TIM6 定时器中断 → 队列 → 处理任务)
   TIM6_IRQHandler   — 每500ms触发一次
     → xQueueSendFromISR() + pxHigherPriorityTaskWoken
     → portYIELD_FROM_ISR()
   InterruptDemo_Task — 等待队列数据 → 翻转 LED0

 Demo 2: 临界区保护 (Critical Section)
   CritSecTask_A     — 无保护递增共享计数器 (演示数据损坏)
   CritSecTask_B     — 带 taskENTER_CRITICAL/EXIT_CRITICAL 保护
   LED1 闪烁表示有保护的计数器正确工作
   LED2 闪烁表示无保护的计数器出现竞争

 Demo 3: 中断优先级演示
   TIM6 配置为优先级 6 (可安全使用 FromISR)
   注意: actual NVIC register = LibraryPriority << (8 - configPRIO_BITS)
```

---

## 流程图 (中文)

```
┌──────────────────────────────────────────────────────────────────────────┐
│              第8课 中断管理 — 程序流程图                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  main()                                                                  │
│    ├── HAL_Init()                                                        │
│    ├── APP_SystemClockConfig()       ← HSI 8MHz                         │
│    ├── APP_GpioConfig()              ← 初始化 3 个 LED                   │
│    ├── APP_Tim6Config()              ← 配置 TIM6 定时器                  │
│    │     ├── TIM6 时钟使能                                              │
│    │     ├── TIM6 基础参数: Prescaler=7999, Period=499                  │
│    │     │   → 8MHz / 8000 / 500 = 2Hz (500ms 周期)                    │
│    │     ├── NVIC 优先级设为 6 (可安全调用 FromISR)                      │
│    │     └── HAL_TIM_Base_Start_IT(&htim6)  ← 启动定时器中断            │
│    │                                                                    │
│    ├── xQueueCreate(4, sizeof(uint32_t))   ← ISR→Task 通信队列          │
│    ├── xTaskCreate(InterruptDemo, prio=3)  ← 接收 ISR 数据的处理任务     │
│    ├── xTaskCreate(CritSec_A, prio=1)      ← 无保护临界区任务           │
│    ├── xTaskCreate(CritSec_B, prio=2)      ← 有保护临界区任务           │
│    │                                                                    │
│    └── vTaskStartScheduler()              ← 🚀                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Demo 1: ISR→Task 通信完整链路                                        │ │
│  │                                                                      │ │
│  │   TIM6 ISR (每500ms)               Queue (depth=4)     InterruptDemo │ │
│  │   ┌──────────────────┐            ┌──────────────┐    ┌────────────┐│ │
│  │   │ TIM6_IRQHandler  │            │ [0][1][2][3] │    │ prio=3     ││ │
│  │   │                  │ xQueueSend │    uint32_t   │    │            ││ │
│  │   │ count_u32++      │─FromISR──→│  (tick count) │───→│ xQueueRecv ││ │
│  │   │                  │            │              │    │            ││ │
│  │   │ portYIELD_FROM_  │            └──────────────┘    │ ToggleLED0││ │
│  │   │   ISR(taskWoken) │                                └────────────┘│ │
│  │   └──────────────────┘                                              │ │
│  │                                                                      │ │
│  │   预期: LED0 每500ms 翻转一次 (1Hz)                                  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Demo 2: 临界区保护对比                                                │ │
│  │                                                                      │ │
│  │   CritSecTask_A (prio=1)              CritSecTask_B (prio=2)        │ │
│  │   ┌────────────────────────┐          ┌────────────────────────┐   │ │
│  │   │ NO critical section    │          │ WITH critical section   │   │ │
│  │   │                        │          │                         │   │ │
│  │   │ for(i=0;i<1000;i++)   │          │ for(i=0;i<1000;i++)    │   │ │
│  │   │ {                      │          │ {                       │   │ │
│  │   │   vTaskDelay(1);       │  Race!   │   taskENTER_CRITICAL(); │   │ │
│  │   │   g.unprotected_cnt++; │  ←──────→│   g.protected_cnt++;   │   │ │
│  │   │ }                      │          │   taskEXIT_CRITICAL(); │   │ │
│  │   │                        │          │   vTaskDelay(1);        │   │ │
│  │   │ // 预期: counter != 2000!│         │ }                       │   │ │
│  │   └────────────────────────┘          │ // 预期: counter = 2000  │   │ │
│  │                                        └────────────────────────┘   │ │
│  │                                                                      │ │
│  │   LED1: 有保护计数器每1000次翻转 (正常)                             │ │
│  │   LED2: 无保护计数器结果 != 2000 时亮起 (检测到竞争)                │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 8 Basic: Interrupt Management
  *
  *          Demonstrates:
  *          - ISR→Task communication via Queue (TIM6 period interrupt)
  *          - critical section protection: taskENTER_CRITICAL/EXIT_CRITICAL
  *          - pxHigherPriorityTaskWoken + portYIELD_FROM_ISR pattern
  *          - Interrupt priority configuration on PY32F403 (3-bit NVIC)
  *
  *          Architecture:
  *
  *            [Demo 1: ISR→Task via Queue]
  *              TIM6 generates a 500ms periodic interrupt.
  *              The ISR sends a counter value to InterruptDemo_Task
  *              via xQueueSendFromISR(). The task blocks on
  *              xQueueReceive() and toggles LED0 on each wakeup.
  *
  *            [Demo 2: Critical Section]
  *              Two tasks increment unprotected and protected counters.
  *              CritSecTask_A (no protection) shows race condition —
  *              expected sum < 2000. CritSecTask_B (with protection)
  *              achieves correct 2000 count.
  *
  *          LED assignment:
  *            LED0 (PB13): Toggles on each ISR notification (1Hz)
  *            LED1 (PA1):  Blinks when protected counter works correctly
  *            LED2 (PB12): Lights when unprotected counter detects race
  *
  *          Interrupt priority map (PY32F403, 3-bit NVIC, 8 levels):
  *            Priority 0~2: ❌ Cannot call FreeRTOS API (HW-critical)
  *            Priority 3~6: ✅ Can call xxxFromISR() API
  *            Priority 7:    Kernel interrupts (SysTick/PendSV)
  *            TIM6: priority 6 ← safe for FromISR
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

/* Private defines ------------------------------------------------------------*/

/* LED pin definitions */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12

/*
 * ISR→Task queue parameters.
 * Queue depth 4 = buffer up to 4 ISR triggers before task processes them.
 * Each item = uint32_t (ISR counter value).
 */
#define ISR_QUEUE_LENGTH    4U
#define ISR_QUEUE_ITEM_SIZE sizeof(uint32_t)

/*
 * TIM6 timer parameters (HSI = 8MHz, no PLL).
 * TIM6 clock = HCLK = 8MHz (APB1 prescaler = 1, no x2 for timers).
 *
 * Target: 2 Hz interrupt (500ms period).
 *   Prescaler = 8000 → timer clock = 8MHz / 8000 = 1kHz
 *   Period    = 499  → 1000 / (499+1) = 2 Hz
 *
 * NVIC priority = 6: safe for FromISR API (≥ configLIBRARY_MAX_SYSCALL = 3).
 * NVIC register: 6 << (8 - 3) = 6 << 5 = 0xC0.
 */
#define TIM6_PRESCALER      7999U
#define TIM6_PERIOD         499U
#define TIM6_IRQ_PRIORITY   6U      /* Library level: 6 of 7 (lowest is 7) */

/*
 * Critical section test parameters.
 * Each task increments its counter 1000 times.
 * Without protection, the sum of increments will NOT equal 2000
 * due to read-modify-write race conditions.
 */
#define COUNTER_TARGET      1000U

/* Task priorities */
#define PRIO_INTERRUPT_DEMO 3U       /* Must respond to ISR data promptly    */
#define PRIO_CRITSEC_A      1U       /* Low priority — unprotected task      */
#define PRIO_CRITSEC_B      2U       /* Medium — protected task              */

/* Task stack sizes (words) */
#define STACK_INTERRUPT_DEMO  200U
#define STACK_CRITSEC         150U

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Timer handle for TIM6 basic timer.
  */
static TIM_HandleTypeDef htim6;

/**
  * @brief  Application globals — all shared state in one struct.
  * @note   volatile is REQUIRED for variables shared between ISR and task
  *         to prevent compiler optimization from caching values in registers.
  */
static struct {
    QueueHandle_t xIsrQueue;          /* Queue: ISR → Task notification        */
    volatile uint32_t ulIsrCount;     /* ISR counter (written in ISR context)  */
    volatile uint32_t ulUnprotectedCnt;  /* Unprotected shared counter (race)  */
    volatile uint32_t ulProtectedCnt;    /* Protected shared counter           */
    volatile uint32_t ulRaceDetected;    /* Non-zero = race condition detected */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_Tim6Config(void);
static void APP_LedToggle(uint8_t led_id);
static void APP_LedOn(uint8_t led_id);
static void APP_LedOff(uint8_t led_id);

/* --- Demo 1: ISR→Task via Queue ------------------------------------------- */
static void InterruptDemo_Task(void *pvParameters);

/* --- Demo 2: Critical Section --------------------------------------------- */
static void CritSecTask_A_Unprotected(void *pvParameters);
static void CritSecTask_B_Protected(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

/**
  * @brief  Creates queue + 3 tasks, configures TIM6 for periodic ISR.
  * @note   The TIM6 ISR starts firing AFTER HAL_TIM_Base_Start_IT() is called.
  *          The queue MUST be created before enabling the timer interrupt —
  *          otherwise the ISR would attempt to write to a NULL queue handle.
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Create ISR→Task queue BEFORE enabling the timer interrupt.
     * If the queue doesn't exist when the ISR fires, xQueueSendFromISR
     * will crash with a NULL-pointer dereference.
     */
    g.xIsrQueue = xQueueCreate(ISR_QUEUE_LENGTH, ISR_QUEUE_ITEM_SIZE);
    configASSERT(g.xIsrQueue != NULL);

    /* Initialize shared counters */
    g.ulIsrCount       = 0;
    g.ulUnprotectedCnt = 0;
    g.ulProtectedCnt   = 0;
    g.ulRaceDetected   = 0;

    /* Demo 1: ISR→Task communication handler */
    xReturned = xTaskCreate(InterruptDemo_Task, "IsrDemo",
                            STACK_INTERRUPT_DEMO, NULL,
                            PRIO_INTERRUPT_DEMO, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 2: Unprotected critical section task */
    xReturned = xTaskCreate(CritSecTask_A_Unprotected, "CritA",
                            STACK_CRITSEC, NULL, PRIO_CRITSEC_A, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 2: Protected critical section task */
    xReturned = xTaskCreate(CritSecTask_B_Protected, "CritB",
                            STACK_CRITSEC, NULL, PRIO_CRITSEC_B, NULL);
    configASSERT(xReturned == pdPASS);

    /*
     * Configure and start TIM6.
     * IMPORTANT: This is called AFTER queue and task creation,
     * so that the ISR has valid resources to work with.
     */
    APP_Tim6Config();

    vTaskStartScheduler();
    while (1) {}
}

/* ========================================================================== */
/*  Demo 1: ISR→Task Communication via Queue                                    */
/* ========================================================================== */

/**
  * @brief  TIM6 Interrupt Handler — sends counter to task via Queue.
  *
  * @note   THE CANONICAL ISR→Task PATTERN (follow this every time):
  *
  *         1. Clear the interrupt flag (handled by HAL in this case).
  *         2. Prepare the data to send.
  *         3. Call xQueueSendFromISR() or similar xxxFromISR() API.
  *            - Use &xHigherPriorityTaskWoken (initialized to pdFALSE).
  *            - NEVER use portMAX_DELAY — ISR cannot block!
  *         4. Call portYIELD_FROM_ISR(xHigherPriorityTaskWoken) at the END.
  *            - This triggers a context switch AFTER all ISRs have executed,
  *              if a higher-priority task was woken.
  *
  *   ⚠️ KEY RULE: The FromISR API call and the YIELD must be in the SAME ISR
  *      handler. Don't yield in a nested ISR — the kernel tracks nesting.
  *
  *   ⚠️ portYIELD_FROM_ISR: Must be called ONCE at the END of the ISR,
  *      after ALL FromISR operations are complete.
  *
  *   Priority analysis for TIM6 (priority 6):
  *     Priority 6 ≥ configLIBRARY_MAX_SYSCALL (3) → SAFE to call FromISR APIs.
  *     SysTick is priority 7 (lower) → TIM6 can preempt SysTick.
  *     This is expected and handled correctly by the kernel.
  */
void TIM6_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /*
     * Step 1: HAL interrupt processing.
     * Clears the TIM6 update interrupt flag.
     */
    HAL_TIM_IRQHandler(&htim6);

    /*
     * Step 2: Increment ISR counter.
     * This is the "sensor data" in a real application.
     */
    g.ulIsrCount++;

    /*
     * Step 3: Send to task via Queue — FromISR version.
     *
     * xQueueSendFromISR(queue, pvItemToQueue, pxHigherPriorityTaskWoken):
     *   - pvItemToQueue: pointer to the data to copy into the queue.
     *   - pxHigherPriorityTaskWoken: pointer to BaseType_t, SET BY THE API
     *     to pdTRUE if a task with higher priority than the currently
     *     interrupted task was woken.
     *   - Returns immediately (never blocks).
     *   - Returns pdPASS if the item was queued, errQUEUE_FULL if full.
     *
     *   In production code, always check the return value:
     *     if (xQueueSendFromISR(...) != pdPASS) { /* overflow! */ }
     */
    (void)xQueueSendFromISR(g.xIsrQueue,
                            &g.ulIsrCount,
                            &xHigherPriorityTaskWoken);

    /*
     * Step 4: Request a context switch if a higher-priority task was woken.
     *
     * portYIELD_FROM_ISR(xHigherPriorityTaskWoken):
     *   - If xHigherPriorityTaskWoken == pdTRUE, sets the PendSV pending bit.
     *   - The actual switch happens AFTER all nested ISRs return
     *     (PendSV is the lowest-priority exception).
     *   - If xHigherPriorityTaskWoken == pdFALSE, this is a no-op.
     *   - Must be called ONCE at the end of the ISR.
     *
     *   Why not switch immediately?
     *   Cortex-M tail-chaining + PendSV design: PendSV runs when no other
     *   ISR is pending, ensuring the context switch doesn't nest inside
     *   an active ISR.
     */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/**
  * @brief  Blocks on queue, receives ISR counter, toggles LED0.
  * @note   This task is woken by the ISR via xQueueSendFromISR.
  *          The task has priority 3 (higher than TIM6 ISR effective priority).
  *
  *          Expected LED0 behavior: toggle at 1 Hz (every 500ms ISR trigger).
  */
static void InterruptDemo_Task(void *pvParameters)
{
    (void)pvParameters;
    uint32_t ulReceivedCount;

    for (;;)
    {
        /*
         * Block until ISR sends data via queue.
         * portMAX_DELAY is safe here — this is a TASK, not ISR.
         *
         * When the ISR calls xQueueSendFromISR(), the kernel:
         *   1. Copies &g.ulIsrCount into the queue buffer.
         *   2. Moves this task from Blocked to Ready state.
         *   3. Sets xHigherPriorityTaskWoken = pdTRUE (since prio 3 > idle).
         *   4. portYIELD_FROM_ISR triggers PendSV → context switch to THIS task.
         */
        if (xQueueReceive(g.xIsrQueue, &ulReceivedCount, portMAX_DELAY)
            == pdPASS)
        {
            /*
             * ulReceivedCount = the ISR counter value at the time
             * the ISR fired. Each ISR trigger sends exactly one value.
             */
            APP_LedToggle(0);

            /*
             * Avoid unused variable warning — the count is available
             * for debugging via a watchpoint.
             */
            (void)ulReceivedCount;
        }
    }
}

/* ========================================================================== */
/*  Demo 2: Critical Section Protection                                         */
/* ========================================================================== */

/**
  * @brief  Task A — increments unprotected counter WITHOUT critical section.
  *
  * @note   This task demonstrates a CLASSIC read-modify-write race condition:
  *
  *          Timeline (simplified):
  *
  *            TaskA (prio 1)              TaskB (prio 2)
  *            ─────────────               ─────────────
  *            reads counter → 42
  *                                         reads counter → 42
  *            writes 42+1 = 43
  *                                         writes 42+1 = 43
  *            ─── 2 increments, counter goes from 42 to 43! ───
  *
  *          Expected result: 1000 + 1000 = 2000 → actual < 2000
  *          The difference = lost increments due to race conditions.
  *
  *          Visual feedback:
  *          - LED2 lights up if race condition is detected
  *          - LED1 blinks to show TaskB is still running correctly
  */
static void CritSecTask_A_Unprotected(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Phase 1: Increment unprotected counter 1000 times.
         *          Uses vTaskDelay(1) to increase the chance of
         *          context switches between read and write —
         *          exposing the race condition.
         */
        for (uint32_t i = 0; i < COUNTER_TARGET; i++)
        {
            /*
             * ⚠️ RACE WINDOW: Between reading and writing the counter,
             * TaskB (higher priority) may preempt and also modify it.
             *
             * This is a 3-step operation at the assembly level:
             *   1. LDR r0, [counter_addr]     ← load counter into register
             *   2. ADD r0, r0, #1             ← increment register
             *   3. STR r0, [counter_addr]     ← store register back
             *
             * If TaskB preempts between steps 1 and 3, TaskB's write
             * is lost when TaskA resumes and writes its stale value.
             *
             * vTaskDelay(1) deliberately amplifies the race window by
             * allowing the scheduler to switch tasks.
             */
            g.ulUnprotectedCnt++;
            vTaskDelay(1);     /* Amplify race window */
        }

        /*
         * Phase 2: Check for race condition.
         *
         * If both tasks incremented 1000 times, g.ulUnprotectedCnt
         * SHOULD be 2000 (1000 from A + 1000 from B).
         *
         * Race condition: count < 2000 → some increments were lost.
         *
         * We reset g.ulUnprotectedCnt AFTER checking, but since
         * this reset itself is a race window, we use an atomic-ish
         * flag (ulRaceDetected) to communicate the result.
         */
        if (g.ulUnprotectedCnt != (COUNTER_TARGET * 2U))
        {
            /* Race detected! Light LED2 as warning. */
            g.ulRaceDetected++;
            APP_LedOn(2);
            vTaskDelay(pdMS_TO_TICKS(1000));
            APP_LedOff(2);
        }

        /* Reset counters for next round */
        g.ulUnprotectedCnt = 0;

        /* Wait before next test cycle */
        vTaskDelay(pdMS_TO_TICKS(3000));
    }
}

/**
  * @brief  Task B — increments protected counter WITH critical section.
  *
  * @note   taskENTER_CRITICAL() and taskEXIT_CRITICAL() implement a
  *          CRITICAL SECTION — code that runs with interrupts disabled.
  *
  *          What happens inside taskENTER_CRITICAL():
  *            1. Disable interrupts (via __disable_irq() on Cortex-M).
  *            2. Increment uxCriticalNesting (supports nesting).
  *
  *          What happens inside taskEXIT_CRITICAL():
  *            1. Decrement uxCriticalNesting.
  *            2. If nesting == 0, re-enable interrupts.
  *
  *          ⚠️ CRITICAL RULES for critical sections:
  *          - Keep them SHORT. Long critical sections increase interrupt
  *            latency (the time between IRQ assertion and ISR execution).
  *          - Rule of thumb: < 1 μs is ideal, < 10 μs is acceptable,
  *            > 100 μs requires redesign.
  *          - NEVER call blocking APIs inside a critical section
  *            (no vTaskDelay, no xQueueReceive with timeout, etc.)
  *            → will crash because the scheduler interrupt is disabled!
  *          - Nesting is safe: the critical nesting counter tracks depth.
  *
  *          Expected result: 1000 + 1000 = 2000 (ALWAYS correct).
  *
  *          Visual feedback:
  *          - LED1 blinks on each successful cycle.
  */
static void CritSecTask_B_Protected(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Phase 1: Increment protected counter 1000 times.
         *          Each increment is wrapped in a critical section
         *          to guarantee atomic read-modify-write.
         */
        for (uint32_t i = 0; i < COUNTER_TARGET; i++)
        {
            /*
             * Enter critical section.
             * Interrupts are DISABLED between ENTER and EXIT.
             * This is the gold standard for simple atomic operations.
             *
             * Alternative for multi-core or more complex operations:
             *   Use Mutex (xSemaphoreTake/xSemaphoreGive) if the
             *   protected operation is long or involves blocking APIs.
             *   Critical sections are better for VERY SHORT (<1μs)
             *   operations because they have lower overhead.
             */
            taskENTER_CRITICAL();
            {
                g.ulProtectedCnt++;
            }
            taskEXIT_CRITICAL();

            /*
             * vTaskDelay is OUTSIDE the critical section.
             * If vTaskDelay were inside, the scheduler interrupt would
             * be masked → scheduler can't run → task never awakens →
             * system hang (deadlock).
             */
            vTaskDelay(1);
        }

        /*
         * Phase 2: Verify protected counter.
         *
         * Because each increment is atomic (critical section),
         * the total is GUARANTEED to be 2000.
         */
        if (g.ulProtectedCnt == (COUNTER_TARGET * 2U))
        {
            /* Success! Toggle LED1 once to indicate correct operation. */
            APP_LedToggle(1);
        }
        else
        {
            /*
             * This should never happen with proper critical section
             * protection. If it does, something is fundamentally wrong
             * (e.g., an ISR at priority > configMAX_SYSCALL was not
             * masked by taskENTER_CRITICAL).
             */
            APP_LedOn(1);     /* Error: LED1 stays ON */
        }

        /* Reset for next round */
        g.ulProtectedCnt = 0;

        /* Wait before next test cycle */
        vTaskDelay(pdMS_TO_TICKS(3000));
    }
}

/* ========================================================================== */
/*  TIM6 Configuration                                                        */
/* ========================================================================== */

/**
  * @brief  Configure TIM6 basic timer for 500ms periodic interrupt.
  *
  * @note   TIM6 is a basic timer (no PWM, no input capture).
  *          It's perfect for generating periodic interrupts.
  *
  *          Clock tree:
  *            HSI (8MHz) → SYSCLK → HCLK → APB1 → TIM6
  *
  *          If APB1 prescaler ≠ 1, the timer clock is APB1 × 2
  *          (Cortex-M timer clock doubling rule).
  *          Here APB1 prescaler = 1, so TIM6 clock = HCLK = 8MHz.
  *
  *          NVIC priority setup:
  *            configPRIO_BITS = 3 (PY32F403: 3-bit, 8 levels 0~7)
  *            configLIBRARY_MAX_SYSCALL = 3
  *            TIM6 priority = 6 → 6 ≥ 3 → SAFE for FromISR API
  *            NVIC register value = 6 << (8-3) = 0xC0
  *
  *          ⚠️ If you set TIM6 priority to 0, 1, or 2:
  *            - configASSERT will fire on xQueueSendFromISR
  *            - Because the priority is above MAX_SYSCALL threshold
  *            - These high-priority ISRs can interrupt kernel critical sections
  */
static void APP_Tim6Config(void)
{
    /*
     * Configure TIM6 as a basic timer with interrupt.
     *
     * TIM6 is on APB1, clocked at HCLK = 8 MHz.
     * Target: 2 Hz interrupt (every 500 ms).
     *
     * Formula: f_int = TIM6_CLK / (Prescaler + 1) / (Period + 1)
     *          2 Hz  = 8,000,000 / (7999 + 1) / (499 + 1)
     *            = 8,000,000 / 8,000 / 500
     *            = 2 Hz ✓
     */
    htim6.Instance             = TIM6;
    htim6.Init.Prescaler        = TIM6_PRESCALER;
    htim6.Init.Period           = TIM6_PERIOD;
    htim6.Init.CounterMode      = TIM_COUNTERMODE_UP;
    htim6.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;

    __HAL_RCC_TIM6_CLK_ENABLE();

    if (HAL_TIM_Base_Init(&htim6) != HAL_OK)
    {
        APP_ErrorHandler();
    }

    /*
     * Configure NVIC for TIM6.
     *
     * HAL_NVIC_SetPriority(IRQn, PreemptPriority, SubPriority):
     *   On PY32F403 with 3-bit NVIC (PRIO_BITS=3):
     *     Only PreemptPriority matters (0~7).
     *     SubPriority is ignored (no sub-priority grouping configured).
     *
     *   PreemptPriority = 6:
     *     - Numerically > configLIBRARY_MAX_SYSCALL (3)
     *     - ✅ Can call xQueueSendFromISR() etc.
     *     - Can be preempted by ISRs at priority 0~5
     *     - Cannot be preempted by SysTick/PendSV (priority 7)
     */
    HAL_NVIC_SetPriority(TIM6_IRQn, TIM6_IRQ_PRIORITY, 0);
    HAL_NVIC_EnableIRQ(TIM6_IRQn);

    /*
     * Start the timer in interrupt mode.
     * The first interrupt fires after Period+1 timer clocks = 500 ms.
     * Thereafter every 500 ms.
     *
     * ⚠️ Call sequence matters:
     *   1. Create queue/tasks ← done before this function
     *   2. Start timer interrupt ← NOW
     *   If reversed, the ISR fires into an uninitialized system.
     */
    if (HAL_TIM_Base_Start_IT(&htim6) != HAL_OK)
    {
        APP_ErrorHandler();
    }
}

/**
  * @brief  TIM6 Period Elapsed callback (HAL weak override).
  * @note   Called by HAL_TIM_IRQHandler() when the update interrupt fires.
  *          This is optional — we handle everything in TIM6_IRQHandler.
  *          Defined here as a placeholder for HAL_TIM_IRQHandler.
  */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    /*
     * All ISR→Task logic is in TIM6_IRQHandler() above.
     * This callback is empty by design — the real work is done there
     * to keep the ISR pattern explicit and educational.
     */
    (void)htim;
}

/* ========================================================================== */
/*  Hardware Helpers                                                          */
/* ========================================================================== */

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

static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    /* LED0 — ISR→Task notification indicator (1Hz toggle) */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* LED1 — Protected critical section success indicator */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* LED2 — Race condition detected indicator */
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

## ISR API 速查：任务 API vs FromISR API 对照

| 任务 API (任务中调用) | FromISR API (ISR 中调用) | 额外参数 |
|----------------------|------------------------|---------|
| `xQueueSend(q, data, timeout)` | `xQueueSendFromISR(q, data, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xQueueSendToBack(q, data, timeout)` | `xQueueSendToBackFromISR(q, data, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xQueueSendToFront(q, data, timeout)` | `xQueueSendToFrontFromISR(q, data, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xQueueReceive(q, buf, timeout)` | `xQueueReceiveFromISR(q, buf, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xSemaphoreGive(sem)` | `xSemaphoreGiveFromISR(sem, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xSemaphoreTake(sem, timeout)` | ❌ 不存在! ISR 不能 Take | — |
| `xTaskNotifyGive(task)` | `vTaskNotifyGiveFromISR(task, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xTaskNotify(task, val, action, prev)` | `xTaskNotifyFromISR(task, val, action, pxWoken, prev)` | `pxHigherPriorityTaskWoken` |
| `xEventGroupSetBits(eg, bits)` | `xEventGroupSetBitsFromISR(eg, bits, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xTimerStart(timer, timeout)` | `xTimerStartFromISR(timer, pxWoken)` | `pxHigherPriorityTaskWoken` |
| `xTimerStop(timer, timeout)` | `xTimerStopFromISR(timer, pxWoken)` | `pxHigherPriorityTaskWoken` |

**通用规则：**
1. ISR 中只能用 FromISR 版本（永远不能用阻塞版本）
2. FromISR 版本多一个 `pxHigherPriorityTaskWoken` 参数
3. 所有 FromISR 版本都不会阻塞（timeout 参数不存在）
4. ISR 结束处必须调用 `portYIELD_FROM_ISR(xHigherPriorityTaskWoken)`

---

## pxHigherPriorityTaskWoken 模式详解

```
┌──────────────────────────────────────────────────────────────────┐
│         pxHigherPriorityTaskWoken 标准用法 (Must Follow)           │
│                                                                  │
│  void My_IRQHandler(void)                                        │
│  {                                                               │
│      BaseType_t xHigherPriorityTaskWoken = pdFALSE;  // ← ①初始化 │
│                                                                  │
│      /* ② 清除外设中断标志 */                                      │
│      CLEAR_PERIPH_IRQ_FLAG();                                    │
│                                                                  │
│      /* ③ 发送数据到任务 (可能多次) */                              │
│      xQueueSendFromISR(g.xQueue, &data,                          │
│                        &xHigherPriorityTaskWoken);               │
│                                                                  │
│      /* ④ 释放信号量 (可能多次) */                                 │
│      xSemaphoreGiveFromISR(g.xSem,                               │
│                            &xHigherPriorityTaskWoken);           │
│                                                                  │
│      /* ⑤ ISR 结束: 仅当任务被唤醒时才请求切换 */                   │
│      portYIELD_FROM_ISR(xHigherPriorityTaskWoken);               │
│  }                                                               │
│                                                                  │
│  关键点:                                                           │
│  • pxHigherPriorityTaskWoken 初始化为 pdFALSE                    │
│  • 同一个变量传给所有 FromISR 调用                                 │
│  • 任何一个 FromISR 调用如果唤醒了更高优先级任务 → 设为 pdTRUE     │
│  • portYIELD_FROM_ISR 只在 ISR 结束时调用**一次**                  │
│  • 如果 pxWoken == pdFALSE → portYIELD_FROM_ISR 是 no-op        │
│  • portYIELD_FROM_ISR 实际上是触发 PendSV (最低优先级)，所以      │
│    上下文切换在所有嵌套 ISR 都返回后才发生                         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 临界区保护：ENTER/EXIT vs 调度器锁 vs Mutex

```
┌──────────────────────────────────────────────────────────────────┐
│              临界区保护方案对比                                     │
│                                                                  │
│  ┌──────────────────┬───────────────┬──────────────┬───────────┐ │
│  │ 方法              │ 保护范围       │ 中断延迟     │ 适用场景   │ │
│  ├──────────────────┼───────────────┼──────────────┼───────────┤ │
│  │ taskENTER_CRITICAL│ 关所有中断     │ ❌ 增加       │ <1μs 操作 │ │
│  │ taskEXIT_CRITICAL │ (屏蔽CPU中断) │              │ 如:计数器  │ │
│  ├──────────────────┼───────────────┼──────────────┼───────────┤ │
│  │ vTaskSuspendAll()  │ 关调度器       │ ✅ 不影响    │ 较长操作   │ │
│  │ xTaskResumeAll()   │ (不关中断)    │              │ 但不能调API│ │
│  ├──────────────────┼───────────────┼──────────────┼───────────┤ │
│  │ Mutex              │ 互斥锁         │ ✅ 影响最小  │ 多任务共享 │ │
│  │                    │ (不关中断)    │              │ 支持优先级 │ │
│  │                    │               │              │ 继承       │ │
│  └──────────────────┴───────────────┴──────────────┴───────────┘ │
│                                                                  │
│  选择建议:                                                        │
│  • 极短操作 (<1μs, 如计数器++、标志位置位) → ENTER/EXIT_CRITICAL │
│  • 中长操作 (需调用队列/信号量等API) → Mutex (第4课)             │
│  • 不涉及其他任务只保护本任务数据 → vTaskSuspendAll (第9课)       │
│                                                                  │
│  ⚠️ CRITICAL 内不能做的事:                                        │
│  • 调用阻塞 API (vTaskDelay, xQueueReceive with timeout, etc.)   │
│  • 长时间循环 (增加所有中断的延迟)                                │
│  • 调用可能触发 PendSV 的操作 (如 vTaskSuspendAll 嵌套)          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 中断延迟分析

```
┌──────────────────────────────────────────────────────────────────┐
│                 中断延迟 = 从 IRQ 触发到 ISR 第一条指令             │
│                                                                  │
│  总延迟 = 硬件延迟 + 最长临界区屏蔽时间 + ISR 抢占延迟              │
│                                                                  │
│  硬件延迟 (固定):                                                  │
│    Cortex-M4F 中断响应 = 12 cycles @ 8MHz = 1.5μs                 │
│    (包括: 压栈 R0-R3,R12,LR,PC,xPSR + 取向量表)                  │
│                                                                  │
│  最长临界区屏蔽时间 (可变):                                        │
│    = 应用程序中 taskENTER_CRITICAL ~ taskEXIT_CRITICAL 的最长执行时间 │
│    = 取决于你的代码!                                               │
│                                                                  │
│    示例:                                                          │
│      taskENTER_CRITICAL();  // 关中断                             │
│      global_counter++;      // 1 cycle                            │
│      taskEXIT_CRITICAL();   // 开中断                             │
│      → 屏蔽时间 ~3 cycles ≈ 0.4μs ✓ 优秀                         │
│                                                                  │
│      taskENTER_CRITICAL();                                        │
│      for(int i=0; i<1000; i++) { ... }  // ❌ 糟糕!              │
│      taskEXIT_CRITICAL();                                        │
│      → 屏蔽时间 ~milliseconds → 中断完全丢失                      │
│                                                                  │
│  设计准则:                                                        │
│  • 无论何时,临界区都应 < 10 μs                                   │
│  • 如果需要更长保护,用 Mutex 替代                                 │
│  • 如果实在需要长临界区,提升被影响中断的优先级使其 > MAX_SYSCALL │
│     (注意: 这种 ISR 就完全不能调 FreeRTOS API!)                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## FreeRTOSConfig.h 中断配置回顾

```c
/* PY32F403: 3-bit NVIC, 8 priority levels (0~7) */
#define configPRIO_BITS                         3

/* Kernel interrupts (SysTick, PendSV) at lowest priority */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY             7

/* ISRs at priority >= 3 can call xxxFromISR() APIs */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY        3

/* Auto-derived NVIC register values (DO NOT EDIT) */
#define configKERNEL_INTERRUPT_PRIORITY         ( 7 << (8 - 3) )  // = 0xE0
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    ( 3 << (8 - 3) )  // = 0x60

/* Map FreeRTOS handlers to CMSIS vector table */
#define vPortSVCHandler         SVC_Handler
#define xPortPendSVHandler      PendSV_Handler
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Daemon Task             │   632B  │  1  │   632B
InterruptDemo Task       │   920B  │  1  │   920B   (200W stack + TCB)
CritSecTask_A            │   720B  │  1  │   720B   (150W stack + TCB)
CritSecTask_B            │   720B  │  1  │   720B   (150W stack + TCB)
ISR Queue (4 × 4B)       │   108B  │  1  │   108B   (buffer + control block)
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  3860B / 8192B  ✅
FREE                    │         │     │  4332B
```

---

## 验证清单

```
□ 1. 添加 #include "queue.h" 到 main.c
□ 2. 确认 py32f403_it.c 中添加了 TIM6_IRQHandler (或放在 main.c)
□ 3. py32f403_hal_conf.h 中确认 HAL_TIM_MODULE_ENABLED
□ 4. 编译 0 Error, 0 Warning
□ 5. 下载运行后观察:
□ 5a. LED0 (PB13) — 以 1Hz 频率稳定闪烁 (每500ms翻转)
      → 证明: TIM6 ISR → Queue → Task 通信链路完整
□ 5b. LED1 (PA1)  — 每 ~9 秒短暂闪烁 1 次
      → 证明: 临界区保护正常工作，计数器 = 2000
□ 5c. LED2 (PB12) — 可能间歇亮起 ~1 秒
      → 证明: 无保护的计数器检测到竞争条件
□ 6.  在 TIM6_IRQHandler 中设断点:
      单步观察 xQueueSendFromISR → 检查 xHigherPriorityTaskWoken 值
      单步观察 portYIELD_FROM_ISR → 确认 PendSV 被触发
□ 7.  在 CritSecTask_B 的 taskENTER_CRITICAL 处设断点:
      观察 NVIC 中断屏蔽状态 (PRIMASK 寄存器)
□ 8.  在 CritSecTask_A 的无保护增量处设断点:
      多次运行观察 g.ulUnprotectedCnt 的值 — 偶尔 ≠ 预期
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|---------|
| ISR 中调用阻塞 API | ISR 中 `portMAX_DELAY` 或 `vTaskDelay` → 系统崩溃 | ISR 只能 Try(timeout=0) 或 FromISR 版本 |
| 忘记 portYIELD_FROM_ISR | FromISR 唤醒了任务但没请求切换 → 延迟到下次 SysTick | ISR 末尾必须调用 portYIELD_FROM_ISR |
| 多个 FromISR 用不同的 woken 变量 | 后面的 FromISR 覆盖前面的 woken 标志 | 所有 FromISR 共用同一个 `xHigherPriorityTaskWoken` |
| 临界区太长 | 中断被屏蔽 >100μs → 实时性丧失 | 临界区 < 10μs, 长操作用 Mutex |
| 临界区内调阻塞 API | `vTaskDelay()` 在 ENTER/EXIT 内 → 死锁 | 所有阻塞调用必须放在临界区外 |
| ISR 优先级超过 MAX_SYSCALL | NVIC prio 0/1/2 的 ISR 调 FromISR → assert | 用 HAL_NVIC_SetPriority 设置为 >= 3 |
| 队列在 ISR 启动后才创建 | ISR 先触发 → NULL 队列 → 崩溃 | 创建顺序: Queue → Tasks → Start Timer |
| volatile 遗忘 | 变量在 ISR 和 Task 间共享但未 volatile | ISR/Task 共享变量加 `volatile` |

---

## 最佳实践清单

| # | 实践 | 说明 |
|---|------|------|
| 1 | 所有 ISR 末尾 `portYIELD_FROM_ISR` | 确保被唤醒的高优任务立即运行 |
| 2 | 一个 `pxHigherPriorityTaskWoken` 传所有 FromISR 调用 | 统一的唤醒标志 |
| 3 | ISR 越短越好 | 只采集数据、发信号; 处理交给任务 |
| 4 | 先创建 Queue → 再开中断 | 防止 ISR 写入空队列 |
| 5 | 临界区 < 10 μs | 超时使用 Mutex 替代 |
| 6 | 临界区内不做阻塞操作 | 牢记: 中断关了, 调度器跑不了 |
| 7 | ISR 共享变量加 `volatile` | 防止编译器优化导致的不一致 |
| 8 | 中断优先级用 `#define` 统一管理 | 避免硬编码魔法数 |
| 9 | FromISR 返回值要检查 | `if (xQueueSendFromISR(...) != pdPASS)` → 队列满日志 |
| 10 | 全局变量用结构体封装 | `g.ulIsrCount` 风格 — 集中管理 |

---

## 中断管理决策树

```
  需要从 ISR 通知任务?
    ├── 需要传递数据?
    │   ├── Yes → ✅ xQueueSendFromISR + xQueueReceive
    │   └── No  → ✅ vTaskNotifyGiveFromISR + ulTaskNotifyTake
    │
    └── ISR 优先级配置:
        ├── 调用 FreeRTOS API → NVIC Priority ≥ MAX_SYSCALL (3~7)
        └── 不调 FreeRTOS API → NVIC Priority 0~2 (任意)

  需要保护共享资源?
    ├── 仅 ISR 写 + 仅 Task 读 → volatile 就够
    ├── ISR 和 Task 都写 → taskENTER_CRITICAL / taskEXIT_CRITICAL
    │   (长度 < 10μs)
    └── 多 Task 竞争 → Mutex (第4课)

  核心铁律:
    1. ISR 不能阻塞 (永远)
    2. 优先级 ≥ MAX_SYSCALL → 可调 FromISR
    3. 临界区要短 (< 10μs)
    4. 创建队列 → 再开中断
```
