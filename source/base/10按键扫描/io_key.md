# IO输入与按键扫描

输出输入是GPIO的两种功能，前面我们点亮LED、控制数码管，用的是IO口输出。现在我们开始学习IO口输入功能。

## 概述

IO口输入的意思就是：读取IO口上的电平：高电平为1，低电平为0。

通常我们默认默认高电平就是CPU工作电压，比如STM32就是3.3V。低电平就是GND电压，也就是0V。

> 但是其实这并不严格，在CPU的规格书中标有输入电压电平范围，比如0~0.6V，就认为是低电平，2.7V~3.3V就是高电平。这些细节除非特殊应用场合，平常不用特别注意。
>
> 还有一个要点，一些复杂的芯片，会有多种输入电压，比如一些ARM9芯片，会有内核电压、内存电压、IO电压，GPIO的高低电平对应的是IO电压。

## IO口配置

一个IO口做为输入，通常有哪些可以选择的配置呢？

不同的CPU会有差异，也有共同点。现在我们看看STM32配置一个IO口为输出，有哪些配置。

不知道大家还是否记得`GPIOMode_TypeDef`枚举定义。请看下面：

```
typedef enum
{ GPIO_Mode_AIN = 0x0,
  GPIO_Mode_IN_FLOATING = 0x04,
  GPIO_Mode_IPD = 0x28,
  GPIO_Mode_IPU = 0x48,
  GPIO_Mode_Out_OD = 0x14,
  GPIO_Mode_Out_PP = 0x10,
  GPIO_Mode_AF_OD = 0x1C,
  GPIO_Mode_AF_PP = 0x18
}GPIOMode_TypeDef;
```

> 上面就是STM32 IO的模式，输入的模式我们讲过了。哪些是输入模式呢？
>
> `GPIO_Mode_IN_FLOATING`、`GPIO_Mode_IPD`、`GPIO_Mode_IPU`这三种模式就是输入模式。
>
> 从名字看，三种模式的区别仅仅是上下拉电阻配置不一样。分别是：FLOATING-浮空、IPD-下拉、IPU-上拉。

所以，如果用ST的库函数配置一个IO位输入，是非常简单的。

```
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12;
   	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
   	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
   	GPIO_Init(GPIOB, &GPIO_InitStructure);
```

> 第1行代码选择要配置的IO，可以同时配置多个。
>
> 第2行选择速度。
>
> 第3行关键，输入模式。
>
> 然后调用GPIO_Init函数初始化即可。

那么如何获取输入状态呢？看库函数头文件都提供了哪些功能函数就可以知道了。

只有两个函数：

```
uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx)
...
uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)

```

>  这两个函数的功能，注释说的很清楚。
>
> 第1个函数是读整组IO的状态，所以返回值是一个uint16_t，也就是一个16位的数，正好对应一组IO口。
>
> 第2个函数是读一个Bit，也就是一个指定的IO口的值。返回值是一个uint8_t，但是并不是会返回8位。看函数：返回值有两种，Bit_SET和Bit_RESET，高电平，返回Bit_SET，低电平返回Bit_RESET。

## 按键原理

理解了IO输入原理，按键原理就简单了。

#### 按键硬件接法

一个IO只能输入两种状态，接在上面的按键，当然也只会有两种状态。

我们教学板上有4个按键接在IO口上，请看原理图：

![][1]

![][2]

4个按键的原理图是一样的。

> 按键一端接到地，另外一端接到IO口。
>
> 在IO口这端，通过一个电阻接到VCC，这个电阻就是我们常说的上拉电阻。
>
> 按键按下，IO口接到地，输入低电平。按键松开，IO口通过电阻接到VCC，输入高电平。

**上拉电阻**

