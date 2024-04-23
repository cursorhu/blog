

## SD初始化流程分析

### SD初始化的起始点

带SD卡启动系统时，初始化SD host和card的调用链：

probe -> mmc_alloc_host -> mmc_start_host -> _mmc_detect_change -> mmc_schedule_delayed_work(&host->detect) -> mmc_rescan（注释1）-> mmc_rescan_try_freq（注释2）

注释1：mmc_alloc_host时用INIT_DELAYED_WORK绑定delayed_work detect和mmc_rescan回调函数

```
INIT_DELAYED_WORK(&host->detect, mmc_rescan);
```

注释2：这里可能执行多次，从SD接口最低频率开始初始化，一般是400KHz，代码如下。 注意host->f_min是host driver设置。	

```
static const unsigned freqs[] = { 400000, 300000, 200000, 100000 };

for (i = 0; i < ARRAY_SIZE(freqs); i++) {
    if (!mmc_rescan_try_freq(host, max(freqs[i], host->f_min)))
        break;
    if (freqs[i] <= host->f_min)
        break;
}
```

### SD初始化的过程

以下具体分析mmc_rescan_try_freq流程，并用飞腾SD卡的初始化log对照理解。

#### 第一阶段：mmc通用初始化：

此时的host还没有指定卡是SD还是eMMC还是SDIO，这个阶段主要做卡power up和host reset，然后对卡reset（go_idle + send_if_cond）

![image-20240419152855006](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191529248.png)

对应log如下，以下CMD8 fail属于正常，因为此时还没指定是SD type

![image-20240419155547556](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191555610.png)

#### 第二阶段：SD卡的初始化入口：

SD模式的初始化入口：

根据host的能力(caps)，依次尝试初始化SDIO, then SD, then MMC，如果host能力指定了不支持SDIO或SD或MMC，将进入host支持的初始化入口

```
/* Order's important: probe SDIO, then SD, then MMC */
if (!(host->caps2 & MMC_CAP2_NO_SDIO))
    if (!mmc_attach_sdio(host))
        return 0;

if (!(host->caps2 & MMC_CAP2_NO_SD))
    if (!mmc_attach_sd(host))
        return 0;

if (!(host->caps2 & MMC_CAP2_NO_MMC))
    if (!mmc_attach_mmc(host))
        return 0;
```

怎么配置host能力：

host能力是指该SD/MMC/SDIO接口具体用作哪种类型，这个和平台外设的设计相关。例如飞腾Pi平台的SOC有两个SD/MMC/SDIO控制器接口：

![Image 14](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191611726.png)

对于SD0控制器, 只作为SD/eMMC接口使用，因此host driver对SD0可以设置host-cap能力如下：

```
host->caps2 |= MMC_CAP2_NO_SDIO | MMC_CAP2_NO_MMC;
```

这样在mmc_rescan_try_freq就可以跳过mmc_attach_sdio和mmc_attach_mmc， 只执行mmc_attach_sd去启动SD初始化流程。

#### SD卡初始化的基本概念

##### SD卡初始化的流程总览

SD spec的UHS-I流程总览：几个关键节点1～9。

![image-20240419170130663](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191701745.png)

以上UHS-I流程兼容SD3.0的SDR50, SDR25, SDR12等UHS模式，以及SD2.0的HS模式, DS模式：

![image-20240419162710488](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191627573.png)

##### SD卡初始化的接口概念

如下图SD host和card接口，卡初始化过程最核心的工作就是协商host侧和card侧的能力，最终以双方都能支持的最高速度通信，即完成卡初始化。

![image-20240419201436682](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404192014746.png)

所谓高速通信，无外乎以下几点：

更高的比特率：可以提高时钟频率和增加带宽。其中时钟CLK是host和card能力决定上限，host通过配置分频去控制接口频率（ios clock）；带宽是指SD接口有DATA[3:0] 4根数据线，但默认1bit模式只用了1根，切换到4bit模式后用4根线可同时传输4个bit。 

更低的信号电平：由于逻辑电平是梯形有建立时间，更低的电平信号能减少建立时间，达到更快的波特率；所以UHS模式一定要1.8V通信电平。

#### 第四阶段：SD卡初始化

在正式卡初始化之前， 有一个状态清理和协商工作

```
mmc_attach_sd： 

mmc_send_app_op_cond （这里是为了确认卡left busy state，有时候刚reset需要等状态恢复到IDLE）

host->ocr_avail = host->ocr_avail_sd; （这里拿到host支持的sd ocr电压值，例如3.3V）

mmc_select_voltage （这里查询卡支持的ocr电压值，以协商一致电压，一般而言这里是3.3V）
```

##### **SD卡初始化（上半部分）：**卡能力查询

