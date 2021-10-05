# 动态扫描数码管

前面我们学习了如何使用一位LED显示数字，很简单是吧？

现在我们加点难度。一位数码管只能显示一位数字，现在我们要显示8位数字（或者显示时间）。

那么我们就需要8位数码管，如果按照1位数码管的硬件接法，8位数码管就需要64根IO。

相当于1个LED使用1根IO口控制。

大家觉得可行吗？当然可行，我们芯片有100根管脚，80多根IO。

但是你只打算用芯片控制8位数码管吗？肯定不是嘛！这样的方案肯定是非常浪费IO的。

那怎么办嗯？要解决这个问题，要用到`一个原理`，`两个芯片`。

### 一个原理

不知道大家是否了解过以前的胶片电影，一张一张的画片，连续播放就能看到活生生的，会动的人。

这是为什么呢？

> 原理是“视觉暂留”。
>
> 科学实验证明，人眼在某个视像消失后，仍可使该物像在视网膜上滞留0.1－0.4秒左右。电影胶片以每秒24格画面匀速转动，一系列静态画面就会因视觉暂留作用而造成一种连续的视觉印象，产生逼真的动感。

我们在数码管上能不能用这个原理呢？

8个数码管都用同样的IO控制亮灭，轮流显示。

只要同一个LED的点亮间隔不大于0.4秒（实际要比这个小），那么我们就会一直看到这个数码管是亮着的。（和真正一直亮会有什么差别？）

这样我们就只要8根IO了？

如何选择8个数码管该点亮哪个？

前面我们用共阴极数码管，阴极是接到地线的。

我们可以用IO口控制阴极，只有对应的IO是低电平，这个数码管才有亮。如果阴极是高电平，数码管就不会亮。

如此，我们需要8+8根IO就够了。省去了48根IO，太有成就了。

### 两个芯片

我们都是高兴太早，通常IO口还是不够， 使用16根IO也是很浪费的。

那这么办呢？利用数字电路，有两个芯片能帮上我们的忙。

**74HC138**和**74HC595**。

138是三八译码器，595是8位串行输入、并行输出的位移缓存器。

> 74是一系列数字功能芯片，注意中间字母的区别。我们选用的是HC类型，HC表示是CMOS电平，或者简单说就是3.3V电压。

* 三八译码器

  三八译码器是什么？我们从数据手册一探究竟。

  打开数据手册，标题：**SNx4HC138 3-Line To 8-Line Decoders/Demultiplexers**

  翻译为中文就是：**SNx4HC138 3线转8线译码器/多路分配器**，怎么转呢？往下看。

  我们选用的型号是**SN74HC138PWR**。型号这些数字和字母都是什么意思呢？

  >  SN74是芯片系列。
  >
  > HC是芯片种类。
  >
  > 138是芯片具体型号。
  >
  > PW是封装，TSSOP16。
  >
  > R包装形式，编带。

  138的功能：**用3根线的电平，选择8根线中的一根线输出低电平，其他输出高电平**。

  芯片电气信号

   ![][5]

  真值表如下：

  ![][6]

  > 左边是输入，右边是输出。
  >
  > ENABLE信号通常我们输出H-L-L，也就是默认使能，不进行控制。
  >
  > C/B/A，38译码器3根输入线，一共有8种组合。
  >
  > 输出信号8根，根据3根输入线的状态，选择其中1根输出**低电平**，其他线输出高电平。
  >
  > 因此，**选中的数码管是低电平，那么就只能用共阴极数码管**。

* 595功能

  打开595手册

  标题：**8-Bit Shift Registers With 3-State Output Registers**

  意思：8位移位寄存器，具有3态输出。

  我们选用的型号是：SN74HC595PWR， 名称含义与138类似。

  **芯片电气信号**

  ![][7]

   **时序图**

  ![][8]

>14脚SER输入，11 脚SRCLK上升沿，从14脚输入1位数据。8次之后，就有一个BYTE的数据保存在595中。当时钟继续输出，数据将从9脚输出，因此，可以通过多个595串联实现更多的移位位数。两个595就可以组成16位移位寄存器。
>
>12脚RCLK上升沿，保存在595中的8位数据，从595的8个并行输出引脚输出（OE需要低电平）
>
>10脚SRCLR是复位脚，低电平有效 ，上电后输出高即可。
>
>更多细节可参考：https://baike.baidu.com/item/74HC595/9886491

