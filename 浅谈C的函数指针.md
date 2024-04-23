---
title: 浅谈C的函数指针
date: 2020-04-01 12:30:00
tags: c
categories: c/c++
---

## 1. 函数指针基本概念
C语言调用函数的本质是什么？
1. CPU PC指针(program counter)跳转到函数的入口地址，传入参数，传入返回位置
2. 在函数栈内生成临时变量并执行函数内容指令，执行完毕后返回参数
3. CPU返回原调用处执行

这里，函数的入口地址实际上就存储在函数名中，也就是说，C代码中的函数名就是函数的入口地址。
既然是地址，就可以用来初始化一个指针，使指针指向该地址。
函数指针，就是存放函数首地址的指针。

### 1.2 函数指针变量
首先声明普通函数是如下格式：
`void Func(int);`
定义一个同类型函数的函数指针变量，只需要用`*p`表示函数名即可：
`void (*p)(int);`
注意，上面是定义了函数指针变量，而不是声明函数指针类型。

函数指针变量的定义，和普通变量格式不一样。
- 普通变量： <类型> <变量名>
- 函数指针：<函数类型 变量名>，按函数声明的格式定义，变量是包含在类型内部

那么此函数指针的类型是什么：
`void (*)(int);`

怎么使用此函数指针：
```
void Func(int x) // 声明一个函数*/
{
    printf("%d",x);
}
void (*p)(int); // 定义一个函数指针*/
p = Func; // 将Func函数的首地址赋给函数指针变量p*/
(*p)(100);  // 通过函数指针调用Func函数
```

### 1.3 函数指针类型
typedef可以定义某种类型的别名，例如将unsigned char定义为u8
`typedef unsigned char u8;`
可见其格式是：typedef <原类型> <别名类型>

那么如何定义函数指针类型：
只需要在函数指针声明的格式前加typedef, p就定义为函数指针类型而不是变量:
`typedef void (*p)(int);`

这里定义了`void (*)(int)`类型的函数指针类型，其别名为p

怎么使用此函数指针类型：
```
//定义类型
typedef void (*pFuncType)(int); 
//定义变量  
pFuncType p;   

void Func(int x)
{
	printf("%d",x);
} 

void main()   
{   
    p = Func; //初始化变量   
    (*p)(100);   //使用变量
} 
```

## 2. 函数指针的应用
### 2.1 Linux驱动软件设计的分层
C++中有多态的概念，父类定义某种函数类型，子类实现具体的函数内容，之后父类对象就可以调用子类的实现内容。
这样实现“父类定义格式，子类实现细节”的软件分层设计。

Linux驱动中也大量使用这种分层设计，C使用结构体和函数指针，分别对应C++的类和方法。
例如某驱动模块，顶层框架定义open, read, write, 而底层驱动根据具体硬件，实现xxx_open, xxx_read, xxx_write，然后按结构体成员赋值实现上下层联通：

以s3c的SDHCI驱动为例：
sdhci_ops即父类框架结构体，sdhci_s3c_ops为s3c芯片的具体实现结构体。
`.set_clock = sdhci_s3c_set_clock`就实现了父类的set_clock方法被子类sdhci_s3c_set_clock实现的效果。
其本质是函数指针的赋值，父类和子类的函数名都指向同一个函数地址，所以父类可以通过调用set_clock来实现调用sdhci_s3c_set_clock的效果。
这样，父类设计者根本不关心底层用什么函数名实现set_clock的功能，只需要调用set_clock这个函数名即可。

```
static struct sdhci_ops sdhci_s3c_ops = {
	.get_max_clock		= sdhci_s3c_get_max_clk,
	.set_clock		= sdhci_s3c_set_clock,
	.get_min_clock		= sdhci_s3c_get_min_clock,
	.set_bus_width		= sdhci_set_bus_width,
	.reset			= sdhci_reset,
	.set_uhs_signaling	= sdhci_set_uhs_signaling,
};
```

### 2.2 函数指针实现指令跳转
调用一个函数，其内部就包含跳转操作(jump指令)
那么，如何用C语言实现跳转到某个绝对地址去执行？具体场景：
在Bootloader过程中，加载完kernel到RAM后，要跳转到kernel首地址开始执行，如何实现？

方案一：C嵌入汇编
以下是基于SPARC CPU的汇编，跳转到0x40000000, 即RAM首地址执行firmware.
对于其他CPU，汇编实现也不同，因此此方法不能跨平台。
```
void boot_exit()
{
    /* jump to RAM entry to execute firmware. */

    asm(
        "set 0x40000000, %g2\n"
        "jmp %g2\n"
        "nop"
    );

}
```

方案二：函数指针
Bootloader中很常用的一种跳转方法
```
typedef void (*pFuncType)(); //定义无入参无返回类型的函数指针类型
pFuncType Reset = (pFuncType)0xF000FFF0; //函数指针指向待跳转地址
Reset(); //调用函数，实际上执行了跳转
```