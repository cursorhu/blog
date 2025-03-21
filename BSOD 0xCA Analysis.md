# BSOD 0xCA Analysis.

根据 Windbg Dump 分析，这个 BSOD 是由 IOMMU 操作失败导致的。具体来看：

错误发生在设备初始化阶段：

```
DevNode State = DeviceNodeResourcesAssigned (0x306)
Previous State = DeviceNodeDriversAdded (0x305)
```

关键问题点：

```
BUGCHECK_CODE: ca (PNP_DETECTED_FATAL_ERROR)
Arg1: 13 (IOMMU operation failure)
Arg2: 1000 (Unblock operation)
```

解决 IOMMU 操作失败的问题。主要通过：

1. 正确初始化 DMA/IOMMU 映射（当前代码已支持，不需修改）

2. 确保状态转换时 DMA 操作安全完成 （当前代码待修改）

代码改动：

设备进入D3/D0以前Complete所有I/O请求（SRB），使用Storport miniport要求的API RequestComplete ： We do not recommend that writers of Storport miniport drivers use this particular Storport interface routine. Instead, the miniport driver should call StorPortNotification( RequestComplete ) for each outstanding request.

![image-20241105201725563](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202411052017667.png)

附录1：BSOD 0xCA的相关文档

https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0xca--pnp-detected-fatal-error?redirectedfrom=MSDN

https://www.sysnative.com/forums/threads/debugging-stop-0xca-dddriver-sys-dddriver64dcsa-sys.35039/

附录2：BSOD 0xCA的Windbg分析log

