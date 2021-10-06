---
title: Linux 如何执行程序 —— 内核态篇（废弃）
date: 2017-03-20 20:35:29
tags: ['linux', 'kernel']
---

**这篇已经被废弃，[x86 架构下 Linux 的系统调用与 vsyscall, vDSO](/2019/06/linux-syscall-and-vsyscall-vdso-in-x86/)与该内容部分相关，建议参考**

> 随着对linux kernel的深入了解，如今感觉这篇文章其实是有不少错误的。为了留念，还是留下吧。
> 2019.03.14笔。

Linux 系统中，双击桌面图标或者在 Shell 键入命令，都可以启动一个程序，好奇其中的具体过程，便找来相关的文章、源码看了看，简要地记录下体会。

加入程序是从 Shell 中启动的...

建议配合参考资料看：

[System calls in the Linux kernel. Part 4. How does the Linux kernel run a program](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/syscall-4.html)
[Linux process execution and the useless ELF header fields](http://shell-storm.org/blog/Linux-process-execution-and-the-useless-ELF-header-fields/)
[从源码中看可执行程序的加载流程](http://burningcodes.net/%E4%BB%8E%E6%BA%90%E7%A0%81%E4%B8%AD%E7%9C%8B%E5%8F%AF%E6%89%A7%E8%A1%8C%E7%A8%8B%E5%BA%8F%E7%9A%84%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B/)
[linux中ELF加载过程分析](http://www.voidcn.com/blog/fivedoumi/article/p-6316406.html)

## 从 Shell 到 execve

<small>=_= 讲真，乱糟糟的代码我看不懂，就用流程图简单概括一下好了。</small>

在 Bash 启动后、开始工作之前，会完成一系列检查，随后等待用户输入...

```md
当当当，用户键入一个命令...
reader_loop () -- 检查用户输入
-->
execute_command
--> execute_command_internal
----> execute_simple_command
------> execute_disk_command
--------> shell_execve
----------> execve -- 系统调用
```

系统调用长成这样：

```log
me@me-U24:~$ man execv
NAME
       execve - execute program

SYNOPSIS
       #include <unistd.h>

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);
...
```

* filename —— 准备执行的新程序的路径名
* argv / envp —— 同 C/C++，参数数组的指针地址，环境变量数组的指针地址

追踪一下 *ls* 的启动过程：

```bash
me@me-U24:~$ strace ls
execve("/bin/ls", ["ls"], [/* 75 vars */]) = 0
...
```

## execve 的执行过程

系统调用 execve 相关的执行过程是这样：

```md
execve (command, args, env);
--> do_execve(getname(filename), argv, envp);
----> do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
}
```

do\_execveat\_common 实参的解释：

* filename —— 一个与文件有关的 `struct filename` 结构
* argv / envp —— argv / envp 的再次封装，`struct user_arg_ptr / user_arg_ptr envp`

do\_execveat\_common 完成了如下的几个工作：

1. 检查 filename 指针是否为 NULL
2. 检查当前进程的 flags，确保 PF\_NPROC\_EXCEEDED 位未设置
  * NPROC: maximum number of processes
3. 对当前进程 unset PF\_NPROC\_EXCEEDED flag，确保 execve 执行成功
4. 解除文件描述符表的共享：复制一份给当前的进程，原有的地址赋给 displaced 变量
  * 目的是减小被执行的文件描述符泄漏的风险
5. 创建 `struct linux_binprm`(linux binary parameter, 存储待执行的二进制文件的参数) 实例 bprm
6. Prepare binprm credentials
  * bprm->cred 是一份当前进程 creds 的拷贝，包含了安全上下文、进程的 uid / guid、虚拟文件系统操作的 uid / guid 等
7. 检查传入的 flags(即 0)，在磁盘上搜索并打开可执行的文件，检查是否从 noexec 挂载点加载二进制文件
8. sched_exec(schedule executive) 函数用于进程调度
9. brpm->file && brpm->filename
  * brpm->file —— `struct file`，二进制文件信息
  * brpm->filename —— 二进制文件名
    * 文件名起始于 `/`，直接赋值(形参 fd 值为 AT_FDCWD)
    * 否则，取决于文件名是否存在，赋值 `/dev/fd/%d` or `/dev/fd/%d/%s`
10. bprm->interp = bprm->filename
11. bprm->mm —— 初始化内存描述符(即 bprm\_mm\_init 函数初始化 `struct mm_struct` 实例，使用了临时堆栈 vm\_area\_struct 填充它)
  * bprm->vma = vma;
    * vma->vm\_mm = bprm->mm;
    * vma->vm\_end = STACK\_TOP\_MAX; —— the first byte after our end address
    * vma->vm\_start = vma->vm\_end - PAGE\_SIZE; —— start address
  * bprm->p = vma->vm\_end - sizeof(void \*); —— current top of mem
12. bprm->argc && bprm->envc —— calculated the number of the command line arguments and environment variables(字符串数量)
13. Prepare binprm —— 读取 inode 来设置 brpm->cred 中的 uid / gid，读取文件的前 128 字节并填充至 bprm->buf
  * 需要可执行文件中的前 128 字节来判断可执行文件的类型
14. 从用户空间拷贝文件名(filename)、命令参数(argv)、环境变量(envp)只分配的内存中(brpm->mm)
  * bprm->exec = bprm->p; —— 堆栈的顶端包含程序的文件名，将此值存储至 exec 中
15. linux\_bprm 结构实例准备完成，执行 `exec_binprm`
  * 核心函数 `search_binary_handler(bprm);` —— 这个函数遍历所有不同的可执行的二进制文件格式，之后调用 `fmt->load_binary(bprm)`
      * binfmt\_script —— support for interpreted scripts that are starts from the [#!](https://en.wikipedia.org/wiki/Shebang_%28Unix%29) line
      * [binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc) —— support differnt binary formats, according to runtime configuration of the Linux kernel
      * infmt\_elf —— support [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) format
      * binfmt\_aout —— support [a.out](https://en.wikipedia.org/wiki/A.out) format
      * binfmt\_flat —— support for [flat](https://en.wikipedia.org/wiki/Binary_file#Structure) formats
      * binfmt\_elf\_fdpic —— Support for [elf FDPIC](http://elinux.org/UClinux_Shared_Library#FDPIC_ELF) binaries
      * binfmt\_em86 —— support for Intel elf binaries running on Alpha machines

## elf 格式文件的执行

假如可执行文件为 elf 类型，正确的 `load_binary` 函数为 `load_elf_binary`，这个函数的行为如下：

1. 获取 bprm->buf 这 128 字节的内容： loc->elf_ex = *((struct elfhdr *)bprm->buf);
2. 使用 loc 进行一系列检查： 标识符、文件类型、架构
3. 尝试加载程序头(program header table)：描述段信息
4. 读取 program program interpreter(.interp 段所描述)，加载共享链接库至内存
5. 为进程分配用户态堆栈，并塞入参数和环境变量
  * `setup_arg_pages` 函数中 `static int shift_arg_pages(struct vm_area_struct *vma, unsigned long shift)` 注释：
> 在 bprm\_mm\_init() 中，在 *STACK_TOP_MAX* 上创建了临时堆栈。一旦 binfmt 代码决定新的堆栈在哪里，便把这临时堆栈转移到那里。过程如下：
>   1. 使用参数 shift 计算新的 vma 起始点
>   2. 调整 vma 的空间，使之包含新、旧 vma 的范围。这样可以确保传给之后执行函数的参数是一致的
>   3. 迁移 vma 页表到新的空间
>   4. 释放进程用户空间的页表(pgb, page tables)
>   5. 调整 vma 大小，使之只包含新的申请的空间
6. 设置堆栈，把 elf 二进制文件映射到正确的位置，设置 bss / brk 段
7. ...
8. start\_thread(regs, elf\_entry, bprm->p);
  * regs -- 新进程的寄存器
  * elf\_entry -- 进程入口
  * bprm->p -- 栈顶

相关的代码如下：

```c
void
start_thread(struct pt_regs *regs, unsigned long new_ip, unsigned long new_sp)
{
        start_thread_common(regs, new_ip, new_sp,
                            __USER_CS, __USER_DS, 0);
}

static void
start_thread_common(struct pt_regs *regs, unsigned long new_ip,
                    unsigned long new_sp,
                    unsigned int _cs, unsigned int _ss, unsigned int _ds)
{
        loadsegment(fs, 0);
        loadsegment(es, _ds);
        loadsegment(ds, _ds);
        load_gs_index(0);
        regs->ip                = new_ip;
        regs->sp                = new_sp;
        regs->cs                = _cs;
        regs->ss                = _ss;
        regs->flags             = X86_EFLAGS_IF;
        force_iret();
}
```

对 `force_iret()` 的解释：
强制系统调用通过 IRET 指令返回，这是因为 IRET 会改变 SS / CS / EFLAGS 寄存器的值，这点是其他指令所办不到的。

IA-32 指令手册关于这点的描述如下：
> the IRET instruction pops the return instruction pointer, return code segment selector, and EFLAGS image from the stack to the EIP, CS, and EFLAGS registers, respectively, and then resumes execution of the interrupted program or procedure. If the return is to another privilege level, the IRET instruction also pops the stack pointer and SS from the stack, before resuming program execution
> 当使用 IRET 指令返回到相同保护级别的任务时，IRET 会从堆栈弹出代码段选择子及指令指针分别到 CS 与 IP 寄存器，并弹出标志寄存器内容到 EFLAGS 寄存器。
> 当使用 IRET 指令返回到一个不同的保护级别时，IRET 不仅会从堆栈弹出以上内容，还会弹出堆栈段选择子及堆栈指针分别到 SS 与 SP 寄存器。

设置好必要的寄存器后， `force_iret()` 通过 `iret` 指令强制中断返回。此时，便准备了新的一个用户空间进程。之后，程序结束后从 `exec_binprm(bprm)` 返回，又回到了 `do_execveat_common`，然后做了些收尾工作：释放分配的空间等，最后返回。

从系统调用 `execve` 返回后，所执行的程序便开始运行。可以做到这样，是因为所有与之有关的上下文信息已经配置好。`execve` 系统调用不将控制权返回给进程，而是给调用者进程的代码，这是因为代码段、数据段等段寄存器已经被所执行的程序所覆盖。退出程序将通过系统调用 `exit` 实现。

## 总结

Linux 系统启动新进程是通过系统调用 `execve` 实现的，核心函数是 `do_execveat_common`，所做的工作无非是创建一个帧，顶部塞文件名、环境变量、参数，初始化 bss、break section、加载 elf 文件至内存等等，最后在启动新进程之前，设置各种段寄存器、标志寄存器、堆栈寄存器以及指令计数器(PC，即 IP)，通过 iret 指令强制返回系统调用后，一个用户空间的进程便启动了。

## 遗言

是根据[System calls in the Linux kernel. Part 4. How does the Linux kernel run a program](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/syscall-4.html)来看源码的，又见识了不少奇奇怪怪的 C 语法，看得头炸，摊手，磕磕绊绊，尽管看懂了，还是遗留下了几个问题，这是文章中没有提到的...

iret 这个指令从堆栈中弹出相关值填到对应的寄存器中，因为是从 Ring 0 模式返回 Ring 3 模式，所以与此次操作有关的寄存器有 SS/ CS /EFLAGS / SS / SP。

如果了解过程序执行的函数栈(没有的话，推荐这篇[C语言函数调用栈(一)](http://www.cnblogs.com/clover-toeic/p/3755401.html))的话，应该知道这个：跳转分为近跳与远跳，根据跳转的种类将 EIP /CS 的值压入栈中。曾经研究过 PintOS 进程切换的实现，栈帧之间的 SP 切换罢了。

问题一：**`force_iret`（ 通过 `iret` 指令返回用户态继续执行，不应该是等待系统调用 `execve` 执行结束后再返回吧？**

冯诺依曼体系的计算机，指令是连续存储的，没有跳转的情况下，一条指令接着另一条指令顺序执行(PC+=1，PC 为程序计数器，存放下一条指令的地址)，遇到分支与跳转的情况，PC 则会重新赋值。在 x86 架构中则为 CS:IP，显然可以通过改变它来进行跳转。`iret` 改变了 CS:IP，不应该是执行完系统调用 `execve` 后再执行用户态程序。那么加入立即切换到用户态执行程序，`execve` 余下的代码怎么继续执行呢？执行完用户态的程序后吗？那 `iret` 指令执行前的 CS:IP 又保存在哪里？用户态的程序执行系统调用 `exit` 退出进程，是否于此有关呢？

假如已经知道 main 函数不是 C/C++ 程序的 entry point 话，反编译一下入口函数：

```bash
me@me-U24:/usr/lib/x86_64-linux-gnu$ objdump -d crt1.o

crt1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <_start>:
   0: 31 ed                 xor    ebp,ebp
   2: 49 89 d1              mov    r9,rdx
   5: 5e                    pop    rsi
   6: 48 89 e2              mov    rdx,rsp
   9: 48 83 e4 f0           and    rsp,0xfffffffffffffff0
   d: 50                    push   rax
   e: 54                    push   rsp
   f: 49 c7 c0 00 00 00 00  mov    r8,0x0
  16: 48 c7 c1 00 00 00 00  mov    rcx,0x0
  1d: 48 c7 c7 00 00 00 00  mov    rdi,0x0
  24: e8 00 00 00 00        call   29 <_start+0x29>
  29: f4                    hl
```

很容易注意到 `mov r9, rdx`，引出第二个问题：**寄存器 rdx 的值在哪里被给予的呢？**

在[x86 Instruction Set Reference](http://x86.renejeschke.de/html/file_module_x86_id_145.html)中找到的相关伪码如下(我猜是它，应该没错)：

```c
if(PE == 0) {
  // Real-Address-Mode
  ...
}
else {
  // Protected Mode
  ...
  if(OperandSize == 32) {
    if(!IsWithinStackLimits(TopStackBytes(12)) Exception(SS); //top 12 bytes of stack not within stack limits
    TemporaryEIP = Pop();
    TemporaryCS = Pop();
    TemporaryEFLAGS = Pop();
  }
  ...
  if(TemporaryEFLAGS.VM == 1 && CPL == 0) {
    //Interrupted procedure was in virtual-8086 mode: PE == 1, VM == 1 in flags image
    if(!IsWithinStackLimits(TopStackBytes(24)) Exception(SS(0)); //top 24 bytes of stack not within stack limits
    if(!IsWithinCodeSegmentLimits(InstructionPointer)) Exception(GP(0));
    CS = TemporaryCS;
    EIP = TemporaryEIP;
    EFLAGS = TemporaryEFLAGS;
    TemporaryESP = Pop();
    TemporarySS = Pop();
    ES = Pop(); //pop 2 words; throw away high-order word
    DS = Pop(); //pop 2 words; throw away high-order word
    FS = Pop(); //pop 2 words; throw away high-order word
    GS = Pop(); //pop 2 words; throw away high-order word
    SS:ESP = TemporarySS:TemporaryESP;
    CPL = 3;
    ResumeExecution() //Resume execution in Virtual-8086 mode
  }
  ...
}
```

按照这文档，iret 指令执行之前，堆栈中对应的值应该如下：

```md
+--------+
|   ...  |
+--------+
|   GS   |
+--------+
|   FS   |
+--------+
|   DS   |
+--------+
|   ES   |
+--------+
| EFLAGS |
+--------+
|   CS   |
+--------+
|   EIP  |
+--------+ <-- RSP
```

没有与通用寄存器 rdx 相关的任何内容，找遍了与变量 regs 有关的任何代码，无果...

啃不动内核源码，唉，这俩问题只能当作遗言了。
-- 2017 / 03 / 22 / 19:24 E-309 西 笔

## 后记

问题一的答案  
>**`force_iret`（ 通过 `iret` 指令返回用户态继续执行，不应该是等待系统调用 `execve` 执行结束后再返回吧？**

《程序员的自我修养》这本书的 174 页有这么一段话：
> 当 load\_elf\_binary() 执行完毕后，返回 do\_execve() 在返回至 sys\_execve() 时，上面的第 5 步(将__系统调用__的返回地址修改成 ELF 可执行文件的入口点...)中已经把操作系统的返回地址改成了被装载的 ELF 程序的入口地址了，所以当 sys\_execve() 系统调用从内核态返回到用户态时，EIP 寄存器直接跳转到了 ELF 程序的入口地址，于是新的程序开始执行，ELF 可执行文件装载完成。

所以，`start_thread_common(regs, new_ip, new_sp,  __USER_CS, __USER_DS, 0);` 中的参数 `regs` 比较有意思。我猜测，它指向的是：调用系统中断(即`execve`)时，保存的返回地址。

`regs` 的值是这么确定的：

```c
struct pt_regs *regs = current_pt_regs();
```

这是一个与 CPU 架构有关的函数，没找到 x86 平台的，瞄了几眼其他平台的，都是根据偏移量来寻找某个值。

这样的话，答案便确定了：是在 `execv` 执行完后开始执行用户空间程序的，通过修改 `execv` 的返回地址来实现。

显然，返回地址被覆盖了，怎可能有返回值之类的呢？

-- 2017 / 05 / 21 / 16:52 E-309 西 补充
