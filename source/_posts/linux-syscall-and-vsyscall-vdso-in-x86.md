---
title: x86架构下Linux的系统调用与vsyscall, vDSO
date: 2019-06-18 11:26:33
tags: ['linux', 'syscall', 'vdso']
---

## 系统调用

现代操作系统的进程空间分为用户空间（user space）与内核空间（kernel space）。通常程序运行在用户空间中，当涉及一些敏感指令执行的时候，比如与硬件交互的操作，需要切换到内核空间，相关指令执行完毕后再返回用户空间继续执行。系统调用（syscall）在此过程中作为沟通用户空间与内核空间的桥梁存在。[\[1\]][1]

具体到x86架构，80286引入保护模式之后，有了特权等级（privilege level, or rings）的概念，它分为0-3 4个等级。[\[2\]][2]Linux下用户空间代码运行在ring 3，内核空间代码运行在ring 0，ring 3与ring 0的互相切换便是通过系统调用进行的。[\[3\]][3]

> Note：
>
> 1. 更准确地说，自x86支持virtualization（Intel VT-x，AMD-V）后rings有5个等级，第五个是ring -1，用于虚拟化模式下。
> 2. 系统调用并非唯一一种特权等级切换的方式，比如Linux中的vDSO也可以使ring 0的代码执行在ring 3下。

涉及到的CPU指令，x86下面有三对：

1. [`int 0x80`][5] / [`iret`][6]
2. 32 bit下引入的fast system call：[`sysenter`][7] / [`sysexit`][8]（自Intel Pentium II始），[`syscall`][9] / [`sysret`][10]（自AMD K6始）[\[4\]][4]
3. 64 bit下的[`syscall`][9] / [`sysret`][10]

Linux提供的系统调用函数可以通过[`$ man 2 syscalls`][11]来查看。自用户空间的程序视角来看，系统调用的执行分为用户态与内核态两部分代码。对于系统调用在内核态执行过程的技术细节，LWN的两篇文章[Anatomy of a system call, part 1][13]与[Anatomy of a system call, part 2][14]，还有[Linux系统调用过程分析][15]与[Linux Inside: System calls in the Linux kernel. Part 1.][41]、[[译] Linux 系统调用权威指南][60]，对此有很好的描述。需要明确的是，多数情况下在用户态使用的系统调用是glibc的封装，但它也提供了[`syscall()`][12]这种很接近手写汇编语言的函数。至于glibc背后隐藏的细节，需要了解一下vDSO。

总结一下系统调用在用户空间与内核空间的执行过程，即：

1. glibc对绝大多数系统调用进行了封装，未被封装的系统调用需要使用`syscall()`使用
2. x86下通过执行三类指令之一进行跳转；进入入口函数执行之前，系统调用号以及相关参数会依照约定的规则入栈或存在寄存器中
   - `int 0x80`走中断流程，跳转地址的初始化在[中断初始化过程][57]中
     - IA-32: [`SYSG(IA32_SYSCALL_VECTOR,   entry_INT80_32)`][58]
     - AMD64: [`SYSG(IA32_SYSCALL_VECTOR,   entry_INT80_compat)`][59]
   - fast system call指令的跳转地址存储在[MSR（Model-specific register）][42]中
     - IA-32: [`wrmsr(MSR_IA32_SYSENTER_EIP, (unsigned long)entry_SYSENTER_32, 0);`][43]
     - AMD64: [`wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);`][44]
3. [trap][17]内核空间；因为函数调用的时候存在栈结构的变化，因此在调用函数之前需要保存现场、设置
4. 使用`call`指令跳转进入入口函数，根据系统调用号在syscall table中查找对应的系统调用并执行
5. 系统调用执行结束后返回，恢复所保存的现场，使用配对的返回指令返回用户空间

