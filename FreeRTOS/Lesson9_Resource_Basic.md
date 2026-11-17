# 第9课 代码实战 — 基础版：资源管理 (调度器锁 + 死锁检测 + 优先级反转)

> **目标：** 掌握 vTaskSuspendAll/xTaskResumeAll、死锁成因与超时检测、优先级反转及 Mutex 优先级继承解决方案
> **硬件：** PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
> **前置：** 第8课中断管理已完成，理解 taskENTER/EXIT_CRITICAL 和 Mutex 基础
> **Heap：** `configTOTAL_HEAP_SIZE` = 8192（已配好）

---

## 核心概念：资源保护三层次

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    共享资源保护的三种粒度                                   │
│                                                                          │
│  层次1: taskENTER_CRITICAL / taskEXIT_CRITICAL                            │
│    机制: 关所有可屏蔽中断 (PRIMASK=1)                                     │
│    开销: 最低 (~3 cycles)                                                │
│    中断延迟代价: 完全屏蔽，中断无法响应                                   │
│    适用: < 10 μs 的原子操作                                              │
│                                                                          │
│  层次2: vTaskSuspendAll / xTaskResumeAll (本课!)                          │
│    机制: 停调度器，但不关中断                                             │
│    开销: 低 (uxSchedulerSuspended++)                                     │
│    中断延迟代价: 无! 中断照常响应                                        │
│    适用: 较长的操作 (10~100 μs)，需要原子性但不想丢中断                   │
│    ⚠️ 限制: 期间不能调用任何可能导致阻塞/切换的 API                       │
│                                                                          │
│  层次3: Mutex (互斥锁)                                                    │
│    机制: 任务间互斥锁，支持优先级继承                                     │
│    开销: 中 (需要内核对象 + 临界区)                                       │
│    中断延迟代价: 无!                                                     │
│    适用: 长时间操作 / 需要调用阻塞 API / 多任务竞争                       │
│    ✅ 唯一支持优先级继承的机制                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 程序架构

```
 Demo 1: 调度器锁 vs 临界区
   SchedulerLock_Task  — 用调度器锁保护多变量原子更新
   对比: 调度器锁允许 ISR 响应，临界区屏蔽 ISR
   LED0: 每轮操作后翻转

 Demo 2: 死锁检测
   DeadlockTask_A(prio=2)  — 先锁 MutexA → delay → 锁 MutexB
   DeadlockTask_B(prio=2)  — 先锁 MutexB → delay → 锁 MutexA
   结果: 互相等待对方释放 → 死锁
   检测: xSemaphoreTake 带超时 → 超时 = 死锁
   LED1: 死锁时亮起

 Demo 3: 优先级反转 + 优先级继承
   Task_High(prio=4) — 需要共享资源，阻塞等待
   Task_Med(prio=3)  — 纯 CPU 消耗，不涉及共享资源
   Task_Low(prio=2)  — 持有共享资源，但被 Task_Med 抢占
   阶段1 (Binary Semaphore): 优先级反转 → Task_High 永久饥饿
   阶段2 (Mutex):          优先级继承 → Task_Low 被提升，快速释放
   LED2: 阶段1慢闪(反转) / 阶段2快闪(继承解决)
```

---

## 流程图 (中文)

