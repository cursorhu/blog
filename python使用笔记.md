---
title: python使用笔记
date: 2022-03-09 19:12:00
tags: python
categories: python
---

## Python使用正则表达式示例
Python的正则表达式比较全面的教程，参考[# Python RegEx](https://www.programiz.com/python-programming/regex)

使用背景：芯片ATE测试中，不同ATE平台的测试模式文件格式有不同，需要匹配字符串并按特定转换
转换前：

> Pattern "pll_dll_100m_test" {
> waveform_start:
> W pll_dll_100m_wft;
>                                  
> //Enter PLL/DLL Mode               
> V {pll_dll_100m_group = 0 0 1 0 0 1 1 0 X X ;}
> V {pll_dll_100m_group = 0 1 1 0 0 1 1 0 X X ;}
> V {pll_dll_100m_group = 0 0 1 0 0 1 1 0 X X ;}

转后后：
> //Enter PLL/DLL Mode               
V {pll_dll_100m_group = 0 0 1 0 0 1 1 0 X X ;} W pll_dll_100m_wft;
V {pll_dll_100m_group = 0 1 1 0 0 1 1 0 X X ;}

规则：将以“W_xxx”的字符串放到下一个以“V_xxx”的字符串后面

利用python正则匹配，配合读取文件到字符串数组，实现如下转换：
```python
import os.path
import re

infile_name = input("Please input the name of file in current directory to convert: ")
name_flag = infile_name.find('.')
if name_flag == -1:
    print("file name error, need input the suffix of file name")
    input("Please press Enter key to exit")
    exit(0)
else:
    if os.path.isfile(infile_name):
        outfile_name = infile_name[0:name_flag] + "_updated" + infile_name[name_flag:]
    else:
        print("no such file!")
        input("Please press Enter key to exit")
        exit(0)

infile = open(infile_name, "r",encoding='utf-8')
outfile = open(outfile_name, "w",encoding='utf-8')

lines = infile.readlines()
infile.close()
flag = 0

for index in range(len(lines)):
    str_obj = re.match('[\s]*W[\s].*', lines[index]) #match the "W ..."
    if str_obj != None:
        flag = 1
        temp_index = index
        temp_str = str_obj.group()
    else:
        str_obj = re.match('[\s]*V[\s].*', lines[index]) #match the "V ..."
        if str_obj != None:
            if flag == 1:
                lines[temp_index] = '\n' #clear last "W ..."
                lines[index] = str_obj.group() + ' ' + temp_str + '\n' #add the "W ..." from "V ..." end
                flag = 0
outfile.writelines(lines)
outfile.close()
print("outputfile is " + outfile_name)
input("Please press Enter key to exit")
```

这里的W和V前面加了额外的匹配项：`[\s]*`，是因为文件存在不可见的回车换行等引起，如果不加匹配不到