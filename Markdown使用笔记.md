---
title: Markdown使用笔记
date: 2020-12-12 14:40:07
tags: markdown
categories: markdown

---

# Markdown语法


## 标题

    #
    ##
    ### 

## 无序列表

    - line 
    或者
    * line

## 有序列表

```
1. line
2. line
```

## 转义字符

有的文字或代码和markdown解析有冲突
如$, @等
在这些字符前加转义字符即可：\$, \@

## tab缩进
markdown本身不支持tab缩进，有以下方法：
1.可以用全角输入+2个空格实现缩进
2.输入`&emsp`，就是全角空格符号
3.输入`>`

## 插入链接

    [标题](URL)

## 表格

    |  表头   | 表头  |
    |  ----  | ----  |
    | 单元格  | 单元格 |
    | 单元格  | 单元格 |

# Typora使用

## 导出和打印

有的markdown文本内容中带换行，而Typora阅读时也有换行，照成换行混乱

如果要打印，导出pdf的换行也混乱

解决办法是导出HTML(without style)，然后打印