```
┌──────────────────────────────────────────────────────────────────────────┐
│              第9课 资源管理 — 程序流程图                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  main()                                                                  │
│    ├── HAL_Init / Clock / GPIO (3 LED 输出)                              │
│    │                                                                     │
│    ├── xSemaphoreCreateMutex() × 2  ← Demo2 死锁演示用                   │
│    ├── xSemaphoreCreateBinary()     ← Demo3 阶段1 (无优先级继承)         │
│    ├── xSemaphoreCreateMutex()      ← Demo3 阶段2 (有优先级继承)         │
│    │                                                                     │
│    ├── xTaskCreate(SchedulerLock, prio=2)  ← Demo1                      │
│    ├── xTaskCreate(Deadlock_A,   prio=2)  ← Demo2                      │
│    ├── xTaskCreate(Deadlock_B,   prio=2)  ← Demo2                      │
│    ├── xTaskCreate(Inversion_High, prio=4) ← Demo3 最高优先级            │
│    ├── xTaskCreate(Inversion_Med,  prio=3) ← Demo3 纯 CPU 消耗           │
│    ├── xTaskCreate(Inversion_Low,  prio=2) ← Demo3 持有资源              │
│    │                                                                     │
│    └── vTaskStartScheduler()                                            │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Demo 1: 调度器锁 (Scheduler Lock)                                  │   │
│  │                                                                    │   │
│  │   SchedulerLock_Task:                                              │   │
│  │   ┌──────────────────────────────────────────────────────────────┐│   │
│  │   │ 每2秒循环一次:                                                ││   │
│  │   │                                                              ││   │
│  │   │ ① vTaskSuspendAll()           ← 调度器停止, 中断照常!        ││   │
│  │   │ ② g.sensor.temp  = read_adc()  ← 原子更新多个变量              ││   │
│  │   │ ③ g.sensor.humid = read_adc()    (不会被其他任务打断)          ││   │
│  │   │ ④ g.sensor.valid  = true                                       ││   │
│  │   │ ⑤ xTaskResumeAll()            ← 调度器恢复                    ││   │
│  │   │                                                              ││   │
│  │   │   对比: 如果用 taskENTER_CRITICAL                             ││   │
│  │   │     → read_adc() 期间中断被屏蔽 → UART/SPI 等中断丢失!        ││   │
│  │   │   用调度器锁 → read_adc() 期间中断仍可响应 ✓                  ││   │
│  │   │                                                              ││   │
│  │   │ ⑥ LED0 翻转                                                   ││   │
│  │   │ ⑦ vTaskDelay(2000)             ← 等待下一轮                    ││   │
│  │   └──────────────────────────────────────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Demo 2: 死锁 (Deadlock)                                            │   │
│  │                                                                    │   │
│  │   DeadlockTask_A (prio=2)              DeadlockTask_B (prio=2)    │   │
│  │   ┌───────────────────────────┐        ┌──────────────────────┐   │   │
│  │   │ ① Take MutexA ✅         │        │ ① Take MutexB ✅     │   │   │
│  │   │ ② vTaskDelay(50ms)       │        │ ② vTaskDelay(50ms)  │   │   │
│  │   │ ③ Take MutexB ⏳等待...  │        │ ③ Take MutexA ⏳等待...│  │   │
│  │   │    └→ 超时 500ms → 死锁! │        │    └→ 超时 500ms → 死锁!│  │   │
│  │   │ ④ LED1 ON (死锁检测)     │        │ ④ LED1 ON (死锁检测) │   │   │
│  │   │ ⑤ 释放所有持有的锁       │        │ ⑤ 释放所有持有的锁   │   │   │
│  │   │ ⑥ vTaskDelay(2000)       │        │ ⑥ vTaskDelay(2000)  │   │   │
│  │   └───────────────────────────┘        └──────────────────────┘   │   │
│  │                                                                    │   │
│  │   死锁四条件 (全部满足才死锁):                                      │   │
│  │   ① 互斥: 资源一次只能被一个任务使用 ✓                             │   │
│  │   ② 持有等待: 持有资源同时等待其他资源 ✓                           │   │
│  │   ③ 非抢占: 资源不能强制剥夺 ✓ (Mutex 不能强制夺走)               │   │
│  │   ④ 循环等待: A 等 B, B 等 A → 环形依赖 ✓                         │   │
│  │                                                                    │   │
│  │  预防死锁: 破坏任一条件即可 → 最常用"统一锁顺序" (破坏条件④)     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Demo 3: 优先级反转 + 优先级继承 (The Mars Pathfinder Bug)          │   │
│  │                                                                    │   │
│  │   阶段1 (Binary Semaphore — 无优先级继承):                         │   │
│  │                                                                    │   │
│  │   Task_Low(prio=2):  Take BinarySem ✅ (拿到"锁")                 │   │
│  │                       vTaskDelay(100ms) — 模拟工作中               │   │
│  │                                                                   │   │
│  │   Task_High(prio=4): 抢占 Low                                      │   │
│  │                       Take BinarySem ⏳ 阻塞等待Low释放!           │   │
│  │                                                                   │   │
│  │   Task_Med(prio=3):  抢走 CPU (比 Low 高，比 High 低)              │   │
│  │                       for(;;) 死循环 — 霸占 CPU                   │   │
│  │                                                                   │   │
│  │   结果: Task_Low 拿不到 CPU 释放锁                                  │   │
│  │         Task_High 被 Task_Med 无限阻塞!  💀                        │   │
│  │         → 优先级反转: 高优任务被中优任务间接阻塞                    │   │
│  │         LED2 慢闪表示反转发生                                       │   │
│  │                                                                   │   │
│  │   阶段2 (Mutex — 有优先级继承):                                    │   │
│  │                                                                    │   │
│  │   Task_Low(prio=2):   Take Mutex ✅                                │   │
│  │                       内核自动将 Low 优先级提升到 4 (继承High)      │   │
│  │                       vTaskDelay — 工作 (不会被Med打断!)           │   │
│  │                                                                   │   │
│  │   Task_High(prio=4):  抢占, Take Mutex ⏳ 阻塞                     │   │
│  │                                                                   │   │
│  │   结果: Low 优先级 = 4 → Task_Med(prio=3) 无法抢占!                │   │
│  │         Low 完成工作 → Give Mutex → Low 恢复 prio=2               │   │
│  │         High 拿到 Mutex → 继续执行!  ✅                             │   │
│  │         LED2 快闪表示继承解决                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 完整 main.c

```c
/**
  ******************************************************************************
  * @file    main.c
  * @brief   FreeRTOS Lesson 9 Basic: Resource Management
  *
  *          Demonstrates:
  *          - Scheduler Lock: vTaskSuspendAll/xTaskResumeAll vs Critical Section
  *          - Deadlock: cause → detection via timeout → proper prevention
  *          - Priority Inversion + Priority Inheritance (Mars Pathfinder bug)
  *
  *          Architecture:
  *
  *            [Demo 1: Scheduler Lock]
  *              SchedulerLock_Task updates 3 shared variables atomically
  *              using vTaskSuspendAll(). Unlike critical sections,
  *              interrupts remain enabled during the operation.
  *
  *            [Demo 2: Deadlock]
  *              DeadlockTask_A holds MutexA, waits for MutexB.
  *              DeadlockTask_B holds MutexB, waits for MutexA.
  *              → Classic circular wait deadlock.
  *              Detected via xSemaphoreTake() timeout (500ms).
  *
  *            [Demo 3: Priority Inversion]
  *              Phase 1 (Binary Semaphore): Low prio holds lock,
  *                Medium prio steals CPU, High prio starves.
  *              Phase 2 (Mutex): Priority inheritance elevates Low
  *                past Medium → High gets through promptly.
  *
  *          LED assignment:
  *            LED0 (PB13): Demo1 SchedulerLock — toggles each cycle
  *            LED1 (PA1):  Demo2 Deadlock — lights on deadlock detect
  *            LED2 (PB12): Demo3 Inversion — slow blink=inversion, fast blink=resolved
  *
  *          Priority map for Demo 3:
  *            Task_High  = 4 (needs shared resource urgently)
  *            Task_Med   = 3 (pure CPU hog, no resource needed)
  *            Task_Low   = 2 (holds the shared resource)
  *
  *          configMAX_PRIORITIES = 5 → priorities 0~4 available.
  ******************************************************************************
  */

