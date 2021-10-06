---
title: Python 函数那些事
date: 2016-02-14
categories: style
tags: ['language', 'python']
---

Python 中使用`def`语句定义的函数，除了非常简单之外，灵活度还非常大。在这里记录一些关于 Python 函数的东东。

## 函数的参数

函数的参数有这么几种：位置参数、默认参数、可变参数、关键字参数、命名关键字参数。

先看看位置参数。

```python
# 一个计算x平方的函数
def power(x):
    return x * x
```

对于`power(x)`函数，参数*x*就是一个位置参数。

但是，若是需要计算x的n次方呢？那就这酱：

```python
def power(x, n):
    s - 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
```

修改后的`power(x, n)`函数具有两个参数：*x*与*n*，这两个参数都是位置参数，调用函数时，传入的两个值按照位置顺序依次赋给*x*与*n*。

But，这样修改后，旧的函数无法使用了，而且n的值经常为2呀，每次都输入n的值太麻烦了。

表着急，默认参数登场了。还是一段代码：

```python
def power(x, n=2):
    s = 1
    while n > 0:
        n = n - 1
        s = s * x
    return s
```

这时候，再次调用`power(x)`，相当于调用`power(x, 2)`

设置默认参数时，还需要注意两点：一是必选参数在前，默认参数在后，否则 Python 的解释器会报错；二是如何设置默认参数：当函数有多个参数时，把变化大的参数放前面，变化小的参数放后面，变化小的参数就可以作为默认参数。

特别说明的一点：**Python 函数在定义的时候，默认参数的值就被计算出来了。**

```python
# 遇到这么个情况，请不要惊讶

def add_end(L=[]):
    L.append('END')
    return L

# 执行
>>> add_end()
['END']
>>> add_end()
['END', 'END']
>>> add_end()
['END', 'END', 'END']
```

在想：Python 中，能不能传递任意数量参数给函数呢？答案是可以的（可变参数）。

```python
def calc(*numbers):
    sum = 0
    for n in numbers:
        sum = sum + n * n
    return sum
```

调用的时候，就可以这么调用啦：`calc(1, 3, 5, 7)`。但如果参数本身是一个 list 或 tuple 呢？比如`nums = (1, 3, 5)`，那可以这么调用：`calc(*nums)`。（`*nums` 表示把`nums`这个 tuple 的所有元素作为可变参数传递进去）

可变参数允许传入0个或任一个参数，这些可变参数在函数调用时自动组床为一个 tuple。而关键字参数允许传入0个或任一个含参数名的参数，这些关键字参数在函数内部自动组装为一个 dict。

```python
def person(name, age, **kw):
    print('name:', name, 'age', age, 'other', kw)

# 执行
>>> person('Adam', 45, gender='M', job='Engineer')
name: Adam age: 45 other: {'gender': 'M', 'job': 'Engineer'}

# 当然也有与关键字参数一样的用法
>>> extra = {'city': 'Beijing', 'job': 'Engineer'}
>>> person('Jack', 24, **extra)
name: Jack age: 24 other: {'city': 'Beijing', 'job': 'Engineer'}
```

需要注意的是：关键字参数只能出现在最后一个参数。

还有一种是命名关键字参数。它是用来限制关键字参数的名字。

```python
# 比如只接受 city 和 job 作为关键字参数

def person(name, age, *, city, job):
    print(name, age, city, job)
# * 是一个特殊的分隔符，*后面的参数被视为命名关键字参数

# 执行
>>> person('Jack', 24, city='Beijing', job='Engineer')
Jack 24 Beijing Engineer
```

命名关键字参数还可以有缺省值，以简化调用。

## 增加函数参数的元信息

写好了一个函数，但是，怎让使用者知道如何使用函数呢？

使用函数参数注解十一个好办法，下面是一个被注解了的函数：

```python
def add(x:int, y:int) -> int:
    return x + y

# 执行
>>> help(add)
Help on function add in module __main__:
add(x: int, y: int) -> int

# 函数注解只存储在函数的 __annotations__ 属性中。
>>> add.__annotations__
{'return': <class 'int'>, 'x': <class 'int'>, 'y': <class 'int'>}
```

## 返回多值的函数

```python
def myfun():
    return 1, 2, 3

# 执行
>>> a, b, c = myfun() # 解压赋值，可用于任何可迭代对象
>>> a
1
>>> b
2
>>> c
3

>>> x = myfun()
>>> x
(1, 2, 3)
```

`myfun()`看上去返回了多个值，实际上是**创建了一个tuple后返回的。tuple的生成是用逗号而非括号，可迭代对象能够解压赋值。**

```python
>>> a = (1, 2) # With parentheses
>>> a
(1, 2)
>>> b = 1, 2 # Without parentheses
>>> b
(1, 2)
```

## 匿名或内联函数

目的是创建一个很短的回调函数，但又不想用`def`去写一个单行函数，希望通过某个快捷方式以内联方式来创建这个函数。

```python
add = lambda x, y: x + y

# 效果同
def add(x, y):
    return x + y
```

使用限制：只能指定单个表达式。不能包含其他语言特性，包括多个语句、条件表达式、迭代以及异常处理等等。

对于匿名函数捕获变量值，存在一些问题：

