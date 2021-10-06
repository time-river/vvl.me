---
title: 数字传输
date: 2016-10-01 05:25:24
categories: style
tags: ['data communications and networking', 'physical layer and transmission medium']
mathjax: true
---

> 模拟数据到数字数据的转换
> 数字数据到数字信号的转换

## 模拟数据到数字数据的转换

数字信号是优于模拟信号的模拟信号的。可有时候只有模拟信号，诸如麦克风或照相机产生的信号。现在的趋势是把模拟信号转换成数字信号，存在两种方案：脉冲码调制(pluse code modulation, PCM)，Delta 调制(delta modulation, DM)。

### 脉冲码调制

PCM 编码器有三个过程：

![components of PCM encoder](/images/16/10/components-of-PCM-encoder.png)

1. 对模拟信号进行采样(sampling)
2. 对采样后信号进行量化(quantization)
3. 量化后的值编码成位流

采样的一个重要考虑是采样频率(sampling frequency)。每隔 {% raw %}$T_s${% endraw %} 秒对模拟信号进行采样。根据奈圭斯特定理(Nyquist theorem)，为了再生原始模拟信号，采样速率必须至少是信号所含最高频率的两倍。实际应用中，往往使用比奈圭斯特频率更高的采样频率。
{% raw %}
$$
f_s = 2 \cdot f_{max}
$$
{% endraw %}

采样后的结果是一系列振幅介于信号最大振幅与最小振幅之间的脉冲。振幅集可能是无限个介于两个限制集间的非整数值，这些值无法用于编码过程，因此需要量化。均匀量化的高度是固定的，小信号无法被量化表示，因此采用非均匀量化或压扩函数的方法。

最后一步是编码。在每个样本量化并且每个样本的位数确定后，每个样本可以转换成 {% raw %}$n_b${% endraw %} 个位的码字。如果量化等级为 {% raw %}$L${% endraw %}，那么 {% raw %}$n_b = \log_2 L${% endraw %}
{% raw %}
$$
比特率 = 采样速率 \cdot 每个样本的位数 = f_s \cdot n_b
$$
{% endraw %}

下图是原始信号的恢复过程。原始信号的恢复需要 PCM 解码器。解码器先使用电路把码字转换成保持下一个脉冲前振幅的脉冲。当阶梯信号完成后，它经过一个低通滤波器把阶梯信号平滑成模拟信号。过滤器有同发送方原来一样的截断频率。如果信号是以奈圭斯特采样速率采样的，并且有足够的量化等级，那么就可以重新生成原始信号。

![components of a PCM decoder](/images/16/10/components-of-a-PCM-decoder.png)

数字信号的带宽可利用下面的公式求出。数字信号的最小带宽是模拟信号带宽的 {% raw %}$c \cdot n_b \cdot \frac {1}{r}${% endraw %} 倍，这就是数字化付出的代价。
{% raw %}
$$
B_{min} = c \cdot N \cdot \frac {1}{r} = c \cdot n_b \cdot f_s \cdot \frac {1}{r} = c \cdot n_b \cdot 2 \cdot B_{analog} \cdot \frac {1}{r}
$$
{% endraw %}

### Delta 调制

PCM 是一种十分复杂的技术，已有其他的技术来减少 PCM 的复杂性。最简单的便是 delta 调制。下图显示了 delta 调制的过程。这里并没有任何码字，位一个接一个被发送。调制器用在发送方站点，用来从模拟信号中产生位流。这个处理记录了小的正/负改变，称为 delta {% raw %}$\delta${% endraw %}。如果 delta 是正的，就记录成 1，否则记录成 0。

![the process of delta modulation](/images/16/10/the-process-of-delta-modulation.png)

## 数字数据到数字信号的转换

把数字数据转换成数字信号，涉及三种技术：线路编码(line coding)、块编码(block coding)、扰动(scrambling)。

### 线路编码

线路编码是将数字数据转换为数字信号的过程。在发送方，数字数据被编码成数字信号

在数据通信中，信号元素(signal element)承载数据元素(data element)。数据元素是需要被发送的，而信号元素是能发送的。__信号电平数是数字信号独有的概念，意思是每一个信号元素(电平)可表示的比特位。__定义比率 r 为每个信号元素承载的数据元素，下图说明了几种不同的情况：

![data element and signal element](/images/16/10/data-element-and-signal-element.png)

打个比方：当 {% raw %}$r = 1${% endraw %}，意味着每个人驾驶一辆车。当 {% raw %}$r > 1${% endraw %}，意味着多个人驾驶一辆车。一个人也可以驾驶一辆车和一辆拖车({% raw %}$r = \frac {1}{2}${% endraw %})。

