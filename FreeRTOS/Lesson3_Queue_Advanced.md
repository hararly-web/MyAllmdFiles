# 第3课 代码实战 — 进阶版：邮箱模式 + 队列集 + 队列注册表

> **目标：** 掌握 xQueueOverwrite（邮箱）、xQueueSelectFromSet（队列集）、vQueueAddToRegistry（注册表）的用法
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第3课基础版 Producer-Consumer 已完成，理解队列阻塞与超时机制

---

## 三个概念的定位

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  邮箱 (Mailbox)    → "只关心最新值"  → xQueueOverwrite          │
│  队列集 (Queue Set) → "同时等多个队列" → xQueueSelectFromSet     │
│  注册表 (Registry)  → "给队列贴标签"  → vQueueAddToRegistry      │
│                                                                  │
│  基础队列 = 管道 (Pipe)    数据先进先出, 不能丢                   │
│  邮箱     = 公告板          新数据覆盖旧数据                       │
│  队列集   = select()/poll() 一个任务等 N 个数据源                  │
│  注册表   = 调试标签        给句柄分配人类可读的名字               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 3 Advanced: Mailbox + Queue Set + Queue Registry
  *
  *          Demonstrates:
  *          - xQueueOverwrite()        — single-slot mailbox, new data overwrites old
  *          - xQueueCreateSet()        — create a queue set for multi-source select
  *          - xQueueAddToSet()         — add queues to the set
  *          - xQueueSelectFromSet()    — block on multiple queues simultaneously
  *          - vQueueAddToRegistry()    — register queues for debug inspection
  *          - pcQueueGetName()         — retrieve registered name at runtime
  *
  *          Architecture:
  *
  *            [Concept 1: Mailbox]
  *              SensorGenerator ──xQueueOverwrite──▶ Mailbox(1) ──xQueueReceive──▶ SensorDisplay
  *
  *            [Concept 2: Queue Set]
  *              CmdSender  ──xQueueSend──▶ CmdQ ──╮
  *                                                 ├── QueueSet ──xQueueSelectFromSet──▶ Central
  *              KeySim     ──xQueueSend──▶ KeyQ ──╯
  *
  *            [Concept 3: Queue Registry]
  *              QueueMonitor ──pcQueueGetName/uxQueueMessagesWaiting──▶ diagnose queues
  *
  *          LED assignment:
  *            LED0 (PB13): Mailbox  — flash speed varies with sensor value
  *            LED1 (PA1):  QueueSet — toggles on key/cmd events
  *            LED2 (PB12): Registry — heartbeat + queue health indicator
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

/* Private typedef -----------------------------------------------------------*/

/**
  * @brief  Sensor data passed through the mailbox.
  * @note   Overwritten on each sensor reading — only the latest value matters.
  */
typedef struct {
    uint32_t value;     /* Simulated sensor reading (0~999) */
} SensorData_t;

/**
  * @brief  LED toggle command sent through regular queue.
  */
typedef struct {
    uint8_t  count;     /* Number of toggles (1~5)       */
    uint16_t delay_ms;  /* Toggle interval in ms (50~300)*/
} LedCmd_t;

/**
  * @brief  Simulated key event sent through regular queue.
  */
typedef struct {
    uint8_t  key_id;    /* Key identifier: 0='A', 1='B'  */
    uint8_t  event;     /* 0=press, 1=release, 2=long    */
} KeyEvent_t;

/* Private defines ------------------------------------------------------------*/

/* Queue depths */
#define MAILBOX_DEPTH       1U      /* Single slot — overwrite on each send  */
#define CMD_Q_DEPTH         4U      /* LED command queue depth               */
#define KEY_Q_DEPTH         4U      /* Key event queue depth                 */

/* Task priorities */
#define PRIO_SENSOR_DISPLAY 3U      /* Consumer: reads mailbox, drives LED0  */
#define PRIO_CENTRAL        3U      /* Consumer: selects from queue set      */
#define PRIO_SENSOR_GEN     2U      /* Producer: overwrites mailbox           */
#define PRIO_CMD_SENDER     2U      /* Producer: sends LED commands          */
#define PRIO_KEY_SIM        1U      /* Producer: simulates key events        */
#define PRIO_QUEUE_MONITOR  1U      /* Monitor: inspects queue registry      */

