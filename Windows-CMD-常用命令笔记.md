---
title: Windows CMD 常用命令笔记
date: 2023-01-30 10:38:16
tags: cmd
categories: windows

---

# Tree命令生成目录树

> tree 命令的目录格式：TREE 【drive：】【path】【/F】【/A】
> - 可在cmd内输入（help tree 或 tree / ？）查看
> - /F  显示每个文件夹中文件的名称
> - /A  使用ASCII字符，而不使用拓展字符 

示例一：只显示路径名不显示文件名

`TREE 【drive：】【path】`

![image-20230130104934713](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202301301049746.png)

示例二：显示路径名和文件名

`TREE 【drive：】【path】【/F】`

![image-20230130104835813](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202301301048851.png)

示例三：将目录树存入指定文件

`TREE 【drive：】【path】 > 文件路径】`

![image-20230130105102054](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202301301051100.png)
