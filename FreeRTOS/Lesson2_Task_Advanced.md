# 第2课 代码实战 — 进阶版：栈监控 + 结构体全局变量

> **目标：** 在基础版三任务闪烁之上，添加栈使用率实时监控和结构化全局变量管理
> **前置：** 已理解基础版三任务调度原理
> **对应理论：** `FreeRTOS_Guidance.md` 第2课 §2.5~§2.6

---

## 代码流程图

```
main()
  │
  ├── HAL_Init(), Clock, GPIO 初始化
  │
  ├── xTaskCreate(FastLED,   pri=4, stack=80W)
  ├── xTaskCreate(MediumLED, pri=3, stack=80W)
  ├── xTaskCreate(SlowLED,   pri=2, stack=80W)
  ├── xTaskCreate(StackMonitor, pri=1, stack=200W)  ← 新增: 最低应用优先级
  │
  ├── 初始化 g_Monitor.task[x] 数组                 ← 结构体全局变量
  │
  └── vTaskStartScheduler()

调度后并行世界:

  FastLED (pri4)   MediumLED (pri3)  SlowLED (pri2)   StackMonitor (pri1)
  ┌──────────┐     ┌──────────┐      ┌──────────┐     ┌────────────────┐
  │ PB13     │     │ PA1      │      │ PC13     │     │ 每5秒:         │
  │ 100ms    │     │ 300ms    │      │ 500ms    │     │ 读3个任务的    │
  │ Toggle   │     │ Toggle   │      │ Toggle   │     │ StackHighWater │
  └──────────┘     └──────────┘      └──────────┘     │ 更新peakUsed   │
                                                      │ 读FreeHeapSize │
                                                      └────────────────┘

栈监控数据流:

  Monitor 任务
    │
    ├── uxTaskGetStackHighWaterMark(FastLED.handle)
    │   → peakUsed = allocated - remaining
    │   → 写入 g_Monitor.task[0].peakUsedWords
    │
    ├── (同上) MediumLED → g_Monitor.task[1]
    ├── (同上) SlowLED   → g_Monitor.task[2]
    │
    └── xPortGetFreeHeapSize() → g_Monitor.heapFreeBytes
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 2 Advanced: Multi-Task + Stack Monitor
  *
  *          Compared to Basic version, this adds:
  *          - Stack usage monitoring via uxTaskGetStackHighWaterMark()
  *          - Heap usage monitoring via xPortGetFreeHeapSize()
  *          - Global variables grouped in a struct (per project convention)
  *          - Task info passed via struct array (table-driven pattern)
  *
  *          Stack sizing rationale:
  *          - GPIO toggle only:  actual ~50W used  → allocate 80W (60% margin)
  *          - Stack monitor:     actual ~150W used → allocate 200W (33% margin)
  *
  *          Heap budget (configTOTAL_HEAP_SIZE = 4096 bytes):
  *          - Idle task:         512B + 120B TCB =  632B
  *          - Timer service:     512B + 120B TCB =  632B (if configUSE_TIMERS=1)
  *          - FastLED:           320B + 120B TCB =  440B
  *          - MediumLED:         320B + 120B TCB =  440B
  *          - SlowLED:           320B + 120B TCB =  440B
  *          - Monitor:           800B + 120B TCB =  920B
  *          ─────────────────────────────────────
  *          Total:              ~3504B / 4096B (592B free)
  ******************************************************************************
  */

#include "main.h"
#include "FreeRTOS.h"
#include "task.h"

/*===========================================================================
 * Defines
 *===========================================================================*/

/* LED blink periods (ms) */
#define FAST_PERIOD_MS      100U
#define MEDIUM_PERIOD_MS    300U
#define SLOW_PERIOD_MS      500U

/* Monitor report interval (ms) */
#define MONITOR_PERIOD_MS   5000U

/* Task stack sizes (words) — tuned per task complexity */
#define STACK_GPIO_TOGGLE   80U     /* LED tasks: ~50W used, 80W safe */
#define STACK_MONITOR       200U    /* Monitor: formatting + API calls */

/* Task priorities (configMAX_PRIORITIES = 5 → 0..4) */
#define PRIO_MONITOR        1U      /* Lowest app priority — won't steal CPU */
#define PRIO_SLOW           2U
#define PRIO_MEDIUM         3U
#define PRIO_FAST           4U      /* Highest app priority */

/* Number of tracked tasks */
#define MONITOR_TASK_COUNT  3U

/*===========================================================================
 * Types — Struct-based global variable pattern
 *===========================================================================*/

/**
  * @brief  Per-task stack usage descriptor.
  *         Each tracked task has one entry in g_Monitor.task[].
  */
typedef struct {
    TaskHandle_t  handle;           /* Task handle for API calls */
    const char   *name;             /* Debug name (max 16 chars) */
    UBaseType_t   allocatedWords;   /* Stack size passed to xTaskCreate */
    UBaseType_t   peakUsedWords;    /* Max words ever used (filled by Monitor) */
} TaskStackInfo_t;

/**
  * @brief  Monitor globals — all global state in one struct, not scattered.
  *
  *         Rationale:
  *         - Centralized visibility: grep "g_Monitor" finds all global state
  *         - Clean initialization: g_Monitor = {0} resets everything
  *         - Thread-safe grouping: related data lives together
  */
typedef struct {
    TaskStackInfo_t task[MONITOR_TASK_COUNT];  /* Tracked task array */
    size_t          heapFreeBytes;             /* FreeRTOS heap remaining */
} MonitorGlobals_t;

/*===========================================================================
 * Global Variables
 *===========================================================================*/

/* Single struct instance — all monitor globals live here */
static MonitorGlobals_t g_Monitor = {0};

/*===========================================================================
 * Private Function Prototypes
 *===========================================================================*/

static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);

/* Task functions */
static void FastLED_Task(void *pvParameters);
static void MediumLED_Task(void *pvParameters);
static void SlowLED_Task(void *pvParameters);
static void StackMonitor_Task(void *pvParameters);

/*===========================================================================
 * main()
 *===========================================================================*/

int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /* Create LED tasks.
     * Creation order is irrelevant — scheduler picks by priority. */
    xReturned = xTaskCreate(FastLED_Task,   "Fast",
                            STACK_GPIO_TOGGLE, NULL, PRIO_FAST, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(MediumLED_Task, "Medium",
                            STACK_GPIO_TOGGLE, NULL, PRIO_MEDIUM, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(SlowLED_Task,   "Slow",
                            STACK_GPIO_TOGGLE, NULL, PRIO_SLOW, NULL);
    configASSERT(xReturned == pdPASS);

    /* Create monitor task at lowest app priority */
    xTaskCreate(StackMonitor_Task, "Monitor",
                STACK_MONITOR, NULL, PRIO_MONITOR, NULL);

    /* Populate task tracking array.
     * Must happen AFTER xTaskCreate but BEFORE scheduler starts,
     * because xTaskGetHandle looks up by name in the kernel's task list. */
    g_Monitor.task[0].handle         = xTaskGetHandle("Fast");
    g_Monitor.task[0].name           = "Fast";
    g_Monitor.task[0].allocatedWords = STACK_GPIO_TOGGLE;

    g_Monitor.task[1].handle         = xTaskGetHandle("Medium");
    g_Monitor.task[1].name           = "Medium";
    g_Monitor.task[1].allocatedWords = STACK_GPIO_TOGGLE;

    g_Monitor.task[2].handle         = xTaskGetHandle("Slow");
    g_Monitor.task[2].name           = "Slow";
    g_Monitor.task[2].allocatedWords = STACK_GPIO_TOGGLE;

    vTaskStartScheduler();  /* Never returns */
    for (;;) {}
}

/*===========================================================================
 * LED Tasks
 *===========================================================================*/

static void FastLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_13);
        vTaskDelay(pdMS_TO_TICKS(FAST_PERIOD_MS));
    }
}

static void MediumLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_1);
        vTaskDelay(pdMS_TO_TICKS(MEDIUM_PERIOD_MS));
    }
}

static void SlowLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        vTaskDelay(pdMS_TO_TICKS(SLOW_PERIOD_MS));
    }
}

/*===========================================================================
 * Stack Monitor Task (NEW — not in Basic version)
 *===========================================================================*/

/**
  * @brief  Periodic stack/heap monitor — runs at lowest app priority.
  *
  *         Reports per-task peak stack usage and total heap free.
  *         In production: send via UART. For now: inspect with debugger.
  *
  *         Stack margin interpretation:
  *           > 50% : wasteful — reduce allocation
  *           20-50%: healthy
  *           < 20% : DANGER — increase stack or risk overflow
  */
static void StackMonitor_Task(void *pvParameters)
{
    (void)pvParameters;
    UBaseType_t uxHighWaterMark;

    for (;;)
    {
        /* Sample all tracked tasks */
        for (uint32_t i = 0U; i < MONITOR_TASK_COUNT; i++)
        {
            if (g_Monitor.task[i].handle != NULL)
            {
                /* Returns remaining words at point of deepest stack usage.
                 * FreeRTOS fills stack with 0xA5 pattern on creation;
                 * this function scans from stack bottom to find first
                 * non-0xA5 word. */
                uxHighWaterMark = uxTaskGetStackHighWaterMark(
                                      g_Monitor.task[i].handle);

                /* Store peak usage for debugger inspection.
                 * peakUsed = (allocated - remaining) = actual max words consumed */
                g_Monitor.task[i].peakUsedWords =
                    g_Monitor.task[i].allocatedWords - uxHighWaterMark;
            }
        }

        /* Sample heap free space */
        g_Monitor.heapFreeBytes = xPortGetFreeHeapSize();

        /* Inspect g_Monitor in debugger Watch window:
         *
         *   g_Monitor.task[0].peakUsedWords  → FastLED peak stack use
         *   g_Monitor.task[1].peakUsedWords  → MediumLED peak stack use
         *   g_Monitor.task[2].peakUsedWords  → SlowLED peak stack use
         *   g_Monitor.heapFreeBytes          → remaining heap
         *
         * Expected (after warm-up):
         *   Fast/Midium/Slow: ~50W used (out of 80W, 37% margin)
         *   Monitor:          ~150W used (out of 200W, 25% margin)
         *   heapFreeBytes:    ~592 bytes
         */

        vTaskDelay(pdMS_TO_TICKS(MONITOR_PERIOD_MS));
    }
}

/*===========================================================================
 * GPIO Configuration
 *===========================================================================*/

static void APP_GpioConfig(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();

    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;

    GPIO_InitStruct.Pin = GPIO_PIN_13;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_SET);

    GPIO_InitStruct.Pin = GPIO_PIN_1;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);

    GPIO_InitStruct.Pin = GPIO_PIN_13;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
}

/*===========================================================================
 * System Clock: HSI 8MHz
 *===========================================================================*/

static void APP_SystemClockConfig(void)
{
    RCC_OscInitTypeDef OscInitstruct = {0};
    RCC_ClkInitTypeDef ClkInitstruct = {0};

    OscInitstruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
    OscInitstruct.HSIState       = RCC_HSI_ON;
    OscInitstruct.HSEState       = RCC_HSE_OFF;
    OscInitstruct.PLL.PLLState   = RCC_PLL_OFF;
    if (HAL_RCC_OscConfig(&OscInitstruct) != HAL_OK)
        APP_ErrorHandler();

    ClkInitstruct.ClockType      = RCC_CLOCKTYPE_SYSCLK | RCC_CLOCKTYPE_HCLK
                                 | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    ClkInitstruct.SYSCLKSource   = RCC_SYSCLKSOURCE_HSI;
    ClkInitstruct.AHBCLKDivider  = RCC_SYSCLK_DIV1;
    ClkInitstruct.APB1CLKDivider = RCC_HCLK_DIV1;
    ClkInitstruct.APB2CLKDivider = RCC_HCLK_DIV2;
    if (HAL_RCC_ClockConfig(&ClkInitstruct, FLASH_LATENCY_0) != HAL_OK)
        APP_ErrorHandler();
}

/*===========================================================================
 * Hook Functions
 *===========================================================================*/

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    /* Stack overflow detected — set breakpoint here.
     * Inspect xTask (which task) and pcTaskName (name). */
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

## FreeRTOSConfig.h 额外配置

相比第1课，第2课需要启用以下 API（如果尚未启用）：

```c
/* Verify these are set to 1 in Inc/FreeRTOSConfig.h */

