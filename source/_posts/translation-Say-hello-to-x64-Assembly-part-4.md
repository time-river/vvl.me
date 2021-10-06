---
title: '译文: Say hello to x64 Assembly [part 4]'
date: 2016-08-14 00:46:08
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

看一看一些字符串与字符串操作。

## 字符串反转(Reverse string)

既然谈论汇编语言，当然不能遗漏__字符串(string)__数据类型，通常来说，它就是我们使用单位字节长度的数组来构建字符串数据类型。写一些简单的栗子：我们将会定义字符串数据，反转他们后输出。这个例子非常简单，但也很有用，尤其学习一门新语言的时候。

```nasm
section .data
    SYS_WRITE equ 1
    STD_OUT   equ 1
    SYS_EXIT  equ 60
    EXIT_CODE equ 0

    NEW_LINE db 0xa
    INPUT    db "Hello world!"
```

四个常量：

* SYS_WRITE -- 'write' syscall number
* STD_OUT   -- 标准输出文件描述符(stdout file descriptor)
* SYS_EXIT  -- 'exit' syscall number
* EXIT_CODE -- exit code
定义了一些变量：
* NEW_LINE  -- 换行符号(\n)
* INPUT     -- 输入字符串

接下来为缓存(buffer)定义 **bss** 段，这里将会放入被反转的字符串：

```nasm
section .bss
    OUTPUT  resb  12
```

现在有了存放数据与缓存的地方，那么轮到放代码的 __text__ 段啦。以 **_start** 开始吧：

```nasm
_start:
    mov rsi, INPUT
    xor rcx, rcx
    cld
    mov rdi, $ + 15
    call calculateStrLength
    xor rax, rax
    xor rdi, rdi
    jmp reverseStr
```

这里是一些新的东西。看看他们怎么工作：在第二行先把 **INPUT** 的地址放入 *si* 寄存器。为了输出字符串，**rcx** 赋值为 0，它是计算字符串长度的计数器。第四行见到了 *cld* 操作符。它重置了 **df** 标志，使之为 0。因为稍后需要利用它计算字符串长度 -- 会遍历字符串中的字符，当 **df** 标志为 0 时，会从左到右处理字符。接下来调用了 **calculateStrLength** 函数。对了，遗忘了第五行的`mov rdi, $ + 15`，不用着急，会在稍后提到。先看一看 **calculateStrLength**：

```nasm
calculateStrLength:
    ;; check is it end of string
    cmp byte [rsi], 0
    ;; if yes exit from function
    je exitFromRoutine
    lodsb
    ;; push symbol to Stack
    push rax
    inc rcx
    ;; loop again
    jmp calculateStrLength
```

正如它的名字，这函数的功能是计算字符串的长度并把长度存储到 *rcx*。首先检查 *rsi* 是否为 0(NULL)，若为 0 则意味着计算结束。接下来是 *lodsb*，它仅仅把 1 字节放入了 **al** 寄存器(16 位寄存器 **ax** 的低 8 位)，并改变了 *rsi* 指针。当执行 **cld** 指令后，每次执行 **lodsb** 都会把 **rsi** 从左至右移动一字节(译者注：x86 体系是小端存储)，这样便可以移动字符了。之后，把 *rax* 放入堆栈，这样堆栈中便包含了字符串中的一个字符(**lodsb** 把一字节从 **si** 放入了 **al**，**al** 为 **rax** 的低 8 位)。我们怎能把字符放入堆栈呢？这必须得记得堆栈是怎么工作的：它的工作原理为 LIFO(后入先出)。我们把字符依次从 **si** 放入堆栈。这样，最后一个字符已定在堆栈的顶端。然后仅仅需要从堆栈从弹出(pop)字符，再写入 **OUTPUT** 缓存就行了。`push rax`后，自增了计数器(**rcx**)，循环执行这段代码。

把所有的字符放入堆栈后，跳转到 **exitFromRoutine**，再返回 **_start**。怎么做呢？有 **ret** 指令：

```nasm
exitFromRoutine:
    ;; return to _start
    ret
```

但是它并不会工作。为啥？这很诡异。要知道我们的确在 **_start** 中调用了 **calculateStrLength**.但是在调用的过程中究竟发生了什么呢？起初，所有函数参数从右至左放入了堆栈，之后返回了地址，它也放入了堆栈。因此函数在调用结束后知道从哪里返回。但是看一下 **calculateStrLength**，我们把符字符串中的字符都放入了堆栈，现在的堆栈顶部并不是要返回的地址，函数调用结束后当然不知道从哪里返回了。现在怎么办呢？先看一看前面被忽略的诡异的指令吧：

