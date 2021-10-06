---
title:  PEP 0492 Coroutines with async and await syntax 中文翻译
date:   2015-10-31 12:48:15
tags:   Python async PEPs
---

原文地址: [PEP-0492](https://www.python.org/dev/peps/pep-0492/)

|PEP|492|
|:---|:---|
|标题|协程与async/await语法|
|作者|Yury Selivanov `<yury at magic.io>`|
|翻译|ipfans `<ipfanscn at gmail.com>`|
|状态|最终稿|
|Python版本|3.5|
|翻译最后更新|2015-11-03|

目录

[摘要](#Abstract)
[API设计和实现的备注](#API-Design-and-Implementation-Note)
[基本原理和目标](#Rationale-and-Goals)
[语法规范](#Specification)

   * [新协程声明语法](#New-Coroutine-Declaration-Syntax)
   * [types.coroutine()](#Type-Coroutine)
   * [Await表达式](#Await-Expression)
     * [新的操作符优先级列表](#Updated-operator-precedence-table)
     * [await表达式示例](#Examples-of-await-expressions)
   * [异步上下文管理与`async with`](#Asynchronous-Context-Managers)
      * [新语法](#Async-With-New-Syntax)
      * [例子](#Async-With-Example)
   * [异步迭代器与`async for`](#Asynchronous-Iterators)
      * [新语法](#Async-For-New-Syntax)
      * [例子1](#Async-For-Example-1)
      * [例子2](#Async-For-Example-2)
      * [为什么使用`StopAsyncIteration`](#Why-StopAsyncIteration)
   * [协程对象](#Coroutine-objects)
     * [与生成器的不同之处](#Differences-from-generators)
     * [协程对象方法](#Coroutine-object-methods)
   * [调试特性](#Debugging-Features)
   * [新的标准库函数](#New-Standard-Library-Functions)
   * [新的抽象基类](#New-Abstract-Base-Classes)

[专用术语表](#Glossary)   
[函数与方法列表](#List-of-functions-and-methods)
[移植计划](#Transition-Plan)

   * [向后兼容性](#Backwards-Compatibility)
     * [asyncio](#asyncio)
     * [asyncio移植策略](#asyncio-migration-strategy)
     * [CPython代码中的`async/await`](#async-await-in-CPython-code-base)
   * [语法更新](#Grammar-Updates)
   * [失效计划](#Deprecation-Plans)

设计思路(暂时不考虑翻译)
[性能](#Performance)

   * [总体影响](#Overall-Impact)
   * [编译器修改](#Tokenizer-modifications)
   * [`async/await`](#async-await)

[实现参考](#Reference-Implementation)

   * [上层修改和新协议列表](#List-of-high-level-changes-and-new-protocols)
   * [可以工作的实例](#working-example)

参考

致谢

版权信息

## <a name="Abstract"></a>摘要

不断增长的网络连通性需求带动了对响应性、伸缩性代码的需求。这个PEP的目标在于回答如何更简单的、Pythinic的实现显式的异步/并发的Python代码。

我们把协程概念独立出来，并为其使用新的语法。最终目标是建立一个通用、易学的Python异步编程模型，并尽量与同步编程的风格保持一致。

这个PEP假设异步任务被一个事件循环器（类似于标准库里的 asyncio.events.AbstractEventLoop）管理和调度。不过，我们并不会依赖某个事件循环器的具体实现方法，从本质上说只与此相关：使用yield作为给调度器的信号，表示协程将会挂起直到一个异步事件（如IO）完成。

我们相信这些改变将会使Python在这个异步编程快速增长的领域能够保持一定的竞争性，就像许多其它编程语言已经、将要进行的改变那样。

## <a name="API-Design-and-Implementation-Note"></a>API设计和实现的备注

根据Python 3.5 Beta期间的反馈，我们进行了重新设计：明确的把协程从生成器里独立出来---原生协程现在拥有了自己完整的独立类型，而不再是一种新的生成器类型。

这个改变主要是为了解决在Tornado Web服务中里集成协程时出现的一些问题。

## <a name="Rationale-and-Goals"></a>基本原理和目标

现在版本的Python支持使用生成器实现协程功能([PEP-342](https://www.python.org/dev/peps/pep-0342))，后面通过[PEP-380](https://www.python.org/dev/peps/pep-0380)引入了`yield from`语法进行了增强。但是这样仍有一些缺点：

   * 协程与常规的生成器在相同语法时用以混淆，尤其是对心开发者而言。
   * 一个函数是否是协程需要通过是否主体代码中使用了`yield`或者`yield from`语句进行检测，这样在重构代码中添加、去除过程中容易出现不明显的错误
   * 异步调用的支持被`yield`支持的语法先定了，导致我们无法使用更多的语法特性，比如`with`和`for`语句。

这个提议的目的是将协程作为原生Python语言特性，并且将他们与生成器明确的区分开。它避免了生成器/协程中间的混淆请困高，方便编写出不依赖于特定库的协程代码。这个也方便linter和IDE能够实现更好的进行静态代码分析和重构。

原生协程和相关的新语法特性使得可以在异步框架下可以定义一个上下文管理器和迭代协议。在这个提议后续中，新的`async with`语法让Python程序在进入和离开运行上下文时实现异步调用，新的`async for`语法可以在迭代器中实现异步调用。

## <a name="Specification"></a>语法规范

这个提议介绍了新的语法用于增强Python中的协程支持。

这个语法规范假设你已经了解Python现有协程实现方法([PEP-342](https://www.python.org/dev/peps/pep-0342)和[PEP-380](https://www.python.org/dev/peps/pep-0380))。这次语法改变的动机来自于asyncio框架([PEP-3156](https://www.python.org/dev/peps/pep-3156))和`Cofunctions`提议([PEP-3152](https://www.python.org/dev/peps/pep-3152)，现在此提议已被废弃)。

从本文档中，我们使用`原生协程`代指新语法生命的函数，`基于生成器的协程`用于表示那些基于生成器语法实现的协程。`协程`则表示两个地方都可以使用的内容。


### <a name="New-Coroutine-Declaration-Syntax"></a>新协程声明语法

下面的新语法用于声明原生协程：

```python
async def read_data(db):
    pass
```

协程的主要属性包括：

   * `async def`函数始终为协程，即使它不包含`await`表达式。
   * 如果在`async`函数中使用`yield`或者`yield from`表达式会产生`SyntaxError`错误。
   * 在内部，引入了两个新的代码对象标记：
      * `CO_COROUTINE`用于标记原生协程（和新语法一起定义）
      * `CO_ITERABLE_COROUTINE`用于标记基于生成器的协程，兼容原生协程。(通过`types.coroutine()`函数设置)
   * 常规生成器在调用时会返回一个`genertor`对象，同理，协程在调用时会返回一个`coroutine`对象。
   * 协程不再抛出`StopIteration`异常，而是替代为`RuntimeError`。常规生成器实现类似的行为需要进行引入`__future__`([PEP-3156](https://www.python.org/dev/peps/pep-3156))
   * 当协程进行垃圾回收时，一个从未被`await`的协程会抛出`RuntimeWarning`异常。(参考[调试特性](#Debugging-Features))
   * 更多内容请参考[协程对象](#Coroutine-objects)一节。

### <a name="Type-Coroutine"></a>types.coroutine()

在`types`模块中新添加了一个函数`coroutine(fn)`用于`asyncio`中基于生成器的协程与本PEP中引入的原生携协程互通。

```python
@types.coroutine
def process_data(db):
    data = yield from read_data(db)
    ...
```

这个函数将生成器函数对象设置`CO_ITERABLE_COROUTINE`标记，将返回对象变为`coroutine`对象。

如果`fn`不是一个生成器函数，那么它会对其进行封装。如果它返回一个生成器，那么它会封装一个`awaitable`代理对象(参考下面`awaitable`对象的定义)。

注意：`CO_COROUTINE`标记不能通过`types.coroutine()`进行设置，这就可以将新语法定义的原生协程与基于生成器的协程进行区分。

types模块添加了一个新函数coroutine(fn)，使用它，“生成器实现的协程”和“原生协程”之间可以进行互操作。

### <a name="Await-Expression"></a>Await表达式

下面新的`await`表达式用于获取协程执行结果：

```python
async def read_data(db):
    data = await db.fetch('SELECT ...')
    ...
```

`await`与`yield from`相似，挂起`read_data`协程的执行直到`db.fetch`这个`awaitable`对象完成并返回结果数据。

它复用了`yield from`的实现，并且添加了额外的验证参数。`await`只接受以下之一的`awaitable`对象：

* 一个原生协程函数返回的原生协程对象。
* 一个使用`types.coroutine()`修饰器的函数返回的基于生成器的协程对象。
* 一个包含返回迭代器的`__await__`方法的对象。
   任意一个`yield from`链都会以一个`yield`结束，这是`Future`实现的基本机制。因此，协程在内部中是一种特殊的生成器。每个`await`最终会被`await`调用链条上的某个`yield`语句挂起（参考[PEP-3156](https://www.python.org/dev/peps/pep-3156)中的进一步解释）。
   为了启用协程的这一特点，一个新的魔术方法`__await__`被添加进来。在`asyncio`中，对于对象在await语句启用`Future`对象只需要添加`__await__ = __iter__`这行到`asyncio.Future`类中。
   在本PEP中，带有`__await__`方法的对象也叫做`Future-like`对象。
   同样的，请注意到`__aiter__`方法（下面会定义）不能用于这种目的。它是不同的协议，有点类似于用`__iter__`替代普通调用方法的`__call___`。
   如果`__await__`返回非迭代器类型数据，会产生一个`TypeError`.
* CPython C API中使用`tp_as_async.am_await`定义的函数，并且返回一个迭代器（类似`__await__`方法）。

#### <a name="Updated-operator-precedence-table"></a>新的操作符优先级列表

关键词`await`与`yield`和`yield form`操作符的区别是`await`表达式大部分情况下不需要括号包裹。

同样的，`yield from`允许允许任意表达式做其参数，包含表达式如`yield a()+b()`，这样通常处理作为`yield from (a()+b())`，这个通常会造成Bug。通常情况下任意算数操作的结果都不会是`awaitable`对象。为了避免这种情况，我们将await的优先级调整为低于`[], ()和.`，但是高于` ** `操作符。

|操作符|描述|
|:---|:----|
|yield x , yield from x |Yield表达式|
|lambda|Lambda表达式|
|if -- else|条件表达式|
|or |布尔或|
|and|布尔与|
|not x|布尔非|
|in , not in , is , is not , < , <= , > , >= , != , ==|比较，包含成员测试和类型测试|
|\||字节或|
|^|字节异或|
|&|字节与|
|<< , >>|移位|
|+ , -  |加和减|
|* , @ , / , // , %|乘，矩阵乘法，除，取余|
|+x , -x , ~x|正数, 复数, 取反|
|** |平方|
|await x    |Await表达式|
|x[index] , x[index:index] , x(arguments...) , x.attribute|子集，切片，调用，属性|
|(expressions...) , [expressions...] , {key: value...} , {expressions...}|类型显示|

#### <a name="Examples-of-await-expressions"></a>await表达式示例

有效的语法例子:

|表达式|会被处理为|
|:---|:---|
|if await fut: pass |if (await fut): pass|
|if await fut + 1: pass |if (await fut) + 1: pass|
|pair = await fut, 'spam'|  pair = (await fut), 'spam'|
|with await fut, open(): pass|with (await fut), open(): pass|
|await foo()['spam'].baz()()    |await ( foo()['spam'].baz()() )|
|return await coro()|   return ( await coro() )|
|res = await coro() ** 2    |res = (await coro()) ** 2|
|func(a1=await coro(), a2=0)|   func(a1=(await coro()), a2=0)|
|await foo() + await bar()  |(await foo()) + (await bar())|
|-await foo()|  -(await foo())|

错误的语法例子:

|表达式|应写作|
|:---|:---|
|await await coro()|    await (await coro())|
|await -coro()| await (-coro())|

### <a name="Asynchronous-Context-Managers"></a>异步上下文管理与`async with`

一个异步上下文管理器是用于在`enter`和`exit`方法中管理暂停执行的上下文管理器。

为此，我们设置了新的异步上下文管理器。添加了两个魔术方法： `__aenter__`和`__aexit__`。这两个方法都返回`awaitable`对象。

异步上下文管理器例子如下：

```python
class AsyncContextManager:
    async def __aenter__(self):
        await log('entering context')

    async def __aexit__(self, exc_type, exc, tb):
        await log('exiting context')
```

#### <a name="Async-With-New-Syntax"></a>新语法

一个新的异步上下文管理语法被接受：

```python
async with EXPR as VAR:
    BLOCK
```

语义上等同于：

```python
mgr = (EXPR)
aexit = type(mgr).__aexit__
aenter = type(mgr).__aenter__(mgr)
exc = True

VAR = await aenter
try:
    BLOCK
except:
    if not await aexit(mgr, *sys.exc_info()):
        raise
else:
    await aexit(mgr, None, None, None)
```

和普通的`with`语句一样，可以在单个`async with`语句里指定多个上下文管理器。

在使用`async with`时，如果上下文管理器没有`__aenter__`和`__aexit__`方法，则会引发错误。在`async def`函数之外使用`async with`则会引发`SyntaxError`异常。

#### <a name="Async-With-Example"></a>例子

通过异步上下文管理器更容易实现协程对数据库事务的正确管理：

```python
async def commit(session, data):
    ...

    async with session.transaction():
        ...
        await session.update(data)
        ...
```

代码看起来也更加简单：

```python
async with lock:
    ...
```

而不是

```python
with (yield from lock):
    ...
```

### <a name="Asynchronous-Iterators"></a>异步迭代器与`async for`

一个异步迭代器能够在它的迭代实现里调用异步代码，也可以在它的`__next__`方法里调用异步代码。为了支持异步迭代，需要：

1. 一个对象必须实现`__aiter__`方法（或者，使用CPython C API的tp\_as\_async.am\_aiter定义），返回一个异步迭代器对象中的`awaitable``结果。
2. 一个异步迭代器必须实现`__anext__`方法（或者，使用CPython C API的tp\_as\_async.am\_anext定义）返回一个`awaitable`。
3. 停止迭代器的`__anext__`必须抛出一个`StopAsyncIteration`异常。

一个异步迭代的例子：

```python
class AsyncIterable:
    async def __aiter__(self):
        return self

    async def __anext__(self):
        data = await self.fetch_data()
        if data:
            return data
        else:
            raise StopAsyncIteration

    async def fetch_data(self):
        ...
```

#### <a name="Async-For-New-Syntax"></a>新语法

一种新的异步迭代方案被采纳：

```python
async for TARGET in ITER:
    BLOCK
else:
    BLOCK2
```

语义上等同于：

```python
iter = (ITER)
iter = await type(iter).__aiter__(iter)
running = True
while running:
    try:
        TARGET = await type(iter).__anext__(iter)
    except StopAsyncIteration:
        running = False
    else:
        BLOCK
else:
    BLOCK2
```

如果对一个普通的不含有`__aiter__`方法的迭代器使用`async for`，会引发`TypeError`异常。如果在`async def`函数外使用`async for`会已发`SyntaxError`异常。

和普通的`for`语法一样，`async for`有可选的`else`分支。

#### <a name="Async-For-Example-1"></a>例子1

通过异步迭代器，就可以实现通过迭代实现异步缓冲数据：

```python
async for data in cursor:
    ...
```

当`cursor`是一个异步迭代器时，就可以在N次迭代后从数据库中预取N行数据。

下面的代码演示了新的异步迭代协议：

```python
class Cursor:
    def __init__(self):
        self.buffer = collections.deque()

    async def _prefetch(self):
        ...

    async def __aiter__(self):
        return self

    async def __anext__(self):
        if not self.buffer:
            self.buffer = await self._prefetch()
            if not self.buffer:
                raise StopAsyncIteration
        return self.buffer.popleft()
```

那么这个`Cursor`类可以按照下面的方式使用：

```python
async for row in Cursor():
    print(row)
```

这个等同于下面的代码：

```python
i = await Cursor().__aiter__()
while True:
    try:
        row = await i.__anext__()
    except StopAsyncIteration:
        break
    else:
        print(row)
```

#### <a name="Async-For-Example-2"></a>例子2

下面的工具类用于将普通的迭代转换为异步。这个并没有什么实际的作用，这个代码只是用于演示普通迭代与异步迭代之间的关系。

```python
class AsyncIteratorWrapper:
    def __init__(self, obj):
        self._it = iter(obj)

    async def __aiter__(self):
        return self

    async def __anext__(self):
        try:
            value = next(self._it)
        except StopIteration:
            raise StopAsyncIteration
        return value

async for letter in AsyncIteratorWrapper("abc"):
    print(letter)
```

#### <a name="Why-StopAsyncIteration"></a>为什么使用`StopAsyncIteration`

协程在内部实现中依旧是依赖于迭代器的。因此，在[PEP-479](https://www.python.org/dev/peps/pep-0479)生效之前，下面两者并没有区别：

```python
def g1():
    yield from fut
    return 'spam'
and

def g2():
    yield from fut
    raise StopIteration('spam')
```

但是在PEP 479接受并且默认对协程开启时，下面的例子中的`StopIteration`会被封装成`RuntimeError`。

```python
async def a1():
    await fut
    raise StopIteration('spam')
```

所以，想通知外部代码迭代已经结束，抛出一个`StopIteration`异常的是不行的。因此，一个新的内置异常类`StopAsyncIteration`被引入进来了。

另外，根据PEP 479，所有协程中抛出的`StopIteration`异常都会被封装成`RuntimeError`。

### <a name="Coroutine-objects"></a>协程对象

#### <a name="Differences-from-generators"></a>与生成器的不同之处

这节进适用于`CO_COROUTINE`标记的原生协程，即，使用`async def`语法定义的对象。

__现有的asyncio库中的*基于生成器的协程*的行为未做变更。__

为了将协程与生成器区别开来，定义了下面的概念：

1. 原生协程对象不实现`__iter__`和`__next__`方法。因此，他们不能够通过`iter()，list()，tuple()`和其他一些内置函数进行迭代。他们也不能用于`for...in`循环。
   在原生协程中尝试使用`__iter__`或者`__next`会触发`TypeError`异常。
2. 未被装饰的生成器不能够`yield from`一个原生协程：这样会引发`TypeError`。
3. 基于生成器的协程(asyncio代码必须使用`@asyncio.coroutine`)可以`yield from`一个原生协程。
4. 对原生协程对象和原生协程函数调用`inspect.isgenerator()`和`inspect.isgeneratorfunction()`会返回False。

#### <a name="Coroutine-object-methods"></a>协程对象方法

协程内部基于生成器，因此他们同享实现过程。类似于生成器对象，协程包含`throw()`，`send()`和`close()`方法。`StopIteration`和`GeneratorExit`在协程中扮演者同样的角色（尽管PEP 479默认对协程开启了）。参考[PEP-342](https://www.python.org/dev/peps/pep-0342), [PEP-380](https://www.python.org/dev/peps/pep-0380)和Python文档了解更多细节。

协程的`throw()`和`send()`方法可以用于将返回值和抛出异常推送到类似于`Future`的对象中。

### <a name="Debugging-Features"></a>调试特性

一个初学者普遍会犯的错误是忘记在协程中使用`yield from`。

```python
@asyncio.coroutine
def useful():
    asyncio.sleep(1) # this will do noting without 'yield from'
```

为了调试这类错误，asycio提供了一种特殊的调试模式：装饰器`@coroutine`封装所有的函数成一个特殊对象，这个对象的析构函数中记录警告。当封装的生成器垃圾回收时，会产生详细的记录信息，包括具体定义修饰函数、回收时的栈信息等等。封装对象同样提供一个`__repr__`函数用于输出关于生成器的详细信息。

唯一的问题是如何启用这些调试功能。这些调试工具在生产模式中什么都不做，`@coroutine`修饰符在系统变量`PYTHONASYNCIODEBUG`设置后才会提供调试功能。这种方式可以让asyncio程序使用asyncio自己的函数分析。`EventLoop.set_debug`是另外一个调试工具，他不会影响`@coroutine`修饰符行为。

根据本提议，协程是原生的与生成器不同的概念。当抛出`RuntimeWarning`异常的协程是从来没有被`awaited`过的。因此添加了两条新的函数到sys模块：`set_coroutine_wrapper`和`get_coroutine_wrapper`。这个用于开启asyncio或者其他框架中的高级调试(比如显示协程创建的位置和垃圾回收时的栈信息)。

### <a name="New-Standard-Library-Functions"></a>新的标准库函数

* `types.coroutine(gen)`。参考[types.coroutine()](#Type-Coroutine)节中的内容。
* `inspect.iscoroutine(obj)`当obj是原生协程时返回True。
* `inspect.iscoroutinefunction(obj)`当obj是原生协程函数时返回为True。
* `inspect.isawaitable(obj)`当obj是`awaitable`时返回为True。
* `inspect.getcoroutinestate(coro)`返回原生协程对象的当前状态（是`inspect.getfgeneratorstate(gen)`的镜像）。
* `inspect.getcoroutinelocals(coro)`返回原生协程对象的局部变量的映射（是`inspect.getgeneratorlocals(gen)`的镜像）。
* `sys.set_coroutine_wrapper(wrapper)`允许拦截原生协程对象的创建。`wrapper`必须是一个接受一个参数`callable`（一个协程对象），或者是`None`。`None`会重置`wrapper`。当调用第二次时，新的`wrapper`会替代之前的封装。这个函数是线程专有的。参考[调度调试](#Debugging-Features)了解更多细节。
* `sys.get_coroutine_wrapper()`返回当前的封装对象。如果封装未设置会返回None。这个函数是线程专有的。参考[调度调试](#Debugging-Features)了解更多细节。

### <a name="New-Abstract-Base-Classes"></a>新的抽象基类

为了允许更好的与现有的框架（比如Tornado）和编译器（比如Cython）整合，我们添加了两个新的抽象基类(ABC)
`collections.abc.Awaitable`是`Future-like`类的抽象基类，它实现了`__await__`方法。
`collections.abc.Coroutine`是协程对象的抽象基类，它实现了 `send(value)`，`throw(type, exc, tb)`，`close()`和`__await__()`方法。

值得注意的是，带有`CO_ITERABLE_COROUTINE`标记的基于生成器的协程并没有实现`__await__`方法，因此他不是`collections.abc.Coroutine`和`collections.abc.Awaitable`抽象类的实例：

```python
@types.coroutine
def gencoro():
    yield

assert not isinstance(gencoro(), collections.abc.Coroutine)

# 然而:
assert inspect.isawaitable(gencoro())
```

为了方便对异步迭代的调试，添加了另外两个抽象基类：

* `collections.abc.AsyncIterable` -- 用于测试`__aiter__`方法
* `collections.abc.AsyncIterator` -- 用于测试`__aiter__`和`__anext__`方法。

## <a name="Glossary"></a>专用术语表

## <a name="List-of-functions-and-methods"></a>函数与方法列表

## <a name="Transition-Plan"></a>移植计划

### <a name="Backwards-Compatibility"></a>向后兼容性

#### <a name="asyncio"></a>asyncio

#### <a name="asyncio-migration-strategy"></a>asyncio移植策略

#### <a name="async-await-in-CPython-code-base"></a>CPython代码中的`async/await`

### <a name="Grammar-Updates"></a>语法更新

### <a name="Deprecation-Plans"></a>失效计划

## <a name="Performance"></a>性能

### <a name="Overall-Impact"></a>总体影响

这个提议并不会造成性能影响。这是Python官方性能测试结果：

```log
python perf.py -r -b default ../cpython/python.exe ../cpython-aw/python.exe

[skipped]

Report on Darwin ysmac 14.3.0 Darwin Kernel Version 14.3.0:
Mon Mar 23 11:59:05 PDT 2015; root:xnu-2782.20.48~5/RELEASE_X86_64
x86_64 i386

Total CPU cores: 8

### etree_iterparse ###
Min: 0.365359 -> 0.349168: 1.05x faster
Avg: 0.396924 -> 0.379735: 1.05x faster
Significant (t=9.71)
Stddev: 0.01225 -> 0.01277: 1.0423x larger

The following not significant results are hidden, use -v to show them:
django_v2, 2to3, etree_generate, etree_parse, etree_process, fastpickle,
fastunpickle, json_dump_v2, json_load, nbody, regex_v8, tornado_http.
```

### <a name="Tokenizer-modifications"></a>编译器修改

修改后的编译器处理Python文件没有明显的性能下降：处理12MB大小的文件（`Lib/test/test_binop.py`重复1000次）消耗时间相同。

### <a name="async-await"></a>`async/await`

下面的小测试用于检测『async』函数和生成器的性能差异：

```python
import sys
import time

def binary(n):
    if n <= 0:
        return 1
    l = yield from binary(n - 1)
    r = yield from binary(n - 1)
    return l + 1 + r

async def abinary(n):
    if n <= 0:
        return 1
    l = await abinary(n - 1)
    r = await abinary(n - 1)
    return l + 1 + r

def timeit(func, depth, repeat):
    t0 = time.time()
    for _ in range(repeat):
        o = func(depth)
        try:
            while True:
                o.send(None)
        except StopIteration:
            pass
    t1 = time.time()
    print('{}({}) * {}: total {:.3f}s'.format(
        func.__name__, depth, repeat, t1-t0))
```

结果显示并没有明显的性能差异：

```log
binary(19) * 30: total 53.321s
abinary(19) * 30: total 55.073s

binary(19) * 30: total 53.361s
abinary(19) * 30: total 51.360s

binary(19) * 30: total 49.438s
abinary(19) * 30: total 51.047s
```

注意：19层意味着1,048,575调用。

## <a name="Reference-Implementation"></a>实现参考

实现参考可以在[这里](https://github.com/1st1/cpython/tree/await)找到。

### <a name="List-of-high-level-changes-and-new-protocols"></a>上层修改和新协议列表

1. 新的协程定义语法：`async def`和新的`await`关键字。
2. `Future-like`对象提供新的`__await__`方法和新的`PyTypeObject`的`tp_as_async.am_await`。
3. 新的异步上下文管理器语法： `async with`，协议提供了`__aenter__`和`__aexit__`方法。
4. 新的异步迭代语法：`async for`，协议提供了`__aiter`、`__aexit`和新的内置异常`StopAsyncIteration`。`PyTypeObject`提供了新的`tp_as_async.am_aiter`和`tp_as_async.am_anext`。
5. 新的AST节点：`AsyncFunctionDef`，`AsyncFor`，`AsyncWith`和`Await`。
6. 新函数 `sys.set_coroutine_wrapper(callback)`，`sys.get_coroutine_wrapper()`，`types.coroutine(gen)`，`inspect.iscoroutinefunction(func)`，`inspect.iscoroutine(obj)`，`inspect.isawaitable(obj)`，`inspect.getcoroutinestate(coro)`和`inspect.getcoroutinelocals(coro)`。
7. 新的代码对象标记`CO_COROUTINE`和`CO_ITERABLE_COROUTINE`。
8. 新的抽象基类`collections.abc.Awaitable`，`collections.abc.Coroutine`，`collections.abc.AsyncIterable`和`collections.abc.AsyncIterator`。
9. C API变更：新的`PyCoro_Type`（将Python作为 `types.CoroutineType`输出）和`PyCoroObject`。`PyCoro_CheckExact(*o)`用于检测o是否为原生协程。

虽然变化和新内容列表并不短，但是重要的是理解：大部分用户不会直接使用这些特性。他的目的是在于框架和库能够使用这些为用户提供便捷的使用和明确的API用于`async def`，`await`，`async for`和`async with`语法。

### <a name="working-example"></a>可以工作的实例

本PEP提出的所有概念都[已经实现](https://github.com/1st1/cpython/tree/await)，并且可以被测试。

```python
import asyncio

async def echo_server():
    print('Serving on localhost:8000')
    await asyncio.start_server(handle_connection,
                               'localhost', 8000)

async def handle_connection(reader, writer):
    print('New connection...')

    while True:
        data = await reader.read(8192)

        if not data:
            break

        print('Sending {:.10}... back'.format(repr(data)))
        writer.write(data)

loop = asyncio.get_event_loop()
loop.run_until_complete(echo_server())
try:
    loop.run_forever()
finally:
    loop.close()
```

{% raw %}
<style>
li {
  word-wrap: break-word;
  word-break: break-all;
}
</style>
{% endraw %}