/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"          /* Binary Semaphore + Mutex */

/* Private defines ------------------------------------------------------------*/

/* LED pin definitions */
#define LED0_PORT           GPIOB
#define LED0_PIN            GPIO_PIN_13
#define LED1_PORT           GPIOA
#define LED1_PIN            GPIO_PIN_1
#define LED2_PORT           GPIOB
#define LED2_PIN            GPIO_PIN_12

/*
 * Deadlock detection timeout (ms).
 * Each deadlock task waits this long for the second mutex.
 * If it times out, deadlock is diagnosed.
 */
#define DEADLOCK_TIMEOUT_MS         500U
#define DEADLOCK_CYCLE_DELAY_MS     2000U

/*
 * Priority inversion test timing (ms).
 * Phase 1: Binary Semaphore → inversion visible
 * Phase 2: Mutex → priority inheritance resolves it
 * Each phase lasts several seconds for visual observation.
 */
#define INVERSION_PHASE_DURATION_MS  4000U    /* observation window per phase      */

/*
 * Critical shared data: sensor readings updated under scheduler lock.
 * Multiple variables must be updated together (atomic unit).
 */
typedef struct {
    uint16_t temperature;        /* Simulated temperature (ADC reading)      */
    uint16_t humidity;           /* Simulated humidity (ADC reading)         */
    uint8_t  valid;              /* 1 = data pair is consistent              */
} SensorData_t;

/* Task priorities */
#define PRIO_SCHED_LOCK     2U
#define PRIO_DEADLOCK_A     2U
#define PRIO_DEADLOCK_B     2U
#define PRIO_INV_HIGH       4U            /* Highest app priority; needs resource   */
#define PRIO_INV_MED        3U            /* Mid priority; CPU-intensive, no lock    */
#define PRIO_INV_LOW        2U            /* Lowest; holds the resource              */

/* Task stack sizes (words) */
#define STACK_SCHED_LOCK    200U
#define STACK_DEADLOCK      200U
#define STACK_INV_HIGH      150U
#define STACK_INV_MED       150U
#define STACK_INV_LOW        300U          /* Larger: contains work simulation       */

/* Private variables ---------------------------------------------------------*/

/**
  * @brief  Application globals — all shared state in one struct.
  *
  *          Note: fields accessed by multiple tasks MUST use proper
  *          synchronization (mutex, scheduler lock, or critical section).
  */
static struct {
    /* Demo 1: shared data updated under scheduler lock */
    SensorData_t   sensor;                /* Protected by scheduler lock       */

    /* Demo 2: mutex handles for deadlock demo */
    SemaphoreHandle_t xMutexA;            /* Mutex for resource A              */
    SemaphoreHandle_t xMutexB;            /* Mutex for resource B              */
    volatile uint32_t ulDeadlockCount;    /* Deadlock detection counter        */

    /* Demo 3: semaphore/mutex for priority inversion demo */
    SemaphoreHandle_t xBinarySem;         /* Binary sem (no inheritance!)      */
    SemaphoreHandle_t xMutexInv;          /* Mutex (with priority inheritance)  */
    volatile uint32_t ulInversionPhase;   /* 0=BinarySem test, 1=Mutex test   */
    volatile uint32_t ulHighBlockedCnt;   /* Times High task was blocked       */
} g;

/* Private function prototypes -----------------------------------------------*/
static void APP_SystemClockConfig(void);
static void APP_GpioConfig(void);
static void APP_LedToggle(uint8_t led_id);
static void APP_LedOn(uint8_t led_id);
static void APP_LedOff(uint8_t led_id);

/* --- Demo 1: Scheduler Lock ----------------------------------------------- */
static void SchedulerLock_Task(void *pvParameters);

/* --- Demo 2: Deadlock ----------------------------------------------------- */
static void DeadlockTask_A(void *pvParameters);
static void DeadlockTask_B(void *pvParameters);

/* --- Demo 3: Priority Inversion ------------------------------------------- */
static void InversionTask_High(void *pvParameters);
static void InversionTask_Med(void *pvParameters);
static void InversionTask_Low(void *pvParameters);

/* ========================================================================== */
/*  Main                                                                      */
/* ========================================================================== */

