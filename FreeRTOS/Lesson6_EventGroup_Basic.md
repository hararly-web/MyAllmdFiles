# 第6课 代码实战 — 基础版：事件组 (OR / AND / Barrier 同步)

> **目标：** 掌握 xEventGroupCreate / xEventGroupSetBits / xEventGroupWaitBits / xEventGroupSync
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第5课软件定时器已完成，理解阻塞等待和回调约束
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（已配好）

---

## 核心概念：事件组 = 24 位通知板

```
┌──────────────────────────────────────────────────────────────────┐
│          事件组 vs 信号量 vs 队列                                  │
│                                                                  │
│  信号量 (Semaphore):                                              │
│    二进制状态, 计数型                                             │
│    Take = 消耗 (计数-1), Give = 释放 (计数+1)                     │
│    "有一个空位" / "有一个数据"                                    │
│                                                                  │
│  队列 (Queue):                                                   │
│    FIFO管道, 数据拷贝                                             │
│    Send → Queue → Receive                                       │
│    "这是给你的数据"                                               │
│                                                                  │
│  事件组 (Event Group):                    ← 本课                  │
│    24-bit 位图 (bitmap)                                           │
│    每个 bit 独立, 可同时等待多个 bit                                │
│    SetBits = 置1, WaitBits = 等待                                │
│    bit 不会自动清除! (除非指定 CLEAR_ON_EXIT)                      │
│    "事件A发生了吗？事件B呢？事件C呢？"                              │
│                                                                  │
│    ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐  │
│    │23│22│21│20│19│18│17│16│15│14│13│12│11│10│ 9│ 8│ 7│ 6│ 5│  │
│    ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤  │
│    │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
│    └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘  │
│                                          └─────────┬─────────┘   │
│                                          高8位: FreeRTOS内部使用  │
│                                          低24位: 应用程序可用     │
│                                                                  │
│  关键特性:                                                        │
│    - bit 是持久(persistent)的, 不会自动清零                        │
│    - 可以同时等待 N 个 bit: AND (全等) 或 OR (任一)                 │
│    - 等待时可以指定 "满足条件后自动清除"                            │
│    - xEventGroupSync = 多任务集合点 (Barrier)                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 6 Basic: Event Groups
  *
  *          Demonstrates:
  *          - xEventGroupCreate()     — create a 24-bit event flag group
  *          - xEventGroupSetBits()    — set one or more event bits
  *          - xEventGroupWaitBits()   — block until bits match (AND or OR)
  *          - xEventGroupSync()       — multi-task barrier/rendezvous
  *          - Clear-on-exit behavior  — auto-clear matched bits on wakeup
  *
  *          Architecture:
  *
  *            [Concept 1: OR Sync]
  *              ProducerA/B/C set bits at different rates
  *              OrHandler waits on ANY bit, processes, clears
  *
  *            [Concept 2: AND Sync]
  *              InitSeq sets 3 bits sequentially
  *              AndHandler waits on ALL bits, then fires once
  *
  *            [Concept 3: Barrier (Sync Point)]
  *              WorkerA and WorkerB rendezvous every 2 seconds
  *              xEventGroupSync ensures both arrive before proceeding
  *
  *          Event bit allocation (24-bit available, bits 0~23):
  *            BIT0 = SensorA data ready    (OR wait group)
  *            BIT1 = SensorB data ready    (OR wait group)
  *            BIT2 = SensorC data ready    (OR wait group)
  *            BIT3 = Clock init complete   (AND wait group)
  *            BIT4 = GPIO init complete    (AND wait group)
  *            BIT5 = Peripheral init done  (AND wait group)
  *            BIT6 = WorkerA sync flag     (Barrier group)
  *            BIT7 = WorkerB sync flag     (Barrier group)
  *
  *          LED assignment:
  *            LED0 (PB13): OR handler — blinks when ANY sensor reports
  *            LED1 (PA1):  AND handler — toggles on ALL-init-complete cycle
  *            LED2 (PB12): Barrier sync — toggles on each sync rendezvous
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "event_groups.h"

/* Private defines ------------------------------------------------------------*/

/* Event bit masks — one bit per event source */
#define BIT_SENSOR_A        (1UL << 0)    /* ProducerA sets this every 200ms  */
#define BIT_SENSOR_B        (1UL << 1)    /* ProducerB sets this every 350ms  */
#define BIT_SENSOR_C        (1UL << 2)    /* ProducerC sets this every 500ms  */
#define BIT_INIT_CLK        (1UL << 3)    /* Clock init phase done            */
#define BIT_INIT_GPIO       (1UL << 4)    /* GPIO init phase done             */
#define BIT_INIT_PERIPH     (1UL << 5)    /* Peripheral init phase done       */
#define BIT_SYNC_A          (1UL << 6)    /* WorkerA barrier flag             */
#define BIT_SYNC_B          (1UL << 7)    /* WorkerB barrier flag             */

/* Composite masks for Wait operations */
#define OR_SENSOR_MASK      (BIT_SENSOR_A | BIT_SENSOR_B | BIT_SENSOR_C)
#define AND_INIT_MASK       (BIT_INIT_CLK | BIT_INIT_GPIO | BIT_INIT_PERIPH)
#define BARRIER_MASK        (BIT_SYNC_A | BIT_SYNC_B)

/* Timing (ms) */
#define PROD_A_PERIOD_MS    200U    /* Fastest producer     */
#define PROD_B_PERIOD_MS    350U    /* Medium producer      */
#define PROD_C_PERIOD_MS    500U    /* Slowest producer     */
#define INIT_STEP_DELAY_MS  600U    /* Delay between init steps */
#define BARRIER_PERIOD_MS   2000U   /* Sync barrier interval    */

/* Task priorities */
#define PRIO_OR_HANDLER     3U      /* OR handler: process sensors ASAP   */
#define PRIO_AND_HANDLER    3U      /* AND handler: fire when all ready   */
#define PRIO_PRODUCER       2U      /* All producers at same priority     */
#define PRIO_WORKER         1U      /* Barrier workers at lowest priority */

/* Task stack sizes (words) */
#define STACK_OR_HANDLER    200U
#define STACK_AND_HANDLER   200U
#define STACK_PRODUCER      150U
#define STACK_INIT_SEQ      150U
#define STACK_WORKER        150U

/* LED pin definitions */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — single event group handle in a struct.
  */
static struct {
    EventGroupHandle_t xEventGroup; /* Handle to the 24-bit event group */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);
static void APP_LedOn(uint8_t led_id);
static void APP_LedOff(uint8_t led_id);

/* --- Concept 1: OR Synchronization Tasks ---------------------------------- */
static void ProducerA_Task(void *pvParameters);
static void ProducerB_Task(void *pvParameters);
static void ProducerC_Task(void *pvParameters);
static void OrHandler_Task(void *pvParameters);

/* --- Concept 2: AND Synchronization Tasks --------------------------------- */
static void InitSeq_Task(void *pvParameters);
static void AndHandler_Task(void *pvParameters);

/* --- Concept 3: Barrier Sync Tasks ---------------------------------------- */
static void WorkerA_Task(void *pvParameters);
static void WorkerB_Task(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

/**
  * @brief  Creates 1 event group + 8 tasks demonstrating 3 sync patterns.
  * @note   Event group bits are PERSISTENT — they stay set until explicitly
  *          cleared. This is unlike semaphores which are "consumed" on Take.
  *
  *          When using xEventGroupWaitBits with xClearOnExit=pdTRUE:
  *            AND mode: ALL waited-for bits are cleared on successful return
  *            OR mode:  ALL waited-for bits are cleared on successful return
  *
  *          This auto-clear prevents having to manually clear bits.
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Create event group.
     * No parameters — always creates a 24-bit event group.
     * Bits 24~31 are reserved for internal FreeRTOS use.
     */
    g.xEventGroup = xEventGroupCreate();
    configASSERT(g.xEventGroup != NULL);

    /* --- Concept 1: OR Sync (3 producers + 1 handler) --- */
    xReturned = xTaskCreate(ProducerA_Task, "ProdA",
                            STACK_PRODUCER, NULL, PRIO_PRODUCER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(ProducerB_Task, "ProdB",
                            STACK_PRODUCER, NULL, PRIO_PRODUCER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(ProducerC_Task, "ProdC",
                            STACK_PRODUCER, NULL, PRIO_PRODUCER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(OrHandler_Task, "OrHdlr",
                            STACK_OR_HANDLER, NULL, PRIO_OR_HANDLER, NULL);
    configASSERT(xReturned == pdPASS);

    /* --- Concept 2: AND Sync (1 init sequence + 1 handler) --- */
    xReturned = xTaskCreate(InitSeq_Task, "InitSeq",
                            STACK_INIT_SEQ, NULL, PRIO_PRODUCER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(AndHandler_Task, "AndHdlr",
                            STACK_AND_HANDLER, NULL, PRIO_AND_HANDLER, NULL);
    configASSERT(xReturned == pdPASS);

    /* --- Concept 3: Barrier Sync (2 workers) --- */
    xReturned = xTaskCreate(WorkerA_Task, "WorkerA",
                            STACK_WORKER, NULL, PRIO_WORKER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(WorkerB_Task, "WorkerB",
                            STACK_WORKER, NULL, PRIO_WORKER, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();
    while (1) {}
}

/* ========================================================================== */
/*  Concept 1: OR Synchronization                                               */
/* ========================================================================== */

/**
  * @brief  ProducerA — sets BIT_SENSOR_A every 200ms (fastest).
  */
static void ProducerA_Task(void *pvParameters)
{
    (void)pvParameters;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * xEventGroupSetBits:
         *   - Sets the specified bits to 1
         *   - Returns the NEW value of the event group (after setting)
         *   - If a higher-priority task was waiting on these bits,
         *     it will be woken AFTER this call returns
         *
         *   This is NON-BLOCKING — always returns immediately.
         */
        xEventGroupSetBits(g.xEventGroup, BIT_SENSOR_A);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(PROD_A_PERIOD_MS));
    }
}

/**
  * @brief  ProducerB — sets BIT_SENSOR_B every 350ms.
  */
static void ProducerB_Task(void *pvParameters)
{
    (void)pvParameters;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        xEventGroupSetBits(g.xEventGroup, BIT_SENSOR_B);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(PROD_B_PERIOD_MS));
    }
}

/**
  * @brief  ProducerC — sets BIT_SENSOR_C every 500ms (slowest).
  */
static void ProducerC_Task(void *pvParameters)
{
    (void)pvParameters;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        xEventGroupSetBits(g.xEventGroup, BIT_SENSOR_C);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(PROD_C_PERIOD_MS));
    }
}

/**
  * @brief  OR Handler — wakes on ANY sensor bit, processes, clears on exit.
  * @note   xEventGroupWaitBits with xWaitForAllBits=pdFALSE:
  *          Wakes if ANY of the bits in OR_SENSOR_MASK is set.
  *          Returns the event group value at the time of wakeup.
  *
  *          xClearOnExit=pdTRUE:
  *          On successful return, ALL bits in uxBitsToWaitFor
  *          are atomically cleared. This prevents "double-processing"
  *          the same event.
  *
  *          Return value includes which bit(s) triggered the wakeup.
  *          Use bitwise AND to check individual bits.
  *
  *          If multiple bits are set simultaneously, OR handler
  *          wakes once and clears ALL of them. The total events
  *          per interval is shown by LED0 flash count.
  */
static void OrHandler_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Block until ANY sensor bit is set.
         * portMAX_DELAY = wait forever.
         * pdFALSE        = OR logic (wake on any one bit)
         * pdTRUE         = clear matched bits on exit
         */
        EventBits_t uxBits = xEventGroupWaitBits(
                                 g.xEventGroup,
                                 OR_SENSOR_MASK,  /* bits to wait for  */
                                 pdTRUE,          /* clear on exit     */
                                 pdFALSE,         /* OR (not AND)      */
                                 portMAX_DELAY);  /* infinite timeout  */

        /*
         * Process each sensor that triggered.
         * Multiple bits may have been set simultaneously —
         * especially since the Handler runs at higher priority
         * and may accumulate events before processing.
         */
        uint8_t event_count = 0;

        if (uxBits & BIT_SENSOR_A) { event_count++; }
        if (uxBits & BIT_SENSOR_B) { event_count++; }
        if (uxBits & BIT_SENSOR_C) { event_count++; }

        /*
         * Visual feedback: toggle LED0 once per triggered sensor.
         * More events → more toggles → faster visual pattern.
         */
        for (uint8_t i = 0; i < event_count; i++)
        {
            APP_LedToggle(0);
            vTaskDelay(pdMS_TO_TICKS(80));
        }
    }
}

/* ========================================================================== */
/*  Concept 2: AND Synchronization                                              */
/* ========================================================================== */

/**
  * @brief  Init Sequence — sets 3 init bits sequentially with delays.
  * @note   Simulates a multi-stage initialization:
  *          Phase 1 (600ms): Clock init → set BIT_INIT_CLK
  *          Phase 2 (600ms): GPIO init  → set BIT_INIT_GPIO
  *          Phase 3 (600ms): Periph init→ set BIT_INIT_PERIPH
  *
  *          The AND handler waits for ALL THREE before proceeding.
  *          After setting all, this task deletes itself.
  */
static void InitSeq_Task(void *pvParameters)
{
    (void)pvParameters;

    /* Simulate Phase 1: Clock init */
    vTaskDelay(pdMS_TO_TICKS(INIT_STEP_DELAY_MS));
    xEventGroupSetBits(g.xEventGroup, BIT_INIT_CLK);

    /* Simulate Phase 2: GPIO init */
    vTaskDelay(pdMS_TO_TICKS(INIT_STEP_DELAY_MS));
    xEventGroupSetBits(g.xEventGroup, BIT_INIT_GPIO);

    /* Simulate Phase 3: Peripheral init */
    vTaskDelay(pdMS_TO_TICKS(INIT_STEP_DELAY_MS));
    xEventGroupSetBits(g.xEventGroup, BIT_INIT_PERIPH);

    /* Init complete — this task has fulfilled its purpose */
    vTaskDelete(NULL);

    /* Never reaches here */
}

/**
  * @brief  AND Handler — wakes ONLY when ALL 3 init bits are set.
  * @note   xEventGroupWaitBits with xWaitForAllBits=pdTRUE:
  *          Blocks until ALL bits in AND_INIT_MASK are set.
  *          This is like "wait for ALL dependencies to be ready."
  *
  *          With xClearOnExit=pdTRUE, once all bits are set,
  *          the handler wakes, clears them, and executes.
  *
  *          This pattern is used in real firmware for:
  *          - Waiting for all hardware modules to init
  *          - Waiting for N sensors to calibrate
  *          - Waiting for multiple communication links to establish
  */
static void AndHandler_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Block until ALL init bits are set.
         * pdTRUE for xWaitForAllBits = AND logic.
         * pdTRUE for xClearOnExit = clear all 3 bits after wakeup.
         */
        (void)xEventGroupWaitBits(
                  g.xEventGroup,
                  AND_INIT_MASK,    /* bits to wait for  */
                  pdTRUE,           /* clear on exit     */
                  pdTRUE,           /* AND (not OR)      */
                  portMAX_DELAY);   /* infinite timeout  */

        /*
         * All 3 init phases complete!
         * Toggle LED1 in a distinctive pattern to confirm.
         */
        for (uint8_t i = 0; i < 3; i++)
        {
            APP_LedToggle(1);
            vTaskDelay(pdMS_TO_TICKS(200));
        }

        /*
         * Wait a long time before the next AND cycle.
         * In production: restart init sequence, or wait for
         * a "re-init" command.
         */
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

/* ========================================================================== */
/*  Concept 3: Barrier Synchronization (Sync Point)                             */
/* ========================================================================== */

/**
  * @brief  Worker A — synchronizes with Worker B at a barrier every 2 seconds.
  * @note   xEventGroupSync() is a COMBINED OPERATION that atomically:
  *
  *          1. Sets uxBitsToSet (BIT_SYNC_A) in the event group
  *          2. Waits for uxBitsToWaitFor (BIT_SYNC_A | BIT_SYNC_B)
  *             to all be set (AND logic)
  *          3. Clears the bits before returning
  *
  *          This is THE way to implement a multi-task rendezvous
  *          (barrier) in FreeRTOS. Use when:
  *            - N tasks must all reach a point before ANY proceeds
  *            - Coordinated start of a multi-task operation
  *            - Multi-sensor snapshot: wait for all readings
  *
  *          Note: xEventGroupSync uses AND logic internally
  *          (waits for ALL specified bits). There is no OR option.
  *          This makes sense — a barrier means "everyone must arrive."
  */
static void WorkerA_Task(void *pvParameters)
{
    (void)pvParameters;

    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * Simulate "work" before the barrier.
         * Different workers take different amounts of time.
         */
        vTaskDelay(pdMS_TO_TICKS(300));

        /*
         * Rendezvous point — arrive and wait for WorkerB.
         *
         * xEventGroupSync parameters:
         *   g.xEventGroup:  the event group
         *   BIT_SYNC_A:     bits THIS task sets (marks "I'm here")
         *   BARRIER_MASK:   bits to wait for (BIT_SYNC_A | BIT_SYNC_B)
         *   portMAX_DELAY:  wait forever for the other worker
         *
         * Returns: the event group value before bits were cleared
         */
        (void)xEventGroupSync(
                  g.xEventGroup,
                  BIT_SYNC_A,       /* set this to say "A arrived"       */
                  BARRIER_MASK,     /* wait for both A and B             */
                  portMAX_DELAY);   /* wait forever for B                */

        /*
         * Beyond the barrier — both workers reached this point.
         * Toggle LED2 to show synchronization.
         */
        APP_LedToggle(2);

        /* Wait before next sync cycle */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(BARRIER_PERIOD_MS));
    }
}

/**
  * @brief  Worker B — mirror of Worker A, but arrives at a different time.
  */
static void WorkerB_Task(void *pvParameters)
{
    (void)pvParameters;

    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * WorkerB does more "work" before barrier (800ms vs 300ms).
         * This means WorkerA arrives FIRST and must WAIT for WorkerB.
         * The barrier ensures no one proceeds until both arrive.
         */
        vTaskDelay(pdMS_TO_TICKS(800));

        /*
         * Arrive at the barrier.
         * WorkerA is already waiting — when WorkerB calls this,
         * the condition BARRIER_MASK is met → BOTH unblock.
         */
        (void)xEventGroupSync(
                  g.xEventGroup,
                  BIT_SYNC_B,       /* set this to say "B arrived"       */
                  BARRIER_MASK,     /* wait for both A and B             */
                  portMAX_DELAY);

        /* Beyond the barrier */
        APP_LedToggle(2);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(BARRIER_PERIOD_MS));
    }
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

    /* LED0 — OR handler activity */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* LED1 — AND handler activity */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* LED2 — Barrier sync indicator */
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

## Event Group API 速查

| API | 功能 | 关键参数 |
|-----|------|---------|
| `xEventGroupCreate()` | 创建事件组 | 无参数 |
| `xEventGroupSetBits(group, bits)` | 置位 | 任务中调用, 非阻塞 |
| `xEventGroupSetBitsFromISR(group, bits, pxFlag)` | 置位 (ISR) | ISR中调用, 需要标志位 |
| `xEventGroupWaitBits(group, bits, clear, and_logic, timeout)` | 等待位 | **核心:** clear + AND/OR |
| `xEventGroupSync(group, set, wait, timeout)` | 同步点 | 原子操作: SetBits + AND_WaitBits + Clear |
| `xEventGroupClearBits(group, bits)` | 手动清除 | 不等待, 直接清零 |
| `xEventGroupGetBits(group)` | 读取当前值 | 非阻塞 |
| `xEventGroupGetBitsFromISR(group)` | 读取 (ISR) | ISR中调用 |
| `vEventGroupDelete(group)` | 删除事件组 | 确保无任务在等待 |

---

## WaitBits 参数组合矩阵

```
┌──────────────────────────────────────────────────────────────────┐
│  xEventGroupWaitBits(xGroup, uxBits, xClear, xAnd, xTimeout)      │
│                                                                  │
│  xAnd = pdFALSE (OR):   任一bit=1就唤醒                            │
│  xAnd = pdTRUE  (AND):  全部bit=1才唤醒                            │
│                                                                  │
│  xClear = pdFALSE:      bits保持置位, 可被后续任务看到             │
│  xClear = pdTRUE:       返回前原子清除所有 bit                    │
│                                                                  │
│  ┌──────────┬───────────┬──────────────────────────────────┐     │
│  │ xAnd     │ xClear    │ 行为                              │     │
│  ├──────────┼───────────┼──────────────────────────────────┤     │
│  │ pdFALSE  │ pdFALSE   │ 任一=1→唤醒, bits保持不变          │     │
│  │ pdFALSE  │ pdTRUE    │ 任一=1→唤醒, 所有等待的bits清零    │     │
│  │ pdTRUE   │ pdFALSE   │ 全部=1→唤醒, bits保持不变          │     │
│  │ pdTRUE   │ pdTRUE    │ 全部=1→唤醒, 所有等待的bits清零    │     │
│  └──────────┴───────────┴──────────────────────────────────┘     │
│                                                                  │
│  返回值: 唤醒时刻事件组的bit值 (如果xClear=pdTRUE, 返回清除前的值) │
│          → 用 (uxBits & BIT_X) 判断具体哪个/哪些bit触发          │
└──────────────────────────────────────────────────────────────────┘
```

---

## xEventGroupSync 详解

```
┌──────────────────────────────────────────────────────────────────────────┐
│  xEventGroupSync = 多任务集合点 (Barrier / Rendezvous)                     │
│                                                                          │
│  EventBits_t xEventGroupSync(                                            │
│      EventGroupHandle_t xEventGroup,                                     │
│      EventBits_t uxBitsToSet,      /* 本任务设置哪些 bit                   │ */
│      EventBits_t uxBitsToWaitFor,  /* 等待哪些 bit 全为1 (AND逻辑)        │ */
│      TickType_t xTicksToWait       /* 超时时间                            */│
│  );                                                                      │
│                                                                          │
│  时序 (WorkerA + WorkerB):                                                │
│                                                                          │
│    WorkerA:                                                              │
│      ...pre-work(300ms)...                                               │
│      xEventGroupSync(BIT_A, BIT_A|BIT_B, ∞)                             │
│        │                                                                 │
│        ├── SetBits(BIT_A)           ← "A 到了!"                          │
│        ├── WaitBits(BIT_A|BIT_B)    ← "等 B 到..."                       │
│        │   BIT_A=1, BIT_B=0 → 不满足 → 阻塞                              │
│        │                     │                                          │
│        │                     │  WorkerB 到达:                             │
│        │                     │  xEventGroupSync(BIT_B, BIT_A|BIT_B, ∞)  │
│        │                     │    SetBits(BIT_B)                         │
│        │                     │    WaitBits → BIT_A=1, BIT_B=1 → ✅      │
│        │                     │                                          │
│        │   BIT_A=1, BIT_B=1  │                                          │
│        └── 唤醒! ←───────────┘                                          │
│          ClearBits(BIT_A|BIT_B)    ← 清理, 为下次sync准备                │
│          ...post-work...                                                  │
│                                                                          │
│   关键: sync 使用 AND 语义 — 所有参与者必须到达                            │
│         sync 结束后自动清除 — 为下一轮做准备                               │
│         所有参与者同时被唤醒 — 真正的"同时出发"                            │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 事件组 vs 信号量 决策对比

