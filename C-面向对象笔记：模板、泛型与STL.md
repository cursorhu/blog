---
title: C++面向对象笔记：模板、泛型与STL
date: 2020-03-21 16:53:38
tags: C++
categories: C++

---

# 0.概述
C++除了支持面向对象，也支持函数的扩展和重用。继承类和多态充分支持了扩展，即基类进行概括抽象的设计，派生类实现具体的设计。那么重用呢，如何写一个函数，能在多种情况不加修改的套用？
考虑以下问题：

    交换两个整型变量的值的Swap函数：
    void Swap(int & x,int & y)
    {
        int tmp = x;
        x = y;
        y = tmp;
    }
    交换两个double型变量的值的Swap函数:
    void Swap(double & x,double & y)
    {
        double tmp = x;
        x = y;
        y = tmp;
    }

这两个函数除了入参类型不同，函数名，参数个数，返回类型都相同。函数体的处理流程也完全一样。能否只写一个Swap，就能交换各种类型的变量？
模板（template）将解决这种问题。

# 函数模板
## 函数模板的概念
用函数模板，设计仅数据类型不同的一组函数的通用模板：

    template <class 类型参数1，class 类型参数2,……>
    返回值类型 模板名 (形参表)
    {
        函数体
    };
    
    template <class T> //在函数前声明模板，参数类型（class）是T
    void Swap(T & x,T & y)
    {
        T tmp = x;
        x = y;
        y = tmp;
    }

在普通函数前，先用template< class T >声明参数类型T，就可以在函数体使用T，就像使用普通变量一样。在运行时，T传入的类型不同，函数处理的数据类型也不同。  
函数模板是如何实现的？它是一种函数吗？

    int main()
    {
        int n = 1,m = 2;
        Swap(n,m); //编译器自动生成 void Swap(int & ,int & )函数
        double f = 1.2,g = 2.3;
        Swap(f,g); //编译器自动生成 void Swap(double & ,double & )函数
        return 0;
    }

函数模板只是个模子，编译器通过模板和具体参数类型产生不同的函数。编译器会进行两次编译，第一次检测模板代码，第二次检测加参数后的具体函数代码。 
在调用以上函数模板时，实际会生成两个具体函数：

    void Swap(int & x,int & y)
    {
        int tmp = x;
        x = y;
        y = tmp;
    }
    void Swap(double & x,double & y)
    {
        double tmp = x;
        x = y;
        y = tmp;
    }
函数模板，可以理解为一种函数的宏，编译期在宏被调用的地方，全部替换成宏所对应的具体值。
函数模板和运行时函数，也类似于类和对象的关系，类只是类型，对象是给类分配了内存的实例。函数模板只是通用的类型，函数实例是函数模板的实例化对象。
## 函数模板的特性
函数模板中可以有不止一个类型参数

    template <class T1, class T2>
    T2 print(T1 arg1, T2 arg2)
    {
        cout<< arg1 << " "<< arg2<<endl;
        return arg2;
    }

不通过参数也能实例化函数模板

    template <class T>
    T Inc(T n)
    {
        return 1 + n;
    }
    int main()
    {
        cout << Inc<double>(4)/2; //显式实例化模板，输出 2.5
        return 0;
    }

## 函数模板与重载
函数模板和函数重载，在写法上很接近，都是一组参数不同的同名函数。它们有什么联系和区别？

 - 函数重载，关键在参数个数
 - 函数模板，关键在参数类型

函数模板可以重载，只要它们的形参表或类型参数表不同即可

    template<class T1, class T2>
    void print(T1 arg1, T2 arg2) {
        cout<< arg1 << " "<< arg2<<endl;
    }
    template<class T>
    void print(T arg1, T arg2) {
        cout<< arg1 << " "<< arg2<<endl;
    }
    template<class T,class T2>
    void print(T arg1, T arg2) {
        cout<< arg1 << " "<< arg2<<endl;
    }

在有多个函数和函数模板名字相同的情况下，编译器如下处理一条函数调用语句:
1. 先找参数完全匹配的普通函数(非由模板实例化而得的函数)

2. 再找参数完全匹配的模板函数。

3. 再找实参数经过自动类型转换后能够匹配的普通函数。

