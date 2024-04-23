---
title: C++面向对象笔记：类和对象
date: 2020-03-18 16:44:59
tags: C++
categories: C++
---

# 0.概述
C++，加的到底是什么？
除了基础语法的补充和优化，C++另外几个核心特点是：

 - 面向对象设计的支持：

    类和对象对变量和函数的封装
    类和类之间的继承
    继承关系的类之间的函数调用的多态
 - 数据结构和算法的支持
    STL和各种常用数据类型

 - 高可复用、可拓展的支持
    类模板，函数模板
    函数、运算符的重载

本文内容：

 - 面向对象设计的概念
 - 类和对象的概念及使用
 - 类的几种构造函数
 - 类的析构函数
 - 类对象的this指针
 - 类的嵌套：封闭类
 - 成员的属性：友元和常量成员

# 面向对象设计的概念
## 面向过程设计的不足
程序 = 数据结构 + 算法
程序由全局变量以及众多相互调用的函数组成，算法以函数的形式实现，用于对数据结构进行操作。
结构化程序设计风格中，变量和函数的关系:
![image-20221208164648489](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081646566.png)
其缺陷在于：

 - 结构化程序设计中，函数和其所操作的数据结构，没有直观的联系
 - 随着程序规模的增加，程序逐渐难以理解:
        某个数据结构到底有哪些函数可以对它进行操作?
        某个函数到底是用来操作哪些数据结构的?
        任何两个函数之间存在怎样的调用关系?
 - 结构化程序设计难以维护:
由于没有“封装”和“隐藏”的概念，要访问某个数据结构中的某个变量，就可以直接访问，那么当该变量的定义有改动的时候，就要把所有访问该变量的语句找出来修改，不利于程序的维护、扩充。
 - 结构化程序设计难以查错:
当某个数据结构的值不正确时，难以找出到底是那个函数导致的。
 - 结构化程序设计难以重用：
