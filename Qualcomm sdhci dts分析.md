# Qualcomm sdhci dts分析





## clocks部分

sc7280.dtsi中，sdhc的clocks属性如下：

```
/* 定义clocks数组，包含三个时钟的值 */
clocks = <&gcc GCC_SDCC1_AHB_CLK>, /* 第一个值，见gcc对象数组的索引GCC_SDCC1_AHB_CLK的成员的值 */
		<&gcc GCC_SDCC1_APPS_CLK>, /* 第二个值，见...对象的...成员的值 */
		<&rpmhcc RPMH_CXO_CLK>; /* 第三个值，见rpmhcc对象数组的索引RPMH_CXO_CLK的成员的值 */
clock-names = "iface", "core", "xo";  /* 以上三个clocks的数组成员，分别命名, 例如"xo"就对应第三个值 */
```

以"xo"这个clock为例，看下这个clock值到底是多少。

首先找&rpmhcc这个对象，这里有隐藏知识：rpmhcc是个"Clock providers"对象，一定是用”clock别名: clock-controller”的格式定义，在当前SOC dts搜索“rpmhcc: clock-controller”就直接找到其定义：

```
rpmhcc: clock-controller {
    compatible = "qcom,sc7280-rpmh-clk"; /* 这里很重要！ */
    clocks = <&xo_board>; /* rpmhcc的clocks源是xo_board */
    clock-names = "xo"; /* 当前clock别名xo */
    #clock-cells = <1>; /* 有两个clock输入(cells是0-based)，即xo_board应该有两组值 */
};
```

这里很奇怪：

1. rpmhcc没有定义regs，即没有直接定义他提供的时钟的值数值
2. compatible是干吗用的？

原因见下图，clock provider提供的时钟值，是compatible指定的代码里面定义的！

