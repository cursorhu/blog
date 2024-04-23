---
title: Git多人协作下的换行符问题
date: 2020-09-06 11:23:00
tags: Git
categories: Git
---

# 背景
项目文件中有如下类型文件：

    Makefile, .sh, .bat, .cfg, .exe

源码用git管理，客户端用cygwin实现windows内的linux环境

问题：如何解决git多人协作下的linux、windows换行符差异问题？

(1)什么是换行符
LF："\n"，Linux的换行符, 只包含“换行”；
CRLF："\r\n"，Windows的换行符，包含“回车+换行”;

(2)不同换行符带来什么问题
用git管理代码，必定有远程端和本地端两个仓库，两端的操作系统不同，换行符可能有差异;

多人协作时，本地端可能有linux环境和windows, 如果所有人都是linux就不存在换行符差异的问题；如果有windows和linux就有该问题;
例如A上传了Linux换行符LF的代码到远程，B 本地环境是windows, B git pull下来，其git-config设置了自动转换成本地换行，将代码换行全成了CRLF，B上传后，远程仓库变成CRLF换行。此时A git diff查看，所有代码都有换行差异，扰乱真正的代码diff。

不仅是影响git diff， 换行差异还影响脚本执行
- 例如LF换行的.sh，git pull到windows环境后换行转换成CRLF, 导致sh无法正常执行；
- .bat调用.exe读取.cfg内的一行，.exe是windows程序，其readline方法只能识别CRLF换行，无法读取LF换行的.cfg文件内容

# git的自动换行符转换配置
参考：[core.autocrlf](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-%E9%85%8D%E7%BD%AE-Git)

假如你正在 Windows 上写程序，而其他人用的是其他系统（或相反），你可能会遇到 CRLF 问题。 这是因为 Windows 使用回车（CR）和换行（LF）两个字符来结束一行，而 macOS 和 Linux 只使用换行（LF）一个字符。 虽然这是小问题，但它会极大地扰乱跨平台协作。许多 Windows 上的编辑器会悄悄把行尾的换行字符转换成回车和换行， 或在用户按下 Enter 键时，插入回车和换行两个字符。

Git 可以在你提交时自动地把回车和换行转换成换行，而在检出代码时把换行转换成回车和换行。 你可以用 core.autocrlf 来打开此项功能。 如果是在 Windows 系统上，把它设置成 true，这样在检出代码时，换行会被转换成回车和换行：

    $ git config --global core.autocrlf true

如果使用以换行作为行结束符的 Linux 或 macOS，你不需要 Git 在检出文件时进行自动的转换； 然而当一个以回车加换行作为行结束符的文件不小心被引入时，你肯定想让 Git 修正。 你可以把 core.autocrlf 设置成 input 来告诉 Git 在提交时把回车和换行转换成换行，检出时不转换：

    $ git config --global core.autocrlf input

这样在 Windows 上的检出文件中会保留回车和换行，而在 macOS 和 Linux 上，以及版本库中会保留换行。

如果你是 Windows 程序员，且正在开发仅运行在 Windows 上的项目，可以设置 false 取消此功能，把回车保留在版本库中：

    $ git config --global core.autocrlf false

**使用`git config --global core.autocrlf input`就可以做到windows的CRLF换行自动转换成LF换行存储在git远程仓库，且git pull/clone到本地时维持LF换行，不影响.sh等linux shell script执行。**

# 手动换行符转换

 - dos2unix FilePath
 - unix2dos FilePath
 - windows2linux

    sed -i 's/.$//' FilePath

 - linux2windows

    sed -i 's/$/\r/' FilePath