在编写某个程序时，发现其需要的某项功能，在现有的某个程序里已经有了相同或类似的实现，那么自然希望能够将那部分代码抽取出来，在新程序中使用。在结构化程序设计中，随着程序规模的增大，由于程序大量函数、变量之间的关系错综复杂，要抽取这部分代码，会变得十分困难。
## 面向对象的程序设计
面向对象的程序设计方法，能够较好解决上述问题
面向对象的程序 = 类 + 类 + …+ 类
设计程序的过程，就是设计类（class）的过程
面向对象的程序设计方法:
 - 将某类客观事物共同特点（**属性**）归纳出来，形成一个数据结构（可以用多个变量描述事物的属性）
 - 将这类事物所能进行的**行为**也归纳出来，形成一个个函数，这些函数可以用来操作数据结构(这一步叫“ **抽象**”）
 - 然后，通过某种语法形式，将数据结构和操作该数据结构的函数“捆绑”在一起，形成一个“ 类”，从而使得数据结构和操作该数据结构的算法呈现出显而易见的紧密关系，这就是“**封装**”
 - 类与类直接又形成**继承、多态**等关系
 - 面向对象的程序设计具有“抽象”，“封装”“继承”“多态”四个基本特点。

面向对象设计风格中，变量和函数的关系;
![image-20221208164703654](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081647726.png)

## 语言和风格的无关性
注意面向过程、面向对象以及其他的风格（如函数式编程等），只是编程风格，其本质都是组织数据结构（事物属性）和算法（对事物的操作）。
C++有原生的类的概念，更方便写出面向对象风格的程序
Q. C语言没有类，能不能写出面向对象？
可以，C的结构体就是对数据的封装，配合函数指针，也能包含函数成员。利用带函数指针的结构体能实现属性和方法的封装，在Linux内核和设备驱动程序中充满了这种面向对象设计风格。事实上，C++的class在编译器处理后就是类似于C的结构体。
Q. 什么时候应该面向对象？
面向对象对于人的抽象概括的能力要求较高，需要花较多精力在top-down的顶层设计中，通常用于大型的长期维护的程序设计。
面向对象的优势在于数据结构组织化，程序时间和空间的开销可能不如面向过程。例如一个对象里的各个数据的生命周期都是捆绑分配和释放的，而面向过程可以更精细管理。在极端资源紧缺的情况，如部分嵌入式开发，面向过程不论代码设计速度和性能都比面向对象好。

# 类和对象的概念
## 类和对象的定义
设计一个程序，接受输入矩形的长和宽，输出面积和周长
如何用类来封装？

 - 矩形的属性就是长和宽。因此需要两个变量，分别代表长和宽
 - 矩形的操作方法可以有设置长和宽，算面积，算周长。每个操作各用一个函数来实现，且函数都需要用到长和宽这两个属性
 - 将以上属性和方法组合就能形成一个“矩形类”。长、宽变量成为该“矩形类”的“成员变量”，三个函数成为该类的“成员函数”。成员变量和成员函数统称为类的成员。“类”看上去就像“带函数的结构”

类的声明：

    class CRectangle
    {
        public:
            int w, h;
            int Area() {
            return w * h;
        }
        int Perimeter(){
            return 2 * ( w + h);
        }
        void Init( int w_,int h_ ) {
            w = w_; h = h_;
        }
    }; //必须有分号

类的实例化：

    int main( )
    {
        int w,h;
        CRectangle r; //r是一个对象
        cin >> w >> h;
        r.Init( w,h);
        cout << r.Area() << endl <<
        r.Perimeter();
        return 0;
    }

通过类，可以定义变量。类定义出来的变量，也称为类的实例，就是“**对象**”，对象的本质是在内存中分配了一个存放类这个结构的空间。 
C++中，类的名字就是用户自定义的类型的名字。可以像使用基本类型那样来使用。 CRectangle就是一种用户自定义的类型。
## 对象的内存分配

 - 和结构变量一样，对象所占用的内存空间的大小，等于所有成员变量的大小之和（考虑内存对齐可能更大）。对于上面的CRectangle类，sizeof(CRectangle)
   = 8

 - 每个对象各有自己的存储空间。一个对象的某个成员变量被改变了，不会影响到另一个对象

 - 和结构变量一样，对象之间可以用 “=”进行赋值，但是不能用 “==”“!=”“>”“<”“>=”“<=”进行比较，除非这些运算符经过了“重载”。

Q.类分配内存产生对象后，成员变量占用空间，成员函数占不占用空间?
普通成员函数不在对象生成时分配函数空间，因为函数是静态绑定的，即函数体指令只占用代码段的一处空间，对象调用该函数之间跳到该空间入口地址，在对象分配时不会在堆或栈再开辟空间存放函数体。
但是当类中定义了虚函数，分配对象时要分配4字节（多个虚函数也是4个字节）的指针指向虚函数表。函数跳转地址依赖于运行时才产生的对象里的虚函数表，称为动态绑定，对象调用虚函数时不知道准确的跳转地址，只跳转到虚函数表查找跳转地址，再根据查找结果跳转。
## 对象访问其成员
类似于C结构体实例访问其成员的方法，用实例.成员，实例指针->成员，除此之外C++特有的通过引用访问：实例引用.成员
用法1：对象名.成员名

    CRectangle r1,r2;
    r1.w = 5;
    r2.Init(5,4);
Init函数作用在 r2 上，即Init函数执行期间访问的w 和 h是属于r2 这个对象的, 执行r2.Init 不会影响到r1
用法2. 指针->成员名

    CRectangle r1,r2;
    CRectangle * p1 = & r1;
    CRectangle * p2 = & r2;
    p1->w = 5;
    p2->Init(5,4); //Init作用在p2指向的对象上

用法3：引用名.成员名

    CRectangle r2;
    CRectangle & rr = r2;
    rr.w = 5;
    rr.Init(5,4); //rr的值变了， r2的值也变

# 类成员的访问方式
## 类成员的访问控制
Q. C++将数据和函数封装成类的成员，那么类内成员、内间成员的访问权限如何控制？
用下列访问范围关键字来说明类成员可被访问的范围：

 - private: 私有成员，只能在成员函数内访问
 - public : 公有成员，可以在任何地方访问
 - protected: 保护成员，用于继承关系的类的成员访问控制

定义一个带访问控制的类：

    class className {
        private:
        私有属性和函数
        public:
        公有属性和函数
        protected:
        保护属性和函数
    };

如过某个成员前面没有上述关键字，则缺省地被认为是private私有成员:

    class Man {
        int nAge;       //私有成员
        char szName[20]; // 私有成员
    public:
        void SetName(char * szName){
        strcpy( Man::szName,szName);
        }
    };

在类的成员函数内部，能够访问：

 - 当前对象的全部属性、 函数；
 - 同类其它对象的全部属性、函数。

在类的成员函数以外的地方，只能够访问该类对象的公有成员
注意：
通过对象的成员函数，可以访问同类其他对象的任意成员（即使是private）。private、public、protected真正的作用是限制成员变量的直接访问，而通过成员函数来访问成员变量是不受影响的。

## 访问控制与隐藏
成员访问控制可以定义类的成员变量能否被任意访问、或通过成员函数访问、能否被继承的子类访问等。这种机制称为对成员变量的**隐藏**
隐藏的目的是强制对成员变量的访问一定要通过成员函数进行，那么以后成员变量的类型等属性修改后，只需要更改成员函数即可。否则所有直接访问成员变量的语句都需要修改
一个类成员变量隐藏的例子：

     //类定义
        class CEmployee {
        private:
            char szName[30]; //名字
        public :
            int salary; //工资
            void setName(char * name);
            void getName(char * name);
            void averageSalary(CEmployee e1,CEmployee e2);
        };
        
        //成员函数定义
        void CEmployee::setName( char * name) {
            strcpy( szName, name); //ok
        }
        void CEmployee::getName( char * name) {
            strcpy( name,szName); //ok
        }
        void CEmployee::averageSalary(CEmployee e1,CEmployee e2){
            cout << e1.szName; //ok，访问同类其他对象私有成员
            salary = (e1.salary + e2.salary )/2;
        }
        
        //使用类和对象
        int main()
        {
            CEmployee e;
            strcpy(e.szName,"Tom1234567889"); //编译错，不能访问私有成员
            e.setName( "Tom");  // ok
            e.salary = 5000;    //ok
            return 0;
        }

如果将上面的程序移植到内存空间紧张的设备上，希望将szName改为char szName[5]，若szName不是私有，就要找出所有类似strcpy(e.szName,"Tom1234567889");这样的语句进行修改，以防止数组越界。如果将szName变为私有，那么程序中就不可能出现（除非在类的内部）strcpy(e.szName,"Tom1234567889");这样的语句，所有对szName的访问都是通过成员函数来进行，比如：e.setName( “Tom12345678909887”);如果szName改短了，上面的语句也不需要找出来修改，只要改setName成员函数，在里面确保不越界就可以了
除了使用类和隐藏机制，C++兼容C的struct结构体，也称为类。和用"class"的唯一区别是未说明是公有还是私有的成员，struct类的所有成员都是公有的。

    struct CEmployee {
        char szName[30]; //公有!!
        public :
        int salary; //工资
        void setName(char * name);
        void getName(char * name);
        void averageSalary(CEmployee
        e1,CEmployee e2);
    };

## 类成员函数的重载和缺省参数
同普通函数一样，类封装后的成员函数可以重载，可以有缺省参数

    class Location {
        private :
        int x, y;
        public:
        void init( int x=0 , int y = 0 );
        void valueX( int val ) { x = val ;}
        int valueX() { return x; }
    };
    
    void Location::init( int X, int Y)
    {
        x = X;
        y = Y;
    }
    
    int main() {
        Location A,B;
        A.init(5);  //使用init缺省y=0
        A.valueX(5);    //重载，使用valueX(int)
        cout << A.valueX();     //重载，使用valueX()
        return 0;
    }

输出：5
注意：重载和缺省的函数在调用时可能冲突，存在二义性：

    class Location {
        private :
        int x, y;
        public:
        void init( int x =0, int y = 0 );
        void valueX( int val = 0) { x = val; }
        int valueX() { return x; }
    };
    
    Location A;
    A.valueX(); //错误，编译器无法判断调用哪个valueX

# 类对象的创建与释放
## 普通构造函数
定义一个类只是定义一种数据结构类型，类实例化后在内存中才存在改类的对象。类实例化成对象可以在函数的栈中，或者动态分配在堆中

    ClassA a;   //该语句在函数内（如main）时，在main的堆栈中分配内存
    ClassA *pa = new ClassA;    //在堆中分配，需要delete手动释放
那么问题来了，分配的内存里的内容是什么？
不知道是什么值，只知道这块内存是被其他进程释放过，当前程序可以读写，释放时不会把值清零。
在C语言创建一个结构体变量，可以顺便初始化为全0

    StructA a = {0}; //单层结构体
    StructB b = {{0}}； //嵌套的结构体

C++也支持创建类时自动初始化，采用与类同名的成员函数的方法。这就是**构造函数（constructor）**
构造函数：

 - 成员函数的一种，名字与类名相同，可以有参数，不能有返回值(void也不行)
 - 作用是对对象进行初始化，如给成员变量赋初值
 - 如果定义类时没写构造函数，则编译器生成一个默认的无参数的构造函数。默认构造函数无参数，不做任何操作
 - 如果定义了构造函数，则编译器不生成默认的无参数构造函数
 - 对象生成时构造函数自动被调用。对象一旦生成，就再也不能在其上执行构造函数
 - 一个类可以有多个构造函数，即构造函数也可以重载

注意：构造函数不负责对象的内存分配，其关键作用是对象成员的值初始化。真正做对象分配的语句通常是new，new做两件事：给类分配内存形成对象，调用对象的构造函数。考虑一下也可知道，连对象都没有的情况，怎么能调用对象的构造函数分配内存呢？注意构造函数不给自身对象分配内存，但是构造函数可以做分配内存操作，比如对指针成员指向的空间分配内存。
使用默认构造函数：

    class Complex {
        private :
        double real, imag;
        public:
        void Set( double r, double i);
    }; //编译器自动生成默认构造函数
    Complex c1; //默认构造函数被调用
    Complex * pc = new Complex; //默认构造函数被调用

使用自定义的带参构造函数：

    class Complex {
        private :
        double real, imag;
        public:
        Complex( double r, double i = 0);
    };
        Complex::Complex( double r, double i) {
        real = r; imag = i;
    }
    
    Complex c1; // error, 缺少构造函数的参数
    Complex * pc = new Complex; // error, 没有参数
    Complex c1(2); // OK
    Complex c1(2,4), c2(3,5);
    Complex * pc = new Complex(3,4);

使用重载的构造函数：

    class Complex {
        private :
        double real, imag;
        public:
        void Set( double r, double i );
        Complex(double r, double i );
        Complex (double r );
        Complex (Complex c1, Complex c2);
    };
    
    Complex::Complex(double r, double i)
    {
        real = r; imag = i;
    }
    Complex::Complex(double r)
    {
        real = r; imag = 0;
    }
    Complex::Complex (Complex c1, Complex c2);
    {
        real = c1.real+c2.real;
        imag = c1.imag+c2.imag;
    }
    
    Complex c1(3) , c2 (1,0), c3(c1,c2);
    // c1 = {3, 0}, c2 = {1, 0}, c3 = {4, 0};

构造函数应该是public的， private构造函数不能直接用来初始化对象

    class CSample{
    private:
        CSample() {}
    };
    
    int main(){
        CSample Obj; //err. 唯一构造函数是private
        return 0;
    }

对于多个对象的实例化，可以用对象数组,构造函数的调用次数=对象个数，重载哪一个构造函数取决于每个对象的初始化方式。

    class CSample {
        int x;
        public:
        CSample() {
            cout << "Constructor 1 Called" << endl;
        }
        CSample(int n) {
            x = n;
            cout << "Constructor 2 Called" << endl;
        }
    };
    
    int main(){
        CSample array1[2];  //两次默认构造函数
        cout << "step1"<<endl;
        CSample array2[2] = {4,5};  //两次带参构造函数
        cout << "step2"<<endl;
        CSample array3[2] = {3};    //第一个带参构造，第二个默认构造
        cout << "step3"<<endl;
        CSample * array4 = new CSample[2];  //两次默认构造
        delete []array4;
        return 0;
    }

输出：

    Constructor 1 Called
    Constructor 1 Called
    step1
    Constructor 2 Called
    Constructor 2 Called
    step2
    Constructor 2 Called
    Constructor 1 Called
    step3
    Constructor 1 Called
    Constructor 1 Called

## 拷贝构造函数
定义：拷贝构造函数(copy constructor)是构造函数的一种，特点是：

 - 只有一个参数:对同类对象的引用
 - 入参必须是对象的引用，形如 X::X( X& ) 或 X::X(const X &), 后者以常量对象作为参数
 - 如果用户没有定义拷贝构造函数，编译器生成默认的拷贝构造函数，且它完成复制对象的功能。

拷贝构造函数也称为复制构造函数
调用形式如下。默认（普通）构造函数和默认拷贝构造函数都是编译生成，且并存的

    class Complex {
    private :
        double real,imag;
    };
    Complex c1; //调用缺省无参构造函数
    Complex c2(c1);//调用缺省的复制构造函数,将 c2 初始化成和c1一样

如果定义的自己的拷贝构造函数，则默认的拷贝构造函数不会生成
也就是说，自定义的带参拷贝构造函数和编译器生成的默认拷贝构造函数，不存在重载关系；而一个类有多个自定义的带参拷贝构造函数是允许的，可以重载。这一特点对于普通构造函数一样。

    class Complex {
    public :
        double real,imag;
        Complex(){ }
        Complex( const Complex & c ) {
            real = c.real;
            imag = c.imag;
            cout << “Copy Constructor called”;
        }
    };
    Complex c1;
    Complex c2(c1); //调用自己定义的复制构造函数，输出 Copy Constructor called

注意：拷贝构造函数传入的是同类的引用，而不是同类的对象
不允许有形如 X::X( X)的构造函数。因为成员函数入参由实参复制到形参实际会调用拷贝构造函数，拷贝构造函数作为成员函数也是一样，因此会有循环定义，即拷贝构造函数的执行需要调用拷贝构造函数的无限循环，用引用作为入参可以解决此问题。这点类似于C结构体允许有结构体指针成员，指向该结构体类型的实例，而不允许结构体有自身结构体的自接实例，这样会照成分配内存空间上的无限循环。

    class CSample {
        CSample( CSample c ) {} //错，不允许这样的构造函数
    }
### 拷贝构造函数的调用
以下三种情况会调用类对象的拷贝构造函数
1)用一个对象去初始化同类的另一个对象：

    Complex c2(c1);
    Complex c2 = c1; //初始化语句，非赋值语句

