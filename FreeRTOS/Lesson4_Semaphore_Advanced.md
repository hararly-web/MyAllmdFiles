# 第4课 代码实战 — 进阶版：信号量实现生产者-消费者 (Bounded Buffer)

> **目标：** 用 2个计数信号量 + 1个互斥锁 实现经典 Bounded Buffer（有界缓冲区）模式
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第4课基础版信号量已完成，理解计数信号量的 Give/Take 语义
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（第3课进阶版已配好）

---

## 核心概念：信号量 ≠ 队列

```
┌──────────────────────────────────────────────────────────────────┐
│  第3课队列版 Producer-Consumer        第4课信号量版 Bounded Buffer │
│  ─────────────────────────────        ────────────────────────── │
│                                                                  │
│  Producer → Queue → Consumer          Producer → RingBuf → Consumer│
│             ▲                                  ▲         ▲       │
│             │                                  │         │       │
│        数据在队列里                       信号量只计数   数据在Buf里│
│        (队列 = 数据通道)                 (信号量 = 敲门铃)          │
│                                                                  │
│  1个内核对象: 队列                    3个内核对象:                  │
│  - xQueueCreate                       - xSemEmpty (计数, 空槽)    │
│                                       - xSemFilled(计数, 满槽)    │
│                                       - xMutex    (锁, 保护Buf)   │
│                                                                  │
│  每次 Send/Receive 都 memcpy          数据直接写入RingBuf,          │
│                                       memcpy 只发生在 Put/Get     │
│                                                                  │
│  适用: 小数据, 每条必须处理            适用: 大数据, 高性能传输     │
│        命令/事件/消息                               数据流/帧缓冲 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 4 Advanced: Bounded Buffer (Semaphore-based)
  *
  *          Demonstrates Dijkstra's classic Producer-Consumer pattern using:
  *          - xSemaphoreCreateCounting(4, 4) → empty slot counter
  *          - xSemaphoreCreateCounting(4, 0) → filled slot counter
  *          - xSemaphoreCreateMutex()        → shared buffer guard
  *
  *          Architecture:
  *
  *            FastProd (pri2, 150ms) ──┐
  *                                      ├──▶ RingBuf[4] ──▶ Consumer (pri3)
  *            SlowProd (pri2, 400ms) ──┘        ▲
  *                                               │
  *                       xSemEmpty (0~4) ────────┘ controls producer flow
  *                       xSemFilled(0~4) ────────┘ controls consumer flow
  *                       xMutex          ────────┘ protects RingBuf access
  *
  *          LED assignment:
  *            LED0 (PB13): Consumer — toggles each time data is consumed
  *            LED1 (PA1):  Producer — toggles each time data is produced
  *            LED2 (PB12): Monitor  — slow = buffer OK, fast = buffer >50% full
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"

/* Private typedef -----------------------------------------------------------*/

/**
  * @brief  Data item stored in the shared ring buffer.
  * @note   This lives in RingBuf[], NOT inside any semaphore.
  *          Semaphores only COUNT — the data is here.
  */
typedef struct {
    uint32_t value;         /* Arbitrary payload (e.g. sensor reading) */
    uint8_t  producer_id;   /* 0=FastProd, 1=SlowProd                  */
} BufItem_t;

/* Private defines ------------------------------------------------------------*/

/* Ring buffer configuration */
#define RB_SIZE                 4U          /* Buffer capacity — small to demo blocking */
#define RB_MASK                 (RB_SIZE - 1U)  /* Must be power-of-2 for bitwise wrapping */

/* Task priorities — Consumer highest so buffer never overflows in normal case */
#define PRIO_CONSUMER           3U
#define PRIO_FAST_PROD          2U
#define PRIO_SLOW_PROD          2U
#define PRIO_BUF_MONITOR        1U          /* Lowest — background diagnostic */

/* Task stack sizes (words) */
#define STACK_CONSUMER          200U
#define STACK_FAST_PROD         150U
#define STACK_SLOW_PROD         150U
#define STACK_BUF_MONITOR       200U

/* Timing (ms) */
#define FAST_PROD_PERIOD_MS     150U        /* Fast producer generates every 150ms   */
#define SLOW_PROD_PERIOD_MS     400U        /* Slow producer generates every 400ms   */
#define MONITOR_PERIOD_MS       2000U       /* Monitor checks buffer every 2 seconds */

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
    /* Semaphores for the Bounded Buffer pattern */
    SemaphoreHandle_t xSemEmpty;        /* Counting: empty slots (init=RB_SIZE)  */
    SemaphoreHandle_t xSemFilled;       /* Counting: filled slots (init=0)       */
    SemaphoreHandle_t xMutex;           /* Mutex: protects RingBuf[RB_SIZE]      */

    /* Shared ring buffer — the actual data storage */
    BufItem_t RingBuf[RB_SIZE];
    uint8_t   wr_idx;                   /* Write index (producer-owned concept)  */
    uint8_t   rd_idx;                   /* Read index  (consumer-owned concept)  */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);

/* --- Bounded Buffer Tasks -------------------------------------------------*/
static void FastProducer_Task(void *pvParameters);
static void SlowProducer_Task(void *pvParameters);
static void Consumer_Task(void *pvParameters);
static void BufMonitor_Task(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

/**
  * @brief  Creates 2 counting semaphores + 1 mutex + 4 tasks.
  * @note   Semaphore initialization is the KEY design decision:
  *
  *          xSemEmpty  = Counting(4, 4):
  *            max_count=4, initial_count=4
  *            → "4 empty slots available at startup"
  *            → Producer can Take 4 times before blocking
  *
  *          xSemFilled = Counting(4, 0):
  *            max_count=4, initial_count=0
  *            → "0 filled slots at startup"
  *            → Consumer blocks immediately on first Take
  *
  *          Together they enforce:
  *            wr_idx NEVER passes rd_idx + 4 (buffer overflow)
  *            rd_idx NEVER passes wr_idx      (buffer underflow)
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /* --- Semaphore initialization (order matters for readability only) --- */

    /*
     * Empty-slot counter: starts at RB_SIZE (all slots free).
     * Producers Take this before writing → blocks when buffer is full.
     */
    g.xSemEmpty = xSemaphoreCreateCounting(RB_SIZE, RB_SIZE);
    configASSERT(g.xSemEmpty != NULL);

    /*
     * Filled-slot counter: starts at 0 (nothing to consume).
     * Consumer Takes this before reading → blocks when buffer is empty.
     */
    g.xSemFilled = xSemaphoreCreateCounting(RB_SIZE, 0);
    configASSERT(g.xSemFilled != NULL);

    /*
     * Mutex: protects RingBuf[] from concurrent access by 2 producers.
     * Even though both producers are prio2 (can't preempt each other on
     * a single-core system), the mutex is formally correct and documents
     * intent. In a multi-core system, it would be essential.
     */
    g.xMutex = xSemaphoreCreateMutex();
    configASSERT(g.xMutex != NULL);

    /* --- Create tasks --- */

    xReturned = xTaskCreate(FastProducer_Task, "FastProd",
                            STACK_FAST_PROD, NULL, PRIO_FAST_PROD, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(SlowProducer_Task, "SlowProd",
                            STACK_SLOW_PROD, NULL, PRIO_SLOW_PROD, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(Consumer_Task, "Consumer",
                            STACK_CONSUMER, NULL, PRIO_CONSUMER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(BufMonitor_Task, "BufMon",
                            STACK_BUF_MONITOR, NULL, PRIO_BUF_MONITOR, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();  /* Never returns */

    while (1) {}
}

/* ========================================================================== */
/*  Producer Tasks — Both follow the SAME 4-step protocol                       */
/* ========================================================================== */

/**
  * @brief  Fast producer — writes to RingBuf every 150ms.
  * @note   The 4-step protocol is the core of Bounded Buffer:
  *
  *          ① Take xSemEmpty  — claim an empty slot (block if full)
  *          ② Take xMutex      — lock the buffer
  *             write to RingBuf — THE critical section
  *          ③ Give xMutex      — unlock the buffer
  *          ④ Give xSemFilled  — signal "one more item available"
  *
  *          Step order is MANDATORY:
  *            Take xSemEmpty BEFORE xMutex → avoids deadlock
  *            Give xMutex BEFORE xSemFilled→ Consumer proceeds faster
  */
static void FastProducer_Task(void *pvParameters)
{
    (void)pvParameters;

    static uint32_t counter = 0;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * Step ①: Wait for an empty slot.
         * If RingBuf is full (4 items waiting), block here.
         * This is the "flow control" — producer can't outrun consumer.
         *
         * portMAX_DELAY = wait forever. In production, use a timeout
         * and handle buffer-full (e.g. increment dropped counter).
         */
        if (xSemaphoreTake(g.xSemEmpty, portMAX_DELAY) == pdPASS)
        {
            /*
             * Step ②: Lock the shared buffer.
             * Two producers could both have claimed an empty slot.
             * Mutex ensures only ONE writes at a time.
             */
            if (xSemaphoreTake(g.xMutex, portMAX_DELAY) == pdPASS)
            {
                /* CRITICAL SECTION — write to RingBuf */
                g.RingBuf[g.wr_idx].value       = counter++;
                g.RingBuf[g.wr_idx].producer_id = 0;  /* 0 = FastProd */
                g.wr_idx = (g.wr_idx + 1) & RB_MASK;

                /* Step ③: Unlock */
                xSemaphoreGive(g.xMutex);
            }

            /*
             * Step ④: Signal "new data available".
             * This wakes up Consumer if it was blocked on xSemFilled.
             */
            xSemaphoreGive(g.xSemFilled);
        }

        /* Visual feedback: LED1 toggles on each successful produce */
        APP_LedToggle(1);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(FAST_PROD_PERIOD_MS));
    }
}

/**
  * @brief  Slow producer — identical protocol, different timing.
  * @note   Uses the SAME 4-step protocol as FastProd.
  *          The protocol is producer-speed-independent.
  */
static void SlowProducer_Task(void *pvParameters)
{
    (void)pvParameters;

    static uint32_t counter = 1000;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        if (xSemaphoreTake(g.xSemEmpty, portMAX_DELAY) == pdPASS)
        {
            if (xSemaphoreTake(g.xMutex, portMAX_DELAY) == pdPASS)
            {
                g.RingBuf[g.wr_idx].value       = counter++;
                g.RingBuf[g.wr_idx].producer_id = 1;  /* 1 = SlowProd */
                g.wr_idx = (g.wr_idx + 1) & RB_MASK;

                xSemaphoreGive(g.xMutex);
            }

            xSemaphoreGive(g.xSemFilled);
        }

        /* LED1 toggles for slow producer too — mixed rates on LED1 */
        APP_LedToggle(1);

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SLOW_PROD_PERIOD_MS));
    }
}

/* ========================================================================== */
/*  Consumer Task — The other half of Bounded Buffer                            */
/* ========================================================================== */

/**
  * @brief  Consumer — reads from RingBuf, processes data, toggles LED0.
  * @note   The 4-step protocol, mirrored:
  *
  *          ① Take xSemFilled — claim a filled slot (block if empty)
  *          ② Take xMutex      — lock the buffer
  *             read from RingBuf — THE critical section
  *          ③ Give xMutex      — unlock the buffer
  *          ④ Give xSemEmpty   — signal "one more empty slot available"
  *
  *          Consumer prio=3 > both Producers prio=2:
  *            → Consumer gets the CPU immediately when data arrives
  *            → RingBuf rarely has more than 1 item waiting
  *            → (unless Consumer does a long vTaskDelay between reads)
  */
static void Consumer_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        BufItem_t item;

        /*
         * Step ①: Wait for a filled slot.
         * If RingBuf is empty, block here — zero CPU cost.
         * This is event-driven: woken up EXACTLY when a producer
         * calls xSemaphoreGive(xSemFilled).
         */
        if (xSemaphoreTake(g.xSemFilled, portMAX_DELAY) == pdPASS)
        {
            /*
             * Step ②: Lock buffer, read data.
             */
            if (xSemaphoreTake(g.xMutex, portMAX_DELAY) == pdPASS)
            {
                /* CRITICAL SECTION — read from RingBuf */
                item = g.RingBuf[g.rd_idx];
                g.rd_idx = (g.rd_idx + 1) & RB_MASK;

                /* Step ③: Unlock */
                xSemaphoreGive(g.xMutex);
            }

            /*
             * Step ④: Signal "one more empty slot".
             * This wakes up a blocked Producer (if any).
             */
            xSemaphoreGive(g.xSemEmpty);

            /*
             * Process the consumed item.
             * LED0 toggles with delay derived from the data value.
             * This simulates "doing work with the data" and creates
             * a visible pattern:
             *   FastProd data (value small) → short delay → fast blink
             *   SlowProd data (value large) → long delay  → slow blink
             */
            TickType_t proc_delay = pdMS_TO_TICKS(50 + (item.value % 5) * 30);
            APP_LedToggle(0);
            vTaskDelay(proc_delay);
        }
    }
}

/* ========================================================================== */
/*  Monitor Task — Diagnose buffer health                                       */
/* ========================================================================== */

/**
  * @brief  Monitors buffer fill level and provides LED2 visual feedback.
  * @note   Uses uxSemaphoreGetCount() to query semaphore state.
  *
  *          xSemFilled count = number of items in RingBuf right now
  *          LED2: normal heartbeat (500ms) = buffer low
  *                fast blink (100ms)       = buffer > 50% full (warning)
  *
  *          In production: route this to a UART/display instead.
  */
static void BufMonitor_Task(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /*
         * uxSemaphoreGetCount returns the current count value.
         * xSemFilled count = items currently in RingBuf.
         * xSemEmpty  count = empty slots remaining.
         */
        UBaseType_t filled = uxSemaphoreGetCount(g.xSemFilled);
        UBaseType_t empty  = uxSemaphoreGetCount(g.xSemEmpty);

        /*
         * Health check:
         *   filled >= RB_SIZE/2 → buffer getting full, consumer slow
         *   empty  >= RB_SIZE/2 → buffer mostly empty, consumer fast
         *
         *   NOTE: filled + empty ≠ RB_SIZE (one slot may be in use
         *   at the moment of reading — not locked atomically)
         */
        uint8_t is_warning = (filled >= (RB_SIZE / 2)) ? 1 : 0;

        APP_LedToggle(2);
        vTaskDelay(pdMS_TO_TICKS(is_warning ? 100 : 500));

        /*
         * Production-level logging (uncomment for UART):
         *
         *   printf("[BufMon] filled:%lu/%u empty:%lu/%u %s\r\n",
         *          filled, RB_SIZE, empty, RB_SIZE,
         *          is_warning ? "WARN" : "OK");
         */

        /* Prevent unused variable warnings when production logging is off */
        (void)empty;
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

    /* PB13 — LED0: Consumer activity indicator */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* PA1 — LED1: Producer activity indicator */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* PB12 — LED2: Buffer health monitor */
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

## 四步协议详解 (The 4-Step Protocol)

这是本课最核心的知识点。每一次生产/消费都必须严格按顺序执行：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Bounded Buffer 4-Step Protocol                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  PRODUCER                          CONSUMER                              │
│  ────────                          ────────                              │
│                                                                          │
│  ① xSemaphoreTake(xSemEmpty)       ① xSemaphoreTake(xSemFilled)         │
│     "申请一个空槽位"                    "申请一个满槽位"                     │
│     缓冲满 → 阻塞在此                  缓冲空 → 阻塞在此                    │
│                                                                          │
│  ② xSemaphoreTake(xMutex)          ② xSemaphoreTake(xMutex)             │
│     "锁住缓冲区"                       "锁住缓冲区"                        │
│                                                                          │
│     RingBuf[wr++] = data;            item = RingBuf[rd++];              │
│     ↑ 关键: 写数据                    ↑ 关键: 读数据                       │
│                                                                          │
│  ③ xSemaphoreGive(xMutex)          ③ xSemaphoreGive(xMutex)             │
│     "释放锁"                          "释放锁"                            │
│                                                                          │
│  ④ xSemaphoreGive(xSemFilled)      ④ xSemaphoreGive(xSemEmpty)          │
│     "通知有数据了"                     "通知腾出空位了"                     │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ⚠️ 为什么步骤顺序是强制的:                                                 │
│                                                                          │
│    ✅ 正确: ① Take Empty → ② Take Mutex                                  │
│       原因: 如果先在③等Mutex, 而Empty已经用完,                              │
│              Consumer永远拿不到Mutex释放Empty → 死锁                       │
│                                                                          │
│    ✅ 正确: ③ Give Mutex → ④ Give Filled                                 │
│       原因: 先释放锁再发信号, Consumer可以更快地                            │
│              获取Mutex, 减少等待时间                                       │
│                                                                          │
│    这个顺序是Dijkstra 1965年论文中确立的,                                   │
│    半个世纪以来所有操作系统教材都在教这个                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## FreeRTOSConfig.h 配置

第3课进阶版已配置好 `configTOTAL_HEAP_SIZE=8192`，本课不需要额外修改。

**确认以下宏已启用：**

```c
#define configUSE_COUNTING_SEMAPHORES    1   /* xSemaphoreCreateCounting 需要 */
#define configUSE_MUTEXES                1   /* xSemaphoreCreateMutex 需要   */
#define configSUPPORT_DYNAMIC_ALLOCATION 1   /* 动态创建, 第1课已配            */
```

**需要在 `main.c` 中新增 include：**

```c
#include "semphr.h"   /* xSemaphoreCreateCounting / xSemaphoreCreateMutex */
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Timer Service Task      │   760B  │  1  │   760B
Consumer Task           │   920B  │  1  │   920B   (200W stack + TCB)
FastProd Task           │   720B  │  1  │   720B   (150W stack + TCB)
SlowProd Task           │   720B  │  1  │   720B   (150W stack + TCB)
BufMonitor Task         │   920B  │  1  │   920B   (200W stack + TCB)
xSemEmpty (Counting)    │    80B  │  1  │    80B   (control block)
xSemFilled (Counting)   │    80B  │  1  │    80B   (control block)
xMutex (Mutex)          │    80B  │  1  │    80B   (control block)
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  5040B / 8192B  ✅
```

---

## 验证清单

```
□ 1. 添加 #include "semphr.h" 到 main.c
□ 2. 编译 0 Error, 0 Warning
□ 3. 下载运行后观察:
□ 3a. LED0 (PB13) — 持续闪烁, 频率随数据值变化
      → 证明: Consumer 持续从 RingBuf 消费数据
