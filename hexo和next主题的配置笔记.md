---
title: hexo和next主题的配置笔记
date: 2023-03-15 12:00:15
tags: hexo
categories: hexo
---

### hexo相关配置

hexo各页面的配置，参考 [jianshu-Hexo的Next主题详细配置](https://www.jianshu.com/p/3a05351a37dc)

hexo主页显示摘要，参考 [Hexo Next主题首页配置为只显示部分摘要](https://tohugo.com/2021/01/26/%E5%B7%A5%E5%85%B7%E9%85%8D%E7%BD%AE/Hexo%20Next%E4%B8%BB%E9%A2%98%E9%A6%96%E9%A1%B5%E5%8F%AA%E6%98%BE%E7%A4%BA%E9%83%A8%E5%88%86%E6%91%98%E8%A6%81%EF%BC%88%E4%B8%8D%E6%98%BE%E7%A4%BA%E5%85%A8%E6%96%87%EF%BC%89/)

### next设置字体

参考 [tzynwang.github.io/2021/next-theme-edit](https://tzynwang.github.io/2021/next-theme-edit/#:~:text=Search%20for%20the%20font%20family%20%E2%80%9CRoboto%E2%80%9D%20Click%20%E2%80%9C%2B,as%20the%20value%20for%20%E2%80%9Chost%E2%80%9D%20key%20in%20_config.next.yml)

下面重点描述如何使用Google Font来配置next主题的字体，基于next version 8.0.0

- 推荐英文字体使用Roboto，中文字体使用 Noto Serif (注：Noto Serif字符集包含chinese/Japanese/korea等，参考 [noto-cjk](https://github.com/notofonts/noto-cjk)；Noto Serif 是宋体但不是宋体思源，见后文)
- 在[Google字体中国网站](https://www.googlefonts.cn/)搜索框搜索字体英文名添加以上两种字体，产生URI(Uniform Resource Identifier)，复制href字段的引号内容

![image-20230315145422134](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303151454240.png)

- 在hexo的next配置文件`hexo\themes\next\_config.yml`的font字段添加host URI和字体名

```
font:
  enable: true

  # Uri of fonts host, e.g. https://fonts.googleapis.com (Default).
  host: https://fonts.googlefonts.cn/css?family=Noto+Serif|Roboto

  # Font options:
  # `external: true` will load this font family from `host` above.
  # `family: Times New Roman`. Without any quotes.
  # `size: x.x`. Use `em` as unit. Default: 1 (16px)

  # Global font settings used for all elements inside <body>.
  global:
    external: true
    family: Noto Serif
    size:

  # Font settings for site title (.site-title).
  title:
    external: true
    family:
    size:

  # Font settings for headlines (<h1> to <h6>).
  headings:
    external: true
    family:
    size:

  # Font settings for posts (.post-body).
  posts:
    external: true
    family:

  # Font settings for <code> and code blocks.
  codes:
    external: true
    family: Roboto
```

- 在静态页面的base style配置文件`hexo\themes\next\source\css\_variables\base.styl`指定中文字体font-family-chinese为'Noto Serif'（注意看这里get_font_family解析到next配置文件_config.yml的字段'global', 'title' ... 'codes'等作为静态页面的配置）

```
// Font families.
$font-family-chinese      = 'Noto Serif';

$font-family-base         = $font-family-chinese, sans-serif;
$font-family-base         = get_font_family('global'), $font-family-chinese, sans-serif if get_font_family('global');

$font-family-logo         = $font-family-base;
$font-family-logo         = get_font_family('title'), $font-family-base if get_font_family('title');

$font-family-headings     = $font-family-base;
$font-family-headings     = get_font_family('headings'), $font-family-base if get_font_family('headings');

$font-family-posts        = $font-family-base;
$font-family-posts        = get_font_family('posts'), $font-family-base if get_font_family('posts');

$font-family-monospace    = monospace, consolas, Menlo, $font-family-chinese;
$font-family-monospace    = get_font_family('codes'), monospace, consolas, Menlo, $font-family-chinese if get_font_family('codes');
```

自此next中英文字体都应该生效，`hexo g + hexo s` 重新部署验证一下发现中文字体似乎不是思源宋体？

![image-20230315160648032](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303151606100.png)

原因是Noto Serif != Noto Serif SC (simplified chinese)，Noto Serif SC才是思源宋体

[Google字体中国网站](https://www.googlefonts.cn/)搜索不到思源宋体，[google font原站](https://fonts.google.com/)又打不开，因此需要直接替换URI，将fonts.googlefonts.cn替换为fonts.googleapis.com，Noto Serif替换为Noto Serif SC

next配置文件改动如下：

```
hexo\themes\next\_config.yml:

font:
    - host: https://fonts.googlefonts.cn/css?family=Noto+Serif|Roboto
    + host: https://fonts.googleapis.com/css?family=Noto+Serif+SC|Roboto

    global:
    - family: Noto Serif
    + family: Noto Serif SC
    
hexo\themes\next\source\css\_variables\base.styl:

// Font families.
- $font-family-chinese = 'Noto Serif';
+ $font-family-chinese = 'Noto Serif SC';
```

验证结果为思源宋体：

![image-20230315155911647](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202303151559719.png)
