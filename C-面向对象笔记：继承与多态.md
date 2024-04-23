---
title: C++面向对象笔记：继承与多态
date: 2020-03-20 16:49:06
tags: C++
categories: C++
---

# 0.概述
前文分析了C++类内成员的关系，本文讨论类和类之间的关系。
考虑用C++对现实世界的交通工具进行描述。

 - 汽车可能包含各种类型，小汽车，公交车，但他们能抽象出四个轮子，烧油这些基本属性
 - 飞机也有各种类型，但也能抽象出机翼，机身等基本属性
 - 轮船...

如果自顶向下设计，如何设计这些对象的类？

 - 提炼这些交通工具的共有属性，如材质，耗油量，价格，设计成一个交通工具基础类；然后设计一些操作方法，比如制造，启动，停止。
 - 分别设计汽车、飞机、轮船等更具体的类的属性，比如轮子、排水量等，注意，他们也包含基础类的材质，耗油量，价格等基本属性；然后也设计一些方法，比如制造汽车、开汽车和造飞机、开飞机等
 - 然后再设计更细节的类，作为汽车、飞机、轮船类的细化，比如A品牌的汽车，B品牌汽车，作为两个具体类。

仔细考虑以上步骤，有以下问题：

 - 这些类的属性（成员变量）是相互独立的吗？
 - 这些类的方法（成员函数）是相互独立的吗？

C++用类的“继承”描述层层细化的类及其成员变量的关系，用“多态”描述各层方法的实现关系。
# 类的继承
## 继承关系的概念
继承：在定义一个新的类B时，如果该类与某个已有的类A相似(指的是B拥有A的全部特点)，那么就可以把A作为一个**基类**（也叫父类），而把B作为基类的一个**派生类**(也叫子类)。

 - 派生类是通过对基类进行修改和扩充得到的。在派生类中，可以扩充新的成员变量和成员函数
 - 派生类一经定义后，可以独立使用，不依赖于基类。
 - 派生类拥有基类的全部成员函数和成员变量，不论是private、 protected、 public。但是派生类的成员函数不能访问基类中的private成员

一个管理学生的类继承：
![image-20221208165130260](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081651314.png)
派生类语法:

    class 派生类名： public 基类名
    {
    };

学生类的派生:

    class CStudent {
        private:
        string sName;
        int nAge;
        public:
        bool IsThreeGood() { };
        void SetName( const string & name )
        { sName = name; }
            //......
    };
    
    class CUndergraduateStudent: public CStudent {
        private:
        int nDepartment;
        public:
        bool IsThreeGood() { ...... }; //覆盖
        bool CanBaoYan() { .... };
    }; // 派生类的写法是：类名: public 基类名

## 类继承的存储空间

在类与对象一文讲过，类对象的存储空间，实际就是成员变量的空间，成员函数不在对象空间内（虚函数包含一个虚函数表指针）。那么基类和派生类的对象空间有什么相关性？
派生类对象的体积，等于基类对象的体积，再加上派生类对象自己的成员变量的体积。 在派生类对象中，包含着基类对象，而且基类对象的存储位置位于派生类对象新增的成员变量之前。
一个示例：

    class CBase
    {
        int v1, v2;
    };
    class CDerived:public CBase
    {
        int v3;
    };
![image-20221208165140659](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081651715.png)

## 类继承的覆盖
类内的同名非同参的函数叫函数重载，那么基类与派生类的同名函数呢？
派生类可以定义和基类成员同名的成员，这叫**覆盖**。在派生类中访问这类成员时，默认访问派生类中定义的成员，基类的成员函数或变量被“覆盖”掉了。如果要在派生类中访问基类定义的同名成员时，要使用作用域符号::
一个例子：

    class base {    //基类
        int j;  //默认private
        public:
        int i;
        void func();
    };
    class derived : public base{    //派生类
        public:
        int i;  //覆盖基类i
        void access();
        void func(); //覆盖基类func()
    };
    
    void derived::access() { //访问派生类成员
        j = 5; //error
        i = 5; //引用的是派生类的 i
        base::i = 5; //引用的是基类的 i
        func(); //派生类的
        base::func(); //基类的
    }
