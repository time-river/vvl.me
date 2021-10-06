---
title: Linux Kernel代码中的异常检测与日志类API
date: 2020-12-05 23:21:12
tags: ["linux", "printk"]
---

## Background

Linux Kernel的代码中存在一些异常检测、日志输出相关的API，也存在用于debug字段的内核参数。本文聚焦：

1. 编译时用于静态检查API的种类与作用
2. 运行时日志输出类的API与相关的内核参数
  - 内核参数`kernel.print`与启动参数`loglevel / quiet / debug`
  - Dynamic Debug
  - 日志类API的主要种类
3. 运行时异常检测与处理API

## Force Compiler Error

某些内核代码依赖于特定的静态变量，尽管`Kconfig`提供了语法[`depends on`][1]用于指定模块间的依赖关系，可这种方式不够灵活，无法用于非config间的静态检查，因此提供了在编译期间起作用的静态检查方式，相关的macro声明在[`include/linux/build_bug.h`][2]，根据用处的不同分为以下几种：

- [`BUILD_BUG_ON_ZERO(e)`][3]: 利用C位域不为负值原理，若表达式`e`为真则编译报错`negative width in bit-field '<anonymous>'`，为假则产生int类型的0返回
- [`BUILD_BUG_ON_INVALID(e)`][4]: 在表达式`e`不产生副作用且不产生多余代码的情况下对表达式进行检查，要求表达式的返回值能被强制转换为long类型
- [`BUILD_BUGxxx()`系列][5]: 符合POSIX assert macro语义，允许自定义错误信息。利用了编译器对存在常量表达式条件的分支优化原理，若condition为真则会跳转到异常分支，使用编译器参数`__attribute__((__error__(message)))`输出message
- [`static_assert(expr, ...)`][6]: 不同于`BUILD_BUGxxx()`，它能够放在函数作用域之外，是对编译器关键词[`_Static_assert`][7]的封装

## Kernel Log

Linux能够用`$ dmesg``/dev/kmsg`读取来自内核的日志信息（数据来源于`/dev/kmsg`），这些信息多数是在内核中通过函数`printk()`打印。有内核参数[`kernel.printk`（`/proc/sys/kernel/printk`）][8]控制console loglevel，其值有4个，分别代表*current*、*default*、*minimum*、*boot-time-default*。[*boot-time-default* / *default* / *current*][9]于编译时指定，即*boot-time-default* / *current*为[`CONFIG_CONSOLE_LOGLEVEL_DEFAULT`][10]，*default*为[`CONFIG_MESSAGE_LOGLEVEL_DEFAULT`][11]，**minimum*在源码中静态[定义][12]。可通过修改内核参数`kernel.printk`改变他们的值，但修改*boot-time-default*的值并没有意义（并没有在源码中翻到哪里用到了这个值），通常通过`# dmesg -n 8`（即`echo 8 > /proc/sys/kernel/printk`来令console打印出所有的内核日志，亦可通过kernel boot parameters `loglevel=<loglevel> / quiet / debug`来指定。其中启动参数[`quiet`][27]的日志行为由[`CONFIG_CONSOLE_LOGLEVEL_QUIET`][28]决定，[`debug`][30]的值则为[`CONSOLE_LOGLEVEL_DEBUG (10)`][29]。

> Note: *loglevel*、*quiet*、*debug*三者是互相冲突的，我觉得谁写在后面以谁为准，kernel参数按顺序扫描嘛。

### LOG_LEVEL, Log Types and Usage

#### LOG_LEVEL and Function `printk()`

内核中在[include/linux/kern_levels.h][13]定义了8+1+1类loglevel，使用方式为`printk(loglevel fmt, ...)`。`KERN_DEFAULT`代表的是使用内核参数`kernel.printk`中*defaul*定义的loglevel级别；`KERN_CONT`控制着log flush的时刻：若当前log buffer没满且log message的前缀为`KERN_CONT`、后缀不是`\n`，则该message不立即输出，输出时刻取决于下个message。

