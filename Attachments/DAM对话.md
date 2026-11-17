

# **六、DMA 中断与 HAL 回调机制**

## **6.1 为什么 DMA 完成后需要中断？**

### **6.1.1 DMA 完成传输后还需要做什么？**

在第五章中，我们已经知道，CPU 完成 DMA 的配置和启动后，DMA 就能够根据 SPI 外设产生的 DMA Request，自主完成数据搬运，而 CPU 可以继续执行其他程序。

假设现在需要通过 SPI 发送一个数组：

```
uint8_t data[4] = {0x11, 0x22, 0x33, 0x44};

HAL_SPI_Transmit_DMA(&hspi1, data, 4);
```

在启动 DMA 之后，CPU、DMA、SPI 三者会并行工作。

CPU、DMA、SPI 并行工作的简化示意，不代表精确硬件时序。

当 DMA 将最后一个数据写入 SPI 数据寄存器后，本次 DMA 搬运任务就结束了。但是对于整个应用程序而言，还有一些工作没有完成：

- 确认 SPI 是否真正发送完毕。
    
- 关闭 SPI 的 TX DMA 请求功能。
    
- 将 HAL 库中的 SPI 状态恢复为 READY。
    
- 拉高 LCD 的 CS 片选引脚。
    
- 启动下一次 DMA 传输，或者通知正在等待的 FreeRTOS 任务。
    

这些工作不属于 DMA 控制器的职责，需要由 CPU 执行。

但是，此时 CPU 正在执行其他程序，它并不知道 DMA 什么时候完成了数据搬运。

**因此，我们需要建立一种机制，让 CPU 获知 DMA 的传输结果，并执行相应的后续处理。**

### **6.1.2 CPU 如何知道 DMA 已经完成传输？**

在第四章中已经介绍过 `DMA_CNDTR` 寄存器，该寄存器记录 DMA 剩余需要搬运的数据项数量。

当 DMA 完成最后一次数据搬运时，硬件会将传输完成标志 TCIF（Transfer Complete Interrupt Flag）置 1。

Normal 模式下，传输计数与完成标志之间的关系。

此时，CPU 就可以通过 TCIF 标志位判断 DMA 是否已经完成搬运。

但是，**标志位置位并不意味着 CPU 会主动执行相应的处理程序。** 标志位只是硬件对事件发生情况的一种记录。

为了让 CPU 获知这个事件，有两种处理方式。

**方式一：轮询 DMA 完成标志**

CPU 主动读取 TCIF 标志位，如果该标志位为 0，则表示 DMA 尚未完成；如果为 1，则说明 DMA 已经完成传输。

```
while ((DMA1->ISR & DMA_ISR_TCIF3) == 0)
{
    // 等待 DMA 完成
}

// 执行传输完成后的处理
```

上述代码只是用于说明轮询原理，省略了超时、错误检测、标志清除等操作。

这种方式虽然将数据搬运的工作交给了 DMA，但是 CPU 仍然需要不断查询 DMA 的状态，无法充分利用 DMA 带来的并行执行能力。当然，也可以在主循环中周期性查询标志位，而不是一直阻塞等待。

**方式二：使用 DMA 完成中断**

如果开启 DMA 的传输完成中断，当 TCIF 置 1 时，DMA 就会向 NVIC 提出中断请求。

CPU 不需要主动查询 DMA 的状态，而是继续执行其他程序。当 CPU 响应该中断时，再执行与 DMA 传输完成相关的处理程序。

#### **轮询方式**

#### **中断方式**

现在就能够理解原文提到的“DMA 传输本身不依赖中断”这句话了。

**DMA 负责数据搬运，中断负责事件通知，两者承担不同的职责。** 即使不开启 DMA 中断，硬件仍然能够正常搬运数据，只不过 CPU 需要通过其他方式获知传输结果。

## **6.2 DMA 中断的产生机制**

在了解为什么需要 DMA 中断之后，接下来需要了解 DMA 在什么情况下产生中断，以及中断请求是如何传递给 CPU 的。

### **6.2.1 DMA 的中断类型**

除了传输完成以外，DMA 还存在其他需要 CPU 处理的事件。

STM32F103 的每个 DMA 通道都支持以下三种中断事件：

|中断事件|状态标志位|中断使能位|
|---|---|---|
|HT（Half Transfer）|HTIF|HTIE|
|TC（Transfer Complete）|TCIF|TCIE|
|TE（Transfer Error）|TEIF|TEIE|

其中，状态标志位位于 `DMA_ISR` 寄存器中，中断使能位位于对应通道的 `DMA_CCRx` 寄存器中。

- **HT：传输半完成。** 可以提前处理已经搬运完成的部分数据。
    
- **TC：传输完成。** 可以处理本次传输结果，或者启动下一次传输。
    
- **TE：传输错误。** 通知 CPU 处理 DMA 搬运过程中出现的错误。
    