调用函数:

    derived obj;
    obj.i = 1;  //访问派生类成员i
    obj.base::i = 1; //访问基类成员i

内存分布:
![image-20221208165150851](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081651894.png)
以上只是示例，一般来说，基类和派生类不定义同名成员变量，但经常有同名成员函数，所以覆盖通常用于成员函数覆盖。

## 类继承的成员访问控制

 - 基类的private成员：可以被下列函数访问
– 基类的成员函数
– 基类的友元函数
 - 基类的public成员：可以被下列函数访问
– 基类的成员函数
– 基类的友元函数
– 派生类的成员函数
– 派生类的友元函数
– 其他的函数
 - 基类的protected成员：可以被下列函数访问
– 基类的成员函数
– 基类的友元函数
– 派生类的成员函数可以访问当前对象的基类的保护成员

一个示例：

    class Father {
        private: int nPrivate; //私有成员
        public: int nPublic; //公有成员
        protected: int nProtected; // 保护成员
    };
    class Son :public Father{
        void AccessFather () {
            nPublic = 1; // ok;
            nPrivate = 1; // wrong
            nProtected = 1; // OK，访问从基类继承的protected成员
            Son f;
            f.nProtected = 1; //wrong ， f不是当前对象
        }
    };
    
    int main()
    {
        Father f;
        Son s;
        f.nPublic = 1; // Ok
        s.nPublic = 1; // Ok
        f.nProtected = 1; // error
        f.nPrivate = 1; // error
        s.nProtected = 1; //error
        s.nPrivate = 1; // error
        return 0;
    }
## 类继承的构造函数
类似于嵌套类（封闭类）的构造函数，使用初始化列表来实现层层构造，基类和派生类只初始化他们能访问的成员

    class Bug {
    private :
        int nLegs; int nColor;
        public:
        int nType;
        Bug ( int legs, int color);
        void PrintBug (){ };
    };
    
    class FlyBug: public Bug // FlyBug是Bug的派生类
    {
        int nWings;
        public:
        FlyBug( int legs,int color, int wings);
    };
    
    Bug::Bug( int legs, int color) //Bug类的构造函数
    {
        nLegs = legs;
        nColor = color;
    }
    
    //错误的FlyBug构造函数！
    FlyBug::FlyBug ( int legs,int color, int wings)
    {
        nLegs = legs; // 不能访问
        nColor = color; // 不能访问
        nType = 1; // ok
        nWings = wings;
    }
    
    //正确的FlyBug构造函数：使用初始化列表
    FlyBug::FlyBug ( int legs, int color, int wings):Bug( legs, color)
    {
        nWings = wings;
    }
    
    int main() {
        FlyBug fb ( 2,3,4);
        fb.PrintBug();
        fb.nType = 1;
        fb.nLegs = 2 ; // error. nLegs is private
        return 0;
    }
## 类继承的构造析构时序
在创建派生类的对象时，需要调用基类的构造函数：初始化派生类对象中从基类继承的成员。在执行一个派生类的构造函数之前，总是先执行基类的构造函数。
调用基类构造函数的两种方式:

 - 显式方式：在派生类的构造函数中，为基类的构造函数提供参数.

    derived::derived(arg_derived-list):base(arg_base-list)

 - 隐式方式：在派生类的构造函数中，省略基类构造函数时，派生类的构造函数则自动调用基类的默认构造函数

析构函数执行时序:
派生类的析构函数被执行时，执行完派生类的析构函数后，自动调用基类的析构函数。
一个例子：

    class Base {
        public:
        int n;
        Base(int i):n(i)
        { cout << "Base " << n << " constructed" << endl;}
        ~Base()
        { cout << "Base " << n << " destructed" << endl; }
    };
        
    class Derived:public Base {
        public:
        Derived(int i):Base(i)
        { cout << "Derived constructed" << endl; }
        ~Derived()
        { cout << "Derived destructed" << endl;}
    };
    int main() { Derived Obj(3); return 0; }

输出结果:

    Base 3 constructed
    Derived constructed
    Derived destructed
    Base 3 destructed
