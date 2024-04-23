---
title: Vim使用笔记.
date: 2021-04-17 12:30:05
tags: vim
categories: linux
---
最近在ChromeOS上做一些shell script测试用例开发，ChromeOS基于Debian9，但没有Ubuntu那种GNOME的gedit编辑器，更不谈安装Linux版VSCode，正好借此机会练习一下之前一直不熟悉的vim编辑器。

ChromeOS不方便截图，所以本文以ubuntu上的linux0.11代码为例，整理vim最常用的操作。

关于Linux上的文本编辑器基础概念，可以参考<Linux命令行与shell脚本编程大全.第3版>

## 1. 三种编辑模式
我将vim归为三种编辑模式：
- 文本编辑模式
文本编辑模式是默认模式，vim编辑器会将按键解释成命令。在任意模式按esc进入此默认模式。

- 文本插入模式
文本插入模式， vim会将你在当前光标位置输入的每个键都插入到缓冲区，即文本输入字符。在普通模式下按下"i 键" 进入(含义:insert)

- 命令行模式
命令行模式和shell命令行类似，在普通模式下按下": 键"进入(形似shell terminal的冒号)

怎么知道当前处于哪种模式？
vim左下角是状态行，以下是三种模式的状态示例：
- `vim init/main.c`默认进入文本编辑模式，下面显示文件名和行号
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205171047554.png)

输入i, 进入文本插入模式，下面显示insert状态
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205171052433.png)

按esc退出文本编辑，再输入`:` 进入命令行模式，例如输入`:wq`保存文件
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205171053124.png)

还有一种visual模式是复制粘贴时会用到：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205181032041.png)


下面讲文本编辑模式和命令行模式的常用命令
主要分为几类场景：
- 光标移动
- 增删改查
- 文件保存

光标移动类：
| 操作      | 作用         |
| --------- | ----------- |
| gg    | 移到第一行 (gg重来)       |
| G | 移到最后一行 (记为大G)        |
| PageUp/PageDown    | 翻页       |
| :行号 | 光标移动到指定行(属于命令行模式)       |

增删改查类：
| 操作      | 作用         |
| --------- | ----------- |
| i    | 进入insert模式，在当前光标的左侧输入       |
| a | 追加文本（append），在当前光标的右侧输入        |
| o | 插入空行，在空行光标处可输入        |
| dd    | 删除当前行 (记为双击delete)       |
| dw | 删除当前词（记为delete word）       |
| delete键，或x键 | 删除当前字符，注意，Backspace在vim没有删除的作用！       |
| v+方向键选中+y | 复制选中的文本，v: visual，可视光标选中的文本范围， y: yank 复制       |
| yw | 复制当前词       |
| yy | 复制当前行       |
| p | 在复制之后，粘贴文本(paste)，注意粘贴内容来自vim缓冲区，而不是外部剪切板的       |
| dw/dd + p | 剪切，d操作删除的文本位于缓冲区，可以直接用p粘贴       |
| /字符串 | (当前文件内)查找字符串，按n查找下一个       |
| :s/old/new/g | (当前文件内)全局查找和替换       |
| u    | 撤销上一步       |

文件保存类：
| 操作      | 作用         |
| --------- | ----------- |
| :q!    | 不保存文件退出       |
| :wq    | 保存文件退出         |

## 2.多文件编辑
下面讲多个文本的常用命令
主要分为几类场景：
- 多文本搜索
- 多文件编辑

多文本搜索类：
参考[# Vim Search and Replace With Examples](https://thevaluable.dev/vim-search-find-replace/)
本文只以quickfix方式为例：
| 操作      | 作用         |
| --------- | ----------- |
| `:vimgrep pattern **`    | 搜索当前目录和子目录的包含指定pattern的文件，vimgrep可缩写为vim, ** 表示递归子目录       |
| `:vimgrep pattern **/*.c` | 同上，只搜索.c文件        |
| :copen    | 搜索完后使用此命令打开文件列表，才能用光标选择       |
| :cn (cnext) 和 :cp (cprev)  | 上下选择搜索文件列表       |

示例：搜索linux0.11下的所有包含main的.c文件
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205171959744.png)

quickfix list即文件列表，copen后可方向键选择打开文件
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205172002631.png)

- 多文件编辑
  **打开多个文件，分隔并列显示**
1. 用vim打开文件后，命令行输入`:vs newfile`，竖排并列打开新文件（vs是vertical split缩写，竖排分隔）
2. 特殊用法：`:vs ./`可以打开当前路径下的所有文件列表
3. 在窗口间切换：`ctrl + ww`
4. 关闭文件只需要先切换到窗口再`:q!`
5. 调整竖排的窗口比例：
    先按ctrl+w选择窗口模式，再按<>+-调整。< 左移，> 右移，+ 上移， - 下移。

示例：实现类似IDE的界面，左侧是文件列表，下侧是查找栏，右侧文件内容
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205181006035.png)

  **打开多个文件，不并列显示**
直接`:open file`打开新文件, 用 `:bn 和 :bN` (buffer next)切换文件, 

  **多文件之间复制粘贴**
vim的多个文件直接可以直接用 y + p 命令复制粘贴，因为共用vim环境的缓冲区

  退出所有文件
`:qall!` 和 `:wqall`

## 3.类似IDE的跳转功能
推荐cscope插件，具体参考[## The Vim/Cscope tutorial](http://cscope.sourceforge.net/cscope_vim_tutorial.html)

关键步骤：
- 建立cscope.vim
将  http://cscope.sourceforge.net/cscope_maps.vim  另存到文件`~/.vim/plugin/cscope_maps.vim`
- 源码目录建立cscope.out
`cscope -R` 建立符号索引，`ctrl+D` 退出
- 打开某符号的代码
例如 `vim -t main` 打开main所在文件
- 查找函数的定义和调用
如果光标已经在函数上，用 "`ctrl +＼`" 再输入s，查找所有调用、定义该函数的列表，输入索引号回车
更推荐用cscope的命令行，`:cs f s 函数名` 是一样的结果，且光标不需要位于函数上。参数含义 f: find, s: symbol
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202205181103274.png)
- 跳转回之前的位置
  "`ctrl + t`

## 4.vim配置文件修改配色，行号

在有的Linux服务器上，Vim默认深蓝色亮瞎眼，修改配色为流行的Molokai.

效果对比:

默认配色看不清注释内容
![image-20221206143528332](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061435384.png)
Molokai配色
![image-20221206143701354](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212061437401.png)

配置过程：

默认的配色方案：

    ls /usr/share/vim/vim74/colors

下载molokai配色文件,拷贝到vim配色文件目录

    cd ~
    git clone git@github.com:tomasr/molokai.git
    cd molokai/colors
    cp molokai.vim /usr/share/vim/vim74/colors

在home下创建.vimrc用于配色详细设置

    cd ~
    vim .vimrc

.vimrc设置如下：

      set t_Co=256
      set background=dark
      set ts=4
      set nu!
      syntax on
      colorscheme molokai

`:wq`保存后即生效      
如果要全局用户通用，`vim /etc/vimrc`

