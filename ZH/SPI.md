# 目录
1. 物理总线    
2. 通信时序

    - 2.1 初始化函数
    - 2.2 起始信号
    - 2.3 终止信号
    - 2.4 模式 0 交换字节（4 个模式选其一）
    - 2.5 模式 1 交换字节（4 个模式选其一）
    - 2.6 模式 2 交换字节（4 个模式选其一）
    - 2.7 模式 3 交换字节（4 个模式选其一）

3. 补充说明与最终参考代码

    - 3.1 关于模式 0 与 1、模式 2 与 3
    - 3.2 通信速度适配优化
    - 3.3 最终参考代码

<br><br><br>






# 1、物理总线
SPI 的物理总线有四条：
<br>（1）片选线 CS：由主机控制，用于选中目标从机（也写作 SS、CS# 等，CS# 强调低电平有效）
<br>（2）时钟线 SCK：由主机提供时钟信号（也写作 SCL 等）
<br>（3）从机数据输入线 SI：由主机向从机发送指令或数据（也写作 DI、MOSI 等，MOSI 强调方向为主机发送、从机接收）
<br>（4）从机数据输出线 SO：由从机向主机发送指令或数据（也写作 DO、MISO 等，MISO 强调方向为从机发送、主机接收）
<br>主机的 CS、SCK、SI 为推挽模式，SO 为浮空输入模式。建议在 CS 上加入上拉电阻，上电时稳定 CS 电平避免噪声使从机误动作；SCK、SI、SO 则直接连接即可。（3.3V-5V 供电的一般使用场景中，上拉电阻推荐使用 10kΩ。注意：若从机的 SO 引脚为开漏模式，则 SO 也需要上拉电阻，详情请查阅您所用器件的数据手册）
<br>每个从机需要独立的 CS 控制选中状态，SCK、SI、SO 则通过并联的形式挂载多个从机。通常情况下，要求主机拉低与指定从机的 CS，使其进入通信状态，未进入通信状态的从机则不会响应。
<br>所有数据线均为单向信号，CS、SCK、SI 为主机向从机方向，SO 为从机向主机方向。在一些仅用作主机输出的场景中，可能会省略 SO 线。
<br>![SPI_Bus](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Bus.png)
<br><br><br>






# 2、通信时序
正如上所述，从机只需根据 CS 的电平决定是否进入通信状态。但根据时钟线和数据线不同的工作配合方式，SPI 有 4 种不同的通信模式。具体使用哪种模式，需要参考您所用器件的数据手册。
- 模式 0：SCK 低电平空闲或主机调整 SI 的电平，从机在 SCK 上升沿接收、下降沿发送数据
- 模式 1：SCK 低电平空闲或主机调整 SI 的电平，从机在 SCK 上升沿发送、下降沿接收数据
- 模式 2：SCK 高电平空闲或主机调整 SI 的电平，从机在 SCK 下降沿接收、上升沿发送数据
- 模式 3：SCK 高电平空闲或主机调整 SI 的电平，从机在 SCK 下降沿发送、上升沿接收数据

器件保持默认或经过设置后，只会遵循其中一种模式通信，不会更换，选其一使用即可；又依据不同模式对 SCK 电平的要求，需要不同的初始化函数。那么驱动一个 SPI 外设共需 1 个初始化函数和 3 种时序：起始信号、终止信号、交换字节。按器件的通信要求，将所需的时序组合起来，即可实现 SPI 通信。时序细节将在下文展开讲解。
<br>此处假设前缀“T_”表示主机发出的主机字节、“R_”表示主机接收的从机字节，同下。
<br>![SPI_Timing](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Timing.png)
<br>



## 2.1 初始化函数
*防止从机因不确定的电平状态误动作导致首次通信异常，同时将总线电平初始化为已知状态。*
<br>
<br>逻辑要求：使 CS 为**高电平**，并且：使用模式 0 或 1 则使 SCK 为**低电平**，使用模式 2 或 3 则使 SCK 为**高电平**。
<br>
使用模式 0 或 1 的代码形式参考：
```C++
void SPI_Init(void) {
    CS = 1;
    SCK = 0;
}
```
使用模式 2 或 3 的代码形式参考：
```C++
void SPI_Init(void) {
    CS = 1;
    SCK = 1;
}
```
<br>



## 2.2 起始信号
*告诉器件通信开始。*
<br>
<br>逻辑要求：使 CS 为**低电平**。
<br>
代码形式参考：
```C++
void SPI_Start(void) {
    CS = 0;
}
```
<br>



