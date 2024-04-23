---
title: 搭建高可移植的Markdown写作和博客环境：Obsidian+PicGo+Hexo+Github
date: 2022-02-28 15:04:18
tags: markdown
categories: markdown
---
## 0.背景
写技术笔记并发布博客，通常有以下方式：
- 第三方平台CSDN/cnblogs，最简单但是最不可控，例如我写过一篇如何使用shadow-sock，直接被删掉，各种广告也是技术洁癖症不能忍的。
- 使用云服务器自建Leanote博客系统，我用过几年，最大缺点是服务器续费太贵，且文章数据存在数据库很难导出，优点是自建的Leanote写作发布一体化体验极好。
- 使用github+picgo+hexo， github作为图床和博客的云服务，picgo用于建立图床通道，hexo用于发布博客。缺点是github访问慢，用开源CDN可以很好解决；优点是全部免费，数据可移植(图片链接都在github图床)，可长期使用（只要github不倒闭不锁区）

最终我选择github+picgo+hexo方案。
为什么不用gitee: gitee上传图片限制1M, github有25M。有了CDN, github的速度也不是问题

现在只需要确定Markdown编辑器

VSCode加Markdown插件试过，体验并不完美
[Typora](https://www.typora.io/)是很不错的选择，除了收费
[Obsidian](https://obsidian.md/)免费且优雅：
- 支持动态渲染，即写出的Markdown语句自动显示预览
- 支持各种快捷键，无需手动输入Markdown语法格式（[Markdown 基本语法](https://markdown.com.cn/basic-syntax/)）。比如标题，链接，列表，引用，可以设置标准的Markdown快捷键。
- 支持关联PicGo实现粘贴图片即上传到云端图床，这一点对于写作体验和文章的可迁移性很重要
- 特色的Zettelkasten笔记管理方法，本文不描述这部分，参考[玩转 Obsidian 01：打造知识循环利器](https://sspai.com/post/62414#!)

## 1. 搭建可移植的Markdown写作环境
Markdown文档本身是可移植的，但是其图片资源不是, 因为图片不是以二进制嵌入文档，而只是个图片地址的链接。

如果只是在本地写写Markdown，完全不需要考虑图片的可移植，只需要存本地固定路径即可。
如果想在本地写Markdown，且这个文档拿到别的机器，或者放到博客论坛发布，别人都可以看到图片，就要一个云端的图片存储服务，即图床。

本节讲如何实现一个“一处写作，到处可见”的Markdown写作环境，并优化工具设置，让写作顺滑流程

### 1.1 Github+PicGo搭建图床
完整流程参考[PicGo + GitHub 搭建个人图床工具](https://blog.csdn.net/yefcion/article/details/88412025)

流程包含以下部分：
- 一个public的Github仓库，用来存放文章链接的图片，注意单仓库最大容量1GB。
- 一个personal access token，用于PicGo免密访问github实现自动上传图片
- 一个图片上传工具：PicGo及其插件，用它上连Github图床仓库，下连本地markdown编辑器，实现“图片粘贴即上传”
- 一个CDN(推荐，很好用)：使用开源CDN jsdelivr加速Github访问，避免因访问速度造成图片上传失败

最终我的配置如下：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281014786.png)

单元测试：
手动上传本地图片，如果上传成功，且图片可预览，则功能正常
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281024870.png)

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281024344.png)

遇到的坑：
仓库名必须填<用户名><仓库名>，不是完整的http或git仓库链接！否则上传图片error404

PicGo支持log调试，参考：
[[PicGo上传图片到GitHub总是失败的特殊解决办法](https://www.shopee6.com/web/web-tutorial/picgo-github-fail.html)](https://www.shopee6.com/web/web-tutorial/picgo-github-fail.html)

2022/5/17更新：
最近发现jsdelivr在大陆挂了，那么只能放弃使用CDN, 直接使用github图床的raw链接.

将Picgo默认图床路径从jsdelivr的
https://cdn.jsdelivr.net/gh/账户名/图床仓库名@master
改为：
https://raw.githubusercontent.com/账户名/图床仓库名/master

注意master前是@还是反斜杠 ''/''.

对于已发布的博客的图床链接，直接用VSCode全局查找替换以上前缀即可，后面的图片id是不变的。

**如何删除Github图床的图片：**
PicGo默认的Github图床不能通过相册删除github仓库的图片，需要使用第三方插件github-plus，建议作为默认图床工具，支持相册删除同步到github仓库:
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281208516.png)

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281209075.png)


### 1.2 Obsidian链接图床
流程参考[# 在Obsidian中使用图床实现“一次上传、多次搬运”省心又省力](https://www.jianshu.com/p/4c30495f4325)

重点部分：配置自动上传插件。写Markdown文章时粘贴的图片自动上传到Github+PicGo图床，无需手动上传和写链接
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281029686.png)

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281029586.png)

可以看到，此插件相当于一个PicGo客户端，而本地运行的PicGo Server已开启端口36677的监听。
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281033479.png)

