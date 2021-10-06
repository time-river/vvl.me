---
title: '译文: Say hello to x64 Assembly [part 7]'
date: 2016-08-17 15:39:10
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

## C 与汇编的嵌套

Say hello to x86_64 Assembly 的第七篇咯，这里会看一下怎么在 C 中使用汇编。
事实上有三种方法：

* 在 C 中调用汇编(Call assembly routines from C code)
* 在汇编中调用 C(Call c routines from assembly code)
* 在 C 中使用内嵌汇编(Use inline assembly in C code)

下面写三个简单的 Hello world 程序来演示这些。

## 在 C 中调用汇编

```c
#include <string.h>

int main() {
    char* str = "Hello World\n";
    int len = strlen(str);
    printHelloWorld(str, len);
    return 0;
}
```

可以看到 C 代码中定义了两个变量：*Hello world* 字符串与其长度。接下来调用 **printHelloWorld** 这个汇编函数，并且把这两变量当作参数传递给了它。因为我们使用的是 x86_64 Linux 系统，所以必须知道 x86_64 Linux 是怎样传参的，这样我们便知道怎么写 **printHelloWorld** 了。当我们调用函数的时候，通过通过 rdi, rsi, rdx, rcx, r8, r9 六个通用寄存器传递前六个参数，其余的通过堆栈传递。这样我们便可以从 rdi 与 rsi 中获得前两个参数，从而进行系统调用，再通过 **ret** 指令返回：

```nasm
global printHelloWorld

section .text
printHelloWorld:
    ;; 1 arg
    mov r10, rdi
    ;; 2 arg
    mov r11, rsi
    ;; call write syscall
    mov rax 1
    mov rdi 1
    mov rsi r10
    mov rdx, r11
    syscall
    ret
```

OK，这么编译(Makefile 文件)：

```makefile
build:
    nasm -f elf64 -o casm.o casm.asm
    gcc casm.o casm.c -o casm
```

## 内嵌汇编

这方法允许在 C 中直接使用汇编代码，不过有特殊的语法。大概是这样的：

```nasm
asm [volatile] ("assembly code" : output operand: input operand : clobbeers);
```

在 gcc 的文档中，**volatile** 关键词：

> 典型的 asm 拓展语法的使用是操作输入来产生输出。然而，你的 asm 语法可能带来副作用。若这样的话，你可能需要使用 the volatile qualifier 来禁止某些优化。

每个操作数都由约束字符串修饰，他们在 C 表达式中被括号括起来。这里有一些约束规范：

* r -- 在通用寄存器中变量的值保持不变
* g -- 任何寄存器、内存或者立即数都被允许，除非那些寄存器并不是通用寄存器
* f -- 浮点寄存器
* m -- 内存操作数被允许操作任何种类的地址，这些地址都是一般机器支持的

hello world 是这样：

```c
#include <string.h>

int main() {
    char* str = "Hello World\n";
    long len = strlen(str);
    int ret = 0;

    __asm__("movq $1, %%rax \n\t"
            "movq $1, %%rdi \n\t"
            "movq %1, %%rsi \n\t"
            "movl %2, %%edx \n\t"
            "syscall"
            : "=g"(ret)
            : "g"(str), "g" (len));

    return 0;
}
```

同先前的栗子一样，内嵌汇编也定义了同样的两个变量。首先，正如平时写 Hello world 汇编程序一样，我们把 1 放入 **rax** 与 **rdi** 寄存器(写入系统调用标号与标准输出)。接下来对 **rsi** 与 **rdi** 做了相似的操作，但第一个操作数开始于 **%** 标志，而不是 **$**。这是因为 %1 代指的是 **str**，它是要输出的字符串，而字符串长度(**len**)则是 %2，这样我们便利用 %n 符号把 **str** 与 **len** 放入了 **rsi** 与 **sdi**，n 为 output operand。**%%** 为寄存器的前缀。

> 这是为了帮助 GCC 区分操作数与寄存器。操作数有单独的 % 前缀。

这么编译：

```makefile
build:
    gcc casm.c -o casm
```

关于 GCC 的所有内嵌汇编文档，可以在[这](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)找到。

## 在汇编中调用 C

最后一部分。打印一个 Hello world 的函数：

```c
#include <stdio.h>

extern int print();

int print() {
    printf("Hello World\n");
    return 0;
}
```

可以在汇编代码中定义这个外部函数，然后正如先前文章中使用 **call** 指令一样调用它：

```nasm
global _start

extern print

section .text

_start:
    call print
    mov rax, 60
    mov rdi, 0
    syscall
```

编译：

```makefile
build:
    gcc -c casm.c -o c.o
    nasm -f elf64 casm -o casm.o
    ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 -lc casm.o c.o -o casm
```

## 结语

...

## [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-7.html) / [源码](https://github.com/0xAX/asm/tree/master/casm)
