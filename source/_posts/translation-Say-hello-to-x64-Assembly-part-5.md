---
title: '译文: Say hello to x64 Assembly [part 5]'
date: 2016-08-15 15:51:19
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

Say hello to x86_64 Assembly 系列的第五篇，这篇会看到宏(macros)。但这与 x86_64 并没有关系，仅仅是 [nasm](http://nasm.us/) 这种汇编工具与其预处理的。若有兴趣的话可以接着阅读。

## 宏(Macros)

NASM 支持以下两种形式的宏：

* 单行(single-line)
* 多行(multiline)

所有单行定义的宏必须以`%define`开始，像下面的形式一样：

```nasm
%define macro_name(parameter) value
```

Nasm 宏的行为与 C 非常相似。比如，我们可以这样创建单行宏：

```nasm
%define argc rsp + 8
%define cliArg1 rsp + 24
```

之后可以在代码中这么使用他们：

```nasm
;;
;; argc 将会被 扩展为 rsp + 8 (argc will be expanded rsp + 8)
;;
mov rax, [argc]
cmp rax, 3
jne .mustBe3args
```

多行宏以`%macro`起始，以`%endmacro`结束。通常的形式是这样的：

```nasm
%macro number_of_parameters
    instruction
    instruction
    instruction
%endmacro
```

例如(译者注：1 代表参数的个数为 1)：

```nasm
%macro bootstrap 1
    push ebp
    mov ebp, esp
%endmacro
```

然后这样使用它：

```nasm
_start:
    bootstrap
```

再举几个 **PRINT** 宏的例子：

```nasm
%macro PRINT 1
    pusha
    pushf
    jmp %%astr
%%str   db %1, 0
%%strln equ $-%%str
%%astr: _syscall_write %%str, %%strln
popf
popa
%endmacro

%macro _sytcall_write 2
    mov rax, 1
    mov rdi, 1
    mov rsi, %%str
    mov rdx, %%strln
    syscall
%endmacro
```

让我们浏览一遍、懂得他们是怎样工作的：第一行定义了 PRINT 宏，参数为 1。之后利用 **pusha** 与 **pushf** 指令把所有数据与标志(flag, pushf 指令)入栈。然后跳向 **%%astr** 标签继续执行。注意到宏定义的所有的标签都以 **%%** 开始。现在转向 **_syscall_write** 宏，它的参数为 2。**_syscall_write** 是怎么执行的呢？是否还记得我们使用的 **write** 系统调用来打印字符至标准输出。它长成这样：

```nasm
;; write syscall number
mov rax, 1
;; file descriptor, standard output
mov rdi, 1
;; message address
mov rsi, msg
;; length of message
mov rdx, 14
;; call write syscall
syscall
```

在我们的 **_syscall_write** 宏中，我们先使用两条指令把 1 放入 rax(write system call number)与 rdi(stdout file descriptor)。之后把 **%%str** 放入 rsi 寄存器(指向字符串)，%%str 是局部标签，它获取 **PRINT** 宏的参数(要注意宏的参数可以使用 $parameter_number)，%%str 以 0 结束(每个字符串必须以 0 结束)。%%strln 为计算出的字符串的长度。最后进行了系统调用。

现在便可以这样使用它：

```nasm
label: PRINT "Hello World!"
```

## 有用的标准宏(Useful standard macros)

NASM 支持下列形式的标准宏：

`STRUC`

可以为数据指令(data structure)使用`STRUC`与`ENDSTRUC`。例如：

```nasm
struc person
    name: resb 10
    age:  resb 1
endstruc
```

现在可以这样为上面的指令创建一个实例：

```nasm
section .data
    p: istruc person
      at name db "name"
      at age  db 25
    iend

section .text

_start:
    mov rax, [p + person.name]
```

`%include`

可以 include 其他的汇编文件，然后跳转到这些文件中的某一行继续执行，或者调用其中的函数。

## 结语

...

译者罗嗦几句：感觉这结语没啥好翻译的，与前面都差不多；也并没学会宏，使用的话看还是再找些资料读吧。

## [原文](http://0xax.blogspot.com/2014/11/say-hello-to-x8664-assembly-part-5.html)
