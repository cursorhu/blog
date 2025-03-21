# SD Spec常用流程摘要

## SD legacy初始化流程

SD legacy初始化是指所有SD卡在SD模式下的通用初始化流程，在此流程之后才可能指向UHS-I，UHS-II的初始化流程。

流程图见SD physical spec:

CMD0：GO_IDLE_STATE：使卡进入Idle state，即卡复位状态

CMD8：SEND_IF_COND：验证卡接口（I/F）的operating conditon（COND），根据返回值的bit判断卡类型，这里不细讲，legacy初始化只看卡有没有返回CMD8，如果返回，获取到卡支持的电压（3.3V/1.8V），后面再用这个电压值。

ACMD41（CMD55+CMD41）：SD_SEND_OP_COND：设置operating conditon，相当于根据CMD8的结果，host发起对卡的配置请求。ACMD41并不指向具体的卡设置，可理解为设置流程之前的握手。

CMD11：Voltage Switch：切换Host和SD card之间的信号电平，一般从3.3V切到1.8V，电压切换包括CLK，DATA，CMD线。注意这里的电压切换不是指通信电平的电压，而不是SD卡的供电电压VDD。

CMD2：SEND_CID：Card Identification，获取卡的CID，此命令完毕后卡从idle state进入identify state。

CMD3：SEND_RCA：Relative Card Address，获取卡的地址（来自于卡的RCA register），此命令完毕后卡进入stand-by state(属于data transfer mode中的一种子状态)。

![image-20250103175846219](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031758279.png)

![image-20250103191821054](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031918136.png)

下图详细描述ACMD41的S18R和S18A，以及CMD11电压切换

![image-20250103193000571](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031930634.png)

## SD UHS-I的初始化流程

legacy初始化的CMD3使卡进入transfer mode的standby状态后，CMD7->CMD42->ACMD6->CMD6->CMD19即UHS-I的初始化流程：

![image-20250103193305212](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501031933260.png)