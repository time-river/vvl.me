---
title: 如何提高代码运行效率
date: 2016-02-13
categories: style
tags: ['language', 'python']
---

这里总(抄）结（袭）一些关于单线程、多线程与多进程，阻塞、非阻塞，同步、异步，协程的东西。

## 线程与进程

现代操作系统都是支持“多任务”的操作系统。什么是多任务呢？打个比方：一边在用浏览器上网，一边在听MP3，一边在用Word赶作业，这就是多任务。过去CPU即使是单核，也可以执行多任务：操作系统让各个任务交替执行，任务1执行0.01秒，切换到任务2，任务2执行0.01秒，再切换到任务3，执行0.01秒……表面上看，每个任务都是交替执行的，但是，由于CPU的执行速度实在是太快了，感觉就像所有任务都在同时执行一样。真正的并行执行多任务只能在现在的多核CPU上实现，但任务数量是远远多于CPU核心数的，所以操作系统也会自动把很多任务轮流调度到每个核心上执行。

对于操作系统来说，一个任务就是一个进程（Process），一个进程至少有一个线程（Thread）。多任务的实现有这三种方式，并且可以这么理解：

* 单进程单线程：一个人在一个桌子上吃菜。
* 单进程多线程：多个人在同一个桌子上一起吃菜。
* 多进程单线程：多个人每个人在自己的桌子上吃菜。

要实现多任务，通常会设计成 Master-Worker 模式， Master 负责分配任务， Worker 负责执行任务；因此多任务环境下，通常是一个 Master，多个 Worker。如果用多线程实现 Master-Worker，主线程就是 Master，其他线程就是 Worker。如果用多进程实现 Master-Worker，主进程就是 Master，其他进程就是 Worker。

下面浅显的谈一谈线程与进程的优劣。

多线程与多进程的目的都是想尽可能的利用 CPU 资源，减少 CPU 的空闲时间，特别是多核环境。对多进程来说：它最大的优点是稳定性高，因为一个子进程崩溃里，不会影响其他进程（当然主进程挂了就全挂了，不过概率极低）；但创建进程的开销巨大，操作系统能同时运行的进程数也是有限的。多线程呢：比多进程快那么一点；但是致命缺点是任何一个线程崩溃都可能直接造成整个进程的崩溃，因为所有线程共享进程的内存。无论是多线程还是多进程，只要 Worker 数量一多，效率肯定上不去，因为 Worker 一多， CPU 就忙着切换工作，根本没多少时间去执行任务了。

是否采用多任务，还得考虑任务类型。

可以把任务划分为计算密集型和 IO 密集型。计算密集型任务的特点是要进行大量的计算，消耗 CPU 的资源，比如对视频进行高清解码、计算圆周率。这种任务同时进行的数量应当等于 CPU 的核心数，这么做是为了降低切换任务所需要的时间，并且它还适合使用运行效率高的代码编写 IO 密集型任务呢，涉及到网络、磁盘的 IO，这类任务的特点是 CPU 消耗很少（CPU 的速度远远高于 IO 的速度），任务的大部分时间都在等待 IO 操作完成。对于这类任务，任务越多， CPU 效率越高，但也有一个限度。

对与 IO 密集型任务，还可以通过使用异步的方式来提高代码效率。不过需要先理解阻塞与非阻塞。

## 阻塞与非阻塞

阻塞与非阻塞关注的是**程序在等待调用结果时候的状态**。

阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。

抄个栗子：

打电话问书店老板有没有《分布式系统》这本书，如果是阻塞式调用，会一直把自己“挂起”，直到得到这本书有没有的结果，如果是非阻塞式调用，不管老板有没有告诉你，你自己先一边去玩了。在这里**阻塞与非阻塞与是否同步异步无关**。跟老板通过什么方式回答你结果无关。

## 同步与异步

同步和异步关注的是**消息通信机制**。

这次山寨个栗子：

我的名字叫哈雅贴。是一位称职的管家。一个阳光明媚的早晨，我准备好了早饭，神清气爽地去喊楼上妹子们起床。可是妹子们都很懒的... 于是：

同步：

我来到妹子们的门前，不管用什么方式通知，都得守在门外等不是？毕竟不知道妹子什么时候会起床，要等到她们出来后才能给她们盛饭，可不能让人家干等着吃不上饭那。

这种尴尬的局面就叫**同步**。

异步：

我可没那么傻。于是我要到了所有妹子的手机号（嘿嘿）。以后，只要“起床了！”一喊完，我就可以潇洒地、头也不会地爱干什么干什么去了。不管是在屋子里画大饼，还是跑出去维护世界和平，都行，完全不用考虑妹子们有没有起床。因为每个人起来时候都会给我打电话，我只要在接到电话后回去就好了。什么，万一我跑得老远了才接到电话，要怎么回去？没问题，电脑世界里的上下文切换就跟瞬间移动似的。不管我走多远，切换一次的耗时都是 O(1)，不会线性增长）

这种局面就叫做**异步**。

我拷贝一个观点——**目前应用中阻塞和非阻塞是针对同步应用而言的。**对异步来说，原理上也应有阻塞、非阻塞的区别，毕竟他们针对的是两件事情。

IO 模型就是处理 IO 操作的方式。 Linux 中常见的 IO 模型如下：

* 同步模型 (Synchronous)
  * 阻塞模型 (Blocking)
  * 非阻塞模型 (Non-Blocking)
  * IO 多路复用 (IO multiplexing)
* 异步模型 (Asynchronous)

再抄袭几个栗子，用来解释同步中各个模型的区别：

还是喊妹子们吃饭。

阻塞模型（Blocking）

我来到第一个妹子的门前：“起床了！”，然后就站在这等着。过了半个小时，妹子出来了，我帮她盛好饭，再去叫第二个妹子。

非阻塞模型（Non-blocking）

我发现一直站在门外挺傻的，于是这次，叫完后我就转身走了。随便在周围干点什么，每隔一小会儿回来看看妹子起来了没有。如果起来了，我就去盛饭。然后叫下一个妹子。

IO多路复用（Multiplexing)

