如果我们想获取或者修改系统的任何信息(例如，闪烁LED，检测到按钮的按下或与某种总线上的外设进行通信)，我们将不得不深入了解外设及其“内存映射寄存器”。

现在已经存在不少访问外设的crate，他们可以大致进行如下分类：
* 处理器架构相关Crate (Micro-architecture Crate) - 这种crate比较通用, 可处理CPU相关的通用例程以及一些通用外设。例如，[cortex-mcrate](https://crates.io/crates/cortex-m)为您提供了启用和禁用中断的功能，这些功能对于所有基于Cortex-M的CPU都是相同的。它还使您可以访问所有基于Cortex-M的微控制器附带的时钟外设(SysTick)。
* 外设相关Crate(PAC) 这种crate实际上是对特定CPU型号的内存映射寄存器的一个简单封装。例如，[tm4c123x](https://crates.io/crates/tm4c123x)这个crate是对德州仪器(TI)Tiva-C TM4C123系列CPU的封装，[stm32f30x](https://crates.io/crates/stm32f30x)这个crate是对ST-Micro STM32F30x系列CPU的封装。借助这些crate，您可以按照CPU参考手册中给出的每个外设的操作说明直接与寄存器进行交互。
* HAL crate - 这些crate通过实现[embedded-hal](https://crates.io/crates/embedded-hal)中定义的一些常见Trait，来提供更友好的处理器相关API。例如，此crate可能提供一个`Serial`结构体，该结构体提供一个构造函数来配置一组GPIO引脚和波特率，并提供某种`write_byte`函数来发送数据。
* 开发板相关crate - 通过预先配置各种外设和GPIO引脚以适合特定的开发板，例如针对TM32F3DISCOVERY开发板的[F3](https://crates.io/crates/f3)crate，这些crate相比HAL类crate更易用。

我们是研究操作系统和Cortex-M的架构的贴合部分，以上封装好的外设并不是在我们研究范围之内。处理器架构相关Crate (Micro-architecture Crate) 可能需要去调用一些支持。