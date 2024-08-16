# Windows Driver Model Architecture for SD Host projects

## Driver Pair Model和Device Driver Interface(DDI)的概念

### Driver Pair Model

Microsoft provides the general driver, and typically an independent hardware vendor provides the specific driver.

Driver Pair Model有两种不同层次的实现方式：

(1) Microsoft定义的某类设备的driver pair model: Microsoft实现XX device port driver + 设备厂商实现XX device miniport driver。这类driver pair model包括：

- (display miniport driver, display port driver)
- (audio miniport driver, audio port driver)
- (storage miniport driver, storage port driver)
- (battery miniclass driver, battery class driver)
- (HID minidriver, HID class driver)
- (changer miniclass driver, changer port driver)
- (NDIS miniport driver, NDIS library)

Miniport Driver Pair Model：https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/minidrivers-and-driver-pairs

(2) Microsoft定义的general driver pair model：Microsoft实现WDF框架 + 设备厂商实现KMDF(Kernel Mode WDF)驱动。

- The driver is split into two pieces: one that handles general processing and one that handles processing that is specific to a particular device.
- The general piece, called the Framework, is written by Microsoft.
- The specific piece, called the KMDF driver, may be written by Microsoft or an independent hardware vendor.

 KMDF as a generic driver pair model: https://learn.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/kmdf-as-a-generic-pair-model

### Device Driver Interface(DDI)

The DDI Compliance rules define requirements for the proper interaction between a driver and the kernel interface of the operating system

https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/static-driver-verifier-rules

DDI相当于应用层的API规范，对于每种类型的Driver Pair Mode，都有对应的DDI要求：

[Rules for Audio Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/rules-for-audio-drivers)
[Rules for AVStream Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/rules-for-avstream-drivers)
[Rules for WDM Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/sdv-rules-for-wdm-drivers)
[Rules for KMDF Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/sdv-rules-for-kmdf-drivers)
[Rules for NDIS Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/sdv-rules-for-ndis-drivers)
[Rules for Storport Drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/sdv-rules-for-storport-drivers)

重点对比以下KMDF（Kernel Mode WDF）的DDI和Storport的DDI：

KMDF DDI rules：只说明了一些WDF函数接口的调用条件和先后顺序要求，没有禁用任何WDF函数接口。

Storport DDI rules: 专门定义了禁用的WDM函数列表HwStorPortProhibitedDDIs: a list of WDM DDIs that should not be called in physical StorPort miniport drivers

https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/ddi-usage-rule-set--storport-

### 几个关键问题

（1）Driver Pair Model和Dirver接口函数DDI是什么关系？

Driver Pair Model是Diver接口函数DDI的集合，独立的功能例如创建设备，处理中断，处理IO请求，这些都是Diver接口函数去实现；这些Diver接口函数的集合使用分层设计（部分是Microsoft实现，部分是设备厂商实现）就是Driver Pair Model。

举个例子：Storport-miniport的Driver Pair Model中，默认使用WDM DDI去实现port driver部分和miniport driver部分。

（2）为什么Storport Driver Pair Model会禁用很多WDM接口函数(WDM DDI)，尤其是从Win11开始？

原因：微软定义的Storport driver pair model本身也是基于WDM和WDF函数(DDI)上实现，即用WDM或者WDF Model去实现Storage设备（例如SCSI, NVMe）的数据结构和特殊流程函数，封装成Storport的DDI。而微软的Driver开发趋势是弃用WDM逐步过渡到WDF，因此Storport为了适配Win11系统，可能有很多Storport DDI用WDF DDI重写了，因此Storport禁用了设备厂商去直接调用很多WDM DDI，避免造成同一个功能用WDM和WDF都去实现了而产生冲突。

## SD host的Driver Model选型

### 支持SD host驱动开发的Driver Model

对于SD设备生态可用的Driver Pair Model有：

- KMDF Framework + KMDF driver：通用框架，任意厂商可用KMDF去实现任意类型设备的host/slave驱动；微软也在KMDF去升级已有的特定设备框架，例如微软定义的usbport驱动框架：USB 1.0/2.0使用WDM（usbport.sys），USB 3.0使用WDF（Wdf01000.sys）

