---

　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家介绍的是**i.MXRT下使能DMA链式传输可达到SPI从设备接收速率上限50Mbps**。

　　最近痞子衡在帮一个 RT600 的 AR 眼镜客户优化 SPI 从设备接收数据的速率，我们知道 SPI 从设备接收数据方法一般有三种：1) 轮询模式，2) 中断模式，3) DMA 模式。前两种模式都会受到 CPU 性能的限制，而 DMA 模式则可以最大程度地降低 CPU 负载，提高数据传输效率以及速率。

　　然而使用 DMA 传输也会有潜在问题，单次 DMA 传输数据长度有上限（受限于 DMA 通道缓冲区长度），如果在一次 DMA 传输结束之后才开始手动启动下一次 DMA 传输，中间的延迟则有可能导致漏收数据，这时我们就需要使用 DMA 链式传输（Linked Transfer）来解决潜在漏收数据问题。这便是今天我们要讨论的话题：

> * Note1：本文方法主要针对 RT500/600 上的 DMA（又称LPC\_DMA），其来自于恩智浦 LPC 系列。
> * Note2：RT700/RT4digits 上的 eDMA 与 LPC\_DMA 完全不同，其来自于原飞思卡尔 Kinetis 系列（KL25 DMA是第一代，K60 eDMA算第二代）。

### 一、Flexcomm SPI速率

　　在讨论这个话题之前，我们先来看一下 RT500/600 上的 SPI 外设本身速率。我们知道 RT3digits 上有一个非常神奇的外设 Flexcomm（与之对应的是 RT4digits 上的 FlexIO），这个外设可以按照用户需求被配置成 USART, SPI, I2C 或者 I2S 外设功能之一。

　　RT3digits 内部一般会有多个 Flexcomm，而其本身又分为普通和专用两种类型：普通 Flexcomm 内部结构复杂，因为外设功能配置灵活性而稍稍放弃了一点传输性能；专用 Flexcomm 则限定用于特定外设功能，放弃了灵活性，但是传输性能更高。下面是芯片数据手册里找到的 SPI 速率：

| 芯片系列 | 普通SPI Master/Slave:TX/RX - 25Mbps | 高速SPI Master:TX/RX - 50Mbps Slave:RX - 50Mbps, TX - 35Mbps |
| --- | --- | --- |
| RT500 | Flexcomm 0-8, 10-12 | 专用 Flexcomm 14,16 |
| RT600 | Flexcomm 0-7 | 专用 Flexcomm 14 |

### 二、为什么必须要用DMA？

　　下图是 Flexcomm 模块简图，其内部对于收发均配置了一个深度为 8 entries 的 FIFO（对于 SPI frame 长度硬件上能直接支持 4-16 bits，所以这里 entries 是以 frame 长度为单位。如果配置为最常用的 8bits frame，那 FIFO 就能缓存 8bytes 数据），有一定的数据缓存能力，不至于因 CPU 响应不及而立即漏数据。

![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRTxxx_FCSPI_RX_DMA_block_diagram_fc.png)

　　文章开头说了 SPI 从设备接收数据方法有三种，我们来一一具体分析：

> * 轮询方式： CPU 每隔一段时间就来读一次 SPI RX FIFO 状态寄存器，一旦有数据就立刻取走，这种方法能够达到速率上限 50Mbps，但是代价是 275/300MHz 的 CPU 需要付出相当大的负载地在这轮询 SPI RX FIFO 寄存器状态，这对于应用程序设计太不友好，稍稍不慎就会漏数据，显得匆匆忙忙，一般不会这么用。
> * 中断方式： 预先设置一下 SPI RX FIFO 的 level trigger point（1-8 entries），当 RX FIFO 中数据达到这个水平时就触发中断，在 ISR 里把数据取走。这种方法可以降低 CPU 负载，但是由于 Cortex-M33 中断延迟较大，再加上 ISR 代码执行时间消耗导致可能达不到速率上限 50Mbps（理想情况下需要 FIFO 触发设 4 entries，然后一次 ISR 读取 4 个 entries 数据），ISR 加点代码都需要谨慎，显得小心翼翼，因此也不太推荐。
> * DMA方式： 利用 DMA 来自动搬运 SPI RX FIFO 中的数据到指定 Buffer 中（最长 1024 个 SPI frame），完全不需要 CPU 参与，这种方式可以最大程度地降低 CPU 负载，并且可以轻松达到 50Mbps 的速率上限，此时才算是游刃有余，唯一需要注意的是，单次 DMA 传输长度有上限，需要使用 DMA 链式传输来避免数据漏收。

