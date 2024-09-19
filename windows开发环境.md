# windows-PC开发环境

## 多桌面和分屏

Win+Ctrl+D， 新建桌面窗口

Win+Ctrl + ←/→，切换桌面窗口

win+Tab， 任务视图，常用操作：拖动程序到指定窗口

win键+←/→，快速分屏成两列

win键 + ←/→ + ↑/↓，快速4分屏

注意：多个桌面的相同应用是独立的，例如某桌面打开chrome很多页面，其他桌面打开chrome会是新网页

## 区分Linux文件大小写

需要4个条件：

Windows 10 1803以上版本
启用 Linux 子系统，即 Windows Subsystem for Linux
所在分区为 NTFS 格式
以管理员权限运行 PowerShell

必须启用WSL环境。powershell管理员运行：

```
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

再将指定目录启用大小写 

```
C:\Windows\system32> fsutil.exe file setCaseSensitiveInfo C:\github enable
已启用目录 C:\github 的区分大小写属性。
```

之后解压linux kernel就不会报错同名大小写文件重复。

## 未激活Windows去水印和关闭WindowsUpdate和桌面壁纸

win+R regedit设置Start值为4，去桌面水印

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\sppsvc -> Start -> 4
```

永久关闭WindowsUpdate，否则下次Update之后又有水印

```
services.msc -> Windows Update -> 
(1)常规 -> 禁用
(2)恢复 -> 第一次失败 -> 无操作, 重置计数 -> 9999
```

更换壁纸：下载图片直接右键设置为壁纸