# 程序各种要素说明

这节课我们用一个最简单的程序跟大家讲清楚程序的构成。（请看视频）

### 概述

* 硬件

首先要知道硬件的组成。

在前面章节我们说过，芯片包含Flash和RAM。

他们虽然不是一样的东西，但是都属于同一个地址空间，32位芯片的地址空间大小是4G。

比如ST32，FLASH通常从0X8000000开始，而RAM就从0x20000000开始。

高级点的芯片，可能会有外部SDRAM，内核也会为这SDRAM分配一段地址。

* 程序

程序包含什么？

写代码的时候包含函数过程和变量。

编译得到的目标文件包含函数过程和变量的初始化值。

* 变量

变量有很多种：全局变量，局部变量、静态变量。。。

变量保存在哪里？

下面我们就从一个简单的程序来分析上面问题。

### 包罗万象的小程序

#### 程序入口

程序入口，程序启动执行的第一条代码就叫程序入口。

或者说，芯片上电开始执行的第一条代码。

这条代码在哪？

我们写代码，通常都是从main函数开始写，我们也会把main函数叫做函数入口。

那么main函数是芯片复位的第一条代码吗？

实际不是，在执行main函数之前，已经执行了很多代码了。

其中最早执行的，也就是芯片复位的第一条代码，就是我们经常说的启动代码。

在我们的STM32工程中，启动代码就是startup_stm32f10x_hd.s。

这时一个汇编文件。现在我们一起来看看这个启动代码。这个文件是一个汇编文件。

```
; Vector Table Mapped to Address 0 at Reset
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size

__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                DCD     MemManage_Handler          ; MPU Fault Handler
                DCD     BusFault_Handler           ; Bus Fault Handler
                DCD     UsageFault_Handler         ; Usage Fault Handler
                DCD     0                          ; Reserved
```

这就是启动代码的入口。但是这里放的并不是代码，而是函数指针，这些函数指针就是中断向量。

DCD的意思是分配一个空间来保存后面的值。

__Vectors是一个标号，等下在分散加载文件中再说。

现在我们只要知道这里保存的是中断向量，并且，复位也是一个中断。

当芯片复位时，芯片从这里找到对应的函数指针Reset_Handler，然后调到这个函数执行。

这个函数同样在启动文件中，如下：

```
; Reset handler
Reset_Handler   PROC
                EXPORT  Reset_Handler             [WEAK]
                IMPORT  __main
                IMPORT  SystemInit
                LDR     R0, =SystemInit
                BLX     R0               
                LDR     R0, =__main
                BX      R0
                ENDP
```

复位后芯片做了什么呢？

1. 调用SystemInit函数。
2. 调用__main。

> SystemInit在system_stm32f10x.c文件中，这个函数完成芯片的时钟配置。
>
> __main函数在哪呢？在工程中找不到的。是不是main？不是。
>
> 这是一个编译系统根据不同芯片生成的一个库函数。
>
> 在这个库函数中完成变量（RAM）的初始化。居然后跳到真正的main函数执行。

#### 函数

int main(void)是我们接触的第一个函数。

函数的定义包含名称、参数、返回值。

我们可以定义一些子函数。

#### 变量

* 全局变量

在函数外定义的叫做局部变量。

比如main函数中，SegTab就是一个全局变量，这个变量的类型是一个uint16_t数组。

```
/*
	定义一个全局数组SegTab，数组成员类型是uint16_t
	并初始化数组。
	这个数组是数码管显示0-9的段定义。
	请看seg_display函数，
	例如，第一个值是0x3f00
	在seg_display，取这个数，输出到IO口，LED就能显示0。
*/
uint16_t SegTab[10]={0x3f00, 0x0600, 0x5b00, 0x4f00, 0x6600, 0x6d00, 0x7d00, 0x0700, 0x7f00, 0x6700};
```

变量是保存在RAM中的，我们都知道RAM是易失性存储，掉电数据就没了，那数组的些值是如何赋值给数组的呢？

这问题有两个方面：

1. 编译的时候，这些值会保存在代码中。同时还保存这些值和变量的关系。（细节暂时不研究）
2. 在启动代码中，执行__main函数后，会根据这些关系执行初始化变量的过程。然后才执行main函数。
3. 这个过程就是编译器生成的，如果你用一些很便宜的单片机，比如台湾的一下小单片机，这个过程就需要自己写代码实现。通常是用汇编写。

* 局部变量

在函数内定义的变量就是局部变量，例如seg_display函数中的tmp就是一个局部变量。

 ```
/*
	定义一个seg_display
	输入参数有2个，分别是char型的num，char型的dot
	没有返回值。
*/
void seg_display(char num, char dot)
{
		uint16_t tmp;
 ```

局部变量同样也是在RAM上。但是具体在哪呢？地址是哪里？

局部变量的地址是不固定的。当调用函数时，从栈上分配。函数退出后就释放了。

* 变量有效域

  局部变量只在函数中有效。

  全局变量呢？

  这个起始不是芯片的专有知识，是C语言的知识。

  EXTERN如何使用？

### 分散加载文件

为什么启动代码就是上电执行的第一条指令呢？

因为我们用分散加载文件（链接文件）指定启动代码保存在芯片复位时指向的位置。

分析分散加载文件。

### 编译结果如何看？





### 为什么能控制外设？





---

end

2020-03-12

