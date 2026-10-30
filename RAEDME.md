已按最小改动修订如下（仅替换错误处，其余原文不变）：

# 总结

# 1 关键数据结构和api

## 数据结构

TCB: 链表节点嵌入到tcb中

```c
typedef struct taskTaskControlBlock TCB_t;
struct taskTaskControlBlock
{
    // 栈顶
    volatile StackType_t *pxTopOfStack;
    // 任务节点
    ListItem_t xStateListItem;
    // 任务栈起始地址
    StackType_t *pxStack;
    // 任务名称, 字符串
    char pcTaskName[configMAX_TASK_NAME_LEN];
    // 延时, 等于0时延时结束
    TickType_t xTicksToDelay;
    // 优先级, 数字越大优先级越高
    UBaseType_t uxPriority;
};
```

TaskHandle_t 是指向任务控制块 TCB 的句柄（本质是 TCB_t *）；任务执行函数的类型是 TaskFuntion_t

双向链表

## api

vTaskStartScheduler

xTaskCreateStatic -> 可与动态创建任务的api对比

vTaskDelay

# 2 启动流程(静态创建任务的版本)

## 首先创建静态任务

### xTaskCreateStatic

将静态的栈buf注册到tcb中

将栈顶指针先移到数组最高位(栈是由上至下生长的, 因此SP初始应该是最高位置), 按照8字节对齐(AAPCS要求函数调用边界SP 8字节对齐, pxTopOfStack &= ~0x7), 这个操作使得pxTopOfStack向低位置移动, 因此不会越界到数组外

将任务名注册到tcb中(有必要的话限制字符串长度)

初始化TCB中的链表节点并让节点中表示owner的指针指向TCB

将优先级uxPriority注册到tcb中

初始化任务栈, 根据前面的pxTopOfStack和任务函数入口以及任务形式参数, 构建异常上下文: 异常(中断,系统调用,任务切换)发生时, cpu自动从该任务栈中弹出(或者压入)数据, 因此先规划这些数据, 手动将xPSR初始值(0x01000000L)放入栈顶, R15(PC, 即当前任务入口), R14(LR) 任务的异常返回地址(正常情况下任务是不会返回异常地址的), 紧接着写入prvTaskExitError作为LR的地址, 然后pxTopOfStack -= 5跨过R12, R3, R2, R1四个槽位（这4个槽位未显式初始化，首次运行时值不重要）, 然后再pxTopOfStack -= 8保留8个软件弹栈的位置, 最后返回当前的栈指针值, 并存入pxTopOfStack(更新这个值)

让任务句柄指向任务控制块

添加新任务到任务就绪列表: 进入临界区, 增加uxCurrentNumberOfTasks全局变量计数, 若全局指针pxCurrentTCB为空, 将新的任务tcb的地址给它, 并且若uxCurrentNumberOfTasks == 1则调用prvInitialiseTaskLists()循环初始化所有优先级的就绪链表(同时初始化两个延时链表), 否则比较pxCurrentTCB与新的任务的优先级大小, 新任务优先级不低于当前任务时被赋值给pxCurrentTCB, 然后将新的任务添加到就绪链表(在此过程中, 使用位运算将当前的优先级标记到全局标量uxTopReadyPriority(uint32_t)的某一位上, 这个变量将来会被31 - __clz提取最高优先级索引, 用来查找最高优先级), 退出临界区

返回任务句柄（成功时为TCB指针，非NULL；失败为NULL）

此时任务栈的结构:

```text
        高地址
        ┌──────────────────────────┐  ← pxStack + ulStackDepth（数组末尾）
        │  对齐余量（最多 7 字节）   │     不参与使用
        ├──────────────────────────┤  ← 8 字节对齐后的初始 SP
        │  xPSR  = 0x01000000      │ ┐
        ├──────────────────────────┤ │
        │  PC    = 任务入口地址      │ │  硬件自动弹栈区
        ├──────────────────────────┤ │  （异常返回时由 CPU 自动弹出）
        │  LR    = prvTaskExitError │ │  共 8 个字
        ├──────────────────────────┤ │
        │  R12   = 0               │ │
        ├──────────────────────────┤ │
        │  R3    = 0               │ │
        ├──────────────────────────┤ │
        │  R2    = 0               │ │
        ├──────────────────────────┤ │
        │  R1    = 0               │ │
        ├──────────────────────────┤ │
        │  R0    = pvParameters    │ ┘
        ├──────────────────────────┤  ← pxTopOfStack 初始值指向这里（R4）
        │  R11   = 未定义/0         │ ┐
        ├──────────────────────────┤ │
        │  R10   = 未定义/0         │ │  软件手动弹栈区
        ├──────────────────────────┤ │  （SVC/PendSV 中 ldmia 弹出）
        │  R9    = 未定义/0         │ │  共 8 个字
        ├──────────────────────────┤ │
        │  R8    = 未定义/0         │ │
        ├──────────────────────────┤ │
        │  R7    = 未定义/0         │ │
        ├──────────────────────────┤ │
        │  R6    = 未定义/0         │ │
        ├──────────────────────────┤ │
        │  R5    = 未定义/0         │ │
        ├──────────────────────────┤ │
        │  R4    = 未定义/0         │ ┘
        ├──────────────────────────┤
        │                          │
        │      栈的剩余空间          │
        │      （任务运行后向下用）    │
        │                          │
        └──────────────────────────┘  ← pxStack（数组起始，最低地址）
        低地址
```

## 然后开启调度器

### vTaskStartScheduler

首先静态创建闲时任务idletask

设置全局变量xNextTaskUnblockTime = portMAX_DELAY和xTickCount = 0U

进入xPortStartScheduler: 将systick和pendsv设置为优先级15(最低优先级, 优先保证非rtos调度的中断发生), 将中断嵌套计数器(全局变量)uxCriticalNesting设置为0, 然后初始化systick\
启动第一个任务prvStartFirstTask: 使用SCB_VTOR寄存器的值寻找向量表起始地址, 使用该起始地址处存放的值设置主堆栈指针msp的值, 调用svc中断启动第一个任务

vPortSVCHandler, svc中断句柄: 取pxCurrentTCB中的pxTopOfStack, 以这个值为基础(这个值先放入r0), 软件弹栈8个寄存器的值(r4~r11), 此时r0也会跟着**增加**8个字（32字节）, 然后将r0装入psp(psp堆栈指针用于执行任务函数), 此时将basepri设置为0(BASEPRI屏蔽的是优先级数值大于等于BASEPRI的中断；写0表示关闭BASEPRI屏蔽，即开放所有中断), 设置r14的值构造EXC_RETURN，让svc退出后回到Thread模式、使用psp、保持Thumb状态（Cortex-M只有Thumb状态，没有ARM状态）, bx r14返回, 此时cpu根据psp的值自动将栈中剩下的内容弹出cpu寄存器中(r0~r3, r12, r14, r15(PC), xPSR)

然后程序就恢复了pxCurrentTCB指向的任务的上下文, 调度器启动后正常运行的情况下不再返回

此时任务栈的结构: 

```text
        ┌──────────────────────────┐
        │  xPSR                    │ ┐
        ├──────────────────────────┤ │
        │  PC                      │ │  ② 硬件自动弹栈
        ├──────────────────────────┤ │  （bx r14 触发异常返回）
        │  LR                      │ │
        ├──────────────────────────┤ │
        │  R12                     │ │
        ├──────────────────────────┤ │
        │  R3                      │ │
        ├──────────────────────────┤ │
        │  R2                      │ │
        ├──────────────────────────┤ │
        │  R1                      │ │
        ├──────────────────────────┤ │
        │  R0                      │ ┘  ← 弹完后 PSP 指向这里
        ├──────────────────────────┤
        │  R11                     │ ┐
        ├──────────────────────────┤ │
        │  R10                     │ │  ① 软件手动弹栈
        ├──────────────────────────┤ │  （ldmia r0!, {r4-r11}）
        │  R9                      │ │
        ├──────────────────────────┤ │
        │  R8                      │ │
        ├──────────────────────────┤ │
        │  R7                      │ │
        ├──────────────────────────┤ │
        │  R6                      │ │
        ├──────────────────────────┤ │
        │  R5                      │ │
        ├──────────────────────────┤ │
        │  R4                      │ ┘  ← pxTopOfStack 原来指向这里
        ├──────────────────────────┤
        │  栈剩余空间                │
        └──────────────────────────┘
```

## 切换上下文(vtaskdelay->yield或者systick中断->yield)

### vTaskDelay

这时还在任务执行过程中, 因此psp就是该任务的堆栈指针