### 三、LPC\_DMA功能介绍

　　看起来这种情况下 DMA 是必须要用了，那我们就来简单了解一下 LPC\_DMA 功能，下图是其原理框图。首先我们要知道一个 DMA 会包含多个 channel（RT600 上是 33 个，RT500上是 37 个），各 channel 均可以独立工作（当然也可以合作）。对于每个 channel 独立工作，我们需要重点了解四个最基本的概念：

> * src/dest data： DMA 本质上是数据搬运，从源地址到目的地址，源/目的地址既可以是一般内存，也可以是外设寄存器，因此从类型上就分为：内存->内存、内存->外设、外设->内存、外设->外设（这种类型一般需要特殊设计，RT3digits 不直接支持）。
> * XFER Count： 一次 DMA 传输搬运的数据量，是可配置的，RT500/600 里上限均是 1024 个 units（每个 unit 长度 8/16/32bits 可配）。
> * DMA requests： 指 DMA 数据搬运涉及外设寄存器时，源/目的地址对应哪种外设，这里每个 channel 设计是定死的，比如 RT600 上 DMA0 channel 26 仅能接收 Flexcomm 14 SPI 的 RX 请求。
> * DMA triggers： 指触发 DMA 开始工作的条件，每个 channel 相同，除了最基本的软件直接触发之外，均可以配置不同硬件触发条件（RT600 上是 25 个，RT500 上是 27 个），这些硬件触发条件包含各种外设中断、其他 DMA channel trigger output 等。

![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRTxxx_FCSPI_RX_DMA_block_diagram_dma.png)

　　了解了 DMA 基本概念，我们还需要再进一步了解下 DMA 数据传输的几种方式（这里均针对单 channel 而言）：

> * Single buffer： 这是最基础的单次 DMA 传输方式（一般用于内存到内存），并且源/目的地址均是线性不间隔增长。
> * Linked transfers： 这是将多次 DMA 传输串起来（数量不限，仅受限于内存容量，即只要你有足够的 Buffer），一次传输结束自动转到下一次传输。这里包含了一个非常常用的场景：当仅串 2 次 DMA 传输时（利用双 Buffer 循环工作），这叫乒乓传输（Ping-pong Transfer）。
> * Interleaved transfers： 这是一种特殊的 DMA 传输方式，可建立在 Linked transfers 基础之上，但是源/目标地址增长可按一定步长，适用于预处理音频/图像数据场合（比如二维图像数据里仅提取每行/列数据，多通道音频数据里仅提取单通道数据）。

　　有了上面的铺垫，现在我们很自然地想到可以把多个 DMA channel 串起来一起工作，channel A 完成触发 channel B 继续工作，那就是所谓的 Channel chaining 模式。此外，虽然原则上多个 channel 可以并行工作，但是如果涉及到总线带宽限制或者内存访问冲突，我们还可以为其设置响应优先级，RT500/600 上一共支持 8 档优先级（每次仲裁时始终从最高优先级 channel 开始检查，并选择第一个处于激活状态的 channel 进行服务）。

### 四、使能SPI DMA链式传输方法

　　关于 DMA 本身的链式传输示例可直接参考 \SDK\_25\_09\_00\_EVK-MIMXRT685\boards\evkmimxrt685\driver\_examples\dma\linked\_transfer 例程，这个例程设计非常简单清晰，代码过程一目了然，实现的功能就是将 s\_srcBuffer1 和 s\_srcBuffer2 数据往 s\_destBuffer 里搬，如果 DMA\_Callback 里不做任何处理，那么将会一直循环搬移。

