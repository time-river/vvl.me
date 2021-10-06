---
title: 'linux下的进程关系与pid namespace'
date: 2018-09-22 19:49:17
tags: ['linux', 'relationship']
---

<!-- TITLE: Relationship -->
<!-- SUBTITLE: A quick summary of Relationship -->

## Overview

Linux使用`struct task_struct`数据结构来关联所有与进程有关的数据和结构，定义了*pid*、*tgid*、*pgid*、*sid*四种进程类型。

Note：

* *pid* —— Process identifier，进程标识符。
* *tid* —— Thread identifier，线程标识符。
* *tgid* —— Thread group identifier，线程组标识符。
* *pgid* —— Process group identifier，进程组标识符。
* *sid* —— Session identifier，会话标识符。

在v4.12中，与四种进程类型有关的系统调用有以下8个：

| | |
| --: | --: |
| getpid | gettid |
| getppid | getpgrp |
| getpgid | setpgid |
| getsid | setsid |

## Introduction

- *pid* —— 在特定PID namespaces中，Linux使用*pid*唯一标识一个进程。使用`fork / clone`创建一个进程均会被分配一个新的且唯一的*pid*。
- *tid* —— Linux的线程由Native POSIX Thread Library（NPTL）实现，本质上是用进程模拟线程，因此内核中并没有*tid*这种数据，它由*pid*表示。线程是通过带有`CLONE_THREAD`*flag*参数`clone`建立的进程。
- *tgid* —— 同一个进程通过`clone`建立了的线程处于同一个线程组，该线程组的ID叫做*tgid*，线程组组长的*tgid*与其*pid*相同。若一个进程没有线程，则其*tgid*与*pid*也相同。
- *pgid* —— 进程组是一个或多个进程的集合，每个进程都属一个进程组，拥有相同的*pgid*。进程组组长（process group leader）的*pid*是进程组的ID，用以识别进程，可通过`setpgrp`为进程设置*pgid*。
- *sid* —— 几个进程组使用`setsid`可以合并成为一个会话组，会话组内的所有进程拥有相同的*sid*。

## Pre-internal

### PID namespace

PID namespaces隔离了进程ID，指的是不同的进程在不同的PID namespaces可以有着相同的PID。这个特性是在主机之间迁移容器的先决条件；只有一个PID namespaces的时候，在保持PID不变（保持PID不变是一种需求）的情况下将其迁移到另一个主机可能会失败，因为目标节点上可能存在相同PID的进程。

PID namespaces具有层级顺序，低级的PID namespaces对高级的不可见。一个PID namespaces可以具有多个child PID namespaces，每一个PID namespaces都可以看做parent PID namespaces的一个局部视图。一个新的PID namespaces *B*从当前PID namespaces *A*被创建后，处于当前PID namespaces *A*的进程可以看到*B*中的所有进程，但反之不成立。

![PID Namespaces](/images/18/09/pid-namespaces.png)

新的PID namespaces通过带有`CLONE_NEWPID`*flag*参数的`clone`系统调用创建。新namespace种第一个进程的PID是1，它是这个namespace的init进程，属于此namespace的孤儿进程将会被它收养；若这个进程死亡，则整个namespace都会被中止。

Note: reference to [1][1][2][2][3][3]

### Overview of `struct task_struct`

#### design

##### 一个进程对应一个`struct task_struct`

不考虑进程之间的关系、命名空间，仅仅是一个*pid*对应一个`struct task_struct`，可以设计如下数据结构：
```c
struct task_struct {
    ...
    struct pid_link pids;
    ...
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};

struct pid {
    struct list_head tasks; // 指回pid_link的node
    int nr; // pid
    struct hlist_node pid_chain; // pid hash table node
};
```

能用下图描述：
![task_struct design 1](/images/18/09/task-struct-design-1.png)

上图中，

- `pid_hash[]` —— 是一个hash表的结构，根据`struct pid`的*nr*值哈希到其某个表项，若有多个*nr*值对应到同一个表项，则使用散列表法解决冲突。利用链表的`container_of`机制[13][13]可以利用`tasks`反向得到`task_struct`。
- `pid_map` —— 是一个位图（bitmap），是用来唯一分配*pid*值的结构。

这种设计可以达到：

1. 快速地给新进程在可见的命名空间内分配一个唯一的*pid*
2. 快速地利用`task_struct`找到`pid`
3. 快速地利用`pid`反向得到`task_struct`

##### 进程区分了*id*类型

