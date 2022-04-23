---
layout: mypost
title: SEC PEI DXE基础介绍
categories: [UEFI基础,Tiancore Training]
---

## 常见定义

### Legacy Bios and Uefi

80年代 IBM在Intel 16位 8086 架构开发了Legacy BIOS 后来到32位 

UEFI最初在EFI spec1.10提出,定义为操作系统与平台固件之间的软件接口。UEFl体系结构允许用户在命令行接口上执行应用程序。它具有内在的网络功能，并被设计用于多处理器(MP)系统。

UEFl为“boot services”和“runtime services”使用了不同的接口，但是UEFl没有指定如何实现“POST”(POST)和Setup 这是BIOS的主要功能。

### Hardware Bios->UEFI->OS Loader

BIOS是固件层,具有底层硬件的所有信息，通常存储在可以拦截处理器架构复位矢量reset vector的 ROM 位置.
BIOS 层确定硬件层的工作方式特征和硬件上的组件数量，并列出这些资源，以便以后将其传达给操作系统

The Unified Extensible Firmware Interface (UEFI) layer 负责设置或生成其他软件或固件组件可以使用的 services and protocols .该接口层提供了与硬件层的基本通信，以便其他组件通过通信来初始化和启动系统中的各种设备。使用这些协议的其他组件或设备如PCI或PCI扩展卡、控制台、USB设备等。



UEFI 与 BIOS 区别 主要是ACPI SMBIOS 访问方法不同



### UEFI 和 PI 

UEFl仅限于用于操作系统和系统固件之间交互的编程接口。PI 规范是一种固件实现，旨在执行初始化平台所需的全部操作--从电源到将控制转移到操作系统。

PI spec 定义系统如何从开机状态变为 UEFI可用的状态,还定义了未包含在 UEFI 规范中的操作系统所需的其他特定硬件平台的元素.当 UEFl 与操作系统通信时，UEFl 不处理内存初始化、恢复或平台初始化这些platform firmware负责的内容。Pl规范在基础架构中工作;Pl 采用您构建的组件并定义它们如何正确交互以实现引导过程。

![image-20220421215439541](image-20220421215439541.png)

1.“PI”界定了不同的阶段引导过程. PI定义了为在该阶段运行而设计的主题模块的服务和约束。

2.每个阶段都建立在之前，直到系统为操作系统做好准备    

3.UEFl接管以支持Option ROMs 和OS.

![image-20220421220029004](image-20220421220029004.png)

SEC阶段是安全验证阶段 就平台启动上电确保Firmware完整

PEI阶段的主要目的是完成必要的处理器、芯片组和平台的配置 用于发现内存。PEI为下一阶段建立了基本环境。
实际上，当PEI代码转到下一阶段时，它就消失了。传递到下一个阶段的唯一东西就是 是PEI阶段完成的HOBs表。
HOBs用于在启动早期阶段传递系统信息的机制。

DXE阶段是进行大部分引导过程的地方：设备枚举和初始化，UEFl服务得到支持，Protocol和driver已经实现。此外，还会生成创建EFI接口的表。

BDS阶段负责决定了怎样在哪里启动OS

TSL阶段转交控制权给OS

### EDK II

EDK 是开源的 EFI development kit 提供了开发环境.EDK II 是跨平台的 完整包

## SEC & PEI

### 开发环境

源码包: EDK II 、EFI Toolkit、EFI Shell 、UEFI Self-Certification Test

代码：开源、商业编译人员兼容

库、驱动

### SEC

SEC阶段,多为底层 ，你可能有最少的代码并且可能多为汇编.一般从Flash中执行,所以没有被压缩.如果你想用硬件debugger,你必须从reset vector开始执行.

SEC阶段主要负责:

1. SEC阶段负责平台所有重启活动
2. 创建临时的内存存储
3. SEC是系统中信任的根源，包含控制系统的初始代码。该代码可以选择对PEI进行身份验证
4. 转交控制权给PEI 

![image-20220422082955552](image-20220422082955552.png)

### PEI

PEI 是PI第2阶段,目的就是为了初始化内存和其他平台资源来允许DXE(Driver Execution Environment)阶段运行在C环境中

PEI阶段特点:

1. PEI(和SEC)使用reset、INIT和机器检查结构(MCA)。在ltanium上，reset进入处理器抽象层(PAL)调用firmware。入口有一个INIT，它执行处理器reset。
2. PEl阶段是一个小而紧凑的启动代码，因为它在ROM中执行. 允许模块驱动程序位于绝对地址。这些地址是直接执行的，因为它们是通过使用build tools解析的。
3. PEl 利用即将推出的英特尔架构处理器中的新架构支持，让您更容易用 C 来reset。由于 C 需要堆栈，因此需要可用的临时内存
4. 内核定位、验证、调度和执行 PEl modules （PEIM - 在 PEl中运行,支持chipset或平台功能的固件代码模块化块)。
5. PEI有自己的protocol PPI (PEIM to PEIM Interface)
6. 
   PEI的主要目标是发现boot 模式；启动初始化主内存的模块；发现并启动DXE core；并将平台信息传到DXE。

为什么要用PEI?

1. 没内存写代码难
2. 老代码的寄存器规则复杂,许多供应商用不同的IA32 寄存器来处理 比如一家用EBP,另外一家用EBX来存储,就会导致很难port 代码
3. PEI是你最先内存初始化和基本的chipset初始化,并且在S3、睡眠重启 、恢复中扮演主要角色



PEI功能：

1. 发现和初始化一些不会被重新配置的RAM
2. 描述了DXE core 和 Architecture Protocols（AP）等firmware volums （FVs）的位置
3. 描述了一些其他的复杂的 平台定义的PEI独有的资源

PEI 内容组件

1. 二进制文件PEI Core和一些 PEIMs ：标准头，包含 执行代码/数据、重新定位信息、身份验证信息.
2. 接口 PEIM间通信的方法：PEI Services , PPIs ，一些简单的Notifies

PEI执行环境

1. 在ROM中执行
2. 一小部分在临时RAM中



#### PEI 阶段中内存

PEI  Core 最小需要的临时内存取决于 处理器和 chipset 比如 IA32 你只需要8K. Intel 架构有个不驱逐模式 允许PEI用Cache 来 作为临时RAM (CAR).CAR module只存数据 ,在flash中执行。好处是为IA32 和 IPF 处理器系统代码普及.

​																			IA32 Memory Map

![image-20220422091923471](image-20220422091923471.png)

- BFV (Boot firmware volum)是PEI 代码存储的地方 通常被认为是 recovery block
- Firmware Volume (FV) 是firmware数据或代码的逻辑存储库，通常组织到文件系统中。每个firmware volumes都具有相关属性定义 具体 在PI Spec Volume 3:Shared Architectural Elements中.
- temporary memory(T-RAM) 是系统内存初始化前 栈和 数据区 的位置
- PEI发现和初始化了系统内存

#### PEI Core

​	PEI Core 是负责dispatch PEIM 和提供一些基础服务的主要PEI 可执行 二进制文件。PEI存储在BFV中,主要由C写,也有一些为了性能用汇编写。主要包含了一些Core Dispatcher 用来locate PEIM 和 按顺序执行modules.另外还有些PEI Services 

示例:PEIMain ,入口就是PEI core

```c
VOID
EFIAPI
PeiCore (
  IN CONST EFI_SEC_PEI_HAND_OFF    *SecCoreDataPtr, //用来初始化core数据区的旧数据入口,如果为空说明是第一个peicore
  IN CONST EFI_PEI_PPI_DESCRIPTOR  *PpiList,//用PEI core 最初安装的一个或多个list入口 每个PPI list都至少有一个end-tag描述
  IN VOID                          *Data //用于包含PEI core操作环境的数据结构入口 ,比如临时RAM,栈,BFV位置啊 
  )
{
    // lnitialize PEl Core Services
	InitializePpiServices (&PrivateData, OldCoreData);
	// Call PEIM dispatcher
	PeiDispatcher (SecCoreData, &PrivateData);
  // Call to DXE IPL Entry
  
```


![image-20220422100739138](image-20220422100739138.png)


#### PEI Services

PPI :PEIM间通讯接口,通常在T-RAM数据库中

Boot Mode: 管理系统S3 S5 正常boot 和诊断等

HOB : 传递信息给下一阶段

FV : 帮助Firmware 文件系统 (FFS)找到 PEIM 和其他 flash 设备中的firmware文件

PEI memory : 在内存找到前后 提供内存管理服务

Status Code : 提供了一些常见进程和错误码 比如080h 或者 串口来简单文字输出 来debug

Reset : 提供了系统常规重启方法

![image-20220422102137301](image-20220422102137301.png)

