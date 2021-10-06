---
title: Linux下的时钟
date: 2019-04-15 14:39:56
tags: ['linux', 'clock']
---

## 时间的基本概念

秒的定义：

1. 以铯133的振荡频率来定义秒
2. 依据地球自转和公转来定义秒

[格林威治平时(Greenwich Mean Time, GMT)][1]采用的是第二种定义，是符合人类习惯的。
由于地球每天的自转是不规则的，而且正在缓慢减速，因此天文观测本身具有缺陷。
它后来被修正为UTC时间。

[协调世界时(Coordinated Universal Time, UTC)][2]采用的是第一种定义。
它基于国际原子钟(International Atomic Time, TAI)，并通过不规则地加入闰秒来抵消地球自转变慢的影响。

闰秒(Leap Second)是在协调世界时中增加或减少一秒，使它与符合人类习惯的时间贴近所做的调整。
TAI时钟与UTC时钟在1972年进行了对准（相差10秒），此后变独立运行了。
在大部分时间点，UTC时钟跟随TAI时钟；但也在适当的时间点，UTC时钟会进行闰秒补偿。
自1972年至2019年1月以来一共进行了27次补偿，即UTC时钟与TAI时钟之间的偏移为37秒。

时间具有绝对和享对的概念。
对一个系统而言，需要定义一个epoch，所有的时间表示是基于这个基准点的。
[UNIX time(also POSIX time, UNIX Epoch time)][3]是从UTC 1970年1月1日0时0分0秒起至现在的秒数。
一个符合POSIX标准的系统必须提供系统时钟，以不小于秒的精度来记录到epoch的时间值。

## Linux下的时钟

### 基础

- 时间的度量

Linux采用基于原子钟的方式定义秒，也就是UTC所定义的时间。

- Epoch

在定义时间轴原点方面，Linux kernel的参考点(linux epoch)采用的同UNIX time一样。
此外，对于系统的启动时间也有一个epoch。

- 时间调整

UTC时间基于原子钟，普通的PC基于系统的本地振荡器来计时，虽然精度不理想，但可以通过NTP协议与外部的时间服务器进行同步来调整时间。
NTP协议使用的基准点是UTC 1900年1月1日0点0分0秒。

时间同步的调整方式有两种：
一是直接设定当前的时间值，这种会导致时间轴上的时间会向前或向后的跳跃，无法保证时间的连续性和单调性；
另一种是对本地振荡器的输出进行矫正，对时间轴缓慢地调整，从而保证了时间的连续性和单调性。

- 闰秒

Linux不关心闰秒的问题，它交给那些时间同步服务器处理。

- 计时范围

有一类特殊的时钟称作秒表，启动后开始计时，中间可以暂停、恢复。
Linux中也有这样的工具，用来计算一个进程/线程的执行时间。
[Elapsed real time(also real time, wall-clock time, wall time)][4]代表着一个程序从开始执行到结束执行所占用的时间。

- 时间精度

最初的Linux只支持10ms级别的时间精度，想要取得us甚至ns级别的精度是不可能的，因此后来发展出了高精度时钟。
虽然高精度时钟出现了，但并没有取代低精度时钟，俩者目前处于共存状态。

> Note:
> [“低精度timer是基于tick的，实际上，高精度timer不是base on tick的，它是基于clock event和clock source的，假设一个晶振是13M，那么每个clock就是1/13个us，通过对clock的计数，当然很容易达到us的精度了。” ][11] 
>
> Q：什么是timer？  
> A：timer是Linux内核中的一类定时器，另一种类型是timeout。使用timer类型的定时器往往要求在精确的时钟条件下完成特定的事件，timeout则相反。  
> Q：什么是tick？  
> A：系统中有许多日常性的事情需要处理，比如更新系统时间，采用按照固定频率的方式来进行这些操作。一般而言，硬件会有HW timer(hardware timer)可以周期性触发中断，让系统去处理日常任务。日常生活中的tick指的是钟表发出的周期性滴答声音，CPU和OS拓展了这个概念：周期性产生的timer中断事件被称为tick，能够产生tick的设备称为tick device。除了periodic tick之外，还有one-shot tick，它设定后只能产生一次tick event，若需要连续产生tick event则需要每次都进行设定。  
> Q：什么是clock event与clock source？  
> A：这里的clock source，应该指的是clock-source device(时钟源设备)，它是可以提供一定精度的计时设备，产生clock event。Clock-event device(时钟事件设备)指的是系统中可以触发one-shot(单次)或周期性中断的设备。Tick device是clock-event device一个wrapper。