□ 3b. LED1 (PA1)  — 快速不规则闪烁 (两个Producer的叠加)
      → 证明: FastProd + SlowProd 同时写入 RingBuf
□ 3c. LED2 (PB12) — 正常时500ms心跳, RingBuf >50%满时100ms快闪
      → 证明: BufMonitor 正确读取信号量计数值
□ 4.  在 Consumer 的 xSemaphoreTake(xSemFilled) 设断点:
      逐步执行观察 item.value 和 item.producer_id 变化
□ 5.  在 FastProd 的 xSemaphoreTake(xSemEmpty) 设断点:
      如果断点触发间隔 > 150ms → Producer 因缓冲满而阻塞了
□ 6.  Keil Watch 窗口添加:
      uxSemaphoreGetCount(g.xSemEmpty)
      uxSemaphoreGetCount(g.xSemFilled)
      → 实时观察两个计数的此消彼长
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|----------|
| Take/ Give 顺序错误 | 先 Take Mutex 再 Take Empty → 死锁 | 始终先 Take 信号量, 再 Take Mutex |
| Counting 初始值写反 | xSemEmpty init=0, xSemFilled init=4 → 消费者立刻消费"不存在"的数据 | Empty init=容量, Filled init=0 |
| 忘记 Give 对应的信号量 | Prod写完忘记 Give Filled → Consumer 永久阻塞 | ④步缺一不可 |
| 信号量 Give 次数 > Take | 计数信号量超过 max → Give 返回失败且计数不变 | 成对 Take/Give, 用 Monitor 任务验证 |
| Mutex 不释放 | 异常路径忘记 Give Mutex → 死锁 | 用 do-while(0) + break, 或 RAII 模式 |
| 只用一个计数信号量 | 不能同时限制生产者和消费者的速度 | 必须两个: Empty+ Filled |
| 信号量和mutex用同一个变量 | 拿Mutex当信号量用 → 所有权混乱 | 分开创建, 语义清晰 |

> 📎 **基础版:** 见 `Lesson4_Semaphore_Basic.md` — 二值信号量 + 计数信号量 + 互斥锁（入门）
