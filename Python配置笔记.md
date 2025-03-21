---
title: Python使用笔记
date: 2023-10-07 10:25:29
tags: python
categories: python
---

# pip install换源

pip install，有的包一直timeout无法安装

```
ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out
```

解决办法参考：https://www.runoob.com/w3cnote/pip-cn-mirror.html

## 单次换源

使用 -i 指定国内源

```
pip install numpy -i https://pypi.tuna.tsinghua.edu.cn/simple 
```

常用国内源：

```
清华大学：https://pypi.tuna.tsinghua.edu.cn/simple
阿里云：http://mirrors.aliyun.com/pypi/simple
豆瓣：http://pypi.douban.com/simple
```

## 设为默认

```
#使用清华源更新pip版本
python -m pip install --upgrade pip -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple 
#对当前pip版本，设置默认使用清华源
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
```

## 全局换源

windows下，直接在user目录中创建一个pip目录，如：C:\Users\用户名\pip，即 %HOMEPATH%\pip，然后新建文件pip.ini，输入以下内容（以tsinghua镜像为例）：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = pypi.tuna.tsinghua.edu.cn
```

Linux下，配置~/.pip/pip.conf，内容和windows .ini内容一样

# pip install指定版本

安装pandas报错：

```
 Host machine cpu: x86_64
 Program python found: YES (C:\Users\cursorhu\AppData\Local\Programs\Python\Python312-32\python.exe)
 Need python for x86_64, but found x86
 Run-time dependency python found: NO (tried sysconfig)
 ..\..\pandas\_libs\tslibs\meson.build:32:7: ERROR: Python dependency not found
```

当前使用的是32位Python（Python 3.12.0 32位），而pandas 2.2.3需要64位Python环境。

解决方案有两种：

1. 安装64位版本的Python

1. 安装较旧版本的pandas，它可能兼容32位Python

安装较旧版本的pandas，这样您可以不用更换Python环境：

```
pip install pandas==1.5.3 -i https://pypi.tuna.tsinghua.edu.cn/simple
```

要升级pandas可以：

```
#升级到最新版
python -m pip install --upgrade pandas -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
#升级到指定版
python -m pip install --upgrade pandas==2.2.3 -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
```

