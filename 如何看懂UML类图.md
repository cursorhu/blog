---
title: 如何看懂UML类图
date: 2020-12-09 14:33:00
tags: UML
categories: UML
---

# 前言
统一建模语言（Unified Modeling Language，UML）是用来设计软件蓝图的可视化建模语言，1997年被国际对象管理组织（OMG）采纳为面向对象的建模语言的国际标准。它的特点是简单、统一、图形化、能表达软件设计中的动态与静态信息。
统一建模语言能为软件开发的所有阶段提供模型化和可视化支持。而且融入了软件工程领域的新思想、新方法和新技术，使软件设计人员沟通更简明，进一步缩短了设计时间，减少开发成本。它的应用领域很宽，不仅适合于一般系统的开发，而且适合于并行与分布式系统的建模。
UML从目标系统的不同角度出发，定义了用例图、类图、对象图、状态图、活动图、时序图、协作图、构件图、部署图等 9 种图
本文介绍开发中常用的类图

# 类图
类（Class）是指具有相同属性、方法和关系的对象的抽象，它封装了数据和行为，是面向对象程序设计（OOP）的基础，具有封装性、继承性和多态性等三大特性。在 UML 中，类使用包含类名、属性和操作且带有分隔线的矩形来表示。
首先讲解关系, 先来看一个例子：
![image-20221205114810203](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148263.png)

分析一下上面的图, 首先从动物开始
动物是一个类 动物依赖氧气和水
然后鸟继承了动物，所以鸟的父类是动物 所以鸟是属于动物
然后鸟和翅膀是组合关系 一只鸟有两个翅膀
大雁鸭子和企鹅都是鸟所以继承了鸟类
大雁会有大雁群，大雁群是由大雁组成所以是聚合关系
企鹅和气候是关联关系因为企鹅需要依赖气候
然后再看大雁 大雁会飞翔 所以就实现了飞翔接口
唐老鸭是属于鸭子的 所以唐老鸭继承了鸭子这个类
上图是借鉴了大话设计模式里面的图。下面具体介绍各个符号的作用

## 类
类一般是用三层矩形框表示，第一层表示类的名称，第二层表示的是字段和属性，第三层则是类的方法。第一层中，如果是抽象类，需用斜体显示
![image-20221205114819689](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148740.png)

## 类符号
![image-20221205114830981](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148030.png)
看上面的学生类里面有五个属性和两个方法

    +号表示公共的 public
    -表示 私有的 private
    #表示protected

带下划线表示静态属性，一般表示方法: +属性:类型。
括号内表示参数，后面是返回类型, 没有表示无返回值
## 包
包(Package)： 是一种常规用途的组合机制。在UML中用一个Tab框表示，Tab里写上包的名称，框里则用来放一些其他子元素，比如类，子包等等。
![image-20221205114837947](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148985.png)

## 接口
接口(interface)：接口包含操作但不包含属性，且它没有对外界可见的关联
![image-20221205114843929](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148968.png)

## 关系
### 依赖
依赖(Dependency) 表示的是类之间的调用关系。UML中用带箭头的虚线表示依赖关系，而箭头所指的则是被依赖的类。
![image-20221205114849672](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148704.png)

### 泛化
泛化(Generalization)： 表示的是类之间的继承关系，注意是子类指向父类。UML中用带空心三角箭头的实线表示泛化关系，箭头指向的是一般个体。
![image-20221205114855112](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051148143.png)

### 关联
关联(Association) 表示的是类与类之间存在某种特定的对应关系。UML中用双向带箭头的虚线表示关联关系，箭头两端为相互关联的两个类
![image-20221205114902153](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051149187.png)

### 聚合
聚合(Aggregation)： 是关联关系的一种特例，表示的是整体与部分之间的关系，部分不能离开整体单独存在。UML中用空心菱形头的实线表示聚合关系，菱形头指向整体
![image-20221205114909206](C:\Users\thomas.hu\AppData\Roaming\Typora\typora-user-images\image-20221205114909206.png)

### 组合
组合(Composition)： 是聚合的一种特殊形式，表示的是类之间更强的组合关系。UML中用实心菱形头的实线来表示组合，菱形头指向整体。
![image-20221205114949528](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051149567.png)
