---
layout: post
date: 2016-02-20
title: Python之yield
categories: style
tags: ['language', 'python']
---

`yield`是 Python 关键字之一，用于创建一个生成器（Generator）。生成器是迭代器（Iterator）的一种。这里将会记录一些关于生成器与迭代器的东西。

## 迭代器

迭代器是访问集合内元素的一种方式。迭代器对象从集合的第一个元素开始访问，直到所有的元素都被访问一遍后结束。

迭代器不能回退，只能往前进行迭代。这并不是什么很大的缺点，因为人们几乎不需要在迭代途中进行回退操作。

迭代器也不是线程安全的，在多线程环境中对可变集合使用迭代器是一个危险的操作。但如果小心谨慎，或者干脆贯彻函数式思想坚持使用不可变的集合，那这也不是什么大问题。

对于原生支持随机访问的数据结构（如 tuple、list），迭代器和经典for循环的索引访问相比并无优势，反而丢失了索引值（可以使用内建函数`enumerate()`找回这个索引值，这是后话）。但对于无法随机访问的数据结构（比如 set）而言，迭代器是唯一的访问元素的方式。

迭代器的另一个优点就是它不要求你事先准备好整个迭代过程中所有的元素。迭代器仅仅在迭代至某个元素时才计算该元素，而在这之前或之后，元素可以不存在或者被销毁。这个特点使得它特别适合用于遍历一些巨大的或是无限的集合，比如几个 *GB* 的文件，或是斐波那契数列等等。这个特点被称为延迟计算或惰性求值(Lazy evaluation)。

迭代器更大的功劳是提供了一个统一的访问集合的接口。只要是实现了`__iter__()`方法的对象，就可以使用迭代器进行访问。

迭代器的创建与使用很简单，如下：

```python
>>> lst = range(3)
>>> lst.__iter__()
<range_iterator object at 0x7f020673d660>
>>> it = iter(lst) # iter(lst) 等价于 lst.__iter__()
>>> it
<range_iterator object at 0x7f020e4f24e0>
>>> next(it)
0
>>> it.__next__() # 等价于 next(it)
1
>>> next(it)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
>>> for i in range(3): # 上述代码可以这么写
...     i
...
0
1
2
>>> list(range(3)) # 一个特殊的栗子
[0, 1, 2]
```

实际上，自定义容器对象只需要拥有`__iter__()`方法，将迭代操作代理到容器内部对象上去，那么当将该对象当作参数被工厂函数`iter()`调用后，`iter()`的返回值就是个可迭代对象。使用`next()`访问迭代器的下一个元素。可以这么理解：`iter(obj)`调用了对象 obj 的`obj.__iter__()`方法，`next(obj)`调用了`obj.__next__()`方法。语法糖`for`干了这么几件事：自动调用工厂函数`iter()`获得迭代器，自动调用`next()`获取元素，还完成检查 *StopIteration* 异常的工作。

tips：*itertools* 模块实现了许多迭代功能，当碰到看上去有些复杂的迭代问题时，不妨先去看看 *itertools* 模块。

## 生成器

生成器是迭代器，同时也并不仅仅是迭代器，不过迭代器之外的用途实在是不多，所以我们可以大声地说：生成器提供了非常方便的自定义迭代器的途径。

* *yield* 可以使函数**返回一个生成器**。这种方法被对象中的`__iter__`与`__next__`定义。
* 生成器使**迭代器的协议**得以实现，因此你可以使用迭代的方法遍历生成器。不同于迭代器，一旦计算，生成器是不可能回复或重置的（迭代器可以通过索引或其他方法再次计算）。
* 生成器可以 **send information**， 可以让生成器成为概念上的**协程**。
* 在 Python3 中，你可以使用`yield from`关键字双向 **delegate** 一个生成器到另一个生成器。

上面的四条到底是神马！怎么看着晕乎乎的。:D真不好意思，这是我翻译的~_~我自己第一眼看到也是同样苦恼，不过没关系，先看看我参考的资料呗：

