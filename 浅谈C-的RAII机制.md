---
title: 浅谈C++的RAII机制
date: 2020-11-08 16:57:03
tags: C++
categories: C++

---

# 1.资源与内存分配
资源的概念:资源是数量有限且对系统正常运转具有一定作用的元素。比如，内存，文件句柄，网络套接字（network sockets），互斥锁（mutex locks）等等
对于进程，这些资源都作为某种数据结构存储在内存中。
程序运行需要分配内存来管理以上资源，内存分配可以分为三类：

 - 静态分配：如创建一个进程执行某段代码，需要加载该代码的代码段，数据段等数据到内存中，其中数据段包含已初始化的全局数据，可以称为是静态的内存分配
 - 自动分配：进程内函数的调用和返回，以及其内部的局部变量创建和销毁，对应该进程高地址的入栈出栈，这个是操作系统自动处理的，无需应用程序控制
 - 动态分配：静态数据和堆栈之前的空间（称为堆），可由应用程序动态分配，同时，也必须由应用程序释放。所谓的内存的动态分配与释放，通常讨论的是这种情况

以32位Linux环境的应用程序为例，每个进程可见的（虚拟）内存分布如下，C/C++常用的malloc/free, new/delete对应的内存分配释放都在.heap段内
![image-20221208165846274](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212081658341.png)

# 2.动态内存管理的缺陷
我们在使用资源时必须严格遵循的步骤是：
 1. 获取资源
 2. 使用资源
 3. 释放资源

代码形式：

    void UseResources()    
    {  
        // 获取资源1  
        // ...  
        // 获取资源n  
         
        // 使用这些资源  
         
        // 释放资源n  
        // ...  
        // 释放资源1  
    } 

当代码量和复杂度达到一定程度，这种手动资源管理容易出错，且难以避免
例如C++使用new和delete时可能发生的一些错误是：

 - 内存泄漏：例如，使用new分配对象，而忘记删除该对象，打开文件，忘记关闭文件等等
 - 过早删除（或悬挂引用）：持有指向对象的另一个指针，删除该对象，但是还有其他指针在引用它。
 - 双重删除：尝试两次删除一个对象

# 3.RAII：将资源管理交给系统
 - 自动内存管理，局部变量能在调用函数时分配，退出函数时释放
 - 类是 C++ 中的主要抽象工具，将资源抽象为类，用局部对象来表示资源，把管理资源的任务转化为管理局部对象的任务

RAII 就是基于以上思想，折中了全手动和全自动的内存管理，手动的选择管理哪些资源，自动的分配和释放资源。有效地实现了 C++ 资源管理的自动化

RAII（Resource Acquisition Is Initialization, 资源获取即初始化）: 
是80年代，Bjarne Stroustrup为C++发明了的范例。
具体实现方法：将资源的声明周期，绑定到对象的生命周期，即将资源分配和释放操作，包含到指定对象的构造函数和析构函数中，这些构造函数和析构函数在适当的时候由编译器自动调用，资源数据包含到对象的成员中。

一个简单示例：

（1）常规内存管理

    #include <iostream> 
    using namespace std; 
    int main() 
    { 
        int *testArray = new int [10]; 
        // Here, you can use the array 
        delete [] testArray; 
        testArray = NULL ; 
        return 0; 
    }

（2）RAII方式

    #include <iostream> 
    using namespace std; 
    
    class ArrayOperation 
    { 
    public : 
        ArrayOperation() 
        { 
            m_Array = new int [10]; //构造函数包含资源的分配
        } 
     
        void InitArray()  //使用资源
        { 
            for (int i = 0; i < 10; ++i) 
            { 
                *(m_Array + i) = i; 
            } 
        } 
     
        void ShowArray() //使用资源
        { 
            for (int i = 0; i <10; ++i) 
            { 
                cout<<m_Array[i]<<endl; 
            } 
        } 
     
        ~ArrayOperation()  //析构函数包含资源的释放
        { 
            cout<< "~ArrayOperation is called" <<endl; 
            if (m_Array != NULL ) 
            { 
                delete[] m_Array;  
                m_Array = NULL ; 
            } 
        } 
     
    private : 
        int *m_Array;  //成员变量包含资源
    }; 
     
    int main() 
    { 
        ArrayOperation arrayOp; //资源自动分配
        arrayOp.InitArray(); 
        arrayOp.ShowArray(); 
        return 0;           //资源自动释放
    }

