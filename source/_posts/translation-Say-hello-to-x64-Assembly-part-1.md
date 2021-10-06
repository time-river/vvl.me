---
title: '译文: Say hello to x64 Assembly [part 1]'
date: 2016-08-11 23:45:49
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

## 介绍

很容易写出这样简单的代码:

```c
#include <stdio.h>

int main(){
    int x = 10;
    int y = 100;
    printf("x + y = %d", x + y);
    return 0;
}
```

也很容易理解这段 C 代码做了什么事情。但是，在计算机的底层，这段代码是怎么工作的呢？并非每个人都可以回答这个问题，我也一样，尽管自己的高级语言写得很棒。因此我决定深入一些，从汇编的角度分析他们。我记录下我的学习过程，希望它有趣一些，也不仅仅是为了我自己。

## 准备

在开始之前，需要做一些准备工作。我使用 Ubuntu(Ubuntu 14.04.1 LTS 64 bit)系统，因此这些文章针对的是这个操作系统及体系。不同的 CPU 支持不同的指令集，使用的处理器为 *Intel Core i7 870 processor*，所有的代码也是为这个处理器写的。同时使用的是 [nasm](http://www.nasm.us/) 汇编工具。在 Ubuntu 上可以这样安装它:

```shell
$sudo apt-get install install nasm
```

nasm 的版本必须大于等于 2.0.0。我使用的 NASM 的版本为 2.10.09，于 2013.12.29 日编译完成。

译者注:

1. AMD64 架构的 CPU 是[向后兼容](https://zh.wikipedia.org/wiki/%E5%90%91%E4%B8%8B%E5%85%BC%E5%AE%B9)的，新的处理器会兼容旧的指令集。
2. 操作系统，是 Linux 系统应该没有问题。这是因为系统函数的原因，并不兼容 Windows，包括 Mac。
3. 使用 Emacs 的童鞋，原文有相应的配置文件，并没有翻译。

## x64 语法

并不会介绍全部的语法，仅仅是关于本文所需要的。

通常，NASM 程序分为两个部分:

* data section  
* text section

data section 用于声明常量。常量在运行时并不会改变。你可以声明各种常量。声明 data section 部分的语法是这样的:

```nasm
section .data
```  

text section 部分用于写代码。这部分必须开始于 `global _start`，用来告诉内核这是程序执行的起点。

```nasm
section .text
global _start
_start:
```

注释以`;`符号开始。每一行 NASM 代码是这样组成的:

```nasm
[label:] instruction [operands] [; comment]
```

`[ ]`包含在内的字段可选。一条基本的 NASM 指令包含两个部分：指令名称 + 指令的操作数。比如：

```nasm
MOV COUNT, 48 ; COUNT = 48 -- 把48放入变量COUNT中
```

## Hello world

Hello World 的汇编代码长成这样:

```nasm
section .data
    msg db  "hello, world!"

section .text
    global _start
_start:
    mov     rax, 1
    mov     rdi, 1
    mov     rsi, msg
    mov     rdx, 13
    syscall
    mov     rax, 60
    mov     rdi, 0
    syscall
```

这可长得一点也不像`printf("Hello, world!")`。我们得需要好好的理解下。

1-2 行定义了 *data* section，令`msg = "Hello, world!"`。这样就可以在代码中使用常量了。接下来声明了 *text* section，由此进入程序。程序开始于第 7 行。我们已经知道`mov`指令了，但是，*rax*, *rdi* 这类又是什么鬼？ Wikipedia 上面是这样写的：

> 中央处理器（英语：Central Processing Unit，缩写：CPU），是计算机的主要设备之一，功能主要是解释计算机指令以及处理计算机软件中的数据。

那么，CPU 操作数据，那么它从哪里获取数据呢？首先想到的是内存，但是从内存里面存取数据需要的时间太长了，因此 CPU 拥有属于自己的内存空间，叫做__寄存器__。
![register](/images/16/08/redisters.png)
可是，什么时候使用 *rax, rdx...* 呢？

* rax -- 临时寄存器；当进行系统调用的时候(syscall)，rax 必须包含 syscall number
* rdx -- 存储传递给函数的第三个参数
* rdi -- 存储传递给函数的第一个参数
* rsi -- 指向传递给函数的第二个参数

换句话说，在汇编程序中，我们仅仅进行了一次`sys_write`系统调用。它长成这样：

```c
ssize_t sys_write(unsigned int fd, const char * buf, size_t count)
```

3 个参数:

* fd -- 文件描述符。0 代表标准输入，1 代表标准输出，2 代表标准错误。
* buf -- 字符数组指针，buf 存储的内容来自被指向的文件，fd 获得这些存储的内容。
* count -- 由文件写入字符数组的字节数。

译者注:
从 C 语言的角度理解，fd 是一个文件描述符，buf 是一个字符串指针，指向将要流向 fd 的内容，count 是这个字符串的长度，单位字节(bytes)。

我们知道了 *sys_write* 有三个参数，在系统调用表(syscall table)中的标号为 1。再回过头来看一看 Hello wrold 的执行。我们在 rax 寄存器中放入 1，它意味着我们使用 *sys_write*。下一行把 1 放入 rdi 寄存器，它代表 *syscall* 的第一个参数， 1 -- 标准输出。然后我们把指针 *msg* 放进 rsi 寄存器，它是 *sys_write* 的第二个参数 buf 的值。之后，我们传递了最后一个(第三个)参数(字符串的长度)给 rdx，这是 *sys_write* 的第三个参数。现在，*sys_write* 拥有了所有的参数，这样我们就可以在第 11 行调用 *sys_write* 函数了。我们就打印出来了 "Hello, world!"。现在，我们需要正确地退出程序。令 rax 寄存器存储 60，60 是 exit 的系统调用标号。传递 0 给 rdi 寄存器，它是错误代码，0 代表我们的程序正确地退出了。以上就是 "Hello World" 的全部解释。非常简单 :) 那么，让我们编译这段程序吧。比如，我们这段程序写在 *hello.asm* 文件中。

```makefile
$nasm -f elf64 -o hello.o hello.asm
$ld -o hello hello.o
```

之后，我们拥有了可执行文件 *hello*，可以这么执行它`./hello`。

## 结语

这是这系列文章第一部分。在下一节内容我们会见到一些运算。

## [原文](http://0xax.blogspot.com/2014/08/say-hello-to-x64-assembly-part-1.html) / [源码](https://github.com/0xAX/asm/tree/master/hello)

译者注:
从留言来看，不同体系的计算机编译 hello.asm 有不同的结果。

1. 对 IA32 体系(x86 指令集 32 位版本)而言，*sys_write* 的标号为 4，而不是 AMD64 体系(x86 指令集 64 位版本)的 1。[Linux System Call Table for x86](http://docs.cs.up.ac.za/programming/asm/derick_tut/syscalls.html) & [Linux System Call Table for x86_64](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64) 有说明。
2. Mac 的系统调用标号与 Linux 并不相同。PS: 其余的对话并看不懂。
3. 是否有疑问？接着看呗。
4. 如有错误，欢迎指证。
