---
title: '译文: Basics of Futexes'
date: 2019-05-20 21:02:09
tags: [translation, futex, ipc]
---

Futex(全称 "fast userspace mutex")机制由IBM在2002年[1][1]提出；在2003年并入主线内核。它的核心思想是尽可能地减少内核的参与，在用户空间使用一种更有效的方法来进行线程的同步。

在这篇文章中给出了futex的概述：他们是怎么工作的，以及他们如何在高层次的APIs与语言中是实现同步原语的。

重要的声明：futexes在Linux内核中属于非常低层次的特性，适合在诸如C/C++标准库这种基础运行时组件中使用。非常不建议在应用代码中使用它。

## 动机

在介绍futex之前，资源的互斥需要使用系统调用（比如`semop`）。因为需要从用户空间切换到内核空间，因此使用系统调用的代价相对昂贵；程序由串行转向并行，与锁有关的操作在总运行的时间中占据了很大的一部分比例。不幸的是，锁并不参与完成任何实际的工作（“业务逻辑”），所作的只有保证对共享资源的访问是安全的。

Futex的提出基于一个观察：大部分情况下锁并没有被争用。如果一个线程试图获取一个空闲的锁，持有它的代价是非常的廉价，这是因为此时很有可能不存在其他的线程试图获取它。因此，我们可以通过先尝试更廉价的原子操作[2][2]，在没有系统调用参与的情况下完成。原子操作很大可能会成功。

但是，总有可能真的出现另一个线程在同时尝试获取锁的事件，这种情况下原子操作可能会失败。此时有两种选择。一种是再次尝试，直到加锁成功（busy-loop）；尽管全是用户空间的操作，但它也可能非常浪费，毕竟在加锁成功之前可能需要经过很长的时间，这段时间里会一直占有CPU资源。另一个可选的方法是在锁被解除（至少大概率下它未被持有）之前“睡眠”；这需要内核的帮助，也就是futex使用的地方。

## futex的简单使用 —— 等待与唤醒

[futex(2) 系统调用][3]在单个API上复用了很多功能。我在这不会讨论任何的高级功能（一些功能相当深奥，也没有再官方文档中提及），仅仅针对`FUTEX_WAIT`与`FUTEX_WAKE`。手册页（man page）给了很好的描述：

> The `futex()` system call provides a method for waiting until a certain
> condition becomes true.  It is typically used as a blocking construct
> in the context of shared-memory synchronization.  When using futexes,
> the majority of the synchronization operations are performed in user
> space.  A user-space program employs the `futex()` system call only
> when it is likely that the program has to block for a longer time
> until the condition becomes true.  Other `futex()` operations can be
> used to wake any processes or threads waiting for a particular
> condition.

简单地说，futex是内核的构造，用于帮助用户空间中的代码在共享的事件中同步。一些用户态的进程（或者线程）会等待事件（`FUTEX_EVENT`）。其他用户态进程可以发送事件（`FUTEX_WAKE`）来通知等待事件的进程。等待的效率很高——在等待过程中它被内核停止执行，只有唤醒信号发来的时候它才被重新调度。

一定要阅读`futex`的手册页；这篇文章不适合当作文档！至少需要阅读`FUTEX_WAIT`与`FUTEX_WAKE`的调用、参数、返回值与可能的错误。