假设现在 DMA 需要搬运 8 个数据：

在传输过程中，DMA 可以分别产生半完成事件和完成事件。如果对应的中断已经使能，就可以通知 CPU 处理这些事件。

半完成中断在 ADC 连续采样、音频数据采集等场景中比较常见，CPU 可以处理缓冲区的前半部分，而 DMA 继续填充后半部分。

### **6.2.2 中断标志位与中断使能位的关系**

以 SPI1_TX 对应的 DMA1 Channel 3 为例：

```
DMA1->ISR              // 包含 TCIF3
DMA1_Channel3->CCR     // 包含 TCIE
```

这两个寄存器中的标志位具有不同的作用：

- `TCIF3`：记录传输完成事件是否已经发生。
    
- `TCIE`：决定是否允许该事件产生中断请求。
    

两者之间的关系可以用一个与门表示。

|TCIF3|TCIE|是否产生 TC 中断请求|
|---|---|---|
|0|0|否|
|0|1|否|
|1|0|否|
|1|1|是|

由此可见，中断使能位并不控制事件是否发生，而是控制事件发生后是否产生中断请求。

### **6.2.3 DMA 中断请求如何传递给 CPU？**

在 DMA 产生中断请求后，还需要经过 NVIC，CPU 才能进入相应的中断服务函数。
![696](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20180%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%2210%22%20width%3D%2290%22%20height%3D%2252%22%20rx%3D%225%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.1%22%20stroke%3D%22%236488b8%22%2F%3E%3Ctext%20x%3D%2253%22%20y%3D%2231%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EDMA%20%E6%8E%A7%E5%88%B6%E5%99%A8%3C%2Ftext%3E%3Ctext%20x%3D%2253%22%20y%3D%2248%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E4%B8%AD%E6%96%AD%E8%AF%B7%E6%B1%82%3C%2Ftext%3E%3Crect%20x%3D%22125%22%20y%3D%2210%22%20width%3D%2290%22%20height%3D%2252%22%20rx%3D%225%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.1%22%20stroke%3D%22%236488b8%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%2231%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ENVIC%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%2248%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E4%B8%AD%E6%96%AD%E7%AE%A1%E7%90%86%3C%2Ftext%3E%3Crect%20x%3D%22242%22%20y%3D%2210%22%20width%3D%2290%22%20height%3D%2252%22%20rx%3D%225%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.1%22%20stroke%3D%22%236488b8%22%2F%3E%3Ctext%20x%3D%22287%22%20y%3D%2231%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ECPU%3C%2Ftext%3E%3Ctext%20x%3D%22287%22%20y%3D%2248%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E5%93%8D%E5%BA%94%E4%B8%AD%E6%96%AD%3C%2Ftext%3E%3Cpath%20d%3D%22M98%2036H124M215%2036H241M287%2062V87H170V105%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.5%22%20fill%3D%22none%22%2F%3E%3Cpath%20d%3D%22M117%2031L124%2036L117%2041M234%2031L241%2036L234%2041M165%2099L170%20106L175%2099%22%20fill%3D%22none%22%20stroke%3D%22%2387929e%22%2F%3E%3Crect%20x%3D%2248%22%20y%3D%22107%22%20width%3D%22244%22%20height%3D%2258%22%20rx%3D%225%22%20fill%3D%22%2387929e%22%20fill-opacity%3D%22.1%22%20stroke%3D%22%2387929e%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22128%22%20text-anchor%3D%22middle%22%20font-size%3D%2211%22%20fill%3D%22currentColor%22%3E%E6%A0%B9%E6%8D%AE%E4%B8%AD%E6%96%AD%E5%90%91%E9%87%8F%E8%A1%A8%E8%BF%9B%E5%85%A5%20ISR%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22149%22%20text-anchor%3D%22middle%22%20font-size%3D%2211.5%22%20font-weight%3D%22500%22%20fill%3D%22currentColor%22%3EDMA1_Channel3_IRQHandler\(\)%3C%2Ftext%3E%3C%2Fsvg%3E)
因此，要让 CPU 正常响应 DMA 完成中断，除了 DMA 内部的 TCIE 使能以外，还需要使能 NVIC 中对应的中断通道。

CubeMX 通常会生成以下代码：

```
HAL_NVIC_SetPriority(DMA1_Channel3_IRQn, 5, 0);
HAL_NVIC_EnableIRQ(DMA1_Channel3_IRQn);
```

而 DMA 内部的完成中断使能，通常由 `HAL_DMA_Start_IT()` 在启动传输时完成。

需要注意的是，即使 DMA 已经产生中断请求，CPU 也不一定立即响应，它还受到中断优先级、中断屏蔽等因素影响。

## **6.3 DMA 完成与 SPI 完成的区别**

在继续讲解 HAL 中断处理流程之前，需要先解决一个问题：

**既然 DMA 已经产生了传输完成中断，为什么 HAL 不能直接通知用户 SPI 发送完成？**

