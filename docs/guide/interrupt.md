# 中断

## 概述

中断是指出现需要时，CPU暂停执行当前程序，转而执行新程序的过程。即在程序运行过程中，系统出现了一个必须由CPU立即处理的事务。此时，CPU暂时中止当前程序的执行转而处理这个事务，这个过程就叫做中断。  

众多周知，CPU的处理速度比外设的运行速度快很多，外设可以在没有CPU介入的情况下完成一定的工作，但某些情况下需要CPU为其做一定的工作。  

通过中断机制，在外设不需要CPU介入时，CPU可以执行其它任务，而当外设需要CPU时通过产生中断信号使CPU立即中断当前任务来响应中断请求。这样可以使CPU避免把大量时间耗费在等待、查询外设状态的操作上，因此将大大提高系统实时性以及执行效率。  

Huawei LiteOS的中断支持：  

- 中断初始化  

- 中断创建  

- 开/关中断  

- 恢复中断  

- 中断使能  

- 中断屏蔽  

### 中断的介绍

与中断相关的硬件可以划分为三类：设备、中断控制器、CPU本身。  

设备：发起中断的源，当设备需要请求CPU时，产生一个中断信号，该信号连接至中断控制器。  

中断控制器：中断控制器是CPU众多外设中的一个，它一方面接收其它外设中断引脚的输入，另一方面，它会发出中断信号给CPU。可以通过对中断控制器编程实现对中断源的优先级、触发方式、打开和关闭源等设置操作。常用的中断控制器有VIC（Vector Interrupt Controller）和GIC（General Interrupt Controller），在ARM Cortex-M系列中使用的中断控制器是NVIC（Nested Vector Interrupt Controller）。  

CPU：CPU会响应中断源的请求，中断当前正在执行的任务，转而执行中断处理程序。  

### 和中断相关的名词解释

**中断号**：每个中断请求信号都会有特定的标志，使得计算机能够判断是哪个设备提出的中断请求，这个标志就是中断号。  

**中断请求**：“紧急事件”需向CPU提出申请（发一个电脉冲信号），要求中断，及要求CPU暂停当前执行的任务，转而处理该“紧急事件”，这一申请过程称为中断申请。  

**中断优先级**：为使系统能够及时响应并处理所有中断，系统根据中断时间的重要性和紧迫程度，将中断源分为若干个级别，称作中断优先级。Huawei LiteOS支持中断控制器的中断优先级及中断嵌套，同时中断管理未对优先级和嵌套进行限制。  

**中断处理程序**：当外设产生中断请求后，CPU暂停当前的任务，转而响应中断申请，即执行中断处理程序。  

**中断触发**：中断源发出并送给CPU控制信号，将接口卡上的中断触发器置“1”，表明该中断源产生了中断，要求CPU去响应该中断,CPU暂停当前任务，执行相应的中断处理程序。  

**中断触发类型**：外部中断申请通过一个物理信号发送到NVIC，可以是电平触发或边沿触发。  

**中断向量**：中断服务程序的入口地址。  

**中断向量表**：存储中断向量的存储区，中断向量与中断号对应，中断向量在中断向量表中按照中断号顺序存储。  

## 开发指导

### 使用场景   

当有中断请求产生时，CPU暂停当前的任务，转而去响应外设请求。根据需要，用户通过中断申请，注册中断处理程序，可以指定CPU响应中断请求时所执行的具体操作。 

### 功能

Huawei LiteOS 系统中的中断模块为用户提供下面几种功能。 

| **接口名**      | **描述**                       |
|-----------------|--------------------------------|
| `LOS_HwiCreate`  | 硬中断创建，注册硬中断处理程序 |
| `LOS_IntUnLock`  | 开中断                         |
| `LOS_IntRestore` | 恢复到关中断之前的状态         |
| `LOS_IntLock`    | 关中断                         |
| `LOS_HwiDelete`  | 硬中断删除                     |

### 开发流程

1.  修改配置项

	- 打开硬中断裁剪开关：`LOSCFG_PLATFORM_HWI` 定义为 YES   

	- 配置硬中断使用最大数：`LOSCFG_PLATFORM_HWI_LIMIT` 

2.  调用中断初始化 `Los_HwiInit` 接口。  

3.  调用中断创建接口 `LOS_HwiCreate` 创建中断，根据需要使能指定中断。  

4.  调用 `LOS_HwiDelete` 删除中断。  

## 注意事项
  
- 根据具体硬件，配置支持的最大中断数及中断初始化操作的寄存器地址。  

- 中断处理程序耗时不能过长，影响 CPU 对中断的及时响应。  

- 关中断后不能执行引起调度的函数。  

- 中断恢复 `LOS_IntRestore` 的入参必须是与之对应的 `LOS_IntLock` 保存的关中断之前的PRIMASK的值。  

- Cortex-M 系列处理器中 1-15 中断为内部使用，因此不建议用户去申请和创建。  

## 编程实例

### 实例描述

1.  初始化硬件中断  

2.  中断注册  

3.  触发中断  

4.  查看打印结果  

**前提条件：**

在 los_config.h 中，将 `LOSCFG_PLATFORM_HWI` 定义为YES。  

在 los_config.h 中，设置最大硬中断个数 `OS_HWI_MAX_USED_NUM` 。 

目前的中断测试代码提供了基本框架，中断硬件初始化代码请用户根据开发板硬件情况在 Example_Exti0_Init() 函数中自行实现。

代码实现如下：  
```c
static void Example_Exti0_Init()
{
    /*add your IRQ init code here*/

    return;

}

static VOID User_IRQHandler(void)
{
    dprintf("\n User IRQ test\n");
    //LOS_InspectStatusSetByID(LOS_INSPECT_INTERRUPT,LOS_INSPECT_STU_SUCCESS);
    return;

}

UINT32 Example_Interrupt(VOID)
{
    UINTPTR uvIntSave;
    uvIntSave = LOS_IntLock();
    
    Example_Exti0_Init();
    
    LOS_HwiCreate(6, 0, 0, User_IRQHandler, 0); //创建中断
    
    LOS_IntRestore(uvIntSave);
    
    return LOS_OK;
}  
```
### 结果验证

