---
title: Python之闭包与装饰器
date: 2016-02-17
categories: style
tags: ['language', 'python']
---

Python 函数那些事的后续之一：这里记录一些关于 Python 作用域和命名空间、闭包、装饰器的东西。

## Python作用域和命名空间

命名空间是从命名到对象的集合。Python 使用它来记录变量的轨迹。当前命名空间主要是通过 Python 字典实现的。关于命名空间需要了解的一件很重要的事情就是**不同命名空间中的命名没有任何联系**。不同的命名空间在不同的时刻创建，有不同的生存期。

* 包含内置命名的命名空间（它被包含在一个模块中，这个模块被称作 *builtins*。 ）在 Python 解释器启动时创建，会一直保留，不被删除。
* 模块的全局命名空间在模块定义被读入时创建，通常，模块命名空间也会一直保存到解释器退出。
* 当调用函数时，就会为它创建一个局部命名空间，并且在函数返回或抛出一个并没有在函数内部处理的异常时被删除。
* 每个递归调用都有自己的局部命名空间。
* 由解释器在最高层调用执行的语句，不管它是从脚本文件中读入还是来自交互式输入，都是 *__main__* 模块的一部分。
* 命名空间可以像 dictionary 一样进行访问的。
* 使用`locals()`与`global()`来分别打印局部变量与全局变量的命名空间。`local()`返回局部名字的一个拷贝，因此是只读的。`global()`则不然，因此不是只读的。
* `sys._getframe()`返回堆栈上的数据，参数表示栈的深度，默认参数为0。`sys._getframe().f_locals`访问当前的命名空间。

作用域就是一个 Python 程序可以直接访问命名空间的正文区域。

Python 函数的作用域为：

* L: local 函数的内部作用域
* E: enclosing 函数内部与内嵌函数之间
* G: global 全局作用域
* B: build-in 内置作用域

访问顺序从左至右： L > E > G > B。

使用`nonlocal`语句可以绑定最里层作用域之外的变量；`global`语句将变量引入到全局作用域。

