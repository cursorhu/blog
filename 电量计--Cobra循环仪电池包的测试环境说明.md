# 电量计--Cobra/循环仪/电池包的测试环境说明

以Newton FW项目(77561, 77226)为例，介绍cobra工具配合客户环境电池包的使用。

客户环境指：电量计内置在电池包，通信接口只有I2C，不支持串口调试和Jtag下载FW；而开发环境的电量计是独立开发板，支持串口打印和Jtag下载。

在实验室环境：使用Chroma循环仪模拟真实的客户Charger，对电量计和电池系统充放电测试，用cobra配置电量计FW和参数并采集数据，调试客户遇到的问题。

## Cobra编辑和下载project文件

1. 启动项目匹配的Cobra shell版本，加载项目oce文件

   cobra shell是程序启动器，版本号在 About查看，目前使用1.01.19版本

   oce是cobra功能文件，决定具体项目支持的功能，不同项目的oce不同；oce分为X版本和Y版本，X为发布给用户使用版本，Y仅用于内部Debug。

   下图在Extension Manager中select SD77226SBS_X_20250315.oce，oce路径在COBRA\Extensions

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191558616.png)

2. （对于Newton项目）加载prj

prj文件(project)是cobra对Newton FW bin和Flash参数文件xml（parameter_newton，OCV table， RCtable，user_setting）的打包。

加载prj后默认显示newton FW bin：

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191608196.png)

加载prj后可查看(show)参数：根据电池规格书配置Design Capacity容量，Limited Charge Voltage电压等参数；根据常温和高低温需求配置CC转CV的电压值(Constant Current切换的Constant Voltage转折点)的电压值和温度阈值。

![image-20250319161351611](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191613694.png)

![image-20250319161401395](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191614471.png)

OCV(Open Circuit Voltage) table和RC(Remaining Capacity) table都是根据具体电池实测的电池数据，一般不会更改（除非测试数据有问题或者更换电池）。

以RC table为例，数组对应关系如下同颜色的标号

![image-20250319163556911](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191635013.png)

**关于RC table：**

RC table反映的是在不同环境下的充放电过程中的电压和剩余容量的对应关系。

RC table的采集方法是对电池放电（从满电以恒流放到截止电压），在不同温度下，用不同电流放电，分别采集到电池输出电压和电池剩余容量的关系。

为什么RC table是采集放电过程而不是充电过程：（可能）是因为放电过程可以保持恒流，如果采集充电过程，充电CC-CV阶段开始后电流会降低，破坏了恒定电流值的要求。

为什么RC table要考虑问题温度和电流：锂电池特性决定，温度影响锂电池的放电容量，温度降低，电池内阻加大，电池放电容量下降；温度过高有风险，有时应用上需要调低满电容量。电流实际能反映负载的分压，由于电池开路电压一定，电池内置在某时刻一定，那么电池带负载时输出电流越大说明外部负载电阻小，大电流放电，电池内阻导致的压降更多，放电到截止电压就会提前到来，满电对应的输出电压也不一样，最终反映到RC table。

**关于OCV table：**

OCV table反映的是电池开路(非充放电)状态下的电压和剩余容量的对应关系。

严格的OCV table应该是用极小的电流放电采集容量变化。小电流说明负载电阻极大，可以近似开路环境。但这种采集时间太长，一般没搞。

项目实际用的OCV table是RC table采集过程中提取出来的数据，会有误差。

对于电量计FW，OCV table一般用于电池在持续Idle/sleep状态下查询剩余容量，因为此时没有电流，只能靠电压查询。

充放电过程中查询容量主要使用RC table，初始充电时也会参考OCV table。

**关于电量计计算SOC（State-of-charge）是否符合标准的判定方法：**

SOC（State-of-charge）可以简单理解成剩余电荷容量（RC）占满充总容量(FCC，Full Charge Capacity)的百分比[0 ~ 100]

理论上，电量计应该在每个时刻都反映精确的剩余容量百分比，但很难做到（原因？？？？）

项目的SOC算法的目标：

(1) 应用层目标：在充电达到截止电流时，SOC要报100；在放电达到截止电压时，SOC要报0。SOC误差 < 3%

(2) 基于(1)的两个测试标准，如果FW只靠RC table和OCV table计算SOC，可能不能达到要求。因此SOC算法添加了追赶机制：

当充电进入CV阶段恒流升压转恒压降流，SOC算法可能会开启SOC追赶，确保电流达到截止时SOC能到100。CV过程的SOC并不能反映真实电量变化。

当放电接近截止电压，SOC算法可能会开启SOC追赶，确保达到截止电压时SOC能到0。这个过程的SOC也不能反映真实电量变化。

中间的过程SOC值主要来自RC table数据，SOC算法可能只做一些消抖平滑处理。

3.（对于Newton项目）下载prj到电量计芯片(ARM M0)的Flash

Full download下载当前prj内的所有bin和参数数据到电量计（通信方式是I2C）