##封闭派生类的构造函数
封闭类的构造用初始化列表，派生类也用初始化列表，那么封闭派生类呢？
还是初始化列表

    class Bug {
        private :
        int nLegs; int nColor;
        public:
        int nType;
        Bug ( int legs, int color);
        void PrintBug (){ };
    };
    
    class Skill {
        public:
        Skill(int n) { }
    };
    class FlyBug: public Bug {
        int nWings;
        Skill sk1, sk2;
        public:
        FlyBug( int legs, int color, int wings);
    };
    FlyBug::FlyBug( int legs, int color, int wings):
        Bug(legs,color),sk1(5),sk2(color) ,nWings(wings) { //初始化列表，不能访问的通通交给下层构造函数
    }

## 封闭派生类的构造析构时序
在创建派生类的对象时:
1) 先执行基类的构造函数，用以初始化派生类对象中从基类继承的成员
2) 再执行成员对象类的构造函数，用以初始化派生类对象中成员对象
3) 最后执行派生类自己的构造函数
在派生类对象消亡时：
1) 先执行派生类自己的析构函数
2) 再依次执行各成员对象类的析构函数
3) 最后执行基类的析构函数
析构函数的调用顺序与构造函数的调用顺序相反
# 类的复合
在数学上，两个集合有无关、相交和包含的关系。对于多个类来说，也应该有以上三种关系。无关类=两个成员不相关的类；继承类=类成员间有继承关系的类；那么相交的类呢？
## 复合关系的概念
C++用“复合”表示类的相交关系。
1)继承：“是”的关系
基类是A， B是基类A的派生类，逻辑上要求：“一个B对象也是一个A对象”
2)复合：“有”的关系
类C中“有” 成员变量k，k是类D的对象，则C和D是复合关系，逻辑上要求：“D对象是C对象的固有属性或组成部分

下面比较一下继承和复合在具体设计的实例：
继承关系顶层设计例子:

 - 写了一个 CMan 类代表男人
 - 后来又发现需要一个CWoman类来代表女人
 - CWoman类和CMan类有共同之处,让CWoman类从CMan类派生而来，是否合适？
 - 错！从一开始就应该设计CHuman类，代表“人” ,然后CMan和CWoman都从
CHuman派生

继承逻辑关系：
![image-20221208165210195](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081652246.png)

复合关系顶层设计例子：

 - 几何形体程序中，需要写“点”类，也需要写“圆”类
 - 每个圆都有圆心，那么点类应该从圆类派生出来吗？
 - 错！”点“不仅在圆内有，在其他图形也有，不是圆独有，非继承关系
 - 实际上，圆和点是复合关系，每一个“圆”对象里都包含(**有**)一个“点”对象
 - 逻辑上，复合关系就是，我的一部分可以看成是你的，但是我的全部东西不都属于你

复合关系的类通常用友元实现：

    class CPoint
    {
        double x,y;
        friend class CCircle;
        //便于Ccirle类操作其圆心
    };
    
    class CCircle
    {
        double r;
        CPoint center;
    };
## 复合关系的典型示例
如果要写一个小区养狗管理程序，需要写一个“业主”类，还需要写一个“狗” 类
狗是归宿于业主的，一个业主可以有多条狗，狗也可以随时脱离业主
考虑以下设计方法：
设计人和狗两个类，相互包含对方类

    class CDog;
    class CMaster
    {
        CDog dogs[10];
    };
    class CDog
    {
        CMaster m;
    };
 这样有循环定义错误！且逻辑上，狗和人并非相互包含关系
 这种关系上相互相关，对象本身又完全独立的情况，用对象指针表示

    class CMaster; //CMaster必须提前声明，不能先写CMaster类后写Cdog类
    class CDog {
        CMaster * pm;
    };
    class CMaster {
        CDog * dogs[10];
    };
逻辑关系:
![image-20221208165220338](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081652390.png)

# 继承的不足
## 继承方式的访问限制
基类和派生类是包含的关系，那么基类对象和派生类对象是什么关系？
对于类的public派生方式:

    class base { };
    class derived : public base { };
    base b;
    derived d;