/* Task stack sizes (words) */
#define STACK_SENSOR_DISPLAY    200U
#define STACK_CENTRAL           256U
#define STACK_SENSOR_GEN        150U
#define STACK_CMD_SENDER        150U
#define STACK_KEY_SIM           150U
#define STACK_QUEUE_MONITOR     256U

/* Timing (ms) */
#define SENSOR_PERIOD_MS        200U    /* Sensor generates data every 200ms */
#define DISPLAY_PERIOD_MS       500U    /* Display reads mailbox every 500ms */
#define CMD_PERIOD_MS           300U    /* CmdSender sends every 300ms       */
#define MONITOR_PERIOD_MS       5000U   /* QueueMonitor checks every 5s      */
#define QUEUE_SEND_TIMEOUT_MS   100U    /* Send timeout when queue is full   */

/* LED pin definitions */
#define LED0_PORT               GPIOB
#define LED0_PIN                GPIO_PIN_13
#define LED1_PORT               GPIOA
#define LED1_PIN                GPIO_PIN_1
#define LED2_PORT               GPIOB
#define LED2_PIN                GPIO_PIN_12

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — all handles in a single struct.
  */
static struct {
    QueueHandle_t   xMailbox;       /* Mailbox handle (depth=1)              */
    QueueHandle_t   xCmdQueue;      /* LED command queue handle              */
    QueueHandle_t   xKeyQueue;      /* Key event queue handle                */
    QueueSetHandle_t xQueueSet;     /* Queue set handle (CmdQ + KeyQ)        */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);

/* --- Mailbox Tasks (Concept 1) ------------------------------------------- */
static void SensorGenerator_Task(void *pvParameters);
static void SensorDisplay_Task(void *pvParameters);

/* --- Queue Set Tasks (Concept 2) ----------------------------------------- */
static void CmdSender_Task(void *pvParameters);
static void KeySimulator_Task(void *pvParameters);
static void Central_Task(void *pvParameters);