下面是一个[例子](https://github.com/eliben/code-for-blog/blob/master/2018/futex-basics/futex-basic-process.c)，演示了futex在进程间同步的基本使用方法。`main`函数启动了子进程，做了如下的事情：

1. 等待共享内存中`0xA`的写入
2. 把`0xB`写入共享内存中

同时，父进程：

1. 把`0xA`写入共享内存
2. 等待共享内存中`0xB`的写入

这是俩进程之间简单的握手。下面是相关代码：

```c
int main(int argc, char** argv) {
  int shm_id = shmget(IPC_PRIVATE, 4096, IPC_CREAT | 0666);
  if (shm_id < 0) {
    perror("shmget");
    exit(1);
  }
  int* shared_data = shmat(shm_id, NULL, 0);
  *shared_data = 0;

  int forkstatus = fork();
  if (forkstatus < 0) {
    perror("fork");
    exit(1);
  }

  if (forkstatus == 0) {
    // Child process

    printf("child waiting for A\n");
    wait_on_futex_value(shared_data, 0xA);

    printf("child writing B\n");
    // Write 0xB to the shared data and wake up parent.
    *shared_data = 0xB;
    wake_futex_blocking(shared_data);
  } else {
    // Parent process.

    printf("parent writing A\n");
    // Write 0xA to the shared data and wake up child.
    *shared_data = 0xA;
    wake_futex_blocking(shared_data);

    printf("parent waiting for B\n");
    wait_on_futex_value(shared_data, 0xB);

    // Wait for the child to terminate.
    wait(NULL);
    shmdt(shared_data);
  }

  return 0;
}
```

Note，POSIX共享内存API创建了一块共享内存，映射到了两个进程中。因为同一块物理内存在不同的进程中的虚拟地址可能不同[3][4]，因此无法使用常规的指针。

这并非`futex`典型的使用方法，更好的方式是从某处等待一个值，而非到某处等待。这里仅仅是展示`futex`返回值的各种可能性。稍后会在实现mutex的时候展示更规范的用法。

这里是`wait_on_futex_value`：

```c
void wait_on_futex_value(int* futex_addr, int val) {
  while (1) {
    int futex_rc = futex(futex_addr, FUTEX_WAIT, val, NULL, NULL, 0);
    if (futex_rc == -1) {
      if (errno != EAGAIN) {
        perror("futex");
        exit(1);
      }
    } else if (futex_rc == 0) {
      if (*futex_addr == val) {
        // This is a real wakeup.
        return;
      }
    } else {
      abort();
    }
  }
}
```

这个函数的在`futex`系统调用之上添加循环的作用是，当唤醒是假的时候可以继续等待。这种情况发生在当`val`并非期待值，或者其他进程在该进程之前被唤醒（这种情况无法在演示中发生，单很可能在其他场景下发生）。

Futex的语义很棘手[4]。若futex地址中的值不等于`val`，`FUTEX_WAIT`会立即返回。在我们的例子中，如果紫禁城在父进程写入`0xA`之前发起了等待，这种场景便会发生。例如，这种情况下，`futex`调用会返回`EAGAIN`。

这是`wake_futex_blocking`：

```c
void wake_futex_blocking(int* futex_addr) {
  while (1) {
    int futex_rc = futex(futex_addr, FUTEX_WAKE, 1, NULL, NULL, 0);
    if (futex_rc == -1) {
      perror("futex wake");
      exit(1);
    } else if (futex_rc > 0) {
      return;
    }
  }
}
```

这是`FUTEX_WAKE`的封装，无论有多少等待者被唤醒他通常都会立即返回。在我们的例子中，这种等待是握手的一部分，但在许多情况下你不会看到它。

## 对用户空间代码来说futex是内核队列

简单地说，futex是内核为了便于用户空间代码管理的队列。它使不满足特定条件的用户空间代码睡眠，让其他用户空间代码发送消息唤醒等待的进程。之前我们提到使用busy-loop的方法来等待原子操作的成功执行；内核管理的队列作为一种可选的高效方案，解决了用户空间代码在循环等待过程中对CPU资源过多占用的情况。

下图来自LWN ["A futex overview and update"](https://lwn.net/Articles/360699/)：

![the relationships of user threads and futex queue](/images/19/05/futex-lwn-diagram.png)

在Linux内核中，futex相关源码位于`kernel/futex.c`。内核维护一个由地址作为关键字的hash table，通过它来快速查找队列中正确的数据结构，再把调用的进程添加进查找到的等待队列中。当然，因此内核本身的细粒度锁以及各种高级特性，因此futex本身存在相当多的复杂性。

## FUTEX_WAIT的超时阻塞

`futex`系统调用有参数`timeout`，它可以使等待的进程超时返回。

`futex-wait-timeout`[例子](https://github.com/eliben/code-for-blog/blob/master/2018/futex-basics/futex-wait-timeout.c)展示了这种用法。下面是与子进程等待futex有关的源码：

```c
printf("child waiting for A\n");
struct timespec timeout = {.tv_sec = 0, .tv_nsec = 500000000};
while (1) {
  unsigned long long t1 = time_ns();
  int futex_rc = futex(shared_data, FUTEX_WAIT, 0xA, &timeout, NULL, 0);
  printf("child woken up rc=%d errno=%s, elapsed=%llu\n", futex_rc,
         futex_rc ? strerror(errno) : "", time_ns() - t1);
  if (futex_rc == 0 && *shared_data == 0xA) {
    break;
  }
}
```

若等待时间超过500ms，进程会再次等待。这个例子可以让你设置等待时间以便观察等待效果。

## 使用futex实现简单的mutex

在开篇的动机部分，我解释了futex任何在常见的低争用情况下实现有效的互斥。是时候展示使用futex与原子操作对此的实现了。这基于Ulrich Drepper的*"Futexes are Tricky"*论文的第二种实现。

为了使用标准的原子操作，这里使用C++（C++11可用）。所有的[这里](https://github.com/eliben/code-for-blog/blob/master/2018/futex-basics/mutex-using-futex.cp)下面是最重要的部分：

```c++
class Mutex {
public:
  Mutex() : atom_(0) {}

  void lock() {
    int c = cmpxchg(&atom_, 0, 1);
    // If the lock was previously unlocked, there's nothing else for us to do.
    // Otherwise, we'll probably have to wait.
    if (c != 0) {
      do {
        // If the mutex is locked, we signal that we're waiting by setting the
        // atom to 2. A shortcut checks is it's 2 already and avoids the atomic
        // operation in this case.
        if (c == 2 || cmpxchg(&atom_, 1, 2) != 0) {
          // Here we have to actually sleep, because the mutex is actually
          // locked. Note that it's not necessary to loop around this syscall;
          // a spurious wakeup will do no harm since we only exit the do...while
          // loop when atom_ is indeed 0.
          syscall(SYS_futex, (int*)&atom_, FUTEX_WAIT, 2, 0, 0, 0);
        }
        // We're here when either:
        // (a) the mutex was in fact unlocked (by an intervening thread).
        // (b) we slept waiting for the atom and were awoken.
        //
        // So we try to lock the atom again. We set teh state to 2 because we
        // can't be certain there's no other thread at this exact point. So we
        // prefer to err on the safe side.
      } while ((c = cmpxchg(&atom_, 0, 2)) != 0);
    }
  }

  void unlock() {
    if (atom_.fetch_sub(1) != 1) {
      atom_.store(0);
      syscall(SYS_futex, (int*)&atom_, FUTEX_WAKE, 1, 0, 0, 0);
    }
  }

private:
  // 0 means unlocked
  // 1 means locked, no waiters
  // 2 means locked, there are waiters in lock()
  std::atomic<int> atom_;
};
```

`cmpchg`是对C++原子操作原语的封装：

```c++
// An atomic_compare_exchange wrapper with semantics expected by the paper's
// mutex - return the old value stored in the atom.
int cmpxchg(std::atomic<int>* atom, int expected, int desired) {
  int* ep = &expected;
  std::atomic_compare_exchange_strong(atom, ep, desired);
  return *ep;
}
```

代码中的注释解释了它如何工作的；十分建议阅读Drepper的论文，因为它通过一个更的错误实现来构建了这个实现。这段代码中，传递给`futex`系统调用的一个参数类`int *`，这它是由`std::atomic`类型的`atom_`地址强制转换得到的。这是因为`fute期望一个简单的地址，但是C++原子类型封装了未知的数据类型。这段代码可以在x64Linux上工作，却是不可移植的。为了使`std::atomic`与`futex`的可移植性更好，我须添加一个抽象层。但这并非实践中的需要——`futex`与C++11的协同并非每个人都应的。这段代码仅仅用来演示。

有意思的是成员`atom_`的含义。回想一下，`futex`系统调用不会赋值——赋值操作取决用户。0、1、2约定对mutex非常有用，也是*glibc*对低级锁（low-level）实现的方式。

## glibc mutex与低级锁

这里带来的是*glibc*中POSIX线程与之有关的实现，它的类型是`pthread_mutex_t`。如篇所述，futex并非用于普通用户的代码，它被低级的运行时（runtime）和库（libraries来实现高级的原语。这里会看到[NPTL](https://en.wikipedia.org/wiki/Native_POSIX_Thread_Library)中有关mutex的使用。在*glibc*中的源码树中，这部分代码在`nptl/pthread_mutex_lock.c`。

因为需要对不同类型的mutex支持，这部分代码异常复杂，但如果挖得足够深，我们会发许多熟悉的身影。除了上面提到的文件外，还有（针对x86`sysdeps/unix/sysv/linux/x86_64/lowlevellock.h`与`nptl/lowlevellock.c`。这些码复杂，单原子操作compare-and-exchange与`futex`的组合是显而易见的。低级锁机（`lll_`或者`LLL_`前缀）被广泛用在*glibc*中，不仅仅是POSIX线程。

`sysdeps/nptl/lowlevellock.h`中开头的注释应该是非常熟悉了：

```c
/* Low-level locks use a combination of atomic operations (to acquire and
   release lock ownership) and futex operations (to block until the state
   of a lock changes).  A lock can be in one of three states:
   0:  not acquired,
   1:  acquired with no waiters; no other threads are blocked or about to block
       for changes to the lock state,
   >1: acquired, possibly with waiters; there may be other threads blocked or
       about to block for changes to the lock state.

   We expect that the common case is an uncontended lock, so we just need
   to transition the lock between states 0 and 1; releasing the lock does
   not need to wake any other blocked threads.  If the lock is contended
   and a thread decides to block using a futex operation, then this thread
   needs to first change the state to >1; if this state is observed during
   lock release, the releasing thread will wake one of the potentially
   blocked threads.
 ..
 */
```

## Go runtime中的futex

Go的运行时大部分情况下不使用libc。因此它在自己的代码中无法以来POSIX线程。它直接调用系统调用。

这也是另一个不错地学习futex使用方法的例子。它无法使用`pthread_mutex_t`，所以它必须实现自己的锁。相关的实现是`sync.Mutex`类型（在`src/sync/mutex.go`）中。

`sync.Mutex`的`Lock`方法非常复杂。它首先使用原子交换尝试获取锁。如果必须等待，则会转向`runtime_SemacquireMutex`，在其中调用`runtime.lock`。这个函数定义在`src/runtime/lock_futex.go`[5]，其中定义的常量非常熟悉。

```go
const (
  mutex_unlocked = 0
  mutex_locked   = 1
  mutex_sleeping = 2

...
)

// Possible lock states are mutex_unlocked, mutex_locked and mutex_sleeping.
// mutex_sleeping means that there is presumably at least one sleeping thread.
```

`runtime.lock`也尝试预测性地使用原子操作获取锁；这个函数在Go运行时中的某些地方使用，它的存在是有意义的，但是我不清楚当它被`Mutex.lock`调用的时候，是否无法优化两个连续的原子操作。

若它发现不得不睡眠，会跳转`futexsleep`，它是OS相关的代码，在`src/runtime/os_linux.go`中。这个函数调用具有`FUTEX_WAIT_PRIVATE`的`futex`系统调用（这对于单个进程的Go运行时足够了）。

---

- [[1][1]] See "Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux" byFranke, Russell, Kirkwood. Published in 2002 for the Ottawa Linux Symposium.
- [[2][2]] Most modern processors have built-in atomic instructions implementedin HW. For example on Intel architectures `cmpxhg` is an instruction. Whileit's not as cheap as non-atomic instructions (especially in multi-core systems),it's significantly cheaper than system calls.
- [[3][4]] [The code repository for this post](https://github.com/eliben/code-for-blog/tree/master/2018/futex-basics)also contains an equivalent sample using threads instead of processes. There wedon't need to use shared memory but can instead use the address of a stackvariable.
- [[4][5]] There's a paper written by Ulrich Drepper named *"Futexes are Tricky"*that explores some of the nuances. I'll be using it later on for the mutexdiscussion. It's a very good paper - please read it if you're interested in thetopic.
- [[5][6]] For OSes that expose the `futex(2)` system call. The Go runtime has afallback onto the semaphore system calls if `futex` is not supported.

## [原文：Basics of Futexes](https://eli.thegreenplace.net/2018/basics-of-futexes/)

[1]: https://eli.thegreenplace.net/2018/basics-of-futexes/#id6
[2]: https://eli.thegreenplace.net/2018/basics-of-futexes/#id7
[3]: http://man7.org/linux/man-pages/man2/futex.2.html
[4]: https://eli.thegreenplace.net/2018/basics-of-futexes/#id3
[5]: https://eli.thegreenplace.net/2018/basics-of-futexes/#id9
[6]: https://eli.thegreenplace.net/2018/basics-of-futexes/#id5