先将任务插入到延时列表prvAddCurrentTaskToDelayedList: 首先将任务从就绪列表移出；若移除后该链表为空，就清除uxTopReadyPriority中对应的优先级位(每个就绪链表都代表一个优先级, 不同优先级任务挂在不同的就绪链表中)。随后无论链表是否为空，都根据任务延时计算到期时间, 更新tcb中链表节点的xItemValue用于节点排序, 如果到期时间加法计算溢出, 就将当前任务插入到溢出延时链表, 否则插入到普通延时链表。插入链表的过程是升序的插入, 因此每一个systick到来, 都会检索到延时链表中第一个节点代表的任务, 若到期时间小于xNextTaskUnblockTime(全局变量), 就将xNextTaskUnblockTime更新为到期时间, 保证最接近的任务能及时恢复运行

然后执行yield, 触发pendsv: cpu先自动压栈xPSR, r15(PC), r14, r12, r3~r0；PendSV中先读PSP到r0，再手动压栈r4-r11, 此时r0指向任务栈中代表r4的位置（PSP仍指向硬件压栈后的R0位置）, 然后根据pxCurrentTCB指向的地址获取当前任务的pxTopOfStack, 将r0存入pxTopOfStack

根据主堆栈指针msp压栈保护r3(此时存储pxCurrentTCB的地址), r14

```md
By DeepSeek:

这两条指令是配套的：

asm
stmdb sp!, {r3, r14}   ; 调用 C 函数前，把 R3、R14 压到 MSP 保护
...
bl vTaskSwitchContext  ; 调用 C 函数，会破坏 caller-saved 寄存器
...
ldmia sp!, {r3, r14}   ; 调用后恢复 R3、R14
核心原因一句话：vTaskSwitchContext() 是一个 C 函数，按 AAPCS 它可以随意破坏 R0–R3、R12 和 LR；而 PendSV 在调用它之后还要用 R3 和 R14，所以必须先自己保存。

一、先看调用 C 函数前，R3 和 R14 里是什么
asm
ldr r3, =pxCurrentTCB   ; R3 = &pxCurrentTCB（全局变量地址）
ldr r2, [r3]            ; R2 = pxCurrentTCB（当前 TCB 指针）
...
str r0, [r2]            ; 把当前任务栈顶存进 TCB
...
stmdb sp!, {r3, r14}    ; 此时 R3 和 R14 都很重要
bl vTaskSwitchContext
R3：保存的是 pxCurrentTCB 这个全局变量的地址。调用完 C 函数后，pxCurrentTCB 已经被更新成新任务，但 R3 还要用来重新读它：

asm
ldr r1, [r3]   ; 恢复后用 R3 读 pxCurrentTCB 的新值
ldr r0, [r1]   ; 得到新任务的 pxTopOfStack
R14（LR）：异常进入 PendSV 时，CPU 自动把 EXC_RETURN（如 0xFFFFFFFD）写进 LR。最后 bx r14 就是靠这个值告诉硬件“从异常返回、用 PSP、返回 Thread 模式”。但 bl vTaskSwitchContext 会覆盖 LR，把它变成函数的返回地址。不保存的话，最后 bx r14 就回不去了。

二、为什么必须手动保存，不能指望 C 函数保留
ARM AAPCS 规定：

寄存器	调用约定	C 函数是否保证保留
R0–R3	caller-saved（参数/返回值/临时）	不保证，可随意改写
R12	caller-saved	不保证
LR	caller-saved	不保证
R4–R11	callee-saved	保证保留
R3 和 LR 都属于 caller-saved，所以调用者（这里是 PendSV 汇编）如果调用后还要用，就必须自己压栈保护。stmdb sp!, {r3, r14} 就是干这个的。

三、为什么压到 MSP，而不是 PSP/任务栈
这一点很关键，和你之前看的栈布局直接相关。

此时 PSP 已经保存好了当前任务的完整上下文：

asm
mrs r0, psp
stmdb r0!, {r4-r11}   ; 软件压 R4~R11 到任务栈
str r0, [r2]          ; pxTopOfStack = 指向 R4
到这一步，当前任务的栈帧已经“封口”了，pxTopOfStack 指向 R4。不能再往 PSP 上压任何东西，否则会打乱任务栈布局，下次恢复时 ldmia r0!, {r4-r11} 读到的就不是 R4~R11 了。

而 stmdb sp!, ... 用的是 SP = MSP。PendSV 是异常，异常处理用主堆栈 MSP，这是独立的栈，不碰任务栈。临时保存 R3、R14 放在 MSP 上，用完就弹掉，任务栈完全不受影响。

用图表示：

text
MSP（主堆栈，异常用）              PSP（任务栈）
┌──────────────────┐              ┌──────────────────┐
│ ...              │              │ xPSR             │
├──────────────────┤              │ PC               │
│ R14 (EXC_RETURN) │ ← 临时保护    │ LR               │
├──────────────────┤              │ R12              │
│ R3  (&pxCurrentTCB)│            │ R3               │
├──────────────────┤              │ R2               │
│ ...              │              │ R1               │
└──────────────────┘              │ R0               │
                                  ├──────────────────┤
                                  │ R11 ... R4       │ ← pxTopOfStack
                                  ├──────────────────┤
                                  │ 剩余栈空间        │
                                  └──────────────────┘
四、为什么不用 R4–R11 来保存
R4–R11 是 callee-saved，C 函数保证不破坏，理论上可以用 R4 存 &pxCurrentTCB、用 R5 存 EXC_RETURN。但 FreeRTOS 没这么做，原因：

R4–R11 在任务上下文里已经被 stmdb r0!, {r4-r11} 保存过了，handler 里再用它们，语义上容易和“任务上下文”混淆。

用 MSP 压栈最简单：stmdb sp!, {r3, r14} / ldmia sp!, {r3, r14} 一对指令，不需要额外分析哪些寄存器空闲。

R3、R14 本来就是 caller-saved，压栈保护是标准做法，符合 AAPCS 的 caller 责任。
```

