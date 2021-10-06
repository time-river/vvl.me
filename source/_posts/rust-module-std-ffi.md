---
title: Rust Module std::ffi 1.0.0
date: 2018-03-31 23:04:44
tags:  ['language', 'Rust', 'std', 'translation'] 
---

Module std::ffi

> 与FFI绑定的有关utilities。

这个模块提供了一些utilities，用于帮助操作非Rust接口的数据，比如不同的编程语言或者操作系统底层接口。它主要用于FFI（Foreign Function Interface）bindings或代码中需要与其他类C语言的字符串进行数据交换的地方。

## Overview

Rust的原生字符串是`String`类型，或者使用`str`原语的slices。两者都是UTF-8编码，可能在中间含有nul字节，比如由字节组成的字符串，这些字节中可能有`\0`。无论是`String`还是`str`都存储了字符串的长度；字符串并没有像C一样以nul结束。

C字符串与Rust的相比，有以下不同：

* __编码（Encodings）__ —— Rust的字符串是UTF-8编码的，但C可能使用其他的编码格式。如果使用来自C的字符串，你应该检查它的编码类型，而非假设它使用UTF-8。

* __字符大小（Character size）__ —— C字符串的字符长度是`char`或`wchat_t`；但切记C的`chat`与Rust的不同。标准C没有规定这些字符串的大小怎么确定，但为不同的字符类型定义了不同的API。Rust字符串总是UTF-8编码的，因此不同的Unicode字符可能会被编码成不同的字节长度。Rust中的`chat`类型代表了'Unicode scalar value'，它类似于'Unicode code point'，但不完全一样。

* __Null terminators and implicit string lengths__ —— C字符串以nul结束，`\0`终止符总是位于字符串的最后。字符串的长度并不存起来，而是由计算得知；为了得知字符串的长度，C代码总要为`chat`类型的字符串手动地调用`strlen()`，或者通过`wcslen()`计算`wchat_t`类型的字符串长度。这些函数返回nul终止符前的字符数量，真实的字符串长度是`len+1`。Rust字符串不含有nul终止符；字符串长度总是被存起来，而非由计算得到。因此Rust访问字符串长度的时间复杂度是$O(1)$（因为长度被存了起来）；C语言是$O(length)$，因为需要扫描nul终止符之前的整个字符串。

* __字符串中间的nul字符（Internal nul characters）__ —— 当C字符串中含有nul中止字符时，这通常意味着它不能存在于字符串中间——一个nul字符意味着字符串的分割。Rust中nul字符*可以*位于字符串中间，因为nul并不标志着字符串的结束。

## Representations of non-Rust strings

如果你需要转换UTF-8编码的字符串，或者通过C ABI得到字符串，`CString`和`CStr`在此时非常有用。

* __From Rust to C__：`CString`是C类型的字符串：以nul终止符结束，字符串中间没有nul字符。Rust代码能创建`CString`的字符串（字符串中间没有nul字符），可以使用他们通过多种方法来获取原始的`*mut u8`，这种类型的变量可以如同C语言的字符串一样，被使用这种类型作为参数的函数使用。

* __From C to Rust__：`CStr`代表了被借用的C字符串类型；他是一种封装了`*const u8`、从C函数中获得的数据。`CStr`保证了它是以nul终止符结束的字节数组。如果`CStr`是有效的UTF-8编码类型的字符串，那么它便可以被转换成Rust的`&str`，或者在转换的时候添加了一些替代字符。

当你需要在操作系统本身与Rust之间转换字符串的时候，或者捕捉外部命令的输出时，`OsString`和`OsStr`十分有用。在`OsString`，`OsStr`与Rust字符串之间的转换与`CString`和`CStr`很像。

* `OsString`代表着操作系统默认的字符串类型。Rust标准库中，存在着多种在操作系统与Rust字符串之间转换的API，他们使用`OsString`而非纯文本字符串（plain string）。比如，`env::var_os()`用于确认环境变量；它返回的是`Option<OsString>`。如果环境变量存在则会得到`Some(os_string)`，之后把它转换成Rust字符串。它会产生`Result<>`，因此如果环境变量不包含有效的Unicode数据的话，你的代码可以检测到错误。

* `OsStr`代表了一种可以传递给操作系统的字符串借用。它可以转换成Rust UTF-8编码类型的slice，方法与转换成`OsString`差不多。

## Conversions

### On Unix

Unix上，`OsStr`由`std::os::unix::ffi::OsStrExt`trait实现，存在两种方法：`from_bytes`和`as_bytes`。这为与在UTF-8字节的slices之间转换提供了方便。

此外，Unix系统上`OsString`实现了`std::os::unix::ffi::SsStringExt`trait，提供了`from_vec`和`into_vec`方法，使参数转换成`u8`类型的向量（consume their arguments, and take or produce vectors of `u8`）。

### On Windows

Windows上，`OsStr`实现了`std::os::windows::ffi::OsStrExt`trait，提供了`encode_wide`方法。它提供了以中国迭代器，可以把字符串`collect`为`u16`的向量。

此外，Windows的`OsString`实现了`std::os::windows::ffi::OsStringExt`trait，提供了`from_wide`方法。此方法的结果是`OsString`，可以无损地转换成Windows字符串。

### Structs

...

### Original

[Module std::ffi 1.0.0](https://doc.rust-lang.org/std/ffi/)
