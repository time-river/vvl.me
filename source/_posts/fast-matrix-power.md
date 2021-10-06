---
title: 矩阵快速幂
date: 2016-04-22 14:41:31
categories: style
tags: algorithm
mathjax: true
---

ACM校赛，get到新技能--关键词：矩阵快速幂、1000000007。╮(╯_╰)╭，为了方便理解，就使用 Python 代码了，最后附上 C 代码。

吐槽时间：
为啥会写这篇文章？ →_→ ACMer 的 blog 不堪入目 -代码凌乱不说，人话也不多说几句- 找了好多篇，不知所以然。而且，网上的实现代码大都是 C++ 的，但我看不懂唉。更何况，C++ 创建一个矩阵类，重载乘法运算，方便多了。用 C 呢，指针乱飞……

## 从 Fibonacci 到 矩阵快速幂 ;)

Fibonacci 数列的定义：
{% raw %}
$$
\begin{equation}
    \rm A(n) = \left\{\begin{array}{ll}
    A(n-1) + A(n-2) & x > 1 \\
    1 & x = 0, 1
    \end{array}
    \right.
\end{equation}
$$
{% endraw %}
那么，怎么求数列的第 n 项呢？首先想到的是递归：

```python
def fib_recursion(n):
    if (n == 0 or n == 1):
        return 1
    else:
        return fib_recursion(n-1) + fib_recursion(n-2)
```

考虑到递归一般都可以改成循环：

```python
def fib_loop(n):
    a, b = 1, 1
    while(n > 1):
        a, b = a+b, a
        n -= 1
    return a
```

恩，$O(n)$，线性的时间复杂度，现在看起来蛮不错。但当 n 是 $2^{32}$ 这种大数呢，$O(n)$都嫌弃了。OK，开始介绍矩阵快速幂--把$O(n)$变成$log(n)$的神奇方法。

它其实就是一个公式，重点在于矩阵的构造：
{% raw %}
$$
\begin{bmatrix}
A(n+1) \\
A(n)
\end{bmatrix}
=
\begin{bmatrix}
1 & 1 \\
1 & 0
\end{bmatrix}
*
\begin{bmatrix}
A(n) \\
A(n-1)
\end{bmatrix}
$$
{% endraw %}
更一般的：
{% raw %}
$$
\begin{bmatrix}
A(n+1) \\
A(n)
\end{bmatrix}
=
\begin{bmatrix}
1 & 1 \\
1 & 0
\end{bmatrix}
^n
*
\begin{bmatrix}
A(1) \\
A(0)
\end{bmatrix}
$$
{% endraw %}
其中， n 又可以这么表示（就是转换成二进制啦）：
{% raw %}
$$
n = \sum_{i=0}^k 2^i
$$
{% endraw %}
代码是这样的：

```python
import numpy as np
def fib_fast_matrix_power(n):
    frequency = n - 1 # frequency is less one than the index of factor
    result = np.matrix([[1], [1]])
    factor = np.matrix([[1, 1], [1, 0]])
    while(frequency > 0):
        if(frequency%2 == 1):
            result = factor * result
        factor = factor * factor
        frequency //= 2 # be divided with no remainder
        print("frequency:", frequency)
    return np.array(result)[0][0] # return A(n)
```

tips: 若是求从 0 到 n 项的和呢？有人给他起了个名字——拓展的 Fibonacci 数列。公式是这样的：
{% raw %}
$$
\begin{bmatrix}
S(n+1) \\
A(n+1) \\
A(n)
\end{bmatrix}
=
\begin{bmatrix}
1 & 1 & 1 \\
0 & 1 & 1 \\
0 & 1 & 0
\end{bmatrix}
^n
*
\begin{bmatrix}
S(1) \\
A(1) \\
A(0)
\end{bmatrix}
$$
{% endraw %}
那么，这种递推公式呢？
{% raw %}
$$
\begin{equation}
    \rm A(n) = \left\{\begin{array}{ll}
    \alpha A(n-1) + \beta A(n-2) + \gamma A(n-3) & x > 2 \\
    1 & x = 0, 1, 2
    \end{array}
    \right.
\end{equation}
$$
{% endraw %}
懒得构造矩阵了~，现在运算时间变短了，但若尝试大数的话，会出现溢出哦。

## 1000000007

这是一个质数，恩，很大的一个质数 - $10^9+7$ - 出现这种数，肯定是关于高精度整数求模啦。这是几个公式：

```math
( a + b ) % c = (( a % c ) + ( b % c )) % c
( a * b ) % c = (( a % c ) * ( b % c )) % c
( a – b ) % c = (( a % c ) – ( b % c )) % c
( a / b ) % c != (( a % c ) / ( b % c )) % c
```

一定要注意，第四个公式并__不成立__唉！
据说，这些公式在离散数学里面有提及，但是完全没有印象=_=。

## 总结

* 满足递归公式的数列，可以利用__矩阵快速幂__算法减少运算时间
* 矩阵快速幂的重点在于__矩阵的构造__
* $10^9+7$这样大质数的存在往往伴随着溢出
* 3+1 个公式

## 参考资料

[斐波那契数列：BestCoder Round #29 1002 || hdu 5171](http://blog.csdn.net/u012717411/article/details/43648457)  
[“OUTPUT THE ANSWER MODULO 10^9 + 7”](https://codeaccepted.wordpress.com/2014/02/15/output-the-answer-modulo-109-7/)  
[What is special/different with number 10^9+7? As most of the coding competitive websites asks to output for large number as modulo 10^9+7.](https://www.quora.com/What-is-special-different-with-number-10-9+7-As-most-of-the-coding-competitive-websites-asks-to-output-for-large-number-as-modulo-10-9+7)  

## 经历

下面娱乐时间：写一段经历——两天速成矩阵快速幂，也当作矩阵快速幂的一个栗子啦。省略不必要的废话，题目是这样的：

求 Tribonacci 数列从 l 到 r 项的和，结果取$10^9+7$的模。Tribonacci 数列定义如下：
{% raw %}
$$
\begin{equation}
    \rm A(n) = \left\{\begin{array}{ll}
    A(n-1) + A(n-2) + A(n-3) & x > 2 \\
    1 & x = 0, 1, 2
    \end{array}
    \right.
\end{equation}
$$
{% endraw %}
关键字：

* 大质数 - $10^9+7$ -
* 很多项的和

傻瓜都知道起码得用循环来解，于是第一版本来了：

```c
#include <stdio.h>
#define MOD 1000000007

void tribonacci(long long int l, long long int r){
    long long int x=1, y=1, z=1, tmp;
    long long int result=0;
    long long int i;

    if (l < 3){
        for(i = l; i <= r && i < 3; i++)
            result += 1;
    }
    for(i = 3; i <= r; i++){ // the i number is z
        tmp = x;
        x = y;
        y = z;
        z = (tmp + x + y) % MOD;
        if(i >= l)
            result = (result + z) % MOD;
    }
    printf("%lld\n", result);
    return;
}
```

恩，开始测试（就别纠结同数量级下 clock 的大小了，我也不知道为啥前三个会递减）。

---
r | l | result | clock
--- | --- | --- | ---
0 | 10 | 423 | 19
0 | 100 | 440169199 | 12
0 | 1000 | 397128969 | 10
0 | 10000 | 749926090 | 73
0 | 100000 | 142414638 | 619
0 | 1000000 | 817893089 | 5502
0 | 10000000 | 628546478 | 55406
0 | 100000000 | 455282242 | 540773
0 | 1000000000 | 38716501 | 5394326
0 | 10000000000 | 906890177 | 55216246
---

到 10 个 0 的时候就有点慢了，但是，给的最大的数据量可是会有 18 个 0 ……我要好好鄙视下[@石老板](http://shiguangyin.xyz/)，做出来了也不告诉我关键词。还好有[@我荇](http://nanf.me/)。

我不会告诉你使用矩阵快速幂后差距会这么大：

---
r | l | result | clock
--- | --- | --- | ---
0 | 10 | 423 | 167
0 | 100 | 440169199 | 24
0 | 1000 | 397128969 | 47
0 | 10000 | 749926090 | 72
0 | 100000 | 142414638 | 77
0 | 1000000 | 817893089 | 94
0 | 10000000 | 628546478 | 92
0 | 100000000 | 455282242 | 117
0 | 1000000000 | 38716501 | 119
0 | 10000000000 | 906890177 | 155

```c
#include <stdio.h>
#define MOD 1000000007

void tribonacci(long long int l, long long int r){
    long long int x=1, y=1, z=1, tmp;
    long long int result=0;
    long long int i;


    if (l < 3){
        for(i = l; i <= r && i < 3; i++)
            result += 1;
    }
    else{
        before_start_sum = 0;
        if(l < 3){
            for(i=0; i < l; i++)
                before_start_sum +=1;
        }
        else{
            before_start_sum = fast_matrices(l-3); // frequency == l-3
        }
        end_sum = fast_matrices(r-2); // frequency == r-2
        result = end_sum - before_start_sum;
        if(result < 0)
            result = (end_sum + MOD) - before_start_sum;
    }
    printf("%lld\n", result);
    return;
}

long long int fast_matrices(long long int frequency){
    martix result = init(3);
    martix factor = init(2);
    long long int ans;

    while(frequency){
        if(frequency & 1){ // frequency %= 2
            martix_mul(factor, result);
        }
        martix_mul(factor, factor);
        frequency >>= 1; // frequency /= 2
    }
    martix_des(factor);
    ans = result->array[0][0];
    martix_des(result);
    return ans;
}
```

还有个小坑，体现在`if(result < 0) result = (end_sum + MOD) - before_start_sum;`。再次鄙视一下[@石老板](http://shiguangyin.xyz/)——因我意识到错误了 - debug 时候偶然输入`35 35`测试 - 后寻问他，他给我了个错误的答案 =_=

能看到这估计也不容易，在这里说说一些思考的过程以及其他的解法：

先是推导出了一个公式：
{% raw %}
$$
S(n+2) = 2 * \sum_{i=1}^{n-1} A_i + A_0 + A_2 + A_n
$$
{% endraw %}
但是不能用，因为得用到除法……

[@石老板](http://shiguangyin.xyz/)的思路：

从第六项开始，S(n)也是一个 Tribonacci 数列……

附录：

测试代码

```c
#include <stdio.h>
#include <time.h>

// clock() 返回从“开启这个程序进程”到“程序中调用 clock() 函数”时之间的CPU时钟计时单元（clock tick）数

int main(void){
    int i;
    long long int l, r, start, end;
    for(l=0, i=r=1; i < 10; i++){
        r *= 10;
        start = clock()
        tribonacci(l, r);
        end = clock();
        printf("end-start: %lld\n", end-start);
    }
    return 0;
}
```

矩阵相关

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    long long int **array;
    int params[2]; // martix's shape
} martix_, *martix;

// initialization
martix init(int flag){
    long long int
        ones[4][4] = {
            {1, 0, 0, 0},
            {0, 1, 0, 0},
            {0, 0, 1, 0},
            {0, 0, 0, 1}
        },
         factor[4][4] = {
             {1, 1, 1, 1},
             {0, 1, 1, 1},
             {0, 1, 0, 0},
             {0, 0, 1, 0}
         },
         first[4][1] = {
             {3},
             {1},
             {1},
             {1}
         };

    martix array = (martix)malloc(sizeof(martix_));
    if(array == NULL)
        exit(-1);
    array->params[2] = {4};
    switch(flag){
        case 1:
        case 2:
            array->params[1] = 4;
            break;
        case 3:
            array->params[1] = 1;
            break;
    }
    array->array = (long long int **)malloc(array->params[0] * sizeof(long long int *));
    if (NULL == array->array)
        exit(-1);
    int i, j;
    for(i = 0; i < array->params[0]; i++){
        array->array[i] = (long long int *)malloc(array->params[1] * sizeof(long long int));
        if(array->array[i] == NULL)
            exit(-1);
        for(j = 0; j < array->params[1]; j++){
            switch(flag){
                case 1:
                    array->array[i][j] = ones[i][j];
                    break;
                case 2:
                    array->array[i][j] = factor[i][j];
                    break;
                case 3:
                    array->array[i][j] = first[i][j];
                    break;
            }
        }
    }
    return array;
}

// y = (x + y) % MOD = ((x % MOD) + (y % MOD)) % MOD
void martix_sum(martix x, martix y){
    int i, j;
    for(i = 0; i < x->params[0]; i++)
        for(j = 0; j < x->params[1]; j++)
            y->array[i][j] = (x->array[i][j] + y->array[i][j]) % MOD;
    return;
}

// y = (x * y) % MOD = ((x % MOD) * (y % MOD)) % MOD
void martix_mul(martix x, martix y){
    long long int tmp[x->params[0]][y->params[1]];
    int i, j, k;
    for(i = 0; i < x->params[0]; i++)
        for(j = 0; j < y->params[1]; j++){
            tmp[i][j] = 0;
            for(k = 0; k < x->params[1]; k++){
                tmp[i][j] += (x->array[i][k] * y->array[k][j]) % MOD;
            }
            tmp[i][j] %= MOD;
        }
    for(i = 0; i < x->params[0]; i++)
        for(j = 0; j < y->params[1]; j++){
            y->array[i][j] = tmp[i][j];
        }
    y->params[0] = x->params[0];
    return;
}

// destroy
void martix_des(martix x){
    int i;
    for(i = 0; i < x->params[0]; i++)
        free(x->array[i]);
    free(x->array);
    free(x);
    return;
}
```
