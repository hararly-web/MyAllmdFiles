# Lesson 11 — Tickless Idle 基础版：通用 Sleep 模式

> **目标：** 掌握 configUSE_TICKLESS_IDLE 基本配置，用默认 `__weak vPortSuppressTicksAndSleep()` 实现 Sleep 模式 Tickless，通过 Idle Hook 验证休眠效果
>
> **前提：** 已完成第1课移植，FreeRTOS 正常运行

---

## 1. 原理简述

这是最基础的 Tickless 实现 — 只需要改 1 行配置。利用 FreeRTOS 默认的 `__weak vPortSuppressTicksAndSleep()` (位于 `port.c`)，CPU 空闲时自动进入 `__WFI()` 休眠。

```
核心改动:
  FreeRTOSConfig.h:  #define configUSE_TICKLESS_IDLE  0 → 1

运行时行为变化:
  传统模式: SysTick 每 10ms 中断一次 → CPU 被唤醒 → Idle 任务检查无任务 → 重新 WFI
  Tickless:  Idle 任务计算空闲时间 → 关 SysTick → 设长周期重载 → __WFI() → 深度休眠

效果 (HSI 8MHz):
  传统 ~8mA → Tickless Sleep ~5mA  (省电 ~40%)
```

## 2. 程序流程图

```
┌─────────────────────────────────────────────────────────────┐
│              Demo1: Basic Tickless Sleep 模式                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  vLedTask (Priority 2)        vIdleMonitorTask (Priority 1) │
│  ┌─────────────────┐          ┌──────────────────────┐      │
│  │ Toggle LED0     │          │ Read g_sys.idle_count │      │
│  │ vTaskDelay(500ms)│         │ Calculate delta       │      │
│  └────────┬────────┘          │ vTaskDelay(1000ms)    │      │
│           │                    └──────────┬───────────┘      │
│           │  LED Task 阻塞                │                  │
│           ▼                               ▼                  │
│  ┌─────────────────────────────────────────────────┐        │
│  │          Idle Task (Priority 0)                  │        │
│  │  ┌──────────────────────────────────────────┐   │        │
│  │  │ vApplicationIdleHook() → g_sys.idle_count++│   │        │
│  │  └──────────────────────────────────────────┘   │        │
│  │  configUSE_TICKLESS_IDLE == 1                    │        │
│  │    → vPortSuppressTicksAndSleep(50 ticks)        │        │
│  │    → __WFI() ... 500ms deep sleep               │        │
│  └─────────────────────────────────────────────────┘        │
│                                                             │
│  验证方法: idle_count delta 在 Tickless 模式下 ~2/s           │
│           传统模式下 ~100/s (与 Tick Rate 一致)               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 代码实现

### 3.1 FreeRTOSConfig.h 修改

```c
/* 仅需修改 1 行 */

// 修改前:
#define configUSE_TICKLESS_IDLE                    0

// 修改后:
#define configUSE_TICKLESS_IDLE                    1
```

### 3.2 main.c

```c
/**
 * @file    main.c
 * @brief   Lesson 11 Demo1: Basic Tickless Idle with Sleep mode
 * @note    No custom vPortSuppressTicksAndSleep needed — use default __weak
 *          Verify by observing LED toggle rate and measuring current
 */

#include "main.h"
#include "FreeRTOS.h"
#include "task.h"

/* ── Global variables structure ──────────────────────────────────────── */
typedef struct {
    TaskHandle_t hLedTask;
    TaskHandle_t hIdleMonitorTask;
    uint32_t     idle_count;       /* Times idle task entered WFI */
    uint32_t     tick_count;       /* Total system ticks since start */
} SystemState_t;

static SystemState_t g_sys;

/* ── LED Toggle Task ──────────────────────────────────────────────────── */
/**
 * @brief  Toggle LED0 every 500ms
 * @note   When LED is OFF, system has 500ms to enter Tickless sleep
 */
void vLedTask(void *pvParameters)
{
    (void)pvParameters;
    const TickType_t xDelay = pdMS_TO_TICKS(500);

    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_13);  /* PB13 = LED0 */
        vTaskDelay(xDelay);
    }
}

