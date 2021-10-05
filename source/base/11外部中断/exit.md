# 外部中断与按键触发

## 概述

* 中断是啥？

> 中断是指芯片运行过程中，发生特殊事件，需芯片干预时，芯片自动停止正在运行的程序，并转入处理刚刚中断发生的事件，处理完后返回被打断暂停的程序继续运行。

在前面的程序中，程序没有使用中断，执行流程就只有main函数中的while(1)循环执行。

比如前面一章节按键扫描：

```
 while(1) {
	 	/*-----------------驱动--------------*/
		/* 使用显示缓冲方法，要改变显示内容，
		调用函数seg_fill_disbuf改变Seg8DisBuf中的内容即可 */
		seg_display_task();

		/*-----------------应用-----------------*/
		new_sta = GPIO_ReadInputData(GPIOB);	
		new_sta = new_sta&0xf000;

		......
		
		/*----------------------------------*/
		delay(1000);
	 }
```

> 这就是程序的全部流程，驱动和应用都放在这个流程中，循环执行。

* 为什么要中断？

在按键的while循环中，不断查询IO口的状态，判断是否有按键按下。有按键按下时，就进行去抖动。

这里有两个事情是要记住的：

1. **循环查询，速度再快，也是有间隔的。**
2. 连续不断查询，是要使用CPU时间片的。

那么，查询的越频繁，需要的CPU时间越多，那就不能去干其他事情了。

查询的不频繁呢？间隔就大了，响应的速度就不及时了。

现在我们拿一个**按键输入**和你在家**等快递**对比。

按键按下，就相当于快递敲门。

在while循环中查询IO口，就相当于你想知道有没有人敲门，跑到门口看看有没有人。

如果你频繁的去门口查看是否有人来，甚至一直坐在门口等，你就没时间干其他事情。

如果你一个小时才去看一次，那很可能就会让快递员等一个小时，或者，快递员都不等你，没人开门就走了。**这样，你就会丢掉一个快递。**

**如果使用中断呢**

相当于在门口装个门铃，你在家里做其他事情，快递来了，按下门铃，然后你出去开门，拿了快递后，回家接着做刚才做的事。

## STM32中断管理

我们常说某某芯片的中断管理，就像本节标题。

实际上，中断是内核在管理。所以，要搞懂芯片的中断，需要学习两部分知识。

1. 内核如何管理中断。（中断响应）
2. 芯片中断如何连接到内核。

#### NVIC

NVIC的全称是Nested vectoredinterrupt controller，即嵌套向量中断控制器。

NVIC属于内核的一部分，这东西怎么实现的我们不需要管。只需要知道：

1. 要在NVIC中使能对应中断，才会产生中断
2. 通过NVIC可以配置中断的优先级，优先级有两种：响应优先级和嵌套优先级。

![][1]

内核还决定了中断向量表。

> 什么是中断向量？这个问题在知乎有一个讨论。
>
> 我的观点是何必浪费时间。

#### 中断向量表

在`startup_stm32f10x_hd.`s文件中，有以下这段代码:

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
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     0                          ; Reserved
                DCD     SVC_Handler                ; SVCall Handler
                DCD     DebugMon_Handler           ; Debug Monitor Handler
                DCD     0                          ; Reserved
                DCD     PendSV_Handler             ; PendSV Handler
                DCD     SysTick_Handler            ; SysTick Handler

                ; External Interrupts
                DCD     WWDG_IRQHandler            ; Window Watchdog
                DCD     PVD_IRQHandler             ; PVD through EXTI Line detect
                DCD     TAMPER_IRQHandler          ; Tamper
                DCD     RTC_IRQHandler             ; RTC
                DCD     FLASH_IRQHandler           ; Flash
                DCD     RCC_IRQHandler             ; RCC
                DCD     EXTI0_IRQHandler           ; EXTI Line 0
                DCD     EXTI1_IRQHandler           ; EXTI Line 1
                DCD     EXTI2_IRQHandler           ; EXTI Line 2
                DCD     EXTI3_IRQHandler           ; EXTI Line 3
                DCD     EXTI4_IRQHandler           ; EXTI Line 4