```
mmc_sd_init_card：

	mmc_sd_get_cid：

		mmc_go_idle (节点1，CMD0使卡进入IDLE state)

		mmc_send_if_cond（节点2，CMD8使卡配置OCR电压）

		mmc_send_app_op_cond（节点3，CMD41查询卡容量，是否支持1.8V信号电）

		mmc_set_uhs_voltage（节点4，CMD11切换卡到1.8V信号电）

			mmc_host_set_uhs_voltage （调用host->ops->start_signal_voltage_switch切host侧到1.8V信号电，使两端通信电平一致）	
		
		mmc_send_cid（节点5，CMD2获取卡CID）
```

###### 关键节点描述：

节点2：CMD8应该设置arg[19:8] = 1aa,  1 来自前面host ocr = 2.7-3.6V support.

![image-20240419165618097](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191656210.png)

有时候CMD41没设置arg bit24=1, 可能是host cap没有正确配置：

```
phytium_mci_common_probe：
mmc->ocr_avail_sd = MMC_VDD_32_33 | MMC_VDD_33_34;
```

节点3：CMD41应该设置arg bit24  = 1去查询卡是否支持S18R，去切换信号电到1.8V， 这个切换能力决定是否启动后面的UHS模式的初始化。这个CMD41有一定retry次数。

![image-20240419170626765](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191706848.png)

有时候CMD41没设置arg bit24=1, 可能是host cap没有使能UHS模式和4bit bus width

```
mmc_sd_get_cid：
/*
 * If the host supports one of UHS-I modes, request the card
 * to switch to 1.8V signaling level. If the card has failed
 * repeatedly to switch however, skip this.
 */
if (retries && mmc_host_uhs(host)){
    ocr |= SD_OCR_S18R;
    pr_info("%s: %s: set SD_OCR_S18R \n", mmc_hostname(host), __func__);
}

mmc_host_uhs：
static inline int mmc_host_uhs(struct mmc_host *host)
{
	pr_info("%s: %s: host->caps = 0x%X \n", mmc_hostname(host), __func__, host->caps);
	return host->caps &
		(MMC_CAP_UHS_SDR12 | MMC_CAP_UHS_SDR25 |
		 MMC_CAP_UHS_SDR50 | MMC_CAP_UHS_SDR104 |
		 MMC_CAP_UHS_DDR50) &&
	       host->caps & MMC_CAP_4_BIT_DATA;
}
```

###### **host设置caps的两种方式：probe赋值，使用DTS**

一种方式是在host probe中对某个已知地址空间的mmc device直接设置：

```
phytium_mci_probe：

host->sd0 = (0 == strcmp(dev_name(dev), "28000000.mmc"));
if(host->sd0){
	host->caps |= MMC_CAP_UHS_SDR12 | MMC_CAP_UHS_SDR25 | MMC_CAP_UHS_SDR50 | MMC_CAP_4_BIT_DATA;
}
```

另外的方式是设备树中添加property描述，通过mmc_of_parse解析能力，参考：https://gitee.com/phytium_embedded/phytium-linux-kernel/issues/I9H1ZS?from=project-issue

![image-20240422110810589](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404221108662.png)

DTS位于arch/arm/boot/dts, 用于描述某个开发板的设备信息；DTS是.dts或.dtsi后缀，dtsi是共性的dts配置，相当于头文件可被dts include。

搜索sd-uhs-sdrxx相关的dts，一个示例如下（rk3288-veyron-sdmmc.dtsi）

```
&sdmmc {					//该SD/MMC/SDIO控制器是用于SD/MMC模式，不是SDIO模式
	status = "okay";		//用于enable或者disable此dts项，一般probe调用of_device_is_available检测此status.
	bus-width = <4>;		//SD支持4bit线宽
	cap-mmc-highspeed;		//支持eMMC HS
	cap-sd-highspeed;		//支持SD HS（SD2.0）
	card-detect-delay = <200>;
	cd-gpios = <&gpio7 RK_PA5 GPIO_ACTIVE_LOW>;		//卡检测（CD#）用的是GPIO7，低有效
	rockchip,default-sample-phase = <90>;
	sd-uhs-sdr12;			//支持UHS SDR12速率（SD3.0），时钟频率不超过24M
	sd-uhs-sdr25;			//支持UHS SDR25速率（SD3.0），时钟频率不超过50M
	sd-uhs-sdr50;			//支持UHS SDR50速率（SD3.0），时钟频率不超过100M
	sd-uhs-sdr104;			//支持UHS SDR104速率（SD3.0），时钟频率不超过208M
	vmmc-supply = <&vcc33_sd>;	//卡供电VCC = 3.3V
	vqmmc-supply = <&vccio_sd>;
};
```



##### **SD卡初始化（下半部分-part1）：UHS高速模式（SD3.0）配置**

在UHS下半部分初始化之前：

```
	host->ops->init_card：如果host有什么自定义寄存器配置，在此完成
```

开始UHS下半部分：

![image-20240419170130663](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191701745.png)