- 休眠/关机状态下的时钟

出于一些需要，比如关机闹铃，可以让系统在休眠/关机状态下时钟仍然可以运作，推动事件的发生。

### 总结

Linux定义了以下的时钟：

```c
/*
 * The IDs of the various system clocks (for POSIX.1b interval timers):
 */
#define CLOCK_REALTIME                  0
#define CLOCK_MONOTONIC                 1
#define CLOCK_PROCESS_CPUTIME_ID        2
#define CLOCK_THREAD_CPUTIME_ID         3
#define CLOCK_MONOTONIC_RAW             4
#define CLOCK_REALTIME_COARSE           5
#define CLOCK_MONOTONIC_COARSE          6
#define CLOCK_BOOTTIME                  7
#define CLOCK_REALTIME_ALARM            8
#define CLOCK_BOOTTIME_ALARM            9
#define CLOCK_SGI_CYCLE                 10      /* Hardware specific */
#define CLOCK_TAI                       11
```

- `CLOCK_REALTIME`描述真实世界的时钟。Realtime clock也称为wall time clock。
- `CLOCK_MONOTONIC`单调递增，无法设置，但可通过NTP调整，基准点不一定是Linux epoch
- `CLOCK_MONOTONIC_RAW`类似`CLOCK_MONOTONIC`，是一个完全基于本地晶振的时钟
- `CLOCK_PROCESS_CPUTIME_ID`与`CLOCK_THREAD_CPUTIME_ID`这两个时钟属于CPU-time clock，专门用于计算进程/线程的执行时间
- 后缀`_COARSE`表示这是低精度时钟
- `CLOCK_BOOTTIME`类似`CLOCK_MONOTONIC`，不同点在于它包含计算机的睡眠时间
- 后缀`_ALARM`表示在休眠状态下该时钟仍然递增
- 原子钟`CLOCK_TAI`

根据分类因素，可以将时钟总结如下：

| | leap second? | clock set? | clock tuning? | original poing | resolution | active in suspend? |
| - | - | - | - | - | - | - |
| realtime | yes | yes | yes | Linux epoch | ns | no |
| monotonic | yes | no | yes | Linux epoch | ns | no |
| monotonic raw | yes | no | no | Linux epoch | ns | no |
| realtime coarse | yes | yes | yes | Linux epoch | tick | no |
| monotonic coarse | yes | no | yes | Linux epoch | tick | no |
| boot time | yes | no | yes | machine start | ns | no |
| realtime alarm | yes | yes | yes | Linux epoch | ns | yes |
| boottime alarm | yes | no | yes | machine start | ns | yes |
| tai | no | no | no | Linux epoch | ns | no |

## misc

在timeline上以Linux epoch为参考点，方便了计算机却不方便人类。
人类更习惯broken-down time，也就是年月日时分秒的表示形式。
相关的数据结构是`struct tm`。

传统的unix使用基于秒的时间定义，相关的数据结构是`time_t`，它是POSIX标准定义的一个以秒计的时间值。
微妙精度表示的时间，相关的数据结构是`struct timeval`。
而纳秒，数据结构是`struct timespec`。

[`$ man 7 time`][8]文档也蛮不错的。

## 用户空间接口函数

从应用程序的角度看，Linux kernel/glibc提供的与时间有关的服务分三种：

1. 与系统时间相关的服务
2. 与进程睡眠相关的服务
3. 与timer相关的服务