```

这就是中断向量表，表内是什么呢？

> 第2行，AREA    RESET, DATA, READONLY，指定了这段代码放在RESET 段，这个RESET会在分散加载文件中指定在芯片启动的位置0x8000000。所以芯片能在复位时执行Reset_Handler函数。从这里可见，复位也是一个中断。
>
> 后面一堆DCD，DCD的意思是，分配一个4个字节，内容是后面跟着的标志，通常是一个函数名，也就是一个函数指针，一个函数的地址。比如：DCD     EXTI4_IRQHandler，意思就是把EXTI4_IRQHandler这个函数的地址放在这个地方。如此一来，当产生EXTI4中断时，芯片就会自动到这个位置取这个函数地址，然后执行这个函数。

#### 外设中断

一个芯片有很多外设，每个外设都有各种中断。NVIC决定了芯片能响应多少中断，但是，芯片能在一个中断入口中实现多种中断。

比如，串口5在中断向量表中只有一个中断入口，但是串口5有很多中断，接收到数据中断，发送数据完成中断，接收溢出中断等。

有个更特别的，EXTI15_10_IRQHandler，从名字就知道，这个中断入口时外部中断15~外不中断10，总共6个外不中断的入口。

所以，要正确配置一个中断，除了NVIC之外，对应外设的中断配置相当重要。

下面我们看看STM32F103这款芯片的IO口外部中断。

#### STM32 EXIT中断

1. 所有IO都支持中断。
2. 中断方式有电平、边沿（上升沿、下降沿）
3. 中断线只有16根，所有PORT的相同编号的IO，共用中断线，只能用同时有一个IO具有中断功能。
4. 中断入口并没有16个，EXTI15_10_IRQHandler，EXTI9_5_IRQHandler。

## 外部中断例程

参考官方例程：STM32F10x_StdPeriph_Lib_V3.5.0\Project\STM32F10x_StdPeriph_Examples\EXTI\EXTI_Config

配置中断如下，分4部分：

```
void EXTI0_Config(void)
{
  /* Enable GPIOA clock */
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
  
  /* Configure PA.00 pin as input floating */
  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
  GPIO_Init(GPIOA, &GPIO_InitStructure);

  /* Enable AFIO clock */
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);

  /* Connect EXTI0 Line to PA.00 pin */
  GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);

  /* Configure EXTI0 line */
  EXTI_InitStructure.EXTI_Line = EXTI_Line0;
  EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
  EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Rising;  
  EXTI_InitStructure.EXTI_LineCmd = ENABLE;
  EXTI_Init(&EXTI_InitStructure);

  /* Enable and set EXTI0 Interrupt to the lowest priority */
  NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
  NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x0F;
  NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x0F;
  NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
  NVIC_Init(&NVIC_InitStructure);
}
```

1. IO口初始化，将IO配置为浮空输入。（纯粹的浮空是不行的，外部应该带上下拉）

2. 使能AFIO时钟，并用函数GPIO_EXTILineConfig将GPIOA的GPIO_Pin_0连接到中断源GPIO_PinSource0。

3. 配置外部中断，调用EXTI_Init函数配置EXTI_Line0为中断模式，Rising上升沿触发，ENABLE。

4. 配置NVIC，调用函数NVIC_Init配置EXTI0_IRQn， Pre优先级（）为0x0f，Sub优先级（）也配置为0x0f，使能（ENABLE）

   >  对于NVIC的配置，特别是优先级的配置，需要先设置优先级分组。不同的分组支持不同的pre和sub优先级。
   >
   > 此处我们暂时不管优先级问题。

中断配置好后，还需要编写中断服务函数。通常我们统一将中断函数放在`stm32f10x_it.c`

```
void EXTI0_IRQHandler(void)
{
  if(EXTI_GetITStatus(EXTI_Line0) != RESET)
  {
    /* Toggle LED1 */
     STM_EVAL_LEDToggle(LED1);

    /* Clear the  EXTI line 0 pending bit */
    EXTI_ClearITPendingBit(EXTI_Line0);
  }
}
```

> 函数名EXTI0_IRQHandler跟中断向量表中的名字一致。
>
> 进入中断后，先判断是不是EXTI_Line0中断，是才进入处理，STM_EVAL_LEDToggle 转换LED状态，这就是用户功能。然后EXTI_ClearITPendingBit，清楚中断标志，如果不清楚，中断就会重复进入。

由此可知一个中断的响应流程：

> IO状态变化-->产生EXIT事件标志-->如果使能了中断，就会产生设备中断标志，并产生信号发送给NVIC-->NVIC使能了中断，芯片自动操作，从当前执行位置跳转到中断向量指定的函数-->执行中断服务程序-->执行完之后就返回原来的位置继续执行。

## 按键中断

板子的4个按键连接在GPIOB的12/13/14/15四个IO。

参考例程配置即可，代码如下：

```
void EXTI15_10_Config(void)
{
	EXTI_InitTypeDef   EXTI_InitStructure;
	GPIO_InitTypeDef   GPIO_InitStructure;
	NVIC_InitTypeDef   NVIC_InitStructure;
	

  /* Configure PG.08 pin as input floating */
 /*  外部中断的IO，属于输入IO，先配置未输入
	输入IO有3中模式，浮空，上拉，下拉。做为按键通常是上拉，按下按键就是接地。
  */
  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12|GPIO_Pin_13|GPIO_Pin_14|GPIO_Pin_15;
  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
  GPIO_Init(GPIOB, &GPIO_InitStructure);

  /* Enable AFIO clock */
  RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
  
  /* Connect EXTI8 Line to PG.08 pin */
  /*  配置中断线，中中断源接到制定的IO口*/
  GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource12);
  GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource13);
  GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource14);
  GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource15);
  
  /* Configure EXTI8 line */
  /* 使能外部中断*/
  EXTI_InitStructure.EXTI_Line = EXTI_Line12|EXTI_Line13|EXTI_Line14|EXTI_Line15;
  EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
  EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
  EXTI_InitStructure.EXTI_LineCmd = ENABLE;
  EXTI_Init(&EXTI_InitStructure);

  /* Enable and set EXTI9_5 Interrupt to the lowest priority */
  /* 配置NVIC */
  NVIC_InitStructure.NVIC_IRQChannel = EXTI15_10_IRQn;
  NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0x0F;
  NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0x0F;
  NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
  NVIC_Init(&NVIC_InitStructure);
}
```

中断服务函数也是一样的。

```
/*
	EXTI15_10_IRQHandler 中断服务函数名，要跟中断限量表一致。		
*/
void EXTI15_10_IRQHandler(void)
{
	/* EXTI15_10_IRQHandler 是10~15中断线的入口。
	   产生中断后，只能通过查询EXTI状态来判断到底是那根IO产生了中断。	
	*/
  if(EXTI_GetITStatus(EXTI_Line12) != RESET)
  {
	app_display_num(4, 0, 1);

	/* Clear the  EXTI line 8 pending bit */
	
	/* 请中断标志 */
	EXTI_ClearITPendingBit(EXTI_Line12);
  }

......
```

> 响应中断后，调用app_display_num在数码管上显示数字。

编译下载，按4个按键，对应在最后一位数码管上显示1~4四个数字。

## 使用中断的按键处理

1. 中断作为启动，不需要一直查询IO状态。
2. IO口产生中断后，禁止中断，开始用原来的方式扫描按键。
3. 扫描结束后，开启中断。

---

2020-05-04

end

[1]: pic/pic1.jpg

