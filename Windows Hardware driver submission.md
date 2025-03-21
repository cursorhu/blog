# Windows Hardware driver submission



## Driver Update failure (rejected) 问题分析

### 背景描述：

V10700 + Hardware ID：PCI\VEN_1217&DEV_8621&SUBSYS_389517aa

在windows hardware driver submission被reject，提交单和关键信息如下：

https://partner.microsoft.com/en-us/dashboard/hardware/driver/14391505706251263/submission/1152921505697924694/ShippingLabel/401531646

Rejection Theme: Measure Failure Rejection Reason: Systemic Measure Failure Provide Measure ID(s): 26387215 ADO bugId: 55312863: Bug ID(s) and Partner ID(s): Link to documentation: Link to reliability report (if applicable): Details: Rejected because Measure ID 26387215 (Percent of machines where the driver install process completes successfully) is failing. Please read more on "https://docs.microsoft.com/en-us/windows-hardware/drivers/dashboard/pct-machines-where-driver-install-completes-successfully" See the Plug and Play Extended Flight Report document for additional information on the failures. The "driver flight report" bug can be found by searching on Collaborate with the Submission ID. More info: https://docs.microsoft.com/en-us/windows-hardware/drivers/dashboard/pnp-failure-report and https://docs.microsoft.com/en-us/collaborate/feedback-items-search#search

### 报告分析

根据以上信息，在Microsoft partner center的feedback/bugs页面找到提交单对应的failure报告：

![image-20250103150628028](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031506203.png)

（1）RejectionReport中的关键信息：

![image-20250103152056064](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031520135.png)

小结：微软测试了118个machine，其中在10台硬件平台ID PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA的机器上发生install failure. 

错误码是ERROR_ACCESS_DENIED, 可能是微软环境问题或者硬件访问异常导致。

（2）report.html中的关键信息：

![image-20250103151330025](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031513121.png)

![image-20250103151409828](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031514877.png)

小结：Driver install failure基本都发生在硬件平台ID PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA上，而且是不带Problem Code Description & Problem Status 类型的安装错误，也就是说不是PNP error引起的install failure。

### Driver分析

为了确认此install error是driver相关还是硬件平台相关（PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA），做如下对比：

1.找到相同Driver installer V10700的其他提交单，和当前failure提交单的区别只在于支持的硬件平台ID不一样：

https://partner.microsoft.com/en-us/dashboard/hardware/driver/14391505706251263

![image-20250103153612882](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031536917.png)

2.两个提交单只有Hardware ID有区别，系统都是Win11 24H2：

![image-20250103153449036](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031534126.png)

3.两个提交单的微软反馈报告的区别：一个pass，一个failure（failure install发生在Bad hardware ID上）

![image-20250103153815661](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031538765.png)

https://partner.microsoft.com/en-us/dashboard/collaborate/engagements/5133/feedback/wits/Bugs/1056149

https://partner.microsoft.com/en-us/dashboard/collaborate/engagements/5133/feedback/wits/Bugs/1069371

### 结论

**分析小结：**

微软的自动化测试流程发现V10700 installer在平台PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA上有install failure，超出了允许的failure rate（<5%）,所以reject driver提交单。

根据Driver提交单对比分析，这不是Driver installer的问题，因为相同的driver在其他平台的提交单是100% pass，此问题只发生在平台PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA上。

**Debug方向：**

不确定微软是如何在平台PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA上测试此driver install，有可能是物理机器或者虚拟机器上测试，也就是说，不排除是微软软件环境问题引起。

另外一方面，如果是物理机器确实有此问题，建议检查平台PCI\VEN_1217&DEV_8621&SUBSYS_38DF17AA上是否有特殊系统设置，或者PCIe bug，导致Bayhub SD Host硬件有时无法访问？

**当前解决办法：**

重新提交此driver，不需要做修改，以排除是微软软件环境引起的偶然性问题。