这是因为 DMA 和 SPI 是两个相对独立的硬件模块，它们所负责的工作不同。

### **6.3.1 SPI 内部的数据传输过程**

回顾 SPI 的内部结构，SPI 发送数据时主要涉及两个部分：

- 发送缓冲区（Transmit Buffer）：暂存即将发送的数据，通过 SPI_DR 访问。
    
- 移位寄存器（Shift Register）：按照 SPI 时钟将数据逐位发送出去。
![696](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20198%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Cdefs%3E%3Cmarker%20id%3D%22sp6%22%20markerWidth%3D%226%22%20markerHeight%3D%226%22%20refX%3D%225%22%20refY%3D%223%22%20orient%3D%22auto%22%3E%3Cpath%20d%3D%22M0%200L6%203L0%206Z%22%20fill%3D%22%2387929e%22%2F%3E%3C%2Fmarker%3E%3C%2Fdefs%3E%3Crect%20x%3D%228%22%20y%3D%228%22%20width%3D%22100%22%20height%3D%2250%22%20rx%3D%225%22%20fill%3D%22%2387929e%22%20fill-opacity%3D%22.1%22%20stroke%3D%22%2387929e%22%2F%3E%3Ctext%20x%3D%2258%22%20y%3D%2228%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22500%22%20fill%3D%22currentColor%22%3ESRAM%3C%2Ftext%3E%3Ctext%20x%3D%2258%22%20y%3D%2245%22%20text-anchor%3D%22middle%22%20font-size%3D%2211%22%20fill%3D%22currentColor%22%3Edata%5B%5D%3C%2Ftext%3E%3Crect%20x%3D%228%22%20y%3D%2291%22%20width%3D%22100%22%20height%3D%2249%22%20rx%3D%225%22%20fill%3D%22%2326966f%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%2326966f%22%2F%3E%3Ctext%20x%3D%2258%22%20y%3D%22111%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20font-weight%3D%22500%22%20fill%3D%22currentColor%22%3EDMA%3C%2Ftext%3E%3Ctext%20x%3D%2258%22%20y%3D%22127%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E6%95%B0%E6%8D%AE%E6%90%AC%E8%BF%90%3C%2Ftext%3E%3Cpath%20d%3D%22M58%2058V89M108%20115H153%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.5%22%20fill%3D%22none%22%20marker-end%3D%22url\(%23sp6\)%22%2F%3E%3Crect%20x%3D%22136%22%20y%3D%228%22%20width%3D%22196%22%20height%3D%22161%22%20rx%3D%227%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.04%22%20stroke%3D%22%236488b8%22%20stroke-dasharray%3D%224%203%22%2F%3E%3Ctext%20x%3D%22234%22%20y%3D%2228%22%20text-anchor%3D%22middle%22%20font-weight%3D%22600%22%20font-size%3D%2212%22%20fill%3D%22currentColor%22%3ESPI%20%E5%A4%96%E8%AE%BE%E5%86%85%E9%83%A8%3C%2Ftext%3E%3Crect%20x%3D%22155%22%20y%3D%2242%22%20width%3D%22158%22%20height%3D%2238%22%20rx%3D%224%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%236488b8%22%2F%3E%3Ctext%20x%3D%22234%22%20y%3D%2257%22%20text-anchor%3D%22middle%22%20font-size%3D%2211%22%20fill%3D%22currentColor%22%3E%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%3C%2Ftext%3E%3Ctext%20x%3D%22234%22%20y%3D%2271%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3ESPI_DR%20%E5%86%99%E5%85%A5%E8%B7%AF%E5%BE%84%3C%2Ftext%3E%3Cpath%20d%3D%22M153%20115H145V61H154M234%2080V106%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%20marker-end%3D%22url\(%23sp6\)%22%2F%3E%3Crect%20x%3D%22155%22%20y%3D%22108%22%20width%3D%22158%22%20height%3D%2240%22%20rx%3D%224%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%23d3953d%22%2F%3E%3Ctext%20x%3D%22234%22%20y%3D%22124%22%20text-anchor%3D%22middle%22%20font-size%3D%2211%22%20fill%3D%22currentColor%22%3E%E7%A7%BB%E4%BD%8D%E5%AF%84%E5%AD%98%E5%99%A8%3C%2Ftext%3E%3Crect%20x%3D%22174%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22181%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E1%3C%2Ftext%3E%3Crect%20x%3D%22189%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22196%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0%3C%2Ftext%3E%3Crect%20x%3D%22204%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22211%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E1%3C%2Ftext%3E%3Crect%20x%3D%22219%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22226%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0%3C%2Ftext%3E%3Crect%20x%3D%22234%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22241%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0%3C%2Ftext%3E%3Crect%20x%3D%22249%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22256%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E1%3C%2Ftext%3E%3Crect%20x%3D%22264%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22271%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0%3C%2Ftext%3E%3Crect%20x%3D%22279%22%20y%3D%22130%22%20width%3D%2214%22%20height%3D%2213%22%20stroke%3D%22%23d3953d%22%20stroke-width%3D%22.7%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22286%22%20y%3D%22140%22%20font-size%3D%229%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E1%3C%2Ftext%3E%3Cpath%20d%3D%22M234%20148V181H302%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%20marker-end%3D%22url\(%23sp6\)%22%2F%3E%3Ctext%20x%3D%22270%22%20y%3D%22174%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EMOSI%3C%2Ftext%3E%3C%2Fsvg%3E)
DMA 的职责到将数据写入 SPI_DR 为止，之后的数据转移和逐位发送都由 SPI 硬件完成。

