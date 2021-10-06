---
title: Linux下的wait()
date: 2019-12-27 14:55:11
tags: ['linux', 'syscall']
categories: style
---

## Background

写了点代码，用于测试开启seccomp机制下的系统调用是否正常工作，思路很简单——先创建子进程，再执行系统调用测试用例，通过使用测试用例返回的`errno`打印测试用例错误信息，考虑到非测试系统调用的报错，决定返回`-errno`。核心代码如下：

```c
// main function
#define REPORT(func, status)    \
    do {    \
        if ((status) < 0) { \
            fprintf(stdout, "run '%s' failed during other syscall: %s\n",   \
                    (func), strerror(-status)); \
        } else if ((status) != EXIT_SUCCESS) {  \
            fprintf(stdout, "'%s' test failed: %s\n",   \
                            (func), strerror(status));  \
        } else {    \
            fprintf(stdout, "'%s' test pass!\n", (func));   \
        }   \
    } while(0)

int main(int argc, char *argv[]) {
    char *syscalls[] = {};
    ...
    for (int i = 0; i < sizeof(syscalls)/sizeof(char *); i++) {
        /* 构建执行路径 */
        retval = snprintf(path, sizeof(path), "%s/%s", prefix, syscalls[i]);
        path[retval] = '\0';

        /* fork子进程并execv */
        retval = posix_spawnp(&prog, path, NULL, NULL, argv, environ);
        if (retval != 0) {
            fprintf(stderr, "posix_spawnp('%s') failed: %s\n",
                            path, strerror(errno));
        }

        waitpid(prog, &status, 0);

        if (retval == 0)
            REPORT(syscalls[i], status);
    }
    ...

// write测试用例
int main(int argc, char *argv[]) {
    ...
    fd = syscall(__NR_open, "/dev/null", O_WRONLY);
    if (fd < 0)
        return -errno;

    syscall(__NR_write, fd, buffer, strlen(buffer));

    return errno;
}
```

上面的代码有两个大问题：

1. `waitpid()`的返回
2. `waitpid(prog, &status, 0)`执行后对`status`的处理

## Question Answer

### `waitpid()`的返回

`waitpid()`返回的是进程pid，印象中存在不成功等待子进程退出便返回的情况(被信号中断)，便查阅了《Linux环境编程》与[man page][1]，修改成了这样：

```c
while (waitpid(prog, &status, 0) == -1 && errno == EINTR)
    continue;
```

### 子进程的返回值(exit code)

`waitpid()`的参数`status`记录了子进程的返回值(即main func最后一行的`return xxx`，或`exit(xxx)`中的xxx)。根据[man page][1]与[bits/waitstatus.h][2]头文件，可以看出`status`的有效位只有16 bits，高8 bit (`status & 0xff00`)代表子进程的返回值，低8 bit (`status & 0x00ff`)指示进程因哪个信号而终止(terminate)，其中0代表正常结束。

查阅到[errno的值][3]当前最大是124，因此修改成了这样：

```c
#define ERRNO_OFFSET    0x80

// write测试用例
int main(int argc, char *argv[]) {
    ...
    fd = syscall(__NR_open, "/dev/null", O_WRONLY);
    if (fd < 0)
        return errno+ERRNO_OFFSET;
    ...

// main function
#define REPORT(func, status)    \
    do {    \
        if (WIFEXITED((status)) &&  \
                WEXITSTATUS((status)) > ERRNO_OFFSET) { \
            fprintf(stdout, "run '%s' failed during other syscall: %s\n",   \
                    (func), strerror(WEXITSTATUS((status))-ERRNO_OFFSET));  \
        } else if (WIFEXITED((status)) &&   \
                    WEXITSTATUS((status)) != EXIT_SUCCESS) {    \
            fprintf(stdout, "'%s' test failed: %s\n",   \
                    (func), strerror(WEXITSTATUS((status))));   \
        } else if (WIFEXITED((status)) &&   \
                    WEXITSTATUS((status)) == EXIT_SUCCESS) {    \
            fprintf(stdout, "'%s' test pass!\n", (func));   \
        } else {    \
            fprintf(stdout, "'%s' test terminated abnormally\n", func); \
        }   \
    } while(0)
```

#### details

有个疑问：`wait()`参数`wstatus`为int类型，为什么只用到了16 bits？在[Stack Overflow][4]看到了下面的回答：