> Note:
>
> 一个普遍的观点是系统调用要比函数调用慢，而采用中断方式的系统调用又比另外两种指令慢[\[33\]][33]。背后的原因是？
> 直觉告诉我一种可能是系统调用替换了CS、IP寄存器，导致了CPU的流水线中断，从而带来了额外的开销。但实际上，函数调用`call`指令也分为near call、far call（扫了一眼[指令描述][19]——看不懂），总之做的事情也会导致流水线中断。看了一下[这里（系统调用真正的效率瓶颈在哪里？）][18]的回答，的确在执行系统调用的代码中，CPU做了很多的事情（保存、调整了许多寄存器），软件做了许多事情（条件判断），还有一些同步操作（比如开中断内联汇编`"sti": : :"memory"`会阻止编译器层次的乱序执行[\[21\]][21]）等，这些事情是函数调用没有的。`int 0x80`对比fast system call, 它做了更多的工作，所以速度最慢也不奇怪了。

## vsyscall与vDSO

### Overview

对于vsyscall（virtual syscall）与vDSO（virtual dynamic shared object），[Linux的man page][20]、[slides: The vDSO on arm64][22]、[LWN.net: On vsyscalls and the vDSO][23]、[Linux Inside: System calls in the Linux kernel. Part 3.][24]提供了不少有用的信息。总结一下：

- vsyscall（virtual system call）提供了一种在用户空间下快速执行系统调用的方法
  - 加速原理是对特定的系统调用使用函数调用代替
  - map的起始地址固定（0xffffffffff600000)，有潜在的安全风险
- 为了改善vsyscall的局限性，设计了vDSO
  - 可以利用ASLR（address space layout randomization）增强安全性
  - 用户无需在用户空间的代码中考虑CPU的差异性
    - 比如用户空间代码无需考虑IA-32下的两类fast system call指令
  - 出于兼容性的考虑保留了vsyscall
- vDSO是一个动态链接库，它
  - 由内核提供
  - map至每一个进程

终端上执行一系列操作后，64 bit的Linux v4.12会输出下面的内容：

```bash
$ sudo echo 0 > /proc/sys/kernel/randomize_va_space # disable ASLR
$ ldd /usr/bin/ls # print shared object dependencies
    linux-vdso.so.1 (0x00007ffff7ffa000)
    libc.so.6 => /lib64/libc.so.6 (0x00007ffff73cd000)
    /lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd7000)
...

$ cat /proc/self/maps # print currently mapped memory regions and their access permissions
...
7ffff7ff7000-7ffff7ffa000 r--p 00000000 00:00 0                          [vvar]
7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

$ gdb -q ls
...
>>> b _start
>>> info program
    Using the running image of child process 39525.
...
>>> shell cat /proc/39525/maps | grep vdso
7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0                          [vdso]
>>> dump memory /tmp/vdso.so 0x7ffff7ffa000 0x7ffff7ffc000
...
$ objdump -T /tmp/vdso.so # print vdso symbols

/tmp/vdso.so:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
00000000000006f0  w   DF .text  0000000000000305  LINUX_2.6   clock_gettime
0000000000000a00 g    DF .text  00000000000001c2  LINUX_2.6   __vdso_gettimeofday
0000000000000a00  w   DF .text  00000000000001c2  LINUX_2.6   gettimeofday
0000000000000bd0 g    DF .text  0000000000000015  LINUX_2.6   __vdso_time
0000000000000bd0  w   DF .text  0000000000000015  LINUX_2.6   time
00000000000006f0 g    DF .text  0000000000000305  LINUX_2.6   __vdso_clock_gettime
0000000000000000 g    DO *ABS*  0000000000000000  LINUX_2.6   LINUX_2.6
0000000000000bf0 g    DF .text  000000000000002a  LINUX_2.6   __vdso_getcpu
0000000000000bf0  w   DF .text  000000000000002a  LINUX_2.6   getcpu
```

可以看到：

1. `linux-vdso.so.1`的确不存在对应的文件
2. vsyscall映射一页：[vsyscall]
3. vDSO会映射两块内存区域：代码段[vdso]，只读变量区[vvar]
4. vDSO在AMD64下暴露出4种系统调用

Note：无前缀`__vdso_`的函数名是相应函数的[弱符号别名][25]（GCC属性：`__attribute__ ((weak, alias ("<alias name>")));`）。

### 使用