## 2.3 终止信号
*告诉器件通信结束。*
<br>
<br>逻辑要求：使 CS 为**高电平**。
<br>
代码形式参考：
```C++
void SPI_Stop(void) {
    CS = 1;
}
```
<br>



## 2.4 模式 0 交换字节（4 个模式选其一）
*向从机发送字节，同时接收来自从机的字节，同下。*
<br>
![SPI_Mode0](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode0.png)
<br>逻辑要求：SCK 产生**上升沿**之前，在 SI 上准备好要发送的字节的位，SCK 产生上升沿之后采样 SO 上的电平，然后使 SCK 产生**下降沿**。字节按从高位到低位的顺序发送，同样以从高到低的顺序接收，重复 8 次完成发送和接收字节，同下。（即 D7，D6，D5，...，D0）
<br>举例：假设主机发送的字节是 0xCA，其二进制为 1100 1010，则每次 SCK 产生下降沿时，SI 电平依次是 1、1、0、0、1、0、1、0；假设从机返回的字节是 0xA2，其二进制为 1010 0010，则每次 SCK 产生上升沿后，SO 电平依次是 1、0、1、0、0、0、1、0。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SI = T_D7|在 SI 上准备主机字节 D7 位的电平|
|2|SCK = 1|在 SCK 上产生一个上升沿|
|3|R_D7 = SO|采样 SO 的电平作为从机字节的 D7 位|
|4|SCK = 0|在 SCK 上产生一个下降沿|
|5|重复步骤 1-4，其中改 SI = T_D6、R_D6 = SO|发送主机字节 D6 位，同时接收从机字节 D6 位|
|6|重复步骤 1-4，其中改 SI = T_D5、R_D5 = SO|发送主机字节 D5 位，同时接收从机字节 D5 位|
|...|...|...|
|11|重复步骤 1-4，其中改 SI = T_D0、R_D0 = SO|发送主机字节 D0 位，同时接收从机字节 D0 位|

