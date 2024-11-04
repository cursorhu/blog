---
title: Linux PCI driver分析与调试记录
date: 2023-08-25 11:56:14
tags: PCI
categories: linux

---



### PCI tree结构

关于PCIe tree的bus/device的详细architecture，参考LDD3和Mastering Linux Device Driver Development - John Madieu

![image-20230829111444515](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291114824.png)

![image-20230829111727760](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291117111.png)

> Root complex (RC): This refers to the PCIe host controller in the SoC. It can access the main memory without CPU intervening, which is a feature used by other devices to access the main memory. They are also known as Host-to-PCI bridges.
>
> Bridges: These provide an interface to other buses, such as PCI or PCI X, or even another PCIe bus. A bridge can also provide an interface to the same bus
>
> Endpoint (EP): Endpoints are PCIe devices, and are represented by type 00h configuration space headers. They never appear on a switch's internal bus and have no downstream port



### lspci使用示例

下面介绍如何找到一个PCI(e)设备的信息，及其上游端口信息，以及设备的register space内容

(1)查看所有pci设备：

```
cursorhu@ubuntu-PC:~$ lspci
00:00.0 Host bridge: Intel Corporation 11th Gen Core Processor Host Bridge/DRAM Registers (rev 01)
00:02.0 VGA compatible controller: Intel Corporation TigerLake-LP GT2 [Iris Xe Graphics] (rev 01)
00:04.0 Signal processing controller: Intel Corporation TigerLake-LP Dynamic Tuning Processor Participant (rev 01)
00:05.0 Multimedia controller: Intel Corporation Device 9a19 (rev 01)
00:07.0 PCI bridge: Intel Corporation Tiger Lake-LP Thunderbolt 4 PCI Express Root Port #0 (rev 01)
00:07.1 PCI bridge: Intel Corporation Tiger Lake-LP Thunderbolt 4 PCI Express Root Port #1 (rev 01)
00:07.2 PCI bridge: Intel Corporation Tiger Lake-LP Thunderbolt 4 PCI Express Root Port #2 (rev 01)
00:07.3 PCI bridge: Intel Corporation Tiger Lake-LP Thunderbolt 4 PCI Express Root Port #3 (rev 01)
...
00:1c.0 PCI bridge: Intel Corporation Tiger Lake-LP PCI Express Root Port #8 (rev 20)
...
a9:00.0 Non-Volatile memory controller: Device 0012:1cc1
```

(2)查看PCI设备上下游信息

下面关注PCIe device a9:00.0, 用 lspci -v (verbose)查看详细信息:

是一个nvme设备，使用的driver是nvme；device id是0012:1cc1

```
cursorhu@ubuntu-PC:~$ lspci -v
a9:00.0 Non-Volatile memory controller: Device 0012:1cc1 (prog-if 02 [NVM Express])
	Subsystem: Device 3456:5344
	Physical Slot: 7
	Flags: bus master, fast devsel, latency 0, IRQ 19, NUMA node 0, IOMMU group 20
	Memory at 88c00000 (64-bit, non-prefetchable) [size=16K]
	Capabilities: <access denied>
	Kernel driver in use: nvme
	Kernel modules: nvme
```

查看这个设备的上游信息，包括它所在的PCIe bridge（即PCIe port，端口也是PCI设备）

如下lspci -t (tree)列出PCI tree，其中[]内的是bus号，-xx.x是设备号

```
cursorhu@ubuntu-PC:~$ lspci -t
-[0000:00]-+-00.0
           +-02.0
           +-04.0
           +-05.0
           +-07.0-[01-2a]--
           +-07.1-[2b-54]--
           +-07.2-[55-7e]--
           +-07.3-[7f-a8]--
          ....
           +-1c.0-[a9]----00.0
           +-1d.0-[aa]--
          ....
           \-1f.6
```

我们关注的设备a9:00.0表示该设备的 bus是a9, device是00.0，对应PCI tree的 [a9]----00.0

其上游设备是1c.0，完整设备号为00:1c.0，是个PCIe bridge；每个bridge都是PCIe设备，只不过它比较特殊，是连接其他PCIe设备的设备。根据lspci信息查看该PCIe bridge为：