2)类的对象作为函数入参：如果某函数有参数是类A的对象，那么该函数被调用时，类A的拷贝构造函数将被调用：

    class A
    {
    public:
        A() { };
        A( A & a) {
            cout << "Copy constructor called" <<endl;
        }
    };
    
    void Func(A a1){ };
    int main(){
        A a2;
        Func(a2);  //传参是类A的对象
        return 0;
    }

输出: Copy constructor called
3) 类的对象作为函数返回值：如果函数的返回值是类A的对象，函数返回时，A的拷贝构造函数被调用：

    ```
    class A
    {
    public:
        int v;
        A(int n) { v = n; };
        A( const A & a) {
            v = a.v;
            cout << "Copy constructor called" <<endl;
        }
    };
    
    A Func() {
        A a(4);
        return a;
    }
    int main() {
        cout << Func().v << endl;
        return 0;
    }
    ```

输出：

    Copy constructor called
    4

小结：对象作为入参和返回值会调用拷贝构造函数，对象初始化新对象也会调用。
### 禁用拷贝构造函数
Q. 调用拷贝构造函数会形成对象的复制品，开销较大，如何禁用拷贝构造函数？
使用对象的引用，不自接把对象作为函数的入参出参。
Q.对象的引用会导致新问题：函数内修改了引用怎么办，原对象也会改
使用const引用，对象实参就不存在被函数修改的可能
使用对象的常引用，应用于对象作为函数入参出参，又不希望调用拷贝构造函数的情况

    void fun(const CMyclass & obj) {
    //函数中任何试图改变 obj值的语句都将是变成非法
    }

### 对象的赋值和复制
注意区分对象的赋值和复制：

 - 对象赋值是类的所有数据成员的一一对应赋值，其本质是对已分配内存的对象，进行数据成员的初始化

 - 对象复制 = 分配新对象对象空间 + 对新对象成员的赋值初始化。对象复制是要包含空间分配操作的

两个已分配内存的对象间的赋值并不会导致拷贝构造函数被调用

    //声明及初始化，调用拷贝构造函数
    Complex c2 = c1; 
    //先声明对象，再赋值,不调用拷贝构造函数，调用默认构造函数然后赋值
    Complex c2；
    c2 = c1;    

### 浅拷贝和深拷贝
当类对象有指针成员时，拷贝构造函数遇到一个问题，是只拷贝指针，还是连同指针指向的空间一起拷贝？

 - 浅拷贝：只拷贝指针成员
 - 深拷贝：拷贝指针成员，并拷贝其指向的内存空间数据
 由于深拷贝的实现用到“=”运算符重载，在运算符重载一节详述

## 转换构造函数
构造函数是能创建对象并初始化值的函数，将普通变量转换从类对象并分配内存空间的构造函数是转换构造函数。

 - 定义转换构造函数的目的是实现类型的自动转换（变量->对象）
 - 只有一个参数，且不是拷贝构造函数的构造函数，就是转换构造函数
 - 变量被赋值给对象时，编译器会自动调用转换构造函数，建立一个无名的临时对象

