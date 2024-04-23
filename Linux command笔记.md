---
title: Linux command笔记
date: 2020-02-15 14:30:12
tags: linux
categories: linux
---

## 查找包含指定内容的文件

    grep -r 字符串 目录
示例：查找当前目录的包含“stream”内容的文件：

```
grep -r "stream" ./
```

## zip/unzip

```
zip xxx.zip -r <DIR>  
unzip xxx.zip -d <DIR>
```