![img](https://img-blog.csdnimg.cn/20190215150254844.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzU0MjMwNQ==,size_16,color_FFFFFF,t_70)

rpmhcc本身的clocks源是xo_board对象，这里#clock-cells = <1>是表示rpmhcc clocks被几个消费者使用（输出几路时钟），0-based，#clock-cells = <0> 表示只有一个成员。

搜索driver/clk/qcom找rpmh时钟的源文件：

linux-6.7\drivers\clk\qcom\clk-rpmh.c

```
static const struct of_device_id clk_rpmh_match_table[] = {
	{ .compatible = "qcom,qdu1000-rpmh-clk", .data = &clk_rpmh_qdu1000},
	{ .compatible = "qcom,sa8775p-rpmh-clk", .data = &clk_rpmh_sa8775p},
	{ .compatible = "qcom,sc7180-rpmh-clk", .data = &clk_rpmh_sc7180},
	....
	{ .compatible = "qcom,sc7280-rpmh-clk", .data = &clk_rpmh_sc7280},
	{ }
};

static const struct clk_rpmh_desc clk_rpmh_sc7280 = {
	.clks = sc7280_rpmh_clocks,
	.num_clks = ARRAY_SIZE(sc7280_rpmh_clocks),
};

static struct clk_hw *sc7280_rpmh_clocks[] = {
	[RPMH_CXO_CLK]      = &clk_rpmh_bi_tcxo_div4.hw,
	[RPMH_CXO_CLK_A]    = &clk_rpmh_bi_tcxo_div4_ao.hw,
	[RPMH_LN_BB_CLK2]   = &clk_rpmh_ln_bb_clk2_a2.hw,
	[RPMH_LN_BB_CLK2_A] = &clk_rpmh_ln_bb_clk2_a2_ao.hw,
	[RPMH_RF_CLK1]      = &clk_rpmh_rf_clk1_a.hw,
	[RPMH_RF_CLK1_A]    = &clk_rpmh_rf_clk1_a_ao.hw,
	[RPMH_RF_CLK3]      = &clk_rpmh_rf_clk3_a.hw,
	[RPMH_RF_CLK3_A]    = &clk_rpmh_rf_clk3_a_ao.hw,
	[RPMH_RF_CLK4]      = &clk_rpmh_rf_clk4_a.hw,
	[RPMH_RF_CLK4_A]    = &clk_rpmh_rf_clk4_a_ao.hw,
	[RPMH_IPA_CLK]      = &clk_rpmh_ipa.hw,
	[RPMH_PKA_CLK]      = &clk_rpmh_pka.hw,
	[RPMH_HWKM_CLK]     = &clk_rpmh_hwkm.hw,
};
```

其中"qcom,sc7280-rpmh-clk"最终data是sc7280_rpmh_clocks[]数组，因此按index RPMH_CXO_CLK拿到的值是&clk_rpmh_bi_tcxo_div4.hw

这个值到底是多少？



最后看下rpmhcc的上游：xo_board对象：

xo_board有两个clocks源：xo_board和sleep_clk，这里的clock-frequency是是固定晶振时钟，整个时钟树的最上游。

```
clocks {
    xo_board: xo-board {
        compatible = "fixed-clock";
        clock-frequency = <76800000>;
        #clock-cells = <0>;
	};

	sleep_clk: sleep-clk {
        compatible = "fixed-clock";
        clock-frequency = <32000>;
        #clock-cells = <0>;
    };
};
```

<&rpmhcc RPMH_CXO_CLK>中的index RPMH_CXO_CLK是哪个值，在SOC dts头文件可以看到：

```
#include <dt-bindings/clock/qcom,rpmh.h>
```

去kernel/include的dt-bindings/clock/qcom,rpmh.h查找到是0

```
/* RPMh controlled clocks */
#define RPMH_CXO_CLK				0 
#define RPMH_CXO_CLK_A				1
#define RPMH_LN_BB_CLK2				2
#define RPMH_LN_BB_CLK2_A			3
```

这个头文件是厂商自定义的索引，其实没必要关系具体值是多少。



看下xo clock在sdhci代码怎么用的：sdhci-msm.c的probe函数：

```
/* devm_clk_get是获取clock dts的API，此处获取别名未"xo"的clock值 */
msm_host->xo_clk = devm_clk_get(&pdev->dev, "xo");
if (IS_ERR(msm_host->xo_clk)) {
    ret = PTR_ERR(msm_host->xo_clk);
    dev_warn(&pdev->dev, "TCXO clk not present (%d)\n", ret);
}
```

dts clock参考：[设备树中时钟的使用](https://blog.csdn.net/weixin_43542305/article/details/87363277)

下面看下core这个clock，和xo clock不一样：

```
clocks = <&gcc GCC_SDCC2_AHB_CLK>,
        <&gcc GCC_SDCC2_APPS_CLK>, /* core对应的clock */
        <&rpmhcc RPMH_CXO_CLK>;
clock-names = "iface", "core", "xo";
```

GCC_SDCC2_APPS_CLK在当前SOC的qcom,gcc-sc7280.h是

```
#define GCC_SDCC2_APPS_CLK				114
```

gcc: clock-controller内容如下

```
gcc: clock-controller@100000 {
compatible = "qcom,gcc-sc7280";
reg = <0 0x00100000 0 0x1f0000>;
clocks = <&rpmhcc RPMH_CXO_CLK>,
	<&rpmhcc RPMH_CXO_CLK_A>, <&sleep_clk>,
	<0>, <&pcie1_phy>,
	<0>, <0>, <0>,
	<&usb_1_qmpphy QMP_USB43DP_USB3_PIPE_CLK>;
clock-names = "bi_tcxo", "bi_tcxo_ao", "sleep_clk",
	"pcie_0_pipe_clk", "pcie_1_pipe_clk",
	"ufs_phy_rx_symbol_0_clk", "ufs_phy_rx_symbol_1_clk",
	"ufs_phy_tx_symbol_0_clk",
	"usb3_phy_wrapper_gcc_usb30_pipe_clk";
#clock-cells = <1>;
#reset-cells = <1>;
#power-domain-cells = <1>;
power-domains = <&rpmhpd SC7280_CX>;
};
```

重点看下reg = <0 0x00100000 0 0x1f0000>;

一般reg都是`<address size>`，这里怎么定义了两个从0开始的空间，size还不一样？

这是因为此SOC dts使用的是双32bit（64bit）定义，见SOC dts开头的定义：

```
#address-cells = <2>; /* address使用两个cell，每个cell 32bit表示地址 */
#size-cells = <2>; /* size使用两个cell，每个cell 32bit表示地址 */
```

因此"0 0x00100000"是高位为0，低位为0x00100000，这就是address；"0 0x00100000"也是同理， 0x00100000就是address。

下面看下gcc: clock-controller@100000 的compatible有没有data：

linux-6.7\drivers\clk\qcom\gcc-sc7280.c：没有定义data，因此这个clock的值是reg区间定义的

```
static const struct of_device_id gcc_sc7280_match_table[] = {
	{ .compatible = "qcom,gcc-sc7280" },
	{ }
};
```

为了确保这个理解正确，全局搜索一下"qcom,gcc-sc7280"，没有其他位置有data，有个示例文档：

linux-6.7\linux-6.7\Documentation\devicetree\bindings\clock\qcom,gcc-sc7280.yaml

```
    clock-controller@100000 {
      compatible = "qcom,gcc-sc7280";
      reg = <0x00100000 0x1f0000>;
```

进一步验证了之前分析的address和size使用双32bit定义。

结论："core" clock的值要看SOC sc7280的0x00100000区间的offset 114 bytes（GCC_SDCC2_APPS_CLK）的值，取决于硬件，dts无法直接看到值。

题外话：在调查GCC_SDCC2_APPS_CLK时找到一些容易产生误导的代码：

linux-6.7\drivers\clk\qcom\gcc-sc7280.c

```
static struct clk_branch gcc_sdcc2_apps_clk = {
	.halt_reg = 0x14004,
	.halt_check = BRANCH_HALT,
	.clkr = {
		.enable_reg = 0x14004,
		.enable_mask = BIT(0),
		.hw.init = &(struct clk_init_data){
			.name = "gcc_sdcc2_apps_clk",
			.parent_hws = (const struct clk_hw*[]){
				&gcc_sdcc2_apps_clk_src.clkr.hw,
			},
			.num_parents = 1,
			.flags = CLK_SET_RATE_PARENT,
			.ops = &clk_branch2_ops,
		},
	},
};

static const struct freq_tbl ftbl_gcc_sdcc2_apps_clk_src[] = {
	F(400000, P_BI_TCXO, 12, 1, 4),
	F(19200000, P_BI_TCXO, 1, 0, 0),
	F(25000000, P_GCC_GPLL0_OUT_EVEN, 12, 0, 0),
	F(50000000, P_GCC_GPLL0_OUT_EVEN, 6, 0, 0),
	F(100000000, P_GCC_GPLL0_OUT_EVEN, 3, 0, 0),
	F(202000000, P_GCC_GPLL9_OUT_MAIN, 4, 0, 0),
	{ }
};

static struct clk_rcg2 gcc_sdcc2_apps_clk_src = {
	.cmd_rcgr = 0x1400c,
	.mnd_width = 8,
	.hid_width = 5,
	.parent_map = gcc_parent_map_9,
	.freq_tbl = ftbl_gcc_sdcc2_apps_clk_src,
	.clkr.hw.init = &(struct clk_init_data){
		.name = "gcc_sdcc2_apps_clk_src",
		.parent_data = gcc_parent_data_9,
		.num_parents = ARRAY_SIZE(gcc_parent_data_9),
		.flags = CLK_OPS_PARENT_ENABLE,
		.ops = &clk_rcg2_floor_ops,
	},
};
```