```
  需要 N 个任务全部就绪才继续?
    ├── Yes → 事件组 AND Wait (或 xEventGroupSync)
    │         例: 5个传感器全部校准完成 → 开始采集
    │
    └── No → 需要等待任一事件发生?
              ├── Yes → 事件组 OR Wait
              │         例: 任一按键按下 → 唤醒处理
              │
              └── No → 需要计数/资源池?
                        ├── Yes → 计数信号量
                        │         例: 3个DMA通道可用
                        │
                        └── No → 简单二值同步?
                                  └── Yes → 二值信号量
                                            例: ISR通知Task "数据传输完"
```

---

## FreeRTOSConfig.h 配置

```c
/* FreeRTOSConfig.h — 已配置, 确认即可 */
#define configUSE_EVENT_GROUPS    1     /* 必须为 1 (第1课已配) */

/* main.c 需要 include */
#include "event_groups.h"
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Daemon Task             │   632B  │  1  │   632B
OrHandler Task          │   920B  │  1  │   920B   (200W stack + TCB)
AndHandler Task         │   920B  │  1  │   920B   (200W stack + TCB)
ProducerA/B/C (×3)      │  2160B  │  1  │  2160B   (3 × 720B)
InitSeq Task            │   720B  │  1  │   720B   (150W stack + TCB)
WorkerA/B (×2)          │  1440B  │  1  │  1440B   (2 × 720B)
Event Group             │    60B  │  1  │    60B
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  7532B / 8192B  ✅
```