**我们用三八译码器控制刷管的共阴极，595控制数码管的正极。三八译码器决定哪个数码管亮，595决定亮的内容。如此，我们就只需要7个IO口就搞定了。**

### 硬件原理

节省IO是一种共识，所以要用8位数码管时，我们不需要用8位单独的数码管组成。

而是用2个内部连接好信号的4位数码管。如下图：

![][1]

这种数码管内部已经将共用的信号连在一起。同样，也有共阴极和共阳极数码管之分。

内部连接信号如下:

![][2]

![][3]

电路图根据前面分析的原理设计，如下图：

![][4]

### 调试

#### 第一步 

静态显示，38译码器设定一个固定输出，选中一个数码管，控制595输出，让数码管显示不同数字。

* 初始化硬件

  ```
  /*
  	595_SDI--- ADC-TPX---PB0---数据输入
  	595_LCLK---ADC-TPY---PB1---数据锁存---上升沿锁存
  	595_SCLK---TP-S0---PC5---数据移位---上升沿移位
  	595_RST---TP-S1---PC4---芯片复位--低电平复位
  
  	A138_A0---FSMC_D2---PD0
  	A138_A1---FSMC_D1---PD15
  	A138_A2---FSMC_D0---PD14
  */
  void seg_init(void)
  {
  	/* GPIOD Periph clock enable */
  	  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
  	  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
  	  RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);
  	
  	  /* 38译码器输入0 ，选中第4个数码管*/
  	  GPIO_ResetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_14|GPIO_Pin_15);
  	  /* Configure PD0 and PD2 in output pushpull mode */
  	  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0|GPIO_Pin_14|GPIO_Pin_15;
  	  GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
  	  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
  	  GPIO_Init(GPIOD, &GPIO_InitStructure);
  	
  	  GPIO_ResetBits(GPIOB, GPIO_Pin_0|GPIO_Pin_1);
  	  /* Configure PD0 and PD2 in output pushpull mode */
  	  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0|GPIO_Pin_1;
  	  GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
  	  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
  	  GPIO_Init(GPIOB, &GPIO_InitStructure);
  	
  	  GPIO_ResetBits(GPIOC, GPIO_Pin_4|GPIO_Pin_5);
  	  /* Configure PD0 and PD2 in output pushpull mode */
  	  GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4|GPIO_Pin_5;
  	  GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
  	  GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
  	  GPIO_Init(GPIOC, &GPIO_InitStructure);
  	
  	  /* 拉高复位信号 */
  	  GPIO_SetBits(GPIOC, GPIO_Pin_4);
  
  }
  ```

* 138驱动

  ```
  /* 
  	选择数码管，控制138选中对应数码管
  	pos参数就是位置
  */
  void seg_select(uint8_t pos)
  {
  	if (pos == 1) {
  		GPIO_SetBits(GPIOD, GPIO_Pin_14);
  		GPIO_ResetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_15);
  	} else if (pos == 2) {
  		GPIO_SetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_14);
  		GPIO_ResetBits(GPIOD, GPIO_Pin_15);	
  	} else if(pos == 3) {
  		GPIO_SetBits(GPIOD, GPIO_Pin_15|GPIO_Pin_14);
  		GPIO_ResetBits(GPIOD, GPIO_Pin_0);
  	} else if(pos == 4) {
  		GPIO_SetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_15|GPIO_Pin_14);
  	} else if (pos == 5) {
  		GPIO_ResetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_15|GPIO_Pin_14);
  	} else if(pos == 6) {
  		GPIO_ResetBits(GPIOD, GPIO_Pin_15|GPIO_Pin_14);
  		GPIO_SetBits(GPIOD, GPIO_Pin_0);
  	} else if(pos == 7) {
  		GPIO_ResetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_14);
  		GPIO_SetBits(GPIOD, GPIO_Pin_15);
  	} else if(pos == 8) {
  		GPIO_ResetBits(GPIOD, GPIO_Pin_14);
  		GPIO_SetBits(GPIOD, GPIO_Pin_0|GPIO_Pin_15);
  	}
  }
  ```

