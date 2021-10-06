---
title: 数据与信号
date: 2016-09-30 07:23:43
categories: style
tags: ['data communications and networking', 'physical layer and transmission medium', 'summary']
mathjax: true
---

> 为什么理想状态下数字信号的带宽是无穷大的呢？

## 模拟数据与数字数据

数据可以是模拟的也可以是数字的。模拟数据(analog data)指的是连续状态的信息，而数字数据(digital data)指的是离散状态的信息。模拟数据采用连续值，数字数据采用离散值。例如一个模拟时钟有时针、分针和秒针，他们以连续的方式给出时间信息，针的移动是连续的；报告小时和分钟的数字时钟会突然从 08:05 变成 08:06，他们以离散方式给出信息。

## 模拟信号与数字信号

表示数据的信号也可以是模拟或离散的。模拟信号(analog signal)在一个范围内可以有无穷多个取值，而数字信号(digital signal)只能有有限个数值。在数据通信过程中，通常使用周期模拟信号和非周期数字信号。

显然模拟信号可以表示数据——世界上有什么东西不可用数学方程描述呢？如果现在没有，那么将来一定会有。

傅里叶分析告诉我们，任何信号都可以分解为简单正弦波的叠加，分解后的公式叫做傅里叶级数(Fourier series)：

{% raw %}
$$
f(t) = \frac{A_0}{2} + \sum_{n=1}^N A_n \cdot \sin(2 \pi n \omega t + \phi_n)
$$
{% endraw %}

学过[《信号与线性系统分析》](https://book.douban.com/subject/1679604/)应该知道，信号可分为连续信号与离散信号，同时亦可划分为周期信号与非周期信号。就像相对论中物质的质能关系、麦克斯韦方程组中的电磁关系一样，信号在时域(time domain)与频域(frequency domain)也有相互联系，这种关系由[傅里叶变换对](https://zh.wikipedia.org/wiki/%E5%82%85%E9%87%8C%E5%8F%B6%E5%8F%98%E6%8D%A2)描述。

若信号在时域上是周期性的，分解得到的是一系列具有离散频率的信号；非周期性的时域信号则是连续频率的正弦波的组合。

单个正弦波信号可用三个参数表示：振幅(amplitude)、频率(frenquency)与相位(phase)，必然为模拟信号。

复合信号由许多简单正弦波组成，它包含的频率范围称为带宽(bandwidth)。值得注意的是，直流信号的带宽是无穷大的。下面是单位直流信号与单位阶跃函数的傅里叶变换：

{% raw %}
$$
1 \longleftrightarrow 2\pi\delta(\omega) \\
\epsilon(t) \longleftrightarrow  \pi\delta(\omega) + \frac{1}{j\omega}
$$
{% endraw %}

数字信号也能表示数据。比如 1 可以编码为正电平，0 可以编码为零电平。一个数字信号可以有多于两个电平，也就是每个电平可以承载多个比特位。它显然是一种复合模拟信号。

{% raw %}
$$
x_{square}(t) = \frac{4}{\pi} \sum_{k=1}^\infty \frac{\sin ((2k-1)\omega t)}{2k-1} = \frac{4}{\pi} \left[ \sin (\omega t) + \frac{1}{3} \sin (3 \omega t) + \frac{1}{5} \sin (5 \omega t) + \ldots \right]
$$
{% endraw %}

这个傅里叶级数表达了一个理想的方波，它由无限个不同频率的正弦波合成，下图演示了怎么利用这些正弦波来制造方波：

![synthesis square.gif](/images/16/09/synthesis_square.gif)

可以看出，带宽越大，谐波数量便越多，叠加后的正弦波也越近似于方波。
