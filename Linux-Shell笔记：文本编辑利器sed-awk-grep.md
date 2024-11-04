---
title: Linux Shell笔记：文本编辑利器sed+awk+grep
date: 2020-05-30 14:50:22
tags: shell
categories: linux
---

# shell增删改查概述
Linux Shell环境对文本增删改查，可以通过sed，awk和grep命令完成。

 - sed: 支持特定模式匹配的增加、插入、删除、替换、提取等操作
 - awk: 支持特定模式匹配的过滤查找，如提取某行列的字符串
 - grep: 支持包含特定字符串的结果提取，可配合find和awk实现过滤查找

这几种命令都支持流输入和文本输入，支持管道传递参数，也可以命令行传参。对于多个步骤的处理，可以几个命令串行使用。

# shell的输入参数概述
Shell的命令，如`cat, echo, sed, awk, grep`, 管道命令`|`等，都要有输入参数，即待处理的数据。
输入参数有两种类型：

 - 标准输入：本质是stdin文件，即读取该文件的内容作为命令的输入参数，该文件内容可由其他命令的输出，或者用户输入来产生
 - 命令行输入：直接接受用户在命令行输入的字符串，一次性使用,不存储在stdin

支持标准输入作为参数的命令：`cat, sed, awk, grep, |` 等
只支持命令行输入字符串的命令：`echo, ls`等
标准输入示例：

    cat /etc/passwd | grep root

上面的代码使用了管道命令`|`，管道命令的作用是将左侧命令`cat /etc/passwd`的标准输出转换为标准输入，提供给右侧命令`grep root`作为参数。
以上命令也可以写成命令行输入形式：

    grep root /etc/passwd

不支持标准输入的示例：

    echo "hello world" | echo

输出为空，管道右侧的echo不接受管道传来的标准输入作为参数。
xargs的作用：将标准输入转为命令行参数

    echo "hello world" | xargs echo

输出“hello world”，xargs和管道配合使用，能使管道也支持非标准输入命令。
xargs支持一些选项做进一步细化处理，如-p(打印), -d(分割) 等。

# sed命令
## sed命令概述
sed支持文本编辑，实现增、删、改的功能。
sed命令格式：

    sed [options] 'command' filename

sed的输入参数可以用命令行，管道和xargs传入：
    
    //命令行传入文件名参数
    sed [options] 'command' filename 
    //管道传入文件名参数
    cat filename | sed [options] 'command'
    //xargs传入文件名参数
    cat filename | xargs sed [options] 'command'

sed对文件的编辑在缓冲区，不直接修改文本。要直接修改文本有以下方法：

 - 重定向覆盖文本， `sed - x 'XXX' file.txt > file.txt`
 - 特定的sed命令支持直接修改文本，如`sed -i 'XXX' file.txt`

sed的常用选项：

    -n ：关闭默认输出,只显示匹配的行
    -i ：直接修改读取的文件内容，而不是输出到终端。
    -e ：直接在命令列模式上进行sed的动作编辑；
    -f ：直接将sed的动作写在一个文件内，-f filename 则可以运行filename内的sed动作；
    -r ：启用扩展的正则表达式

sed的常用命令：

    a ：新增行，在指定行的后面附加一行，[address]a\新文本内容
    i ：插入行，在指定行的前面插入一行，[address]i\新文本内容
    s ：替换特定模式匹配的字符串, [address]s/pattern/replacement/flags
    c ：将指定行中的所有内容，替换成该选项后面的字符串, [address]c\用于替换的新文本
    d ：删除行，[address]d
    p ：打印， 通常与参数 -n 一起用，[address]p
    w : 将文本中指定行的内容写入文件, [address]w filename

## sed命令详解
本节从sed文本操作的“增删改查”举例说明其具体命令用法
### 新增和插入：a和i
sed的命令a和i都能实现新增行，其区别在于：

 - a ：append, 指定行后面新增一行
 - i : insert, 表示在指定行前面插入一行

注意区分i命令和i选项
a和i命令的基本格式完全相同：

    [address]a（或 i）\新文本内容