隐式的转换构造函数：

        class Complex {
        public:
            double real, imag;
            Complex( int i) {//类型转换构造函数
                cout << "IntConstructor called" << endl;
                real = i; imag = 0;
            }
            Complex(double r,double i) {real = r; imag = i; }
        };
        
        int main ()
        {
            Complex c1(7,8);
            Complex c2 = 12;
            c1 = 9;     // 9被自动转换成一个临时Complex对象
            cout << c1.real << "," << c1.imag << endl;
            return 0;
        }

显式的转换构造函数：

    class Complex {
    public:
        double real, imag;
        explicit Complex( int i) {  //显式类型转换构造函数
            cout << "IntConstructor called" << endl;
            real = i; imag = 0;
        }
        Complex(double r,double i) {real = r; imag = i; }
    };
    int main () {
        Complex c1(7,8);
        Complex c2 = Complex(12);
        c1 = 9;         // error, 9不能被自动转换成一个临时Complex对象
        c1 = Complex(9) //ok
        cout << c1.real << "," << c1.imag << endl;
        return 0;
    }

## 析构函数
### 析构函数的概念
**析构函数(destructors)**用于对象生命周期结束前（如函数中的对象在函数返回时消失），释放对象的内存占用，以及其他的准备工作。
构造函数和析构函数在对象生命周期的角色从逻辑上讲是开始和结束的关系，但具体操作不一样：构造函数不为对象分配内存，只给成员赋初值；而析构函数一般要释放对象的内存
析构函数的特点：

 - 名字与类名相同，在前面加‘~’，没有参数和返回值，一个类最多只能有一个析构函数
 - 析构函数对象消亡时即自动被调用。可以定义析构函数来在对象消亡前做善后工作，比如释放分配的空间等。
 - 如果定义类时没写析构函数，则编译器生成缺省析构函数。缺省析构函数什么也不做
 - 如果定义了析构函数，则编译器不生成缺省析构函数

析构函数例子：

    class String{
    private :
        char * p;
        public:
        String () {
            p = new char[10];
        }
        ~ String () ;
    };
    
    String ::~ String()
    {
        delete [] p;
    }
对象数组的生命期结束时，每个对象的析构函数都会被调用。

    class Ctest {
    public:
        ~Ctest() { cout<< "destructor called" << endl; }
    };
    
    int main () {
        Ctest array[2];
        cout << "End Main" << endl;
        return 0;
    }
输出：

    End Main
    destructor called
    destructor called
### 析构函数的调用
析构函数被调用有以下几种情况
1)delete运算导致析构函数调用：

    Ctest * pTest;
    pTest = new Ctest;  //构造函数调用
    delete pTest;       //析构函数调用
    ---------------------------------------------------------
    pTest = new Ctest[3];   //构造函数调用3次
    delete [] pTest;        //析构函数调用3次
若new一个对象数组，那么用delete释放时应该写 []。否则只delete一个对
象(调用一次析构函数)
2)析构函数在对象作为函数返回值返回后被调用。其原理是，对象作为函数的入参，出参时，都是临时生成的对象，传完就调用析构函数销毁。

    class CMyclass {
    public:
        ~CMyclass() { cout << "destructor" << endl; }
    };
    CMyclass obj;
    CMyclass fun(CMyclass sobj ) { //参数对象消亡也会导致析
                                    //构函数被调用
        return sobj;                //函数调用返回时生成临时对象返回
    }
    int main(){
        obj = fun(obj); //函数调用的返回值（临时对象）被
        return 0;       //用过后，该临时对象析构函数被调用
    }

输出：

    destructor
    destructor
    destructor

## 构造与析构的时序
总体原则：类似堆栈的先入后出原则：先构造的后析构
几个关键分类：
临时对象：赋值时创建，赋完值就消亡，生命周期似乎就一条指令
局部对象：在{}范围内存在，{}结束时消亡
全局、静态对象：从创建开始，在程序整个运行期间存在，程序结束时消亡。
一个例子：

    class Demo {
            int id;
        public:
            Demo(int i) {
                id = i;
                cout << "id=" << id << " constructed" << endl;
            }
            ~Demo() {
                cout << "id=" << id << " destructed" << endl;
            }
    };
    
    Demo d1(1);
    void Func()
    {
        static Demo d2(2);
        Demo d3(3);
        cout << "func" << endl;
    }
    
    int main () {
        Demo d4(4);
        d4 = 6;
        cout << "main" << endl;
        { 
            Demo d5(5);
        }
        Func();
        cout << "main ends" << endl;
        return 0;
    }

输出结果:

    id=1 constructed    //全局对象d1
    id=4 constructed    //构造函数d4
    id=6 constructed    //转换构造函数d4
    id=6 destructed     //临时对象赋值完毕，消亡
    main
    id=5 constructed    //构造函数d5
    id=5 destructed     //d5作用域结束，消亡
    id=2 constructed    //Fun构造静态对象d2(等同全局对象)
    id=3 constructed    //构造局部对象d3
    func
    id=3 destructed     //Fun返回，d3消亡
    main ends       
    id=6 destructed     //Main的局部对象d4消亡（id=6）
    id=2 destructed     //整个程序结束，全局对象d2消亡
    id=1 destructed     //整个程序结束，全局对象d1消亡

# 类对象的指针：this指针
this指针是在类成员函数内，指向当前类对象的指针。
注意：

 - this指针是指向当前对象的，所谓当前，是指调用成员函数时，是通过所在的对象的指针来调用
 - this指针体现的是成员函数和对象的关系，如果是静态成员函数，没有this指针，因为静态成员函数不从属于对象

为什么this指针如此特殊，需要单独命名？这涉及到C++的类的实现原理。
## C++的类与C的结构体
在C++早期，C++代码被编译器翻译成C代码，再由C编译器编译
类的实现原理和C的结构体有密切关系，下面是类和结构体的转换：
1)C++的类：

    class CCar {
        public:
            int price;
            void SetPrice(int p);
    };
    
    void CCar::SetPrice(int p)
    { price = p; }
    
    int main()
    {
        CCar car;
        car.SetPrice(20000);
        return 0;
    }

2)C的结构体实现类的功能

    struct CCar {
        int price;
    };
    
    void SetPrice(struct CCar * this, int p)
    { this->price = p; }
    
    int main() {
        struct CCar car;
        SetPrice( & car,
        20000);
        return 0;
    }

用C实现面向对象(CCar结构体)，方法(SetPrice)传入的参数是结构体对象的指针(struct CCar * this)
## C++的this指针
成员函数（非static）可以直接使用this来代表指向该函数作用的对象的指针

    class Complex {
    public:
        double real, imag;
        void Print() { cout << real << "," << imag ; }
        Complex(double r,double i):real(r),imag(i){ }   //初始化列表
        Complex AddOne() {
            this->real ++;  //等价于 real++
            this->Print();  //等价于 Print()
            return * this;
        }
    };
    
    int main() {
        Complex c1(1,1),c2(0,0);
        c2 = c1.AddOne();
        return 0;
    } //输出 2,1

对象的this指针通常隐式存在：

 - 成员函数（非static）的入参实际隐式地有一个this指针参数
 - 成员函数访问成员变量，也是隐式的通过this指针访问
 - 通过对象的指针调用成员函数，本质也是传入this指针

如果成员函数不访问成员变量，可以传入NULL的对象指针：

    class A
    {
        int i;
        public:
        void Hello() { cout << "hello" << endl; }
    };  //等价于 void Hello(A * this ) { cout << "hello" << endl; }
    int main()
    {
        A * p = NULL;
        p->Hello(); //等价于Hello(p)
    } // 输出： hello
