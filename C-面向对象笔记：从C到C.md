---
title: C++面向对象笔记：从C到C++
date: 2020-03-14 16:41:50
tags: C++
categories: C++

---

# 0.概述
本章介绍C++语言和C语言相近的部分基础用法，包括

 - 引用: &
 - 常关键字: const
 - 动态内存分配: new delete
 - 函数内联: inline
 - 函数重载

# 引用和指针
## 引用的概念
下面的写法定义了一个引用，并将其初始化为引用某个变量。

    类型名 & 引用名 = 某变量名;

某个变量的引用，等价于这个变量，相当于该变量起了一个别名。别名类似于操作系统的文件链接或快捷方式的概念，访问它变量本身的存储空间。

    int n = 4;
    int & r = n; // r引用了 n, r的类型是int &
    r = 4;
    cout << r; //输出 4
    cout << n; //输出 4
    n = 5;
    cout << r; //输出5

注意：
1.定义引用时一定要将其初始化成引用某个变量。
2.初始化后，它就一直引用该变量，不会再引用别
的变量了。
3.引用只能引用变量，不能引用常量和表达式。
## 引用的示例
引用常用于函数传参和返回值
1.引用作为函数入参
C语言写一个swap函数，交换两个变量的值，要传指针而不能传值，因为直接传值实际修改的是函数局部作用域的一份拷贝。

    void swap( int * a, int * b)
    {
        int tmp;
        tmp = * a; * a = * b; * b = tmp;
    }
    int n1, n2;
    swap(& n1,& n2) ; // n1,n2的值被交换
C++中，除了传指针，也可以传引用

    void swap( int & a, int & b)
    {
        int tmp;
        tmp = a; a = b; b = tmp;
    }
    int n1, n2;
    swap(n1,n2) ; // n1,n2的值被交换
2.引用作为函数返回值

    int n = 4;
    int & SetValue() { return n; }
    int main()
    {
    SetValue() = 40;
    cout << n;
    return 0;
    } //输出： 40

## 引用和指针的区别
看上去引用和指针的功能相同，那区别在哪？
1.存储类型不同

 - 指针是一种变量，存储指向变量的地址值，通常占内存4字节（64位系统8字节）
 - 引用只是变量的别名，它本身不另外占存储空间，对其求大小（sizeof）就是变量本身的大小

指针是变量，因此可以为空（0x0）,而引用是标签（别名），不可为空，先有变量才能有其引用。
2.作用方式不同

 - 指针作为函数入参本质上还是是值传递，只不过传递的是变量的地址值，函数局部拷贝的也是地址。
 - 引用作为函数入参，被调函数的形参作为局部变量在栈中开辟了内存空间，但存放的是主调函数的实参变量的地址。被调函数对形参的任何操作都被处理成间接寻址，即通过栈中存放的地址访问主调函数中的实参变量。

对于函数传参，形参都是用地址值达成对实参的修改，但传指针是显式的，而传引用是编译器隐式处理的。
指针和引用在内存中的示意图：
![image-20221208164349783](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212081643838.png)

![](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212081644997.png)

指针和引用的应用比较：
引用比指针使用起来形式上更为美观，使用引用指向的内容时可以之间用引用变量名，而不像指针一样要使用*；定义引用的时候也不用像指针一样使用&取址。
引用比指针更安全。由于不存在空引用，并且引用一旦被初始化为指向一个对象，就不能被改变为另一个对象的引用，因此引用很安全。对于指针来说，它可以随时指向别的对象，并且可以不被初始化，或为NULL，所以不安全。并且有可能产生野指针（即多个指针指向一块内存，free掉一个指针之后，别的指针就成了野指针)。

# 常量关键字const
## 定义常量
常量：不可被修改的内存单元
    const int MAX_VAL = 23；
    const string SCHOOL_NAME = "Peking University"；
## 定义常引用
定义引用时，前面加const关键字，即为“常引用”。不能**通过常引用修改**其引用的变量，但可直接修改变量的值，引用本身也不能改变

    int n;
    const int & r = n;
    r = 5; //error
    n = 4; //ok
**const T & 和T & 是不同的数据类型!!!**
T & 类型的引用或T类型的变量可以用来初始化const T & 类型的引用，const T 类型的常变量和const T & 类型的引用则不能用来初始化T &类型的引用，除非进行强制类型转换
一句话，常指针和常引用不能出现在“=”左边

## 定义常指针
常指针也叫常量指针。但指针不是常量，指向的也不是常量，只是限制了改写方式：不可**通过常量指针修改**其指向变量的值，但可直接修改变量的值，也可以改变常量指针的指向地址值。

    int n,m;
    const int * p = & n;
    * p = 5; //编译出错
    n = 4; //ok
    p = &m; //ok, 常量指针的指向可以变化
不能把常量指针赋值给非常量指针，反过来可以

    const int * p1; int * p2;
    p1 = p2; //ok
    p2 = p1; //error
    p2 = (int * ) p1; //ok,强制类型转换
