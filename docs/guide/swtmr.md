# 软件定时器

## 概述

### 基本概念

**软件定时器**，是基于系统 Tick 时钟中断且由软件来模拟的定时器，当经过设定的 Tick 时钟计数值后会触发用户定义的回调函数。定时精度与系统 Tick 时钟的周期有关。  

硬件定时器受硬件的限制，数量上不足以满足用户的实际需求，因此为了满足用户需求，提供更多的定时器，Huawei LiteOS 操作系统提供软件定时器功能。 软件定时器扩展了定时器的数量，允许创建更多的定时业务。  

软件定时器功能上支持：  
 
- 静态裁剪：能通过宏关闭软件定时器功能。
 
- 软件定时器创建。
 
- 软件定时器启动。
 
- 软件定时器停止。
 
- 软件定时器删除。
 
- 软件定时器剩余Tick数获取

### 运作机制

软件定时器是系统资源，在模块初始化的时候已经分配了一块连续的内存，系统支持的最大定时器个数由 `los_config.h` 中的 `LOSCFG_BASE_CORE_SWTMR_LIMIT` 宏配置。  

软件定时器使用了系统的一个队列和一个任务资源，软件定时器的触发遵循队列规则，先进先出。定时时间短的定时器总是比定时时间长的靠近队列头，满足优先被触发的准则。  

软件定时器以 Tick 为基本计时单位，当用户创建并启动一个软件定时器时，Huawei LiteOS 会根据当前系统 Tick 时间及用户设置的定时间隔确定该定时器的到期 Tick 时间，并将该定时器控制结构挂入计时全局链表。  

当 Tick 中断到来时，在 Tick 中断处理函数中扫描软件定时器的计时全局链表，看是否有定时器超时，若有则将超时的定时器记录下来。  

Tick 中断处理函数结束后，软件定时器任务（优先级为最高）被唤醒，在该任务中调用之前记录下来的定时器的超时回调函数。  

**定时器状态**

- OS_SWTMR_STATUS_UNUSED（未使用）  

系统在定时器模块初始化的时候将系统中所有定时器资源初始化成该状态。  

- OS_SWTMR_STATUS_CREATED（创建未启动/停止）  

在未使用状态下调用 `LOS_SwtmrCreate` 接口或者启动后调用 `LOS_SwtmrStop` 接口后，定时器将变成该状态。  

- OS_SWTMR_STATUS_TICKING（计数）  

在定时器创建后调用 `LOS_SwtmrStart` 接口，定时器将变成该状态，表示定时器运行时的状态。

**定时器模式**

Huawei LiteOS 的软件定时器提供两类定时器机制：  

- 第一类是单次触发定时器，这类定时器在启动后只会触发一次定时器事件，然后定时器自动删除。

- 第二类是周期触发定时器，这类定时器会周期性的触发定时器事件，直到用户手动地停止定时器，否则将永远持续执行下去。  

## 开发指导

### 使用场景

- 创建一个单次触发的定时器，超时后执行用户自定义的回调函数。

- 创建一个周期性触发的定时器，超时后执行用户自定义的回调函数。

### 功能

Huawei LiteOS系统中的软件定时器模块为用户提供下面几种功能，下面具体的API详见软件定时器对外接口手册。  

| 功能分类                 | 接口名            | 描述                     |
|--------------------------|-------------------|--------------------------|
| 创建、删除定时器         | `LOS_SwtmrCreate`  | 创建定时器               |
|                          | `LOS_SwtmrDelete`  | 删除定时器               |
| 启动、停止定时器         | `LOS_SwtmrStart`   | 启动定时器               |
|                          | `LOS_SwtmrStop`    | 停止定时器               |
| 获得软件定时器剩余Tick数 | `LOS_SwtmrTimeGet` | 获得软件定时器剩余Tick数 |

### 开发流程

软件定时器的典型开发流程：  

1.  配置软件定时器。  

	- 确认配置项 `LOSCFG_BASE_CORE_SWTMR` 和 `LOSCFG_BASE_IPC_QUEUE` 为 `YES` 打开状态；

	- 配置 `LOSCFG_BASE_CORE_SWTMR_LIMIT` 最大支持的软件定时器数；

	- 配置 `OS_SWTMR_HANDLE_QUEUE_SIZE` 软件定时器队列最大长度；

2.  创建定时器 `LOS_SwtmrCreate`。  

	- 创建一个指定计时时长、指定超时处理函数、指定触发模式的软件定时器；

	- 返回函数运行结果，成功或失败；  

3.  启动定时器 `LOS_SwtmrStart`。  

4.  获得软件定时器剩余 Tick 数 `LOS_SwtmrTimeGet`。  

5.  停止定时器 `LOS_SwtmrStop`。  

6.  删除定时器 `LOS_SwtmrDelete`。  

### 软件定时器错误码