> 如果我们把IO口配置为上拉模式，内部已经有一个上拉电阻。但是这个电阻阻值通常是固定的。
>
> 在某些情况，我们需要灵活配置上拉电阻。
>
> 上拉电阻作用是当按键没有按下时，把IO口的状态维持在文档的高电平，防止程序读到意外状态。
>
> 但是，这个电阻还有一个重要的要点，就是电流。当按键按住，VCC将通过这个电阻连接到地线。
>
> 流过的电流=VCC/R。
>
> 所以这个电阻不能选太小的，太小电流大。比如在一些低功耗设备，按键会一直按住，进入睡眠，如果电阻很小，电流就很大。有时我们会用1M的电阻，这样一个按键的电流就只有3uA。
>
> 但是电阻也不能选太大，越大的电阻本身寄生的电容电感就很大，很容有受到外部干扰。
>
> 通常，如果不是要求超低功耗的设备，用几K几十K的电阻。
>
> 超低功耗设备选择1M电阻，最大不超过3M。

## 调试

### 第一步

使用单步调试，确定IO口输入有反应。

初始化代码：

```
/*  按键接在 PB12 PB13 PB14 PB15*/
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
   	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_12|GPIO_Pin_13|GPIO_Pin_14|GPIO_Pin_15;
   	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
   	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
   	GPIO_Init(GPIOB, &GPIO_InitStructure);
```

测试代码

```
	while(1)
	{	
		/* 读一组IO口输入 */
		sta = GPIO_ReadInputData(GPIOB);
		/* 16位IO，按键用高四位，通过与操作 屏蔽其他低位
		如果高四位不等于0xf，说明有按键按下*/
		if ((sta & 0xf000) != 0xf000) {
			/*没用的语句，用来测试是否进入这里*/
			sta = 1;	
		}
	}
```

编译后下载，进入debug模式。

在sta=1的地方加上断点，然后全速运行。

分别按下四个按键，看是否在断点处停止。

测试正常进入下一步。

### 第二步

IO口能检测到按键按下了，但是还不能确认按键能用。

我们现在是检测到按下就停住了，正常程序是不会的吧？

如果全速运行，会发生什么呢？

前面我们调试了8位数码管，我们现在可以将IO口的状态显示在数码管上，看看全速运行时按键是什么状态。

在8位数码管的应用中，将定时刷新数码管应用改为下面

```
io_sta = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_12);
	if (io_sta == Bit_RESET) {
		seg_fill_disbuf(SegTab[0], 1);	
	} else {
		seg_fill_disbuf(SegTab[1], 1);	
	}

	io_sta = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_13);
	if (io_sta == Bit_RESET) {
		seg_fill_disbuf(SegTab[0], 2);	
	} else {
		seg_fill_disbuf(SegTab[1], 2);	
	}

	io_sta = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_14);
	if (io_sta == Bit_RESET) {
		seg_fill_disbuf(SegTab[0], 3);	
	} else {
		seg_fill_disbuf(SegTab[1], 3);	
	}

	io_sta = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_15);
	if (io_sta == Bit_RESET) {
		seg_fill_disbuf(SegTab[0], 4);	
	} else {
		seg_fill_disbuf(SegTab[1], 4);	
	}
```

> 按下对应按键时将对应数码管显示为1，松开按键，数码管显示0。

下载测试，看起来还很好，能反应IO口状态变化。那是不是真的呢？

### 第三步

上一步测试，数码管能正常显示IO口状态，但是其实是不稳定的。

我们改一个程序试下：把第一个按键的程序改为下面：

```
	io_sta = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_12);
	if (io_sta == Bit_RESET) {
		cnt++;
		if(cnt > 9)
			cnt = 0;
		seg_fill_disbuf(SegTab[cnt], 1);	
	} else {
		
	}
```

> 按键按下一次，显示数字加1，到9后返回0，循环显示。

编译下载测试。什么效果？

数码管1数字确实在加，但是并不是我们要的效果，我们想象的效果是按下一次，数码管加1。现在按下一次，数码管变化几个数字。