```
00:1c.0 PCI bridge: Intel Corporation Tiger Lake-LP PCI Express Root Port #8 (rev 20)
```

PCIe bridge的更上游即bus 00，是PCIe RC (root complex)

上图中有的bridge可以支持一定范围的bus号，例如bus范围为01-2a：

```
+-07.0-[01-2a]--
```

(3)查看PCI设备的register space

使用`lspci -s [bus]:[device].[function] -xxxx` 查看完整的PCIe register space(需要root权限)， -s: show, `lspci --help`查看各选项

```
cursorhu@ubuntu-PC:~$ sudo lspci -s 00:1c.0 -xxxx (或者-xxx，显示00~ff)
[sudo] password for cursorhu: 
00:1c.0 PCI bridge: Intel Corporation Tiger Lake-LP PCI Express Root Port #8 (rev 20)
00: 86 80 bf a0 07 04 10 00 20 00 04 06 00 00 81 00
10: 00 00 00 00 00 00 00 00 00 a9 a9 00 40 40 00 20
20: c0 88 50 89 01 7e 91 7e 60 00 00 00 60 00 00 00
30: 00 00 00 00 40 00 00 00 00 00 00 00 ff 04 02 00
40: 10 80 42 01 01 80 00 00 20 00 11 00 13 4c 72 08
50: 42 00 13 70 60 b2 3c 00 08 10 40 00 08 00 00 00
60: 00 00 00 00 37 08 00 00 00 04 00 00 0e 00 00 00
70: 03 00 1f 00 00 00 00 00 00 00 00 00 00 00 00 00
80: 05 90 01 00 78 02 e0 fe 00 00 00 00 00 00 00 00
90: 0d a0 00 00 00 00 00 00 00 00 00 00 00 00 00 00
a0: 01 00 03 c8 00 00 00 00 00 00 00 00 00 00 00 00
b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
d0: 11 10 00 07 42 18 01 40 0a 00 9e 09 00 00 00 00
e0: 00 03 e3 00 03 90 03 90 16 00 10 00 00 00 00 00
f0: 50 01 00 00 00 00 00 4c b5 0f 21 01 04 00 00 84
100: 01 00 01 22 00 00 00 00 00 40 00 00 11 00 06 00
110: 01 20 00 00 00 20 00 00 00 00 00 00 00 00 00 00
120: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
...
```

我们要确认PCI bridge的 link status register(位于capability register offset 0x12, bit 13 Link active bit) 以知道 PCIe link 是否 active. 
如何查看：如下图 0x00 ~ 0x3C 是config space标准空间；0x34 capability pointer是地址，指向capability register空间，其值为0x40，因此capability register空间是从0x40开始；因此 link status regsiter 是 0x40+0x12 = 0x52, 其bit13 即 0x53 的 bit5. 

详细register mapping参考PCI Express Base Spec.

![image-20230829112915119](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291129433.png)

### PCI device的创建过程

参考LDD3 LDM(Linux Device Model)

PCI device包括PCI driver, PCI core driver, Kobject三个层次，并在用户层sysfs反映device和driver。

![image-20230830170650433](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308301706671.png)

### 使用sysfs操作pci device

**(1)使用/sys/bus/pci文件接口对device操作：**

remove设备：

```
echo 1 > /sys/bus/pci/devices/AAAA:BB:CC.D/remove
```

AAAA:BB:CC.D为bus-info，分别为[Domain:Bus:Device.Function](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci)

rescan设备

```
echo 1 > /sys/bus/pci/rescan
```

reset设备

```
echo 1 > /sys/bus/pci/devices/AAAA:BB:CC.D/reset
```

**(2) lspci & dmesg & 源码 分析：**

首先lspci设备，以设备03:00.0 (SD Host controller)为例

```
cursorhu@ubuntu-PC:~/linuxkernel/linux-6.2$ lspci
...
00:1c.7 PCI bridge: Intel Corporation Cannon Lake PCH PCI Express Root Port #8 (rev f0)
...
03:00.0 SD Host controller: O2 Micro, Inc. Device 9862 (rev 01)
```

