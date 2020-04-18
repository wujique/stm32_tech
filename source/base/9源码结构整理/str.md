# 代码结构调整

**传说有人写的程序只有一个main.c，一万行，这是一个神奇的故事**

## 概述

代码结构调整有很多方式，我们这里只说最简单的。

1. 源码模块化----接口标准化
2. 硬件相关宏定义

## 源码模块化

操作：

1. 将一个功能、一个模块、一种设备的相关代码封装在同一个.c源文件中。

2. 内部使用的函数用static控制，不允许外部使用。
3. 内部的定义，比如宏、结构体。定义在c文件。
4. 这个源文件有一个相同名字的头文件。
5. 对外的定义，宏、结构体等，定义在头文件。
6. 变量只能定义在源文件，不对外暴露。

我们拿上一节的代码整理。

数码管，就是一个设备。

这个设备提供的功能是什么呢？

点亮对应的段？显示数字和小数点？

我认为点亮对应的段，才是八段数码管的根本功能。

显示数字属于应用层功能。

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

  其中ifndef等三个宏定义是为了解决重复包含的问题。每个头文件都会有这三行指令，后面的宏不一样而已。

* 为了防止外部函数调用内部函数，对没有在头文件声明的函数加上`static`，这样声明的函数当在外部调用时，编译会出错。

* 把调试过程的函数剪切到seg.c，定义一个函数seg_test

* 把变量`BufIndex`、`Seg8DisBuf`也加上static前缀，seg.c文件外部就不能直接操作这两个变量了。

* 在main.c头部增加`#include "seg.h"`，包含seg驱动的头文件。

* 打开MDK工程，在工程目录新建一个目录，并添加seg.c到目录，并修改工程头文件路径。

* 重新编译下载。

## 宏定义

宏定义的第一个好处：将某个定义在一个地方定义，后续要修改这个定义的时候，只要改一个地方即可。

在数码管的驱动中，硬件IO口的定义在`seg_init`、`seg_display_1seg`、`seg_select`、`seg_clear`，如果我们要修改某个IO的硬件连接，可能就需要修改这3个函数。

如果我们将这些函数中硬件相关的函数统一到一个宏定义，只要修改一个地方，就能改变整个数码管驱动的硬件连接。

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



---

end

20200418