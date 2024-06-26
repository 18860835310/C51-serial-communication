# C51-serial-communication
C51-串口通信（简单的回显程序与定时器互相占用问题）

# 问题与原理
## 问题
在使用C51的串口通信过程中，我们会发现串口通信时无法使用定时器/计时器T1精准延时

或者是，使用计时器1精准延时程序时无法使用串口通信收发数据。

## 原理
串口通信功能使用了定时器T1来产生串口波特率，并占用了单片机的P3.0（RXD）和P3.1（TXD）引脚来进行串口数据接收和发送。
定时器T1被配置为模式2，即8位自动重装模式。该模式的特点是定时器只使用TL1作为8位计数寄存器，并且当计数器溢出时，会自动将TH1的值重新加载到TL1，从而实现周期性的定时中断。这对于产生稳定的波特率非常有用。
所以，如果使用定时器T1来给串口通信产生波特率，那么只能用定时器T0来精准延时，反之亦然。

# 测试程序
## 一、利用串口实现回显程序
先独立的使用串口通信，并看看配置与效果
```c
#include "reg52.h"

typedef unsigned int u16;	//对系统默认数据类型进行重定义
typedef unsigned char u8;

void uart_init(u8 baud)
{
	TMOD|=0X20;	//设置计数器工作方式2
	SCON=0X50;	//设置为工作方式1
	PCON=0X80;	//波特率加倍
	TH1=baud;	//计数器初始值设置
	TL1=baud;
	ES=1;		//打开接收中断
	EA=1;		//打开总中断
	TR1=1;		//打开计数器		
}

void main()
{	
	uart_init(0XFA);//波特率为9600

	while(1)
	{			
							
	}		
}

void uart() interrupt 4 //串口通信中断函数
{
	u8 rec_data;

	RI = 0;			//清除接收中断标志位
	rec_data=SBUF;	//存储接收到的数据
	SBUF=rec_data;	//将接收到的数据放入到发送寄存器
	while(!TI);		//等待发送数据完成
	TI=0;			//清除发送完成标志位				
}
```
这个程序的功能是实现单片机的串口通信中断。
当通过串口助手发送数据给单片机时，单片机接收到数据后会原封不动地将数据发送回串口助手进行显示。
这是一个简单的回显程序。

### 程序功能
1.初始化串口通信：在main函数中调用uart_init函数，设置串口通信的波特率以及中断配置。
2.串口中断处理：当接收到数据时，进入串口中断服务函数uart，将接收到的数据原封不动地发送回去。
### 片上资源
1.计数器T1：用于设置串口波特率。程序中使用方式2，即8位自动重装方式。
2.串口控制寄存器（SCON）：用于设置串口工作方式和控制寄存器的各个标志位。
3.电源控制寄存器（PCON）：用于设置波特率加倍（SMOD位）。
4.串口缓冲寄存器（SBUF）：用于存放接收和发送的数据。
5.串口中断：使用串口接收/发送中断（RI和TI）。
6.P3.0（RXD）：接收数据引脚。P3.1（TXD）：发送数据引脚。

### 详细解释
#### 1、初始化过程（uart_init函数）：
TMOD |= 0X20;：设置T1为方式2，8位自动重装模式。
SCON = 0X50;：设置串口为模式1，8位UART，允许接收。
PCON = 0X80;：设置SMOD位为1，波特率加倍。
TH1 = 0XFA;和TL1 = 0XFA;：设置T1初始值为0xFA，对应波特率9600（假设晶振频率为11.0592MHz）。
ES = 1;：允许串口中断。
EA = 1;：允许总中断。
TR1 = 1;：启动T1计数器。
#### 2、主函数（main函数）：
仅调用uart_init进行初始化配置，然后进入无限循环等待中断发生。
uart_init配置细节：
```c
void uart_init(u8 baud)
{
    TMOD |= 0x20;  // 设置定时器T1为模式2（8位自动重装）
    SCON = 0x50;   // 设置串口为模式1（8位UART），允许接收
    PCON = 0x80;   // 设置波特率加倍
    TH1 = baud;    // 设置TH1初始值
    TL1 = baud;    // 设置TL1初始值
    ES = 1;        // 允许串口中断
    EA = 1;        // 允许全局中断
    TR1 = 1;       // 启动定时器T1
}
```
#### 3、中断服务函数（uart函数）：
RI = 0;：清除接收中断标志位。
rec_data = SBUF;：读取接收到的数据。
SBUF = rec_data;：将接收到的数据写入发送缓冲寄存器进行发送。
while(!TI);：等待发送完成。
TI = 0;：清除发送完成标志位。
#### 4.总结
这个程序实现了一个简单的串口通信功能，将接收到的数据原样发送回去（回显）。
它使用了定时器T1来产生串口波特率，占用了单片机的P3.0（RXD）和P3.1（TXD）引脚来进行串口数据接收和发送。

## 二、串口回显程序与计时器精准延时
为了确保定时器0用于延时的同时，串口通信中断能够正常工作，需要确保全局中断和串口中断都是启用状态。
以下是整体的代码：
```c
#include "reg52.h"

typedef unsigned int u16;  // 对系统默认数据类型进行重定义
typedef unsigned char u8;

sbit LED1 = P2^0;

void uart_init(u8 baud)
{
    TMOD |= 0x20;   // 设置计数器工作方式2
    SCON = 0x50;    // 设置为工作方式1
    PCON = 0x80;    // 波特率加倍
    TH1 = baud;     // 计数器初始值设置
    TL1 = baud;
    ES = 1;         // 打开接收中断
    EA = 1;         // 打开总中断
    TR1 = 1;        // 打开计数器
}

void time0_init(void)
{
    TMOD |= 0x01;   // 设置定时器0为模式1（16位定时器）
    TH0 = 0xFC;     // 设置初始值高8位
    TL0 = 0x66;     // 设置初始值低8位
    ET0 = 0;        // 禁用定时器0中断
    TR0 = 0;        // 停止定时器0
}

void delay_ms(u16 ms)
{
    while (ms--)
    {
        TH0 = 0xFC;    // 重新加载定时器初始值
        TL0 = 0x66;
        TR0 = 1;       // 启动定时器0
        while (!TF0);  // 等待定时器溢出
        TF0 = 0;       // 清除溢出标志
        TR0 = 0;       // 停止定时器0
    }
}

void main()
{
    uart_init(0xFA); // 波特率为9600
    time0_init();  
    while(1)
    {
        delay_ms(500);
        LED1 = !LED1;   			
    }		
}

void uart() interrupt 4 // 串口通信中断函数
{
    u8 rec_data;

    RI = 0;         // 清除接收中断标志位
    rec_data = SBUF; // 存储接收到的数据
    SBUF = rec_data; // 将接收到的数据放入到发送寄存器
    while (!TI);    // 等待发送数据完成
    TI = 0;         // 清除发送完成标志位
}
```

























