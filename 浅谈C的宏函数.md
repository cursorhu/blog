---
title: 浅谈C的宏函数
date: 2020-04-16 15:07:37
tags: c
categories: c/c++
---

# 1. 连接操作符:## 

    #define Conn(x,y) x##y

`##` 表示连接 , `x##y` 表示x连接y

示例：

    int n = Conn(123,456);
         ==> int n=123456;
    char* str = Conn("asdf", "adf");
         ==> char* str = "asdfadf";

`##` 的左右符号必须能够组成一个有意义的符号，否则预处理器会报错

# 2.字符串化和字符化: #, #@

(1) # 把任意类型的宏入参转化成字符串：

    #define ToString(x) #x

符号 # 表示字符串化操作符（stringification）。
其作用是：将宏定义中的传入参数名转换成用一对双引号括起来参数名字符串。
其只能用于有传入参数的宏定义中，且必须置于宏定义体中的参数名前。

示例：

     char* str = ToString(123132);
     ==> char* str="123132";

如果要对展开后的宏参数进行字符串化，则需要使用两层宏。

    #define xstr(s) str(s)
    #define str(s) #s
    #define foo 4
    
    str (foo)
         ==> "foo"
    xstr (foo)
         ==> xstr (4)
         ==> str (4)
         ==> "4"

(2) #@ 把任意类型的宏入参转化成单字符：

    #define ToChar(x) #@x

示例：

    char a = ToChar(1);
         ==> char a='1'

# 3. 不定参数宏: `__VA_ARGS__`

`__VA_ARGS__`宏用来接受不定数量的参数。例如：

    #define eprintf(...) fprintf (stderr, __VA_ARGS__)
    
    eprintf ("%s:%d: ", input_file, lineno)
    ==>  fprintf (stderr, "%s:%d: ", input_file, lineno)

当`__VA_ARGS__`宏前面加 `##` 时，可以省略参数输入。
例如：

    #define eprintf(format, ...) fprintf (stderr, format, ##__VA_ARGS__)
    
    eprintf ("success!\n")
    ==> fprintf(stderr, "success!\n");

# 4. 宏函数定义: do-while(0)与换行

(1) 用 do{}while(0) 定义宏函数

    #define foo() do{...}while(0)

宏函数可能在任意地方被调用，如果在if-else语句中，为了确保宏函数作为整体单元被编译器替换后不产生歧义，一般用do-while(0)包起来定义
这种定义有点像其他语言的“闭包函数”，纯粹是C语言中为了避免语法歧义的可靠性定义。

(2) 用显式换行符

宏函数定义不能直接回车换行，需要在回车换行前，用\（反斜线）表示下一行继续此宏的定义
预处理器在编译之前会自动将\与换行回车去掉。

例如：

    #define PRINT_INT(a)    \
    do{                     \
        printf("%d \n", a); \
    }while(0)
