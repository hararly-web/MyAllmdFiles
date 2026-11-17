# 第3课 代码实战 — 基础版：Producer-Consumer 队列通信

> **目标：** 掌握 xQueueCreate / xQueueSend / xQueueReceive 的基本用法，理解队列阻塞与超时机制
> **硬件：** PY32F403-STK (PB13=LED3, PA1=外接LED, PC13=外接LED)
> **前置：** 第2课多任务调度已完成，理解任务优先级和抢占

---

## 代码流程图

```
main()
  │
  ├── HAL_Init()                    ← SysTick + 外设复位
  ├── APP_SystemClockConfig()       ← HSI 8MHz
  ├── APP_GpioConfig()              ← 初始化 PB13, PA1, PC13
  │
  ├── xQueueCreate(8, sizeof(LedCmd_t))  ← 创建队列, 深度=8, item=4字节
  │
  ├── xTaskCreate(Handler, pri=3)   ← 消费者: 从队列取命令, 控制LED
  ├── xTaskCreate(Sender,  pri=2)   ← 生产者: 生成命令, 发送到队列
  │
  └── vTaskStartScheduler()         ← 🚀 永不返回

调度后数据流:

  Sender (prio=2)           队列 (深度=8)            Handler (prio=3)
  ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
  │ 循环生成命令  │─xQueueSend→│  [0][1][2]   │─xQueueReceive→│ 接收命令     │
  │              │          │  [3][4][5]   │              │ 解析led_id  │
  │ led_id: 0~2  │          │  [6][7]      │              │ 循环翻转LED │
  │ count: 1~5   │          └──────────────┘              │ vTaskDelay  │
  │ delay: 80~500│                                        │ 100~500ms   │
  │ 每500ms发1条 │                                        │ 队列空→永久等│
  └──────────────┘                                        └──────────────┘

运行时序:

  时间 →  0     500   1000   1500   2000   2500   3000
          │      │      │      │      │      │      │
  Handler │█████│░░░░░░│██████│░░░░░░░│██████│░░░░░░│...
  (prio3) │处理0 │等待   │处理1  │等待    │处理2  │等待   │
          │      │      │      │      │      │      │
  Sender  │生成0 ││生成1 ││生成2 ││生成3 ││生成4 ││生成5 │
  (prio2) │发送  ││发送  ││发送  ││发送  ││发送  ││发送  │

预期: PB13 快闪3次 → PA1 慢闪2次 → PC13 闪1次 → PB13 快闪5次 → ...循环
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 3 Basic: Producer-Consumer via Queue
  *
  *          Demonstrates:
  *          - xQueueCreate() — dynamic queue creation
  *          - xQueueSend()   — send with timeout (producer)
  *          - xQueueReceive()— receive with infinite block (consumer)
  *          - Priority preemption: Handler(pri3) preempts Sender(pri2)
  *          - Queue blocking: full queue blocks sender, empty queue blocks receiver
  *
  *          Architecture:
  *            CmdSender_Task  ──xQueueSend──▶  Queue(8)  ──xQueueReceive──▶  CmdHandler_Task
  *            (Producer, pri2)                                        (Consumer, pri3)
  *
  *          Queue data (LedCmd_t):
  *            led_id  → target LED: 0=PB13, 1=PA1, 2=PC13
  *            count   → number of toggles: 1~5
  *            delay_ms→ toggle interval: 50~500ms
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

/* Private typedef -----------------------------------------------------------*/

/**
  * @brief  LED control command passed through the queue.
  * @note   Data is copied by value (memcpy) — each task has its own copy.
  *         sizeof(LedCmd_t) = 4 bytes (compact struct layout).
  */
typedef struct {
    uint8_t  led_id;    /* Target LED: 0=PB13, 1=PA1, 2=PC13 */
    uint8_t  count;     /* Number of toggles (1~5)              */
    uint16_t delay_ms;  /* Toggle interval in ms (50~500)       */
} LedCmd_t;

/* Private defines ------------------------------------------------------------*/

/* Queue configuration */
#define CMD_QUEUE_LENGTH    8U          /* Max pending commands before sender blocks */
#define CMD_QUEUE_ITEM_SIZE sizeof(LedCmd_t)

/* Task priorities — Handler higher so commands are consumed promptly */
#define PRIO_CMD_HANDLER    3U
#define PRIO_CMD_SENDER     2U

/* Task stack sizes (words) — generous for prototype */
#define STACK_CMD_HANDLER   256U        /* 1KB — LED toggling with nested loops */
#define STACK_CMD_SENDER    200U        /* 800B — command generation + queue send */

/* Timing */
#define SENDER_PERIOD_MS    500U        /* Send one command every 500ms          */
#define SEND_TIMEOUT_MS     100U        /* Max wait when queue is full           */

/* LED pin definitions (match PY32F403-STK board) */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — encapsulated in a single struct.
  * @note   Per project convention: no scattered global variables.
  */
static struct {
    QueueHandle_t xCmdQueue;    /* Handle to the command queue */
} g;                            /* g = globals */

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_GpioToggle(LedCmd_t *cmd);
static void CmdSender_Task(void *pvParameters);
static void CmdHandler_Task(void *pvParameters);

/**
  * @brief  Main program — creates queue + 2 tasks, then starts the scheduler.
  */
int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /*
     * Create the command queue.
     * - Length: 8 items (allows buffering if Handler is momentarily slow)
     * - Item size: sizeof(LedCmd_t) = 4 bytes
     * - Returns: handle used by xQueueSend/xQueueReceive, or NULL on heap exhaustion
     */
    g.xCmdQueue = xQueueCreate(CMD_QUEUE_LENGTH, CMD_QUEUE_ITEM_SIZE);
    configASSERT(g.xCmdQueue != NULL);

    /*
     * Create tasks.
     * Handler (prio 3) > Sender (prio 2):
     *   - Sender posts data → Handler immediately preempts Sender
     *   - Handler blocks on empty queue → Sender gets CPU time to fill it
     */
    xReturned = xTaskCreate(CmdHandler_Task, "Handler", STACK_CMD_HANDLER, NULL,
                            PRIO_CMD_HANDLER, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(CmdSender_Task, "Sender", STACK_CMD_SENDER, NULL,
                            PRIO_CMD_SENDER, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();  /* Never returns */

    while (1) {}            /* Should never reach here */
}

/* --- Producer Task (CmdSender) --------------------------------------------- */

/**
  * @brief  Producer task: generates a sequence of LED commands, sends via queue.
  * @note   xQueueSend with timeout:
  *          - Queue has space → data copied immediately, returns pdPASS
  *          - Queue full → task blocks up to SEND_TIMEOUT_MS
  *          - Timeout expired → returns errQUEUE_FULL, command is dropped
  */
static void CmdSender_Task(void *pvParameters)
{
    (void)pvParameters;

    /* Predefined command sequence — cycles through different LEDs
     * with varying toggle counts and speeds for visual variety. */
    static const LedCmd_t cmd_sequence[] = {
        {0, 3, 100},   /* PB13: toggle 3 times, 100ms apart (fast burst)   */
        {1, 2, 200},   /* PA1:  toggle 2 times, 200ms apart (medium)       */
        {2, 1, 300},   /* PC13: toggle 1 time,  300ms apart (single pulse) */
        {0, 5, 80},    /* PB13: toggle 5 times, 80ms apart  (fast train)   */
        {1, 4, 150},   /* PA1:  toggle 4 times, 150ms apart                */
        {2, 2, 250},   /* PC13: toggle 2 times, 250ms apart                */
        {0, 1, 500},   /* PB13: toggle 1 time,  500ms apart (long blink)   */
        {1, 3, 120},   /* PA1:  toggle 3 times, 120ms apart                */
    };
    static const uint8_t cmd_count = sizeof(cmd_sequence) / sizeof(cmd_sequence[0]);
    uint8_t idx = 0;

    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        LedCmd_t cmd = cmd_sequence[idx];
        idx = (idx + 1) % cmd_count;   /* Cycle through the sequence */

        /*
         * Send with timeout.
         * In normal operation (Handler processes faster than Sender sends),
         * the queue is empty → xQueueSend succeeds immediately.
         * If Handler were too slow, the queue would fill up,
         * and Sender would block here for up to SEND_TIMEOUT_MS.
         */
        BaseType_t result = xQueueSend(g.xCmdQueue, &cmd,
                                       pdMS_TO_TICKS(SEND_TIMEOUT_MS));
        if (result != pdPASS)
        {
            /*
             * Queue full + timeout expired → command dropped.
             * In production: increment a dropped counter for diagnostics.
             * Here: silently skip (acceptable for a visual LED demo).
             */
            (void)result;
        }

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(SENDER_PERIOD_MS));
    }
}

/* --- Consumer Task (CmdHandler) -------------------------------------------- */

/**
  * @brief  Consumer task: receives commands from queue and toggles LEDs.
  * @note   xQueueReceive with portMAX_DELAY:
  *          - Queue has data → copies into &cmd, returns pdPASS
  *          - Queue empty → task suspends (blocked state), zero CPU cost
  *          - Data arrives → task wakes, receives, processes
  *          portMAX_DELAY = infinite timeout — the only reason to return
  *          errQUEUE_EMPTY is if the queue is deleted (not in this demo).
  */
static void CmdHandler_Task(void *pvParameters)
{
    (void)pvParameters;
    LedCmd_t cmd;

    for (;;)
    {
        /*
         * Block indefinitely until a command arrives.
         * While blocked, this task consumes ZERO CPU cycles —
         * the scheduler simply skips it.
         */
        if (xQueueReceive(g.xCmdQueue, &cmd, portMAX_DELAY) == pdPASS)
        {
            /*
             * Execute toggle sequence for this command:
             *   for i in 0..cmd.count:
             *       toggle target LED, delay cmd.delay_ms
             */
            for (uint8_t i = 0; i < cmd.count; i++)
            {
                APP_GpioToggle(&cmd);
                vTaskDelay(pdMS_TO_TICKS(cmd.delay_ms));
            }
        }
        /* Loop back — wait for next command */
    }
}

/* --- Hardware Helper ------------------------------------------------------- */

/**
  * @brief  Toggle the LED specified in cmd->led_id.
  * @param  cmd: pointer to LedCmd_t (non-NULL).
  */
static void APP_GpioToggle(LedCmd_t *cmd)
{
    switch (cmd->led_id)
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
        break;  /* Invalid led_id — ignore */
    }
}

/* --- GPIO Configuration ---------------------------------------------------- */

/**
  * @brief  Initialize 3 GPIO pins as push-pull outputs with pull-up.
  */
static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    /* PB13 — on-board LED3 */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);  /* start HIGH = LED off */

    /* PA1 — external LED (optional) */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* PC13 — external LED (optional) */
    GPIO_InitStruct.Pin = LED2_PIN;
    HAL_GPIO_Init(LED2_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED2_PORT, LED2_PIN, GPIO_PIN_SET);
}

/* --- System Clock: HSI 8MHz, No PLL ---------------------------------------- */

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

/* --- Hook Functions -------------------------------------------------------- */

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

相比第2课，第3课不需要额外的 FreeRTOSConfig.h 改动。队列功能 (`queue.c`) 是内核基础模块，始终编译。

**确认以下宏已启用** (当前项目已配好):

```c
#define configSUPPORT_DYNAMIC_ALLOCATION   1   /* xQueueCreate needs heap     */
#define configUSE_QUEUE_SETS               0   /* Queue Sets — see Advanced   */
#define configQUEUE_REGISTRY_SIZE          0   /* Queue Registry — for debug  */
```

---

## 验证清单

```
□ 1. 编译 0 Error, 0 Warning (需要 #include "queue.h")
□ 2. 下载运行后, LED 按命令序列闪烁:
     PB13 闪3次(100ms间隔) → PA1 闪2次(200ms) → PC13 闪1次(300ms) →
     PB13 闪5次(80ms) → PA1 闪4次(150ms) → PC13 闪2次(250ms) →
     PB13 闪1次(500ms) → PA1 闪3次(120ms) → ...循环
□ 3. 用示波器/逻辑分析仪测 PB13: 可以看到不同阶段的频率变化
□ 4. 在 xQueueReceive 调用处设断点, 观察 cmd 值逐步变化
```

---

## 观察要点

| 现象 | 说明 |
|------|------|
| Handler 空闲时 LED 全灭 | Handler 阻塞在 xQueueReceive(), 不消耗 CPU |
| 同一 LED 忽快忽慢 | 因为每次 command 的 count/delay 不同 |
| Sender 发完全部8条后回到第1条 | 循环复用, 8条命令不断重复 |
| 队列里实际只有 0~1 条数据 | Handler(pri3) 立即抢占 Sender(pri2) 并消费数据 |

---

## 调试技巧

### 1. 断点观察队列数据

```
在 CmdHandler_Task 的 xQueueReceive() 行设断点:
  → 每次触发时查看变量 cmd 的值
  → cmd.led_id 变化: 0 → 1 → 2 → 0 → 1 → 2 → 0 → 1 → ...
  → cmd.count 变化: 3 → 2 → 1 → 5 → 4 → 2 → 1 → 3 → ...
```

### 2. 检查队列状态

```c
/* 在 Watch 窗口或代码中调用: */
uxQueueMessagesWaiting(g.xCmdQueue);    /* 当前队列中有几条数据  */
uxQueueSpacesAvailable(g.xCmdQueue);    /* 队列还有多少空位      */
```

### 3. 模拟队列满的情况

```c
/* 临时修改 SENDER_PERIOD_MS 为 10ms, SEND_TIMEOUT_MS 为 0:
 * Sender 快速连发 → 队列填满 → Handler 来不及处理 →
 * xQueueSend 返回 errQUEUE_FULL (因为 timeout=0) */
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|----------|
| itemSize 写错 | `sizeof(MyStruct)` vs `sizeof(MyStruct*)` | 用 `sizeof(LedCmd_t)`, 永远不要用指针的 sizeof |
| 队列数据是指针 | 发送 `&local_var` → 接收时 local_var 可能已失效 | 队列传递值(拷贝), 不要传指针 |
| portMAX_DELAY 在 ISR 中 | ISR 中不能阻塞等待 | 用 `xQueueSendFromISR()` + 0 timeout |
| 忘记 `#include "queue.h"` | 编译报 implicit declaration 警告 | 每个用到队列 API 的文件都 include |

> 📎 **进阶版:** 见 `Lesson3_Queue_Advanced.md` — 添加队列状态监控和 Queue Set 多队列监听
