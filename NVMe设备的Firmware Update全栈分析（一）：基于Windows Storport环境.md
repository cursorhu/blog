---
title: NVMe设备的Firmware Update全栈分析（一）：基于Windows Storport环境
date: 2022-11-30 17:41:00
tags: NVMe
categories: NVMe
---





## 1.1 Windows Storport Driver环境下的NVMe设备Firmware Update

Windows系统下，NVMe设备的Firmware Update都是基于以下Microsoft API文档 ：

[upgrading-firmware-for-an-nvme-device](https://learn.microsoft.com/en-us/windows-hardware/drivers/storage/upgrading-firmware-for-an-nvme-device)