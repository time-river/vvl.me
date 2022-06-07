---
title: criu的原理、局限与性能优化方向
date: 2022-06-07 23:51:19
tags: ["criu", "linux"]
---

> 读研的时候做的是容器，便接触过criu，同门师兄还写过几篇源码分析的文章，工作后参与的项目也与它有关。迄今为止自己对criu进行二次开发也有一年了，写点东西给它呗。

## 介绍

criu完全在用户空间实现，不依赖特定的Linux核模块，它能够冻结正在运行中的应用，扫描该进程的状态与资源并将其记录到文件中；之后criu便可以利用这些文件恢复应用。criu的功能是平台相关的，只能在Linux下使用。并非Linux内核提供的所有功能criu都可以支持，criu采用白名单的方式支持能够被保存、恢复的进程状态与资源，对于有状态的资源，比如TCP链接，需要在内核中定制一些特别的功能以恢复这些资源。

## 基本原理

criu的基本原理很hack，采用了类似gdb的debug技术来控制、获取进程的信息。这些代码集中在进程的中断、寄生控制与恢复上。

### 冻结进程

`compel_interrupt_task()`[[1]]是criu用于默认中断（或者称之为冻结）进程的处理函数，核心是`ptrace(PTRACE_SEIZE)`与`ptrace(PTRACE_INTERRUPT)`。

在`ptrace()`中，观察者进程称为tracer，被观察者进程叫tracee。令一个进程成为tracee`ptrace()`提供了两类方法：1. 由进程自身调用`ptrace(PTRACE_TRACEME)`；2. 由A进程调用`ptrace(PTRACE_ATTACH)`+`ptrace(PTRACE_SEIZE)`令B进程成为tracee。他们三者的区别在于：tracee调用`PTRACE_TRACEME`后，tracer要调用`PTRACE_ATTACH`或`PTRACE_SEIZE`令其与tracee关联起来，`PTRACE_ATTACH`会打断tracee的执行，而与`PTRACE_SEIZE`不会，`PTRACE_SEIZE`与`PTRACE_INTERRUPT`的共同使用可以达到**类似**`PTRACE_ATTACH`的效果。

当tracee被`PTRACE_ATTACH`中断时，tracee会收到由tracer发出的SIGSTOP信号，该信号在系统调用执行完毕返回用户空间时进行处理（`getsignal()`[[2]]）。`PTRACE_SEIZE`+`PTRACE_INTERRUPT`则有些不同，tracee收到的是SIGTRAP信号，该信号是在seized的tracee从内核空间返回用户空间进行信号处理前，检测到自身被设置了JOBCTL_TRAP[[3]]时向tracer发送的。这也是使用*类似*描述`PTRACE_ATTACH`与`PTRACE_SEIZE`+`PTRACE_INTERRUPT`的原因。

> Note: Linux下的信号处理的，后续打算仔细总结下。

criu也可以通过`--freeze-cgroup`选项指定cgroup freezer controller作为冻结进程的方式。

### 寄生控制

criu的寄生控制有两类，一类是控制tracee执行特定的系统调用，另一类是控制tracee在特定的上下文中执行一段代码。

Linux提供了`ptrace(PTRACE_SYSCALL)`用于监听进程的系统调用执行情况，它的hook点有两处：第一处是进程进入系统调用流程后、执行系统调用前[[4]]，第二处是进程执行系统调用完毕后、从系统调用流程退出前[[5]]。若在`ptrace(PTRACE_SYSCALL)`的第一处hook点使用`ptrace(PTRACE_SETREGSET)`系统调用参数，便可以令tracee执行特定的系统调用了。这即是criu控制tracee执行特定系统调用的基本原理。

很多系统调用的使用具有上下文，比如`read()`的参数有一个指针，上述方法对此就无能为力了。对此criu通过控制tracee的系统调用令其使用`mmap()`创建了一段内存空间，再将一段地址无关的静态链接代码拷贝进这段内存中，使用`ptrace(PTRACE_SETREGSET)`令tracee在这段内存空间中执行地址无关的代码。criu使用unix socket与tracee进行通信，控制traccee执行特定的PIE code。

### 恢复进程

Linux进程的组织方式为session、process group、process、thread，criu能够以进程树的方式保存程序的信息，那么在恢复程序的时候也可以恢复多进程、多线程的程序，同时也要恢复这些进程间的关系。

criu在恢复程序时，使用`clone()`的方式创建进程与线程。尽管进程与进程间是互相独立的，但是所有进程的同一类资源信息大都被保存在同一个文件中，而且考虑到tracer (criu)与tracee (被恢复的进程)间需要传递信息(比如tracee通知tracer它恢复完成了)与thread间的行为同步，因此criu使用了futex来进行进程、线程间的同步。

当tracee的资源完全恢复时，它需要返回冻结时的状态继续执行，criu采取的措施是sigreturn-oriented programming (SROP)[[6]]：构造冻结时的用户空间栈，使用`sigreturn()`系统调用从当前的执行现场返回冻结时的状态。criu使用的是breakpoint或者`ptrace(PTRACE_SYSCALL)`捕获被恢复进程的`sigreturn()`事件。

> Note: breakpoint / watchpoint是cpu硬件实现的一种debug机制，也是`ptrace()`的request之一，但它是硬件相关的，不同架构下有不同的request。

### 资源的保存与恢复

criu采用遍历`/proc/<pid>/`目录下特定文件的方式来搜集进程的信息，对于无法从外部搜集到信息，criu采用寄生控制的方式让进程主动向它报告。criu采用白名单的方式支持能够被保存、恢复的进程状态与资源，使用protobuf将搜集到的信息记录在文件中。