#### vsyscall的使用

glibc在源码中定义了三个地址，他们由`__vsyscall_page + <vsyscall address offset>`得到，可赋值给函数指针后使用。

```bash
$ grep -rn VSYSCALL_ADDR_ # in glibc v2.21
sysdeps/unix/sysv/linux/x86_64/gettimeofday.c:22:# define VSYSCALL_ADDR_vgettimeofday   0xffffffffff600000ul
sysdeps/unix/sysv/linux/x86_64/init-first.c:48:#define VSYSCALL_ADDR_vgetcpu    0xffffffffff600800
sysdeps/unix/sysv/linux/x86_64/time.c:20:#define VSYSCALL_ADDR_vtime    0xffffffffff600400
```

> Note：glibc在v2.22移除了对vsyscall的支持，commit [7cbeabac0fb28e24c99aaa5085e613ea543a2346][34]。

一个使用vsyscall `time()` function的例子：

```c
#include <time.h>
#include <stdio.h>

typedef time_t  (*time_func)(time_t *);

int main(int argc, char *argv[]) {
    time_t tloc;
    int retval = 0;

    time_func func = (time_func)0xffffffffff600000;

    retval = func(&tloc);
    if (retval < 0) {
        perror("time_func");
        return -1;
    }
    printf("%ld\n", tloc);

    return 0;
}
```

#### vDSO的使用

一些背景知识：

- 辅助向量（auxiliary vector）
- 动态链接库的加载
- ELF（Executable and Linkable Format）格式

[LWN.net: getauxval() and the auxiliary vector][26]与[About ELF Auxiliary Vectors][27]给出了许多有关辅助向量的信息。大意是：辅助向量是内核向用户空间传递信息的一种机制，大多数类UNIX系统都提供了这项特性。它是可执行文件载入进程时构建的键值对，位于传递了一些运行时程序自身的信息，比如ELF文件相关、权限等，当然包括了vDSO的起始地址，这些信息的主要使用者是linker。GLIBC提供了`getauxval()`函数查找辅助向量的值。

在Linux下的终端中打印辅助向量：

```bash
$ od -t d8 /proc/self/auxv
0000000                   33      140737354113024
...
$ printf %#x 140737354113024 # 十进制转换为十六进制
0x7ffff7ffa000

$ env LD_SHOW_AUXV=1 sleep 1
AT_SYSINFO_EHDR: 0x7ffff7ffa000
...
```

除了隐式运行时由linker自动加载动态链接库之外，glibc提供了`dl*`系列API手动解析。只是`dlopen()`的参数之一要求文件地址，而vDSO是由内核map至内存，并不存在对应的文件地址，所以也就无法使用上述API了。在glibc下，可以使用参数`AT_SYSINFO_EHDR`通过`getauxval()`获取vDSO map后的内存起始地址，再通过解析ELF 符号表得到对应的函数指针。还有一个辅助向量`AT_SYSINFO`，它是系统调用函数在vDSO的入口地址，用于解决CPU的差异导致的系统调用指令不同的问题。

ELF很复杂，Wikipedia上有它的[介绍][28]，[这里][29]是AMD64的specification。

Linux提供了关于使用vDSO的demo，[vdso_test.c][30]依赖标准库，不依赖标准库的版本是[vdso_standalone_test_x86.c][31]。

### 在内核中的实现

Linux version 5.0，AMD64 architecture。

#### vsyscall的实现

与vsyscall有关的源码在[arch/x86/entry/vsyscall/][32]目录下。

固定映射区间的[代码][36]：

```c
/**
 *  arch/x86/entry/vsyscall/vsyscall_64.c
 *    void __init map_vsyscall(void)
 *
 *  note:
 *    VSYSCALL_PAGE is constant: 0xffffffffff600000
 *    void __set_fixmap(enum fixed_addresses idx, phys_addr_t phys, pgprot_t flags)
 */
__set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
     PAGE_KERNEL_VVAR);
```