所以 DMA 搬运完成时，SPI 内部可能仍然存在尚未发出的数据。

### **6.3.2 TXE 与 BSY 标志位**

为了判断 SPI 内部的数据是否已经全部发送完毕，需要使用 SPI_SR 寄存器中的两个状态标志。

|标志位|全称|含义|
|---|---|---|
|TXE|Transmit Buffer Empty|发送缓冲区为空|
|BSY|Busy Flag|SPI 是否处于忙状态|

TXE 反映的是发送缓冲区的状态，而 BSY 反映的是 SPI 通信是否仍然处于忙状态。

由于发送缓冲区中的数据可以转移到移位寄存器，因此 TXE = 1 时，SPI 仍然可能正在发送数据。

下面通过三个阶段理解它们之间的关系。
DMA完成
![696](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20140%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%3C%2Ftext%3E%3Crect%20x%3D%2220%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%220.25%22%20stroke%3D%22%236488b8%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0x44%3C%2Ftext%3E%3Cpath%20d%3D%22M133%2060H163M156%2055L163%2060L156%2065%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.4%22%20fill%3D%22none%22%2F%3E%3Crect%20x%3D%22165%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%23d3953d%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A7%BB%E4%BD%8D%E5%AF%84%E5%AD%98%E5%99%A8%3C%2Ftext%3E%3Crect%20x%3D%22177%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%220.25%22%20stroke%3D%22%23d3953d%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0x33%3C%2Ftext%3E%3Cpath%20d%3D%22M290%2060H332M325%2055L332%2060L325%2065%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22313%22%20y%3D%2247%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3EMOSI%3C%2Ftext%3E%3Crect%20x%3D%228%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%2326966f%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%2257.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22%2326966f%22%3ECNDTR%20%3D%200%3C%2Ftext%3E%3Crect%20x%3D%22115%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22164.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ETXE%20%3D%200%3C%2Ftext%3E%3Crect%20x%3D%22222%22%20y%3D%22105%22%20width%3D%22110%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22277%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EBSY%20%3D%201%3C%2Ftext%3E%3C%2Fsvg%3E)
*DMA 已经将最后一个字节写入发送缓冲区，本次 DMA 搬运结束。但 SPI 仍在发送前一个字节。*
缓冲区空
![696](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20140%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%3C%2Ftext%3E%3Crect%20x%3D%2220%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%220.04%22%20stroke%3D%22%236488b8%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A9%BA%3C%2Ftext%3E%3Cpath%20d%3D%22M133%2060H163M156%2055L163%2060L156%2065%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.4%22%20fill%3D%22none%22%2F%3E%3Crect%20x%3D%22165%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%23d3953d%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A7%BB%E4%BD%8D%E5%AF%84%E5%AD%98%E5%99%A8%3C%2Ftext%3E%3Crect%20x%3D%22177%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%220.25%22%20stroke%3D%22%23d3953d%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E0x44%3C%2Ftext%3E%3Cpath%20d%3D%22M290%2060H332M325%2055L332%2060L325%2065%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22313%22%20y%3D%2247%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3EMOSI%3C%2Ftext%3E%3Crect%20x%3D%228%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%2326966f%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%2257.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22%2326966f%22%3ECNDTR%20%3D%200%3C%2Ftext%3E%3Crect%20x%3D%22115%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22164.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ETXE%20%3D%201%3C%2Ftext%3E%3Crect%20x%3D%22222%22%20y%3D%22105%22%20width%3D%22110%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22277%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EBSY%20%3D%201%3C%2Ftext%3E%3C%2Fsvg%3E)
*最后一个字节进入移位寄存器，发送缓冲区为空，但最后一个字节仍在通过 MOSI 输出。*
SPI完成
![696](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20140%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E5%8F%91%E9%80%81%E7%BC%93%E5%86%B2%E5%8C%BA%3C%2Ftext%3E%3Crect%20x%3D%2220%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%220.04%22%20stroke%3D%22%236488b8%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%2270%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A9%BA%3C%2Ftext%3E%3Cpath%20d%3D%22M133%2060H163M156%2055L163%2060L156%2065%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.4%22%20fill%3D%22none%22%2F%3E%3Crect%20x%3D%22165%22%20y%3D%2214%22%20width%3D%22125%22%20height%3D%2276%22%20rx%3D%226%22%20stroke%3D%22%23d3953d%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2236%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A7%BB%E4%BD%8D%E5%AF%84%E5%AD%98%E5%99%A8%3C%2Ftext%3E%3Crect%20x%3D%22177%22%20y%3D%2245%22%20width%3D%22101%22%20height%3D%2232%22%20rx%3D%223%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%220.04%22%20stroke%3D%22%23d3953d%22%20stroke-opacity%3D%22.7%22%2F%3E%3Ctext%20x%3D%22227%22%20y%3D%2266%22%20font-size%3D%2214%22%20font-weight%3D%22600%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E7%A9%BA%3C%2Ftext%3E%3Cpath%20d%3D%22M290%2060H332M325%2055L332%2060L325%2065%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22313%22%20y%3D%2247%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3EMOSI%3C%2Ftext%3E%3Crect%20x%3D%228%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%2326966f%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%2257.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22%2326966f%22%3ECNDTR%20%3D%200%3C%2Ftext%3E%3Crect%20x%3D%22115%22%20y%3D%22105%22%20width%3D%2299%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22164.5%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ETXE%20%3D%201%3C%2Ftext%3E%3Crect%20x%3D%22222%22%20y%3D%22105%22%20width%3D%22110%22%20height%3D%2227%22%20rx%3D%224%22%20fill%3D%22%23d3953d%22%20fill-opacity%3D%22.12%22%2F%3E%3Ctext%20x%3D%22277%22%20y%3D%22122%22%20font-size%3D%2211.5%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EBSY%20%3D%200%3C%2Ftext%3E%3C%2Fsvg%3E)
*最后一个字节发送完毕，SPI 退出忙状态，本次 SPI 硬件发送结束。*