这时我又忽然想到，干嘛非得一个一个来呢？既然我都有时间四处转悠了，完全可以一次多叫几个妹子啊？于是我一口气叫醒了所有妹子，然后在楼梯口等着，谁先出来，先帮谁盛饭。（注意到了吗，这和 Non-blocking 下的行为在本质上是一样的）

Multiplexing 的意义是：把原本依次执行的任务，变为同时执行。就像原来的单车道，现在变成了四车道。这样在任务数比较多的情况下，效率能够得到极大的提升。（不过如果任务数很少，那么反而有可能降低效率，因为维护任务队列也是要成本的）。

无论是非阻塞还是多路复用，带来的另一个变化是：任务完成的顺序变得不确定了。例如最后一个任务可能会第一个完成。大部分情况下这不是问题，就像妹子们谁先去吃饭都没关系。不过就算对任务进行处理的顺序真的很重要，也可以通过技术手段解决：只要给每个任务编个号就好了。当一个任务完成时，检查一下，如果还有其他编号比它小的任务没完成，就先把它放置在这，等前面的任务都完成了再处理它。

## 协程

协程，又称微线程，纤程。英文名Coroutine。搜集资料的过程中，发现篇有趣的科普文：

> 没有啥复杂的东西，考虑清楚需求，就可以很自然的衍生出这些解决方案。
> 一开始大家想要同一时间执行那么三五个程序，大家能一块跑一跑。特别是UI什么的，别一上计算量比较大的玩意就跟死机一样。于是就有了并发，从程序员的角度可以看成是多个独立的逻辑流。内部可以是多cpu并行，也可以是单cpu时间分片，能快速的切换逻辑流，看起来像是大家一块跑的就行。
> 但是一块跑就有问题了。我计算到一半，刚把多次方程解到最后一步，你突然插进来，我的中间状态咋办，我用来储存的内存被你覆盖了咋办？所以跑在一个cpu里面的并发都需要处理上下文切换的问题。进程就是这样抽象出来个一个概念，搭配虚拟内存、进程表之类的东西，用来管理独立的程序运行、切换。
> 后来一电脑上有了好几个cpu，好咧，大家都别闲着，一人跑一进程。就是所谓的并行。
> 因为程序的使用涉及大量的计算机资源配置，把这活随意的交给用户程序，非常容易让整个系统分分钟被搞跪，资源分配也很难做到相对的公平。所以核心的操作需要陷入内核(kernel)，切换到操作系统，让老大帮你来做。
> 有的时候碰着I/O访问，阻塞了后面所有的计算。空着也是空着，老大就直接把CPU切换到其他进程，让人家先用着。当然除了I\O阻塞，还有时钟阻塞等等。一开始大家都这样弄，后来发现不成，太慢了。为啥呀，一切换进程得反复进入内核，置换掉一大堆状态。进程数一高，大部分系统资源就被进程切换给吃掉了。后来搞出线程的概念，大致意思就是，这个地方阻塞了，但我还有其他地方的逻辑流可以计算，这些逻辑流是共享一个地址空间的，不用特别麻烦的切换页表、刷新TLB，只要把寄存器刷新一遍就行，能比切换进程开销少点。
> 如果连时钟阻塞、 线程切换这些功能我们都不需要了，自己在进程里面写一个逻辑流调度的东西。那么我们即可以利用到并发优势，又可以避免反复系统调用，还有进程切换造成的开销，分分钟给你上几千个逻辑流不费力。这就是用户态线程。从上面可以看到，实现一个用户态线程有两个必须要处理的问题：一是碰着阻塞式I\O会导致整个进程被挂起；二是由于缺乏时钟阻塞，进程需要自己拥有调度线程的能力。如果一种实现使得每个线程需要自己通过调用某个方法，主动交出控制权。那么我们就称这种用户态线程是协作式的，即是协程。
> 本质上协程就是用户空间下的线程。

