---
title: strncat溢出杂谈
date: 2019-09-30 17:14:50
tags: ['c', 'string']
---

## Step 1

遇到一个[问题](/others/2019/buffer-overflow-by-strncat.log)，在Debian上安装了libc-bin-dbgsym，有以下的日志打印到终端中：

```log
*** buffer overflow detected ***: lxcfs terminated
======= Backtrace: =========
/lib/libc.so.6.1(+0x8cb88)[0x2000611cb88]
/lib/libc.so.6.1(__fortify_fail+0x64)[0x200061c19d4]
/lib/libc.so.6.1(__chk_fail+0x24)[0x200061be864]
/lib/libc.so.6.1(__strncat_chk+0x48)[0x200061bd678]
/usr/lib/sw_64-linux-gnu/lxcfs/liblxcfs.so(dynmem_task+0x868)[0x20004832cb8]
/lib/libpthread.so.0(+0x80fc)[0x2000776a0fc]
/lib/libc.so.6.1(+0x119864)[0x200061a9864]
...
fish: “lxcfs -l -m docker /var/lib/lxc…” terminated by signal SIGABRT (Abort)
```

看得出是由`strncat`导致的，于此有关的[代码](https://gist.github.com/time-river/4cff4fb3b59a8d82dfc3043483084b94)类似这样：

```c
int main(int argc, char *argv[]) {
	char base64[64], buf[16];

	while (true) {
        ...
		strncat(buf, base64, 12);
        ...
	}

	return 0;
}
```

原因是我把`strcat(s1, s2)`当成`strcpy(dst, src)`用了。`strcat`是将两个字符串拼接，`strcpy`是将src拷贝至dst中。因为s1的大小不足以容纳`strlen(s1)+strlen(s2)`个字符，因此发生了溢出。

但溢出并非总是发生，AMD64 CPU / GCC 7.4.1 / GLIBC 2.26下有这这样的输出：

```log
./strncpy 
time: 1569841068 sha256(long): H8Zw/CnxF9d/kG10Ck7VfDMNqWADDGMsrD7Wx0wTshY=
sha256(short): H8Zw/CnxF9d/
time: 1569841069 sha256(long): uyzMNURXeyOjyCZ1ANI6Lze6lwF1bKUPjU3rHBZyTNY=
sha256(short): H8Zw/CnxF9d/uyzMNURXeyOj
time: 1569841070 sha256(long): 1zdqJgOCuILZkti92I2YUuT/KGNuqmSPOioA8rjxEeg=
sha256(short): H8Zw/CnxF9d/uyzM1zdqJgOCuILZkti92I2YUuT/KGNuqmSPOioA8rjxEeg=1zdqJgOCuILZ
time: 1569841071 sha256(long): Bvf5PF9cd0q01tccsNNvNtDjPMgUSWbMx2rNgLVFgEo=
sha256(short): H8Zw/CnxF9d/uyzMBvf5PF9cd0q01tccsNNvNtDjPMgUSWbMx2rNgLVFgEo=Bvf5PF9cd0q0
time: 1569841072 sha256(long): S/06FjYG/uc3FxAY/DQbEIxezMw+xGfEbg0i373utQs=
sha256(short): H8Zw/CnxF9d/uyzMS/06FjYG/uc3FxAY/DQbEIxezMw+xGfEbg0i373utQs=S/06FjYG/uc3
time: 1569841073 sha256(long): miGQZLfdfwozwdkunw1RUA+v+eqlx6DjO7hryjuIcQc=
sha256(short): H8Zw/CnxF9d/uyzMmiGQZLfdfwozwdkunw1RUA+v+eqlx6DjO7hryjuIcQc=miGQZLfdfwoz
```

这是因为`base64`与`buf`的地址空间连续，并且`base64`即`&buf[16]`，因此才会有第三次输出之后，sha256(short)的前16个字节相同的情况。

[`strncat`](https://github.com/bminor/glibc/blob/glibc-2.28/string/strncat.c#L27)的行为类似：

```c
char*
strncat(char *dest, const char *src, size_t n)
{
    size_t dest_len = strlen(dest);
    size_t i;

   for (i = 0 ; i < n && src[i] != '\0' ; i++)
        dest[dest_len + i] = src[i];
    dest[dest_len + i] = '\0';

   return dest;
}
```

[`EVP_EncodeBlock`](https://www.openssl.org/docs/man1.1.0/man3/EVP_EncodeBlock.html)会为输出自动加上一个NUL，因此自第三次输出开始，`strncat`的作用是在`buf[16+strlen(base64)]`后面添加12个字符。

总结如下：

1. `strcat(dst, src)`是将字符串src拼接在dst后面
2. 因为`buf`地址在`base64`之前且相邻，所以`buf`溢出后会使用`base64`的空间，不一定导致buffer overflow
3. 因`EVP_EncodeBlock`的自动添加NULL行为，`strncat`的行为致使第三行起在`base64[44]`起添加12个字节与NUL

## Step 2

字符串长度的定义，有两种：1. NUL(`'\0'`)结尾；2. 带有长度标识。C采用的是第一种。

C字符处理API中代有`n`的大都是为了解决缓冲器溢出(buffer overflow)问题而提出。但[`strncpy`](http://man7.org/linux/man-pages/man3/strncpy.3p.html)与[`strncat`](http://man7.org/linux/man-pages/man3/strcat.3.html)并非以NUL截止。它们的手册页]这样写着：

> strncpy:
>   Warning: If there is no null byte among the first n bytes of src the
>   string placed in dest will not be null-terminated.
>
> strncat:
>   src does not need to be null-terminated if it contains n or more bytes.

[Why does strncpy not null terminate?][1]是个在StackOver Flow中提到的问题。[LWN: The ups and downs of strlcpy()](https://lwn.net/Articles/507319/)提到，BSD派生的`strlcpy`与`strlcat`保证了NUL的存在，但因为截断数据可能导致安全性问题，它并没有进入glibc，并且`gcc -D_FORTIFY_SOURCE`能够捕获大部分`strlcpy`与`strlcat`意在解决的问题。

Note: `gcc -D_FORTIFY_SOURCE`也是导致产生SIGABRT的原因之一。

## Step 3

[`memcpy`](http://man7.org/linux/man-pages/man3/memcpy.3.html)与[`memmove`](http://man7.org/linux/man-pages/man3/memmove.3.html)也是[string.h](http://man7.org/linux/man-pages/man3/string.3.html)的API之二。

9月3日面试的时候，面试官提到了`memcpy`无法用在dst与src内存区域重叠(overlap)的场景下，指出了`memmove`便意在解决该问题。

dst与src的内存区域分为三种情况：

1. dst与src未发生重叠
2. dst的末端与src的起始发生重叠
3. dst的起始与src的末端发生重叠

据此，[`memmove`](https://github.com/bminor/glibc/blob/glibc-2.28/string/memmove.c)的实现思路如下：

```c
void *memmove(void *dst, const void *src, size_t n) {
    unsigned long int dstp = (long int) dst;
    unsigned long int srcp = (long int) src;

    if (dstp - srcp >= len) { /* *unsigned* compare ! */
    /* 符合情况1与2 */
        memcpy(dest, src, n);
    } else {
    /* 负荷情况3 */
        for (size_t i=n-1; i>=0; i--)
            ((char *)dst)[i] = ((char *)src)[i];
    }
    return dst;
}
```

### 类型转换

关于`dstp - srcp >= len`为什么符合情况1与2，参考demo:

```c
int main(int argc, char *argv[]) {
	unsigned int a = 10, b = 20;
	size_t n = 5;

	printf("%s\n", a-b>=n ? "yes" : "no");
    printf("%s\n", a-b>=5 ? "yes" : "no");

	return 0;
}
```

这是因为C类型转换的原因：`size_t`在AMD64下大小为8字节，相当于`signed long int`，它与`unsigned int`进行运算，都会被转换成`unsigned long int`。因为`a-b`小于0，计算机使用补码表示数字。因此任意的由`signed long int`表示的负数转换成`unsigned long int`后，都会比`LONG_MAX`大。

## Reference

- [Why does strncpy not null terminate?][1]
- [C string handling][2]
- [LWN: The ups and downs of strlcpy()][3]
- [Linux C编程一站式学习: 类型转换][4]

[1]: https://stackoverflow.com/questions/1453876/why-does-strncpy-not-null-terminate
[2]: https://en.wikipedia.org/wiki/C_string_handling
[3]: https://lwn.net/Articles/507319/
[4]: https://akaedu.github.io/book/ch15s03.html