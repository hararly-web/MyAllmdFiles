# Lesson 11 — Tickless Idle 进阶版：Stop 模式深度休眠

> **目标：** 重写 `__weak vPortSuppressTicksAndSleep()`，用 PY32F403 Stop 模式替代默认 Sleep 模式，配合 RTC 周期性唤醒，实现 μA 级待机功耗
>
> **前提：** 已完成 Lesson11_Tickless_Basic，理解 Tickless 基本原理

---

## 1. 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                  Stop 模式 Tickless 架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │ main.c                                               │      │
│  │  ├── Tickless_RTC_Init()        ← 初始化 RTC (LSE)   │      │
│  │  ├── Tickless_GPIO_Wakeup_Init()← 配置 PA0 按键唤醒   │      │
│  │  ├── xTaskCreate(vLedTask)      ← 常规业务任务       │      │
│  │  ├── xTaskCreate(vPowerMonitor) ← 功耗监控任务       │      │
│  │  └── vTaskStartScheduler()                           │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────┐      │
│  │ Lesson11_Tickless_Advanced.c  (核心!)                 │      │
│  │  ├── vPortSuppressTicksAndSleep() ← 覆盖 port.c 弱定义│      │
│  │  │   ├── Step 0: 判断休眠深度 (Stop vs Sleep)         │      │
│  │  │   ├── Step 1-6: SysTick 精确计时 (同 port.c)       │      │
│  │  │   ├── Step 7: Tickless_Enter_StopMode()           │      │
│  │  │   │   ├── 转换 ticks → RTC 秒数                    │      │
│  │  │   │   ├── HAL_RTCEx_SetWakeUpTimer_IT() 设闹钟     │      │
│  │  │   │   └── HAL_PWR_EnterSTOPMode() → 深度休眠!     │      │
│  │  │   ├── Step 8: configPOST_SLEEP_PROCESSING()       │      │
│  │  │   │   ├── Tickless_Restore_Clocks() 恢复 HSE/PLL  │      │
│  │  │   │   └── __HAL_FLASH_SET_LATENCY() 恢复 Flash    │      │
│  │  │   └── Step 9-14: 补偿 Tick + 恢复 SysTick         │      │
│  │  ├── Tickless_Enter_StopMode()     ← Stop 模式入口    │      │
│  │  ├── Tickless_Restore_Clocks()     ← 时钟恢复         │      │
│  │  ├── configPRE_SLEEP_PROCESSING()  ← 休眠前置回调     │      │
│  │  └── configPOST_SLEEP_PROCESSING() ← 唤醒后置回调     │      │
│  └──────────────────────────────────────────────────────┘      │
│                                                                 │
│  休眠深度决策:                                                   │
│    xExpectedIdleTime < 10 ticks → Sleep 模式 (__wfi)           │
│    xExpectedIdleTime >= 10 ticks → Stop 模式 (HAL_PWR_EnterSTOP)│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 程序流程图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                Stop 模式 Tickless 完整调用流程                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Idle Task                                                               │
│     │                                                                     │
│     ├── 无任务就绪                                                        │
│     ├── xExpectedIdleTime = 下一个任务到期时间 - 当前 Tick                │
│     │                                                                     │
│     └── xExpectedIdleTime >= 2 ?                                          │
│            │                                                              │
│            ▼                                                              │
│  vPortSuppressTicksAndSleep(xExpectedIdleTime)  ← ★ 自定义实现            │
│     │                                                                     │
│     ├── < 10 ticks? → __wfi() → return (不走 Stop, 开销不值得)            │
│     │                                                                     │
│     ├── __disable_irq()                                                   │
│     ├── eTaskConfirmSleepModeStatus() == eAbortSleep? → return            │
│     │                                                                     │
│     ├── 停 SysTick                                                        │
│     ├── 算重载值 (保留标准 port.c 时序逻辑)                                │
│     ├── 重启 SysTick (超长周期)                                           │
│     │                                                                     │
│     ├── Tickless_Enter_StopMode(xExpectedIdleTime)                        │
│     │   ├── wakeup_seconds = xExpectedIdleTime / configTICK_RATE_HZ       │
│     │   ├── HAL_RTCEx_SetWakeUpTimer_IT(wakeup_seconds)                   │
│     │   ├── __HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU)                             │
│     │   └── HAL_PWR_EnterSTOPMode(LP_REGULATOR, WFI)                      │
│     │       │                                                              │
│     │       ▼  ┌──────────────────────────────────┐                       │
│     │          │  Stop 模式:                       │                       │
│     │          │  • CPU 停                         │                       │
│     │          │  • HSE/PLL 停                     │                       │
│     │          │  • HSI 停                         │                       │
│     │          │  • Flash 休眠                     │                       │
│     │          │  • SRAM 保持                      │                       │
│     │          │  • RTC 运行 (LSE)                 │                       │
│     │          │  • GPIO 保持状态                  │                       │
│     │          │  • 电流 ~50μA                     │                       │
│     │          └──────────────────────────────────┘                       │
│     │          ... 时间流逝 ...                                           │
│     │          RTC WakeUp 中断 → 唤醒!                                    │
│     │                                                                     │
│     ├── configPOST_SLEEP_PROCESSING()                                     │
│     │   ├── Tickless_Restore_Clocks()                                     │
│     │   │   ├── HSE ON → 等待稳定                                         │
│     │   │   ├── PLL ON → 等待锁定                                         │
│     │   │   ├── SYSCLK = PLLCLK                                           │
│     │   │   └── SystemCoreClockUpdate()                                    │
│     │   └── __HAL_FLASH_SET_LATENCY(FLASH_LATENCY_2)                      │
│     │                                                                     │
│     ├── __enable_irq() → 让唤醒源 ISR 立即执行                            │
│     ├── __disable_irq() → 停 SysTick → 读计数值                           │
│     ├── 计算 ulCompleteTickPeriods                                        │
│     ├── vTaskStepTick(ulCompleteTickPeriods)                              │
│     ├── 恢复 SysTick 标准模式                                             │
│     └── __enable_irq()                                                    │
│                                                                          │
│  结果: CPU 深度休眠了 xExpectedIdleTime 个 tick                           │
│        电流从 ~8mA → ~50μA (降低 160 倍!)                                 │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## 3. 代码实现