/* --- Registry Monitor Task (Concept 3) ----------------------------------- */
static void QueueMonitor_Task(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Concept 1 — Mailbox:
     *   Create a queue of depth=1. xQueueOverwrite() will replace
     *   old data with new data without blocking.
     */
    g.xMailbox = xQueueCreate(MAILBOX_DEPTH, sizeof(SensorData_t));
    configASSERT(g.xMailbox != NULL);
    vQueueAddToRegistry(g.xMailbox, "Sensor_Mailbox");

    /*
     * Concept 2 — Regular queues for the Queue Set:
     *   These are standard queues. The Queue Set will watch both.
     */
    g.xCmdQueue = xQueueCreate(CMD_Q_DEPTH, sizeof(LedCmd_t));
    configASSERT(g.xCmdQueue != NULL);
    vQueueAddToRegistry(g.xCmdQueue, "LED_CmdQ");

    g.xKeyQueue = xQueueCreate(KEY_Q_DEPTH, sizeof(KeyEvent_t));
    configASSERT(g.xKeyQueue != NULL);
    vQueueAddToRegistry(g.xKeyQueue, "Key_EventQ");

    /*
     * Concept 2 — Queue Set:
     *   Queue set length = sum of all member queue lengths.
     *   CmdQ(4) + KeyQ(4) = 8.
     */
    g.xQueueSet = xQueueCreateSet(CMD_Q_DEPTH + KEY_Q_DEPTH);
    configASSERT(g.xQueueSet != NULL);

    /* Add both queues to the set so Central can select on them */
    xReturned = xQueueAddToSet(g.xCmdQueue, g.xQueueSet);
    configASSERT(xReturned == pdPASS);
    xReturned = xQueueAddToSet(g.xKeyQueue, g.xQueueSet);
    configASSERT(xReturned == pdPASS);

    /*
     * Create tasks.
     */

    /* Concept 1: Mailbox producer + consumer */
    xReturned = xTaskCreate(SensorGenerator_Task, "SensGen",
                            STACK_SENSOR_GEN, NULL, PRIO_SENSOR_GEN, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(SensorDisplay_Task, "SensDisp",
                            STACK_SENSOR_DISPLAY, NULL, PRIO_SENSOR_DISPLAY, NULL);
    configASSERT(xReturned == pdPASS);

    /* Concept 2: Two producers + queue set consumer */
    xReturned = xTaskCreate(CmdSender_Task, "CmdSnd",
                            STACK_CMD_SENDER, NULL, PRIO_CMD_SENDER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(KeySimulator_Task, "KeySim",
                            STACK_KEY_SIM, NULL, PRIO_KEY_SIM, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(Central_Task, "Central",
                            STACK_CENTRAL, NULL, PRIO_CENTRAL, NULL);
    configASSERT(xReturned == pdPASS);

    /* Concept 3: Registry monitor */
    xReturned = xTaskCreate(QueueMonitor_Task, "QMon",
                            STACK_QUEUE_MONITOR, NULL, PRIO_QUEUE_MONITOR, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();  /* Never returns */

    while (1) {}
}

/* ========================================================================== */
/*  Concept 1: Mailbox — xQueueOverwrite                                      */
/* ========================================================================== */

/**
  * @brief  Sensor simulator — overwrites the mailbox with latest value.
  * @note   xQueueOverwrite():
  *          - Queue empty → behaves like xQueueSend (writes and returns)
  *          - Queue full  → overwrites the OLDEST item with new data
  *          - NEVER blocks — always returns pdPASS
  *
  *          This is the key difference from xQueueSend:
  *          old data is sacrificed for new data. Perfect for
  *          sensor readings where stale data is useless.
  */
static void SensorGenerator_Task(void *pvParameters)
{
    (void)pvParameters;

    TickType_t xLastWakeTime = xTaskGetTickCount();
    uint32_t fake_sensor = 0;

    for (;;)
    {
        /*
         * Simulate a sensor reading that ramps 0→999→0.
         * In real code, this would be ADC_Read() or similar.
         */
        SensorData_t data;
        fake_sensor = (fake_sensor + 37) % 1000;  /* pseudo-linear ramp */
        data.value = fake_sensor;

        /*
         * xQueueOverwrite: always succeeds, never blocks.
         * If the Display task hasn't read the previous value yet,
         * it gets overwritten — Display always sees the latest.
         */
        (void)xQueueOverwrite(g.xMailbox, &data);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SENSOR_PERIOD_MS));
    }
}

/**
  * @brief  Reads the mailbox periodically and toggles LED0.
  * @note   The flash speed encodes the sensor value:
  *          - value=0   → slow blink (500ms)
  *          - value=999 → fast blink (50ms)
  *
  *          Important: values written between two Display reads
  *          are LOST. This is INTENTIONAL — a mailbox is for
  *          "what's the current state?", not "what happened?"
  */
static void SensorDisplay_Task(void *pvParameters)
{
    (void)pvParameters;
    SensorData_t data;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * Non-blocking read with timeout=0:
         * If the mailbox has data → read it.
         * If not (unlikely, since Generator runs faster) → use previous.
         */
        if (xQueueReceive(g.xMailbox, &data, 0) == pdPASS)
        {
            /* Data read successfully — will use data.value below */
        }
        /* else: keep previous data.value */

        /*
         * Map sensor value (0~999) to toggle interval (50ms~500ms).
         * Higher value = faster blinking → visually shows "sensor" activity.
         */
        TickType_t blink_delay = pdMS_TO_TICKS(500 - (data.value * 450 / 999));

        APP_LedToggle(0);                    /* Toggle LED0 */
        vTaskDelay(blink_delay);

        /*
         * Note: using vTaskDelay (not vTaskDelayUntil) means the period
         * includes blink_delay. For this demo, visual variety is the goal.
         */
    }
}

/* ========================================================================== */
/*  Concept 2: Queue Set — xQueueSelectFromSet                                */
/* ========================================================================== */

/**
  * @brief  Sends LED toggle commands to CmdQ at a fixed interval.
  * @note   This is a regular queue producer — no different from Lesson 3 Basic.
  */
static void CmdSender_Task(void *pvParameters)
{
    (void)pvParameters;

    /* Cycle through commands with different toggle counts */
    static const LedCmd_t cmd_seq[] = {
        {1, 100},   /* 1 toggle,  100ms — quick flash  */
        {3, 80},    /* 3 toggles, 80ms  — burst         */
        {1, 300},   /* 1 toggle,  300ms — long flash    */
        {2, 150},   /* 2 toggles, 150ms — medium        */
    };
    static const uint8_t cmd_count = sizeof(cmd_seq) / sizeof(cmd_seq[0]);
    uint8_t idx = 0;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        LedCmd_t cmd = cmd_seq[idx];
        idx = (idx + 1) % cmd_count;

        /*
         * Send with timeout — if Central is busy processing a key event,
         * CmdQ might temporarily fill up. Wait up to QUEUE_SEND_TIMEOUT_MS.
         */
        (void)xQueueSend(g.xCmdQueue, &cmd,
                         pdMS_TO_TICKS(QUEUE_SEND_TIMEOUT_MS));

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(CMD_PERIOD_MS));
    }
}