4) 上面的都找不到，则报错
   如果有函数重载，在第（1）步就找到，如果没有，才找（2）的函数模板

    
   
   ```
   template <class T>
    T Max( T a, T b) {
        cout << "TemplateMax" <<endl; return 0;
    }
    template <class T,class T2>
    T Max( T a, T2 b) {
        cout << "TemplateMax2" <<endl; return 0;
    }
    double Max(double a, double b){
        cout << "MyMax" << endl;
    return 0;
    }
   
    int main() {
        int i=4, j=5;
        Max( 1.2,3.4); // 输出MyMax
        Max(i, j); //输出TemplateMax
        Max( 1.2, 3); //输出TemplateMax2
        return 0;
    }
   ```
   
   
   

注意上文的步骤（2）（3）是分开的，匹配模板函数时，是不进行类型自动转换的，不匹配就是不匹配。输入参数有二义性时。编译器不会为开发者做决定

    template<class T>
    T myFunction( T arg1, T arg2)
    { cout<<arg1<<" "<<arg2<<"\n"; return arg1;}
    ……
    myFunction( 5, 7); //ok： replace T with int
    myFunction( 5.8, 8.4); //ok： replace T with double
    myFunction( 5, 8.4); //error， no matching function for call to 'myFunction(int, double)'

# 类模板
## 类模板的概念
类也能使用模板，来生成不同成员类型的类
类模板：在定义类的时候，加上一个/多个类型参数。在使用类模板时，指定类型参数应该如何替换成具体类型，编译器据此生成相应的模板类

    template <class 类型参数1， class 类型参数2， ……> //类型参数表
    class 类模板名
    {
        成员函数和成员变量
    };

类模板的成员函数的定义写法：

    template <class 类型参数1， class 类型参数2， ……> //类型参数表
    返回值类型 类模板名<类型参数名列表>::成员函数名（ 参数表）
    {
        ……
    }
用类模板实例化对象的写法：

    类模板名 <真实类型参数表> 对象名(构造函数实参表);

一个例子：map类型中的pair类的实现：

    template <class T1,class T2>    //pair是类模板
    class Pair
    {
    public:
        T1 key; //关键字
        T2 value; //值
        Pair(T1 k,T2 v):key(k),value(v) { }; //构造函数
        bool operator < ( const Pair<T1,T2> & p) const; //运算符重载函数
    };
    
    template<class T1,class T2>
    bool Pair<T1,T2>::operator < ( const Pair<T1,T2> & p) const
    //Pair的运算符重载函数的定义
    {
        return key < p.key;
    }
    
     int main()
    {
        Pair<string,int> student("Tom",19); //实例化出一个类 Pair<string,int>
        cout << student.key << " " << student.value;
        return 0;
    }

输出：Tom 19
编译器由类模板生成类的过程叫类模板的实例化。 由类模板实例化得到的类， 叫模板类
同一个类模板的两个模板类是不兼容的，即两个不同的类

    Pair<string,int> * p;
    Pair<string,double> a;
    p = & a; //错误，不是同类也不是继承类，不能赋值

函数模版可以作为类模板成员

    template <class T>
    class A
    {
    public:
        template<class T2>
        void Func( T2 t) { cout << t; } //成员函数模板
    };
    int main()
    {
        A<int> a;
        a.Func('K'); //成员函数模板 Func被实例化
        a.Func("hello"); //成员函数模板 Func再次被实例化
        return 0;
    } //输出： KHello

类模板的“<类型参数表>”中可以出现非类型参数：

    template <class T, int size>
    class CArray{
        T array[size];
        public:
        void Print( )
        {
            for( int i = 0;i < size; ++i)
            cout << array[i] << endl;
        }
    };
    
    CArray<double,40> a2;
    CArray<int,50> a3;

## 类模板的派生
类模板也支持类的派生：
• 类模板从类模板派生
• 类模板从模板类派生
• 类模板从普通类派生
• 普通类从模板类派生

(1)类模板从类模板派生

    template <class T1,class T2>
    class A {
        T1 v1; T2 v2;
    };
    
    template <class T1,class T2>
    class B:public A<T2,T1> {
        T1 v3; T2 v4;
    };
    
    template <class T>
    class C:public B<T,T> {
        T v5;
    };
    
    int main() {
        B<int,double> obj1;
        C<int> obj2;
        return 0;
    }