```python
# 错误的
>>> x = 10
>>> a = lambda y: x + y
>>> x = 20
>>> b = lambda y: x + y
>>> a(10)
30
>>> b(10)
30

# 正确的
>>> x = 10
>>> a = lambda y, x=x: x + y
>>> x = 20
>>> b = lambda y, x=x: x + y
>>> a(10)
20
>>> b(10)
30

# 这是因为 lambda 表达式中的x是一个自由变量， 在运行时绑定值，而不是定义时就绑定，这跟函数的默认值参数定义是不同的。
# 个人理解： Python 变量的特殊性——Python变量区别于其他编程语言的申明&赋值方式，采用的是创建&指向的类似于指针的方式实现的。

# 错误的
>>> funcs = [lambda x: x+n for n in range(5)]
>>> for f in funcs:
... print(f(0))
...
4
4
4
4
4

# 正确的
>>> funcs = [lambda x, n=n: x+n for n in range(5)]
>>> for f in funcs:
... print(f(0))
...
0
1
2
3
4

# 解释
# 1. 循环体运行后 n 并非立即销毁
# 2. lambda 的运行是惰性的（late binding behavior），循环体执行后，lambda 函数再运行
```

## 减少可调用对象的参数个数

是否遇到这么个问题：

“What！这函数一坨参数，我可不想每次都写这么一长串！”

“新修改的函数又不向后兼容了，怎么办呢？”

没关系，`functools.partial()`来帮助。`partial()`函数允许你给一个或多个参数设置固定的值，减少接下来被调用时的参数个数。

```python
def spam(a, b, c, d):
    print(a, b, c, d)

# 演示
>>> from functools import partial
>>> s1 = partial(spam, 1) # a = 1
>>> s1(2, 3, 4)
1 2 3 4
>>> s3 = partial(spam, 1, 2, d=42) # a = 1, b = 2, d = 42
>>> s3(3)
1 2 3 42
```

## 将单方法的类转变为函数

大多数情况下这是利用 Python 闭包的一个方法。先贴出代码：

示例中的类允许使用者根据某个模板方案来获取到URL链接地址。

```python
from urllib.request import urlopen

class UrlTemplate:
    def __init__(self, template):
        self.template = template

    def open(self, **kwargs):
        return urlopen(self.template.format_map(kwargs))

# Example use. Download stock data from yahoo
yahoo = UrlTemplate('http://finance.yahoo.com/d/quotes.csv?s={names}&f={fields}')
for line in yahoo.open(names='IBM,AAPL,FB', fields='sl1c1v'):
    print(line.decode('utf-8'))
```

这个类可以被一个更简单的函数来代替：

```python
def urltemplate(template):
    def opener(**kwargs):
        return urlopen(template.format_map(kwargs))
    return opener

# Example use
yahoo = urltemplate('http://finance.yahoo.com/d/quotes.csv?s={names}&f={fields}')
for line in yahoo(names='IBM,AAPL,FB', fields='sl1c1v'):
    print(line.decode('utf-8'))
```

大部分情况下，拥有一个单类方法的原因是需要存储某些额外的状态来给方法使用。这时候使用一个内部函数或者闭包的方案通常会更优雅一些。

## 其他

[python3-cookbook](http://python3-cookbook.readthedocs.org/zh_CN/latest/index.html)第七章（函数）还提到了这三点内容：

* 带额外状态信息的回调函数
* 内联回调函数
* 访问闭包中定义的变量

涉及到 python 中的*闭包*与*异步*，想利用较短的篇幅说明白也不太现实，遂有意另起几篇。

## 总结

* Python 中存在位置参数、默认参数、可变参数、关键字参数、命名关键字参数。
* `_*args_ 是可变参数，_**kwargs_ 是关键字参数。
* 定义命名的关键字参数不要忘了写分隔符*，否则定义的将是位置参数。目的是为了限制调用者可以传入的参数名，同时可以提供默认值。
* Python 中使用逗号生成 tuple。
* Python 的函数可以返回多值。实际上是创建一个 tuple 返回的。
* 一些简单的函数，比如仅仅只是计算一个表达式的值的时候，就可以使用 _lambda_ 表达式来代替了。
* 任何时候碰到需要给某个函数增加额外状态信息的问题，都可以考虑使用闭包。
* `functools.partial()` 可以更加直观的表达你的意图(给某些参数预先赋值)。

## 参考资料

[廖雪峰Python教程 函数的参数](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431752945034eb82ac80a3e64b9bb4929b16eeed1eb9000)  
[python3-cookbook 第七章 函数](http://python3-cookbook.readthedocs.org/zh_CN/latest/chapters/p07_functions.html)  
[Python list comprehension with lambdas [duplicate]](http://stackoverflow.com/questions/28268439/python-list-comprehension-with-lambdas)  
[Python中的闭包](http://blog.csdn.net/marty_fu/article/details/7679297)  
[Become More Advanced: Avoid the 10 Most Common Mistakes That Python Programmers Make](http://www.toptal.com/python/top-10-mistakes-that-python-programmers-make)  
[Python函数参数默认值的陷阱和原理深究](http://cenalulu.github.io/python/default-mutable-arguments/)