int main(void)
{
    BaseType_t xReturned;

    HAL_Init();
    APP_SystemClockConfig();
    APP_GpioConfig();

    /* Initialize global shared data */
    g.sensor.temperature = 0;
    g.sensor.humidity    = 0;
    g.sensor.valid       = 0;
    g.ulDeadlockCount    = 0;
    g.ulInversionPhase   = 0;
    g.ulHighBlockedCnt   = 0;

    /*
     * Demo 2: Create two Mutexes for deadlock demonstration.
     * Both tasks will take them in OPPOSITE order → deadlock.
     */
    g.xMutexA = xSemaphoreCreateMutex();
    configASSERT(g.xMutexA != NULL);
    g.xMutexB = xSemaphoreCreateMutex();
    configASSERT(g.xMutexB != NULL);

    /*
     * Demo 3: Binary Semaphore (no priority inheritance) + Mutex (with it).
     *
     * Binary Semaphore: created with xSemaphoreCreateBinary().
     * Although semaphores CAN be used for mutual exclusion, they lack
     * priority inheritance — the key feature that distinguishes Mutex.
     *
     * The semaphore is given first so the Low task can take it.
     */
    g.xBinarySem = xSemaphoreCreateBinary();
    configASSERT(g.xBinarySem != NULL);
    xSemaphoreGive(g.xBinarySem);      /* Initialize to "available"    */

    g.xMutexInv = xSemaphoreCreateMutex();
    configASSERT(g.xMutexInv != NULL);

    /* Demo 1: Scheduler lock task */
    xReturned = xTaskCreate(SchedulerLock_Task, "SchedLock",
                            STACK_SCHED_LOCK, NULL, PRIO_SCHED_LOCK, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 2: Two deadlock-prone tasks */
    xReturned = xTaskCreate(DeadlockTask_A, "DeadA",
                            STACK_DEADLOCK, NULL, PRIO_DEADLOCK_A, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(DeadlockTask_B, "DeadB",
                            STACK_DEADLOCK, NULL, PRIO_DEADLOCK_B, NULL);
    configASSERT(xReturned == pdPASS);

    /* Demo 3: Priority inversion tasks (3 levels) */
    xReturned = xTaskCreate(InversionTask_High, "InvHigh",
                            STACK_INV_HIGH, NULL, PRIO_INV_HIGH, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(InversionTask_Med, "InvMed",
                            STACK_INV_MED, NULL, PRIO_INV_MED, NULL);
    configASSERT(xReturned == pdPASS);

    xReturned = xTaskCreate(InversionTask_Low, "InvLow",
                            STACK_INV_LOW, NULL, PRIO_INV_LOW, NULL);
    configASSERT(xReturned == pdPASS);

    vTaskStartScheduler();
    while (1) {}
}

/* ========================================================================== */
/*  Demo 1: Scheduler Lock — Atomic Multi-Variable Update                       */
/* ========================================================================== */

/**
  * @brief  Demonstrates vTaskSuspendAll/xTaskResumeAll for atomicity.
  *
  * @note   KEY CONCEPT: Scheduler Lock vs Critical Section
  *
  *          taskENTER_CRITICAL():
  *            - Disables ALL maskable interrupts (sets PRIMASK).
  *            - Even high-priority ISRs (UART, ADC, SPI) are blocked.
  *            - Use ONLY for sub-microsecond operations.
  *
  *          vTaskSuspendAll():
  *            - Stops the SCHEDULER from switching tasks.
  *            - Interrupts remain FULLY ENABLED.
  *            - ISRs can fire and even call FromISR APIs (data goes to queue).
  *            - BUT: the task context won't change until xTaskResumeAll().
  *            - Use for longer operations (10~100 μs) where you need atomicity
  *              but can't afford to miss interrupts.
  *
  *          RESTRICTIONS inside scheduler-locked region:
  *            ❌ No vTaskDelay() — requires scheduler to run
  *            ❌ No blocking API calls (xQueueReceive with timeout, etc.)
  *            ❌ No API that might yield (taskYIELD, vTaskSuspend)
  *            ✅ Read peripherals, do computation, write shared variables
  *            ✅ Read/write memory, call non-blocking functions
  *
  *          This demo simulates reading multiple ADC channels — if you used
  *          critical sections, you'd block all interrupts during the ADC read.
  *          With scheduler lock, only task switching is prevented.
  */
static void SchedulerLock_Task(void *pvParameters)
{
    (void)pvParameters;

    TickType_t xLastWakeTime = xTaskGetTickCount();
    uint16_t dummy_adc = 0;

    for (;;)
    {
        /*
         * START of atomic region.
         * No other task can run between SuspendAll and ResumeAll.
         * But interrupts still work — an ISR can fire, queue data,
         * and return without disturbing our atomicity.
         */
        vTaskSuspendAll();
        {
            /*
             * Simulate reading multiple ADC channels.
             * In a real app, these might be slow SPI/I2C reads.
             * With critical sections these would block ISRs;
             * with scheduler lock they don't.
             */
            dummy_adc++;
            g.sensor.temperature = 250U + (dummy_adc & 0x3F);   /* 25.0°C + drift  */
            g.sensor.humidity    = 600U + (dummy_adc & 0x1F);   /* 60.0% + drift   */

            /*
             * Mark data as valid LAST.
             * If a task reads valid=1, it can trust both temperature
             * and humidity are from the same ADC cycle.
             * This is the "atomicity boundary" — all 3 writes happen
             * without any other task seeing an intermediate state.
             */
            g.sensor.valid = 1;
        }
        xTaskResumeAll();
        /* END of atomic region */

        /*
         * After releasing the scheduler lock:
         * - If a higher-priority task was woken by an ISR during the lock,
         *   xTaskResumeAll() will trigger a context switch here.
         * - This is another advantage over critical sections — pending
         *   task switches are not lost, just deferred.
         */

        /* Visual feedback: LED0 toggles to show the demo is running */
        APP_LedToggle(0);

        /* Invalidate data between cycles (simulates consumption) */
        g.sensor.valid = 0;

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(2000));
    }
}


/* ========================================================================== */
/*  Demo 2: Deadlock — Detection via Timeout                                    */
/* ========================================================================== */

/**
  * @brief  DeadlockTask_A: Takes MutexA first, then MutexB.
  *
  * @note   DEADLOCK SETUP:
  *          Task_A: Take A → delay → Take B → release both
  *          Task_B: Take B → delay → Take A → release both
  *
  *          The delay between takes is CRITICAL — it creates the window
  *          where each task holds one resource and is waiting for the other.
  *
  *          Without the delay, the faster task might take both before
  *          the other starts, masking the deadlock.
  *
  *          Deadlock requires ALL FOUR conditions simultaneously:
  *          1. Mutual exclusion:    Mutex ensures one holder at a time
  *          2. Hold and wait:       Task holds one mutex while waiting for another
  *          3. No preemption:       Mutex can't be taken away from its holder
  *          4. Circular wait:       A waits for B-held-by-B, B waits for A-held-by-A
  *
  *          Prevention strategies:
  *          • Strategy 1: Uniform lock ordering (most practical!)
  *            All tasks lock in the SAME order: always A then B.
  *            → Breaks condition 4 (circular wait).
  *
  *          • Strategy 2: Lock timeout + retry (used here)
  *            If lock fails, release all held locks, wait, retry.
  *            → Breaks condition 2 (hold and wait).
  *
  *          • Strategy 3: Lock hierarchy
  *            Assign each mutex a number; can only lock higher-numbered
  *            mutexes while holding lower-numbered ones.
  *            → Also breaks condition 4.
  *
  *          • Strategy 4: Use xSemaphoreGetMutexHolder()
  *            Check who holds the mutex; if it's a lower-priority task,
  *            yield and try again.
  */
static void DeadlockTask_A(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        BaseType_t xDeadlockDetected = pdFALSE;

        /*
         * Step 1: Take the first mutex.
         * This should always succeed quickly (no contention yet).
         */
        if (xSemaphoreTake(g.xMutexA, pdMS_TO_TICKS(10)) == pdPASS)
        {
            /*
             * Step 2: DELAY on purpose — this is the window where
             * Task_B takes MutexB. Without this delay, Task_A might
             * race through and take both before Task_B even starts.
             */
            vTaskDelay(pdMS_TO_TICKS(50));

            /*
             * Step 3: Try to take the second mutex — WITH TIMEOUT.
             *
             * If Task_B already holds MutexB (and is waiting for MutexA),
             * this will block until timeout → deadlock detected.
             *
             * Using a timeout is a DEADLOCK DETECTION mechanism.
             * Without timeout (portMAX_DELAY) → permanent hang.
             */
            if (xSemaphoreTake(g.xMutexB,
                               pdMS_TO_TICKS(DEADLOCK_TIMEOUT_MS)) == pdPASS)
            {
                /*
                 * Both mutexes acquired! No deadlock this time.
                 * In production: access the shared resource here.
                 */
            }
            else
            {
                /*
                 * TIMEOUT! Task_B likely holds MutexB and is waiting
                 * for MutexA (which WE hold). Classic deadlock.
                 */
                xDeadlockDetected = pdTRUE;
            }

            /*
             * Step 4: Release whatever we hold.
             * ALWAYS release in REVERSE ORDER of acquisition.
             * This isn't just convention — it prevents the "release window"
             * where a third task could grab MutexB while we still hold MutexA.
             */
            xSemaphoreGive(g.xMutexB);   /* Safe to give even if not held    */
            xSemaphoreGive(g.xMutexA);   /* Release last acquired first      */
        }

        /*
         * Step 5: Visual feedback.
         * LED1 lights up briefly when deadlock is detected.
         * In a real system, you'd log this and possibly reset the subsystem.
         */
        if (xDeadlockDetected == pdTRUE)
        {
            g.ulDeadlockCount++;
            APP_LedOn(1);
            vTaskDelay(pdMS_TO_TICKS(200));
            APP_LedOff(1);
        }

        /*
         * Wait before the next deadlock cycle.
         * This gives Task_B time to complete/fail its own attempt.
         */
        vTaskDelay(pdMS_TO_TICKS(DEADLOCK_CYCLE_DELAY_MS));
    }
}

/**
  * @brief  DeadlockTask_B: Takes MutexB first, then MutexA.
  * @note   Mirror of Task_A, but lock order is REVERSED.
  *          This is what creates the circular wait.
  *
  *          FIX: If both tasks took locks in the same order
  *          (A then B), deadlock would be impossible.
  */
static void DeadlockTask_B(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        BaseType_t xDeadlockDetected = pdFALSE;

        /*
         * Step 1: Take MutexB FIRST (opposite of Task_A).
         * This is the root cause — different lock ordering.
         */
        if (xSemaphoreTake(g.xMutexB, pdMS_TO_TICKS(10)) == pdPASS)
        {
            /*
             * Step 2: Delay to create the overlap window.
             * Both tasks now hold one mutex and are about to
             * request the other → guaranteed deadlock.
             */
            vTaskDelay(pdMS_TO_TICKS(50));

            /*
             * Step 3: Try to take MutexA — which Task_A holds.
             */
            if (xSemaphoreTake(g.xMutexA,
                               pdMS_TO_TICKS(DEADLOCK_TIMEOUT_MS)) == pdPASS)
            {
                /* Both acquired — no deadlock (rare, requires timing luck) */
            }
            else
            {
                xDeadlockDetected = pdTRUE;
            }

            /* Step 4: Release (reverse order) */
            xSemaphoreGive(g.xMutexA);
            xSemaphoreGive(g.xMutexB);
        }

        if (xDeadlockDetected == pdTRUE)
        {
            g.ulDeadlockCount++;
            APP_LedOn(1);
            vTaskDelay(pdMS_TO_TICKS(200));
            APP_LedOff(1);
        }

        vTaskDelay(pdMS_TO_TICKS(DEADLOCK_CYCLE_DELAY_MS));
    }
}

