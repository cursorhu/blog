# Windows Driver -- INF Verifier使用笔记

WDK有INF verifier用于检测Driver安装包的INF信息文件的内容是否符合要求：如果INF不符合要求，可能在安装Driver报错或者Driver运行时功能报错。

## INF verifier使用

```
.\infverif.exe /w /v C:\path\driver.inf
```

https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/running-infverif-from-the-command-line

## 排错示例

新建WDF驱动项目时产生默认的INF，但其中一些符号需要替换，否则安装driver时会直接报错安装失败

1. DIRID 13问题

driver.sys一般的路径符号 DIRID是12，较新的OS WDK要求DIRID使用13，用于实现driver package isolation：

参考：https://learn.microsoft.com/en-us/windows-hardware/drivers/develop/porting-inf-to-windows-driver

错误信息：

```
PS C:\Program Files (x86)\Windows Kits\10\Tools\10.0.26100.0\x64>
ARM64\Debug\O2SD\O2SD.inf
ERROR(1322) in C:\Users\cursorhu\source\repos\O2SD\ARM64\Debug\O2SD\O2SD.inf, line 49: Destination file path 'C:\Windows\System32\drivers' for file 'O2SD.sys' is not isolated to DIRID 13.
```

解决(git diff)：设置DestinationDirs = 13，并设置SourceDisksNames的TargetOSVersion

```
 [DestinationDirs]
-DefaultDestDir = 12 ;DIRID_DRIVERS; 
+DefaultDestDir = 13 ;DIRID_DRIVERS; %13% only supported since OS build 16299

[SourceDisksNames]
...
+%ManufacturerName% = Generic,NTarm64.10.0...26100 ; with TargetOSVersion

+[Generic.NTarm64.10.0...26100]
 ServiceType    = 1               ; SERVICE_KERNEL_DRIVER
 StartType      = 3               ; SERVICE_DEMAND_START
 ErrorControl   = 1               ; SERVICE_ERROR_NORMAL
-ServiceBinary  = %12%\O2SD.sys
+ServiceBinary  = %13%\O2SD.sys  ; DestinationDirs
```

2. 变量未定义问题

错误信息：

```
报错1：Unresolved $ARCH$ token for section [generic.nt$arch$]. Must run stampinf tool to resolve case sensitive $ARCH$ tokens.

报错2：KmdfLibraryVersion directive has invalid value "$KmdfLibraryVersion"
```

解决(git diff)：根据平台环境指定变量值

```
-%ManufacturerName% = Generic,NT$ARCH$
+%ManufacturerName% = Generic,NTarm64.10.0...26100 

-[Generic.NT$ARCH$]
+[Generic.NTarm64.10.0...26100]

-KmdfLibraryVersion = $KMDFVERSION$
+KmdfLibraryVersion = 1.33
```

全部错误修复后，显示如下：INF is VALID

```
.\infverif.exe /w /v C:\Users\cursorhu\source\repos\O2SD\O2SD.inf
Running in Verbose
Running Windows Driver INF check

Validating O2SD.inf
INF is VALID
```