1）派生类的对象可以赋值给基类对象
b = d;
2）派生类对象可以初始化基类引用
base & br = d;
3）派生类对象的地址可以赋值给基类指针
base * pb = & d;
如果派生方式是 private或protected，则上述三条不可行

对于类的protected和private派生方式:

    class base {};
    class derived : protected base {};
    base b;
    derived d;

• protected继承时，基类的public成员和protected成员成为派生类的protected成员。
• private继承时，基类的public成员成为派生类的private成员，基类的protected成员成为派生类的不可访问成员。
• protected和private继承不是“是”的关系
## 派生类的对象指针转换
public派生的情况下,派生类对象的指针可以直接赋值给基类指针

    Base * ptrBase = &objDerived;
    //ptrBase指向的是一个Derived类的对象；

*ptrBase可以看作一个Base类的对象，访问它的public成员直接通过ptrBase即可，但不能通过ptrBase访问objDerived对象中属于Derived类而不属于Base类的成员
过强制指针类型转换，可以把ptrBase转换成Derived类的指针

    Base * ptrBase = &objDerived;
    Derived *ptrDerived = (Derived * ) ptrBase;

程序员要保证ptrBase指向的是一个Derived类的对象，否则很容易会错

派生类的指针赋值给基类后，基类指针也不能访问派生类的特有成员

    #include <iostream>
    using namespace std;
    class Base {
        protected:
        int n;
        public:
        Base(int i):n(i){cout << "Base " << n <<" constructed" << endl; }
        ~Base() {cout << "Base " << n <<" destructed" << endl;}
        void Print() { cout << "Base:n=" << n << endl;}
    };
    
    class Derived:public Base {
        public:
        int v;
        Derived(int i):Base(i),v(2 * i) {
        cout << "Derived constructed" << endl;
    }
    
    ~Derived() {
        cout << "Derived destructed" << endl;
    }
    
    void Func() { } ;
        void Print() {
            cout << "Derived:v=" << v << endl;
            cout << "Derived:n=" << n << endl;
        }
    };
    
    int main() {
        Base objBase(5);
        Derived objDerived(3);
        Base * pBase = & objDerived ;
        //pBase->Func(); //err;Base类没有Func()成员函数
        //pBase->v = 5; //err; Base类没有v成员变量
        pBase->Print();
        //Derived * pDerived = & objBase; //error
        Derived * pDerived = (Derived *)(& objBase);
        pDerived->Print(); //慎用，可能出现不可预期的错误
        pDerived->v = 128; //往别人的空间里写入数据，会有问题
        objDerived.Print();
        return 0;
    }

输出：

    Base 5 constructed
    Base 3 constructed
    Derived constructed
    Base:n=3
    Derived:v=1245104 //pDerived->n 位于别人的空间里
    Derived:n=5
    Derived:v=6
    Derived:n=3
    Derived destructed
    Base 3 destructed
    Base 5 destructed

从逻辑上来说，派生类指针既然能被赋值给基类指针，那么通过基类指针，应该能调用派生类的成员函数，获取派生类的成员变量。在下一章，继承类的多态将实现这个目的。
## 多级继承
类A派生类B，类B派生类C，类C派生类D……
– 类A是类B的直接基类
– 类B是类C的直接基类，类A是类C的间接基类
– 类C是类D的直接基类，类A、 B是类D的间接基类
在声明派生类时， 只需要列出它的直接基类
– 派生类沿着类的层次自动向上继承它的间接基类
– 派生类的成员包括
• 派生类自己定义的成员
• 直接基类中的所有成员
• 所有间接基类的全部成员

# 多态：在继承上更进一步
前面派生类的对象指针转换一节，基类指针强转后也不能访问派生类私有对象。考虑一下本文开始讲的交通工具顶层设计思路，在顶层设计时就要设计类的成员函数，在派生类也要设计成员函数，这些函数会有重合的情况吗？如果有重合，基类指针也不能访问派生类成员，这样基类和派生类不就失去联系了吗？多级继承这种情况不是更加严重？
为了解决这种问题，本节引入继承类的“多态”
多态能够增强程序的可扩充性，即程序需要修改或增加功能的时候，需要改动和增加的代码较少
## 虚函数与多态
在类的定义中，前面有 virtual 关键字的成员函数就是虚函数

    class base {
        virtual int get() ;
    };
    int base::get(){ }