/**
  * @brief  Simulates random key press/release events.
  * @note   Key events arrive at pseudo-random intervals (200ms~800ms).
  *          This exercises the Queue Set's ability to handle
  *          unpredictable event sources alongside regular ones.
  */
static void KeySimulator_Task(void *pvParameters)
{
    (void)pvParameters;

    static const KeyEvent_t key_seq[] = {
        {0, 0},   /* Key 'A' press     */
        {1, 0},   /* Key 'B' press     */
        {0, 1},   /* Key 'A' release   */
        {0, 2},   /* Key 'A' long press*/
        {1, 1},   /* Key 'B' release   */
        {1, 0},   /* Key 'B' press     */
    };
    static const uint8_t key_count = sizeof(key_seq) / sizeof(key_seq[0]);
    uint8_t idx = 0;

    for (;;)
    {
        KeyEvent_t key = key_seq[idx];
        idx = (idx + 1) % key_count;

        (void)xQueueSend(g.xKeyQueue, &key,
                         pdMS_TO_TICKS(QUEUE_SEND_TIMEOUT_MS));

        /* Pseudo-random delay: 200ms + idx*100ms → 200~700ms */
        vTaskDelay(pdMS_TO_TICKS(200 + (idx * 100)));
    }
}

/**
  * @brief  Central task — uses QUEUE SET to wait on BOTH CmdQ and KeyQ.
  * @note   This is THE key difference from Lesson 3 Basic.
  *
  *          xQueueSelectFromSet() blocks until ANY member queue
  *          has data, then returns the handle of that queue.
  *
  *          After xQueueSelectFromSet returns, you MUST still
  *          call xQueueReceive() on the returned queue to
  *          actually retrieve the data — the select only
  *          tells you WHICH queue is ready.
  *
  *          Think: select() in Linux — tells you which fd is
  *          readable, you still need to call read().
  */
static void Central_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * Block here waiting for ANY queue in the set to have data.
         * portMAX_DELAY = sleep until something arrives.
         * Returns: the QueueHandle_t of the ready queue.
         *
         * If both queues have data simultaneously, the set returns
         * the one added first (FIFO order within the set).
         */
        QueueHandle_t xReadyQueue = xQueueSelectFromSet(g.xQueueSet,
                                                        portMAX_DELAY);

        /*
         * Now check WHICH queue woke us up, and receive from it.
         * MUST use timeout=0 here — the data is guaranteed to
         * be there (xQueueSelectFromSet already confirmed it).
         */
        if (xReadyQueue == g.xCmdQueue)
        {
            LedCmd_t cmd;
            if (xQueueReceive(g.xCmdQueue, &cmd, 0) == pdPASS)
            {
                /*
                 * Execute toggle sequence: toggle LED1 cmd.count times.
                 * While we toggle here, KeyQ events can accumulate
                 * (they'll be processed in the next loop iteration).
                 */
                for (uint8_t i = 0; i < cmd.count; i++)
                {
                    APP_LedToggle(1);              /* LED1 = CmdQ indicator */
                    vTaskDelay(pdMS_TO_TICKS(cmd.delay_ms));
                }
            }
        }
        else if (xReadyQueue == g.xKeyQueue)
        {
            KeyEvent_t key;
            if (xQueueReceive(g.xKeyQueue, &key, 0) == pdPASS)
            {
                /*
                 * Key events produce a single quick toggle on LED1.
                 * Different key events produce different visual patterns:
                 *   press  → 1 toggle (50ms flash)
                 *   release→ 1 toggle (200ms flash)
                 *   long   → 1 toggle (500ms flash, visually distinct)
                 */
                TickType_t key_delay;
                switch (key.event)
                {
                case 0:  key_delay = pdMS_TO_TICKS(50);   break; /* press  */
                case 1:  key_delay = pdMS_TO_TICKS(200);  break; /* release*/
                case 2:  key_delay = pdMS_TO_TICKS(500);  break; /* long   */
                default: key_delay = pdMS_TO_TICKS(100);  break;
                }
                APP_LedToggle(1);
                vTaskDelay(key_delay);
            }
        }
        /* else: unknown queue — should not happen */
    }
}