每秒发送的位数称为比特率(bit rate, 也叫数据速率, data rate)，以每秒位(bits per second, bps, b/s)表示，是数据速率。信号速率(signal rate)是每秒发送的信号元素数量，也叫波特率(baud rate)，单位是波特(baud)。他们的关系如下：
{% raw %}
$$
S = c * N * \frac{1}{r} baud
$$
{% endraw %}
这里 N 是数据速率(bps)，c 是情形因子(它会根据每种情形——最好、一般、最坏而改变)，S 是信号元素数量。
数字通信的目标是增加数字速率而降低信号速率。

数字信号理论上带宽是无限的，但是许多成分的振幅很小，可以忽略不计，有效的带宽是有限的。波特率而不是比特率决定了数字信号的带宽。

线路编码方案有三个要求：基线偏移、直流成分、自同步。
接受方计算收到信号功率的平均值叫做基线(baseline)，输入信号的功率会与基线比较来确定数据元素的值。0 或 1 的长子字符串会引起基线偏移，使得接受方不能正确地进行解码。
接近于零的频率称为 DC (直流)成分。会给不允许通过低频率的系统或者使用电子耦合的系统(如变压器)带来问题。
自同步是指接受方的位间隔与发送方的位间隔严格对应与匹配。

![lack of self-synchronization results](/images/16/10/lack-of-self-synchronization-results.png)

线路编码的方案有这么几种：

![line coding schemes](/images/16/10/line-coding-schemes.png)

> 不归零(non-return-to-zero, NRZ)编码的名称来源于在位中间信号不会回到零。

### 单极编码方案

单极(unipolar)编码中的不归零——正电平定义为 1 而零电平定义为 0，这导致它的标准功率很高，因此现在这个方案不用与数据通信中。
![unipolar NRZ scheme](/images/16/10/unipolar-NRZ-scheme.png)

### 极性编码方案

极性(polar)编码方案中，电平在时间轴的两边，0 电平可能是正的，1 电平可能是负的。

#### NRZ-L 与 NRZ-I

NRZ-L(NRZ-Level, NRZ 电平编码)——信号电平决定了位值、NRZ-I(NRZ-Invert, NRZ 反相编码)——信号电平是否反转决定了位值。除了基线偏移问题外，这两种编码方案还都有 DC 问题，而且没有自同步能力。NRZ-L 还会因系统中极性的意外改变而导致所有 0 解释成 1、1 解释成 0。
![polar NRZ-L and NRZ-I schemes](/images/16/10/polar-NRZ-L-and-NRZ-I-schemes.png)

#### RZ

归零(RZ)编码方案解决了自同步问题——在每个位中间信号变为零、DC 问题，尽管也会存在因极性的意外改变而产生的问题。它使用三个电平，无疑增加了复杂性。同时使用两个信号变化来编码一个位，表明它会占用更大的带宽。
![polar RZ scheme](/images/16/10/polar-RZ-scheme.png)

#### 双向编码

曼彻斯特编码与差分曼彻斯特编码又称双向编码。

RZ 的思想(位中间跳变)和 NRZ-L 的思想共同组成了曼彻斯特(Manchester)编码方案。在曼彻斯特编码中，位的持续时间被二等分——前半部分保持一个水平、后半部分保持另一个水平，位中间跳变提供了自同步能力，它克服了 NRZ-L 的一些问题。差分曼彻斯特(Differential Manchester)组合了 RZ 和 NRZ-I 的思想——在位中间有一个跳变，但是位置在开始时确定，它克服了 NRZ-I 的一些问题。这两种方案都没有基线偏移，也没有 DC 成分。唯一的缺点是他们的最小带宽是 NRZ 的两倍。
![biphase scheme](/images/16/10/biphase-scheme.png)

### 双极性方案

在双极(bipolar)编码(也称为多电平二进制, multilevel binary)中，有三个电平：正值、负值和零。一个数据元素的电平是零，另一个数据元素的电平在正直、负值间交替。它是 NRZ 的替代方案。信号速率(波特率)与 NRZ 一样，但是没有直流成分。

### AMI 和伪三元编码

AMI 和伪三元编码是双极编码的两种编码方案。
AMI(交替信号反转, alternate mark inversion)是交替的 1 的反换，它常用于长距离通信。它的一个变型是伪三元编码(pseudoternary)——位 1 编码成 0 电平，而位 0 编码成交替正负电平。
![polar schemes](/images/16/10/bipolar-schemes.png)

### 多电平方案

### 多线路传输

## 有趣的例子

这是一个有趣的例子：如果针对诸如时钟指针旋转的周期性事件进行采样，我们会看到什么？时钟分针周期为六十秒。根据奈圭斯特定理，我们需要每隔三十秒对秒针进行采样，在 a 图中，样本点依次为 12、6、12、6、12，并不知道时钟是向前走还是向后走。c 图中，以低于奈圭斯特率的采样率进行采样，虽然时钟向前走，单接受方认为时钟在向后走。

![sampling of a clock with only one hand](/images/16/10/sampling-of-a-clock-with-only-one-hand.png)