v4.16之前，flags标志位可以是`PAGE_KERNEL_VSYSCALL`（configuration: `LEGACY_VSYSCALL_NATIVE`)或`PAGE_KERNEL_VVAR`（configuration `LEGACY_VSYSCALL_EMULATE`），他们的区别只是RX（可读可执行）与RO（只读）的区别，[commit: 076ca272a14cea558b1092ec85cea08510283f2a][55]后只有EMULATE了。对用户空间程序的影响是使用`emulate_vsyscall()`函数还是vsyscall_emu_64.S中的代码进行vsyscall。

[vsyscall_emu_64.S][37]定义了vsyscall page大小与vsyscall允许的系统调用入口地址（三个：`gettimeofday() / time() / getcpu()`。对其的字节数是1024（即0x400），所以那些系统调用之间会有0x400的固定偏移。

使用`time()`这类系统调用获取的信息是动态变化的，[vsyscall_gtod.c][38]实现了更新这些信息的函数。大致工作过程是一个tick到来后，timekeeper模块会维护一些信息，更新的过程中便会[调用他们][39]。不过，这些信息在v5.0下不再提供给vsyscall，而是提供给vDSO使用（`vsyscall_gtod_data`是被维护的信息，我在vsyscall相关的源码中并没有找到使用该变量的地方）。

```c
/**
 *  kernel/time/timekeeping.c
 */
static void timekeeping_update(struct timekeeper *tk, unsigned int action)
{
    ...
    update_vsyscall(tk);
    update_pvclock_gtod(tk, action & TK_CLOCK_WAS_SET);
    ...
}
```

> Note: 有关Linux下的时钟介绍：[Linux 下的时钟][40]。

实际上，v5.0在AMD64架构上对vsyscall的实现只有emulat。当产生缺页中段的时候，会[检测][54]地址是否位于vsyscall中，被模拟的系统调用选择，是根据[内存地址][53]决定的：

```c
/**
 *  arch/x86/mm/fault.c
 */
/* Handle faults in the user portion of the address space */
static inline
void do_user_addr_fault(struct pt_regs *regs,
            unsigned long hw_error_code,
            unsigned long address)
{
    ...
    if ((hw_error_code & X86_PF_INSTR) && is_vsyscall_vaddr(address)) {
        if (emulate_vsyscall(regs, address))
            return;
    }
    ...
}

bool emulate_vsyscall(struct pt_regs *regs, unsigned long address)
{
    ...
    vsyscall_nr = addr_to_vsyscall_nr(address);
    ...
}
```

#### vDSO的实现

与vDSO有关的源码在[arch/x86/entry/vdso][45]目录下，实际上在编译的过程中会生成若干文件，最终被编译进内核的是vdso\-image\-\*.c、vma.c、vdso32\-setup.c。vdso\-image\-\*.c是使用程序vdso2c（由vdso2c.c编译）生成的，输入是vdso\*.so\*，其中有`struct vdso_image`，它描述了vDSO的镜像内容。vdso.so的link script即vdso\-layout.lds.S，vdso.lds.S，vdsox32.lds.S，vdso32.lds.S，第一个描述了vdso的layout，后三个是[LD Version Scripts][46]。[这里][47]给出了一些关于Linker Script的介绍。

在exec载入ELF的过程中，会调用[`arch_setup_additional_pages()`][48]，在[`map_vdso()`][49]中完成对[vdso]、[vvar]的map（因此vdso会在/proc/self/maps中出现两处map）。自v4.5起Linux采用了缺页中断的方式加载vdso的内容，[commit: f872f5400cc01373d8e29d9c7a5296ccfaf4ccf3][50]。map vdso后，会设置[ELF的辅助向量][51]。

vDSO中加速系统调用的原理与vsyscall一样，复用了vsyscall中获取、动态更新信息的代码（关键变量[`vsyscall_gtod_data`][52]）。

借用一张ARM64下的vDSO原理图：

![anatony of the vDSO on arm64](/images/19/06/anatony-of-the-vDSO-on-arm64.png)

[LWN.net: Implementing virtual system calls][56]描述了如何在vDSO中实现virtual system call。

## 系统调用追踪

这张图给出了一些系统调用追踪工具（或者说在system libraries、system call interfaces层面的dynamic trace tools）：

