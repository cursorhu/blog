---
title: Office常用操作笔记
date: 2019-07-06 14:19:23
tags: office
categories: Office
---

# Word

## Word设置自动推导的标题列表

word自动标题列表的是写文档必不可少的，自动标题能自动推导更新各级标题的序号，增删改查任何标题都不需要手动的写标题序号。

（1）创建多级列表
![image-20221206142307899](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061423943.png)

（2）设置一级标题

- 设置一级标题的序号样式为1,2,3
- 链接一级标题的字体样式到word文档的一级标题字体样式
![image-20221206142315070](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061423121.png)

（2）设置二级、三级、n级标题

需要设置列表序号，标题字体两部分。注意列表序号的正确设置是序号自动推导的关键。
以二级标题为例，其他子级类推。

- 设置二级标题中的一级序号来自于level1。这一步保证二级标题中的一级序号是自动推导的。
![image-20221206142327739](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061423795.png)

- 设置二级标题中的二级序号的样式，二级标题中的一、二级序号用.号隔开
![image-20221206142401132](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061424185.png)

- 二级标题最终的序号样式如下
![image-20221206142411436](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061424492.png)

- 然后设置二级标题的字体风格，直接链接到word的二级标题字体风格
![image-20221206142420685](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061424730.png)

- 二级标题列表的所有设置完毕，如下
![image-20221206142428828](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202212061424880.png)

(3)依次完成所有标题列表设置，例如三级标题，前两级的值来自于level1,level2，第三级设置数字格式即可，中间用.号隔开。完成以后各级标题就可以自动推导。

## Word导出原图

Word默认图片如果直接复制出来，不是原图是压缩后的图。 
保存原图方法： 文档另存为html网页格式，会把word文档转换成资源文件夹，里面有原始图片。



# Excel

## excel设置筛选

参考[筛选区域或表中的数据](https://support.microsoft.com/zh-cn/office/%E7%AD%9B%E9%80%89%E5%8C%BA%E5%9F%9F%E6%88%96%E8%A1%A8%E4%B8%AD%E7%9A%84%E6%95%B0%E6%8D%AE-01832226-31b5-4568-8806-38c37dcc180e#:~:text=%E5%AF%B9%E8%A1%A8%E4%B8%AD%E7%9A%84%E6%95%B0%E6%8D%AE%E8%BF%9B%E8%A1%8C%E7%AD%9B%E9%80%89%201%20%E9%80%89%E6%8B%A9%E8%A6%81%E7%AD%9B%E9%80%89%20%E5%88%97%E7%9A%84%E5%88%97%E6%A0%87%E9%A2%98%E7%AE%AD%E5%A4%B4%E3%80%82%202%20%E5%8F%96%E6%B6%88%20%28%E9%80%89%E6%8B%A9%22%29%20%22%EF%BC%8C,%E5%8D%95%E5%87%BB%E2%80%9C%20%E7%A1%AE%E5%AE%9A%20%E2%80%9D%E3%80%82%20%E5%88%97%E6%A0%87%E9%A2%98%E7%AE%AD%E5%A4%B4%20%22%E7%AD%9B%E9%80%89%20%E6%9B%B4%E6%94%B9%20%E3%80%82%20%E9%80%89%E6%8B%A9%E6%AD%A4%E5%9B%BE%E6%A0%87%E5%8F%AF%E6%9B%B4%E6%94%B9%E6%88%96%E6%B8%85%E9%99%A4%E7%AD%9B%E9%80%89%E3%80%82)

1. 选中列，点击筛选

   ![image-20230209114432580](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202302091144706.png)

2. 可以按文字或者颜色筛选

   ![image-20230209114508910](https://cdn.jsdelivr.net/gh/cursorhu/blog-images-on-picgo@master/images/202302091145967.png)

## 冻结首行

筛选行一般需要固定显示，因此设置冻结首行：

选中要固定显示行的下一行, 视图 -> 冻结窗格 -> 冻结首行
