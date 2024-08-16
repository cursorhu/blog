# STM32--串口：UART和USB-COM

背景描述：STM32板子有TTL UART连接下游IC，同时有USB口连接上游的上位机PC。

本文描述STM32如何直接使用UART通信，如何用USB CDC实现虚拟串口USB-COM也用UART通信。

## UART项目配置

![image-20240430115304007](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404301153104.png)

## UART的轮询与中断

## USB的CDC类实现USB-COM

![image-20240430141713692](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404301417744.png)



## 双串口的实现：UART和USB-COM

![image-20240430141738799](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404301417818.png)

![image-20240430141746453](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404301417518.png)

参考：

[ST: Introduction to USB with STM32](https://wiki.st.com/stm32mcu/wiki/Introduction_to_USB_with_STM32#Communications_Devices_Class_(CDC))

[send-and-receive-data-to-pc-without-uart-stm32-usb-com](https://controllerstech.com/send-and-receive-data-to-pc-without-uart-stm32-usb-com/)

[stm32：实现USB虚拟串口（CDC_VPC）](https://www.cnblogs.com/FBsharl/p/17847962.html)

[如何让CDC类USB设备批量接收64字节以上数据](https://shequ.stmicroelectronics.cn/thread-637593-1-1.html)