```
#include "fsl_dma.h"
static dma_handle_t s_DMA_Handle;
SDK_ALIGN(dma_descriptor_t s_dma_table[2], 16U);
SDK_ALIGN(uint32_t s_srcBuffer1[4], sizeof(uint32_t));
SDK_ALIGN(uint32_t s_srcBuffer2[4], sizeof(uint32_t));
SDK_ALIGN(uint32_t s_destBuffer[8], sizeof(uint32_t));
// 一次 DMA 传输结束用户回调（对应一个 DMA 传输描述符里的工作）
void DMA_Callback(dma_handle_t *handle, void *param, bool transferDone, uint32_t tcds)
{
    // Do someting
}
// 初始化 DMA0 通道 0
DMA_Init(DMA0);
DMA_CreateHandle(&s_DMA_Handle, DMA0, 0);
DMA_EnableChannel(DMA0, 0);
DMA_SetCallback(&s_DMA_Handle, DMA_Callback, NULL);
// DMA 传输属性配置（uint大小为4bytes，源和目标地址均按1个unit自增，一次传输16bytes，使能reload特性和INTB）
uint32_t xferCfg = DMA_SetChannelXferConfig(true, false, false, true, 4U, kDMA_AddressInterleave1xWidth, kDMA_AddressInterleave1xWidth, 16U);
// 初始化两个 DMA 传输描述符，并且将其互相链接
DMA_SetupDescriptor(&(s_dma_table[0]), xferCfg, s_srcBuffer1, &s_destBuffer[0], &(s_dma_table[1]));
DMA_SetupDescriptor(&(s_dma_table[1]), xferCfg, s_srcBuffer2, &s_destBuffer[4], &(s_dma_table[0]));
// 将第一个 DMA 传输描述符赋给 DMA0 通道 0
DMA_SubmitChannelDescriptor(&s_DMA_Handle, &(s_dma_table[0]));
// 软件触发 DMA0 通道 0 开始工作
DMA_StartTransfer(&s_DMA_Handle);
```

　　但是很遗憾的是 SDK 中并没有现成的 SPI RX DMA 链式传输例程，在仅有的 \SDK\_25\_09\_00\_EVK-MIMXRT685\boards\evkmimxrt685\driver\_examples\spi\dma\_b2b\_transfer\slave\ 例程里，它也仅是启动单次传输，查看其代码，我们发现根本原因是 fsl\_spi\_dma.c 驱动（V2.2.2）里从设计上就不支持链式传输。

```
SPI_SlaveTransferDMA() -> SPI_MasterTransferDMA() ->
    SPI_TransferSetupRxContextDMA(handle, xfer);
    SPI_EnableRxDMA(base, true);
    SPI_TransferSubmitNextRxDMA(base, handle);  // 问题出在这个函数设计上
    handle->rxInProgress = true;
    DMA_StartTransfer(handle->rxHandle);
```

　　在 SPI\_TransferSubmitNextRxDMA() 函数里，其默认使用内部 s\_dma\_descriptor\_table 描述符，每次只提交单次 DMA 传输，禁止了 reload 功能，这个函数显然是为了被多次调用去连续顺序数据传输而设计的，因此要想实现 DMA 链式传输，我们必须改造这个函数。

![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRTxxx_FCSPI_RX_DMA_block_diagram_code.png)

　　具体改造过程，痞子衡就不一一赘述了，大家直接看下面代码吧，只是改造过程中还是有一些坑需要注意的，这些都是痞子衡调试时的血泪教训。

```
https://github.com/JayHeng/perf-rt600-fcspi-dma-linked-transfer/blob/main/devices/MIMXRT685S/drivers/fsl_spi_dma.c

坑1：链式传输时 DMA_SubmitTransfer() 函数里不能判断 DMA_ChannelIsActive()，否则初始化提交第二个 DMA 传输描述符时会直接返回 busy
坑2：链式传输时 SPI_TransferRxHandlerDMA() 中断处理要重新设计，不要根据 rxInProgress、rxRemainingBytes 状态来决定是否调用用户回调函数
坑3：链式传输时 spiHandle->state 状态不要在中断里改变成 kSPI_Idle 状态，否则 SPI_MasterTransferGetCountDMA() 函数会失效
```

　　至此，i.MXRT下使能DMA链式传输可达到SPI从设备接收速率上限50Mbps痞子衡便介绍完毕了，掌声在哪里~~~

### 欢迎订阅

文章会同时发布到我的 [博客园主页](https://github.com)、[CSDN主页](https://github.com)、[知乎主页](https://github.com):[milou加速器](https://milou6.com)、[微信公众号](https://github.com) 平台上。

微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。

![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/github/pzhMcu_qrcode_258x258.jpg)
