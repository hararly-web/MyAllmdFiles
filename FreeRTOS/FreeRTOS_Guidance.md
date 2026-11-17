# FreeRTOS 完整学习指南 (基于 PY32F403 + Keil MDK5)

> **适用平台:** PY32F403 / STM32F4xx / GD32F4xx 等 Cortex-M4F 系列 MCU
> **编译器:** Keil MDK5 (ARM Compiler 5 / RVDS)
> **FreeRTOS 版本:** V11.3.0 LTS
> **配套文件:** `FreeRTOSConfig_Reference.md` — FreeRTOSConfig.h 完整配置参考

***

## 目录与知识全景索引

### [第一部分：FreeRTOS 体系架构与学习路线图](#第一部分完整学习路线图)
- [1. 核心思维转变：从裸机轮询到 RTOS 抢占式多任务](#1-核心思维转变从裸机轮询到-rtos-抢占式多任务)
- [2. FreeRTOS 五大核心技术支柱（知识架构全景）](#2-freertos-五大核心技术支柱知识架构全景)
- [3. FreeRTOS 内核运行机理与上下文切换全景](#3-freertos-内核运行机理与上下文切换全景)
- [4. 5 阶段循序渐进学习路线图（12 课时全览）](#4-5-阶段循序渐进学习路线图12-课时全览)
- [5. 核心 IPC 通信与同步机制工程选型指南](#5-核心-ipc-通信与同步机制工程选型指南)
- [6. 单课高效研读闭环与技能自测清单](#6-单课高效研读闭环与技能自测清单)

### 第二部分：核心课时精讲与实战
- **【阶段一：内核基石与系统点亮】**
  - [第1课 — FreeRTOS 移植教程](#第二部分第1课--freertos-移植教程)
- **【阶段二：多任务核心与调度机理】**
  - [第2课 — 任务管理 (Task)](#第二部分第2课--任务管理)
- **【阶段三：并发通信与同步协作 (IPC)】**
  - [第3课 — 队列 (Queue)](#第二部分第3课--队列)
  - [第4课 — 信号量 (Semaphore & Mutex)](#第二部分第4课--信号量)
  - [第5课 — 软件定时器 (Software Timer)](#第二部分第5课--软件定时器)
  - [第6课 — 事件组 (Event Group)](#第二部分第6课--事件组)
  - [第7课 — 任务通知 (Task Notification)](#第二部分第7课--任务通知)
- **【阶段四：硬件交互与系统安全】**
  - [第8课 — 中断管理 (Interrupt & ISR)](#第二部分第8课--中断管理)
  - [第9课 — 资源管理 (临界区与死锁防范)](#第二部分第9课--资源管理)
  - [第10课 — 内存管理 (堆模型与底层栈机制)](#第二部分第10课--内存管理-memory-management)
- **【阶段五：进阶调优与综合实战】**
  - [第11课 — 低功耗 (Tickless Idle 模式)](#第二部分第11课--低功耗--tickless-idle-模式)

### [第三部分：踩坑记录与解决方案](#第三部分踩坑记录与解决方案)
### [附录：移植检查清单](#附录移植检查清单)

***

# 第一部分：完整学习路线图

> **本部分主旨：** 建立学习 FreeRTOS 的宏观心智模型（Mental Model）。  
> 嵌入式 RTOS 的学习切忌“一头扎进具体 API，只见树木不见森林”。本部分从**思维转变**、**内核全局架构**、**分阶段知识树**、**IPC 工程选型**到**学习自测标准**进行全景梳理，助你先搭建起坚实系统的知识骨架。

---

## 1. 核心思维转变：从裸机轮询到 RTOS 抢占式多任务

学习 RTOS 首先不是记函数名，而是打破传统的单片机前后台裸机开发思维定势。

| 维度 | 传统裸机开发 (Bare-Metal) | FreeRTOS 抢占式多任务 |
| :--- | :--- | :--- |
| **程序架构** | 单一 `main()` 主循环 (`while(1)`) + 中断服务函数 (ISR) | 拆分为多个相互独立的任务函数，每个任务逻辑上独占 CPU |
| **时间利用与阻塞** | 调用延时函数时 CPU 空转死等 (`delay_ms`)，浪费算力且无法响应其他业务 | 任务调用 `vTaskDelay()` 主动进入**阻塞态**，自动让出 CPU 供其他就绪任务运行 |
| **实时性响应** | 后台大循环执行周期不可预测（易受耗时函数拖累），强依赖中断 | **基于优先级抢占**：高优先级任务一旦就绪（微秒级）立刻抢占当前运行任务 |
| **多业务协同** | 依赖全局变量或标志位，存在高风险竞态隐患与并发冲突 | 依赖操作系统提供的线程安全机制：**队列、信号量、事件组、任务通知** |
| **内存与堆栈** | 全局共用单一系统栈（MSP），局部变量或深层调用极易发生隐式内存踩踏 | **每个任务拥有独立私有栈空间**，且具备运行时栈溢出监测与高水位线统计 |

> 💡 **核心心智模型：**  
> 在 FreeRTOS 中，CPU 就像一个高效的流水线调度站。所有任务遵循**“有事做事，无事阻塞出让”**的原则。调度器时刻保证**最高优先级的就绪任务独占 CPU**；当所有业务任务都处于阻塞态时，CPU 自动运行**空闲任务 (Idle Task)** 或进入低功耗休眠。

---

## 2. FreeRTOS 五大核心技术支柱（知识架构全景）

整个 FreeRTOS 内核的精髓可以解构为以下 **5 大核心技术支柱**，后续所有课程均围绕这 5 大支柱展开：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FreeRTOS 核心技术体系 (5 大支柱)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [支柱 1: 任务管理与调度] ──► 任务控制块 (TCB)、状态迁移机、优先级抢占、时间片轮转    │
│                                                                             │
│  [支柱 2: 任务间通信与同步] ─► 消息队列 (数据管道)、信号量 (同步/互斥)、             │
│                               事件组 (多标志组合)、任务通知 (轻量级私有信箱)        │
│                                                                             │
│  [支柱 3: 中断与硬件协同] ──► Cortex-M NVIC 优先级分层、xxxFromISR API 规范、       │
│                               上下文切换安全阈值 (MAX_SYSCALL)                      │
│                                                                             │
│  [支柱 4: 资源互斥与系统安全] ─► 临界区 (CRITICAL)、互斥锁与优先级继承、调度器锁、     │
│                               死锁防范与诊断 (configASSERT)                        │
│                                                                             │
│  [支柱 5: 内存管理与底层移植] ─► 动态内存算法 (heap_1~5)、静态分配、任务栈高水位检测、 │
│                               Cortex-M4 异常 (SVC/PendSV/SysTick) 与上下文出入栈    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. FreeRTOS 内核运行机理与上下文切换全景

理解 FreeRTOS 的运行机理，关键在于掌握**时钟节拍 (SysTick)**、**上下文切换 (PendSV)** 与 **中断服务 (ISR)** 之间的三角协同关系：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FreeRTOS 系统运行时全局事件驱动流                            │
└─────────────────────────────────────────────────────────────────────────────┘

     硬件时钟源 (SysTick 定时器，默认 1ms 产生一次中断)
               │
               ▼
   [ SysTick_Handler() 中断服务程序 ]
               │
               ├─► xTaskIncrementTick()：系统节拍计数自增
               │     └─► 遍历阻塞列表，检查是否有延时结束或超时的任务需要唤醒
               │
               └─► 判断就绪列表中是否存在比当前任务更高优先级的任务？
                     │
         ┌───────────┴───────────┐
         ▼                       ▼
       [ 是 ]                  [ 否 ]
         │                       │
         ▼                       ▼
   触发 PendSV 异常中断      退出中断，当前任务继续执行
         │
         ▼
   [ PendSV_Handler() 上下文切换核心 (汇编实现) ]
         │
         ├─► ① 保存旧任务上下文：硬件自动压入 R0~R3, R12, LR, PC, xPSR；
         │                      软件代码手动压入 R4~R11 到旧任务的私有栈
         ├─► ② 更新旧任务 TCB 的顶层栈指针 (pxCurrentTCB->pxTopOfStack)
         ├─► ③ 将 pxCurrentTCB 指向最高优先级的就绪任务 (新任务)
         ├─► ④ 恢复新任务上下文：从新任务私有栈手动弹出 R4~R11
         └─► ⑤ 硬件异常退出机制自动恢复 R0~R3, R12, LR, PC, xPSR
               │
               ▼
     新任务无缝从上次被打断处继续执行！
```

---

## 4. 5 阶段循序渐进学习路线图（12 课时全览）

为了避免学习过程中的迷茫与碎片化认知，本指南将 12 门核心课时划分为 **5 个循序渐进的学习阶段**。建议按照阶段顺序稳步推进：

### 阶段一：内核基石与系统点亮 (Getting Started)
*目标：理清 FreeRTOS 源码组织结构，在硬件平台上跑通多任务第一行代码。*
- **第 1 课 ◆ FreeRTOS 移植教程与启动流程**
  - **核心内容：** 内核纯 C 文件 vs 硬件移植层 (`port.c`, `portmacro.h`)；`FreeRTOSConfig.h` 关键宏详解；ARM Cortex-M 核心中断映射（SVC 启动首个任务，PendSV 负责上下文切换，SysTick 维护内核心跳）；`vTaskStartScheduler()` 底层汇编跳转流程。
  - **交付成果：** Keil 工程编译 0 Error / 0 Warning，点亮 LED 心跳多任务。

---

### 阶段二：多任务核心与调度机理 (Tasks & Scheduling)
*目标：彻底搞懂任务的生命周期、调度策略与内存分配边界。*
- **第 2 课 ◆ 任务管理 (Task) — RTOS 的灵魂**
  - **核心内容：** 动态创建 (`xTaskCreate`) 与静态创建 (`xTaskCreateStatic`)；任务的 4 大状态迁移机（运行 Running、就绪 Ready、阻塞 Blocked、挂起 Suspended）；相对延时 (`vTaskDelay`) vs 绝对周期延时 (`vTaskDelayUntil`)；空闲任务与钩子函数；栈溢出检测（金丝雀标记法）与栈高水位线 (`uxTaskGetStackHighWaterMark`)。
  - **交付成果：** 掌握任务优先级规划技巧，建立无死锁、无栈溢出的任务骨架。

---

### 阶段三：并发通信与同步协作 (IPC: Inter-Process Communication)
*目标：掌握任务之间、中断与任务之间传递数据和事件的全部核心手段。*
- **第 3 课 ◆ 队列 (Queue) — 任务间通信的基础**
  - **核心内容：** 环形缓冲区机制、值拷贝设计、发送与接收阻塞超时机制、队列集 (Queue Set) 与邮箱模式 (Mailbox)。
- **第 4 课 ◆ 信号量 (Semaphore & Mutex) — 同步与互斥**
  - **核心内容：** 二值信号量（事件同步通知）、计数信号量（有限资源池管理）、互斥锁 (Mutex) 与优先级继承机制（解决优先级反转）、递归互斥锁。
- **第 5 课 ◆ 软件定时器 (Software Timer) — 节约硬件定时器资源**
  - **核心内容：** 单次定时器 (One-shot) 与周期定时器 (Auto-reload)；定时器守护任务 (`prvTimerTask`) 与定时器命令队列；回调函数严禁阻塞设计准则。
- **第 6 课 ◆ 事件组 (Event Group) — 多事件复合同步**
  - **核心内容：** 24-bit 事件位管理；多任务 AND 逻辑等待（所有条件满足才运行）与 OR 逻辑等待（任一事件满足即运行）；任务汇合点广播同步 (Sync Points)。
- **第 7 课 ◆ 任务通知 (Task Notification) — 轻量级私有通信神器**
  - **核心内容：** 任务控制块 TCB 内嵌通知值；模拟二值信号量、计数信号量、事件组和轻量队列；性能极限测试（零内存申请、速度提升 45%）。

---

### 阶段四：硬件交互与系统安全边界 (Interrupts & System Safety)
*目标：攻克嵌入式系统的最难关卡——中断分层、资源死锁与堆内存碎片。*
- **第 8 课 ◆ 中断管理 — RTOS 与硬件世界的安全握手**
  - **核心内容：** Cortex-M NVIC 优先级分组规范；`configMAX_SYSCALL_INTERRUPT_PRIORITY` 的安全防线；`xxxFromISR()` API 使用准则与 `portYIELD_FROM_ISR()` 现场快速切换；中断延迟与前后台解耦。
- **第 9 课 ◆ 资源管理 — 共享资源的保护与死锁防范**
  - **核心内容：** 临界区保护 (`taskENTER_CRITICAL`) vs 调度器锁 (`vTaskSuspendAll`) vs 互斥锁 (Mutex) 性能与适用场景横向评测；死锁四大必要条件；优先级反转本质与防御。
- **第 10 课 ◆ 内存管理 — 堆与栈的幕后真相**
  - **核心内容：** `heap_1` ~ `heap_5` 内存算法底层原理与适用场景；Keil Program Size (Code/RO/RW/ZI) 内存分布与散列加载；Cortex-M4F PC/LR/SP 寄存器与调用栈故障分析。

---

### 阶段五：进阶调优与综合实战 (Advanced & Integration)
*目标：工程落地调优，掌握低功耗策略与大型系统集成设计。*
- **第 11 课 ◆ 低功耗设计 — Tickless Idle 模式**
  - **核心内容：** 传统空闲浪费 vs Tickless 机制；停止 SysTick、利用低功耗定时器计算长睡眠补偿；MCU Sleep / Stop / Standby 模式的平滑唤醒流程。
- **第 12 课 ◆ 综合项目实战 — 多任务数据采集 + 状态机 + GUI 联动**
  - **核心内容：** 传感器流水线设计；FreeRTOS 与嵌入式 GUI (如 LVGL) 线程安全集成模型；系统运行状态监控看板与实时性能压测。

---

## 5. 核心 IPC 通信与同步机制工程选型指南

在实际工程开发中，“该用队列、信号量、事件组还是任务通知？”是高频纠结的核心痛点。以下提供清晰的工程决策指引：

### 5.1 快速决策逻辑树

```
                         需要传递数据或事件同步？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
      [ 需要传递数据 ]                                  [ 仅需事件/状态同步 ]
           │                                                 │
   数据量大小与目标对象？                              同步关系与场景？
     ├─ 复杂结构体 / 多字节数据 / 多接收者               ├─ 仅通知单一指定任务
     │   └─► 【消息队列 Queue】                            │   └─► 【任务通知 Task Notification】(最快最省)
     │                                                   │
     ├─ 单个 32-bit 整数 / 接收者唯一                    ├─ 中断通知任务 (且无需特定目标)
     │   └─► 【带值任务通知 Task Notification】            │   └─► 【二值信号量 Binary Semaphore】
     │                                                   │
     └─ 大块缓冲区数据 (如图像/音频流)                   ├─ 多个事件组合等待 (与/或逻辑)
         └─► 【指针入队 (零拷贝 Queue) 或 StreamBuffer】    │   └─► 【事件组 Event Group】
                                                         │
                                                         ├─ 多实例资源池访问计数 (如空闲缓冲区数)
                                                         │   └─► 【计数信号量 Counting Semaphore】
                                                         │
                                                         └─ 独占硬件外设 / 全局资源保护
                                                             └─► 【互斥锁 Mutex】(带优先级继承)
```

### 5.2 四大常用 IPC 机制全方位对比矩阵

| 特性指标 | 消息队列 (Queue) | 信号量 (Semaphore) | 事件组 (Event Group) | 任务通知 (Task Notification) |
| :--- | :--- | :--- | :--- | :--- |
| **主要定位** | 数据传输与缓冲 | 行为同步 / 资源互斥 | 多条件复合逻辑等待 | 任务专属轻量级信箱 |
| **数据承载能力** | 任意大小（支持值拷贝或指针） | 无数据（仅有计数值/标志） | 24 个独立二进制事件位 | 单个 32 位整型数值 |
| **通信目标关系** | 多对多（任意任务/中断可发可收） | 多对多（任意任务/中断可发可收） | 多对多（支持一对多广播） | **严格一对一（必须指定接收任务）** |
| **阻塞与排队机制**| 发送方与接收方均可设置超时阻塞 | Take 端可设置超时阻塞 | 等待端可设置超时阻塞 | 接收端可设置超时阻塞 |
| **RAM 开销** | 较高（需分配独立队列控制块+缓冲）| 中等（基于队列控制块实现） | 中等（需分配独立事件组控制块） | **零开销（直接复用 TCB 内置字段）** |
| **执行耗时** | 较长（包含内存拷贝与锁列表检查）| 中等 | 中等 | **极短（比队列快约 45%）** |
| **ISR 支持** | 支持 `xQueueSendFromISR` | 支持 `xSemaphoreGiveFromISR` | 支持 `xEventGroupSetBitsFromISR` | 支持 `vTaskNotifyGiveFromISR` |
| **典型应用场景** | 串口/CAN 接收帧分发、传感器采样流水线 | 外设就绪中断触发、I2C/SPI 总线独占访问 | 系统自检多模块就绪、多传感器就绪合并处理 | 高频按键触发、单任务状态驱动、极速响应 |

---

## 6. 单课高效研读闭环与技能自测清单

为了确保每一节课学懂、吃透、能落地，建议在后续课程学习中严格践行以下 **单课 6 步工程闭环**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       单课标准 6 步学习闭环方法论                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ① 理论剖析 ──► 搞清核心概念、痛点背景与底层运行逻辑 (拒绝死记硬背 API)       │
│       │                                                                     │
│  ② 时序图解 ──► 研读函数调用时序、任务状态转换机与数据流动方向图            │
│       │                                                                     │
│  ③ 最小代码 ──► 在 PY32F403 平台上构建可运行的最简核心验证代码              │
│       │                                                                     │
│  ④ 编译烧录 ──► 在 Keil MDK5 中验证 0 Error / 0 Warning，实机烧录运行       │
│       │                                                                     │
│  ⑤ 调试观测 ──► 利用串口调试助手、断点单步、GPIO 翻转配合逻辑分析仪观察波形 │
│       │                                                                     │
│  ⑥ 最佳实践 ──► 总结典型易错踩坑点 (如栈溢出、优先级反转、ISR 阻塞) 与防御方案│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 学习效果自测通关标准 (Milestone Checklist)

学完本指南全部内容后，如果你能自信回答以下 6 个问题，说明你已经真正掌握了 FreeRTOS 的内核与工程实践：

- [ ] **Q1. 状态流转：** 能否手绘出任务从就绪态、运行态、阻塞态到挂起态的完整状态机转移图，并指出各 API（`vTaskDelay`, `xQueueReceive`, `vTaskSuspend` 等）引发的状态跃迁？
- [ ] **Q2. 现场保护：** 能否讲清上下文切换发生时，硬件自动压栈了哪些寄存器？软件代码手动压栈了哪些寄存器？它们保存在哪儿？
- [ ] **Q3. 中断安全：** 能否讲清为什么中断服务函数里绝对不能调用非 `FromISR` 后缀的 API？`configMAX_SYSCALL_INTERRUPT_PRIORITY` 设置为 `0x50` 究竟切断了哪些中断？
- [ ] **Q4. 同步选型：** 面对“串口中断接收一包定长数据，交给处理任务解析”的需求，你为什么选队列而不是任务通知？若接收方是唯一任务且数据放在全局 DMA 缓冲区，你的选型会发生什么变化？
- [ ] **Q5. 锁与死锁：** 能否说明二值信号量和互斥锁的关键区别？什么是优先级反转？优先级继承是如何解决这一难题的？
- [ ] **Q6. 内存与稳定性：** 怎样利用 `uxTaskGetStackHighWaterMark()` 估算任务的栈余量？FreeRTOS 栈溢出钩子函数是在何时被触发的？

***

# 第二部分：第1课 — FreeRTOS 移植教程

> **本课目标：**  
> 1. 理清 FreeRTOS 源码分层架构，明确“哪些文件必加、哪些文件坚决不加”。  
> 2. 掌握 `FreeRTOSConfig.h` 核心配置逻辑与 V11.x 关键宏定义。  
> 3. 攻克移植中最易崩溃的难点：Cortex-M 中断优先级分组与 `MAX_SYSCALL` 安全生死线。  
> 4. 实现 HAL 库与 FreeRTOS 对 SysTick 的平滑兼容，在 Keil MDK5 中点亮首个多任务工程。  
> 5. 深入底层硬件：掌握双堆栈指针 (MSP/PSP)、中断三大金刚 (SVC/PendSV/SysTick) 与上下文切换全过程。

---

## 1.1 什么是“移植”？内核的分层解耦设计

移植 (Porting) 的本质，就是**把通用的操作系统的调度逻辑，适配到具体的单片机硬件与编译器上**。

FreeRTOS 的代码设计极其精妙——约 **90% 是纯 C 语言编写的内核算法**（队列、链表、任务调度逻辑），与硬件完全无关；只有约 **10% 的代码需要直接操作 CPU 寄存器与中断**（称为移植层 Port Layer）。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FreeRTOS 内核架构分层模型                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   [ 应用业务层 ] ──► main.c、各类用户 Task 业务代码 (你负责编写)             │
│                                                                             │
│   [ API 接口层 ] ──► task.h, queue.h, semphr.h, event_groups.h ... (纯 C)   │
│                                                                             │
│   [ 内核核心层 ] ──► tasks.c (调度), queue.c (IPC), list.c (双向链表) (纯 C) │
│                      ★ 这部分代码官方原样提供，无论什么芯片一个字符都不动！    │
│                                                                             │
│   [ 硬件移植层 ] ──► port.c / portmacro.h / heap_4.c (汇编 + 硬件相关 C)    │
│                      ★ 针对不同 CPU 内核 (Cortex-M4F) 与编译器 (Keil ARMCC) │
│                                                                             │
│   [ 硬件底层 ]   ──► ARM Cortex-M4F 内核 (NVIC, SysTick, FPU) + PY32F403 外设│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> 📌 **移植黄金法则：**  
> 移植 FreeRTOS **绝不需要修改内核源码的一行代码**！移植的核心工作只有三件事：  
> 1. **选对移植层文件**（编译器 RVDS + 架构 ARM_CM4F + 内存管理 heap_4.c）；  
> 2. **配对配置文件**（提供专属当前项目的 `FreeRTOSConfig.h`）；  
> 3. **接管中断向量**（将系统异常 SVC/PendSV/SysTick 接入 FreeRTOS 调度逻辑）。

---

## 1.2 源码精简清单：抛弃多余文件，认准 8 个核心文件

从官方下载的 FreeRTOS 源码包通常非常庞大（包含几十种架构和大量 Demo）。初学者切忌把所有文件一股脑拖进工程，必须做“瘦身精简”。

### 1.2.1 源码来源与推荐版本
- **官方仓库：** `https://github.com/FreeRTOS/FreeRTOS-Kernel`（推荐，体积小且仅含内核纯净代码）
- **当前工程版本：** `FreeRTOS V11.3.0 LTS`（包含长效支持与多项安全特性增强）

### 1.2.2 极简工程文件清单（7 个 .c + 1 个 port.c + 1 个 heap_4.c）

在工程中仅需加入以下 **9 个源文件**，其余目录全部忽略：

| 分组分类 | 必须添加的文件 | 文件职责与说明 |
| :--- | :--- | :--- |
| **内核核心** (`FreeRTOS/Kernel`) | `FreeRTOS-Kernel/tasks.c` | 核心中的核心：任务创建、状态迁移、时间片调度 |
| | `FreeRTOS-Kernel/queue.c` | 队列与信号量底层通用实现 |
| | `FreeRTOS-Kernel/list.c` | 就绪/阻塞/延时任务链表的基本操作算法 |
| | `FreeRTOS-Kernel/timers.c` | 软件定时器守护任务与时序管理 |
| | `FreeRTOS-Kernel/event_groups.c` | 多事件标志组等待与广播机制 |
| | `FreeRTOS-Kernel/stream_buffer.c` | 流缓冲区 / 消息缓冲区底层支持 |
| **硬件移植** (`FreeRTOS/Port`) | `portable/RVDS/ARM_CM4F/port.c` | Cortex-M4F 硬件上下文切换、中断屏蔽与节拍启动 |
| **内存管理** (`FreeRTOS/MemMang`) | `portable/MemMang/heap_4.c` | 动态内存分配算法（支持内存块自动合并碎片，最推荐） |

> ⚠️ **新手避坑（坚决不要添加）：**  
> 1. `croutine.c`：协程文件（已被官方废弃数年，切勿添加）。  
> 2. `heap_1.c ~ heap_3.c, heap_5.c`：MemMang 目录下 5 选 1，**只能添加 `heap_4.c`**，绝不可同时添加多个！  
> 3. `cmsis_os.c / cmsis_os2.c`：CMSIS 包装层。学习原生 FreeRTOS 时切勿添加，直接调用标准原生 API 效率最高。

---

## 1.3 Keil MDK5 移植实操四部曲

### 第一步：在 Keil 工程中创建 Group 并添加文件

在 Keil 的 Project 窗口中新建 3 个 Group，将上述 9 个源文件归类添加：

```
Project/
├── FreeRTOS/Kernel       <-- 添加 tasks.c, queue.c, list.c, timers.c, event_groups.c, stream_buffer.c
├── FreeRTOS/Port         <-- 添加 portable/RVDS/ARM_CM4F/port.c
└── FreeRTOS/MemMang      <-- 添加 portable/MemMang/heap_4.c
```

### 第二步：配置 Keil 头文件包含路径 (Include Paths)

点击 `Options for Target` ➔ `C/C++` ➔ `Include Paths`，新增以下 **2 个必要路径**：

```
Include Paths:
  ..\Inc                                                                         <-- 必须排在最前面！
  ... (原有 HAL 驱动和 CMSIS 路径保持不变) ...
  ..\..\..\..\..\..\Middlewares\Third_Party\FreeRTOS-Kernel\include               <-- 内核头文件
  ..\..\..\..\..\..\Middlewares\Third_Party\FreeRTOS-Kernel\portable\RVDS\ARM_CM4F <-- portmacro.h 所在路径
```

> 🚨 **关键细节：**  
> `..\Inc` 必须在 Include Paths 中排在第一位！这样编译器才会优先选用你项目中的 `FreeRTOSConfig.h`，避免因路径覆盖错误引用系统或 SDK 默认旧配置。

### 第三步：在 `Inc/` 目录下创建 `FreeRTOSConfig.h`

`FreeRTOSConfig.h` 是系统的总裁缝，所有的功能裁减与资源分配都通过此文件完成。  
针对 PY32F403 (Cortex-M4F) + V11.x 的核心配置精要如下：

```c
#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

#include <stdint.h>
extern uint32_t SystemCoreClock;

/* ======================== 1. 基础时钟与调度配置 ======================== */
#define configUSE_PREEMPTION                    1                   /* 1: 启用抢占式调度器; 0: 合作式轮流调度 */
#define configUSE_TIME_SLICING                  1                   /* 1: 同优先级任务时间片轮转; 0: 不轮转 */
#define configCPU_CLOCK_HZ                      ( 8000000UL )        /* CPU 主频 (PY32F403 内部 HSI 默认 8MHz, 请与实际系统时钟严格匹配) */
#define configTICK_RATE_HZ                      ( 1000 )            /* 1000Hz = 1ms 一个 Tick 节拍 */
#define configMAX_PRIORITIES                    ( 8 )               /* 可用任务优先级数 (0 ~ 7, 0 为最低优先级) */
#define configMINIMAL_STACK_SIZE                ( ( uint16_t ) 128 )/* 空闲任务栈大小 (单位: 4 字节 Word, 128 = 512 字节) */
#define configMAX_TASK_NAME_LEN                 ( 16 )              /* 任务名称最大字符长度 (包含结束符) */

/* ======================== 2. 内存管理模型 ======================== */
#define configSUPPORT_DYNAMIC_ALLOCATION        1                   /* 1: 支持动态内存分配 (xTaskCreate 使用 heap_4) */
#define configSUPPORT_STATIC_ALLOCATION         1                   /* 1: 支持静态内存分配 (xTaskCreateStatic) */
#define configTOTAL_HEAP_SIZE                   ( ( size_t ) ( 16 * 1024 ) ) /* FreeRTOS 堆大小 (16KB, 从芯片 RAM 中分配) */

/* ======================== 3. V11.x 核心专属宏 ======================== */
#define configTICK_TYPE_WIDTH_IN_BITS           TICK_TYPE_WIDTH_32_BITS  /* 32 位 Tick 计数器 (替换了旧版的 configUSE_16_BIT_TICKS) */
#define configUSE_TIMERS                        1                   /* 启用软件定时器 */
#define configTIMER_TASK_PRIORITY               ( configMAX_PRIORITIES - 1 ) /* 定时器守护任务优先级 */
#define configTIMER_QUEUE_LENGTH                5                   /* 定时器命令队列深度 */
#define configTIMER_TASK_STACK_DEPTH            configMINIMAL_STACK_SIZE
#define configUSE_EVENT_GROUPS                  1                   /* 启用事件组 */
#define configUSE_STREAM_BUFFERS                1                   /* 启用流缓冲区 */

/* ======================== 4. 安全防护与钩子函数 ======================== */
#define configCHECK_FOR_STACK_OVERFLOW          2                   /* 栈溢出检测: 2 = 金丝雀魔数填充检测 (最严格) */
#define configUSE_MALLOC_FAILED_HOOK            0                   /* 内存申请失败钩子 */
#define configUSE_IDLE_HOOK                     0                   /* 空闲任务钩子 */
#define configUSE_TICK_HOOK                     0                   /* 节拍钩子 */

/* ======================== 5. Cortex-M 中断优先级配置 ======================== */
#define configPRIO_BITS                         4                   /* PY32F403 实现的 NVIC 优先级位数 (4 位 = 16 级优先级) */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY          15          /* 库级别最低优先级 (0xF) */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY     5           /* 可调用 API 的最高中断优先级 (0x5) */

/* 转换为硬件 NVIC 寄存器对齐值 (左移 4 位对齐) */
#define configKERNEL_INTERRUPT_PRIORITY         ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << ( 8 - configPRIO_BITS ) )
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << ( 8 - configPRIO_BITS ) )

/* ======================== 6. 中断服务函数映射 (Direct Routing) ======================== */
#define vPortSVCHandler                         SVC_Handler
#define xPortPendSVHandler                      PendSV_Handler
/* 注意: SysTick_Handler 不在此处直接映射，而在 py32f403_it.c 中统一由 HAL 与 FreeRTOS 协同调用 */

#endif /* FREERTOS_CONFIG_H */
```

### 第四步：对接中断处理 (`py32f403_it.c`)

FreeRTOS 需要接管系统的三大异常：

#### 1. SVC 与 PendSV 接管（直接宏路由）
因为在 `FreeRTOSConfig.h` 中已经定义了：
```c
#define vPortSVCHandler     SVC_Handler
#define xPortPendSVHandler  PendSV_Handler
```
所以 `portable/RVDS/ARM_CM4F/port.c` 编译出的中断处理函数就直接名为 `SVC_Handler` 和 `PendSV_Handler`。  
**务必检查 `py32f403_it.c`：** 如果该文件中存在默认的 `SVC_Handler` 和 `PendSV_Handler` 空函数，必须**将其注释掉或删除**，否则链接器会报 `Symbol Multiply Defined`（符号重复定义错误）！

#### 2. SysTick 处理：HAL 库与 FreeRTOS 共享时钟
在嵌入式开发中，HAL 库的 `HAL_Delay()` 依赖 `HAL_IncTick()`，而 FreeRTOS 依赖 `xPortSysTickHandler()`。  
二者共享 SysTick 的最佳工程范式如下（编辑 `Src/py32f403_it.c`）：

```c
#include "py32f403_it.h"
#include "FreeRTOS.h"
#include "task.h"

void SysTick_Handler(void)
{
    /* 1. 始终维持 HAL 库的时基正常累加 (HAL_Delay 依赖此函数) */
    HAL_IncTick();

    /* 2. 调度器启动之后，才允许调用 FreeRTOS 的节拍处理函数 */
    #if (INCLUDE_xTaskGetSchedulerState == 1)
    if (xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED)
    {
        xPortSysTickHandler();
    }
    #else
    /* 若未开启查询状态宏，直接调用 */
    xPortSysTickHandler();
    #endif
}
```

> 💡 **防御式编程细节：**  
> 在 `main()` 函数刚刚执行、调度器尚未启动时（`vTaskStartScheduler` 调用前），SysTick 就可能因初始化而开始中断。如果直接调用 `xPortSysTickHandler()`，由于内部任务链表尚未就绪，极易发生 HardFault。通过 `xTaskGetSchedulerState()` 门禁判断可彻底避免该隐患。


## 1.4 移植最大痛点：NVIC 中断优先级与安全生死线

**这是 90% 的初学者在移植 FreeRTOS 时遇到 HardFault 或系统假死的根源！必须彻底吃透。**

### 1.4.1 Cortex-M 优先级寄存器的高位对齐机制

ARM Cortex-M 的中断优先级寄存器宽度是 **8 位**。但实际芯片厂商为了节约硅片面积，通常只实现高几位：

```
Cortex-M NVIC 优先级寄存器物理位布局:
┌───────┬───────┬───────┬───────┬───────┬───────┬───────┬───────┐
│ Bit 7 │ Bit 6 │ Bit 5 │ Bit 4 │ Bit 3 │ Bit 2 │ Bit 1 │ Bit 0 │
├───────┼───────┼───────┼───────┼───────┼───────┼───────┼───────┤
│    实  现  的  高  4  位     │  未实现 (硬件恒为 0, 写入无效) │
└───────────────────────────────┴───────────────────────────────┘
```

这就导致了一个极易混淆的概念：**“库函数逻辑优先级”** 与 **“NVIC 硬件寄存器值”** 的换算：
- 比如你想配置逻辑优先级为 5；
- 写入 NVIC 寄存器时必须左移 4 位（高位对齐）：`5 << (8 - 4) = 5 << 4 = 0x50`；
- 如果你直接把数字 5 写入寄存器，因为低 4 位未实现，硬件实际读到的值是 `0x00`（无意中变成了不可阻挡的最高优先级 0，从而导致系统在调用 API 时彻底崩溃）！

### 1.4.2 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的安全防线

FreeRTOS 为了保证内核数据结构的原子性，在进入临界区时并不是简单暴力地关闭全局中断 (`CPSID i`)，而是利用了 Cortex-M 的 **`BASEPRI` 寄存器**。

`BASEPRI` 的工作机制是：**屏蔽所有硬件优先级低于或等于 BASEPRI 设定值的中断，而高于该值的中断依然可以快速响应！**

```
中断优先级数值 (越小代表优先级越高):
0 (最高优先级)  ──► NMI / HardFault / 高精度电机故障中断
1               │   ★ 绝对不能调用任何 FreeRTOS API (包括 xxxFromISR)
2               │   ★ 实时性极高，即使 FreeRTOS 在执行临界区任务，也能打断！
3               │
4               ▼
─────────────────── configLIBRARY_MAX_SYSCALL = 5 (BASEPRI 设定的安全边界: 0x50) ───
5 (安全阈值)    ▲
6               │   ★ 可以安全调用 FreeRTOS 的 xxxFromISR() 系列 API
7               │   ★ 当任务进入临界区时，这些中断会被暂时屏蔽，确保内核数据安全
...             │
14              │
15 (最低优先级) ──► SysTick / PendSV (内核自身的异常必须放在最低优先级 0xF0)
```

### 1.4.3 芯片位数判断与通用三段式配置

为了彻底避免写死数值导致的兼容性问题，工业级代码统一采用“三段式”公式：

```c
/* 第一段: 芯片实现的有效物理位数 (PY32F403 通常为 4) */
#define configPRIO_BITS                                 4

/* 第二段: 用户视角的人类友好数值 (0~15) */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY         15   /* 最低: 分配给 SysTick 和 PendSV */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY    5    /* 阈值: 优先级 0~4 为不受管的极速中断, 5~15 受管 */

/* 第三段: 自动通过位移生成寄存器值 (严禁人工硬编码写入) */
#define configKERNEL_INTERRUPT_PRIORITY        ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << ( 8 - configPRIO_BITS ) )
/* 计算结果: 15 << 4 = 0xF0 */

#define configMAX_SYSCALL_INTERRUPT_PRIORITY   ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << ( 8 - configPRIO_BITS ) )
/* 计算结果: 5 << 4 = 0x50 */
```

> 🔴 **死记两条铁律：**  
> 1. **在任何 ISR 中调用 `xxxFromISR()` API，其中断优先级数值必须 ≥ `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`（即硬件优先级不高于安全阈值）。**  
> 2. **`SysTick` 和 `PendSV` 的中断优先级必须配置为整个系统的最低优先级 (`0xF0` 或 `0xE0`)。** 保证它们绝不打断用户的外设中断。


## 1.5 极简验证工程：首个多任务 LED 点亮

以下是用于验证移植成功的标准最小工程 `Src/main.c` 模板：

```c
#include "main.h"
#include "FreeRTOS.h"
#include "task.h"

/* 硬件初始化接口 */
static void SystemClock_Config(void);
static void GPIO_LED_Init(void);

/* 任务函数原型 */
static void Task_LED_Blink(void *pvParameters);

int main(void)
{
    /* 1. 底层硬件与 HAL 初始化 */
    HAL_Init();
    SystemClock_Config();
    GPIO_LED_Init();

    /* 2. 创建第一个业务任务 (动态分配)
     * 参数依次为: 任务函数, 调试名称, 栈大小(Words), 传递参数, 优先级, 任务句柄
     */
    BaseType_t xReturn = xTaskCreate(
        Task_LED_Blink,
        "LED_Blink",
        128,                     /* 128 Words = 512 字节栈 */
        NULL,
        1,                       /* 优先级 1 */
        NULL
    );

    if (xReturn == pdPASS)
    {
        /* 3. 🚀 启动任务调度器 —— 这一步将开启多任务世界，正常情况下永不返回！ */
        vTaskStartScheduler();
    }

    /* 4. 如果代码运行到了这里，说明堆内存不足导致空闲任务创建失败 */
    for (;;);
}

/* 业务任务实现 */
static void Task_LED_Blink(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        HAL_GPIO_TogglePin(LED3_GPIO_PORT, LED3_PIN);
        
        /* 调用 vTaskDelay 阻塞延时 500ms
         * 注意: pdMS_TO_TICKS 宏根据 configTICK_RATE_HZ 自动计算 tick 数，严禁写死数字
         */
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* 栈溢出保护钩子函数 (当 configCHECK_FOR_STACK_OVERFLOW > 0 时必须实现) */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    (void)xTask;
    (void)pcTaskName;
    
    /* 关中断，死循环在此处，方便调试器挂载查看崩溃现场 */
    __disable_irq();
    for (;;);
}
```


## 1.6 深入底层：Cortex-M4 启动与上下文切换机理（内功心法）

在硬件成功点亮之后，我们需要探究其底层运作机理。

### 1.6.1 双堆栈指针架构：MSP 与 PSP
ARM Cortex-M 架构内建了两个物理栈指针寄存器（但在同一时刻通用寄存器 R13 只映射其中一个）：

```
                  ┌───────────────────────────────┐
                  │   Cortex-M4 物理寄存器 R13    │
                  └──────────────┬────────────────┘
                                 │
                 ┌───────────────┴───────────────┐
                 ▼                               ▼
       [ MSP (Main Stack) ]            [ PSP (Process Stack) ]
       • 系统复位后默认使用            • 由 FreeRTOS 调度器激活
       • 所有 中断服务函数(ISR) 使用   • 每个 Task 切换运行时私有使用
       • 保护系统栈安全                • 任务栈溢出时不会冲垮 ISR 栈
```

### 1.6.2 中断“三大金刚”协同作战全景

FreeRTOS 的高效运行依赖三个核心中断的紧密配合：

```
┌──────────────────┬─────────────────────────────────────────────────────────────┐
│ 中断名称         │ 核心职责与分工                                              │
├──────────────────┼─────────────────────────────────────────────────────────────┤
│ SVC (0x0B)       │ 启动调度器发令枪：在特权模式下触发，从任务栈中加载首个任务的现场 │
│ SysTick (0x0F)   │ 系统心跳脉搏：定时产生 1ms 中断，推进时间片并唤醒休眠任务       │
│ PendSV (0x0E)    │ 真正的上下文切换工兵：在最低优先级执行任务现场的保存与恢复     │
└──────────────────┴─────────────────────────────────────────────────────────────┘
```

#### 为什么上下文切换必须由最低优先级的 PendSV 完成？
假设在某个外设中断（如 UART 接收）正在执行时，SysTick 中断到来并检测到需要切换任务。  
如果 SysTick 立即强行执行上下文切换，就会**把中断服务程序的执行现场打断**，导致严重的中断嵌套错乱！  
因此，FreeRTOS 的设计精髓是：SysTick 仅仅是给 NVIC 的悬起寄存器打上一个标记（“悬起 PendSV”）。等到系统中**所有正在执行的高优先级中断都安全退出了**，CPU 才会响应最低优先级的 PendSV，在干干净净的环境下进行任务上下文切换！

```
     UART 中断正在执行中...
              │
              ├───► SysTick 中断打断 UART 中断
              │        │
              │        └─► 判定需要任务切换，设置 PendSV 悬起标志 (ICSR |= PENDSVSET)
              │        │
              │        └─► SysTick 退出
              │
              ├───► 返回并继续执行 UART 中断剩余代码...
              │
              ▼
     UART 中断执行完毕完全退出 (此时系统没有任何硬件中断在执行)
              │
              ▼
     [ PendSV 中断立即响应！执行纯净的任务上下文切换 ]
```

### 1.6.3 栈帧初始化与现场出入栈全过程

当调用 `xTaskCreate()` 创建任务时，底层调用了 `pxPortInitialiseStack()`。这个函数的本质是**在任务的私有栈里“伪造”一个看似刚刚发生过中断的现场**。

```
任务私有栈内存空间 (地址由高到低):

高地址  ┌────────────────────────┐
        │  xPSR (0x01000000)     │  <-- 第 24 位为 1, 强制 CPU 进入 Thumb 运行状态
        ├────────────────────────┤
        │  PC (任务函数入口地址)  │  <-- 任务第一行代码的地址 (如 LED_Task)
        ├────────────────────────┤
        │  LR (prvTaskExitError) │  <-- 任务异常返回时的兜底保护函数
        ├────────────────────────┤
        │  R12                   │
        ├────────────────────────┤
        │  R3                    │
        ├────────────────────────┤
        │  R2                    │
        ├────────────────────────┤
        │  R1                    │
        ├────────────────────────┤
        │  R0 (pvParameters)     │  <-- 传递给任务函数的第一个形参！
        ├────────────────────────┤  ==========================================
        │  R11                   │  ▲ 以上 8 个寄存器在异常返回时由 CPU 硬件自动出栈
        ├────────────────────────┤  ------------------------------------------
        │  R10                   │  ▼ 以下 8 个寄存器在 PendSV 中由汇编指令手动出栈
        ├────────────────────────┤
        │  R9                    │
        ├────────────────────────┤
        │  R8                    │
        ├────────────────────────┤
        │  R7                    │
        ├────────────────────────┤
        │  R6                    │
        ├────────────────────────┤
        │  R5                    │
        ├────────────────────────┤
        │  R4                    │
低地址  └────────────────────────┘  <-- pxCurrentTCB->pxTopOfStack (顶层栈指针)
```

在 `PendSV_Handler` 汇编执行切换时：
1. **保存旧任务：** 硬件自动将 R0~R3, R12, LR, PC, xPSR 压入当前任务的 PSP 栈；软件通过汇编指令 `STMDB r0!, {r4-r11}` 手动将 R4~R11 压入栈中，并更新旧任务 TCB 的 `pxTopOfStack`。
2. **加载新任务：** 改变指针指向新任务 TCB；软件汇编指令 `LDMIA r0!, {r4-r11}` 恢复新任务的 R4~R11；最后退出异常，硬件自动从新任务栈中弹出 R0~R3, R12, LR, PC, xPSR，PC 指针自动跳转到新任务代码执行！


## 1.7 内核常用钩子函数 (Hook) 体系

FreeRTOS 在内核关键节点预留了“钩子插槽”，只需将配置宏置 1 并实现函数即可捕获系统状态：

| 钩子函数名称 | 开启配置宏 | 触发时机 | 运行上下文 | 黄金规则与禁忌 |
| :--- | :--- | :--- | :--- | :--- |
| `vApplicationStackOverflowHook` | `configCHECK_FOR_STACK_OVERFLOW` (1或2) | 任务私有栈发生下溢越界 | PendSV 异常上下文 | **绝不能有阻塞行为**，通常打断点或关中断死循环 |
| `vApplicationMallocFailedHook` | `configUSE_MALLOC_FAILED_HOOK` (1) | `pvPortMalloc` 申请失败返回 NULL | 调用 malloc 的当前任务 | 提示堆内存不足，可记录日志并重启 |
| `vApplicationIdleHook` | `configUSE_IDLE_HOOK` (1) | 空闲任务每次执行循环 | 空闲任务 (优先级 0) | **严禁在此函数内执行阻塞 API** (会导致空闲任务卡死) |
| `vApplicationTickHook` | `configUSE_TICK_HOOK` (1) | 每次 SysTick 节拍中断发生时 | SysTick 中断上下文 | 必须极其精简，绝不能写复杂长循环耗费中断时间 |


## 1.8 第1课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：
- [ ] **源码纯净性：** 工程中是否仅添加了 6 个内核核心 `.c` + 1 个 `port.c` + 1 个 `heap_4.c`？
- [ ] **路径优先级：** `..\Inc` 是否置于 Include Paths 首位，确保 `FreeRTOSConfig.h` 被优先加载？
- [ ] **时钟精确匹配：** `configCPU_CLOCK_HZ` 是否与系统时钟主频绝对一致？（若写错会导致延时时间严重偏离）
- [ ] **中断符号接管：** `py32f403_it.c` 中是否已妥善处理 `SVC_Handler` 与 `PendSV_Handler`，避免符号重复定义？
- [ ] **中断优先级生死线：** 外设中断是否遵从“优先级数值 ≥ `MAX_SYSCALL` 才可调用 `xxxFromISR`”的铁律？
- [ ] **SysTick 防护：** `SysTick_Handler` 中是否加入了调度器启动状态检测，防止开机硬故障？
- [ ] **点灯验证：** 编译实现 0 Error / 0 Warning，实机烧录后 LED 以预期频率平稳闪烁。


***

# 第二部分：第2课 — 任务管理

> **核心主题：** 任务生命周期与抢占调度机理 (Task) — RTOS 的灵魂

> **📎 配套代码文件:**
> 
> - `Lesson2_Task_Basic.md` — 基础版：三任务并行 LED 闪烁（入门）
> - `Lesson2_Task_Advanced.md` — 进阶版：任务栈深水线监控 + 结构体全局状态管理（生产级）

***

```
▎ 课程目标： 掌握 FreeRTOS 多任务创建、优先级抢占、延时函数、任务状态机、挂起/恢复、栈高水位监控
▎ 硬件平台： PY32F403-STK (PB13=LED)
▎ 前置成果： 第1课已完成系统移植与单个 LED 心跳任务点亮
```

---

## 2.1 任务的本质：并发世界的“独立执行线程”

在裸机单片机编程中，所有代码都在一个巨大的 `while(1)` 死循环中排队执行；而在 FreeRTOS 中，复杂的系统被拆分为多个**互不相干、彼此独立运行的小程序**，这就是**任务 (Task)**。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 裸机前后台轮询 vs FreeRTOS 抢占式多任务对比                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   裸机 (Super Loop 串行排队):          FreeRTOS (Multi-Task 抢占并发):      │
│   ┌────────────────────────┐           ┌──────────┐ ┌──────────┐ ┌────────┐ │
│   │ while (1) {            │           │ 任务 A   │ │ 任务 B   │ │ 任务 C │ │
│   │     采集传感器();      │           │ (高优先) │ │ (中优先) │ │(低优先)│ │
│   │     处理通信();        │           │    │     │ │    │     │ │   │    │ │
│   │     刷新显示();        │           │    ▼     │ │    ▼     │ │   ▼    │ │
│   │ }                      │           │ 独占 CPU │ │ 独占 CPU │ │独占 CPU│ │
│   └────────────────────────┘           └──────────┘ └──────────┘ └────────┘ │
│                                                                             │
│   • 任何一处 delay 导致全盘卡死         • 逻辑上各自独占 CPU 连续运行        │
│   • 紧急事件只能靠中断打断             • 高优先级任务就绪即刻抢占执行       │
│   • 业务逻辑高度耦合、难以扩展         • 单个任务阻塞时，其他任务照常运行   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.1.1 任务的三大物理组成
一个在 FreeRTOS 中合法存在的任务，在内存中必须包含三项要素：
1. **任务函数体：** 一个死循环或自删除的 C 语言函数；
2. **私有任务栈 (Stack)：** 一片连续的 RAM 空间，保存任务的局部变量、调用嵌套及被打断时的 CPU 寄存器；
3. **任务控制块 (TCB - Task Control Block)：** 内核管理该任务的大脑，记录栈顶指针 (`pxTopOfStack`)、优先级、任务名和状态链表节点。

### 2.1.2 为什么任务函数绝对不能 return？
普通 C 函数执行到花括号末尾时，会通过 `BX LR` 指令返回到调用者；**但 FreeRTOS 任务函数没有“调用者”！**  
任务是由调度器通过上下文切换（恢复 PC 指针）跳入执行的。如果任务函数执行 `return`，CPU 会跳入初始伪造的 `LR` 寄存器地址，即内核的死胡同函数：

```c
/* port.c: 任务意外退出时的内核安全兜底函数 */
static void prvTaskExitError(void)
{
    configASSERT(uxCriticalNesting == ~0UL);  /* 故意触发断言失败 */
    portDISABLE_INTERRUPTS();
    for (;;);                                 /* 关中断死循环，系统直接卡死！ */
}
```

#### 任务函数的正确书写范式

```c
/* ✅ 推荐写法 1: for (;;) 无限循环 (标准常驻任务) */
static void Task_Sensor(void *pvParameters)
{
    for (;;)
    {
        Read_Sensor();
        vTaskDelay(pdMS_TO_TICKS(100));  /* 阻塞出让 CPU */
    }
}

/* ✅ 推荐写法 2: while (1) 无限循环 (与写法 1 完全等价) */
static void Task_Display(void *pvParameters)
{
    while (1)
    {
        Update_Screen();
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

/* ✅ 推荐写法 3: 自删除 (适合“一次性执行”的任务) */
static void Task_SystemInit(void *pvParameters)
{
    Hardware_SelfCheck();
    Load_Configuration();
    
    /* 执行完毕，销毁自身并回收资源，永不返回 */
    vTaskDelete(NULL);
}

/* ❌ 严重错误写法: 函数自然 return */
static void Bad_Task(void *pvParameters)
{
    Do_Something();
    /* 隐式 return 触发 prvTaskExitError() -> 系统崩溃卡死 */
}
```

---

## 2.2 任务创建 API 详解：动态 vs 静态

### 2.2.1 动态创建：`xTaskCreate()`（通用推荐）

动态创建时，任务控制块 (TCB) 与任务栈由 FreeRTOS 堆 (`pvPortMalloc`) 自动在运行时分配。

```c
BaseType_t xReturn = xTaskCreate(
    LED_Task,            /* ① pvTaskCode:    任务函数指针 (入口地址) */
    "LED_Task",          /* ② pcName:        任务名称 (调试跟踪使用，最长 16 字节) */
    128,                 /* ③ usStackDepth:  栈深度 (⚠️ 注意: 单位是 Word/字，128 = 128*4 = 512 字节) */
    (void *)0x1234,      /* ④ pvParameters:  传递给任务函数的入参 (可为 NULL 或任意指针) */
    2,                   /* ⑤ uxPriority:    任务优先级 (0 ~ configMAX_PRIORITIES - 1，数值越大越优先) */
    &xLEDTaskHandle      /* ⑥ pxCreatedTask: 输出型参数，保存任务句柄 (不需要操作该任务时可传 NULL) */
);

/* 必须校验返回值！ */
if (xReturn != pdPASS)
{
    /* 内存不足 (errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY) -> 报警处理 */
}
```

### 2.2.2 静态创建：`xTaskCreateStatic()`（安全关键场景）

在汽车电子、航空或严苛工控领域，为避免堆内存碎片引发内存耗尽风险，通常禁止使用动态内存，改用**静态创建**（所有内存由用户在编译期静态声明）：

```c
/* 1. 静态分配任务栈与 TCB 内存块 */
static StackType_t  xTaskStack[128];       /* 128 字 = 512 字节静态数组 */
static StaticTask_t xTaskTCB;              /* 静态 TCB 结构体 */

/* 2. 静态创建任务 */
TaskHandle_t xHandle = xTaskCreateStatic(
    LED_Task,            /* 任务函数指针 */
    "StaticLED",         /* 任务名 */
    128,                 /* 栈大小 (Words) */
    NULL,                /* 传入参数 */
    2,                   /* 优先级 */
    xTaskStack,          /* 静态栈数组指针 (必须由用户提供) */
    &xTaskTCB            /* 静态 TCB 结构体指针 (必须由用户提供) */
);
```

### 2.2.3 动态创建与静态创建全方位对比

| 对比维度 | 动态创建 (`xTaskCreate`) | 静态创建 (`xTaskCreateStatic`) |
| :--- | :--- | :--- |
| **内存来源** | FreeRTOS 堆 (`configTOTAL_HEAP_SIZE`) | 全局静态数组 (`.bss` 段) |
| **内存确定性** | 存在碎片化风险，可能创建失败返回 NULL | **100% 编译期确定**，只要编译通过永不失败 |
| **代码复杂度** | 极其简单，一行代码搞定 | 需用户手动声明 Stack 数组和 TCB 变量，较繁琐 |
| **系统内存要求** | 依赖 `heap_x.c` 实现 | 无需任何堆内存管理器 |
| **典型适用场景** | 大多数常规项目、快速原型开发 | 医疗、汽车、航空等 SIL/MISRA 安全关键系统 |

---

## 2.3 任务优先级与抢占调度算法

### 2.3.1 抢占式调度核心铁律
FreeRTOS 的抢占式调度器工作原理遵循以下三条核心铁律：
1. **高优先级绝对垄断：** 只要就绪列表中有更高优先级的任务，CPU 会立刻被其抢占，低优先级任务必须无条件让路；
2. **同优先级时间片轮转：** 若存在多个处于同一优先级的就绪任务，且开启了 `configUSE_TIME_SLICING = 1`，每个任务轮流运行 1 个 SysTick 周期（默认 1ms）；
3. **空闲任务永驻垫底：** 空闲任务优先级恒为 `0` (`tskIDLE_PRIORITY`)，仅在系统中所有用户任务都处于阻塞态时运行。

```
优先级抢占与时间片流转时序图:
时间轴 ────────────────────────────────────────────────────────────►

优先级 3 (Task A):  ████████                    ████████ (就绪即抢占)
优先级 2 (Task B):          ████████    ████████
优先级 2 (Task C):                  ████
优先级 0 (Idle):                                        ████████ (全阻塞时运行)
```

### 2.3.2 典型工程任务优先级规划阶梯

优先级的分配切忌随意设定。推荐采用以下**工程分层金字塔**：

| 优先级区间 | 业务类型 | 典型工程代表 | 运行特征 |
| :--- | :--- | :--- | :--- |
| **极高 (`Max - 1`)** | 硬实时保护与突发控制 | 电机紧急刹车、高精度温控、电源故障 | 极少运行，一旦事件到来必须微秒级响应 |
| **较高 (`Max - 2 ~ 3`)** | 高速协议与外设通信 | CAN/串口协议解析、DMA 接收缓冲流 | 事件驱动，通过队列/通知由中断唤醒后快速处理完毕 |
| **中等 (`2 ~ 3`)** | 传感器周期性采集 | ADC 采样滤波、温湿度/气压计读取 | 周期性循环执行，使用延时函数出让 CPU |
| **较低 (`1`)** | 人机交互与后台展示 | 屏幕 UI 渲染刷新、状态日志打印、LED 状态灯 | 对毫秒级实时性不敏感，耗时较长 |
| **最低 (`0`)** | 系统内核空闲任务 | `IDLE Task` | 负责内存回收与低功耗 WFI 休眠 |

---

## 2.4 任务四大状态机（生命周期全景图）

在 FreeRTOS 中，任何一个时刻，任务必定处于以下 **4 种状态之一**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FreeRTOS 任务 4 大状态机转移图                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                      ┌──────────────────────┐                               │
│                      │      运行态 (Running) │ ◄── 独占 CPU (单核系统仅 1 个)│
│                      └──────────┬───────────┘                               │
│                                 │                                           │
│               更高优先级就绪抢占 │ ▲ 被调度器选中运行                        │
│               或时间片轮转让出   │ │ (就绪列表中优先级最高)                  │
│                                 ▼ │                                         │
│                      ┌────────────┴─────────┐                               │
│         ┌──────────► │      就绪态 (Ready)   │                              │
│         │            └──────────────────────┘                               │
│         │                        ▲                                          │
│         │ 延时结束 / 收到信号量   │ vTaskResume() /                         │
│         │ 收到队列数据           │ xTaskResumeFromISR()                    │
│         │                        │                                          │
│  ┌──────┴───────────────┐        │        ┌────────────────────────┐        │
│  │     阻塞态 (Blocked)  │        └─────── │   挂起态 (Suspended)   │        │
│  │ (等待延时/事件超时)  │                 │  (被冻结，完全脱离调度) │        │
│  └──────────────────────┘                 └────────────────────────┘        │
│         ▲                                             ▲                     │
│         │ 调用 vTaskDelay()                           │ 调用                │
│         │ 或调用带超时的 IPC API                      │ vTaskSuspend()      │
│         └─────────────────────────────────────────────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 状态定义与流转要点
- **运行态 (Running)：** 任务当前正在被 CPU 执行。单核单片机在同一瞬时**只能有 1 个任务处于运行态**。
- **就绪态 (Ready)：** 任务一切准备就绪，只要调度器选中即可立刻运行（暂时没运行只是因为有同级或更高优先级的任务正在跑）。
- **阻塞态 (Blocked)：** 任务正在等待某种事件（如等待固定时间 `vTaskDelay`，或等待队列、信号量、事件标志到来）。**处于阻塞态的任务不消耗任何 CPU 算力**。
- **挂起态 (Suspended)：** 任务被调用 `vTaskSuspend()` 显式冷冻。除非其他任务调用 `vTaskResume()`，否则调度器永远忽略该任务。

> 💡 **深入内核：僵尸任务与资源回收**  
> 当任务调用 `vTaskDelete(NULL)` 销毁自身时，任务会被移入“终止列表（Terminated List）”。  
> 任务无法在执行中自己释放自己的栈内存和 TCB！**这项清理工作是由空闲任务 (Idle Task) 负责完成的**。因此，如果系统中有高优先级任务持续空转死等而不阻塞，导致空闲任务永远得不到调度，被删除任务的内存就永远无法回收，最终导致内存泄漏！

---

## 2.5 延时函数的奥秘：相对延时 vs 绝对周期延时

### 2.5.1 相对延时 `vTaskDelay()` 与时间漂移

```c
void vTaskDelay(const TickType_t xTicksToDelay);
```
`vTaskDelay()` 的含义是：**“从本语句被执行的时刻起，让出 CPU 并休眠指定的 Tick 数”**。  
如果用于周期性任务，会因为任务本身的运行耗时和其他高优先级任务的打断，导致**时间累积漂移**：

```
vTaskDelay(10ms) 实际周期漂移示意:

周期 1: [执行任务 2ms] ──► [vTaskDelay 休眠 10ms] ──► 实际总周期 = 12ms
周期 2: [执行任务 2ms] ──► [高优先中断打断 3ms] ──► [vTaskDelay 10ms] ──► 实际总周期 = 15ms
结论: 随着时间推移，任务执行的绝对时刻会不断向后漂移，无法胜任精准周期采样！
```

### 2.5.2 绝对周期延时 `vTaskDelayUntil()`（精准周期推荐）

```c
BaseType_t vTaskDelayUntil(TickType_t *pxPreviousWakeTime, const TickType_t xTimeIncrement);
```
`vTaskDelayUntil()` 的含义是：**“以指定的时间戳为绝对基准点，严格间隔固定 Tick 数唤醒”**。它会自动把任务自身的运行耗时从休眠时间中扣除，彻底消除累积误差！

```c
/* 周期性任务的标准工业级写法 */
static void Task_PeriodicSample(void *pvParameters)
{
    /* 1. 获取当前系统节拍数作为初次基准时间戳 */
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /* 2. 绝对对齐延时: 严格每 100ms 唤醒一次 (无论循环体执行耗时多久) */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100));

        /* 3. 执行周期性采样业务 */
        Sample_Sensors();
        Process_Data();
    }
}
```

```
vTaskDelayUntil(10ms) 绝对对齐时序:

周期 1: [执行 2ms] ──► [自动休眠 8ms]  ──► 实际周期 = 严格 10ms
周期 2: [执行 3ms] ──► [自动休眠 7ms]  ──► 实际周期 = 严格 10ms
```

### 2.5.3 核心工程辨析：高频任务 ≠ 无脑拉高 Tick 频率

有些开发者为了让任务每隔 1ms 或 2ms 执行一次，便盲目把 `configTICK_RATE_HZ` 改成 2000Hz 甚至更高。**这是严重的架构设计误区！**

| 机制 | 为什么不可取？ | 工程正确设计模式 |
| :--- | :--- | :--- |
| **盲目拉高 Tick 频率** | SysTick 中断极其频繁，CPU 大量时间浪费在中断出入栈与调度开销上；更会彻底破坏低功耗 Tickless 休眠。 | **Tick 频率维持 100Hz~1000Hz 即可**，用于调度毫秒级常规任务。 |
| **超高频任务 (<2ms)** | 调度器上下文切换本身耗时数微秒，不适合走纯 Task 轮询调度。 | **使用硬件通用定时器 (TIM) 产生高频中断**；在 ISR 中处理关键逻辑，或通过 `vTaskNotifyGiveFromISR()` 唤醒专职处理任务。 |

---

## 2.6 栈空间管理与高水位监控（系统安全生命线）

**单片机死机崩溃有超过 60% 是由于任务栈溢出踩踏造成的！**

### 2.6.1 任务栈里究竟存了什么？
每个任务的私有栈由低地址向高地址生长（或由高向低，ARM 架构为向下生长 Full Descending）：

```
任务私有栈内存全景分布:

高地址  ┌────────────────────────────────────────────────────────┐
        │ 1. 局部变量 (函数内定义的数组、大型 struct 结构体)      │ ◄── 易爆点！
        ├────────────────────────────────────────────────────────┤
        │ 2. 函数多层嵌套调用帧 (每层嵌套保存返回地址 LR 和 FP) │
        ├────────────────────────────────────────────────────────┤
        │ 3. 上下文切换寄存器保护区 (R0~R3, R12, LR, PC, xPSR +   │
        │    软件保存的 R4~R11，固定占用 64 字节)                │
        ├────────────────────────────────────────────────────────┤
        │ 4. FPU 浮点寄存器区 (S0~S31，若启用硬件浮点额外占用)   │
低地址  └────────────────────────────────────────────────────────┘ ◄── 栈顶指针 (Top of Stack)
```

> 🚨 **栈爆炸的两大元凶：**  
> 1. 在任务函数内部定义大数组（如 `uint8_t buffer[256];`），瞬间吃光栈空间！**大缓冲区应使用 `static` 修饰或动态申请。**  
> 2. 深度函数递归调用或调用了未优化的 `sprintf/printf`（部分 C 库的浮点格式化会一次性占用数百字节栈）。

### 2.6.2 诊断利器：`uxTaskGetStackHighWaterMark()`

内核提供了极具价值的“深水线（高水位）”监测 API。该函数返回**自该任务创建以来，私有栈历史上距离溢出最近时，所剩余的最小空闲栈空间（单位：Words）**。

```c
/* 获取任务历史剩余的最小栈字数 */
UBaseType_t uxHighWaterMark = uxTaskGetStackHighWaterMark(xMyTaskHandle);
```

#### 栈健康度判定标准（黄金 30% 冗余法则）
- **`剩余 > 50%`：** 栈分配过度奢靡，浪费了珍贵的 SRAM，可适当缩减；
- **`20% ~ 40%`：** **黄金健康区间**，既保障了安全裕量，又没有过度浪费；
- **`剩余 < 15%`：** **极度危险！** 稍有深层嵌套或中断到来就会发生栈溢出崩溃，必须立刻加大栈分配。

### 2.6.3 工业级实战：独立栈监控任务架构

在大型工程开发与压力测试期间，推荐编写一个独立的监控任务，周期性巡检并输出全系统任务栈余量：

```c
static void Task_StackMonitor(void *pvParameters)
{
    (void)pvParameters;
    
    for (;;)
    {
        /* 每 5 秒巡检一次系统状态 */
        vTaskDelay(pdMS_TO_TICKS(5000));

        /* 测量各核心任务的高水位线 */
        UBaseType_t hwmLed = uxTaskGetStackHighWaterMark(hTaskLED);
        UBaseType_t hwmComm = uxTaskGetStackHighWaterMark(hTaskComm);

        /* 若剩余少于 20 Words (80 字节)，触发告警 */
        if (hwmLed < 20 || hwmComm < 20)
        {
            // Print_Warning("CRITICAL: Stack near overflow!");
        }
    }
}
```

---

## 2.7 任务挂起、恢复与删除操作规范

### 2.7.1 任务挂起与恢复
挂起意味着将任务“冷冻”，彻底退出调度队列，不再分配任何 CPU 时间：

```c
TaskHandle_t xHandleWorker;

/* 1. 挂起别人 (传入句柄) */
vTaskSuspend(xHandleWorker);

/* 2. 自我挂起 (传 NULL，任务在当前语句立刻停止运行) */
vTaskSuspend(NULL);

/* 3. 在普通任务中唤醒挂起的任务 */
vTaskResume(xHandleWorker);

/* 4. 在中断服务函数 ISR 中唤醒挂起的任务 */
BaseType_t xYieldRequired = xTaskResumeFromISR(xHandleWorker);
portYIELD_FROM_ISR(xYieldRequired);
```

### 2.7.2 任务销毁 `vTaskDelete()`

```c
/* 1. 销毁自身 (最常见于初始化任务) */
vTaskDelete(NULL);  /* 此行之后绝不会执行 */

/* 2. 销毁其他任务 */
vTaskDelete(xHandleWorker);
```

> ⚠️ **内存自理警示：**  
> `vTaskDelete()` 只负责回收内核为该任务分配的 TCB 与栈空间。**如果任务内部自己调用了 `malloc()` 申请了内存，或创建了队列/信号量，内核绝不会自动释放！** 必须在销毁前手动释放，否则将永久内存泄漏。

---

## 2.8 空闲任务 (Idle Task) 与空闲钩子 (Idle Hook)

当启动调度器 `vTaskStartScheduler()` 时，FreeRTOS 会自动在后台创建一个优先级最低 (`0`) 的**空闲任务 (Idle Task)**。

### 空闲任务两大神圣职责
1. **回收被删除任务的内存**（执行 `prvCheckTasksWaitingTermination`）；
2. **没有活干时执行低功耗等待指令**（进入 `WFI` - Wait For Interrupt，休眠节能）。

### 空闲钩子函数 `vApplicationIdleHook()`
若在 `FreeRTOSConfig.h` 中配置 `#define configUSE_IDLE_HOOK 1`，空闲任务每次循环都会调用此函数：

```c
void vApplicationIdleHook(void)
{
    /* 可以执行简单的后台状态统计，或调用 __WFI() 降低功耗 */
    __WFI();
}
```

> 🔴 **空闲钩子第一天条：**  
> **空闲钩子函数中绝对不能调用任何会发生阻塞的 API**（如 `vTaskDelay` 或带有超时时间的 `xQueueReceive`）！  
> 如果空闲任务被阻塞，整个系统在没有其他就绪任务时将陷入无法调度的死锁状态！

---

## 2.9 实战：三任务并行 LED 工程与波形观测

本节展示最经典的 3 任务并行系统（详见配套代码 `Lesson2_Task_Basic.md`）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 三任务并行调度架构流程图                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  main()                                                                     │
│    │                                                                        │
│    ├── HAL_Init() & 硬件时钟配置                                            │
│    ├── GPIO 配置 (PB13, PA1, PC13 分别驱动 LED1, LED2, LED3)                │
│    │                                                                        │
│    ├── xTaskCreate(Task_FastLED,   "Fast",   128, NULL, 3, NULL)            │
│    ├── xTaskCreate(Task_MediumLED, "Medium", 128, NULL, 2, NULL)            │
│    ├── xTaskCreate(Task_SlowLED,   "Slow",   128, NULL, 1, NULL)            │
│    │                                                                        │
│    └── vTaskStartScheduler() ──► 开启多任务世界                             │
│                                                                             │
│  并行运行效果 (逻辑分析仪波形观测):                                         │
│                                                                             │
│  FastLED (Pri 3):   ┌──┐  ┌──┐  ┌──┐  ┌──┐ (100ms 翻转周期, 5Hz)           │
│  (PB13)             └──┘  └──┘  └──┘  └──┘                                  │
│                                                                             │
│  MediumLED (Pri 2): ┌──────┐    ┌──────┐   (300ms 翻转周期, 1.67Hz)         │
│  (PA1)              └──────┘    └──────┘                                    │
│                                                                             │
│  SlowLED (Pri 1):   ┌──────────┐           (500ms 翻转周期, 1Hz)            │
│  (PC13)             └──────────┘                                            │
│                                                                             │
│  三个任务彼此独立运行，各自拥有精确的时序，完全打破了单片机裸机的延时阻塞制约！│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 任务管理开发常见陷阱与避坑防线

| 易错陷阱 | 危害与症状 | 正确工程做法 |
| :--- | :--- | :--- |
| **任务中写死硬延时 `delay_ms`** | 霸占 CPU 空转死等，同级和低级任务无法运行 | 改用 `vTaskDelay()`，让出 CPU 给其他任务 |
| **忘记判断 `xTaskCreate` 返回值** | 堆内存不足时任务静默创建失败，功能离奇失效 | 必须使用 `if (xReturn == pdPASS)` 结合断言检查 |
| **任务栈设为 `configMINIMAL_STACK_SIZE`** | 复杂业务中局部变量增加导致内存踩踏 | 该宏仅供空闲任务使用，用户任务建议实测后留出 30% 裕量 |
| **高优先级任务死循环无阻塞** | 低优先级任务和空闲任务永久饥饿 | 任务体内部必须有引发阻塞的 API 出让 CPU |
| **误用 `vTaskDelay(0)`** | 不代表延时 0ms，而是强制让出时间片给同级任务 | 仅在需要协同式让出时主动调用 |

---

## 2.10 第2课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **任务模型：** 能否向他人讲清任务函数与普通 C 函数的三大本质区别？
- [ ] **动态与静态：** 能否根据工控或消费级场景，正确选择 `xTaskCreate` 或 `xTaskCreateStatic`？
- [ ] **状态机闭环：** 能否准确画出任务从就绪、运行、阻塞到挂起的状态流转条件？
- [ ] **延时精准度：** 能否解释为何周期性任务必须采用 `vTaskDelayUntil` 而非 `vTaskDelay`？
- [ ] **栈深水线：** 能否使用 `uxTaskGetStackHighWaterMark()` 估算任务的栈余量并调整安全阈值？
- [ ] **空闲任务铁律：** 是否牢记空闲钩子函数中绝不能调用任何阻塞 API？
- [ ] **实操验证：** 编译烧录 `Lesson2_Task_Basic`，通过 LED 闪烁或逻辑分析仪验证多任务并发。

***

# 第二部分：第3课 — 队列

> **核心主题：** 任务间通信基石 (Queue) — 数据管道与事件驱动并发
> 
> **📎 配套代码文件:**
> - `Lesson3_Queue_Basic.md` — 基础版：生产者-消费者 (Producer-Consumer) 队列实战
> - `Lesson3_Queue_Advanced.md` — 进阶版：单槽邮箱 (Mailbox) + 队列集 (Queue Set) + 队列注册表 (Registry)

***

```
▎ 课程目标： 掌握 xQueueCreate / xQueueSend / xQueueReceive，理解数据值拷贝、阻塞超时与生产者-消费者模型
▎ 硬件平台： PY32F403-STK (PB13=LED1, PA1=LED2, PC13=LED3)
▎ 前置成果： 第2课已掌握多任务创建、状态机迁移与栈高水位监控
```

---

## 3.1 队列的本质：FreeRTOS 的安全“数据管道”

在多任务操作系统中，如果两个任务直接通过全局变量传递数据，很容易发生**读写冲突（竞态条件）**，且接收方只能通过死循环轮询去检查变量变化，白白浪费 CPU 算力。  
FreeRTOS 提供的**消息队列 (Queue)** 就是为了彻底解决这一痛点而生的核心基础设施。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FreeRTOS 队列数据通道模型                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   任务 A (生产者)               FreeRTOS 队列 (环形缓冲区)       任务 B (消费者)   │
│   ┌──────────────┐             ┌───┬───┬───┬───┐             ┌──────────────┐   │
│   │ 产生数据帧   │ ──深拷贝──► │D0 │D1 │D2 │   │ ──深拷贝──► │ 阻塞等待数据 │   │
│   │              │   (memcpy)  └───┴───┴───┴───┘   (memcpy)  │ 取出立刻执行 │   │
│   └──────────────┘               ▲           ▲               └──────────────┘   │
│                                  │           │                                  │
│                             pcReadFrom   pcWriteTo                              │
│                                                                             │
│   三大核心机制：                                                             │
│   1. FIFO 先进先出：先入队的数据先被消费 (亦支持紧急数据插队至队首)          │
│   2. 深拷贝传递 (By Value)：发送时整块内存拷贝，发送与接收方持有独立副本     │
│   3. 双向阻塞等待：队列空时接收者休眠出让 CPU，队列满时发送者休眠等待空间    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1.1 队列底层控制结构 (`Queue_t`)
队列在内核底层由一个**环形缓冲区 (Ring Buffer)** 和**两个任务挂起链表**组成：

```c
/* 队列控制块核心成员简述 (queue.c) */
typedef struct QueueDefinition {
    int8_t *pcHead;                      /* 指向环形缓冲区内存首地址 */
    int8_t *pcWriteTo;                   /* 下一个数据写入的目标位置 */
    int8_t *pcReadFrom;                  /* 最后一个数据被读出的位置 */
    UBaseType_t uxMessagesWaiting;       /* 当前队列中积压的有效消息条数 */
    UBaseType_t uxLength;                /* 队列最大容量 (深度，能容纳多少个 item) */
    UBaseType_t uxItemSize;              /* 每个 item 的单条数据字节大小 */
    List_t xTasksWaitingToSend;          /* 因队列已满而阻塞休眠的发送任务链表 */
    List_t xTasksWaitingToReceive;       /* 因队列为空而阻塞休眠的接收任务链表 */
} xQUEUE;
```

---

## 3.2 队列核心 API 与调用契约

FreeRTOS 队列 API 设计极其规整，任务上下文与中断上下文有严格的区分：

| 操作类型 | 任务上下文 API | 中断服务程序专属 API (ISR) | 核心行为说明 |
| :--- | :--- | :--- | :--- |
| **动态创建** | `xQueueCreate(length, itemSize)` | ❌ 仅限任务中创建 | 从 FreeRTOS 堆中动态分配控制块与缓冲区 |
| **尾部发送** | `xQueueSend(q, &item, timeout)` | `xQueueSendFromISR(q, &item, &woken)` | 标准 FIFO 模式，数据追加到队尾 (同 `xQueueSendToBack`) |
| **头部插队** | `xQueueSendToFront(q, &item, timeout)` | `xQueueSendToFrontFromISR(...)` | 紧急数据特权，直接插入到队头，优先被消费 |
| **阻塞接收** | `xQueueReceive(q, &buf, timeout)` | `xQueueReceiveFromISR(q, &buf, &woken)` | 从队首取出一条数据并从队列中抹去该数据 |
| **只看不取** | `xQueuePeek(q, &buf, timeout)` | `xQueuePeekFromISR(q, &buf)` | 拷贝出队首数据，但**不从队列中删除** |
| **覆盖写入** | `xQueueOverwrite(q, &item)` | `xQueueOverwriteFromISR(...)` | 仅限深度=1 的单槽邮箱，满时直接覆盖旧值，永不阻塞 |
| **状态查询** | `uxQueueMessagesWaiting(q)` | `uxQueueMessagesWaitingFromISR(q)` | 返回当前队列积压的消息条数 |
| **清空/删除**| `xQueueReset(q)` / `vQueueDelete(q)` | ❌ | 清空队列所有数据 / 删除队列释放全部内存 |

---

## 3.3 核心机理：事件驱动阻塞与实时唤醒时序

初学者最常犯的错误，就是拿裸机思维看待队列。队列不是普通的数组，它的精髓在于**事件驱动的实时阻塞与唤醒**。

### 3.3.1 队列阻塞 vs `vTaskDelay()` 的本质区别

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 盲目盲等 vs 事件驱动精准唤醒对比                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 方式 1: vTaskDelay(100ms) 轮询 ]                                         │
│  • 任务不管有没有数据，盲目睡 100ms。                                       │
│  • 即使数据在第 1ms 就到了，也必须白白等满 100ms 才能被处理 (响应严重滞后)！ │
│                                                                             │
│  [ 方式 2: xQueueReceive(q, &buf, portMAX_DELAY) 事件驱动 ]                 │
│  • 队列为空时，任务立刻休眠出让 CPU，不浪费半点算力；                       │
│  • 一旦生产者把数据压入队列，内核在【同一个微秒内】立刻唤醒接收任务！       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3.2 队列即刻唤醒与抢占时序全景

假设消费者任务优先级为 `3`，生产者任务优先级为 `2`：

```
时间轴 ──────────────────────────────────────────────────────────────────────►

消费者 (Pri 3):  ██████ (调 xQueueReceive 发现队列空，立刻进入 Blocked 态休眠)
                 ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ██████████ (立地抢占执行!)
                                                           ▲
生产者 (Pri 2):        ██████████████████████████████████  │
                                              │            │
                                              └─► xQueueSend() 入队成功
                                                  内核发现 Pri 3 的消费者正在等待！
                                                  同一次 API 调用内部立刻触发 PendSV 抢占！
```

> 💡 **实时系统的基石：**  
> 数据的“入队”与等待任务的“就绪唤醒”发生在**同一个 `xQueueSend()` 函数调用内部**！绝不是等生产者时间片跑完了才被动切换。

---

## 3.4 经典架构设计：生产者-消费者 (Producer-Consumer) 模型

这是嵌入式多任务系统中最经典的解耦设计模式：  
一个任务（或中断）专门负责**采集/接收数据（生产者）**，另一个任务专门负责**解析/控制硬件（消费者）**。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  生产者-消费者解耦系统架构设计                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────┐                    ┌──────────────────────────┐  │
│  │ CmdSender (生产者)    │                    │ CmdHandler (消费者)      │  │
│  │ 优先级: 2 (中)        │    命令队列 (深度8)│ 优先级: 3 (高)           │  │
│  │                       │    ┌───┬───┬───┐   │                          │  │
│  │ 周期性生成控制命令:   │ ─► │cmd│cmd│   │ ─►│ xQueueReceive 阻塞等待   │  │
│  │ {led_id, count, speed}│    └───┴───┴───┘   │ 收到命令立刻控制硬件翻转 │  │
│  └───────────────────────┘                    └──────────────────────────┘  │
│                                                                             │
│  黄金法则 1: 消费者优先级 (3) ≥ 生产者优先级 (2)                             │
│  • 保证数据一旦生产出来，立刻被高优先级的消费者取走并执行；                  │
│  • 队列中通常只积压 0~1 条数据，永远不会堆死溢出。                          │
│                                                                             │
│  黄金法则 2: 队列深度的工程物理意义                                         │
│  • 深度 = 8 不是为了“攒够 8 条才处理”，而是作为【缓冲波谷】；               │
│  • 当消费者因为执行复杂长操作短时间无法取数据时，队列为生产者提供安全缓冲。 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4.1 数据载荷设计：传值 (By Value) vs 传指针 (By Pointer)

这是所有新手最容易踩翻翻车的一道分水岭：

#### 模式 A：传值 (Pass by Value) —— 小数据无脑首选（推荐 ≤ 32 字节）
发送时将结构体数据完整拷贝进队列缓冲区；接收时再完整拷贝出来。

```c
/* 紧凑型命令结构体 (总共仅 4 字节) */
typedef struct {
    uint8_t  led_id;     /* 目标 LED 编号: 0=PB13, 1=PA1, 2=PC13 */
    uint8_t  count;      /* 翻转闪烁次数: 1~5 */
    uint16_t delay_ms;   /* 翻转时间间隔: 50~500 ms */
} LedCmd_t;
```
- **优点：** 绝对安全！发送方的变量哪怕是局部变量或离开作用域，接收方拿到的都是一份完全独立的内存快照，**天然免疫并发踩踏与野指针**。
- **缺点：** 每次入队出队都要 `memcpy`，如果结构体达上千字节，会拖慢系统速度。

#### 模式 B：传指针 (Pass by Pointer) —— 大块数据零拷贝（如图像、网络大包）
队列里面只存放一个 4 字节的指针地址：`sizeof(uint8_t *)`。

```c
/* ❌ 致命错误典范 (新手必踩): 传递栈上局部变量的指针 */
void Bad_Producer(void)
{
    uint8_t local_buf[128];
    Read_Sensor(local_buf);
    uint8_t *ptr = local_buf;
    xQueueSend(xQueue, &ptr, 0); 
    /* 函数退出! local_buf 内存被回收! 接收方拿到野指针，系统必定随机崩溃! */
}
```
- **正确做法：** 传指针时，内存必须来自**全局静态缓冲区池**，或通过 `pvPortMalloc()` 动态申请；并且**在接收方消费完毕后，必须由接收方负责释放该指针**（明确所有权转移契约）。

---

## 3.5 极简代码实战与时序解析

### 3.5.1 全局状态聚合规范
工业级代码坚决反对全局变量满天飞。推荐将队列句柄统一收敛到一个全局静态结构体 `g` 中：

```c
/* 统一管理系统全局内核对象 */
static struct {
    QueueHandle_t xCmdQueue;  /* LED 命令队列句柄 */
} g;
```

### 3.5.2 完整交互代码（生产者 + 消费者）

```c
/* 队列深度与大小定义 */
#define CMD_QUEUE_LEN       ( 8 )
#define CMD_QUEUE_ITEM_SZ   ( sizeof( LedCmd_t ) )

/* 1. 主函数初始化与创建 */
void APP_SystemInit(void)
{
    /* 创建队列并强制断言检查 (杜绝堆内存不足导致的静默失败) */
    g.xCmdQueue = xQueueCreate(CMD_QUEUE_LEN, CMD_QUEUE_ITEM_SZ);
    configASSERT(g.xCmdQueue != NULL);

    /* 消费者任务: 优先级 3 (高) */
    xTaskCreate(Task_CmdHandler, "Handler", 128, NULL, 3, NULL);

    /* 生产者任务: 优先级 2 (中) */
    xTaskCreate(Task_CmdSender,  "Sender",  128, NULL, 2, NULL);
}

/* 2. 生产者任务实现 (周期性发射命令) */
static void Task_CmdSender(void *pvParameters)
{
    (void)pvParameters;
    static const LedCmd_t s_cmdList[] = {
        {0, 3, 100},  /* PB13 快闪 3 次 (100ms) */
        {1, 2, 200},  /* PA1  中速 2 次 (200ms) */
        {2, 1, 300}   /* PC13 慢闪 1 次 (300ms) */
    };
    uint8_t idx = 0;
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        LedCmd_t currentCmd = s_cmdList[idx];
        idx = (idx + 1) % 3;

        /* 发送数据到队尾: 若队列满，最多容忍阻塞等待 100ms */
        BaseType_t xStatus = xQueueSend(g.xCmdQueue, &currentCmd, pdMS_TO_TICKS(100));
        if (xStatus != pdPASS)
        {
            /* 队列溢出告警 (说明消费者卡死或严重过载) */
        }

        /* 保持绝对周期: 每 500ms 发送一条新命令 */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(500));
    }
}

/* 3. 消费者任务实现 (事件驱动控制硬件) */
static void Task_CmdHandler(void *pvParameters)
{
    (void)pvParameters;
    LedCmd_t recvCmd;

    for (;;)
    {
        /* 关键点: portMAX_DELAY 挂起等待
         * 队列无数据时本任务彻底休眠，0% CPU 占用；数据到来微秒级唤醒！
         */
        if (xQueueReceive(g.xCmdQueue, &recvCmd, portMAX_DELAY) == pdPASS)
        {
            /* 按照收到的指令序列翻转指定的 LED */
            for (uint8_t i = 0; i < recvCmd.count; i++)
            {
                APP_ToggleLed(recvCmd.led_id);
                vTaskDelay(pdMS_TO_TICKS(recvCmd.delay_ms));
            }
        }
    }
}
```

---

## 3.6 队列三大高级工程扩展模式

### 3.6.1 模式一：单槽邮箱 (Mailbox) — `xQueueOverwrite()`

在许多工程场景中，数据代表的是**“当前瞬时状态”**而非“必须按部就班执行的事件”：
- 比如当前环境温度、油门踏板当前开度、陀螺仪姿态角；
- 历史旧读数没有意义，我们只关心**最新的那一个数值**。

```
普通队列 (深度=8): [老数据1] ──► [老数据2] ──► [新数据3] (按部就班排队消费)
邮箱模式 (深度=1): [最新数据] (新数据强行覆盖旧数据，读出来的永远是实时值！)
```

```c
/* 邮箱模式标准配置: 深度必须严格为 1 */
QueueHandle_t xMailbox = xQueueCreate(1, sizeof(float));

/* 生产者: 无论消费者读没读，强行写入最新采样值，永不阻塞 */
float currentTemp = 26.5f;
xQueueOverwrite(xMailbox, &currentTemp);

/* 消费者: 读取最新值 (超时时间设 0，立即返回不阻塞) */
float latestTemp;
if (xQueuePeek(xMailbox, &latestTemp, 0) == pdPASS)
{
    /* 成功读取到最新温度 */
}
```

### 3.6.2 模式二：队列集 (Queue Set) — `xQueueSelectFromSet()`

在复杂业务中，一个网关任务往往需要**同时监听多个数据源**（如同时监听串口队列、CAN 总线队列、按键事件信号量）。  
如果直接调用 `xQueueReceive`，任务就会在第一个队列上死等，导致另外两个队列哪怕爆仓也无法感知！  
**队列集 (Queue Set)** 类似于 Linux 的 `select()/epoll()` 多路复用机制：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         队列集多路复用监听模型                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   串口队列 ───► ┐                                                           │
│   CAN 队列 ───► ┼──► 统一加入队列集 ──► 网关任务: xQueueSelectFromSet()     │
│   按键信号量 ─► ┘   (xQueueSet)         (哪个有数据就返回哪个的句柄)        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```c
/* 1. 创建子队列并计算总深度 */
QueueHandle_t qUart = xQueueCreate(4, sizeof(Msg_t));
QueueHandle_t qCan  = xQueueCreate(4, sizeof(Msg_t));

/* 2. 创建队列集: 深度必须等于所有成员队列深度之和 (4 + 4 = 8) */
QueueSetHandle_t xSet = xQueueCreateSet(8);
xQueueAddToSet(qUart, xSet);
xQueueAddToSet(qCan,  xSet);

/* 3. 网关任务多路复用监听 */
for (;;)
{
    /* 阻塞在集合上: 任何一个队列有数据，立即返回该队列的句柄 */
    QueueSetMemberHandle_t activeQ = xQueueSelectFromSet(xSet, portMAX_DELAY);

    if (activeQ == qUart)
    {
        Msg_t msg;
        xQueueReceive(qUart, &msg, 0);  /* 此时读取绝对不会阻塞 */
        Process_Uart(&msg);
    }
    else if (activeQ == qCan)
    {
        Msg_t msg;
        xQueueReceive(qCan, &msg, 0);
        Process_Can(&msg);
    }
}
```

### 3.6.3 模式三：队列注册表 (Queue Registry) — 调试利器

在大型工程中，可能有几十个队列。当通过 Keil 或实时监控工具调试时，看到的只是一串无意义的十六进制地址（如 `0x200014B0`）。  
队列注册表允许给每个队列贴上**人类可读的字符串调试标签**：

```c
/* 在 FreeRTOSConfig.h 中启用: #define configQUEUE_REGISTRY_SIZE 5 */

/* 为队列绑定调试名称 (名称必须是全局常量字符串) */
vQueueAddToRegistry(g.xCmdQueue, "LED_CmdQueue");

/* 调试器或系统监控任务可通过 pcQueueGetName 获取名称 */
printf("Queue [%s] current load: %lu\r\n", 
       pcQueueGetName(g.xCmdQueue), 
       uxQueueMessagesWaiting(g.xCmdQueue));
```

---

## 3.7 内存开销、常见陷阱与避坑防线

### 3.7.1 队列的 RAM 开销精确计算
很多工程师以为队列只占用数据数组大小，这是极其危险的预算误区！

$$\text{总 RAM 开销} = \text{Queue\_t 控制块大小} (约 76 \sim 96 字节) + (\text{队列深度} \times \text{单项大小})$$

以深度为 `8`，单项为 `4 字节` 为例：
总开销 $\approx 80 + (8 \times 4) = 112$ 字节。  
**结论：** 队列控制块自身占用了绝大部分内存！如果仅仅是为了通知一个标志，千万不要滥用队列，而应选用开销极低的**信号量**或**任务通知**。

### 3.7.2 队列开发 5 大致命陷阱

| 致命陷阱 | 严重后果 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **`itemSize` 误写成指针大小** | 例如写成 `sizeof(&MyStruct)`，只拷贝了 4 字节，数据被截断损坏 | 严格使用 `sizeof(MyStruct_t)`，创建时使用静态断言检查 |
| **向队列发送栈局部变量指针** | 函数返回后局部变量被销毁覆盖，接收方读取发生野指针崩溃 | 小数据坚持**按值拷贝**；大数据传指针必须严格保证堆内存所有权交接 |
| **在中断 ISR 中使用阻塞超时** | 中断上下文严禁休眠，否则直接触发内核严重断言或卡死 | 中断中只能调用 `xQueueSendFromISR`，且超时参数绝不可传 |
| **消费者优先级低于生产者** | 生产速度大于消费速度，队列瞬间被打满丢包 | 消费者优先级设计为大于或等于生产者，做到“随到随清” |
| **忽略创建失败返回值** | 堆内存耗尽时静默失败，后续所有 Send/Receive 访问空指针 HardFault | 创建后必须跟进 `configASSERT(q != NULL)` 强制暴露隐患 |

---

## 3.8 第3课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **深拷贝本质：** 能否向他人解释为何队列按值拷贝能够天然保证线程安全性？
- [ ] **事件驱动响应：** 能否讲清为何 `xQueueReceive(q, buf, portMAX_DELAY)` 比 `while(1) + delay` 高效无数倍？
- [ ] **优先级配置：** 为什么生产者-消费者架构中，消费者优先级通常应当大于或等于生产者？
- [ ] **数据传递选型：** 面对 16 字节的数据帧，你为什么选择直接传值？面对 2KB 的相机图像，你会如何安全地传指针？
- [ ] **邮箱覆盖模式：** 为什么传感器采样状态更新推荐使用 `xQueueOverwrite()`？此时队列深度必须设为几？
- [ ] **多路复用场景：** 当一个任务需要同时监听按键和网络数据时，如何使用队列集 `xQueueCreateSet` 实现？
- [ ] **实操验证：** 编译烧录 `Lesson3_Queue_Basic`，通过逻辑分析仪或断点观测命令在队列中的流动。

***

# 第二部分：第4课 — 信号量

> **核心主题：** 任务同步与资源互斥 (Semaphore & Mutex) — 同步原语深度解析
> 
> **📎 配套代码文件:**
> - `Lesson4_Semaphore_Basic.md` — 基础版：二值信号量 + 计数信号量 + 互斥锁（3合1实战）
> - `Lesson4_Semaphore_Advanced.md` — 进阶版：Dijkstra 有界缓冲区 (Bounded Buffer) 生产者-消费者实战

***

```
▎ 课程目标： 掌握 xSemaphoreCreateBinary / xSemaphoreCreateCounting / xSemaphoreCreateMutex
            透彻理解二值、计数与互斥锁的本质区别，攻克优先级反转，掌握 GateKeeper 与 Bounded Buffer 模式
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 前置成果： 第3课已掌握队列数据管道、深拷贝机制与阻塞超时
```

---

## 4.1 信号量的物理本质：从铁路信号到内核队列

“信号量 (Semaphore)”一词最早源于 19 世纪的铁路臂板信号机：
- 🚦 **信号臂举起 (Green / Give / V操作)：** 表示铁轨空闲，列车可以通行；
- 🚦 **信号臂放下 (Red / Take / P操作)：** 表示列车驶入，铁轨被占用，后续列车必须停车排队。

荷兰计算机科学家 Dijkstra 将其抽象为计算机并发控制的核心工具：
- **P 操作 (Proberen / 尝试)：** 获取资源，若可用则计数值减 1；若不可用（值为 0）则任务挂起等待；
- **V 操作 (Verhogen / 增加)：** 释放资源，计数值加 1，唤醒一个正在等待该资源的任务。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FreeRTOS 信号量底层大一统真相                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   在 FreeRTOS 中，semphr.h 是一个【纯宏定义文件】，内核里甚至没有 semphr.c！   │
│   所有信号量在底层完全基于【消息队列 (Queue_t)】实现，只是承载的数据大小为 0:     │
│                                                                             │
│   ┌────────────────┬──────────────────────┬───────────────────────────────┐ │
│   │  信号量类型    │  底层队列初始化参数  │  核心物理本质                 │ │
│   ├────────────────┼──────────────────────┼───────────────────────────────┤ │
│   │  二值信号量    │  Queue(深度=1, 初始=0)│  0 或 1，无所有权，无数据流   │ │
│   │  计数信号量    │  Queue(深度=N, 初始=M)│  0 ~ N，无所有权，管理资源池  │ │
│   │  互斥锁 Mutex  │  Queue(深度=1, 初始=1)│  带【任务所有权】+【优先级继承】│ │
│   │  递归互斥锁    │  特殊 Mutex + 嵌套计数│  允许同一任务重复 Take 多次   │ │
│   └────────────────┴──────────────────────┴───────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4.2 信号量核心 API 矩阵与调用契约

FreeRTOS 信号量操作宏与队列一一对应，任务上下文与 ISR 上下文有着极其严格的物理边界：

| 操作类型 | 任务上下文 API | 中断服务程序专属 API (ISR) | 关键行为与约束 |
| :--- | :--- | :--- | :--- |
| **创建二值信号量** | `xSemaphoreCreateBinary()` | ❌ 仅限任务 | 创建后计数值默认为 **0**（必须先 Give 才能 Take） |
| **创建计数信号量** | `xSemaphoreCreateCounting(max, init)` | ❌ 仅限任务 | 指定最大配额 `max` 与初始可用配额 `init` |
| **创建互斥锁** | `xSemaphoreCreateMutex()` | ❌ 仅限任务 | 初始状态默认为**可用 (1)**，带优先级继承属性 |
| **创建递归互斥锁** | `xSemaphoreCreateRecursiveMutex()` | ❌ 仅限任务 | 允许持有者重复获取，必须配对递归 Give |
| **获取信号量 (P)** | `xSemaphoreTake(sem, timeout)` | `xSemaphoreTakeFromISR(...)` | 计数值减 1；若为 0，任务按超时阻塞休眠 |
| **释放信号量 (V)** | `xSemaphoreGive(sem)` | `xSemaphoreGiveFromISR(...)` | 计数值加 1；唤醒等待链表中的最高优先级任务 |
| **查询当前计数值** | `uxSemaphoreGetCount(sem)` | `uxSemaphoreGetCountFromISR(sem)` | 返回当前可用资源数 |
| **销毁释放内存** | `vSemaphoreDelete(sem)` | ❌ 仅限任务 | 释放底层的队列控制块 RAM |

> 🚨 **第一天条：互斥锁 (Mutex) 绝对严禁在中断 (ISR) 中使用！**  
> 互斥锁的核心机制是“任务所有权”和“优先级继承”。中断服务程序不是任务，没有 TCB 任务控制块，也不存在任务优先级，在 ISR 中获取或释放互斥锁在逻辑上是完全荒谬的！**中断与任务的同步只能使用二值信号量或计数信号量。**

---

## 4.3 三驾马车深度剖析：二值 vs 计数 vs 互斥

### 4.3.1 二值信号量 (Binary Semaphore)：“无记名令牌”与事件同步
- **本质特征：** 计数值只能是 `0` 或 `1`。它没有所有权概念，谁都可以 Give，谁都可以 Take。
- **经典场景：中断向任务打信号（ISR ➔ Task 同步）。**
  - 比如 UART 收到一帧数据，触发中断；
  - 中断服务程序调用 `xSemaphoreGiveFromISR()` 扔出一枚令牌；
  - 正在调用 `xSemaphoreTake()` 阻塞等待的解析任务瞬间被唤醒，开始处理数据；
  - 任务处理完毕后再次进入阻塞等待下一枚令牌。

### 4.3.2 计数信号量 (Counting Semaphore)：“停车位计数”与资源池配额
- **本质特征：** 计数值范围为 `0 ~ max`。同样无所有权。
- **经典场景：有限硬件/内存资源池访问控制。**
  - 假设系统拥有 3 个空闲 DMA 发送通道；
  - 初始化创建计数信号量：`xSemaphoreCreateCounting(3, 3)`；
  - 任何任务发起 DMA 发送前，必须先 `xSemaphoreTake()` 申请一个“车位”；
  - 申请成功后计数减 1；当 3 个通道全被占满（计数值为 0）时，第 4 个任务自动阻塞排队；
  - DMA 传输完成中断中调用 `xSemaphoreGiveFromISR()` 归还车位，唤醒排队任务。

### 4.3.3 互斥锁 (Mutex)：“实名制门禁卡”与资源独占
- **本质特征：** 初始可用值为 1。具有两大与生俱来的专属特权：
  1. **所有权检查 (Ownership)：** 只有成功 Take 锁的任务，才被允许 Give 释放锁；其他任务若试图释放不属于自己的锁，直接触发断言报错！
  2. **优先级继承 (Priority Inheritance)：** 能够动态提升持有者的优先级，彻底解决优先级反转难题。
- **经典场景：保护独占式硬件外设（如单一串口打印、I2C 总线读写）。**

### 4.3.4 灵魂拷问：为什么二值信号量绝不能用来做互斥保护？
很多新手会问：“二值信号量的值也是 0 和 1，为什么不能当锁用？”  
**因为二值信号量是‘无记名令牌’，不查身份证，在多任务环境下必定翻车！**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 二值信号量做互斥必定导致的数据踩踏事故                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   时间轴 ────────────────────────────────────────────────────────►          │
│                                                                             │
│   Task A (Pri 2): Take(成功) ═══════ 正在写 UART ══════════════             │
│                   "我正在独占串口!"  ████████████████████                   │
│                                                                             │
│   Task B (Pri 3):            误调用了一句 Give!                             │
│                              "我没拿锁，但我随手扔了个令牌"                 │
│                              二值信号量不查身份: 令牌值瞬间变回 1!          │
│                                                                             │
│   Task C (Pri 4):                   Take(成功!) "门开着，我也进!"           │
│                                     ═══════ 也在写 UART ═══════             │
│                                     █████████████████████████               │
│                                                                             │
│   💥 灾难发生：Task A 与 Task C 同时向同一个串口发数据，字符完全混杂崩溃！   │
│                                                                             │
│   如果换用 Mutex:                                                           │
│   Task B 尝试 Give 时，内核检测其不是持有者，立刻拦截并触发 configASSERT！   │
│   门绝对不会被第三方非法打开！                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4.4 优先级反转 (Priority Inversion) 与优先级继承

**优先级反转是实时操作系统历史上最著名的经典缺陷（1997年火星探路者号曾因此险遭毁灭）。**

### 4.4.1 优先级反转物理灾难时序
假设系统中存在低优先级任务 `LowPri` (Pri 1)、中优先级任务 `MidPri` (Pri 2)、高优先级任务 `HighPri` (Pri 3)：

```
无优先级继承时的灾难时序 (使用普通二值信号量):
时间轴 ──────────────────────────────────────────────────────────────────────►

HighPri (Pri 3):                ░░░░░░░░░░░░░░░░░░░░░░░░░░████████ (被活活饿死数十秒!)
                                尝试拿锁被挂起休眠 ▲
MidPri  (Pri 2):        ░░░░░░░░██████████████████████████░░░░░░░░ (疯狂独占 CPU!)
                        就绪即抢占 LowPri ▲
LowPri  (Pri 1):  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████ (无 CPU 时间释放锁)
                  拿着锁开始干活 ▲

灾难根因：
1. LowPri 拿着共享资源锁在慢慢跑；
2. HighPri 就绪想要拿锁，发现被占，被迫阻塞休眠等待；
3. 此时不需要该资源的 MidPri 突然就绪！由于 MidPri 优先级高于 LowPri，MidPri 抢占了 CPU；
4. MidPri 一直狂跑，LowPri 得不到 CPU 就无法释放锁；
5. 最终导致：优先级最高、最紧急的 HighPri 竟然被优先级更低的 MidPri 无限期卡死！
```

### 4.4.2 互斥锁的救赎：优先级继承 (Priority Inheritance)

FreeRTOS 的 `Mutex` 内置了优先级继承机制：

```
有优先级继承时的优雅化解 (使用 Mutex):
时间轴 ──────────────────────────────────────────────────────────────────────►

HighPri (Pri 3):                ░░░░░░░░████████████████████ (拿到锁，极速响应!)
                                等待锁 ▲
MidPri  (Pri 2):        ░░░░░░░░        ░░░░░░░░████████████ (抢不过临时 Pri 3)
                        想抢占但抢不过 ▲
LowPri  (Pri 1):  ██████████████████████ (释放锁，优先级降回 Pri 1)
                  拿锁  ▲ 内核将其优先级临时提升为 Pri 3！

化解过程：
当内核发现 HighPri 正在等待 LowPri 手里的锁时，【立刻将 LowPri 的优先级动态提升到与 HighPri 一样高 (Pri 3)】！
这样 MidPri (Pri 2) 就无法抢占 LowPri，LowPri 得以火速执行完毕并释放锁；
锁一释放，LowPri 恢复原本优先级，HighPri 立刻秒级夺回 CPU 运行！
```

### 4.4.3 优先级继承的物理边界
优先级继承能够极大地缓解优先级反转，但**它并不是万能的**：
1. **不能预防死锁：** 互斥锁只能解决任务抢占造成的倒置，如果两个任务互相持有对方所需的锁，依然会发生死锁；
2. **单层继承机制：** FreeRTOS 默认实现的是单层优先级继承；
3. **微小的上下文开销：** 每次 Take 和 Give 时内核需要检测等待链表并动态调权。

---

## 4.5 递归互斥锁 (Recursive Mutex)：嵌套调用保护

在软件模块化设计中，经常出现函数嵌套调用的情况：

```c
static void Driver_WriteRegister(void)
{
    xSemaphoreTakeRecursive(xMutex, portMAX_DELAY); /* 第 2 次 Take: 嵌套计数 = 2 */
    /* 执行底层寄存器写入 */
    xSemaphoreGiveRecursive(xMutex);                /* 嵌套计数减为 1，锁仍未释放 */
}

static void Subsystem_UpdateConfig(void)
{
    xSemaphoreTakeRecursive(xMutex, portMAX_DELAY); /* 第 1 次 Take: 嵌套计数 = 1，获得锁 */
    Driver_WriteRegister();                         /* 内部调用了也需要锁的子函数 */
    xSemaphoreGiveRecursive(xMutex);                /* 嵌套计数减为 0，锁真正释放！ */
}
```

> ⚠️ **为什么普通 Mutex 不行？**  
> 如果在 `Subsystem_UpdateConfig` 已经持有普通 Mutex 的情况下，内部子函数再次对其调用 `xSemaphoreTake`，普通互斥锁会认为该资源已被占用，当前任务将**自己把自己永远锁死**（直接触发断言报错）！  
> 递归互斥锁记录了“持有者任务句柄”与“递归嵌套计数器”，只有 `Give` 的次数严格等于 `Take` 的次数，锁才会真正向外释放。

---

## 4.6 经典工程并发设计模式

### 4.6.1 模式一：看门人模式 (GateKeeper Task) —— 独占外设解耦
在复杂项目中，如果 10 个任务都拿着 Mutex 去抢着写串口，虽然保证了不乱码，但各个任务会频繁发生阻塞和优先级颠簸。  
**最佳架构是设计一个看门人任务：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         看门人任务 (GateKeeper) 架构                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Task A ──发送打印请求──► ┐                                                │
│   Task B ──发送打印请求──► ┼──► [ 打印队列 (Queue) ] ──► [ GateKeeper 任务 ] │
│   Task C ──发送打印请求──► ┘                             (独占真实 UART 硬件)│
│                                                                             │
│   核心优势：                                                                │
│   1. 只有 GateKeeper 任务一人接触硬件，彻底免除 Mutex 争抢；               │
│   2. 请求任务无需等待硬件发送完成，把数据压入队列后即可立刻返回；            │
│   3. 方便统一集中实现 DMA 批量发送、速率控制与日志分级过滤。               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.6.2 模式二：Dijkstra 有界缓冲区 (Bounded Buffer) —— 信号量版生产者-消费者

详见配套进阶代码 `Lesson4_Semaphore_Advanced.md`。使用 **2 个计数信号量 + 1 个互斥锁** 控制共享循环数组：

```
核心对象组合:
1. xSemEmpty:  计数信号量，初始 = N (代表剩余空槽数)
2. xSemFilled: 计数信号量，初始 = 0 (代表已有满槽数)
3. xMutex:     互斥锁，保护环形数组的并发写入索引
```

#### Dijkstra 经典 4 步协议（顺序绝对不可调换！）

```c
/* 生产者标准 4 步 */
xSemaphoreTake(xSemEmpty, portMAX_DELAY);  /* ① 申请一个空槽位 (没空槽则休眠) */
xSemaphoreTake(xMutex, portMAX_DELAY);     /* ② 拿到互斥锁保护内存数组 */
RingBuffer[wrIdx++] = data;                /*    写入数据 */
xSemaphoreGive(xMutex);                    /* ③ 释放互斥锁 */
xSemaphoreGive(xSemFilled);                /* ④ 信号量通知消费者: 新增一条数据 */

/* 消费者标准 4 步 */
xSemaphoreTake(xSemFilled, portMAX_DELAY); /* ① 申请一条有效数据 (没数据则休眠) */
xSemaphoreTake(xMutex, portMAX_DELAY);     /* ② 拿到互斥锁保护内存数组 */
data = RingBuffer[rdIdx++];                /*    读取数据 */
xSemaphoreGive(xMutex);                    /* ③ 释放互斥锁 */
xSemaphoreGive(xSemEmpty);                 /* ④ 信号量通知生产者: 腾出一个空槽 */
```

> 💀 **致命死锁顺序陷阱：**  
> 如果开发者把步骤 ① 和 ② 颠倒：先拿 `xMutex` 锁，再去申请 `xSemEmpty`。  
> 一旦缓冲区满了，生产者拿着 Mutex 锁进入阻塞休眠；而消费者想要消费必须先拿 Mutex 锁，但锁被生产者攥着！**双向死锁，系统瞬间暴毙！**  
> **铁律：永远先申请资源配额信号量，再获取内存互斥锁！**

---

## 4.7 实战代码范式与运行观测

### 4.7.1 全局对象收敛规范

```c
/* 集中收敛本模块所有信号量句柄 */
static struct {
    SemaphoreHandle_t xBinSync;     /* 二值信号量: 任务同步 */
    SemaphoreHandle_t xCountRes;    /* 计数信号量: 资源池 */
    SemaphoreHandle_t xUartMutex;   /* 互斥锁: 外设互斥保护 */
} g;
```

### 4.7.2 互斥锁保护硬件外设标杆范式

```c
void ThreadSafe_Print(const char *msg)
{
    /* 获取互斥锁，设置合理的容忍超时 (如 200ms) */
    if (xSemaphoreTake(g.xUartMutex, pdMS_TO_TICKS(200)) == pdPASS)
    {
        /* 拿到所有权，安全独占输出 */
        HAL_UART_Transmit(&huart1, (uint8_t *)msg, strlen(msg), 100);
        
        /* 严格在同一任务中配对释放 */
        xSemaphoreGive(g.xUartMutex);
    }
    else
    {
        /* 打印超时告警 */
    }
}
```

---

## 4.8 选型决策树、常见陷阱与避坑防线

### 4.8.1 信号量秒级工程选型决策树

```
                         你的工程核心诉求是什么？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
     [ 任务同步 / 事件通知 ]                           [ 保护共享资源 / 互斥访问 ]
           │                                                 │
     有数据帧需要传递吗？                              有几个并发任务需要访问？
     ├─ 是 ──► 【消息队列 Queue】                       ├─ 仅通知单一任务 ──► 【任务通知 Task Notify】
     │                                                 │
     └─ 否 ──► 是否来自中断或单事件触发？              └─ 多个任务争抢共享外设/全局变量
           ├─ 单个事件通知 ──► 【二值信号量 Binary Sem】       ├─ 存在嵌套函数调用 ──► 【递归互斥锁 Recursive Mutex】
           │                                                 │
           └─ 有限资源配额池 ──► 【计数信号量 Counting Sem】  └─ 常规独占访问 ──► 【标准互斥锁 Mutex】
                                                                   (坚决不用二值信号量做互斥!)
```

### 4.8.2 信号量开发 6 大致命陷阱

| 致命陷阱 | 严重后果 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **误用二值信号量做互斥** | 第三方任务误 Give 打开大门，导致数据严重踩踏 | 保护共享资源必须严格使用 `xSemaphoreCreateMutex` |
| **在中断 ISR 中调用 Mutex API** | 中断无任务所有权，直接触发内核严重断言崩溃 | ISR 中只允许调用 `xSemaphoreGiveFromISR` 等专用宏 |
| **在 ISR 中调用带有超时的 API** | 中断上下文不可挂起，系统直接卡死在异常状态 | 中断中超时参数必须传 0，使用 `pxHigherPriorityTaskWoken` |
| **获取 Mutex 后中途异常 return** | 锁永远无法释放，后续所有请求该资源的任务永久死锁 | 确保 `Take` 与 `Give` 处于同一层级，或通过严谨的退出标签统一释放 |
| **普通 Mutex 嵌套获取** | 任务第二次调用时自己等待自己释放，直接死锁挂起 | 嵌套函数调用必须选用 `xSemaphoreCreateRecursiveMutex` |
| **Bounded Buffer 先拿锁再拿信号量** | 缓冲区满时生产者带锁休眠，引发系统全局死锁 | 永远遵循：**先申请资源信号量，再获取内存互斥锁** |

---

## 4.9 第4课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **大一统本质：** 能否向他人解释为何 FreeRTOS 信号量底层完全就是一个没有数据 payload 的队列？
- [ ] **三者差异：** 能否清楚阐述二值信号量、计数信号量、互斥锁在状态模型与所有权上的本质区别？
- [ ] **反转与继承：** 能否画出优先级反转的发生时序，并解释互斥锁是如何通过动态调权解决反转的？
- [ ] **中断禁区：** 为什么互斥锁绝对不能在中断 ISR 中被获取或释放？
- [ ] **二值互斥陷阱：** 为什么“二值信号量能做互斥”是完全错误的伪经验？
- [ ] **死锁预防：** 在有界缓冲区 (Bounded Buffer) 模式中，为什么必须先 Take 计数信号量，后 Take 互斥锁？
- [ ] **实操验证：** 编译烧录 `Lesson4_Semaphore_Basic`，观察 3 组 LED 分别代表的同步、计数与互斥效果。

***

# 第二部分：第5课 — 软件定时器

> **核心主题：** 时间管理与定时服务 (Software Timer) — 守护任务机制与事件驱动
> 
> **📎 配套代码文件:**
> - `Lesson5_Timer_Basic.md` — 基础版：单次 (One-shot) + 周期 (Auto-reload) 定时器实战
> - `Lesson5_Timer_Daemon.md` — 深入专题：定时器守护任务 (Daemon Task) 源码级精讲

***

```
▎ 课程目标： 掌握 xTimerCreate / xTimerStart / xTimerStop / xTimerChangePeriod / xTimerReset
            透彻理解 Daemon Task 守护任务与命令队列运行机制，攻克定时器回调约束与多通道复用
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 前置成果： 第4课已掌握二值/计数信号量、互斥锁与优先级继承
```

---

## 5.1 软件定时器的物理本质：从硬件定时器到内核调度

在嵌入式开发中，我们经常需要在“未来的某个时间点”执行一段逻辑（例如：超时检测、按键消抖、LED 周期闪烁、传感器定时上报）。  
单片机虽然有硬件定时器（TIM1~TIMx），但硬件定时器资源极其宝贵，而且配置中断服务程序不仅麻烦，还容易增加系统的中断嵌套负载。  
FreeRTOS 提供了**软件定时器 (Software Timer)**，允许你在不消耗额外硬件定时器的前提下，创建**成百上千个独立的定时闹钟**！

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  硬件定时器 vs 软件定时器全方位对比矩阵                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  对比维度       │ 硬件定时器 (Hardware TIM)    │ 软件定时器 (Software Timer)│
│  ───────────────┼──────────────────────────────┼─────────────────────────── │
│  硬件依赖       │ 必须占用芯片真实的定时器外设 │ 仅依赖系统时钟节拍 SysTick │
│  数量限制       │ 受限于芯片硅片资源 (通常仅数个)│ 仅受限于 RAM 大小 (可创建成百上千个)│
│  时间精度       │ 极高 (可精确至微秒甚至纳秒级)│ 毫秒级 (以系统 Tick 为最小分辨率) │
│  执行上下文     │ 硬件中断上下文 (ISR)         │ 普通任务上下文 (Daemon Task) │
│  抖动性 (Jitter)│ 极低 (由硬件计数器直接翻转)  │ 存在轻微抖动 (受高优先级任务抢占影响)│
│  典型适用场景   │ 电机 PWM、高频编码器采样、纳秒捕获│ 超时报警、按键长按检测、周期心跳 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5.2 核心机理：守护任务 (Daemon Task) 与命令队列深度解构

**很多新手最容易产生两大误解：**  
1. 以为“每个软件定时器都在各自的独立线程里运行”；  
2. 以为“定时器回调函数是在中断服务程序里被调用的”。  
**这两个理解全都是错的！**

### 5.2.1 揭开幕后黑手：`prvTimerTask` 守护任务
在开启 `#define configUSE_TIMERS 1` 后，启动调度器时内核会自动在后台静默创建一个名为 **`Tmr Svc` (定时器服务 / Daemon Task)** 的专职任务，并配套一个**定时器命令队列**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      软件定时器命令解耦运行全景图                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   用户业务任务                   定时器命令队列 (Queue)          Daemon 守护任务    │
│   ┌──────────────┐               ┌───┬───┬───┐               ┌────────────┐ │
│   │ xTimerStart()│ ──写控制命令─►│CMD│CMD│CMD│ ──读命令唤醒──►│ prvTimerTask│ │
│   │              │ (非阻塞或超时)└───┴───┴───┘               │ (串行执行) │ │
│   └──────────────┘                                           └─────┬──────┘ │
│                                                                    │        │
│   SysTick 中断服务 (ISR)                                           │        │
│   ┌──────────────────────────┐                                     │        │
│   │ 每次 Tick 检测到定时器到期│ ──写到期事件─► [ 入队唤醒 ]        │        │
│   └──────────────────────────┘                                     ▼        │
│                                                           执行对应的回调函数│
│                                                           Callback_Func()   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2.2 守护任务模型的三大生死洞察

1. **全系统串行执行：**  
   所有的软件定时器回调函数，无论你创建了 3 个还是 30 个，**全部排队在同一个 `Daemon Task` 任务中被依次串行调用！** 它们不是并发运行的。
2. **多米诺骨牌效应：**  
   如果其中一个定时器的回调函数执行耗时较长（比如耗时 50ms），**全系统中后续所有定时器的回调都将被推迟整整 50ms！**
3. **优先级必须压制业务任务：**  
   `configTIMER_TASK_PRIORITY` 必须设置得足够高（通常配置为 `configMAX_PRIORITIES - 1`，最高或次高优先级）。如果守护任务优先级太低，被普通业务任务抢占，定时器的触发时间将会发生严重的不可控延迟！

---

## 5.3 核心 API 矩阵与状态机迁移

### 5.3.1 核心 API 速查

| API 名称 | 核心功能 | 关键行为与注意事项 |
| :--- | :--- | :--- |
| `xTimerCreate(...)` | 动态创建软件定时器 | 创建后处于**休眠态 (Dormant)**，必须调用 Start 才会开始计时 |
| `xTimerStart(timer, timeout)` | 启动或重新启动定时器 | 本质是向定时器命令队列发送一条 `tmrCOMMAND_START` 消息 |
| `xTimerStop(timer, timeout)` | 停止定时器计时 | 状态转入 Dormant，不再触发回调 |
| `xTimerReset(timer, timeout)` | 重新复位计时起点 | 无论之前跑了多久，从当前时刻重新计算完整的定时周期 |
| `xTimerChangePeriod(...)` | 动态修改定时周期 | 改变定时时长（下个周期生效），若处于停止态则会隐式启动 |
| `xTimerDelete(timer, timeout)` | 销毁定时器并释放 RAM | 向命令队列发送销毁指令，Daemon 任务随后回收其内存 |
| `pvTimerGetTimerID(timer)` | 读取定时器的用户私有 ID | 在回调函数中区分当前触发的是哪一个定时器 |
| `vTimerSetTimerID(timer, id)` | 更新定时器的用户私有 ID | 动态修改用户上下文标识 |
| `xTimerIsTimerActive(timer)` | 查询当前是否正在计时 | 返回 `pdTRUE` 代表正在运行；`pdFALSE` 代表处于休眠态 |

### 5.3.2 单次 (One-shot) vs 周期 (Auto-reload) 状态机全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 单次定时器 vs 自动重载定时器生命周期状态机                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 单次定时器 One-shot (uxAutoReload = pdFALSE) ]                           │
│                                                                             │
│    xTimerCreate() ──► ┌──────────────┐   xTimerStart()    ┌──────────────┐  │
│                       │ 休眠态 Dormant│ ─────────────────►│ 运行态 Running│  │
│                       └──────────────┘                    └──────┬───────┘  │
│                              ▲                                   │          │
│                              └────── 定时器到期并执行回调 ───────┘          │
│                                    (自动停机，回到休眠态，等待下次手动 Start)│
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  [ 自动重载定时器 Auto-reload (uxAutoReload = pdTRUE) ]                     │
│                                                                             │
│    xTimerCreate() ──► ┌──────────────┐   xTimerStart()    ┌──────────────┐  │
│                       │ 休眠态 Dormant│ ─────────────────►│ 运行态 Running│  │
│                       └──────────────┘                    └──────┬───────┘  │
│                              ▲                                   │ 循环到期 │
│                              │          xTimerStop()             ▼ 自动重启 │
│                              └───────────────────────────── [ 执行回调函数 ]│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- **单次定时器 (One-shot)：** 适合“动作触发型”场景。例如：按键按下后启动一个 3 秒单次定时器，超时未松开判定为长按；收到请求后启动 500ms 单次定时器，超时未收到响应则触发重传。
- **周期定时器 (Auto-reload)：** 适合“心跳自律型”场景。例如：每 500ms 翻转一次运行指示灯，每 1 秒读取一次电池电量。

---

## 5.4 定时器回调函数第一天条：三要与三绝不

**这是所有使用软件定时器的工程师必须刻在骨子里的铁律：**  
> 💡 **核心心智模型：把定时器回调函数当做“低优先级软中断”来写！**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     定时器回调函数的“三要与三绝不”                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ 三项绝对严禁 (违者全系统定时器瞬间瘫痪)：                               │
│  1. 绝不能调用 vTaskDelay()！(Daemon 任务被挂起，所有定时器全部罢工)        │
│  2. 绝不能调用带有阻塞超时的 API (如 xQueueReceive(q, buf, portMAX_DELAY))！ │
│  3. 绝不能执行耗时庞大的复杂算法、大量浮点运算或慢速串口长打印！           │
│                                                                             │
│  ✅ 三项标准推荐 (极速执行，立刻退出)：                                     │
│  1. 操作简单的 GPIO 引脚翻转 (微秒级耗时)；                                 │
│  2. 向目标队列发送非阻塞数据包 (timeout = 0)；                              │
│  3. 给出二值信号量 (xSemaphoreGive) 唤醒专职 Worker 任务处理重活！          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 重活解耦标准设计范式：发信号 ➔ 唤醒 Worker 任务

如果定时器到期后，需要进行长达 200ms 的传感器读取和复杂滤波算法，**绝对不能在回调函数里直接算！** 标准架构如下：

```c
/* 定时器回调函数: 只负责敲门 */
static void Timer_SensorCallback(TimerHandle_t xTimer)
{
    /* 快速扔出一个二值信号量，立刻退出回调，耗时 < 1 微秒 */
    xSemaphoreGive(g.xSemSensorTrigger);
}

/* 专职后台 Worker 任务: 负责干重活 */
static void Task_SensorWorker(void *pvParameters)
{
    for (;;)
    {
        /* 阻塞等待定时器敲门 */
        xSemaphoreTake(g.xSemSensorTrigger, portMAX_DELAY);
        
        /* 这里可以放心执行长达数百毫秒的重度算法，绝不影响其他定时器！ */
        Read_Sensors_Heavy_Algorithm();
    }
}
```

---

## 5.5 实战标杆代码与多通道 ID 复用

在工业级代码中，如果系统有 8 个不同周期的 LED 或通信通道，为每个定时器单独写一个回调函数会浪费大量代码空间。  
**最佳工程实践是复用同一个回调函数，通过 `pvTimerGetTimerID()` 区分通道**：

```c
/* 1. 集中收敛定时器句柄与全局对象 */
static struct {
    TimerHandle_t xTmrOneShot;   /* 超时单次定时器 */
    TimerHandle_t xTmrLed1;      /* 周期 500ms 定时器 */
    TimerHandle_t xTmrLed2;      /* 周期 200ms/1000ms 可变定时器 */
} g;

/* 统一复用定时器回调函数 */
static void Timer_CommonCallback(TimerHandle_t xTimer)
{
    /* 读取定时器创建时绑定的私有 ID 标号 */
    uint32_t timerChannel = (uint32_t)pvTimerGetTimerID(xTimer);

    switch (timerChannel)
    {
        case 0: /* One-shot 到期: 超时报警并关闭蜂鸣器 */
            Buzzer_Off();
            break;
            
        case 1: /* Channel 1 到期: 翻转心跳 LED1 */
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_1);
            break;
            
        case 2: /* Channel 2 到期: 翻转状态 LED2 */
            HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_12);
            break;
            
        default:
            break;
    }
}

/* 2. 初始化与创建 */
void APP_TimerInit(void)
{
    /* 创建单次定时器 (3 秒超时，绑定 ID 为 0) */
    g.xTmrOneShot = xTimerCreate("OneShot", pdMS_TO_TICKS(3000), pdFALSE, (void *)0, Timer_CommonCallback);
    configASSERT(g.xTmrOneShot != NULL);

    /* 创建周期定时器 (500ms 周期，绑定 ID 为 1) */
    g.xTmrLed1 = xTimerCreate("Led1Tmr", pdMS_TO_TICKS(500), pdTRUE, (void *)1, Timer_CommonCallback);
    configASSERT(g.xTmrLed1 != NULL);

    /* 创建可变周期定时器 (200ms 周期，绑定 ID 为 2) */
    g.xTmrLed2 = xTimerCreate("Led2Tmr", pdMS_TO_TICKS(200), pdTRUE, (void *)2, Timer_CommonCallback);
    configASSERT(g.xTmrLed2 != NULL);

    /* 启动周期定时器 (命令超时设为 0，不阻塞) */
    xTimerStart(g.xTmrLed1, 0);
    xTimerStart(g.xTmrLed2, 0);
}

/* 3. 动态改变周期妙用 (如遇到故障时让 LED2 闪烁频率急剧加快) */
void Set_AlertMode(BaseType_t isAlert)
{
    if (isAlert)
    {
        /* 紧急模式: 变更为 100ms 急闪 */
        xTimerChangePeriod(g.xTmrLed2, pdMS_TO_TICKS(100), 0);
    }
    else
    {
        /* 常规模式: 恢复 1000ms 慢闪 */
        xTimerChangePeriod(g.xTmrLed2, pdMS_TO_TICKS(1000), 0);
    }
}
```

---

## 5.6 内存预算与关键配置宏调优

要使用软件定时器，必须在 `FreeRTOSConfig.h` 中配置以下 4 个核心宏：

```c
#define configUSE_TIMERS                        1       /* 1: 启用软件定时器子系统 */
#define configTIMER_TASK_PRIORITY               ( 7 )   /* 守护任务优先级: 建议设为最高或次高 */
#define configTIMER_QUEUE_LENGTH                10      /* 定时器命令队列深度: 缓冲并发控制命令 */
#define configTIMER_TASK_STACK_DEPTH            128     /* 守护任务栈大小 (单位: Word, 128 = 512 字节) */
```

### 内存占用精细拆解

$$\text{定时器子系统基础开销} = \text{Daemon 任务 TCB} (\approx 80\text{B}) + \text{Daemon 栈} (512\text{B}) + \text{命令队列} (\approx 120\text{B}) \approx 712 \text{ 字节}$$

每个新增的定时器结构体 (`Timer_t`) 占用约 **64 字节**。  
对于资源紧凑的单片机，如果系统只需要 1~2 个简单的慢速周期任务，使用普通的 `vTaskDelayUntil()` 即可；只有在定时器数量较多、需要随时动态启停改变周期时，启用软件定时器才最具性价比。

---

## 5.7 常见陷阱与避坑防线

| 致命陷阱 | 严重后果与症状 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **回调函数中调用 `vTaskDelay`** | Daemon 守护任务陷入阻塞休眠，**全系统所有定时器全部停止响应！** | 回调函数严禁任何阻塞操作，耗时业务发信号量给 Worker 任务 |
| **在中断 ISR 中误调 `xTimerStart`** | 产生未定义行为或触发断言死机 | 中断中必须严格使用专用的 `xTimerStartFromISR()` 宏 |
| **命令队列深度设得太小** | 多任务突发启停定时器时，队列满导致控制命令静默丢弃 | `configTIMER_QUEUE_LENGTH` 至少应大于系统中定时器数量的 1.5 倍 |
| **Daemon 任务优先级设得过低** | 被长时间运行的普通业务任务抢占压制，定时器触发严重超时滞后 | 确保 `configTIMER_TASK_PRIORITY` 设置为系统最高或次高优先级 |
| **单次定时器到期后以为自动循环** | 执行一次后自动停止，后续再无动静 | 需要周期触发必须将 `uxAutoReload` 参数显式设置为 `pdTRUE` |
| **误以为 `xTimerChangePeriod` 立刻切断当前周期** | 修改周期在**下一次到期之后**才正式生效 | 如需立即重置生效，需显式执行 `xTimerReset()` 或 `Stop` 后重新 `Start` |

---

## 5.8 第5课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **物理机理：** 为什么说 FreeRTOS 软件定时器不是硬件外设，而是一个后台守护任务？
- [ ] **串行特性：** 如果定时器 A 的回调函数执行了 10ms，定时器 B 会受到什么直接影响？
- [ ] **第一天条：** 为什么在定时器回调函数中调用 `vTaskDelay()` 会导致全系统灾难？
- [ ] **重活解耦：** 面对“定时器到期后需要执行耗时 50ms 算法”的需求，标准架构应该怎么设计？
- [ ] **单次与重载：** 能否讲清 One-shot 与 Auto-reload 的状态机转换区别与典型工程场景？
- [ ] **动态调频：** 如何使用 `xTimerChangePeriod()` 在不重新创建定时器的前提下改变 LED 闪烁周期？
- [ ] **实操验证：** 编译烧录 `Lesson5_Timer_Basic`，观察 1 个单次定时器与 2 个周期定时器协调驱动 LED 的实际现象。

***

# 第二部分：第6课 — 事件组

> **核心主题：** 多事件复合同步与广播通信 (Event Group) — 24-bit 位图与集合点模式
> 
> **📎 配套代码文件:**
> - `Lesson6_EventGroup_Basic.md` — 基础版：OR(任一) / AND(全部) / Barrier(集合点) 三种同步模式实战

***

```
▎ 课程目标： 掌握 xEventGroupCreate / xEventGroupSetBits / xEventGroupWaitBits / xEventGroupSync
            透彻理解 24 位持久性位图、OR 逻辑与 AND 逻辑差异，攻克 Barrier 多任务集合点架构
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 前置成果： 第5课已掌握软件定时器、Daemon Task 机制与命令队列解耦
```

---

## 6.1 什么是“事件组”？多事件复合同步与广播通信

在前面几节课中，我们学习了二值信号量和队列：
- **二值信号量：** 只能表达“单一事件是否发生”，且是“点对点消费”——一个任务 Take 了，其他任务就拿不到了；
- **队列：** 适合承载具体的数据管道，但无法高效表达“同时等待事件 A 和事件 B 发生”。

在实际嵌入式工程中，我们经常遇到**多条件复合等待**与**一对多广播**的场景：
> ✈️ **场景 1（AND 与逻辑）：** 飞机起飞前，必须等待“动力自检完成” AND “航向校准完成” AND “通信链路握手成功”，三者**全部满足**才能起飞！  
> 🚨 **场景 2（OR 或逻辑）：** 监控任务在休眠中等待，只要“按键按下” OR “串口收到指令” OR “温度过热报警”**任一发生**，就立刻被唤醒！  
> 📢 **场景 3（一对多广播）：** 系统完成初始化，一个任务发出广播信号，**3 个后台工作任务同时被唤醒出发**！

**事件组 (Event Group)** 就是专门为了优雅解决上述场景而生的终极同步原语。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 事件组公告板模型 (Event Group Notice Board)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   事件产生者 (Producer A/B/C 或 ISR)          事件组 (24-bit 共享位图)      │
│   ┌──────────────┐                             ┌────────────────────────┐   │
│   │ 电机就绪     │ ──SetBits(BIT_0)──────────► │ Bit 0: [ 1 ] 电机就绪  │   │
│   ├──────────────┤                             ├────────────────────────┤   │
│   │ 传感器就绪   │ ──SetBits(BIT_1)──────────► │ Bit 1: [ 1 ] 传感器就绪│   │
│   ├──────────────┤                             ├────────────────────────┤   │
│   │ 通信建立     │ ──SetBits(BIT_2)──────────► │ Bit 2: [ 0 ] 通信未建立│   │
│   └──────────────┘                             └────────────────────────┘   │
│                                                            │                │
│                                                            ▼                │
│                                              消费者任务: xEventGroupWaitBits│
│                                              • OR 模式: 任一为 1 即刻唤醒   │
│                                              • AND 模式: 3个全为 1 才能起飞 │
│                                              • 广播机制: 多个任务可同时唤醒 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.1.1 为什么是 24 位？底层物理真相
在 32 位 MCU (如 Cortex-M4) 上，事件组的数据类型定义为 `EventBits_t`（本质是 `uint32_t`，32 位无符号整数）。  
**但用户最多只能支配低 24 位（Bit 0 ~ Bit 23）！**  
**原因：** 内核把**最高 8 位（Bit 24 ~ Bit 31）保留为内部控制标志位**（用于记录任务等待条件是否满足、是否处于清除阶段等元数据）。因此在定义事件掩码时，有效范围为 `(1UL << 0)` 到 `(1UL << 23)`。

### 6.1.2 核心特性：事件位的“持久性 (Persistent)”与假触发防范
- **信号量是“消耗型”的：** `xSemaphoreTake()` 成功后，信号量计数值自动减 1；
- **事件组是“持久型”的：** 调用 `xEventGroupSetBits()` 将某位置 1 后，**该位会一直保持为 1，直到被显式清零！**

> ⚠️ **新手陷阱：事件位不清除引发的“假触发”**  
> 如果任务等待事件 `BIT_0`，事件到来任务被唤醒处理；如果退出时没有清除 `BIT_0`，当下一次任务再次调用 `WaitBits` 时，发现 `BIT_0` 依然是 1，任务将**完全不等待直接瞬时返回！** 导致死循环空转。  
> **工程铁律：** 在绝大多数等待场景中，必须配置 `xClearOnExit = pdTRUE`（退出时自动清零关心的位）。

---

## 6.2 核心 API 矩阵与 `WaitBits` 5 大参数深度拆解

### 6.2.1 核心 API 全景

| API 名称 | 核心功能 | 上下文 | 关键行为与约束 |
| :--- | :--- | :--- | :--- |
| `xEventGroupCreate()` | 创建 24 位事件组 | 任务 | 初始状态下所有位皆为 0，无参数 |
| `xEventGroupSetBits(eg, bits)` | 将指定位置 1 | 任务 | 非阻塞，置位后内核自动检查并唤醒等待的任务 |
| `xEventGroupSetBitsFromISR(...)` | 在中断中将指定位置 1 | ISR | **异步委托机制**（见下文 6.2.3 节特殊说明） |
| `xEventGroupWaitBits(...)` | 阻塞等待指定事件位组合 | 任务 | 支持 OR/AND 模式，支持退出自动清零 |
| `xEventGroupSync(...)` | 多任务原子集合点 (Barrier)| 任务 | 原子执行“设置己方位 + AND 等待全方位 + 清零” |
| `xEventGroupClearBits(eg, bits)` | 手动将指定位清零 | 任务 | 不会引发阻塞 |
| `xEventGroupGetBits(eg)` | 读取当前事件位快照 | 任务/ISR | 非阻塞即时读取 |
| `vEventGroupDelete(eg)` | 销毁事件组 | 任务 | 确保没有任务仍在挂起等待该事件组 |

### 6.2.2 核心 API `xEventGroupWaitBits` 5 大参数拆解

```c
EventBits_t xEventGroupWaitBits(
    EventGroupHandle_t xEventGroup,        /* ① 事件组句柄 */
    const EventBits_t  uxBitsToWaitFor,    /* ② 感兴趣的事件位掩码 (如 BIT_A | BIT_B) */
    const BaseType_t   xClearOnExit,       /* ③ pdTRUE: 满足唤醒条件后，自动将关心的位清 0 */
    const BaseType_t   xWaitForAllBits,    /* ④ pdTRUE: AND 逻辑 (全部满足); pdFALSE: OR 逻辑 (任一满足) */
    TickType_t         xTicksToWait        /* ⑤ 超时阻塞时间 (如 portMAX_DELAY) */
);
```

#### 行为判定矩阵表

| `xWaitForAllBits` (逻辑模式) | `xClearOnExit` (退出清零) | 唤醒行为与结果 |
| :--- | :--- | :--- |
| **`pdFALSE` (OR 或门)** | **`pdTRUE` (推荐)** | **任一关注位为 1 立即唤醒**；唤醒后关注位自动被清零，防止重复假触发 |
| **`pdFALSE` (OR 或门)** | `pdFALSE` | 任一关注位为 1 立即唤醒；但事件位保持 1，后续等待会立刻再次触发 |
| **`pdTRUE` (AND 与门)** | **`pdTRUE` (推荐)** | **所有关注位全部为 1 才唤醒**；唤醒后全套关注位被清零，完成一轮同步 |
| **`pdTRUE` (AND 与门)** | `pdFALSE` | 所有关注位全部为 1 才唤醒；事件位保持 1，用于需要保留状态的只读场景 |

> 💡 **重要诊断技巧：必须用返回值判断事件！**  
> `xEventGroupWaitBits` 的**返回值是任务被唤醒瞬间的事件位快照**！  
> 务必直接用返回值进行判断：`if (xRetBits & BIT_A)`；  
> 绝不要在唤醒后再调一次 `xEventGroupGetBits()`，因为在多任务并发下，该位可能早已被其他同级任务取走或清除了！

### 6.2.3 中断专属 `xEventGroupSetBitsFromISR` 的特殊机制
很多工程师好奇：“为什么队列和信号量在中断里置位只需几行代码，而事件组置位却很复杂？”  
**原因：** 一个事件组置位，可能会**同时唤醒多个处于不同优先级的等待任务**！如果在中断中直接执行多任务就绪链表解绑和优先级调度，会导致 ISR 执行时间不可控，严重破坏中断实时性。  
**内核的精妙解法：**  
`xEventGroupSetBitsFromISR()` 实际上只是向**定时器守护任务 (Daemon Task)** 发送了一条命令，由后台守护任务在任务上下文中去执行真正的唤醒操作！  
**因此使用该 API 必须满足：** `FreeRTOSConfig.h` 中必须开启 `#define configUSE_TIMERS 1`！

---

## 6.3 三大核心工程模式全景详解

### 6.3.1 模式一：OR 同步 (任一就绪即响应) —— 多路事件集中监控
- **工程场景：** 一个数据集中处理任务，负责汇总来自串口 DMA 接收完成 (`BIT_UART`)、按键按下 (`BIT_KEY`)、温度报警 (`BIT_ALARM`) 的事件。
- **配置要点：** `xWaitForAllBits = pdFALSE`，`xClearOnExit = pdTRUE`。

```
BIT_UART  (Bit 0) ──┐
BIT_KEY   (Bit 1) ──┼──► [ OR 或门等待 ] ──► 任一事件发生，立刻唤醒工作任务！
BIT_ALARM (Bit 2) ──┘
```

### 6.3.2 模式二：AND 同步 (全部满足才启动) —— 多模块自检屏障
- **工程场景：** 系统上电后，三个独立的硬件初始化任务分别初始化外设，主控主循环必须等待三个硬件全部就绪后才允许开启总线通信。
- **配置要点：** `xWaitForAllBits = pdTRUE`，`xClearOnExit = pdTRUE`。

```
时钟初始化完成 (Bit 3) ──┐
GPIO 配置完成  (Bit 4) ──┼──► [ AND 与门等待 ] ──► 三者全为 1，系统才正式点亮！
传感器校准完成 (Bit 5) ──┘
```

### 6.3.3 模式三：Barrier 集合点同步 (`xEventGroupSync`) —— 多任务协同出发
在并发编程中，有时需要多个任务**互相等待，步调一致**（例如：多轴机械臂控制，X轴、Y轴、Z轴任务各自完成插补运算后，必须在集合点互相等待，直到 3 轴全部就绪，再同时驱动电机发出脉冲）。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 xEventGroupSync 多任务集合点 (Barrier / Rendezvous)         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│    Worker A (Pri 1)              Worker B (Pri 1)              Worker C     │
│    ┌──────────────────┐          ┌──────────────────┐          ┌──────────┐ │
│    │ 计算耗时 300ms   │          │ 计算耗时 500ms   │          │耗时 200ms│ │
│    │ 调 xEventGroupSync          │ 调 xEventGroupSync          │调 Sync   │ │
│    │ 置 BIT_A=1       │          │ 置 BIT_B=1       │          │置 BIT_C=1│ │
│    │ 挂起等待全就绪   │          │ 挂起等待全就绪   │          │挂起等待  │ │
│    │ ░░░░░░░░░░░░░░░░ │          │ ░░░░░░░░░░░░░░░░ │          │░░░░░░░░░ │ │
│    │                  │          │                  │          │          │ │
│    └──────────────────┴──────────┴──────────────────┴──────────┴──────────┘ │
│                                  ▲                                          │
│     Worker B 最后一个到达集合点，BIT_A|B|C 全为 1！                         │
│     内核原子清零位图，【同时唤醒全部 3 个任务，步调一致开始下一阶段运动！】  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.4 实战标杆代码与流程时序

### 6.4.1 事件位掩码统一规范定义

```c
/* 统一声明清晰的事件位掩码 (必须在 0~23 位之间) */
#define EVENT_BIT_SENSOR_A      ( 1UL << 0 )   /* Bit 0: 传感器 A 就绪 */
#define EVENT_BIT_SENSOR_B      ( 1UL << 1 )   /* Bit 1: 传感器 B 就绪 */
#define EVENT_BIT_KEY_PRESSED   ( 1UL << 2 )   /* Bit 2: 按键按下 */

#define EVENT_BIT_INIT_CLK      ( 1UL << 3 )   /* Bit 3: 时钟初始化完成 */
#define EVENT_BIT_INIT_FLASH    ( 1UL << 4 )   /* Bit 4: 存储器检测完成 */
#define EVENT_BIT_INIT_ALL      ( EVENT_BIT_INIT_CLK | EVENT_BIT_INIT_FLASH )

#define EVENT_SYNC_WORKER_1     ( 1UL << 5 )   /* Bit 5: 工人 1 抵达集合点 */
#define EVENT_SYNC_WORKER_2     ( 1UL << 6 )   /* Bit 6: 工人 2 抵达集合点 */
#define EVENT_SYNC_ALL          ( EVENT_SYNC_WORKER_1 | EVENT_SYNC_WORKER_2 )
```

### 6.4.2 经典 OR 同步与返回值判断实战

```c
static void Task_OrHandler(void *pvParameters)
{
    (void)pvParameters;
    
    for (;;)
    {
        /* OR 模式等待: 任一事件发生即唤醒，退出时自动清零 */
        EventBits_t xRecvBits = xEventGroupWaitBits(
            g.xEventGroup,
            EVENT_BIT_SENSOR_A | EVENT_BIT_SENSOR_B | EVENT_BIT_KEY_PRESSED,
            pdTRUE,              /* xClearOnExit = pdTRUE: 自动清零防止假触发 */
            pdFALSE,             /* xWaitForAllBits = pdFALSE: OR 模式 */
            portMAX_DELAY
        );

        /* 根据快照返回值，逐一判定触发源 */
        if (xRecvBits & EVENT_BIT_SENSOR_A)
        {
            Process_SensorA();
        }
        if (xRecvBits & EVENT_BIT_SENSOR_B)
        {
            Process_SensorB();
        }
        if (xRecvBits & EVENT_BIT_KEY_PRESSED)
        {
            Process_KeyAction();
        }
    }
}
```

### 6.4.3 经典 AND 同步实战

```c
static void Task_SystemStartup(void *pvParameters)
{
    (void)pvParameters;

    /* AND 模式等待: 所有初始化标志必须全部到位 */
    EventBits_t xInitBits = xEventGroupWaitBits(
        g.xEventGroup,
        EVENT_BIT_INIT_ALL,
        pdTRUE,                  /* 全部满足后清零 */
        pdTRUE,                  /* xWaitForAllBits = pdTRUE: AND 模式 */
        pdMS_TO_TICKS(5000)      /* 最多容忍 5 秒启动超时 */
    );

    if ((xInitBits & EVENT_BIT_INIT_ALL) == EVENT_BIT_INIT_ALL)
    {
        /* 所有模块正常就绪，安全启动主业务 */
        Start_Main_Application();
    }
    else
    {
        /* 超时自检失败，报警停机 */
        System_Error_Halt();
    }

    vTaskDelete(NULL);  /* 一次性任务执行完毕自删除 */
}
```

---

## 6.5 内存预算、常见陷阱与避坑防线

### 6.5.1 极度轻量的内存开销
与队列相比，事件组的 RAM 开销极低：  
$$\text{事件组控制块开销} \approx 40 \sim 60 \text{ 字节 (无数据缓冲区)}$$  
如果你不需要在任务间拷贝数据实体，仅仅是表达多种状态标志和同步，**事件组的内存性价比远超队列**。

### 6.5.2 事件组开发 6 大致命陷阱

| 致命陷阱 | 严重后果与症状 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **忘记设置 `xClearOnExit`** | 事件位置 1 后永久保持，后续 `WaitBits` 陷入死循环假触发 | OR 等待模式必须强制开启 `xClearOnExit = pdTRUE` |
| **使用超过 23 的高位** | Bit 24~31 为内核保留位，越界使用直接损坏等待判定逻辑 | 事件位掩码严格限制在 `(1UL << 0)` ~ `(1UL << 23)` 之间 |
| **唤醒后再次调用 `GetBits` 判断**| 多任务并发下该位可能已被同级任务消耗，读取到全 0 漏处理 | **严格使用 `xEventGroupWaitBits` 的返回值快照** 进行位与判断 |
| **在 ISR 中误调 `xEventGroupSetBits`**| 缺少中断保护，直接导致系统在中断内崩溃 | 中断中必须使用 `xEventGroupSetBitsFromISR`，且需开启定时器 |
| **`xWaitForAllBits` 参数写错** | 本意是 AND 却写成 pdFALSE，第一个事件到来就提前早产唤醒 | 仔细复核第四个参数：`pdTRUE` 为与门全部，`pdFALSE` 为或门任一 |
| **集合点 `Sync` 超时未处理** | 某 Worker 超时提前返回，步调不一致导致数据破坏 | 必须检查 `xEventGroupSync` 返回值是否等于全集位掩码 |

---

## 6.6 第6课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **物理位数：** 为什么 32 位的 `EventBits_t` 用户只能使用其中的低 24 位？
- [ ] **广播优势：** 相比于信号量和队列，事件组在多任务唤醒（一对多）上有什么独到优势？
- [ ] **持久性机理：** 什么是事件位的持久性？为什么 OR 等待模式如果不清零会导致系统死循环？
- [ ] **返回值判断：** 为什么判断哪个事件触发必须使用 `WaitBits` 的返回值，而不能重新调用 `GetBits`？
- [ ] **中断依赖：** 为什么在中断中调用 `xEventGroupSetBitsFromISR` 必须开启软件定时器服务 (`configUSE_TIMERS`)？
- [ ] **Barrier 集合点：** `xEventGroupSync` 底层是如何原子完成“我已到达”与“等待全员到齐”的？
- [ ] **实操验证：** 编译烧录 `Lesson6_EventGroup_Basic`，观察 OR、AND 和 Barrier 驱动 3 组 LED 的对应时序。

***

# 第二部分：第7课 — 任务通知

> **核心主题：** 轻量级极速通信 (Task Notification) — TCB 内嵌私有信箱与 IPC 终极选型
> 
> **📎 配套代码文件:**
> - `Lesson7_TaskNotify_Basic.md` — 基础版：Give/Take + Notify/Wait + 性能对比全套实战

***

```
▎ 课程目标： 掌握 xTaskNotifyGive/ulTaskNotifyTake（轻量同步）和 xTaskNotify/xTaskNotifyWait（全能带值通知）
            深度理解零堆内存开销原理与 45% 速度飞跃，攻克 IPC 四大家族终极选型决策
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 前置成果： 第6课已掌握事件组 24-bit 位图与多任务集合点同步
```

---

## 7.1 任务通知的物理本质：TCB 内嵌的“零堆内存私有信箱”

在前面学习队列、信号量和事件组时，我们都必须显式调用相应的 `Create` 函数：
- `xQueueCreate`、`xSemaphoreCreateBinary`、`xEventGroupCreate` 都会从堆中申请数十甚至上百字节的控制块内存；
- 每次发送或接收数据，内核都需要解引用这些独立的内核对象指针，并维护相应的双向挂起链表。

FreeRTOS 创造性地提出了一个革命性的 IPC 工具——**任务通知 (Task Notification)**！  
> 💡 **核心真相：每个任务天生自带一个 32 位通知信箱！**  
> 只要一个任务被创建（`xTaskCreate`），它的**任务控制块 (TCB)** 内部就已经原生嵌入了通知字段，**完全不需要调用任何创建函数，0 额外堆内存分配！**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 任务控制块 (TCB) 内部原生内嵌的通知结构                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   每个任务的 TCB 内存空间 (tasks.c 内部结构):                               │
│   ┌─────────────────────────────────────────────────────────┐               │
│   │  volatile StackType_t *pxTopOfStack;     // 栈顶指针    │               │
│   │  ... (优先级、任务名、状态链表节点等)                   │               │
│   │                                                         │               │
│   │  uint32_t     ulNotifiedValue;           // ◄── 32 位通知值 (数据载荷)  │
│   │  eNotifyValue eNotifyState;              // ◄── 通知状态机枚举          │
│   └─────────────────────────────────────────────────────────┘               │
│                                                                             │
│   这 2 个字段总共仅占用 8 字节，编译期直接并入 TCB，永不发生堆碎片！        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.1.1 任务通知与传统三大 IPC 全方位对比矩阵

| 对比指标 | 消息队列 (Queue) | 信号量 (Semaphore) | 事件组 (Event Group) | 任务通知 (Task Notification) |
| :--- | :--- | :--- | :--- | :--- |
| **独立内核对象** | 需要 (`Queue_t`) | 需要 (`Queue_t`) | 需要 (`EventGroup_t`) | **❌ 不需要 (原生内嵌于 TCB)** |
| **额外 RAM 堆开销** | 很高 (~100B ~ 几KB) | 中等 (~80B) | 中等 (~60B) | **🎉 0 字节 (零额外动态开销)** |
| **数据承载能力** | 任意大小字节流 | 无数据 (仅有状态) | 24 个独立标志位 | **单个 32 位整型数值 (可打包)** |
| **通信拓扑关系** | 多对多 (任意收发) | 多对多 | 多对多 (支持一对多广播) | **严格单播 (必须指定唯一接收任务)** |
| **消息缓冲能力** | ✅ 支持 N 条排队 | ❌ 无缓冲 | ❌ 无缓冲 | **❌ 仅有 1 槽 (无缓冲，需覆盖/丢弃)**|
| **发送方阻塞机制** | ✅ 队列满时可休眠 | ❌ | ❌ | **❌ 发送方绝对永不阻塞** |
| **API 执行速度** | ★★★ (~45μs) | ★★★★ (~25μs) | ★★★★ (~25μs) | **★★★★★ (~15μs，快约 45%!)** |

---

## 7.2 两大 API 族系与 5 大发送动作 (eAction) 全景

FreeRTOS 为任务通知提供了两套层级的 API，分别面向“极简轻量同步”和“复杂带值通信”：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        任务通知两大 API 族系分层架构                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 族系 1: 轻量同步族 (Give / Take) ] ──► 极速模拟二值信号量与计数信号量    │
│  • 发送: xTaskNotifyGive() / vTaskNotifyGiveFromISR()  (通知值直接自增 1)   │
│  • 接收: ulTaskNotifyTake(xClearOnExit, xTimeout)      (获取并清零或递减)   │
│                                                                             │
│  [ 族系 2: 全能通知族 (Notify / Wait) ] ──► 携带 32 位数据，支持 5 种动作   │
│  • 发送: xTaskNotify(xTask, ulValue, eAction)                               │
│  • 接收: xTaskNotifyWait(ulBitsToClearOnEntry, ulBitsToClearOnExit, ...)    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2.1 族系 1 核心速查：`Give` / `Take`（轻量同步）
- `xTaskNotifyGive(xTaskToNotify)`：直接将目标任务的通知值 `++`，并将目标任务置为 Pending 唤醒；
- `ulTaskNotifyTake(xClearCountOnExit, xTicksToWait)`：
  - **`xClearCountOnExit = pdTRUE`（二值信号量模式）：** 退出时直接将通知值清零（只要大于 0 即刻唤醒）；
  - **`xClearCountOnExit = pdFALSE`（计数信号量模式）：** 退出时将通知值仅减 1；
  - 🚨 **返回值关键警示：** **返回值是清除前的原计数值！** （`0` 表示超时未等到，`>0` 表示等到且数值代表累积次数；绝对不能写 `if (ret == pdPASS)`！）。

### 7.2.2 族系 2 发送端的 5 种核心动作 (`eNotifyAction`)

通过 `xTaskNotify(xTask, ulValue, eAction)` 发送时，可通过枚举 `eAction` 灵活定义底层行为：

| `eAction` 枚举值 | 通知值物理改变逻辑 | 适用典型场景 | 类比传统 IPC |
| :--- | :--- | :--- | :--- |
| **`eNoAction`** | 通知值保持不变，仅置位 Pending 状态 | 纯粹的事件唤醒打招呼，不传递任何数据 | 二值信号量 Give |
| **`eSetBits`** | `ulNotifiedValue |= ulValue` (位或操作) | 任务专属事件位等待（支持多标志组合） | 任务私有事件组 (EventGroup) |
| **`eIncrement`** | `ulNotifiedValue++` (计数值自增 1) | 统计外部事件发生次数（如中断产生脉冲数） | 计数信号量 (同 `NotifyGive`) |
| **`eSetValueWithOverwrite`** | `ulNotifiedValue = ulValue` (无条件覆盖) | 传感器最新采样值、最新控制状态（只关心最新） | 单槽邮箱 (`xQueueOverwrite`) |
| **`eSetValueWithoutOverwrite`** | 仅当状态为 Not Pending 时才写入；若已是 Pending 则**放弃写入并返回 pdFAIL** | 首次有效数据保护（如只记录第一次发生的故障码） | 保护首发参数 |

---

## 7.3 性能暴走密码：为什么任务通知比信号量快 45%、比队列快 3 倍？

### 7.3.1 调用路径精简度对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                FreeRTOS 三大典型 IPC 底层执行路径深度比对                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 任务通知 xTaskNotifyGive 路径 (极致精简) ]                                │
│  1. 直接通过传入的 TaskHandle (即目标 TCB 指针) 访问内存；                   │
│  2. 执行 targetTCB->ulNotifiedValue++ 并置位 Pending；                      │
│  3. 检查目标任务是否挂起在等待链表，若是则移入 Ready 链表；                  │
│  4. 触发 PendSV 抢占 ──► 【立刻返回！全流程仅操作 2 个内存变量，无锁无拷贝】│
│                                                                             │
│  [ 信号量 xSemaphoreGive 路径 (中等) ]                                       │
│  1. 校验 SemaphoreHandle 指针有效性；                                        │
│  2. 进入队列临界区保护；                                                     │
│  3. 解引用 Queue_t 控制块，检查 xTasksWaitingToReceive 链表；                │
│  4. 遍历等待节点并调整链表；退出临界区 ──► 【涉及对象解引用与链表进出开销】  │
│                                                                             │
│  [ 消息队列 xQueueSend 路径 (最长耗时) ]                                     │
│  1. 校验队列控制块有效性并进入临界区；                                       │
│  2. 检查环形缓冲区指针与剩余空间；                                           │
│  3. 调用 memcpy 进行整块内存逐字节物理拷贝；                                │
│  4. 调整读写指针，检查唤醒链表；退出临界区 ──► 【内存拷贝耗时严重】         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

在 Cortex-M4F 平台实测表明：
- 任务通知往返仅需 $\approx 15\mu\text{s}$；
- 二值信号量往返需 $\approx 25\mu\text{s}$（慢约 1.7 倍）；
- 消息队列往返需 $\approx 45\mu\text{s}$（慢约 3 倍）！

---

## 7.4 物理边界与 4 大绝不能使用的禁区

**任务通知虽然性能极其优异，但为了追求速度和零内存开销，它牺牲了通用性：**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       任务通知的 4 大不可逾越的边界                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ① 严格单播限制 (Unicast Only)：                                            │
│  • 任务通知是“点对点私有投递”，发送方必须明确持有唯一接收者的 TaskHandle！  │
│  • ❌ 绝不能实现一对多广播 (如需一个事件唤醒多个任务，必须选 EventGroup)。 │
│                                                                             │
│  ② 零缓冲单槽限制 (No Buffering)：                                          │
│  • 每个任务只有 1 个 32 位通知槽位；                                         │
│  • ❌ 无法排队缓冲多个不同的数据包（若需保证多包不丢失排队，必须选 Queue）。│
│                                                                             │
│  ③ 发送方绝对永不阻塞：                                                     │
│  • xTaskNotify 永远在几微秒内直接返回，绝不可能因为接收方没处理而休眠；      │
│  • ❌ 无法实现生产者的反压流控 (Backpressure) 协调机制。                     │
│                                                                             │
│  ④ 中断 ISR 只能发、绝不能收：                                               │
│  • 中断服务程序中可以调用 vTaskNotifyGiveFromISR / xTaskNotifyFromISR 发送；│
│  • ❌ 中断上下文中绝对不能调用 ulTaskNotifyTake / xTaskNotifyWait 接收等待！│
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.5 实战标杆代码与时序解析

### 7.5.1 场景一：模拟极速二值信号量 (ISR ➔ Task 唤醒)

这是任务通知最经典的工程应用场景：替代原有的二值信号量，实现中断向专职任务的微秒级唤醒：

```c
static TaskHandle_t s_hUartTask = NULL;

/* 1. 串口接收中断服务程序 */
void USART1_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    if (USART1->SR & USART_SR_RXNE)
    {
        /* 极速通知目标任务，通知值加 1 */
        vTaskNotifyGiveFromISR(s_hUartTask, &xHigherPriorityTaskWoken);
    }

    /* 如果目标任务优先级高于被打断的任务，立即触发快速上下文切换 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* 2. 专职 UART 处理任务 */
static void Task_UartProcessor(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        /* 二值模式等待: pdTRUE 表示唤醒后自动将计数清零，防止重复处理 */
        uint32_t ulNotifyCount = ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        if (ulNotifyCount > 0)
        {
            /* 被中断微秒级唤醒，处理接收缓冲 */
            Process_Uart_Rx_Data();
        }
    }
}
```

### 7.5.2 场景二：带值通知与 32 位数据打包传递

当需要向某个专职任务传递一条**小型单值控制命令**时，无需创建队列，直接利用 32 位整型进行位拼装打包：

```c
/* 宏定义: 将 led_id (高 16 位) 与 delay_ms (低 16 位) 打包为一个 32 位整数 */
#define PACK_CMD(led, delay)    ( ((uint32_t)(led) << 16) | ((uint32_t)(delay) & 0xFFFF) )
#define UNPACK_LED(val)         ( (uint8_t)((val) >> 16) )
#define UNPACK_DELAY(val)       ( (uint16_t)((val) & 0xFFFF) )

/* 发送者任务: 发送带值通知 (覆盖模式) */
void Send_LedCommand(TaskHandle_t hReceiver, uint8_t ledId, uint16_t delayMs)
{
    uint32_t packedData = PACK_CMD(ledId, delayMs);
    
    /* eSetValueWithOverwrite 确保哪怕接收者忙，也一定能收到最新的控制参数 */
    xTaskNotify(hReceiver, packedData, eSetValueWithOverwrite);
}

/* 接收者任务: 解包并执行控制 */
static void Task_LedWorker(void *pvParameters)
{
    (void)pvParameters;
    uint32_t recvData = 0;

    for (;;)
    {
        /* 等待通知到达: 0UL, 0UL 表示不进行入队出队位掩码清除 */
        if (xTaskNotifyWait(0UL, 0UL, &recvData, portMAX_DELAY) == pdPASS)
        {
            uint8_t  ledId   = UNPACK_LED(recvData);
            uint16_t delayMs = UNPACK_DELAY(recvData);

            APP_ToggleLed(ledId);
            vTaskDelay(pdMS_TO_TICKS(delayMs));
        }
    }
}
```

---

## 7.6 IPC 终极选型决策树、常见陷阱与复盘自测

### 7.6.1 FreeRTOS IPC 四大家族终极决策树

```
                         系统间需要进行通信或同步？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
      [ 纯粹的状态/事件同步 ]                           [ 包含具体数据载荷 ]
           │                                                 │
   接收目标任务数量？                                  数据大小与是否需要排队？
     ├─ 仅通知【单一指定任务】                           ├─ 大于 4 字节 或 【需要多包队列缓冲】
     │   └─► 🎉【任务通知 TaskNotifyGive/Take】          │   └─► 【消息队列 Queue】(第3课)
     │       (最快、0 内存堆开销、无脑首选!)                 │
     │                                                   └─ ≤ 4 字节 标量数据 且 【单播无缓冲】
     ├─ 唤醒【多个任务】或需要【AND 与门复合等待】           └─► 🎉【带值任务通知 TaskNotify】
     │   └─► 【事件组 Event Group】(第6课)
     │
     ├─ 中断通知多个未知任务
     │   └─► 【二值信号量 Binary Semaphore】(第4课)
     │
     └─ 独占硬件外设 / 共享资源互斥
         └─► 必须使用【互斥锁 Mutex】(带优先级继承，第4课)
```

### 7.6.2 任务通知开发 6 大致命陷阱

| 致命陷阱 | 严重后果与症状 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **误将 `ulTaskNotifyTake` 当 bool 判断** | 返回的是清除前计数值，用 `== pdPASS` 导致无法正确识别超时 | 必须判断 `if (ulCount > 0)` 代表成功收到通知 |
| **多包发送被无缓冲无情吞没** | 快速连续发送 3 次数据，接收方只能收到最后 1 条（或第 1 条） | 业务需要排队缓冲多条数据时，必须老老实实用 **Queue** |
| **误用单播 API 尝试广播多任务** | 只能指定一个 TaskHandle，其他关注该事件的任务完全得不到通知 | 多任务一对多广播同步必须改用 **EventGroup** |
| **在中断 ISR 中调用 Take/Wait** | 中断没有上下文，不可阻塞，直接触发内核硬断言死机 | ISR 中只能调用 `FromISR` 发送通知，绝不能接收等待 |
| **`eSetValueWithoutOverwrite` 静默丢弃**| 目标任务未处理完旧值时新值直接被丢弃（返回 pdFAIL） | 仔细评估是否改用 `eSetValueWithOverwrite` 覆盖模式 |
| **使用未初始化的 TaskHandle 发送** | 向 NULL 或野指针发送通知，直接引发内存踩踏崩溃 | 创建任务时务必保存句柄，确保接收任务已成功创建 |

---

## 7.7 第7课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **零开销本质：** 为什么任务通知完全不需要调用 `Create` 函数即可直接使用？它内嵌在哪个结构体中？
- [ ] **速度优势：** 为什么任务通知比二值信号量快 45%、比队列快 3 倍？它在调用路径上砍掉了哪些开销？
- [ ] **返回值铁律：** `ulTaskNotifyTake` 的返回值到底代表什么？为什么写 `if (ret == pdPASS)` 是严重错误？
- [ ] **动作选型：** `eSetValueWithOverwrite` 与 `eSetValueWithoutOverwrite` 的核心区别是什么？
- [ ] **四大禁区：** 为什么说“需要多任务广播”和“需要消息缓冲排队”的场景绝对不能用任务通知？
- [ ] **终极选型：** 面对“串口单字节中断唤醒专职任务”，在队列、信号量和任务通知中，你为什么会毫不犹豫选择任务通知？
- [ ] **实操验证：** 编译烧录 `Lesson7_TaskNotify_Basic`，通过 LED 与逻辑分析仪观察任务通知对比信号量的极速表现。

***

# 第二部分：第8课 — 中断管理

> **核心主题：** 硬件世界与内核的安全握手 (Interrupt Management) — NVIC 优先级、FromISR 规范与临界区
> 
> **📎 配套代码文件:**
> - `Lesson8_Interrupt_Basic.md` — 基础版：TIM6 硬件定时器中断 ➔ 队列通信 + 临界区保护实战

***

```
▎ 课程目标： 掌握 xxxFromISR() API 规范、pxHigherPriorityTaskWoken 模式、BASEPRI 临界区保护、中断优先级分层与延迟分析
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)，内置 TIM6 硬件定时器
▎ 前置成果： 第7课已掌握任务通知极速同步与 IPC 四大家族选型
```

---

## 8.1 嵌入式中断哲学与分层架构

在实时操作系统中，中断服务程序 (ISR) 拥有至高无上的权力——**无论当前正在执行哪个优先级的任务，硬件中断一旦触发，CPU 都会立刻强行打断任务并跳入 ISR 执行！**  
如果中断管理不当（如在中断里执行耗时漫长的计算、打印），整个操作系统的任务调度将被彻底打乱，实时性荡然无存。

### 8.1.1 经典工程架构：两阶段处理模型 (Top-Half / Bottom-Half)

工业级嵌入式系统无一例外地遵循**“上半部 (Top-Half) 快速响应 + 下半部 (Bottom-Half) 慢速处理”**的解耦哲学：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  中断“上半部 / 下半部”两阶段解耦设计模型                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   [ 上半部: 硬件 ISR 中断服务程序 ] ──► 追求“极致短小，微秒级退出”          │
│   • 硬件外设触发中断 (如串口收到数据)；                                     │
│   • 仅做最基本的寄存器读取与硬件中断标志清除；                              │
│   • 调用 xxxFromISR() 将数据或事件塞入队列/任务通知；                       │
│   • 耗时通常控制在 < 5 微秒，立刻退出中断！                                 │
│                                                                             │
│                                    │                                        │
│                                    ▼ (通过 IPC 解耦传递)                    │
│                                                                             │
│   [ 下半部: 专职后台 Worker 处理任务 ] ──► 追求“完整健壮，支持耗时业务”     │
│   • 平时在队列或通知上处于 Blocked 阻塞休眠，0% CPU 占用；                  │
│   • 被中断发来的事件唤醒后，在普通任务上下文中运行；                        │
│   • 执行复杂的数据校验 (CRC)、协议解包、浮点滤波、落库存储甚至网络转发；    │
│   • 允许被其他更高优先级的硬件中断再次打断，完全不破坏系统实时性。          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.1.2 Cortex-M4 NVIC 优先级分级与 `BASEPRI` 保护线

FreeRTOS 在 Cortex-M3/M4F 上没有采用暴力关闭全局中断的 `PRIMASK` 寄存器，而是巧妙利用了 **`BASEPRI` 寄存器**，将硬件中断划分为两级世界：

```
Cortex-M4 硬件中断物理分层 (以 PY32F403 3-bit NVIC 为例，数值越小硬件优先级越高):

优先级 0 (最高) ──► NMI 不可屏蔽中断 / HardFault 硬件错误
优先级 1        │   【零延迟极速中断区 (Zero Latency ISR)】
优先级 2        │   ★ 绝对不能调用任何 FreeRTOS API (包括 FromISR 版本)！
                │   ★ 实时性极高（用于电机短路保护、高精度 FOC 电流环）；
                ▼   ★ 即使 FreeRTOS 进入了临界区，也绝不屏蔽它们！
─────────────────── configLIBRARY_MAX_SYSCALL = 3 (BASEPRI 设定的安全边界: 0x60) ───
优先级 3 (阈值) ▲
优先级 4        │   【受管中断区 (FreeRTOS Managed ISR)】
优先级 5        │   ★ 可以安全调用 FreeRTOS 的 xxxFromISR() 系列 API；
优先级 6        │   ★ 在任务进入临界区时，会被 BASEPRI 暂时屏蔽，确保内核数据安全；
优先级 7 (最低) ──► SysTick / PendSV (内核自身异常必须配置在最低优先级 0xE0)
```

---

## 8.2 ISR ➔ Task 交互黄金标准模板与 `FromISR` API 契约

这是整个 FreeRTOS 开发中最关键的模板，必须形成肌肉记忆：

```c
/* ======================= 中断服务程序 ISR 端 ======================= */
void USART1_IRQHandler(void)
{
    /* ① 第一步: 定义并初始化高优先级任务唤醒标志为 pdFALSE (极其关键!) */
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /* ② 第二步: 清除硬件中断标志位，读取外设寄存器数据 (上半部极速操作) */
    if (USART1->SR & USART_SR_RXNE)
    {
        uint8_t rxData = (uint8_t)(USART1->DR & 0xFF);

        /* ③ 第三步: 调用 FromISR 版本 API 发送数据，传入唤醒标志指针 */
        xQueueSendFromISR(g.xUartQueue, &rxData, &xHigherPriorityTaskWoken);
    }

    /* ④ 第四步: 中断退出前，仅调用一次宏，根据标志决定是否请求上下文切换 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* ======================= 专职后台 Worker 任务端 ======================= */
static void Task_UartWorker(void *pvParameters)
{
    (void)pvParameters;
    uint8_t byte;

    for (;;)
    {
        /* 阻塞等待中断发来数据，无数据时 0% CPU 占用 */
        if (xQueueReceive(g.xUartQueue, &byte, portMAX_DELAY) == pdPASS)
        {
            /* 下半部: 放心执行复杂的通信协议解析 */
            Protocol_Parse_Byte(byte);
        }
    }
}
```

### 8.2.1 任务版本 API vs `FromISR` 版本 API 的核心契约差异

| 对比维度 | 任务上下文 API (如 `xQueueSend`) | 中断上下文专属 API (如 `xQueueSendFromISR`) |
| :--- | :--- | :--- |
| **调用环境** | 仅限在 Task 任务函数中调用 | **仅限在中断服务程序 (ISR) 中调用** |
| **阻塞与超时** | 支持设置 `xTicksToWait` (如 `portMAX_DELAY`) | **❌ 绝无超时参数！永远非阻塞直接返回** |
| **唤醒反馈** | 由调度器自动决策唤醒与即刻抢占 | **多出一个 `pxHigherPriorityTaskWoken` 输出指针参数** |
| **上下文切换** | 在 API 内部自动完成上下文切换 | **API 内部绝不切换，由开发者在 ISR 末尾统一请求** |

---

## 8.3 `pxHigherPriorityTaskWoken` 与延迟切换机理

初学者经常困惑：“为什么 `FromISR` API 不在内部直接切换任务，非要多传一个参数，并在最后一行调用 `portYIELD_FROM_ISR`？”  
**这体现了实时操作系统极其严谨的架构美感：**

### 8.3.1 为什么必须在 ISR 最末尾统一切换？
1. **防止中断嵌套中途被切走：**  
   如果 `xQueueSendFromISR` 内部立刻执行上下文切换，当前中断就会被硬生生挂起在中途，导致 CPU 栈帧错乱，破坏硬件中断返回机制；
2. **批量操作效率最优化：**  
   如果一个中断里连续调用了 3 次 `xQueueSendFromISR`（如读取 FIFO 中的 3 个字节），共用同一个 `xHigherPriorityTaskWoken` 变量。前一次置为 `pdTRUE` 后，后续不会被覆盖；在中断最末尾**只需发起一次任务切换**，避免了 3 次无谓的上下文颠簸！

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 portYIELD_FROM_ISR 与 PendSV 延迟切换时序                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   硬件中断触发 ──► 执行 ISR 上半部代码                                      │
│                         │                                                   │
│                         ├──► xQueueSendFromISR(..., &xWoken)                │
│                         │    (内核检测到 Worker 任务优先级高于当前被打断的任务)│
│                         │    (自动将 xWoken 置为 pdTRUE)                    │
│                         │                                                   │
│                         └──► portYIELD_FROM_ISR(xWoken)                     │
│                              (仅设置 NVIC 的 PendSV 悬起标志位，此时暂不切换)│
│                                                                             │
│   CPU 执行硬件中断异常返回指令 (BX LR)                                      │
│   (确保所有外设中断、嵌套中断全部彻底退出干净！)                             │
│                         │                                                   │
│                         ▼                                                   │
│   NVIC 响应最低优先级的 PendSV 异常中断 ──► 【执行纯净的任务上下文切换！】  │
│                         │                                                   │
│                         ▼                                                   │
│   CPU 立即进入高优先级的 Worker 任务执行！                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> ⚠️ **漏写 `portYIELD_FROM_ISR` 的严重后果：**  
> 如果开发者忘记在 ISR 末尾写这一行，被唤醒的高优先级 Worker 任务**不会立刻运行**！它必须等到下一次系统时钟节拍（SysTick）中断到来时才会被动调度，白白产生了最多 1ms 的不确定延迟，彻底破坏了实时响应保证！

---

## 8.4 临界区保护 (Critical Section) 深度解构

### 8.4.1 什么是临界区？原子操作守护
在多任务和多中断并发环境下，很多 C 语言代码看起来只有一行，但在汇编指令层面是由**多条指令**组成的：
```c
global_counter++;  /* 看起来是一行，实际是 3 条汇编指令！ */
```
```assembly
LDR  r0, [addr]    ; 1. 从内存读入寄存器
ADD  r0, r0, #1    ; 2. 寄存器加 1
STR  r0, [addr]    ; 3. 寄存器写回内存
```
如果在执行完指令 1 后，突然发生硬件中断打断并修改了 `global_counter`，随后任务恢复执行指令 3，就会**将中断写入的最新值硬生生覆盖掉**，造成严重的数据踩踏！  
**临界区 (Critical Section)** 就是将这段代码保护起来，保证其在执行期间绝不被打断（原子化执行）。

### 8.4.2 任务级临界区 vs 中断级临界区

```c
/* ================= 1. 在普通任务上下文中保护 ================= */
taskENTER_CRITICAL();
{
    /* 在 Cortex-M4 上，内核设置 BASEPRI 屏蔽受管中断
     * 支持嵌套调用：内部通过 uxCriticalNesting 计数器管理
     */
    g_system_ticks++;
}
taskEXIT_CRITICAL();

/* ================= 2. 在中断服务程序 ISR 中保护 ================= */
UBaseType_t uxSavedInterruptStatus;

uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
{
    /* 保存当前中断屏蔽状态，进一步加锁 */
    g_shared_hw_resource++;
}
taskEXIT_CRITICAL_FROM_ISR(uxSavedInterruptStatus);
```

### 8.4.3 三种保护机制终极对比与选型

| 保护方案 | 物理机制 | 对中断响应的影响 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **`taskENTER_CRITICAL`** | 设置 `BASEPRI` 屏蔽受管中断 | 增加中断延迟（微秒级） | **< 10μs 极短变量修改、标志位翻转** |
| **`vTaskSuspendAll`** | 仅锁住调度器，**不关任何中断** | **完全零中断延迟！** | 耗时稍长的链表遍历（不能调用阻塞 API） |
| **互斥锁 `Mutex`** | 操作系统级互斥对象 | **完全零中断延迟！** | 毫秒级长耗时共享资源保护（支持阻塞等待） |

> 🔴 **临界区三项不可饶恕之罪（违者必定死锁卡死）：**  
> 1. **绝不能在临界区内调用 `vTaskDelay()`**（中断被关，时钟节拍无法推进，任务永远无法唤醒！）；  
> 2. **绝不能在临界区内调用任何带阻塞超时的 API**（如 `xQueueReceive(q, buf, portMAX_DELAY)`）；  
> 3. **绝不能在临界区内执行长耗时大循环或慢速串口打印**（会导致外部中断发生严重丢包）。

---

## 8.5 中断延迟 (Latency) 分析与实战基准代码

### 8.5.1 中断延迟由哪些部分构成？
中断延迟是指从外部硬件引脚产生信号，到 CPU 开始执行 ISR 第一条指令的时间：

$$\text{总中断延迟} = \text{硬件采样与 NVIC 仲裁} + \text{硬件自动压栈 (8个寄存器)} + \text{临界区屏蔽时间 (最大不确定因素)}$$

在 8MHz 主频的 PY32F403 上：
- 硬件固有延迟仅约 **1.5 ~ 2.0 微秒**；
- 如果你的软件在某个任务里执行了 `taskENTER_CRITICAL()` 耗时 50μs，那么这 50μs 期间到来的受管中断将被迫延后整整 50μs 才能响应！**因此临界区代码必须越短越好（原则上 < 10μs）。**

### 8.5.2 TIM6 硬件定时器与队列实战代码

以下为生产级规范的硬件定时器配置与中断通信范式：

```c
static TIM_HandleTypeDef s_htim6;

/* 1. 硬件定时器与 NVIC 初始化 (500ms 周期) */
void APP_Tim6Config(void)
{
    __HAL_RCC_TIM6_CLK_ENABLE();

    s_htim6.Instance         = TIM6;
    s_htim6.Init.Prescaler   = 7999;     /* 8MHz / 8000 = 1kHz 时钟基准 */
    s_htim6.Init.Period      = 499;      /* 1kHz / 500 = 2Hz (500ms 触发一次) */
    s_htim6.Init.CounterMode = TIM_COUNTERMODE_UP;
    HAL_TIM_Base_Init(&s_htim6);

    /* 配置 NVIC 优先级: 必须 ≥ configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY */
    HAL_NVIC_SetPriority(TIM6_IRQn, 5, 0); /* 优先级 5 受管，安全调用 FromISR */
    HAL_NVIC_EnableIRQ(TIM6_IRQn);

    /* 启动定时器中断 */
    HAL_TIM_Base_Start_IT(&s_htim6);
}

/* 2. TIM6 中断服务程序 */
void TIM6_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /* 清除硬件定时器更新中断标志 */
    HAL_TIM_IRQHandler(&s_htim6);

    g.ulIsrCount++;

    /* 向上半部队列发送计数值 */
    xQueueSendFromISR(g.xIsrQueue, &g.ulIsrCount, &xHigherPriorityTaskWoken);

    /* 统一请求上下文切换 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## 8.6 中断管理 7 大致命陷阱、决策树与复盘自测

### 8.6.1 中断管理秒级决策树

```
                         中断服务程序 (ISR) 需要与系统交互？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
     [ 中断通知任务 ]                                  [ 保护共享资源 ]
           │                                                 │
     有数据包要传递吗？                                访问频次与耗时？
     ├─ 是 ──► 【xQueueSendFromISR】                   ├─ 极短操作 (<10μs) ──► 【taskENTER_CRITICAL】
     │                                                 │
     └─ 否 (纯信号) ──► 【vTaskNotifyGiveFromISR】      ├─ 稍长操作 (<100μs, 多任务) ──► 【互斥锁 Mutex】
         (最快最省，单播首选!)                         │
                                                       └─ 中断写，任务读 ──► 声明为 【volatile】 变量
```

### 8.6.2 中断开发 7 大致命陷阱

| 致命陷阱 | 严重后果与症状 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **在 ISR 中调用普通任务 API** | 没有 `FromISR` 后缀的 API 包含临界区和调度锁，中断内调用直接崩溃 | 中断中严格只允许调用带 `FromISR` 后缀的 API |
| **在 ISR 中使用阻塞超时** | 中断没有任务上下文，不可挂起，直接触发内核硬断言卡死 | `FromISR` 系列 API 绝无超时参数，永远立即返回 |
| **漏写 `portYIELD_FROM_ISR`** | 高优先级任务醒来却没及时抢占，产生高达 1 个 Tick 的滞后 | 中断退出前的最后一行必须配对调用该宏 |
| **先开启外设中断、后创建队列** | 硬件先触发中断向 NULL 队列写入数据，开机瞬间 HardFault | **系统初始化铁律：先创建所有 IPC 对象，最后开启中断** |
| **外设优先级数值小于 MAX_SYSCALL** | 不受管的高优先级中断误调用了 FromISR API，破坏内核链表 | 调用 API 的外设优先级数值必须 $\ge$ `MAX_SYSCALL` |
| **在临界区内调用 `vTaskDelay`** | 调度器时钟被屏蔽，延时永远不会到期，系统全局死锁 | 临界区内严禁任何阻塞或可能引发调度的 API |
| **共享变量漏加 `volatile`** | 编译器优化导致任务只读寄存器缓存，感知不到 ISR 的修改 | 所有被中断修改、被任务读取的全局变量必须加 `volatile` |

---

## 8.7 第8课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **两阶段模型：** 为什么工业级代码强调中断“上半部极速响应 + 下半部慢速处理”？
- [ ] **BASEPRI 机制：** Cortex-M4 上 FreeRTOS 临界区为何使用的是 `BASEPRI` 而非 `PRIMASK`？零延迟中断是如何实现的？
- [ ] **FromISR 差异：** `FromISR` 系列 API 相比普通任务 API，在参数和阻塞行为上有哪两大根本区别？
- [ ] **延迟切换：** 为什么任务切换要推迟到 ISR 最末尾由 `portYIELD_FROM_ISR` 发起？
- [ ] **初始化时序：** 为什么在 `main()` 函数中，队列与任务的创建必须严格早于硬件外设中断的开启？
- [ ] **临界区铁律：** 为什么在 `taskENTER_CRITICAL` 中绝对不能调用 `vTaskDelay`？
- [ ] **实操验证：** 编译烧录 `Lesson8_Interrupt_Basic`，通过 TIM6 中断与队列稳定驱动 LED 翻转。

***

# 第二部分：第9课 — 资源管理

> **核心主题：** 共享资源安全与死锁防御 (Resource Management) — 保护三层次、调度器锁与死锁破局
> 
> **📎 配套代码文件:**
> - `Lesson9_Resource_Basic.md` — 基础版：调度器锁 + 死锁检测与自动恢复 + 优先级反转与继承全套实战

***

```
▎ 课程目标： 掌握 vTaskSuspendAll/xTaskResumeAll 调度器锁、死锁四大成因与预防、以及 Mutex 优先级继承全生命周期
            构建“临界区 ➔ 调度器锁 ➔ 互斥锁”的三层资源防护体系金字塔
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 前置成果： 第8课已掌握 NVIC 中断分层、BASEPRI 临界区保护与两阶段解耦
```

---

## 9.1 共享资源防护金字塔：三层体系全景深度比对

在实时多任务系统中，只要有多个执行上下文（任务与任务、任务与中断）同时访问同一块内存变量或硬件外设，就会产生**数据竞态与破坏**。  
FreeRTOS 提供了**三种不同粒度与开销的资源保护手段**，构成了一个阶梯式的防护金字塔：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       FreeRTOS 共享资源保护三层金字塔                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                      ▲                                                      │
│                     / \                                                     │
│                    /   \         [ 层次 1: 硬件临界区 taskENTER_CRITICAL ]  │
│                   /  1  \        • 物理机制: BASEPRI 屏蔽所有受管中断       │
│                  /───────\       • 中断延迟代价: ❌ 产生微秒级中断延迟      │
│                 /         \      • 适用耗时: 极短原子操作 (< 10 微秒)       │
│                /     2     \                                                │
│               /─────────────\    [ 层次 2: 调度器锁 vTaskSuspendAll ]       │
│              /               \   • 物理机制: 暂停任务调度，【完全不关中断】 │
│             /        3        \  • 中断延迟代价: ✅ 零中断延迟，ISR 照常跑! │
│            /───────────────────\ • 适用耗时: 多变量一致性更新 (10~100 微秒) │
│                                                                             │
│   [ 层次 3: 互斥锁 Mutex ]                                                  │
│   • 物理机制: OS 级所有权对象，内置【优先级继承】防反转                     │
│   • 中断延迟代价: ✅ 零中断延迟，调度照常跑 (仅争抢同锁的任务挂起)          │
│   • 适用耗时: 毫秒级长耗时保护、允许调用阻塞 API、多任务竞争                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.1.1 三种保护方案核心指标横向评测

| 对比指标 | 硬件临界区 (`taskENTER_CRITICAL`) | 调度器锁 (`vTaskSuspendAll`) | 互斥锁 (`Mutex`) |
| :--- | :--- | :--- | :--- |
| **物理作用目标** | 屏蔽低于安全阈值的硬件中断 | 暂停调度器切换，**不影响任何中断** | 仅阻塞争抢同一把锁的任务 |
| **底层执行开销** | 极低 ($\approx 3$ 个时钟周期) | 极低 (`uxSchedulerSuspended++`) | 中等 (需出入临界区与链表操作) |
| **中断延迟影响** | ❌ 会产生微秒级中断响应延迟 | **✅ 零中断延迟，ISR 微秒级响应** | **✅ 零中断延迟** |
| **能否调用阻塞 API** | ❌ **绝对严禁！** (会死锁) | ❌ **绝对严禁！** (会死锁) | **✅ 允许！** (支持超时挂起) |
| **防优先级反转** | 不存在任务切换，天然免疫 | 不存在任务切换，天然免疫 | **✅ 内置优先级继承机制** |
| **典型工程用例** | 变量递增 `cnt++`、单寄存器置位 | 多传感器多变量“全有或全无”一致性更新 | 独占串口长打印、I2C/SPI 总线读写 |

---

## 9.2 调度器锁 (`vTaskSuspendAll` / `xTaskResumeAll`) 深度剖析

很多初学者不知道：**在既需要保证原子性、又绝对不能丢失外部高频硬件中断的场景下，调度器锁是比临界区好上十倍的武器！**

### 9.2.1 核心物理机制：`uxSchedulerSuspended` 计数器
当任务调用 `vTaskSuspendAll()` 时，内核实际上只做了一件事：
```c
/* 内核底层逻辑简述 (tasks.c) */
void vTaskSuspendAll( void )
{
    ++uxSchedulerSuspended;  /* 调度器挂起嵌套计数器加 1 */
}
```
- **此时硬件中断完全开放！** 外部 UART 接收中断、ADC 转换完成中断依然能够微秒级响应并执行；
- 中断服务程序依然可以正常调用 `xQueueSendFromISR()` 向队列压入数据；
- **唯一的变化是：** 哪怕中断唤醒了更高优先级的任务，内核也会**将上下文切换强制暂存推迟**，绝不在调度器锁期间切走当前任务！

### 9.2.2 黄金应用场景：多变量强一致性原子更新
假设系统有一个包含温度、湿度和时间戳的传感器全局数据包：

```c
typedef struct {
    int16_t  temperature;
    int16_t  humidity;
    uint32_t timestamp;
} SensorReport_t;

SensorReport_t g_sensorData;
```
如果使用普通的赋值，当更新完 `temperature` 后突然发生任务切换，显示任务读取到的就会是**“新的温度 + 旧的湿度”**，产生严重的数据撕裂！  
**调度器锁是解决此问题的最完美方案：**

```c
/* 生产者任务: 原子更新多变量 */
vTaskSuspendAll();
{
    /* 在此代码块内，没有其他任何任务能够打断我们 */
    g_sensorData.temperature = Read_Adc_Temp();
    g_sensorData.humidity    = Read_Adc_Humid();
    g_sensorData.timestamp   = xTaskGetTickCount();
    /* 数据包具有绝对强一致性！*/
}
xTaskResumeAll();
```

### 9.2.3 锁内不可逾越的禁忌红线
在 `vTaskSuspendAll()` 与 `xTaskResumeAll()` 之间：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     调度器锁期间的“绝对禁止行为”                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ 绝不能调用任何可能引发任务阻塞或让出 CPU 的 API：                       │
│     • vTaskDelay() / vTaskDelayUntil()                                      │
│     • xQueueReceive(q, buf, portMAX_DELAY) / xQueueSend(..., timeout)       │
│     • xSemaphoreTake(sem, timeout)                                          │
│     • taskYIELD()                                                           │
│                                                                             │
│  原因：当前任务把调度器锁死了，它一旦尝试休眠，调度器却无法调度其他任务，  │
│        CPU 将陷入死锁瘫痪！                                                 │
│                                                                             │
│  ✅ 允许做的事：                                                            │
│     • 纯内存读写、数据拷贝 (memcpy / memset)、数据转换与校验和计算；        │
│     • 极短的无阻塞硬件寄存器读取。                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9.3 并发终极梦魇：死锁 (Deadlock) 的本质与破局之道

### 9.3.1 死锁的四大充分必要条件（Dijkstra / Coffman 准则）
在多任务操作系统中，死锁**仅在以下四个条件同时成立时才会发生**。只要成功打破其中任何一条，死锁便绝不可能产生！

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          死锁四大充分必要条件                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ① 互斥条件 (Mutual Exclusion)：资源是独占排他的，一次只能被一个任务持有；  │
│  ② 持有并等待 (Hold and Wait)：任务攥着手中的锁 A，同时在排队等待另一把锁 B；│
│  ③ 不可剥夺 (No Preemption)：别人手里的锁不能被强行夺走，只能由持有者释放；│
│  ④ 循环等待 (Circular Wait)：Task A 等 Task B 的锁，Task B 等 Task A 的锁，  │
│     构成了一个死咬不放的闭环等待链！                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3.2 典型死锁时序：交叉加锁的灾难

```
死锁发生的经典悲剧时序:
时间轴 ──────────────────────────────────────────────────────────────────────►

Task A:  [ 成功获取锁 A ] ──► 想拿锁 B ──► [ 发现锁 B 被占，进入阻塞休眠... ] 💀
                                ▲               │ (Task A 攥着锁 A 不放)
                                │               ▼
Task B:  [ 成功获取锁 B ] ──► 想拿锁 A ──► [ 发现锁 A 被占，进入阻塞休眠... ] 💀
                                                (Task B 攥着锁 B 不放)

两任务互相等待对方释放手中的锁，且彼此都在休眠无法唤醒 ──► 系统永久卡死！
```

### 9.3.3 工程破局双剑客：预防与检测

#### 破局之剑 1：统一加锁全局顺序 (Lock Ordering) —— 【破坏条件 ④】
这是工业级系统中最优雅、开销为零的根治方案。  
**全系统规定死：所有任务如果需要同时获取多把锁，必须严格遵循相同的顺序！**
- 比如规定全系统统一按照 `锁 A ➔ 锁 B` 的顺序加锁；
- Task A：`Take(A) ➔ Take(B)`；
- Task B：`Take(A) ➔ Take(B)`（哪怕 Task B 只需要一瞬间的锁 A，也必须先拿 A 再拿 B）；
- **效果：** 循环等待闭环被彻底粉碎，从根本上杜绝死锁可能！

#### 破局之剑 2：带超时的锁回退与随机退避 (Timeout & Backoff) —— 【破坏条件 ②】
绝对不要无脑传入 `portMAX_DELAY` 无限期死等！

```c
/* 生产级防御范式: 带超时检测与主动放弃 */
if (xSemaphoreTake(xMutexA, pdMS_TO_TICKS(100)) == pdPASS)
{
    /* 获取第一把锁成功，尝试获取第二把锁，设置超时限制 */
    if (xSemaphoreTake(xMutexB, pdMS_TO_TICKS(100)) == pdPASS)
    {
        /* 两把锁都拿到了，安全执行临界操作 */
        Execute_Critical_Work();
        
        xSemaphoreGive(xMutexB);
    }
    else
    {
        /* ⚠️ 警报: 获取锁 B 超时！可能存在死锁风险！*/
        g_deadlock_alert_count++;
        /* 立刻主动释放手中的锁 A，打破“持有并等待”条件！*/
    }
    
    xSemaphoreGive(xMutexA);
}
```

#### 调试武器：`xSemaphoreGetMutexHolder()`
在调试器中，如果你怀疑系统死锁了，可以通过该 API 查看某把锁当前正被哪个任务持有：
```c
TaskHandle_t hHolder = xSemaphoreGetMutexHolder(g_xMutexA);
/* 在 Keil Watch 窗口中查看 hHolder->pcTaskName，立刻抓住凶手！ */
```

---

## 9.4 优先级反转与继承的终极攻防 (Mars Pathfinder 实录)

### 9.4.1 火星探路者号真实事故全景复盘
1997 年 7 月，火星探路者号 (Mars Pathfinder) 着陆数天后，开始频繁发生无法解释的系统全面复位，险些导致数十亿美元任务失败：
1. **低优先级任务 (气象气象仪采集)：** 周期性获取信息总线锁 (Mutex)；
2. **高优先级任务 (姿态与轨道控制)：** 每秒必须按时通过信息总线传输姿态数据；
3. **中优先级任务 (长耗时通信与通信数据解包)：** 耗时很长，但**完全不需要信息总线**；
4. **事故爆发：** 气象任务拿到总线锁后，中优先级通信任务突然就绪并抢占了 CPU；气象任务无法释放锁，导致最高优先级的姿态控制任务被中优先级通信任务间接卡死！系统看门狗超时，触发火星车整机复位！

### 9.4.2 互斥锁优先级继承 (Priority Inheritance) 运行机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 优先级继承动态调权全生命周期时序图                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   时间轴 ────────────────────────────────────────────────────────►          │
│                                                                             │
│   姿态控制 Task (Pri 4):              ░░░░░░░░████████████████████          │
│                                       等待总线锁 ▲ 拿到锁极速响应!           │
│                                                                             │
│   通信任务 Task (Pri 3):      ░░░░░░░░        ░░░░░░░░████████████          │
│                               想抢占但抢不过 ▲ (因为低优先已被提至 Pri 4)   │
│                                                                             │
│   气象任务 Task (Pri 2):  ██████████████████████                            │
│                           拿总线锁 ▲ 内核将其优先级临时提权为 Pri 4!        │
│                                    (火速跑完并释放总线锁，优先级降回 Pri 2) │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9.5 实战标杆代码与硬件测试体系

### 9.5.1 调度器锁 vs 临界区实战对比模板

```c
/* ================= 演示 1: 调度器锁保护多变量一致性 ================= */
static void Task_SchedulerLockDemo(void *pvParameters)
{
    (void)pvParameters;
    
    for (;;)
    {
        /* 暂停调度器: 外部中断照常响应，但无任务能切走我们 */
        vTaskSuspendAll();
        {
            g.sensor.temp  = Read_Sensor_Channel_0();
            g.sensor.humid = Read_Sensor_Channel_1();
            g.sensor.valid = 1;
        }
        xTaskResumeAll();

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* ================= 演示 2: 死锁超时检测与自愈机制 ================= */
static void Task_DeadlockRecoveryDemo(void *pvParameters)
{
    (void)pvParameters;

    for (;;)
    {
        if (xSemaphoreTake(g.xMutexA, pdMS_TO_TICKS(20)) == pdPASS)
        {
            /* 模拟轻微延迟 */
            vTaskDelay(pdMS_TO_TICKS(10));

            /* 获取第二把锁，超时设为 200ms */
            if (xSemaphoreTake(g.xMutexB, pdMS_TO_TICKS(200)) == pdPASS)
            {
                /* 正常临界区 */
                Perform_Safe_Operation();
                xSemaphoreGive(g.xMutexB);
            }
            else
            {
                /* 检测到死锁征兆: LED 翻转告警，并放弃锁 A 避让 */
                HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_12);
            }
            
            xSemaphoreGive(g.xMutexA);
        }
        
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

## 9.6 资源保护终极决策树、常见陷阱与复盘自测

### 9.6.1 资源保护方案终极决策树

```
                         多任务间需要保护共享资源？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
     [ 包含中断上下文参与 (ISR ↔ Task) ]             [ 仅在纯任务上下文中 (Task ↔ Task) ]
           │                                                 │
     保护操作执行耗时多长？                            保护操作执行耗时与复杂度？
     ├─ 极短 (< 10μs) 变量读写/标志翻转                ├─ 极短 (< 10μs) ──► 【taskENTER_CRITICAL】
     │   └─► 【taskENTER_CRITICAL】                    │
     │                                                 ├─ 10 ~ 100μs, 多变量一致性 且【不需调阻塞API】
     └─ 中断写，任务读                                 │   └─► 🎉【vTaskSuspendAll 调度器锁】(零中断延迟!)
         └─► 声明为 【volatile】 变量                  │
             或通过 【FromISR 队列/通知】 传递         └─ > 100μs, 或【需要调用阻塞等待 API】
                                                           └─► 🎉【互斥锁 Mutex】(带优先级继承!)
```

### 9.6.2 资源管理 6 大致命陷阱

| 致命陷阱 | 严重后果与症状 | 正确工程防御准则 |
| :--- | :--- | :--- |
| **调度器锁内调用 `vTaskDelay`** | 调度器无法调度其他任务，当前任务永久挂起死锁 | 调度器锁内严禁任何阻塞或可能让出 CPU 的 API |
| **交叉加锁顺序不一致** | Task A 拿 A 等 B，Task B 拿 B 等 A，系统瞬间暴毙 | **全工程严格执行统一加锁顺序 (如始终先 A 后 B)** |
| **滥用 `portMAX_DELAY` 盲等锁** | 一旦发生死锁，任务永久卡死，系统彻底丧失自愈能力 | 优先使用有限超时机制（如 200ms），超时后主动回退 |
| **误用临界区保护长耗时代码** | 屏蔽中断时间长达毫秒级，外部通信与传感器严重丢包 | 耗时超过 10μs 的操作必须改用调度器锁或 Mutex |
| **普通 Mutex 重复嵌套获取** | 任务在子函数里再次 Take，自己把自己永久锁死 | 嵌套函数调用必须换用 `Recursive Mutex` 递归锁 |
| **任务异常退出前未释放锁** | 任务 `vTaskDelete` 或异常终止，导致锁被永久持有死锁 | 确保获取与释放严格处于同一作用域，异常分支统一清理 |

---

## 9.7 第9课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **金字塔三层：** 能否向他人清晰阐述临界区、调度器锁与互斥锁在中断延迟和适用场景上的本质区别？
- [ ] **调度器锁机理：** 为什么在 `vTaskSuspendAll` 期间外部硬件中断依然可以正常响应？
- [ ] **一致性更新：** 面对“温度与湿度双变量必须原子同步更新”的需求，为什么调度器锁是最佳选择？
- [ ] **死锁四条件：** 能否背出死锁四大充分必要条件，并指出“统一加锁顺序”破坏了哪一个条件？
- [ ] **超时自愈：** 在申请第二把锁时设置超时时间并主动释放第一把锁，破坏了死锁的哪个条件？
- [ ] **继承救赎：** 能否讲清火星探路者号优先级反转的根因，以及 Mutex 是如何通过动态调权解决反转的？
- [ ] **实操验证：** 编译烧录 `Lesson9_Resource_Basic`，通过 LED 观察调度器锁、死锁检测与优先级继承效果。

***

# 第二部分：第10课 — 内存管理 (Memory Management)

> **核心主题：** 内存全景与底层硬件机制 (Memory Management) — 从 FreeRTOS 堆算法到 Cortex-M4 启动与调用栈
> 
> **📎 配套代码文件:**
> - `Lesson10_Memory_Basic.md` — 基础版：heap_4 动态分配原理 + 栈高水位监控
> - `Lesson10_Memory_Advanced.md` — 进阶版：静态内存分配 + 自定义内存池 + Bootloader 跳转规范

***

```
▎ 课程目标： 掌握 heap_1~heap_5 算法选型、静态/动态内存分配、栈高水位与金丝雀溢出检测
            读懂 Keil Program Size 四项指标与 64KB RAM 极限剪裁，攻克 Cortex-M4 调用栈、向量表重定位与 Bootloader 跳转
▎ 硬件平台： PY32F403-STK (Cortex-M4F, 64KB SRAM, 384KB Flash)
▎ 前置成果： 第9课已掌握共享资源防护三层体系与死锁防御
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       第10课 四大核心知识模块导航全景                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [ 模块一: FreeRTOS 动态与静态内存模型 ] ──► 10.1 ~ 10.6                    │
│  • MCU 物理内存模型、heap_1~5 算法原理、静态分配、栈溢出检测与高水位监控   │
│                                                                             │
│  [ 模块二: 编译构建与 MCU 资源预算实战 ] ──► 10.7 ~ 10.9                    │
│  • Keil Program Size 四项指标、64KB RAM 跑 FreeRTOS+LVGL 剪裁、容量分级    │
│                                                                             │
│  [ 模块三: Cortex-M4 硬件调用栈与内存底层机理 ] ──► 10.10 ~ 10.12           │
│  • PC/LR/SP 寄存器协同、链接脚本 scatter 散列加载自定义段、数组越界踩踏     │
│                                                                             │
│  [ 模块四: 系统启动、向量表与 Bootloader 跳转生态 ] ──► 10.13 ~ 10.16       │
│  • 单片机加电至 main() 全流程、VTOR 重定位、Bootloader 5 步安全跳转协议     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 模块一：FreeRTOS 动态与静态内存模型

### 10.1 前置概念：MCU 内存模型

在讲 FreeRTOS 内存管理之前，先了解 PY32F403 (Cortex-M4F) 的物理内存布局：

```
PY32F403 内存布局 (简化)
┌──────────────────────────┐ 0xFFFFFFFF
│   系统控制空间 (SCS)      │ NVIC, SysTick, MPU 等
├──────────────────────────┤ 0xE0000000
│   外部设备 (Peripheral)   │ GPIO, USART, SPI, ADC...
├──────────────────────────┤ 0x40000000
│   SRAM (64KB)            │ ← .data + .bss + heap + stack
├──────────────────────────┤ 0x20000000
│   Flash (384KB)          │ ← .text (代码) + .rodata (常量)
├──────────────────────────┤ 0x08000000
│   Flash 别名区            │
└──────────────────────────┘ 0x00000000
```

### 各段的含义

| 段 | 位置 | 说明 |
|----|------|------|
| `.text` | Flash | 可执行代码 |
| `.rodata` | Flash | 只读常量 (const 变量, 字符串字面量) |
| `.data` | Flash→SRAM | 已初始化全局/静态变量，启动时从 Flash 拷贝到 SRAM |
| `.bss` | SRAM | 未初始化全局/静态变量，启动时清零 |
| **Heap** | SRAM | 动态分配区 (`malloc`/`pvPortMalloc` 从这里拿) |
| **Stack** | SRAM | 函数调用栈，每个 FreeRTOS 任务有自己的栈 |

```
 SRAM 内部布局 (由链接脚本决定)

┌──────────────────────┐ SRAM 高地址 (0x20010000)
│       Stack          │ ← MSP (主栈指针) 向下增长
├──────────────────────┤
│         ↓            │
│       (空闲)          │
│         ↑            │
├──────────────────────┤
│       Heap           │ ← FreeRTOS 管理的池 (ucHeap[])
├──────────────────────┤ ← ucHeap[] 数组起点
│       .bss           │
├──────────────────────┤
│       .data          │
└──────────────────────┘ SRAM 低地址 (0x20000000)
```

### 10.1.1 内存映射的源头 — 为什么 Flash 是 0x08000000？

**这个问题芯片厂商（Puya）不负责决定，而是 ARM 公司定死的。**

ARM 在设计 Cortex-M 系列时，预先将 4GB 地址空间切成固定分区，所有 Cortex-M 芯片必须遵守：

```
ARM Cortex-M 架构规范定义的内存分区 (所有Cortex-M芯片通用)
══════════════════════════════════════════════════════════════
4GB = 2^32 bytes, 从 0x00000000 到 0xFFFFFFFF

┌──────────────────────────────┐ 0xFFFFFFFF
│  System (0.5GB)              │ 私有外设总线:
│  NVIC · SysTick · MPU · SCB  │ 内核自带的控制寄存器
│  Debug · ITM · TPIU          │ 地址范围: 0xE0000000-0xFFFFFFFF
├──────────────────────────────┤ 0xE0000000
│  External Device (1.0GB)     │ 外部设备, 如 QSPI Flash XIP
│  (保留给扩展使用)             │ 地址范围: 0xA0000000-0xDFFFFFFF
├──────────────────────────────┤ 0xA0000000
│  External RAM (1.0GB)        │ 外扩 SRAM/SDRAM (FSMC/FMC)
│  (保留给扩展使用)             │ 地址范围: 0x60000000-0x9FFFFFFF
├──────────────────────────────┤ 0x60000000
│  Peripheral (0.5GB)          │ 片上外设寄存器:
│  你的 GPIO/USART/SPI/TIM 都在这│ 地址范围: 0x40000000-0x5FFFFFFF
├──────────────────────────────┤ 0x40000000
│  SRAM (0.5GB)                │ 片上静态 RAM:
│  .data + .bss + heap + stack │ 地址范围: 0x20000000-0x3FFFFFFF
├──────────────────────────────┤ 0x20000000
│  Code (0.5GB)                │ 片上 Flash 映射到这里:
│  .text + .rodata + 向量表     │ 0x08000000 起 (厂家约定)
│  0x00000000 上电后映射到Flash │ 0x00000000 起 (启动别名)
└──────────────────────────────┘ 0x00000000
```

```
关键结论:

  0x08000000 → ARM 规范中 Code 区的"片上Flash标准起始地址" — 行规，不是Puya决定的
  0x20000000 → ARM 规范中 SRAM 区的基地址 — 所有 Cortex-M 都一样
  0x40000000 → ARM 规范中外设区的基地址 — 你的 GPIOA 一定在这附近
  0xE0000000 → ARM 规范中系统控制区的基地址 — NVIC/SysTick 在这里

  做一个思想实验:
    如果 Puya 把 PY32F403 的 Flash 放在 0x12345678 行不行？
    技术上可行，但:
    ❌ 所有 ARM 调试器 (J-Link/ST-Link/DAP) 不认识
    ❌ 所有 ARM 编译器默认链接脚本全废
    ❌ Keil/IAR 的 Flash 烧录算法要重写
    ❌ 工程师换芯片学习成本爆炸
    → 所以没人这么干, 整个行业统一遵守 ARM 的内存分区规范
```

---

### 10.1.2 CPU 核心寄存器 — R0-R15 到底是什么？

**一句话：R0-R15 是 CPU 核心里面实实在在的物理触发器，不在总线上，没有内存地址。**

```
┌──────────────────────────────────────────────────────────────────┐
│                    Cortex-M4F 处理器核心 (硅片微观视图)             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              寄存器堆 (Register File)                       │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     ┌─────┐             │  │
│  │  │ R0  │ │ R1  │ │ R2  │ │ R3  │ ... │ R15 │   ← 物理D触发器│  │
│  │  │32bit│ │32bit│ │32bit│ │32bit│     │32bit│     每个值32个  │  │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘     └──┬──┘     晶体管组成   │  │
│  │     │       │       │       │           │                    │  │
│  │     └───────┴───────┼───────┴───────────┘                    │  │
│  │                     │                                        │  │
│  │          ┌──────────┴──────────┐                             │  │
│  │          │   寄存器选择器       │ ← 指令中的Rn字段(4-bit)       │  │
│  │          │   多路复用器 (MUX)  │   直接选通哪一组触发器        │  │
│  │          └──────────┬──────────┘                             │  │
│  │                     │                                        │  │
│  │          ┌──────────┴──────────┐                             │  │
│  │          │       ALU           │ ← 加法器/移位器/逻辑单元     │  │
│  │          │   ADD R0, R1, R2    │                             │  │
│  │          └─────────────────────┘                             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                            │                                     │
│                  总线接口 (Bus Interface)                         │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │
               AHB/APB 总线 (有地址的世界)
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                     ▼
   Flash控制器          SRAM控制器            GPIO模块
   @ 0x08000000        @ 0x20000000          @ 0x40020000
```

#### 核心寄存器 vs 系统外设寄存器 — 关键区分

两者都是芯片上的物理触发器（晶体管构成），区别在于**挂在总线的哪一侧**：

| | CPU 核心寄存器 (R0-R15) | 外设寄存器 (GPIO->ODR) |
|---|---|---|
| **物理本质** | 硅片上的 D 触发器 | 硅片上的 D 触发器 ← 一样！ |
| **位置** | CPU 核心内部，紧贴 ALU | GPIO 模块内部，挂在 AHB 总线上 |
| **访问方式** | 指令编码中的 4-bit 字段直接选通 | 通过地址总线 + 数据总线访问 |
| **有地址吗** | ❌ 没有。指令里写的是寄存器编号 | ✅ 有。`0x40020014` |
| **延迟** | 0 周期（直连 ALU） | 3~5 周期（穿过多级总线） |
| **类比** | 你**口袋里的钥匙**，伸手就摸到 | 物业**前台墙上的钥匙柜(第3排第5格)** |

```
汇编验证 — 同一个操作，不同编码:

; ✅ 核心寄存器 R0：指令内部编码，零延迟
MOVS    R0, #0x55       ; 编码: 0x2055
                        ; bit[10:8]=000 → 硬件直接选通R0触发器
                        ; 周期: 1个时钟

; ✅ 外设寄存器 GPIOA_ODR：必须走总线
LDR     R1, =0x40020014 ; 1. 先把目标地址放进 R1
STR     R0, [R1]        ; 2. CPU把地址放总线 → GPIOA解码器认出 →
                        ;    连通ODR触发器 → R0的值写入ODR触发器
                        ;    周期: 3~5个时钟
```

```
⚠️ 常见混淆澄清:

  SCB->VTOR (向量表偏移寄存器) — 有地址 0xE000ED08
  NVIC->ISER (中断使能寄存器)   — 有地址 0xE000E100
  SysTick->LOAD (滴答重载值)    — 有地址 0xE000E014

  这些虽然和 CPU 内核紧密相关，但它们是"系统外设寄存器"
  挂在私有外设总线上 → 有地址, 可以用 LDR/STR 访问

  R0-R15, PRIMASK, CONTROL — 这些才是真正的"核心寄存器"
  只在 CPU 核心里面 → 无地址, 只能用 MOV/MRS/MSR 访问
```

---

### 10.1.3 Load-Store 架构 — CPU 如何与外部世界"交流"

**Cortex-M 遵循 RISC 的铁律：ALU（运算单元）只能直接操作 R0-R15。总线上的一切必须先"搬进"R寄存器，处理完再"搬出去"。**

```
                    R0-R15 (CPU 的"双手")
                    ═══════════════════════
                    ALU 只能碰到这些
                    │
                    │  LDR (Load)   = 从外部"抓"进来
                    │  STR (Store)  = 从手中"丢"出去
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   GPIOA->ODR   SRAM[0x100]  USART1->DR
   0x40020014   0x20000100   0x40013804
   (输出数据)    (变量)       (串口数据)

═══════════════════════════════════════════════
  铁律: ALU 不直接碰总线, 总线上的东西不直接进 ALU
═══════════════════════════════════════════════
```

```
C 代码 → 汇编 → 硬件动作 (完整演示)

目标: GPIOA Pin5 输出高电平 → GPIOA->BSRR = (1 << 5);

; ═══ 编译后的汇编 ═══
MOVS    R0, #0x20          ; ① 手(R0)里抓了 0x20
LDR     R1, =0x40020018    ; ② 手(R1)里抓了 GPIOA_BSRR 的地址
STR     R0, [R1]           ; ③ 把手(R0)里的值, 扔到 R1 指向的地址

; ═══ STR 指令时硬件发生的细节 ═══
; 1. CPU 在地址总线上输出 0x40020018
; 2. AHB 总线矩阵根据地址高位 (0x4002...) 选通 GPIOA 模块
; 3. GPIOA 内部地址解码器认出 0x18 → 连通 BSRR 那 32 个触发器
; 4. CPU 在数据总线上输出 R0 的值 (0x00000020)
; 5. GPIOA 模块把数据总线的值锁存到 BSRR 触发器
; 6. BSRR 触发器的 bit5 变成 1 → 硬件自动将 Pin5 输出拉高
; 7. 完成!
```

```
总结: 为什么"挂在总线上的物理寄存器一定有地址"？

                    CPU
                     │
            ┌────────┴────────┐
            │   总线互连矩阵    │
            │  (Bus Matrix)   │
            └───┬──┬──┬──┬───┘
                │  │  │  │
           ┌────┘  │  │  └────┐
           ▼       ▼  ▼       ▼
        GPIOA   USART1 SRAM  Flash
       0x4002  0x4001 0x2000 0x0800
       0800    3800   0000   0000

  每个从设备内部都有一个"地址解码器":
    if (总线地址 ∈ [我的起始地址, 我的结束地址])
        响应! 把数据放上总线或从总线取走
    else
        沉默，不响应

  → 没有地址 = 总线上的"隐形人" = CPU 无法和它通信
  → 地址是总线上物理寄存器的"身份证"
```

---

### 10.1.4 横向对比 — ARM vs 8051 vs AVR vs PIC

**不是所有架构都要求 Load-Store。8位单片机的世界百花齐放。**

```
┌─────────────────────────────────────────────────────────────────┐
│              三大 8-bit 架构与 ARM Cortex-M 对比                  │
├────────────┬──────────────┬──────────────┬──────────────────────┤
│  架构      │  通用寄存器   │  能直接操作   │  设计哲学              │
│            │              │  内存/SFR?   │                      │
├────────────┼──────────────┼──────────────┼──────────────────────┤
│ ARM CM4   │ R0-R15(16个) │ ❌ 绝对不能   │ 严格Load-Store(RISC) │
│ 8051      │ R0-R7 (8个)  │ ✅ 可以!     │ 内存-寄存器混合操作    │
│           │              │ INC 30h      │                      │
│ AVR       │ R0-R31(32个) │ ❌ 不能      │ Load-Store (最像ARM)  │
│           │              │ SFR需IN/OUT  │                      │
│ PIC       │ W (1个!)     │ ❌ 不能      │ 单累加器              │
└────────────┴──────────────┴──────────────┴──────────────────────┘
```

#### 8051 为什么可以不遵守 Load-Store？

```
8051 的内存架构截然不同:

  ARM Cortex-M4:
    ┌─────┐   总线(慢)    ┌─────────────────┐
    │ CPU │ ←──────────→ │ Flash/SRAM/外设   │
    │ 核心 │   ~10倍延迟   │ 在总线另一侧      │
    └─────┘              └─────────────────┘
    寄存器在CPU内           所有外部设备在总线另一侧
    速度: ?ns              速度: ?ns × N

  8051:
    ┌──────────────────────────────────────────┐
    │          同一块硅片，同一条内部总线         │
    │  ┌────┐      ┌───────────────────┐       │
    │  │ALU │←────→│ 内部RAM (128B)     │       │
    │  │R0~7│      │ 含R0~R7(4组Bank)  │       │
    │  └────┘      │ SFR(P0,P1,SBUF…)  │       │
    │              └───────────────────┘       │
    └──────────────────────────────────────────┘
    都是内部RAM，延迟一样，ALU可以直接操作任何地址

ARM 强制 Load-Store 的原因:
  → Flash/SRAM/外设在总线另一侧
  → 物理距离远，延迟大
  → 如果不隔离，流水线频繁停顿等待总线 → 性能崩溃
  → 所以 "中间隔一条总线" = "必须 Load-Store" 的铁律诞生
```

#### 直观代码对比

```asm
; ═══════════════════════════════════════════════
; 同一个操作 "把一个变量加1" 在三架构上的实现
; ═══════════════════════════════════════════════

; ARM Cortex-M (Load-Store, 必经R寄存器):
LDR  R0, [R1]        ; 从地址取到寄存器
ADD  R0, R0, #1      ; 在寄存器里加
STR  R0, [R1]        ; 写回地址

; 8051 (直接操作内存):
INC  30h             ; 直接把0x30地址的值加1！
                     ; 一条指令，不经任何寄存器中转

; AVR (Load-Store，和ARM同类):
LD   R16, X+         ; 从指针X指向的地址取到R16
INC  R16             ; R16加1
ST   X, R16          ; 写回指针X指向的地址
```

---

## 10.2 FreeRTOS 的堆：它在管什么？

FreeRTOS 动态创建对象时，需要从堆中分配 RAM：

| 动态创建 API | 分配对象 | 典型大小 |
|-------------|---------|---------|
| `xTaskCreate()` | TCB + 任务栈 | ~100 + stack_size 字节 |
| `xQueueCreate()` | 队列控制块 + 数据区 | ~80 + item_size × length |
| `xSemaphoreCreateBinary()` | 信号量控制块 | ~40 字节 |
| `xTimerCreate()` | 定时器控制块 | ~48 字节 |
| `xEventGroupCreate()` | 事件组控制块 | ~32 字节 |

所有这些 API 内部都调用 `pvPortMalloc()` 来分配内存。**五种 `heap_x.c` 的核心区别就是 `pvPortMalloc()` 和 `vPortFree()` 的实现方式不同。**

---

## 10.3 heap_1 ~ heap_5 深度对比

### 10.3.1 heap_1 — "只借不还" 模型

**一句话总结：** 只能分配，不能释放。最简单、最确定。

```
实现原理:

   ucHeap[N]
   ┌────────────────────────────────────────────────────┐
   │████████████████████│                                │
   │    已分配           │      空闲                      │
   │   TaskA  TaskB      │                                │
   └────────────────────┴──────────────────────────────┘
                         ↑
                    xNextFreeByte
                   (单向增长指针, 永不回退)

   算法:
   pvPortMalloc(size):
       1. 对齐到 8 字节边界
       2. 检查 xNextFreeByte + size <= TOTAL_HEAP_SIZE
       3. 返回 &ucHeap[xNextFreeByte], 然后 xNextFreeByte += size
       4. 如果越界 → 返回 NULL

   vPortFree(pv):  // ⚠️ 空函数! 什么也不做
       (void)pv;
```

**流程图：**

```
pvPortMalloc(size)
      │
      ▼
  字节对齐检查
      │
      ▼
  剩余空间 >= size?
    │          │
   否          是
    │          │
    ▼          ▼
  返回NULL   返回指针
             xNextFreeByte += size
```

**适用场景：**
- ✅ 只有创建、从不删除的静态系统（如安全关键系统）
- ✅ 单次初始化所有对象 → 启动调度器 → 永远运行
- ✅ 确定性极高，无碎片

**不适用场景：**
- ❌ 需要运行时创建/删除任务
- ❌ 需要动态创建/删除队列或信号量

---

### 10.3.2 heap_2 — "有空就回头填" 模型（经典链表）

**一句话总结：** 支持释放，但不会合并相邻空闲块 → 碎片化严重。

```
实现原理 (单向链表):

   ucHeap[N]
   ┌──────────────────────────────────────────────────────┐
   │ [Free] len=200  │ [Used] len=64  │ [Free] len=120  │
   │  next ──────────▶  next ─────────▶  next ─────▶NULL │
   └──────────────────────────────────────────────────────┘

   数据结构:
   typedef struct A_BLOCK_LINK {
       struct A_BLOCK_LINK *pxNextFreeBlock;
       size_t xBlockSize;  // 含链表头大小
   } BlockLink_t;

   算法核心:
   pvPortMalloc(size):
       遍历空闲链表 → 找到第一个足够大的块 (First-Fit)
       → 如果块比请求大很多 → 切分: 前段分配, 后段归还空闲链表
   
   vPortFree(pv):
       把块插回空闲链表头 (简单O(1)插入)
       ⚠️ 不合并相邻空闲块!
```

**碎片化演示（这是heap_2最大的坑）：**

```
初始: [████████████        空闲块 1024字节        ]
分配A: [██A 100███│        空闲 924                 ]
分配B: [██A 100███│ B 200  │     空闲 724           ]
释放A: [ 空洞100 │ B 200  │     空闲 724           ]
          ↑ 这块100字节被释放了, 但和后面的空闲块不相邻
分配C: [██C 50███│空洞50│ B 200  │   空闲 724       ]
释放B: [██C 50███│空洞50│空洞200│   空闲 724        ]
          ↑ 三个小空洞, 彼此不相邻, 永远无法合并!

最终: 虽然总空闲 = 50+200+724 = 974 字节
     但如果来了一个 300 字节的请求 → 分配失败! ← 碎片化之痛
```

**适用场景：**
- ⚠️ 创建/删除次数极少，且分配的块大小固定
- ⚠️ 实际上很少推荐使用，大部分场景被 heap_4 取代

---

### 10.3.3 heap_3 — "套壳标准库" 模型

**一句话总结：** 直接封装 C 标准库的 `malloc()`/`free()`，加上临界区保护。

```
实现原理:

   pvPortMalloc(size):
       vTaskSuspendAll();        // 挂起调度器
       pv = malloc(size);        // 调 C 库 malloc
       xTaskResumeAll();         // 恢复调度器
       return pv;

   vPortFree(pv):
       vTaskSuspendAll();
       free(pv);                 // 调 C 库 free
       xTaskResumeAll();

   ⚠️ heap_3 不定义 ucHeap 数组!
      堆内存来自链接脚本保留的 heap 区, 由 C 运行时库管理
```

**注意事项：**
- 需要链接脚本保留足够的 heap 空间（不像 heap_1/2/4/5 用 `ucHeap[]` 静态数组）
- 依赖编译器 C 库的 `malloc` 实现质量
- 线程安全性由 `vTaskSuspendAll()` 保证
- 某些 MCU 的 C 库 `malloc` 不是线程安全的 → 必须用 `vTaskSuspendAll()`

**适用场景：**
- ⚠️ 希望用标准库 `malloc`（例如已有大量依赖 `malloc` 的第三方代码）
- ⚠️ 不推荐在资源受限的 MCU 上使用（C 库 malloc 通常开销较大且不可控）

---

### 10.3.4 heap_4 — "黄金方案"（最常用 ⭐⭐⭐⭐⭐）

**一句话总结：** heap_2 的升级版——支持相邻空闲块合并，解决碎片化问题。

```
实现原理:

   与 heap_2 的最大区别:
   释放块时, 检查前后邻居是否空闲 → 如果是, 合并成一个更大的空闲块!

   空闲链表按地址排序 (而非 heap_2 的插入顺序)
   
   vPortFree(pv) 关键流程:
   ┌──────────────────────────────────────────────┐
   │  1. 找到待释放块 pxBlock                     │
   │  2. 检查 pxBlock->前一个物理邻居是否空闲?     │
   │     ├ 是 → 合并, 从前一个块开始              │
   │     └ 否 → 保持 pxBlock                      │
   │  3. 检查 pxBlock->后一个物理邻居是否空闲?     │
   │     ├ 是 → 合并, 从链表中移除后邻居          │
   │     └ 否 → 保持                              │
   │  4. 将合并后的块按地址有序插入空闲链表        │
   └──────────────────────────────────────────────┘
```

**合并示意（对比 heap_2）：**

```
heap_2 的悲剧:
释放A后: [空洞100 │ B 200 │   空闲724   ]
释放B后: [空洞100 │空洞200│   空闲724   ]  ← 永远不合, 碎片化!

heap_4 的拯救:
释放A后: [空洞100 │ B 200 │   空闲724   ]
             ↑ 此时 B 还在用, 不合
释放B后: [空洞100 │空洞200│   空闲724   ]
             ↓ B 释放时发现: 左边是空洞, 右边是空洞
         → 合并! [         空闲1024               ]
             ↑ 完美恢复! 无碎片!
```

**适用场景：**
- ✅ **这是默认推荐方案，几乎所有场景**
- ✅ 需要动态创建/删除任务、队列、信号量
- ✅ 内存块大小不一，生命周期各异
- ✅ 绝大多数 FreeRTOS 项目

---

### 10.3.5 heap_5 — "跨多块内存" 模型

**一句话总结：** heap_4 的算法 + 支持多块不连续内存池。

```
实现原理:

   heap_1~4 只有一个 ucHeap[N] 连续数组
   heap_5 可以有多个不连续的内存块!

   ┌────────────────────────────────────────────────┐
   │  内存块1: SRAM 内部 64KB (0x20000000)         │
   │  内存块2: CCM RAM 64KB (0x10000000)           │
   │  内存块3: 外部 PSRAM 512KB (0x60000000)       │
   └────────────────────────────────────────────────┘
           ↓ 通过 vPortDefineHeapRegions() 注册

   初始化流程:
   1. 定义 HeapRegion_t 数组:
      const HeapRegion_t xHeapRegions[] = {
          { (uint8_t *)0x20000000, 0x10000 }, // 64KB SRAM
          { (uint8_t *)0x10000000, 0x10000 }, // 64KB CCM
          { NULL, 0 }                          // 终止标记
      };
   2. vPortDefineHeapRegions(xHeapRegions);  // 必须比任何 malloc 都早
   3. pvPortMalloc 会自动跨区域查找可用空间
```

**适用场景：**
- ✅ MCU 有 CCM RAM（紧密耦合内存，快但 DMA 不可用）
- ✅ 有外部 SDRAM / PSRAM
- ✅ 有多个不连续的 SRAM 区域需要统一管理
- ⚠️ PY32F403 只有一块 64KB SRAM，一般不涉及

---

### 10.3.6 五种 heap 速查表

```
┌────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│  特性   │ heap_1   │ heap_2   │ heap_3   │ heap_4   │ heap_5   │
├────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│  释放    │ ❌       │ ✅       │ ✅       │ ✅       │ ✅       │
│  合并    │ N/A     │ ❌       │ 依库     │ ✅       │ ✅       │
│  碎片    │ 无       │ ⚠️ 严重   │ 依库     │ 极低     │ 极低     │
│  确定性  │ ★★★★★   │ ★★★     │ ★★      │ ★★★★    │ ★★★★    │
│  多内存块│ ❌       │ ❌       │ 依链接   │ ❌       │ ✅       │
│  RAM开销 │ 极小     │ 中       │ 中大     │ 中       │ 中+      │
│  推荐度  │ 特殊场景 │ ❌ 不推荐 │ 谨慎     │ ⭐默认   │ 特殊场景 │
└────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**一句话选型指南：** 99% 的项目用 `heap_4.c`，只有明确知道为什么需要其他方案时才换。

---

## 10.4 静态内存分配（Static Allocation）

### 10.4.1 为什么需要静态分配？

动态分配（`pvPortMalloc`）的问题：
- 分配可能失败（返回 NULL），需要处理错误 → 代码变复杂
- 碎片化（即使 heap_4 也有极端情况的碎片）
- 在安全关键系统（如汽车、医疗）中，动态分配通常被禁止
- 内存使用不可预测

静态分配的哲学：**"在编译时就确定一切"** — 确定性最高的方案。

### 10.4.2 如何启用静态分配

```c
// FreeRTOSConfig.h
#define configSUPPORT_STATIC_ALLOCATION    1   // 启用!
#define configSUPPORT_DYNAMIC_ALLOCATION   1   // 可同时保留动态
```

### 10.4.3 动态 vs 静态 API 对照表

| 动态 API | 静态 API | 需要额外提供的 |
|----------|---------|--------------|
| `xTaskCreate()` | `xTaskCreateStatic()` | 栈数组 + TCB |
| `xQueueCreate()` | `xQueueCreateStatic()` | 队列数据区 + 控制块 |
| `xSemaphoreCreateBinary()` | `xSemaphoreCreateBinaryStatic()` | 控制块 |
| `xTimerCreate()` | `xTimerCreateStatic()` | 控制块 |
| `xEventGroupCreate()` | `xEventGroupCreateStatic()` | 控制块 |

### 10.4.4 代码示例：静态任务创建

```c
/* ── 静态任务创建 ── */

/* Stack buffer — 栈是静态分配的数组 */
#define TASK1_STACK_SIZE    256
static StackType_t  xTask1Stack[TASK1_STACK_SIZE];  // 栈数组

/* TCB — 任务控制块 */
static StaticTask_t xTask1TCB;                      // TCB 结构体

/* Task function */
static void vTask1Handler(void *pvParameters)
{
    for (;;) {
        LED_TOGGLE();
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main(void)
{
    /* 静态创建任务 */
    TaskHandle_t hTask1 = xTaskCreateStatic(
        vTask1Handler,          // 任务函数
        "Task1",                // 名字
        TASK1_STACK_SIZE,       // 栈大小
        NULL,                   // 参数
        1,                      // 优先级
        xTask1Stack,            // ← 栈数组指针
        &xTask1TCB              // ← TCB 指针
    );

    vTaskStartScheduler();
    for (;;);
}
```

### 10.4.5 代码示例：静态创建队列

```c
/* Queue buffer — 数据存储区 */
#define QUEUE_LENGTH    5
#define ITEM_SIZE       sizeof(uint32_t)
static uint8_t       xQueueStorage[QUEUE_LENGTH * ITEM_SIZE];
static StaticQueue_t xQueueControlBlock;

/* Create static queue */
QueueHandle_t hQueue = xQueueCreateStatic(
    QUEUE_LENGTH,
    ITEM_SIZE,
    xQueueStorage,          // ← 数据存储区
    &xQueueControlBlock     // ← 控制块
);
```

### 10.4.6 静态分配的内存布局对比

```
动态分配:
┌──────────────────────────────────────────────────────┐
│                ucHeap[] (堆)                          │
│  [TCB_A][栈A][TCB_B][栈B][队列C][信号量D]...        │
│    ↑ 运行时分配, 位置不确定, 可能碎片化               │
└──────────────────────────────────────────────────────┘

静态分配:
┌──────────────────────────────────────────────────────┐
│  .bss / .data 段                                      │
│  xTask1Stack[256]   ← 栈, 编译时确定位置              │
│  xTask1TCB          ← TCB, 编译时确定位置             │
│  xTask2Stack[128]   ← 栈, 编译时确定位置              │
│  xTask2TCB          ← TCB, 编译时确定位置             │
│  xQueueStorage[20]  ← 队列, 编译时确定位置            │
│  xQueueCB           ← 控制块, 编译时确定位置          │
│    ↑ 布局完全确定, 可通过 .map 文件精确查看           │
└──────────────────────────────────────────────────────┘
```

### 10.4.7 动态 vs 静态 选型指南

| 场景 | 推荐 | 原因 |
|------|------|------|
| 安全关键系统 (ISO 26262) | 静态 | 无运行时分配，确定性最强 |
| 任务/队列数量固定 | 静态 | 内存使用完全可计算 |
| 需要运行时创建/销毁 | 动态 | 静态做不到 |
| 内存非常受限 | 静态 | 精确控制每字节 |
| 快速原型开发 | 动态 | 方便，用宏切换 heap_x.c 即可 |
| 库/中间件开发 | 同时支持 | 用 `configSUPPORT_*_ALLOCATION` 宏 |

---

## 10.5 栈溢出检测

### 10.5.1 问题背景

每个 FreeRTOS 任务有自己的栈。栈溢出 = **嵌入式系统的头号杀手**：

```
任务栈布局 (向下增长, Cortex-M 惯例):
高地址 ┌──────────────────┐ ← pxStack (栈顶, 初始 SP)
       │  ...              │
       │  local var a      │
       │  local var b      │
       │  函数调用帧         │
       │        ↓          │ ← SP 向下增长
       │        ↓          │
       │  [未使用区]         │
       │   ...             │
低地址 └──────────────────┘ ← 栈底

当 SP 越过栈底 → 栈溢出!
→ 踩踏 TCB 或相邻任务的栈 → 系统行为不可预测!
```

### 10.5.2 两种检测方法

**方法1：指针检查**

```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW    1  // 方法1

// 原理: 每次任务切换时, 检查任务栈顶是否仍为合法地址
// 开销: 极小 (仅检查一个地址)
// 缺点: 仅检测溢出发生在任务"当前运行周期"内，可能漏检
```

**方法2：金丝雀标记（Canary）**

```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW    2  // 方法2 (更彻底)

// 原理: 创建任务时, 用已知模式 (0xA5A5A5A5) 填充栈底区域
//       每次切换时检查这段"金丝雀"是否被踩踏
```

```
金丝雀原理可视化:

任务栈创建时:
高地址 ┌──────────────────┐
       │  pxStack         │
       │  ...             │
       │  [栈空间]         │
       │  A5 A5 A5 A5     │  ← 金丝雀
       │  A5 A5 A5 A5     │     末尾 16 字节
       │  A5 A5 A5 A5     │     填充为 0xA5A5A5A5
低地址 └──────────────────┘

溢出后:
高地址 ┌──────────────────┐
       │  pxStack         │
       │  ...             │
       │  [栈膨胀]         │
       │  02 01 00 FF     │  ← 这些值覆盖了金丝雀! 💀
       │  45 33 22 11     │
       │  A5 A5 08 7F     │  ← 部分被覆盖
低地址 └──────────────────┘
     → 检查发现 != 0xA5A5A5A5 → 触发钩子!
```

### 10.5.3 栈溢出钩子函数

```c
// 必须实现的回调函数
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    // 溢出发生! 这个函数中不要用栈!
    // 不要调用 printf (会消耗大量栈)
    // 不要调用 vTaskDelay (需要调度器)
    
    // 正确做法:
    // 1. 记录到全局变量 (放在 .bss, 不消耗栈)
    // 2. 硬件复位/进入死循环
    // 3. 通过 UART 直接发送 (轮询模式, 不用中断)
    
    volatile uint32_t *p = (uint32_t *)0xE000ED0C;  // NVIC SCB AIRCR
    *p = (0x05FA << 16) | (1 << 2);                 // 系统复位
    
    // 或者进入安全死循环
    // taskDISABLE_INTERRUPTS();
    // for (;;);
}
```

### 10.5.4 方法1 vs 方法2 对比

```
┌────────┬──────────────────────┬──────────────────────┐
│        │ 方法1 (CHECK=1)       │ 方法2 (CHECK=2)       │
├────────┼──────────────────────┼──────────────────────┤
│  机制   │ 检查栈顶是否合法      │ 检查金丝雀 + 栈顶     │
│  时机   │ 任务切换时            │ 任务切换时            │
│  精度   │ 可能漏检!!!          │ 更不容易漏检          │
│  开销   │ 几乎可忽略            │ O(n) 每任务，n很小    │
│  推荐   │ 开发阶段用2           │ 发布后可用1或保留2    │
└────────┴──────────────────────┴──────────────────────┘
```

---

## 10.6 uxTaskGetStackHighWaterMark() — 栈使用率监控

### 10.6.1 原理

```
高水位线 (High Water Mark) = 任务栈历史上最少剩余空间

高地址 ┌──────────────────┐
       │  [已使用区]        │
       │  峰值位置 ← 高水位  │ ← uxTaskGetStackHighWaterMark() 返回
       │  ████████████████ │     这个值! (单位: words 不是 bytes)
       │  ████████████████ │
       │  ████████████████ │
       ├──────────────────┤ ← 从栈底到此, 曾经被用过的最远位置
       │  [从未使用区]      │
       │  A5 A5 A5 A5     │ ← 一直是初始值
       │  A5 A5 A5 A5     │
低地址 └──────────────────┘
```

**工作机制（依赖 configCHECK_FOR_STACK_OVERFLOW=2）：**
1. 创建任务时，用 0xA5 填满整个栈
2. 运行时，任务使用栈 → 有些 0xA5 被覆盖为实际数据
3. `uxTaskGetStackHighWaterMark()` 从栈底向上扫描，数出还剩多少"干净的 0xA5"
4. 返回剩余字数 (word count, 不是 byte count!)

### 10.6.2 代码示例：栈监控任务

```c
/* Global structure for system monitoring */
typedef struct {
    uint32_t ulTask1StackMin;   // Task1 minimum free stack (words)
    uint32_t ulTask2StackMin;
    uint32_t ulTask3StackMin;
    uint32_t ulHeapMin;         // Minimum free heap (bytes)
} SystemMonitor_t;

static SystemMonitor_t g_xMonitor = {0};

/* Monitoring task — runs at lowest priority, reports system health */
static void vMonitorTask(void *pvParameters)
{
    /* 从其他模块获得 TaskHandle */
    extern TaskHandle_t hTask1, hTask2, hTask3;

    for (;;) {
        /* Read stack high water marks (returns words!) */
        g_xMonitor.ulTask1StackMin = uxTaskGetStackHighWaterMark(hTask1);
        g_xMonitor.ulTask2StackMin = uxTaskGetStackHighWaterMark(hTask2);
        g_xMonitor.ulTask3StackMin = uxTaskGetStackHighWaterMark(hTask3);

        /* Read heap free size */
        g_xMonitor.ulHeapMin = xPortGetFreeHeapSize();

        /* Runtime check — if dangerously low, assert or warn */
        configASSERT(g_xMonitor.ulTask1StackMin > 32);  // at least 32 words
        configASSERT(g_xMonitor.ulTask3StackMin > 32);
        configASSERT(g_xMonitor.ulHeapMin > 512);        // at least 512 bytes

        /* Periodic report via UART */
        xUART_Printf("Stack: T1=%u T2=%u T3=%u | HeapFree=%u\r\n",
                      (unsigned)g_xMonitor.ulTask1StackMin,
                      (unsigned)g_xMonitor.ulTask2StackMin,
                      (unsigned)g_xMonitor.ulTask3StackMin,
                      (unsigned)g_xMonitor.ulHeapMin);

        vTaskDelay(pdMS_TO_TICKS(5000));  // report every 5 seconds
    }
}
```

### 10.6.3 栈大小测定经验法则

```
┌──────────────────────────────────────────────────────┐
│              Stack Sizing 方法论                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  步骤1: 初始估算                                       │
│    - 简单任务: 128 words (512 bytes)                  │
│    - 中等任务: 256 words (1024 bytes)                 │
│    - 复杂任务 (含 printf / fat-fs): 512~1024 words    │
│                                                      │
│  步骤2: 运行并监控                                     │
│    - 让系统跑最坏情况的测试                              │
│    - 读取 uxTaskGetStackHighWaterMark()               │
│                                                      │
│  步骤3: 安全余量                                       │
│    - 最终大小 = HighWaterMark × 1.3 + 中断嵌套深度     │
│    - 对于 Cortex-M4F: 中断帧 = 16 words               │
│    - 务必保留至少 64 words 余量                        │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 10.7 configTOTAL_HEAP_SIZE 调优

### 10.7.1 这个宏在控制什么？

```c
// FreeRTOSConfig.h
#define configTOTAL_HEAP_SIZE    ((size_t)(10 * 1024))  // 10KB

// heap_4.c 内部:
static uint8_t ucHeap[configTOTAL_HEAP_SIZE];  // 这就是整个堆!
```

**configTOTAL_HEAP_SIZE 就是一个静态数组的大小。** 设太小 → 动态分配失败；设太大 → 浪费 SRAM。

### 10.7.2 如何精确测定？

```
调优流程:

  步骤1: 设一个大的初始值 (如 20KB)
         ↓
  步骤2: 创建所有任务和对象后, 调用 xPortGetFreeHeapSize()
         剩余 = 初始 - 已用
         ↓
  步骤3: 运行完整测试, 持续监控 xPortGetFreeHeapSize()
         记录下最小值
         ↓
  步骤4: 最终值 = 已用最大值 × 1.2
         例: 已用 6KB → 设为 7.2KB → 取整 8KB
         ↓
  步骤5: 验证 — 设为最终值后重新编译运行
```

### 10.7.3 代码辅助：启动时打印堆使用

```c
void vPrintHeapStats(void)
{
    size_t xTotalHeap     = configTOTAL_HEAP_SIZE;
    size_t xFreeHeap      = xPortGetFreeHeapSize();
    size_t xUsedHeap      = xTotalHeap - xFreeHeap;
    size_t xMinEverFree   = xPortGetMinimumEverFreeHeapSize();

    /* Should be called BEFORE scheduler starts to avoid race */
    volatile char buf[128];
    snprintf((char *)buf, sizeof(buf),
             "\r\n=== Heap Stats ===\r\n"
             "Total:     %u bytes\r\n"
             "Used:      %u bytes (%u%%)\r\n"
             "Free:      %u bytes\r\n"
             "Min Free:  %u bytes\r\n",
             (unsigned)xTotalHeap,
             (unsigned)xUsedHeap,    (unsigned)(xUsedHeap * 100 / xTotalHeap),
             (unsigned)xFreeHeap,
             (unsigned)xMinEverFree);
    UART_SendString((char *)buf);
}
```

**重要：** 这个函数必须在 `vTaskStartScheduler()` **之前**调用，因为调度器启动后会立即创建 Idle Task 和可能的 Timer Task，此时测量才最准确。

---

## 10.8 自定义 pvPortMalloc

### 10.8.1 为什么要自定义？

场景：
- 用 TLSF（Two-Level Segregated Fit）等更高效的分配器替代 heap_4
- 需要内存池（Memory Pool）给特定尺寸的对象
- 需要调试信息（记录分配者、调用栈等）

### 10.8.2 实现方式：固定大小内存池

```c
/* ── MyMalloc.c 自定义内存管理 ── */

/* Pool-based allocator for fixed-size blocks (92 bytes each) */
#define POOL_BLOCK_SIZE     92
#define POOL_BLOCK_COUNT    32

typedef struct {
    uint8_t     aucBuffer[POOL_BLOCK_SIZE * POOL_BLOCK_COUNT]; // raw buffer
    uint8_t     aucBitmap[(POOL_BLOCK_COUNT + 7) / 8];         // allocation bitmap
    uint32_t    ulAllocCount;
    uint32_t    ulFreeCount;
    uint32_t    ulMaxUsed;
} FixedPool_t;

static FixedPool_t g_xPool = {0};

/* Allocate from pool */
static void *prvPoolAlloc(size_t xWantedSize)
{
    if (xWantedSize > POOL_BLOCK_SIZE) return NULL;

    /* Find first free block via bitmap scan */
    for (uint32_t i = 0; i < POOL_BLOCK_COUNT; i++) {
        uint32_t byte = i / 8;
        uint32_t bit  = i % 8;
        if (!(g_xPool.aucBitmap[byte] & (1 << bit))) {
            /* Block is free — allocate it */
            g_xPool.aucBitmap[byte] |= (1 << bit);
            g_xPool.ulAllocCount++;
            g_xPool.ulFreeCount--;
            if (g_xPool.ulAllocCount > g_xPool.ulMaxUsed)
                g_xPool.ulMaxUsed = g_xPool.ulAllocCount;
            return &g_xPool.aucBuffer[i * POOL_BLOCK_SIZE];
        }
    }
    return NULL;  // pool exhausted
}

/* Free back to pool */
static void prvPoolFree(void *pv)
{
    /* Calculate block index */
    uint32_t index = (uint32_t)((uint8_t *)pv - g_xPool.aucBuffer) / POOL_BLOCK_SIZE;
    
    if (index < POOL_BLOCK_COUNT) {
        g_xPool.aucBitmap[index / 8] &= ~(1 << (index % 8));
        g_xPool.ulFreeCount++;
        g_xPool.ulAllocCount--;
    }
}

/* ── Hooking into FreeRTOS ── */

void *pvPortMalloc(size_t xWantedSize)
{
    /* If size fits in our pool, use pool; otherwise fallback to heap_4 */
    if (xWantedSize <= POOL_BLOCK_SIZE) {
        void *pv = prvPoolAlloc(xWantedSize);
        if (pv != NULL) return pv;
    }
    /* Fallback to default heap_4 implementation */
    return _default_pvPortMalloc(xWantedSize);
}

void vPortFree(void *pv)
{
    /* Check if this pointer belongs to our pool */
    if ((uint8_t *)pv >= g_xPool.aucBuffer &&
        (uint8_t *)pv <  g_xPool.aucBuffer + sizeof(g_xPool.aucBuffer)) {
        prvPoolFree(pv);
        return;
    }
    /* Otherwise let heap_4 handle it */
    _default_vPortFree(pv);
}
```

### 10.8.3 调试增强版 pvPortMalloc

```c
/* Debug wrapper — tracks every allocation */
typedef struct {
    void       *ptr;           // allocated pointer
    size_t      size;          // requested size
    const char *pcCaller;      // __func__ of caller
    uint32_t    ulLine;        // __LINE__ of caller
} AllocRecord_t;

#define MAX_RECORDS    64
static AllocRecord_t g_axRecords[MAX_RECORDS];
static uint32_t      g_ulRecordIdx = 0;

/* DEBUG version — captures caller info */
void *pvPortMalloc_Debug(size_t xWantedSize, const char *caller, uint32_t line)
{
    void *pv = pvPortMalloc(xWantedSize);
    
    if (pv != NULL && g_ulRecordIdx < MAX_RECORDS) {
        g_axRecords[g_ulRecordIdx].ptr      = pv;
        g_axRecords[g_ulRecordIdx].size     = xWantedSize;
        g_axRecords[g_ulRecordIdx].pcCaller = caller;
        g_axRecords[g_ulRecordIdx].ulLine   = line;
        g_ulRecordIdx++;
    }
    
    /* Dump all records when allocation fails */
    if (pv == NULL) {
        UART_SendString("ALLOC FAILED! Active:\r\n");
        for (uint32_t i = 0; i < g_ulRecordIdx; i++) {
            char buf[64];
            snprintf(buf, sizeof(buf), "  %s:%u size=%u\r\n",
                     g_axRecords[i].pcCaller,
                     (unsigned)g_axRecords[i].ulLine,
                     (unsigned)g_axRecords[i].size);
            UART_SendString(buf);
        }
    }
    
    return pv;
}

/* Convenience macro — use everywhere instead of bare pvPortMalloc */
#define DEBUG_MALLOC(size)  pvPortMalloc_Debug((size), __func__, __LINE__)
```

---

## 第10课 前半部分速查图

```
┌──────────────────────────────────────────────────────────────────┐
│                     FreeRTOS 内存管理决策树                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  你的系统是否允许动态分配?                                         │
│     │                      │                                     │
│    否                     是                                     │
│     │                      │                                     │
│     ▼                      ▼                                     │
│  静态分配                  RAM 只有一块连续区域?                   │
│  (xTaskCreateStatic)        │          │                         │
│  ───────────────────       是          否                        │
│  ✅ 启用                    │          │                         │
│  configSUPPORT_STATIC_      ▼          ▼                         │
│  ALLOCATION = 1          动态创建/删除?  heap_5                  │
│                              │        │                          │
│  ✅ 每个对象独立栈          是        否                          │
│  + TCB 数组                 │         │                          │
│                              ▼         ▼                         │
│  ✅ 用 .map 文件验证      heap_4    heap_1                      │
│  内存使用量                 ⭐默认    简单系统                    │
│                                                                  │
│  第二步：防栈溢出                                                 │
│  • configCHECK_FOR_STACK_OVERFLOW = 2  (开发期)                  │
│  • vApplicationStackOverflowHook()                                │
│  • uxTaskGetStackHighWaterMark() → 串口周期上报                   │
│                                                                  │
│  第三步：堆大小调优                                               │
│  • 初始设大 → xPortGetFreeHeapSize() → 缩小到 120%                │
│  • 用 xPortGetMinimumEverFreeHeapSize() 观察最坏值                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 前半部分常见陷阱

1. **栈设太小** → 症状：随机崩溃、HardFault、数据错乱。解决：`uxTaskGetStackHighWaterMark()` 压测后加 30%
2. **堆设太小** → 症状：`xTaskCreate` 返回 `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY`。解决：用 `xPortGetFreeHeapSize()` 实测
3. **中断中使用动态分配** → 不能在 ISR 中调 `pvPortMalloc`！要用 `pvPortMalloc` 只能在任务上下文
4. **heap_2 碎片** → 如果每次分配/释放的大小不同，碎片会在数小时后让系统崩溃。**直接换 heap_4**
5. **configCHECK_FOR_STACK_OVERFLOW 只在任务切换时检查** → 即使方法2也检测不到"刚好溢出但没有任务切换时"的极端情况

---

## 模块二：编译构建与 MCU 资源预算实战

### 10.7 读懂 Keil 编译输出：Program Size 四项指标深度拆解
### 为什么必须读懂这一行？

每次 Keil 编译完成后输出：

```
Program Size: Code=88032 RO-data=85840 RW-data=108 ZI-data=52428
```

这行数字直接决定了 Flash 和 RAM 的占用。不理解它就无法做内存规划。

### 四项分别是什么？

```
┌──────────┬──────────────────────┬────────────────────────────────────┐
│   项目    │  存放位置              │  内容                               │
├──────────┼──────────────────────┼────────────────────────────────────┤
│  Code    │ Flash                │  机器指令 (.text)                     │
│  RO-data │ Flash                │  只读数据: const变量、字符串字面量      │
│  RW-data │ Flash (初始值副本)     │  已初始化全局变量的初始值               │
│          │ + RAM  (.data 段)     │  运行时变量本体                        │
│  ZI-data │ RAM   (.bss 段)      │  未初始化全局变量, 启动时清零            │
└──────────┴──────────────────────┴────────────────────────────────────┘
```

### 核心公式

```
Total Flash = Code + RO-data + RW-data
Total RAM   = RW-data + ZI-data + Stack + Heap
                     ↑               ↑        ↑
                  .data 段       .bss 段   未计入 Keil 输出!
                                            需要自己算!
```

### RW-data 的双重身份 — 最容易被误解

```c
int g_count = 42;   // ← 有初始值的全局变量, 产生 RW-data

/*
 * Flash 中存: 0x0000002A (42 的二进制) — 初始值副本
 *     ↑ 必须烧在 Flash 里, 断电不能丢

 * RAM  中存: 4 字节变量空间 (.data 段) — 运行时本体
 *     ↑ 程序运行时变量在这里
 *
 * Reset_Handler 做的事:
 *     memcpy(RAM的.data段, Flash的初始值副本, RW-data大小);
 *     // 把所有初始化值从 Flash 拷到 SRAM
 */
```

```
Flash (384KB)                       SRAM (64KB)
┌──────────────────────┐           ┌──────────────────────┐
│ Code = 88,032        │           │                      │
│  (你的所有函数)       │           │                      │
├──────────────────────┤           │                      │
│ RO-data = 85,840     │           │                      │
│  (const数组/字符串)   │           │                      │
├──────────────────────┤  拷贝     ├──────────────────────┤
│ RW-data 初始值副本    │──────▶   │ RW-data = 108       │ ← .data
│ (108 bytes)          │  Reset   │  g_count = 42        │
│  0x0000002A...       │  Handler │  (运行时变量在这)     │
└──────────────────────┘           ├──────────────────────┤
                                   │ ZI-data = 52,428    │ ← .bss (清零)
                                   │  g_buffer[50000]    │
                                   │  (启动时全部清 0)   │
                                   ├──────────────────────┤
                                   │  Heap  (ucHeap[])    │ ← FreeRTOS 堆
                                   │  Stack (MSP + 任务栈) │ ← 向下增长
                                   └──────────────────────┘
                                   64KB SRAM total
```

### 用实际数据算一遍

以 PY32F403 项目的实际输出为例：

```
Program Size: Code=88032 RO-data=85840 RW-data=108 ZI-data=52428

Flash 占用:
  Code:       88,032 bytes
  RO-data:    85,840 bytes
  RW-data:       108 bytes  (初始值副本)
  ────────────────────────
  Flash 总计: 173,980 bytes ≈ 170 KB
  总容量:     384 KB (393,216 bytes)
  使用率:     44.2%  ✅ 很充足

RAM 静态占用 (仅 .data + .bss):
  RW-data:       108 bytes  (.data 段)
  ZI-data:    52,428 bytes  (.bss 段)
  ────────────────────────
  静态总计:   52,536 bytes ≈ 51.3 KB
  总容量:     64 KB (65,536 bytes)
  使用率:     80.2%

  剩余给栈+堆: 13,000 bytes ≈ 12.7 KB
```

### ⚠️ 关键警告

**Keil 输出的 ZI-data 不包含栈和堆！** 很多工程师误以为 ZI-data 就是总 RAM 占用，实际上：

```
真正的 RAM 占用 = RW-data + ZI-data
                  + MSP 中断栈    (链接脚本 Stack_Size, 通常 1KB)
                  + Heap          (configTOTAL_HEAP_SIZE)
                  + 任务栈 × N    (动态分配时在 heap 中, 静态分配时在 .bss 中)
```

如果用静态分配（`xTaskCreateStatic`），任务栈在 `.bss` 段 → 已计入 ZI-data → Keil 输出更准确。

如果用动态分配（`xTaskCreate`），任务栈在 `ucHeap[]` 中 → 必须在 `configTOTAL_HEAP_SIZE` 中额外预留 → Keil 输出不体现！

---

### 10.8 极限挑战：64KB RAM 如何流畅运行 FreeRTOS + LVGL

### 问题背景

LVGL 的显示缓冲区是 RAM 大户，64KB SRAM 的 MCU 同时跑 LVGL + FreeRTOS 到底行不行？

### 结论先行

**不是"不能用"，而是"要用得更精明"。** 64KB SRAM + LVGL + FreeRTOS 是可行的，但必须满足两个条件：
1. 用静态分配（`xTaskCreateStatic`）
2. 严格控制每个任务的栈大小 + 缩小 LVGL 显示缓冲

### LVGL 显示缓冲区 — 最大的 RAM 消费者

```
┌──────────────────────────────────────────────────────────────────┐
│  LVGL 显示缓冲区方案对比 (240×320 屏幕, RGB565 = 2 字节/像素)       │
├──────────────────┬──────────────┬───────────┬───────────────────┤
│  方案              │  RAM 占用     │  性能      │  适用 MCU         │
├──────────────────┼──────────────┼───────────┼───────────────────┤
│  双全缓冲 ×2      │  307,200 B   │  最快     │  需要外部 SDRAM   │
│  单全缓冲 ×1      │  153,600 B   │  快       │  至少 256KB SRAM  │
│  1/4 屏缓冲 ×2   │   38,400 B   │  中等     │  128KB SRAM      │
│  1/10 屏缓冲 ×2  │   15,360 B   │  ✅ 推荐   │  64KB SRAM       │
│  1/20 屏缓冲 ×2  │    7,680 B   │  可接受    │  32KB SRAM       │
│  (直接刷屏)       │        0 B   │  很慢      │  极小 MCU         │
└──────────────────┴──────────────┴───────────┴───────────────────┘
```

```c
// 64KB SRAM 的推荐配置: 1/10 屏幕行缓冲
#define MY_DISP_HOR_RES     240

static lv_disp_draw_buf_t draw_buf;
static lv_color_t buf_1[MY_DISP_HOR_RES * 32];  // 240 × 32 × 2 = 15,360 bytes

lv_disp_draw_buf_init(&draw_buf, buf_1, NULL, MY_DISP_HOR_RES * 32);
// 单缓冲 15KB — 对 64KB MCU 来说是可接受的代价
```

### 三个真实场景模拟

```
┌─────────────────────────────────────────────────────────────────┐
│  场景A: 豪华配置 → 放不下 ❌                                      │
├─────────────────────────────────────────────────────────────────┤
│  双全缓冲 240×320×2×2 = 307KB → 已经超过 64KB，直接不可能         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  场景B: 静态分配 + 1/10行缓冲 → 健康 ✅                            │
├─────────────────────────────────────────────────────────────────┤
│  LVGL 1/10 行缓冲 + 字体表    = 18 KB                           │
│  应用数据 + HAL 库            =  8 KB                           │
│  C 运行时                     =  1 KB                           │
│  ─────────────────────────────────────                           │
│  ZI-data (应用层)              = 27 KB                           │
│                                                                 │
│  FreeRTOS 静态分配任务 (均计入 .bss，含在 ZI-data 中):           │
│    Task1 (GUI)   栈 = 2 KB    (静态分配, 数组在 .bss)            │
│    Task2 (Sensor)栈 = 1 KB                                      │
│    Task3 (Comm)  栈 = 1 KB                                      │
│    队列 × 2       = 0.5 KB                                      │
│    MSP 中断栈      = 1 KB     (链接脚本 Stack_Size)              │
│    ─────────────────────────────────                             │
│    合计            = 5.5 KB                                     │
│                                                                 │
│  总 ZI-data = 27 + 5.5 = 32.5 KB                                │
│  剩余 SRAM   = 31.5 KB → ✅ 宽裕!                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  场景C: 动态分配版本 → 危险 ⚠️                                     │
├─────────────────────────────────────────────────────────────────┤
│  ZI-data (应用, 不含 FreeRTOS) = 27 KB                           │
│  configTOTAL_HEAP_SIZE = 10 KB                                   │
│                                                                 │
│  运行时从 heap 分配:                                              │
│    Task1 (GUI)  TCB+栈 = 2.1 KB                                 │
│    Task2         TCB+栈 = 1.1 KB                                 │
│    Task3         TCB+栈 = 1.1 KB                                 │
│    队列 × 2              = 0.5 KB                                │
│    Idle + Timer 栈       = 0.3 KB                                │
│    ─────────────────────────────────                             │
│    合计                  ≈ 5.1 KB                                │
│                                                                 │
│  heap 剩余: 10 - 5.1 = 4.9 KB ← 够用但紧                         │
│                                                                 │
│  ⚠️ 如果应用 ZI-data 涨到 35KB+:                                   │
│    SRAM 剩余 = 64 - 35 = 29 KB                                   │
│    configTOTAL_HEAP_SIZE 最多设 ~15KB (预留 MSP + 碎片)           │
│    → 动态分配依然可行, 但余量很紧张                                │
└─────────────────────────────────────────────────────────────────┘
```

### 为什么 LVGL + FreeRTOS 反而是推荐组合？

LVGL 的设计本身就是事件驱动的，官方推荐配合 RTOS 使用：

```
Bare metal (超级循环):
    while (1) {
        lv_timer_handler();    // 处理 LVGL 定时器
        lv_tick_inc(5);        // 手动推进时钟
        // 传感器必须在主循环中轮询 → 一个慢操作阻塞 UI
    }
    ❌ 其他任务阻塞 UI 刷新 → 界面卡顿

FreeRTOS 多任务:
    void vGUITask(void *pv) {     // 优先级: 低 (UI 不怕慢一点)
        while (1) {
            lv_timer_handler();
            vTaskDelay(pdMS_TO_TICKS(5));
        }
    }
    void vSensorTask(void *pv) {  // 优先级: 高 (传感器不能丢数据)
        while (1) {
            ReadSensor();
            xQueueSend(hQueue, &data, portMAX_DELAY);
        }
    }
    ✅ GUI 不被传感器阻塞, 传感器实时采集, 事件驱动
```

### 最终决策树

```
┌──────────────────────────────────────────────────────────────────┐
│              64KB SRAM MCU + LVGL 的架构选择                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  你的 ZI-data (应用层) 目前多大?                                    │
│     │                                                             │
│     ├── < 30 KB ──→ 放心用 FreeRTOS + 静态分配                    │
│     ├── 30~45 KB ──→ 可以用 FreeRTOS, 必须静态分配 + 持续栈监控    │
│     └── > 45 KB ──→ 优化它! 缩小 LVGL 显示缓冲 / 裁剪字体          │
│                     通常砍掉一个全屏缓冲就能省 10~30KB              │
│                                                                  │
│  如果放弃 FreeRTOS 能省多少?                                       │
│     • 无 ucHeap         = +8~10 KB                               │
│     • 无 TCB × N        = +~0.5 KB                               │
│     • 无任务栈 × N      = +2~6 KB                                │
│     • 无队列/信号量     = +~0.5 KB                                │
│     ──────────────────────────────────────                        │
│     总计省出 ≈ 11~17 KB → 可以给 LVGL 显示缓冲                    │
│     但代价是: 传感器轮询阻塞 UI, 代码全在 main() 超级循环中        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

### 10.9 行业全景：嵌入式 MCU 容量分级与 PY32F403 定位

### PY32F403 在行业中处于什么位置？

```
 RAM 容量                          典型 MCU                    市场定位
───────                           ────────                   ────────

 1MB+  ┌────────────────────┐   STM32H7 (1MB)              高端/旗舰
       │  大容量             │   i.MX RT1050 (1MB+)         工业HMI/AI边缘
       │  >256KB RAM        │   ESP32-S3 (512KB)            
       │                    │   
 256KB ├────────────────────┤   STM32F4 (192KB)            中高端
       │  中等容量           │   STM32F7 (320KB)            复杂GUI/RTOS
       │  64~256KB RAM      │   
       │                    │   
  64KB ├────────────────────┤ ══ PY32F403 (64KB) ══════════ 中等/主流
       │  中等偏小           │   STM32F103 (64KB)           工业控制/LVGL
       │  32~64KB RAM       │   AT32F403 (96KB)            +FreeRTOS
       │                    │   GD32F303 (64KB)             
  32KB ├────────────────────┤   STM32F0 (32KB)             低端32位
       │  小容量             │   PY32F030 (32KB)            简单控制
       │  <32KB RAM         │   ATtiny (512B~2KB)          
  0    └────────────────────┘   8051 (256B~1KB)            8位世界
```

### 市场出货量金字塔

```
                     ┌─────────┐
                     │ >256KB  │  ~5%   高端 (汽车/工业HMI/AI)
                     │  RAM    │        
                     ├─────────┤        
                     │ 64~256KB│  ~15%  中高端 (复杂RTOS/LVGL/WiFi)
                     │  RAM    │        
   PY32F403位置 ──▶ ├─────────┤        
                     │ 32~64KB │  ~30%  ← 最大的一块市场!
                     │  RAM    │        工业控制/家电/消费电子
                     ├─────────┤        
                     │ 8~32KB  │  ~25%  低端32位/8位 (传感器)
                     │  RAM    │        
                     ├─────────┤        
                     │ <8KB    │  ~25%  8051/STM8 (玩具/小家电)
                     │  RAM    │        
                     └─────────┘        
```

### 业内工程师的直观分级

| 叫法 | RAM | Flash | 代表型号 | 能跑什么 |
|------|-----|-------|---------|---------|
| "8位机" | 256B~8KB | 1~128KB | 8051, STM8, AVR | 裸机，超级循环 |
| "小容量" | 8~32KB | 32~128KB | STM32F0, GD32E230 | 裸机，简单 RTOS |
| **"主流/中坚"** | **32~128KB** | **128~512KB** | **PY32F403, STM32F103, GD32F303** | **FreeRTOS, LVGL(精简)** |
| "大容量" | 128~512KB | 512KB~2MB | STM32F4/F7, ESP32-S3 | RTOS, LVGL(完整), 文件系统 |
| "旗舰/MPU" | 1MB+ | 外部Flash | STM32H7, i.MX RT | Linux, 复杂GUI, AI |

### PY32F403 在同价位竞品中的位置

```
┌──────────────────┬───────┬────────┬────────┬─────────────────────┐
│  型号             │ 内核   │ 主频    │ 单价   │ 定位               │
├──────────────────┼───────┼────────┼────────┼─────────────────────┤
│  STM32F103VET6   │ M3    │ 72MHz  │ ¥15~25│ 经典工业级          │
│  GD32F303VET6    │ M4    │ 120MHz │ ¥12~20│ 国产高性能替代       │
│  AT32F403AVGT7   │ M4    │ 240MHz │ ¥15~25│ 国产高端             │
│  PY32F403        │ M4F   │ 168MHz │ ¥5~10 │ ← 性价比王!          │
│  ESP32-C3        │ RISC-V│ 160MHz │ ¥8~15 │ WiFi/BLE 加持       │
└──────────────────┴───────┴────────┴────────┴─────────────────────┘
```

### 不同容量 MCU 的工程实践总结

```
   < 32KB RAM:  裸机 (Bare Metal)
                 └─ 超级循环 + 中断 + DMA
                 └─ 不用 RTOS (不划算, OS 本身就要 4~8KB overhead)

   32~64KB RAM: RTOS 可选 (必须静态分配)
                 └─ FreeRTOS (xTaskCreateStatic) + 精简 LVGL
                 └─ 显示缓冲必须克制 (1/10 行)
                 └─ 这是 PY32F403 的情况 → 能用, 但要精打细算

   64~256KB RAM: RTOS 标配 (动态分配可用)
                 └─ FreeRTOS (heap_4) + LVGL (1/4 屏缓冲)
                 └─ 文件系统 + 网络协议栈
                 └─ 可以"不那么省着用"

   >256KB RAM:   全套软件栈
                 └─ RTOS + LVGL (全缓冲) + FatFS + lwIP
                 └─ A/B 分区 OTA
                 └─ 甚至可以上 MicroPython
```

### 最终结论

> **PY32F403 的 64KB RAM 不属于"小容量"，也不属于"大容量"，而是嵌入式行业最主流的"中坚容量"。**

它处于一个微妙的位置：
- **对 LVGL 来说偏小** — 需要克制显示缓冲（1/10 行，15KB）
- **对 FreeRTOS 来说刚好** — 静态分配完全够用
- **对裸机来说浪费** — 168MHz M4F 不用 OS 是浪费了这颗芯片
- **对工业控制来说标准** — CAN + USART + SPI + Timer 齐活
- **对消费电子来说够用** — 跑个 LCD + 按键 + 传感器 + 通信

你感到"资源紧张"不是因为芯片太小，而是因为 **LVGL 的显示缓冲**是 RAM 大户。把双缓冲砍成 1/10 行缓冲，省出的 ~30KB 够 FreeRTOS 跑 10 个任务——这是工程上的 trade-off，不是硬件不够用。

---

## 模块三：Cortex-M4 硬件调用栈与内存底层机理

### 10.10 核心寄存器与调用栈追踪：PC、LR 与 SP (MSP/PSP) 协同原理解密

### 10.9.1 三个角色的定义

```
┌──────────────────────────────────────────────────────────────────┐
│                    Cortex-M4F 核心寄存器                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   R0~R3    │ 通用 / 函数参数传递 (调用者保存)                      │
│   R4~R11   │ 通用 / 局部变量 (被调用者保存)                        │
│   R12      │ 通用 / IP (Intra-Procedure-call scratch)            │
│                                                                  │
│   R13 (SP) │ Stack Pointer — 栈指针 (MSP 或 PSP)                  │
│   R14 (LR) │ Link Register  — 返回地址                            │
│   R15 (PC) │ Program Counter — 当前执行指令地址                    │
│                                                                  │
│   xPSR     │ Program Status Register — 标志位 (N/Z/C/V/Q/ICI)     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.9.2 PC (R15) — 程序计数器

**一句话：CPU 永远在执行 `PC` 指向的那条指令。**

```
Flash 地址空间 (0x08000000 ~ 0x08060000)
┌─────────────────────────────────────────────────────┐
│ 0x08000200:  MOVS  R0, #0x05       ← PC 指在这里    │
│ 0x08000202:  BL    0x08000400       ← 下一条         │
│ 0x08000206:  ADD   R1, R0, #1                       │
│ 0x08000208:  STR   R1, [R2, #0]                     │
│ ...                                                 │
└─────────────────────────────────────────────────────┘

PC 的特性:
• Cortex-M 中 PC 的 bit[0] = 1 表示 Thumb 模式 (必须!)
• 取指时 PC 自动 +2 或 +4 (Thumb/Thumb2 混合)
• 分支指令 (B/BL/BX) 直接改写 PC
• 异常返回时，硬件从栈中弹出 PC 值 → 回到被中断的位置
```

**PY32F403 上 PC 永远在 Flash 区 (0x08xxxxxx)，因为代码在 Flash 中执行。**

### 10.9.3 LR (R14) — 链接寄存器

**一句话：LR 记住"回家"的地址——函数调用后该回到哪里。**

```
函数调用链条示例:

main() @ 0x08001000:
    BL  vTask1Handler     ← 执行 BL 时:
                           ① LR = 0x08001004 (下一条指令地址)
                           ② PC = vTask1Handler 入口地址
                           ③ 跳转!

vTask1Handler() @ 0x08000500:
    PUSH {LR}             ← 把 LR(0x08001004) 压栈 (因为要再调子函数)
    BL   LED_Toggle       ← LED_Toggle 内部会覆盖 LR!
    POP  {PC}             ← 从栈恢复 → PC = 0x08001004 → 回到 main!
```

**BL 指令编码：**

```
BL <target> 是一条 32 位指令:
┌──────────────────────────────────────────────────────────┐
│ 11110  S  imm10  11J1 1 J2  imm11                        │
│                              ↑                           │
│                      26-bit 有符号偏移                     │
│                                                          │
│  新 PC = 当前 PC + offset                                 │
│  同时: LR = 当前 PC + 4  (返回地址)                        │
└──────────────────────────────────────────────────────────┘
```

**关键点：发生异常时，硬件也自动把返回地址写入 LR，但值不同：**

```
进入异常时的 LR (EXC_RETURN):
  0xFFFFFFF1 → 返回 Handler 模式, 使用 MSP
  0xFFFFFFF9 → 返回 Thread 模式, 使用 MSP
  0xFFFFFFFD → 返回 Thread 模式, 使用 PSP (FreeRTOS 任务用这个!)

当异常返回时, CPU 看到 LR 是 0xFFFFFFxx → 触发硬件自动出栈:
  从栈中弹出 R0,R1,R2,R3,R12,LR,PC,xPSR → 回到被打断的代码
```

### 10.9.4 Stack 地址 — 栈到底在哪里？

**PY32F403 启动后（调度器启动前）：**

```
SRAM 64KB (0x20000000 ~ 0x20010000)
┌──────────────────────────────────────┐ 0x20010000 (SRAM 顶)
│  MSP 初始位置 (从向量表[0]加载)       │
│  例如 0x20010000                     │ ← __initial_sp
├──────────────────────────────────────┤
│         ↓ MSP 向下增长 ↓             │
│  main() 的栈帧                       │
│  HAL_Init() 的局部变量               │
│  SystemClock_Config() 的局部变量     │
│         ↓                           │
├──────────────────────────────────────┤
│         .bss (未初始化全局变量)       │
├──────────────────────────────────────┤
│         .data (已初始化全局变量)       │
├──────────────────────────────────────┤
│  (若使用 heap_4) ucHeap[]            │  ← configTOTAL_HEAP_SIZE 决定
└──────────────────────────────────────┘ 0x20000000 (SRAM 底)
```

**调度器启动后（多任务）：**

```
SRAM:
┌──────────────────────────────────────┐ 0x20010000
│  MSP 栈 (中断/内核使用)               │
│         ↓                            │
├──────────────────────────────────────┤
│  PSP: Task3 栈 [512 bytes]           │
├──────────────────────────────────────┤
│  PSP: Task2 栈 [256 bytes]           │
├──────────────────────────────────────┤
│  PSP: Task1 栈 [1024 bytes]          │
├──────────────────────────────────────┤
│  TCB + 队列 + 信号量 (heap_4 分配)    │
├──────────────────────────────────────┤
│  ucHeap[] (FreeRTOS 堆)              │
├──────────────────────────────────────┤
│  .bss / .data                        │
└──────────────────────────────────────┘ 0x20000000

栈地址的直接验证方法:
  void *sp = NULL;
  __asm volatile ("MOV %0, SP" : "=r" (sp));
  printf("Current SP = 0x%08X\r\n", (uint32_t)sp);
  // 在任务中 → PSP, 结果在 0x2000xxxx
  // 在中断中 → MSP, 结果在 0x2000xxxx (靠近 SRAM 顶)
```

---

### 10.11 链接脚本 (Scatter Loading) 自定义段：变量精准绑定

### 10.10.1 PY32F403 默认链接脚本结构

```c
// PY32F403 默认 scatter file (Keil .sct 格式)
LR_IROM1 0x08000000 0x00060000  {    // 384KB Flash
  ER_IROM1 0x08000000 0x00060000  {
    *.o (RESET, +First)
    *(InRoot$$Sections)
    .ANY (+RO)
  }
  
  RW_IRAM1 0x20000000 0x00010000  {   // 64KB SRAM
    .ANY (+RW +ZI)                      // 所有可读写、零初始化数据放这里
  }
}
```

**这只有一个 SRAM 区域。但如果想隔离变量，就要自定义 section。**

### 10.10.2 修改链接脚本 — 在 SRAM 中开辟特殊区域

```c
// 修改后的 scatter file (PY32F403_custom.sct)
LR_IROM1 0x08000000 0x00060000  {
  ER_IROM1 0x08000000 0x00060000  {
    *.o (RESET, +First)
    *(InRoot$$Sections)
    .ANY (+RO)
  }
  
  // ═══ SRAM 64KB 分成 4 个区域 ═══
  
  // 区域1: 普通数据区 (0x20000000 ~ 0x2000C000, 48KB)
  RW_IRAM1 0x20000000 0x0000C000  {
    .ANY (+RW +ZI)
  }
  
  // 区域2: FreeRTOS 堆专用 (0x2000C000 ~ 0x2000E800, 10KB)
  FREERTOS_HEAP 0x2000C000 0x00002800  {
    heap_4.o (+RW +ZI)
  }
  
  // 区域3: DMA 缓冲区专用 (0x2000E800 ~ 0x2000F800, 4KB)
  //        UNINIT = 上电不初始化, 保留上次数据
  DMA_BUFFER 0x2000E800 UNINIT 0x00001000  {
    dma_buffers.o (+ZI)
  }
  
  // 区域4: 关键参数区 (0x2000F800 ~ 0x20010000, 2KB)
  CRITICAL_PARAMS 0x2000F800 0x00000800  {
    critical_params.o (+RW +ZI)
  }
}
```

### 10.10.3 C 代码中指定变量到特定段

```c
/* ── dma_buffers.c ── */

/* Method 1: __attribute__((section("name"))) — ARMCC/GCC 通用 */
__attribute__((section("DMA_BUFFER"))) 
static uint8_t g_aucADCBuffer[2048];  // 自动放入 DMA_BUFFER 区域

__attribute__((section("DMA_BUFFER")))
static uint8_t g_aucUSARTTxBuf[512];

__attribute__((section("DMA_BUFFER")))
static uint8_t g_aucUSARTRxBuf[512];


/* ── critical_params.c ── */

/* Method 2: Keil 专用 #pragma arm section */
#pragma arm section zidata = "CRITICAL_PARAMS"
static uint32_t g_ulSystemFlags;
static float    g_fCalibrationValue;
static uint8_t  g_ucSerialNumber[16];
#pragma arm section zidata   // 恢复默认


/* Method 3: 带初始值的变量放指定段 */
#pragma arm section rwdata = "CRITICAL_PARAMS"
static const uint32_t g_ulFirmwareVersion = 0x01000001;
#pragma arm section rwdata
```

### 10.10.4 .map 文件验证

编译后查看 `.map` 文件中的内存分配：

```
Execution Region FREERTOS_HEAP (Base: 0x2000c000, Size: 0x00002800, Max: 0x00002800)
    heap_4.o                                   0x00002800  Zero  RW
    ucHeap                         0x2000c000  0x00002800  Zero  Data  heap_4.o
    
Execution Region DMA_BUFFER (Base: 0x2000e800, Size: 0x00000c00, Max: 0x00001000)
    dma_buffers.o                              0x00000800  Zero  RW
    g_aucADCBuffer                 0x2000e800  0x00000800  Zero  Data  dma_buffers.o
    g_aucUSARTTxBuf                0x2000f000  0x00000200  Zero  Data  dma_buffers.o
    g_aucUSARTRxBuf                0x2000f200  0x00000200  Zero  Data  dma_buffers.o
    
Execution Region CRITICAL_PARAMS (Base: 0x2000f800, Size: 0x0000001c, Max: 0x00000800)
    g_ulSystemFlags                0x2000f800  0x00000004  Data  RW  critical_params.o
```

### 10.10.5 最终内存布局可视化

```
              SRAM 64KB (0x20000000 ~ 0x20010000)
┌────────────────────────────────────────────────────────┐ 0x20010000
│                                                        │
│  CRITICAL_PARAMS (2KB)    ← 0x2000F800                 │
│    g_ulSystemFlags @ 0x2000F800                        │
│    g_fCalibrationValue @ 0x2000F804                    │
│    g_ucSerialNumber @ 0x2000F808                       │
│                                                        │
├────────────────────────────────────────────────────────┤ 0x2000F800
│                                                        │
│  DMA_BUFFER (4KB)          ← 0x2000E800                │
│    g_aucADCBuffer @ 0x2000E800 (2048 bytes)            │
│    g_aucUSARTTxBuf @ 0x2000F000 (512 bytes)            │
│    g_aucUSARTRxBuf @ 0x2000F200 (512 bytes)            │
│    [剩余 1KB]                                          │
│                                                        │
├────────────────────────────────────────────────────────┤ 0x2000E800
│                                                        │
│  FREERTOS_HEAP (10KB)      ← 0x2000C000                │
│    ucHeap[10240]                                       │
│                                                        │
├────────────────────────────────────────────────────────┤ 0x2000C000
│                                                        │
│  RW_IRAM1 (48KB)            ← 0x20000000               │
│    .data + .bss + 调用栈                                │
│                                                        │
└────────────────────────────────────────────────────────┘ 0x20000000
```

---

### 10.12 内存踩踏大解剖：数组越界是如何摧毁邻居变量的？

### 10.11.1 实验设计

```c
/* ── 故意构造一个越界场景 ── */

/* 三个临近的全局变量 (链接器决定顺序!) */
static uint8_t  g_ucSecretFlag   = 0xAA;       // 地址: 0x20000040 (假设)
static uint8_t  g_aucGap[4];                   // 地址: 0x20000041 ~ 0x20000044
static uint32_t g_ulSystemState  = 0x12345678; // 地址: 0x20000044 (假设对齐后紧接)

/* 故意越界写入 */
static void vArrayOverflowDemo(void)
{
    printf("Before:\r\n");
    printf("  g_ucSecretFlag  = 0x%02X  @ 0x%08X\r\n", 
           g_ucSecretFlag, (uint32_t)&g_ucSecretFlag);
    printf("  g_ulSystemState = 0x%08X @ 0x%08X\r\n", 
           g_ulSystemState, (uint32_t)&g_ulSystemState);

    /* 输出示例:
       g_ucSecretFlag  = 0xAA       @ 0x20000040
       g_ulSystemState = 0x12345678 @ 0x20000044
    */

    /* ── 越界写入! 看起来无害的循环 ── */
    printf("Writing 8 bytes to a 4-byte array...\r\n");
    for (int i = 0; i < 8; i++) {
        g_aucGap[i] = 0xFF;  // ← 从 i=4 开始越界!
    }

    printf("After:\r\n");
    printf("  g_ucSecretFlag  = 0x%02X  @ 0x%08X  ← CORRUPTED!\r\n", 
           g_ucSecretFlag, (uint32_t)&g_ucSecretFlag);
    printf("  g_ulSystemState = 0x%08X @ 0x%08X ← CORRUPTED!\r\n", 
           g_ulSystemState, (uint32_t)&g_ulSystemState);

    /* 输出:
       g_ucSecretFlag  = 0xFF       @ 0x20000040 ← 0xAA 变成 0xFF!
       g_ulSystemState = 0xFFFFFFFF @ 0x20000044 ← 0x12345678 变成 0xFFFFFFFF!
    */
}
```

### 10.11.2 内存级可视化

```
内存地址                    变量布局                     越界写入过程
────────                    ────────                    ──────────
0x2000003F  [其他变量]      ┌──────────┐
0x20000040  g_ucSecretFlag  │   0xAA   │ ← i=0 写入 g_aucGap[0], 正常
0x20000041  g_aucGap[0]    │   0x??   │ ← i=1 → g_aucGap[1], 正常
0x20000042  g_aucGap[1]    │   0x??   │ ← i=2 → g_aucGap[2], 正常
0x20000043  g_aucGap[2]    │   0x??   │ ← i=3 → g_aucGap[3], 正常
0x20000044  g_aucGap[3]    │   0x??   │ ← i=4 → 越界!!! 写入 g_aucGap[4]
            g_ulSystemState ├──────────┤           实际写入 g_ulSystemState 的 byte0!
0x20000045  (byte1)         │   0x56   │ ← i=5 → 越界!!! 覆盖 byte1
0x20000046  (byte2)         │   0x34   │ ← i=6 → 越界!!! 覆盖 byte2
0x20000047  (byte3)         │   0x12   │ ← i=7 → 越界!!! 覆盖 byte3
0x20000048  [下一个变量]     │  ??????  │

最终结果:
  g_aucGap[0..3] → 0xFF 0xFF 0xFF 0xFF  (正常)
  g_aucGap[4..7] → 实际上写到了 g_ulSystemState 的四个字节!
  g_ulSystemState: 0x12345678 → 0xFFFFFFFF
```

### 10.11.3 越界踩踏关键变量的致命后果

```c
/* 更危险的场景 — 踩踏在"另一边" */

static uint8_t       g_aucSmallBuf[4];   // 地址: 0x20000080
static TaskHandle_t  g_hCriticalTask;     // 地址: 0x20000084 (紧跟其后)

// 越界代码...
for (int i = 0; i < 12; i++) {
    g_aucSmallBuf[i] = 0x00;  // i=4~11 越界, 覆盖 g_hCriticalTask!
}

// 后果:
// g_hCriticalTask 现在 = 0x00000000 (NULL!)
// 后续调用:
vTaskResume(g_hCriticalTask);  // → 参数是 NULL, 未定义行为!
//                         ↑
//         可能触发 configASSERT, 可能 HardFault, 可能静默损坏!
```

### 10.11.4 防止越界的工程手段

```c
/* 策略1: 编译时边界检查 — static_assert (C11) */
#define ARRAY_SIZE(arr)  (sizeof(arr) / sizeof((arr)[0]))

_Static_assert(ARRAY_SIZE(g_aucGap) >= 4, "Buffer too small!");

void vSafeCopy(uint8_t *pDst, size_t dstSize, 
               const uint8_t *pSrc, size_t srcSize)
{
    if (srcSize > dstSize) {
        configASSERT(!"Buffer overflow prevented!");
        return;
    }
    memcpy(pDst, pSrc, srcSize);
}

/* 策略2: 运行时哨兵 — 卫兵变量 */
typedef struct {
    uint32_t ulGuardStart;   // = 0xDEADBEEF
    uint8_t  aucData[256];
    uint32_t ulGuardEnd;     // = 0xCAFEBABE
} GuardedBuffer_t;

#define GUARD_START  0xDEADBEEF
#define GUARD_END    0xCAFEBABE

void vBufferIntegrityCheck(GuardedBuffer_t *pBuf)
{
    if (pBuf->ulGuardStart != GUARD_START || 
        pBuf->ulGuardEnd   != GUARD_END) {
        // 越界发生! 哨兵被破坏!
        vApplicationStackOverflowHook(NULL, "BufferOverflow");
    }
}

/* 策略3: MPU 内存保护 — 硬件级隔离 */
// PY32F403 带 MPU (Memory Protection Unit)
// 可配置: DMA_BUFFER 区域设为不可执行、不可缓存
// CRITICAL_PARAMS 区域设读写保护
```

---

## 模块四：系统启动、向量表与 Bootloader 跳转生态

### 10.13 单片机从加电复位到 main() 的全流程解析

### 10.12.1 从加电到 main() — 完整流程图

```
┌──────────────────────────────────────────────────────────────────┐
│               PY32F403 上电启动完整流程                             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  时间线 ──────────────────────────────────────────────────────▶  │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ ① 上电/复位    │ VDD 上升到工作电压                             │
│  │   POR 释放    │ 内部 RC 振荡器起振                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ ② 硬件初始化  │ CPU 从复位向量中读取:                           │
│  │              │   • SP = *((uint32_t *)0x08000000)  栈顶        │
│  │              │   • PC = *((uint32_t *)0x08000004)  复位入口    │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ ③ Reset_Handler│ (启动文件 startup_py32f403.s)                │
│  │              │   • 禁用全局中断 (CPSID I)                      │
│  │              │   • 初始化 .data 段 (从 Flash 拷贝到 SRAM)       │
│  │              │   • 清零 .bss 段                                │
│  │              │   • 调用 SystemInit()                           │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ ④ SystemInit()│ system_py32f4xx.c (HAL 库)                   │
│  │              │   • 配置时钟: HSI → PLL → HCLK = 168MHz        │
│  │              │   • 配置 Flash 等待周期 (Latency)               │
│  │              │   • 配置向量表偏移 (SCB->VTOR)                  │
│  │              │   • 使能 FPU (Cortex-M4F)                       │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ ⑤ __main()   │ C 运行时库初始化 (ARMCC / GCC libc)            │
│  │  (C 库入口)   │   • 调用 __scatterload (Keil)                   │
│  │              │   • 初始化 C++ 静态对象 (如果有)                  │
│  │              │   • 最终调用 main()                             │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ ⑥ main()     │ 用户代码入口                                    │
│  │              │   • HAL_Init()                                 │
│  │              │   • SystemClock_Config()                       │
│  │              │   • 外设初始化                                   │
│  │              │   • 创建 FreeRTOS 任务                           │
│  │              │   • vTaskStartScheduler()                      │
│  └──────────────┘                                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.12.2 启动文件的汇编代码解读

```armasm
; ═══════════════════════════════════════════════════════════════
; startup_py32f403.s — Reset_Handler 逐行解读
; ═══════════════════════════════════════════════════════════════

Stack_Size      EQU     0x00000400                    ; MSP 栈大小 1KB

                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem       SPACE   Stack_Size                    ; 在 SRAM 中预留 1KB
__initial_sp                                            ; 编译器保留此符号

; ── 向量表 (放 Flash 最开头 0x08000000) ──
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
__Vectors
                DCD     __initial_sp                  ; [0] SP 初始值
                DCD     Reset_Handler                 ; [1] PC 初始值 = Reset_Handler 地址
                DCD     NMI_Handler                   ; [2] NMI
                DCD     HardFault_Handler             ; [3] HardFault
                DCD     MemManage_Handler             ; [4] MPU 违例
                DCD     BusFault_Handler              ; [5] 总线错误
                DCD     UsageFault_Handler            ; [6] 未定义指令等
                ; ... 其余中断向量 ...

; ── 复位处理的执行流程 ──
                AREA    |.text|, CODE, READONLY
Reset_Handler   PROC
                EXPORT  Reset_Handler
                
                ; Step A: 加载数据段基址
                LDR     R0, =|Load$$ER_IROM1$$Base|    ; Flash 中 .data 副本的位置
                LDR     R1, =|Image$$RW_IRAM1$$Base|    ; SRAM 中 .data 应放的位置
                LDR     R2, =|Image$$RW_IRAM1$$Length|  ; .data 段的长度
                
                ; Step B: 拷贝 .data 段 (初始化全局变量!)
CopyData
                CMP     R2, #0                          ; 如果长度为 0, 跳过
                BEQ     ZeroBss
                LDR     R3, [R0], #4                    ; 从 Flash 读 4 字节
                STR     R3, [R1], #4                    ; 写到 SRAM
                SUBS    R2, R2, #4
                BNE     CopyData
                
                ; Step C: 清零 .bss 段 (未初始化全局变量 = 0)
                LDR     R0, =|Image$$RW_IRAM1$$ZI$$Base|
                LDR     R1, =|Image$$RW_IRAM1$$ZI$$Length|
ZeroBss
                CMP     R1, #0
                BEQ     CallSystemInit
                MOVS    R2, #0
                STR     R2, [R0], #4
                SUBS    R1, R1, #4
                BNE     ZeroBss
                
                ; Step D: 调用系统初始化
CallSystemInit
                LDR     R0, =SystemInit
                BLX     R0
                
                ; Step E: 跳到 C 运行时入口 (__main → 最终调用 main())
                LDR     R0, =__main
                BX      R0
                ENDP
```

### 10.12.3 关键数据流可视化

```
Flash (0x08000000)                        SRAM (0x20000000)
┌──────────────────────┐                 ┌──────────────────────┐
│  向量表 [0]: SP 初值  │─────────────▶  │ MSP 初始指向这里      │
│  向量表 [1]: PC 初值  │──┐              │                      │
│  向量表 [2]: NMI      │  │              │  启动后:              │
│  向量表 [3]: HardFault│  │              │  ┌─────────────────┐ │
│  ...                  │  │              │  │ MSP 栈 (1KB)     │ │
├──────────────────────┤  │              │  ├─────────────────┤ │
│  代码 .text           │  │              │  │ .data (已初始化) │ │ ← 从 Flash 拷贝
│  Reset_Handler ───────┘  │              │  ├─────────────────┤ │
│  SystemInit              │              │  │ .bss = 0         │ │ ← 启动代码清零
│  main                    │              │  ├─────────────────┤ │
├──────────────────────┤  │              │  │ ucHeap[]         │ │
│  .rodata (常量)         │  │              │  ├─────────────────┤ │
├──────────────────────┤  │              │  │ 动态分配区         │ │
│  .data 初始化值副本 ────┼──┘              │  └─────────────────┘ │
│  (LMA = Flash)          │  (拷贝)        │                      │
└──────────────────────┘                 └──────────────────────┘
```

### 10.12.4 启动流程核心总结

| 步骤 | 谁做的 | 做什么 | 位置 |
|------|--------|--------|------|
| ① POR | 硬件 | 电压稳定，释放复位 | - |
| ② 取向量 | CPU 硬件 | SP = [0x08000000], PC = [0x08000004] | - |
| ③ Reset_Handler | 启动文件 | 拷贝 .data、清零 .bss | Flash→SRAM |
| ④ SystemInit | HAL库 | 时钟、Flash延迟、VTOR、FPU | Flash |
| ⑤ __main | C运行时 | scatterload, C++ 静态初始化 | Flash |
| ⑥ main | 用户代码 | 外设初始化，创建RTOS任务 | Flash |

---

### 10.14 中断向量表原理与 VTOR 动态重定位机制

### 10.13.1 向量表是什么？

**一句话：向量表是一张"事故处理通讯录"，存在 Flash 最开头，CPU 一出事就按索引查这张表找到处理函数地址。**

```
向量表 (Vector Table) 本质:
  一个 const 函数指针数组，硬件按异常编号索引

格式:
  uint32_t __Vectors[N] = {
      [0]  = 初始栈指针值,         // 不是函数指针! 是 SP 的值
      [1]  = Reset_Handler,        // 函数指针
      [2]  = NMI_Handler,          // 函数指针
      [3]  = HardFault_Handler,    // 函数指针
      [4]  = MemManage_Handler,    // 函数指针
      ...
      [16] = IRQ0_Handler,         // 外设中断开始
      [17] = IRQ1_Handler,
      ...
  };
```

### 10.13.2 PY32F403 的完整向量表

```
┌──────────────────────────────────────────────────────────────────┐
│              PY32F403 (Cortex-M4F) 向量表完整结构                  │
├──────┬──────────────────┬────────────────────────────────────────┤
│ 编号  │ 名称              │ 说明                                   │
├──────┼──────────────────┼────────────────────────────────────────┤
│   0  │ SP 初值            │ 0x20010000 (SRAM 顶)                   │
│   1  │ Reset              │ 上电后第一条指令地址                     │
│   2  │ NMI                │ 不可屏蔽中断 (硬件引脚)                  │
│   3  │ HardFault          │ 硬件错误 (默认错误处理)                  │
│   4  │ MemManage          │ MPU 内存保护违例                        │
│   5  │ BusFault           │ 总线访问错误 (取指/数据异常)              │
│   6  │ UsageFault         │ 未定义指令/除零/非对齐访问               │
│  7~10│ 保留                │                                        │
│  11  │ SVCall             │ SVC 指令触发 (FreeRTOS 用!)             │
│  12  │ DebugMonitor       │ 调试监视器                              │
│  13  │ 保留                │                                        │
│  14  │ PendSV             │ 可挂起系统调用 (FreeRTOS 上下文切换用!)   │
│  15  │ SysTick            │ 系统滴答定时器 (FreeRTOS 心跳!)          │
│  16+ │ 外设中断            │ GPIO, USART, SPI...                   │
│      │                    │ 共约 80+ 个外部中断                     │
└──────┴──────────────────┴────────────────────────────────────────┘
```

### 10.13.3 向量表在内存中的布局

```
Flash 地址      内容                         向量编号
────────        ────                         ──────
0x08000000      [0x20010000]         ← SP 初值   [0]
0x08000004      [0x08000101]         ← Reset     [1]  (bit0=1 → Thumb)
0x08000008      [0x08000189]         ← NMI       [2]
0x0800000C      [0x08000201]         ← HardFault [3]
0x08000010      [0x08000251]         ← MemManage [4]
0x08000014      [0x080002A1]         ← BusFault  [5]
0x08000018      [0x080002F1]         ← UsageFault[6]
0x0800001C      [0x00000000]         ← 保留      [7~10]
...
0x0800002C      [0x08000301]         ← SVCall    [11] ← FreeRTOS!
0x0800003C      [0x08000351]         ← PendSV    [14] ← FreeRTOS上下文切换!
0x08000040      [0x080003A1]         ← SysTick   [15] ← FreeRTOS心跳!
0x08000044      [0x080003E1]         ← WWDG      [16]
0x08000048      [0x08000431]         ← PVD       [17]
...

SCB->VTOR 寄存器 (默认 = 0x08000000):
  可以重定位向量表到 SRAM (用于 IAP/OTA 升级)
  例: SCB->VTOR = 0x20000000;  // 向量表移到 SRAM 开头
```

### 10.13.4 异常处理流程图 — CPU 如何查表

```
异常发生时 (例如 PendSV 被触发):

  ┌─────────────────────────────────────────────────────┐
  │ ① CPU 识别异常编号 N (PendSV = 14)                   │
  │                                                     │
  │ ② 硬件自动压栈 (R0-R3,R12,LR,PC,xPSR → 当前栈)       │
  │                                                     │
  │ ③ 从向量表读取:                                      │
  │    HandlerAddr = *( (uint32_t *)(VTOR + 14 * 4) )   │
  │    即: *(0x08000000 + 0x38) = *(0x08000038)          │
  │        = 0x08000351 (PendSV_Handler 入口)           │
  │                                                     │
  │ ④ LR = EXC_RETURN (0xFFFFFFF9 / 0xFFFFFFFD)         │
  │                                                     │
  │ ⑤ PC = HandlerAddr  → CPU 跳到 PendSV_Handler 执行   │
  │                                                     │
  │ ⑥ PendSV_Handler 完成后:                             │
  │    BX LR  (LR = 0xFFFFFFFD)                         │
  │    → CPU 识别 EXC_RETURN                             │
  │    → 硬件自动出栈 (恢复 R0-R3,R12,LR,PC,xPSR)        │
  │    → 回到被中断的代码                                 │
  └─────────────────────────────────────────────────────┘
```

### 10.13.5 FreeRTOS 如何使用三个系统异常

```
┌──────────────┬─────────────────────────────────────────────────┐
│   异常        │ FreeRTOS 中的角色                                │
├──────────────┼─────────────────────────────────────────────────┤
│              │                                                 │
│  SVCall      │ 启动第一个任务!                                  │
│  (向量[11])  │ vPortSVCHandler():                              │
│              │   ① 设置 MSP 为中断栈                            │
│              │   ② 设置 PSP 为第一个任务栈                      │
│              │   ③ 设置 CONTROL 寄存器 → 使用 PSP              │
│              │   ④ 异常返回 → 直接进入第一个任务!               │
│              │                                                 │
├──────────────┼─────────────────────────────────────────────────┤
│              │ 上下文切换的核心!                                 │
│  PendSV      │ xPortPendSVHandler() (port.c):                  │
│  (向量[14])  │   ① 保存当前任务的 R4-R11 到它的栈               │
│              │   ② 保存当前 PSP 到 pxCurrentTCB                │
│              │   ③ 选择下一个最高优先级任务                     │
│              │   ④ 从新任务的 TCB 恢复 PSP                      │
│              │   ⑤ 从新任务栈恢复 R4-R11                       │
│              │   ⑥ 异常返回 → 新任务开始执行!                   │
│              │                                                 │
│              │   🔑 为什么用 PendSV 不用 SVCall?                │
│              │   PendSV 是最低优先级异常 → 保证只在所有 ISR    │
│              │   完成后才切换 → 不会在中断中切换任务!            │
│              │                                                 │
├──────────────┼─────────────────────────────────────────────────┤
│              │ 系统心跳!                                        │
│  SysTick     │ xPortSysTickHandler():                          │
│  (向量[15])  │   ① xTickCount++ (系统时间)                     │
│              │   ② 检查延时列表 → 唤醒到期任务                  │
│              │   ③ 如果需要切换 → 触发 PendSV                  │
│              │                                                 │
└──────────────┴─────────────────────────────────────────────────┘
```

### 10.13.6 向量表重定位 — OTA 升级的核心

```c
/* IAP / OTA 场景: 两个程序, 两个向量表 */

// Bootloader: 向量表在 0x08000000
// Application: 向量表在 0x08004000 (16KB 偏移)

// Bootloader 跳转到 Application:
void vJumpToApplication(uint32_t ulAppAddr)
{
    typedef void (*pFunc)(void);
    uint32_t ulAppSP;
    pFunc    pfAppReset;

    /* Read Application's vector table */
    ulAppSP    = *((uint32_t *)(ulAppAddr + 0));       // SP 初值
    pfAppReset = (pFunc)(*((uint32_t *)(ulAppAddr + 4))); // Reset Handler

    /* Set MSP to Application's stack */
    __set_MSP(ulAppSP);

    /* Set VTOR to Application's vector table */
    SCB->VTOR = ulAppAddr;

    /* Jump to Application's Reset_Handler */
    pfAppReset();  // 永不返回!
}

/* Application 的 main 中必须做的第一件事: */
int main(void)
{
    SCB->VTOR = 0x08004000;  // 重新确认向量表位置!
    // ... 正常初始化 ...
}
```

---

### 10.15 Cortex-M4F 核心寄存器全集与状态字详解

### 10.14.1 寄存器位宽

**Cortex-M4F 所有核心寄存器都是 32 位的。** 这是 ARMv7-M 架构的基本约定。

```
┌──────────────────────────────────────────────────────────────────┐
│              Cortex-M4F 核心寄存器全集 (×21个, 各32位)             │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─ 通用寄存器 (R0~R12) ──────────────────────────────────────┐ │
│  │  R0  │ 参数1 / 返回值           │ R7  │ 被调用者保存        │ │
│  │  R1  │ 参数2                    │ R8  │ 被调用者保存        │ │
│  │  R2  │ 参数3                    │ R9  │ 被调用者保存        │ │
│  │  R3  │ 参数4                    │ R10 │ 被调用者保存        │ │
│  │  R4  │ 被调用者保存              │ R11 │ 被调用者保存        │ │
│  │  R5  │ 被调用者保存              │ R12 │ IP (scratch)       │ │
│  │  R6  │ 被调用者保存              │      │                    │ │
│  └──────┴──────────────────────────┴──────┴────────────────────┘ │
│                                                                  │
│  ┌─ 特殊功能寄存器 ──────────────────────────────────────────┐  │
│  │  R13(SP) │ Stack Pointer — MSP + PSP (Banked)             │  │
│  │  R14(LR) │ Link Register — 返回地址                        │  │
│  │  R15(PC) │ Program Counter                                │  │
│  └──────────┴────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌─ 程序状态寄存器 ──────────────────────────────────────────┐  │
│  │  xPSR    │ 32 bits │ 由 3 个子寄存器拼接:                  │  │
│  │          │         │  APSR (bit31~27): N/Z/C/V/Q 标志     │  │
│  │          │         │  EPSR (bit26~24,15~10): ICI/IT/T     │  │
│  │          │         │  IPSR (bit8~0): 当前异常编号          │  │
│  └──────────┴─────────┴──────────────────────────────────────┘  │
│                                                                  │
│  ┌─ 特殊寄存器 (MRS/MSR 访问) ─────────────────────────────┐    │
│  │  PRIMASK  │ 全局中断屏蔽 (实际只用 bit0)                 │    │
│  │  FAULTMASK│ HardFault 以外全屏蔽 (实际只用 bit0)         │    │
│  │  BASEPRI  │ 按优先级屏蔽中断                             │    │
│  │  CONTROL  │ bit1=FPCA, bit0=SP 选择 (MSP/PSP)           │    │
│  └───────────┴─────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─ FPU 寄存器 (Single Precision) ─────────────────────────┐    │
│  │  S0~S31   │ 32 bits each │ 浮点通用寄存器                 │    │
│  │  (可配对为 D0~D15, 各 64 bits)                            │    │
│  │  FPSCR    │ 32 bits │ 浮点状态控制寄存器                   │    │
│  │  FPEXC    │ 32 bits │ 浮点异常寄存器                       │    │
│  └───────────┴──────────┴───────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**32 位的含义：** 每个寄存器就是芯片内部的一个 32-bit 触发器阵列（flip-flop bank），有 32 条物理数据线。MSP/PSP/PC 的值在 PY32F403 上虽然实际只用 20-bit 地址空间（1MB 寻址范围），但寄存器本身是 32-bit 宽的。

### 10.14.1.1 R13 栈顶地址指针
R13 始终指向**当前栈的栈顶地址**。Cortex-M4 采用**满递减栈（Full Descending, FD）**：（栈从高位向低位增加）

- **PUSH 入栈**：SP 先递减，再写入数据（`STMDB` / `STMFD` 指令）
- **POP 出栈**：先读出数据，SP 再递增（`LDMIA` / `LDMFD` 指令）
- 栈向**低地址方向生长**，栈底在高地址，栈顶在低地址

**R13**:Banked 双物理栈指针,**两个独立的物理寄存器**，通过 `CONTROL` 寄存器 bit0 动态切换：
表格

| 栈指针 | 全称 | 使用场景 |
| --- | --- | --- |
| **MSP** | Main Stack Pointer（主栈指针） | 复位后默认使用；**Handler 模式（所有中断 / 异常）强制使用**；裸机系统或 RTOS 内核态 |
| **PSP** | Process Stack Pointer（进程栈指针） | 仅 Thread 模式可选用；**RTOS 中每个用户任务独立使用 PSP** |
#### 硬件强制规则：

- 进入异常（IRQ、SVC、PendSV 等）时，硬件自动将 `CONTROL.SPSEL` 清零，**强制切换到 MSP**
- 异常返回时，根据 EXC_RETURN 值自动恢复之前的栈指针选择
- Handler 模式下**无法通过软件切换到 PSP**，这是硬件安全机制，确保中断栈不会被任务栈溢出破坏

### 10.14.2 寄存器数据的保存与恢复 — 三级机制

这是理解 FreeRTOS 上下文切换的关键。数据保存分为**硬件自动**和**软件手动**两级。

### 10.14.3 异常入口 — 硬件自动压栈（8个字 = 32字节）

当任何异常/中断发生时，CPU 硬件**自动、不可编程地**把 8 个寄存器的值压入当前栈：

```
硬件压栈顺序（不可更改, ARMv7-M 规范):
─────────────────────────────────────────────

异常发生前:
  SP → ┌──────────────────┐  高地址 (栈顶, 异常前 SP 指向这里)
       │  (正在使用的栈)   │
       │  ...             │
       │                  │

硬件压栈后 (SP 自动 -32, 向低地址移动 8 个字):
       ┌──────────────────┐  ← 异常前 SP
       │  xPSR            │  ← [SP, +28]  程序状态寄存器
       │  PC (返回地址)    │  ← [SP, +24]  返回后从哪条指令继续
       │  LR (R14)        │  ← [SP, +20]  Link Register
       │  R12             │  ← [SP, +16]  IP
       │  R3              │  ← [SP, +12]
       │  R2              │  ← [SP, +8]
       │  R1              │  ← [SP, +4]
  SP → │  R0              │  ← [SP, +0]  异常后 SP 指向这里
       └──────────────────┘  低地址

  共压栈: R0, R1, R2, R3, R12, LR, PC, xPSR  ← 8 个寄存器, 32 bytes
```

**为什么是这 8 个？** ARM 的设计哲学：

| 寄存器 | 保存原因 |
|--------|---------|
| R0~R3 | "调用者保存" 寄存器，子函数可能随意修改，异常打断后必须恢复 |
| R12 | IP (过程内暂存)，同样可能被随时修改 |
| LR | 被打断前 LR 存着返回地址，会被异常入口覆盖为 EXC_RETURN |
| PC | 必须保存，否则不知道回到哪 |
| xPSR | 必须保存，否则标志位丢失（影响条件分支） |

### 10.14.4 压栈过程的逐周期分解

```
┌──────────────────────────────────────────────────────────────────┐
│           异常入栈的 CPU 总线周期 (Timing Diagram)                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  T0: 异常请求到达 → CPU 识别, 进入异常序列                        │
│  T1: SP -= 4; STORE xPSR → [SP]                                 │
│  T2: SP -= 4; STORE PC   → [SP]                                 │
│  T3: SP -= 4; STORE LR   → [SP]                                 │
│  T4: SP -= 4; STORE R12  → [SP]                                 │
│  T5: SP -= 4; STORE R3   → [SP]                                 │
│  T6: SP -= 4; STORE R2   → [SP]                                 │
│  T7: SP -= 4; STORE R1   → [SP]                                 │
│  T8: SP -= 4; STORE R0   → [SP]                                 │
│  T9: LR = EXC_RETURN; PC = 向量表[异常编号] 中的处理函数地址       │
│  T10: 开始执行 ISR 第一条指令                                     │
│                                                                  │
│  总延迟: 12 个周期 (硬件压栈 8 周期 + 取向量 2 周期 + 流水线)      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.14.5 那 R4~R11 谁保存？— 软件手动压栈（PendSV_Handler）

硬件只负责 R0~R3, R12, LR, PC, xPSR。**R4~R11 必须由软件保存。** 这就是 FreeRTOS 的 `PendSV_Handler` 做的事：

```armasm
; ═══════════════════════════════════════════════════════════════
; FreeRTOS PendSV_Handler — R4~R11 手动压栈/出栈
; 文件: FreeRTOS-Kernel/portable/RVDS/ARM_CM4F/port.c
; ═══════════════════════════════════════════════════════════════

PendSV_Handler:
    ; ─── 第一部分: 保存当前任务的 R4~R11 ───
    MRS     R0, PSP                 ; R0 = 当前任务的 PSP

    ; 此时栈上已有硬件压的 8 个字
    ; STMDB R0!, {R4-R11} 把 R4~R11 也压到同一栈上
    STMDB   R0!, {R4-R11}          ; Store Multiple Decrement Before
                                   ; 等价 8 条 STR, R0 最终指向 R4 位置

    ; ─── 第二部分: 保存 PSP 到 TCB ───
    STR     R0, [R1]                ; pxCurrentTCB->pxTopOfStack = PSP

    ; ─── 第三部分: 选择新任务 ───
    ; 调用 vTaskSwitchContext() → 更新 pxCurrentTCB
    ; ...

    ; ─── 第四部分: 恢复新任务的 R4~R11 ───
    LDR     R0, [R1]                ; R0 = 新 TCB->pxTopOfStack (新 PSP)
    LDMIA   R0!, {R4-R11}          ; 从新栈恢复 R4~R11

    ; ─── 第五部分: 异常返回 ───
    MSR     PSP, R0                 ; 设置 PSP = 新栈帧顶部
    MOV     LR, #0xFFFFFFFD         ; EXC_RETURN: 返回 Thread 模式, 用 PSP
    BX      LR                      ; 异常返回! → 硬件出栈 → 新任务开始!
```

### 10.14.6 完整上下文在栈上的精确布局

```
一个任务的完整上下文快照 (暂停时在栈上):

高地址 ┌──────────────────────┐
       │                      │ ← 任务栈顶 (初始 SP)
       │   (空闲栈空间)        │
       │                      │
       ├──────────────────────┤ ← 硬件压栈帧底部
       │  xPSR (32 bits)      │    0x01000000 (初始值, Thumb 位=1)
       │  PC   (32 bits)      │    任务函数入口地址
       │  LR   (32 bits)      │    prvTaskExitError (安全网)
       │  R12  (32 bits)      │    0x00000000 (初始值)
       │  R3   (32 bits)      │    0x00000000
       │  R2   (32 bits)      │    0x00000000
       │  R1   (32 bits)      │    0x00000000
       │  R0   (32 bits)      │    pvParameters (任务参数指针)
       ├──────────────────────┤ ← 软件压栈帧底部
       │  R11  (32 bits)      │    0x00000000
       │  R10  (32 bits)      │    0x00000000
       │  R9   (32 bits)      │    0x00000000
       │  R8   (32 bits)      │    0x00000000
       │  R7   (32 bits)      │    0x00000000
       │  R6   (32 bits)      │    0x00000000
       │  R5   (32 bits)      │    0x00000000
       │  R4   (32 bits)      │    0x00000000
       ├──────────────────────┤ ← pxTopOfStack 指向这里 (即 R4 位置)
       │  ... 函数调用帧 ...   │
       │  局部变量             │
       │         ↓            │ ← 实际 SP 向下增长
低地址 └──────────────────────┘ ← 任务栈底

总计: 16 words = 64 bytes 的上下文帧
      8 words (硬件) + 8 words (软件)
```

### 10.14.7 异常返回 — 硬件自动出栈

```
BX LR  (LR = EXC_RETURN 0xFFFFFFFD / 0xFFFFFFF1 等)

CPU 检测到 EXC_RETURN 的特殊位模式:
  ┌────────────────────────────────────────────────┐
  │ 0xFFFFFFFD → 返回 Thread 模式, 用 PSP          │
  │ 0xFFFFFFF9 → 返回 Thread 模式, 用 MSP          │
  │ 0xFFFFFFF1 → 返回 Handler 模式, 用 MSP          │
  │ 0xFFFFFFE1 → 返回 Handler 模式, 用 MSP (FPU)   │
  │ 0xFFFFFFED → 返回 Thread 模式, 用 PSP (FPU)    │
  └────────────────────────────────────────────────┘

硬件自动出栈 (8 周期, 与压栈对称):
  T1: R0   = [SP]; SP += 4
  T2: R1   = [SP]; SP += 4
  T3: R2   = [SP]; SP += 4
  T4: R3   = [SP]; SP += 4
  T5: R12  = [SP]; SP += 4
  T6: LR   = [SP]; SP += 4
  T7: PC   = [SP]; SP += 4     ← 取 PC 后, CPU 从新地址开始执行!
  T8: xPSR = [SP]; SP += 4
```

### 10.14.8 FPU 额外保存

如果启用了 FPU（Cortex-M4F 的"F"），还有浮点寄存器的保存：

```
FreeRTOS 使用 LAZY STACKING 优化:
  • 异常入口时不立即压 FPU 寄存器 (节省时间)
  • 设置 CONTROL.FPCA = 1, 表示"这个任务用了 FPU"
  • 只有当另一个也用了 FPU 的任务被调度时, 才真正压 FPU 寄存器
  • 如果任务从未用过 FPU, 根本不产生 FPU 上下文开销!

完整上下文大小对比:
  不用 FPU: 16 words = 64 bytes   (仅 R0~R11 + PC + LR + xPSR)
  用 FPU:   16+17 words = 132 bytes (额外 S0~S15 + FPSCR, lazy stacking)
```

### 10.14.9 上下文切换数据流全景图

```
┌──────────────────────────────────────────────────────────────────┐
│            Cortex-M4F 上下文保存/恢复 数据流全景                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  正常执行中                                                       │
│  ┌────────┐                                                      │
│  │ 任务 A  │  R0~R15 在 CPU 寄存器中                               │
│  │         │  PSP → 任务 A 的栈                                   │
│  └───┬────┘                                                      │
│      │                    SysTick 中断!                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────┐                         │
│  │  硬件自动:                           │                         │
│  │  PUSH {R0,R1,R2,R3,R12,LR,PC,xPSR} │  ← 8 寄存器 → 栈         │
│  │  LR = EXC_RETURN                    │  ← 进入 Handler 模式     │
│  │  SP 从 PSP 切换到 MSP               │  ← 中断栈                │
│  │  PC = SysTick_Handler              │                         │
│  └──────────────┬──────────────────────┘                         │
│                 ▼                                                │
│  ┌─────────────────────────────────────┐                         │
│  │  SysTick_Handler():                 │                         │
│  │  → xTaskIncrementTick()            │                         │
│  │  → 检测到需要切换                    │                         │
│  │  → 触发 PendSV                      │                         │
│  └──────────────┬──────────────────────┘                         │
│                 │ PendSV 触发 (最低优先级, 等所有 ISR 退完)         │
│                 ▼                                                │
│  ┌─────────────────────────────────────┐                         │
│  │  硬件再次自动:                        │                         │
│  │  PUSH {R0,R1,R2,R3,R12,LR,PC,xPSR} │  ← 任务 A 的寄存器       │
│  │  到任务 A 的栈 (PSP)                 │    再次压栈!             │
│  │  PC = PendSV_Handler               │                         │
│  └──────────────┬──────────────────────┘                         │
│                 ▼                                                │
│  ┌─────────────────────────────────────┐                         │
│  │  PendSV_Handler (软件):              │                         │
│  │  PUSH {R4~R11} → 任务 A 栈          │  ← 完成全程保存!          │
│  │  保存 PSP → A.TCB                   │     此时栈 = 16 words    │
│  │                                     │                         │
│  │  vTaskSwitchContext()               │                         │
│  │    → pxCurrentTCB = B.TCB          │                         │
│  │                                     │                         │
│  │  从 B.TCB 加载 PSP                   │                         │
│  │  POP {R4~R11} ← 从任务 B 栈         │                         │
│  │  设置 PSP                            │                         │
│  │  BX LR (EXC_RETURN)                │                         │
│  └──────────────┬──────────────────────┘                         │
│                 ▼                                                │
│  ┌─────────────────────────────────────┐                         │
│  │  硬件自动出栈 (从任务 B 的栈):        │                         │
│  │  POP {R0,R1,R2,R3,R12,LR,PC,xPSR}  │                         │
│  │  PC = 任务 B 上次被打断的位置!        │                         │
│  │  SP 切回 PSP                         │                         │
│  └──────────────┬──────────────────────┘                         │
│                 ▼                                                │
│  ┌────────┐                                                      │
│  │ 任务 B  │  继续执行 ── 像什么都没发生过一样!                    │
│  └────────┘                                                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.14.10 关键数字速记

```
┌─────────────────────────────────────────────────────────────┐
│           Cortex-M4F 上下文切换 数字速记                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  寄存器总数量:      21 个 (R0~R15 + xPSR + 4 个特殊寄存器)     │
│  每个寄存器宽度:    32 bits = 4 bytes                        │
│                                                             │
│  硬件自动压栈:      8 registers = 32 bytes                   │
│  软件手动压栈:      8 registers = 32 bytes (R4~R11)          │
│  FPU lazy:         17 registers = 68 bytes (仅当需要时)      │
│                                                             │
│  一次完整切换:      16 words = 64 bytes 在栈上               │
│  中断延迟:          12 cycles (最简情况)                      │
│  上下文切换:        ~80-150 cycles (含 PendSV + 调度逻辑)     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 10.16 工业级 Bootloader 跳转 App 的 5 步安全规范契约

### 10.15.1 问题引出

```
┌──────────────────────────────────────────────────────────────────┐
│                    三个核心问题                                    │
├──────────────────────────────────────────────────────────────────┤
│  Q1: Bootloader 怎么跳到软件区？                                  │
│  Q2: 跳转后为什么不回 Bootloader？                                │
│  Q3: 是否可以多级跳来跳去？                                       │
│                                                                  │
│  三个问题指向同一个核心:                                           │
│  "CPU 只知道 PC 指向哪里就执行哪里，根本不懂什么叫 Bootloader"      │
└──────────────────────────────────────────────────────────────────┘
```

### 10.15.2 物理布局

```
PY32F403 Flash 384KB (0x08000000 ~ 0x08060000)
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  0x08000000 ┌──────────────────────┐                             │
│             │  Bootloader (16KB)    │  ← 出厂固化/烧录一次         │
│             │  • 向量表 @ 0x08000000│                              │
│             │  • 代码    @ 0x08000000~0x08003FFF                │
│             │  • 负责检查是否有新固件│                              │
│             │  • 有→升级; 无→跳转   │                              │
│  0x08004000 ├──────────────────────┤ ← APP_START_ADDRESS         │
│             │                      │                              │
│             │  Application (368KB)  │  ← 用户固件                  │
│             │  • 向量表 @ 0x08004000│                              │
│             │  • 代码    @ 0x08004000~0x0805FFFF                │
│             │                      │                              │
│  0x08060000 └──────────────────────┘                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.15.3 Q1：Bootloader 怎么跳到软件区？

**跳转前必须完成的清理工作：**

```c
void vBootloader_JumpToApplication(uint32_t ulAppAddr)
{
    /* ── Step 1: 禁止所有中断 ── */
    __disable_irq();                          // CPSID I

    /* ── Step 2: 关闭所有已使能的外设 ── */
    SysTick->CTRL = 0;                        // 停 SysTick (FreeRTOS 心跳!)
    // USART_DeInit(...);  GPIO_DeInit(...);  SPI_DeInit(...);
    // 每个打开的外设都要关，否则跳到 App 后外设还在跑!

    /* ── Step 3: 清除所有挂起的中断 ── */
    for (uint32_t i = 0; i < 8; i++) {
        NVIC->ICPR[i] = 0xFFFFFFFF;           // 清除所有挂起的 IRQ
    }

    /* ── Step 4: 恢复 MSP 为默认值 ── */
    __set_MSP(*(uint32_t *)0x20010000);       // MSP = SRAM 顶
}
```

**跳转的核心三步：**

```c
#define APP_START_ADDRESS  0x08004000   // App 固件起始地址

void vBootloader_JumpToApplication(uint32_t ulAppAddr)
{
    typedef void (*pFunction)(void);    // 函数指针类型
    pFunction pfAppEntry;               // App 复位入口
    uint32_t ulAppStackTop;             // App 栈顶

    /* ── 步骤1: 读 App 向量表的 [0] 和 [1] ── */
    /* 向量表 [0] = SP 初值, 向量表 [1] = Reset_Handler 地址 */
    ulAppStackTop = *((__IO uint32_t *)ulAppAddr);       // 读地址 0x08004000
    pfAppEntry    = (pFunction)(*((__IO uint32_t *)(ulAppAddr + 4))); // 读 0x08004004

    /*
     * 内存视角:
     * 0x08004000: [0x20010000]  ← App 要求的栈顶 (例如 SRAM 顶)
     * 0x08004004: [0x08004101]  ← App 的 Reset_Handler 地址 (bit0=1 → Thumb)
     *                              ↑
     *                              └── 注意 LSB = 1 表示 Thumb 代码
     *                                  如果 LSB = 0 → INVSTATE UsageFault!
     */

    /* ── 步骤2: 设置 MSP 为 App 的栈顶 ── */
    __set_MSP(ulAppStackTop);

    /* ── 步骤3: 设置 VTOR → 跳转 ── */
    SCB->VTOR = ulAppAddr;  // VTOR = 0x08004000

    /* 跳转! 这一步永不返回 */
    pfAppEntry();  // 等效: PC = 0x08004101, CPU 开始执行 App 的 Reset_Handler
}
```

### 10.15.4 跳转发生时 CPU 视角的逐周期分解

```
┌──────────────────────────────────────────────────────────────────┐
│          pfAppEntry() 执行时的 CPU 微观过程                        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  汇编层面:                                                        │
│    pfAppEntry = (pFunction)(0x08004101);                         │
│    pfAppEntry();                                                 │
│                                                                  │
│  对应的汇编指令:                                                   │
│    LDR  R0, =0x08004101     ; R0 = App Reset_Handler 地址        │
│    BLX  R0                  ; Branch with Link and eXchange      │
│                                                                  │
│  BLX R0 执行时 CPU 做的事:                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  ① LR = 下一条指令地址                                      │  │
│  │     ↑ 但 App 永远用不到这个 LR, 因为 App 的 Reset_Handler    │  │
│  │       不会 BX LR, 而是直接跑 main()                         │  │
│  │                                                           │  │
│  │  ② PC = R0 & 0xFFFFFFFE    ; 跳到 0x08004100 (清 bit0)     │  │
│  │     ↑ BLX 清 bit0 用于取指, 但 CPU 内部记住 bit0=1          │  │
│  │       → EPSR.T = 1 →  CPU 处于 Thumb 状态                 │  │
│  │                                                           │  │
│  │  ③ App 的 Reset_Handler 第一条指令开始执行                   │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  关键点:                                                          │
│  • Bootloader 的 PC 和 LR 值凭空消失 (没有代码保存它们)            │
│  • App 的 Reset_Handler 就像 CPU 刚上电一样从头初始化              │
│  • 唯一区别: 不是硬件复位触发, 而是函数调用触发                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.15.5 App 的 main() 必须做的第一件事

```c
int main(void)
{
    /* ⚠️ 必须放在所有代码之前! */
    SCB->VTOR = APP_START_ADDRESS;  // 0x08004000

    /*
     * 为什么 Bootloader 已经设过 VTOR, App 还要设一次?
     *
     * 答: 如果 App 发生了任何复位 (WDT, 软件复位 NVIC_SystemReset),
     *     CPU 硬件复位后 VTOR 自动回到 0x08000000!
     *     如果此时 App 的向量表在 0x08004000,
     *     但 VTOR 指回了 0x08000000 → HardFault!
     *
     *     所以在 main() 开始时重新设一次 VTOR 是防御性的.
     */

    HAL_Init();
    SystemClock_Config();
    // ... 其他初始化 ...
}
```

### 10.15.6 Q2: 为什么跳过去后不会回到 Bootloader？

**答案：因为 CPU 层面"没有人保存回去的路"。**

```
错误的心智模型 (函数调用):
  Bootloader                       App
  ┌─────────┐                    ┌─────────┐
  │  ...    │                    │         │
  │  Jump()─┼──────────────────▶ │ 处理...  │
  │  (等待) │ ← 期待返回?         │         │
  │  继续... │                    │  return ─┼──▶ 回到 Bootloader
  └─────────┘                    └─────────┘
  ❌ 不是这样的!

正确的模型 (程序替换):
  Bootloader                       App
  ┌─────────┐                    ┌─────────┐
  │  ...    │                    │         │
  │  Jump()─┼────BLX─────────▶  │ Reset_Handler
  │  (永远  │                    │ → SystemInit
  │   消失) │                    │ → main()
  └─────────┘                    │ → vTaskStartScheduler()
       ↑                         │ → 任务循环 (死循环)
       代码还在 Flash 里           └─────────┘
       但 CPU 再也不会碰它              ↑
                                 接管了 CPU 的控制权
```

### 10.15.7 那 App 怎么回到 Bootloader？— 只有通过"复位"

```c
/* 方法1: 软件触发系统复位 (推荐) */
void vJumpBackToBootloader_ByReset(void)
{
    __disable_irq();
    
    /*
     * 在备份寄存器中写标志 (复位后保持):
     * PY32F403 有 BKP 备份寄存器
     */
    BKP->DR1 = 0xDEADBEEF;  // 写标志: "下次启动进 Bootloader"

    NVIC_SystemReset();     // 系统复位!
    /*
     * 复位后:
     * PC  = *(0x08000004) = Bootloader 的 Reset_Handler ← 回到 Bootloader!
     * MSP = *(0x08000000) = SRAM 顶
     *
     * Bootloader 的 main() 中检查 BKP->DR1:
     * if (BKP->DR1 == 0xDEADBEEF) {
     *     留在 Bootloader 模式 (等待升级)
     *     BKP->DR1 = 0;
     * } else {
     *     跳到 App
     * }
     */
}

/* 方法2: 直接跳到 Bootloader 的 Reset_Handler (危险! 不推荐) */
void vJumpBackToBootloader_Direct(void)
{
    /*
     * 这是可以的, 但极度危险:
     * • App 打开的外设还在运行!
     * • 中断可能未清
     * • Bootloader 不会重新执行 __main() 的运行时初始化
     * • 全局/静态变量不会重新初始化 → 值不可预测!
     *
     * 所以: 直接用 NVIC_SystemReset() 是最干净的方案
     */
}
```

### 10.15.8 Q3: 可以跳来跳去吗？多级跳转？

**答案：可以！但每一跳都是"有去无回"。**

```
┌──────────────────────────────────────────────────────────────────┐
│                    多级引导链 (Chain Loading)                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  启动链:                                                          │
│    硬件复位                                                     │
│      │                                                           │
│      ▼                                                           │
│    Bootloader (验证签名 + 决定启动哪个 App)                        │
│      │                                                           │
│      ├─ 条件1 → 跳到 App_A @ 0x08004000                         │
│      │              │                                            │
│      │              └─ OTA 升级完成 → 更新标志 → 复位             │
│      │                                                           │
│      └─ 条件2 → 跳到 App_B @ 0x08032000                         │
│                     │                                            │
│                     └─ 验证失败 → 更新标志 → 复位                 │
│                                                                  │
│  每一跳都是:                                                      │
│    目标向量表[0] → MSP                                           │
│    目标向量表[1] → PC  (Reset_Handler)                           │
│    → 上一级永久消失                                               │
│                                                                  │
│  安全的 "来回跳" 方案: 永远走系统复位 + 备份寄存器标志            │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.15.9 安全的多级跳转方案

```c
/* ═══════════════════════════════════════════════════════════════
   推荐的 "跳来跳去" 方案: 永远走系统复位 + 标志位
   ═══════════════════════════════════════════════════════════════ */

/* 在 Bootloader 中统一管理启动目标: */
int main(void)
{
    /* ── 读启动原因 ── */
    uint32_t ulBootReason = BKP->DR2;    // 备份寄存器 (复位不丢失)
    BKP->DR2 = 0;                         // 清除

    /* 根据标志设置 VTOR */
    if (ulBootReason == 0x0000AAAA) {
        SCB->VTOR = APP_A_ADDR;          // 启动 App_A
    } else if (ulBootReason == 0x0000BBBB) {
        SCB->VTOR = APP_B_ADDR;          // 启动 App_B
    } else {
        SCB->VTOR = BOOT_ADDR;           // 正常进 Bootloader
    }
    
    /* 此后跳转... */
}

/* App 中触发切换: */
void vApp_RequestRebootToAppB(void)
{
    BKP->DR2 = 0x0000BBBB;       // 写标志
    NVIC_SystemReset();          // 复位
    /* 复位 → Bootloader → 检查标志 → 跳转到 App_B */
}
```

### 10.15.10 多级跳转的四大陷阱

```
┌──────────────────────────────────────────────────────────────────┐
│                    四大陷阱                                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  陷阱1: 外设状态污染                                              │
│  App_A 使能了 USART1 DMA → 跳到 App_B 时 DMA 还在工作!            │
│  解决: 跳转前 DEINIT 所有外设, 或通过系统复位跳转                 │
│                                                                  │
│  陷阱2: 中断向量表偏移                                            │
│  跳转前必须更新 SCB->VTOR = 目标地址                              │
│  否则目标程序中发生中断 → 跳到源程序的 ISR → 代码跑飞             │
│                                                                  │
│  陷阱3: C 运行时状态残留                                          │
│  直跳 (非复位) 方式下, 目标程序的 Reset_Handler 需要完整           │
│  重新初始化 .data 和 .bss                                         │
│                                                                  │
│  陷阱4: 栈冲突                                                    │
│  如果两个程序都把栈放在同一位置, 需要 __set_MSP 重新设置          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 10.15.11 跳转机制本质总结

```
┌──────────────────────────────────────────────────────────────────┐
│              跳转的本质 — 三层抽象                                  │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  第一层: 物理层 — 代码在 Flash, 跳转只是改变地址                   │
│    [Bootloader]        [App_A]             [App_B]               │
│    0x08000000          0x08004000          0x08032000            │
│    都是 Flash 中的代码, CPU 执行哪段完全由 PC 决定                 │
│                                                                  │
│  第二层: 约定层 — 向量表决定了"程序的起点"                          │
│    每个 "程序" 的要求:                                             │
│    • [0] = 栈顶地址 (SP 初始值)                                   │
│    • [1] = 第一条指令 (Reset_Handler)                             │
│    • 自己的异常处理函数表                                          │
│                                                                  │
│  第三层: 运行时层 — 跳转 = 破坏当前状态 + 建立新状态                │
│    跳转做的事:                                                     │
│      ① MSP = 目标的栈顶        → "栈换掉"                         │
│      ② VTOR = 目标的向量表     → "出事了找对的人"                  │
│      ③ PC  = 目标的 Reset_Handler → "开始执行新程序"               │
│                                                                  │
│    这些操作不可逆: 旧程序的 PC/LR/SP 全部丢失                      │
│    → 所以跳转是 "有去无回"                                        │
│    → 想 "回去" 只能通过硬件复位 + 标志位                           │
│                                                                  │
│  结论:                                                             │
│    Q1: 跳转 = 读目标向量表 → 设 MSP → 设 VTOR → 跳 PC            │
│    Q2: 不会回来 = 没有人保存"回家的路" + 所有状态被覆盖             │
│    Q3: 可以多级跳 = 每一跳都是新的有去无回                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**一句话总结：** CPU 没有"程序"的概念，只有 PC 指向哪里就执行哪里。Bootloader 跳 App 的本质是**故意丢弃 Bootloader 的整个运行时上下文**，用 App 的向量表重建一个干净的执行环境。这个动作是不可逆的——除非通过硬件复位重新开始。

---

## 第10课课后总结图

```
               ┌──────────────────────┐
               │  链接脚本自定义段       │
               │  变量放指定 RAM 区域    │
               └──────────┬───────────┘
                          │ 定义变量存储位置
                          ▼
┌──────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ 向量表          │   │     SRAM 64KB         │   │ 数组越界              │
│ 存在 Flash 头    │──▶│  0x20000000~0x20010000 │◀──│ 破坏邻居变量          │
│ 异常处理入口     │   │                        │   │ 本质是内存布局问题     │
└──────────────┘   └───────────┬────────────┘   └──────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
     ┌────────────┐   ┌────────────┐   ┌────────────┐
     │ 上电启动     │   │ PC/LR/SP   │   │ 栈空间分配  │
     │ Flash→SRAM │   │ 寄存器角色  │   │ 任务栈/MSP  │
     │ 向量表→PC   │   │ CPU 执行流  │   │ 局部变量存储 │
     └────────────┘   └────────────┘   └────────────┘

Cortex-M4F 上下文切换:
  ┌──────────────────────────────────────────────────────┐
  │  硬件压栈 (8×32bit)  +  软件压栈 (8×32bit)            │
  │  =  16 words  =  64 bytes  一个完整任务栈帧           │
  │                                                      │
  │  PendSV (软件) 负责 R4~R11                           │
  │  异常返回 (硬件) 自动恢复 R0~R3,R12,LR,PC,xPSR       │
  └──────────────────────────────────────────────────────┘

Bootloader 跳转:
  ┌──────────────────────────────────────────────────────┐
  │  读目标向量表[0]→MSP, [1]→PC, 设VTOR → 跳!          │
  │  一去不回, 想回需要 NVIC_SystemReset()               │
  └──────────────────────────────────────────────────────┘

一切围绕一个核心:
  "CPU 从哪里取指令(PC), 数据存在哪里(SRAM),
   出错了去哪处理(向量表), 这些都由链接脚本和启动代码决定"
```

---

### 10.17 第10课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **堆算法选型：** 能否向他人解释 `heap_4` 相比于 `heap_2` 最大的质变在哪里？为什么工控推荐静态分配？
- [ ] **金丝雀检测：** FreeRTOS 栈溢出检测方法 2（Canary）是如何利用魔数 `0xA5` 精准捕获栈越界的？
- [ ] **Program Size：** 能否根据 `Code`, `RO`, `RW`, `ZI` 准确算出程序占用了多少 Flash 和 RAM？
- [ ] **显存预算：** 面对 64KB RAM 跑图形界面的需求，为什么必须将全屏缓冲剪裁为 1/10 行缓冲？
- [ ] **调用栈寄存器：** 能否讲清 PC 指针、LR 链接寄存器与 MSP/PSP 在上下文切换中的分工？
- [ ] **启动顺序：** 单片机加电后执行的第一行 C 语言代码前，汇编代码完成了哪些必要的数据段搬运？
- [ ] **Bootloader 跳转：** 在跳转进入 App 前，为什么必须执行关中断和关闭 SysTick 的操作？

***

# 第二部分：第11课 — 低功耗 — Tickless Idle 模式

> **核心主题：** 低功耗架构与动态节拍抑制 (Tickless Idle) — 从毫秒空闲到微安深度休眠
> 
> **📎 配套代码文件:**
>
> - `Lesson11_Tickless_Basic.md` — 基础版：通用 Tickless Idle (Sleep 模式)
> - `Lesson11_Tickless_Advanced.md` — 进阶版：Stop 模式深度休眠 + 功耗测量任务

***

```
▎ 课程目标： 掌握 configUSE_TICKLESS_IDLE 工作原理、vPortSuppressTicksAndSleep() 源码级分析、
           PY32F403 三种低功耗模式 (Sleep / Stop / Standby) 集成、唤醒源配置、实际功耗测量与优化
▎ 硬件平台： PY32F403-STK (PB13=LED0, PA1=LED1, PB12=LED2)
▎ 配置基础： FreeRTOSConfig.h (configUSE_TICKLESS_IDLE=1, configCPU_CLOCK_HZ=8000000)
```

## 11.1 什么是 Tickless Idle？— 核心原理

### 11.1.1 问题的根源：SysTick 的能耗代价

在默认的 FreeRTOS 调度中，SysTick 定时器以固定频率（例如 100Hz）产生中断。这意味着：

```
传统 Tick 模式 (configTICK_RATE_HZ=100):
┌─────────────────────────────────────────────────────┐
│  时间轴:  0ms    10ms   20ms   30ms   40ms   50ms  │
│  SysTick:  ↑      ↑      ↑      ↑      ↑      ↑    │
│  CPU状态:  RUN  SLEEP  RUN  SLEEP  RUN  SLEEP      │
│                                                     │
│  问题: 即使无任务可运行，SysTick 每 10ms 唤醒一次    │
│        CPU 从 Sleep 被唤醒 → 执行 ISR → 检查无任务  │
│        → 再次 Sleep → 10ms 后又醒来...               │
│                                                     │
│  功耗图:  ████░████░████░████░████░████            │
│           ↑ 每次尖峰 = SysTick ISR 唤醒功耗          │
│                                                     │
│  电流消耗: ~5-10mA (持续周期性唤醒)                  │
└─────────────────────────────────────────────────────┘
```

**关键洞察：** 假设系统空闲 500ms，传统模式会产生约 50 次无效唤醒。每次唤醒的过程：
1. SysTick 中断触发 → CPU 退出 Sleep
2. 执行 xPortSysTickHandler() → 递增 Tick 计数
3. 检查是否有任务就绪 → 没有
4. 空闲任务重新进入 Sleep
5. 来回功耗约 2-3μJ，50次 = 100-150μJ 白白浪费

### 11.1.2 Tickless 的解决方案

```
Tickless 模式 (configUSE_TICKLESS_IDLE=1):
┌─────────────────────────────────────────────────────┐
│  时间轴:  0ms                          500ms        │
│  SysTick:  ↑              (SLEEPING)               ↑ │
│  CPU状态:  RUN ▄▄▄▄▄▄▄▄ DEEP SLEEP ▄▄▄▄▄▄▄▄  RUN │
│                                                     │
│  策略: 空闲时关掉 SysTick, 用硬件定时器设一个        │
│         "闹钟", CPU 进入深度休眠, 到期自动唤醒       │
│                                                     │
│  功耗图:  ████________________________________████  │
│           ↑                                      ↑  │
│           只有必要的唤醒                             │
│                                                     │
│  电流消耗: ~μA 级 (深度休眠期间)                     │
└─────────────────────────────────────────────────────┘
```

**核心思想：** "没事干就彻底睡，设个闹钟，到点再起来工作。" FreeRTOS 空闲任务检测到没有任务需要运行后，调用 `vPortSuppressTicksAndSleep()`：
1. **计算** 距离下一个任务到期还有多长时间
2. **关闭** SysTick 定时器
3. **设置** 一个硬件定时器在该时间后唤醒
4. **进入** 低功耗休眠模式
5. **唤醒后** 补偿丢失的 Tick 计数，恢复 SysTick

### 11.1.3 Tickless vs 传统 Tick 的本质区别

```
┌──────────────────┬─────────────────────┬─────────────────────┐
│     对比维度      │   传统 Tick 模式     │   Tickless 模式      │
├──────────────────┼─────────────────────┼─────────────────────┤
│ SysTick 行为      │ 始终以固定频率中断   │ 空闲时停止，有任务时恢复│
│ CPU 空闲功耗      │ 高 (持续唤醒)       │ 极低 (长时间深休眠)   │
│ 时间精度          │ 精确 (周期固定)     │ 微小漂移 (可接受)     │
│ 适用场景          │ 有精确时序要求       │ 电池供电 / 低功耗    │
│ RAM 开销          │ 极小                │ +~16 字节            │
│ ROM 开销          │ 0                   │ +vPortSuppressTicks  │
│ 实现复杂度        │ 无需额外配置         │ 需适配 MCU 低功耗    │
└──────────────────┴─────────────────────┴─────────────────────┘
```

## 11.2 configUSE_TICKLESS_IDLE 工作机制

### 11.2.1 使能条件

在 `FreeRTOSConfig.h` 中：

```c
#define configUSE_TICKLESS_IDLE  1   // 0 = 禁用, 1 = 启用

// 预编译检查 (FreeRTOS.h 内部):
// #if configUSE_TICKLESS_IDLE == 1
//     #if configUSE_PREEMPTION == 0
//         #error Tickless idle mode can only be used with pre-emption enabled.
//     #endif
//     #if configUSE_TIME_SLICING == 0
//         / * 没有时间片轮转也能工作, 但需注意... */
//     #endif
// #endif
```

### 11.2.2 触发路径：从空闲任务到深度休眠

```
调用链:

vTaskStartScheduler()
    │
    ▼
prvIdleTask()                           ← 空闲任务 (优先级 0, 永远就绪)
    │
    ├── 检查: 就绪列表中有更高优先级任务吗?
    │       ├── 有 → taskYIELD() → 切换
    │       └── 没有 → 系统真正空闲!
    │
    ▼
#if ( configUSE_TICKLESS_IDLE == 1 )
    │
    ├── 计算: xExpectedIdleTime = 距离下一个任务到期还有多少 Tick?
    │        │
    │        ├── 计算方法: 遍历延时列表 (xDelayedTaskList1/2),
    │        │   取最近到期时间 - 当前 Tick 计数
    │        │   → 例如: 现在 tick=1000, 下一个任务在 tick=1050 到期
    │        │   → xExpectedIdleTime = 50 ticks
    │        │
    │        └── 约束: xExpectedIdleTime >= 2 (至少 2 个 tick 才值得休眠)
    │                 configEXPECTED_IDLE_TIME_BEFORE_SLEEP (默认 2)
    │
    ├── if (xExpectedIdleTime >= 2):
    │       vPortSuppressTicksAndSleep(xExpectedIdleTime);  ← 关键函数!
    │   else:
    │       正常执行 __WFI() (不关 SysTick)
    │
#else
    └── __WFI();  ← 仅等待中断, SysTick 继续运行
#endif
```

### 11.2.3 关键宏定义

```c
// FreeRTOSConfig.h 中可配置的 Tickless 相关宏:

/* 休眠前最少需要多少 Tick 的空闲时间 (默认 2)
 * 设为 5 可避免短时间休眠带来的开销 */
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP    2

/* configPRE_SLEEP_PROCESSING(x):
 * 休眠前回调 — 用户在此配置 MCU 进入低功耗模式 (Sleep/Stop/Standby)
 * 参数 x: 期望空闲时间 (Tick 数), 可由回调修改
 * 若回调内部已执行 __WFI(), 将 x 设为 0, port.c 会跳过自己的 WFI */
#define configPRE_SLEEP_PROCESSING(x)    // 默认空, 需用户实现

/* configPOST_SLEEP_PROCESSING(x):
 * 唤醒后回调 — 恢复系统时钟、外设等
 * 参数 x: 原始期望空闲时间 (Tick 数) */
#define configPOST_SLEEP_PROCESSING(x)   // 默认空, 需用户实现
```

## 11.3 vPortSuppressTicksAndSleep() 源码级分析

> 源码位置：`Middlewares/Third_Party/FreeRTOS/Source/portable/RVDS/ARM_CM4F/port.c` 第 544 行
> 函数属性：`__weak` — 用户可在应用层重写以适配具体 MCU

### 11.3.1 完整执行流程

```
vPortSuppressTicksAndSleep(xExpectedIdleTime)
│
├─ Step 1: 边界检查
│   if (xExpectedIdleTime > xMaximumPossibleSuppressedTicks)
│       xExpectedIdleTime = xMaximumPossibleSuppressedTicks;
│   // 防止 ulReloadValue 溢出 24-bit SysTick 计数器
│   // xMaximumPossibleSuppressedTicks = 0xFFFFFF / ulTimerCountsForOneTick
│
├─ Step 2: 关中断 (自定义方式, 不用 taskENTER_CRITICAL)
│   __disable_irq();
│   __dsb(portSY_FULL_READ_WRITE);
│   __isb(portSY_FULL_READ_WRITE);
│   // 注意: 不用 taskENTER_CRITICAL() 因为它会屏蔽唤醒源中断!
│   //        这里关中断是为了原子性地操作 SysTick 寄存器
│
├─ Step 3: 确认允许休眠
│   if (eTaskConfirmSleepModeStatus() == eAbortSleep)
│   {
│       __enable_irq();   // 有任务在等待, 放弃休眠, 恢复中断
│       return;
│   }
│   // eAbortSleep 的场景:
│   //   - 有上下文切换正在等待 (PendSV pending)
│   //   - 调度器被暂停 (vTaskSuspendAll)
│   //   - 有更高优先级任务刚就绪
│
├─ Step 4: 停止 SysTick
│   portNVIC_SYSTICK_CTRL_REG = (SYSTICK_CLK_BIT | SYSTICK_INT_BIT);
│   // 保留: 时钟源 + Tick 中断使能
│   // 清除: ENABLE 位 → SysTick 计数器停止
│
├─ Step 5: 读取当前 SysTick 计数剩余值
│   ulSysTickDecrementsLeft = portNVIC_SYSTICK_CURRENT_VALUE_REG;
│   if (ulSysTickDecrementsLeft == 0)
│       ulSysTickDecrementsLeft = ulTimerCountsForOneTick;
│   // WHY: 如果计数器刚好到 0, 意味着一个完整 Tick 已过去
│   //       但中断还没处理, 所以剩余值 = 完整 Tick 周期
│
├─ Step 6: 计算重载值
│   ulReloadValue = ulSysTickDecrementsLeft
│                 + (ulTimerCountsForOneTick * (xExpectedIdleTime - 1));
│   //
│   // 公式分解:
│   //   当前 tick 周期剩余部分: ulSysTickDecrementsLeft
│   //   接下来的完整 tick 周期: (xExpectedIdleTime - 1) × ulTimerCountsForOneTick
│   //
│   // 例如: HSI=8MHz, Tick=100Hz
│   //   ulTimerCountsForOneTick = 8,000,000 / 100 = 80,000
│   //   xExpectedIdleTime = 50 ticks
│   //   ulSysTickDecrementsLeft = 30,000 (当前 tick 已过了 50,000)
│   //   ulReloadValue = 30,000 + 80,000 × 49 = 3,950,000
│
├─ Step 7: 检查 SysTick IRQ 是否 pending
│   if (portNVIC_INT_CTRL_REG & portNVIC_PEND_SYSTICK_SET_BIT)
│   {
│       portNVIC_INT_CTRL_REG = portNVIC_PEND_SYSTICK_CLEAR_BIT;
│       ulReloadValue -= ulTimerCountsForOneTick;
│   }
│   // 如果 SysTick 中断已经 pending, 说明一个 tick 已经完成
│   // 需要抑制它 (清理 pending) 并校正重载值
│
├─ Step 8: 补偿停止 SysTick 的时间
│   ulReloadValue -= ulStoppedTimerCompensation;
│   // ulStoppedTimerCompensation = 编译器相关的常量 (~45)
│   // 补偿我们在 Step 4~7 操作 SysTick 寄存器期间经过的 CPU 周期
│
├─ Step 9: 设置新重载值并重启 SysTick
│   portNVIC_SYSTICK_LOAD_REG = ulReloadValue;
│   portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;    // 清计数器
│   portNVIC_SYSTICK_CTRL_REG |= SYSTICK_ENABLE_BIT;  // 重启!
│   // 现在 SysTick 将在 ulReloadValue 个时钟周期后产生中断
│
├─ Step 10: 执行休眠前置处理并进入休眠
│   xModifiableIdleTime = xExpectedIdleTime;
│   configPRE_SLEEP_PROCESSING(xModifiableIdleTime);
│   // ★ 用户在这里配置 MCU 低功耗模式 (Sleep/Stop/Standby)
│   // ★ 如果 xModifiableIdleTime 被设为 0 → 表示用户已自行 WFI
│
│   if (xModifiableIdleTime > 0)
│   {
│       __dsb(portSY_FULL_READ_WRITE);
│       __wfi();   // ← CPU 在此处休眠!
│       __isb(portSY_FULL_READ_WRITE);
│   }
│
│   configPOST_SLEEP_PROCESSING(xExpectedIdleTime);
│   // ★ 用户在这里恢复系统时钟、外设
│
├─ Step 11: 开中断 (让唤醒源 ISR 立即执行)
│   __enable_irq();
│
├─ Step 12: 重新关中断 → 停 SysTick → 计算补偿
│   __disable_irq();
│   portNVIC_SYSTICK_CTRL_REG = (SYSTICK_CLK_BIT | SYSTICK_INT_BIT); // 停
│
│   if (portNVIC_SYSTICK_CTRL_REG & portNVIC_SYSTICK_COUNT_FLAG_BIT)
│   {
│       // COUNTFLAG=1 表示 SysTick 已经从 RELOAD 数到 0
│       // → 意味着我们休眠了计划的完整时间
│       // → 补偿完整的 xExpectedIdleTime 个 tick
│       ulCompleteTickPeriods = xExpectedIdleTime - 1UL;
│   }
│   else
│   {
│       // COUNTFLAG=0 表示 SysTick 还没数到 0
│       // → 我们被提前唤醒了 (外部中断)
│       // → 按比例补偿: 已完成的 tick 数 + 当前周期已过的部分
│       ulCompletedSysTickDecrements =
│           ulTimerCountsForOneTick - portNVIC_SYSTICK_CURRENT_VALUE_REG;
│       ulCompleteTickPeriods =
│           ulCompletedSysTickDecrements / ulTimerCountsForOneTick;
│   }
│
├─ Step 13: 恢复 SysTick 正常运行
│   portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;
│   portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
│   portNVIC_SYSTICK_CTRL_REG |= SYSTICK_ENABLE_BIT;
│   // 恢复为标准 Tick 模式, 以 ulTimerCountsForOneTick 为周期
│
├─ Step 14: 补偿 Tick 计数
│   vTaskStepTick(ulCompleteTickPeriods);
│   // 在 tasks.c 中: xTickCount += ulCompleteTickPeriods;
│   // 确保系统时间连续
│
└─ Step 15: 恢复中断
    __enable_irq();
    // 此时 SysTick 已恢复标准模式, 调度恢复正常
```

### 11.3.2 为什么 `__weak` — 需要用户重写?

默认实现只做 `__WFI()` (Sleep 模式)。对于需要深度休眠 (Stop/Standby) 的场合：

```c
// 默认实现的问题:
// ① 只进入 Sleep 模式 → 功耗优化有限 (仍有 mA 级)
// ② 没有配置唤醒源 → 可能永远醒不过来 (Stop/Standby 唤醒源需显式配置)
// ③ 没有处理时钟恢复 → Stop 模式关闭 HSE/PLL, 唤醒后需重新配置

// 最佳实践: 在 main.c 中提供自定义实现 (覆盖 __weak 版本)
```

## 11.4 PY32F403 三种低功耗模式深度对比

> 参考手册：PY32F403 Reference Manual — Power Control (PWR) 章节
> HAL 驱动：`hal_pwr.c` / `hal_pwr.h` — `HAL_PWR_EnterSLEEPMode()` / `HAL_PWR_EnterSTOPMode()` / `HAL_PWR_EnterSTANDBYMode()`

### 11.4.1 模式总览

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│    特性       │    Sleep     │    Stop      │   Standby    │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ CPU          │ 停止         │ 停止         │ 停止         │
│ HSE/PLL      │ 运行         │ 停止         │ 停止         │
│ HSI          │ 运行         │ 停止         │ 停止         │
│ LSI/LSE      │ 运行         │ 可选         │ 可选         │
│ SRAM         │ 保持         │ 保持         │ 丢失         │
│ 外设         │ 运行         │ 停止         │ 停止         │
│ GPIO 状态     │ 保持         │ 保持         │ 高阻态       │
│ I/O 唤醒      │ 任意 EXTI    │ 任意 EXTI    │ 仅 WakeUp Pin│
│ RTC 唤醒      │ ✅           │ ✅  (LSE)   │ ✅  (LSE)   │
│ 唤醒时间      │ <1μs (最快)  │ ~10μs       │ ~100μs (冷启动)│
│ 典型电流      │ ~5mA @8MHz   │ ~50μA       │ ~2μA        │
│ 唤醒入口      │ 中断返回     │ 中断返回     │ 复位向量    │
│ FreeRTOS 适配 │ 默认支持     │ 需自定义     │ 需自定义     │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### 11.4.2 Sleep 模式 — 默认 Tickless 方案

```c
// Sleep 模式是 Cortex-M 原生的 __WFI() 行为
// CPU 时钟停止, 所有外设继续运行
// 任何中断都能唤醒

// PY32F403 HAL 进入方式:
HAL_PWR_EnterSLEEPMode(PWR_SLEEPENTRY_WFI);  // 等同于 __WFI()
// 或
HAL_PWR_EnterSLEEPMode(PWR_SLEEPENTRY_WFE);  // __WFE()
```

**优点：** 唤醒延迟最小 (< 1μs)，任何外设中断都能唤醒，FreeRTOS 默认 port.c 已支持。
**缺点：** 省电有限 — 外设时钟仍在跑，UART/SPI/TIM 等外设仍在耗电。
**适用：** 对唤醒延迟敏感、外设需要持续运行 (如 UART 接收) 的场景。

### 11.4.3 Stop 模式 — 深度休眠 (推荐!)

```
Stop 模式下:
┌──────────────────────────────────────┐
│  HSE 振荡器 ──关                      │
│  PLL ──关                             │
│  HSI ──关                             │
│  Flash ──进入低功耗模式               │
│  电压调节器 ──可选低功耗模式 (LPDS)   │
│  SRAM ──保持                          │
│  GPIO ──保持状态                      │
│                                      │
│  可运行: LSI (内部低速 40KHz)         │
│          LSE (外部低速 32.768KHz)     │
│          RTC (基于 LSE)               │
│          IWDG (基于 LSI)              │
└──────────────────────────────────────┘
```

```c
// Stop 模式进入前必须:
// 1. 确保至少一个唤醒源已配置 (否则永远醒不过来!)
// 2. 恢复 HSE/PLL 的时钟配置 (唤醒后 HSI 自动启动)
// 3. 恢复 SysTick 时钟源 (基于 HCLK)

// PY32F403 HAL 进入方式:
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
// 参数1: PWR_MAINREGULATOR_ON       → 主调压器 (快速唤醒, 功耗较高)
//        PWR_LOWPOWERREGULATOR_ON    → 低功耗调压器 (较慢唤醒, 功耗更低)
// 参数2: PWR_STOPENTRY_WFI 或 PWR_STOPENTRY_WFE
```

**Stop 模式唤醒源：**
| 唤醒源 | 配置方式 | 说明 |
|--------|---------|------|
| 任意 GPIO EXTI | `HAL_GPIO_Init()` + `HAL_NVIC_EnableIRQ()` | 按键、外部信号 |
| RTC Alarm | `HAL_RTC_SetAlarm()` + `HAL_NVIC_EnableIRQ(RTC_Alarm_IRQn)` | 定时唤醒 |
| RTC WakeUp Timer | `HAL_RTCEx_SetWakeUpTimer()` | RTC 自带周期性唤醒 |
| IWDG 复位 | (不推荐 — 会发生复位) | 仅作为安全兜底 |
| USART (仅 LSE) | `HAL_UARTEx_EnableStopMode()` | 需 LSE 时钟源 |

**唤醒后的时钟恢复 (关键!):**
```c
// Stop 模式唤醒后:
// → HSI 自动启动 (8MHz)
// → HSE 和 PLL 处于关闭状态 (需手动恢复)
// → Flash 等待周期回到默认值
// → SysTick 使用 HSI 作为时钟源

// 必须在 configPOST_SLEEP_PROCESSING 中恢复:
void configPOST_SLEEP_PROCESSING(TickType_t xExpectedIdleTime)
{
    // ① 恢复 HSE
    // ② 恢复 PLL (如果使用)
    // ③ 切换系统时钟回 PLL
    // ④ 更新 SystemCoreClock 变量
    // ⑤ 恢复 Flash 等待周期
}
```

### 11.4.4 Standby 模式 — 极致省电

```c
// Standby 模式:
// → 1.2V 域断电 → SRAM/寄存器全部丢失
// → 仅备份域保持 (RTC, 备份寄存器)
// → 唤醒 = 冷启动 (通过复位向量)
// → FreeRTOS 所有上下文丢失!

HAL_PWR_EnterSTANDBYMode();
// 唤醒后: 检测 PWR_FLAG_SB 标志 → 判断是 Standby 唤醒还是正常上电
```

**Standby 唤醒源 (极有限):**
- WKUP Pin (PA0, PC13, PE6, PA2, PC5) 上升沿
- RTC Alarm
- RTC WakeUp Timer
- RTC Tamper

**⚠️ Standby 与 FreeRTOS 不兼容!** 因为唤醒后会复位，FreeRTOS 的所有 TCB、任务栈、队列等全部丢失。如需在 Standby 后使用 FreeRTOS，必须 **整个系统重新初始化**。

## 11.5 唤醒源配置 (基于 PY32F403-STK)

### 11.5.1 EXTI 按键唤醒 (GPIO)

```c
// PY32F403-STK 可用按键:
//   - PA0 (WKUP 按键) → 可用作 Standby 唤醒
//   - 任意 GPIO 配置为 EXTI

void Wakeup_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();   // Enable GPIOA clock

    /* PA0: WKUP button — configured as EXTI on falling edge */
    GPIO_InitStruct.Pin   = GPIO_PIN_0;
    GPIO_InitStruct.Mode  = GPIO_MODE_IT_FALLING;  // Interrupt on falling edge
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* Enable and set EXTI line 0 interrupt to priority 5 (above kernel) */
    HAL_NVIC_SetPriority(EXTI0_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
}
```

### 11.5.2 RTC 周期性唤醒 (最常用!)

```c
// RTC 基于 LSE (32.768KHz 外部晶振)
// 在 Stop 模式下仍可运行
// 可用于实现周期性数据采集 (如每 1 秒唤醒一次)

void RTC_WakeUp_Init(uint32_t wakeup_seconds)
{
    RTC_HandleTypeDef hrtc = {0};
    hrtc.Instance = RTC;
    hrtc.Init.HourFormat     = RTC_HOURFORMAT_24;
    hrtc.Init.AsynchPrediv   = 127;    // 32.768K / (127+1) = 256 Hz
    hrtc.Init.SynchPrediv    = 255;    // 256 / (255+1) = 1 Hz
    hrtc.Init.OutPut         = RTC_OUTPUT_DISABLE;
    HAL_RTC_Init(&hrtc);

    /* Enable RTC WakeUp timer */
    HAL_RTCEx_SetWakeUpTimer_IT(&hrtc, wakeup_seconds,
                                 RTC_WAKEUPCLOCK_CK_SPRE_16BITS);
    // wakeup_seconds = 1 → 每秒唤醒一次
}
```

### 11.5.3 低功耗定时器 (LPTIM) 唤醒

```c
// PY32F403 如果有 LPTIM 外设, 可以使用 LPTIM 作为唤醒定时器
// LPTIM 基于 LSI (40KHz) 或 LSE (32.768KHz), Stop 模式下可运行

// LPTIM 优点: 灵活的分频和周期配置
// LPTIM 缺点: 精度不如 RTC, 但配置更简单
```

## 11.6 程序流程图

### 11.6.1 Tickless Idle 全流程

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Tickless Idle 完整流程图                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────┐                                                        │
│   │ 调度器运行    │                                                       │
│   │ 查找就绪任务  │                                                       │
│   └──────┬──────┘                                                        │
│          │                                                               │
│          ├── 有就绪任务? → YES → 切换并运行任务                          │
│          │                                                               │
│          └── NO (系统空闲)                                                │
│               │                                                          │
│               ▼                                                          │
│   ┌──────────────────────────┐                                          │
│   │ prvIdleTask() 执行        │                                          │
│   │ ① 执行 Idle Hook         │                                          │
│   │ ② 检查: 有任务就绪吗?     │                                          │
│   └──────────┬───────────────┘                                          │
│              │                                                           │
│              ├── 有 → taskYIELD() → 切换                                │
│              │                                                           │
│              └── 没有 → 真正空闲!                                        │
│                   │                                                      │
│                   ▼                                                      │
│   ┌──────────────────────────────────────┐                              │
│   │ configUSE_TICKLESS_IDLE == 1 ?        │                              │
│   ├──────────────────────────────────────┤                              │
│   │                                      │                              │
│   │  YES ─────────────────────┐          │                              │
│   │                           │          │                              │
│   │  NO (传统模式)             │          │                              │
│   │    │                      │          │                              │
│   │    ▼                      ▼          │                              │
│   │  __WFI()        计算 xExpectedIdle   │                              │
│   │  (Sleep)         Time                │                              │
│   │                      │               │                              │
│   │                 < 2 ticks?            │                              │
│   │                   │                  │                              │
│   │             YES ──┼── NO             │                              │
│   │              │       │               │                              │
│   │              ▼       ▼               │                              │
│   │          __WFI()  vPortSuppress      │                              │
│   │          (不走    TicksAndSleep()     │                              │
│   │           Tickless)      │            │                              │
│   │                          ▼            │                              │
│   │              ┌─────────────────────┐ │                              │
│   │              │ ① __disable_irq()   │ │                              │
│   │              │ ② eTaskConfirm      │ │                              │
│   │              │    SleepModeStatus()│ │                              │
│   │              │    ├ eAbortSleep?   │ │                              │
│   │              │    │  → YES: 放弃   │ │                              │
│   │              │    └ NO: 继续       │ │                              │
│   │              │ ③ 停 SysTick        │ │                              │
│   │              │ ④ 读剩余计数值       │ │                              │
│   │              │ ⑤ 算重载值          │ │                              │
│   │              │ ⑥ 设新重载 → 重启   │ │                              │
│   │              │ ⑦ configPRE_SLEEP   │ │                              │
│   │              │    _PROCESSING()    │ │                              │
│   │              │    → HAL_PWR_Enter  │ │                              │
│   │              │      STOPMode()     │ │                              │
│   │              │ ⑧ __WFI() ─────┐   │ │                              │
│   │              └────────────────┼───┘ │                              │
│   │                               │     │                              │
│   │              ┌────────────────┘     │                              │
│   │              │ (唤醒!)               │                              │
│   │              ▼                      │                              │
│   │  ┌──────────────────────────┐       │                              │
│   │  │ ⑨ configPOST_SLEEP      │       │                              │
│   │  │    _PROCESSING()         │       │                              │
│   │  │    → 恢复HSE/PLL         │       │                              │
│   │  │    → 恢复Flash等待周期    │       │                              │
│   │  ├──────────────────────────┤       │                              │
│   │  │ ⑩ __enable_irq()        │       │                              │
│   │  │    → 唤醒源ISR立执       │       │                              │
│   │  ├──────────────────────────┤       │                              │
│   │  │ ⑪ 计算补偿Tick数         │       │                              │
│   │  │    ├ COUNTFLAG=1?        │       │                              │
│   │  │    │  YES→补偿完整Tick    │       │                              │
│   │  │    │  NO →按比例补偿      │       │                              │
│   │  │    │                     │       │                              │
│   │  │    vTaskStepTick(n)      │       │                              │
│   │  │    → xTickCount += n     │       │                              │
│   │  ├──────────────────────────┤       │                              │
│   │  │ ⑫ 恢复 SysTick 标准模式  │       │                              │
│   │  │ ⑬ __enable_irq()        │       │                              │
│   │  └──────────────────────────┘       │                              │
│   │                                     │                              │
│   └─────────────────────────────────────┘                              │
│                                                                          │
│   结果: CPU 深度休眠了 xExpectedIdleTime 个 Tick                         │
│         节省了 (xExpectedIdleTime - 1) 次无效唤醒                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 11.6.2 功耗模式选择决策树

```
                    ┌─────────────┐
                    │ 系统空闲了    │
                    │ 预计 > 2ms?  │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │             │
                    NO           YES
                    │             │
                    ▼             ▼
              简单 __WFI()   ┌─────────────────┐
              (Sleep模式)    │ 需要外设持续运行?  │
                             │ (UART/SPI/TIM)   │
                             └────────┬────────┘
                                      │
                               ┌──────┴──────┐
                               │             │
                              YES            NO
                               │             │
                               ▼             ▼
                          Sleep 模式     ┌──────────────┐
                          (__WFI)        │ 需要保持SRAM?  │
                          ~5mA           └──────┬───────┘
                                                │
                                         ┌──────┴──────┐
                                         │             │
                                        YES            NO
                                         │             │
                                         ▼             ▼
                                    Stop 模式     Standby 模式
                                    (LPDS)        (冷启动)
                                    ~50μA         ~2μA
                                    推荐方案!      需重新初始化
```

## 11.7 代码实战

> **📎 完整代码已独立为配套文件，以下为架构概述。**

### 11.7.1 Demo1: 通用 Tickless Idle (基础 Sleep 模式) → [Lesson11_Tickless_Basic.md](Lesson11_Tickless_Basic.md)

最基础方案 — 仅需 1 行配置变更：`configUSE_TICKLESS_IDLE = 1`。利用 FreeRTOS 默认 `__weak vPortSuppressTicksAndSleep()` 进入 Sleep 模式。

**核心组成：**
| 文件/配置 | 作用 |
|-----------|------|
| `FreeRTOSConfig.h` | `configUSE_TICKLESS_IDLE = 1` |
| `vLedTask` | 500ms LED 翻转 (制造空闲窗口) |
| `vIdleMonitorTask` | 每秒打印 Idle Hook 调用次数 |
| `vApplicationIdleHook` | 累加 `g_sys.idle_count` 验证 Tickless |

**验证效果：** Idle Hook 调用频率从 ~100/s (传统) → ~2/s (Tickless)，电流 ~8mA → ~5mA。

### 11.7.2 Demo2: Stop 模式深度休眠 (进阶) → [Lesson11_Tickless_Advanced.md](Lesson11_Tickless_Advanced.md)

自定义 `vPortSuppressTicksAndSleep()` 覆盖 port.c 的 `__weak` 版本，用 **RTC WakeUp 定时器 + Stop 模式** 实现 μA 级待机。

**架构要点：**

```
自定义 vPortSuppressTicksAndSleep() 的 15 步执行:
  Step 0:  判断 xExpectedIdleTime → < 10 ticks 用 Sleep, ≥ 10 用 Stop
  Step 1-2: 关中断 + 确认允许休眠
  Step 3-6: 保留 port.c 原始 SysTick 计时逻辑
  Step 7:   调用 Tickless_Enter_StopMode()
               → RTC WakeUp 设闹钟 → HAL_PWR_EnterSTOPMode()
  Step 8:   configPOST_SLEEP_PROCESSING() → 恢复 HSE/PLL/Flash
  Step 9-14: 开中断 → 停 SysTick → 补偿 Tick → 恢复标准模式
```

**新增文件：**
- `Lesson11_Tickless_Advanced.h` — `TicklessStats_t` 结构体 + 函数声明
- `Lesson11_Tickless_Advanced.c` — 完整自定义 Tickless 实现 (~350 行)
- `main.c` 补充 — `vPowerMonitorTask` 功耗监控任务

**功耗对比：**
| 模式 | 电流 | 2000mAh 电池寿命 |
|------|------|-----------------|
| 传统 Tick | ~8mA | ~10 天 |
| Tickless Sleep | ~5mA | ~16 天 |
| **Tickless Stop** | **~50μA** | **~4.5 年** |

## 11.8 编译验证 → 详见 [Lesson11_Tickless_Advanced.md §5](Lesson11_Tickless_Advanced.md)

关键编译前提：
- 工程需添加 `py32f403_hal_pwr.c`、`py32f403_hal_rtc.c`、`py32f403_hal_flash.c`
- `FreeRTOSConfig.h` 中 `configUSE_PREEMPTION=1` (Tickless 强制要求)
- `py32f403_hal_conf.h` 中启用 `HAL_RTC_MODULE_ENABLED`

## 11.9 调试技巧

### 11.9.1 验证 Tickless 是否生效 — 三招

**方法 1: Idle Hook 计数法**

```c
// 在 vApplicationIdleHook 中累加计数器
// Tickless 生效时, Idle Hook 调用频率 = 任务周期频率 (非 SysTick 频率)
// 100Hz Tick, 500ms LED delay:
//   传统模式: Idle Hook ~100次/秒
//   Tickless:  Idle Hook ~2次/秒 (只在 LED 任务到期时唤醒)
```

**方法 2: 万用表电流测量**

```
连接: 万用表串接在 VDD 线上, 量程 20mA / 200μA
┌─────────────────────────────────────────────────┐
│  模式            │ configUSE_TICKLESS_IDLE │ 电流 │
├──────────────────┼───────────────────────┼──────┤
│  传统模式         │          0            │ ~8mA │
│  Tickless Sleep   │          1 (默认)     │ ~5mA │
│  Tickless Stop    │          1 (自定义)    │ ~50μA│
│  Tickless Standby │          1 (自定义)    │ ~2μA │
└──────────────────┴───────────────────────┴──────┘
```

**方法 3: 串口输出 Tick 补偿信息**

```c
// 在 vPortSuppressTicksAndSleep 的 configPOST_SLEEP_PROCESSING 后:
// 记录补偿的 Tick 数 → 输出到串口
// 如果 ulCompleteTickPeriods > 0 → Tickless 确实发生了!
```

### 11.9.2 Stop 模式调试注意事项

```
⚠️ 重要: MCU 在 Stop 模式下 SWD 调试接口不可用!

调试 Stop 模式技巧:
  ① 用串口输出代替断点 (Stop 期间 UART 也停了, 但唤醒后可输出)
  ② 用 LED 闪烁指示状态 (GPIO 在 Stop 期间保持)
  ③ 在 configPOST_SLEEP_PROCESSING 开始时翻转一个 GPIO → 示波器测唤醒时刻
  ④ 使用 RTC 备份寄存器记录进入/退出次数
  ⑤ 在开发和调试阶段, 用 Sleep 模式代替 Stop 模式 (SWD 保持可用)
```

### 11.9.3 常见问题诊断

```
症状: 进入 Stop 模式后 MCU 再也醒不过来
原因: ① 没有配置唤醒源 (没有 EXTI/RTC 中断)
      ② 唤醒源中断优先级太高 → 被 FreeRTOS 屏蔽
      ③ 唤醒源的时钟在 Stop 期间停了 (用了 HSI/HSE 而非 LSE/LSI)

诊断方法:
  // 在进入 Stop 前加这段代码 — 如果 RTC 没配置好, 不会真正进入 Stop
  if (__HAL_RTC_GET_FLAG(&hrtc_tickless, RTC_FLAG_INITS) == RESET)
  {
      /* RTC not initialized — fallback to simple WFI */
      __wfi();
      return;
  }
```

## 11.10 常见陷阱

### 陷阱 1: 唤醒后忘记恢复时钟

```c
/* ❌ 错误: Stop 唤醒后直接使用 USART */
void configPOST_SLEEP_PROCESSING(TickType_t x)
{
    // 什么都没做! HSE/PLL 还是关着的!
    // SysTick 现在基于 HSI 8MHz, 但 tick rate 是按 72MHz 算的
    // → Tick 时间完全错误!
}

/* ✅ 正确: 必须恢复完整时钟树 */
void configPOST_SLEEP_PROCESSING(TickType_t x)
{
    SystemClock_Config();  // 完整重新配置时钟
    SystemCoreClockUpdate();
}
```

### 陷阱 2: configPRE_SLEEP_PROCESSING 中再次 WFI

```c
/* ❌ 错误: 双重 WFI */
void configPRE_SLEEP_PROCESSING(TickType_t x)
{
    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
    // 返回后 port.c 又执行一次 __wfi()
    // → 第二次 WFI 会立即返回 (已经醒过一次了)
    // → 但计时逻辑会乱掉
}

/* ✅ 正确: 告诉 port.c 我们已经做了 WFI */
void configPRE_SLEEP_PROCESSING(TickType_t x)
{
    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
    x = 0;  // ← 重要! 设为 0 告诉 port.c 跳过它的 __wfi()
}
```

### 陷阱 3: Stop 模式前未清除唤醒标志

```c
/* ❌ 错误: 旧的 WKUP 标志导致误唤醒 */
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
// 如果上次唤醒的 WUF 标志还在 → 立即唤醒, Stop 形同虚设

/* ✅ 正确: 进入前清除 */
__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU);
__HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
```

### 陷阱 4: 调试模式下的 SWD 问题

```
Stop 模式下 SWD (Serial Wire Debug) 的 DBG 模块也停止了
→ Keil 调试器 "断开连接"
→ 无法单步调试 Stop 模式内的代码

解决方案:
  ① 开发期间使用 Sleep 模式 (SWD 保持连接)
  ② 或使用 DBGMCU 冻结调试:
     HAL_DBGMCU_EnableDBGStopMode();  // 调试时不停 DBG 时钟
     // 但这样做功耗更高! 仅用于调试!
```

---

## 11.11 最佳实践与工程避坑准则

| 维度 | 关键原则 | 核心设计考虑 |
| :--- | :--- | :--- |
| **评估门槛** | 先评估是否真正需要 Tickless | 若 CPU 利用率 > 80%，进入休眠概率极低，开启收益有限反而增加时钟抖动 |
| **休眠阈值** | 适当调大 `EXPECTED_IDLE_TIME` | 设为 5~10 Ticks，避免休眠仅 1~2 Ticks 带来进入/退出的无谓开销 |
| **模式选择** | 首选 Stop 深度休眠 | 兼具 μA 级极致省电、SRAM 完整保持与 <15μs 快速唤醒特性 |
| **时钟恢复** | `POST_SLEEP` 必须重构时钟 | 唤醒后必须完整调用 `SystemClock_Config`，恢复 HSE/PLL 和 Flash 等待周期 |
| **防双重休眠**| 进入低功耗后设置 `x = 0` | 告知 port.c 跳过自带的 `__wfi()`，避免多余二次唤醒混乱 |
| **调试支持** | 调试时开启 DBG 模块时钟 | 调用 `HAL_DBGMCU_EnableDBGStopMode()`，防止 Stop 模式直接切断 SWD 调试器 |

---

## 11.12 低功耗模式终极选型决策树

```
                         系统有电池供电或低功耗要求？
                                    │
           ┌────────────────────────┴────────────────────────┐
           ▼                                                 ▼
      [ 有低功耗要求 ]                                  [ 传统供电/全速模式 ]
           │                                                 │
   预计空闲时间占比？                                  传统 Tick 固定轮转
     ├─ < 50% 忙碌系统 ──► 保持传统 Tick 或仅用 Sleep   (configUSE_TICKLESS_IDLE=0)
     │
     └─ > 50% 存在长空闲窗口
           │
           ├─ 需要 UART/SPI 持续高速接收?
           │   └─► 【Sleep 模式 Tickless】(~5mA，零唤醒延迟)
           │
           ├─ 空闲时间在 10ms ~ 几小时 且【需保留 RAM 状态】
           │   └─► 🎉【Stop 模式 Tickless + RTC 唤醒】(~50μA，电池续航数年!)
           │
           └─ 空闲长达数天且【允许冷启动复位重置】
               └─► 【Standby 待机模式】(~2μA，从复位向量重新引导)
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       三大低功耗模式与电池寿命对比 (2000mAh)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  传统固定 Tick 模式 (100Hz):   ~ 8 mA   ──► 续航仅约 10 天                  │
│  Tickless 基础 Sleep 模式:     ~ 5 mA   ──► 续航约 16 天                    │
│  Tickless 进阶 Stop 模式:      ~ 50 μA  ──► 续航长达 4.5 年 (!)             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11.13 第11课 知识复盘与自测清单

学完本课后，请在开发板前快速自检是否掌握以下要点：

- [ ] **能耗黑洞：** 传统固定 SysTick 节拍在长空闲场景下为何会白白浪费大量电能？
- [ ] **核心算法：** `vPortSuppressTicksAndSleep` 是如何计算休眠时长并关闭 SysTick 的？
- [ ] **时钟补偿：** 唤醒后内核是如何通过 `vTaskStepTick` 补偿丢失的系统节拍的？
- [ ] **三大模式：** 能否讲清 Sleep、Stop 和 Standby 在时钟、SRAM 保持和唤醒源上的核心差异？
- [ ] **时钟恢复：** 为什么从 Stop 模式唤醒后，必须在 `configPOST_SLEEP_PROCESSING` 中重新初始化 HSE/PLL？
- [ ] **双重休眠：** 为什么在 `configPRE_SLEEP_PROCESSING` 中进入低功耗后，必须将入参修改为 0？
- [ ] **实操验证：** 编译烧录 `Lesson11_Tickless_Basic`，通过万用表或 Idle 计数器观测到明显的功耗改善。
***

# 第三部分：踩坑记录与解决方案

> 📎 所有编译、链接、运行时错误的完整排查链路已独立至 **[FreeRTOS\_Debug\_Log.md](FreeRTOS_Debug_Log.md)**。
>
> 涵盖：
>
> - Session 1 — 3 个编译/链接错误 (configTICK\_TYPE\_WIDTH\_IN\_BITS / configMAX\_SYSCALL / vApplicationStackOverflowHook)
> - Session 2 — 6 个运行时/LED 不闪烁调试 (xPortSysTickHandler 隐式声明 / SVC\_Handler 重复定义 / configCPU\_CLOCK\_HZ 不匹配 / Indirect Routing 失败 / 诊断测试 / configASSERT 触发)
> - 最终有效配置总结

***

# 附录：移植检查清单

```
每次移植到新项目时，逐项检查：

□ 1. 添加了 6 个内核 .c + 1 个 port.c + 1 个 heap_4.c 到 Keil 工程
□ 2. Include Paths 新增了 FreeRTOS-Kernel/include 和 portable/RVDS/ARM_CM4F
□ 3. ..\Inc 在 Include Paths 中排在第一位
□ 4. 删除了旧版 FreeRTOS/Source/include 的 Include Path (避免头文件冲突)
□ 5. Inc/FreeRTOSConfig.h 存在且使用 V11.x 宏:
     □ configTICK_TYPE_WIDTH_IN_BITS TICK_TYPE_WIDTH_32_BITS
     □ configUSE_TIMERS / configUSE_EVENT_GROUPS / configUSE_STREAM_BUFFERS
     □ configSUPPORT_STATIC_ALLOCATION / configSUPPORT_DYNAMIC_ALLOCATION
□ 6. configMAX_SYSCALL_INTERRUPT_PRIORITY != 0 (通常 = 0x50)
□ 7. 删除了 configMAX_API_CALL_INTERRUPT_PRIORITY (让 FreeRTOS.h 自动推导)
□ 8. main.c 包含 #include "FreeRTOS.h" 和 #include "task.h"
□ 9. 实现了 vApplicationStackOverflowHook() (或 configCHECK_FOR_STACK_OVERFLOW=0)
□ 10. SysTick_Handler 中调用了 xPortSysTickHandler() (且在调度器启动后才调)
□ 11. SVC_Handler / PendSV_Handler 映射: FreeRTOSConfig.h 中 #define 宏映射 (Direct Routing)
□ 12. py32f403_it.c 中 注释掉 SVC_Handler 和 PendSV_Handler (由 port.c 经宏映射提供)
□ 13. configCPU_CLOCK_HZ 必须与实际系统时钟一致 (HSI = 8000000)
□ 14. 使用 pdMS_TO_TICKS(ms) 宏计算延时, 不硬编码 tick 数
□ 15. configASSERT 若触发 → 临时用 ((void)(x)) 绕过, 需进一步排查 NVIC 兼容性
□ 16. main() 中调用了 vTaskStartScheduler()
□ 17. 编译 → 0 Error, 0 Warning → 下载运行 → LED 闪烁
```

***

> **配套文件:**
>
> - `FreeRTOSConfig_Reference.md` — 完整 FreeRTOSConfig.h 模板
> - `FreeRTOS_Debug_Log.md` — 所有问题及解决方案
> - `Lesson2_Task_Basic.md` — 第2课 多任务
> - `Lesson2_Task_Advanced.md` — 第2课 栈监控
> - `Lesson3_Queue_Basic.md` — 第3课 队列通信
> - `Lesson3_Queue_Advanced.md` — 第3课 多队列架构
> - `Lesson4_Semaphore_Basic.md` — 第4课 信号量基础
> - `Lesson4_Semaphore_Advanced.md` — 第4课 生产者-消费者
> - `Lesson5_Timer_Basic.md` — 第5课 软件定时器
> - `Lesson5_Timer_Daemon.md` — 第5课 定时器守护任务
> - `Lesson6_EventGroup_Basic.md` — 第6课 事件组
> - `Lesson7_TaskNotify_Basic.md` — 第7课 任务通知
> - `Lesson8_Interrupt_Basic.md` — 第8课 中断管理
> - `Lesson9_Resource_Basic.md` — 第9课 资源管理
> - `Lesson10_Memory_Basic.md` — 第10课 内存管理 (heap_4)
> - `Lesson10_Memory_Advanced.md` — 第10课 内存管理 (静态分配+Bootloader)
> - `Lesson11_Tickless_Basic.md` — 第11课 Tickless Idle (Sleep模式)
> - `Lesson11_Tickless_Advanced.md` — 第11课 Tickless Idle (Stop深度休眠)