```c
#define KERN_EMERG	KERN_SOH "0"	/* system is unusable */
#define KERN_ALERT	KERN_SOH "1"	/* action must be taken immediately */
#define KERN_CRIT	KERN_SOH "2"	/* critical conditions */
#define KERN_ERR		KERN_SOH "3"	/* error conditions */
#define KERN_WARNING	KERN_SOH "4"	/* warning conditions */
#define KERN_NOTICE	KERN_SOH "5"	/* normal but significant condition */
#define KERN_INFO	KERN_SOH "6"	/* informational */
#define KERN_DEBUG	KERN_SOH "7"	/* debug-level messages */

#define KERN_DEFAULT	""		/* the default kernel loglevel */

#define KERN_CONT	KERN_SOH "c"
```

#### Function `pr_xxx()` and `dev_xxx()`, `[subsystem eg: netdev]_xxx()`

相比`printk()`，[Linux更建议][14]使用[`[subsystem eg: netdev]_xxx()`][15]、[`dev_xxx()`][16]、[`pr_xxx()`][17]之类的API（按优先顺序排列，高优先级在前），在实现上他们几乎都是为[`vprintk_emit()`][18]做的封装（除了*debug*级别的log），面向子系统的日志打印API，会在日志中自动添加一些该子系统独有的信息。

`pr_xxx()`对`printk()`做了简单的封装，能够在单个源码文件中利用macro `#define pr_fmt(fmt) xxx`为该文件中的所有`pr_xxx()` API自定义打印的日志前缀或后缀。此外，`pr_xxx()`定义了devel级别的日志打印，当源文件存在macro `#DEFINE DEBUG`时，它等价于`printk(KERN_DEBUG, ...)`，因此偶尔可以在源文件中看到注释掉的macro `//#define DEBUG`。比如[drivers/misc/vmw_balloon.c][19]:

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * VMware Balloon driver.
 *
 * Copyright (C) 2000-2018, VMware, Inc. All Rights Reserved.
 *
 * This is VMware physical memory management driver for Linux. The driver
 * acts like a "balloon" that can be inflated to reclaim physical pages by
 * reserving them in the guest and invalidating them in the monitor,
 * freeing up the underlying machine pages so they can be allocated to
 * other guests.  The balloon can also be deflated to allow the guest to
 * use more physical memory. Higher level policies can control the sizes
 * of balloons in VMs in order to manage physical memory resources.
 */

//#define DEBUG
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