内存映射（即vma）的有关信息可以从`/proc/<pid>/smaps`中获取。vma的类型可以分为两大类：匿名内存映射与文件映射。后者还需要记录有关文件的信息，它可以从`/proc/<pid>/map_files/`下获取被映射的文件。

至于文件描述符，有些文件属性是运行时具有的，无法通过读取文件系统中的元数据来取得。并且有些文件描述符关联的是匿名文件（ *anon_inode:* 前缀）或者orphan inode（ *(deleted)* 后缀），无法通过open `/proc/<pid>/fd/<fd>`来打开，因此需要寄生控制使用`sendmsg()`将文件描述符从需要保存的进程中传递给criu。每一类文件描述符需要保存何种信息，这是需要定制，相关的代码在criu的`images`目录下。

## 一些问题与局限性

criu并非十全十美，在实际使用上，尤其保存的进程与硬件驱动强相关、或者使用自定义文件行为的内核模块时，它并不能够直接使用，这些被用到的资源都需要定制。

从使用者的视角看，对于文件映射，Linux的文件描述符并不与特定的vma相关联，因此criu在保存文件映射信息时无法记录其关联着的文件描述符，恢复文件映射时先打开一个文件描述符，映射恢复完毕后再把这个文件描述符关闭。这就导致当自定义内核模块文件的close操作与资源释放相关联时，在恢复该类型文件映射的vma时需要对其进行特别的处理。

criu能够很好地回复无状态的资源，但对于有状态的资源，比如unix socket、tcp socket，他们往往有两端，若这两端的状态都由被保存的程序维护，那么并不需要进行特别的处理，可若只有一端由被保存的程序维护，那么该类资源的恢复往往需要通过修改Linux内核来达到目的。

后者的典型场景是tcp socket：tcp是有状态的，若令一个刚创建的tcp socket恢复至特定的establish状态，必然需要另一端的配合，因此criu提出了repair mode[[7]]用于协助tcp socket的恢复；同时为了阻止另一端在进程消失期间对tcp关闭事件的感知，criu使用netfilter来drop另一端发来的tcp packet，这就构造了网络链路丢包的假象，因为本身网络层就是不可靠的，这就在另一端看来此时就是发生了网络异常罢了。

类似但与之不同的是unix stream socket，unix stream socket在Linux内核的实现上类似pipe，它不像网络一样承载上层数据的是不可靠协议，因此一端被关闭时，另一端在做IO操作时是能够感知到的。那么在为这类资源定制Linux内核时，就需要采取一种机制来达到另一端无法感知socket关闭事件、数据不会丢失的效果。一种思路是悬挂被保存进程的unix stream socket、阻塞另一端的IO；另一种思路是为unix stream socket实现类似tcp+ip协议组合的机制，通过防火墙丢弃另一端传输过来的数据伪造本端在线的场景，使用序列号机制确认数据被送达。

## 性能优化方向

criu的性能并不出色，在实践上，先后从以下几个方向优化过criu的性能。

**使用nftables/iptables API替代iptabes/nft命令**：当前criu通过调用iptables/nft命令对网络链接进行封锁，若被保存的进程在host netns上具有大量的tcp establish链接，这个速度会非常慢（十毫秒级吧），采用nftables netlink API则可节约90%+的tcp lock时间[[8]]。

**捕获特定的系统调用**：criu默认情况下使用`ptrace(PTRACE_SYSCALL)`将tracee停止在特定的指令上，`ptrace(PTRACE_SYSCALL)`无法捕获特定的系统调用，tracee与tracer直接的通信也是十毫秒级的，当进程数过多时，尤其在进程恢复末尾会占用较多的时间。因此，criu为x86实现了breakpoint机制[[9]]，它利用cpu硬件实现的debug feature令进程在指定的地址处暂停，自己尝试实现了arm64的版本[[10]]。另一种软件实现的加速机制是定制`ptrace(PTRACE_SYSCALL)`，令其能够捕获指定的系统调用。

**异步的内存释放**：当进程映射了大量的内存时，进程退出时`mmput()`[[11]]的执行会出现秒级的情况。因此可以采用异步的内存释放（`mmput_async()`[[12]]）方式来减少进程的退出时间。

**使用搜集文件信息的快速路径**：criu搜集文件描述符fd与文件映射vma时需要读取`/proc/<pid>/fdinfo/<fd>`与`/proc/<pid>/smaps`的内容，大量的文件open、read、close会带来大量的malloc、kvmalloc操作，使用定制的API短路掉这些操作，在需要搜集成千上万个fd与vma的情况下有利于性能的提升。

[1]: https://github.com/checkpoint-restore/criu/blob/v3.16.1/compel/src/lib/infect.c#L115
[2]: https://github.com/torvalds/linux/blob/v5.18/arch/x86/kernel/signal.c#L867
[3]: https://github.com/torvalds/linux/blob/v5.18/kernel/signal.c#L2716
[4]: https://github.com/torvalds/linux/blob/v5.18/kernel/entry/common.c#L63
[5]: https://github.com/torvalds/linux/blob/v5.18/kernel/entry/common.c#L249
[6]: https://en.wikipedia.org/wiki/Sigreturn-oriented_programming
[7]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ee9952831c
[8]: https://gitee.com/mistyriver/criu/blob/openEuler-20.03-LTS-Next/backport-0034--nftables-implement-nft-api-for-tcp.patch
[9]: https://github.com/checkpoint-restore/criu/commit/0b1b815
[10]: https://github.com/checkpoint-restore/criu/pull/1815
[11]: https://github.com/torvalds/linux/blob/v5.18/kernel/fork.c#L1200
[12]: https://github.com/torvalds/linux/blob/v5.18/kernel/fork.c#L1218