/* ========================================================================== */
/*  Demo 3: Priority Inversion — The Mars Pathfinder Bug                        */
/* ========================================================================== */

/**
  * @brief  Priority Inversion demonstration — two phases.
  *
  * @note   THE CLASSIC SCENARIO (inspired by Mars Pathfinder, 1997):
  *
  *          Three tasks at different priorities share one resource.
  *          The resource is protected by a lock.
  *
  *          Phase 1: Binary Semaphore (NO priority inheritance)
  *
  *            Timeline:
  *              t=0:   Low (prio 2) takes the semaphore, starts working.
  *              t=10:  High (prio 4) preempts Low, tries to take sem → BLOCKS.
  *              t=11:  Medium (prio 3) runs — it needs NO lock, just hogs CPU.
  *              t=12~: Medium runs forever. Low can't get CPU to release lock.
  *              t=∞:   High is permanently blocked by Medium.
  *                     → This is PRIORITY INVERSION: High waits on Medium,
  *                       even though Medium doesn't hold the lock!
  *
  *          Phase 2: Mutex (WITH priority inheritance)
  *
  *            Timeline:
  *              t=0:   Low takes Mutex, starts working.
  *              t=10:  High preempts Low, tries to take Mutex → BLOCKS.
  *                     *** Kernel detects: Low holds the Mutex High needs. ***
  *                     *** Kernel ELEVATES Low's priority to 4 (High's prio). ***
  *              t=11:  Medium (prio 3) cannot preempt Low (prio 4).
  *                     Low finishes work uninterrupted.
  *              t=12:  Low releases Mutex → its priority drops back to 2.
  *              t=13:  High (now unblocked) takes Mutex, runs immediately.
  *                     → Priority inheritance RESOLVES the inversion.
  *
  *          LED2 feedback:
  *            Phase 1 (0~4s): LED2 toggles SLOWLY → inversion blocks High
  *            Phase 2 (4~8s): LED2 toggles QUICKLY → inheritance works
  *            Repeat.
  */