virtual关键字只用在类定义里的函数声明中使用，定义函数体时不用。
使用虚函数，来实现“多态”效果。多态有通过指针和引用两种表现形式:

 - 能通过基类的指针调用派生类虚函数，访问其特有成员变量

派生类的指针可以赋给基类指针
通过基类指针调用基类和派生类中的同名虚函数时:
（1）若该指针指向一个基类的对象，那么被调用是
基类的虚函数；
（2）若该指针指向一个派生类的对象，那么被调用
的是派生类的虚函数

    class CBase {
    public:
        virtual void SomeVirtualFunction() { }
    };
    class CDerived:public CBase {
    public :
        virtual void SomeVirtualFunction() { }
    };
    int main() {
        CDerived ODerived;
        CBase * p = & ODerived;
        p -> SomeVirtualFunction(); //调用哪个虚函数取决于p指向哪种类型的对象
        return 0;
    } 

   - 能通过基类的引用调用派生类虚函数

派生类的对象可以赋给基类引用
通过基类引用调用基类和派生类中的同名虚函数时:
（1）若该引用引用的是一个基类的对象，那么被调
用是基类的虚函数；
（2）若该引用引用的是一个派生类的对象，那么被
调用的是派生类的虚函数。

    class CBase {
    public:
        virtual void SomeVirtualFunction() { }
    };
    class CDerived:public CBase {
    public :
        virtual void SomeVirtualFunction() { }
    };
    int main() {
        CDerived ODerived;
        CBase & r = ODerived;
        r.SomeVirtualFunction(); //调用哪个虚函数取决于r引用哪种类型的对象
        return 0;
    } 

是不是所有成员函数加virtual都是多态？不是！

 - 在非构造或析构函数的成员函数中调用虚函数，是多态。在运行时才确定到底调用哪一层派生类函数
 - 在构造函数和析构函数中调用虚函数，不是多态。调用的函数是当前类的函数，编译时即确定

多层继承实现多态，每一层都要加virtual关键字吗？

 - 派生类中和基类中虚函数同名同参数表的函数，不加virtual也自动成为虚函数

## 多态与对象指针
一个变量有两方面属性：类型、值
那么多态把derived类的地址值，赋值给base类的指针，访问对象成员时是什么效果？
以下例子的this指针指向什么？

    class Base {
    public:
        void fun1() { this->fun2(); } //this是基类指针， fun2是虚函数，所以是多态
        virtual void fun2() { cout << "Base::fun2()" << endl; }
    };
    class Derived:public Base {
    public:
        virtual void fun2() { cout << "Derived:fun2()" << endl; }
    };
    int main() {
        Derived d;
        Base * pBase = & d;
        pBase->fun1();
        return 0;
    }
pBase被Derived对象的地址赋值后，其值为Derived对象的地址，但类型还是Base的指针（多态指针赋值不会强转）。pBase->fun1()会先在Base类访问其fun1()，传入this指针（指向fun2）,而this->fun2()会调用Derived类的fun2()
输出： 

    Derived:fun2()

虚函数也可以定义为private：

    class Base {
    private:
        virtual void fun2() { cout << "Base::fun2()" << endl; }
    };
    class Derived:public Base {
    public:
        virtual void fun2() { cout << "Derived:fun2()" << endl; }
    };
    Derived d;
    Base * pBase = & d;
    pBase -> fun2(); // 编译出错
pBase已经被赋值为指向derived d的指针，不能调用base类的private函数。
## 多态的实例:游戏开发
游戏中有很多种怪物，每种怪物都有一个类与之对应。某个玩家创建的具体怪物就是对象
怪物的主要动作（成员函数）有：

 - 攻击（Attack），针对不同的被攻击者有不同的函数
 - 反击（FightBack），被某个怪物攻击时做出的相应动作
 - 掉血（Hurted），被攻击时会掉血，血量值不同有不同处理，如死亡

