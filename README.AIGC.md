# 详细工程文档：基于Cortex‑M3的轻量级RTOS

## 目录
- [详细工程文档：基于Cortex‑M3的轻量级RTOS](#详细工程文档基于cortexm3的轻量级rtos)
  - [目录](#目录)
  - [1. 工程概述](#1-工程概述)
  - [2. 文件结构与详细解析](#2-文件结构与详细解析)
    - [2.1 main.c](#21-mainc)
    - [2.2 task.c / task.h](#22-taskc--taskh)
      - [2.2.1 全局变量（task.c）](#221-全局变量taskc)
      - [2.2.2 核心函数](#222-核心函数)
      - [2.2.3 调度算法 `vTaskSwitchContext`](#223-调度算法-vtaskswitchcontext)
      - [2.2.4 延时列表管理](#224-延时列表管理)
    - [2.3 list.c / list.h](#23-listc--listh)
      - [2.3.1 数据结构](#231-数据结构)
      - [2.3.2 重要操作函数](#232-重要操作函数)
      - [2.3.3 关键宏](#233-关键宏)
    - [2.4 port.c / portmacro.h](#24-portc--portmacroh)
      - [2.4.1 portmacro.h 中的重要宏](#241-portmacroh-中的重要宏)
      - [2.4.2 port.c 关键函数](#242-portc-关键函数)
      - [2.4.3 栈初始化细节（`pxPortInitialiseStack`）](#243-栈初始化细节pxportinitialisestack)
    - [2.5 rtos.h / rtos\_config.h / projectdefs.h](#25-rtosh--rtos_configh--projectdefsh)
  - [3. 核心数据结构详解](#3-核心数据结构详解)
    - [3.1 链表节点 (ListItem\_t)](#31-链表节点-listitem_t)
    - [3.2 链表根节点 (List\_t)](#32-链表根节点-list_t)
    - [3.3 任务控制块 (TCB\_t)](#33-任务控制块-tcb_t)
    - [3.4 就绪列表与优先级位图](#34-就绪列表与优先级位图)
    - [3.5 延时列表与系统节拍](#35-延时列表与系统节拍)
  - [4. 上电工作流程（完整）](#4-上电工作流程完整)
    - [4.1 从 main() 到创建任务](#41-从-main-到创建任务)
    - [4.2 启动调度器 vTaskStartScheduler()](#42-启动调度器-vtaskstartscheduler)
    - [4.3 硬件初始化 xPortStartScheduler()](#43-硬件初始化-xportstartscheduler)
    - [4.4 启动第一个任务 (SVC 异常)](#44-启动第一个任务-svc-异常)
    - [4.5 系统节拍与任务切换 (PendSV)](#45-系统节拍与任务切换-pendsv)
  - [5. API 详细说明与使用示例](#5-api-详细说明与使用示例)
    - [5.1 静态创建任务 xTaskCreateStatic](#51-静态创建任务-xtaskcreatestatic)
    - [5.2 任务延时 vTaskDelay](#52-任务延时-vtaskdelay)
    - [5.3 主动让出 CPU taskYIELD](#53-主动让出-cpu-taskyield)
    - [5.4 临界段保护](#54-临界段保护)
      - [5.4.1 任务内使用（不可嵌套于中断）](#541-任务内使用不可嵌套于中断)
      - [5.4.2 中断内使用（可嵌套）](#542-中断内使用可嵌套)
  - [6. 配置选项详解 (rtos\_config.h)](#6-配置选项详解-rtos_configh)
  - [7. 如何自定义添加或删改功能](#7-如何自定义添加或删改功能)
    - [7.1 添加新用户任务](#71-添加新用户任务)
    - [7.2 修改优先级数量](#72-修改优先级数量)
    - [7.3 改变系统节拍频率](#73-改变系统节拍频率)
    - [7.4 增加动态任务创建 (xTaskCreate)](#74-增加动态任务创建-xtaskcreate)
    - [7.5 添加空闲任务钩子](#75-添加空闲任务钩子)
  - [8. 注意事项与调试建议](#8-注意事项与调试建议)
    - [8.1 栈大小估算](#81-栈大小估算)
    - [8.2 临界段使用原则](#82-临界段使用原则)
    - [8.3 调试方法](#83-调试方法)
    - [8.4 已知限制](#84-已知限制)
    - [8.5 扩展建议](#85-扩展建议)
  - [9. 开发调试与问题修复记录](#9-开发调试与问题修复记录)
    - [9.1 重复链表插入导致节点指针破坏](#91-重复链表插入导致节点指针破坏)
    - [9.2 优先级位图溢出（`__clz(0)` 导致负数）](#92-优先级位图溢出__clz0-导致负数)
    - [9.3 SysTick 中断入口名与向量表不匹配](#93-systick-中断入口名与向量表不匹配)
  - [总结](#总结)

---

## 1. 工程概述

本工程实现了一个**抢占式、优先级调度**的实时操作系统内核，专为 **ARM Cortex‑M3** 处理器（如 STM32F103）设计。核心特性：

- 静态分配任务控制块（TCB）和任务栈，无需动态内存分配。
- 支持最多 **32 个优先级**（由 `configMAX_PRIORITIES` 决定，不超过 32）。
- 同优先级任务采用**时间片轮转**（通过就绪列表的循环遍历实现）。
- 提供 `vTaskDelay()` 实现**非阻塞延时**，任务延时期间 CPU 运行其他就绪任务。
- 临界段保护支持嵌套，区分**任务级**与**中断级**版本。
- 使用 **SysTick** 作为系统心跳，**PendSV** 异常实现零延迟上下文切换。

工程代码风格清晰，去除了 FreeRTOS 中的动态内存和复杂配置，非常适合学习 RTOS 原理或用于资源受限的简单项目。

---

## 2. 文件结构与详细解析

### 2.1 main.c

**作用**：演示 RTOS 的使用方法，创建两个用户任务并启动调度器。

**关键代码解读**：

```c
// 任务控制块与栈的静态分配
TCB_t Task1TCB = {0};
StackType_t Task1Stack[TASK1_STACK_SIZE];

// 任务入口函数声明
void Task1_Entry(void *p_arg);

int main(void)
{
    // 静态创建任务1（优先级1）和任务2（优先级2）
    Task1_Handle = xTaskCreateStatic(..., (UBaseType_t)1, ...);
    Task2_Handle = xTaskCreateStatic(..., (UBaseType_t)2, ...);
    
    // 启动调度器（永不返回）
    vTaskStartScheduler();
    
    while(1); // 永远不会执行到这里
}
```

**任务入口实现**：

```c
void Task1_Entry(void *p_arg)
{
    for (;;)
    {
        flag1 = 1;
        vTaskDelay(10);   // 延时 10 个 tick
        flag1 = 0;
        vTaskDelay(10);
    }
}
```

- 通过 `vTaskDelay()` 让任务进入延时状态，让出 CPU。
- 两个任务优先级不同（1 和 2），优先级 2 的任务会抢占优先级 1 的任务。

> **提示**：`main.c` 中有一段被 `#if 0` 屏蔽的代码，展示了直接操作就绪列表插入节点的方式，用于理解底层原理。

---

### 2.2 task.c / task.h

**作用**：RTOS 核心模块，实现任务管理、调度、延时等核心功能。

#### 2.2.1 全局变量（task.c）

| 变量名                       | 类型                        | 说明                                                         |
| ---------------------------- | --------------------------- | ------------------------------------------------------------ |
| `pxReadyTasksLists`          | `List_t [configMAX_PRIORITIES]` | 就绪列表数组，每个优先级一条双向链表。                       |
| `pxCurrentTCB`               | `TCB_t *`                   | 指向当前正在运行的任务的 TCB。                               |
| `uxCurrentNumberOfTasks`     | `UBaseType_t`               | 已创建的任务总数（包括空闲任务）。                           |
| `uxTopReadyPriority`         | `UBaseType_t`               | 当前就绪任务中的最高优先级（位图编码，用于快速查找）。       |
| `xDelayedTaskLists1/2`       | `List_t`                    | 两个延时列表，用于处理系统节拍计数器溢出。                   |
| `pxDelayedTaskList`          | `List_t *`                  | 当前使用的延时列表（未溢出）。                               |
| `pxOverflowDelayedTaskList`  | `List_t *`                  | 溢出时使用的延时列表。                                       |
| `xNextTaskUnblockTime`       | `TickType_t`                | 下一个需要被唤醒的任务的解锁时刻（用于优化）。               |
| `xNumOfOverflows`            | `BaseType_t`                | 系统节拍计数器溢出的次数。                                   |

#### 2.2.2 核心函数

| 函数                               | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `prvInitialiseTaskLists()`         | 初始化所有就绪链表和延时链表，设置 `pxDelayedTaskList` 指向 `xDelayedTaskLists1`。 |
| `prvInitialiseNewTask()`           | 初始化 TCB：复制任务名、设置优先级、初始化状态链表节点、初始化任务栈（调用 `pxPortInitialiseStack`）。 |
| `prvAddNewTaskToReadyList()`       | 将新创建的任务添加到就绪列表，若当前无运行任务则设置 `pxCurrentTCB` 指向该任务，若新任务优先级更高也会切换当前任务指针。 |
| `prvAddCurrentTaskToDelayedList()` | 将当前任务从就绪列表移除，计算唤醒时间，插入到合适的延时列表（若溢出则插入 `pxOverflowDelayedTaskList`），并更新 `xNextTaskUnblockTime`。 |
| `prvResetNextTaskUnblockTime()`    | 当延时列表变化时，重新计算下一个任务的解锁时刻。             |
| `xTaskCreateStatic()`              | 用户 API：静态创建任务，内部调用 `prvInitialiseNewTask` 和 `prvAddNewTaskToReadyList`。 |
| `vTaskStartScheduler()`            | 创建空闲任务，初始化延时列表，调用 `xPortStartScheduler()` 启动硬件调度。 |
| `vTaskSwitchContext()`             | 选择下一个要运行的任务（核心调度函数）。                     |
| `vTaskDelay()`                     | 用户 API：延时当前任务（调用 `prvAddCurrentTaskToDelayedList` 后触发任务切换）。 |

#### 2.2.3 调度算法 `vTaskSwitchContext`

本工程提供了两种实现（通过 `#if 0` 切换），最终使用的是 **基于优先级位图** 的优化版本：

```c
void vTaskSwitchContext(void)
{
    UBaseType_t uxTopPriority = 0UL;
    if (uxTopReadyPriority != 0UL)
        portGET_HIGHEST_PRIORITY(uxTopPriority, uxTopReadyPriority);
    listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, &(pxReadyTasksLists[uxTopPriority]));
    uxTopReadyPriority = uxTopPriority;
}
```

- `portGET_HIGHEST_PRIORITY` 利用 `__clz` 指令（Count Leading Zeros）快速找到最高优先级位。
- `listGET_OWNER_OF_NEXT_ENTRY` 将 `pxIndex` 指向该优先级链表的下一个节点（实现时间片轮转），并取出该节点所属的 TCB。
- 该算法时间复杂度 O(1)。

#### 2.2.4 延时列表管理

当 `xTickCount` 溢出时，通过宏 `taskSWITCH_DELAYED_LISTS()` 交换 `pxDelayedTaskList` 和 `pxOverflowDelayedTaskList`，并重置 `xNextTaskUnblockTime`。这解决了 32 位 tick 计数器溢出导致的延时计算错误。

---

### 2.3 list.c / list.h

**作用**：实现通用双向循环链表，用于管理就绪任务和延时任务。

#### 2.3.1 数据结构

```c
// 链表节点（完整节点）
struct xLIST_ITEM {
    TickType_t xItemValue;    // 排序值（用于延时列表时为唤醒时间，就绪列表时不使用）
    ListItem_t *pxNext;
    ListItem_t *pxPrevious;
    void *pvOwner;            // 指向拥有该节点的对象（通常是 TCB_t*）
    void *pvContainer;        // 指向该节点所在的链表根节点（List_t*）
};

// 精简节点（仅用于链表末尾哨兵）
struct xMINI_LIST_ITEM {
    TickType_t xItemValue;
    ListItem_t *pxNext;
    ListItem_t *pxPrevious;
};

// 链表根节点
struct xLIST {
    UBaseType_t uxNumberOfItems;
    ListItem_t *pxIndex;      // 用于遍历链表的索引指针
    MiniListItem_t xListEnd;  // 哨兵节点（永远在链表末尾）
};
```

#### 2.3.2 重要操作函数

| 函数                       | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| `vListInitialise()`        | 初始化根节点：`uxNumberOfItems = 0`，`pxIndex` 指向哨兵节点，哨兵节点的 `pxNext/pxPrevious` 指向自身。 |
| `vListInitialiseItem()`    | 初始化节点：设置 `pvContainer = NULL`。                      |
| `vListInsertEnd()`         | 将节点插入链表尾部（实际是插入到 `pxIndex` 之前，因为链表是循环的）。用于就绪列表。 |
| `vListInsert()`            | 按 `xItemValue` 升序插入节点。用于延时列表。                 |
| `uxListRemove()`           | 从链表中删除指定节点，若删除的是 `pxIndex` 指向的节点，则将 `pxIndex` 指向其前驱。 |

#### 2.3.3 关键宏

- `listGET_OWNER_OF_NEXT_ENTRY(pxTCB, pxList)`：获取链表下一个节点的 owner，并将 `pxIndex` 后移（用于轮转调度）。
- `listGET_OWNER_OF_HEAD_ENTRY(pxList)`：获取链表第一个节点的 owner（不移动索引）。
- `listLIST_IS_EMPTY(pxList)`：判断链表是否为空。

---

### 2.4 port.c / portmacro.h

**作用**：硬件移植层，封装了 Cortex‑M3 的中断控制、栈初始化、上下文切换汇编代码。

#### 2.4.1 portmacro.h 中的重要宏

| 宏 / 内联函数                      | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `portYIELD()`                      | 触发 PendSV 中断：写 `ICSR` 寄存器的 PendSV 挂起位。         |
| `portDISABLE_INTERRUPTS()`         | 关中断：写 `BASEPRI` 为 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（屏蔽所有优先级 ≥ 该值的中断）。 |
| `portENABLE_INTERRUPTS()`          | 开中断：写 `BASEPRI` 为 0。                                  |
| `portENTER_CRITICAL()`             | 调用 `vPortEnterCritical()`（嵌套计数 +1，同时关中断）。     |
| `portEXIT_CRITICAL()`              | 调用 `vPortExitCritical()`（嵌套计数 -1，减到 0 时开中断）。 |
| `portSET_INTERRUPT_MASK_FROM_ISR()` | 带返回值的关中断（用于中断内临界段），返回旧的 `BASEPRI`。   |
| `portCLEAR_INTERRUPT_MASK_FROM_ISR(x)` | 恢复 `BASEPRI` 为 `x`。                                    |
| `portRECORD_READY_PRIORITY()`      | 设置优先级位图对应位为 1。                                   |
| `portRESET_READY_PRIORITY()`       | 清除优先级位图对应位为 0。                                   |
| `portGET_HIGHEST_PRIORITY()`       | 使用 `__clz` 指令计算最高优先级。                            |

#### 2.4.2 port.c 关键函数

| 函数                                 | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `pxPortInitialiseStack()`            | 初始化任务栈，构建异常返回时的上下文（模拟 CPU 自动压栈的 8 个寄存器和手动保存的 R4‑R11）。 |
| `vPortSetupTimerInterrupt()`         | 初始化 SysTick：设置重装载值，使能中断。                     |
| `xPortStartScheduler()`              | 设置 PendSV 和 SysTick 的中断优先级为最低，调用 `vPortSetupTimerInterrupt()`，然后调用 `prvStartFirstTask()`。 |
| `prvStartFirstTask()`                | 汇编函数：从向量表获取 MSP 初始值，开启全局中断，触发 SVC 异常。 |
| `vPortSVCHandler()`                  | SVC 异常处理：从 `pxCurrentTCB` 取出第一个任务的栈指针，恢复 R4‑R11，设置 PSP，开中断，异常返回进入用户任务。 |
| `xPortPendSVHandler()`               | PendSV 异常处理：保存当前任务上下文（R4‑R11）到栈中，调用 `vTaskSwitchContext()` 选择新任务，恢复新任务的上下文，返回。 |
| `xTaskIncrementTick()`               | SysTick 中断中调用：增加 `xTickCount`，处理延时任务到期，如果到期则将其从延时列表移到就绪列表，最后调用 `portYIELD()` 触发 PendSV（如果当前有更高优先级任务就绪，则立即切换）。 |
| `vPortEnterCritical()` / `vPortExitCritical()` | 临界段嵌套实现，带断言检查。                                 |

#### 2.4.3 栈初始化细节（`pxPortInitialiseStack`）

Cortex‑M3 在异常发生时自动压栈 8 个寄存器：**R0, R1, R2, R3, R12, LR (R14), PC (R15), xPSR**。`pxPortInitialiseStack` 手动构建这个压栈后的内存布局，并额外预留了 R4‑R11 的空间（这些寄存器需要手动保存/恢复）。初始栈布局如下（从高地址向低地址）：

| 偏移（相对栈顶） | 内容           | 说明                                 |
| ---------------- | -------------- | ------------------------------------ |
| +0               | R4             | 手动保存区（初始为 0）               |
| -4               | R5             |                                      |
| ...              | ...            |                                      |
| -28              | R11            |                                      |
| -32              | R0 (参数)      | 自动压栈区（异常返回时加载）         |
| -36              | R1             | 0                                    |
| -40              | R2             | 0                                    |
| -44              | R3             | 0                                    |
| -48              | R12            | 0                                    |
| -52              | LR (R14)       | `prvTaskExitError` 地址（防止任务返回） |
| -56              | PC (R15)       | 任务入口函数地址（必须为 Thumb 模式） |
| -60              | xPSR           | 0x01000000（Thumb 状态，ISP=1）      |

最终 `pxTopOfStack` 指向 R4 的位置（最低地址）。当第一次切换到这个任务时，PendSV 会手动恢复 R4‑R11，然后异常返回时 CPU 自动恢复其余的寄存器，从而跳转到任务入口。

---

### 2.5 rtos.h / rtos_config.h / projectdefs.h

- **rtos.h**：定义任务控制块结构 `TCB_t`，包含栈顶指针、状态链表节点、栈底指针、任务名、优先级等。
- **rtos_config.h**：系统配置参数（时钟频率、节拍频率、优先级数量、栈大小等），用户可修改。
- **projectdefs.h**：定义 `TaskFunction_t` 类型、`pdTRUE`/`pdFALSE` 等基础宏。

---

## 3. 核心数据结构详解

### 3.1 链表节点 (ListItem_t)

- `xItemValue`：用于排序（在延时列表中存放唤醒时刻的 tick 值）。
- `pvOwner`：指向拥有此节点的 TCB。
- `pvContainer`：指向此节点所属的 `List_t`（用于快速判断节点是否已插入链表）。

### 3.2 链表根节点 (List_t)

- `uxNumberOfItems`：节点数量。
- `pxIndex`：遍历索引，在就绪列表中用于实现时间片轮转。
- `xListEnd`：哨兵节点，其 `xItemValue` 初始化为 `portMAX_DELAY`（最大值），保证永远在链表末尾。

### 3.3 任务控制块 (TCB_t)

```c
struct taskTaskControlBlock {
    volatile StackType_t *pxTopOfStack;   // 当前栈顶（随任务切换变化）
    ListItem_t xStateListItem;            // 挂入就绪或延时列表的节点
    StackType_t *pxStack;                 // 栈起始地址（静态分配时由用户提供）
    char pcTaskName[configMAX_TASK_NAME_LEN];
    TickType_t xTicksToDelay;             // 保留字段（未使用，可用作剩余延时）
    UBaseType_t uxPriority;
};
```

> **注意**：`xTicksToDelay` 在此实现中并未真正使用，延时通过将任务节点插入延时列表实现，而非轮询递减该变量。该字段可能是旧设计遗留，可忽略。

### 3.4 就绪列表与优先级位图

- `pxReadyTasksLists` 数组：索引为优先级，值为链表根节点。
- `uxTopReadyPriority`：32 位无符号整数，每一位代表一个优先级是否有就绪任务（bit0 对应优先级 0，bit31 对应优先级 31）。
- 宏 `portRECORD_READY_PRIORITY` 设置对应位，`portRESET_READY_PRIORITY` 清除对应位（仅在对应优先级链表变为空时清除）。
- 查找最高优先级：`__clz(uxTopReadyPriority)` 返回前导零个数，最高优先级 = 31 - __clz。

### 3.5 延时列表与系统节拍

- 两个延时列表 `xDelayedTaskLists1/2` 解决 `xTickCount` 32 位溢出问题。
- 当 `xTickCount` 从 0xFFFFFFFF 加 1 变成 0 时，`taskSWITCH_DELAYED_LISTS` 宏交换当前延时列表和溢出列表。
- 延时任务按照 `xItemValue`（唤醒时刻）升序插入，系统节拍中断中从列表头部开始检查，将 `xTickCount >= 唤醒时刻` 的任务移到就绪列表。

---

## 4. 上电工作流程（完整）

### 4.1 从 main() 到创建任务

1. 系统复位 → 启动文件调用 `main()`。
2. `main()` 中：
   - 分配两个任务的 TCB 和栈（全局变量，位于 .bss 段，初始为零）。
   - 调用 `xTaskCreateStatic` 创建 Task1（优先级 1）和 Task2（优先级 2）。
   - 内部：`prvInitialiseNewTask` 初始化 TCB 和栈，`prvAddNewTaskToReadyList` 将任务插入就绪列表并更新优先级位图。
3. 调用 `vTaskStartScheduler()`。

### 4.2 启动调度器 vTaskStartScheduler()

- 调用 `vApplicationGetIdleTaskMemory` 获取空闲任务的 TCB 和栈（全局变量 `IdleTaskTCB` 和 `IdleTaskStack`）。
- 调用 `xTaskCreateStatic` 创建空闲任务（优先级 0）。
- 设置 `xNextTaskUnblockTime = portMAX_DELAY`。
- 调用 `xPortStartScheduler()`。

### 4.3 硬件初始化 xPortStartScheduler()

- 设置 PendSV 和 SysTick 的中断优先级为最低（`configKERNEL_INTERRUPT_PRIORITY`）。
- 设置 `uxCriticalNesting = 0`。
- 调用 `vPortSetupTimerInterrupt()` 初始化 SysTick（重装载值 = CPU 频率 / 节拍频率，使能中断）。
- 调用 `prvStartFirstTask()` 汇编函数，**不再返回**。

### 4.4 启动第一个任务 (SVC 异常)

`prvStartFirstTask` 汇编代码解析：

```assembly
ldr r0, =0xE000ED08   ; SCB_VTOR 寄存器地址（向量表偏移）
ldr r0, [r0]          ; 获取向量表起始地址（通常是 0x00000000）
ldr r0, [r0]          ; 获取向量表第一个字（初始 MSP 值）
msr msp, r0           ; 设置主堆栈指针
cpsie i               ; 使能全局中断（清除 PRIMASK）
cpsie f               ; 使能 fault 中断（清除 FAULTMASK）
dsb / isb             ; 指令同步
svc 0                 ; 触发 SVC 异常
```

SVC 处理函数 `vPortSVCHandler`：

```assembly
ldr r3, =pxCurrentTCB
ldr r1, [r3]          ; r1 = pxCurrentTCB
ldr r0, [r1]          ; r0 = pxTopOfStack (第一个任务的栈顶)
ldmia r0!, {r4-r11}   ; 恢复 R4~R11
msr psp, r0           ; 设置 PSP 为用户任务栈
mov r0, #0
msr basepri, r0       ; 开中断（BASEPRI=0）
orr r14, #0x0D        ; 设置 EXC_RETURN 值（返回后使用 PSP，Thumb 模式）
bx r14                ; 异常返回，CPU 自动从 PSP 恢复剩余寄存器并跳转到任务入口
```

至此，第一个任务（优先级最高的就绪任务）开始运行。

### 4.5 系统节拍与任务切换 (PendSV)

- **SysTick 中断**：每 `1/configTICK_RATE_HZ` 秒触发一次，调用 `xPortSysTickHandler` → `xTaskIncrementTick`：
  - `xTickCount++`，若溢出则交换延时列表。
  - 检查 `xNextTaskUnblockTime`，若有任务到期，则将其从延时列表移到就绪列表，更新优先级位图。
  - 最后调用 `portYIELD()` 触发 PendSV。
- **PendSV 中断**：`xPortPendSVHandler` 实现上下文切换：
  ```assembly
  mrs r0, psp               ; 获取当前任务的 PSP
  ldr r3, =pxCurrentTCB
  ldr r2, [r3]
  stmdb r0!, {r4-r11}       ; 保存 R4~R11 到当前任务栈
  str r0, [r2]              ; 更新 pxTopOfStack
  stmdb sp!, {r3, r14}      ; 保存 R3(TCB地址) 和 LR 到主堆栈
  mov r0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
  msr basepri, r0           ; 关中断（保护调度器临界区）
  bl vTaskSwitchContext     ; 选择下一个任务
  mov r0, #0
  msr basepri, r0           ; 开中断
  ldmia sp!, {r3, r14}      ; 恢复 R3, LR
  ldr r1, [r3]              ; 新的 pxCurrentTCB
  ldr r0, [r1]              ; 新任务的 pxTopOfStack
  ldmia r0!, {r4-r11}       ; 恢复新任务的 R4~R11
  msr psp, r0               ; 更新 PSP
  bx r14                    ; 异常返回，切换到新任务
  ```

- **任务主动让出**：调用 `taskYIELD()` 直接写 PendSV 挂起位，同样触发 PendSV 中断。

---

## 5. API 详细说明与使用示例

### 5.1 静态创建任务 xTaskCreateStatic

```c
TaskHandle_t xTaskCreateStatic( TaskFuntion_t pxTaskCode,
                                const char * const pcName,
                                const uint32_t ulStackDepth,
                                void * const pvParameters,
                                UBaseType_t uxPriority,
                                StackType_t * const puxStackBuffer,
                                TCB_t * const pxTaskBuffer );
```

**参数**：
- `pxTaskCode`：任务函数指针，原型为 `void func(void *param)`。
- `pcName`：任务名称，长度不超过 `configMAX_TASK_NAME_LEN`。
- `ulStackDepth`：栈深度（**字**为单位），例如 128 表示 128 * 4 = 512 字节。
- `pvParameters`：传递给任务函数的参数。
- `uxPriority`：优先级（0 最低，`configMAX_PRIORITIES-1` 最高）。
- `puxStackBuffer`：用户分配的栈数组（类型 `StackType_t[]`）。
- `pxTaskBuffer`：用户分配的 TCB 结构体变量地址。

**返回值**：任务句柄（即 TCB 指针），可用于后续 API（本工程未使用）。

**示例**：见 `main.c`。

### 5.2 任务延时 vTaskDelay

```c
void vTaskDelay(const TickType_t xTicksToDelay);
```

- 使当前任务进入阻塞态，持续 `xTicksToDelay` 个系统节拍。
- 延时到期后，任务自动回到就绪态。
- 若参数为 0，则不会阻塞，仅触发一次任务调度（等价于 `taskYIELD()`）。

### 5.3 主动让出 CPU taskYIELD

```c
taskYIELD();
```

- 立即触发 PendSV 中断，重新调度下一个就绪任务。
- 当前任务如果仍然是最高优先级的就绪任务，则可能马上又被选中。

### 5.4 临界段保护

#### 5.4.1 任务内使用（不可嵌套于中断）

```c
taskENTER_CRITICAL();
// 访问共享资源
taskEXIT_CRITICAL();
```

- 内部调用 `portENTER_CRITICAL()`，通过 `uxCriticalNesting` 计数，支持嵌套（但不可在中断中调用）。
- 关中断屏蔽所有优先级 ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的中断。

#### 5.4.2 中断内使用（可嵌套）

```c
UBaseType_t uxSaved;
uxSaved = taskENTER_CRITICAL_FROM_ISR();
// 访问共享资源
taskEXIT_CRITICAL_FROM_ISR(uxSaved);
```

- 保存当前 `BASEPRI` 值，然后写 `BASEPRI` 关中断。
- 退出时恢复原来的 `BASEPRI`，安全嵌套。

---

## 6. 配置选项详解 (rtos_config.h)

| 宏定义                                     | 范围/说明                                    | 示例值        |
| ------------------------------------------ | -------------------------------------------- | ------------- |
| `configCPU_CLOCK_HZ`                       | CPU 主频（Hz），用于 SysTick 重装载计算      | 72000000      |
| `configTICK_RATE_HZ`                       | 系统节拍频率（Hz），典型 100 或 1000         | 100           |
| `configMINIMAL_STACK_SIZE`                 | 空闲任务栈大小（字）                         | 128           |
| `configUSE_16_BIT_TICKS`                   | 设为 0 使用 32 位 `TickType_t`               | 0             |
| `configMAX_TASK_NAME_LEN`                  | 任务名最大长度（含 '\0'）                    | 16            |
| `configSUPPORT_STATIC_ALLOCATION`          | 必须为 1（静态创建）                         | 1             |
| `configMAX_PRIORITIES`                     | 最大优先级数量（1~32）                       | 3（可改为 32）|
| `configMAX_SYSCALL_INTERRUPT_PRIORITY`     | 临界段屏蔽的中断优先级阈值（数值越大优先级越低）| 191（0xBF）  |
| `configPRIO_BITS`                          | Cortex‑M3 优先级位宽（4 位）                 | 4             |
| `configLIBRARY_LOWEST_INTERRUPT_PRIORITY`  | 最低优先级数值（15）                         | 15            |
| `configKERNEL_INTERRUPT_PRIORITY`          | 内核中断（PendSV/SysTick）优先级，自动计算   | 15 << 4 = 240|
| `configUSE_PORT_OPTIMISED_TASK_SELECTION`  | 使用 CLZ 优化，设为 1                        | 1             |

> **优先级数值说明**：Cortex‑M3 中，优先级寄存器高 4 位有效，数值越大优先级越低。例如 `configMAX_SYSCALL_INTERRUPT_PRIORITY = 191`（0xBF），其高 4 位为 0xB（11），所以所有优先级 ≥ 11 的中断都会被屏蔽。用户中断应设为 12~15，以保证可被 RTOS 临界段屏蔽。

---

## 7. 如何自定义添加或删改功能

### 7.1 添加新用户任务

1. 在 `main.c` 或单独模块中定义 TCB 和栈：
   ```c
   TCB_t MyTaskTCB;
   StackType_t MyTaskStack[256];  // 根据需要调整大小
   ```
2. 编写任务函数：
   ```c
   void MyTask(void *p_arg)
   {
       for (;;) {
           // 用户代码
           vTaskDelay(pdMS_TO_TICKS(100)); // 若需毫秒宏，可自行定义
       }
   }
   ```
3. 在 `main()` 中创建：
   ```c
   xTaskCreateStatic(MyTask, "MyTask", 256, NULL, 3, MyTaskStack, &MyTaskTCB);
   ```
4. 确保优先级不超过 `configMAX_PRIORITIES - 1`。

### 7.2 修改优先级数量

- 编辑 `rtos_config.h`，将 `configMAX_PRIORITIES` 改为所需值（≤32）。
- 无需修改其他代码，就绪列表数组大小会自动调整。

### 7.3 改变系统节拍频率

- 修改 `configTICK_RATE_HZ`，例如改为 1000（1ms 中断）。
- **注意**：`vTaskDelay` 的参数含义改变，原来 `vTaskDelay(10)` 延时 100ms，现在变成 10ms。

### 7.4 增加动态任务创建 (xTaskCreate)

当前工程仅支持静态分配。若要支持动态创建：

1. 实现内存分配函数 `void *pvPortMalloc(size_t size)` 和 `void vPortFree(void *ptr)`（可简单使用 `malloc/free` 或自定义堆管理）。
2. 在 `task.c` 中添加 `xTaskCreate` 函数：
   ```c
   TaskHandle_t xTaskCreate(TaskFunction_t pxTaskCode, const char *pcName,
                            uint32_t ulStackDepth, void *pvParameters,
                            UBaseType_t uxPriority, TaskHandle_t *pxCreatedTask)
   {
       TCB_t *pxNewTCB = (TCB_t *)pvPortMalloc(sizeof(TCB_t));
       StackType_t *pxStack = (StackType_t *)pvPortMalloc(ulStackDepth * sizeof(StackType_t));
       if (pxNewTCB && pxStack) {
           // 调用 prvInitialiseNewTask 和 prvAddNewTaskToReadyList
           // 设置 pxNewTCB->pxStack = pxStack
       }
       // 错误处理
   }
   ```
3. 同时需要实现 `vTaskDelete` 来释放内存。

### 7.5 添加空闲任务钩子

在 `task.c` 的 `prvIdleTask` 函数中添加弱引用函数调用：

```c
void __attribute__((weak)) vApplicationIdleHook(void) {}

void prvIdleTask(void *p_arg)
{
    for (;;) {
        vApplicationIdleHook();
        taskYIELD();
    }
}
```

用户可以在自己的代码中定义 `vApplicationIdleHook` 实现低功耗或后台处理。

---

## 8. 注意事项与调试建议

### 8.1 栈大小估算

- 每个任务的栈使用量取决于局部变量、函数调用深度和中断嵌套。可使用调试器观察任务栈的 watermark（初始填充为 0，运行时查看最高地址的更改情况）。
- 建议先分配较大值（如 256 字），稳定后逐步减小。

### 8.2 临界段使用原则

- 临界段内代码应尽量短，避免长时间关中断导致中断丢失。
- 不要在临界段内调用任何可能引起任务切换的函数（如 `vTaskDelay`、`taskYIELD`），因为关中断时无法触发 PendSV，会导致系统死锁。
- 中断服务函数中如果访问了与任务共享的变量，必须使用 `FROM_ISR` 版本的临界段。

### 8.3 调试方法

- **模拟器**：可在 QEMU 的 STM32 虚拟板上运行，观察串口输出。
- **硬件调试**：在 `vTaskSwitchContext`、`xPortPendSVHandler` 等处设置断点，观察 `pxCurrentTCB` 的变化。
- **宏开关**：`rtos_config.h` 中定义 `DEBUG___` 会启用简单的调度器实现（非 CLZ 版本），便于单步跟踪。

### 8.4 已知限制

- 不支持任务删除（需自行添加）。
- 不支持优先级继承（未实现互斥量）。
- 未实现消息队列、信号量等同步机制（可基于临界段和标志变量简单实现）。
- 同优先级任务按时间片轮转，但时间片长度固定为一个 tick（即每个 tick 都会触发 PendSV 切换同优先级任务）。

### 8.5 扩展建议

- **添加软件定时器**：可复用延时列表，创建定时器任务。
- **支持动态优先级变更**：实现 `vTaskPrioritySet`，修改 TCB 的优先级并重新插入就绪列表。
- **支持挂起/恢复**：添加 `vTaskSuspend` 和 `vTaskResume`，引入挂起列表。

---

## 9. 开发调试与问题修复记录

在编写和调试该RTOS内核的过程中，遇到并解决了以下三个典型问题：

### 9.1 重复链表插入导致节点指针破坏

- **现象**：系统启动后，`xTaskIncrementTick` 卡死在 `for(;;)` 循环中，`xTickCount` 不再更新。
- **定位**：通过调试发现，`listGET_OWNER_OF_HEAD_ENTRY(pxDelayedTaskList)` 返回的 `pxTCB` 指针为无效地址，后续链表操作（`uxListRemove`, `vListInsert`）无法正确更新节点计数，导致延时列表永远非空。
- **原因**：`main.c` 中在调用 `xTaskCreateStatic` 创建任务后，又手动执行了 `vListInsertEnd(&(pxReadyTasksLists[1]), ...)` 将同一任务的 `xStateListItem` 再次插入就绪列表。静态创建函数内部已经通过 `prvAddNewTaskToReadyList` 完成了插入，重复插入破坏了节点的 `pxNext/pxPrevious` 指针，进而污染了整个链表结构。
- **解决**：删除 `main.c` 中所有手动插入就绪列表的代码（已用 `#if 0` 屏蔽），只保留 `xTaskCreateStatic` 内的插入操作。

### 9.2 优先级位图溢出（`__clz(0)` 导致负数）

- **现象**：当所有任务都进入延时状态、就绪列表为空时，系统触发 HardFault。
- **定位**：查看 `vTaskSwitchContext` 中的调度代码：
  ```c
  UBaseType_t uxTopPriority = 31 - __clz(uxTopReadyPriority);
  ```
  当 `uxTopReadyPriority == 0` 时，`__clz(0)` 返回 32，计算得 `uxTopPriority = -1`，导致后续数组访问越界。
- **原因**：未判断 `uxTopReadyPriority` 是否为 0 就直接调用 `__clz`。
- **解决**：在调用前增加条件判断：
  ```c
  if (uxTopReadyPriority != 0UL) {
      portGET_HIGHEST_PRIORITY(uxTopPriority, uxTopReadyPriority);
  }
  ```
  同时确保在无就绪任务时不会执行调度（实际调度器会确保空闲任务始终就绪，此修复作为防御性编程）。

### 9.3 SysTick 中断入口名与向量表不匹配

- **现象**：调用 `vTaskDelay` 后任务永远无法被唤醒，系统看起来“卡死”。
- **定位**：检查发现 `port.c` 中定义了 `xPortSysTickHandler` 函数，但启动文件的向量表期望的中断处理函数名为 `SysTick_Handler`。由于名称不匹配，SysTick 中断发生时并未执行 `xTaskIncrementTick`，导致 `xTickCount` 始终为 0，延时列表中的任务永远不会被移回就绪列表。
- **原因**：移植时未正确关联中断处理函数。
- **解决**：在 `port.c` 中使用预处理器宏将 `xPortSysTickHandler` 重定义为 `SysTick_Handler`：
  ```c
  #define xPortSysTickHandler SysTick_Handler
  ```
  同时确保 `SysTick_Handler` 未被其他模块重复定义。

以上问题的修复使内核稳定运行，并通过多任务并发测试。


## 总结

本工程实现了一个简洁但功能完整的 RTOS 内核，展示了任务管理、优先级调度、延时、临界段保护等核心概念，代码量适中且易于移植。通过阅读本文档和源码，读者可以深入理解 RTOS 的工作原理，并在此基础上进行功能扩展或应用于实际项目。

**文件依赖关系**：
```
main.c
  ├── task.c (提供 xTaskCreateStatic, vTaskStartScheduler, vTaskDelay)
  │     ├── list.c (链表操作)
  │     └── port.c (硬件相关)
  └── port.c (提供 xPortStartScheduler)
        └── portmacro.h (宏和内联函数)
```

所有模块均通过 `rtos_config.h` 进行配置，无需修改核心代码即可调整系统参数。