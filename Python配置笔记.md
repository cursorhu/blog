---
title: Python配置笔记
date: 2023-10-07 10:25:29
tags: python
categories: python
---

## pip install换源

pip install，有的包一直timeout无法安装

```
ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out
```

### 单次换源

使用 -i 指定国内源

```
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pyside6
```

常用源：

```
清华大学：https://pypi.tuna.tsinghua.edu.cn/simple
阿里云：http://mirrors.aliyun.com/pypi/simple
豆瓣：http://pypi.douban.com/simple
```

### 全局换源

windows下，直接在user目录中创建一个pip目录，如：C:\Users\用户名\pip，即 %HOMEPATH%\pip，然后新建文件pip.ini，输入以下内容（以tsinghua镜像为例）：

```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = pypi.tuna.tsinghua.edu.cn
```

Linux下，配置~/.pip/pip.conf，内容和windows .ini内容一样