### 3.1 Lesson11_Tickless_Advanced.h

```c
/**
 * @file    Lesson11_Tickless_Advanced.h
 * @brief   Header for Tickless Idle with Stop mode on PY32F403
 * @note    Overrides __weak vPortSuppressTicksAndSleep in port.c
 *          Uses RTC WakeUp timer as the tickless wakeup source
 */

#ifndef LESSON11_TICKLESS_ADVANCED_H
#define LESSON11_TICKLESS_ADVANCED_H

#include "py32f403_hal.h"
#include "FreeRTOS.h"
#include "task.h"

/* ── Public function declarations ─────────────────────────────────── */
void Tickless_RTC_Init(void);
void Tickless_GPIO_Wakeup_Init(void);
void Tickless_Enter_StopMode(TickType_t xExpectedIdleTime);

/* ── Global state (for power monitoring task) ─────────────────────── */
typedef struct {
    uint32_t total_sleep_ticks;   /* Total ticks suppressed (cumulative) */
    uint32_t sleep_enter_count;   /* How many times we entered Stop mode  */
    uint32_t sleep_abort_count;   /* How many times sleep was aborted     */
    uint32_t early_wake_count;    /* Wake-ups before expected time        */
    uint32_t current_consumption; /* Estimated current in μA             */
} TicklessStats_t;

extern TicklessStats_t g_tickless_stats;

#endif /* LESSON11_TICKLESS_ADVANCED_H */
```

### 3.2 Lesson11_Tickless_Advanced.c