根据RAII对资源的所有权控制，分为常性类型和外部初始化类型
上述示例即为常性类型，也是最纯粹的RAII形式，最容易理解，最容易编码。获取资源的地点是构造函数，释放点是析构函数，并且在这两点之间的一段时间里，任何对该RAII类型实例的操纵都不应该从它手里夺走资源的所有权
外部初始化类型是指资源在外部被创建，并被传给RAII实例的构造函数，后者进而接管了其所有权。boost::shared_ptr<>和std::auto_ptr<>都是此类型

# 4.RAII的应用场景
常见的应用有：

 - 文件操作
 - 智能指针
 - 互斥量

## 4.1文件操作
（1）常规形式

    void UseFile(char const* fn)  
    {  
        FILE* f = fopen(fn, "r");        // 获取资源  
        // 在此处使用文件句柄f...代码          // 使用资源  
        fclose(f);                       // 释放资源  
    }  

（2）RAII
文件类：

    class FileHandle {  
    public:  
        FileHandle(char const* n, char const* a) { p = fopen(n, a); } 
        ~FileHandle() { fclose(p); }  
    private:   
        FileHandle(FileHandle const&);  
        FileHandle& operator= (FileHandle const&); // 禁止拷贝操作  
        FILE *p;  
    }; 

 FileHandle 类的构造函数调用 fopen() 获取资源；FileHandle类的析构函数调用 fclose()释放资源。请注意，考虑到FileHandle对象代表一种资源，它并不具有拷贝语义，因此将拷贝构造函数和赋值运算符声明为私有成员
 使用：

    void UseFile(char const* fn)  
    {  
        FileHandle file(fn, "r");  
        // 在此处使用文件句柄  
        // 超出此作用域时，系统会自动调用file的析构函数，从而释放资源
    }  

## 4.2互斥量
C++标准库提供lock_guard类实现mutex分配与释放，其实现就是RAII方式。

    template<class... _Mutexes>
    	class lock_guard
    	{	// class with destructor that unlocks mutexes
    public:
    	explicit lock_guard(_Mutexes&... _Mtxes)
    		: _MyMutexes(_Mtxes...)
    		{	// construct and lock
    		_STD lock(_Mtxes...);
    		}
     
    	lock_guard(_Mutexes&... _Mtxes, adopt_lock_t)
    		: _MyMutexes(_Mtxes...)
    		{	// construct but don't lock
    		}
     
    	~lock_guard() _NOEXCEPT
    		{	// unlock all
    		_For_each_tuple_element(
    			_MyMutexes,
    			[](auto& _Mutex) _NOEXCEPT { _Mutex.unlock(); });
    		}
     
    	lock_guard(const lock_guard&) = delete;
    	lock_guard& operator=(const lock_guard&) = delete;
    private:
    	tuple<_Mutexes&...> _MyMutexes;
    	};

使用多线程时，经常会涉及到共享数据的问题，C++中通过实例化std::mutex创建互斥量，通过调用成员函数lock()进行上锁，unlock()进行解锁。不过这意味着必须记住在每个函数出口都要去调用unlock()，也包括异常的情况，这非常麻烦，而且不易管理。C++标准库为互斥量提供了一个RAII语法的模板类std::lock_guard，其会在构造函数的时候提供已锁的互斥量，并在析构的时候进行解锁，从而保证了一个已锁的互斥量总是会被正确的解锁。上面的代码属于mutex头文件