点击上方三个阶段，观察最后两个字节的传输过程。此处为一种可能的硬件时序。

总结以上三个阶段：

|阶段|CNDTR|TXE|BSY|
|---|---|---|---|
|DMA 完成最后一次搬运|0|0|1|
|最后一个数据进入移位寄存器|0|1|1|
|SPI 完成全部数据发送|0|1|0|

因此，在 STM32F103 的 SPI 主机发送场景下，DMA 搬运完成之后，还需要确认 TXE = 1、BSY = 0，才能正常结束本次 SPI 发送事务。

这也决定了接下来 HAL 库的中断处理流程为什么需要分为 DMA 层和 SPI 层。

## **6.4 DMA 中断与 HAL 回调函数的关系**

### **6.4.1 为什么需要多层回调函数？**

DMA 是一个通用的数据搬运外设，它不仅可以服务于 SPI，还可以服务于 UART、ADC、I2C 等外设。

然而，不同外设在 DMA 完成搬运后，需要执行的后续处理并不相同。
![](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20180%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%2290%22%20y%3D%225%22%20width%3D%22160%22%20height%3D%2243%22%20rx%3D%225%22%20fill%3D%22%2387929e%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%2387929e%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%2222%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EDMA%20%E5%AE%8C%E6%88%90%E5%A4%84%E7%90%86%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%2237%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3E%E9%80%9A%E7%94%A8%20DMA%20%E5%B1%82%3C%2Ftext%3E%3Cpath%20d%3D%22M170%2048V67H59V86M170%2067V86M170%2067H281V86%22%20stroke%3D%22%2387929e%22%20fill%3D%22none%22%20stroke-width%3D%221.3%22%2F%3E%3Crect%20x%3D%226%22%20y%3D%2287%22%20width%3D%22106%22%20height%3D%2279%22%20rx%3D%225%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%2259%22%20y%3D%22107%22%20text-anchor%3D%22middle%22%20font-size%3D%2213%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3ESPI%3C%2Ftext%3E%3Ctext%20x%3D%2259%22%20y%3D%22132%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E6%A3%80%E6%9F%A5%20TXE%E3%80%81BSY%3C%2Ftext%3E%3Ctext%20x%3D%2259%22%20y%3D%22149%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E7%BB%93%E6%9D%9F%20SPI%20%E4%BA%8B%E5%8A%A1%3C%2Ftext%3E%3Crect%20x%3D%22117%22%20y%3D%2287%22%20width%3D%22106%22%20height%3D%2279%22%20rx%3D%225%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22107%22%20text-anchor%3D%22middle%22%20font-size%3D%2213%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EUART%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22132%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E5%A4%84%E7%90%86%20UART%20TC%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22149%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E7%BB%93%E6%9D%9F%20UART%20%E5%8F%91%E9%80%81%3C%2Ftext%3E%3Crect%20x%3D%22228%22%20y%3D%2287%22%20width%3D%22106%22%20height%3D%2279%22%20rx%3D%225%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.08%22%2F%3E%3Ctext%20x%3D%22281%22%20y%3D%22107%22%20text-anchor%3D%22middle%22%20font-size%3D%2213%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EADC%3C%2Ftext%3E%3Ctext%20x%3D%22281%22%20y%3D%22132%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E5%A4%84%E7%90%86%E8%BD%AC%E6%8D%A2%E4%BA%8B%E4%BB%B6%3C%2Ftext%3E%3Ctext%20x%3D%22281%22%20y%3D%22149%22%20text-anchor%3D%22middle%22%20font-size%3D%2210.5%22%20fill%3D%22currentColor%22%3E%E9%80%9A%E7%9F%A5%E9%87%87%E6%A0%B7%E5%AE%8C%E6%88%90%3C%2Ftext%3E%3C%2Fsvg%3E)
不同外设需要不同的传输完成处理逻辑。

