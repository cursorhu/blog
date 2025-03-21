# Keil ARM配置笔记

## Keil community版本

https://www.keil.arm.com/mdk-community/

## Keil community安装ARM compiler v5

Keil community 5.37默认只包含ARM compile v6，编译v5的项目通常会报错（报错通常和v6要求C99相关），可以有两种方式解决：

（1）基于v6报错，更新代码，符合v6的规则要求

（2）安装v5 compiler，适配原项目

如何在Keil community >= 5.37版本上安装ARM compiler v5：

参考：https://community.arm.com/support-forums/f/keil-forum/52719/how-can-i-install-compiler-version-5-for-keil-vision-5

1.下载ARM compiler v5

https://developer.arm.com/documentation/ka005198/latest

登录ARM账户后，下载win32或者linux32版本

![image-20241223152607237](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202412231526350.png)

2.安装ARM compiler v5

注意：安装路径必须是Keil路径下的ARM文件夹

For example, if your Keil MDK installation is in `C:\Keil_v5` the recommended installation path is `C:\Keil_v5\ARM\ARM_Compiler_5.06u7`

如果不这样安装，Keil会找不到ARM compiler v5的license，编译会缺license报错；在Keil路径下安装会使用Keil已有的community license，不会报错。

3.配置Keil项目，添加v5选项

https://developer.arm.com/documentation/101407/0541/Creating-Applications/Tips-and-Tricks/Manage-Arm-Compiler-Versions

如何知道项目是基于ARM compiler v5还是v6创建：

查看项目文件uvprojx，如果是 “礦ision5 Project (.uvprojx)”则是基于ARM compiler v5

## 破解ARMCC(ARM compiler v5)

community的Keil license并不能支持ARMCC(ARM compiler v5)编译32K以上的项目，而一些老的严重依赖ARM v5项目移植到ARM v6不是很容易，还得用破解版ARMCC编译。

（1）下载ARM keygen2032注册机

（2）在File->License Management中deactive当前的community license，然后以管理员启动Keil，复制CID (Computer ID)到keygen，产生LIC

（3）禁用Keil联网：禁用UV4.exe出站规则 https://blog.csdn.net/weixin_41623723/article/details/105765273