单元测试：
以上配置好后，Obsidian新建一个Markdown文档，粘贴图片会显示updating... ，成功后可在github图床仓库看到该图片，且Markdown本地预览也可以看到。

遇到的坑：
一开始上传的图片，github可预览，Obsidian预览失败
在PicGo客户端查看刚才上传图片预览也不正常，PicGo客户端手动上传一张后才正常预览。
此后Obsidian再粘贴上传图片，预览正常。
可能是PicGo首次链接Obsidian插件的bug。

### 1.3 优化Obsidian写作体验
#### 1.3.1 Markdown快捷键
很多Markdown编辑器支持各种快捷键，写作时不需要手动输入Markdown语法，例如一个ctrl+h形成标题，一个ctrl+k形成代码格式。
Obsidian默认快捷键不完整，需要补全优化
我之前用的Leanote写博客，Obsidian也快捷键设置如下。

> 提升标题 Ctrl + H (header)
> 有序列表 Ctrl + O (Ordered)
> 无序列表 Ctrl + U (Un-Ordered)
> 插入链接 Ctrl + L (Link)
> 插入代码 Ctrl + K (K = Code, 因为ctrl+C用于粘贴)
> 加粗 Ctrl + B (Bold)
> 斜体 Ctrl + I (Italian体)
> 引用 Ctrl + Q (Quote)
> 插入图片 Ctrl + G (Graph, 有了“粘贴即上传”，这个实际上用不上)

其他我基本不用的就没设置，例如表格，删除线，分割线等等。
如果经常用表格，推荐 Advanced Tables插件。
读者也可以参考VS Code的Markdown插件设置。

#### 1.3.2 实时预览和标题大纲
实时预览是你写一句Markdown，自动转换成预览格式，而不显示Markdown源码。这样不需要开两个窗口，一个写源码一个看预览。在设置->编辑器里开启。
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281103907.png)

标题大纲是侧边栏显示文章标题列表，就像word的导航窗口一样。在设置-核心插件开启。
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202202281104255.png)

### 1.4 typora设置自动上传

![](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202404181658086.png)