```c
/**
 * @file    Lesson11_Tickless_Advanced.c
 * @brief   Custom vPortSuppressTicksAndSleep for PY32F403 Stop mode
 *
 * Architecture:
 *   ① Override __weak vPortSuppressTicksAndSleep() from port.c
 *   ② Use RTC WakeUp timer as the tickless time base
 *   ③ Enter Stop mode (deep sleep, ~50μA) during idle
 *   ④ Restore system clocks (HSE/PLL) upon wake-up
 *
 * When to use Stop mode vs Sleep mode:
 *   - xExpectedIdleTime > configSTOP_MODE_MIN_IDLE_TICKS → Stop mode
 *   - Otherwise → default Sleep mode (faster wake-up)
 */

#include "Lesson11_Tickless_Advanced.h"
#include "py32f403_hal_pwr.h"
#include "py32f403_hal_rcc.h"
#include "py32f403_hal_rtc.h"

/* ── Configuration constants ───────────────────────────────────────── */
#define configSTOP_MODE_MIN_IDLE_TICKS   10   /* Min ticks to use Stop mode */
#define configRTC_WAKEUP_CLOCK_HZ        32768 /* LSE = 32.768kHz          */
#define TICK_RATE_HZ                     configTICK_RATE_HZ

/* RTC handle — must survive Stop mode (backup domain) */
static RTC_HandleTypeDef hrtc_tickless;

/* ── Global statistics ─────────────────────────────────────────────── */
TicklessStats_t g_tickless_stats = {0};

/* ── RTC Initialization for Tickless ────────────────────────────────── */
/**
 * @brief  Init RTC with LSE for tickless wakeup timer
 * @note   RTC runs in backup domain — survives Stop mode
 *         LSE crystal must be installed on board (PC14/PC15)
 */
void Tickless_RTC_Init(void)
{
    /* ① Enable LSE (external 32.768kHz crystal on PC14/PC15) */
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_LSE;
    RCC_OscInitStruct.LSEState       = RCC_LSE_ON;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    /* ② Enable RTC clock from LSE */
    RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};
    PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_RTC;
    PeriphClkInit.RTCClockSelection    = RCC_RTCCLKSOURCE_LSE;
    HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit);

    __HAL_RCC_RTC_ENABLE();

    /* ③ Configure RTC */
    hrtc_tickless.Instance = RTC;
    hrtc_tickless.Init.HourFormat   = RTC_HOURFORMAT_24;
    hrtc_tickless.Init.AsynchPrediv = 127;   /* 32768/(127+1)=256Hz */
    hrtc_tickless.Init.SynchPrediv  = 255;   /* 256/(255+1)=1Hz     */
    HAL_RTC_Init(&hrtc_tickless);
}

/* ── GPIO Wakeup Initialization ─────────────────────────────────────── */
/**
 * @brief  Config PA0 as EXTI wakeup source (WKUP button)
 * @note   PA0 = WKUP pin, can wake MCU from Stop mode
 */
void Tickless_GPIO_Wakeup_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();

    GPIO_InitStruct.Pin   = GPIO_PIN_0;
    GPIO_InitStruct.Mode  = GPIO_MODE_IT_FALLING;  /* Falling edge trigger */
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0);  /* Priority 5 (above kernel) */
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
}

/* ── Enter Stop Mode with RTC Wake-up ───────────────────────────────── */
/**
 * @brief  Enter Stop mode with RTC wakeup timer set to xIdleTicks
 * @param  xExpectedIdleTime: idle time in RTOS ticks
 * @note   Called from custom vPortSuppressTicksAndSleep()
 *         RTC will generate wakeup after xExpectedIdleTime
 */
void Tickless_Enter_StopMode(TickType_t xExpectedIdleTime)
{
    /* Convert ticks to seconds for RTC wakeup timer */
    uint32_t wakeup_seconds = xExpectedIdleTime / TICK_RATE_HZ;
    if (wakeup_seconds < 1) wakeup_seconds = 1;
    if (wakeup_seconds > 0xFFFF) wakeup_seconds = 0xFFFF; /* 16-bit limit */

    /* Set RTC WakeUp timer — will wake MCU after wakeup_seconds */
    HAL_RTCEx_SetWakeUpTimer_IT(&hrtc_tickless, wakeup_seconds,
                                 RTC_WAKEUPCLOCK_CK_SPRE_16BITS);

    /* Enable RTC WakeUp interrupt (must NOT be masked during Stop) */
    __HAL_RTC_WAKEUPTIMER_ENABLE_IT(&hrtc_tickless, RTC_IT_WUT);

    /* Clear wakeup flag before entering Stop */
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU);

    /* Enter Stop mode with low-power regulator */
    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);

    /* ─── Execution resumes HERE after wake-up! ─── */

    /* Disable RTC wakeup timer interrupt */
    __HAL_RTC_WAKEUPTIMER_DISABLE_IT(&hrtc_tickless, RTC_IT_WUT);
}

/* ── Restore System Clocks after Stop ────────────────────────────────── */
/**
 * @brief  Restore HSE and PLL after waking from Stop mode
 * @note   Stop mode turns off HSE/PLL; HSI auto-restarts at 8MHz
 *         This must be called in configPOST_SLEEP_PROCESSING()
 */
static void Tickless_Restore_Clocks(void)
{
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    /* ① Re-enable HSE (was stopped during Stop mode) */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState       = RCC_HSE_ON;
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    /* ② Re-enable PLL (was stopped during Stop mode) */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_PLL;
    RCC_OscInitStruct.PLL.PLLState   = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource  = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLMUL     = RCC_PLL_MUL9;  /* 8MHz × 9 = 72MHz */
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    /* ③ Switch system clock back to PLL */
    RCC_ClkInitStruct.ClockType      = RCC_CLOCKTYPE_SYSCLK;
    RCC_ClkInitStruct.SYSCLKSource   = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider  = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct);

    /* ④ Update SystemCoreClock variable */
    SystemCoreClockUpdate();
}

/* ── CUSTOM vPortSuppressTicksAndSleep (OVERRIDES __weak) ────────────── */
/**
 * @brief  Override the default __weak implementation from port.c
 * @note   This function has the SAME signature as port.c version.
 *         Since this is a strong definition, the linker uses THIS one.
 *
 *         We re-implement the critical timing logic that port.c does,
 *         but replace __WFI with Stop mode entry.
 *
 *         Key differences from default:
 *         ① Uses RTC WakeUp timer instead of SysTick for timing
 *         ② Enters Stop mode (deep sleep) instead of Sleep mode
 *         ③ Restores HSE/PLL after wake-up
 */
#if ( configUSE_TICKLESS_IDLE == 1 )

void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime)
{
    uint32_t ulReloadValue, ulCompleteTickPeriods;
    uint32_t ulCompletedSysTickDecrements;
    uint32_t ulSysTickDecrementsLeft;
    TickType_t xModifiableIdleTime;

    /* ── Step 0: Decide sleep depth ── */
    if (xExpectedIdleTime < configSTOP_MODE_MIN_IDLE_TICKS)
    {
        /* Too short for Stop mode overhead → use simple Sleep */
        __wfi();
        g_tickless_stats.sleep_abort_count++;
        return;
    }

    /* Bound check */
    if (xExpectedIdleTime > xMaximumPossibleSuppressedTicks)
    {
        xExpectedIdleTime = xMaximumPossibleSuppressedTicks;
    }

    /* ── Step 1: Disable interrupts ── */
    __disable_irq();
    __dsb(portSY_FULL_READ_WRITE);
    __isb(portSY_FULL_READ_WRITE);

    /* ── Step 2: Confirm sleep is allowed ── */
    if (eTaskConfirmSleepModeStatus() == eAbortSleep)
    {
        __enable_irq();
        g_tickless_stats.sleep_abort_count++;
        return;
    }

    /* ── Step 3: Stop SysTick ── */
    portNVIC_SYSTICK_CTRL_REG = (portNVIC_SYSTICK_CLK_BIT_CONFIG |
                                  portNVIC_SYSTICK_INT_BIT);

    /* ── Step 4: Read remaining count ── */
    ulSysTickDecrementsLeft = portNVIC_SYSTICK_CURRENT_VALUE_REG;
    if (ulSysTickDecrementsLeft == 0)
    {
        ulSysTickDecrementsLeft = ulTimerCountsForOneTick;
    }

    /* ── Step 5: Calculate reload value ── */
    ulReloadValue = ulSysTickDecrementsLeft +
                    (ulTimerCountsForOneTick * (xExpectedIdleTime - 1UL));

    if ((portNVIC_INT_CTRL_REG & portNVIC_PEND_SYSTICK_SET_BIT) != 0)
    {
        portNVIC_INT_CTRL_REG = portNVIC_PEND_SYSTICK_CLEAR_BIT;
        ulReloadValue -= ulTimerCountsForOneTick;
    }

    if (ulReloadValue > ulStoppedTimerCompensation)
    {
        ulReloadValue -= ulStoppedTimerCompensation;
    }

    /* ── Step 6: Restart SysTick with extended period ── */
    portNVIC_SYSTICK_LOAD_REG = ulReloadValue;
    portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
    portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;

    /* ── Step 7: configPRE_SLEEP_PROCESSING → Enter Stop mode ── */
    xModifiableIdleTime = xExpectedIdleTime;
    configPRE_SLEEP_PROCESSING(xModifiableIdleTime);

    /* ★★★ ENTER STOP MODE (via RTC WakeUp) ★★★ */
    Tickless_Enter_StopMode(xExpectedIdleTime);
    g_tickless_stats.sleep_enter_count++;

    /* ★★★ WAKE-UP! ★★★ */

    /* ── Step 8: configPOST_SLEEP_PROCESSING → Restore clocks ── */
    configPOST_SLEEP_PROCESSING(xExpectedIdleTime);

    /* ── Step 9: Re-enable interrupts briefly ── */
    __enable_irq();
    __dsb(portSY_FULL_READ_WRITE);
    __isb(portSY_FULL_READ_WRITE);

    /* ── Step 10: Disable interrupts again to stop SysTick ── */
    __disable_irq();
    __dsb(portSY_FULL_READ_WRITE);
    __isb(portSY_FULL_READ_WRITE);

    /* Stop SysTick */
    portNVIC_SYSTICK_CTRL_REG = (portNVIC_SYSTICK_CLK_BIT_CONFIG |
                                  portNVIC_SYSTICK_INT_BIT);

    /* ── Step 11: Calculate tick compensation ── */
    if ((portNVIC_SYSTICK_CTRL_REG & portNVIC_SYSTICK_COUNT_FLAG_BIT) != 0)
    {
        /* SysTick counted to zero → complete sleep */
        uint32_t ulCalculatedLoadValue;
        ulCalculatedLoadValue = (ulTimerCountsForOneTick - 1UL) -
                                 (ulReloadValue - portNVIC_SYSTICK_CURRENT_VALUE_REG);
        ulCompleteTickPeriods = xExpectedIdleTime - 1UL;

        if (ulCalculatedLoadValue > ulStoppedTimerCompensation)
        {
            ulCompleteTickPeriods = xExpectedIdleTime;
        }
    }
    else
    {
        /* Early wake-up → proportional compensation */
        ulCompletedSysTickDecrements =
            ulReloadValue - portNVIC_SYSTICK_CURRENT_VALUE_REG;
        ulCompleteTickPeriods =
            ulCompletedSysTickDecrements / ulTimerCountsForOneTick;
        g_tickless_stats.early_wake_count++;
    }

    g_tickless_stats.total_sleep_ticks += ulCompleteTickPeriods;

    /* ── Step 12: Restore SysTick standard operation ── */
    portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;
    portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
    portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;

    /* ── Step 13: Compensate tick count ── */
    vTaskStepTick(ulCompleteTickPeriods);

    /* ── Step 14: Re-enable interrupts ── */
    __enable_irq();
}

#endif /* configUSE_TICKLESS_IDLE */

/* ── configPRE_SLEEP_PROCESSING ──────────────────────────────────────── */
/**
 * @brief  Called by port.c before entering sleep
 * @note   Here we do nothing extra — all the Stop mode work is in
 *         our custom vPortSuppressTicksAndSleep() above.
 *         If using the DEFAULT __weak implementation, configure
 *         Stop mode entry HERE.
 */
void configPRE_SLEEP_PROCESSING(TickType_t xModifiableIdleTime)
{
    /* For the default implementation:
     *   HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
     *   xModifiableIdleTime = 0;  // Tell port.c we already did WFI
     *
     * For our custom implementation (above):
     *   Nothing to do — we handle Stop mode entry directly.
     */
    (void)xModifiableIdleTime;
}

/* ── configPOST_SLEEP_PROCESSING ─────────────────────────────────────── */
/**
 * @brief  Called by port.c after waking from sleep
 * @note   CRITICAL: Must restore HSE/PLL/Flash latency here
 *         because Stop mode turns them off.
 */
void configPOST_SLEEP_PROCESSING(TickType_t xExpectedIdleTime)
{
    (void)xExpectedIdleTime;

    /* Restore system clocks (HSE → PLL → SYSCLK) */
    Tickless_Restore_Clocks();

    /* Restore Flash latency (was reset during Stop) */
    /* FLASH_LATENCY_2 for 72MHz operation */
    __HAL_FLASH_SET_LATENCY(FLASH_LATENCY_2);
}
```