#### PEIM

PEIM 特征

- 可执行对象
- 单独构建，未压缩的二进制 image

- 在FVs 中的文件中包含,可以在多个FVs中，只要提供了搜索它们的机制。

- Execute ln Place (XIP): PEIM从其在闪存设备中的位置直接运行(它们不会加载到RAM中，然后运行)
- 定义其他PEIM的接口，称为PPI
- PEIM 用它所需要的PPI s来描述它的要求。

![image-20220422102928206](image-20220422102928206.png)

##### PPI

Architectural PPI : 其128位GUID被PEI Foundation知道,为平台特定应用提供服务接口, 比如ReportStatusCode();

Additional PPI : 不为基础定义但需要为其他PPI服务 ,比如InstallPPI() , LocatePPI() , NotifyPPI() 等 

PPI 数据库 包含了 GUID 指针 GUID , PPI 指针 PPI ,  Flag等  通过一些additional PPI访问和操作

```c
EFI_STATUSEFIAPI
PeiFfsFindNextVolume(
	IN CONST EFI_PEI_SERVICES **PeiServices,
	IN UINTN lnstance,
	IN OUT EFI_PEI_FV_HANDLE *VolumeHandle
)
/*
Search the firmware volumes by index
@param PeiServices 		An indirect pointer to the EFl_PEl_SERVICES table published by the PEI Foundation
@param Instance 		This instance of the firmware volume to find.The value 0 is the Boot Firmware Volume (BFV).
@param VolumeHandle  	On exit, points to the next volume handle or NULL if it does not exist.
@retval EFl_INVALID_PARAMETER VolumeHandle is NULL
@retval EFl_NOT_FOUND	The volume was not found.
@retval EFl_SUCCESS	The volume was found.
*/
```

#### PEI Core Dispatcher

Dispatch 判断:

- DEPEX (dependency expression)算法 一般在core/simple BNF(Backus Naur Form)中
- 文件验证状态: OEM厂商的算法或平台 policy

哪些PEIM可以运行

- 还未运行
- 存在验证
- 其GUID出现在其他PPI database 描述中已经跑过的DEPEX中

多个FV dispatch会复杂 ,所以一个PEIM必须描述其他FV . Core用FindFv 并且当一个FV被发现就应该被添加到Core算法搜索顺序中.

![image-20220422110127910](image-20220422110127910.png)

#### HOBs

HOBs是PI spec 定义的一种数据结构, 在PEI阶段产生存在系统内存中供DXE 使用 , PEI通过HOBs收集资源。HOBs 按成堆的块分配，这意味着HOBs所在的内存通常是连续的。当DXE开始时，DXE会记住HOBs。那些HOBS中的指针描述了无法移除的的物理信息，例如：FVS；物理的内存属性；或由PEIM分配的特定内存。

##### HOB List

DXE core invoke的时候PEIM会传递HOB list

必须的HOBs 有

1. PEI Hand-off Information Table(PHIT) :  HOB链表指针
2. Boot Strap Processor(BSP) Stack : 告诉DXE core 当前栈位置以便其移动
3. Resource Descriptor : 告诉DXE core要用的物理系统内存

其他的HOBs

1. Firmware volume : DXE 需要信息来加载平台驱动用
2. GUIDed : 传递一些不满足预定义格式的其他信息
3. CPU : 描述了一些CPU 地址信息 IO空间容量 信息

##### PEI pre-permanent memory 环境

为了初始化内存使用, 在32位保护模式下, 发现启动类型.入口处需要FV和PEI Service 表位置.此时，还没有真正的系统内存，使用处理器缓存作为数据和内存调用堆栈，允许使用C风格的调用习惯。事实上，高速缓存内存是英特尔处理器体系结构的优势之一，也被称为“无驱逐模式”。最后，要注意环境约束，取决于处理器架构。

##### PEI post-permanent 

为了过渡到DXE 并准备Hobs,也是运行在32位保护模式下.需要内存来存储stack和hob list.需要提供第一个HOB的指针PEI Hand-off Information Table (PHIT).除了C还能FV中 XIP

![image-20220422113225376](image-20220422113225376.png)

#### PEI End

当FV中发现的所有PEIM 及其 依赖 都dispatch结束 .PEI core 将调用 DXE Initial Program Load (IPL) ,DXE IPL将会把DXE core代码加载到内存中,然后进入DXE 阶段

