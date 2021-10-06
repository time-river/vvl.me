---
title: '译文: Say hello to x64 Assembly 汇总'
date: 2016-08-18 20:17:29
categories: style
tags: [summary, 'Say hello to x64 Assembly', translation]
---

这里，可以找到 Say hello to x64 Assembly 系列的文章：

* [介绍](/2016/08/translation-Say-hello-to-x64-Assembly-part-1/) / [原文](http://0xax.blogspot.com/2014/08/say-hello-to-x64-assembly-part-1.html)
* [主要概念](/2016/08/translation-Say-hello-to-x64-Assembly-part-2/) / [原文](http://0xax.blogspot.com/2014/09/say-hello-to-x64-assembly-part-2.html)
* [堆栈](/2016/08/translation-Say-hello-to-x64-Assembly-part-3/) / [原文](http://0xax.blogspot.com/2014/09/say-hello-to-x64-assembly-part-3.html)
* [字符串](/2016/08/translation-Say-hello-to-x64-Assembly-part-4/) / [原文](http://0xax.blogspot.com/2014/11/say-hello-to-x64-assembly-part-4.html)
* [预处理](/2016/08/translation-Say-hello-to-x64-Assembly-part-5/) / [原文](http://0xax.blogspot.com/2014/11/say-hello-to-x8664-assembly-part-5.html)
* [AT & T 语法](/2016/08/translation-Say-hello-to-x64-Assembly-part-6/) / [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-6.html)
* [C + asm](/2016/08/translation-Say-hello-to-x64-Assembly-part-7/) / [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-7.html)
* [浮点数](/2016/08/translation-Say-hello-to-x64-Assembly-part-8/) / [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-8.html)

后记：

其实找到这系列文章已三月有余，一直念念不忘，虽已学完《计算机组成原理》、阅读《Intel 微处理器》一百余页，仍觉得还是读一遍为妙。念读书不认真，尤为英文，遂译之...译一遍终归是有收获，想到了许多不曾想到的问题。也看得出，作者前半部分写得很用心，后面却不尽人意；但对于当初的目标 -- *深入一些，从汇编的角度分析他们*，也的确够了。想学汇编已经很久了，这系列文章算是我的起点，不是终点。曾经在上学期期末读了百余页《Intel 微处理器》，我想我会复习一遍，然后继续，当然也会在此记录下我的笔记。

也许问：学了这些有什么用呢？我既不关心硬件，也对反汇编没有兴趣，编译型语言只有课堂上学习的 C、而且已经很久很久没用过了。今年年初，[Gentoo](https://gentoo.org/)安装了数十遍，它让我学会了如何解决各种引导问题；我同样想深入一些，看一看 x86 架构的处理器是怎样启动的，[这篇文章](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-1.html)给了我答案，然而并看不懂，恰好此页提供了本系列文章的一个入口 ~ 为啥不为未来的课程存储知识呢？也可以看到，今年八月份的文章有两篇关于 iptables 的 -- 为维护社区服务器所学，可学完并没有啥感觉，便有了翻译期间利用腾讯云部署各种 VPN 来实践。的确有所收获，现在欠缺的只是响应的网络知识罢了。我想，学到的时候自然会有新的认知，而不是大写的懵。

古人云：行八百里者半九十。通过留言可以看出，相当一部分人看到前几篇是兴奋的，后面却没有什么人的足迹了，也许已经不需要讨论了。这并不是我的第一次翻译，早在上年的九月份，就在尝试翻译文档。可这的确是第一次我翻译出完整的文章，而且校对了一遍(不好意思的说，有点不认真)。我的英文并不好，CET-4 压线，CET-6 仅 300 多。英文不好与读懂否有联系吗？只不过是借口罢了，更别提那些英语不错的了。

相信自己，相信未来！