#include <linux/types.h>
#include <linux/io.h>
```

而`pr_debug()`的行为与`printk(DEBUG, ...)`并不等价，分为两种情况：

1. 若开启了dynamic debug功能，它的行为即dynamic debug的行为
2. 未开启dynamic debug，它的行为与`pr_devel()`等价（即是否有日志输出取决于macro `DEBUG`）

#### Dynamic Debug

Dynamic Debug允许用户动态地调整内核日志的输出。开启全局的Dynamic Debug需要设置`CONFIG_DYNAMIC_DEBUG`；启用局部的Dynamic Debug需要设置`CONFIG_DYNAMIC_DEBUG_CORE`，并为启用Dynamic Debug的module添加*ccflags* `DYNAMIC_DEBUG_MODULE`（即`ccflags := -DDYNAMIC_DEBUG_MODULE`）。

Dynamic Debug启用后，允许输出debug信息的函数可在`<debugfs>/dynamic_debug/control`中查询，也可向该文件写入符合语法规范的字符串动态地调整相应日志格式与输出，语法详见[Linux doc: Dynamic debug -- Command Language Reference][21]。

Dynamic Debug支持在启动阶段激活日志的输出。Kernel的启动参数为`dyndbg="QUERY"`；若为内核模块，则需使用参数`module.dyndbg="QUERY"`（其中*module*为模块名称），这等价于在`/etc/modprob.d/*.conf`中采用语法形如`options module dyndbg=QUERY`，或使用modprobe时添加参数`# modprobe module dyndbg=QYERY`，`insmod`用法同`modprobe`。

> Note：`dyndgb`参数能够使用引号`""`、`''`包裹。

#### Source

写了个内核模块测试这部分内容，相关代码在[这](https://gist.github.com/time-river/fff83f3db1b1c9c01168d029aa28a7c7)。

#### Misc

还看到三类不同于`printk()`行为的日志API：

- 一类的后缀为[`_once()`][23]，即相关日志只打印一次的API。它通过在kernel的`.data.once` section静态定义了一个变量来实现。
- 一类的后缀为[`_ratelimited()`][24]，即限制了打印频率的日志API。它采用了令牌桶算法进行限流。
- 一类的后缀为[`_deferred()`][25]，即推迟打印的日志的API。它的原后缀为`_sched()`。

此外，还有一些用于导出内核运行时信息的日志API，比如`print_hex_dump_debug()`、`dump_stack()`，这些可在[`include/linux/printk.h`][26]查阅得到。

## Kernel Check Itself

Linux Kernel中存在一些自我进行[异常检测类的API][31]，行为有两大类：

- *BUG*：输出异常信息，令kernel crash
- *WARN*：仅输出异常信息

*BUG*分为条件BUG `BUG_ON(condition)`与无条件BUG `BUG()`。*WARN*分为条件WARN `WARN_ON(condition)`、自定义信息的WARN `WARN(condition, fmt...)`之外，还有`_ONCE()`后缀的WARN、含有[*TAINT*][32]字段的WARN（可被忍受的异常）。

## References

1. [Linux doc: Message logging with printk][8]
2. [Linux doc: Dynamic debug][20]
3. [eLinux wiki: Debugging by printing][22]

[1]: https://github.com/torvalds/linux/blob/v5.9/fs/Kconfig#L66
[2]: https://github.com/torvalds/linux/blob/v5.9/include/linux/build_bug.h
[3]: https://github.com/torvalds/linux/commit/8c87df457cb58fe75b9b893007917cf8095660a0
[4]: https://github.com/torvalds/linux/commit/baf05aa9271bdbc07d3160035a231abc5fbd429a
[5]: https://github.com/torvalds/linux/commit/9a8ab1c39970a4938a72d94e6fd13be88a797590
[6]: https://github.com/torvalds/linux/commit/6bab69c65013bed5fce9f101a64a84d0385b3946
[7]: https://gcc.gnu.org/wiki/C11Status
[8]: https://www.kernel.org/doc/html/latest/core-api/printk-basics.html
[9]: https://github.com/torvalds/linux/blob/v5.9/kernel/printk/printk.c#L64
[10]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L60
[11]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L48
[12]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L52
[13]: https://github.com/torvalds/linux/blob/v5.9/include/linux/kern_levels.h
[14]: https://github.com/torvalds/linux/blob/v5.9/scripts/checkpatch.pl#L4261
[15]: https://github.com/torvalds/linux/blob/v5.9/include/linux/netdevice.h#L4979
[16]: https://github.com/torvalds/linux/blob/v5.9/include/linux/dev_printk.h#L33
[17]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L308
[18]: https://github.com/torvalds/linux/blob/v5.9/kernel/printk/printk.c#L1987
[19]: https://github.com/torvalds/linux/blob/v5.9/drivers/misc/vmw_balloon.c#L16
[20]: https://www.kernel.org/doc/html/latest/admin-guide/dynamic-debug-howto.html
[21]: https://www.kernel.org/doc/html/latest/admin-guide/dynamic-debug-howto.html#command-language-reference
[22]: https://elinux.org/Debugging_by_printing
[23]: https://github.com/torvalds/linux/commit/f036be96dd9ce442ffb9ab33e3c165f5178815c0
[24]: https://github.com/torvalds/linux/commit/8a64f336bc1d4aa203b138d29d5a9c414a9fbb47
[25]: https://github.com/torvalds/linux/commit/3ccf3e8306156a28213adc720aba807e9a901ad5
[26]: https://github.com/torvalds/linux/blob/master/include/linux/printk.h
[27]: https://github.com/torvalds/linux/blob/master/init/main.c#L238
[28]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L61
[29]: https://github.com/torvalds/linux/blob/v5.9/include/linux/printk.h#L53
[30]: https://github.com/torvalds/linux/blob/master/init/main.c#L232
[31]: https://github.com/torvalds/linux/blob/master/include/asm-generic/bug.h
[32]: https://github.com/torvalds/linux/commit/b2be05273a1744d175bf4b67f6665637bb9ac7a8