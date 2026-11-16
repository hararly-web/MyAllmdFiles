# 第2课 代码实战 — 基础版：三任务 LED 闪烁

> **目标：** 验证多任务优先级抢占、独立延时、任务并行运行
> **硬件：** PY32F403-STK (PB13=LED3, PA1=外接LED, PC13=外接LED)
> **前置：** 第1课移植已完成，configASSERT 已修复

---

## 代码流程图

```
main()
  │
  ├── HAL_Init()                    ← SysTick + 外设复位
  ├── APP_SystemClockConfig()       ← HSI 8MHz
  ├── APP_GpioConfig()              ← 初始化 PB13, PA1, PC13
  │
  ├── xTaskCreate(FastLED,   pri=3) ← 最高优先级, 100ms 周期
  ├── xTaskCreate(MediumLED, pri=2) ← 中优先级,   300ms 周期
  ├── xTaskCreate(SlowLED,   pri=1) ← 低优先级,   500ms 周期
  │
  └── vTaskStartScheduler()         ← 🚀 永不返回, 调度器接管

调度后并行世界:

  FastLED (pri3)     MediumLED (pri2)    SlowLED (pri1)
  ┌──────────┐       ┌──────────┐        ┌──────────┐
  │ Toggle   │       │ Toggle   │        │ Toggle   │
  │ PB13     │       │ PA1      │        │ PC13     │
  │ Delay    │       │ Delay    │        │ Delay    │
  │ 100ms    │       │ 300ms    │        │ 500ms    │
  └──────────┘       └──────────┘        └──────────┘

预期: PB13 闪最快 (5Hz) > PA1 中速 > PC13 最慢 (1Hz)
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 2 Basic: 3 independent LED tasks
  *
  *          Demonstrates:
  *          - Multiple xTaskCreate() with different priorities
  *          - Preemptive priority scheduling
  *          - vTaskDelay(pdMS_TO_TICKS()) for precise timing
  *          - Independent task execution (each runs on its own timeline)
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"

/* Private defines ------------------------------------------------------------*/
#define FAST_PERIOD_MS      100U    /* 5Hz  — verify with logic analyzer */
#define MEDIUM_PERIOD_MS    300U    /* ~3Hz — verify with logic analyzer */
#define SLOW_PERIOD_MS      500U    /* 1Hz  — verify visually */

#define TASK_STACK          128U    /* words, generously sized for GPIO toggle */

/* configMAX_PRIORITIES=5 → valid priorities: 0 (lowest) ~ 4 (highest) */
#define PRIO_FAST           3U
#define PRIO_MEDIUM         2U
#define PRIO_SLOW           1U

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void FastLED_Task(void *pvParameters);
static void MediumLED_Task(void *pvParameters);
static void SlowLED_Task(void *pvParameters);

/**
  * @brief  Main program.
  * @retval int
  */
int main(void)
{
    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /* Create tasks in any order — scheduler picks highest priority first */
    xTaskCreate(FastLED_Task,   "Fast",   TASK_STACK, NULL, PRIO_FAST,   NULL);
    xTaskCreate(MediumLED_Task, "Medium", TASK_STACK, NULL, PRIO_MEDIUM, NULL);
    xTaskCreate(SlowLED_Task,   "Slow",   TASK_STACK, NULL, PRIO_SLOW,   NULL);

    vTaskStartScheduler();  /* Never returns */

    /* Should never reach here */
    while (1) {}
}

/* --- LED Tasks ------------------------------------------------------------- */

/* Fast: PB13 toggles every 100ms (5Hz) — highest priority */
static void FastLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_13);
        vTaskDelay(pdMS_TO_TICKS(FAST_PERIOD_MS));
    }
}

/* Medium: PA1 toggles every 300ms (~3Hz) */
static void MediumLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_1);
        vTaskDelay(pdMS_TO_TICKS(MEDIUM_PERIOD_MS));
    }
}

/* Slow: PC13 toggles every 500ms (1Hz) — lowest priority */
static void SlowLED_Task(void *pvParameters)
{
    (void)pvParameters;
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
        vTaskDelay(pdMS_TO_TICKS(SLOW_PERIOD_MS));
    }
}

/* --- GPIO Configuration ---------------------------------------------------- */

/**
  * @brief  Initialize 3 GPIO pins as push-pull outputs with pull-up.
  *         PB13 = on-board LED3, PA1 & PC13 = optional external LEDs.
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
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_SET);  /* start HIGH = LED off */

    /* PA1 — external LED (optional) */
    GPIO_InitStruct.Pin = GPIO_PIN_1;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);

    /* PC13 — external LED (optional) */
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
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
  * @brief  Stack overflow hook — called by FreeRTOS when a task overflows its stack.
  *         configCHECK_FOR_STACK_OVERFLOW=2 enables this check.
  */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    __disable_irq();
    for (;;) {}  /* Set breakpoint here to inspect xTask and pcTaskName */
}

/**
  * @brief  HAL error handler.
  */
void APP_ErrorHandler(void)
{
    __disable_irq();
    for (;;) {}
}
```

---

## 验证清单

```
□ 1. 编译 0 Error, 0 Warning
□ 2. 下载运行后, PB13 快速闪烁 (~5Hz)
□ 3. 如果接了外接 LED: PA1 中速, PC13 最慢
□ 4. 用示波器/逻辑分析仪测 PB13: 周期 = 200ms (100ms ON + 100ms OFF)
□ 5. 三个 GPIO 波形完全独立, 互不干扰
```

---

## 观察要点

| 现象 | 说明 |
|------|------|
| PB13 不一定最先开始 | 创建顺序 ≠ 执行顺序，调度器选最高优先级 |
| PA1 被 PB13 抢占 | FastLED 延时到期后，立即抢占 MediumLED |
| 三个频率完全独立 | 每个任务有自己的延时计数器, 互不干扰 |

> 📎 **进阶版:** 见 `Lesson2_Task_Advanced.md` — 添加栈使用率监控和结构体全局变量