---

## 验证清单

```
□ 1. 添加 #include "event_groups.h" 到 main.c
□ 2. 编译 0 Error, 0 Warning
□ 3. 下载运行后观察:
□ 3a. LED0 (PB13) — 不规则闪烁, 快闪=多个传感器同时就绪
      → 证明: OR Handler 处理 SensorA/B/C 事件
□ 3b. LED1 (PA1)  — 约 1.8s 后连闪 3 次, 然后每 5s 重复
      → 证明: AND Handler 等齐 BIT3|BIT4|BIT5 后才触发
□ 3c. LED2 (PB12) — 约每 2s 翻转 2 次 (两个Worker同时toggle)
      → 证明: xEventGroupSync Barrier 正常工作
□ 4.  在 OrHandler 的 xEventGroupWaitBits 处设断点:
      查看 uxBits 返回值, 对比 BIT_SENSOR_A/B/C
□ 5.  在 AndHandler 的 xEventGroupWaitBits 处设断点:
      确认 BIT3=BIT4=BIT5=1 时才会通过
□ 6.  在 WorkerA 的 xEventGroupSync 后设断点:
      确认 WorkerB 也同时通过了 (查看LED2变化)
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|---------|
| 忘记 Bit 持久性 | SetBits 后 bit 永远置1 (除非手动清除) | WaitBits 用 xClearOnExit=pdTRUE |
| AND/OR 参数搞反 | 默认 OR, 需要 AND 时必须显式指定 | `xWaitForAllBits = pdTRUE` |
| 24位限制 | 高8位 FreeRTOS 内部用, 用户可用低24位 | `#define BIT_X (1UL << N)` N<24 |
| 多位同时返回处理 | AND 等待返回时所有位都是1, OR 返回时可能多位是1 | 遍历检查每个bit |
| xEventGroupSync 超时 | 有 Worker 没到达, 超时的 Worker 继续执行 (不同步!) | 检查返回值, 超时=失败 |
| ISR 中 SetBits | 不能用 xEventGroupSetBits | 用 `xEventGroupSetBitsFromISR()` + 标志位 |
| WaitBits 后不清除 | bit 一直为1, 下次 WaitBits 立即返回 (假触发!) | OR 等待几乎必用 xClearOnExit=pdTRUE |
