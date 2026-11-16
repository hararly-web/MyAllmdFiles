# FreeRTOSConfig.h 完整配置参考 (V11.3.0)

> **适用:** Cortex-M4F + Keil MDK5 + V11.3.0
> **用途:** 以下完整模板可直接复制到项目 `Inc/FreeRTOSConfig.h` 使用。
> **配套:** `FreeRTOS_Guidance.md` — 完整的移植教程和学习路线图。

---

## 完整配置模板 (可直接复制)

```c
/*
 * FreeRTOS Kernel V11.3.0
 * Project-specific Configuration for Cortex-M4F + Keil MDK5
 */

#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

/*===========================================================================
 * [SECTION A] Hardware & Scheduling
 *===========================================================================*/

/* System core clock frequency. Must match actual configuration.
 * Py32F403: HSI=24MHz, HSE up to 32MHz, PLL up to 120MHz.
 * This example uses HSI at 24MHz.
 * IMPORTANT: This value configures SysTick reload value. */
#define configCPU_CLOCK_HZ                      ( ( unsigned long ) SystemCoreClock )

/* configSYSTICK_CLOCK_HZ — optional, only needed if SysTick clock != CPU clock.
 * Leave undefined to let the port use configCPU_CLOCK_HZ directly. */
/* #define configSYSTICK_CLOCK_HZ             ( ( unsigned long ) SystemCoreClock ) */

/* RTOS tick interrupt frequency (Hz).
 * 1000Hz = 1ms per tick. Most common value.
 * Effects: Higher = more responsive (but more CPU overhead).
 *          Lower = less overhead (but coarser timing). */
#define configTICK_RATE_HZ                      ( ( TickType_t ) 1000 )

/* 1 = Preemptive scheduling. Higher-priority tasks preempt lower ones.
 * 0 = Cooperative scheduling. Tasks run until they yield or block.
 * ALWAYS use 1 for real-time systems. */
#define configUSE_PREEMPTION                    1

/* 1 = Enable time-slicing between equal-priority tasks on each tick.
 * 0 = Equal-priority tasks run until they yield (FIFO).
 * V11.3.0 NEW: separated from configUSE_PREEMPTION. */
#define configUSE_TIME_SLICING                  1

/* 1 = Hardware-optimized priority lookup using __clz (count leading zeros).
 * 0 = Generic C loop.
 * Cortex-M4F supports CLZ → set to 1 for best performance.
 * Constraint: configMAX_PRIORITIES must be ≤ 32. */
#define configUSE_PORT_OPTIMISED_TASK_SELECTION 1

/* 1 = Enable tickless idle (low-power mode). Requires port-specific support.
 * 0 = SysTick runs continuously.
 * Leave at 0 for now; see Lesson 11 for low-power optimization. */
#define configUSE_TICKLESS_IDLE                 0

/* Number of task priority levels: 0 (lowest) to (configMAX_PRIORITIES-1) (highest).
 * Idle task uses priority 0. Application tasks use 1~7.
 * RAM cost per extra priority level: 4 bytes (ready list pointer).
 * Typical range: 5 (simple) ~ 32 (complex). */
#define configMAX_PRIORITIES                    8

/* Stack size for the Idle task (in words, not bytes).
 * On 32-bit Cortex-M: 128 words = 512 bytes.
 * This is the MINIMUM; your tasks can use more. */
#define configMINIMAL_STACK_SIZE                ( ( unsigned short ) 128 )

/* Maximum task name length (including '\0').
 * Used only for debugging. Keep it reasonable (16 is fine). */
#define configMAX_TASK_NAME_LEN                 ( 16 )

/* V11.3.0 REPLACES configUSE_16_BIT_TICKS:
 * TICK_TYPE_WIDTH_16_BITS — max delay ~65s @1kHz (tiny RAM)
 * TICK_TYPE_WIDTH_32_BITS — max delay ~49 days @1kHz (recommended)
 * TICK_TYPE_WIDTH_64_BITS — NOT supported by RVDS/ARM_CM4F port */
#define configTICK_TYPE_WIDTH_IN_BITS           TICK_TYPE_WIDTH_32_BITS

/* 1 = Idle task yields to same-priority application tasks.
 * 0 = Idle task uses its full time slice.
 * Set to 1 unless you have specific reasons not to. */
#define configIDLE_SHOULD_YIELD                 1

/* Number of per-task notification slots.
 * 1 is enough for most apps. Increase if using multiple notif types per task. */
#define configTASK_NOTIFICATION_ARRAY_ENTRIES   1

/* Queue registry size (for kernel-aware debuggers). 0 = disabled. */
#define configQUEUE_REGISTRY_SIZE               0

/* 1 = Map old API names to new ones (e.g., vTaskDelayUntil → xTaskDelayUntil).
 * Keep at 1 for backward compatibility with existing code/tutorials. */
#define configENABLE_BACKWARD_COMPATIBILITY     1

/* Thread-local storage pointers per task. 0 = disabled.
 * Used for reentrant C library support or per-task context storage. */
#define configNUM_THREAD_LOCAL_STORAGE_POINTERS 0

/* 1 = MiniListItem_t saves 3 fields (12 bytes) vs full ListItem_t.
 * Saves RAM at the cost of stricter compiler aliasing rules.
 * Normally safe; set to 1. */
#define configUSE_MINI_LIST_ITEM                1

/* Type used for task stack depth parameters in API calls.
 * size_t = platform-dependent (usually 32-bit on ARM). */
#define configSTACK_DEPTH_TYPE                  uint32_t

/* Type used to store message buffer message lengths.
 * size_t is fine for most cases. Use uint8_t to save space if messages are short. */
#define configMESSAGE_BUFFER_LENGTH_TYPE        size_t

/* 1 = Clear freed heap memory to zero (security + easier bug detection).
 * 0 = Don't clear (slightly faster free). */
#define configHEAP_CLEAR_MEMORY_ON_FREE         1

/* Buffer size for vTaskList/vTaskGetRunTimeStats legacy APIs.
 * New code should use vTaskListTasks/vTaskGetRunTimeStatistics instead. */
#define configSTATS_BUFFER_MAX_LENGTH           0xFFFF

/* 1 = Allocate a newlib reent structure per task (for thread-safe C lib).
 * 0 = No newlib support. Set to 0 unless using newlib printf/scanf in tasks. */
#define configUSE_NEWLIB_REENTRANT              0

/*===========================================================================
 * [SECTION B] Software Timer
 *===========================================================================*/

/* V11.3.0 NEW: Explicitly enable software timers.
 * When enabled, timers.c must be added to the project.
 * A "Timer Service Daemon Task" is created automatically. */
#define configUSE_TIMERS                        1

/* Priority of the Timer Service task. configMAX_PRIORITIES-1 = highest.
 * Keep it high if timer callbacks are time-critical. */
#define configTIMER_TASK_PRIORITY               ( configMAX_PRIORITIES - 1 )

/* Stack depth of the Timer Service task (words). */
#define configTIMER_TASK_STACK_DEPTH            128

/* Length of the command queue used by the Timer Service task.
 * Each Start/Stop/Reset call sends one command to this queue. */
#define configTIMER_QUEUE_LENGTH                10

/*===========================================================================
 * [SECTION C] Event Groups
 *===========================================================================*/

/* V11.3.0 NEW: Explicitly enable event groups.
 * When enabled, event_groups.c must be added to the project. */
#define configUSE_EVENT_GROUPS                  1

/*===========================================================================
 * [SECTION D] Stream Buffers
 *===========================================================================*/

/* V11.3.0 NEW: Explicitly enable stream buffers.
 * When enabled, stream_buffer.c must be added to the project.
 * Stream buffers are the foundation for message buffers and FreeRTOS+TCP. */
#define configUSE_STREAM_BUFFERS                1

/*===========================================================================
 * [SECTION E] Memory Allocation
 *===========================================================================*/

/* 1 = Enable xTaskCreateStatic(), xQueueCreateStatic(), etc.
 * Static allocation = you provide the memory, no heap needed. */
#define configSUPPORT_STATIC_ALLOCATION         1

/* 1 = Enable xTaskCreate(), xQueueCreate(), etc. (dynamic, uses heap).
 * Dynamic allocation = FreeRTOS allocates memory from its heap. */
#define configSUPPORT_DYNAMIC_ALLOCATION        1

/* Total FreeRTOS heap size (bytes). Used by heap_1, heap_2, heap_4.
 * 15KB is enough for ~5-8 tasks with moderate stack + a few queues.
 * Monitor with xPortGetFreeHeapSize().
 * Tune this value based on your application! */
#define configTOTAL_HEAP_SIZE                   ( ( size_t ) ( 15 * 1024 ) )

/* 1 = Application provides the heap array (ucHeap[]).
 * 0 = Linker allocates ucHeap[] in .bss. */
#define configAPPLICATION_ALLOCATED_HEAP        0

/* 1 = Task stacks allocated from a separate heap (e.g., CCM RAM for speed).
 * 0 = Stacks come from the standard FreeRTOS heap.
 * If 1, you must implement pvPortMallocStack() and vPortFreeStack(). */
#define configSTACK_ALLOCATION_FROM_SEPARATE_HEAP 0

/* 1 = Enable pointer obfuscation and bounds checking in heap_4/heap_5.
 * Helps detect heap corruption. Slight performance cost. */
#define configENABLE_HEAP_PROTECTOR             0

/*===========================================================================
 * [SECTION F] Cortex-M4F Interrupt Priority (CRITICAL — DO NOT GUESS!)
 *===========================================================================*/

/* Number of implemented priority bits in the NVIC.
 * PY32F403: 4 bits → 16 priority levels (0=highest, 15=lowest). */
#define configPRIO_BITS                         4

/* Library-format (0-15) priority values: */

/* Lowest possible interrupt priority = 15 (lowest urgency).
 * SysTick and PendSV run at this level. */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY             15

/* Highest priority that CAN safely call FreeRTOS FromISR() APIs.
 * Interrupts with priority 0-4 (lower number = higher urgency)
 * CANNOT call any FreeRTOS API — they would corrupt kernel state. */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY        5

/* Convert library values (0-15) to NVIC register values (8-bit, upper bits).
 * Formula: NVIC_value = library_value << (8 - configPRIO_BITS)
 * DO NOT CHANGE THIS FORMULA.
 *
 * 15 << 4 = 0xF0 — kernel interrupt (SysTick/PendSV) at lowest priority
 *  5 << 4 = 0x50 — highest FromISR-safe priority boundary */
#define configKERNEL_INTERRUPT_PRIORITY         ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )

/* DO NOT define configMAX_API_CALL_INTERRUPT_PRIORITY.
 * V11.3.0 FreeRTOS.h derives it from configMAX_SYSCALL_INTERRUPT_PRIORITY.
 * Defining it to a different value causes hard-to-debug priority issues. */

/*===========================================================================
 * [SECTION G] Interrupt Handler Mapping
 *===========================================================================*/

/* Map FreeRTOS port.c handlers to CMSIS-standard interrupt vector names.
 *
 * vPortSVCHandler()    → SVC exception (starts the first task, called ONCE)
 * xPortPendSVHandler() → PendSV exception (all context switches)
 *
 * These are STRONG definitions in port.c — they override the WEAK
 * definitions in py32f403_it.c / startup file. */
#define vPortSVCHandler                         SVC_Handler
#define xPortPendSVHandler                      PendSV_Handler

/* SysTick: intentionally NOT mapped here.
 * We call xPortSysTickHandler() manually from HAL's SysTick_Handler,
 * so both HAL_IncTick() and FreeRTOS tick processing occur in one ISR. */
/* #define xPortSysTickHandler                 SysTick_Handler */

/* 1 = Validate that SVC/PendSV handlers are correctly installed.
 * V11.3.0 NEW: adds compile-time and run-time checks. */
#define configCHECK_HANDLER_INSTALLATION        1

/*===========================================================================
 * [SECTION H] Hook Functions
 *===========================================================================*/

/* 1 = Enable the named hook. You MUST provide the corresponding function.
 * 0 = Disabled. Saves a few bytes. */

#define configUSE_IDLE_HOOK                     0   /* vApplicationIdleHook()        */
#define configUSE_TICK_HOOK                     0   /* vApplicationTickHook()        */
#define configUSE_MALLOC_FAILED_HOOK            0   /* vApplicationMallocFailedHook()*/
#define configUSE_DAEMON_TASK_STARTUP_HOOK      0   /* vApplicationDaemonTask...Hook()*/
#define configUSE_SB_COMPLETED_CALLBACK         0   /* Stream buffer callbacks       */

/* Stack overflow detection level:
 * 0 = disabled
 * 1 = Method 1: Check stack pointer is within bounds (fast but less effective)
 * 2 = Method 2: Check "canary" pattern at stack bottom (slower but catches more)
 * If > 0, you MUST provide vApplicationStackOverflowHook() in your code. */
#define configCHECK_FOR_STACK_OVERFLOW          2

/*===========================================================================
 * [SECTION I] Run-Time Statistics (optional)
 *===========================================================================*/

/* 1 = Collect per-task CPU time data. Requires a high-resolution timer.
 * 0 = Disabled. */
#define configGENERATE_RUN_TIME_STATS           0

/* If configGENERATE_RUN_TIME_STATS=1, provide macros for the timer: */
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()
#define portGET_RUN_TIME_COUNTER_VALUE()        0

/* 1 = Add extra fields to TCB for trace/debug tools (task state, etc.).
 * Enable for development; disable for production if RAM is tight. */
#define configUSE_TRACE_FACILITY                1

/* 1 = Include legacy vTaskList() and vTaskGetRunTimeStats().
 * These pull in sprintf-like formatting code. */
#define configUSE_STATS_FORMATTING_FUNCTIONS    0

/*===========================================================================
 * [SECTION J] Feature Inclusion Switches
 *===========================================================================*/

#define configUSE_TASK_NOTIFICATIONS            1   /* Task notification API          */
#define configUSE_MUTEXES                       1   /* Mutex (with priority inherit)  */
#define configUSE_RECURSIVE_MUTEXES             1   /* Recursive mutex               */
#define configUSE_COUNTING_SEMAPHORES           1   /* Counting semaphore            */
#define configUSE_QUEUE_SETS                    0   /* Queue sets (rarely used)      */
#define configUSE_APPLICATION_TASK_TAG          0   /* Per-task tag (debugging)      */
#define configUSE_POSIX_ERRNO                   0   /* Per-task errno variable       */

/*===========================================================================
 * [SECTION K] API Function Inclusion
 *
 * Set to 1 to include, 0 to exclude from build.
 * Most linkers will garbage-collect unused functions, but these macros
 * can save compilation time and code size.
 *===========================================================================*/

#define INCLUDE_vTaskPrioritySet                1   /* vTaskPrioritySet()            */
#define INCLUDE_uxTaskPriorityGet               1   /* uxTaskPriorityGet()           */
#define INCLUDE_vTaskDelete                     1   /* vTaskDelete()                 */
#define INCLUDE_vTaskSuspend                    1   /* vTaskSuspend() / Resume()     */
#define INCLUDE_xTaskDelayUntil                 1   /* xTaskDelayUntil()             */
#define INCLUDE_vTaskDelay                      1   /* vTaskDelay()                  */
#define INCLUDE_xTaskGetSchedulerState          1   /* xTaskGetSchedulerState()      */
#define INCLUDE_xTaskGetCurrentTaskHandle       1   /* xTaskGetCurrentTaskHandle()   */
#define INCLUDE_uxTaskGetStackHighWaterMark     1   /* uxTaskGetStackHighWaterMark() */
#define INCLUDE_xTaskGetIdleTaskHandle          1   /* xTaskGetIdleTaskHandle()      */
#define INCLUDE_eTaskGetState                   1   /* eTaskGetState()               */
#define INCLUDE_xTimerPendFunctionCall          1   /* xTimerPendFunctionCall()      */
#define INCLUDE_xTaskAbortDelay                 1   /* xTaskAbortDelay()             */
#define INCLUDE_xTaskGetHandle                  1   /* xTaskGetHandle()              */
#define INCLUDE_xTaskResumeFromISR              1   /* xTaskResumeFromISR()          */
#define INCLUDE_xQueueGetMutexHolder            1   /* xQueueGetMutexHolder()        */
#define INCLUDE_xEventGroupSetBitFromISR        1   /* xEventGroupSetBitFromISR()    */

/*===========================================================================
 * [SECTION L] Assert
 *===========================================================================*/

/* Runtime assertion for catching API misuse during development.
 * Disable (comment out) for production builds to save code space. */
#define configASSERT( x )                       \
    if( ( x ) == 0 ) {                          \
        taskDISABLE_INTERRUPTS();               \
        for( ; ; );                             \
    }

/*===========================================================================
 * [SECTION M] MPU-Specific (only for MPU-enabled ports, safe defaults)
 *===========================================================================*/

#define configINCLUDE_APPLICATION_DEFINED_PRIVILEGED_FUNCTIONS 0
#define configTOTAL_MPU_REGIONS                 8
#define configTEX_S_C_B_FLASH                   0x07UL
#define configTEX_S_C_B_SRAM                    0x07UL
#define configENFORCE_SYSTEM_CALLS_FROM_KERNEL_ONLY 0
#define configALLOW_UNPRIVILEGED_CRITICAL_SECTIONS   0

#endif /* FREERTOS_CONFIG_H */
```