- Storport + storage miniport：https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/storport-miniport-drivers

  这个Driver Model支持厂商实现SD host驱动，但其数据和流程定义很多是针对SCSI/NVMe等大容量Disk设备，对于SD host/device没有专门的定义；此外Storport从Win11开始禁用了很多WDM DDI，对Win11兼容性不好评估（可能有Bug）

- sdport + sdhci-miniport：https://learn.microsoft.com/en-us/samples/microsoft/windows-driver-samples/standard-sd-host-controller-miniport/

  这个Driver Model支持厂商实现SD host驱动，但是从2016年就停止维护，微软不建议使用。

- sdbus + sddisk: https://learn.microsoft.com/en-us/windows-hardware/drivers/sd/sd-card-driver-stack

  微软的Inbox SD驱动使用此框架；但不支持其他厂商自定义SD host驱动，只支持SD card厂商去写disk driver。这个框架相当于微软私有的SD host框架。

结论：在Storport Model能很好兼容最新Windows OS，且满足SD host功能的情况下，优先使用Storport（miniport默认使用WDM DDI）；在Storport Model有不可避免的限制导致无法满足Windows OS的SD host功能开发时，使用Kernel Mode WDF框架，完全放弃Storport框架。

### 微软的Driver Model选型的参考文档

https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/creating-a-new-driver

其中分为两种选项：

（1）KMDF框架

（2）miniport框架

微软推荐miniport框架都用WDM DDI开发；不推荐miniport + WDF DDI的混合开发方式（尽管可行），参考：

If your device technology has a minidriver model, and you aren't able to find a specific template for your type of minidriver, the Windows Driver Model (WDM) template is most likely going to be your starting point. Refer to your technology-specific documentation for guidance. In rare cases, you can use KMDF to write a minidriver, but usually the starting point is WDM.

微软开发论坛（OSR）也提出过storport的WDM不够灵活，又不能兼容WDF DDI的问题：

https://www.osr.com/nt-insider/2014-issue4/win7-vs-win8-storport-miniports/

https://community.osr.com/t/wdfdeviceminiportcreate-and-virtual-storport/46617/8

结论：KMDF框架是万能方案

## 从Storport-miniport框架过渡到KMDF框架

（1）问题：Storport-miniport框架一定能porting到KMDF框架吗？

参考：https://learn.microsoft.com/en-us/windows-hardware/drivers/wdf/which-drivers-can-be-ported

Which WDM drivers can I port to WDF?

For some device types, system-supplied device class and port drivers provide driver dispatch functions and call a vendor-supplied miniport driver to handle specific I/O details. These miniport drivers are essentially callback libraries and are not supported by WDF.

结论1：如果该Storage设备驱动依赖于Storport框架定义的流程，例如SCSI， NVMe设备依赖于Storport定义的SCSI和NVMe初始化和I/O流程。那么就不能直接porting到WDF，因为WDF没有定义特定设备的请求流程。

结论2：对于SD设备，Storport框架本身就没有实现SD协议的初始化和I/O流程。SD host设备的Storport-miniport驱动完全可以porting到WDF框架，**唯一可能有问题的是SD disk相关的Storport请求对应WDF的哪些请求？移植到WDF必须要求SD host和Disk整个生态都能顺利过渡**。

（2）问题：Storport代码如何porting到KMDF

主要工作是将Storport-miniport中的DDI函数，映射到KMDF的DDI函数。

Storport-miniport的DDI列表：https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/storport-driver-support-routines

KMDF的DDI列表：https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/_wdf/

基于此方法，后面两章分为两部分：

- Storport DDI在SD host miniport driver中的使用（第4章）：SD host miniport driver用到了哪些DDI，实现什么功能。

- 从Storport DDI到KMDF DDI（第5章）：SD host miniport driver用到的Storport DDI应该用哪些KMDF DDI去重写功能。

## Storport DDI在SD host miniport driver中的使用

Storport DDI routine分为两大类：

（1）callback routine：是Storport定义函数形式，miniport实现函数内容

（2）normal routine：是Storport定义函数形式和内容，miniport只调用