将一个新行插入到数据流第三行前：

    sed '3i\This is an inserted line.' data6.txt
    
    This is line number 1.
    This is line number 2.
    This is an inserted line.
    This is line number 3.
    This is line number 4.
将一个新行附加到数据流中第三行后:

    sed '3a\This is an appended line.' data6.txt
    
    This is line number 1.
    This is line number 2.
    This is line number 3.
    This is an appended line.
    This is line number 4.
将一个多行数据添加到数据流中，只需对要行末尾添加反斜线即可，类似C语言的代码换行

    sed '1i\
    This is one line of new text.\
    This is another line of new text.' data6.txt
    
    This is one line of new text.
    This is another line of new text.
    This is line number 1.
    This is line number 2.
    This is line number 3.
    This is line number 4.

### 删除：d

 - d: delete, 删除行

格式：

    [address]d

删除第三行：

    [root@localhost ~]# cat data6.txt
    This is line number 1.
    This is line number 2.
    This is line number 3.
    This is line number 4.
    [root@localhost ~]# sed '3d' data6.txt
    This is line number 1.
    This is line number 2.
    This is line number 4.

删除二、三行：

    sed '2,3d' data6.txt
    This is line number 1.
    This is line number 4.

删除第三行开始的后续所有行：

    [root@localhost ~]# sed '3,$d' data6.txt
    This is line number 1.
    This is line number 2.

注意：sed d 命令并不会修改原始文件，这里被删除的行只是从 sed 的缓冲区中消失了，原始文件没做任何改变，若要修改源文件，可重定向sed的输出到源文件，覆盖原内容使修改生效。

### 匹配定位：/pattern/
sed的增删操作，可以针对包含匹配字符串的行进行操作，其原理是活用[address], 支持匹配字符串的定位，以插入命令'i'为例，匹配格式如下：

    sed [option] '/匹配字符串/i \插入字符串'
    [option] 通常为 -i, 修改直接在源文件生效

原文件：

    cat testfile 
    hello

在包含"hello"的一行的上一行，插入"upline":
    
    sed -i '/hello/i\upline' testfile

"hello"下一行插入"upline":

    sed -i '/hello/a\down' testfile

修改后的文件：

    cat testfile 
    up
    hello
    down

删除匹配到"hello"的行：

    sed -i '/hello/d' testfile

如果匹配字符串有“/”，为了和sed命令的分隔符“/”，使用“\”转义。
例如删除匹配某个路径字符串的行：
    
    匹配"\etc\install.sh"
    set -i '/\/etc\/install.sh/d' test.txt

sed 命令包含一些预定义特殊符号，代表行尾，行首等。
删除以A开头的行：

    sed -i '/^A.*/d' test.txt
    ^A表示开头是A, .*表示后跟任意字符串

在行尾追加一行内容:

    sed -i '$a\added-content' test.txt
    $表示定位到行尾，a是追加命令，added-content是追加内容

### 替换修改: s
s替换命令内部格式为：

    [address]s/pattern/replacement/flags

 - address 指定要操作的具体行
 - pattern 指定需要替换的内容
 - replacement 指定替换的新内容
 - flags 指定特殊功能

常用的flags:

 - n	1~512 之间的数字，表示指定要替换的字符串出现第几次时才进行替换
 - g	对数据中所有匹配到的内容进行替换，否则只会在第一次匹配成功时做替换操作
 - p	会打印与替换命令中指定的模式匹配的行。此标记通常与 -n    选项一起使用
 - \	转义（转义替换部分包含：&、\ 等）。

替换每行第二个匹配字符串：

    sed 's/test/trial/2' data.txt
    原文本：
    This is first test of the test script
    This is second test of the test script
    输出：
    This is first test of the trial script
    This is second test of the trial script

只替换第二行的匹配字符串：

    sed '2s/test/trial/' data.txt
    原文本：
    This is first test of the test script
    This is second test of the test script
    输出：
    This is first test of the test script
    This is second test of the trial script