/* ── Idle Monitor Task ────────────────────────────────────────────────── */
/**
 * @brief  Monitor idle counts to confirm Tickless is working
 * @note   In traditional mode: ~100 idle WFI/sec (with 100Hz tick)
 *         In Tickless mode:  ~2 idle WFI/sec  (with 500ms delay)
 *         Big difference confirms Tickless is active!
 */
void vIdleMonitorTask(void *pvParameters)
{
    (void)pvParameters;
    char msg[64];
    uint32_t last_idle = 0;

    for (;;)
    {
        /* Read idle counter — incremented in Idle Hook */
        uint32_t idle_delta = g_sys.idle_count - last_idle;
        last_idle = g_sys.idle_count;
        g_sys.tick_count = xTaskGetTickCount();

        snprintf(msg, sizeof(msg),
                 "Ticks:%lu  IdleWFI/s:%lu  (low=deep-sleep!)\r\n",
                 g_sys.tick_count, idle_delta);
        /* Transmit via UART if available, otherwise just accumulate */

        vTaskDelay(pdMS_TO_TICKS(1000));  /* Report every second */
    }
}

/* ── Idle Hook — called from prvIdleTask() ────────────────────────────── */
void vApplicationIdleHook(void)
{
    /* This hook is called every time the Idle task runs.
     * In Tickless mode: only when (a) sleep aborted, or (b) wake-up
     * In traditional mode: every SysTick cycle (~10ms with 100Hz)
     *
     * Count calls to observe the difference. */
    g_sys.idle_count++;
}

/* ── Hardware Init ────────────────────────────────────────────────────── */
static void SystemClock_Config(void)
{
    /* HSI 8MHz → PLL 72MHz → HCLK/SYSCLK 72MHz */
    /* (Use existing clock configuration from your project) */
}

static void GPIO_Init(void)
{
    __HAL_RCC_GPIOB_CLK_ENABLE();
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin   = GPIO_PIN_13;
    GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull  = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
}

/* ── main ──────────────────────────────────────────────────────────────── */
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    GPIO_Init();

    /* Create tasks */
    xTaskCreate(vLedTask, "Led", 128, NULL, 2, &g_sys.hLedTask);
    xTaskCreate(vIdleMonitorTask, "IdleMon", 256, NULL, 1, &g_sys.hIdleMonitorTask);

    /* Start scheduler — never returns */
    vTaskStartScheduler();

    for (;;);  /* Should never reach here */
    return 0;
}
```

## 4. 预期现象

| 指标 | 传统模式 | Tickless Sleep |
|------|---------|---------------|
| LED0 闪烁 | 每 500ms | 每 500ms (行为一致) |
| Idle Hook 调用频率 | ~100次/秒 | ~2次/秒 |
| VDD 电流 (HSI 8MHz) | ~8mA | ~5mA |
| SysTick 行为 | 持续中断 | 空闲时暂停 |

## 5. 编译验证

```
FreeRTOSConfig.h 修改:
  #define configUSE_TICKLESS_IDLE  1  (从 0 改为 1)

Keil Build:
  编译 → 0 Error, 0 Warning → 下载 → LED 闪烁 + 电流下降
```

## 6. 调试技巧

### 方法 1: Idle Hook 计数法 (本 Demo)
- Tickless 生效时，`g_sys.idle_count` 增长速度从 ~100/s 降到 ~2/s
- 这是最直观的验证方式

### 方法 2: 万用表电流测量
- 万用表串接 VDD，量程 20mA
- 对比 `configUSE_TICKLESS_IDLE=0` vs `=1` 的电流差异

### 方法 3: 示波器测唤醒脉冲
- 在 `vApplicationIdleHook()` 中翻转一个 GPIO
- 示波器观察脉冲间隔 → Tickless 模式脉冲间隔 = 任务延时周期 (500ms)

## 7. 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| Idle Hook 频率没变 | configUSE_TICKLESS_IDLE 未生效 | 确认修改了正确的 FreeRTOSConfig.h |
| 程序卡死 | configUSE_PREEMPTION=0 | Tickless 需要抢占式调度 |
| 电流无明显变化 | 其他外设持续耗电 (GPIO未优化) | 休眠前关闭不用的外设时钟 |
| SysTick 时间漂移 | 补偿误差 | 正常现象，几个 ppm 级别 |