为什么？

### 第四步

第三步的效果跟我们想象不一样。两个原因：

1. 按键抖动。
2. CPU跑的很快，读IO口的状态也很快。

* 按键抖动

  按键，是机械结构，按下时接触点之间会弹动。我们认为按键按下，电平就立刻从高电平转为低电平，实际并不是这样。

  实际的图如下：

![][3]

> 无论按键按下还是松开，IO口的状态都不是立刻、稳定变化。电平状态会来回抖动。
>
> 当CPU读IO口的速度很快时，将会读都多个低电平。



> **这个图只是说明了按键造成的抖动问题**
>
> 实际上，还有一个问题：IO口的状态变化也是要时间的。
>
> IO口从1变为0，并不是立刻变化，0变为1也不是立刻变化。是要时间的，尽管这个时间很快很快，那还是要时间的。
>
> 普通的IO口速度只有10M，高速的可以做到100M以上。
>
> 这个知识点在这里暂时不会有影响，记住就好。

如何去抖动呢？

原理很简单：

**重复读，当连续多次都是低电平时，按键就是低电平了。**

这是很多例程去抖动的方法。方法是对的，但是状态抽象不对，状态抽象不对，程序写的逻辑就不好。

我认为按键去抖动，或者说按键扫描的逻辑是：

**当状态连续多次变化到另外一种状态，说明按键状态变化了。**

确定变化后，再根据状态判断是按下还是松开。

如此一来，松开和按下都有防抖动，而程序只有一个防抖动流程。

代码如下：

```
		/*-----------------应用-----------------*/
		new_sta = GPIO_ReadInputData(GPIOB);	
		new_sta = new_sta&0xf000;

		/*  这里其实有一个提高可靠性的问题，
			old_sta是稳定状态，
			sta是本次读到的状态，
			如果本次读到的状态跟上次读到的状态不一样呢？		*/
		if (sta != new_sta) {
			/* 不相等，说明状态变化*/
			debcnt++;
			if (debcnt >= 10) {
				/* 已经连续10次状态变化，说明变化已经稳定*/
				debcnt = 0;
				
				/*	求出变化的位	*/
				chg_bit = sta ^ new_sta;
				/* 检测变化IO现在的状态 */
				if ((new_sta & chg_bit) != 0) {
					/*按键松开*/
				} else {
					/* 按键按下 */

					/*根据变化的位，判断是哪个按键按下 */
					if(chg_bit == 0x8000) {
						/* 应用 */
						cnt++;
						if(cnt > 9)
							cnt = 0;
						seg_fill_disbuf(SegTab[cnt], 1);	
					}
				}

				/*更新状态*/
				sta = new_sta;
			}
		} else {
			debcnt = 0;	
		}
```

> new_sta是当前读到的IO口状态。
>
> 两个状态比较，是否有变化？
>
> 有变化，debcnt自加，去抖动。（这有个隐患，大家能想到吗？）
>
> 通过异或求出变化的bit
>
> 用变化的bit和new_sta比较，判断到底是按键按下还是按键松开。
>
> 根据 cha_bit，确定是哪个按键按下。（用相等比较，有隐患？能想到吗？）

如果是最高位的按键按下，显示数字就自加。

编译下载测试，效果杠杠的，按下按键数字就加1，没有误按键。

### 总结

这种扫描方法，有一个很容易理解的切入点：滑窗法

### 最后

那，现在按键驱动算完成了吗?

还差点，差什么呢？差模块化。

上面的程序，应用功能是数码管数字自加。这几行代码属于应用，却镶嵌在按键驱动中。

所以我们要把按键驱动整理为一个模块。按键驱动扫描到按键后，放在缓冲。

应用从缓冲中提取按键。

按键就是生产者，应用就是消费者。

---

end

20200419



[1]: pic/pic1.jpg
[2]: pic/pic2.jpg
[3]: pic/pic3.jpg

