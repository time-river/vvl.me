---
title: Python之asyncio
date: 2016-03-12
categories: style
tags: ['language', 'python', 'spider']
---

二月中旬以来一直在看 Python 协程方面的东东（或者说是使用 aiohttp 这个异步库写个爬虫）。上周末，终于写出了一个差强人意的版本。就在这里写一写自己的心得。

## async / await

这在 Python 的协程中无足轻重。而我在二月份写的几篇博文却是专为这做的前序。一个月前我还对他们一窍不通，很不理解协程，通过这一个月的学习，我豁然开朗，大致了解了协程的前世今生。

`async` 与 `await`， 这是在 Python3.5 中引入的新语法，用于增强 Python 协程方面的支持。但是，现在关于他们的资料太少，尤其中文。感谢`ipfans <ipfanscn at gmail.com>`把 [PEP-0492](https://www.python.org/dev/peps/pep-0492/) 翻译成了中文。这里是超链接：[PEP 0492 Coroutines with async and await syntax 中文翻译](http://ipfans.github.io/2015/10/coroutines-with-async-and-await-syntax-chinese/)。若链接失效，可以尝试我的[备份](/2015/10/2015-10-31-coroutines-with-async-and-await-syntax-chinese/)

## 初识 asyncio

自己对它的认知实在过于浅薄，仅仅限于使用的初步阶段（这也是我使用的第一个异步库），若有错误欢迎指证。

在介绍 asyncio 之前，先谈论一下 Future，尽管这个 Future __不同于__ asyncio 中的 Future。[PEP-3184-- futures - execute computations asynchronously](https://www.python.org/dev/peps/pep-3148/)解释了它。下面是我的一部分翻译：

> PEP 3148 -- futures - 执行异步的计算  
>
> 摘要  
> 这个 PEP 提出了一种便于使用的线程和进程的可调用的评估包设计。  
>
> 动机  
>
> Python 当前拥有很强大的原语，来构建多线程和多进程的应用，但是简单的并行操作却需要大量工作。即：启动线程/进程，构建生产/消费队列，等待完成或者其他终止条件（如故障，超时）。当每个部件拥有自己的并行执行策略的时候，设计一个具有全局进程/线程限制的应用也是十分困难的。  
>
> 规范  
>
> 命名  
>
> 推荐的包称为 “futures”， 他会存在于一个新的 “concurrent” 顶级包中。futures 库存在于 “concurrent” 命名空间，原因有多个。第一，防止 “from __future__ import x” 新语法与 Python 中原有的语法产生冲突。此外，添加 “concurrent” 前体的名称表明了这个库被什么依赖 - 即并发（concurrency） - 这可以消除任何奇异，因为它已经注意到社会上并非所有人都熟悉 Java 的 Futures，或者排除它涉及到的 US stock market 中的 Future term。  
>
> 最后，我们为这个标准库开拓的新的命名空间 - 明显命名为 “concurrent -。我们希望要么在将来添加或者移除现有的，并发所依赖的相关库到到这里。一个典型的例子是 multiprocessing.Pool 工作，已及把他们的插件收录到这个模块中，这项工作覆盖了线程与进程。

asyncio 是 Python 3.4 版本引入的标准库，至少是 Python3.3 （需要手动安装）才可以使用。因为自 3.3 版本以来，`yield from`的引入使 Python 具备运行 asyncio 的基本条件。 3.5 版本中，新语法的引入使 Python 具备了原生协程，而不再是一种新的生成器类型。

这是[官方文档](https://docs.python.org/3/library/asyncio.html)的介绍：

> 这个模块提供了使用协程编写单线程并发代码，多路复用 I/O 访问，运行网络客户端和服务器，与其他相关的基础设施。  
> 下面是包内容的详细列表：  
> 一个可用于多种特定系统实现的可插拔的事件循环  
> 传输和协议的抽象（类似于 Twisted）-- Twisted 是一种运行在 Python 下的异步库  
> 对 TCP、UDP、SSL、子进程管道、延迟通信以及其他的具体支持（有些可能是系统相关的）  
> 一个模仿了 concurrent.futures 模块的 Fulture 类，但是适合事件循环使用  
> 在 [PEP 380](https://www.python.org/dev/peps/pep-0380) 基础上实现的协程和任务，用来以连续的样式写并行代码  
> 对 Future 和 协程 提供终止支持  
> 单线程中，在协程间使用同步原语来模拟那些线程模块  
> 一个用于传递工作的线程池的接口，为的是当你不得不使用一个库来阻塞 I/O 调用的时候使用  

[The New asyncio Module in Python 3.4: Event Loops]，这也是我对它的一部分翻译：

> 所述的 asyncio 模块包括以下主要组件：  
> 事件循环（Event loop）  
> 事件循环复用 I/O， 序列化事件处理，而且以一种策略模式工作，这种策略模式对自定义平台与框架是非常灵活的。例如，Tornado，Twisted，以及 Gevent 可以与 asyncio 一起工作，也可以构建在 asyncio 的顶层。事实上，事件循环受到了 Tornado 与 Twisted 的影响。此外， asyncio 为每个平台选择了最佳的 I/O 机制。Unix 和 LInux 利用 selectors 工作，而基于 Windows 的系统使用 IOCP 工作（I/O completion ports 的简称）。  
> Futures  
> 这是那些延迟生产者的抽象。例外也被任务是一个结果。`asyncio.futures.Future`类与 Python3.2 中引入的 Future 类似，后者在 PEP-3148 中介绍。即，`concurrent.futures.Future`类。但是，在这种情况下，Future 适用于协程，这是一个具有不同 API 的不同于 PEP-3148 所描述的类。该 asyncio 模块不适用现有的`concurrent.futures.Future`类，因为它被设计用于线程工作。该模块鼓励在协程中使用 `yield from` 锁住当前的任务来等带结果，从而避免阻塞你的应用。你的协程代码块；也就是说，你的协程被挂起，直到产生了结果，但是事件循环却没有被阻塞。若同一个 event loop 还有其他的任务序列，他们可能会运行。当协程产生结果的时候，暂停的协程会恢复，你可以编写同顺序执行一样的代码。你可以阅读代码，无需考虑 `yield from` 的存在。当你使用在函数中使用 `yield from` 返回一个 `yield from` 对象，你可以忘记 Future 执行的特殊的细节和它特定的 API。如果产生异常，比如你调用了函数，它没有返回 Future，却做了顺序执行，那么异常会被抛出。所以，编写异步代码同编写同步代码一样，除了添加 `yield from`。如果你有使用 Twisted 的经验，你会注意在 Twisted 中有同样的效果的装饰器 `@defer.inlineCallbacks`。  
> 协程（Coroutine）  
> 这些是生成器函数可以接受的值，他们必须被 `@coroutine`（`@asyncio.coroutine`）装饰。`@coroutine` 装饰器表明你使用 `yield from` 来传递每个 Future。装饰器可以确保你无论任何时候阅读代码，你都会知道这段代码使用了异步模式。在这些生成器函数中你必须使用`yield from`。若熟悉 C# 5.0，你会注意到`@coroutine`和`yield from`与 C# 中的关键字`async`和`await`有相同的作用。  
> Tasks  
> 每个 task 是一个被 Future 包裹的协程，随着 event loop 的运行而运行。`asyncio.Task`类是`asyncio.Future`的子类。你也可能猜到，tasks 也与`yield from`一起工作。
> 传输（Transports）  
> 他们相当于连接，比如套接字（sockets）与管道（pipes）。  
> 协议（Protocols）  
> 他们相当于应用，比如 HTTP server、SMTP 与 FTP。  

上面罗列了一大堆，其实就是想说明这几个东西：

* 在 asyncio 中，Future 是一种抽象
* Future 用于协程中
* 被`asyncio.coroutine`装饰器装饰的函数是一个 Future 类
* `asyncio.Task`是`asyncio.Future`的子类
* Future 随 event loop 的运行而运行

对于 Python3.5 中的新语法`async`与`await`，可以简单的认为是`asyncio.coroutine`与`yield from`的简化版。随着新语法到来的还有`async with EXPR as VAR`这个异步上下文管理器（取代`with (yield from EXPR) as VAR`)与`async for TARGET in ITER`（取代`for TARGET in (yield from ITER)`)这个异步迭代器。__不同于__`yield from`，`await`__只__适用于 CO_COROUTINE 标记的原生协程，即，使用`async def`语法定义的对象。

那么，怎么写出适用于 Python3.5+ 版本使用的协程呢？这是官方文档中的一个粒子：

```python
#Example of coroutine displaying "Hello World"
import asyncio

async def hello_world():
    print("Hello World!")

loop = asyncio.get_event_loop()
# Blocking call which returns when the hello_world() coroutine is done
loop.run_until_complete(hello_world())
loop.close()
```

这段代码干了下面几件事：

1. 创建了一个协程（`async def`），它是 Future 对象
2. 创建一个默认的事件循环（`loop = asyncio.get_event_loop()`）
3. 运行 event loop 的`loop.run_until_complete`方法
4. 关闭事件循环

`asyncio`的编程模型就是一个消息循环。从`asyncio`模块中直接获取一个 event loop 的引用，然后把需要执行的协程扔到 event loop 中执行，就实现了协程。

## asyncio 中的队列

我也不知道这部分放在哪里好唉，但是下面的需要它唉，没处放了~

开头强调一点：`async def`标记的始终为 _coroutine_，_coroutine_ 是可 awaitable 对象，`await`用于可 awaiteble 对象中。

假设已经对 Python 的队列由一定的了解（似乎不了解也没啥关系）。

asyncio 模块中，队列也分为三种：FifoQueue - first in first out，先入先出 -、PriorityQueue - 优先级 -、LifoQueue（Last in first out，先入后出）。他们__不是线程安全__的。下面介绍几个常用的方法：

* _coroutine_ get() -- 出队，阻塞操作：从队列中删除并返回一个数据。若队列是空的，那么该操作将会阻塞线程，直到有可用的数据出现。这个方法是一个 coroutine。  
* get_nowait() -- 出队，非阻塞操作：从队列中删除并返回一个数据。若数据存在，则被返回，否则引发一个 QueueEmpty 异常。  
* _coroutine_ join() -- 阻塞当前线程，直到队列中所有的数据都得到处理。数据入队后，未完成的任务数会增加。任何消费者调用 task_done() ，意味着有消费者取得并完成任务，未完成的任务数就会减少。当未完成的任务计数下降到零， join() 阻塞解除。该方法是一个 coroutine。  
* _coroutine_ put(item) -- 入队。若队列满，等待，直到有空位可用。该方法是一个 coroutine。  
* put_nowait(item) -- 非阻塞入队。若队列满，抛出 QueueFull 异常。  
* task_done() -- 意味着之前排队的某个任务完成了。由线程的消费者调用。每一个 get() 调用得到一个任务，接下来的 task_done() 调用告诉队列该任务已经处理完毕。若当前一个 join() 正在阻塞线程，它将在队列中的所有任务都处理完时恢复执行（即每一个由 put() 调用入队的任务都有一个对应的 task_done() 调用）。若该方法被调用的次数多于被放入队列中的任务个数， ValueError 异常会被抛出。  

接下来展示两个栗子：

```python
# Example of the queue how to work
import asyncio
from asyncio import Queue

async def work(q):
    while True:
        i = await q.get()
        try:
            print(i)
            print('q.qsize(): ', q.qsize())
        finally:
            q.task_done()

async def run():
    q = Queue()
    await asyncio.wait([q.put(i) for i in range(10)])
    tasks = [asyncio.ensure_future(work(q))]
    print('wait join')
    await q.join()
    print('end join')
    for task in tasks:
        task.cancel()

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(run())
    loop.close()
```

这是一个简单的消费者队列，主要是想说明具有 _coroutine_ 标识的函数的用法（`asyncio.wait()`见[asyncio 中的 Task](#function-introduction)）。注意到他们的相同之处了吗？

但是，协程内同时出现生产者与消费者，那么是什么样的情况呢？

```python
import asyncio
from asyncio import Queue

class Test:
    def __init__(self):
        self.que = Queue()
        self.pue = Queue()

    async def consumer(self):
        while True:
            try:
                print('consumer', await self.que.get())
            finally:
                try:
                    self.que.task_done()
                except ValueError:
                    if self.que.empty():
                        print("que empty")

    async def work(self):
        while True:
            try:
                value = await self.pue.get()
                print('producer', value)
                await self.que.put(value)
            finally:
                try:
                    self.pue.task_done()
                except ValueError:
                    if self.pue.empty():
                        print("pue empty")

    async def run(self):
        await asyncio.wait([self.pue.put(i) for i in range(10)])
        tasks = [asyncio.ensure_future(self.work())]
        tasks.append(asyncio.ensure_future(self.consumer()))
        print('p queue join')
        await self.pue.join()
        print('p queue is done & q queue join')
        await self.que.join()
        print('q queue is done')
        for task in tasks:
            task.cancel()

if __name__ == '__main__':
    print('----start----')
    case = Test()
    loop = asyncio.get_event_loop()
    loop.run_until_complete(case.run())
    print('----end----')
```

没啥好解释的，只是关注下`task.cancel()`的行为。`task.cancel()`被调用后，在下一个事件循环中会产生 CancelledError 异常（在 try block 中），`finally`为啥会出现异常处理，还记得`task_done()`的说明吗？

## 再识 asyncio

通过上面的简单示例，相信你已经学会了编写协程。在这里我将会以一个使用协程的爬虫演示我对 asyncio 的认识。最后的爬虫将会达到这样达到效果：多个 worker 完成对网页 ip 代理的抓取与分离，并传送到队列当中；另一些 worker 会对抓取的 ip 进行测试。当没有更多任务做的时候，worker 被暂停，但队列中存在需要工作的数据的时候，暂停的 worker 会立刻醒来并进行工作。程序会存在一个运行时间，时间到后程序立刻结束。

先摆出问题：

1. 使用 aiohttp 库的协程爬虫怎么写？
2. 如何在一个事件循环中同时抓取多个网页（达到多线程/多进程的效果）？
3. 协程之间怎么进行通信？

第一个问题很简单：

```python
import asyncio
import aiohttp
import re

URL = "http://www.ip84.com/gn-http/"

async def fetch_page(url):
    async with aiohttp.get(url) as response:
        try:
            assert response.status == 200
            print("OK!", response.url)
            return await response.text()
        except AssertionError:
            print('Error!', response.url, response.status)

async def filter_page(url):
    page = await fetch_page(url)
    if page:
        pattern = re.compile(r'<tr>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>.*?</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?</tr>', re.S)
        data = pattern.findall(page)
        for item in data:
            print(item)

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    for i in range(1, 21):
        loop.run_until_complete(filter_page(URL+repr(i)))
    loop.close()
```

对第二个问题，先展示下我的初始方案：

```python
# only main function is different
if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    for i in range(1, 21, 4):
        fs = asyncio.wait([filter_page(URL+repr(i+j)) for j in range(4)])
        loop.run_until_complete(fs)
    loop.close()
```

`asyncio.wait(fs, )`：_fs_ 是一个协程列表，由这个函数封装成一个 task（具体干了啥我也不明所以，但是下面的一个栗子或许能给点启发）

其实到这里，尽管方案简陋，但第二个问题算是解决了。协程之间的通信，就是考虑到需要测试 ip，获取/测试 ip 在不同的协程之间进行。那么，这不就是生产者/消费者问题嘛！

```python
import asyncio
from asyncio import Queue
import aiohttp
import time
import re

class Crawl:
    def __init__(self, url, test_url, *, number=10, max_tasks=5):
        self.url = url
        self.test_url = test_url
        self.number = number
        self.max_tasks = max_tasks
        self.url_queue = Queue()
        self.raw_proxy_queue = Queue()
        self.session = aiohttp.ClientSession() # tips: connection pool

    async def fetch_page(self, url):
        async with aiohttp.get(url) as response:
            try:
                assert response.status == 200
                print("OK!", response.url)
                return await response.text()
            except AssertionError:
                print('Error!', response.url, response.status)

    async def filter_page(self, url):
        page = await self.fetch_page(url)
        if page:
            pattern = re.compile(r'<tr>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>.*?</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?<td>(.*?)</td>.*?</tr>', re.S)
            data = pattern.findall(page)
            print(len(data))
            for raw in data:
                item = list(map(lambda word: word.lower(), raw))  
                await self.raw_proxy_queue.put({'ip': item[0], 'port': item[1], 'anonymous': item[2], 'protocol': item[3], 'speed': item[4], 'checking-time': item[5]})
            if not self.raw_proxy_queue.empty():
                print('OK! raw_proxy_queue size: ', self.raw_proxy_queue.qsize())

    async def verify_proxy(self, proxy):
        addr = proxy['protocol'] + '://' + proxy['ip'] +':'+proxy['port']
        conn = aiohttp.ProxyConnector(proxy=addr)
        try:
            session = aiohttp.ClientSession(connector=conn)
            with aiohttp.Timeout(10):
                start = time.time()
                async with session.get(self.test_url) as response: # close connection and response, otherwise will tip: Unclosed connection and Unclosed response
                    end = time.time()
                    try:
                        assert response.status == 200
                        print('Good proxy: {} {}s'.format(proxy['ip'], end-start))
                    except: #ProxyConnectionError, HttpProxyError and etc?
                        print('Bad proxy: {}, {}, {}s'.format(proxy['ip'], response.status, end-start))
        except:
            print('timeout {}, q size: {}'.format(proxy['speed'], self.raw_proxy_queue.qsize()))
        finally: # close session when timeout
            session.close()

    async def fetch_worker(self):
        while True:
            url = await self.url_queue.get()
            try:
                await self.filter_page(url)
            finally:
                self.url_queue.task_done()

    async def verify_worker(self):
        while True:
            raw_proxy = await self.raw_proxy_queue.get()
            if raw_proxy['protocol'] == 'https': # only http can be used
                continue
            try:
                await self.verify_proxy(raw_proxy)
            finally:
                try:
                    self.raw_proxy_queue.task_done()
                except:
                    pass

    async def run(self):
        await asyncio.wait([self.url_queue.put(self.url+repr(i+1)) for i in range(self.number)])
        fetch_tasks = [asyncio.ensure_future(self.fetch_worker()) for _ in range(self.max_tasks)]
        verify_tasks = [asyncio.ensure_future(self.verify_worker()) for _ in range(10*self.max_tasks)]
        tasks = fetch_tasks + verify_tasks
        await self.url_queue.join()
        self.session.close() # close session, otherwise shows error
        print("url_queue done")
        self.raw_proxy_queue.join()
        print("raw_proxy_queue done")
        await self.proxy_queue.join()
        for task in tasks:
            task.cancel()

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    crawler = Crawl('http://www.ip84.com/gn-http/', test_url='https://www.baidu.com')
    loop.run_until_complete(crawler.run())
    loop.close()
```

运行一下试试呗。

问题出现在这里：消费者的速度太快了。 *proxy_queue* 在生产者还在处理数据的时候，数据就被消费完了， join() 的阻塞便随之解除，可 coroutine 中仍有数据在处理 ... 那么怎么办呢？思考一个 task 的方法`wait_for()`。

至此结束~答案就不告诉你 -_-

希望你可以阅读下[An example of web spider using aiohttp]的源码，可以看懂的。

## <a id="function-introduction"></a>asyncio 中的 Task

[Task 函数](https://docs.python.org/3/library/asyncio-task.html#task-functions)  

注意：Task functions 允许在底层任务或协程显式设置 event loop。若 event loop 没有被提供，将会使用默认的 event loop。

_coroutine_ asyncio.wait(fs, *, loop=None, return_when='ALL_COMPLETED')

* 等待由 _fs_ - _futures_ 队列 - 给出的 Futures 和协程对象完成  
* _futures_ 队列必须不为空  
* 协程将被包裹进 Tasks  
* 返回两个 Future - (done, pending) - 的集合  
* _timeout_ 可被用来控制协程的最长运行时间，单位为秒，它可以是整数或者浮点数。若 _timeout_未指定或者为`None`，等待时间是无限长。  
* *return_when* 说明这个函数应该被返回。它必须是`concurrent.futures`模块中的某个常量：  

常量 | 描述
--- | ---
FIRST_COMPLETED | 当__任何一个__ future 结束或者取消的时候，函数会返回
FIRST_EXCEPTION | 当__任何一个__ future 因抛出异常而结束的时候，函数会返回。若没有 future 抛出异常，那么它等价于 ALL_COMPLETED
ALL_COMPLETED | 当__所有__ futures 结束或者取消的时候，函数会返回

* 这个函数是一个协程
* 用法：  
*   `done, pending = yield from asyncio.wait(fs)`
* 注意：  
* 这不会引发 TimeoutError！当超时出现在第二个集合中的时候，Futures 并不会结束。  

cancel()

* 取消 task。
* 它会安排一个 CancelledError 异常，在 event loop 的下一个事件周期中抛进 coroutine 中。那么 coroutine 就有机会清理，甚至使用 try/except/finally 来拒绝请求。
* 不同于 Future.cancel()，它并不保证该任务将被取消：可能捕获到异常并处理它，延迟任务的取消或者完全阻止任务取消。这个 task 可能返回一个值，或引发一个不同的异常。
* 该方法被调用后会立即执行， cancelled() 并不会返回 True（除非任务已经被取消）。

_coroutine_ asyncio.wait_for(fut, timeout, *, loop=None)

* 等待一个 Future 或者 coroutine 对象在指定的时间内完成。若 _timeout_ 为 `None`，阻塞线程，直到 future 完成。  
* coroutine 将被包裹入 task 中。  
* 返回 Future 或者 coroutine 的返回结果。若超时，它将会取消 task，并且引发 asyncio.TimeoutError。为了避免 task 被取消，把它包裹进`shield()`。  
* 若 wait 被取消，那么 future fut 也将被取消。  
* 该函数是一个协程，用法：
*   `result = yield from asyncio.wait_for(fut, 60.0)`

shield() 也是 task 中的一个方法，但是我没有使用过~

## 关于爬虫的一些思考

从哪里看到，协程比线程廉价多了，的确。尽管同时开启 1000 个协程发起 HTTP request，I5-4200U 这个CPU使用量不过 5% 左右。但是进行正则匹配，仅 5 个协程却占用了大量的 CPU。正则匹配属于计算密集型任务， HTTP 请求属于 I/O 密集型任务。我想：把 I/O 密集型任务与计算密集型任务分离是能提高爬虫的效率的。据说， redis 适合做缓存队列~ 那么，下一个爬虫可以这么干：

```md
        url  http request <---- Redis URL queue
   coroutine
        http response     ----> Redis response queue ----> multiprocessing filter ----> MongoDB
```

前几天开源社区中某个即将就职的学长说：公司有一套很有意思的反爬虫系统，在不影响正常运营的情况下是不会管爬虫的，但过度爬取却会被干掉。一个不过分的爬虫也可能会在某次系统升级后挂掉。至于使用代理，全球代理服务器就那么多，而有个公开的代理服务器黑名单系统，这些 ip 肯定是重点关注的对象。

那么，怎么能尽量模仿人的行为呢？

## 参考资料

[18.5. asyncio – Asynchronous I/O, event loop, coroutines and tasks](https://docs.python.org/3/library/asyncio.html)  
[PEP-3184-- futures - execute computations asynchronously](https://www.python.org/dev/peps/pep-3148/)  
[The New asyncio Module in Python 3.4: Event Loops](http://www.drdobbs.com/open-source/the-new-asyncio-module-in-python-34-even/240168401)  
[Getting values from functions that run as asyncio tasks](http://stackoverflow.com/questions/32456881/getting-values-from-functions-that-run-as-asyncio-tasks)  
[廖雪峰 异步I/O](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/00143208573480558080fa77514407cb23834c78c6c7309000)  
[<译> A Web Crawler With asyncio Coroutines](http://damnever.github.io/2015/10/12/a-web-crawler-with-asyncio-coroutines/)  
[An example of web spider using aiohttp]

[An example of web spider using aiohttp]: https://github.com/niklak/aiohttp_spider