如果成员函数访问了成员变量，实际是通过成员函数传入的this指针来访问，此时指针不可为NULL

    class A
    {
        int i;
        public:
        void Hello() { cout << i << "hello" << endl; }
    };  
    //等价于void Hello(A * this ) { cout << this->i << "hello" << endl; }
    //this若为NULL，则出错！！
    int main()
    {
        A * p = NULL;
        p->Hello(); //等价于Hello(p);
    } //出错

## 静态成员的概念
静态成员：在定义前面加了static关键字的成员、

    class CRectangle
    {
        private:
        int w, h;
        static int nTotalArea; //静态成员变量
        static int nTotalNumber;
        public:
        CRectangle(int w_,int h_);
        ~CRectangle();
        static void PrintTotal(); //静态成员函数
    };


 - 普通成员变量每个对象有各自的一份；而静态成员变量是全局共有的一份，为所有对象共享
 - 同一个类的成员函数，不论静不静态都是一份代码段
 - 普通成员函数必须具体作用于某个对象（也可以理解为绑定），而静态成员函数并不具体作用于某个对象
 - 因此静态成员（变量或者函数），不需要通过对象就能访问

sizeof求类大小，不会计算静态成员变量，因为不属于类的一部分（从空间占用上讲）。

    class CMyclass {
    int n;
    static int s;
    };  // sizeof(CMyclass) 等于 4

## 访问静态成员
一下几种方法访问,可以归纳为两种：通过类名访问，通过对象访问
1)类名::成员名

    CRectangle::PrintTotal();

2)对象名.成员名

    CRectangle r; 
    r.PrintTotal();

3)指针->成员名

    CRectangle * p = &r; 
    p->PrintTotal();
4)引用.成员名

    CRectangle & ref = r; 
    int n = ref.nTotalNumber;

## 静态成员函数与this指针

 - 静态成员函数中不能使用 this 指针！
 - 因为静态成员函数并不具体作用与某个对象!
 - 因此静态成员函数的真实的参数的个数，就是程序中写出的参数个数！

前面讲，C++的作用是封装数据，静态成员似乎破坏这一目的，那么静态成员有什么作用？
为了兼容C的全局变量与函数
 - 静态成员变量本质上是全局变量，哪怕一个对象都不存在，类的静态成员变量也存在
 - 静态成员函数本质上是全局函数
 - 设置静态成员这种机制的目的是将和某些类紧密相关的全局变量和函数写到类里面，看上去像一个整体，易于维护和理解

## 静态成员函数的使用场景
对于需要全局维护的数据，可以使用静态成员变量，并通过静态成员函数访问。
考虑一个图形处理程序，需要随时知道矩形的总数和总面积
 - 每个矩形封装成类的对象
 - 总数和总面积是类的静态成员（等价于全局变量）

类定义:

```
class CRectangle
    {
    private:
        int w, h;
        static int nTotalArea;
        static int nTotalNumber;
    public:
        CRectangle(int w_,int h_);
        ~CRectangle();
        static void PrintTotal();
    };
```

成员函数定义：

    CRectangle::CRectangle(int w_,int h_)
    {
        w = w_;
        h = h_;
        nTotalNumber ++;
        nTotalArea += w * h;
    }
    CRectangle::~CRectangle()
    {
        nTotalNumber --;
        nTotalArea -= w * h;
    }
    void CRectangle::PrintTotal()
    {
        cout << nTotalNumber << "," << nTotalArea << endl;
    }
类对象的调用：

    int CRectangle::nTotalNumber = 0;
    int CRectangle::nTotalArea = 0;
    // 必须在定义类的文件中对静态成员变量进行一次说明或初始化。否则编译能通过，链接不能通过。
    int main()
    {
        CRectangle r1(3,3), r2(2,2);
        //cout << CRectangle::nTotalNumber; 
        //错误 , 静态的私有变量也只能通过成员函数访问，静态不等于全局可访问
        CRectangle::PrintTotal();
        r1.PrintTotal();
        return 0;
    }
输出结果：

    2,13
    2,13
注意两点：

 - 静态成员变量是全局共有的一份存储，但private的静态成员只能通过类的成员函数访问。注意区分全局存储和全局访问，静态成员只有全局存储特性，没有全局可访问特性。
 - 静态成员函数，不能访问非静态成员变量，也不能调用非静态成员函数。

以下静态成员函数访问错误：

    void CRectangle::PrintTotal()
    {
        cout << w << "," << nTotalNumber << "," << nTotalArea << endl; //错误
    }
    CRetangle::PrintTotal(); //解释不通 w 到底是属于那个对象的

 以上例子还有缺陷：
 在使用静态成员时，特别是类的构造和析构会修改该静态成员，如前文的CRectangle类的构造函数有nTotalNumber++操作，析构有nTotalNumber--。这个时候要考虑构造和析构函数是否覆盖到所有类型（普通构造，拷贝构造，转换构造）
 在使用CRectangle类时，有时会调用复制构造函数生成临时的隐藏的CRectangle对象：

 - 调用一个以CRectangle类对象作为参数的函数时
 - 调用一个以CRectangle类对象作为返回值的函数时

临时对象在消亡时会调用析构函数，减少nTotalNumber和nTotalArea的值，可是这些临时对象在生成时却没有增加nTotalNumber和nTotalArea的值，因为设计类时漏掉了拷贝构造的情况
解决办法：为CRectangle类写一个拷贝构造函数：

    CRectangle :: CRectangle(CRectangle & r )
    {
        w = r.w; h = r.h;
        nTotalNumber ++;
        nTotalArea += w * h;
    }
这样nTotalNumber和nTotalArea全局计数就是准确的

# 类的嵌套：封闭类
## 封闭类的基本概念
再来把C++的类和C结构体对比下：

 - C：结构体的成员可以是基础变量，基础变量的指针，结构体的指针，其他复合类型的指针
 - C++：类的成员变量可以是基础变量，及其指针、引用，可不可以是类对象？类对象的引用和指针？

于是引入类嵌套类对象的情况：有成员对象的类叫封闭类（enclosing class)
一个示例：写一个汽车类，包含轮胎和引擎类对象
轮胎和引擎类：

    class CTyre //轮胎类
        {
        private:
            int radius; //半径
            int width; //宽度
        public:
            CTyre(int r,int w):radius(r),width(w) { }   //用初始化列表构造
        };
        
    class CEngine //引擎类
    {
    };

汽车类：

    class CCar { //汽车类
    private:
        int price; //价格
        CTyre tyre;
        CEngine engine;
    public:
        CCar(int p,int tr,int tw );
    };
    CCar::CCar(int p,int tr,int tw):price(p),tyre(tr, tw) //用初始化列表构造
    {
    };

汽车类的使用：

    int main()
    {
        CCar car(20000,17,225); //传入初始化列表
        return 0;
    }

## 初始化列表构造封闭类
对于封闭类，有几个问题就凸显出来：

 - 构造一个封闭类，还要构造其嵌套的类
 - 构造时序是怎样的
 - 析构时序是怎样的