代码形式参考：
```C++
u8 SPI_TransferByte(u8 SendByte) {    // <-- u8 即 unsigned char，同下
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}   // <-- 这样写是因为用的是 (SendByte & 0x80) 的真值，即只看 (SendByte & 0x80) = 0 或 (SendByte & 0x80) ≠ 0。直接的 SI = SendByte & 0x80 写法不适配或可能导致引脚寄存器工作异常，尤其是 (SendByte & 0x80) > 1 时，同下
        else {SI = 0;}
        SCK = 1;
        if(SO) {ReceiveByte |= 0x01;}
        SCK = 0;
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.5 模式 1 交换字节（4 个模式选其一）
![SPI_Mode1](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode1.png)
<br>逻辑要求：使 SCK 产生**上升沿**，然后 SCK 产生**下降沿**之前，在 SI 上准备好要发送的字节的位，SCK 产生下降沿之后采样 SO 上的电平。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SCK = 1|在 SCK 上产生一个上升沿|
|2|SI = T_D7|在 SI 上准备主机字节 D7 位的电平|
|3|SCK = 0|在 SCK 上产生一个下降沿|
|4|R_D7 = SO|采样 SO 的电平作为从机字节的 D7 位|
|5|重复步骤 1-4，其中改 SI = T_D6、R_D6 = SO|发送主机字节 D6 位，同时接收从机字节 D6 位|
|6|重复步骤 1-4，其中改 SI = T_D5、R_D5 = SO|发送主机字节 D5 位，同时接收从机字节 D5 位|
|...|...|...|
|11|重复步骤 1-4，其中改 SI = T_D0、R_D0 = SO|发送主机字节 D0 位，同时接收从机字节 D0 位|

代码形式参考：
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        SCK = 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 0;
        if(SO) {ReceiveByte |= 0x01;}
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.6 模式 2 交换字节（4 个模式选其一）
![SPI_Mode2](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode2.png)
<br>逻辑要求：SCK 产生**下降沿之前**，在 SI 上准备好要发送的字节的位，SCK 产生**下降沿之后**采样 SO 上的电平，然后使 SCK 产生**上升沿**。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SI = T_D7|在 SI 上准备主机字节 D7 位的电平|
|2|SCK = 0|在 SCK 上产生一个下降沿|
|3|R_D7 = SO|采样 SO 的电平作为从机字节的 D7 位|
|4|SCK = 1|在 SCK 上产生一个上升沿|
|5|重复步骤 1-4，其中改 SI = T_D6、R_D6 = SO|发送主机字节 D6 位，同时接收从机字节 D6 位|
|6|重复步骤 1-4，其中改 SI = T_D5、R_D5 = SO|发送主机字节 D5 位，同时接收从机字节 D5 位|
|...|...|...|
|11|重复步骤 1-4，其中改 SI = T_D0、R_D0 = SO|发送主机字节 D0 位，同时接收从机字节 D0 位|

代码形式参考：
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 0;
        if(SO) {ReceiveByte |= 0x01;}
        SCK = 1;
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br>



## 2.7 模式 3 交换字节（4 个模式选其一）
![SPI_Mode3](https://github.com/INKPTR-STUDIO/MCU-BusBase/blob/main/Images/SPI_Mode3.png)
<br>逻辑要求：使 SCK 产生**下降沿**，然后 SCK 产生**上升沿之前**，在 SI 上准备好要发送的字节的位，SCK 产生**上升沿之后**采样 SO 上的电平。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SCK = 0|在 SCK 上产生一个下降沿|
|2|SI = T_D7|在 SI 上准备主机字节 D7 位的电平|
|3|SCK = 1|在 SCK 上产生一个上升沿|
|4|R_D7 = SO|采样 SO 的电平作为从机字节的 D7 位|
|5|重复步骤 1-4，其中改 SI = T_D6、R_D6 = SO|发送主机字节 D6 位，同时接收从机字节 D6 位|
|6|重复步骤 1-4，其中改 SI = T_D5、R_D5 = SO|发送主机字节 D5 位，同时接收从机字节 D5 位|
|...|...|...|
|11|重复步骤 1-4，其中改 SI = T_D0、R_D0 = SO|发送主机字节 D0 位，同时接收从机字节 D0 位|

代码形式参考：
```C++
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        SCK = 0;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SCK = 1;
        if(SO) {ReceiveByte |= 0x01;}
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}
```
<br><br><br>






# 3、补充说明与最终参考代码
## 3.1 关于模式 0 与 1、模式 2 与 3
相比于标题“2.x 模式 x 交换字节（4 个模式选其一）”部分各模式逻辑要求的描述，标题“2、通信时序”部分内容中，时钟线和数据线配合方式所述的要求看起来似乎更宽松，即可能会产生一个误解：把 SCK 上下边沿合并看做一个脉冲，脉冲前准备发送，脉冲后采样接收即可。这在主机速度很低时可能是成功的，但通信速度稍高就极易出现异常：
<br>
<br>不论是主机还是从机，SCK 的边沿只作为触发数据线电平采样的标志，并非在这一时刻就完成采样，实际采样到的电平来自 SCK 边沿的极小一段时间之后。即可以认为实际采样的时刻，是在 SCK 某一边沿后的电平稳定期间，而非边沿处。（“SCK 边沿的极小一段时间之后”：这个时间是相对固定的，由器件结构决定）
<br>不同模式下，主从双方的有效数据输出的时间段，重叠分布在 SCK 的其中一种电平期间：
- 模式 0：SCK 低电平期间允许 SI、SO 改变电平（此模式 SCK 低电平空闲，空闲时可改）
- 模式 1：SCK 高电平期间允许 SI、SO 改变电平（此模式 SCK 低电平空闲，空闲时不可改）
- 模式 2：SCK 高电平期间允许 SI、SO 改变电平（此模式 SCK 高电平空闲，空闲时可改）
- 模式 3：SCK 低电平期间允许 SI、SO 改变电平（此模式 SCK 高电平空闲，空闲时不可改）

主机在编辑 SI 时，从机也在编辑 SO，把 SO 采样代码和 SI 编辑代码放一起，可能因为从机刚编辑后 SO 电平尚未稳定、甚至从机还未对 SO 编辑就采样，导致采样到不正确的电平。因为主机速度很低时，采样时刻恰好到了 SO 电平稳定期，所以有可能成功，但 SPI 协议的重要优势之一就是通信速率，这个牺牲是不值得的。
<br>
<br>因此，标题“2、通信时序”部分的描述只能作为模式匹配的辅助。（有的数据手册不会指明器件所用的是模式几，而是说它在哪个边沿如何工作）
<br>



## 3.2 通信速度适配优化
如“3.1 关于模式 0 与 1、模式 2 与 3”所述，受器件接口和布线等原因附带的 RC 特性影响，总线上的电平被主机或从机翻转后，需要一定的时间才能稳定；从机的逻辑处理单元对于信号的处理速度也有上限，两者是限制 SPI 通信速度的主要因素。（除非主机自身的电平操作速度已经足够慢。若主机速度足够慢，本节所述优化可忽略）
<br>为了使通信在足够快速的同时保证可靠性，在布线尽量短、保护总线尽量不受电磁干扰、依据总线挂载情况选择合适上拉电阻的同时，还需要从单片机程序中主动限制单片机的引脚操作速度。
<br>以绝大多数器件支持的 1MHz 为例，可以在每次编辑引脚电平的代码前后各延时 250ns。那么代码形式参考中可以封装引脚函数：
```C++
void SPI_Delay250ns(void) {
    // 250ns 延时代码
}

