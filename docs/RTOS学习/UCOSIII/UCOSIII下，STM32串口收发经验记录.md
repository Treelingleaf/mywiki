# UCOSIII下，STM32串口收发经验记录

#### 1. 初始化配置串口

和普通的裸机程序一样：

```c
void uart_init(u32 bound)
{
	// GPIO端口设置
	GPIO_InitTypeDef GPIO_InitStructure;
	USART_InitTypeDef USART_InitStructure;
	NVIC_InitTypeDef NVIC_InitStructure;

	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1 | RCC_APB2Periph_GPIOA, ENABLE); // 使能USART1，GPIOA时钟
	USART_DeInit(USART1);														  // 复位串口1
																				  // USART1_TX   PA.9
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;									  // PA.9
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP; // 复用推挽输出
	GPIO_Init(GPIOA, &GPIO_InitStructure);			// 初始化PA9

	// USART1_RX	  PA.10
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING; // 浮空输入
	GPIO_Init(GPIOA, &GPIO_InitStructure);				  // 初始化PA10

	// Usart1 NVIC 配置
	NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 3; // 抢占优先级3
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 3;		  // 子优先级3
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;			  // IRQ通道使能
	NVIC_Init(&NVIC_InitStructure);							  // 根据指定的参数初始化VIC寄存器

	// USART 初始化设置
	USART_InitStructure.USART_BaudRate = bound;										// 一般设置为9600;
	USART_InitStructure.USART_WordLength = USART_WordLength_8b;						// 字长为8位数据格式
	USART_InitStructure.USART_StopBits = USART_StopBits_1;							// 一个停止位
	USART_InitStructure.USART_Parity = USART_Parity_No;								// 无奇偶校验位
	USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None; // 无硬件数据流控制
	USART_InitStructure.USART_Mode = USART_Mode_Rx | USART_Mode_Tx;					// 收发模式

	USART_Init(USART1, &USART_InitStructure);	   // 初始化串口
	USART_ITConfig(USART1, USART_IT_RXNE, ENABLE); // 开启中断
	USART_Cmd(USART1, ENABLE);					   // 使能串口
}
```

‍

#### 2. 接收部分

首先是定义一个结构体作为串口接收的缓冲区：

```c
/*接收*/
#define USART_REC_LEN 64  //缓冲区最大长度
typedef struct
{
    u8 rcv_buf[USART_REC_LEN]; // 接收缓冲
    u16 rcv_buf_ptr;           // 接收缓冲区指针
    u8 rcv_flag;               // 接收完成标志
} USART1RxBuf;
```

然后是串口中断服务函数，前提是使能了窗串口中断和配置了NVIC：

```c
// 串口接收
void USART1_IRQHandler(void)
{
#ifdef SYSTEM_SUPPORT_OS // 如果时钟节拍数定义了,说明要使用ucosII了.
	OSIntEnter();
#endif
	if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET) {
        char receivedChar = USART_ReceiveData(USART1);
      
        if (receivedChar == '\r' || receivedChar == '\n') {
            // 收到回车或换行，表示字符串接收完成
			newflag++;
			if( newflag ==2){
				usart1RxBuf.rcv_flag = 1;
				usart1RxBuf.rcv_buf[usart1RxBuf.rcv_buf_ptr] = '\0'; // 在字符串末尾添加空字符
				usart1RxBuf.rcv_buf_ptr = 0;
				printf("回显: %s", usart1RxBuf.rcv_buf);
				newflag = 0;
			}
        } else if (usart1RxBuf.rcv_buf_ptr < USART_REC_LEN - 1) {
            // 正在接收字符串，将字符存储到接收缓冲区
            usart1RxBuf.rcv_buf[usart1RxBuf.rcv_buf_ptr++] = receivedChar;
        }
    }
#ifdef SYSTEM_SUPPORT_OS // 如果时钟节拍数定义了,说明要使用ucosII了.
	OSIntExit();
#endif
}
```

🐼**解释：**

1. 如果收到回车（'\r'）或换行（'\n'）字符，表示字符串接收完成。在这种情况下，它会将 `usart1RxBuf.rcv_flag`​ 设置为1，将接收缓冲区的最后一个字符设置为 null 终止符（'\0'），并打印接收到的字符串，相当于自定义帧格式了。
2. 如果没有收到回车或换行字符，且接收缓冲区还有足够的空间，它会将接收到的字符存储到接收缓冲区中。

‍

#### 3.发送部分

发送部分很简单，重定向`printf`​:

```c
int _write(int file, char *ptr, int len) {
    int i;
    for (i = 0; i < len; i++) {
        USART_SendData(USART1, ptr[i]);
        while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    }
    return len;
}
```

‍

#### 4. 任务实现

```c
// 串口接收任务
void task2_task(void *p_arg)
{
	u8 task2_num = 0;
	LCD_DrawRectangle(5, 214, 234, 314);	  // 画一个矩形
	LCD_DrawLine(5, 214 + 40, 234, 214 + 40); // 画线
	LCD_DrawLine(5, 214 + 60, 234, 214 + 60); // 画线
	LCD_ShowString(70, 256, 110, 16, 16, "LED_OFF");

	while (1)
	{

		if (usart1RxBuf.rcv_flag == 1)
		{
			//打印接收到的字符串到LCD
	
			if (strncmp((char *)usart1RxBuf.rcv_buf, "LED_ON", 6) == 0)
			{
				// Turn on the LED
				GPIO_ResetBits(GPIOD, GPIO_Pin_2);
				GPIO_ResetBits(GPIOA, GPIO_Pin_8);
				LCD_ShowString(70, 256, 110, 16, 16, "           ");
				LCD_ShowString(70, 256, 110, 16, 16, usart1RxBuf.rcv_buf);
				printf(">>LED已打开");
				task2_num++;
			}
			else if (strncmp((char *)usart1RxBuf.rcv_buf, "LED_OFF", 7) == 0)
			{
				// Turn off the LED
				GPIO_SetBits(GPIOD, GPIO_Pin_2);
				GPIO_SetBits(GPIOA, GPIO_Pin_8);
				LCD_ShowString(70, 256, 110, 16, 16, usart1RxBuf.rcv_buf);
				printf(">>LED已关闭");
				task2_num++;
			}

			// 清除接收缓冲区和标志
			memset(usart1RxBuf.rcv_buf, 0, sizeof(usart1RxBuf.rcv_buf));
			usart1RxBuf.rcv_buf_ptr = 0;
			usart1RxBuf.rcv_flag = 0;
		}
        LCD_Fill(6, 215, 233, 253, lcd_discolor[task2_num % 14]); // 填充区域
		LCD_Fill(6, 275, 233, 314, lcd_discolor[task2_num % 14]); // 填充区域
		delay_ms(100);// 休眠一段时间
	}
}
```
