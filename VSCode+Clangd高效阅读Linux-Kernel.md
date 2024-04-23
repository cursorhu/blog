---
title: VSCode+Clangd高效阅读Linux Kernel.md
date: 2023-05-29 11:59:26
tags: vscode
categories: vscode

---

安装kernel编译环境

```
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev
```

安装bear

```
sudo apt install bear
```

使用bear编译kernel，生成compile_commands.json，参考：[Ubuntu22 直接 make 内核成功，但不能使用 bear 命令](https://forums.100ask.net/t/topic/1656/2)

```
bear -- make -j4
```

在编译Kernel的源代码环境安装clangd服务端：

```
sudo apt install clangd
```

在查看代码的VSCode环境安装clangd客户端(即VSCode的clangd插件)：一般通过windows机器的VSCode SSH连接Linux的clangd服务，因此需要将VSCode的remote SSH登陆到Linux机器（注意不要同时使用Xshell等其他SSH工具，否则VSCode remote SSH连不上）

VSCode remote SSH中打开代码后，clangd自动indexing(完成Kernel index需要相当长时间)，CTRL+鼠标左键查看定义，ALT+左键头返回跳转

clangd方式可以很方便找到C函数指针的实现，而VSCode的C++ intellisense跳转不到



参考：[使用VSCode进行linux内核代码阅读和开发](https://zhuanlan.zhihu.com/p/558286384)