* 595驱动

  ```
  /*
  	输出一位数码管显示数据
  */
  void seg_display_1seg(uint8_t segbit)
  {
  	uint8_t tmp;
  	uint8_t cnt = 0;
  
  	tmp = segbit;
  	
  	cnt = 0;
  	/* 拉低 595_LCLK*/
  	GPIO_ResetBits(GPIOB, GPIO_Pin_1);
  	
  	while(1) {
  		/* 拉低 595_SCLK*/
  		GPIO_ResetBits(GPIOC, GPIO_Pin_5);
  		/* 将数据从 SDI发出去*/	
  		if((tmp & 0x80)== 0x00)//注意操作符的优先级
  		{	
  			GPIO_ResetBits(GPIOB, GPIO_Pin_0);		
  		} else	{	
  			GPIO_SetBits(GPIOB, GPIO_Pin_0);
  		}
  		
  		tmp = tmp<<1; //移位
  		
  		delay(100);
  		/* 拉高 595_SCLK 移位数据 */
  		GPIO_SetBits(GPIOC, GPIO_Pin_5);
  		delay(100);
  		
  		cnt++;
  		if(cnt >= 8)
  			break;
  	}
  	
  	GPIO_SetBits(GPIOB, GPIO_Pin_1);
  	delay(100);
  		
  }
  ```

* 应用

  在main中初始化数码管，138固定输出值，调用595驱动函数输出各种数字。

  ```
  seg_init();
  
  	/*
  		第一步，调试595和138功能
  		在第1个数码管显示0-9
  	*/
  	seg_select(1);
  	seg_display_1seg(0x3f);
  	seg_display_1seg(0x06);
  	seg_display_1seg(0x5b);
  	seg_display_1seg(0x4f);
  	seg_display_1seg(0x66);
  	seg_display_1seg(0x6d);
  	seg_display_1seg(0x7d);
  	seg_display_1seg(0x07);
  	seg_display_1seg(0x7f);
  	seg_display_1seg(0x67);
  	seg_display_1seg(0x3f|0x80);
  ```

  输出数字对应的数码管段值，列入一个数组，索引就是数字，比如显示数字1，输出的数码管段值就是SegTab[1]， 也就是0x06。

  ```
  uint8_t SegTab[10]={0x3f, 0x06, 0x5b, 0x4f, 0x66, 0x6d, 0x7d, 0x07, 0x7f, 0x67};
  ```

  单步运行看效果。

#### 第二步

固定显示，595输出固定值，调试38译码器，让数字在数码管上轮流显示相同的数字。

代码和第一步类似。

经过第一第二步调试后，138和595驱动就完成了。

#### 第三步

第一第二步只是实现了单个数码管显示，属于静态显示。

前面讲原理时讲过，8位数码管使用动态显示方法。需要将38译码器和595配合，才能动态刷8位数据，我们现在尝试固定显示12345678。

动态刷新是一个循环，因此放在while循环中实现。

代码如下：

```

	seg_select(1);
	seg_display_1seg(0x3f);

	seg_select(2);
	seg_display_1seg(0x06);

	seg_select(3);
	seg_display_1seg(0x5b);

	seg_select(4);
	seg_display_1seg(0x4f);

	seg_select(5);
	seg_display_1seg(0x66);

	seg_select(6);
	seg_display_1seg(0x6d);

	seg_select(7);
	seg_display_1seg(0x7d);

	seg_select(8);
	seg_display_1seg(0x07);
```

单步运行，看效果。发现一个问题，在调用138切换数码管时，会将前面显示的内容显示到下一个位置。

比如，数码管1显示1，调用数码管138切换显示位置到数码管2，这时，数码管2会显示1。这是个问题。

我们全速运行程序看看效果。显示的内容并不是87654321，而是76543218，而且有重影，影子隐隐约约是我们要的效果：87654321。

如何解决这个问题呢？

方法是：**在切换显示位置前，将显示内容清零**，也就是将595的输出内容输出为0。

增加一个seg_clear函数实现这个功能。实现如下：

```
void seg_clear(void)
{
	uint8_t cnt = 0;

	cnt = 0;
	/* 拉低 595_LCLK*/
	GPIO_ResetBits(GPIOB, GPIO_Pin_1);
	
	while(1) {
		/* 拉低 595_SCLK*/
		GPIO_ResetBits(GPIOC, GPIO_Pin_5);
		/* 将数据从 SDI*/	
		GPIO_ResetBits(GPIOB, GPIO_Pin_0);		
		delay(100);
		/* 拉高 595_SCLK 移位数据 */
		GPIO_SetBits(GPIOC, GPIO_Pin_5);
		delay(100);
		
		cnt++;
		if(cnt >= 8)
			break;
	}
	
	GPIO_SetBits(GPIOB, GPIO_Pin_1);
	delay(100);
		
}
```