上例中，如果 CCar类不定义构造函数，下面的语句会编译出错：CCar car;
因为CCar不传初始化值给嵌套类CTyre，编译器不知道该如何初始化car.tyre的成员变量
而car.engine的初始化没问题，因为不用初始化成员变量，用默认构造函数即可
为了解决封闭类的嵌套类成员的初始化问题，构造函数引入新的初始化方法：

 - 初始化列表：将成员初始化从构造函数体，移到函数名后面，只是换了形式，但是方便了封闭类各嵌套类的初始化，不用开发者自己到函数体写构造函数内容
 - 成员对象初始化列表中的参数可以是任意复杂的表达式，可以包括函数，变量，只要表达式中的函数或变量有定义就行。

封闭类都是通过构造函数的初始化列表，层层传入嵌套类的构造函数：

    CCar::CCar(int p,int tr,int tw):price(p),tyre(tr, tw){};
    //p, tr, tw是传入的初始化值; price,tyre是CCar对象的两个成员
    CCar car(20000,17,225);
    //Car的price = 20000, Car的tyre的radius = 17，width = 225

 上例是普通构造函数，对于封闭类的拷贝构造函数：

 - 封闭类对象是用拷贝构造函数初始化的，其成员对象也用拷贝构造函数初始化

测试用例：

    class A
    {
    public:
        A() { cout << "default" << endl; }
        A(A & a) { cout << "copy" << endl;}
    };
    class B { A a; };
    
    int main()
    {
        B b1,b2(b1);
        return 0;
    }

输出：

    default
    Copy

 


下面考虑封闭类构造和析构的时序

 - 封闭类对象生成时，先执行所有对象成员的构造函数，然后才执行封闭类的构造函数、
 - 对象成员的构造函数调用次序和对象成员在类中的说明次序一致，与它们在成员初始化列表中出现的次序无关
 - 当封闭类的对象消亡时，先执行封闭类的析构函数，然后再执行成员对象的析构函数。次序和构造函数的调用次序相反

一个测试示例


    class CTyre {
        public:
            CTyre() { cout << "CTyre contructor" << endl; }
            ~CTyre() { cout << "CTyre destructor" << endl; }
    };
    class CEngine {
        public:
            CEngine() { cout << "CEngine contructor" << endl; }
            ~CEngine() { cout << "CEngine destructor" << endl; }
    };
    class CCar {
        private:
            CEngine engine;
            CTyre tyre;
        public:
            CCar( ) { cout << “CCar contructor” << endl; }
            ~CCar() { cout << "CCar destructor" << endl; }
    };
    
    int main(){
    CCar car;
    return 0;
    }

输出结果：

    CEngine contructor
    CTyre contructor
    CCar contructor
    CCar destructor
    CTyre destructor
    CEngine destructor

# 类的成员属性：友元和常量成员
## 友元函数和友元类
友元(friend)分为友元函数和友元类两种
一个类的private成员，只能通过类自己的成员函数访问，那么其他类的成员函数想访问这个类的private成员怎么办？友元可以解决这种需求
1) 友元函数: 一个类的友元函数可以访问该类的私有成员
   即类A内可以声明其他类B的成员函数或者全局函数，加前缀friend，这些以friends开头的函数就可访问类A的成员。

   ```
   class CCar ; //提前声明 CCar类，以便后面的CDriver类使用
    class CDriver
    {
        public:
            void ModifyCar( CCar * pCar) ; //改装汽车
    };
    class CCar
    {
        private:
            int price;
            friend int MostExpensiveCar( CCar cars[], int total); //声明友元
            friend void CDriver::ModifyCar(CCar * pCar); //声明友元
    };
   
    void CDriver::ModifyCar( CCar * pCar)
    {
        pCar->price += 1000; //访问CCar成员，汽车改装后加价
    }
   
    int MostExpensiveCar( CCar cars[],int total)//求最贵汽车的价格
    {
        int tmpMax = -1;
        for( int i = 0;i < total; ++i )
        if( cars[i].price > tmpMax) //访问CCar成员
        tmpMax = cars[i].price;
        return tmpMax;
    }
   
    int main()
    {
        return 0;
    }
   ```
   
   
   

除了普通成员函数，也可以将类构造、析构函数说明为另一个类的友元

2)友元类: 如果A是B的友元类，那么A的成员函数可以访问B的私有成员
如果是类的嵌套（封闭类），声明为friend的类A可以调用自己的成员函数访问与它为friend关系的类B的私有成员，而不必调用类B的成员函数。

    class CCar
    {
    private:
        int price;
        friend class CDriver; //声明CDriver为友元类
    };
    class CDriver
    {
    public:
        CCar myCar;
        void ModifyCar() {  //改装汽车
        myCar.price += 1000;   //因CDriver是CCar的友元类，故此处可以访问其私有成员
        }
    };
    
    int main(){ return 0; }

友元类之间的关系不能传递，不能继承。就是说A和B是friend,B和C是friend,但A和C不一定是friend。父类之间的friend关系，子类不一定能传承。
## 常量成员函数
如果不希望某个对象的值被改变，定义该对象的时候可以在前面加 const关键字
在类的成员函数说明后面加const关键字，则该成员函数成为常量
成员函数。
常量成员函数内部不能改变属性的值，也不能调用非常量成员函数
在定义常量成员函数和声明常量成员函数时都应该使用const 关键字。

    class Sample {
    private :
        int value;
        public:
        void PrintValue() const;
    };
    void Sample::PrintValue() const {             //此处不使用const会导致编译出错
        cout << value;
    }
    void Print(const Sample & o) {
        o.PrintValue(); 
    }//若 PrintValue非const则编译错

以下是错误示例：

    class Sample {
    private :
        int value;
        public:
        void func() { };
        Sample() { }
        void SetValue() const {
            value = 0; // wrong
            func(); //wrong
        }
    };
    const Sample Obj;
    Obj.SetValue (); //常量对象上可以使用常量成员函数
什么场景定义成常量成员函数？
如果一个成员函数中没有调用非常量成员函数，也没有修改成员变量的值，最好将其写成常量成员函数

常量成员函数的重载：
两个成员函数，名字和参数表都一样，但是一个是const,一个不是，算重载关系，而非重定义。
# 类的运算：运算符重载
C++定义了类，可以像基本类型那样创建、销毁、初始化。那么类和类之间的运算呢？
+、 -、 *、 /、 %、 ^、 &、 ~、 !、 |、 =、 << 、>>、 !=、
考虑以下方法实现类的运算：

 - 设计类的成员函数，支持类运算操作
 - 设计某种机制，把运算符关联成函数操作，在函数内定义具体类运算方法。进行类的运算时，形式上可以像基本类型的运算一样

例如complex_a和complex_b是两个复数对象；求两个复数的和, 希望能直接写：complex_a + complex_b
运算符重载将解决类和对象的运算需求
## 运算符重载的概念
运算符重载，就是对已有的运算符(C++中预定义的运算符)赋予多重的含义，使同一运算符作用于不同类型的数据时导致不同类型的行为
运算符重载的目的是：扩展C++中提供的运算符的适用范围，使之能作用于对象
期望效果:同一个运算符，对不同类型的操作数，所发生的行为不同

    complex_a + complex_b //生成新的复数对象
    5 + 4 = 9 //基本运算符操作
从行为上看，运算符重载类似于把运算符进行了重定义成函数操作（类似C的typedef）
运算符重载写法：

    返回值类型 operator 运算符（形参表）
    {
    ……  //定义该运算符的运算规则
    }

