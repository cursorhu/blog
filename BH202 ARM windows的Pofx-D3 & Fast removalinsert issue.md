# BH202 ARM windows的Pofx->D3 & Fast removal/insert issue.

##  issue现象和driver分析

客户问题：

客户平台5/100的平台有首次插卡不弹卡问题，复现测试发现快速插拔卡可以等效复现，有概率性插卡不识别问题，且不识别后再插拔卡也不识别。

WH lab分析：

1.BH202状态机可能有问题，卡侧和Host侧的通信方向不一致导致CMD8持续报错。

2.Workaround：通过Driver的error recovery flow的power cycle操作（power off + power on）去复位硬件状态。

Driver问题分析和优化点：

1.快速插拔卡后设备不能进D3（power off），原因是BH202 driver的pofx-> D3\pofx-> D0流程有问题，按driver issue fix architecture修改后问题解决。

3.BH202 driver的card change -> pofx request流程需要优化，确保pofx->D state时序正确

4.BH202 driver的card change event触发条件需要优化，避免快速插拔卡事件过多的问题

## Driver Issue Fix Architecture

### Power cycle diagram

![image-20250114135103318](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501141351408.png)

上图描述Issue fix的Driver Architecture，其中card handle和pofx handle属于BH202 miniport driver的模块，Storport AdapterControl属于miniport driver和Storport driver（and windows OS）的接口模块。

Note1：vdd1_power_cycle函数实现，内部调用pofx_request_d3和pofx_request_d0去实现power cycle（BH202 power off 再power on）。vdd1_power_cycle在以下情况被调用：

a.首次插卡，首次是指BH202没做过tuning

b.异常恢复流程，异常是卡初始化过程失败后的retry初始化

Note2：wait D3 response设置了超时时间5s，最大不能超过~8s，否则会触发SCSI BUS reset

Note3 and Note4：Storport返回的AdapterPowerControl D3只有一次，BH202 driver用作两个用途：

a.pofx handle认为系统响应了D3，即pofx->D3流程成功

b.card handle执行enter D3之前的准备流程，包括card stop transfer，card thread pending，card power off等函数，涉及到card一些标志的清理。

Note5: Storport driver收到D3请求的ACK（即请求返回）后，认为Adapter Idle；根据miniport注册的Adapter Pofx idle time 10ms，通知PMIC对BH202执行power off。

Note6：准备从D3返回D0，因为D3状态下没有任务或者中断插入执行，此处等待D0设置为2s

Note7：理论上应该是PMIC先上电后，Storport再通知miniport执行enter D0流程

Note8：进D0会发两次请求，首先是AdapterPowerControl - D0，BH202 driver由于实现了RestartAdapter，不需要在此请求做什么操作；之后是RestartAdapter，BH202 driver执行：

a.pofx handle认为pofx->D0流程成功

b.card handle执行enter D0的流程，包括host register reset，check card present，set card change event。

### pofx->D0/D3 diagram

![image-20250114142323274](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501141423340.png) 

### Card Event change优化

快速插拔卡时，有两个问题：

1. Driver对各种事件的处理是FIFO形式，当某次插拔卡事件（card change event）的处理流程遇到错误耗时较长，插拔卡事件会越积越多，但插拔卡实际是无记忆行为，driver只要能正确处理最终状态就可以，最终状态包括卡在位和卡不在位两种情况。
2. 有多种应用场景会设置card change event，driver需要减少不必要的event设置

Card event过滤如下：设置最多两个event的event FIFO：

1. 如果当前没有card change事件待处理，或者只有一个card change事件正在处理，此时shared task count = 0，可以排队新增一个card change event待处理，并设置shared task count = 1；

2. 如果shared task count = 1，不允许继续发起card change事件

3. 每个card change event事件处理完毕后会shared task count - 1，直到为0.

![bh202-event](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202501141442768.png)

## Bugfix状况

1.BH202 D0、D3问题：

原issue：快速插拔卡后拔卡，不能进D3，持续保持在D0。

现状：快速插拔卡后拔卡，能进D3（BH202能被power off），下次插卡能弹卡。

2.卡不识别问题：

原issue：快速插拔卡后，有时不识别卡，且不识别卡之后再插拔卡也不识别，除非Disable/Enable驱动或者重启，会对用户造成困扰。

现状：快速插拔卡后，有时不识别卡，但再插拔一次可以识别；原因是快速插拔卡影响了驱动的卡识别正常流程，只需要再插拔一次就可以识别，不会对用户造成困扰。

结论：

当前版本可以release