```nasm
    mov rdi, $ + 15
```

开始前：

* $ -- 返回一地址，这地址是这条汇编语句在内存中的位置
* $$ -- 也返回一地址，这地址为当前段(section)的起点

利用`mov rdi, $ + 15`我们便得到了一个地址，但为啥加了 15？我们需要知道 **calculateStrLength** 的下一条语句的地址。现在看一看使用 objdump(译者注：反汇编工具) 查看文件后的结果吧：

```log
$objdump -D reverse

reverse:     file format elf64-x86-64

Disassembly of section .text:

00000000004000b0 <_start>:
  4000b0:   48 be 41 01 60 00 00    movabs $0x600141,%rsi
  4000b7:   00 00 00
  4000ba:   48 31 c9                xor    %rcx,%rcx
  4000bd:   fc                      cld
  4000be:   48 bf cd 00 40 00 00    movabs $0x4000cd,%rdi
  4000c5:   00 00 00
  4000c8:   e8 08 00 00 00          callq  4000d5 <calculateStrLength>
  4000cd:   48 31 c0                xor    %rax,%rax
  4000d0:   48 31 ff                xor    %rdi,%rdi
  4000d3:   eb 0e                   jmp    4000e3 <reverseStr>
```

瞧，第十二行(`mov rdi, $ + 15`)的命令占用了 10 字节(译者注：C8h-BEh=10)，第 16 行调用函数占用了 5 字节(译者注：CDh-C8h=5)，加在一起不就是 15 字节？这地址就是我们需要的返回地址。这样便可以把 *rdi* 的值抛进堆栈，再从函数中返回至 **_start**：

```nasm
exitFromRoutine:
    ;; push return address to stack again
    push rdi
    ;; return to _start
    ret
```

调用 **calculateStrLength** 后，**rax** 和 **rdi** 都写入了 0，之后跳转到 **reverseStr**。它是这样的：

```nasm
reverseStr:
    cmp rcx, 0
    je printResult
    pop rax
    mov [OUTPUT + rdi], rax
    dec rcx
    inc rdi
    jmp reverseStr
```

这里我们检查了我们的字符串计数器，若为零则表示我们已经把所有的字符写入了缓存，现在可以打印了。不为零就从堆栈中弹出字符，放入 **rax**，再写入 **OUTPUT** 缓存。之后自增 **rdi** ，移动到 **OUTPUT** 缓存的下一个位置，同时自字符串长度计数器自减，程序跳转到开始处。

执行完 **reverseStr** 后，就已经把字符串反转了，反转后的字符串存放在 **OUTPUT** 缓存中。该用新的一行输出他们了：

```nasm
printResult:
    mov rdx, rdi
    mov rax, 1
    mov rdi, 1
    mov rsi, OUTPUT
    syscall
    jmp printNewLine

printNewLine:
    mov rax, SYS_WRITE
    mov rdi, STD_OUT
    mov rsi, NEW_LINE
    mov rdx, 1
    syscall
    jmp exit
```

这样退出：

```nasm
exit:
    mov rax, SYS_EXIT
    mov rdi, EXIT_CODE
    syscall
```

就这么多，现在可以编译我们的程序了：

all:

```makefile
    nasm -g -f elf64 -o reverse.o reverse.asm
    ld -o reverse reverse.o

clean:
    rm reverse reverse.o
```

译者注：Makefile， GNU Make 工具使用的文件。

这是执行结果：

![part 4 run](/images/16/08/part-4-run.png)

## 字符串操作(String operations)

肯定有许多其他的字符串/比特(string/bytes)相关的指令操作啦：

* REP -- 当 rcx 不为零时重复执行
* MOVSB -- 拷贝一比特大小的字符(MOVSW, MOVSD 等)
* CMPSB -- 比特大小的字符比较
* SCASB -- 比特大小的字符输入
* STOSB -- 存储一比特大小的字符至某寄存器

## 结语

此系列第四篇。下一节谈论 nasm 的宏(macroses)。

## [原文](http://0xax.blogspot.com/2014/11/say-hello-to-x64-assembly-part-4.html) / [源码](https://github.com/0xAX/asm/tree/master/strings)

译者注:
看不懂/忘记一些命令？尤其是 REP / MOVSB 这些？为何不 Google 之？[Intel Assemble Instruction Set](http://www.skywind.me/maker/intel.htm) 的确是一个好地方。