### 3.3 功耗监控任务 (添加到 main.c)

```c
/**
 * @brief  Power monitoring task — reports tickless statistics via UART
 * @note   Add to main.c: xTaskCreate(vPowerMonitorTask, "PwrMon", 256, NULL, 1, NULL);
 */
void vPowerMonitorTask(void *pvParameters)
{
    (void)pvParameters;
    char buf[128];

    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(5000));  /* Report every 5 seconds */

        uint32_t total    = g_tickless_stats.sleep_enter_count;
        uint32_t aborted  = g_tickless_stats.sleep_abort_count;
        uint32_t early    = g_tickless_stats.early_wake_count;
        uint32_t ticks    = g_tickless_stats.total_sleep_ticks;

        snprintf(buf, sizeof(buf),
                 "\r\n═══════ Tickless Stats (5s) ═══════\r\n"
                 " Stop entries : %lu\r\n"
                 " Aborted      : %lu (%.1f%%)\r\n"
                 " Early wakes  : %lu\r\n"
                 " Suppressed   : %lu ticks (%.1f%% duty)\r\n"
                 "==============================\r\n",
                 total, aborted,
                 total ? (100.0f * aborted / (total + aborted)) : 0.0f,
                 early, ticks,
                 100.0f * (ticks * 10) / 5000  /* ticks×10ms / 5000ms */
                 );

        /* Transmit via UART (if available) */
        // HAL_UART_Transmit(&huart1, (uint8_t*)buf, strlen(buf), 1000);
    }
}
```