对软件定时器存在失败可能性的操作，包括创建、删除、暂停、重启定时器等等，均需要返回对应的错误码，以便快速定位错误原因。  

| 序号 | 定义                                      | 实际数值   | 描述                                         | 参考解决方案                                               |
|------|-------------------------------------------|------------|----------------------------------------------|------------------------------------------------------------|
| 1    | `LOS_ERRNO_SWTMR_PTR_NULL`              | 0x02000300 | 软件定时器回调函数为空                       | 定义软件定时器回调函数                                     |
| 2    | `LOS_ERRNO_SWTMR_INTERVAL_NOT_SUITED`  | 0x02000301 | 软件定时器间隔时间为0                        | 重新定义间隔时间                                           |
| 3    | `LOS_ERRNO_SWTMR_MODE_INVALID`          | 0x02000302 | 不正确的软件定时器模式                       | 确认软件定时器模式，范围为[0,2]                            |
| 4    | `LOS_ERRNO_SWTMR_RET_PTR_NULL`         | 0x02000303 | 软件定时器ID指针入参为NULL                   | 定义ID变量，传入指针                                       |
| 5    | `LOS_ERRNO_SWTMR_MAXSIZE`                | 0x02000304 | 软件定时器个数超过最大值                     | 重新定义软件定时器最大个数，或者等待一个软件定时器释放资源 |
| 6    | `LOS_ERRNO_SWTMR_ID_INVALID`            | 0x02000305 | 不正确的软件定时器ID入参                     | 确保入参合法                                               |
| 7    | `LOS_ERRNO_SWTMR_NOT_CREATED`           | 0x02000306 | 软件定时器未创建                             | 创建软件定时器                                             |
| 8    | `LOS_ERRNO_SWTMR_NO_MEMORY`             | 0x02000307 | 软件定时器链表创建内存不足                   | 申请一块足够大的内存供软件定时器使用                       |
| 9    | `LOS_ERRNO_SWTMR_MAXSIZE_INVALID`       | 0x02000308 | 不正确的软件定时器个数最大值                 | 重新定义该值                                               |
| 10   | `LOS_ERRNO_SWTMR_HWI_ACTIVE`            | 0x02000309 | 在中断中使用定时器                           | 修改源代码确保不在中断中使用                               |
| 11   | `LOS_ERRNO_SWTMR_HANDLER_POOL_NO_MEM` | 0x0200030a | membox内存不足                               | 扩大内存                                                   |
| 12   | `LOS_ERRNO_SWTMR_QUEUE_CREATE_FAILED`  | 0x0200030b | 软件定时器队列创建失败                       | 检查用以创建队列的内存是否足够                             |
| 13   | `LOS_ERRNO_SWTMR_TASK_CREATE_FAILED`   | 0x0200030c | 软件定时器任务创建失败                       | 检查用以创建软件定时器任务的内存是否足够并重新创建         |
| 14   | `LOS_ERRNO_SWTMR_NOT_STARTED`           | 0x0200030d | 未启动软件定时器                             | 启动软件定时器                                             |
| 15   | `LOS_ERRNO_SWTMR_STATUS_INVALID`        | 0x0200030e | 不正确的软件定时器状态                       | 检查确认软件定时器状态                                     |
| 16   | `LOS_ERRNO_SWTMR_SORTLIST_NULL`         | null       | 暂无                                         | 该错误码暂不使用                                           |
| 17   | `LOS_ERRNO_SWTMR_TICK_PTR_NULL`        | 0x02000310 | 用以获取软件定时器超时tick数的入参指针为NULL | 创建一个有效的变量                                         |

**错误码定义：**

错误码是一个 32 位的存储单元，31~24 位表示错误等级，23~16 位表示错误码标志，15~8 位代表错误码所属模块，7~0 位表示错误码序号，如下  

```c
#define LOS_ERRNO_OS_NORMAL(MID,ERRNO)  \
(LOS_ERRTYPE_NORMAL | LOS_ERRNO_OS_ID | ((UINT32)(MID) << 8) | (ERRNO))
```
::: tip
- `LOS_ERRTYPE_NORMAL` ：Define the error level as critical
- `LOS_ERRNO_OS_ID` ：OS error code flag.
- `MID` ：OS_MOUDLE_ID
- `ERRNO` ：error ID number    

例如：  
```c  
#define LOS_ERRNO_SWTMR_PTR_NULL  \
LOS_ERRNO_OS_ERROR(LOS_MOD_SWTMR, 0x00)  
```
:::

## 注意事项

- 软件定时器的回调函数中不要做过多操作，不要使用可能引起任务挂起或者阻塞的接口或操作。

- 软件定时器使用了系统的**一个队列**和**一个任务资源**，软件定时器任务的优先级设定为 0，且不允许修改 。