(2)类模板从模板类派生

    template <class T1,class T2>
    class A {
        T1 v1; T2 v2;
    };
    
    template <class T>
    class B:public A<int,double> {
        T v;
    };
    
    int main() {
        B<char> obj1; //自动生成两个模板类：A<int,double> 和 B<char>
        return 0;
    }

(3)类模板从普通类派生

    class A {
        int v1;
    };
    
    template <class T>
    class B:public A { //所有从B实例化得到的类， 都以A为基类
        T v;
    };
    
    int main() {
        B<char> obj1;
        return 0;
    }

(4)普通类从模板类派生

    template <class T>
    class A {
        T v1;
        int n;
    };
    
    class B:public A<int> {
        double v;
    };
    int main() {
        B obj1;
        return 0;
    }

## 类模板与友元
• 函数、类、类的成员函数作为类模板的友元
• 函数模板作为类模板的友元
• 函数模板作为类的友元
• 类模板作为类模板的友元

(1)函数、类、类的成员函数作为类模板的友元

    void Func1() { }
    class A { };
    class B
    {
        public:
        void Func() { }
    };
    
    template <class T>
    class Tmpl
    {
        friend void Func1();
        friend class A;
        friend void B::Func();
    }; //任何从Tmp1实例化来的类， 都有以上三个友元
(2)函数模板作为类模板的友元

    template <class T1,class T2>
    class Pair
    {
    private:
        T1 key; //关键字
        T2 value; //值
    public:
        Pair(T1 k,T2 v):key(k),value(v) { };
        bool operator < ( const Pair<T1,T2> & p) const;
        template <class T3,class T4>
        friend ostream & operator<< ( ostream & o,
        const Pair<T3,T4> & p);
    };
    
    template<class T1,class T2>
    bool Pair<T1,T2>::operator < ( const Pair<T1,T2> & p) const
    { //"小"的意思就是关键字小
        return key < p.key;
    }
    template <class T1,class T2>
    ostream & operator<< (ostream & o,const Pair<T1,T2> & p)
    {
        o << "(" << p.key << "," << p.value << ")" ;
        return o;
    }
    
    int main()
    {
        Pair<string,int> student("Tom",29);
        Pair<int,double> obj(12,3.14);
        cout << student << " " << obj;
        return 0;
    }
    
    输出：
    (Tom,29) (12,3.14)

任意从 `template <class T1,class T2> ostream & operator<< (ostream & o,const Pair<T1,T2> & p)`生成的函数，都是任意Pair摸板类的友元

(3)函数模板作为类的友元

    class A
    {
        int v;
        public:
        A(int n):v(n) { }
        template <class T>
        friend void Print(const T & p);
    };
    template <class T>
    void Print(const T & p)
    {
        cout << p.v;
    }
    
    int main() {
        A a(4);
        Print(a);
        return 0;
    }
    
    输出：
    4
所有从 `template <class T> void Print(const T & p)`
生成的函数，都成为 A 的友元

(4)类模板作为类模板的友元

    template <class T>
    class B {
        T v;
        public:
        B(T n):v(n) { }
        template <class T2>
        friend class A;
    };
    
    template <class T>
    class A {
    public:
        void Func( ) {
            B<int> o(10);
            cout << o.v << endl;
        }
    };
    
    int main()
    {
        A< double > a;
        a.Func ();
        return 0;
    }
    
    输出：
    10

A< double>类，成了B<int>类的友元。任何从A模版实例化出来的类，都是任何B实例化出来的类的友元

## 类模板与静态成员
类模板中可以定义静态成员，那么从该类模板实例化得到的所有类，都包含同样的静态成员

    template <class T>
    class A
    {
    private:
        static int count;
        public:
        A() { count ++; }
        ~A() { count -- ; };
        A( A & ) { count ++ ; }
        static void PrintCount() { cout << count << endl; }
    };
    
    template<> int A<int>::count = 0;
    template<> int A<double>::count = 0;
    int main()
    {
        A<int> ia;
        A<double> da;
        ia.PrintCount();
        da.PrintCount();
        return 0;
    }
    
    输出：
    1 1