4.（对于Newton项目）更新prj

如果FW bin需要修改，可以在当前prj界面open用KEIL MDK编译出的新newton_encript.bin，再save as新的prj

如果有参数需要修改，直接改参数，再save as新的prj

## Cobra使用SBS采集数据

### 连接电量计

Bus setting选择port连接。使用o2link gen1 USB转I2C转接板(8051芯片)，需要先安装windows转接板驱动。

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191930845.png)

### 初始界面

默认可勾选查询哪些电量计信息，下方可切换页面到log信息页面

![image-20250319193558973](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191935049.png)

### 开启调试内存GGMEM#（可选）

GGMEM0~GGMEM8是Cobra查询电量计内存中指定RAM区域数据的接口，是Cobra环境监测电量计参数变化的主要手段。

选中FWVersion，Ctrl+ALT+Y输入密码888888，使能GGMEM0~GGMEM8

![image-20250319194120118](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191941173.png)

![image-20250319194304045](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191943097.png)

### 查看Cobra log

Cobra log是每秒发送一次之前勾选的SBS命令给电量计，查询电量计已获取的电池信息（电量计可能每秒轮询一次电池信息）。

主要关注电压(Battery Voltage)，外部温度(ETHM), 电流(Battery Current)和电量计算法输出的SOC容量（RSOC）

![image-20250319194542869](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503191945945.png)

### 查看GGMEM

查看GGMEM0值

![image-20250319200834671](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192008717.png)

查看GGMEM7的值和Cobra参数设定一致（高温环境，EOC电压是4100mV，EOC电流是500）

![image-20250319202040571](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192020659.png)

### 详细了解ggmem的映射过程

GGMEM字段在电量计FW中的定义：

![image-20250319201430891](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192014937.png)

GGMEM是如何从电量计RAM对应到Cobra SBS命令的：

![image-20250319201605094](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192016127.png)

如果要打开更多GGMEM9显示到Cobra log，需要更新cobra oce支持新SBSD9_GGMEM9，修改FW新增绑定SBS SBSD9_GGMEM9 = gg_mem[X]，X可以是任何偏移，不一定是72。

## 	Cobra电流校准

更新新的prj到电量计后，有时候Cobra采集的电流和charger/循环仪的真实电量不一致，需要calibration校准电流，分为3步：

0电流校准；3V充电校准；-3V放电校准。

先用循环仪设置对应的电流，再点击cobra calibrate按钮即可；分别校准3种电流情况之后，SBS log的电流误差应该会消除。

如果没有更新prj，校准过的数据无需再校准

![image-20250319203521951](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192035028.png)

## Cobra应用示例

如下图是充电进入CV阶段的部分log，电压恒压，电流减少。但RSOC只有62%，后面算法应该会加速追赶，确保在达到满充（电流降到500）时SOC到100%

![image-20250319204006543](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503192040622.png)

这是export log导出csv文件，用曲线图分析



## 循环仪Chroma 17020的使用

### 选择充电和放电配方

选择1A恒流充电（CC_1A），勾选连接到电量计的通道1-3，可开始充电和停止充电。

![image-20250320102129025](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201021090.png)

1A恒流放电（DS_1A）同理：

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201023474.PNG)

在Cobra电流校准时，使用CC_3A和DS_3A，启动充电和放电后，再点击Cobra校准。

### 编辑充放电配方

配方编辑器可查看现有的配方，也可以编辑和新增配方

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201024587.PNG)

查看已有的CC_3A配方的具体设置：

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201025579.PNG)

CC转CV电压模式充电：

CC阶段恒流3A充电电压逐渐升高到4.53，之后开始CV阶段恒压4.53，电流从3A逐渐减少到截止电流0.12A停止；关闭所有过流保护。

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201025475.PNG)

查看DS_1A放电的配方设置：

恒流放电：

1A恒流放电，电压从4.xV左右逐渐降低到截止电压3V就停止；关闭所有过流保护。

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201027970.PNG)对于不同的电池测试需求，可以修改参数之后存储档案。

### 新建充放电配方

可以配置“放电+静置+充电”的自动运行配方，方便自动测试

![img](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503201034525.PNG)

## Cobra采集数据分析

示例数据：[Scan_03_19_2025_11_44_14.csv](https://o2micro-my.sharepoint.com/:x:/p/thomas_hu/EftK5pMRry5HtS552GzuDRcBmb_yms6TSJNWoHF33muY4A?e=noJtCx)

从放电快结束开始记录，静置，然后CC-CV充电到截止电流。

电压-电流曲线：

![image-20250321152603963](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211526999.png)

比较真实电荷变化(mAh)和电量计评估的容量（RSOC）：

![image-20250321152851154](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211528188.png)

![image-20250321152538723](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202503211525757.png)

![image-20250321152447765](C:\Users\cursorhu\AppData\Roaming\Typora\typora-user-images\image-20250321152447765.png)