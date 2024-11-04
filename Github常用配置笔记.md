---
title: Github常用配置笔记
date: 2020-08-27 10:57:00
tags: Git
categories: Git
---

# Github常用配置笔记
## 配置SSH登录（Github，Gitlab等各种git server通用）

配置SSH登录的目的是git操作免密码验证，方便拉取和上传代码。

参考：https://segmentfault.com/a/1190000043924833

下面是全局使用唯一的git账号和SSH key；若要为个人和公司使用不同git账号，参考链接。

```
cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ git config --global user.name thomas.hu

cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ git config --global user.email thomas.hu@o2micro.com

cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/cursorhu/.ssh/id_rsa):
/c/Users/cursorhu/.ssh/id_rsa already exists.
Overwrite (y/n)?

cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ cat /c/Users/cursorhu/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
....
2hdYrBeOK+vu1LAAAACGN1cnNvcmh1AQI=
-----END OPENSSH PRIVATE KEY-----

cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ cat /c/Users/cursorhu/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDONvU2p10NVjRhu6UGlEMsRWqhbo16zK2Tnqg8....chI60tVZHozCK9PMKZd4dE9RoYMXpJWTo6uIRKEV41qHfaiipfsu1ibRCj1drz/3BTs= cursorhu
```

最后将id_rsa.pub公钥内容添加到github或者gitlab或其他git server的账号设置页面中，无需登录即可git clone，git push。

```
cursorhu@DESKTOP-73G2O3N MINGW64 /c/gitlab-bht
$ git clone git@10.52.1.103:software/storport.git
Cloning into 'storport'...
remote: Enumerating objects: 12830, done.
remote: Counting objects: 100% (12830/12830), done.
remote: Compressing objects: 100% (3441/3441), done.
remote: Total 12830 (delta 9621), reused 12368 (delta 9251)
Receiving objects: 100% (12830/12830), 259.92 MiB | 52.21 MiB/s, done.
Resolving deltas: 100% (9621/9621), done.
```



## 初始化git和github仓库

1.安装git

2.进入本地源码目录

    git init

会出现.git目录
首次需要配置github账户和邮箱

    git config --global user.name "github注册的用户名"
    git config --global user.mail "github注册的邮箱"

3.添加远程仓库

在github网页新建仓库

    git remote add origin git@github.com:github用户名/仓库名.git

.git/config文件内容会出现remote等内容，ssh方式的url是git开头，http(s)方式是http(s)开头
![image-20221205111653141](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051116187.png)
如果是从别人拉过来的仓库，修改后新建仓库，上传遇到`fatal: remote origin already exists`问题，解决方法:

    git remote rm origin
    git remote add origin git@github.com:github用户名/仓库名.git

4.git add, commit, push三连

    git add -A
    git commit -m 'first commit'
    git push -f --set-upstream origin master //首次提交

![image-20221205111703940](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051117987.png)
完成以后远程可以看得到仓库的文件   

5.创建分支

如果已经有主线，在本地`git checkout branchname`, 远程创建分支，记录.git链接， 然后关联远程分支即可：

    git remote add origin https://github.com/*/*.git

然后推送

    git push origin branchname

## 首次配置可能的问题：
### push时有RSA key错误

![image-20221205111713234](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051117281.png)
因为Git使用SSH连接，而SSH第一次连接需要验证GitHub服务器的Key。确认GitHub的Key的指纹信息是否真的来自GitHub的服务器。解决办法是在本地生成key，配置到github服务器
（1)创建ssh key

    ls -al ~/.ssh
    ssh-keygen -t rsa -C "github用户名"
    cat ~/.ssh/id_rsa.pub
![image-20221205111721197](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051117255.png)
在push三连过程可以设置global全局配置，以后默认push到github
![image-20221205111728996](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051117055.png)

（2）配置ssh key到github
登陆github,头像-settings-new SSH,复制新生成的SSH配置到服务器
![image-20221205111737339](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202212051117386.png)
（3）需要重新add origin新建仓库（或者网页上新建仓库)，再push，`git status`和`git log`查看分支和日志

### push时不能使用密码登陆

```
Support for password authentication was removed on August 13, 2021.
```

使用personal token替代密码登陆：

Github setting -> Developer setting -> Personal access token -> Generate a New Token (classic) -> 设置不过时，所有权限勾上 -> 首次会显示token字符，下次不会显示，记得备份token！-> 再次输入账号密码时用token代替密码即可push

### git clone有HTTP2错误

错误码：RPC failed; curl 16 Error in the HTTP2 framing layer

解决办法：Git使用HTTP1.1

```
git config --global http.version HTTP/1.1
```

## Github clone使用国内镜像

国内搞开发最痛苦的就是限速+断开连接，github clone经常失败。推荐国内镜像服务作为代理进行git clone，将原git地址的github.com替换成代理地址即可。参考 [无需代理直接加速各种 GitHub 资源拉取](https://zhuanlan.zhihu.com/p/463954956)

```
#git clone原地址
$ git clone https://github.com/kubernetes/kubernetes.git

#手动配置代理地址，任选其一能clone成功即可
$ git clone https://github.com.cnpmjs.org/kubernetes/kubernetes.git
$ git clone https://hub.fastgit.org/kubernetes/kubernetes.git
$ git clone https://gitclone.com/github.com/kubernetes/kubernetes.git

#配置git自动使用代理，配置以后可以用git clone原地址，自动走代理
git config --global url."https://hub.fastgit.org".insteadOf https://github.com
#取消自动代理
$ git config --global --unset url.https://github.com/.insteadof
```

使用国内镜像并不一定能解决所有clone问题，有的recursive clone对依赖包有版本要求，国内镜像版本不匹配导致clone fail，此时不能使用国内镜像。

解决版本：下载release版本的zip包，绕开git clone操作。

## Github连接报错问题 

### OpenSSL errno 10054

一劳永逸的解决办法：git bash -> git config --global http.sslVerify "false"

参考：[OpenSSL SSL_read: Connection was reset, errno 10054](https://blog.csdn.net/qq_29493173/article/details/114534057)