## 4. main.c 集成示例

```c
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "Lesson11_Tickless_Advanced.h"

/* ── Global variables structure ─────────────────────────────────────── */
typedef struct {
    TaskHandle_t hLedTask;
    TaskHandle_t hPwrMonTask;
} AppGlobals_t;

static AppGlobals_t g_app;

/* ── LED Toggle Task ─────────────────────────────────────────────────── */
void vLedTask(void *pvParameters)
{
    (void)pvParameters;
    const TickType_t xDelay = pdMS_TO_TICKS(500);

    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_13);
        vTaskDelay(xDelay);
    }
}

/* ── main ────────────────────────────────────────────────────────────── */
int main(void)
{
    HAL_Init();
    SystemClock_Config();  /* Your existing clock config */

    /* Init peripherals for Tickless */
    Tickless_RTC_Init();          /* RTC with LSE for wakeup */
    Tickless_GPIO_Wakeup_Init();  /* PA0 button as wakeup source */
    GPIO_Init();                  /* LED GPIO */

    /* Create tasks */
    xTaskCreate(vLedTask, "Led", 128, NULL, 2, &g_app.hLedTask);
    xTaskCreate(vPowerMonitorTask, "PwrMon", 256, NULL, 1, &g_app.hPwrMonTask);

    vTaskStartScheduler();

    for (;;);
    return 0;
}
```