> Note:  
> 在一系列用户空间接口函数中，POSIX定义了一套，相关的内核源码在*kernel/time/posix-timers.c*中。
> 他们的用户空间接口前缀为`clock_`或`timer_`。  
> POSIX lock具备实现多种时钟的能力。从clock的生命周期看，可以分为静态和动态的POSIX lock。
> 静态一直存在于内核中，比如realtime clock；动态有创建和销毁的概念，某些硬件在提供计实能力的情况下支持热插拔，这种设备可以实现成一个POSIX clock。

### 与系统时间相关的服务

1. 秒级别的时间函数`time() / stime()`
2. 微秒级别的时间函数`gettimeofday() / settimeofday()`
3. 纳秒级别的时间函数`clock_gettime() / clock_settime() / clock_getres`
4. 渐进式的时间调整函数`adjtime() / adjtimex() / clock_adjtime()`

秒级、微妙级的时间函数都与timekeeper模块有关，获取、设置是与其相关的操作。

纳秒级别的时间函数是POSIX定义的那套，调用了clock source模块的read函数来获取更高的精确度。
参数`clk_id`代表的是时钟的种类。

与暴力设定系统时间的`set`系列函数不同，linux提供渐进式的时间调整函数。
`adjtimex()`是根据RFC 5905实现的时间调整函数，比`adjtime()`强大，调整的是timekeeper。
`clock_adjtime()`是支持时钟源调整的函数。

> Note:  
> timekeeper是一个提供时间服务的基础模块，相关代码在*kernel/time/timekeeping.c*中。  
> Linux kernel提供各类timeline(realtime clock, monotonic clock等)，timekeeper负责跟踪、维护各类timeline，并向其他模块（timer相关模块、用户空间的时间服务等）提供服务。  
> timekeeper模块维护timeline的基础是基于clock source模块和tick模块。通过tick模块的tick事件，可以周期性地更新timeline；通过clock source模块，可以获取tick之间更精准的事件信息。

### 与进程睡眠相关的服务

1. 秒级别的睡眠函数`sleep()`
2. 微妙级别的睡眠函数`usleep()`
3. 纳秒级别的睡眠函数`nanosleep()`
4. 参数更多的纳秒级别睡眠函数`clock_nanosleep()`

### 与timer相关的服务

1. `alarm()`函数
2. Interval timer函数`getitimer() / setitimer()`
3. POSIX定义的更高级的timer函数
   - timer创建与删除函数`timer_create() / timer_delete()`
   - timer管理函数`timer_settime() / timer_gettime() / timer_getoverrun()`
4. 使用文件描述符通知的timer函数`timerfd_create() / timerfd_settime() / timerfd_gettime()`

## References

- [Wikipedia: 格林尼治时间GTM][1]
- [Wikipedia: 协调世界时UTC][2]
- [Wikipedia: Unix时间][3]
- [Wikipedia: Elapsed real time][4]
- [蜗窝科技: Linux的时钟][5]
- [IBM developerWorks: Linux时钟管理][6]
- [Embedded Linux Wiki: Kernel Timer Systems][7]
- [Man pages: time - overview of time and timers][8]
- [LWN.net: ptp: IEEE 1588 hardware clock support][9]

[1]: https://en.wikipedia.org/wiki/Greenwich_Mean_Time
[2]: https://en.wikipedia.org/wiki/Coordinated_Universal_Time
[3]: https://en.wikipedia.org/wiki/Unix_time
[4]: https://en.wikipedia.org/wiki/Elapsed_real_time
[5]: http://www.wowotech.net/timer_subsystem/clock-id-in-linux.html
[6]: https://www.ibm.com/developerworks/cn/linux/l-cn-timerm/index.html
[7]: https://elinux.org/Kernel_Timer_Systems
[8]: http://man7.org/linux/man-pages/man7/time.7.html
[9]: https://lwn.net/Articles/420175/
[10]: http://www.wowotech.net/timer_subsystem/time_subsystem_index.html
[11]: http://www.wowotech.net/timer_subsystem/time_concept.html