remove设备：

非root用户执行时需要用sudo sh -c "命令内容"

```
sudo sh -c "echo 1 > /sys/bus/pci/devices/0000:03:00.0/remove"
```

dmesg显示remove调用了该device driver的 .remove

```
[ 241.962185] BHT-OSENTRY: BHT sd remove begin
```

remove之后`lspci`看不到设备03:00.0，`lspci -t`可见其bus号03下挂的设备为空

```
cursorhu@ubuntu-PC:~/linuxkernel/linux-6.2$ lspci -t
-[0000:00]-+-00.0
           +-02.0
           +-12.0
           +-14.0
           +-14.2
           +-16.0
           +-17.0
           +-1b.0-[01]--
           +-1c.0-[02]--
           +-1c.7-[03]--
           +-1d.0-[04]--
           +-1f.0
           +-1f.3
           +-1f.4
           +-1f.5
           \-1f.6
```

rescan设备：

```
sudo sh -c "echo 1 > /sys/bus/pci/rescan"
```

再`lspci -t`可见其bus号03下挂的设备为为00.0，即设备的[bus:device.function]为03:00.0

```
-[0000:00]-+-00.0
           +-02.0
           +-12.0
           +-14.0
           +-14.2
           +-16.0
           +-17.0
           +-1b.0-[01]--
           +-1c.0-[02]--
           +-1c.7-[03]----00.0
           +-1d.0-[04]--
           +-1f.0
           +-1f.3
           +-1f.4
           +-1f.5
           \-1f.6
```

dmesg显示rescan调用了该device driver的 .probe

```
[ 617.171735] BHT-OSENTRY: BHT sd probe begin
```

reset设备：

