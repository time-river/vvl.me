---
title: Linux 如何执行程序 —— 用户态篇（废弃）
date: 2017-03-22 19:38:41
tags:
---

**这篇已经被废弃，[x86 架构下 Linux 的系统调用与 vsyscall, vDSO](/2019/06/linux-syscall-and-vsyscall-vdso-in-x86/)与该内容部分相关，建议参考**

## C/C++ 语言中的 `main` 函数

**第一个问题： C/C++ 中的 `main` 函数可以有几个参数？**

与其称为 `main` 函数，不如称为程序入口(entry point)。典型的形式：

```c
int main();
int main(void);

int main(int argc, char **argv);
int main(int argc, char *argv[]);
```

* argc（argument count）—— 指令列给予的参数数量。
* argv（argument vector）—— 参数数组的指针地址，字符串数组，以 `NULL` 结束。

所以，最多只可以有两个吗？显然不是这样。在一些系统上，有第三个参数，比如 Linux：

```c
int main(int argc, char *argv[], char *envp[])
```

* envp(environment variables path ?) —— 环境变量，字符串数组，以 `NULL` 结束。

写过 Win32 程序的话，对这个不会陌生：

```cpp
int WINAPI WinMain( HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow);
```

所以答案是不确定。`main` 函数只是程序的入口而已。具体的实现看操作系统咯。

参见 [维基百科 主函数](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%87%BD%E5%BC%8F)

...待填坑，没时间写...