#### PEI 到 DXE 过程

PEI阶段起初 ,没有内存被初始化 接着PEI Core dispatch PEIMs并初始化 并且同时将初始化结果记录在HOB 表中.直到DXE IPL PEIM执行, 有许多HOBs 表 ,然后DXE IPL PEIM转交控制权给DXE Core, 一旦完成转交,就不在拥有访问PEI阶段PPI服务的权利.传给DXE 阶段只有HOB list.

PEI也并非总是转到DXE 也有可能根据启动模式转到其他path ,取决于PEIM决定的boot mode 实际上每个PEIM都能使用PEI Service的 SetBootMode()方法来操作boot mode

#### DXE IPL

- 禁止硬编码地址。

- 找到最大的物理memory HOB-它应该接近memory的顶端。

- 从内存顶部 分配DXE 堆栈。

- 在HOB list的FV中搜索DXE core，将DXE core 加载到内存中，并构建描述DXE核心的HOB。

- 转换栈和DXE core

  EDK II 中call DXE Main 主要通过 HandOffToDxeCore(DxeCoreEntryPoint, HobList) 

```c
EFIAPI
PeiCore (
  IN CONST EFI_SEC_PEI_HAND_OFF    *SecCoreDataPtr,
  IN CONST EFI_PEI_PPI_DESCRIPTOR  *PpiList,
  IN VOID                          *Data
  )
{
	// Lookup DXE IPL PPI
  Status = PeiServicesLocatePpi (
             &gEfiDxeIplPpiGuid,
             0,
             NULL,
             (VOID **)&TempPtr.DxeIpl
             );
  ASSERT_EFI_ERROR (Status);

  // Enter DxeIpl to load Dxe core.
  DEBUG ((DEBUG_INFO, "DXE IPL Entry\n"));
  Status = TempPtr.DxeIpl->Entry (
                             TempPtr.DxeIpl,
                             &PrivateData.Ps,
                             PrivateData.HobList
                             );
    
  // Should never reach here.
  ASSERT_EFI_ERROR (Status);
  CpuDeadLoop ();
}
```

## DXE

DXE保证了BIOS功能通用,易于porting,由一个文件控制boot流程 易于dubug, 可以方便写driver

### DXE Foundation

- 初始化平台 : 初始化了chipset 和 platform ,其有一项功能就是加载构建环境的drivers 并最终加载boot manager 和OS
- Dispatch DXE : 负责dispatch DXE  drivers 这些drivers是 独特的 有特定的依赖 语法规则 决定了dispatch的顺序
- Dispatch UEFI : DXE drivers 结束后 dispatch UEFI drivers, 这些drivers没有依赖规则因此后运行.当没有依赖时暗含依赖 : 所有的boot 和 runtime services 将被初始化
- Load Boot Manager : DXE Foundation dispatcher 会call 每个driver的入口 , 在入口处代码 driver会负责注册相应的protocol,直到决定了boot设备是否OK才完成初始化 并最终load boot manager 然后转交控制权给BDS

特点: 

- 由PEI传过来的 获取HOB List 信息开始. state 初始化需要的信息如 硬件和资源的state都是通过HOB从PEI传来
- 并非都是硬件定义的,比如特定IO被写了就可能导致错误.DXE Foundation用 Architecture Protocols(APs) ,抽象访问硬件
- 可以在系统内存任何地方load DXE Foundation 因为 不是 hard-coded 地址 

构成和PEI差不多

1. Drivers : 负责初始化处理器 chipset 平台元件.更具体的说,提供AP将平台中DXE Core抽象出来

2. Foundation : 主要的DXE执行文件 负责dispatch 驱动和生成一系列Boot、Runtime 、DXE服务 

3. Architectural Protocols : 将硬件抽象成 driver

4. Dispatcher : 用来正确顺序搜索执行驱动

5. UEFI System Table : 包含了所有UEFI 服务表、 配置表、handle 数据库 和console设备的指针

   ### 

### DEX 流程 和数据结构

![image-20220422132855635](image-20220422132855635.png)

![image-20220422133643099](image-20220422133643099.png)

DXE 能用event来触发其他活动事件 具体见PI spec

包含一些signal event 如 timer event （periodic timer event或one shot timer event），Exit Boot Services Events（当call UEFI Boot Service ExitBootService（）时触发），Set Virtual Address Map Events（call UEFI Runtime Service SetVirtualAddressMap()触发 ）还有些wait events