/* ========================================================================== */
/*  Concept 3: Queue Registry — vQueueAddToRegistry / pcQueueGetName           */
/* ========================================================================== */

/**
  * @brief  Periodically inspects all registered queues and toggles LED2.
  * @note   The Queue Registry is a DEBUG feature. It lets you:
  *          - Name each queue for easy identification in the debugger
  *          - Enumerate all queues at runtime via the registry array
  *          - Get a queue's name via pcQueueGetName()
  *
  *          LED2 behavior:
  *          - All queues healthy (≤50% full) → toggle every 500ms (heartbeat)
  *          - Any queue >50% full               → toggle every 100ms (warning!)
  *
  *          In a real project, you'd output this to a UART console instead.
  */
static void QueueMonitor_Task(void *pvParameters)
{
    (void)pvParameters;

    /*
     * Queue registry array — must match configQUEUE_REGISTRY_SIZE in
     * FreeRTOSConfig.h. We registered 3 queues, so 3 or more is enough.
     *
     * The registry array is a built-in FreeRTOS structure:
     *   typedef struct {
     *       char *pcQueueName;
     *       QueueHandle_t xHandle;
     *   } QueueRegistryItem_t;
     *
     *   extern QueueRegistryItem_t xQueueRegistry[];
     *
     * Each entry: xQueueRegistry[i].xHandle is the queue handle,
     *             xQueueRegistry[i].pcQueueName is the registered name.
     *
     * CAUTION: xQueueRegistry is NOT part of the public API.
     * Use pcQueueGetName() for name lookup. Direct registry access
     * is shown here for educational purposes only.
     */

    for (;;)
    {
        /*
         * Inspect each registered queue's message count.
         * uxQueueMessagesWaiting returns 0 if queue is NULL.
         */
        UBaseType_t mailbox_count = uxQueueMessagesWaiting(g.xMailbox);
        UBaseType_t cmdq_count    = uxQueueMessagesWaiting(g.xCmdQueue);
        UBaseType_t keyq_count    = uxQueueMessagesWaiting(g.xKeyQueue);

        /*
         * Determine health: is any queue more than 50% full?
         * Mailbox depth=1 → warning if 1 (100%, always overwritten anyway)
         * CmdQ depth=4     → warning if >=2
         * KeyQ depth=4     → warning if >=2
         */
        uint8_t healthy = 1;
        if (cmdq_count >= (CMD_Q_DEPTH / 2))  { healthy = 0; }
        if (keyq_count >= (KEY_Q_DEPTH / 2))  { healthy = 0; }
        /* Mailbox: being full is NORMAL (it's always full after first write) */

        /*
         * LED2 visual feedback:
         *   healthy → toggle every 500ms (steady heartbeat)
         *   warning → toggle every 100ms (rapid flicker = attention!)
         */
        APP_LedToggle(2);
        vTaskDelay(pdMS_TO_TICKS(healthy ? 500 : 100));

        /*
         * In production, you'd log to UART instead:
         *
         *   printf("[QMon] Mailbox:%lu CmdQ:%lu/%lu KeyQ:%lu/%lu %s\r\n",
         *          mailbox_count, cmdq_count, (uint32_t)CMD_Q_DEPTH,
         *          keyq_count, (uint32_t)KEY_Q_DEPTH,
         *          healthy ? "OK" : "WARN");
         */
    }
}

/* ========================================================================== */
/*  Hardware Helpers                                                          */
/* ========================================================================== */

/**
  * @brief  Toggle LED by ID.
  * @param  led_id: 0=PB13, 1=PA1, 2=PB12
  */
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

    /* PB13 — LED0: Mailbox display (blink speed = sensor value) */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* PA1 — LED1: Queue Set demo (CmdQ + KeyQ events) */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* PB12 — LED2: Queue Registry health monitor */
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

## FreeRTOSConfig.h 改动

相比第3课基础版，进阶版需要 **两处** 配置变更：

### 1. 启用队列集

```c
// FreeRTOSConfig.h, line ~664
// 修改前:
#define configUSE_QUEUE_SETS               0

// 修改后:
#define configUSE_QUEUE_SETS               1   /* Enable xQueueCreateSet/SelectFromSet */
```

### 2. 启用队列注册表

```c
// FreeRTOSConfig.h, line ~158
// 修改前:
#define configQUEUE_REGISTRY_SIZE          0

// 修改后:
#define configQUEUE_REGISTRY_SIZE          6   /* Enable vQueueAddToRegistry (we register 3 queues) */
```