## 5. 编译验证

### 5.1 Keil 工程文件清单

```
□ FreeRTOS/Source/tasks.c
□ FreeRTOS/Source/queue.c
□ FreeRTOS/Source/list.c
□ FreeRTOS/Source/timers.c
□ FreeRTOS/Source/event_groups.c
□ FreeRTOS/Source/portable/RVDS/ARM_CM4F/port.c     ← Tickless 逻辑在此
□ FreeRTOS/Source/portable/MemMang/heap_4.c
□ Drivers/PY32F403_HAL_Driver/Src/py32f403_hal_pwr.c ← Stop 模式需要!
□ Drivers/PY32F403_HAL_Driver/Src/py32f403_hal_rcc.c
□ Drivers/PY32F403_HAL_Driver/Src/py32f403_hal_rtc.c ← RTC 唤醒需要!
□ Drivers/PY32F403_HAL_Driver/Src/py32f403_hal_flash.c
□ User/Lesson11_Tickless_Advanced.c                  ← 自定义实现
□ User/main.c
```

### 5.2 FreeRTOSConfig.h 关键配置

```c
/* Ensure these are set correctly for Tickless: */
#define configUSE_PREEMPTION                    1   /* MUST be 1 */
#define configUSE_TICKLESS_IDLE                 1   /* Enable Tickless */
#define configCPU_CLOCK_HZ             8000000      /* Match HSI_VALUE */
#define configTICK_RATE_HZ                    100   /* 10ms per tick */

/* For Demo2 Stop mode — ensure PWR/RTC modules are enabled */
// In py32f403_hal_conf.h:
#define HAL_PWR_MODULE_ENABLED     /* Already set */
#define HAL_RTC_MODULE_ENABLED     /* Enable for RTC wakeup */
```