关中断, 进入临界段, 使用vTaskSwitchContext进行任务切换, 将当前任务切换为下一个任务(选择最高优先级的任务taskSELECT_HIGHEST_PRIORITY_TASK()): 在uxTopReadyPriority不为0的情况下使用31 - __clz(uxTopReadyPriority)获取最高置位位索引(即最高优先级), 在对应优先级的链表中通过listGET_OWNER_OF_NEXT_ENTRY轮转取节点对应的tcb, 并将pxCurrentTCB更新为这个tcb, 根据msp弹出保护的r3, r14, 然后根据r3存放的pxCurrentTCB的地址找到对应tcb的psp, 根据这个psp弹栈即可恢复新任务的上下文

## systick触发的上下文切换

将到期任务移出延时列表

然后用portyield触发pendsv

# 3 几个疑问

## 为什么要使用pendsv做切换, 为什么要使用svc启动第一个任务

- PendSV：可挂起、优先级设为最低，保证所有中断先处理完；在异常上下文里
  安全保存/恢复上下文，避免在中断嵌套中间切换。portYIELD() 只是置挂起位，
  真正切换发生在所有更高优先级异常之后。
- SVC：同步异常，由 svc 指令主动触发。启动调度器时还在 Thread+MSP，
  需要借 SVC 进入 Handler 模式，设置 PSP、构造 EXC_RETURN，再 bx r14
  触发异常返回，硬件自动恢复任务上下文并切换到 PSP/Thread 模式。
  只有“异常返回”机制能一次性完成换栈、换模式、恢复寄存器、跳任务入口。

## 延时列表和溢出延时列表怎么切换(多次溢出)

- 两个列表：pxDelayedTaskList 放未溢出到期任务，pxOverflowDelayedTaskList
  放 xTimeToWake < xConstTickCount 的溢出任务。
- xTickCount 回绕到 0 时调用 taskSWITCH_DELAYED_LISTS()：
  交换两个列表指针、xNumOfOverflows++、prvResetNextTaskUnblockTime()。
- 多次溢出就是每次回绕交换一次；同一时刻只有一个“当前延时列表”，
  两个列表足够，因为单个延时不会跨越两个完整回绕周期。

## msp和psp

- MSP：主堆栈，Handler 模式（中断/异常）永远用 MSP；复位默认 Thread 也用 MSP。
- PSP：进程堆栈，任务代码在 Thread 模式用 PSP（CONTROL.SPSEL=1）。
- 任务切换只换 PSP；PendSV 里 stmdb sp!,... 压的是 MSP，不碰任务栈。
- 好处：任务栈隔离，中断栈独立，切换简单。

## arm和thumb模式

- Cortex-M 只有 Thumb 状态，没有 ARM 状态；xPSR.T 必须为 1。
- orr r14, #0xd 构造 EXC_RETURN（0xFFFFFFFD），表示异常返回后
  进入 Thread 模式、使用 PSP；不是切换 ARM/Thumb。
- 向量表要求异常向量 bit0=1（Thumb 代码）；栈里存 PC 时清 bit0
  只是地址对齐，与状态切换无关。

# 调试问题





