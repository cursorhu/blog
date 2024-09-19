---
title: VSCode配置笔记
date: 2022-12-08 11:50:24
tags: vscode
categories: vscode
---

# VSCode配置笔记

## 修改工作区存储目录

VSCode会将每个工作区的一些配置、扩展、缓存等默认保存在C盘的AppData\Code\workspaceStorage，使用一段时间后数据能达到上十GB。

当C盘空间不足，用SpaceSniffer可以找到这些“数据垃圾”，但每隔一段时间清理也不是一劳永逸。

修改workspaceStorage存储路径到非系统盘：

1.首先选择VSCode在开始栏，状态栏，或桌面栏的快捷方式图标，常用哪个就修改哪个，右键属性：

添加启动的命令行选项，指定user-data-dir:

```
--user-data-dir "目标路径，例如F:\VSCodeWorkspaceStorage"
```

![image-20221208120051137](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212081200216.png)

2.转移已有的workspaceStorage.

修改完成后，将%AppData%\Code下的所有内容拷贝到设置的目录中;  也可以删除%AppData%\Code，但是需要重新配置VSCode。

## 常用快捷键

### 代码注释

以双斜杠//注释和取消注释:

```
方法一：
注释：ctrl + / 
取消注释：ctrl + /
```

```
方法二：
注释：ctrl + k, ctrl + c 
取消注释：ctrl + k, ctrl + u
```

以星号/**/注释和取消注释:

```
注释：shift + alt + a 
取消注释：shift + alt + a
```

### 更改快捷键

File->Preference->KeyboardShortCuts

例如可以把块注释/**/快捷键改成`ctrl+Alt+/`，和行注释`ctrl+/`达成统一：

选择recording keys，直接录入要修改的快捷键

![image-20230220110133891](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202302201101988.png)

## 项目文件过滤

在项目的顶层目录中新建 **.vscode** 文件夹，在该文件夹下面新建 **settings.json** 文件

例如，对于Linux kernel项目，编译过的目录有大量编译输出文件(.o, .ko, .mod等)，只想查看和搜索驱动目录下的源码，过滤示例如下：

```
{
    "files.exclude": {
        "**/*.cmd": true, //当前所有目录的所有以.cmd结尾的文件
        "**/*.a": true,
        "**/*.o": true,
        "**/*.d": true,
        "**/*.mod": true,
        "**/*.mod.c": true,
        "**/*.ko": true,

        "[^drivers]*": true, //除了包含'd''r''i''v''e''r''s'目录以外的所有目录，近似等效于除了"drivers"文件夹以外的文件都被files.exclude
        "[^include]*": true,
    },

    "search.exclude": {
        "[^driver]*": true,
        "[^include]*": true,
    }
}

```