考虑到进程/线程之间的复杂关系，原来的`struct task_struct`中的`pid_link`需要增加几项，用以指向到其组长进程的*pid*，相应的`struct pid`也需要增加几项用以链接那些以该*pid*为组长的所有进程组组内进程：

```c
enum pid_type {
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
}；

struct task_struct {
    ...
    struct pid_link pids[PIDTYPE_MAX];
    struct task_struct *group_leader; // thread group leader

    struct pid_links pids[PIDTYPE_MAX];
        ...
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};

struct pid {
    struct list_head tasks[PIDTYPE_MAX]; // 指回pid_link的node
    int nr; // pid
    struct hlist_node pid_chain; // pid hash table node
};
```

新的结构设计示意图如下：
![task_struct design 2](/images/18/09/task-struct-design-2.png)

##### 增加了*pid namespaces*的`struct task_struct`

在第二种情形下再增加*pid namespaces*，同一个进程在不同的*pid namespaces*下有不同的*pid*，因此新的数据结构如下：

```c
enum pid_type {
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
}；

struct task_struct {
    ...
    struct pid_link pids[PIDTYPE_MAX];
    struct task_struct *group_leader; // thread group leader

    struct pid_links pids[PIDTYPE_MAX];
        ...
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};

struct pid {
    unsigned int level; // 表示pid namespaces level
    struct list_head tasks[PIDTYPE_MAX]; // 指回pid_link的node
    struct upid numbers[1]; // 指向特定命名空间的upid
};

struct upid {
    struct hlist_node pid_chain; // pid hash table node
    int nr; // pid
    struct pid_namespace *ns; // 该进程所属的命名空间
};
```

最终成了这样：
![task_struct design 3](/images/18/09/task-struct-design-3.png)

Note：`upid`是`unique pid`的缩写。

#### `real_parent` vs `parent`

`struct task_struct`中有俩*parent*：

```c
struct task_struct {
    ...
    /* Real parent process: */
    struct task_struct __rcu    *real_parent;

    /* Recipient of SIGCHLD, wait4() reports: */
    struct task_struct __rcu    *parent;
        ...
}
```

Wikipedia中的parent process中有提到[4][4][5][5][6][6]，大意是：
> Linux内核的process与POSIX threads差异极其小，对于每个进程/线程来说都有两种类型的parent process，分别为*real_parent*与*parent*。*Real_parent*仅仅是使用`clone`创建子进程/线程的的进程，对创建出来的进程/线程并没有控制权，而*parent*进程才是在子进程/线程中止的时候接收`SIGCHLD`的进程。通常情况下，他们的值是相同的；但对于POSIX threads来说，他们的值可能会有差异。

## Internal

### `getpid` & `gettid` & `getpgrp` & `getpgid` & `getsid`

在linux v4.12的*include/linux/pid.h*中定义了五种与进程/线程有关的id类型：

```c
enum pid_type
{
    PIDTYPE_PID,   // pid
    PIDTYPE_PGID,  // pgid
    PIDTYPE_SID,   // sid
    PIDTYPE_MAX,
    /* only valid to __task_pid_nr_ns() */
    __PIDTYPE_TGID // tgid
};
```

实际上，与进程有关的各类id都是*group_leader*的id，与线程有关的*tid*才是线程所属进程的*pid*。至于`__PIDTYPE_TGID`，它有着如下做法：

```c
        if (type != PIDTYPE_PID) {
            if (type == __PIDTYPE_TGID)
                type = PIDTYPE_PID;
            task = task->group_leader;
        }
```

因此，源码中有了以下的注释（*thread group id*即*process group leader pid*）

```c
/*
  * sys_getpid - return the thread group id of the current process
  * 
  * Note, despite the name, this returns the tgid not the pid.  The tgid and
  * the pid are identical unless CLONE_THREAD was specified on clone() in
  * which case the tgid is the same in all threads of the same group.
  */
SYSCALL_DEFINE0(getpid)
...

/* Thread ID - the internal kernel "pid" */
SYSCALL_DEFINE0(gettid)
...
```

下图展示了一个进程`fork`一次、每个进程再`pthread_create`两次的大致关系[8][8]：

![pid & tid & grp & sid](/images/18/09/pid-tid-grp-sid.png)

### `setsid`

`setsid`的实现中，有这么一行代码：

```c
    proc_clear_tty(group_leader);
```

它的作用使脱离终端。《那些永不消逝的进程》[7][7]一文对此有相应的解释，梗概如下：