在前面的测试函数中所有seg_select函数之前都添加本函数。

编译下载全速运行，效果正常。

#### 第四步

经过第三步调试，8位数码管的功能已经实现了。

那，驱动算完成了吗？没有。为什么？

先介绍一个很重要的概念：**时间片**。

>  什么是时间片呢？拿LED和8位数码管进行对比。
>
>  LED，只要将IO置位，就能点亮，之后如果不改变LED状态，不需要再管它。
>
>  8位数码管呢？因为我们用动态扫描方法，不能仅仅将8位数码管输出一次内容之后就不管了，要一直刷新。
>
>  前面原理也说过，每个数码管刷一次的时间间隔不能小于24ms。
>
>  这种需要定时操作的，我们通常就说这个功能需要时间片。

好，了解了时间片。那么应用程序要如何使用呢？

> 应用程序只是想在数码管上显示一些数字而已，数码管怎么显示的，它是不管的。
>
> 为了显示数字，让应用程序间隔24ms就调用你的程序刷新显示，这明显不合理，专业术语叫强耦合，本来不相关的。

讲到这，不知道大家是否明白。不了解也没关系，后面再慢慢理解。

总之，矛盾就是：**应用只是想显示一数字，数码管驱动要时间片维持显示**。

怎么实现呢？用**缓冲**。缓冲就是一个组数，这个数组是应用和驱动之间的联系。

应用程序将要显示的内容放到缓冲。驱动将缓冲中的内容显示到数码管。

如此，就达到了最简单的**模块分离**。

> **程序设计中有一个理论：生产者和消费者。**
>
> 数码管驱动和应用虽然不是真正的生产者和消费者，但是使用缓冲的逻辑是相似的。

* 有8位数码管，就定义包含8个空间的数组。

  ```
  /* 动态扫描 添加缓冲功能 */
  /* 8位数码管的显示内容 */
  char BufIndex = 0;
  /* 缓冲，保存的是对应数码管段值 */
  char Seg8DisBuf[8]={0x7f,0x07,0x7d,0x6d,0x66,0x4f,0x5b,0x06};
  ```

* 定义一个函数，用于动态刷新数码管。这个函数最好放在定时或者RTOS的定时任务中执行。现在我们还没学会，可以放在main函数中的while运行。

  ```
  /*
  	动态刷新
  	定时调用本函数，
  	本函数对应用层屏蔽，意思是：应用层不知道我是通过动态刷新实现8位数码管功能。
  */
  void seg_display_task(void )
  {
  	seg_clear();
  	seg_select(BufIndex+1);
  	seg_display_1seg(Seg8DisBuf[BufIndex]);
  
  	BufIndex++;
  	if(BufIndex >=8)
  		BufIndex = 0;
  }
  ```

* 定义一个函数，给应用程序调用，改变数码管缓冲的值。

  ```
  /*
  	segbit 数码管段值，为1的bit点亮
  	seg 数码管位置，1~8
  */
  void seg_fill_disbuf(uint8_t segbit, uint8_t seg)
  {
  	Seg8DisBuf[seg-1] = 	segbit;
  	return;
  }
  ```

* 在main函数中调用数码管刷新功能。

  ```
      /*-----------------驱动--------------*/
  	/* 使用显示缓冲方法，要改变显示内容，
  	调用函数seg_fill_disbuf改变Seg8DisBuf中的内容即可 */
  	seg_display_task();
  
  	/*-----------------应用-----------------*/
  	cnt++;
  	if(cnt >= 1000) {
  		cnt=0;
  		disnum ++;
  		if(disnum > 9) 
  			disnum = 0;
  
  		seg_fill_disbuf(SegTab[disnum], 1);
  
  	}
  	/*----------------------------------*/
  	delay(1000);
  
  ```

  驱动是数码管的内容，while循环最后delay 1000，也就是刷新间隔。现在没定时器，暂时定一个值，数码管不闪烁即可。

  应用就是延时1000次个delay(1000)后，改变数码管1显示的数字，从0显示到9。

  编译下载看效果。

---

end

20200418



[1]: pic/pic1.jpg
[2]: pic/pic2.jpg
[3]: pic/pic3.jpg
[4]: pic/pic4.jpg

[5]: pic/pic5.jpg
[6]: pic/pic6.jpg
[7]: pic/pic7.jpg
[8]: pic/pic8.jpg