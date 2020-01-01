# PWM输出与蜂鸣器

PWM输出是定时器的一个功能。

什么是PWM？

配合一根IO才能输出。

查原理图和datasheet，看IO是哪个定时的的哪个通道。

PC6 TIM8 CH1

官方例程
STM32F10x_StdPeriph_Lib_V3.5.0\Project\STM32F10x_StdPeriph_Examples\TIM\PWM_Output

除了设置定时器，还需要配置PWM，还需要配置IO口。

查蜂鸣器响应频率，4K

调试时，可以用示波器测试是否有PWM输出。