/**
  * @brief  Task_High — highest priority, needs the shared resource.
  * @note   Takes the lock, blinks LED2 to indicate successful access.
  *          During Phase 1 (semaphore): blocked by Medium (inversion).
  *          During Phase 2 (mutex): gets through quickly (inheritance).
  */
static void InversionTask_High(void *pvParameters)
{
    (void)pvParameters;

    SemaphoreHandle_t xLock;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /*
         * Select which lock to use based on current test phase.
         * Phase 0 = binary semaphore (no inheritance)
         * Phase 1 = mutex (with inheritance)
         */
        xLock = (g.ulInversionPhase == 0) ? g.xBinarySem : g.xMutexInv;

        /*
         * Attempt to take the lock. Block indefinitely.
         *
         * Phase 1 (Semaphore):
         *   Low holds it. Medium preempts Low. Low can't release.
         *   High waits a LONG time → g.ulHighBlockedCnt increments slowly.
         *
         * Phase 2 (Mutex):
         *   Low holds it. High preempts. Kernel raises Low to prio 4.
         *   Low finishes quickly. High gets through → counter increments fast.
         */
        if (xSemaphoreTake(xLock, portMAX_DELAY) == pdPASS)
        {
            /* Got the lock — toggle LED2 to show successful access */
            APP_LedToggle(2);
            g.ulHighBlockedCnt++;

            /* Release the lock immediately (minimal critical section) */
            xSemaphoreGive(xLock);
        }

        /*
         * Delay between access attempts.
         * Shorter delay = more frequent attempts = faster LED2 blink.
         */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(200));
    }
}

/**
  * @brief  Task_Med — medium priority, pure CPU consumer.
  * @note   This task does NOT need the shared lock.
  *          It just consumes CPU in a busy loop to keep Low from running.
  *
  *          During Phase 1: starvation of High (Medium blocks Low).
  *          During Phase 2: CANNOT preempt Low (Low's priority is elevated).
  *
  *          In a real system, this task might be:
  *          - A communication stack processing (TCP/IP)
  *          - A data logging/flash write operation
  *          - Any medium-priority background activity
  */
static void InversionTask_Med(void *pvParameters)
{
    (void)pvParameters;

    /*
     * Volatile prevents compiler from optimizing away the busy-wait.
     * Without volatile, a smart compiler could eliminate the loop entirely.
     */
    volatile uint32_t ulBusyCounter;

    for (;;)
    {
        /*
         * Busy loop — keeps the CPU occupied.
         * On PY32F403 @ 8MHz, ~500,000 iterations ≈ 100ms.
         * This is long enough to demonstrate the inversion effect.
         *
         * In production code, you'd use a CPU-bound operation like:
         *   - CRC calculation on a large buffer
         *   - Image processing
         *   - Flash sector erase (actually blocks via hardware, similar effect)
         */
        for (ulBusyCounter = 0; ulBusyCounter < 500000UL; ulBusyCounter++)
        {
            __NOP();   /* Prevent compiler optimization, 1 cycle each */
        }

        /*
         * Brief yield after each busy period.
         * This lets higher-priority tasks (High at prio 4) preempt.
         * But critically, it does NOT let Low (prio 2) run because
         * Medium itself is prio 3.
         *
         * During Phase 2 (inheritance): Low is elevated to prio 4,
         * so when Medium yields, Low runs instead!
         */
        taskYIELD();
    }
}