## 4.3智能指针
先看一个例子，用RAII管理指针

    #include <iostream>
    #include <mutex>
    #include <fstream>
    using namespace std;
    
    enum class shape_type {
        circle,
        rectangle,
    };
    
    class shape {
    public:
        shape() { cout << "shape" << endl; }
        virtual void print() {
            cout << "I am shape" << endl;
        }
        virtual ~shape() {}
    };
    
    class circle : public shape {
    public:
        circle() { cout << "circle" << endl; }
        void print() {
            cout << "I am circle" << endl;
        }
    };
    
    class rectangle : public shape {
    public:
        rectangle() { cout << "rectangle" << endl; }
        void print() {
            cout << "I am rectangle" << endl;
        }
    };
    
    // 利用多态上转,如果返回值为shape,会存在对象切片问题。
    shape *create_shape(shape_type type) {
        switch (type) {
            case shape_type::circle:
                return new circle();
            case shape_type::rectangle:
                return new rectangle();
        }
    }
    
    class shape_wrapper {
    public:
        explicit shape_wrapper(shape *ptr = nullptr) : ptr_(ptr) {}
    
        ~shape_wrapper() {
            delete ptr_;
        }
    
        shape *get() const {
            return ptr_;
        }
    
    private:
        shape *ptr_;
    };


​    
​    
    int main() {
    
        // 第一种方式, 手动管理指针
        shape *sp = create_shape(shape_type::circle);
        sp->print();
        delete sp; //显式delete
    
        // 第二种方式， RAII管理指针，一般封装到函数，更快释放
        shape_wrapper ptr(create_shape(shape_type::circle));
        ptr.get()->print();
    
        return 0;
    }

C++标准库的智能指针：auto_ptr(C++11弃用), unique_ptr,shared_ptr, weak_ptr
可以参考[WindSun:详解C++11智能指针](https://www.cnblogs.com/WindSun/p/11444429.html)

## 4.4实现自己的RAII类
一般情况下，RAII临时对象不允许复制和赋值，当然更不允许在heap上创建，所以先写下一个RAII的base类，使子类私有继承Base类来禁用这些操作：

    class RAIIBase  
    {  
    public:  
        RAIIBase(){}  
        ~RAIIBase(){}//由于不能使用该类的指针，定义虚函数是完全没有必要的  
          
        RAIIBase (const RAIIBase &);  
        RAIIBase & operator = (const RAIIBase &);  
        void * operator new(size_t size);   
        // 不定义任何成员  
    };

要写自己的RAII类时就可以直接继承该类的实现

    template<typename T>  
    class ResourceHandle: private RAIIBase //私有继承 禁用Base的所有继承操作  
    {  
    public:  
        explicit ResourceHandle(T * aResource):r_(aResource){}//获取资源  
        ~ResourceHandle() {delete r_;} //释放资源  
        T *get()    {return r_ ;} //访问资源  
    private:  
        T * r_;  
    };

将Handle类做成模板类，这样就可以将class类型放入其中。另外，ResourceHandle可以根据不同资源类型的释放形式来定义不同的析构函数。由于不能使用该类的指针，所以不使用虚函数。

# 5.GC和RAII
在没有RAII的时代，GC和非GC语言是水火不容，GC追求开发效率和稳健设计，非GC如C++最求极致性能和绝对控制。RAII的设计机制，兼顾了两者的优点。
如果用三个等级代表程序员对系统资源的使用权限，如下：

 - 动态分配：C++的new/delete之类，程序员100%负责内存使用和释放，编译器、操作系统不额外干预
 - 垃圾回收(GC)：java/go语言之类，程序员只负责要内存，而不用管，也管不了内存释放，其由该语言运行环境管理，规则可以描述成：如果一个资源没被任何对象使用(例如没有指针指向它)，运行环境定时或者其他方式检测到后，自动释放该资源，该过程对程序员不可控。可以说程序员有50%的权限，即想要就能要，但想甩却不能甩
 - RAII：程序员负责资源编排，运行时的分配与释放由系统自动完成，可以说程序员有90%的权限，放权10%给系统

# 小结
RAII的本质内容是用对象代表资源，把管理资源的任务转化为管理对象的任务，将资源的获取和释放与对象的构造和析构对应起来，从而确保在对象的生存期内资源始终有效，对象销毁时资源必被释放。换句话说，拥有对象就等于拥有资源，对象存在则资源必定存在。
具体实现：

 - 资源在构造函数中获取
 - 资源在析构函数中释放
 - 资源是类的成员变量
 - 类的实例是堆栈分配的

相关文章
[C++那些事：RAII](https://light-city.club/sc/codingStyleIdioms/RAII/)
