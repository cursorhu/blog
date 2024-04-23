---
title: esp32笔记之PWM实现LED呼吸灯
date: 2023-05-05 16:50:16
tags: esp32
categories: esp32

---

## PWM简介

先从应用上讲讲PWM：

有一盏日光灯，一般我们只能打开它或者关闭它，不存在中间状态；

有另一个LED灯，支持在一秒以内极快速的速度开关开关，其变化超过人眼识别的24帧率，LED灯看上去就像一直开着，但亮度比常开暗一些；如果控制灯快速开关过程中的打开时间和关闭时间的比例，就可以调节人眼看到的灯亮度。

以上就是PWM的大概应用原理：用高频率的开关信号，控制输出信号的平均强度，使输出信号能在0%到100%强度间任意调节。

用电路语句讲PWM原理：用数字信号的占空比来调制模拟信号的幅度(电压)。

PWM详细介绍参考：[What is PWM: Pulse Width Modulation](https://circuitdigest.com/tutorial/what-is-pwm-pulse-width-modulation)

脉冲宽度(pulse width)是指单位时间的高电平的持续时间，脉冲宽度越大被调制的模拟信号电压越大。

- 在一定的频率下，通过不同的(高电平)占空比即可得到不同脉冲宽度，进而调节输出的模拟电压信号
- 在一定的占空比下，通过不同的频率实现不同的调节速度；频率要适配不同设备，不能任意设置，例如电机频率50HZ，MCU外设1000Hz。频率不决定被调制电压的幅度。

PWM的调制信号如下：

![img](https://circuitdigest.com/sites/default/files/inlineimages/pulse-width-modulation-duty-cycle.gif)

PWM调制电路通常用RC filter实现：

![Converting-PWM-signals-into-Analog](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305051659127.jpg)

PWM一般对具体设备使用固定频率，再调整高电平的占空比决定模拟信号的幅度。

如下图，占空比从0%调节到100%，对应输出电压为0V~5V

![Pulse-Width-Modulation](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305061100234.jpg)

从原理上讲就是开关控制，在一个周期内调制信号的高电平时间越长，RC电荷积分更多，输出电压越大：

![image-20230505165748199](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305051657299.png)

## MicroPython控制PWM

官方tutorial参考：

[Quick reference for the ESP32](https://docs.micropython.org/en/latest/esp32/quickref.html) PWM (pulse width modulation)

[Pulse Width Modulation](https://docs.micropython.org/en/latest/esp32/tutorial/pwm.html#esp32-pwm) 其中有调整频率和占空比的sample code:

- Example of a smooth frequency change:

  ```
  from utime import sleep
  from machine import Pin, PWM
  
  F_MIN = 500
  F_MAX = 1000
  
  f = F_MIN
  delta_f = 1
  
  p = PWM(Pin(5), f)
  print(p)
  
  while True:
      p.freq(f)
  
      sleep(10 / F_MIN)
  
      f += delta_f
      if f >= F_MAX or f <= F_MIN:
          delta_f = -delta_f
  ```

- Example of a smooth duty change:

  ```
  from utime import sleep
  from machine import Pin, PWM
  
  DUTY_MAX = 2**16 - 1
  
  duty_u16 = 0
  delta_d = 16
  
  p = PWM(Pin(5), 1000, duty_u16=duty_u16)
  print(p)
  
  while True:
      p.duty_u16(duty_u16)
  
      sleep(1 / 1000)
  
      duty_u16 += delta_d
      if duty_u16 >= DUTY_MAX:
          duty_u16 = DUTY_MAX
          delta_d = -delta_d
      elif duty_u16 <= 0:
          duty_u16 = 0
          delta_d = -delta_d
  ```

## 呼吸灯示例

参考：[itproject.cn/Python+ESP32快速上手/3.PWM呼吸灯](https://doc.itprojects.cn/0006.zhishi.esp32/02.doc/index.html#/03.PWMhuxideng)

esp32的micropython代码以script形式执行，主程序必须命名为main.py(参考 [Running your first script](https://docs.micropython.org/en/v1.9.3/pyboard/pyboard/tutorial/script.html)):

```
from machine import Pin, PWM
import time

led2 = PWM(Pin(2))
led2.freq(1000)

while True:
    for i in range(0, 1024):
        led2.duty(i)
        time.sleep_ms(1)
        
    for i in range(1023, -1, -1):
        led2.duty(i)
        time.sleep_ms(1)
```

LED渐变呼吸闪烁：

![mmexport1683287925729](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202305052003803.gif)

如果将led duty调整为512，最大亮度会变小，验证了最大占空比决定最大电压

如果将led freq调整为50，最大亮度不变，但led渐变过程中会闪烁，也就是说开关调节频率太低，导致人眼都可以观察到led的开关电，看上去就是led闪烁
