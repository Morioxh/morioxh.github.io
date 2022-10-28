---
layout: mypost
title: Platform初始化
categories: [UEFI基础]
---

## 常用术语

| 文件  | 说明                                                                                                                          |
| --- | --------------------------------------------------------------------------------------------------------------------------- |
| veb | 项目描述-cif列表 参考的源位置                                                                                                           |
| cif | Apito 项目组件描述文件                                                                                                              |
| sdl | 配置和build options的描述组件                                                                                                       |
| fdf | 描述了二进制image的内容和layout                                                                                                       |
| dsc | 平台描述文件(component/inf+lib+Library class instance mappings+pxd entries),与fdf ,dec,module inf 和component inf结合一起生成PE32类型的二进制文件 |
| dec | 定义了一下可以用在不同EDK II模块间的信息                                                                                                     |
| inf | EDK II build信息文件                                                                                                            |
| mak | GNU Make utility执行makefiles                                                                                                 |

 ![](2022-10-24-09-08-29-image.png)<img title="" src="2022-10-27-10-25-16-image.png" alt="" width="349" data-align="center">

## SEC

> 为什么需要SEC阶段？
> 
> + 需要汇编，比如C无法处理一些CPU的特殊寄存器(eg: MSR,MTRR,CRx等)
> 
> + SEC阶段内存还没初始化，C需要内存来存放变量buffer以及local 变量需要stack
> 
> + 切换CPU模式 从实模式到保护模式

SEC阶段做了啥？

+ CPU初始化
  
  + 进入保护模式
  
  + MTRR/MP初始化
  
  + CAR：初始化Cache as RAM

+ 切换到C代码,转交控制权给PEI阶段，包含BFV(Basic Firmware Volume)，Stack，CAR....

![](2022-10-27-09-06-26-image.png)

> CPU 第一条指令一般是两次NOP（90h），主要是来更新CS寄存器,其位置一般由硬件决定,CS.base(0000_0000_FFFF_0000)+RIP(0000_0000_0000_FFF0)

## PEI

![](2022-10-27-09-06-41-image.png)

+ PEI 阶段目的
  
  + 硬件基本初始化
  
  + 内存大小
  
  + S3 resume
  
  + recovery

+ DXEIPL 是最后的PEIM

+ 当PEIM需要向DXE阶段传递一些信息时，它就会调用PEI Service创建一个HOB并将其挂在HOB列表上。DXEIPL将HOB列表从PEI传递给DXE。

![](2022-10-18-10-51-27-image.png)![](2022-10-18-10-51-48-image.png)![](2022-10-18-10-52-00-image.png)![](2022-10-18-10-52-41-image.png)

## DXE

> DXE foundation 是负责生产UEFI sysetem table和相关联的一些服务的boot service image，DXE foundation初始化后，控制权传递给DXE dispatcher，其负责load和调用FV中发现的DXE drivers

![](2022-10-27-09-07-06-image.png)

![](2022-10-18-10-54-14-image.png)

DXE阶段主要作用

+ 运行所有的DXE 驱动

+ 检查Architectural Protocols

+ 调用BDS

> 代码在ROM中执行完成，驱动都是被压缩的，代码运行在RAM中，CPU cache启用，内存资源都能访问到，DXE core与平台不相干，避免使用了中断，单线程

#### System table

![](2022-10-24-09-20-37-image.png)

#### Runtime Services

![](2022-10-26-13-13-09-image.png)![](2022-10-26-13-13-25-image.png)

#### Boot Services

![](2022-10-26-13-14-56-image.png)![](2022-10-26-13-15-10-image.png)![](2022-10-26-13-15-24-image.png)![](2022-10-26-13-15-36-image.png)

### Event

Boot Services中事件相关函数有6个，CreateEvent/CreateEventEx、
SignalEvent,及CloseEvent、WaitForEvent和CheckEvent。

Aptio V定义的Event Types:

```c
#define EFI_EVENT_TIMER
#define EFI_EVENT_RUNTIME
#define EFI_EVENT_RUNTIME_CONTEXT
#define EFI_EVENT_NOTIFY_WAIT
#define EFI_EVENT_NOTIFY_SIGNAL
#define EFI_EVENT_NOTIFY_SIGNAL_ALL
#define EFI_EVENT_SIGNAL_EXIT_BOOT_SERVICES
#define EFI_EVENT_SIGNAL_VIRTUAL_ADDRESS_CHANGE
#define EFI_EVENT_SIGNAL_READY_TO_BOOT
#define EFI_EVENT_SIGNAL_LEGACY_BOOT
```

## BDS

> 为什么需要BDS?
> 
> 提供OS，设置硬件status接口，选择boot device

主要作用：

+ Binding 必要的驱动
  
  + PCI枚举
  
  + 连接控制台（显示器，KB/Mouse等）
  
  + 启动设备初始化 (IDE/SATA/SCSi)

+ goto Setup 一些系统信息，设备设置，启动管理等

+ 选择启动设备到OS

![](2022-10-27-09-07-39-image.png)

## SIO

进入操作模式->选择logic 设备 -> 激活 - > 操作logic 设备 - >退出配置模式

通过ISA 方式访问 SIO 选择的index/data 是2E/2F  (0x87)还是4E/4F (0xA5)

+ 硬件拉线决定

+ Global Control Register CR26 Bit6（HEFRAS），并且要打开LPC 0x80-0x83相应的decode

寄存器设置

<img src="2022-10-27-13-30-13-image.png" title="" alt="" width="513">

CR30  : bit0 置1 激活

CR60 - 63 : IO 基地址

CR70 : bit 0-3 IRQ

CRF0 : clock rate，  Port 92，Gate A20