忽略`SIGHUP`信号的子进程不会因父进程的退出而退出，这是通过*nohup*运行的子程序可以在后台保持运行、不随终端退出而中止的原理。但是，为什么忽略了`SIGHUP`信号的子进程就不会随着父进程的结束而消逝？在什么样的场景下，一个进程会收到`SIGHUP`信号呢？这与Linux系统中描述进程关系的两个术语*进程组（process group）*和*会话（session）*有关。

在早期Unix的设计中，每当有一个终端（terminal）通过某一tty来访问服务器，一个包含login shell进程的进程组就会被建立起来，所有在该shell中被建立的进程都会自动隶属于同一进程组之下，同时该tty也会被设置成该进程组下所有进程共有的终端控制器（controlling terminal）。但是，进程组对控制终端缺乏有效的管理手段、所有进程无差别地共享控制终端等的设计带来了许多弊端。因此作业控制（job control）的概念被提了出来，会话的设计也被引入，简单地说：新的设计将控制终端（tty或pty）的访问和控制完全置于了会话的管理之下。

`SIGHUP`是当终端连接中断或关闭时，直接或间接地被发送给会话中的所有进程组，一般进程对于该信号的默认处理方式也同样是终结自己。但是，*nohup*通过屏蔽`SIGHUP`的方式来实现守护进程并不理想，会出现缺少控制终端（`SIGHUP`不再起作用）、守护进程的工作目录无法`umount`等一系列问题。

Glibc把实现守护进程所需的工作封装到了`daemon`函数中，通过`fork`创建出的子进程执行`setsid`、同时父进程退出来实现，并且执行`setsid`成功的进程不仅成为新的会话组长和新的进程组长，还不会关联任何终端。

TODD：Question —— 为什么当前进程为process group leader的时候要使调用失败？

```c
    if (pid_task(sid, PIDTYPE_PGID))
        goto out;
```

## References

- [LWN.net: PID namespaces in the 2.6.24 kernel][1]
- [Linux Programmer's Manual: pid_namespaces][2]
- [Professional Linux Kernel Architecture - Linux][3]
- [Wikipedia: Parent process][4]
- [trace 30個基本Linux系統呼叫第九日：getpid與getppid][5]
- [A Sneak-Peek into Linux Kernel - Chapter 2: Process Creation][6]
- [IBM developerWorks：那些永不消逝的进程][7]
- [cnblogs: Linux 内核进程管理之进程ID][8]
- [CSDN: Linux中的进程关系详解][9]
- [CSDN: Linux内核原理-pid namespace][10]
- [LWN.net: Namespaces in operation, part 3: PID namespaces][11]
- [CSDN: linux内核PID管理][12]


[1]: https://lwn.net/Articles/259217/ "LWN.net: PID namespaces in the 2.6.24 kernel"
[2]: http://man7.org/linux/man-pages/man7/pid_namespaces.7.html "Linux Programmer's Manual: pid_namespaces"
[3]: https://cse.yeditepe.edu.tr/~kserdaroglu/spring2014/cse331/termproject/BOOKS/ProfessionalLinuxKernelArchitecture-WolfgangMauerer.pdf "Professional Linux Kernel Architecture - Linux"
[4]: https://en.wikipedia.org/wiki/Parent_process#Linux "Wikipedia: Parent process"
[5]: https://ithelp.ithome.com.tw/articles/10185515 "trace 30個基本Linux系統呼叫第九日：getpid與getppid"
[6]: https://sunnyeves.blogspot.com/2010/09/sneak-peek-into-linux-kernel-chapter-2.html "A Sneak-Peek into Linux Kernel - Chapter 2: Process Creation"
[7]: https://www.ibm.com/developerworks/cn/linux/1702_zhangym_demo/index.html "IBM developerWorks：那些永不消逝的进程"
[8]: https://www.cnblogs.com/hazir/p/linux_kernel_pid.htm "cnblogs: Linux 内核进程管理之进程ID"
[9]: https://blog.csdn.net/zhangyifei216/article/details/49705515 "CSDN: Linux中的进程关系详解"
[10]: https://blog.csdn.net/zhangyifei216/article/details/49926459 "CSDN: Linux内核原理-pid namespace"
[11]: https://lwn.net/Articles/531419/ "LWN.net: Namespaces in operation, part 3: PID namespaces"
[12]: https://blog.csdn.net/zhanglei4214/article/details/6765913 "CSDN: linux内核PID管理"
[13]: https://linux.cn/article-7321-1.html "Linux 内核里的数据结构——双向链表"