> [POSIX requires][5] that the full exit value be passed in the `si_status` member of the `siginfo_t` structure passed to the SIGCHLD handler, if it is appropriately established via a call to `sigaction` with `SA_SIGINFO` specified in the flags:
>> If si_code is equal to CLD_EXITED, then si_status holds the exit value of the process; otherwise, it is equal to the signal that caused the process to change state. **The exit value in si_status shall be equal to the full exit value** (that is, the value passed to _exit(), _Exit(), or exit(), or returned from main()); **it shall not be limited to the least significant eight bits of the value**.
> (Emphasis mine).
>
> **Note that upon testing, it appears that Linux does not honour this requirement and returns only the lower 8 bits of the exit code in the si_status member**. Other operating systems may correctly return the full status; FreeBSD does. [See test program here][6].
>
> Be wary, though, that is not completely clear that you will receive an individual SIGCHLD signal for every child process termination (multiple pending instances of a signal can be merged), so this technique is not completely infallible. It is probably better to find another way to communicate a value between processes if you need more than 8 bits.

大意是：POSIX标准并没有限制进程退出时候的返回值bit数，Linux自己做了8 bits的限制，FreeBSD没有这种限制。每个子进程结束后父进程都会收到SIGCHLD信号，也可以通过处理SIGCHLD信号获取子程序的返回值。

```c
sig_atomic_t exit_status;

void sigchld_handler(int n, siginfo_t *si, void *v) {
    exit_status = si->si_status;
}

int main(int argc, char **argv) {
    ...
    struct sigaction act;

    act.sa_sigaction = sigchld_handler;
    act.sa_flags = SA_SIGINFO;
    sigemptyset(&act.sa_mask);

    sigaction(SIGCHLD, &act, NULL);
    ...
}
```

更进一步地，Linux是怎么限制进程退出时候的返回值为8 bits的？

系统调用[`exit()`][7]与[`exit_group()`][8]中有代码，还有`wait4()`[获取exit code][9]：

```c
/* in kernel/exit.c */
SYSCALL_DEFINE1(exit, int, error_code)
{
    do_exit((error_code&0xff)<<8);
}

SYSCALL_DEFINE1(exit_group, int, error_code)
{
    do_group_exit((error_code & 0xff) << 8);
    /* NOTREACHED */
    return 0;
}

// wait4
static int wait_task_zombie(struct wait_opts *wo, struct task_struct *p)
{
    ...
    status = (p->signal->flags & SIGNAL_GROUP_EXIT)
        ? p->signal->group_exit_code : p->exit_code;
    wo->wo_stat = status;
    ...
out_info:
    infop = wo->wo_info;
    if (infop) {
        if ((status & 0x7f) == 0) {
            infop->cause = CLD_EXITED;
            infop->status = status >> 8;
        } else {
            infop->cause = (status & 0x80) ? CLD_DUMPED : CLD_KILLED;
            infop->status = status & 0x7f;
        }
    ...
```

至于`wait()`获取的进程的低8 bit，或许与这[一行代码][10]有关：

```c
/* in kernel/signal.c */
static void complete_signal(int sig, struct task_struct *p, enum pid_type type)
{
    ...
    signal->flags = SIGNAL_GROUP_EXIT;
    signal->group_exit_code = sig;
    signal->group_stop_count = 0;
    ...
```

### 完整代码

完整代码在[这里](https://gist.github.com/time-river/a554700a47d4927071ee933c075bae6d)。

## Reference

- [Linux Programmer's Manual: wait(2)][1]
- [Stack Overflow: How to get the full returned value of a child process?][4]

[1]: http://man7.org/linux/man-pages/man2/waitid.2.html
[2]: https://github.com/bminor/glibc/blob/glibc-2.26/bits/waitstatus.h
[3]: https://www-numi.fnal.gov/offline_software/srt_public_context/WebDocs/Errors/unix_system_errors.html
[4]: https://stackoverflow.com/questions/50982730/how-to-get-the-full-returned-value-of-a-child-process
[5]: http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/signal.h.html
[6]: https://gist.github.com/davmac314/e4243431ed57eb1ae6dfbf885d72f5ba
[7]: https://github.com/torvalds/linux/blob/v4.19/kernel/exit.c#L939
[8]: https://github.com/torvalds/linux/blob/v4.19/kernel/exit.c#L981
[9]: https://github.com/torvalds/linux/blob/v4.19/kernel/exit.c#L1138
[10]: https://github.com/torvalds/linux/blob/v4.19/kernel/signal.c#L980