* 当然首先推荐这个啦：[What does the yield keyword do in Python?]（置顶的回答）。我猜能玩全看懂的不多。
* 那就看看这个：[Python函数式编程指南（四）：生成器]
* 想了解第二条的含义，看看这两篇：[python3-cookbook 4.2 代理迭代]、[python3-cookbook 4.3 使用生成器创建新的迭代模式]

能把前两个看一半，最后一个看完，我相信前两条一定可以明白的。我在下面写一写我当初的困惑：

* 函数中有多个`yield`，`next()`是啥情况
* next(generator)执行到哪里（这点很重要噢）
* `result = yield`这个表达式的含义（拜托，不要质疑，没少写，这是来源：[python3-cookbook 7.10 带额外状态信息的回调函数]。不相信？俺人好：[Python Cookbook:Carrying Extra State with Callback Functions]，“啪啪啪”）

第一个很简单，我就懒得写了。关于第二个，先看看生成器的三个方法：

`send()`：简单地说，`next()`方法可以恢复生成器状态并继续执行，`send()`是除`next()`外另一个恢复生成器的方法。在 Python3 中，`yield`语句是个表达式，那么表达式可以有一个值，这个值就是`send(msg)`所传递的 *msg*。`send(None)`（或者说是`send()`）等价与`next()`，这时候`yield`表达式的值嘛，猜呗^^ 下面是个栗子：

```python
>>> def foo():
...     i = 0
...     while True:
...         value = yield i
...         print("value is", value)
...         i += 1
...
>>> func = foo()
>>> next(func)
0
>>> func.__next__() # 囧，第二个问题的答案暴露了
value is None
1
>>> func.send() # 相当于 next(func), func.__next__()
value is None
2
>>> func.send("hello")
value is hello
3
```

剩下两个方法：`close()`——这个方法用于关闭生成器，对关闭的生成器后再次调用`next()`或`send(msg)`将抛出 StopIteration 异常。`throw()`—— 可以引发任何类型的异常。在这里就不过多说了（其实我也不懂T_T）。到这里已经结束第二个问题了。纳尼！答案呢？我鄙视你，自己看栗子。

第三个嘛，下面是我自己“猜”的，若有错误欢迎指证：

```python
>>> def foo1():
...     yield 1
...
>>> func = foo1()
>>> print(next(func))
1
>>> def foo2():
...     yield None
...
>>> func = foo2()
>>> print(next(func))
None
>>> def foo2():
...     yield
...
>>> func = foo2()
>>> print(next(func))
None
```

So easy？到这里，是不是更清晰了呢？**send infomation** 是不是也明白了呢？下面是关于`yield from`的，困扰了我好久。Ready？ Go！

我又懒了，又想让你戳屏幕了(~_~)

* [Python3中的yield from语法] PEP-380 这一段很赞！
* [python3-cookbook 4.4 实现迭代器协议] 找张纸，拿支笔，慢慢琢磨。
* 现在看看[What does the yield keyword do in Python?]，是不是很清晰了？
* →_→ 看不懂，我的理解在下面。

我在这里只能写（抄）一写（抄）目前的拙见：

```python
def A():
   ...
   yield B()
   ...
```

`B()`返回的是一个可迭代对象 b，那么`A()`会返回一个 generator ——暂且叫 a 吧。

1. b 迭代产生的每个值都直接传递给 a 的调用者。  
2. 所有通过`send(msg)`发送到 a 的值都被直接传递给 b。若`send(None)`，则执行 b 的`__next__()`方法。  
3. 若对 b 的方法调用（`send(msg)`或`__next__()`方法）产生 *StopIteration* 异常，a 会继续执行`yield from`后面的语句；而其他异常则会传播到 a，导致 a 在执行`yield from`的时候抛出异常。  
4. 若除 *GeneratorExit* 以外的异常被`throw`到 a 中，该异常会被直接`throw`到 b 中。b `throw`方法产生的任何异常，a 的行为同对 b 的方法调用产生的异常。  
5. 如果 *GeneratorExit* 异常被`throw`到 a 中，或者 a 的`close`方法被调用，若 b 也有`close`方法，b `close`方法也会被调用。b 的`close`方法抛出异常，则会导致 a 也抛出异常。若 b 成功`close`，a 会抛出 *GeneratorExit* 异常。  
6. a 中`yield from`表达式的求值结果是 b 迭代结束时抛出的 *StopIteration* 异常的第一个参数。  
7. b 中的`return <expr>`语句实际上会抛出 *StopIteration* 异常，b 中`return`的值会成为a中`yield from`表达式的值  

