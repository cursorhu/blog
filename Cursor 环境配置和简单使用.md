# Cursor 环境配置和简单使用

## Cursor的free trial无限次数

Cursor website: https://www.cursor.com

Cursor's free trial for Pro(Premium models) has limitation for two weeks time and up to 500 request count. Use below steps to get more free-trial.

(1) In Account Setting, Detele wasted account, and re-register account with same email address. Log in Cursor desktop and 500 pro request is supported for more two weeks.

![image-20250226145359068](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502261454158.png)

refer to guide: https://www.bilibili.com/video/BV1YAtReqEkH?spm_id_from=333.788.videopod.sections

(2) After several times re-register account, Cursor record your PC's MAC and refuse to re-register, use below tool to re-fresh MAC. 

https://github.com/yuaotian/go-cursor-help

## Cursor使用方式

### Chat

Chat is most used to analysis whole project code or partly code.

Use @codebase is better to make the AI engine think based on current code. otherwise it may give suggestions without context.

below is two examples:

(1)Analysis the project based on codebase.

Use CTRL+L to open chat, select AI model to claude-3.5 or 3.7(newer), write prompt based on codebase.

![image-20250226150407616](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502261504728.png)

(2)Modifying partly function

Select some code to Chat, and ask what to do, you can apply the generated code. 

If the code is wrong after test, you can reject the change like git diff GUI.

![image-20250226150927338](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502261509413.png)

### Composer

Composer适用于创建新项目，不适用于对已有的项目局部修改，因为composer改动是跨文件的，composer模式经常会把已有项目写好的功能又改坏，建议多用git管理composer代码，调试现有项目使用Chat。

以下是composer创建WDF项目过程中的示例，对于AI给出的不靠谱的回答，需要用 @Web或者给定Specification document让AI反复确认。对于编译和调试问题也能让AI给出建议，但是尤其要注意AI对于windows driver这种开源资料较少的领域，给出的方案和建议有一定概率是错得离谱而且迷惑性很强。

（1）让AI根据Web资料修正设计

![image-20250226152037888](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502261520937.png)

（2）让AI根据报错信息给出调试建议

![image-20250226152110538](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202502261521572.png)