/**
  * @brief  Task_Low — lowest priority, holds the shared lock.
  * @note   This task simulates a sensor read + processing that takes time.
  *          With priority inheritance, its priority gets boosted while
  *          holding the mutex that High needs.
  */
static void InversionTask_Low(void *pvParameters)
{
    (void)pvParameters;

    SemaphoreHandle_t xLock;

    for (;;)
    {
        xLock = (g.ulInversionPhase == 0) ? g.xBinarySem : g.xMutexInv;

        /*
         * Take the lock.
         * Low priority → normally preemptible by Medium and High.
         * But with Mutex inheritance → elevated to High's priority.
         */
        if (xSemaphoreTake(xLock, portMAX_DELAY) == pdPASS)
        {
            /*
             * CRITICAL SECTION: hold the lock while doing "work".
             *
             * In a real system, this might be:
             *   - Reading an I2C sensor (100~400 kHz, takes ms)
             *   - Writing to flash (takes ms)
             *   - Computing a checksum over a large buffer
             *
             * The key: while Low holds the lock, High can't proceed.
             * Without inheritance: Medium steals CPU, Low starves.
             * With inheritance: Low's priority = High's → Medium can't steal.
             */
            for (volatile uint32_t i = 0; i < 200000UL; i++)
            {
                __NOP();   /* Simulated work: ~40ms @ 8MHz */
            }

            xSemaphoreGive(xLock);
        }

        /*
         * Toggle the test phase every INVERSION_PHASE_DURATION_MS.
         *
         * Phase 0 (seconds 0-4): Binary Semaphore — priority inversion
         * Phase 1 (seconds 4-8): Mutex — priority inheritance
         * Repeats every 8 seconds.
         *
         * LED2 serves as the visual indicator:
         *   Phase 0: Blinks SLOWLY (High is blocked by Medium through Low)
         *   Phase 1: Blinks FAST (High gets through quickly via inheritance)
         */
        vTaskDelay(pdMS_TO_TICKS(INVERSION_PHASE_DURATION_MS));

        /* Toggle phase for next round */
        g.ulInversionPhase ^= 1U;

        /*
         * Reset the blocked counter for the next phase.
         * This makes the LED blink rate comparison between phases clearer.
         */
        g.ulHighBlockedCnt = 0;
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

    /* LED0 — Scheduler lock demo */
    GPIO_InitStruct.Pin = LED0_PIN;
    HAL_GPIO_Init(LED0_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED0_PORT, LED0_PIN, GPIO_PIN_SET);

    /* LED1 — Deadlock detection */
    GPIO_InitStruct.Pin = LED1_PIN;
    HAL_GPIO_Init(LED1_PORT, &GPIO_InitStruct);
    HAL_GPIO_WritePin(LED1_PORT, LED1_PIN, GPIO_PIN_SET);

    /* LED2 — Priority inversion */
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

## Demo 3 优先级反转时序图 (教学用)

```
┌──────────────────────────────────────────────────────────────────────────┐
│  阶段1: Binary Semaphore (无优先级继承) — 优先级反转发生                   │
│                                                                          │
│  Priority                                                                 │
│    4  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████  High │
│       │ 抢占  │                               被 Medium 阻塞!              │
│    3  ░░░░░░░░████████████████████████████████████████████████░░░░  Med  │
│       │       │ Medium 霸占 CPU, 不让 Low 释放锁                           │
│    2  ░░░░░░░░░░░░░░████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Low   │
│       │             │ 持锁 │ 等 CPU ▶▶▶  永远等不到!                     │
│  ─────┼─────────────┼─────┼─────────────────────────────────────────     │
│  Time →               Lock Taken                                         │
│                                                                          │
│  阶段2: Mutex (有优先级继承) — 反转被解决                                   │
│                                                                          │
│  Priority                                                                 │
│    4  ████████░░░░░░░░░░░░░░░░░░░░░░░░████████████████████████████  High │
│       │ 抢占  │                 拿到锁!                                   │
│    3  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  Med   │
│       │       │ 不能抢占 Low (Low 被提升到 prio 4)                         │
│    2  ░░░░░░░░░██████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░  Low   │
│       │       │  持锁+提升  │ 释放锁, 恢复 prio 2                          │
│  ─────┼───────┼────────────┼────────────────────────────────────────     │
│  Time →       Lock Taken   Lock Released                                  │
│              ▲                                                           │
│              └── 内核自动将 Low 优先级从 2 → 4                             │
│                  (因为 High prio=4 在等待这个 Mutex)                       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 资源保护方案决策树

```
┌──────────────────────────────────────────────────────────────────┐
│              资源保护选型 (综合第4/8/9课)                           │
│                                                                  │
│  操作长度?                                                        │
│    ├── <1 μs → taskENTER/EXIT_CRITICAL                           │
│    │           (开销最低, 但会屏蔽所有中断)                         │
│    │                                                              │
│    ├── 1~100 μs, 不需要调阻塞API → vTaskSuspendAll/xTaskResumeAll│
│    │           (不丢中断, 但调度器暂停)                            │
│    │                                                              │
│    ├── >100 μs, 或需要调阻塞API → Mutex                           │
│    │           (不丢中断, 调度器正常, 支出优先级继承)              │
│    │                                                              │
│    └── ISR 与 Task 共享?                                          │
│          └── 只能: taskENTER/EXIT_CRITICAL (极短)                 │
│               或: volatile + FromISR Queue (数据安全)            │
│                                                                  │
│  多个锁竞争 → 死锁风险?                                            │
│    └── Yes → 统一锁顺序 / 超时检测 / xSemaphoreGetMutexHolder    │
│                                                                  │
│  优先级反转风险?                                                   │
│    ├── 用了 Semaphore 做互斥 → ❌ 无保护                          │
│    └── 用了 Mutex             → ✅ 内核自动优先级继承              │
└──────────────────────────────────────────────────────────────────┘
```

---

## FreeRTOSConfig.h 配置

```c
/* 确认以下宏已启用 (第1课已配置) */
#define configUSE_MUTEXES                  1   /* Mutex (含优先级继承) */
#define configUSE_RECURSIVE_MUTEXES        1   /* 递归Mutex           */

/* 不需要新配置 — 调度器锁是核心功能, 始终可用 */
```

---

## 内存预算

```
Item                    │ Size    │ Qty │ Subtotal
────────────────────────┼─────────┼─────┼─────────
Idle Task               │   760B  │  1  │   760B
Daemon Task             │   632B  │  1  │   632B
SchedulerLock Task      │   920B  │  1  │   920B   (200W stack + TCB)
DeadlockTask_A          │   920B  │  1  │   920B   (200W stack + TCB)
DeadlockTask_B          │   920B  │  1  │   920B   (200W stack + TCB)
Inversion_High          │   720B  │  1  │   720B   (150W stack + TCB)
Inversion_Med           │   720B  │  1  │   720B   (150W stack + TCB)
Inversion_Low           │  1320B  │  1  │  1320B   (300W stack + TCB)
MutexA                  │    60B  │  1  │    60B
MutexB                  │    60B  │  1  │    60B
BinarySem               │    60B  │  1  │    60B
MutexInv                │    60B  │  1  │    60B
────────────────────────┼─────────┼─────┼─────────
TOTAL                   │         │     │  7152B / 8192B  ✅
FREE                    │         │     │  1040B
```

---

## 验证清单

```
□ 1. 确认 #include "semphr.h" 在 main.c 中
□ 2. 编译 0 Error, 0 Warning
□ 3. 下载运行后观察:
□ 3a. LED0 (PB13) — 每2秒翻转1次
      → 证明: 调度器锁正常工作，多变量原子更新正确
□ 3b. LED1 (PA1)  — 间歇性短暂亮起 (~200ms)
      → 证明: 死锁被检测到 (xSemaphoreTake 超时)
      → LED1 亮起频率 = 死锁发生频率
□ 3c. LED2 (PB12) — 慢闪4秒 → 快闪4秒 → 慢闪...
      → 证明: 优先级反转 (慢) → 优先级继承解决 (快) 交替
□ 4.  在 DeadlockTask_A 的 xSemaphoreTake(MutexB) 处设断点:
      断点命中时超时 → 观察 g.xMutexB 被谁持有
□ 5.  在 InversionTask_High 的 xSemaphoreTake 返回后设断点:
      阶段1: 断点长时间不命中 (被反转阻塞)
      阶段2: 断点快速命中 (继承解决)
□ 6.  在 InversionTask_Low 的锁持有区设断点:
      查看 uxTaskPriorityGet(NULL) 的值:
        阶段1 (Semaphore): 优先级 = 2 (Low 自己的)
        阶段2 (Mutex):     优先级 = 4 (被内核提升了!)
```

---

## 常见陷阱

| 陷阱 | 说明 | 正确做法 |
|------|------|---------|
| 用 Semaphore 做互斥 | 无优先级继承 → 优先级反转风险 | 用 Mutex (`xSemaphoreCreateMutex`) |
| 调度器锁内调阻塞 API | `vTaskDelay` 在 Suspend/Resume 之间 → 死锁 | 锁内只做内存操作，不调 API |
| 锁顺序不一致 | TaskA 锁 A→B, TaskB 锁 B→A → 死锁 | 统一锁顺序 (所有任务 A→B) |
| `portMAX_DELAY` 等待第二把锁 | 死锁后永久挂起，无法恢复 | 用超时 + 检测 + 回退 |
| 临界区替代调度器锁 | 关中断做的操作太长 → 丢中断 | 区分使用场景: 长度决定方案 |
| 未释放 Mutex | 任务异常退出 → Mutex 永久锁定 | 始终在获取的同一任务中释放 |
| 优先级继承不理解 | 认为"信号量就够" → 嵌入式版 Mars Pathfinder | 互斥用 Mutex, 同步用 Semaphore |
| RecursiveMutex 遗忘 | 同一任务两次 Lock → 死锁 | 嵌套锁用 `xSemaphoreCreateRecursiveMutex` |

---

## 最佳实践清单

| # | 实践 | 说明 |
|---|------|------|
| 1 | 互斥用 Mutex, 同步用 Semaphore | 语义清晰, 继承保护 |
| 2 | 调度器锁仅用于短原子操作 | <100μs, 不调阻塞API |
| 3 | 临界区仅用于极短操作 | <1μs, 不调任何API |
| 4 | 多锁统一获取顺序 | 防止循环等待 → 消除死锁 |
| 5 | 锁超时 + 回退重试 | 替代 `portMAX_DELAY` — 可检测可恢复 |
| 6 | 获取锁的顺序 = 释放锁的反序 | 防止"释放窗口"被利用 |
| 7 | Mutex 在持有者任务中释放 | 所有权明确, 不跨任务 |
| 8 | 全局状态用结构体封装 | `g.mutexA` 风格 — 集中管理 |
| 9 | 用 `xSemaphoreGetMutexHolder()` 调试 | 查看谁持有锁阻塞了系统 |
| 10 | 死锁预防优于死锁检测 | 统一锁顺序比超时检测更可靠 |
