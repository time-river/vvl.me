---
title: '译文: Say hello to x64 Assembly [part 3]'
date: 2016-08-14 00:46:08
categories: style
tags: ['Say hello to x64 Assembly', translation]
---

## 堆栈(Stack)

这是 Say hello to x64 Assembly 系列的第三篇，有关于堆栈的部分。堆栈是寄存器的特殊区域，它的原理为后入先出(LIFO. Last Input, First Output)。

中央处理器中有 16 个通用寄存器用于存储临时数据，他们为 RAX, RBX, RCX, RDX, RDI, RSI RBP, RSP, 与 R8 - R15。但这对海量的程序来说，这太少了。因此计算机把数据存储到堆栈中。堆栈这么使用：调用一个函数的之前，把当前的地址返回，这个地址会被复制到堆栈中。函数执行结束后，地址复制到指令计数器(commands counter, RIP)中，程序接着函数后的地址继续执行。
例如：

```nasm
global _start

section .text

_start:
    mov rax, 1
    call incRax
    cmp rax, 2
    jne exit
    ;;
    ;; Do something
    ;;

incRax:
    inc rax
    ret
```

程序刚运行时，*rax* 赋值为 1。稍后调用了函数 *incRax*，使 *rax* 自增 1，此时 *rax* 值为 2。函数执行完后，从第 8 行处继续执行，在这里比较 *rax* 与 2 的大小。从 [System V AMD64 ABI](http://www.x86-64.org/documentation/abi.pdf) 中可知，通过寄存器可以传递给函数六个参数，这些寄存器是：

* rdi -- 第一个参数
* rsi -- 第二个
* rdx -- 第三个
* rcx -- 第四个
* r8 -- 第五个
* r9 -- 第六个

更多的参数则通过堆栈传递。若我们看到这样的代码：

```c
int foo(int a1, int a2, int a3, int a4, int a5, int a6, int a7)
{
    return (a1 + a2 - a3 - a4 + a5 - a6) * a7;
}
```

前六个参数会通过寄存器传递，但是第七个会通过堆栈传递。

注: System V AMD64 ABI 若无法打开，Google 之。

## 堆栈指针(Stack pointer)

十六个寄存器中，有两个特别有趣的 -- *RSP* 和 *RBP*。*RBP* 是基址寄存器，它指向堆栈的底部。*RSP* 是堆栈指针，指向当前堆栈的顶部。

## 指令

有俩与堆栈相关的指令：

* push argument -- 堆栈指针(RSP)自增一，同时存储 argument 的值至堆栈指针指向的地方。
* pop argument -- 从当前堆栈指针指向的地方复制数据至 argument。

举一个栗子：

```nasm
global _start

section .text

_start:
    mov rax,1
    mov rdx, 2
    push rax
    push rdx

    mov rax, [rsp + 8]

    ;;
    ;; Do something
    ;;
```

这儿，我们会看到我们把 1 放入了 *rax*，2 放入了 *rdx*。然后把这些寄存器的值放入堆栈。堆栈是后入现出的。因此，堆栈现在长成这样子：

```md

                             +--------------+
                             |              |
                             |              |
                             |              |
      +-----------+          +--------------+
      |   RSP     +--------->+      2       |
      +-----------+          +--------------+
                             |      1       |
                             +--------------+
                             |              |
                             |              |
                             |              |
                             |              |
                             |              |
                             |              |
                             |              |
                             +--------------+
```

稍后，我们从堆栈中复制一个值，这个值是这样获得的： 先获取堆栈顶端的地址(*rsp* 的值) addr，把 addr + 8，获得 address，address 存储的值即为复制的值。这番工作之后，*rax* 存储的值就是 1 了。

译者注：为什么地址要 +8 呢？结果还是 1？

1. x86 体系的堆栈是向下增长的~
2. 要知道，这是 64 位计算机，还按字节寻址。

## 栗子

一个简单的程序：接收命令中的两个参数，把他们相加，然后打印结果。

```nasm
section .data
    SYS_WRITE   equ 1
    STD_IN      equ 1
    SYS_EXIT    equ 60
    EXIT_CODE   equ 0

    NEW_LINE    db  0xa
    WRONG_ARGC  db  "Must be two command line argument", 0xa
```

先定义 .data section。这里面是 linux 系统调用(sys_write, sys_exit 等)的 4 个常量。两字符串：一个为换行标志，一个为错误信息。
下面是 .text section:

```nasm
section .text

    global  _start

_start:
    pop rcx
    cmp rcx, 3
    jne argcError

    add rsp, 8
    pop rsi
    call str_to_int

    mov r10, rax
    pop rsi
    call str_to_int
    mov r11, rax

    add r10, r11
```

*_start* latbel 中，第一个指令从堆栈获取值，再放入 *rcx* 寄存器。对了，命令中的那些参数呢？起始，他们已经保存到了堆栈中：

* [rsp] -- 指向堆栈顶端，存储的为参数的个数
* [rsp + 8] -- 存储 argv[0]
* [rsp + 16] -- 存储 argv[1]
* ...

现在，我们得到了命令行参数的数量，并把它放入了 *rcx*，然后与 3 比较，若不相等，则跳转到 *argcError*，打印错误信息：

```nasm
argcError:
    ;; sys_write syscall
    mov rax, 1
    ;; file descritor, standard output
    mov rdi, 1
    ;; message address
    mov rsi, WRONG_ARGC
    ;; length of message
    mov rdx, 34
    ;; call write syscall
    syscall
    ;; exit from program
    jmp exit
```

再返回到 *_start_*。我们仅仅传递两个参数进去，却为何与 3 比较呢？这太简单啦。第一个参数是程序的名字，之后所有的参数都是命令行输入、要传递给程序的参数。现在我们若传递了两个参数，那么就会到第 10 行继续执行。在这里，我们让 *rsp* 偏移了 8 个单位距离，这个地址保存的值是传入程序的第一个参数，使用 *pop* 把值放入 *rsi* 寄存器中，之后调用函数把字符转换为数字。先跳过 *str_to_int*。函数执行结束后，在 *rax* 寄存器中存放的就是转换后的数了，它也被保存在 *r10* 寄存器中。然后，我们又做了相同的事情，但这次，*r10* 寄存器换成了 *r11*。这些结束后，*r10* *r11* 两个寄存器已经分别保存了一个整数。可以把他们相加了，再把结果转换为字符串，并打印。这么做(这几行代码位于 `add r10, r11`之后)：

```nasm
    mov rax, r10
    ;; number counter
    xor r12, r12  ;; r12 置 0
    ;; convert to string
    jmp int_to_str
```

在这里，我们把命令行参数相加的结果过放入 *rax* 寄存器，令 *r12* 为 0，然后跳转到 *int_to_str*。现在基本的程序框架已经有了。我们已经知道如何打印字符串，让我们看一看 *str_to_int* 与 *int_to_str* 吧。

```nasm
str_to_int:
    xor rax, rax
    mov rcx, 10
next:
    cmp [rsi], byte 0
    je  return_str
    mov bl, [rsi]
    sub bl, 48
    mul rcx
    add rax, rbx
    inc rsi
    jmp next

return_str:
    ret
```

*str_to_int* 中，起初，我们令 *rax* 为 0，*rcx* 为 10。再看看 *next*：正如你看到的栗子那样(在第一次调用 *str_to_int* 之前)，我们把 *argv[1]* 从堆栈放入 *rsi*。现在需要把 *rsi* 的第一个字节 与 0 比较，这是因为我们返回的时候，每一个字符串总是以 NULL 符号结束。若非零则拷贝 *rsi* 的值到 *bl* 寄存器(一个字节大小)，然后减去 48。在 ASCII 码中，48 代表 0。减去 48 之后便可以得到字符串代表的整数值。然后把 *rax* 与 *rcx*(它的值为 10) 相乘，自增 *rsi* 来获取下一比特，继续循环。算法非常简单。举个栗子，若 *rsi* 指向 *'5' '7' '6' '\000'* 字符串序列，那么将会是下面的步骤：

```md
1. rax = 0
2. 获取第一个比特 -- 5，把它放入 rbx
3. rax * 10 --> rax = 0 * 10
4. rax = rax + rbx = 0 + 5
5. 获取第二个比特 -- 7，放入 rbx
6. rax * 10 --> rax = 5 * 10 = 50
7. 继续循环，直至 rsi 不为 \000
```

调用 *str_to_int* 后，我们得到了一个整数，存储在 *rax* 中。现在看一看 *int_to_str*:

```nasm
int_to_str:
    mov rdx, 0
    mov rbx, 10
    div rbx
    add rdx, 48
    add rdx, 0x0
    push rdx
    inc r12
    cmp rax, 0x0
    jne int_to_str
    jmp print
```

这里，我们把 0 放入 *rdx*、把 10 放入 *rbx* 后，执行除法，被除数为 *rbx*。若浏览过调用 *str_to_int* 之前的代码，我们会知道 *rax* 包含了一个整数 -- 两个命令行参数的和。使用这个指令(*div*)做除法运算，被除数 *rax*，除数为 *rbx*，余数存放进 *rdx*，商放在 *rax*。下一步，*rdx* 与 48 和 0x0 相加。与 48 相加，得到其 ASCII 编码，同时所有的字符串必须以 0x0 结束。之后，保存这个字符至堆栈中，*r12*(初始为 0，这是在 *_start* 中定义的)自增后，*rax* 与 0 比较。若比较结果为 0，它意味着整数转换为了字符串。这个算法的演示如下：比如一个整数 23

1. 123 / 10. rax = 12; rdx = 3
2. rdx + 48 = "3"
3. 把 "3" 丢进堆栈(push "3" to stack)
4. rax 与 0 比较，结果不为 0
5. 12 / 10. rax = 1; rdx = 2
6. rdx + "48" = "2"
7. 把 "2" 丢进堆栈(push "2" to stack)
8. rax 与 0 比较，若结果为 0 则结束函数，此时堆栈中便存储了 "2" "3" 等字符。

*int_to_str* 与 *str_to_int* 实现了整数与字符串的相互转换。现在字符串形式的两数之和保存在堆栈中。可以这样打印他们：

```nasm
print:
    ;;;; 计算数的长度
    mov rax, 1
    mul r12
    mov r12, 8
    mul r12
    mov rdx, rax
    ;;;;

    ;;;; print sum
    mov rax, SYS_WRITE
    mov rdi, STD_IN
    mov rsi, rsp
    ;; call sys_write
    syscall
    ;;;;

    jmp exit
```

我们已经知道怎样利用 *sys_write* 打印字符串了，但是这还是一个有趣的地方。我们必须计算字符串的长度。看一看 *int_to_str* 吧，每一次迭代后 *r12* 都自增一次，*r12* 的值肯定为参数和的字符个数啦。必须将它乘以 8(因为我们把每个字符都放进了堆栈)，这个值为我们需要打印的字符串长度。之后我们每次把 1 放入 *rax*(sys_write number)，1 放入 *rdi*(stdin -- 译者注: 似乎写错了唉，应该是 stdout)，指向堆栈顶端的指针放入 *rsi*(字符串的开始)。随后这样结束我们的程序：

```nasm
exit:
    mov rax, SYS_EXIT
    ;; exit code -- 原文这里错误
    mov rdi, EXIT_CODE
    syscall
```

恩，就这么多。

译者注: 写下我的疑惑及答案；恩，还包括一些留言。

1. 关于 *[mul](http://www.skywind.me/maker/intel.htm#810)* *[div](http://www.skywind.me/maker/intel.htm#400)* -- 依据我找到的资料，他们的语法形式就是那么奇怪。寻址方式应该都属于隐含寻址。  
2. 堆栈向上增长还是向下增长？ -- 的确是向下增长的~这样便可以理解`add   rsp, 8`的含义了。
3. 这是我的一个疑惑：text section 区域的 *str_to_int* 中，怎么执行了`mov rcx, 10`后就执行`cmp [rsi], byte 0`了呢？ -- 在 text section 区域，若没有遇到跳转指令，是按顺序执行的；这俩行之间的确没有跳转~至于`next:`，完全是打个 label，后面有`jmp next`呢。
4. `ret` -- 使用`ret`指令使处理器返回到被调用的地方继续执行。
5. `cmp [rsi], byte 0` -- [rsi]，代表对地址取值，rsi 的值为地址。这个`byte 0`：一个字节大小的 0，查 ASCII 码表，知为 NULL。
6. `bl` -- 看[第一节](https://vvl.me/2016/08/translation-Say-hello-to-x64-Assembly-part-1/#Hello-world)的[图](/images/16/08/redisters.png)。
7. `print:` 里面的 `mov r12, 8; mul r12` -- 加不加这俩指令，我在我的计算机上并没有看到区别。我是这样理解的：上一节 -- __堆栈(stack)__ -- 处理器对寄存器的使用有严格的计数方式。堆栈是__一块内存的连续区域__，这块内存是可寻址的特殊寄存器...x64 体系，寄存器位宽 64 bits，但是 ASCII 码只用 8 bits，x86 系列处理器是不会使用其余(64-8)比特的(恩，就是这么设计的，可以查到)。这样，打个比方，比如字符串 "22"，实际占用了 2*64 bits = 2*8*8 bits，可 2*8*6 bits 都是空的。[耸肩] 可我也不明白为啥去掉这俩指令也会正常运行。
8. (这也是在留言中提到却没回复的问题)`add rdx, 0x0`有什么含义？ -- 此篇翻译完之后，还是没有头绪~不知道就是不知道咯。

## 结语

此系列的第三篇~

## [原文](http://0xax.blogspot.com/2014/09/say-hello-to-x64-assembly-part-3.html) / [源码](https://github.com/0xAX/asm/tree/master/stack)