现在的需求是：已经有CWolf、CGhost两种怪物，需要设计新的怪物CThunderBird，并能满足和其他怪物的交互
顶层设计:
设置基类 CCreature，并且使CDragon, CWolf等其他类都从CCreature派生而来
![image-20221208165241029](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081652097.png)
非多态的派生类设计：
由于每个怪物对于其他怪物的攻击和反击都是不同的，每个怪物类都要设计一组Attack和FightBack：

    class class CCreature {
        protected: int nPower ; //代表攻击力
        int nLifeValue ; //代表生命值
    };
    class CThunderBird : public CCreature {
        public:
        void Attack(CWolf * pWolf) {
            ．．．表现攻击动作的代码
            pWolf->Hurted( nPower);
            pWolf->FightBack( this);
        }
        void Attack( CDragon * pDragon) {
            ．．．表现攻击动作的代码
            pDragon->Hurted( nPower);
            pDragon->FightBack( this);
        }
        void FightBack( CWolf * pWolf) {
            ．．．．表现反击动作的代码
            pWolf ->Hurted( nPower / 2);
        }
        void FightBack( CDragon * pDragon) {
            ．．．．表现反击动作的代码
            pDragon->Hurted( nPower / 2 );
        }
        void Hurted ( int nPower) {
            ．．．．表现受伤动作的代码
            nLifeValue -= nPower;
        }
    }

现有n种怪物，CThunderBird类中就得有n个Attack 和n个FightBack成员函数，对于其他类也得新增针对CThunderBird的Attack和FightBack。这种设计工作量过于巨大。原因就在于要区分传入的对象指针。
那么能否传入基类的指针呢，这样就不存在为各种类型写几个函数。基类指针要访问派生类的成员，得用虚函数形成多态。多态实现如下：

    //基类 CCreature：
    class CCreature {
    protected :
        int m_nLifeValue, m_nPower;
        public:
        virtual void Attack( CCreature * pCreature) {}
        virtual void Hurted( int nPower) { }
        virtual void FightBack( CCreature * pCreature) {}
    };
    //派生类 CDragon:
    class CDragon : public CCreature {
    public:
        virtual void Attack( CCreature * pCreature);
        virtual void Hurted( int nPower);
        virtual void FightBack( CCreature * pCreature);
    };
    
    //派生类的成员函数实现具体操作
    void CDragon::Attack(CCreature * p) //传入基类指针
    { …表现攻击动作的代码
        p->Hurted(m_nPower); //多态
        p->FightBack(this); //多态
    }
    void CDragon::Hurted( int nPower)
    { …表现受伤动作的代码
        m_nLifeValue -= nPower;
    }
    void CDragon::FightBack(CCreature * p)
    { …表现反击动作的代码
        p->Hurted(m_nPower/2); //多态
    }
    
    //多态的调用
    CDragon Dragon; CWolf Wolf; CGhost Ghost;
    CThunderBird Bird；
    Dragon.Attack( & Wolf); //调用CWolf::Hurted
    Dragon.Attack( & Ghost); //调用CGhost::Hurted
    Dragon.Attack( & Bird); //调用CBird::Hurted
使用多态，新增某个派生类时，已有的类可以原封不动，因为传入基类指针，会“自动”调用正确的派生类函数，开发者只需要设计新增的派生类和其成员函数即可

## 多态的原理：虚函数表指针
多态” 的关键在于通过基类指针或引用调用一个虚函数时，编译时不确定到底调用的是基类还是派生类的函数，运行时才确定，这叫“动态联编” 
首先分析包含虚函数的类对象的内存分布：

    class Base {
    public:
    int i;
        virtual void Print() { cout << "Base:Print" ; }
    };
    
    class Derived : public Base{
    public:
    int n;
        virtual void Print() { cout <<"Drived:Print" << endl; }
    };
    
    int main() {
        Derived d;
        cout << sizeof( Base) << ","<< sizeof( Derived ) ;
        return 0;
    }