| Storport Callback Routine               | sub-function parameter                                       | Function Description                                         | SD miniport driver's implementation for this callback        |
| --------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **HwStorInitialize**                    | **HwStorPassiveInitializeRoutine**                           | The **HwStorInitialize** routine initializes the miniport driver after a system reboot or power failure occurs. It is called by StorPort after [**HwStorFindAdapter**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter) successfully returns. **HwStorInitialize** initializes the HBA and finds all devices that are of interest to the miniport driver.<br/>The following responsibilities are shared between HwStorInitialize and HwStorPassiveInitializeRoutine:<br/>Initialize hardware for the HBA registers and buffers.<br/>Initialize and allocate all DeviceExtension fields.<br/>Set up and initialize all events and DPCs that are used by the miniport driver.<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_initialize  <br/> <br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_passive_initialize_routine | Support. <br/>HwStorPassiveInitializeRoutine中注册了Adapter的POFX数据（StorPortInitializePoFxPower） |
| **HwStorInterrupt**                     | N/A                                                          | The Storport driver calls the **HwStorInterrupt** routine after the HBA generates an interrupt request. The **HwStorInterrupt** routine should return within 50 microseconds, ideally as short a time as possible. Therefore, all activity does not have to occur at high IRQL should be deferred to the [**HwStorDpcRoutine**] | Support. <br/>用于SD card的插拔卡中断和数据传输完成或错误中断的处理入口 |
| **HwStorFindAdapter**                   | **PORT_CONFIGURATION_INFORMATION** contains configuration information for a host bus adapter (HBA). <br/>https://learn.microsoft.com/zh-cn/windows-hardware/drivers/ddi/storport/ns-storport-_port_configuration_information | The **HwStorFindAdapter** routine uses the supplied configuration to determine whether a specific HBA is supported and, if it is, to return configuration information about that adapter.<br/><br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter<br/><br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-_port_configuration_information | Support. <br/>Main tasks:<br/>1. Zero the device extention<br/>2. Get the memory base, the vendor ID and the device ID<br/>3. Port Configuration Information initialization<br/>4. Driver used memory allocation<br/>5. Initialize the timer, tag queue, host capability, vendor register settings, host software structure<br/> |
| **HwStorAdapterControl**                | **SCSI_ADAPTER_CONTROL_TYPE** <br/> https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ne-storport-scsi_adapter_control_type | A miniport driver's **HwStorAdapterControl** routine is called to perform synchronous operations to control the state or behavior of an adapter, such as stopping or restarting the host bus adapter (HBA) for power management. <br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_adapter_control | 支持部分ControlTypes：<br/>ScsiQuerySupportedControlTypes, ScsiStopAdapter, ScsiRestartAdapter, ScsiSetBootConfig, ScsiSetRunningConfig, ScsiPowerSettingNotification, ScsiAdapterPoFxPowerActive, ScsiAdapterPower, ScsiAdapterPrepareForBusReScan, ScsiAdapterSystemPowerHints, |
|                                         | ScsiQuerySupportedControlTypes                               | Reports the adapter-control operations implemented by the miniport driver. A miniport must support this control type. Using [**SCSI_SUPPORTED_CONTROL_TYPE_LIST**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-scsi_supported_control_type_list) structure | support.                                                     |
|                                         | ScsiStopAdapter                                              | Shuts down the HBA. A miniport must support this control type. | support. <br/>但什么都没干                                   |
|                                         | ScsiRestartAdapter                                           | Reinitializes an HBA. A miniport must support this control type. | support. <br/>调用req_enter_d0()函数，做host reset & init，然后判断卡在位，发起card-change event去处理Adapter Stop期间的card insert or card remove操作。 |
|                                         | ScsiSetBootConfig                                            | Restores any settings on an HBA that the BIOS might need to reboot. A miniport driver must implement **ScsiSetBootConfig** if it must call [**StorPortGetBusData**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportgetbusdata) or [**StorPortSetBusDataByOffset**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetbusdatabyoffset) before the system will be able to reboot. | support. <br/>但什么都没干                                   |
|                                         | ScsiSetRunningConfig                                         | Restores any settings on an HBA that the miniport driver might need to control the HBA while the system is running. A miniport driver must implement **ScsiSetRunningConfig** if it must call [**StorPortGetBusData**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportgetbusdata) or [**StorPortSetBusDataByOffset**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetbusdatabyoffset) to restore the appropriate running configuration to the HBA before it can be restarted. | support. <br/>调用pcr_part_a_restore和pcr_part_b_restore函数去配置SD host PCR register的一些默认值 |
|                                         | ScsiPowerSettingNotification                                 | Notification for a registered power setting change. Using [**STOR_POWER_SETTING_INFO**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_power_setting_info) structure | support. <br/>但什么都没干                                   |
|                                         | ScsiAdapterPower                                             | Reports the adapter power on or power off states. Using [**STOR_ADAPTER_CONTROL_POWER**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_adapter_control_power) structure. 其中包括**STOR_POWER_ACTION**和**STOR_DEVICE_POWER_STATE** 两个子参数 | support. <br/>实现STOR_DEVICE_POWER_STATE = D0~D3和STOR_POWER_ACTION = Sleep/Hibernate/Shutdown/ShutdownReset/ShutdownOff/StorPowerActionNone的设备、系统的电源状态请求 |
|                                         | ScsiAdapterPoFxPowerRequired                                 | Notifies the miniport whether power is required or not for the adapter component. | Not support                                                  |
|                                         | ScsiAdapterPoFxPowerActive                                   | Notifies the miniport whether the adapter component is active or idle.using [**STOR_POFX_ACTIVE_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_active_context) structure | support<br/>只读了Storport下发的STOR_POFX_ACTIVE_CONTEXT，而没使用。相当于什么都没干 |
|                                         | ScsiAdapterPoFxPowerSetFState                                | Notifies the miniport to set the adapter component to the given F-state. using [**STOR_POFX_FSTATE_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_fstate_context) structure | Not support                                                  |
|                                         | ScsiAdapterPoFxPowerControl                                  | Requests that the miniport execute a private power control operation that was initiated for the adapter by a power engine plug-in (PEP). using [**STOR_POFX_POWER_CONTROL**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_power_control) structure. | support. <br/>但什么都没干                                   |
|                                         | ScsiAdapterPrepareForBusReScan                               | Notifies the miniport to prepare the adapter for bus enumeration. | support. <br/>但什么都没干                                   |
|                                         | ScsiAdapterSystemPowerHints                                  | Provides system power hints to the miniport. using [**STOR_SYSTEM_POWER_HINTS**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_system_power_hints) structure | support. <br/>但什么都没干                                   |
|                                         | ScsiAdapterFilterResourceRequirements                        | Filters the required resources for the adapter. Using [**STOR_FILTER_RESOURCE_REQUIREMENTS**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_filter_resource_requirements) structure. | Not support                                                  |
|                                         | ScsiAdapterPoFxMaxOperationalPower                           | Communicates a maximum operational power value to the miniport. Using [**STOR_MAX_OPERATIONAL_POWER**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_max_operational_power) structure | Not support                                                  |
|                                         | ScsiAdapterPoFxSetPerfState                                  | Informs the miniport of the status of a P-State transition requested by a call to [**StorPortPoFxSetPerfState**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportpofxsetperfstate). Using STOR_POFX_PERF_STATE_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_perf_state_context) structure | Not support                                                  |
|                                         | ScsiAdapterSurpriseRemoval                                   | Notifies the miniport that the unit has been surprise-removed. | Not support                                                  |
|                                         | ScsiAdapterSerialNumber                                      | Requests that the miniport retrieve the adapter's serial number. | Not support                                                  |
|                                         | ScsiAdapterQueryFruId                                        | Available starting in Windows 10 version 21H1. Queries the ID of a fault replacement unit (FRU) of the adapter. Storport sends this control only if a miniport has also previously called [**StorPortSetFeatureList**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetfeaturelist) in its [**HwFindAdapter**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter) routine with **StorportFeatureFruIdAdapterControl** specified. | Not support                                                  |
|                                         | ScsiAdapterSetEventLogging                                   | Available starting in Windows 10 version 21H1. Notifies the miniport about whether a specific event channel is enabled or disabled for an adapter. Storport sends this control only if a miniport has also previously called [**StorPortSetFeatureList**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetfeaturelist) in its [**HwFindAdapter**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter) routine with **StorportFeatureFruIdAdapterControl** specified. | Not support                                                  |
|                                         | ScsiAdapterResetBusSynchronous                               | Available starting in Windows 11, version 22H2. Storport sends this control during the handling of an IOCTL_STORAGE_DEVICE_RESET. The miniport driver should handle this control similar to what it does in its [**HwResetBus**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_reset_bus) callback routine. Storport sends this control only if a miniport has also previously called [**StorPortSetFeatureList**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetfeaturelist) in its [**HwFindAdapter**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter) routine with **StorportFeatureResetBusSynchronous** specified. | Not support                                                  |
|                                         | ScsiAdapterPreparePLDR                                       | Storport sends this control to notify miniport to do necessary work before invoking PLDR. Available starting in Windows 11, version 24H2 | Not support                                                  |
|                                         | ScsiNvmeofAdapterOperation                                   | Indicates whether ScsiNvmeofAdapterOperation is supported. Available starting in Windows 11, version 24H2. | Not support                                                  |
|                                         |                                                              |                                                              |                                                              |
| **HwStorStartIo** and **HwStorBuildIo** | **HwStorBuildIo** and **HwStorStartIo** receive the following Srb function types: | The Storport driver calls the **HwStorStartIo** routine one time for each incoming I/O request.<br/>**HwStorStartIo** initiates an I/O operation. StorPort is designed to use a miniport's private data that is prepared in [**HwStorBuildIo**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_buildio) and stored in either **DeviceExtension** or **Srb->SrbExtension**. Because **HwStorBuildIo** is called without spin locks, the best driver performance is achieved by preparing as much data as possible in **HwStorBuildIo**. <br/><br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_startio |                                                              |
|                                         |                                                              | The **HwStorBuildIo** routine processes the SRB with unsynchronized access to shared system data structures before passing it to [**HwStorStartIo**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_startio). |                                                              |
|                                         | SRB_FUNCTION_EXECUTE_SCSI                                    | Sends a CDB to the specified bus/target/lun.                 | 根据如下SCSI operation code分别处理：<br/>SCSIOP_TEST_UNIT_READY, SCSIOP_REQUEST_SENSE, SCSIOP_INQUIRY, SCSIOP_READ_CAPACITY, SCSIOP_READ, SCSIOP_WRITE, SCSIOP_MODE_SENSE, SCSIOP_VERIFY, SCSIOP_LOAD_UNLOAD, SCSIOP_MEDIUM_REMOVAL<br/>注：这部分代码有多存疑的地方，代码本身是从老的SCSI驱动移植过来，SD设备不属于SCSI disk类型，这些SCSI请求哪些应该在SD disk上用哪些不该用，没有微软文档说明 |
|                                         | SRB_FUNCTION_IO_CONTROL                                      | Miniport defined.                                            | 直接返回SRB_STATUS_SUCCESS                                   |
|                                         | SRB_FUNCTION_RESET_LOGICAL_UNIT                              | Reset the specified logical unit (if the device is capable). | Not support                                                  |
|                                         | SRB_FUNCTION_RESET_DEVICE                                    | Reset the specified Scsi Target.                             |                                                              |
|                                         | SRB_FUNCTION_RESET_BUS                                       | Reset all of the targets on the specified SCSI bus.          | 直接返回SRB_STATUS_SUCCESS                                   |
|                                         | SRB_FUNCTION_FLUSH                                           | Instructs the miniport driver to flush all cached data.      | Not support                                                  |
|                                         | SRB_FUNCTION_SHUTDOWN                                        | Instructs the miniport driver to flush all cached data preparatory to shut down. | Not support                                                  |
|                                         | SRB_FUNCTION_DUMP_POINTERS                                   | Supplies information needed for the miniport driver to support crash dump and hibernation. This request is sent to a Storport virtual miniport driver that is used to control the disk that holds the crash dump data. Starting with Windows 8, non-virtual miniport drivers can optionally receive this request. | Not support                                                  |
|                                         | SRB_FUNCTION_FREE_DUMP_POINTERS                              | Starting with Windows 8, this request is sent to the miniport to free resources allocated during the SRB_FUNCTION_DUMP_POINTERS request. | Not support                                                  |
|                                         | SRB_FUNCTION_PNP                                             | https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/srb/ns-srb-_stor_device_capabilities_ex | 只处理STOR_DEVICE_CAPABILITIES_EX定义的StorQueryCapability请求类型去上报Removable，EjectSupported等能力<br/>The **STOR_DEVICE_CAPABILITIES_EX** structure reports device capabilities to the SCSI port driver in response to a capabilities query in a SCSI request block (SRB) with a function of SRB_FUNCTION_PNP<br/>注：此请求是否兼容Storport存疑，因为是作为SCSI port driver定义的结构 |
| **HwStorResetBus**                      | N/A                                                          | The **HwStorResetBus** routine is called by the port driver to clear error conditions. | support. <br/>其中实现card关电，host reset，cancel所有IO的操作 |
| **HwStorUnitControl**                   | **SCSI_UNIT_CONTROL_TYPE** enumeration contains unit control operations, where each control type initiates an action on a unit by the miniport driver. <br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ne-storport-scsi_unit_control_type | A miniport driver's **HwStorUnitControl** routine is called to perform synchronous operations to control the state of storage unit device. <br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_unit_control | Support.                                                     |
|                                         | ScsiQuerySupportedUnitControlTypes                           | Reports the unit-control operations implemented by the miniport driver. A miniport must support this control type. | Support ScsiQuerySupportedUnitControlTypes, ScsiUnitStart, ScsiUnitPower, ScsiUnitPoFxPowerInfo, ScsiUnitPoFxPowerActive, ScsiUnitPoFxPowerSetFState |
|                                         | ScsiUnitUsage                                                | Notifies the miniport whether the logical unit is used for any supported usage types. | Not support                                                  |
|                                         | ScsiUnitStart                                                | Notifies the miniport to start a unit device.                | support. 但实际什么都没干                                    |
|                                         | ScsiUnitPower                                                | Reports the unit power on or power off states.If the miniport supports this control type, it will not receive a storage request block with SRB_FUNCTION_POWER. | support. 但实际什么都没干                                    |
|                                         | ScsiUnitPoFxPowerInfo                                        | Notifies the miniport if idle power management is enabled or disabled on the unit component. The miniport should call [**StorPortInitializePoFxPower**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportinitializepofxpower) within this unit control if idle power management was enabled and if it supports runtime power management for the unit device. | support. 对Disk注册了Pofx(StorPortInitializePoFxPower). <br/>注：这里合理性存疑：Host Driver需要给Disk注册Pofx吗？ |
|                                         | ScsiUnitPoFxPowerRequired                                    | Notifies the miniport whether power is required for the unit component. Using [**STOR_POFX_POWER_REQUIRED_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_power_required_context) structure | Not support                                                  |
|                                         | ScsiUnitPoFxPowerActive                                      | Notifies the miniport that the unit component is either active or idle. Using [**STOR_POFX_ACTIVE_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_active_context) structure | support. 但实际什么都没干                                    |
|                                         | ScsiUnitPoFxPowerSetFState                                   | Notifies the miniport to set the unit component to the given functional power state (F-state). Using [**STOR_POFX_FSTATE_CONTEXT**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_fstate_context) structure | support. 但实际什么都没干                                    |
|                                         | ScsiUnitPoFxPowerControl                                     | Requests that the miniport execute a private power control operation that was initiated for the unit by a power engine plug-in (PEP). Using [**STOR_POFX_POWER_CONTROL**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_pofx_power_control) structure | Not support                                                  |
|                                         | ScsiUnitRemove                                               | Notifies the miniport that the unit has been removed.        | Not support                                                  |
|                                         | ScsiUnitSurpriseRemoval                                      | Notifies the miniport that the unit has been surprise-removed. | Not support                                                  |
|                                         | ScsiUnitRichDescription                                      | The miniport can choose to support this if the device reports a longer vendor ID, model number, or firmware revision than is defined in the SCSI spec | Not support                                                  |
|                                         | ScsiUnitQueryBusType                                         | Queries whether the miniport wants to specify a bus type for a given logical unit (LUN). Using [**STOR_UNIT_CONTROL_QUERY_BUS_TYPE**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-stor_unit_control_query_bus_type) structure. In Windows 10 version 21H1 and later, Storport sends this control only if a miniport has also previously called [**StorPortSetFeatureList**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetfeaturelist) in its [**HwFindAdapter**](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_find_adapter) routine with **StorportFeatureBusTypeUnitControl** specified. | Not support                                                  |
|                                         | ScsiUnitQueryFruId                                           | Queries the ID of a fault replacement unit (FRU). Available in Windows 10 version 21H1 and later. |                                                              |

