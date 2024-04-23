---
title: Qt学习笔记：RGB调色器
date: 2022-04-18 11:49:00
tags: Qt
categories: Qt
---
本文基于Qt官方示例[ A Quick Start to Qt Designer](https://doc.qt.io/qt-5/designer-quick-start.html#:~:text=%20Using%20Qt%20Designer%20involves%20four%20basic%20steps%3A,the%20slots%204%20Preview%20the%20form%20More%20), 实现自定义的slot函数，新增RGB色彩窗口显示色彩。
- 本文源码：[QtSampleTest/1.rgbSlider](https://github.com/cursorhu/QtSampleTest/tree/master/1.rgbSlider)
- 环境：基于Qt5.9 + Qt creater

本文只记录项目过程中的注意事项，以及增量开发，其他部分参考Qt官方示例。

## 1.UI部分
- 建立带UI的项目rgbSlider, 基于Qwidget生成默认自定义类名widget
- 双击widget.ui进入UI编辑

UI 编辑模式下使用两种模式：widget编辑模式， slot/signal编辑模式

1. widget编辑模式如下：使用水平、网格布局
RGB数值控制部分，使用Label,  spinBox和scrollBar三种控件，按先竖直，后水平排列
RGB颜色显示部分，使用 graphicsView窗口
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202204181201206.png)
注意调整布局的比例需要先选中，然后在layout属性调整
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202204181202783.png)

2.  slot/signal编辑模式
直接拖拽起始控件和目标控件，设置控件的信号和槽
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202204181203784.png)

## 2.自定义槽
graphicsView窗口预期效果是：只要调整RGB数值，自动显示对应的颜色
UI界面不能设置控件信号触发自定义槽，需要在代码中实现信号和槽的连接。

1. 右键转到graphicsView窗口的槽函数，自定义为 `Widget::on_rgbChanged()`
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202204181400431.png)
函数实现如下：
```

#include <QColor>

#include <QPalette>


void Widget::on_rgbChanged()

{

 QPalette pal = QPalette();

 QColor color;

 //分别设置R,G,B,透明度

 color.setRgb(ui->spinBoxRed->value(), ui->spinBoxGreen->value(), ui->spinBoxBlue->value(), 255);

 //QPalette::Base

 //Used mostly as the background color for text entry widgets, It is usually white or another light color.

 pal.setColor(QPalette::Base, color);

 ui->graphicsView->setPalette(pal);

}
```

在UI基础上使用控件对象的方法，只需要：
```
ui->控件名->控件的方法
```

注意`setColor`可以给不同图层上色，这里使用`QPalette::Base`，而不能是`QPalette::Window`或`QPalette::Background`

代码设置信号与槽, 注意，手动设置的代码要在`ui->setupUi(this);`的后面添加：
```

Widget::Widget(QWidget *parent) :

 QWidget(parent),

 ui(new Ui::Widget)

{

 ui->setupUi(this);

 connect(ui->spinBoxRed, SIGNAL(valueChanged(int)), this, SLOT(on_rgbChanged()));

 connect(ui->spinBoxGreen, SIGNAL(valueChanged(int)), this, SLOT(on_rgbChanged()));

 connect(ui->spinBoxBlue, SIGNAL(valueChanged(int)), this, SLOT(on_rgbChanged()));

}
```

## 3.测试效果
- 拖动滑块，对应数值会更新，颜色同步更新
- 修改数值，对应滑块更新，颜色更新
![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202204181410181.png)