上面七句话说了这么个意思：

* 1、2说明 a 与 b 之间`send(msg)`的双向 **delegate** 以及 a 与 b 的行为
* 3、4、5说明异常的双向 **delegate** 以及 a 与 b 的行为
* 6、7说明了`yield from`的表达式的值是什么

接下来看那几段代码：

```python
def money_manager(expected_rate):
    under_management = yield     # must receive deposited value
    while True:
        try:
            additional_investment = yield expected_rate * under_management
            if additional_investment:
                under_management += additional_investment
        except GeneratorExit:
            '''TODO: write function to send unclaimed funds to state'''
        finally:
            '''TODO: write function to mail tax info to client'''


def investment_account(deposited, manager):
    '''very simple model of an investment account that delegates to a manager'''
    next(manager)
    # 启动生成器，manager 执行到第一句，under_management 对象未赋值
    manager.send(deposited)
    # under_management 对象被赋值为 desposited
    while True:
        try:
            yield from manager
        except GeneratorExit:
            return manager.close()

# 执行
>>> my_manager = money_manager(.06)
>>> my_account = investment_account(1000, my_manager)
>>> first_year_return = next(my_account)
# 实际上为 next(my_manager)，抛出60.0，做了一趟穿越，赋给 first_year_return
>>> first_year_return
60.0
>>> next_year_return = my_account.send(first_year_return + 1000)
# 实际上为 my_manager.send(first_year_return + 1000)
>>> next_year_return
123.6
```

第二个。嗨！代码太长了，都深夜了，明天还得早起，不干了，自己多画画:>好运！

## 参考资料

[Python函数式编程指南（三）：迭代器](http://www.cnblogs.com/huxi/archive/2011/07/01/2095931.html)  
[Python函数式编程指南（四）：生成器](http://www.cnblogs.com/huxi/archive/2011/07/14/2106863.html)  
[python3-cookbook 4.2 代理迭代](http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p02_delegating_iteration.html)  
[python3-cookbook 4.3 使用生成器创建新的迭代模式](http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p03_create_new_iteration_with_generators.html)  
[python3-cookbook 4.4 实现迭代器协议](http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p04_implement_iterator_protocol.html)  
[python3-cookbook 7.10 带额外状态信息的回调函数](http://python3-cookbook.readthedocs.org/zh_CN/latest/c07/p10_carry_extra_state_with_callback_functions.html)  
[What does the yield keyword do in Python?](http://stackoverflow.com/a/31042491)  
[Python3中的yield from语法](http://blog.theerrorlog.com/yield-from-in-python-3.html)  

[Python函数式编程指南（四）：生成器]: http://www.cnblogs.com/huxi/archive/2011/07/14/2106863.html
[What does the yield keyword do in Python?]: http://stackoverflow.com/a/31042491
[python3-cookbook 4.2 代理迭代]: http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p02_delegating_iteration.html
[python3-cookbook 4.3 使用生成器创建新的迭代模式]: http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p03_create_new_iteration_with_generators.html
[python3-cookbook 7.10 带额外状态信息的回调函数]: http://python3-cookbook.readthedocs.org/zh_CN/latest/c07/p10_carry_extra_state_with_callback_functions.html
[Python Cookbook:Carrying Extra State with Callback Functions]: http://chimera.labs.oreilly.com/books/1230000000393/ch07.html#_solution_116
[Python3中的yield from语法]: http://blog.theerrorlog.com/yield-from-in-python-3.html
[python3-cookbook 4.4 实现迭代器协议]: http://python3-cookbook.readthedocs.org/zh_CN/latest/c04/p04_implement_iterator_protocol.html
