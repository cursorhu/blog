---
title: Linux Shell笔记：判断语句if-then-else-fi
date: 2020-05-05 14:46:30
tags: shell
categories: linux
---

# 概述
shell中if-then-else-fi判断语句如下：

    a="abc"
    
    if [ $a = "abc" ]
    then
       echo "$a = $b"
    else
       echo "$a != $b"
    fi

注意以下几点：

 - shell中的等号：`=`可用于赋值，也可以用于判断；`==`只用于判断，更规范
 - shell中的if语句各符号间都要空格分隔：`if`和`[ ]`之间要空格；`[ ]`和`“ ”`之间要空格； `"`和`=`之间要空格。否则if语句中的符号会解析失败。
 - shell变量没有数据类型的区分，把任何存储在变量中的值，皆视为“字符串”
 - 对于变量可能为空的情况，需要用双括号`[[ $a = "abc" ]]`
 - if-then可以写在同一行，用;分隔两个语句：`if [ $a = "abc" ];then`

# 不同类型的判断语句
## 关系运算符判断

-eq	检测两个数是否相等，相等返回 true。	[ $a -eq $b ] 返回 false。

-ne	检测两个数是否不相等，不相等返回 true。	[ $a -ne $b ] 返回 true。

-gt	检测左边的数是否大于右边的，如果是，则返回 true。	[ $a -gt $b ] 返回 false。

-lt	检测左边的数是否小于右边的，如果是，则返回 true。	[ $a -lt $b ] 返回 true。

-ge	检测左边的数是否大于等于右边的，如果是，则返回 true。	[ $a -ge $b ] 返回 false。

-le	检测左边的数是否小于等于右边的，如果是，则返回 true。	[ $a -le $b ] 返回 true。

    #!/bin/bash
    
    a=10
    b=20
    
    if [ $a -eq $b ]
    then
       echo "$a -eq $b : a 等于 b"
    else
       echo "$a -eq $b: a 不等于 b"
    fi
    if [ $a -ne $b ]
    then
       echo "$a -ne $b: a 不等于 b"
    else
       echo "$a -ne $b : a 等于 b"
    fi
    if [ $a -gt $b ]
    then
       echo "$a -gt $b: a 大于 b"
    else
       echo "$a -gt $b: a 不大于 b"
    fi
    if [ $a -lt $b ]
    then
       echo "$a -lt $b: a 小于 b"
    else
       echo "$a -lt $b: a 不小于 b"
    fi
    if [ $a -ge $b ]
    then
       echo "$a -ge $b: a 大于或等于 b"
    else
       echo "$a -ge $b: a 小于 b"
    fi
    if [ $a -le $b ]
    then
       echo "$a -le $b: a 小于或等于 b"
    else
       echo "$a -le $b: a 大于 b"
    fi

## 布尔和逻辑运算符判断

!	非运算，表达式为 true 则返回 false，否则返回 true。	[ ! false ] 返回 true。

-o	或运算，有一个表达式为 true 则返回 true。	[ $a -lt 20 -o $b -gt 100 ] 返回 true。

-a	与运算，两个表达式都为 true 才返回 true。	[ $a -lt 20 -a $b -gt 100 ] 返回 false。

    #!/bin/bash
    
    a=10
    b=20
    
    if [ $a != $b ]
    then
       echo "$a != $b : a 不等于 b"
    else
       echo "$a == $b: a 等于 b"
    fi
    if [ $a -lt 100 -a $b -gt 15 ]
    then
       echo "$a 小于 100 且 $b 大于 15 : 返回 true"
    else
       echo "$a 小于 100 且 $b 大于 15 : 返回 false"
    fi
    if [ $a -lt 100 -o $b -gt 100 ]
    then
       echo "$a 小于 100 或 $b 大于 100 : 返回 true"
    else
       echo "$a 小于 100 或 $b 大于 100 : 返回 false"
    fi
    if [ $a -lt 5 -o $b -gt 100 ]
    then
       echo "$a 小于 5 或 $b 大于 100 : 返回 true"
    else
       echo "$a 小于 5 或 $b 大于 100 : 返回 false"
    fi

&&	逻辑的 AND	[[ $a -lt 100 && $b -gt 100 ]] 返回 false

||	逻辑的 OR	[[ $a -lt 100 || $b -gt 100 ]] 返回 true

    #!/bin/bash
    
    a=10
    b=20
    
    if [[ $a -lt 100 && $b -gt 100 ]]
    then
       echo "返回 true"
    else
       echo "返回 false"
    fi
    
    if [[ $a -lt 100 || $b -gt 100 ]]
    then
       echo "返回 true"
    else
       echo "返回 false"
    fi

