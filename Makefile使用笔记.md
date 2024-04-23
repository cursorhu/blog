---
title: Makefile使用笔记
date: 2020-06-10 17:20:09
tags: makefile
categories: makefile

---

# 文件名替换
1.wildcard
展开多个文件为使用空格分开的、匹配此模式的列表参数
格式
`$(wildcard PATTERN...)`

示例：

    SRC=$(wildcard *.c)

2.patsubst
替换通配符
格式

    $(patsubst %.c,%.o,$(dir))

示例：

    obj := $(patsubst %.c,%.o,$(wildcard *.c))

3.替换引用
patsubst的示例等价于：

    obj=$(dir:%.c=%.o)

引用替换：

    $(var:a=b) 或 ${var:a=b}

含义是把变量var中的每一个值，用b替换掉a

# PHONY
Makefile执行的规则是A:B，表示A依赖于B

 - 有B才能执行A对应的编译操
 - B有更新(检测文件时间)，则A会执行，若B没有更新，不执行A

问题来了，clean: 不需要依赖任何对象，如何执行
PHONY定义伪目标,可以解决源文件不是最终目标直接依赖（即间接依赖）带来的不能自动检查更新规则, 示例如下

    .PHONY: clean
    clean:
        rm -f *.o

PHONY不仅用于clean，对于间接依赖也有用，例如A:B， B:C ，有时C为空，就需要把B定义为PHONY

    OBJS = *.o
    program:  $(OBJS)
            gcc *.o -o program
     
    .PHONY : $(OBJS)
    $(OBJS):
            make -C $(dir $@)

不过一般情况，obj依赖于%.c，总之，对于没有依赖项的对象，需要定义为PHONY

# 通配符
常见通配符

    $@, $^, $<, $?
    
    $@  表示目标文件
    $^  表示所有的依赖文件
    $<  表示第一个依赖文件
    $?  表示比目标还要新的依赖文件列表

示例：
编译Test目录下的.cpp文件，输出test可执行程序
直接指定依赖文件名的makefile写法：

    test: $(wildcard Test/*.cpp)
    	$(CXX) $(CFLAGS) -o test $(wildcard Test/*.cpp) 

虽然wildcard实现了所有依赖的.cpp的通配，编译语句再用wildcard写一次依赖文件不优雅，而且test在依赖中已经写了，编译语句又写一遍 -o test。
编译语句使用通配, 称为通用格式：

    test: $(wildcard Test/*.cpp)
    	$(CXX) $(CFLAGS) -o $@ $^

# 多个源文件分别编译
目录下有很多源文件，每个单独编译和执行的，将每一个文件编译成可执行文件，不用单独命名如：gcc -c xxx.c -o xxx
(1)Makefile实现

    SRC=$(wildcard *.c)
    OBJ=$(SRC:%.c=%.o)
    BIN=$(OBJ:%.o=%)
     
    CC=gcc
    CFLAGS=-Wall -g -c
     
    all:$(BIN)
    
    $(BIN):%:%.o
            $(CC) $^ -o $@
    $(OBJ):%.o:%.c
            $(CC) $(CFLAGS) $^ -o $@
    
    .PHONY: clean
    clean:
            rm -rf $(OBJ) $(BIN)

(2)Shell实现

    #! /bin/bash
    for file in ./*.c
    do
    if [ -f $file ]
    then
    file=${file#./}
    target=${file%.c}
    gcc -o $target $file
    echo $target
    fi
    if [ -d $file ]
    then
    echo $file is mu lu
    fi
    done

(2)Makefile编译指定目录
Makefile可以输入参数，直接在make命令的后面加上参数，如:

    make BUILD_DIR=./foldername/

传入的变量将会覆盖相应Makefile中的`BUILD_DIR`
