---
title: '译文: Say hello to x64 Assembly [part 6]'
date: 2016-08-16 20:08:19
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

Say hello to x86_64 Assembly 系列的第六篇。这里会看一看 AT&T 汇编语法。先前我们使用的都是 [nasm](http://nasm.us/) 汇编工具，但也有其他不同语法的汇编工具，比如 [fasm](http://flatassembler.net/)，[yasm](http://yasm.tortall.net/)。接下的的部分我们会看到 gas(GNU 汇编工具)及其与 nasm 语法的不同之处。GCC 使用 GNU 汇编工具。因此若查看下列代码的汇编输出：

```c
#include <unistd.h>

int main(void){
    write(1, "Hello World\n", 15);
    return 0;
}
```

会看到这样的输出：

```gas
.file    "test.c"
.section    .rodata
.LC0:
.string "Hello World\n"
.text
.globl  main
.type   main, @function
main:
.LFB0:
.cfi_startproc
pushq   %rbp
.cfi_def_cfa_offset 16
.cfi_offset 6, -16
movq    %rsp, %rbp
.cfi_def_cfa_register 6
movl    $15, %edx
movl    $.LC0, %esi
movl    $1, %edi
call    write
movl    $0, %eax
popq    %rbp
.cfi_def_cfa 7, 8
ret
.cfi_endproc
.LFE0:
.size   main, .-main
.ident  "GCC: (Ubuntu 4.9.1-16ubuntu6) 4.9.1"
.section    .note.GNU-stack,"",@progbits
```

译者注：文件命名为 test.c，`$gcc -o test.s -S test.c`，可得之。

恩，看起来的确大不相同，接着看吧。

## AT&T 语法

###　段(Section)

我并不知道你的习惯是怎样的，但我的汇编程序通常起于段定义。一个简单的栗子：

```nasm
.data
    //
    // initialized data definition
    //
.text
    .globl _start

_start:
    //
    // main routine
    //
```

可以注意到两点微小的不同：

* 段定义起于 .symbol
* main routine 定义以`.globl`开始，并非我们使用 nasm 写的 `global`

gas 定义数据也使用不同的指令：

```gas
.section .data
    // 1 byte
    var1: .byte 10
    // 2 byte
    var2: .word 10
    // 4 byte
    var3: .int 10
    // 8 byte
    var4: .quad 10
    // 16 byte
    var5: .octa 10

    // 把字符串(不自动追加 NULL)放到连续的地址上
    str1: .asci "Hello world"
    // 同上面一样，但是字符串末尾存在 NULL
    str2: .asciz "Hello world"
    // Copy the characters in str to the object file -- (;_; 没明白
    str3: .string "Hello world"
```

### 操作数指令(Operands order)

使用 nasm 写汇编程序的时候，对数据操作有下列通用的语法：

```nasm
mov destination, source
```

而使用 GNU 汇编工具，顺序却相反：

```gas
mov source, destination
```

举个栗子：

```nasm
;;
;; nasm syntax
;;
mov rax, rcx

//
// gas syntax
//
mov %rcx, %rax
```

寄存器起始于 __%__ 标志；若使用立即数(direct operands)，需要使用 __$__。

```gas
movb $10, %rax
```

### 操作数字长与操作语法(Size of operands and operation syntax)

有时候需要指定操作数的字长，比如 64 位寄存器的第一个字节，使用下面的语法(译者注：这是 nasm 语法)：

```nasm
mov ax, word [rsi]
```

这是在 gas 中的一种操作方法。并不需要在操作数中定义大小，而是在指令中：

```gas
movw (%rsi), %ax
```

GNU 汇编工具的操作码有 6 种后缀：

* b -- 1 字节操作数
* w -- 2 字节操作数
* l -- 4 字节操作数
* q -- 8 字节操作数
* t -- 10 字节操作数
* o -- 16 字节操作数

这规则不仅仅用于 `mov` 指令，也用于所有类似于 addl, xorb, cmpw 等类似的指令。

### 内存访问(Memory access)

注意到在前面的 gas 栗子中使用了 **()** 代替了 nasm 栗子中的 **[]**。在 gas 中取消对值的访问(寻址)为 **(%rax)**，例如：

```gas
movq -8(%rbp), %rdi

movq 8(%rbp), %rdi

;; nasm

mov rdi, [rbp - 8]

mov rdi, [rbp + 8]
```

### 跳转(Jumps)

GNU 汇编工具支持下列的远程函数调用与跳转(far functions call and jumps)操作：

* lcal $section, $offset
  * 远程跳转(far jump) -- 跳转到一个指令，这个指令不在当前代码段(code segment)中，而是位于不同的段(segment)，也具有相同的特权等级，有时也被成为段间跳转(intersegment jump)。

### 注释(comments)

GNU 编译工具支持下面三种形式的注释：

* {%raw%}#{%endraw%} -- 单行注释
* // -- 单行注释
* {%raw%}/* */{%endraw%} -- 多行注释

## 结语

...

## [原文](http://0xax.blogspot.com/2014/12/say-hello-to-x8664-assembly-part-6.html)

译者的罗嗦：讲真，我也没看懂多少，我又没接触过 gas，撇嘴 =_= 意思应该差不多，因为我参考了[这](https://www.ibm.com/developerworks/cn/linux/l-assembly/)。