![Linux Observability tools](/images/19/06/linux_observability_tools.png)

在dynamic trace方面了解很浅，与系统调用追踪相关的知识是从这两篇文章看到的：

- [源码分析：动态分析 Linux 内核函数调用关系][63]
- [ftrace: trace your kernel functions!][64]

总结一下：

- 用户空间追踪工具：
  - [ltrace][65] —— A library call tracer
  - [strace][66] —— trace system calls and signals
  - [callgrind][67] -- Callgrind is a profiling tool that records the call history among functions in a program's run as a call-graph
- 内核空间追踪工具：
  - [trace-cmd][68] -- interacts with Ftrace Linux kernel internal tracer
  - [perf][69] -- Performance analysis tools for Linux

## Reference

1. [Wikipedia](https://www.wikipedia.org/)
2. [x86 and amd64 instruction reference](https://www.felixcloutier.com/x86/)
3. [The Linux man-pages project](https://www.kernel.org/doc/man-pages/)
4. [LWN.net](https://lwn.net/)
5. [Linux系统调用过程分析][15]
6. [64位Linux下的系统调用][16]
7. [知乎：系统调用真正的效率瓶颈在哪里？][18]
8. [Stack Overflow: Working of \_\_asm\_\_ \_\_volatile\_\_ ("" : : : "memory")][21]
9. [slides: The vDSO on arm64](https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/3711/original/LPC_vDSO.pdf)
10. [Linux Inside](https://legacy.gitbook.com/book/0xax/linux-insides)
11. [GNU compilers documents: 6.30 Declaring Attributes of Functions][25]
12. [About ELF Auxiliary Vectors][27]
13. [LKML.org: Intel P6 vs P7 system call performance][33]
14. [GNU Gnulib: 16.3 LD Version Scripts][46]
15. [Category: Linker Script][47]
16. [[译] Linux 系统调用权威指南][60]
17. [[译] ltrace 是如何工作的][61]
18. [[译] strace 是如何工作的][62]
19. [泰晓科技：源码分析：动态分析 Linux 内核函数调用关系][63]
20. [ftrace: trace your kernel functions!][64]
21. [Valgrind Documentation: 6. Callgrind: a call-graph generating cache and branch prediction profiler][67]
22. [brendangregg.com: Linux Performance](http://www.brendangregg.com/linuxperf.html)

## 结语

依稀记得，大三下学期，那时在上嵌入式程序设计这门课（其实就是Linux环境编程），调研了一下程序是如何运行的这个问题。那时写的[Linux如何执行程序——内核态篇（废弃）](/2017/03/how-does-the-Linux-kernel-run-a-program/)与[Linux如何执行程序——用户态篇（废弃）](/2017/03/how-does-the-linux-user-run-a-program/)两篇，不仅烂尾了，而且内容是有不少问题的。

这半年多来，一直在阅读Linux源码与相关的文档，也做了不少与系统调用相关的工作。便用了半周，重新回顾了一下之前的工作、补充下未曾探究过的内容，围绕系统调用这一主题，写下了这篇文章。虽不及过去两篇那般涉及的范围广，但更加详尽、准确，也忽略了绝大部分细节。可惜的是，尽管17年年中的时候与Linux tracing技术有了第一次接触，现在仍然只能复制一下命令来使用。

在此过程中，意外地发现自己可以独立地分析部分内核源码了，许多之前不曾看懂的内容豁然开朗，也算这半年来的一个收获~

[1]: https://en.wikipedia.org/wiki/System_call
[2]: https://en.wikipedia.org/wiki/Protected_mode
[3]: https://en.wikipedia.org/wiki/Protection_ring
[4]: https://en.wikipedia.org/wiki/X86_instruction_listings
[5]: https://www.felixcloutier.com/x86/intn:into:int3:int1
[6]: https://www.felixcloutier.com/x86/iret:iretd
[7]: https://www.felixcloutier.com/x86/sysenter
[8]: https://www.felixcloutier.com/x86/sysexit
[9]: https://www.felixcloutier.com/x86/syscall
[10]: https://www.felixcloutier.com/x86/sysret
[11]: http://man7.org/linux/man-pages/man2/syscalls.2.html
[12]: http://man7.org/linux/man-pages/man2/syscall.2.html
[13]: https://lwn.net/Articles/604287/
[14]: https://lwn.net/Articles/604515/
[15]: https://www.binss.me/blog/the-analysis-of-linux-system-call/
[16]: http://www.lenky.info/archives/2013/02/2199
[17]: https://en.wikipedia.org/wiki/Trap_(computing)
[18]: https://www.zhihu.com/question/32043825
[19]: https://www.felixcloutier.com/x86/call
[20]: http://man7.org/linux/man-pages/man7/vdso.7.html
[21]: https://stackoverflow.com/questions/14950614/working-of-asm-volatile-memory
[22]: /pdf/LPC_vDSO.pdf
[23]: https://lwn.net/Articles/446528/
[24]: https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-3.html
[25]: https://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Function-Attributes.html
[26]: https://lwn.net/Articles/519085/
[27]: http://articles.manugarg.com/aboutelfauxiliaryvectors.html
[28]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[29]: http://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf
[30]: https://elixir.bootlin.com/linux/v5.0/source/tools/testing/selftests/vDSO/vdso_test.c
[31]: https://elixir.bootlin.com/linux/v5.0/source/tools/testing/selftests/vDSO/vdso_standalone_test_x86.c
[32]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vsyscall
[33]: https://lkml.org/lkml/2002/12/9/13
[34]: https://sourceware.org/git/?p=glibc.git;a=commit;h=7cbeabac0fb28e24c99aaa5085e613ea543a2346
[35]: http://www.lenky.info/archives/2013/02/2199
[36]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vsyscall/vsyscall_64.c#L361
[37]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vsyscall/vsyscall_emu_64.S
[38]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vsyscall/vsyscall_gtod.c
[39]: https://elixir.bootlin.com/linux/v5.0/source/kernel/time/timekeeping.c#L665
[40]: https://vvl.me/2019/04/linux-clock/
[41]: https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html
[42]: https://en.wikipedia.org/wiki/Model-specific_register
[43]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/kernel/cpu/common.c#L1439
[44]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/kernel/cpu/common.c#L1538
[45]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vdso
[46]: https://www.gnu.org/software/gnulib/manual/html_node/LD-Version-Scripts.html
[47]: https://wen00072.github.io/blog/categories/linker-script/
[48]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vdso/vma.c#L289
[49]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vdso/vma.c#L146
[50]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f872f5400cc01373d8e29d9c7a5296ccfaf4ccf3
[51]: https://elixir.bootlin.com/linux/v5.0/source/fs/binfmt_elf.c#L245
[52]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vdso/vclock_gettime.c#L25
[53]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/entry/vsyscall/vsyscall_64.c#L138
[54]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/mm/fault.c#L1398
[55]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/arch/x86/entry/vsyscall/vsyscall_64.c?id=076ca272a14cea558b1092ec85cea08510283f2a
[56]: https://lwn.net/Articles/615809/
[57]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/kernel/traps.c#L952
[58]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/kernel/idt.c#L105
[59]: https://elixir.bootlin.com/linux/v5.0/source/arch/x86/kernel/idt.c#L103
[60]: https://arthurchiao.github.io/blog/system-call-definitive-guide-zh/
[61]: https://arthurchiao.github.io/blog/how-does-ltrace-work-zh/
[62]: https://arthurchiao.github.io/blog/how-does-strace-work-zh/
[63]: http://tinylab.org/source-code-analysis-dynamic-analysis-of-linux-kernel-function-calls/
[64]: https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/
[65]: http://man7.org/linux/man-pages/man1/ltrace.1.html
[66]: http://man7.org/linux/man-pages/man1/strace.1.html
[67]: http://valgrind.org/docs/manual/cl-manual.html
[68]: http://man7.org/linux/man-pages/man1/trace-cmd.1.html
[69]: http://man7.org/linux/man-pages/man1/perf.1.html