## 字符串运算符判断

=	检测两个字符串是否相等，相等返回 true。	[ $a = $b ] 返回 false。

!=	检测两个字符串是否相等，不相等返回 true。	[ $a != $b ] 返回 true。

-z	检测字符串长度是否为0，为0返回 true。	[ -z $a ] 返回 false。  

-n	检测字符串长度是否不为 0，不为 0 返回 true。	[ -n "$a" ] 返回 true。

`$` 检测字符串是否为空，不为空返回 true。	[ $a ] 返回 true。

    #!/bin/bash
    
    a="abc"
    b="efg"
    
    if [ $a = $b ]
    then
       echo "$a = $b : a 等于 b"
    else
       echo "$a != $b: a 不等于 b"
    fi
    if [ $a != $b ]
    then
       echo "$a != $b : a 不等于 b"
    else
       echo "$a = $b: a 等于 b"
    fi
    if [ -z $a ]
    then
       echo "-z $a : 字符串长度为 0"
    else
       echo "-z $a : 字符串长度不为 0"
    fi
    if [ -n "$a" ]
    then
       echo "-n $a : 字符串长度不为 0"
    else
       echo "-n $a : 字符串长度为 0"
    fi
    if [ $a ]
    then
       echo "$a : 字符串不为空"
    else
       echo "$a : 字符串为空"
    fi

## 文件检查运算符判断

b file	检测文件是否是块设备文件，如果是，则返回 true。	[ -b $file ] 返回 false。

-c file	检测文件是否是字符设备文件，如果是，则返回 true。	[ -c $file ] 返回 false。

-d file	检测文件是否是目录，如果是，则返回 true。	[ -d $file ] 返回 false。

-f file	检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。	[ -f $file ] 返回 true。

-g file	检测文件是否设置了 SGID 位，如果是，则返回 true。	[ -g $file ] 返回 false。

-k file	检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。	[ -k $file ] 返回 false。

-p file	检测文件是否是有名管道，如果是，则返回 true。	[ -p $file ] 返回 false。

-u file	检测文件是否设置了 SUID 位，如果是，则返回 true。	[ -u $file ] 返回 false。

-r file	检测文件是否可读，如果是，则返回 true。	[ -r $file ] 返回 true。

-w file	检测文件是否可写，如果是，则返回 true。	[ -w $file ] 返回 true。

-x file	检测文件是否可执行，如果是，则返回 true。	[ -x $file ] 返回 true。

-s file	检测文件是否为空（文件大小是否大于0），不为空返回 true。	[ -s $file ] 返回 true。

-e file	检测文件（包括目录）是否存在，如果是，则返回 true。	[ -e $file ] 返回 true。

-S: 判断某文件是否 socket。
-L: 检测文件是否存在并且是一个符号链接。

    #!/bin/bash
    
    file="/root/test.sh"
    
    if [ -r $file ]
    then
       echo "文件可读"
    else
       echo "文件不可读"
    fi
    if [ -w $file ]
    then
       echo "文件可写"
    else
       echo "文件不可写"
    fi
    if [ -x $file ]
    then
       echo "文件可执行"
    else
       echo "文件不可执行"
    fi
    if [ -f $file ]
    then
       echo "文件为普通文件"
    else
       echo "文件为特殊文件"
    fi
    if [ -d $file ]
    then
       echo "文件是个目录"
    else
       echo "文件不是个目录"
    fi
    if [ -s $file ]
    then
       echo "文件不为空"
    else
       echo "文件为空"
    fi
    if [ -e $file ]
    then
       echo "文件存在"
    else
       echo "文件不存在"
    fi



# 判断语句报错："unary operator expected"

在匹配字符串相等时，用了类似这样的语句：

    if [ $STATUS == "OK" ]; then     
    echo "OK"
    fi

在运行时出现了 `[: =: unary operator expected` 的错误

    if [[ $STATUS == "OK" ]]; 
    then     
    echo "OK"
    fi

究其原因，是因为如果变量STATUS值为空，那么就成了 [ = "OK"] ，显然 [ 和 "OK" 不相等并且缺少了 [ 符号，所以报了这样的错误。当然不总是出错，如果变量STATUS值不为空，程序就正常了，所以这样的错误还是很隐蔽的。
或者用下面的方法也能避免这种错误：

    if [ "$STATUS"x == "OK"x ]; 
    then     
    echo "OK"
    fi

当然，x也可以是其他字符。shell中有没有双引号在很多情况下是一致的,因此x可以不加引号。