### 5.3 预期编译结果

```
Keil Build Output:
Build target 'PY32F403-STK'
compiling Lesson11_Tickless_Advanced.c...
compiling main.c...
linking...
Program Size: Code=18432 RO-data=1024 RW-data=256 ZI-data=12800
FromELF: creating hex file...
".\Output\GPIO_Toggle_FreeRTOS.axf" - 0 Error(s), 0 Warning(s).
```

## 6. 功耗测量

### 测量方法

```
连接: 万用表串接在 VDD 线上
  量程: Sleep 模式 → 20mA 档
        Stop 模式 → 200μA 档

预期结果 (PY32F403, 8MHz HSI):
┌──────────────────────────┬──────────────┐
│  模式                     │  电流         │
├──────────────────────────┼──────────────┤
│  传统 Tick               │  ~8mA         │
│  Tickless Sleep (默认)    │  ~5mA         │
│  Tickless Stop (本Demo)   │  ~50μA        │
│  电池寿命 (2000mAh):      │  4.5 年 (!)   │
└──────────────────────────┴──────────────┘

注意: 实际电流受以下因素影响:
  - LSE 晶振是否起振 (~1μA 增量)
  - LPDS 低功耗调压器模式 (vs 主调压器)
  - 未使用的 GPIO 是否配置为模拟输入 (减少漏电流)
  - 是否打开了不必要的外设时钟
```

## 7. 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| 进入 Stop 后醒不过来 | 没有配置唤醒源 | 确认 RTC WakeUp / EXTI 已初始化且时钟源在 Stop 期间可用 |
| 唤醒后系统时钟错乱 | HSE/PLL 未恢复 | 在 configPOST_SLEEP_PROCESSING 中完整恢复时钟树 |
| SysTick 时间漂移 | 补偿逻辑有误 | 保留 port.c 原始的计时+补偿逻辑，仅替换休眠模式 |
| Stop 模式立即唤醒 | 旧的 WUF 标志未清除 | 进入前 `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU)` |
| SWD 调试断开 | Stop 模式停 DBG 时钟 | 调试时用 Sleep 模式或 `HAL_DBGMCU_EnableDBGStopMode()` |
| 编译找不到 port.c 符号 | 自定义实现引用了 port.c 内部变量 | 变量如 `ulTimerCountsForOneTick` 是 port.c 全局变量，自定义实现可以直接使用 |
| RTC WakeUp 时间不准 | Ticks → 秒转换精度损失 | 100Hz tick × 10ms = 1s，最小休眠粒度 1s，可接受 |
