# FreeRTOS 移植踩坑记录

> 所有编译、链接、运行时错误的完整排查链路。按发生时间顺序排列。

---

## 目录

- [Session 1 — 编译/链接阶段错误](#session-1--编译链接阶段错误)
  - [错误 ① configTICK_TYPE_WIDTH_IN_BITS](#错误--configtick_type_width_in_bits)
  - [错误 ② configMAX_SYSCALL_INTERRUPT_PRIORITY](#错误--configmax_syscall_interrupt_priority)
  - [错误 ③ vApplicationStackOverflowHook 未定义](#错误--vapplicationstackoverflowhook)
- [Session 2 — 运行时 LED 不闪烁调试](#session-2--运行时-led-不闪烁调试)
  - [错误 ④ xPortSysTickHandler 隐式声明](#错误--xportsystickhandler-隐式声明)
  - [错误 ⑤ SVC_Handler/PendSV_Handler 重复定义](#错误--svc_handlerpendsv_handler-重复定义)
  - [错误 ⑥ configCPU_CLOCK_HZ 与实际时钟不匹配](#错误--configcpu_clock_hz-与实际时钟不匹配)
  - [错误 ⑦ Indirect Routing 尝试失败](#错误--indirect-routing-尝试失败)
  - [错误 ⑧ 诊断测试：确认调度器死锁](#错误--诊断测试确认调度器死锁)
  - [错误 ⑨ configASSERT 触发 → 调度器无法启动](#错误--configassert-触发--调度器无法启动)
- [Session 3 — configASSERT 根因定位与修复](#session-3--configassert-根因定位与修复)
- [最终有效配置总结](#最终有效配置总结)

---

# Session 1 — 编译/链接阶段错误

## 错误 ① configTICK_TYPE_WIDTH_IN_BITS

```
文件: portable/RVDS/ARM_CM4F/portmacro.h(73)
错误: #35: #error directive:
      configTICK_TYPE_WIDTH_IN_BITS set to unsupported tick type width.
```

**原因链:**

```
V11.3.0 的 portmacro.h 需要 configTICK_TYPE_WIDTH_IN_BITS
    ↓
项目 Inc/ 中的 FreeRTOSConfig.h 使用了旧版宏 configUSE_16_BIT_TICKS
    ↓
V11.3.0 不识别旧宏 → 进入 #else 分支 → #error
```

**修复:** `configUSE_16_BIT_TICKS 0` → `configTICK_TYPE_WIDTH_IN_BITS TICK_TYPE_WIDTH_32_BITS`

---

## 错误 ② configMAX_SYSCALL_INTERRUPT_PRIORITY

```
文件: portable/RVDS/ARM_CM4F/port.c(42)
错误: #35: #error directive:
      configMAX_SYSCALL_INTERRUPT_PRIORITY must not be set to 0.
```

**原因链:**

```
configMAX_SYSCALL_INTERRUPT_PRIORITY 的计算结果为 0
    ↓
可能原因 1: configPRIO_BITS 未定义 → (8 - 未定义) 行为不确定
可能原因 2: 直接写了 #define configMAX_SYSCALL_INTERRUPT_PRIORITY 0
可能原因 3: configMAX_API_CALL_INTERRUPT_PRIORITY 设为 0,
           V11.3.0 的 FreeRTOS.h 可能用它覆盖了前者
```

**修复:**
1. 确保 `configPRIO_BITS`、`configLIBRARY_LOWEST_INTERRUPT_PRIORITY`、`configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` 三个宏全部定义
2. 删除 `configMAX_API_CALL_INTERRUPT_PRIORITY` 的定义 (交给 FreeRTOS.h 自动推导)
3. 使用公式 `configMAX_SYSCALL_INTERRUPT_PRIORITY = (5 << (8 - 4))` → 结果 0x50 ≠ 0

---

## 错误 ③ vApplicationStackOverflowHook

```
Error: L6218E: Undefined symbol vApplicationStackOverflowHook
       (referred from tasks.o).
```

**原因链:**

```
FreeRTOSConfig.h 中 configCHECK_FOR_STACK_OVERFLOW = 2
    ↓
tasks.c 中引用了 vApplicationStackOverflowHook() 的地址
    ↓
链接器在所有 .o 文件中找不到这个函数的定义
    ↓
L6218E 链接错误
```

**修复:** 在 main.c 中实现该钩子函数：

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    __disable_irq();
    while (1) {}
}
```

> **注意:** main.c 必须先 `#include "FreeRTOS.h"` 和 `#include "task.h"`，否则 `TaskHandle_t` 类型和 `taskDISABLE_INTERRUPTS` 宏未定义。

---

# Session 2 — 运行时 LED 不闪烁调试

> 背景：编译 0 Error, 0 Warning，下载后 LED 一直亮/低电平，调度器不工作。

## 错误 ④ xPortSysTickHandler 隐式声明

```
文件: Src/py32f403_it.c(148)
警告: #223-D: function "xPortSysTickHandler" declared implicitly
```

**原因链:**

```
FreeRTOS V11.3.0 的 portmacro.h 没有导出 xPortSysTickHandler() 声明
    ↓
该函数仅在 port.c 内部做了前向声明 (static scope)
    ↓
py32f403_it.c 调用时编译器找不到原型 → 隐式声明警告
```

**修复:** 在 `py32f403_it.c` 顶部添加 extern 声明：

```c
extern void xPortSysTickHandler(void);   /* FreeRTOS tick handler, declared in port.c */
```

---

## 错误 ⑤ SVC_Handler/PendSV_Handler 重复定义

```
Error: L6200E: Symbol SVC_Handler multiply defined (by port.o and py32f403_it.o).
Error: L6200E: Symbol PendSV_Handler multiply defined (by port.o and py32f403_it.o).
```

**原因链:**

```
FreeRTOSConfig.h 中添加了:
  #define vPortSVCHandler    SVC_Handler
  #define xPortPendSVHandler PendSV_Handler
    ↓
port.c 中的 __asm void vPortSVCHandler(void) 被重命名为 SVC_Handler
    ↓
py32f403_it.c 中也定义了同名的 void SVC_Handler(void) {}
    ↓
链接器发现两份强定义 → L6200E
```

**修复:** 从 `py32f403_it.c` 中 **删除** (注释掉) `SVC_Handler` 和 `PendSV_Handler` 的空函数定义。它们由 port.c 经宏映射提供。

---

## 错误 ⑥ configCPU_CLOCK_HZ 与实际时钟不匹配

**现象:** 编译下载后 LED3 (PA1) 一直亮，等待 60s 无变化。

**原因链:**

```
FreeRTOSConfig.h:  configCPU_CLOCK_HZ = 20000000  (20MHz)
system_py32f403.c: HSI_VALUE = 8000000            (8MHz, 实际时钟源)
    ↓
SysTick reload = 20,000,000 / 100 = 200,000
实际 tick 频率 = 8,000,000 / 200,000 = 40 Hz (期望 100Hz)
    ↓
vTaskDelay(250) = 250 ticks / 40Hz = 6.25 秒
完整闪烁周期 = 12.5 秒
    ↓
肉眼观察 2~3 秒不到一次翻转 → 以为是"常亮"
```

**修复:**
1. `configCPU_CLOCK_HZ` 从 `20000000` 改为 `8000000`
2. `vTaskDelay(250)` 改为 `vTaskDelay(pdMS_TO_TICKS(250))` 获得精确 250ms 延时

---

## 错误 ⑦ Indirect Routing 尝试失败

**现象:** 将中断映射方式从 Direct Routing (宏 `#define`) 改为 Indirect Routing (C 函数调用) 后，LED 仍不闪烁。

**尝试的方法:**

```c
// py32f403_it.c
void SVC_Handler(void)
{
    vPortSVCHandler();     // ← 普通 C 函数调用
}
```

**失败根因:**

```
port.c 中的 vPortSVCHandler 是 __asm 函数，设计为 异常处理入口:
  __asm void vPortSVCHandler(void)
  {
      ldmia r0!, {r4-r11, r14}  ← 加载任务上下文，覆盖 R14 (LR)
      msr psp, r0
      bx r14                     ← 期望 R14 = 0xFFFFFFF9 (异常返回)
  }

当通过 C 函数调用时:
  BL vPortSVCHandler   ← C 调用指令，改写 LR = 返回地址
    ↓
  vPortSVCHandler 内部的 bx r14 使用被污染的 LR
    ↓
  返回到了错误的位置 → 栈帧损坏 → HardFault
```

**结论:** Cortex-M FreeRTOS 移植层 **必须** 使用 Direct Routing (宏映射)。`vPortSVCHandler` 和 `xPortPendSVHandler` 设计为异常向量入口，不能作为普通 C 函数被调用。

**回退到方案:** Direct Routing，即在 FreeRTOSConfig.h 中定义：

```c
#define vPortSVCHandler         SVC_Handler
#define xPortPendSVHandler      PendSV_Handler
```

并在 py32f403_it.c 中注释掉 SVC_Handler 和 PendSV_Handler。

---

## 错误 ⑧ 诊断测试：确认调度器死锁

**测试方法:** 将 main() 改为诊断版本，在调度器前后插入 LED 闪烁标记：

```c
int main(void)
{
    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /* Phase 1: 3 快闪 — 确认初始化通过 */
    for (int i = 0; i < 3; i++) {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_SET);
        HAL_Delay(100);
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_RESET);
        HAL_Delay(100);
    }

    /* Phase 2: 设 HIGH, 然后启动调度器 */
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_13, GPIO_PIN_SET);
    xTaskCreate(LED_Task, "LED", 128, NULL, 1, NULL);
    vTaskStartScheduler();    /* 应该永不返回 */

    /* Phase 3: 调度器失败返回! 1s 错误闪烁 */
    while (1) { ... }
}
```

**结果:** Phase 1 的 3 快闪正常 → Phase 2 进入 HIGH 后永不变化。

| 现象 | 诊断 |
|------|------|
| 3快闪 → 一直 HIGH | `vTaskStartScheduler()` 内部死锁, 未返回, 未启动任务 |

死锁只有两种可能:
1. **configASSERT 断言失败** → 关中断 + 死循环
2. **HardFault** → HardFault_Handler 死循环

---

## 错误 ⑨ configASSERT 触发 → 调度器无法启动

**验证方法:** 将 `configASSERT` 临时改为 `((void)(x))` (空操作)：

```c
#define configASSERT( x )    ((void)(x))
```

**结果:** 编译下载后 → LED 开始 250ms 闪烁！**调度器正常启动。**

**根因分析:**

```
vTaskStartScheduler() 或 xPortStartScheduler() 内部
    ↓
某个 configASSERT(x) 的条件 (x) 评估为 0 (false)
    ↓
进入断言处理: taskDISABLE_INTERRUPTS() + for(;;) 死循环
    ↓
所有中断关闭 (包括 SysTick) → CPU 永久挂起
    ↓
GPIO 保持最终状态 (PB13 HIGH) → 看起来 "LED 一直亮"
```

**可能的 assert 位置:**

最可能的触发点在 `xPortStartScheduler()` 的 NVIC 优先级验证代码：

```c
BaseType_t xPortStartScheduler( void )
{
    configASSERT( configMAX_SYSCALL_INTERRUPT_PRIORITY );  // ✅ 0x50 ≠ 0

    #if ( configASSERT_DEFINED == 1 )
    {
        // NVIC 优先级位数检测 — 读写 NVIC 优先级寄存器
        // PY32F403 的 NVIC 实现可能与此检测逻辑存在细微兼容性差异
        ...
        configASSERT( ... );  // ← 疑似触发点
    }
    #endif

    prvStartFirstTask();
}
```

**当前方案:** 在学习阶段使用 `configASSERT( x ) ((void)(x))` 绕过。生产环境需进一步定位具体触发断言的条件并修复。

---

# Session 3 — configASSERT 根因定位与修复

> 日期: 2026-07-08
> 目标: 定位 configASSERT 精确触发位置，修复根因，重新启用调试断言

## 根因定位过程

### 第一步：源码审查 — 列出 xPortStartScheduler() 中所有 configASSERT

审查文件: `portable/RVDS/ARM_CM4F/port.c` (FreeRTOS V11.3.0), 第 318~436 行。

| # | 行号 | 断言条件 | PY32F403 评估 |
|---|------|---------|--------------|
| 1 | 322 | `configASSERT(configMAX_SYSCALL_INTERRUPT_PRIORITY)` | `0x50 ≠ 0`，通过 |
| 2 | 327 | `configASSERT(portCPUID != M7_r0p1_ID)` | M4 ≠ M7，通过 |
| 3 | 328 | `configASSERT(portCPUID != M7_r0p0_ID)` | M4 ≠ M7，通过 |
| 4 | 361 | `configASSERT(ucMaxSysCallPriority)` | 非零，通过 |
| **5** | **365** | `configASSERT((configMAX & (~hw_mask)) == 0)` | **PRIMARY SUSPECT** |
| 6 | 395 | `configASSERT((configMAX & 0x1) == 0)` | 仅 8-bit NVIC 进入，PY32F403 跳过 |

### 第二步：定位断言 #5 (port.c:365) 的语义

```c
// port.c:347-365 (简化)
*pucFirstUserPriorityRegister = 0xFF;        // 写全 1 到 NVIC 优先级寄存器
ucMaxPriorityValue = *pucFirstUserPriorityRegister;  // 回读实际物理位

configASSERT( ( configMAX_SYSCALL_INTERRUPT_PRIORITY
                & ( ~ucMaxPriorityValue ) ) == 0U );
//             ↑ 检查 configMAX 的置位是否都在硬件实现的位范围内
```

此断言验证 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 不会在硬件未实现的优先级位上设置 bit。

### 第三步：查阅 PY32F403 数据手册

数据手册第 11.1.1 节原文：

> **8 个可编程的优先等级(使用了 3 位中断优先级)**

结论: PY32F403 NVIC 仅实现 **3 位** 优先级 (bit[7:5])，而非 FreeRTOSConfig.h 中配置的 4 位。

### 第四步：验证断言失败路径

```
假设条件:
  configPRIO_BITS = 4 (FreeRTOSConfig.h 中的旧值)
  configMAX = 5 << (8-4) = 5 << 4 = 0x50

硬件行为:
  写入 0xFF → NVIC IPR 寄存器 (0xE000E400)
  回读 0xE0 ← 仅 bit[7:5] 物理存在，bit[4:0] 读为零

断言计算:
  ucMaxPriorityValue     = 0xE0
  ~ucMaxPriorityValue     = 0x1F
  configMAX & 0x1F        = 0x50 & 0x1F = 0x10

  0x10 ≠ 0  →  configASSERT 触发 💥
  → taskDISABLE_INTERRUPTS() + for(;;) 死循环
  → SysTick 停止，所有中断关闭
  → GPIO 保持最终电平 → LED "常亮"
```

## 修复方案

### 修改 1: 修正 NVIC 优先级配置

文件: `Inc/FreeRTOSConfig.h`

```c
// 修改前 (错误)
#define configPRIO_BITS                         4
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY             15
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY        5
// → configMAX = 5 << 4 = 0x50 ❌

// 修改后 (正确)
#define configPRIO_BITS                         3
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY             7
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY        3
// → configMAX = 3 << 5 = 0x60 ✅
```

验证:
```
ucMaxPriorityValue = 0xE0,  ~ucMaxPriorityValue = 0x1F
configMAX = 0x60 = 0b01100000
0x60 & 0x1F = 0x00 → 断言通过 ✓
```

### 修改 2: 重新启用 configASSERT

```c
// 修改前 (临时绕过)
#define configASSERT( x )    ((void)(x))

// 修改后 (正常断言)
#define configASSERT( x )                       \
    if( ( x ) == 0 ) {                          \
        __disable_irq();                        \
        for( ; ; );                             \
    }
```

注意: 使用 `__disable_irq()` 而非 `taskDISABLE_INTERRUPTS()`，因为在调度器启动前后者引用未初始化的内核状态。

### 修改 3: USE_RTOS 不能改为 1

`py32f403_hal_conf.h:158` 中的 `USE_RTOS` 保持为 `0`。

```c
// py32f403_hal_def.h:102-104
#if (USE_RTOS == 1U)
    /* Reserved for future use */
    #error "USE_RTOS should be 0 in the current HAL release"
```

PY32F403 当前 HAL 库的 CMSIS-RTOS 互斥锁代码路径为**空实现**，PUYA 用 `#error` 阻断编译。强行改为 1 会导致所有 HAL 源文件编译失败。

对当前项目无影响: 单任务无外设并发访问；SysTick 最终由 FreeRTOS `vPortSetupTimerInterrupt()` 重配置覆盖；`HAL_Delay()` 未被使用。

### 修改 4: 清理 ARMv8-M 配置噪音

FreeRTOSConfig.h 中的以下宏仅被 ARMv8-M 端口使用，对 PY32F403 (ARMv7-M Cortex-M4F) 无任何作用，从 1 改为 0 消除误导:

```
configENABLE_TRUSTZONE      1 → 0
configRUN_FREERTOS_SECURE_ONLY  1 → 0
configENABLE_MPU            1 → 0
configENABLE_FPU            1 → 0
configENABLE_MVE            1 → 0
```

---

# 最终有效配置总结

截至 Session 3 结束，以下配置确认可以在 PY32F403-STK 上正常运行 FreeRTOS V11.3.0（configASSERT 已正常启用）：

### FreeRTOSConfig.h 关键配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `configCPU_CLOCK_HZ` | `8000000` | 必须与实际 HSI_VALUE 一致 |
| `configTICK_RATE_HZ` | `100` | 10ms 每 tick |
| `configMAX_PRIORITIES` | `5` | 0~4 级优先级 |
| `configCHECK_FOR_STACK_OVERFLOW` | `2` | 栈溢出检测启用 |
| `configTOTAL_HEAP_SIZE` | `4096` | 4KB FreeRTOS 堆 |
| `configPRIO_BITS` | `3` | PY32F403 数据手册确认: 3 位优先级 (8 等级) |
| `configLIBRARY_LOWEST_INTERRUPT_PRIORITY` | `7` | 3-bit 范围: 0~7 |
| `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` | `3` | FromISR() 安全边界 |
| `configKERNEL_INTERRUPT_PRIORITY` | `0xE0` | 7 << 5 |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | `0x60` | 3 << 5 |
| `#define vPortSVCHandler SVC_Handler` | — | **必须!** Direct Routing |
| `#define xPortPendSVHandler PendSV_Handler` | — | **必须!** Direct Routing |
| `configCHECK_HANDLER_INSTALLATION` | `1` | 中断安装验证 |
| `configASSERT( x )` | `__disable_irq(); for(;;)` | 已修复，正常启用 |

### py32f403_it.c 中断处理

| 处理器 | 来源 | 说明 |
|--------|------|------|
| `SVC_Handler` | port.c (经宏映射) | **注释掉**本地空函数 |
| `PendSV_Handler` | port.c (经宏映射) | **注释掉**本地空函数 |
| `SysTick_Handler` | py32f403_it.c | 组合 `HAL_IncTick()` + `xPortSysTickHandler()` |
| `extern xPortSysTickHandler` | py32f403_it.c | 显式 extern 声明消除警告 |

### main.c 关键点

- `#include "FreeRTOS.h"` 和 `#include "task.h"` 必须存在
- 必须实现 `vApplicationStackOverflowHook()`
- 使用 `pdMS_TO_TICKS(ms)` 宏计算延时，不要硬编码 tick 数
- `vTaskStartScheduler()` 后不要放死循环轮询代码（调度器启动成功后永不返回）

---

> **下一 Session:** 第2课 — 任务管理 (多任务创建、优先级抢占、挂起/恢复、栈监控)
> 
> **Session 3 备注:**
> - `USE_RTOS` 保持 `0`：PY32F403 HAL 库未实现 CMSIS-RTOS 互斥锁，强制 `#error` 阻断。单任务无影响。
> - ARMv8-M 专属宏 (TrustZone/MPU/FPU/MVE) 已从 1 清零，ARM_CM4F 端口不引用这些宏。