全局替换所有匹配字符串：

    sed 's/test/trial/g' data.txt
    原文本：
    This is first test of the test script
    This is second test of the test script
    输出：
    This is first trial of the trial script
    This is second trial of the trial script

### 提取：p
sed p命令配合字符串匹配，可以输出包含指定字符串的行内容。

    sed -n '/string/p' filename
    提取filename文件中,所有包含string的行的内容，并打印到标准输出
    -n是只打印匹配命中的内容

sed p和grep都能提取内容，其区别在于：

 - `sed '/string/p'`是提取指定文件的行内容，重点在内容提取
 - `grep "string" path`是输出包含指定内容的所有文件路径，重点在查找文件位置

![image-20221205145238133](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051452182.png)

![image-20221205145248149](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051452191.png)


## sed进阶与实战
### 多文件批量追加和删除
背景介绍：
底层固件代码有一些功能由宏定义控制，不同功能需要不同的宏定义组合，因此有不同的宏定义文件，命名为.def后缀。批量编译不同版本，编译脚本依次提取def文件夹内的.def文件和源代码一起编译。单个def内容如下：

![image-20221205145321718](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051453765.png)

每一个宏都有多个取值，因此组合起来，需要生成一堆.def文件，批量编译才能覆盖各自功能。
每新增一个宏，都要修改所有.def文件，如果这个宏有两个取值，.def文件数量将翻倍。

![image-20221205145327892](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051453942.png)

人工修改过于低效，使用sed可解决此问题。

查找指定文件，并批量追加一行内容：

    find . -name '*.def*' | xargs sed -i '$a\added-content'

各命令含义：

    find [path] -name "*.def"
    查找path路径下，以.def结尾的所有文件，结果存储在stdout
    |
    管道，将查找结果转存到标准输入stdin
    xargs
    查找结果有很多个，用xargs转成命令行输入，sed才能批量处理
    sed -i '$a\added-content'
        -i 直接修改文件，'$a\added-content' 最后一行追加added-content

查找指定文件，并批量删除匹配某字符串的行：

    find ./defs -name '*.def' | xargs sed -i "/deleteString/d"

查找指定文件，并批量替换匹配某字符串：

    find ./defs -name '*.def' | xargs sed -i "s/oldString/newString/"

在实际shell脚本中，通常由用户输入变量，`$1, $2, $@` 解析变量。a、i、d和s能否在命令中使用双引号解析变量，实现动态编辑？
实验如下：

    ARGS="AA BB"
    find ./defs -name '*.def' | xargs sed -i "$a\${ARGS}"
    find ./defs -name '*.def' | xargs sed -i "$i\${ARGS}"
    find ./defs -name '*.def' | xargs sed -i "/${ARGS}/d"
    find ./defs -name '*.def' | xargs sed -i "s/aabb/$ARGS/"

 - i 和 a 命令不能解析变量，实际追加的就是是${ARGS}
 - d命令可以解析变量，实际删除的是有"AA BB"的行
 - s命令可以解析变量，实际替换后的结果是"AA BB"

结论：将命令中的单引号改成双引号，理论上可以解析$包含的变量，实际上有的命令支持，有的命令不支持。

### 提取文件中的关键内容
背景介绍：
底层固件需要将代码段、数据段等关键信息存储在固件头部，编译过程中用编译器的size命令可以获取这些信息，并存储到文本，但是固件头部需要按自己的格式存储起始地址和大小信息，因此需要用sed提取并编辑该文本。

原文本：
![image-20221205145348933](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051453983.png)

提取后文本：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051453414.png)

sed命令：

    sed -n '/string/p' oldFile | awk '{print $3}' >> newFile
    提取oldFile内包含string的行，并用awk提取第三列，再写入newFile

该命令在Makefile实现，需要根据Makefile和shell特性做修改：

 - @：编译过程隐藏命令输出，类似于后台执行
 - \$(shell xxxx): Makefile执行shell命令
 - \$$: Makefile不能直接用shell的“$”解析变量，用“$$”

![image-20221205145409877](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051454928.png)