示例：

    class Complex
    {
    public:
        double real,imag;
        Complex( double r = 0.0, double i= 0.0):real(r),imag(i) { }
        Complex operator-(const Complex & c);
    };
    Complex operator+( const Complex & a, const Complex & b)
    {
        return Complex( a.real+b.real,a.imag+b.imag); //返回一个临时对象
    }
    Complex Complex::operator-(const Complex & c)
    {
        return Complex(real - c.real, imag - c.imag); //返回一个临时对象
    }
    
    int main()
    {
        Complex a(4,4),b(1,1),c;
        c = a + b; //等价于c=operator+(a,b);
        cout << c.real << "," << c.imag << endl;
        cout << (a-b).real << "," << (a-b).imag << endl;
        //a-b等价于a.operator-(b)
        return 0;
    }

输出：

    5,5
    3,3

c = a + b; 等价于c=operator+(a,b);
a-b 等价于a.operator-(b)
运算符重载的实现还是成员函数，所以是依赖于对象的。也就是说，运算符重载看上去和类、对象没啥关系，但本质上，重载的运算符是归属于某个类的，因为a-b只是表象现象，真正定义对象运算的，是a.operator-(b)成员函数。
因为运算符重载依赖对象的，因此双目运算，如+，-，在运算符重载时只需要传入另一个对象，而不需要传运算符的当前对象。
重载为成员函数时， 参数个数为运算符目数减一。
重载为普通函数时， 参数个数为运算符目数

运算符重载概念小结：

 - 运算符重载的实质是函数重载
 - 可以重载为普通函数，也可以重载为成员函数
 - 把含运算符的表达式转换成对运算符函数的调用
 - 把运算符的操作数转换成运算符函数的参数
 - 运算符被多次重载时，根据实参的类型决定调用哪个运算符函数

## 赋值运算符的重载
接下来的几节讲几个代表性的运算符重载。本节讲赋值运算符“=”有时候希望赋值运算符两边的类型可以不匹配，比如，把一个int类型变量赋值给一个Complex对象，或把一个 char *类型的字符串赋值给一个字符串对象,此时就需要重载赋值运算符“=”。
赋值运算符“ =”只能重载为成员函数

示例：

    class String {
    private:
        char * str;
        public:
        String ():str(new char[1]) { str[0] = 0;}
        const char * c_str() { return str; };
        String & operator = (const char * s);
        String::~String( ) { delete [] str; }
    };
    String & String::operator = (const char * s)
    { //重载“=”以使得 obj = “hello”能够成立
        delete [] str;
        str = new char[strlen(s)+1];
        strcpy( str, s);
        return * this;
    }
    
    int main()
    {
        String s;
        s = "Good Luck," ; //等价于 s.operator=("Good Luck,");
        cout << s.c_str() << endl;
        // String s2 = "hello!"; //这条语句要是不注释掉就会出错
        s = "Shenzhou 8!"; //等价于 s.operator=("Shenzhou 8!");
        cout << s.c_str() << endl;
        return 0;
    }

输出：

    Good Luck,
    Shenzhou 8!

## 赋值运算符与深拷贝
在类与对象的拷贝构造函数一节讲了拷贝构造函数的作用：用一个已经初始化的对象，去初始化另一个对象，具体操作是讲成员变量一一赋值。
那么更深入考虑一下:对于各种类型的成员变量，能不能达到目的？

 - 对于基础类型的成员变量，如int,char，直接赋值即可
 - 对于指针类型的成员变量，给指针赋值就Ok?需不需要给指针指向的空间也赋值？
 - 对于引用类型的成员变量，直接赋值OK?
 - 对于类对象类型的成员变量，怎么赋值？嵌套调用拷贝构造函数？

引用只是标签，可以直接拷贝，等同变量拷贝。封闭类的构造函数会嵌套调用基础类型的拷贝，直到所有成员赋值完为止。
唯一需要考虑的是包含指针类型成员的类如何拷贝
如果直接赋值指针而不分配并初始化其指向空间，效果如下:
![image-20221208164831061](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212081648147.png)
如不定义自己的赋值运算符，那么S1=S2实际上导致 S1.str和 S2.str
指向同一地方。
如果S1对象消亡，析构函数将释放 S1.str指向的空间，则S2消亡时还
要释放一次，就形成两次delete错误!
如果执行 S1 = "other"；会导致S2.str指向的地方被delete

为了解决以上问题，类的拷贝构造不仅要拷贝指针，还有拷贝指针指向的空间（分配新内存+拷贝）。这种带内存分配的拷贝称为深拷贝

 - 浅拷贝：只拷贝成员，对于指针成员，也只拷贝指针变量
 - 深拷贝：拷贝成员，对于指针成员，拷贝指针变量，且拷贝指针指向的内存空间

为了实现深拷贝，需要重载“=”运算符：

    String & operator = (const String & s) {
        delete [] str;  //先释放指针原本指向的空间,因为新空间和原空间大小可能不一样
        str = new char[strlen( s.str)+1];   //分配指针指向的新空间
        strcpy( str,s.str); //新空间赋值初始化
        return * this;  //返回当前对象的指针
    }
还有可优化的，如果传入对象就是当前对象，没必要释放又分配，直接返回即可

    String & operator = (const String & s){
        if( this == & s)
            return * this;
        delete [] str;
        str = new char[strlen(s.str)+1];
        strcpy( str,s.str);
        return * this;
    }
整个类设计如下：

    class String {
    private:
        char * str;
    public:
        String ():str(new char[1]) { str[0] = 0;}
        const char * c_str() { return str; };
        String & operator = (const char * s){
            delete [] str;
            str = new char[strlen(s)+1];
            strcpy( str, s);
            return * this;
    };
        ~String( ) { delete [] str; }
    };

再考虑一下运算符重载函数的返回值
为什么返回String &
原因：对运算符进行重载的时候，好的风格是尽量保留运算符原本的特性
例如运算符是可以多个连续运算的

    a = b = c;
    (a=b)=c; //会修改a的值
分别等价于：

    a.operator=(b.operator=(c));
    (a.operator=(b)).operator=(c);

对于拷贝构造函数，原指针未初始化，不指向任何空间，直接分配空间在拷贝即可，写法如下：

    String( String & s)
    {
        str = new char[strlen(s.str)+1];
        strcpy(str,s.str);
    }
## 流运算符的重载
C++常用的输入输出是怎么实现的？

    cout << 5 << “this”;

 - cout是什么?
 - "<<"原本是位偏移运算，为什么能作用于cout?
 - "<<"怎么支持连续运算，且支持多种类型

原因就是<<被流运算类重载了。

 - cout是在iostream中定义的，ostream类的对象
 - “<<” 能用在cout上是因为，在iostream里对“ <<” 进行了重载
 - 运算符重载函数返回对象的引用，实现连续运算；多个运算符重载函数的重载，支持多种类型

实现方法：

    ostream & ostream::operator<<(int n)
    {
        …… //输出n的代码
        return * this;
    }
    ostream & ostream::operator<<(const char * s )
    {
        …… //输出s的代码
        return * this;
    }