## 2. 搭建hexo博客发布环境
### 2.1 环境安装
整体流程参考：[# 个人博客第5篇——安装node.js和Hexo](https://zhuanlan.zhihu.com/p/105715224)
官方文档参考：[hexo.io](https://hexo.io/zh-cn/docs/)

简单总结一下：
- hexo是一个基于nodeJS的博客框架，nodeJS提供JavaScript实现后端服务的能力。
- hexo可以将本地写好的Markdown文本和图片资源，以JavaScript网页资源的方式打包。hexo可以在本机运行服务，通过浏览器可以在网页访问。
- 如果要使博客外网可访问，需要一个云服务存放hexo生成的博客，我们使用github仓库作为博客云端。前面已经搭建了github图床仓库，因此图片并不需要放到此hexo仓库。
- 使用git实现hexo博客部署到github仓库

遇到的坑：
如果你以前用过hexo，之后nodeJS升级了版本，一定不要用以前的hexo博客目录，必须新建博客目录，重新`hexo init`安装相关hexo插件。否则nodeJS和hexo插件可能不兼容导致部署失败。

hexo插件版本是根据nodeJS版本自动安装的，配置文件是package.json。以nodeJS 16.3为例，自动生成的依赖版本如下：

`{
  "name": "hexo-site",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "hexo generate",
    "clean": "hexo clean",
    "deploy": "hexo deploy",
    "server": "hexo server"
  },
  "hexo": {
    "version": "6.0.0"
  },
  "dependencies": {
    "hexo": "^6.0.0",
    "hexo-deployer-git": "^3.0.0",
    "hexo-generator-archive": "^1.0.0",
    "hexo-generator-category": "^1.0.0",
    "hexo-generator-index": "^2.0.0",
    "hexo-generator-tag": "^1.0.0",
    "hexo-renderer-ejs": "^2.0.0",
    "hexo-renderer-marked": "^5.0.0",
    "hexo-renderer-stylus": "^2.0.0",
    "hexo-server": "^3.0.0",
    "hexo-theme-landscape": "^0.0.3"
  }
}`

博客的github仓库，主题等关键配置都在`hexo init`对应目录下的`_config.yml`
我修改了几个关键配置如下：
博客框架的语言一定要配，否则默认德语阿拉伯语...
> title: ThinkNotes
   subtitle: Simple is not easy
   author: Cursorhu
   language: zh-CN

Github部署配置
> url: https://cursorhu.github.io
> deploy:
	  type: git
	  repo: https://github.com/cursorhu/cursorhu.github.io
	  branch: master


### 2.2 发布博客

hexo new命令可以从0新建markdown写博客，一般不用这么麻烦。

我们直接将Obsidian写好的Markdown（图片资源是图床链接）放到hexo博客目录的post文件夹，例如我的本地目录是：
> F:\HexoBlog\source\_posts

在Markdown文件前添加hexo博客特有的头，说明博客的标题时间，分类等属性。这个头如果用`hexo new`命令会自动生成，可以先new一个然后拷过来。

> title: 博客文章名
> date: 2022-02-28 15:04:18
> tags: markdown
> categories: markdown

最后使用git生成网页文件并发布到github博客仓库, 注意不是用node命令行！
在hexo博客目录下运行：
````text
hexo clean   #清除缓存文件 db.json 和已生成的静态文件 public
hexo g       #生成网站静态文件到默认设置的 public 文件夹(hexo generate 的缩写)
hexo d       #自动生成网站静态文件，并部署到设定的仓库(hexo deploy 的缩写)
````

### 2.3 博客主题

博客主题(theme)取决于不同人喜好，github有大量的hexo主题，按需配置
推荐几个主流的：
[Next](https://github.com/theme-next/hexo-theme-next)
[hexo-theme-material](https://github.com/bollnh/hexo-theme-material)

以Next主题为例，安装：
> $ cd hexo博客目录
   $ git clone https://github.com/theme-next/hexo-theme-next themes/next

已我的配置为例，就会在F:\HexoBlog\themes\next看到next主题的源码和配置文件config.yml

Next配置可以自定义各种设置，例如子主题:
>  scheme: Muse
#scheme: Mist
#scheme: Pisces
#scheme: Gemini

我们前面已经用jsdelivr开源CDN为PicGo上传图片加速，可以配置Next主题的网页静态文件的相关库也用jsdelivr加速。最好用到才开启，例如motion enable, 其依赖库velocity就可以设置jsdelivr加速：
> motion:
	  enable: true
    velocity: //cdn.jsdelivr.net/npm/velocity-animate@1/velocity.min.js
    velocity_ui: //cdn.jsdelivr.net/npm/velocity-animate@1/velocity.ui.min.js

注意这些配置文件也是Markdown格式，`# ` 注释一定要带空格

hexo要使用next, 在hexo配置文件设置:
> theme: next

hexo clean, hexo g, hexo s 跑本地服务，看一下效果：
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202203041651883.png)

最后hexo clean, g, d 命令三连部署到github

### 2.4 速度优化
参考[# Hexo 网站访问速度优化](https://ednovas.gitee.io/2021/09/23/speeduphexo/)
