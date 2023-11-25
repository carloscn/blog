# DSP-F2812的串行通信接口SCI

今天是五一假期的最后一天，虽然是以清明节的名义放的假，但是觉得放假还不如不放假，放假了学校轻易的给停电，影响了进度。关于DSP我们还差两个外设模块没有整理，现在正在筹集DSP最小系统的各个部件，今天还算是有时间，把SCI模块总结下来！

-----------2014.5.4，沈阳化工大学，5号楼


## 1.SCI模块的概述

SCI（Serial Communication Interface）的简称，就是我们说的串口。但是串口有两种形式，一种是同步的形式，另一种是异步的形式。我们的SCI模块就是异步的形式，就是不需要同时发送或者同时接受。在任何时刻都能进行发送或者接收。在结构上SCI具备了两条线，一条是SCI的发送线，一条是SCI的接收线，可以理解为，SCI的上传和下载功都是分开的，互不干扰。X2812xSCI模块采用NRZ标准格式的异步外围设备，只要遵循这样标准的串口都能进行数据通信。当初我理解串口的时候有个误区，就是串口就是RS232的那种接口。其实不是的，串行数据可以设置成人和的接口，只要是有支持那样的芯片，我们可以用RS232，那么中间需要的芯片是Xlini公司的MAX3232芯片进行中转，我们同样也可以做成通用USB，只是中间的芯片需要更换一下。

X2812x同EV一样，都集成了两个分立而不分开的模块，SCIA,SCIB，还有SCI是需要存储的，即一定要有一个16级深度的FIFO队列。记得ADC中存在一个2级深度的FIFO堆栈。这个SCI拥有的是一个16级深度的FIFO堆栈，而且他们有自己的中断。SCI的工作方式，有三种模式，单工，半双工，全双工模式。

>4.6 Serial Communications Interface (SCI) Module
The F281x and C281x devices include two serial communications interface (SCI) modules. The SCI modules support digital communications between the CPU and other asynchronous peripherals that use the standard non-return-to-zero (NRZ) format. The SCI receiver and transmitter are double-buffered, and each has its own separate enable and interrupt bits. Both can be operated independently or simultaneously in the full-duplex mode. To ensure data integrity, the SCI checks received data for break detection, parity, overrun, and framing errors. The bit rate is programmable to over 65000 different speeds through a 16-bit 

## 2. SCI模块的特点

![image-20221203104523802](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104523802.png)

![image-20221203104532019](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104532019.png)

![image-20221203104538324](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104538324.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104538324.png)

## 3. SCI模块的工作的相关器件

我们主要看其全双工原理。

发送相关的功能单元：SCI存在着SCITXBUF数据发送缓冲寄存器，要发送的数据往这里面存。TXSHF发送移位寄存器，这个寄存器连接的直接就是引脚SCITXD引脚，发送是一位一位的发，我们在SICTXD引脚上有电平变化那个就是数据，它的作用就是把发送缓冲器里面的数据一位一位的运送到SCITXD引脚上。

接收相关的功能单元：相应地，SCI在发送功能上也存在着RXSHF接收移位寄存器，从SCIRXD引脚一位一位的移入SCIRXBUF寄存器中，接受缓冲寄存器，然后CPU就能处理了。

### 3.1 SCI发送和接收的原理

![image-20221203104622426](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104622426.png)

我们从引脚开始。假若我们把ARM发来的数据接到了SCIRXD上了，现在我们CPU想处理ARM的数据，怎么进入CPU呢？信号往里面走，走到了RXSHF Resgister接收移位寄存器，等它积累满了，我设定SCICTL1的RXENA使能，移位寄存器往BUF中存取数据，存满了RXRDY和中断标志位都会被置位，CPU就会检索，哦，RXRDY置位了，我可以去取数据了。CPU读取完了数据S那个RD位还会被清零。我们还能使能FIFO功能，减少CPU的开销，他能从缓冲器中暂时存如那些数值。我们最好是开启FIFO功能。

#### 3.1.1 **SCI****发送的原理**

![image-20221203104719861](https://raw.githubusercontent.com/carloscn/images/main/typoraimage-20221203104719861.png)

发送原理我们就得从内部说了。CPU通过程序写入SCITXBUF寄存器，这时候发送器不再为空，发送缓冲寄存器就就绪标志位TXRDY被清除。如果这时候时候使能了FIFO功能，那么TXSHF将会把数据移入FIFO中。TXSHF也将会直接从FIFO中获取发送的数据，而不是从BUF中。正常的数据从BUF中经过TXSHF的搬移到引脚SCITXD中。我们就可感受到数据了。

#### 3.1.2 **SCI**的中断

SCI可以产生两种中断，一种是接受中断RXINT和发送中断TXINT。

SCI的中断发出有两种，但是分别也有两种触发方式，分别是标准模式下，和FIFO模式下。

在标准模式下，RXSHF将接收到的数据写入SCIRXBUF中，等待CPU来读取的时候接受缓冲寄存器RXRDY被置位了，表示已经接收好了一个数据，同时产生一个接受中断的请求信号。

标准模式下，当缓冲寄存器的就绪标志位TXRDY被置位，表示CPU可以将下一个发送的数据写到SCITXBUF中，同时产生一个发送中断TXINT信号的请求。我们注意一下，发送模式，也就是发数据，当SCITXBUF空了的时候，也就是发送完毕才会产生一个中断。

FIFO模式下可以这么说，当FIFO满了的时候发送中断标志位置位。当FIFO空了的时候接受中断标志位置位。