---

## 内存预算

```
configTOTAL_HEAP_SIZE = 4096 bytes (不变)

Item                │ Size    │ Qty │ Subtotal
────────────────────┼─────────┼─────┼─────────
Idle Task           │   760B  │  1  │   760B
Timer Service Task  │   760B  │  1  │   760B
SensorDisplay Task  │   920B  │  1  │   920B   (200W stack + TCB)
Central Task        │  1144B  │  1  │  1144B   (256W stack + TCB)
SensorGen Task      │   720B  │  1  │   720B   (150W stack + TCB)
CmdSender Task      │   720B  │  1  │   720B   (150W stack + TCB)
KeySim Task         │   720B  │  1  │   720B   (150W stack + TCB)
QueueMonitor Task   │  1144B  │  1  │  1144B   (256W stack + TCB)
Mailbox (1x4B)      │    68B  │  1  │    68B   (buffer + QCB)
CmdQ (4x4B)         │    80B  │  1  │    80B   (buffer + QCB)
KeyQ (4x4B)         │    80B  │  1  │    80B   (buffer + QCB)
Queue Set            │   100B  │  1  │   100B   (set control block)
────────────────────┼─────────┼─────┼─────────
TOTAL               │         │     │  7216B ← ⚠️ 超出 4096!
```

**需要增大堆：**

```c
// FreeRTOSConfig.h
#define configTOTAL_HEAP_SIZE    ((size_t)8192)   /* 修改前: 4096 */
```

---

## 验证清单

```
□ 1.  修改 FreeRTOSConfig.h 三处 (USE_QUEUE_SETS=1, REGISTRY_SIZE=6, HEAP=8192)
□ 2.  替换 main.c 为本课代码, 编译 0 Error 0 Warning
□ 3.  下载运行后观察:
□ 3a.   LED0 (PB13) — 闪烁速度周期性变化 (快→慢→快→慢...)
         → 证明: Mailbox 成功传递最新传感器值
□ 3b.   LED1 (PA1)  — 不规则闪烁, 有时快速burst, 有时单闪
         → 证明: Queue Set 成功同时监听 CmdQ + KeyQ
□ 3c.   LED2 (PB12) — 稳定心跳 (500ms周期)
         → 证明: QueueMonitor 正常运行, 队列健康
□ 4.   在 Keil Watch 窗口查看 g 结构体, 确认所有 handle 非 NULL
□ 5.   在 xQueueSelectFromSet 处设断点, 观察 xReadyQueue 变化
□ 6.   在 QueueMonitor 的 healthy 变量处设断点, 观察队列状态
```

---

## 三个概念的实战原则

| 概念 | 使用时机 | 不使用时机 |
|------|---------|-----------|
| **邮箱** (xQueueOverwrite) | 传感器数据、状态标志、配置参数 — 只关心"现在值" | 命令队列、数据流 — 每条数据都必须处理 |
| **队列集** (QueueSet) | 一个任务需要同时等 N 个数据源 | 只有一个数据源, 或各数据源已有独立消费者 |
| **注册表** (QueueRegistry) | 调试阶段、复杂系统(>5队列)、需要运行时诊断 | 简单系统(1~2队列)、生产固件(可关闭省RAM) |

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|----------|
| xQueueSelectFromSet 后忘记 xQueueReceive | Select 只返回句柄, 不取数据 | Select 后立即 Receive(timeout=0) |
| 队列集深度算错 | 必须 = 所有成员队列深度之和 | 创建前计算: `sum of all queue depths` |
| 删除队列前未移除出队列集 | 队列集持有悬空指针 | 先 `xQueueRemoveFromSet`, 再 `vQueueDelete` |
| pcQueueGetName 传未注册队列 | 返回空字符串, 不报错 | 注册表是 opt-in, 不注册就没名字 |
| 邮箱用 xQueueSend 而非 Overwrite | 发送者会在满邮箱上阻塞 | 邮箱专用 `xQueueOverwrite()` |
| 队列集里放信号量但 Receive 信号量 | 信号量没有数据, Receive 会失败 | 信号量用 `xSemaphoreTake()`, 队列用 `xQueueReceive()` |

> 📎 **基础版:** 见 `Lesson3_Queue_Basic.md` — Producer-Consumer 队列通信（入门）
