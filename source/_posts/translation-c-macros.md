---
title: '译文：C 的宏'
date: 2019-03-14 19:12:57
tags: ['translation', 'c']
---

> 这篇译文印象中于2017年9月底10月初保研之后所写，最初发布在[西电开源社区社区的wiki](https://gitlab.com/XDOSC/WIFI/wikis/macros)上面。
> 这是一篇来自 GNU GCC 关于预处理部分宏相关的文档的总结，原文参考底部Reference。

宏（macro）可以说是一种模式替换，它根据一系列预定义的规则替换一定的文本。有两种宏：对象式宏（Object-like macros），函数式宏（function-like macros）。前者使用的时候就像数据对象一样，后者类似函数调用。

宏的标识符可以是任何字符，甚至是C的关键字（这点比较有用，可以用它来隐藏C语言的关键字，比如`const`关键字在老的编译器中并不支持，可以用此特性实现编译器之间的兼容。但一些预处理的操作符并不可以被定义成宏，参见[这里](https://gcc.gnu.org/onlinedocs/cpp/Defined.html#Defined)，同样一些[C++ Name Operators](https://gcc.gnu.org/onlinedocs/cpp/C_002b_002b-Named-Operators.html#C_002b_002b-Named-Operators)也不可以。

使用`#define`指示创建宏，`#undef`取消宏定义。语法：

```c
#define <identifier> <body>
#undef  <identifier>
```

宏定义必须在一行之中，换行的话要使用`\`。
  
Note: '替换'、'展开'两词语的含义类似。

## 对象式宏(Object-like Macros)

```c
#define BUFFER_SIZE 1024
foo = (char *) malloc (BUFFER_SIZE);
     → foo = (char *) malloc (1024);
/*
  BUFFER_SIZE 被替换为 1024
*/


#define NUMBERS 1, \
                2, \
                3
int x[] = { NUMBERS };
     → int x[] = { 1, 2, 3 };
/*
  NUMBER 被替换为 1, 2, 3
*/


#define TABLESIZE BUFSIZE
#define BUFSIZE 1024
TABLESIZE
     → BUFSIZE
     → 1024
/*
  TABLESIZE 先被替换为 BUFSIZE，再被替换为 1024
*/
```

## 函数式宏(Function-like Macros)

```c
#define lang_init()  c_init()
lang_init()
     → c_init()
/*
  lang_init() 被替换为 c_init()
*/


#define lang_init ()    c_init()
lang_init()
     → () c_init()()
/*
  如果<函数名>与'()'之间有空格的话...
*/


extern void foo(void);
#define foo() /* optimized inline version */
...
  foo();
  funcptr = foo;
/*
  函数式宏当且仅当在它之后出现一对括号才展开
  如果只写了了它的名，比如上面，它的含义其实是一个函数指针
*/
```

## 宏参数(Macro Arguments)

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
  x = min(a, b);          →  x = ((a) < (b) ? (a) : (b));
  y = min(1, 2);          →  y = ((1) < (2) ? (1) : (2));
  z = min(a + 28, *p);    →  z = ((a + 28) < (*p) ? (a + 28) : (*p));
/*
  宏被展开后，在宏所定义的内容中，每一个宏参数都会被替换
*/
```

在宏所定义的内容中，使用括号包裹参数是一个技巧，稍后再讨论: [宏陷阱(Macro Pitfalls): Operator Precedence Problems](#operator-precedence-problems)。

```c
/*
  关于参数为空的情况，下面是一些说明
*/
min(, b)        → ((   ) < (b) ? (   ) : (b))
min(a, )        → ((a  ) < ( ) ? (a  ) : ( ))
min(,)          → ((   ) < ( ) ? (   ) : ( ))
min((,),)       → (((,)) < ( ) ? ((,)) : ( ))

min()      error→ macro "min" requires 2 arguments, but only 1 given
min(,,)    error→ macro "min" passed 3 arguments, but takes just 2


#define foo(x) x, "x"
foo(bar)        → bar, "x"
/*
  在宏所定义的内容中，被 "" 包住的参数并不被替换
*/
```

## 字符串化(stringizing)

把宏参数转换成字符串，预处理操作符`#`具备这个功能，宏参数以`#`开头。

需要说明的是：

1. 序列化后的字符串若被打印出来，一定等同于原始字符串
2. 在被序列化字符串的前后位置，如果存在空白字符，他们会被忽略；在被序列化的字符串中间，如果存在多个空白字符，他们会被缩减成一个空白字符。

```c
#define WARN_IF(EXP) \
do { if (EXP) \
        fprintf (stderr, "Warning: " #EXP "\n"); } \
while (0)
WARN_IF (x == 0);
     → do { if (x == 0)
           fprintf (stderr, "Warning: " "x == 0" "\n"); } while (0);
/*
  EXP →  x == 0
  #EXP →  "x == 0"
*/
```

`do {...} while(0)`是一个技巧，稍后再讨论: [宏陷阱(Macro Pitfalls): Swallowing the Semicolon](#swallowing-the-semicolon)。

```c
#define str(s) #s
str("string\n") →  "\"string\\n\""
str( str  str ) →  "str str"
/*
  需要说明的情况举例
*/
```

## 级联(Concatenation)

*token pasting* 或者称为 *token concatenation*，指的是当展开宏的时候，把两个符号(token)合并成一个。`##`操作符提供了这项功能，位于`##`操作符两边的符号会被合并成为一个符号，它会替代`##`以及两个原始的符号。如果两个符号合并后不是一个有效的符号，预处理器将会出现警告。

处理注释的时间在处理`##`之前，因此在`##`以及它将要拼接的标记之间添加标记并没有啥问题，也可以添加大量的空白，但`##`的后面是以空白结束的话会报错。

```c
struct command
{
  char *name;
  void (*function) (void);
};
#define COMMAND(NAME)  { #NAME, NAME ## _command }

struct command commands[] =
{
  COMMAND (quit),
  COMMAND (help),
  ...
};
     →  struct command commands[] =
        {
            { "quit", quit_command },
            { "help", help_command },
            ...
        };
```

## 宏的可变参数(Variadic Macros)

```c
#define eprintf(...) fprintf (stderr, __VA_ARGS__)
eprintf ("%s:%d: ", input_file, lineno)
     →  fprintf (stderr, "%s:%d: ", input_file, lineno)
/*
  __VA_ARGS__ 标识符会被宏参数替换
*/

#define eprintf(args...) fprintf (stderr, args)
eprintf ("%s:%d: ", input_file, lineno)
     →  fprintf (stderr, "%s:%d: ", input_file, lineno)
/*
  不使用 __VA_ARGS__ 的方法，GNU CPP 支持
*/

#define eprintf(format, ...) fprintf (stderr, format, __VA_ARGS__)
eprintf("success!\n", );
     → fprintf(stderr, "success!\n", );
/*
  标准 C 中，不能省略 ','
*/
eprintf ("success!\n")
     → fprintf(stderr, "success!\n", );
/*
  GNU CPP 中，可以省略 ','
*/


#define eprintf(format, ...) fprintf (stderr, format, ##__VA_ARGS__)
eprintf ("%s:%d: ", inputfile, lineno)
     →  fprintf (stderr, "%s:%d: ", input_file, lineno)
eprintf ("success!\n")
     → fprintf(stderr, "success!\n");
eprintf ()
     →  fprintf (stderr, , )
/*
  添加 '##'
  如果可变被省略，位于 ## 之前的 ',' 会被删除
  如果传入的参数为空，',' 不会被删除
*/
```

## 预定义的宏(Predefined Macros)

三类对象式宏已经被定义：标准(standard)、常见(common)、系统相关(system-specific)。对于C++，还有一类：命名的运算符(the named operators)。

### Standard Predefined Macros

[3.7.1 Standard Predefined Macros](https://gcc.gnu.org/onlinedocs/cpp/Standard-Predefined-Macros.html#Standard-Predefined-Macros)

#### Common Predefined Macros

[3.7.2 Common Predefined Macros](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html#Common-Predefined-Macros)

### System-specific Predefined Macros

[3.7.3 System-specific Predefined Macros](https://gcc.gnu.org/onlinedocs/cpp/System-specific-Predefined-Macros.html#System-specific-Predefined-Macros)

### C++ Named Operators

[3.7.4 C++ Named Operators](https://gcc.gnu.org/onlinedocs/cpp/C_002b_002b-Named-Operators.html#C_002b_002b-Named-Operators)

## 取消或重新定义宏(Undefining and Redefining Macros)

宏定义后，取消定义后再定义，它没有任何限制。但在不取消定义的情况下再次定义则会有一些限制：

1. 宏的类型要一致(对象式或函数式)
2. 宏所定义的内容中标记要相同
3. 如果存在参数，参数要相同
4. 空白字符的数量要一致

```c
#define FOUR (2 + 2)
#define FOUR         (2    +    2)
#define FOUR (2 /* two */ + 2)
/*
  上述重定义的宏是被允许的
  重定义的宏与旧的相同，编译器会忽略重定义的宏
*/


#define FOUR (2 + 2)
#define FOUR ( 2+2 )
#define FOUR (2 * 2)
#define FOUR(score,and,seven,years,ago) (2 + 2)
/*
  上述重定义的宏会产生警告
  编译器会产生警告，使用新定义的宏把旧的替换掉
*/
```

不同的头文件定义相同的宏是被允许的。

## 宏参数中的指令(Directives Within Macro Arguments)

有时候需要在宏的参数中使用预处理指令。标准C/C++不允许这种情况，GNU CPP 预处理器允许这种情况。

```c
#define f(x) x x
f (1
#undef f
#define f 2
f)
     →  1 2 1 2
/*
  在宏调用中，该宏被重新定义，但原始定义仍然用于参数替换，新定义的宏在此基础上进行展开
*/
```

## 宏陷阱(Macro Pitfalls)

## Misnesting

```c
#define twice(x) (2*(x))
#define call_with_1(x) x(1)
call_with_1 (twice)
     → twice(1)
     → (2*(1))
/*
  在宏所定义的内容中，宏参数被替换，替换后的结果被检查，以便决定是否进一步展开
*/


#define strange(file) fprintf (file, "%s %d",
...
strange(stderr) p, 35)
     → fprintf (stderr, "%s %d", p, 35)
/*
  宏展开后并不补全缺失的括号
*/
```

### Operator Precedence Problems

这里讨论使用括号包裹宏参数这个技巧。

```c
#define ceil_div(x, y) (x + y - 1) / y
a = ceil_div (b & c, sizeof (int));
     → a = (b & c + sizeof (int) - 1) / sizeof (int);

// 这是我们想要的结果
a = ((b & c) + sizeof (int) - 1)) / sizeof (int);

// 因为 C 语言运算符号优先级的问题，实际上是这样
a = (b & (c + sizeof (int) - 1)) / sizeof (int);

// 把它定义成这样就不会有问题啦
#define ceil_div(x, y) ((x) + (y) - 1) / (y)
```

### Swallowing the Semicolon

这里讨论`do {...} while(0)`技巧。

```c
#define SKIP_SPACES(p, limit)  \
{ char *lim = (limit);         \
  while (p < lim) {            \
    if (*p++ != ' ') {         \
      p--; break; }}}
/*
  SKIP_SPACES 看起来很像一个函数，往往会在它后面加上 ';'
*/

if (*p != 0)
  SKIP_SPACES (p, lim);
else ...
/*
  因为 ';' 的存在，导致了一个语法错误
*/

#define SKIP_SPACES(p, limit)     \
do { char *lim = (limit);         \
     while (p < lim) {            \
       if (*p++ != ' ') {         \
         p--; break; }}}          \
while (0)
/*
  'SKIP_SPACES (p, lim);' 会被展开成 'do {...} while (0);'
  完美
*/
```

### Duplication of Side Effects

`({...})`是 GNU 的拓展，它的返回值是最后一行语句的值。

```c
#define min(X, Y)  ((X) < (Y) ? (X) : (Y))
/*
  往往这么定义一个求最小值的宏
*/

next = ((x + y) < (foo (z)) ? (x + y) : (foo (z)));
/*
  如果 foo (z) 有副作用就违背了原意
*/

#define min(X, Y)                \
({ typeof (X) x_ = (X);          \
   typeof (Y) y_ = (Y);          \
   (x_ < y_) ? x_ : y_; })
/*
  如果打算使用 GNU C 拓展的话，可以定义成这样
*/

{
  int tem = foo (z);
  next = min (x + y, tem);
}
/*
  不使用 GNU C 拓展的话，这是唯一的解决办法
*/
```

Note: 副作用指的是值被改变。

### Self-Referential Macros

循环调用指的是在一个宏的展开式中存在它本身。

循环调用会导致宏的不断展开，这种情况有个规定：展开一次后就不再展开了。

```c
#define foo (4 + foo)
/*
  循环调用的一个例子
*/


#define foo (4 + foo)
foo    → (4 + foo)
/*
  展开结果举例
*/


#define x (4 + y)
#define y (2 * x)
x    → (4 + y)
     → (4 + (2 * x))

y    → (2 * x)
     → (2 * (4 + y))
/*
  间接(indirect self-reference)的循环调用举例
*/
```

### Argument Prescan

宏参数在替换宏所定义的内容之前，它会被完全展开，除非它被字符串化，或者 pasted with other tokens。替换之后，宏所定义的所有内容，包括被替换后的参数，会被再次扫描，以便再次展开。结果就是参数会被扫描两次。

如果参数包含宏，它在第一次扫描的时候便会被展开，之后在宏所定义的内容中被替换，第二次扫描并没有啥作用，一次扫描也能达到这个效果。

循环引用的宏如果被当作其他宏的参数，它如果在第一次扫描不被展开的话，会被标记，第二次扫描同样也不会展开它。

预扫描还是有点作用的，体现在这几个地方：

* 嵌套调用宏

这种对宏的嵌套调用发生在当一个宏的参数包含了一个对不一般宏调用的时候。比如，`f`是一个宏，有一个参数，`f (f (1))`是一对对`f`的嵌套调用，`f (1)`(内部)会被展开，随后它将替换宏所定义的内容。如果没有预扫描，`f (1)`它自己会被作为参数传递进去，这样内部会存在`f`，从而在下一次扫描中出现间接的循环调用。

```c
#define f(arg) arg

// 预扫描的存在，替换过程如下: 先展开 'f__ (arg)'，在展开 'f_ (arg)'
f_ (f__ (1))
     → f_ (1)
     → 1

// 如果不存在预扫描的话，'f (f (1)' 相当于定义了
#define f(arg) f(arg)

Note: 为了区分两个 'f'，添加了 下标，'f_' / 'f__' 都代之 'f'
```

* 宏有字符串化或级联的作用，同时其他宏作为它的参数

```c
#define AFTERX(x) X_ ## x
#define XAFTERX(x) AFTERX(x)
#define TABLESIZE 1024
#define BUFSIZE TABLESIZE
AFTERX(BUFSIZE)     →  X_BUFSIZE
XAFTERX(BUFSIZE)    →  X_1024
/*
  如果参数被字符串化或者级联，预扫描不会作用于参数
*/
```

* 宏作为参数，但展开式包含未被括起来的逗号

```c
#define foo  a,b
#define bar(x) lose(x)
#define lose(x) (1 + (x))
bar(foo)
     → bar(a,b)
     → lose(a,b)
/*
  报错，因为 lose() 只接收一个参数
  正确的写法是
    #define foo (a,b)
*/

```

### Newlines in Arguments

利用了函数式宏可以展开为多个逻辑行的特性。

```c
#define ignore_second_arg(a,b,c) a; c

ignore_second_arg (foo (),
                   ignored (),
                   syntax error);
```

## References

[宏](https://zh.wikipedia.org/wiki/%E5%B7%A8%E9%9B%86)
[GNU onlinedocs: Macros](https://gcc.gnu.org/onlinedocs/cpp/Macros.html#Macros)

## Summary

个人感觉，能看懂这三行就差不多了：

```c
#define fo(x) #x
#define foo(x) X##x
#define fooo(x) \
  ({ \
    typeof(x) local_x = x * x; \
    local_x; \
  )}
```