函数参数为常量指针时，可避免函数内部不小心改变参数指针所指地方的内容

    void MyPrintf( const char * p )
    {
    strcpy( p,"this"); //编译出错
    printf("%s",p); //ok
    }
## 定义指针常量
定义：本质是一个不可修改指向地址的指针 

    int* const p;
## 定义指向常量的常指针
定义：指针指向的地址值不可修改，且该地址中的值也不可修改

    const int* const p;
# 动态内存分配
动态内存分配是分配内存空间中堆（heap）的内存，实际上是程序内手动的内存分配与释放。并非堆栈中局部变量的入栈出栈，由操作系统控制的动态分配。
## new分配内存
分配一个变量:

    P = new T;

T是任意类型名， P是类型为T * 的指针。
动态分配出一片大小为 sizeof(T)字节的内存空间，并且将该内存空间的起始地址赋值给P：

    int * pn = new int;
    * pn = 5;
分配一个数组：

    P = new T[N];

T :任意类型名
P :类型为T * 的指针
N :要分配的数组元素的个数，可以是整型表达式
动态分配出一片大小为 sizeof(T)*N字节的内存空间，并且将该内存空间的起始地址赋值给P

    int * pn;
    int i = 5;
    pn = new int[i * 20];
    pn[0] = 20;
    pn[100] = 30; //编译没问题。运行时导致数组越界

## delete释放内存
用“new”动态分配的内存空间用完后，一定要用“delete”运算符进行释放，否则操作系统无法再次使用这块内存，造成内存泄露
注意：不能对内存空间delete两次！

    #delete 指针； //该指针必须指向new出来的空间
    int * p = new int;
    * p = 5;
    delete p;
    delete p; //导致异常， 一片空间不能被delete多次

用“delete”释放动态分配的数组，要加“[]”

    #delete [] 指针； //该指针必须指向new出来的数组
    int * p = new int[20];
    p[0] = 1;
    delete [] p;

# 内联函数
普通函数：编译出来的可执行程序加载到内存后，代码段只有一份函数的指令序列，函数的调用处就用一个类似jump的语句跳转到函数指令序列的入口地址
内联函数：函数的每个调用处都存在整个函数指令序列的拷贝
简单讲就是增加编译出来的代码占用空间，换取运行时频繁入栈出栈的时间开销
使用场景：简单函数体且多次调用可以定义为内联
函数调用是有时间开销的。如果函数本身只有几条语句，执行非常快，而且函数被反复执行很多次，相比之下调用函数所产生的这个开销就会显得比较大。为了减少函数调用的开销，引入了内联函数机制。编译器处理对内联函数的调用语句时，是将整个函数的代码插入到调用语句处，而不会产生调用函数的语句。

    inline int Max(int a,int b)
    {
    if( a > b) return a;
    return b;
    }
# 函数重载
## 函数重载概念
重载不是重新载入，更贴切的含义是重复定义，因为重定义是种错误，重载可以理解为编译器能理解的“重定义”，因此能正常加载。
C++重载主要有：

 - 函数重载
 - 运算符重载

C++的类没有重载一说，本节讲函数重载
函数重载：一个或多个函数，名字相同，然而参数个数或参数类型不相同，这叫做函数的重载。

    int Max(double f1,double f2) { }
    int Max(int n1,int n2) { }
    int Max(int n1,int n2,int n3) { }
Q1.重载有什么用？
C语言定义以上几个函数，不能用同名，但是其功能都是相同的，仅参数类型和值不同。如果用MaxDouble(),MaxInt2(),MaxInt3()过于麻烦。
因此函数重载使得函数命名变得简单。
Q2.编译器怎么知道调用的是哪个？
编译器根据调用语句的中的实参的个数和类型判断应该调用哪个函数，注意重载函数不会把入参自动类型转换，调用二义性会报错。

    Max(3.4,2.5); //调用 (1)
    Max(2,4); //调用 (2)
    Max(1,2,3); //调用 (3)
    Max(3,2.4); //error,二义性
Q3.函数仅返回值类型不同是不是重载？
不是，函数重载的区分在于入参。但是有个例外，返回const T和非const T的两个函数是是重载的，其他情况的入参相同，返回类型不同的函数，视为重定义。
## 缺省参数与可拓展性
C++函数支持缺省参数（默认参数值）。定义函数的时候可以让最右边的连续若干个参数有缺省值，那么调用函数的时候，若相应位置不写参数，参数就是缺省值。

    void func( int x1, int x2 = 2, int x3 = 3)
    { }
    func(10 ) ; //等效于 func(10,2,3)
    func(10,8) ; //等效于 func(10,8,3)
    func(10, , 8) ; //不行,只能最右边的连续若干个参数缺省

函数参数可缺省的目的在于提高程序的可扩充性。
如果某个写好的函数要添加新的参数，而原先那些调用该函数的语句，未必需要使用新增的参数，那么为了避免对原先那些函数调用语句的修改，就可以使用缺省参数。
在C语言中，如果函数新增一个入参，所有调用该函数的地方都要传入该入参值；C++支持缺省参数，只需要改函数定义即可，调用处不需要动。