```
mmc_sd_init_card：

	mmc_set_relative_addr (节点6，CMD3获取卡地址：bus-device相对地址)
	mmc_select_card (节点7，CMD7选中bus上的某card)
                    （节点8，CMD42 unlock card，只有在CMD7的response显示card处于lock状态才发，此处一般不会lock，不发CMD42）
                    
	mmc_sd_setup_card：针对该card进行UHS初始化
		step1：一些信息准备工作
    	mmc_app_send_scr & mmc_decode_scr（ACMD51,读卡的SCR register，见注释1）
    	mmc_read_ssr（ACMD13,读卡的status register SSR，见注释2）
		mmc_read_switch（ACMD6读卡能支持的driver strength和bus_width，下一步切换要用）
	mmc_sd_init_uhs_card：切换到UHS模式的初始化（见注释3：需要host和card前面的协商结果是支持1.8V电平 && UHS模式支持 && bus width 4bit支持）
		step2：执行切换：
		一个ACMD6：
            mmc_app_set_bus_width（节点9，ACMD6执行bus width切换4bit）
            mmc_set_bus_width（host-ops也执行bus width切换到4bit）
		几个CMD6如下（见注释4）
            sd_select_driver_type（CMD6,设置卡侧driver strength并调用host ops也设置host侧register）
            sd_set_current_limit（CMD6, 根据vdd设置最大电流）
            sd_set_bus_speed_mode（CMD6，卡切换bus-speed到UHS中的SDRXX模式，并mmc_set_ios也设置host侧bus-speed）
		step3：切完后调频：tuning
			mmc_execute_tuning（节点10,CMD19，注释5）

mmc_sd_init_card end.
```

注释1：SCR register包含卡支持SD3.0以上版本与否，支持3.0以上则卡bus_width支持4BIT

注释2：SSR register包含卡的bus width，secured mode，card type等卡功能的配置

![image-20240419180712652](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404191807768.png)

注释3：切换到UHS模式的条件：

```
/* Initialization sequence for UHS-I cards */
if (rocr & SD_ROCR_S18A && mmc_host_uhs(host)) {
		err = mmc_sd_init_uhs_card(card);
```

注释4：CMD6配合Function Function Group配置不同功能，注意和ACMD6区分.

![image-20240419194003739](/home/cursorhu/.config/Typora/typora-user-images/image-20240419194003739.png)

注释5：tuning是高速模式下利用特定模式的数据（tuning block pattern）去模拟测试数据读写的CRC错误率，来反馈微调SD host和card接口的时钟频率，数据采样相位等，以适配不同PCB环境对高速信号干扰；如下图。

![image-20240419204049080](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404192040132.png)

注意：UHS较低速的SDR12～SDR25 < SDR50 < 100MHz的，不需要tuning, SDR50可以tuning也可以不tuning。

注意：tuning需要SD host硬件支持，有的host时钟分频能支持UHS SDR104但不支持tuning，如下图的SDR fixed-delay， 此时SD UHS卡最高只能工作在SDR50 <100MHz的速度，即这种host最高只能支持UHS SDR50模式。

![image-20240419203428136](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404192034201.png)

##### **SD卡初始化（下半部分-part2）：HS模式（SD2.0）配置**

如果卡不支持UHS模式，即不调用mmc_sd_init_uhs_card流程，或者该流程失败，会进入mmc_sd_switch_hs初始化流程，该模式指非UHS的低速模式，不需要切换信号电平，通信线宽，tuning等操作。

```
mmc_sd_switch_hs：
	mmc_sd_switch（CMD6 switch to high speed）
	mmc_set_clock（设置host ios clock）
	mmc_app_set_bus_width（如果支持4bit bus-width，就CMD6切换）
```

## 附录：SD性能测试

在飞腾Pi上SD是作为系统盘启动，读写性能测试一般用fio或者dd。

如果没联外网下载fio，用系统自带的dd即可：

dd参数注释：

1. if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >
2. of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >
3. bs=bytes：同时设置读入/输出的块大小为bytes个字节。
4. count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。

dd测试写和读的用例如下：

```
#测试写
root@phytiumpi:~# dd if=/dev/zero of=/dd-test.bin bs=1M count=256 conv=fdatasync
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 19.6135 s, 13.7 MB/s

#测试读
root@phytiumpi:~# dd if=/dd-test.bin of=/dev/zero bs=1M count=256 iflag=direct
256+0 records in
256+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 11.3607 s, 23.6 MB/s


```

注意两个参数很重要，是为了绕开写缓存直接写硬盘：

‘fdatasync’：Synchronize output data just before finishing. This forces a physical write of output data. 写数据测试包括写入物理盘，而不是写到buffer就算完。dd命令执行到最后会真正执行一次“同步(sync)”操作。

iflag=direct：读数据测试，直接读不经过读缓存。

这样测试也可以：

```
root@phytiumpi:~# dd if=/dev/zero of=/dd-test.bin bs=256M count=1 conv=fdatasync1+0 records in
1+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 19.4769 s, 13.8 MB/s

root@phytiumpi:~# dd if=/dd-test.bin of=/dev/zero bs=256M count=1 iflag=direct
1+0 records in
1+0 records out
268435456 bytes (268 MB, 256 MiB) copied, 11.327 s, 23.7 MB/s
```