cout << 5 << “this”; 
等价于： cout.operator<<(5).operator<<(“this”);
一个流运算符重载的示例：
假定c是Complex复数类的对象，现在希望写“ cout << c;”，就能以“ a+bi”的形式输出c的值，写“ cin>>c;”，就能从键盘接受“ a+bi”形式的输入，并且使得c.real = a,c.imag = b

    #include <iostream>
    #include <string>
    #include <cstdlib>
    using namespace std;
    class Complex {
        double real,imag;
        public:
        Complex( double r=0, double i=0):real(r),imag(i){ };
        friend ostream & operator<<( ostream & os, const Complex & c);
        friend istream & operator>>( istream & is,Complex & c);
    };
    ostream & operator<<( ostream & os,const Complex & c)
    {
        os << c.real << "+" << c.imag << "i"; //以"a+bi"的形式输出
        return os;
    }
       
    istream & operator>>( istream & is,Complex & c)
    {
        string s;
        is >> s; //将"a+bi"作为字符串读入, “a+bi”中间不能有空格
        int pos = s.find("+",0);
        string sTmp = s.substr(0,pos); //分离出代表实部的字符串
        c.real = atof(sTmp.c_str()); //atof库函数能将const char*指针指向的内容转换成 float
        sTmp = s.substr(pos+1, s.length()-pos-2); //分离出代表虚部的字符串
        c.imag = atof(sTmp.c_str());
        return is;
    }
    
    int main()
    {
        Complex c;
        int n;
        cin >> c >> n;
        cout << c << "," << n;
        return 0;
    }

运行结果可以如下：

    13.2+133i 87    //输入
    13.2+133i, 87   //输出

## 其他运算符重载
类型转换运算符"()"重载：

    #include <iostream>
    using namespace std;
    class Complex
    {
        double real,imag;
        public:
        Complex(double r=0,double i=0):real(r),imag(i) { };
        operator double () { return real; }
        //重载强制类型转换运算符 double
    };
    int main()
    {
        Complex c(1.2,3.4);
        cout << (double)c << endl; //输出 1.2
        double n = 2 + c; //等价于 double n=2+c.operator double()
        cout << n; //输出 3.2
    }

自增自减运算符"++,--"的重载：
自增运算符++、自减运算符--有前置/后置之分，为了区分所重载的是前置运算符还是后置运算符， C++规定：

 - 前置运算符作为一元运算符重载
 - 后置运算符作为二元运算符重载，多写一个没用的参数

前置运算符重载形式：

    重载为成员函数：
    T & operator++();   //不用写入参，当前对象的成员++
    T & operator--();
    重载为全局函数：
    T1 & operator++(T2);
    T1 & operator—(T2);
后置运算符重载形式：    

    重载为成员函数：
    T operator++(int);  //多写一个入参，用于和前置重载区分
    T operator--(int);
    重载为全局函数：
    T1 operator++(T2,int );
    T1 operator—( T2,int);
调用示例：

    int main()
    {
        CDemo d(5);
        cout << (d++ ) << ","; //等价于 d.operator++(0);
        cout << d << ",";
        cout << (++d) << ","; //等价于 d.operator++();
        cout << d << endl;
        cout << (d-- ) << ","; //等价于 operator--(d,0);
        cout << d << ",";
        cout << (--d) << ","; //等价于 operator--(d);
        cout << d << endl;
        return 0;
    }
    
    class CDemo {
    private :
        int n;
        public:
        CDemo(int i=0):n(i) { }
        CDemo & operator++(); //用于前置形式
        CDemo operator++( int ); //用于后置形式
        operator int ( ) { return n; }
        friend CDemo & operator--(CDemo & );
        friend CDemo operator--(CDemo & ,int);
    };
    CDemo & CDemo::operator++()
    { //前置 ++
        n ++;
        return * this;
    } // ++s即为: s.operator++();
    
    CDemo CDemo::operator++( int k )
    { //后置 ++
        CDemo tmp(*this); //记录修改前的对象
        n ++;
        return tmp; //返回修改前的对象
    } // s++即为: s.operator++(0);
    CDemo & operator--(CDemo & d)
    {//前置--
        d.n--;
        return d;
    } //--s即为: operator--(s);
    CDemo operator--(CDemo & d,int)
    {//后置--
        CDemo tmp(d);
        d.n --;
        return tmp;
    } //s--即为: operator--(s, 0);

## 运算符重载注意事项

 - C++不允许定义新的运算符
 - 重载后运算符的含义应该符合日常习惯，即保留原运算符的使用风格
 - 运算符重载不改变运算符的优先级
 - 以下运算符不能被重载：“ .” “ .*” “ ::” “ ?:” “sizeof”
 - 重载运算符()、[]、->、=，运算符重载函数必须声明为
类的成员函数

## 运算符重载的综合示例
实现一个可变长数组类型CArray，实现如下用例：

    int main() { 
        CArray a; //开始里的数组是空的
        for( int i = 0;i < 5;++i)
            a.push_back(i); //要用动态分配的内存来存放数组元素，需要一个指针成员变量
        CArray a2,a3;
        a2 = a; //要重载“=”
        for( int i = 0; i < a.length(); ++i )
            cout << a2[i] << " " ;  //要重载[]
        a2 = a3; //a2是空的
        for( int i = 0; i < a2.length(); ++i )//a2.length()返回0
            cout << a2[i] << " ";
        cout << endl;
        a[3] = 100;
        CArray a4(a);   //要自己写拷贝构造函数
        for( int i = 0; i < a4.length(); ++i )
            cout << a4[i] << " ";
        return 0;
    }
CArray类的设计：

    class CArray {
        int size; //数组元素的个数
        int *ptr; //指向动态分配的数组
        public:
        CArray(int s = 0); //s代表数组元素的个数
        CArray(CArray & a);
        ~CArray();
        void push_back(int v); //用于在数组尾部添加一个元素v
        CArray & operator=( const CArray & a);
        //用于数组对象间的赋值
        int length() { return size; } //返回数组元素个数
        int & CArray::operator[](int i) //返回值为 int 不行!不支持 a[i] = 4
        {//用以支持根据下标访问数组元素，如n = a[i] 和a[i] = 4; 这样的语句
            return ptr[i];
        }
    };
成员函数的实现：

    CArray::CArray(int s):size(s)
    {
        if( s == 0)
        ptr = NULL;
        else
        ptr = new int[s];
    }
    CArray::CArray(CArray & a) {
        if( !a.ptr) {
        ptr = NULL;
        size = 0;
        return;
        }
        ptr = new int[a.size];
        memcpy( ptr, a.ptr, sizeof(int ) * a.size);
        size = a.size;
    }
    
    CArray::~CArray()
    {
        if( ptr) delete [] ptr;
    }
    CArray & CArray::operator=( const CArray & a)
    { //赋值号的作用是使“=”左边对象里存放的数组，大小和内容都和右边的对象一样
        if( ptr == a.ptr) //防止a=a这样的赋值导致出错
        return * this;
        if( a.ptr == NULL) { //如果a里面的数组是空的
        if( ptr ) delete [] ptr;
        ptr = NULL;
        size = 0;
        return * this;
        }
        if( size < a.size) {         //如果原有空间够大，就不用分配新的空间
            if(ptr)
            delete [] ptr;
            ptr = new int[a.size];
        }
        memcpy( ptr,a.ptr,sizeof(int)*a.size);
        size = a.size;
        return * this;
    } // CArray & CArray::operator=( const CArray & a)
    
    void CArray::push_back(int v)
    { //在数组尾部添加一个元素
        if( ptr) {
            int * tmpPtr = new int[size+1]; //重新分配空间
            memcpy(tmpPtr,ptr,sizeof(int)*size); //拷贝原数组
            内容
            delete [] ptr;
            ptr = tmpPtr;
        }
        else //数组本来是空的
        ptr = new int[1];
        ptr[size++] = v; //加入新的数组元素
    }