备份设备的config register 0x0~0x3C，然后调用[pci_dev_restore](https://elixir.bootlin.com/linux/latest/source/drivers/pci/pci.c#L5555):

```
* This function does not just reset the PCI portion of a device, but
 * clears all the state associated with the device.
```

```
[ 1116.993784] bht-sd 0000:03:00.0: saving config space at offset 0x0 (reading 0x98621217)
[ 1116.993790] bht-sd 0000:03:00.0: saving config space at offset 0x4 (reading 0x100406)
[ 1116.993793] bht-sd 0000:03:00.0: saving config space at offset 0x8 (reading 0x8050101)
[ 1116.993796] bht-sd 0000:03:00.0: saving config space at offset 0xc (reading 0x10)
[ 1116.993798] bht-sd 0000:03:00.0: saving config space at offset 0x10 (reading 0x51100000)
[ 1116.993801] bht-sd 0000:03:00.0: saving config space at offset 0x14 (reading 0x51101000)
[ 1116.993804] bht-sd 0000:03:00.0: saving config space at offset 0x18 (reading 0x0)
[ 1116.993806] bht-sd 0000:03:00.0: saving config space at offset 0x1c (reading 0x0)
[ 1116.993809] bht-sd 0000:03:00.0: saving config space at offset 0x20 (reading 0x0)
[ 1116.993811] bht-sd 0000:03:00.0: saving config space at offset 0x24 (reading 0x0)
[ 1116.993813] bht-sd 0000:03:00.0: saving config space at offset 0x28 (reading 0x0)
[ 1116.993816] bht-sd 0000:03:00.0: saving config space at offset 0x2c (reading 0x21217)
[ 1116.993818] bht-sd 0000:03:00.0: saving config space at offset 0x30 (reading 0x0)
[ 1116.993821] bht-sd 0000:03:00.0: saving config space at offset 0x34 (reading 0x6c)
[ 1116.993823] bht-sd 0000:03:00.0: saving config space at offset 0x38 (reading 0x0)
[ 1116.993826] bht-sd 0000:03:00.0: saving config space at offset 0x3c (reading 0x10b)
[ 1118.015083] pcieport 0000:00:1c.7: re-enabling LTR
[ 1118.015133] bht-sd 0000:03:00.0: restoring config space at offset 0x3c (was 0x100, writing 0x10b)
[ 1118.015170] bht-sd 0000:03:00.0: restoring config space at offset 0x14 (was 0x0, writing 0x51101000)
[ 1118.015191] bht-sd 0000:03:00.0: restoring config space at offset 0x10 (was 0x0, writing 0x51100000)
[ 1118.015205] bht-sd 0000:03:00.0: restoring config space at offset 0xc (was 0x0, writing 0x10)
[ 1118.015227] bht-sd 0000:03:00.0: restoring config space at offset 0x4 (was 0x100000, writing 0x100406)
```



### PCI driver的register access API分析

API在LDD3的PCI driver一章已经有较详细说明：

**Linux offers a standard interface to access the configuration space.*
*As far as the driver is concerned, the configuration space can be accessed through 8-*
*bit, 16-bit, or 32-bit data transfers. The relevant functions are prototyped in <linux/*
pci.h>*  

```
int pci_read_config_byte(struct pci_dev *dev, int where, u8 *val);
int pci_read_config_word(struct pci_dev *dev, int where, u16 *val);
int pci_read_config_dword(struct pci_dev *dev, int where, u32 *val);
int pci_write_config_byte(struct pci_dev *dev, int where, u8 val);
int pci_write_config_word(struct pci_dev *dev, int where, u16 val);
int pci_write_config_dword(struct pci_dev *dev, int where, u32 val);
```

内部实现实际是pci_bus_read_config_word，参考：[access.c#L554](https://elixir.bootlin.com/linux/latest/source/drivers/pci/access.c#L554)

```
int pci_read_config_word(const struct pci_dev *dev, int where, u16 *val)
{
	if (pci_dev_is_disconnected(dev)) {
		PCI_SET_ERROR_RESPONSE(val);
		return PCIBIOS_DEVICE_NOT_FOUND;
	}
	return pci_bus_read_config_word(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_read_config_word);
```

pci_bus_read_config_word的内部实现LDD3也说了：

**The actual implementation of pci_read_config_byte(dev, where, val), for instance, expands to:*
dev->bus->ops->read(bus, devfn, where, 8, val);*  

其中bus->ops使用pci_ops结构：

```
struct pci_ops {
int (*read)(struct pci_bus *bus, unsigned int devfn, int where, int size,
u32 *val);
int (*write)(struct pci_bus *bus, unsigned int devfn, int where, int size,
u32 val);
};
```

pci_read_config_byte的实现是宏函数，使用##连接符动态定义byte, word, dword，因此直接搜索不到，实际还是在[access.c#L74](https://elixir.bootlin.com/linux/latest/source/drivers/pci/access.c#L74)，就在EXPORT_SYMBOL(pci_bus_read_config_word); 的前面定义:

```
#define PCI_OP_READ(size, type, len) \
int noinline pci_bus_read_config_##size \
	(struct pci_bus *bus, unsigned int devfn, int pos, type *value)	\
{									\
	int res;							\
	unsigned long flags;						\
	u32 data = 0;							\
	if (PCI_##size##_BAD) return PCIBIOS_BAD_REGISTER_NUMBER;	\
	pci_lock_config(flags);						\
	res = bus->ops->read(bus, devfn, pos, len, &data);		\
	if (res)							\
		PCI_SET_ERROR_RESPONSE(value);				\
	else								\
		*value = (type)data;					\
	pci_unlock_config(flags);					\
	return res;							\
}
```

那么bus->ops->read的底层实现到底是什么？取决于cpu和pci架构，例如arm/x86/ia64...

全局搜索定义struct pci_ops的代码即可见，以[x86为例](https://elixir.bootlin.com/linux/latest/source/arch/x86/pci/common.c#L72)

```
struct pci_ops pci_root_ops = {
	.read = pci_read,
	.write = pci_write,
};

static int pci_read(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 *value)
{
	return raw_pci_read(pci_domain_nr(bus), bus->number,
				 devfn, where, size, value);
}

int raw_pci_read(unsigned int domain, unsigned int bus, unsigned int devfn,
						int reg, int len, u32 *val)
{
	if (domain == 0 && reg < 256 && raw_pci_ops)
		return raw_pci_ops->read(domain, bus, devfn, reg, len, val);
	if (raw_pci_ext_ops)
		return raw_pci_ext_ops->read(domain, bus, devfn, reg, len, val);
	return -EINVAL;
}
```

*struct* pci_raw_ops有几处定义，direct.c, mmconfig_32.c, mmconfig_64.c, 分别对应不同的访问方式。下面以[direct.c中的实现](https://elixir.bootlin.com/linux/latest/source/arch/x86/pci/direct.c#L83)为例：

```
const struct pci_raw_ops pci_direct_conf1 = {
	.read =		pci_conf1_read,
	.write =	pci_conf1_write,
};

static const struct pci_raw_ops pci_direct_conf2 = {
	.read =		pci_conf2_read,
	.write =	pci_conf2_write,
};
```

有两种direct config访问方式，区别在于指令结构不一样，参考PCI express base spec 的 Configuration Space Header。

```
/*
 * Functions for accessing PCI base (first 256 bytes) and extended
 * (4096 bytes per PCI function) configuration space with type 1
 * accesses.
 */

#define PCI_CONF1_ADDRESS(bus, devfn, reg) \
	(0x80000000 | ((reg & 0xF00) << 16) | (bus << 16) \
	| (devfn << 8) | (reg & 0xFC))
	
/*
 * Functions for accessing PCI configuration space with type 2 accesses
 */
#define PCI_CONF2_ADDRESS(dev, reg)	(u16)(0xC000 | (dev << 8) | reg)
```

以pci_conf1_read的内部实现为例：

```
static int pci_conf1_read(unsigned int seg, unsigned int bus,
			  unsigned int devfn, int reg, int len, u32 *value)
{
	unsigned long flags;

	if (seg || (bus > 255) || (devfn > 255) || (reg > 4095)) {
		*value = -1;
		return -EINVAL;
	}

	raw_spin_lock_irqsave(&pci_config_lock, flags);

	outl(PCI_CONF1_ADDRESS(bus, devfn, reg), 0xCF8);

	switch (len) {
	case 1:
		*value = inb(0xCFC + (reg & 3));
		break;
	case 2:
		*value = inw(0xCFC + (reg & 2));
		break;
	case 4:
		*value = inl(0xCFC);
		break;
	}

	raw_spin_unlock_irqrestore(&pci_config_lock, flags);

	return 0;
}
```

最底层调用的是x86的in, out汇编指令，也是宏函数封装，参考[arch/x86/include/asm/shared/io.h#L27](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/shared/io.h#L27)

```
static __always_inline void __out##bwl(type value, u16 port)		\
{									\
	asm volatile("out" #bwl " %" #bw "0, %w1"			\
		     : : "a"(value), "Nd"(port));			\
}									\
									\
static __always_inline type __in##bwl(u16 port)				\
{									\
	type value;							\
	asm volatile("in" #bwl " %w1, %" #bw "0"			\
		     : "=a"(value) : "Nd"(port));			\
	return value;							\
}
```

至此pci_read_config_byte一类的PCI register acess API分析完毕。

### PCIe AER driver分析和debug记录

AER: Advanced Error Reporting  

PCIe的AER是PCIe spec协议的标准功能，AER涉及到Error信号产生，上报，处理，错误恢复等。

#### PCIe base spec摘要：

Error分类：

![image-20230825171440881](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308251714033.png)

Error信号在数字逻辑的处理流水：

![image-20230825171423678](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308251714855.png)

AER的capability regsiter:

![image-20230825171511047](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308251715226.png)

#### PCIe AER driver摘要

PCIe driver的使用aer service driver实现软件处理流程，aer service属于PCIe bus driver的子功能，而PCIe driver又属于PCI driver字类。

什么是PCIe service driver: [2. The PCI Express Port Bus Driver Guide HOWTO](https://www.kernel.org/doc/html/v6.1/PCI/pciebus-howto.html)

什么是PCIe AER driver: [8. The PCI Express Advanced Error Reporting Driver Guide HOWTO](https://www.kernel.org/doc/html/v6.1/PCI/pcieaer-howto.html)

#### PCIe AER driver在Kconfig的开关

配置Kconfig可开关：make menuconfig -> / -> 搜索PCIEAER -> n关闭，y打开

如下Kconfig的AER三个相关项都被关闭：

```
CONFIG_PCIAER=n
CONFIG_ACPI_APEI_PCIEAER=n
CONFIG_PCIAER_INJECT=n
```

![image-20230825175611772](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308251756929.png)

#### 示例：ACS violation error的debug过程

在调试SD express host controller driver过程中，发现部分SD express card（ADATA & Lexar）在linux下无法正常初始化，windows下正常。以下为issue debug过程。

(1)首先打开PCI driver的debug打印

```
make menuconfig -> /搜索 -> PCI_DEBUG -> y
```

参考[Linux driver常用调试技术记录](https://cursorhu.github.io/2023/08/02/Linux-driver%E9%80%9A%E7%94%A8%E8%B0%83%E8%AF%95%E6%8A%80%E6%9C%AF%E8%AE%B0%E5%BD%95/)

(2)对比Good Case和Bad case

SD express card的切换是包含PCIe linkdown和linkup的过程，会有两次hot-plug中断处理。

Bad case的log中，发现如下两种AER error report:

![image-20230829114906631](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291149954.png)

a. RxErr

```
Link Training Error：一插卡就有此error. 此error发生在hot-plug中断之前
[  106.119856] pcie port 0000:00:1b.4: pciehp: pending interrupts 0x0100 from Slot Status
[  106.119863] pcieport 0000:00:1b.4: PCIe Bus Error: severity=Corrected, type=Physical Layer, (Receiver ID) //PCI driver检测到error属于Physical Layer
[  106.119864] pcieport 0000:00:1b.4:   device [8086:43c4] error status/mask=00000001/00002000 //error status bit0 =1
[  106.119866] pcieport 0000:00:1b.4:    [ 0] RxErr                  (First) //error的含义：在old version of PCIe spec表示Link Training Error.
```

b. ACS Violation

```
 ACS Violation error: 在hot-plug driver检测到hot-plug中断之后正在处理hot-plug的card/link状态检测过程中，PCIe driver检测到此error
[  107.493928] enter pciehp_ist
[  107.494136] pcieport 0000:00:1b.4: DPC: containment event, status:0x1f01 source:0x0000
[  107.494140] pcieport 0000:00:1b.4: DPC: unmasked uncorrectable error detected //PCI driver检测到error
[  107.494145] pcieport 0000:00:1b.4: PCIe Bus Error: severity=Uncorrected (Non-Fatal), type=Transaction Layer, (Receiver ID) //PCI driver检测到error属于Transaction Layer
[  107.494148] pcieport 0000:00:1b.4:   device [8086:43c4] error status/mask=00200000/00004000  //error status bit21 =1
[  107.494151] pcieport 0000:00:1b.4:    [21] ACSViol                (First)     //error的含义：ACS Violation
```

在此ACS error发生之后，hot-plug driver检测不到LinkUp（1st 检测）, polling 10s后也检测不到LinkUp（2nd 检测），最终打印No link，PCIe设备未启动configuration。

(3) ACS violation分析

PCIe协议分析抓包发现一个可疑的vendor defined message，可能是对应上述错误信息：

![image-20230829115802177](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291158420.png)

ACS violation在PCIe spec描述如下：

![image-20230829115115392](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202308291151689.png)

(4)原因和解决办法

逻辑链：PCIe Link training到L0之后，PCI driver还没发送Config Write操作定义Bus number的时候, SD express卡就发送了一个Vendor Specific defined message，这个是违反spec规定的；而RC side ACS 功能是enable的，会检测收到的包中的bus number是否和自己已扫描的bus相符，如果不符就报错ACS violation，并放弃卡初始化。

解决办法(workaround)：关闭PCIe bridge driver的ACS enable bit (ACS Control register bit0=1’b0):

具体代码改动为：

1). drivers/pci/pci.c -> pci_std_enable_acs ：pcie默认enable ACS Control register的ACS violation bit，此处修改为disable ACS violation bit.

2). drivers/pci/quirk.c -> pci_quirk_enable_intel_spt_pch_acs: 此处为Intel PCIe的特有配置，此处也 disable ACS violation bit