### DXE Foundation

代码流通过DXE Foundation推进,包含

- 单线程环境 : Boot Strap Processor (BSP)是通过DXE Foundation code 推进的唯一进程,所有的应用处理器在等待模式
- 一步中断 : DXE Foundation 使用 timer tick作为唯一软件触发, 因为计时器中断是唯一使用的,所以可以推出所有轮询过的设备. DXE Foundation 使用events代替了多个中断

#### DXE Main

DXE main是EDK II 官方功能调用 ,主要负责![image-20220422185139341](image-20220422185139341.png)

```c
VOID
EFIAPI
DxeMain (
  IN  VOID  *HobStart
  )
{	
    // Initialize Memory Services
    CoreInitializeMemoryServices (&HobStart, &MemoryBaseAddress, &MemoryLength);
    
   // Initialize the DXE Dispatcher
  CoreInitializeDispatcher ();

  // Invoke the DXE Dispatcher
  CoreDispatcher ();
  // Transfer control to the BDS Architectural Protocol
  gBds->Entry (gBds);

  // BDS should never return
  ASSERT (FALSE);
  CpuDeadLoop ();
}
```

### Architectural Protocol

DXE drivers生成APs 来将硬件抽象成DXE . APs充当isolate平台特定硬件的包装功能.例如，CPU AP管理中断、检索处理器信息和查询基于处理器的定时器。APs 是支持boot 和 runtime services的低层次的protocols ,也为高层次的平台功能通过calls . 还有些APs需要依赖 比如看门狗定时器需要IO和timer ,

以下方法控制依赖载入顺序:

1. 依赖语法确认load DXE顺序正确
2. RegistryProtocolNotify() 来申明当AP 载入的时候
3. Priori List是在FV中包含GUIDs 文件名的表的文件

![image-20220422190638553](image-20220422190638553.png)

一些AP位置

![image-20220422190801601](image-20220422190801601.png)

### DXE Dispatcher

负责载入和dispatch DXE 和 FV中发现的UEFI 驱动

```c
EFI_STATUS
EFIAPI
CoreStartImage (
  IN EFI_HANDLE  ImageHandle,   //Handle of image to be started.
  OUT UINTN      *ExitDataSize,		//Pointer of the size to ExitData
  OUT CHAR16     **ExitData  OPTIONAL	//Pointer to a pointer to a data buffer that includes a Null-terminated string
  )
{	
    Image = CoreLoadedImageInfo (ImageHandle);
    
    // Call the image's entry point
    Image->Started = TRUE;
    Image->Status  = Image->EntryPoint (ImageHandle, Image->Info.SystemTable);
    
     DEBUG_CODE_BEGIN ();
    if (EFI_ERROR (Image->Status)) {
      DEBUG ((DEBUG_ERROR, "Error: Image at %11p start failed: %r\n", Image->Info.ImageBase, Image->Status));
    }

    DEBUG_CODE_END ();
    return Status;
}
```

![image-20220422191754319](image-20220422191754319.png)

B->A->C

![image-20220422191915748](image-20220422191915748.png)

### DXE Driver

在UEFI Driver前执行 有两种:

- Early DXE Phase Driver: 在DXE阶段先执行,包含描述顺序的 DEPEX ,产生 APs,通常包含基础服务、平台初始化代码还有处理器和chipset初始化代码
- UEFI driver ：初始化时不会影响硬件，遵循UEFI Driver 模型 ，提供了Console和boot设备访问权 ，抽象出了Bus 控制权 ，只有boot到OS需要用到的driver才会初始化

Boot Device Selection （BDS）driver是DXE call的最后一个driver 负责建立控制台（键盘、视频等）并且产生UEFI Boot选项

### SMM

System management mode(SMM) 服务是控制系统的高优先级SMI 包括runtime , SMM服务设施在DXE阶段建立.

特征:

- 通过SMI,处理器从已知的预定义好的开始向量出执行
- SMM 代码存在一个特殊的位置 SMRAM 并且在SMM代码初始化后locked
- SMI 通过硬件活动或者系统中断生成,可以被检测,清除和关闭.

服务:SMM服务有许多Handler和DXE Foundation 服务,有些提供了相同的功能,但是位于不同的位置,SMM服务存在SMRAM中

DXE 阶段中 EFI Driver Dispatcher 构成 SMM Constructor ,SMM服务形成相应的SMM Handler 一直可用