反复读几遍，明白了吗？没有，没关系，接着看。

子程序，或者称为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，最后是A执行完毕。所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。子程序调用总是一个入口，一次返回，调用顺序是明确的。而协程的调用和子程序不同。协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。

协程看上去也是子程序，但执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。

举个栗子

```md
function A:
  print("1")
  print("2")
  print("3")

function B:
  print("A")
  print("B")
  print("C")
```

假设由协程执行，在执行A的过程中，可以随时中断，去执行B，B也可能在执行过程中中断再去执行A，结果可能是：

```md
1
2
A
B
C
3
```

但是在A中是没有调用执行B的。协程的执行看起来有点像多线程，但协程的特点在于是一个线程执行。那和多线程比，协程有何优势？

最大的优势就是协程极高的执行效率。因为子程序切换不是线程切换，而是由程序自身控制，因此，没有线程切换的开销，和多线程比，线程数量越多，协程的性能优势就越明显。第二大优势就是不需要多线程的锁机制，因为只有一个线程，也不存在同时写变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。因为协程是一个线程执行，那怎么利用多核CPU呢？最简单的方法是多进程+协程，既充分利用多核，又充分发挥协程的高效率，可获得极高的性能。

总的来说：

协程是一种编程组件，可以在不陷入内核的情况进行上下文切换。Tasks that you execute are called coroutines, and they are scheduled by the event loop.The loop is responsible for deciding which coroutine to begin next.（原谅我的渣英语，无法准确的翻译）

## 参考资料

[廖雪峰Python教程 进程与线程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014319272686365ec7ceaeca33428c914edf8f70cca383000)  
[廖雪峰Python教程 进程VS线程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014319292979766bd3285c9d6b4942a8ea9b4e9cfb48d8000)  
[知乎 怎样理解阻塞非阻塞与同步异步的区别？ 卢毅、陈阿满回答](https://www.zhihu.com/question/19732473)  
[也谈 Python 下的同步，异步，阻塞，非阻塞](http://anjianshi.net/post/yan-jiu-bi-ji/python-blocking)  
[协程的好处是什么？ 阿猫的回答](https://www.zhihu.com/question/20511233/answer/24260355)  
[廖雪峰Python教程 协程](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432090171191d05dae6e129940518d1d6cf6eeaaa969000)  
[Python 中的进程、线程、协程、同步、异步、回调](https://segmentfault.com/a/1190000001813992)  
[Python 3.5 and asyncio](http://quietlyamused.org/blog/2015/10/02/async-python/)