```
!analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

PNP_DETECTED_FATAL_ERROR (ca)
PnP encountered a severe error, either as a result of a problem in a driver or
a problem in PnP itself.  The first argument describes the nature of the
problem, the second argument is the address of the PDO.  The other arguments
vary depending on argument 1.
Arguments:
Arg1: 0000000000000013, IOMMU operation failure
	A critical IOMMU operation has failed.
Arg2: 0000000000001000, Unblock operation
Arg3: ffffffffc0350066, NT status code.
Arg4: ffffb986629afc40, DevNode of the device.

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 4546

    Key  : Analysis.Elapsed.mSec
    Value: 4604

    Key  : Analysis.IO.Other.Mb
    Value: 17

    Key  : Analysis.IO.Read.Mb
    Value: 2

    Key  : Analysis.IO.Write.Mb
    Value: 24

    Key  : Analysis.Init.CPU.mSec
    Value: 968

    Key  : Analysis.Init.Elapsed.mSec
    Value: 4411983

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 92

    Key  : Analysis.Version.DbgEng
    Value: 10.0.27725.1000

    Key  : Analysis.Version.Description
    Value: 10.2408.27.01 amd64fre

    Key  : Analysis.Version.Ext
    Value: 1.2408.27.1

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xca

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xca

    Key  : Bugcheck.Code.TargetModel
    Value: 0xca

    Key  : Dump.Attributes.AsUlong
    Value: 21800

    Key  : Dump.Attributes.DiagDataWrittenToHeader
    Value: 1

    Key  : Dump.Attributes.ErrorCode
    Value: 0

    Key  : Dump.Attributes.LastLine
    Value: Dump completed successfully.

    Key  : Dump.Attributes.ProgressPercentage
    Value: 100

    Key  : Failure.Bucket
    Value: 0xCA_13_nt!PiDmaGuardProcessPreStart

    Key  : Failure.Hash
    Value: {b367b2d8-0cc5-f3e0-e733-3787841dcdd2}

    Key  : Hypervisor.Enlightenments.ValueHex
    Value: 7497cf94

    Key  : Hypervisor.Flags.AnyHypervisorPresent
    Value: 1

    Key  : Hypervisor.Flags.ApicEnlightened
    Value: 1

    Key  : Hypervisor.Flags.ApicVirtualizationAvailable
    Value: 0

    Key  : Hypervisor.Flags.AsyncMemoryHint
    Value: 0

    Key  : Hypervisor.Flags.CoreSchedulerRequested
    Value: 0

    Key  : Hypervisor.Flags.CpuManager
    Value: 1

    Key  : Hypervisor.Flags.DeprecateAutoEoi
    Value: 0

    Key  : Hypervisor.Flags.DynamicCpuDisabled
    Value: 1

    Key  : Hypervisor.Flags.Epf
    Value: 0

    Key  : Hypervisor.Flags.ExtendedProcessorMasks
    Value: 1

    Key  : Hypervisor.Flags.HardwareMbecAvailable
    Value: 1

    Key  : Hypervisor.Flags.MaxBankNumber
    Value: 0

    Key  : Hypervisor.Flags.MemoryZeroingControl
    Value: 0

    Key  : Hypervisor.Flags.NoExtendedRangeFlush
    Value: 0

    Key  : Hypervisor.Flags.NoNonArchCoreSharing
    Value: 1

    Key  : Hypervisor.Flags.Phase0InitDone
    Value: 1

    Key  : Hypervisor.Flags.PowerSchedulerQos
    Value: 0

    Key  : Hypervisor.Flags.RootScheduler
    Value: 0

    Key  : Hypervisor.Flags.SynicAvailable
    Value: 1

    Key  : Hypervisor.Flags.UseQpcBias
    Value: 0

    Key  : Hypervisor.Flags.Value
    Value: 38408431

    Key  : Hypervisor.Flags.ValueHex
    Value: 24a10ef

    Key  : Hypervisor.Flags.VpAssistPage
    Value: 1

    Key  : Hypervisor.Flags.VsmAvailable
    Value: 1

    Key  : Hypervisor.RootFlags.AccessStats
    Value: 1

    Key  : Hypervisor.RootFlags.CrashdumpEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.CreateVirtualProcessor
    Value: 1

    Key  : Hypervisor.RootFlags.DisableHyperthreading
    Value: 0

    Key  : Hypervisor.RootFlags.HostTimelineSync
    Value: 1

    Key  : Hypervisor.RootFlags.HypervisorDebuggingEnabled
    Value: 0

    Key  : Hypervisor.RootFlags.IsHyperV
    Value: 1

    Key  : Hypervisor.RootFlags.LivedumpEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.MapDeviceInterrupt
    Value: 1

    Key  : Hypervisor.RootFlags.MceEnlightened
    Value: 1

    Key  : Hypervisor.RootFlags.Nested
    Value: 0

    Key  : Hypervisor.RootFlags.StartLogicalProcessor
    Value: 1

    Key  : Hypervisor.RootFlags.Value
    Value: 1015

    Key  : Hypervisor.RootFlags.ValueHex
    Value: 3f7

    Key  : SecureKernel.HalpHvciEnabled
    Value: 1

    Key  : WER.OS.Branch
    Value: ge_release

    Key  : WER.OS.Version
    Value: 10.0.26100.1


BUGCHECK_CODE:  ca

BUGCHECK_P1: 13

BUGCHECK_P2: 1000

BUGCHECK_P3: ffffffffc0350066

BUGCHECK_P4: ffffb986629afc40

FILE_IN_CAB:  MEMORY.DMP

TAG_NOT_DEFINED_202b:  *** Unknown TAG in analysis list 202b


DUMP_FILE_ATTRIBUTES: 0x21800

FAULTING_THREAD:  ffffb986625ef040

DEVICE_OBJECT: 0000000000001000

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

LOCK_ADDRESS:  fffff8019cd8a380 -- (!locks fffff8019cd8a380)
KD: Scanning for held locks........................................................

Resource @ nt!PiEngineLock (0xfffff8019cd8a380)    Exclusively owned
    Contention Count = 23
     Threads: ffffb986625ef040-01<*> 
1 total locks

PNP_TRIAGE_DATA: 
	Lock address  : 0xfffff8019cd8a380
	Thread Count  : 1
	Thread address: 0xffffb986625ef040
	Thread wait   : 0x281e98

STACK_TEXT:  
ffffdf84`8a6b7158 fffff801`9c8e5bca     : 00000000`000000ca 00000000`00000013 00000000`00001000 ffffffff`c0350066 : nt!KeBugCheckEx
ffffdf84`8a6b7160 fffff801`9c7db310     : ffffb986`629afc40 00000000`00000000 00000000`00000001 ffffb986`629afc40 : nt!PiDmaGuardProcessPreStart+0x10a7f6
ffffdf84`8a6b71a0 fffff801`9c697621     : ffffb986`629afc40 ffffdf84`8a6b7261 00000000`00000000 00000000`00000001 : nt!PipProcessStartPhase1+0x4c
ffffdf84`8a6b71e0 fffff801`9c8187a7     : ffffb986`36694b20 ffffb986`626da790 ffffdf84`8a6b7300 fffff801`00000002 : nt!PipProcessDevNodeTree+0x645
ffffdf84`8a6b72b0 fffff801`9c2404bd     : 00000001`00000003 ffffb986`36694b20 ffffb986`626da790 00000000`00000000 : nt!PiProcessReenumeration+0x9f
ffffdf84`8a6b7300 fffff801`9c1249d2     : ffffb986`625ef040 ffffb986`366b6cb0 fffff801`9c23fe80 ffffb986`00000000 : nt!PnpDeviceActionWorker+0x63d
ffffdf84`8a6b73c0 fffff801`9c25a9ea     : ffffb986`625ef040 ffffb986`625ef040 fffff801`9c124820 ffffb986`366b6cb0 : nt!ExpWorkerThread+0x1b2
ffffdf84`8a6b7570 fffff801`9c4736f4     : ffffce81`b4759180 ffffb986`625ef040 fffff801`9c25a990 00320033`006d0065 : nt!PspSystemThreadStartup+0x5a
ffffdf84`8a6b75c0 00000000`00000000     : ffffdf84`8a6b8000 ffffdf84`8a6b1000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  nt!PiDmaGuardProcessPreStart+10a7f6

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

STACK_COMMAND:  .process /r /p 0xffffb98636697040; .thread 0xffffb986625ef040 ; kb

BUCKET_ID_FUNC_OFFSET:  10a7f6

FAILURE_BUCKET_ID:  0xCA_13_nt!PiDmaGuardProcessPreStart

OS_VERSION:  10.0.26100.1

BUILDLAB_STR:  ge_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {b367b2d8-0cc5-f3e0-e733-3787841dcdd2}

Followup:     MachineOwner
---------

1: kd> !devnode ffffb986629afc40
DevNode 0xffffb986629afc40 for PDO 0xffffb98665085060
  Parent 0xffffb986629adc40   Sibling 0000000000   Child 0000000000   
  InstancePath is "PCI\VEN_1217&DEV_8621&SUBSYS_00021217&REV_01\4&32cd076f&0&0013"
  ServiceName is "bhtsddr"
  State = DeviceNodeResourcesAssigned (0x306) @ 2024 Oct 24 21:26:21.411
  Previous State = DeviceNodeDriversAdded (0x305) @ 2024 Oct 24 21:26:21.411
  StateHistory[02] = DeviceNodeDriversAdded (0x305)
  StateHistory[01] = DeviceNodeInitialized (0x304)
  StateHistory[00] = DeviceNodeUninitialized (0x301)
  StateHistory[19] = Unknown State (0x0)
  StateHistory[18] = Unknown State (0x0)
  StateHistory[17] = Unknown State (0x0)
  StateHistory[16] = Unknown State (0x0)
  StateHistory[15] = Unknown State (0x0)
  StateHistory[14] = Unknown State (0x0)
  StateHistory[13] = Unknown State (0x0)
  StateHistory[12] = Unknown State (0x0)
  StateHistory[11] = Unknown State (0x0)
  StateHistory[10] = Unknown State (0x0)
  StateHistory[09] = Unknown State (0x0)
  StateHistory[08] = Unknown State (0x0)
  StateHistory[07] = Unknown State (0x0)
  StateHistory[06] = Unknown State (0x0)
  StateHistory[05] = Unknown State (0x0)
  StateHistory[04] = Unknown State (0x0)
  StateHistory[03] = Unknown State (0x0)
  Flags (0x6c0000f0)  DNF_ENUMERATED, DNF_IDS_QUERIED, 
                      DNF_HAS_BOOT_CONFIG, DNF_BOOT_CONFIG_RESERVED, 
                      DNF_NO_LOWER_DEVICE_FILTERS, DNF_NO_LOWER_CLASS_FILTERS, 
                      DNF_NO_UPPER_DEVICE_FILTERS, DNF_NO_UPPER_CLASS_FILTERS
  CapabilityFlags (0x00002000)  WakeFromD3
```

