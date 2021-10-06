---
title: C 的趣(困)味(惑)
date: 2017-03-04 19:54:17
categories: style
tags: ['language', 'c']
---

选了《嵌入式程序设计》这门课，听了两节课，感觉还不错。课上，老师谈及了一些 C 语言的东西，感觉很(特)有(无)趣(语)，尝试回忆并记录一下。

## 内存对齐

```c
struct {
    char  a;
    int   b;
    float c;
} A;

printf("%zu\n", sizeof(A));
```

输出是？

先扯一点别的：`sizeof` 是一个关键字...很久很久以前，一直把它看作函数...它的返回值是 `size_t` 类型，这是啥类型？我也说不清，不是 C 的基本数据类型。支持 C99 标准的话，最好使用 `%zu` 来转义就对了，`u` means __unsigned__。Stack Overflow 上的[这](http://stackoverflow.com/questions/5943840/how-do-i-print-the-size-of-int-in-c)问题挺好。

正确的答案是 9 或 10 或 12。Why？

先看看 9 怎么来的：

* `float` 类型很简单，啥平台都是 4 Bytes
* `int` 是 2 或 4 Bytes。但只有 16 位的计算机是 2，32 / 64 位都是 4 Bytes。[这里](http://stackoverflow.com/questions/11438794/is-the-size-of-c-int-2-bytes-or-4-bytes)说的
* 至于 `char`，没见过不是 1 Byte 的
* 1 + 4 + 4 = 9
* 恭喜你，算出来了，数学真好；）

至于 10 与 12，不得不说起编译器优化。这个结构体里面的数据类型最大是 4 Bytes，所以会按照 4 字节进行内存对齐，这个术语叫做[data structure alignment](Data structure alignment)；大多数 C 编译器默认会进行此优化的，因此最终的大小就是 4*3=12 Bytes。至于 10 Bytes，则是将内存按照 2 Byts 进行对齐(GCC 的编译参数增加 `-fpack-struct=2`)。

多说几句：GCC 支持用 `__attribute__` 为变量、类型、函数、标签指定特殊属性。这些不是编程语言标准里的内容，而属于编译器对语言的扩展。比如 `aligned` 与 `packed` 为变量、类型、函数、标签指定特殊属性。这些不是编程语言标准里的内容，而属于编译器对语言的扩展。比如

`aligned` 属性最常用在变量声明上。它的作用是告诉 GCC，为变量分配内存时，要分配在对齐的内存地址上。比如：

```c
int x __attribute__ ((aligned (16))) = 0;
```

告诉编译器把变量x分配在16字节对齐的内存地址上，而非默认的4字节对齐。

`packed` 属性的主要目的是让编译器更紧凑地使用内存。当它用于变量时，告诉编译器该变量应该有尽可能小的对齐，也就是 1 字节对齐。当它用于结构体时，相当于给该结构体的每个成员加上了 `packed` 属性，这时该结构体将占用尽可能少的内存。比如：

```c
struct {
    char  a;
    int   b;
    float c;
} __attribute__((packed)) A;
```

更多内容参考[GCC中的aligned和packed属性](http://blog.shengbin.me/posts/gcc-attribute-aligned-and-packed)12。

## 参数传递

```c
void get(char *s){
    s = (char *)malloc(sizeof(16));
    strcpy(s, "Hello World.");
    return;
}

int main(void){
    char *s = NULL;
    get(s);
    printf("%s\n", s);
    return 0;
}
```

输出是？

哭丧脸，第一眼没看出来，纠结 `strcpy` 去了。后来确认： `strcpy` 除了复制字符串以外，还会复制字符串的结尾符 `'\0'`，即 `sizeof(str) = strlen(str) + 1` 个字符。而 `strncpy`，让它复制多少就复制多少，复制得比拥有的还多咋办？ `'\0'` 填充呗。__切记，一定要为 `'\0'` 留下空间__。

还得说一下 `NULL` 这个东西。

```c
// libio.h
#ifndef NULL
# if defined __GNUG__ && \
    (__GNUC__ > 2 || (__GNUC__ == 2 && __GNUC_MINOR__ >= 8))
#  define NULL (__null)
# else
#  if !defined(__cplusplus)
#   define NULL ((void*)0)
#  else
#   define NULL (0)
#  endif
# endif
#endif
```

这就是 C 语言中的 `NULL`。这[文章](http://stackoverflow.com/questions/34739117/why-is-the-memory-address-0x0-reserved-and-for-what)给了解释。简单地说，并非所有的语言实现用 `0` 代表 `NULL`，标准 C 语言(C99 或 C11)中标注 [NULL pointer](https://en.wikipedia.org/wiki/Null_pointer)的使用会导致未定义行为、段错误。

这是计算机语言中的参数传递问题。C 语言中的参数传递是按值传递，就是把参数复制一份传递再给调用的函数。所以，因为 `get` 参数接收的是字符指针的值(而不是地址)，所以函数返回后，实参的值并不会改变。要想改变，要么使用全局变量，要么传递指针的指针(即 `void get(char *s[]))，或者接收返回值。

汇编层面的解释：

```asm
# test.c
# #include <stdio.h>
#
# void test(int a){
#     return;
# }
#
# int main(void){
#     int a = 1;
#     test(a);
#     return 0;
# }
# gcc -S test.c -o test.s
    .file   "test.c"
    .text
    .globl  test
    .type   test, @function
test:
.LFB0:
    .cfi_startproc
    pushq   %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp
    .cfi_def_cfa_register 6
    movl    %edi, -4(%rbp)
    nop
    popq    %rbp
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
.LFE0:
    .size   test, .-test
    .globl  main
    .type   main, @function
main:
.LFB1:
    .cfi_startproc
    pushq   %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq    %rsp, %rbp
    .cfi_def_cfa_register 6
    subq    $16, %rsp
    movl    $1, -4(%rbp)
    movl    -4(%rbp), %eax
    movl    %eax, %edi
    call    test
    movl    $0, %eax
    leave
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
.LFE1:
    .size   main, .-main
    .ident  "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.4) 5.4.0 20160609"
    .section   .note.GNU-stack,"",@progbits
```

注： edi —— 存储传递给函数的第一个参数

## the difference between char s[] and char *s in C

> 满脑门子黑线，这我真的是第一次听说...

```c
char *get1(void){
    char *s = "Hello World";
    return s;
}

char *get2(void){
    char s[] = "Hello World";
    return s;
}
```

你说这俩有啥不同？

先说说[数据段(Data segment, 也叫 Text segment)](https://en.wikipedia.org/wiki/Data_segment)这个东西。
![Program memory](https://upload.wikimedia.org/wikipedia/commons/thumb/5/50/Program_memory_layout.pdf/page1-234px-Program_memory_layout.pdf.jpg)
一般来说，一个计算机程序的内存使用情况如图所示：

* 栈从上往下增长，存储临时变量之类
* 堆从下往上增长，动态分配的空间来源于此
* bss 段是为初始化的变量占用的空间，比如 `static int i` 这种
* data 段是初始化变量占用的空间，比如 `char string[] = "Hello World"`，其值 `"Hello World"` 即存储在这里
* code 段，用于存储代码。摊手，没法解释，就是代码

简洁地说，`get1()` 正确，`get2()` 的行为是不可预料的。`get1()` 中的变量 `s` 的值是 bss 段中的某个地址；而 `get2()` 中的变量 `s` 指向堆栈中的某个地址，`char s[] = "Hello World";` 的行为就是：在堆栈中分配一个数组，然后把 `"Hello World"` 复制到这数组中，数组的起始地址赋给 `s`。

参考[What is the difference between char s[] and char *s in C?](http://stackoverflow.com/questions/1704407/what-is-the-difference-between-char-s-and-char-s-in-c)

## 宏函数

> 恩，这是留下的作业。

C 语言中宏的一个有趣的用法：宏可以称为函数，感觉就像[内联函数(Inline function)](https://en.wikipedia.org/wiki/Inline_function)差不多。比如：

```c
// 一个宏, 将 4 个 unsigned char 型变量合成一个 unsigned 型变量
#define MIXTURE(ch, number, n)  ({\
    if(test_endian() == LITTLE){ \
        for(int i=1; i<= n; i++){ \
            number.ch[n-i] = ch[i-1]; \
        } \
    } \
    else { \
        for(int i=0; i< n; i++){ \
            number.ch[i] = ch[i]; \
        } \
    } \
})
```

有意思的是，宏函数也可以有返回值，先看这样一行代码：

```c
if((ch = getchar()) == 'c')
```

经常可以见到类似的。C 语言中，每个表达式都是有值的，如上，不是么？

...半月前在 Linux Kernel 中看到过这样的用法，但我现在给忘了...循环与跳转不是表达式，所以那示例我不会改写...留作思考题，跑

---
03.05 添加
注: 考虑以下的代码：

```c
// 1
#defina max(x, y) ((x)>(y)?(x):(y))
...
a = max(x, y) + z; // a = (x > y ? x : y) + z;
...

// 2
#define max(x, y) x>y?x:y
...
a = max(x, y) + z; // a = x > y ? x : y + z;
```

宏不使用小括号扩起来的话，可能会有隐患，比如上例——优先级的原因，外加 `x / y` 可能是表达式...

感谢 *Silver Bullet* 指出，ckj 的帮助~

[这个宏函数的实例](https://github.com/time-river/a-collection-of-junior/blob/master/homework/embedd-programming/1/tests/6-test.c)