void SPI_EditSCK(u8 Dat) {
    SPI_Delay250ns();
    if(Dat) {SCK = 1;}
    else {SCK = 0;}
    SPI_Delay250ns();
}
```
若单片机引脚翻转速度已低于 1MHz，可省略此延时。
<br>
<br>此外，在起始和终止信号中，从机从 CS 拉低到完全进入通信状态需要时间，从 CS 拉高到完全退出通信状态也需要时间，那么需要在**拉低 CS 之后**和**拉高 CS 之前**加入延时。具体延时要求不同器件各有长短，详情请查阅您所用器件的数据手册。
<br>以绝大多数器件支持的 1us 为例，可以封装引脚函数：
```C++
void SPI_Delay1us(void) {
    // 1us 延时代码
}

void SPI_Start(void) {
    CS = 0;
    SPI_Delay1us();
}

void SPI_Stop(void) {
    SPI_Delay1us();
    CS = 1;
}
```
<br>



## 3.3 最终参考代码
```C++
#ifndef SPI_H
#define SPI_H

void SPI_Init(void);                // 初始化函数
void SPI_Start(void);               // 起始信号
void SPI_Stop(void);                // 终止信号
u8 SPI_TransferByte(u8 SendByte);   // 交换字节

#endif
```
```C++
// 延时函数
static void SPI_Delay1us(void) {
    // 1us 延时代码
}
static void SPI_Delay250ns(void) {
    // 250ns 延时代码
}



// SCK 封装函数
static void SPI_EditSCK(u8 Dat) {
    SPI_Delay250ns();
    if(Dat) {SCK = 1;}
    else {SCK = 0;}
    SPI_Delay250ns();
}



// 初始化函数
void SPI_Init(void) {
    CS = 1;
    SCK = 0;    // <-- 模式 0 或 1 时为 SCK = 0；模式 2 或 3 时为 SCK = 1
}

// 起始信号
void SPI_Start(void) {
    CS = 0;
    SPI_Delay1us();
}

// 终止信号
void SPI_Stop(void) {
    SPI_Delay1us();
    CS = 1;
}

// 模式 0 交换字节
u8 SPI_TransferByte(u8 SendByte) {
    u8 i, ReceiveByte=0x00;

    for(i = 0 ; i < 8 ; i++) {
        ReceiveByte = ReceiveByte << 1;
        if(SendByte & 0x80) {SI = 1;}
        else {SI = 0;}
        SPI_EditSCK(1);
        if(SO) {ReceiveByte |= 0x01;}
        SPI_EditSCK(0);
        SendByte = SendByte << 1;
    }
    return ReceiveByte;
}

// 模式 1 交换字节
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        SPI_EditSCK(1);
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(0);
//        if(SO) {ReceiveByte |= 0x01;}
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}

// 模式 2 交换字节
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(0);
//        if(SO) {ReceiveByte |= 0x01;}
//        SPI_EditSCK(1);
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}

//模式 3 交换字节
//u8 SPI_TransferByte(u8 SendByte) {
//    u8 i, ReceiveByte=0x00;
//
//    for(i = 0 ; i < 8 ; i++) {
//        ReceiveByte = ReceiveByte << 1;
//        SPI_EditSCK(0);
//        if(SendByte & 0x80) {SI = 1;}
//        else {SI = 0;}
//        SPI_EditSCK(1);
//        if(SO) {ReceiveByte |= 0x01;}
//        SendByte = SendByte << 1;
//    }
//    return ReceiveByte;
//}
```
