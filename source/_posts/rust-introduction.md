---
title: Rust介绍
date: 2018-03-31 16:22:51
tags: ['programming language', rust]
---

## rust version

Rust维护三个【发行频道（release channels）】：稳定版（stable），测试版（beta），开发版（nightly）。稳定版和测试版每六周更新一次，而在那时的开发版会变成新的测试版，测试版变为新的稳定版。标记为不稳定（unstable）或者隐藏在特性门控（feature gates）后的语言和标准库特性只能再开发版上使用，新特性最初会被标记为不稳定，一旦呗核心团队和相关的自团队批准的话就变成【通过门控的（ungated）】，这种方法允许实验性变更，并同时为稳定频道提供强有力的向后兼容保证。

并不能在测试或稳定频道上使用不稳定的功能。不稳定的特性意味着不能提供这种特性保证，不希望开发者依赖它。

【特性门控（feature gates）】是Rust用来稳定编译器、语言和标准库特性的机制。一个受【门控】的特性只能在开发版发布渠道使用，且必须显示指定`#[feature]`属性或者命令行参数`-Z unstable-options`。当一个特性稳定了，它才能在稳定版上使用，不需要显示启用。此时，这个特性被认为是通过门控（ungated）的/特性门控允许开发者在稳定版提供之前，在开发中测试实验性的功能。

## rustup

Rust编译器分为三个频道发布，不同频道之间的编译器切换是一个头疼的事情，有人做了切换工具multirust，之后被官方招安，改名为rustup并成为Rust编译器默认的安装途径。rustup在Rust支持的每一个平台上以一致的方式管理这些二进制构建，并且支持每一个发布频道。

## cargo

cargo是Rust的包管理器，用于代码的组织和管理，使用它可以自动解决项目的依赖和编译问题，还可以上传项目至Rust社区的*package registry*[crates.io](https://crates.io/)。

## insatall question

这里只谈论`stable-x86_64-pc-windows-msvc`版本。

### `error: toolchain 'stable-x86_64-pc-windows-msvc' does not ...`

使用官方推荐的[rustup-init.exe](https://win.rustup.rs/)安装完成后，运行`~/.cargo/bin`目录下的一些命令会出现`error: toolchain 'stable-x86_64-pc-windows-msvc' does not ...`这样的错误，比如下面这样：

```powershell
PS C:\Users\river\.cargo\bin> .\rust-gdb.exe
error: toolchain 'stable-x86_64-pc-windows-msvc' does not have the binary `rust-gdb.exe`
```

具体原因不太清楚。对于`rustc.exe, rustfmt.exe`之类，解决方法是使用`cargo`重新安装。但`rust-gdb.exe, rust-lldb.exe`除外，因为这俩分别依赖`gdb`与`lldb`。如果使用的是`*-windows-msvc`版本，因为ABI的兼容性问题，GDB没有办法debug MSVC编译出来的二进制文件。LLDB自2015年开始[支持MSVC debugging](http://blog.llvm.org/2015/01/lldb-is-coming-to-windows.html)，但我不知道怎么正确地编译出`rust-lldb.exe`。

### development in Visual Studio Code

其实吧，想在Visual Studio 2017中使用Rust，但[Rust](https://marketplace.visualstudio.com/items?itemName=DanielGriffen.Rust)目前只支持到Visual Studio 2017 Preview，所以暂时只能用vscode了。在配置开发环境的时候，我使用了[Rust (rls)](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust)来支持Rust语言，它依赖`rust-preview, rust-src, rust-analysis`，`cargo install`就好了。不过之后会出现一些问题，比如下面的：

```log
nightly toolchain not installed. Install?

RLS could not set RUST_SRC_PATH for Racer because it could not read the Rust sysroot.
```

如果没安装nightly版本，会报出这俩错误提示（其实是有点不明白为啥有第二个错误的），在*Preferences->Settings*里面更改一下设置就好了：

```json
{
    "rust-client.channel": "stable"
}
```

因为调试使用的是PDB，所以安装这个插件[C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)，为了能够通过点击行号添加断点，还需要在*Preferences->Settings*添加：

```json
{
    "debug.allowBreakpointsEverywhere": true
}
```

不过在调试的过程中会报错：

```log
Unable to open 'mod.rs': File not found (file:///c:/Uers/river/.rustup/toolchains/stable-x86_64-pc-windows-msvc/lib/rustlib/src/rust/src/libcore/fmt/mod.rs).
```

这个是`launch.json`参数中`sourceFileMap`参数的问题，加上类似下面的这行就好了(`"sourceFileMap": { <project directory>: <source directory> }`):

```json
{
    ...
    "configurations": [

        {
            ...
            "sourceFileMap": {
                "c:/projects/": "C:/Uers/river/.rustup/toolchains/stable-x86_64-pc-windows-msvc/lib/rustlib/src/"
    ...
```

## references

- [Debugging Rust programs on Windows with Visual Studio/WinDbg](https://internals.rust-lang.org/t/debugging-rust-programs-on-windows-with-visual-studio-windbg/3848/8)
- [Debug Rust on Windows with Visual Studio Code and the MSVC Debugger](http://www.brycevandyk.com/debug-rust-on-windows-with-visual-studio-code-and-the-msvc-debugger/)