输出：8, 12
为什么类对象的size比成员变量int（4字节）还多4字节？
因为包含虚函数的基类，实例化的对象除了成员变量，还包含一个指针（一般4字节），指向虚函数的入口地址，如果有多个虚函数，这些地址连续排列形成虚函数表，指针指向首个虚函数地址。如果这个指针指向基类，就能找到基类的所有虚函数入口，如果指针指向派生类，就能找到派生类的的所有虚函数入口。基类和派生类对象的指针赋值，实际会导致虚函数表指针指向的虚函数入口地址不同，从而调用时不同。
如果当前指针指向基类，则调用基类自己的虚函数：

    Base b;
    pBase = &b;
    pBase->Print();

![image-20221208165259927](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081652980.png)
如果当前指针指向派生类，则调用派生类的虚函数：

    Derived d;
    pDerived = &d;
    pBase = pDerived;
    pBase->Print();

![image-20221208165311145](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081653211.png)
动态联编的实现：
多态的函数调用语句被编译成一系列根据基类指针所指向的（或基类引用所引用的)对象中存放的虚函数表的地址，在虚函数表中查找虚函数地址，并调用虚函数的指令
而普通函数是编译过程中确定了成员函数的入口地址，不存在运行时根据对象来改变某个函数的入口地址。

## 虚函数与抽象类
可以想象得到，前文的游戏使用虚函数的例子是通用的，先设计基类，提炼对象属性，定义虚函数；再派生子类，在子类实现局函数的具体操作。那么问题来了，基类的虚函数有必要实现函数体吗？
很多情况，基类只是一个抽象，定义了函数的名称和参数，不需要在基类实现虚函数，全部交给派生类实现。

 - 纯虚函数：没有函数体的虚函数
 - 抽象类：包含纯虚函数的类

纯虚函数写法：没函数体{}，直接=0

    class A {
    private: int a;
    public:
        virtual void Print( ) = 0 ; //纯虚函数
        void fun() { cout << "fun"; }
    };

抽象类特点：

 - 抽象类只能作为基类来派生新类使用，不能创建抽象类的对象
 - 可以创建抽象类的指针和引用，它们可以指向派生类的对象

抽象类的指针：

    A a ; // 错， A 是抽象类，不能创建对象
    A * pa ; // ok,可以定义抽象类的指针和引用
    pa = new A ; //错误, A 是抽象类，不能创建对象

如果一个类从抽象类派生而来，那么当且仅当它实现了基类中的所有纯虚函数，它才能成为非抽象类
抽象类的成员函数可以调用纯虚函数，但是构造函数或析构函数内不能调用纯虚函数

    class A {
    public:
        virtual void f() = 0; //纯虚函数
        void g( ) { 
            this->f( ) ; //ok
        }
        A( ){ 
            f( ); // 错误
        }
    };
    class B:public A{
    public:
        void f(){cout<<"B:f()"<<endl; }
    };

## 虚函数与构造析构函数
前面考虑了普通成员函数加virtual，可以形成虚函数达到继承类的多态效果。那么构造函数和析构函数呢？

 - 不允许以虚函数作为构造函数
 - 类继承需要把基类的析构函数设为虚函数

对于常规析构函数，通过基类指针删除派生类对象时，只能调用基类的析构函数。但是合理的做法是，应该先调用派生类的析构函数，然后调用基类的析构函数。解决的方法：把析构函数定义为virtual，由于基类析构函数是虚函数，派生类的同名析构函数自然也是虚函数。
什么时候定义虚析构函数

 - 一个类只要定义了虚函数，则应该将析构函数也定义成虚函数
 - 一个类打算作为基类使用，则应该将析构函数定义成虚函数

虚析构函数用法：通过基类的指针删除派生类对象，会首先调用派生类的析构函数，然后调用基类的析构函数

    class son{
    public:
        virtual ~son() {cout<<"bye from son"<<endl;};
    };
    class grandson:public son{
    public:
        ~grandson(){cout<<"bye from grandson"<<endl;};
    };
    int main() {
        son *pson;
        pson= new grandson(); //pson指向派生类grandson
        delete pson;
        return 0;
    }

输出： 

    bye from grandson
    bye from son