---

## 配置项速查表

### 必须按项目调整的配置

| 宏 | 默认值 | 说明 |
|----|--------|------|
| `configCPU_CLOCK_HZ` | `SystemCoreClock` | 必须匹配实际系统时钟 |
| `configTICK_RATE_HZ` | `1000` | 常用值: 100, 500, 1000, 10000 |
| `configTOTAL_HEAP_SIZE` | `15 * 1024` | 根据任务数+队列数调整 |
| `configMAX_PRIORITIES` | `8` | 每个优先级占 4 字节 RAM |
| `configMINIMAL_STACK_SIZE` | `128` | Idle 任务栈, 单位 words |
| `configPRIO_BITS` | `4` | 查阅 MCU 数据手册 NVIC 章节 |

### 根据功能需求启用的配置

| 功能 | 需要的宏 | 需要的 .c 文件 |
|------|---------|---------------|
| 基本多任务 | (默认已启用) | tasks.c + list.c |
| 队列 | (默认已启用) | queue.c |
| 互斥锁 | `configUSE_MUTEXES 1` | (不需要额外文件) |
| 递归互斥锁 | `configUSE_RECURSIVE_MUTEXES 1` | (不需要额外文件) |
| 计数信号量 | `configUSE_COUNTING_SEMAPHORES 1` | (不需要额外文件) |
| 软件定时器 | `configUSE_TIMERS 1` | timers.c |
| 事件组 | `configUSE_EVENT_GROUPS 1` | event_groups.c |
| 流缓冲区 | `configUSE_STREAM_BUFFERS 1` | stream_buffer.c |
| 任务通知 | `configUSE_TASK_NOTIFICATIONS 1` | (不需要额外文件) |
| 静态分配 | `configSUPPORT_STATIC_ALLOCATION 1` | (不需要额外文件) |
| 栈溢出检测 | `configCHECK_FOR_STACK_OVERFLOW 2` | 需实现钩子函数 |
| 运行时统计 | `configGENERATE_RUN_TIME_STATS 1` | 需提供硬件定时器 |
| Tickless 低功耗 | `configUSE_TICKLESS_IDLE 1` | 需适配 MCU 休眠 |

### 生产环境优化建议

| 配置 | 开发阶段 | 生产阶段 | 节省 |
|------|---------|---------|------|
| `configUSE_TRACE_FACILITY` | `1` | `0` | ~40 bytes/Task |
| `configCHECK_FOR_STACK_OVERFLOW` | `2` | `0` | ~500 bytes 代码 |
| `configASSERT()` | 启用 | 注释掉 | ~200 bytes 代码 |
| `configUSE_STATS_FORMATTING_FUNCTIONS` | `1` | `0` | ~2KB 代码 (含 sprintf) |

---

> **配套文件:** `FreeRTOS_Guidance.md` — 完整学习路线图 + 第1课移植教程