#define INCLUDE_xTaskGetHandle                  1  /* xTaskGetHandle("name")  */
#define INCLUDE_uxTaskGetStackHighWaterMark     1  /* stack usage monitoring */
#define INCLUDE_xTaskGetIdleTaskHandle          1  /* optional: idle task access */
```

---

## 调试方法

### Keil Debugger Watch 窗口

```
在 StackMonitor_Task 的 for(;;) 循环内设断点, 添加 Watch:

  g_Monitor.task[0].peakUsedWords
  g_Monitor.task[1].peakUsedWords
  g_Monitor.task[2].peakUsedWords
  g_Monitor.heapFreeBytes

每次进入断点时更新, 观察 peakUsed 是否稳定。
```

### 栈大小调整决策表

| peakUsedWords | allocatedWords | 利用率 | 决策 |
|---------------|----------------|--------|------|
| ~50W | 80W | 62% | 可接受, 保持 80W |
| ~65W | 80W | 81% | ⚠️ 危险, 增加到 100W |
| ~30W | 80W | 37% | 浪费, 减少到 64W |

---

## 基础版 vs 进阶版对比

| 特性 | 基础版 | 进阶版 |
|------|--------|--------|
| 多任务 | ✅ | ✅ |
| 优先级抢占 | ✅ | ✅ |
| 栈使用率监控 | ❌ | ✅ `uxTaskGetStackHighWaterMark()` |
| 堆使用率监控 | ❌ | ✅ `xPortGetFreeHeapSize()` |
| 结构体全局变量 | ❌ | ✅ `g_Monitor` struct pattern |
| 栈大小优化 | ❌ (固定 128W) | ✅ (按需 80W) |
| 适用场景 | 快速原型验证 | 生产代码开发 |

> 📎 **基础版:** 见 `Lesson2_Task_Basic.md`
> 📎 **理论篇:** 见 `FreeRTOS_Guidance.md` 第2课章节
