---
title: '译文: Say hello to x64 Assembly [part 2]'
date: 2016-08-12 23:12:04
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

## 术语和概念

第一篇文章中，不少人抱怨内容并不清晰，这也是我为啥阐述一些术语的原因。

__寄存器(Register)__ -- 寄存器在处理器内部，它是很小的存储器。数据处理占处理器的大部分工作。处理器从内存中获取数据，但是它太慢了。这就是为啥处理器拥有自己的内部存储空间，它的名字是 - 寄存器。
__小端序(Little-endian)__ -- 可以把内存想象成一个大的数组。字节(bytes)存储在里面。每个地址存储一个内存“数组”的一部分元素。每一个元素是一个字节。举个例子，我们有 4 比特数据： *AA 56 AB FF* (译者注：十六进制表示，1 字节 = 8 比特，即 8 位二进制)。小端序是下面这样：

```log
0 FF
1 AB
2 56
3 AA
```

0, 1, 2, 3 是内存地址。
__大端序(Big-endian)__ -- 大端序存储字节的形式与小端序相反。因此小端序那个栗子，在大端序中是这样：

```log
0 AA
1 56
2 AB
3 FF
```

__系统调用(syscall)__ -- 它是一个方法：用户级别的程序利用操作系统做一些事情。有系统调用表(syscall table) - [在这](http://blog.rchapman.org/post/36801038863/linux-system-call-table-for-x86-64)。
__堆栈(stack)__ -- 处理器对寄存器的使用有严格的计数方式。堆栈是一块内存的连续区域，这块内存是可寻址的特殊寄存器，比如 RSP, SS, RIP 等。后面的章节会讨论它们。
__段(section)__ -- 每个汇编程序都是由若干段组成。有下列几种段：

* data -- 用于声明初始化的数据或常量
* bss -- 用于声明非初始化变量
* text -- 程序代码在这里

__通用寄存器(general-purpose Registers)__ -- 有十六种通用寄存器 - rax, rbx, rcx, rdx, rbp, rsp, rsi, rdi, r8, r9, r10, r11, r12, r13, r14, r15。

这里并不列出所有与汇编有关的术语和概念。后面的章节也会见到一些陌生的文字，也会有相应的说明。

## 数据形式(Data Type)

基本的数据形式包括字节(bytes)，字(words)，双字(doublewords)，四字(quadwords)，双四字(double quadwords)。一个字节等于八比特，一个字是二字节，一个双字为四字节，一个四字是八字节，双四字是十六字节(128比特)。
至于整数，分为两种：无符号整数与有符号整数。无符号整数是无符号二进制数字。范围为 0 - 255(无符号1字节)，0 - 65,535(无符号1字)，0 - 2^32 - 1(无符号1双字)，0 - 2^64 - 1(无符号1四字)。有符号整数呢，-128 - 127，–32,768 - +32,767，-2^31 - 2^31 - 1，-2^63 - 2^63 - 1。

译者注：从 Intel 8088 开始，一个字规定为二字节，尽管不断更新，为了保证向后兼容，还是未改变。相关知识在[这里](https://zh.wikipedia.org/wiki/%E5%AD%97_(%E8%AE%A1%E7%AE%97%E6%9C%BA))。

## 段(Sections)

每个汇编程序都由段组成，它可以是数据段(data section)，代码段(text section)，bss 段(Block Started by Symbol section)。下面是一个用来声明初始化常量的数据段：

```nasm
section .data
    num1:   equ 100
    num2:   equ 50
    msg:    db  "Sum is correct", 10
```

3 个常量，名字分别为 num1， num2， msg，值分别为 100，500，"Sum is correct", 10。但是，*db*, *equ* 这类又是什么呢？实际上，NASM 支持大量的伪指令：

```nasm
* DB, DW, DD, DQ, DT, DO, DY, DZ -- 他们被用于声明初始化数据。例如

;; 1h, 2h, 3h, 4h 初始化为 4 字节 -- h 代表 hex，十六进制
db 0x01,0x02,0x03,0x04

;; 0x12 0x34 初始化为 1 字
dw 0x1234
```

* RESB, RESW, RESD, RESQ, REST, RESO, RESY, RESZ -- 用于声明非初始化变量。
* INCBIN -- 包含额外的二进制文件。
* EQU -- 定义常量。比如：

```nasm
;; now one is 1
one equ 1
```

* TIMES -- 重复指令或数据(将在下一节描述)。

## 算术运算

一份算术指令的清单：

* SUB -- 减法
* ADD -- 整数相加
* MUL -- 无符号乘法
* IMUL -- 符号乘法
* DIV -- 无符号除法
* IDIV -- 符号除法
* INC -- 自增
* DEC -- 自减
* NEG -- 否定

## 控制流

通常，编程语言能够改变代码执行顺序(使用 if，case，goto 等)，汇编语言同样也可以。在这里我们将会看到他们。*cmp* 指令用于比较两个数值大小，它用在条件跳转指令(conditional jump instruction)旁做决定。比如：

```nasm
;; rax 与 50 相比较
cmp rax, 50
```

*cmp* 指令仅仅比较两个数大小，但不改变他们，也不根据比较结果做任何操作。比较后，若想执行的任何操作，有条件跳转指令来帮忙。它可以是下面的任意一种：

* JE -- 若相等
* JZ -- 若为 0
* JNE -- 若不为零
* JG -- 若第一个操作数大于第二个
* JGE -- 若第一个操作数大于等于第二个
* JA -- 与 JG 功能一样，但是用于无符号数比较
* JAE -- 与 JGE 功能一样，但是用于无符号数比较

例如，用 C 这样写的 if/else：

```c
if (rax != 50) {
    exit();
} else {
    right();
}
```

用汇编这样实现：

```nasm
;; 比较 rax 与 50 的大小
cmp rax, 50
;; 若 rax 不为 50 则执行 .exit
jne .exit
jmp .right
```

无条件跳转语句的语法：

```nasm
JMP label
```

例如：

```nasm
_start:
    ;; ....
    ;; 执行一段代码，然后跳转到 .exit
    ;; ....
    jmp .exit

.exit:
    mov rax, 60
    mov rdi, 0
    syscall
```

若 *_start* 标签后有一段可执行的代码，这段代码执行完后，控制会跳转到 *.exit* 标签，然后继续执行 *.exit* 后的代码。
无条件跳转经常用于循环中。循环会在后面的章节讲到。

## 例子

一个简单的例子：两数相加，他们的和与一个预先定义的数比较，若相等，会在屏幕上打印一些东西，不相等则退出。

```nasm
 ; 初始化 data section
section .data
    ; 定义常量
    num1:   equ 100
    num2:   equ 50
    ; 初始化信息
    msg:    db "Sum is correct\n"

section .text

    global _start

 ;; 入口
_start:
    ; rax 赋值为 num1 的值
    mov rax, num1
    ; rbx 赋值为 num2 的值
    mov rbx, num2
    ; rax 与 rbx 相加，值存至 rax
    add rax, rbx
    ; rax 与 150 比较
    cmp rax, 150
    ; 若 rax 与 150 不相等，跳转至 .exit 标签
    jne .exit
    ; 若相等，跳转至 .rightSum 标签
    jmp .rightSum

; 若 sum 正确则打印信息
.rightSum:
    ;; write syscall
    mov rax, 1
    ;; file descritor, standard output
    mov rdi, 1
    ;; message address
    mov rsi, msg
    ;; length of message
    mov rdx, 15
    ;; call write syscall
    syscall
    ; exit from program
    jmp .exit

; 退出程序
.exit:
    ; exit syscall
    mov rax, 60
    ; exit
    mov rdi, 0
    ; call exit syscall
    syscall
```

译者注: 第七行代码并不正确。程序编译、链接之后，结果为`Sum is correct\`，并且没有换行。原文的评论也有提及。

1. 一个方法是改为`msg: db "Sum is correct\n"`。
2. 另一个方法是：`msg: db "Sum is correct", 10`。这是作者托管在 GitHub 代码网站上的自己的[实现](https://github.com/0xAX/asm/blob/master/sum/sum.asm)。[这](http://geekswithblogs.net/MarkPearl/archive/2011/04/13/more-nasm.aspx)是附带的链接，并没有看懂。)

这段程序做了这些事情：date section 里有 2 常量 *num1*, *num2*，变量 *msg* 值为 "Sum is correct\n"。第十四行为程序的入口。我们把 *num1, num2* 的值赋给通用寄存器 *rax*, *rbx*，然后使用 *add* 指令求和。*add* 会计算出 *rax* 与 *rbx* 的值，再把值放进 *rax*。这样，*rax* 保存的就是 *num1* 与 *num2* 相加的结果了。
num1 与 num2 相加，结果为150。那我们再看看 *cmp* 做了什么。*rax* 与 150 比较之后，程序会检查比较的结果：若 *rax* 与 150 不相等(jne 那行)，跳转到 *.exit*；若相等则跳转至 *.rightSum*。
接下来是俩 label：*.exit* 和 *.rightSum*。*.exit* 中，*rax* 赋值为 60，它是 exit 的系统调用标号，*rdi* 赋值为0， 它是退出码(exit code)。第二个 *.rightSum* 则非常简单，打印字符串 *Sum is correct\n*。若不了解他们，看一看[这篇](/2016/08/translation-Say-hello-to-x64-Assembly-part-1/)吧。

## 结语

这系列文章的第二篇。若有疑问，留言呗~

## [原文](http://0xax.blogspot.com/2014/09/say-hello-to-x64-assembly-part-2.html) / [源码](https://github.com/0xAX/asm/tree/master/sum)
