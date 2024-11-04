# 如何使用CubeMX生成STM32代码

## 示例：

STM32F0的firmware工程文件.ioc用CubeMX打开，如下：

![image-20240529202815926](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292028057.png)

现在要移植一些外设配置到STM32F4，使用UART2为示例：

1.对照F0的工程界面，打开F4的工程界面，搜索要配置的Pin，设置Pin模式和F0一致，为UART2_TX

![image-20240529203013390](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292030441.png)2.对照F0的工程界面，打开F4的工程界面，配置UART2_TX的详细工作模式，例如波特率，中断...

![image-20240529203142664](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292031748.png)

3.配置完毕后，点generate code生成代码

4.理解代码和工程配置的对应关系：

一般自动生成的代码都在Core里面：

（1）main查看该外设的初始化代码

![image-20240529203258870](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292032897.png)

（2）中断查看中断配置相关的代码：it.c实现回调

![image-20240529203407652](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292034688.png)

（2）msp是该外设的最底层配置，main的初始化和中断回调都以此配置为前提生效。

![image-20240529203455499](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202405292034549.png)