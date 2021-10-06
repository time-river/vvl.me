---
title: '译文: Say hello to x64 Assembly [part 8]'
date: 2016-08-18 14:42:40
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

## 浮点数(floating point numbers)

在这里会看到非整数在汇编中是怎样工作的。计算机中有一对与浮点数的工作有关的东西：

* [浮点运算器(FPU)](https://zh.wikipedia.org/wiki/%E6%B5%AE%E7%82%B9%E8%BF%90%E7%AE%97%E5%99%A8)  
* [SSE指令集](https://zh.wikipedia.org/wiki/SSE)

首先，我们看一下浮点数是怎么存储在内存中的。浮点数的类型有这么几种：

* 单精度(single-precision)
* 双精度(double-precision)
* 双精度拓展(double-extended precision)

Intel 的 *64-ia-32-architecture-software-developer-vol-1-manual* 中有这么一个描述：

> 这些数据类型的数据格式符合 <a href="https://zh.wikipedia.org/wiki/IEEE_754">IEEE Standard 754 </a> 对二进制浮点算术的规范。

单精度浮点数在内存中这么表示：

* 符号(sign) -- 1 比特
* 指数(exponent) -- 8 比特
* 尾数(mantissa) -- 23 比特

举个栗子吧，这么一个数：

---
sign | exponent | mantissa
--- | --- | ---
0 | 00001111 | 110000000000000000000000

指数是有符号的 8 比特数据，有符号整数范围为 -128 - 127，无符号整数则为 0 - 255。符号位为 0，表示的数为正，否则为负。指数若编码为 00001111b，十进制表示为 15；对单精度浮点数而言，指数的偏移量为 127，换句话说，实际的指数值应该这么得到： **eponent - 127**，或者 **15 - 127 = -112**。在尾数中，规约形式的浮点数的整数部分始终为 1，因此它被省略，只记录小数部分，因此尾数的二进制数实际上为 **1,110000000000000000000000**。这个数值十进制表示为：

```md
value = mantissa * 2^-112
```

双精度浮点数为 64 比特大小：

* 符号 -- 1 bit
* 指数 -- 11 bits
* 尾数 -- 52 bits

若转换为十进制，可以这么计算：

```md
value = (-1)^sign * (1 + mantissa / 2 ^ 52) * 2 ^ exponent - 1023)
```

双精度拓展为 80 比特：

* sign -- 1 bit
* exponent -- 15 bits
* mantissa -- 112 bit

更多信息在[这里](https://zh.wikipedia.org/wiki/IEEE_754)。

注：自己感觉作者说得不清。754 标准都差不多，以单精度实数为例：

```md
+------------+-------------------+----------------------+
| sign 1 bit |  exponent 8 bits  |   mantissa 23 bits   |
+------------+-------------------+----------------------+
```

一个单精度实数为 32 bits 大小。比特位使用情况如上图。符号为 0 代表正数，1 代表负数。指数使用移码表示，不过偏移量为 127 而不是通常的 128。尾数使用原码表示(忽略符号，或者叫浮点数绝对值的原码)，与[二进制科学计数法](https://zh.wikipedia.org/wiki/%E7%A7%91%E5%AD%A6%E8%AE%B0%E6%95%B0%E6%B3%95)形似，区别为：因整数部分始终为 1，被省略，尾数表示的是小数部分。

## x87 FPU

x87 浮点运算器(Floating-Point Unit, FPU)提供了高性能的浮点数运算。它支持浮点数、整数与压缩 BCD 整数类型与浮点处理算法。x87 提供了这些指令集：

* 数据传送指令(Data transfer instructions)
* 基本算术指令(Basic arithmetic instructions)
* 比较指令(Comparison instructions)
* 载入常量指令(Load constant instructions)
* 超越指令(Transcendental instructions)
* x87 FPU 控制指令(x87 FPU control instructions)

在这肯定不会看到 x87 提供的所有指令，想要获取更多的信息，*64-ia-32-architecture-software-developer-vol-1-manual Chapter 8* 欢迎您。这里有一对数据传输指令：

* FDL -- 载入浮点数
* FST -- 存储浮点数(在 ST(0) 寄存器)
* FSTP -- 存储浮点数与弹出(在 ST(0) 寄存器)

算术指令：

* FADD -- 浮点数加法运算
* FIADD -- 浮点数与整数加法运算
* FSUB -- 浮点数减法运算
* FISUB -- 浮点数减法，从浮点数中减去整数
* FABS -- 取浮点数绝对值
* FIMUL -- 浮点数与整数乘法
* FIDIV --  device integer and floating point(我猜，含义是除数为整数或浮点数)

FPU 有 10 字节大小的寄存器，这些寄存器构成一个环形堆栈。堆栈的顶端为寄存器 **ST(0)**，其余的寄存器为 ST(1), ST(2)...ST(7)。当使用浮点数的时候便会使用他们。例如：

```nasm
section .data
    x dw 1.0

fld dword  [x]
```

把 **x** 的值放入堆栈。x 可以为 32 bits, 64 bits, 80 bits。它的工作方式就像常用的堆栈一样。若用 **fld** 把另一个数 y 放入堆栈，x 的值会在 ST(1)，y 会在 ST(0)。FPU 指令可以使用这些寄存器，例如：

```nasm
;;
;; adds st0 value to st3 and saves it in st0
;;
fadd st0, st3

;;
;; adds x and y and saves it in st0
;;
fld dword [x]
fld dword [y]
fadd
```

这有个简单的栗子。我们知道圆的半径，求它的面积：

```nasm
extern printResult

section .data
    radius      dq 1.7
    result      dq  0

    SYS_EXIT    equ 60
    EXIT_CODE   equ 0

section .text
    global _start

_start:
    fld     qword [radius]
    fld     qword [radius]
    fmul

    fldpi
    fmul
    fstp    qword [result]

    mov     rax, 0
    movq    xmm0, [result]
    call    printResult

    mov     rax, SYS_EXIT
    mov     rdi, EXIT_CODE
    syscall
```

它是这么工作的：data section 里预定义了 radius(半径) 与 result，result 将会存储运算结果。接着是两个用于调用系统退出函数的常量。接下来是程序的入口 -- **_start**。使用 **fld** 指令把 radius 的值放入 **st0** 与 **st1**，并使用 **fmul** 使他们相乘。这样操作之后，在 **st0** 寄存器中便有了 radius 与 radius 相乘的结果。接着，使用 **fldpi** 指令载入 **π** 到 st0 寄存器，此时 radius*radius 的结果会在 st1 寄存器中。**fmul** 指令作用在 st0(值为 π) 与 st1(radius*radius 的结果)，结果会存放在 st0。好了，现在在 st0 寄存器中便有了圆型的面积，可以使用 **fstp** 指令把结果放入 **result** 了。下一部分是我们把结果传入 C 函数并调用它。还知道我们[先前](/2016/08/translation-Say-hello-to-x64-Assembly-part-7/#在汇编中调用-C)怎样在汇编中调用 C 函数吗？我们需要知道 x86_64 体系是怎样传参的，通常我们使用 rdi(arg1), rsi(arg2) 等传参，但是这里是浮点数。不过有这些特殊的寄存器： 由 [sse](https://zh.wikipedia.org/wiki/SSE)提供的 **xmm0 - xmm15**。首先我们需要把 **xmmN** 寄存器的序列号放入 rax(这里为 0)，之后把结果放入 **xmm0** 寄存器。现在，我们可以调用 C 函数来打印了：

```c
#include <stdio.h>

extern int printResult(double result);

int printResult(double result) {
    printf("Circle radius is - %f\n", result);
    return 0;
}
```

编译他们：

```makefile
build:
    gcc  -g -c circle_fpu_87c.c -o c.o
    nasm -f elf64 circle_fpu_87.asm -o circle_fpu_87.o
    ld   -dynamic-linker /lib64/ld-linux-x86-64.so.2 -lc circle_fpu_87.o  c.o -o testFloat1

clean:
    rm -rf *.o
    rm -rf testFloat1
```

执行：

![float fpu](/images/16/08/float_fpu.png)

## 结语

这是这系列文章的第八篇了。这里我们介绍了浮点数在汇编中是怎样使用的。

## [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-8.html) / [源码](https://github.com/0xAX/asm/tree/master/float)