如果将所有外设的处理逻辑都写在 DMA 中断处理函数中，就会导致 DMA 驱动与其他外设驱动高度耦合。

因此，HAL 库采用了函数指针和回调机制，将处理过程划分为三个层次：

|层次|职责|
|---|---|
|DMA 驱动层|处理 DMA 硬件事件|
|外设驱动层|处理 SPI、UART、ADC 等外设特有的工作|
|用户应用层|处理具体的业务逻辑|

每一层只完成自身职责，然后调用下一层对应的处理函数。

### **6.4.2 HAL 如何建立 DMA 层与 SPI 层的关系？**

回顾第五章，在调用 `HAL_SPI_Transmit_DMA()` 时，HAL 会配置 DMA 的回调函数。

其中包括：

```
hspi->hdmatx->XferHalfCpltCallback = SPI_DMAHalfTransmitCplt;

hspi->hdmatx->XferCpltCallback = SPI_DMATransmitCplt;

hspi->hdmatx->XferErrorCallback = SPI_DMAError;
```

这里的 `XferCpltCallback` 是 DMA 句柄中的一个函数指针成员。

上述赋值操作并不是在调用 `SPI_DMATransmitCplt()`，而是在保存该函数的地址，告诉 DMA 驱动：当本次传输完成时，应当调用这个函数。

![](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(13%2C%2013%2C%2013\)%22%20viewBox%3D%220%200%20340%20196%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%228%22%20y%3D%228%22%20width%3D%22324%22%20height%3D%2236%22%20rx%3D%225%22%20stroke%3D%22%2387929e%22%20fill%3D%22%2387929e%22%20fill-opacity%3D%22.1%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%2230%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3EHAL_SPI_Transmit_DMA\(\)%3C%2Ftext%3E%3Cpath%20d%3D%22M170%2044V65%22%20stroke%3D%22%2387929e%22%20stroke-width%3D%221.5%22%2F%3E%3Ctext%20x%3D%22190%22%20y%3D%2258%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3E%E8%AE%BE%E7%BD%AE%E5%9B%9E%E8%B0%83%3C%2Ftext%3E%3Crect%20x%3D%228%22%20y%3D%2266%22%20width%3D%22324%22%20height%3D%2280%22%20rx%3D%225%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.07%22%2F%3E%3Ctext%20x%3D%2220%22%20y%3D%2285%22%20font-size%3D%2211.5%22%20font-weight%3D%22600%22%20fill%3D%22currentColor%22%3Ehdma_spi1_tx%EF%BC%88DMA%20%E5%8F%A5%E6%9F%84%EF%BC%89%3C%2Ftext%3E%3Crect%20x%3D%2219%22%20y%3D%2296%22%20width%3D%22302%22%20height%3D%2238%22%20rx%3D%223%22%20stroke%3D%22%236488b8%22%20fill%3D%22%236488b8%22%20fill-opacity%3D%22.13%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22111%22%20font-size%3D%2211%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EXferCpltCallback%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22125%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%E4%BF%9D%E5%AD%98%20SPI%20%E5%86%85%E9%83%A8%E5%AE%8C%E6%88%90%E5%9B%9E%E8%B0%83%E7%9A%84%E5%9C%B0%E5%9D%80%3C%2Ftext%3E%3Cpath%20d%3D%22M170%20134V161M165%20155L170%20161L175%20155%22%20fill%3D%22none%22%20stroke%3D%22%2326966f%22%20stroke-width%3D%221.5%22%2F%3E%3Crect%20x%3D%2249%22%20y%3D%22162%22%20width%3D%22242%22%20height%3D%2228%22%20rx%3D%224%22%20fill%3D%22%2326966f%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%2326966f%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22180%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3ESPI_DMATransmitCplt\(\)%3C%2Ftext%3E%3C%2Fsvg%3E)

这样一来，DMA 驱动就不需要知道当前正在为哪个外设服务，只需要在对应事件发生时调用已经设置好的回调函数即可。

这种机制也是 HAL 库中广泛使用的驱动设计方式。

## **6.5 从 DMA 中断到用户回调的完整执行流程**