正则表达式参考 [正则表达式排除字符](https://geek-docs.com/regexp/regexp-tutorials/75_regular_expression_exclude_characters.html#:~:text=Regex%20%E6%98%AF%E4%B8%80%E7%A7%8D%E5%BC%BA%E5%A4%A7%E7%9A%84%E6%96%87%E6%9C%AC%E5%A4%84%E7%90%86%E5%B7%A5%E5%85%B7%EF%BC%8C%E8%83%BD%E5%A4%9F%E7%94%A8%E9%AB%98%E6%95%88%E7%9A%84%E6%96%B9%E5%BC%8F%E5%AE%8C%E6%88%90%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%8C%B9%E9%85%8D%E3%80%81%E6%9F%A5%E6%89%BE%E3%80%81%E6%9B%BF%E6%8D%A2%E7%AD%89%E6%93%8D%E4%BD%9C%E3%80%82%20%E5%9C%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%B8%AD%EF%BC%8C%E6%9C%89%E6%97%B6%E9%9C%80%E8%A6%81%E6%8E%92%E9%99%A4%E6%9F%90%E4%BA%9B%E7%89%B9%E5%AE%9A%E7%9A%84%E5%AD%97%E7%AC%A6%E3%80%82%20%E6%9C%AC%E6%96%87%E5%B0%86%E4%BB%8B%E7%BB%8D%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E6%9D%A5%E6%8E%92%E9%99%A4%E6%8C%87%E5%AE%9A%E7%9A%84%E5%AD%97%E7%AC%A6%E3%80%82%20%E6%8E%92%E9%99%A4%E5%8D%95%E4%B8%AA%E5%AD%97%E7%AC%A6%20%E5%9C%A8%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%B8%AD%EF%BC%8C%E6%88%91%E4%BB%AC%E5%8F%AF%E4%BB%A5%E4%BD%BF%E7%94%A8%20%5E%20%E7%AC%A6%E5%8F%B7%E6%9D%A5%E8%A1%A8%E7%A4%BA%E5%8C%B9%E9%85%8D%E4%B8%8D%E5%8C%85%E5%90%AB%E6%9F%90%E4%B8%AA%E7%89%B9%E5%AE%9A%E5%AD%97%E7%AC%A6%E7%9A%84%E5%AD%97%E7%AC%A6%E9%9B%86%E3%80%82,%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%EF%BC%8C%E5%8F%AF%E4%BB%A5%E4%BD%BF%E7%94%A8%E4%BB%A5%E4%B8%8B%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%EF%BC%9A%20%20a%5D%20%E4%B8%8A%E8%BF%B0%E6%AD%A3%E5%88%99%E8%A1%A8%E8%BE%BE%E5%BC%8F%E4%B8%AD%E7%9A%84%20%5E%20%E8%A1%A8%E7%A4%BA%E6%8E%92%E9%99%A4%E5%AD%97%E7%AC%A6%EF%BC%8C%20%5B%5D%20%E5%8C%85%E5%90%AB%E4%B8%80%E4%B8%AA%E5%AD%97%E7%AC%A6%E9%9B%86%E5%90%88%E3%80%82)

## VSCode remote免密码登录(SSH密钥认证)

Windows端的VSCode remote如何配置参考[Remote Development using SSH](https://code.visualstudio.com/docs/remote/ssh)，Linux服务器配置好SSH服务后直接连接即可。

日常使用经常需要重启Linux服务端，需要重新输入密码登录；使用SSH密钥可以免密码登录。

SSH密钥登录的流程：

- 在进行SSH连接之前，SSH客户端需要先生成自己的公钥私钥对，并将自己的公钥存放在SSH服务器上。

- SSH客户端发送登录请求，SSH服务器就会根据请求中的用户名等信息在本地搜索客户端的公钥，并用这个公钥加密一个随机数发送给客户端。

- 客户端使用自己的私钥对返回信息进行解密，并发送给服务器。

- 服务器验证客户端解密的信息是否正确，如果正确则认证通过。

  ![image-20230822110009079](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202308221100363.png)

**(1)Windows客户端生成ssh key**

`win+R -> ssh-keygen` 生成密钥对，id_rsa.pub是公钥，id_rsa是私钥；

如果已经有ssh-key, 不需要重新生成；如果已有的key不能配置生效，参考如下方式生成重命名的ssh-key，后续流程一致。

![image-20230822111509800](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202308221115183.png)

**(2)Linux服务端生成ssh key**

用xshell或samba拷贝windows端的`C:\Users\用户名\.ssh\id_rsa.pub`到Linux服务端的~/**.ssh** 

拷贝到authorized_keys，并修改权限，否则Vscode remote不能访问。

```
cat id_rsa.pub >> authorized_keys
chmod 777 authorized_keys 
service sshd restart
```

**(3)配置VSCode remote**

ssh配置文件`C:\Users\用户名\.ssh\config`

![image-20230822104703184](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202308221047708.png)

添加IdentityFile字段，填写windows本机的id_rsa路径，注意没有.pub后缀

```
Host 10.52.4.63
  HostName 10.52.4.63
  User cursorhu
  IdentityFile "C:\Users\thomas.hu\.ssh\id_rsa"
```

## 关闭宏代码块变暗

setting -> 搜索C_Cpp.dimInactiveRegions -> 关闭

## 代码跳转（Go Back和 Go Forword）

VSCode代码跳转（Go Back和 Go Forword）在Ubuntu 和Windows 不一样，如何将Ubuntu改成和windows一致：

setting -> 分别搜索Go Back和Go Forword -> 分别设置快捷键 alt + 方向 -> 提示和已有快捷键冲突，右键删除冲突的快捷键再添加

## 代码跳转（Definition和Reference）

安装C/C++ intelligence插件即可支持

### 查找文件

win: ctrl + p，输入文件名查找是否存在

## 关闭悬浮框（类型提示）

setting里面设置hover为false

```
"editor.hover.enabled": false,
"editor.parameterHints.enabled": false,
```

