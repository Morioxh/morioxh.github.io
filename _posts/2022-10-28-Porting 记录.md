# Porting 记录

## 添加自己的项目pkg

![](2022-10-21-10-24-13-image.png)

### 平台配置

配置平台 和 CPU  ，支持的dimm 以及 通道数

![](2022-10-21-10-27-01-image.png)

```c
1. NSOCKETS
2. NUMBER_OF_DIMMS_PER_CHANNEL
```

### PCIE配置

#### 添加PCIDEVICE

根据porting guide - PCIE lane分配 +配置

1. 给root port添加热插拔HP(nvme u.2?)
   
   一般好像是配GPP bridge 如 "PCIe GPP Bridge 0 Socket0 NBIO0 TypeA" 设备HotPlug = Yes
   
   ![](2022-10-21-11-29-49-image.png)

2. 然后配置如root port 下 endpoint 设备如 "P0 - x4 Pcie Slot Slimline J157-1 - Slot1"

```cpp
PCIDEVICE
    Title  = "P0 - x4 Pcie Slot Slimline J157-1"
    Parent = "PCIe GPP Bridge 0 Socket0 NBIO0 TypeA"
    Attribute  = "0x0"
    Dev_type  = "PciDevice"
    Dev  = 0h
    Slot  = 01h
    IntA =  LNKG; 24
    IntB =  LNKH; 25
    IntC =  LNKE; 26
    IntD =  LNKF; 27
    DeviceType = Slot
    PCIBusSize = 32bit
    ROMMain = No
    PCIExpress = Yes
    HasSetup = Yes
    Help  = "P0 - x4 Pcie Slot Slimline J157-1 - Slot1"
End

PCIDEVICE
    Title  = "P0 - x4 Pcie Slot Slimline J157-2"
    Parent = "PCIe GPP Bridge 1 Socket0 NBIO0 TypeA"
    Attribute  = "0x0"
    Dev_type  = "PciDevice"
    Dev  = 0h
    Slot  = 02h
    IntA =  LNKG; 28
    IntB =  LNKH; 29
    IntC =  LNKE; 30
    IntD =  LNKF; 31
    DeviceType = Slot
    PCIBusSize = 32bit
    ROMMain = No
    PCIExpress = Yes
    HasSetup = Yes
    Help  = "P0 - x4 Pcie Slot Slimline J157-2 - Slot2"
End

PCIDEVICE
    Title  = "P0 - x8 Pcie Slot Slimline J157"
    Parent = "PCIe GPP Bridge 2 Socket0 NBIO0 TypeA"
    Attribute  = "0x0"
    Dev_type  = "PciDevice"
    Dev  = 0h
    Slot  = 03h
    IntA =  LNKG; 32
    IntB =  LNKH; 33
    IntC =  LNKE; 34
    IntD =  LNKF; 35
    DeviceType = Slot
    PCIBusSize = 32bit
    ROMMain = No
    PCIExpress = Yes
    HasSetup = Yes
    Help  = "P0 - x8 Pcie Slot Slimline J157 - Slot3"
End
```

![](2022-10-21-17-21-44-image.png)

#### 配置内存

配置ApcbData_GN_GID_0x1704_Type_DimmInfoSpd.c 中DIMM_INFO_SMBUS配置

![](2022-10-21-17-25-57-image.png)

ApcbCustomizedDefinitions.h中APCB 选项值 可以参考之前项目 下面比较重要

I2C Mux地址 如PCA9546地址0xe0

<img title="" src="2022-10-21-17-27-34-image.png" alt="" width="559" data-align="center"><img title="" src="2022-10-21-17-41-02-image.png" alt="" width="336" data-align="center">

#### GPIO配置

根据GPIO表配置AmdCpmOemTable.c中AMD_CPM_GPIO_INIT_TABLE

<img src="2022-10-21-17-43-36-image.png" title="" alt="" width="697">

 结构体(socket,die,gpio,function,  output, pullup)

> function主要看ppr对应pin脚IOMUX ，其余看电路图

然后分配DXIO