现在已经具备了理解整个中断处理过程所需要的前置知识。

当 DMA 完成数据搬运之后，CPU 将依次执行以下几个函数。

硬件层

**DMA1 Channel 3**

CNDTR = 0，TCIF3 = 1，产生中断请求

中断入口

`DMA1_Channel3_IRQHandler()`

CPU 根据中断向量表进入 ISR

DMA 层

`HAL_DMA_IRQHandler()`

识别事件、清除标志、恢复 DMA 状态

SPI 层

`SPI_DMATransmitCplt()`

确认 SPI 发送结束、恢复 SPI 状态

应用层

`HAL_SPI_TxCpltCallback()`

执行用户定义的发送完成处理逻辑

下面依次解释每个函数为什么存在，以及它们各自负责什么工作。

### **6.5.1 DMA1_Channel3_IRQHandler()**

这是 DMA1 Channel 3 对应的中断服务函数（ISR），也是 CPU 响应本次 DMA 中断后的程序入口。

CubeMX 通常会生成如下代码：

```
void DMA1_Channel3_IRQHandler(void)
{
    HAL_DMA_IRQHandler(&hdma_spi1_tx);
}
```

由于 DMA1 Channel 3 具有 HT、TC、TE 三种中断事件，因此仅仅进入这个中断服务函数，并不能确定具体发生了哪一种事件。

所以还需要调用 `HAL_DMA_IRQHandler()` 来进一步判断和处理。

### **6.5.2 HAL_DMA_IRQHandler()**

这是 HAL 库提供的通用 DMA 中断处理函数。

它通过传入的 DMA 句柄找到对应的 DMA 通道，然后检查状态标志位和中断使能位，判断当前需要处理的事件。

如果本次需要处理的是 Normal 模式下的传输完成事件，那么它主要执行以下操作：

1. 关闭 DMA 的 TC、TE 中断使能。
    
2. 将 DMA 句柄的状态恢复为 READY。
    
3. 清除传输完成标志位。
    
4. 解除 DMA 句柄的软件锁。
    
5. 调用传输完成回调函数。
    

以下为关键代码的简化示意，并非完整 HAL 源码。

```
void HAL_DMA_IRQHandler(DMA_HandleTypeDef *hdma)
{
    // 省略中断类型判断

    // 处理 Normal 模式下的 TC 事件
    hdma->State = HAL_DMA_STATE_READY;

    // 清除完成标志等收尾操作

    if (hdma->XferCpltCallback != NULL)
    {
        hdma->XferCpltCallback(hdma);
    }
}
```

由于在 DMA 启动之前，SPI HAL 已经将 `XferCpltCallback` 指向了 `SPI_DMATransmitCplt()`，所以最后一条函数指针调用实际进入的就是 SPI 层的内部完成处理函数。

Git at Google+1

至此，DMA 层的主要工作已经完成。

### **6.5.3 SPI_DMATransmitCplt()**

该函数是 SPI HAL 内部使用的 DMA 发送完成回调函数。

首先需要理解它的参数：

```
static void SPI_DMATransmitCplt(DMA_HandleTypeDef *hdma)
```

DMA 层传入的是 DMA 句柄，但当前函数需要操作 SPI 外设，因此必须先通过 DMA 句柄找到 SPI 句柄。

这就需要利用第五章介绍过的 `__HAL_LINKDMA()` 建立的关联关系。

SPI 句柄与 DMA 句柄之间建立了双向关联。

因此可以通过以下代码获得 SPI 句柄：

```
SPI_HandleTypeDef *hspi =
    (SPI_HandleTypeDef *)hdma->Parent;
```

获得 SPI 句柄后，HAL 就可以操作 SPI 外设及其软件状态。

现在回顾 6.3 中讲到的知识：DMA 完成并不意味着 SPI 完成。

所以在 Normal 模式下，`SPI_DMATransmitCplt()` 主要负责：

1. 关闭 SPI 错误中断使能。
    
2. 清除 TXDMAEN，关闭 SPI TX DMA 请求。
    
3. 调用 `SPI_EndRxTxTransaction()`，等待 TXE = 1、BSY = 0，并处理超时错误。
    
4. 在双线通信模式下清除 OVR 标志。
    
5. 将 SPI 软件传输计数清零。
    
6. 将 SPI 软件状态恢复为 READY。
    
7. 根据错误状态，调用用户的完成回调或错误回调。
    
    Git at Google+1
    

这里需要特别注意，DMA 的状态与 SPI 的状态是相互独立的：

```
hdma_spi1_tx.State
hspi1.State
```

DMA 的状态由 DMA 层负责维护，而 SPI 的状态由 SPI 层负责维护。

所以 DMA 已经恢复 READY，并不意味着 SPI 也已经恢复 READY。

### **6.5.4 HAL_SPI_TxCpltCallback()**

前面的几个函数全部属于中断处理或驱动层的内部处理。

