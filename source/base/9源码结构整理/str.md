# 代码结构调整

**传说有人写的程序只有一个main.c，一万行代码，这是一个神奇的故事**

本节主要通过代码讲解如何模块化代码。

## 概述

代码结构调整有很多方式，今天只说最简单的。

1. 源码模块化----**接口标准化**
2. 硬件相关宏定义

## 源码模块化

模块化步奏：

1. 将一个功能、一个模块、一种设备的相关代码封装在同一个.c源文件中。

2. 内部使用的函数用static宏控制，不允许外部使用。
3. 内部的定义，比如宏、结构体，定义在c文件。
4. 这个源文件有一个相同名字的头文件。
5. 对外的定义，宏、结构体等，定义在头文件。
6. 变量只能定义在源文件，不对外直接暴露变量。

我们就拿上一节的代码整理。

数码管，就是一个设备。这个设备提供的功能是什么呢？点亮对应的段？显示数字和小数点？

我认为：**点亮对应的段，才是八段数码管的根本功能**。

显示数字属于应用层功能。

> 为什么这样划分呢？有以下原因：
>
> 设备驱动尽量只提供自己能实现的功能本质，数码管的功能本质就是每个段点亮。
>
> 不同的段点亮后，组成的到底是数字还是字母，最好不要放在设备驱动中。
>
> 当项目越大，程序越复杂，参与开发的工程师多时，越能感觉到这样划分的合理性。
>
> 假如你数码管驱动只提供显示数字的接口，但是新项目需要显示一些自定义的字符，
>
> 那到底是改驱动还是改应用呢？
>
> 不过在实践上，数码管是一个小驱动，将显示数字接口放在设备驱动中，也并不是不可。
>
> 但设备较复杂，例如LCD，可以在驱动和应用间封装一层pannel层，用于实现各种应用需要的接口。

* 建立一个dev目录

  在dev目录下建立两个文件：seg.c、seg.h

  把`seg_init`, `seg_display_1seg`、`seg_select`、`seg_clear`、`seg_display_task`、`seg_fill_disbuf`这几个函数拷贝到seg.c文件。

  同时拷贝`BufIndex`、`Seg8DisBuf`这两个变量。

* 对外的函数是：`seg_display_task`、`seg_fill_disbuf`、`seg_init`，这三个函数拷声明贝到头文件seg.h，并且加上extern前缀。如下：

  ```
  #ifndef __SEG_H__
  #define __SEG_H__
  
  extern void seg_init(void);
  extern void seg_display_task(void );
  extern void seg_fill_disbuf(uint8_t segbit, uint8_t seg);
  
  
  #endif
  ```

  其中`#ifndef`等三个宏定义是为了解决重复包含的问题。每个头文件都会有这三行指令，后面的宏不一样而已。

* 为了防止外部函数调用内部函数，对没有在头文件声明的函数加上`static`，在外部调用此函数时，编译会出错。

* 把调试过程的函数剪切到seg.c，定义一个函数seg_test。

* 把变量`BufIndex`、`Seg8DisBuf`也加上static前缀，seg.c文件外部就不能直接操作这两个变量了。

* 在main.c头部增加`#include "seg.h"`，包含seg驱动的头文件。

* 打开MDK工程，在工程目录新建一个目录，并添加seg.c到目录，并修改工程头文件路径。

* 重新编译下载。

## 宏定义

宏定义的第一个好处：将某个定义在一个地方定义，后续要修改这个定义的时候，只要改一个地方即可。

在数码管的驱动中，硬件IO口的定义在`seg_init`、`seg_display_1seg`、`seg_select`、`seg_clear`四个函数中都有使用，如果我们要修改某个IO的硬件连接，可能就需要修改这四个函数。

如果我们将这些函数中硬件相关的定义统一到一个宏定义，只要修改一个地方，就能改变整个数码管驱动的硬件连接。

```
#define SEG_138_A0_PIN	GPIO_Pin_0
#define SEG_138_A1_PIN	GPIO_Pin_15
#define SEG_138_A2_PIN	GPIO_Pin_14
#define SEG_138_A_PORT  GPIOD

#define SEG_595_SDI_PIN	  GPIO_Pin_0
#define SEG_595_SDI_PORT  GPIOB

#define SEG_595_LCLK_PIN	GPIO_Pin_1
#define SEG_595_LCLK_PORT   GPIOB

#define SEG_595_SCLK_PIN	GPIO_Pin_5
#define SEG_595_SCLK_PORT   GPIOC

#define SEG_595_RST_PIN		GPIO_Pin_4
#define SEG_595_RST_PORT   GPIOC
```

宏定义如上，相关函数修改为宏即可，具体见代码。

编译下载验证

## 推荐

《林锐 高质量C-C++编程指南》

《嵌入式C精华》

---

end

20210819