| Storport Normal Routine that used by SD miniport driver. | Function Description                                         | SD miniport driver's purpose to use the routine              |
| -------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
|                                                          |                                                              |                                                              |
| StorPortWaitForSingleObject                              | A miniport can call **StorPortWaitForSingleObject** function to put the current thread into a wait state until the given dispatcher object is set to signaled state or optionally times out. dispatcher object (event, mutex, semaphore, thread, or timer)<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportwaitforsingleobject | 用于Storport routine和miniport thread的同步：storport routine发起dispatcher object(event), miniport thread使用StorPortWaitForSingleObject等待event信号产生后进入处理流程此时开始占用CPU；如果没有event，miniport thread将sleep，不会占用CPU. |
| StorPortInitializeEvent                                  | **StorPortInitializeEvent** initializes an event object as a synchronization or notification type event, and sets it to a signaled or not-signaled state.<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportinitializeevent | 初始化event事件对象，但不设置事件产生的信号。注意：如果没有初始化event就使用会产生空指针访问，系统会BSOD. |
| StorPortSetEvent                                         | A miniport can call **StorPortSetEvent** to set an event object to the signaled state.<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportsetevent | 设置事件产生的信号                                           |
| StorPortInitializeDpc                                    | The **StorPortInitializeDpc** routine initializes a StorPort DPC.<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportinitializedpc | 创建DPC对象：用于在高IRQL的Storport routine(IRQL >=中断级别)中设置延迟处理任务，目的是减少CPU处在高IRQL的时间，将不紧急的任务放到低IRQL(IRQL <= DPC)执行。这是Driver设计的基本要求。 |
| StorPortIssueDpc                                         | The **StorPortIssueDpc** routine issues a deferred procedure call (DPC).<br/>https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportissuedpc | 发起DPC任务：将延迟处理的任务加到DPC队列                     |
| StorPortCancelDpc                                        | **StorPortCancelDpc** attempts to cancel the execution of a StorPort deferred procedure call (DPC). | 销毁DPC对象：Driver卸载时清理资源                            |
| StorPortRequestTimer                                     | Schedules a callback event for a Storport timer context object. | 启动Timer：OS提供的定时器模块，miniport driver用于定时任务，例如设置指定disk idle时间后进入runtime D3. |
| StorPortFreeTimer                                        | Frees a Storport timer context object previously created by the [StorPortInitializeTimer](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportinitializetimer) routine. | 销毁Timer：Driver卸载时清理资源                              |
| StorPortStallExecution                                   | The **StorPortStallExecution** routine stalls the miniport driver. This call ties up a processor, doing no useful work while stalling in the driver. <br/> 参考：https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/srb/nf-srb-scsiportstallexecution | 使当前storport routine忙等待：miniport driver用于卸载driver时，等待miniport thread退出；如果不等待miniport thread退出就完成storport卸载routine，miniport创建的thread将会在卸载后继续运行，应用层会卸载报错。 |
| StorPortDelayExecution                                   | The **StorPortDelayExecution** function delays the current thread by the given amount of time, in microseconds. If the current IRQL is lower than DISPATCH_LEVEL then the current thread is simply put in the wait state and other threads are allowed to run. Otherwise, this routine performs a busy-wait. | miniport driver用于将当前miniport thread放到Sleep状态，释放CPU的占用，使其它thread（不管是storport还是miniport）可以运行。 |
| StorPortGetCurrentIrql                                   | **StorPortGetCurrentIrql** retrieves the current interrupt request level (IRQL). | miniport driver用于配合StorPortDelayExecution使用：如果当前IRQL高，Sleep实际是忙等. |
| StorPortAllocateRegistryBuffer                           | The **StorPortAllocateRegistryBuffer** routine is called by the miniport driver to allocate a buffer that can be used to read and write registry data. | 以下几个Registry Routine都是Windows Registry的访问操作，INF里面配置的Registry默认值通过这些Registry Routine去写入注册表，Driver动态功能开关是通过Registry Routine去读取注册表 |
| StorPortRegistryRead                                     | The **StorPortRegistryRead** routine reads the registry data for the indicated device and value. |                                                              |
| StorPortRegistryWrite                                    | The **StorPortRegistryWrite** routine is called by the miniport driver to convert the registry data contained in a specified buffer from ASCII to Unicode and to then write the data to the miniport driver's per-HBA storage area. |                                                              |
| StorPortFreeRegistryBuffer                               | The **StorPortFreeRegistryBuffer** routine frees the buffer that was allocated for storing registry data. |                                                              |
| StorPortNotification                                     | The miniport driver uses the **StorPortNotification** routine to notify the Storport driver of certain events and conditions. **StorPortNotification** takes a variable number of parameters depending on the notification type specified. <br/>NotificationType：https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nf-storport-storportnotification | miniport对storport的反向通知机制                             |
|                                                          | BusChangeDetected: Indicates that a target device might have been added or removed from a dynamic bus. | miniport driver用于：当disk有插拔或者card模式切换，都会发BusChangeDetected去通知OS |
|                                                          | RequestComplete: Indicates that the given SRB has finished. After this notification is sent, the port driver owns the request. | miniport driver用于返回Storport的每一个SRB请求，包括inquiry查询和i/O读写等各类收到的SRB请求。 |
|                                                          | ResetDetected: Indicates that the HBA has detected a reset on the bus. After this notification is sent, the miniport driver is still responsible for completing any active requests. The port driver will manage all required bus-reset delays. | miniport driver没使用此notification类型                      |
|                                                          | RequestTimerCall: Indicates that the miniport driver requires the port driver to call the miniport driver's [HwStorTimer](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/nc-storport-hw_timer) routine in the requested number of microseconds. | miniport driver用于在Storport routine里面，将一部分任务放到指定时间后开始执行，相当于一种不精确的DPC. |
|                                                          | IoTargetRequestServiceTime: Indicates to Storport the amount of time that was required to process a specified request. | miniport driver用于指定tagIO任务的完成时间，让Storport driver有个预期不要误判请求完成超时。 |
| StorPortGetBusData                                       | The **StorPortGetBusData** routine retrieves the bus-specific configuration information necessary to initialize the HBA. | miniport driver用于获取Bus(PCIe Bus)的数据，拿到SD host device在PCIe Bus的信息。例如VendorID，DeviceID，区分SD host类型。 |
| StorPortGetUncachedExtension                             | The **StorPortGetUncachedExtension** routine allocates an uncached common buffer to be shared by the CPU and the device. | 在HwStorFindAdapter Callback创建DMA Buffer时使用             |
| StorPortGetPhysicalAddress                               | The **StorPortGetPhysicalAddress** routine converts a given virtual address range to a physical address range for a DMA operation. | 在HwStorFindAdapter Callback创建DMA Buffer时使用             |
| StorPortInitialize                                       | The **StorPortInitialize** routine initializes the port driver parameters and extension data. **StorPortInitialize** also saves the adapter information provided from the [miniport driver](https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/storage-miniport-drivers) **DriverEntry** routine. | miniport driver将Storport指定的HW_INITIALIZATION_DATA全部初始化完成后调用，上报miniport的所有能力配置和回调函数。 |
| StorPortReadRegisterBuffer                               | reads a value from a specified register address.             | miniport driver用于register读（包括SD host register和PCIe Config register，register address由PCIe bar address + offset指定） |
| StorPortWriteRegisterBuffer                              | transfers a given number of unsigned bytes from a buffer to the HBA. | miniport driver用于register写（包括SD host register和PCIe Config register，register address由PCIe bar address + offset指定） |
| StorPortWritePort                                        | transfers an unsigned byte to the HBA                        | miniport driver没有使用，只遗留历史代码                      |
| StorPortReadPort                                         | reads a value from a specified port address                  | miniport driver没有使用，只遗留历史代码                      |
| StorPortInitializePoFxPower                              | A miniport driver calls **StorPortInitializePoFxPower** to register a storage device with the power management framework (PoFx). Using [STOR_POFX_DEVICE](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/storport/ns-storport-_stor_pofx_device), **STOR_POFX_DEVICE_V2**, or **STOR_POFX_DEVICE_V3** | miniport driver用于注册Adapter和Disk Unit的pofx              |
| StorPortPoFxIdleComponent                                | The **StorPortPoFxIdleComponent** routine decrements the activation reference count of a specified component of a storage device. | 在remove card， auto power off时调用                         |
| StorPortPoFxActivateComponent                            | The **StorPortPoFxActivateComponent** routine increments the activation reference count on the specified component of a storage device. | 在insert card调用                                            |
| StorPortGetD3ColdSupport                                 | Get D3Cold support or not by this HBA                        | StorPortInitializePoFxPower其中一个参数是D3Cold enable or not |
| StorPortGetDeviceObjects                                 | Returns the device objects that are associated with the adapter device stack | 处理PNP请求用到，根据SD host extension获取*AdapterDeviceObject*和*PhysicalDeviceObject*和LowerDeviceObject |

## 从Storport DDI到KMDF DDI

doing.