当 SPI HAL 确认发送正常结束、完成必要的收尾工作后，最后才调用用户回调函数：

```
void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)
{

}
```

在默认的弱函数回调机制下，该函数由 HAL 库提供弱定义（`__weak`），用户可以在自己的代码文件中重新定义它，从而实现自定义的处理逻辑。

以之前的 LCD 项目为例：

```
void HAL_SPI_TxCpltCallback(SPI_HandleTypeDef *hspi)
{
    if (hspi->Instance == SPI1)
    {
        HAL_GPIO_WritePin(
            LCD_CS_GPIO_Port,
            LCD_CS_Pin,
            GPIO_PIN_SET
        );
    }
}
```

由于进入该函数时，SPI HAL 已经确认本次 SPI 发送正常结束，因此可以安全地结束当前 LCD 片选事务（还需满足 LCD 自身的片选保持时间要求）。

除此之外，也可以在这个函数中启动下一块数据的 DMA 发送、释放二进制信号量或者发送 FreeRTOS 任务通知。

这些操作属于应用层的工作，不属于 DMA 驱动或 SPI 驱动本身的职责。

## **6.6 中断服务函数与回调函数的区别**

在 HAL 库中经常遇到以下三类函数：

|对比项|中断服务函数|HAL 中断处理函数|回调函数|
|---|---|---|---|
|示例|`DMA1_Channel3_IRQHandler()`|`HAL_DMA_IRQHandler()`|`HAL_SPI_TxCpltCallback()`|
|调用者|CPU 异常响应机制|中断服务函数|HAL 内部函数|
|主要作用|进入中断处理流程|识别并处理硬件事件|处理特定事件|
|是否对应中断向量|是|否|否|
|是否属于普通 C 函数调用|不是普通调用进入|是|是|

虽然 HAL 中断处理函数和回调函数并不是 CPU 直接进入的中断服务函数，但是它们都是在本次 ISR 的执行过程中被调用的。

因此，它们仍然处于中断上下文（Interrupt Context）中。

这也解释了为什么在 DMA 完成回调中使用 FreeRTOS API 时，需要使用带有 `FromISR` 后缀的版本。

例如：

```
xSemaphoreGiveFromISR();
xQueueSendFromISR();
vTaskNotifyGiveFromISR();
```

同时还需要满足 FreeRTOS 对可调用内核 API 的中断优先级限制。

不能因为函数名称中没有 `IRQHandler`，就认为它已经脱离中断上下文。

## **6.7 DMA Request 与 DMA Interrupt 的区别**

现在可以结合整个执行过程，重新理解第二章提到的 DMA Request 和 DMA Interrupt。

|对比项|DMA Request|DMA Interrupt|
|---|---|---|
|信号产生方|外设|DMA 控制器|
|信号接收方|DMA 控制器|NVIC、CPU|
|主要作用|请求搬运数据|通知 DMA 相关事件|
|SPI 发送时的触发条件|TXE 与 TXDMAEN|DMA 事件标志与中断使能|
|是否需要 CPU 参与响应|否|是|
|发生频率|通常每个数据项对应一次搬运请求|由半完成、完成、错误等事件决定|

DMA Request 解决的是硬件之间的数据搬运问题，DMA Interrupt 解决的是硬件与软件之间的事件通知问题。

## **6.8 本章总结：一次 DMA 发送的完整生命周期**

将第五章与第六章的内容结合起来，可以将一次 SPI DMA 发送分为四个阶段。

**阶段一：软件配置与启动**

CPU 负责

配置回调函数、设置 BUSY_TX、启动 DMA、使能 SPI DMA Request。

HAL_SPI_Transmit_DMA()

**阶段二：硬件自主传输**

DMA 与 SPI 负责

SPI 产生请求，DMA 搬运数据，SPI 将数据串行输出。DMA 完成后产生中断请求。

**阶段三：中断与驱动收尾**

CPU、HAL 驱动负责

处理 DMA 完成事件，恢复 DMA 状态，检查 SPI 发送是否结束，恢复 SPI 状态。

HAL_DMA_IRQHandler()

SPI_DMATransmitCplt()

**阶段四：应用层处理**

CPU、用户程序负责

释放 LCD 片选、启动下一次发送、通知 FreeRTOS 任务等。

HAL_SPI_TxCpltCallback()

最后，需要明确四个不同的传输完成节点。

|时间节点|含义|
|---|---|
|启动函数返回 HAL_OK|DMA 发送成功启动|
|DMA TCIF = 1|数据全部搬运完成|
|SPI TXE = 1、BSY = 0|SPI 硬件发送结束|
|进入 SPI 发送完成回调|HAL 正常收尾完成，开始应用层处理|

这四个时间节点的区别，也是第七章 HAL 状态管理、第八章连续 DMA 发送以及第九章 DMA 与 FreeRTOS 结合使用的理论基础。