- 系统可配置的软件定时器资源个数是指：整个系统可使用的软件定时器资源总个数，而并非是用户可使用的软件定时器资源个数。例如：系统软件定时器多占用一个软件定时器资源数，那么用户能使用的软件定时器资源就会减少一个。

- 创建单次软件定时器，该定时器超时执行完回调函数后，系统会自动删除该软件定时器，并回收资源。

## 编程实例

### 实例描述

在下面的例子中，演示如下功能：  

1.  软件定时器创建、启动、删除、暂停、重启操作。  

2.  单次软件定时器，周期软件定时器使用方法。  

### 编程示例

前提条件：

- 在 `los_config.h` 中，将 `LOSCFG_BASE_CORE_SWTMR` 配置项打开。

- 配置好 `LOSCFG_BASE_CORE_SWTMR_LIMIT` 最大支持的软件定时器数。

- 配置好 `OS_SWTMR_HANDLE_QUEUE_SIZE` 软件定时器队列最大长度。

代码实现如下：

```c
static void Timer1_Callback    (UINT32 arg);
static void Timer2_Callback    (UINT32 arg);


static UINT32 g_timercount1 = 0;
static UINT32 g_timercount2 = 0;


static void Timer1_Callback(UINT32 arg)//回调函数1
{  
    unsigned long tick_last1;

    g_timercount1 ++;
    tick_last1=(UINT32)LOS_TickCountGet();//获取当前Tick数
    dprintf("g_timercount1=%d\n",g_timercount1);
    dprintf("tick_last1=%lu\n",tick_last1);
}

static void Timer2_Callback(UINT32 arg)//回调函数2
{
    UINT32 uwRet = LOS_OK;
    unsigned long tick_last2;

    tick_last2=(UINT32)LOS_TickCountGet();
    g_timercount2 ++;
    dprintf("g_timercount2=%d\n",g_timercount2);
    dprintf("tick_last2=%lu\n",tick_last2);
    uwRet = LOS_InspectStatusSetByID(LOS_INSPECT_TIMER,LOS_INSPECT_STU_SUCCESS);
    if (LOS_OK != uwRet)
    {
        dprintf("Set Inspect Status Err\n");
    }
}

UINT32 Example_swTimer(void)
{
    UINT16 id1;
    UINT16 id2;// timer id
    UINT32 uwRet = LOS_OK;

  /*创建单次软件定时器，Tick数为1000，启动到1000Tick数时执行回调函数1 */ 
    uwRet = LOS_SwtmrCreate(1000, LOS_SWTMR_MODE_ONCE,Timer1_Callback,&id1,1);
    if(LOS_OK != uwRet)
    {
        dprintf("create Timer1 failed\n");
    }
    else
    {
        dprintf("create Timer1 success\n");
    }
    
    /*创建周期性软件定时器，每100Tick数执行回调函数2 */
    uwRet = LOS_SwtmrCreate(100,LOS_SWTMR_MODE_PERIOD,Timer2_Callback,&id2,1);
    if(LOS_OK != uwRet)
    {
        dprintf("create Timer2 failed\n");
    }
    else
    {
        dprintf("create Timer2 success\n");
    }
    
    uwRet = LOS_SwtmrStart(id1);//启动单次软件定时器
    if(LOS_OK != uwRet)
    {
        dprintf("start Timer1 failed\n");
    }
    else
    {
        dprintf("start Timer1 sucess\n");
    }
    
    (void)LOS_TaskDelay(200);//延时200Tick数
    
    uwRet = LOS_SwtmrStop(id1);//停止软件定时器
    if(LOS_OK != uwRet)
    {
        dprintf("stop Timer1 failed\n");
    }
    else
    {
        dprintf("stop Timer1 sucess\n");
    }
    
    uwRet = LOS_SwtmrStart(id1);
    if(LOS_OK != uwRet)
    {
        dprintf("start Timer1 failed\n");
    }
    
    (void)LOS_TaskDelay(1000);
    
    uwRet = LOS_SwtmrDelete(id1);//删除软件定时器
    if(LOS_OK != uwRet)
    {
        dprintf("delete Timer1 failed\n");
    }
    else
    {
        dprintf("delete Timer1 sucess\n");
    }
    
    uwRet = LOS_SwtmrStart(id2);//启动周期性软件定时器
    if(LOS_OK != uwRet)
    {
        dprintf("start Timer2 failed\n");
    }
    else
    {
        dprintf("start Timer2 success\n");
    }
    
    (void)LOS_TaskDelay(1000);
    
    uwRet = LOS_SwtmrStop(id2);
    if(LOS_OK != uwRet)
    {
        dprintf("stop Timer2 failed\n");
    }
    
    uwRet = LOS_SwtmrDelete(id2);
    if(LOS_OK != uwRet)
    {
        dprintf("delete Timer2 failed\n");
    }
    
    return LOS_OK;
}  
```  

### 结果验证
得到的结果为：  