Python 的一个特别之处在于：如果没有使用`global`语法，其赋值操作总是在最里层的作用域。**赋值不会复制数据，只是将命名绑定到对象。**（详细看[这里](http://cenalulu.github.io/python/default-mutable-arguments/)）删除也是如此：`del x`只是从局部作用域的命名空间中删除命名 *x* 。事实上，所有引入新命名的操作都作用于局部作用域。特别是`import`语句和函数定将模块名或函数绑定于局部作用域（可以使用`global`语句将变量引入到全局作用域）。

一个栗子：

```python
def scope_test():
    def do_local():
        spam = "local spam"
    def do_nonlocal():
        nonlocal spam
        spam = "nonlocal spam"
    def do_global():
        global spam
        spam = "global spam"
    spam = "test spam"
    do_local()
    print("After local assignment:", spam)
    do_nonlocal()
    print("After nonlocal assignment:", spam)
    do_global()
    print("After global assignment:", spam)

scope_test()
print("In global scope:", spam)

# 输出
>>> After local assignment: test spam
>>> After nonlocal assignment: nonlocal spam
>>> After global assignment: nonlocal spam
>>> In global scope: global spam
```

## 闭包

Python 的闭包从表现形式上定义（解释）为：如果在一个内部函数里，对在外部作用域（但不是在全局作用域，是 enclosing 作用域）的变量进行引用，那么内部函数就被认为是闭包。

先理清一个概念——Python 函数的实质与属性：

* 函数是一个对象（确切的说，python 中一切皆对象）。
* 函数执行完成后内部变量的回收：这涉及到命名空间，还记得上面的一句话吗？赋值不会复制数据，只是将命名绑定到对象！赋值不会复制数据，只是将命名绑定到对象!赋值不会复制数据，只是将命名绑定到对象！重要的事情说三遍。那么当没有命名绑定到对象（也就是引用计数为0）的时候，该对象被回收。
* 函数属性：对象皆有属性，函数的属性是一个 tuple。
* 函数返回值：函数的返回值不会被回收。

举个栗子，下面是一个闭包：

```python
def func(val):
    print('%x' %id(val))
    def in_func(): # (val,)
        print(val)
    return in_func

# 执行
>>> f = func(10)
8ad8a0
>>> f.__closure__
(<cell at 0x7f7995dea948: int object at 0x8ad8a0>,)
```

函数`in_func()`是返回值，因此在函数`func()`执行完毕后它并不会被销毁。那么问题来了：函数`func()`的局部变量 *val* 去了哪里？它成为函数的返回值——`in_func()`函数的**属性**（当内部嵌套的函数引用外部函数中的变量时，我们说嵌套函数相对于引用变量是封闭的。我们可以使用函数对象的一个特殊属性 *__closure__* 来访问这个封闭的变量）。

使用闭包时候，需要注意两点：

* 闭包中是不能修改外部作用域的局部变量的（Python 作用域访问顺序，函数的属性是一个 tuple）
* Python 函数的惰性运算

下面是两个栗子：

```python
# first
def foo():  
    a = 1  
    def bar():  
        a = a + 1  
        return a  
    return bar

# 本意是每次使用都对变量 a 进行递增操作
# 实际上
>>> c = foo()  
>>> print(c())  
Traceback (most recent call last):  
  File "<stdin>", line 1, in <module>  
  File "<stdin>", line 4, in bar  
UnboundLocalError: local variable 'a' referenced before assignment
```

这是因为在执行代码`c = foo()`时，python 会导入全部的闭包函数体`bar()`来分析其的局部变量，python 规则指定所有在赋值语句左面的变量都是局部变量，则在闭包`bar()`中，变量 *a* 在赋值符号 *=* 的左面，被 python 认为是`bar()`中的局部变量。再接下来执行`print c()`时，程序运行至`a = a + 1`时，因为先前已经把 *a* 归为`bar()`中的局部变量，所以 python 会在`bar()`中去找在赋值语句右面的 *a* 的值，结果找不到，就会报错。解决办法：

```python
# 借助 python3 关键字 nonlocal
def foo()
    a = 1
    def bar():
        nonlocal a
        a = a + 1
        return a
    return bar

# 或者
def foo():  
    a = [1]  
    def bar():  
        a[0] = a[0] + 1  
        return a[0]  
    return bar
```

```python
# second
def create_multipliers():
    return [lambda x : i * x for i in range(5)]

>>> for multiplier in create_multipliers():
...     print multiplier(2)
8
8
8
8
8

# 或者这样
first = []
for i in range(5):  
    def foo(x):
        return print(x * i)  
    first.append(foo)

>>> for f in first:
...     f(2)
8
8
8
8
8
```

（这点曾经在《python函数那些事》中的“匿名或内联函数”谈到过）这是因为：当循环结束以后，循环体中的临时变量 *i* 不会销毁，而是继续存在于执行环境中。还有一个 python 的现象是，python 的函数只有在执行时，才会去找函数体里的变量的值（late binding behavior）。

解决办法：

```python
# 第一个
def create_multipliers():
    return [lambda x, i=i : i * x for i in range(5)]

# 第二个
first = []
for i in range(3):  
    def foo(x,y=i): print x + y  
    flist.append(foo)  
```

## 装饰器

闭包的一个用处就是装饰器。

装饰器是用来装饰函数的；它返回的是一个函数对象；被装饰函数的标识符指向返回函数的对象；语法糖`@deco`。

下面是个栗子（天，我自己都感觉栗子有点太多了！）：

```python
def log(func):
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper


@log
def now():
    print('2015-02-17')

# 执行的话会这样
>>> now()
call now():
2016-2-17

# 实际上执行步骤相当于与这样
>>> log(now)()
call wrapper():
call now():
2016-02-17
```

其实到这里已经把装饰器说明白了（我认为，囧），但是这几天经常看到`functools.wraps`这个装饰器。它是应对这种情况的：

```python
# 还是上面那段代码
>>> now.__name__
'wrapper'
```

不加装饰器之前`now.__name__`是`now`的。有些代码依赖函数签名，我想让`now.__name__`变为`now`，那该怎么办？把装饰器改写成这样：

```python
def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kw):
        print('call %s():' % func.__name__)
        return func(*args, **kw)
    return wrapper
```

不信？那您试试呗~

## 参考资料

[Python tutorial Python作用域和命名空间](http://www.pythondoc.com/pythontutorial3/classes.html)  
[Python命名空间的本质](http://www.cnblogs.com/windlaughing/archive/2013/05/26/3100362.html)  
[慕课网视频 Python装饰器](http://www.imooc.com/learn/581)  
[Python中的闭包](http://blog.csdn.net/marty_fu/article/details/7679297)  
[Become More Advanced: Avoid the 10 Most Common Mistakes That Python Programmers Make](http://www.toptal.com/python/top-10-mistakes-that-python-programmers-make)  
[廖雪峰Python教程 装饰器](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014318435599930270c0381a3b44db991cd6d858064ac0000)
