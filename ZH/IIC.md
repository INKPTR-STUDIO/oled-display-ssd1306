# 目录
1. 物理总线
2. 通信时序
3. 补充说明与整理
<br><br><br>






# 1、物理总线
IIC 数据线有两条：
<br>（1）时钟线 SCL
<br>（2）数据线 SDA
<br>总线通过并联的形式挂载多个从机。对于 SCL，在多数应用中，从机不主动控制 SCL，但 IIC 规范允许从机将 SCL 拉低让主机等待。本文暂不讨论，则从机始终为高阻态，只有主机通过发送低电平脉冲向各个从机提供同步信号；对于 SDA，发送的一方可选择拉低电平或以高阻态释放该线为高电平，接收的一方则保持高阻态以读取电平状态。因此 SCL 上只有主机发出的单向信号，SDA 上为双向信号。
<br>SCL、SDA 均为开漏结构，建议在这两根数据线上分别加入上拉电阻，稳定两个引脚被释放时的高电平，同时以合适的阻值限制流入芯片的电流大小。（3.3V-5V供电的一般使用场景中，推荐使用4.7kΩ）
<br>![IIC_Bus](https://github.com/INKPTR-STUDIO/oled-display-ssd1306/blob/main/Images/IIC_Bus.png)
<br><br><br>






# 2、通信时序
从电平逻辑上概括的话，主从双方器件在 SCL 为高电平时对 SDA 采样，然后依情况解析成数据或指令：
- SCL 高电平期间，SDA 不变视为传数据。SDA 为高电平即传递 1，SDA 为低电平即传递 0。
- SCL 高电平期间，SDA 出现电平翻转视为起始或终止指令。SDA 出现下降沿即起始指令，SDA 出现上升沿即终止指令。（其实此处的起始和终止指令，就是下文所述的起始和终止信号时序）

以上几种电平逻辑又组合成 6 种时序：起始信号、终止信号、发送应答、接收应答、发送字节、接收字节。按器件的通信要求，将所需的时序组合起来，即可实现 IIC 通信。时序细节将在下文展开讲解。
<br>![IIC_Timing](https://github.com/INKPTR-STUDIO/oled-display-ssd1306/blob/main/Images/IIC_Timing.png)
<br>



## 2.1 起始信号
*告诉器件通信开始。*
<br>
<br>逻辑要求：SCL 为**高电平**时，SDA 出现**下降沿**。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SDA = 1|使 SDA 为产生下降沿做好准备|
|2|SCL = 1|为 SCL 高电平时检测 SDA 下降沿做好准备|
|3|SDA = 0|使 SDA 产生下降沿|
|4|SCL = 0|使 SCL 进入工作状态|

代码形式参考：
```C++
void IIC_Start(void) {
    SDA = 1;
    SCL = 1;
    SDA = 0;
    SCL = 0;
}
```
<br>



## 2.2 终止信号
*告诉器件本次通信已结束。在上电后，也推荐先发送终止信号，防止从机因不确定的电平状态误动作导致首次通信异常，同时将总线电平初始化为已知状态。*
<br>
<br>逻辑要求：SCL 为**高电平**时，SDA 出现**上升沿**。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SCL = 0|避免 SCL 为高电平时 SDA 误触发|
|2|SDA = 0|使 SDA 为产生上升沿做好准备|
|3|SCL = 1|为检测 SDA 上升沿做好准备|
|4|SDA = 1|使 SDA 产生上升沿|

代码形式参考：
```C++
void IIC_Stop(void) {
    SCL = 0;
    SDA = 0;
    SCL = 1;
    SDA = 1;
}
```
<br>



## 2.3 发送应答
*通常在接收字节时序后使用，可用于控制从机接下来的操作。（例如从存储芯片读取数据时，接收一个字节后发送应答，通常表示接下来要继续从存储芯片接收下一个字节；发送不应答，则表示不再继续接收下一个字节。不同芯片对应答信号的响应方式可能不同，详情请参考你使用的芯片的数据手册）*
<br>
<br>逻辑要求：在 SDA 上准备好要发送的应答的电平后，在 SCL 上产生一个**高电平脉冲**。
<br>应答：**ACK = 0 时认为应答**，反之不应答，同下。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SDA = ACK|在 SDA 上准备应答的电平|
|2|SCL = 1|在 SCL 上产生一个高电平脉冲|
|3|SCL = 0|结束 SCL 上的高电平脉冲|

代码形式参考：
```C++
void IIC_SendACK(u8 ACK) {    // <-- u8 即 unsigned char，同下
    if(ACK) {SDA = 1;}  // <-- 这样写是因为用的是 ACK 的真值，即只看 ACK = 0 或 ACK ≠ 0。直接的 SDA = ACK 写法不适配或可能导致引脚寄存器工作异常，尤其是 ACK > 1 时，同下
    else {SDA = 0;}
    SCL = 1;
    SCL = 0;
}
```
<br>



## 2.4 接收应答
*通常在发送字节时序后使用，可用于判断从机内部是否已经完成相应工作，并准备好继续通信。（例如向存储芯片写入数据时，主机发送数据后，芯片要花时间控制存储单元进行充放电，将数据存储其中。在此过程中芯片一般不应答，等到工作完成后发送应答，然后才能继续接下来的通信。不同芯片的应答信号可能不同，详情请参考你使用的芯片的数据手册）*
<br>
<br>逻辑要求：先确保 SDA 为可读状态，**使 SCL 为高电平时读取 SDA 的电平**，然后恢复 SCL 为低电平。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SDA = 1|释放总线，使 SDA 处于可读状态|
|2|SCL = 1|告诉从机，主机开始采样 SDA 的电平|
|3|ACK = SDA|采样 SDA 的电平作为 ACK|
|4|SCL = 0|告诉从机，主机对当前 SDA 的电平采样已完成|

代码形式参考：
```C++
u8 IIC_ReceiveACK(void) {
    u8 ACK;

    SDA = 1;
    SCL = 1;
    ACK = SDA;
    SCL = 0;
    return ACK;
}
```
<br>



## 2.5 发送字节
逻辑要求：从高到低，在 SDA 上准备好要发送的字节的位，然后在 SCL 上产生一个**高电平脉冲**，重复 8 次。（即 D7，D6，D5，...，D0）
<br>举例：假设要发送的字节是 0xCA，其二进制为 1100 1010，则每次 SCL 产生高电平脉冲时，SDA 电平依次是 1、1、0、0、1、0、1、0。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SDA = D7|在 SDA 上准备字节 D7 位的电平|
|2|SCL = 1|在 SCL 上产生一个高电平脉冲|
|3|SCL = 0|结束 SCL 上的高电平脉冲|
|4|重复步骤 1-3，其中改 SDA = D6|发送字节 D6 位|
|5|重复步骤 1-3，其中改 SDA = D5|发送字节 D5 位|
|...|...|...|
|10|重复步骤 1-3，其中改 SDA = D0|发送字节 D0 位|

代码形式参考：
```C++
void IIC_SendByte(u8 Byte) {
    u8 i;

    for(i = 0 ; i < 8 ; i++) {
        if(Byte & 0x80) {SDA = 1;}
        else {SDA = 0;}
        SCL = 1;
        SCL = 0;
        Byte = Byte << 1;
    }
}
```
<br>



## 2.6 接收字节
逻辑要求：先确保 SDA 为可读状态，使 SCL 为高电平时读取 SDA 的电平，然后恢复 SCL 为低电平，读取到的电平依次作为字节从高到低的位，重复 8 次，得到接收的字节。

|步骤|电平操作|目的|
|:-:|:-:|:-|
|1|SDA = 1|释放总线，使 SDA 处于可读状态|
|2|SCL = 1|告诉从机，主机开始采样 SDA 的电平|
|3|D7 = SDA|采样 SDA 的电平作为接收字节的 D7 位|
|4|SCL = 0|告诉从机，主机对当前 SDA 的电平采样已完成|
|5|重复步骤 2-4，其中改 D6 = SDA|接收字节 D6 位|
|6|重复步骤 2-4，其中改 D5 = SDA|接收字节 D5 位|
|...|...|...|
|11|重复步骤 2-4，其中改 D0 = SDA|接收字节 D0 位|

代码形式参考：
```C++
u8 IIC_ReceiveByte(void) {
    u8 i, Byte=0x00;

    SDA = 1;
    for(i = 0 ; i < 8 ; i++) {
        Byte = Byte << 1;
        SCL = 1;
        if(SDA) {Byte |= 0x01;}
        SCL = 0;
    }
    return Byte;
}
```
<br><br><br>






# 3、补充说明与整理
## 3.1 部分时序中 SCL 的初始状态说明
以上示例中：发送应答、接收应答、发送字节、接收字节，这 4 个时序在开始时没有将 SCL 的电平初始化为 0，是因为这 4 个时序在实际使用时，只会紧挨在**除终止信号时序外**的其他时序之后，而这几个时序最后都将 SCL 的电平恢复到了低电平，所以不影响。
<br>



## 3.2 接收应答超时优化
这是主机最被动的时序，连续通信须等待从机内部完成工作后给出响应，否则从机无法继续；选择等待从机响应的话，假如从机故障或实际上并没有连接，主机进程又会在漫长的等待中卡死。
<br>可以设定一个最大等待时间，如果从机在超时前响应，那么通信继续；否则不再继续。这里的超时判定通过延时+轮询实现，那么可以改善相应的代码形式参考：
```C++
u8 IIC_ReceiveACK(void) {
    u8 ACK, TimeOut=200;    // <-- 此处 TimeOut 的初始值可以根据需要更改

    SDA = 1;
    SCL = 1;
    while(TimeOut--) {
        ACK = SDA;
        if(!ACK) {break;}   // <-- 这里以应答（ACK = 0）作为从机就绪的标志

        IIC_Delay10us();    // <-- 此处可以修改延时函数，但延时过长会严重影响通信速度。以本代码为例，假设此处延时 10us，那么代码实际上会每隔 10us 检查一次 SDA，若有应答则提前跳出轮询的循环，函数最后会返回应答（返回 0）；否则轮询循环会在 200 次轮询后结束，超时时间一共为 200 * 10us = 2ms，函数最后会返回不应答（返回 1）

    }
    SCL = 0;
    return ACK;
}
```
<br>



## 3.3 通信速度适配优化
受器件接口和布线等原因附带的 RC 特性影响，总线上的电平被主机或从机翻转后，需要一定的时间才能稳定，从机的逻辑处理单元对于信号的处理速度也有上限，两者是限制 IIC 通信速度的主要因素。
<br>为了使通信在足够快速的同时保证可靠性，在布线尽量短、保护总线尽量不受电磁干扰、依据总线挂载情况选择合适上拉电阻的同时，还需要从单片机程序中主动限制单片机的引脚操作速度。
<br>以绝大多数器件支持的标准模式 100kHz 为例，可以在每次编辑引脚电平的代码后延时 10us。那么代码形式参考中可以封装引脚函数：
```C++
void IIC_Delay10us(void) {
    // 10us 延时代码
}

void IIC_EditSCL(u8 Dat) {
    if(Dat) {SCL = 1;}
    else {SCL = 0;}
    IIC_Delay10us();
}

void IIC_EditSDA(u8 Dat) {
    if(Dat) {SDA = 1;}
    else {SDA = 0;}
    IIC_Delay10us();
}
```
<br>



## 3.4 最终代码形式参考
```C++
// 电平翻转速度控制延时函数
void IIC_Delay10us(void) {
    // 10us 延时代码
}



// 翻转速度控制引脚封装
void IIC_EditSCL(u8 Dat) {
    if(Dat) {SCL = 1;}
    else {SCL = 0;}
    IIC_Delay10us();
}
void IIC_EditSDA(u8 Dat) {
    if(Dat) {SDA = 1;}
    else {SDA = 0;}
    IIC_Delay10us();
}



// 起始信号
void IIC_Start(void) {
    IIC_EditSDA(1);
    IIC_EditSCL(1);
    IIC_EditSDA(0);
    IIC_EditSCL(0);
}

// 终止信号
void IIC_Stop(void) {
    IIC_EditSCL(0);
    IIC_EditSDA(0);
    IIC_EditSCL(1);
    IIC_EditSDA(1);
}

// 发送应答
void IIC_SendACK(u8 ACK) {
    IIC_EditSDA(ACK);
    IIC_EditSCL(1);
    IIC_EditSCL(0);
}

// 接收应答
u8 IIC_ReceiveACK(void) {
    u8 ACK, TimeOut=200;

    IIC_EditSDA(1);
    IIC_EditSCL(1);
    while(TimeOut--) {
        ACK = SDA;
        if(!ACK) {break;}
        IIC_Delay10us();
    }
    IIC_EditSCL(0);
    return ACK;
}

// 发送字节
void IIC_SendByte(u8 Byte) {
    u8 i;

    for(i = 0 ; i < 8 ; i++) {
        IIC_EditSDA(Byte & 0x80);
        IIC_EditSCL(1);
        IIC_EditSCL(0);
        Byte = Byte << 1;
    }
}

// 接收字节
u8 IIC_ReceiveByte(void) {
    u8 i, Byte=0x00;

    IIC_EditSDA(1);
    for(i = 0 ; i < 8 ; i++) {
        Byte = Byte << 1;
        IIC_EditSCL(1);
        if(SDA) {Byte |= 0x01;}
        IIC_EditSCL(0);
    }
    return Byte;
}
```
