---
title: Linux下文件描述符的统计方法
date: 2019-06-13 10:47:39
tags: [linux, storage, 'file descriptor']
categories: style
---

## Question

Unix-like OS中，存在一切皆文件的思想，文件描述符（file descriptor，fd）是一个用于文件访问的抽象化概念，实际上它是一个索引值，指向为每一个进程所维护的该进程打开文件的记录表。[\[1\]][1]因此文件描述符与文件操作记录存在一一对应的关系。

所以问题来了：如何统计一组会话进程中文件描述符的数量？

## Background

### VFS Overview

VFS（Virtual File System）在Linux内核中是一层软件抽象。它为用户态程序提供文件操作接口，也屏蔽了不同的文件系统之间的差异，提供了统一的操作方式。[\[2\]][2]

VFS中关键的数据结构有：

- The File Object `struct file`
- The Inode Object `struct inode`
- Directory Entry Cache (dcache) `struct dentry`
- The Address Space Object `struct address_space`
- The Superblock Object `struct super_block`

File object记录了进程打开文件的上下文信息，是内核中的文件描述符。结构体成员`struct file_operations`中实现的回调函数是暴露给用户态程序的接口，他们来自inode，由具体的文件系统实现。

Inode object保存了数据的metadata，记录在存储设备上，随着需要拷贝至内存、写回存储设备。Inode与文件是一一对应的关系。由于诸如hard links的存在，它与dentry是一对多的关系。成员`struct inode_operations`是与文件信息相关的inode管理操作。

Dcache提供了一种加速使用文件名（pathname）查找文件的机制。它把pathname hash成特定的dentry，进而与inode相关联。Dentry并不记录在存储设备上，也因为RAM的限制无法把所有的dentry放入内存中，因此dentry是动态生成的，先创建dentry再载入inode。成员`struct dentry_operations`是与此机制有关的回调函数。

Address space object用于管理page cache中的page，这些page采用基数树(ragix tree)的方式组织。成员`struct address_space_operations`是一组维护page cahce的函数指针。

> Note: adress space object中的基数树现在已经改成XArray（commit id: [eb797a8ee0ab4][6]），XArray是一种接口改变的基数树。相关的文章：[阿里云系统组技术博客: The Xarray Data Structure][4]，[LWN: The XArray data structure][5]。

Superblock object代表被mount的文件系统，是文件系统的实例。成员`struct super_operations`是与文件系统有关的操作，包括文件系统相关的inode管理操作。

他们的关系如下图：

![the relationship of file, dentry, inode and address space](/images/19/06/relationship-of-file-dentry-inode-address_space.png)

### about `struct file`

两图胜千言：

![the relationship of files_struct and fdtable in initialization](/images/19/06/fdtable-1.png)

![the relationship of files_struct and fdtable after expanding fd_array](/images/19/06/fdtable-2.png)

Note:

1. bitmap记录已用、可用的文件描述符，`max_fds`代表bit数，即当前可用的文件描述符
2. 当分配的文件描述符大于`max_fds`的时候，会使用`expand_files()`同时拓展bitmap与数组fdt
3. fdt记录与fd相关的的`struct file`指针，它是数组指针
4. fd与`struct file`是多对一的关系
5. 内核存在参数fs.file-max与file.file-nr，分别代表当前系统允许存在的最大文件描述符数量、当前系统的文件描述符使用情况[\[18\]][18][\[19\]][19]

### about `*_CLOEXEC` flag

这个flag在`open()`中为`O_CLOEXEC`，`fcntl()`中为`FD_CLOEXEC / F_DUPFD_CLOEXEC`……总之后缀都是`CLOEXEC`。

fork后，子进程会获得父进程的文件描述符（更具体地说，Linux下是未设置`CLONE_FILES` flag的系统调用`clone()`）。子进程执行exec的后，内核不再保留原有的fdtable，无法再根据文件描述符操作文件。有时候exec前逐一清理文件描述符并不方便，因此出现了这类`*_CLOEXEC` flag：在执行exec的时候这个文件描述符关闭。

## Solution

### exploration

有五类行为会导致文件描述符数量的变化：

1. open类系统调用令文件描述符数量加一，比如`open()`
2. close系统调用关闭特定的文件描述符
3. 引发dup fd行为的系统调用会复制特定的文件描述符，比如`fork() / dup()`
4. 设置了`*_CLOEXEC`行为的exec家族
5. exit系统调用清理进程

对于open类，会发使用`__alloc_fd()`来分配未经使用的文件描述符，close文件描述符的时候调用`__put_unused_fd()`。[分配的规则][8]与[关闭文件描述符时的行为][9]部分代码如下：

```c
/**
 *  int __alloc_fd(struct files_struct *files,
 *                 unsigned start, unsigned end, unsigned flags)
 */
fdt = files_fdtable(files);
fd = start;
if (fd < files->next_fd)
    fd = files->next_fd;
if (fd < fdt->max_fds)
    fd = find_next_fd(fdt, fd);

/**
 *  static void __put_unused_fd(struct files_struct *files, unsigned int fd)
 */
if (fd < files->next_fd)
    files->next_fd = fd;
```

可看到，文件描述符总先尝试从`next_fd`分配，关闭文件描述符的时候会及时更新`next_fd`。但因为由`next_fd`得到的文件描述符并非一定未被使用（`dup`系列系统调用的结果，或比如遇到连续打开10个文件描述符，关闭第一次打开的文件描述符的情况），因此有了函数`find_next_fd()`。同时，`fd`不小于`fdt->max_fds`意味着fd table需要拓展了。

至于第一个if判断，是因为`fork()`与`fcntl()`的原因：`fork()`时会`dup_fd()`，此过程中有[一行代码][12]将`next_fd`置0了；`fcntl()`的dup fd[行为][14]则是分配不小于arg（即`f_dupfd()`参数`from`）大小的文件描述符。注意的是，`fork()`过程中统计fd的数量是根据fd_array来的。

```c
/**
 *  struct files_struct *dup_fd(struct files_struct *oldf, int *errorp)
 */
newf->next_fd = 0;
...
old_fds = old_fdt->fd;
new_fds = new_fdt->fd;
for (i = open_files; i != 0; i--) {
    struct file *f = *old_fds++;
    if (f) {
        get_file(f);
    } else {
        /*
         * The fd may be claimed in the fd bitmap but not yet
         * instantiated in the files array if a sibling thread
         * is partway through open().  So make sure that this
         * fd is available to the new process.
         */
        __clear_open_fd(open_files - i, new_fdt);
    }
    rcu_assign_pointer(*new_fds++, f);
}

/**
 *  int f_dupfd(unsigned int from, struct file *file, unsigned flags)
 */
err = alloc_fd(from, flags);
```

dup系列存在三种：`dup() / dup2() / dup3()`。`dup()`返回新的文件描述符，期间会使用`__alloc_fd()`申请未使用的文件描述符。`dup2()`会指定新描述符的值，同时它的`FD_CLOEXEC`文件描述符标志被清除，与`dup`的区别在于它可以将`close()`与`dup()`的操作合二为一（也就是说，`dup2()`的参数`newfd`可以是已经打开了的文件描述符）。`dup3()`与`dup2()`的区别在于可以置位`FD_CLOEXEC`。[10]调用`do_dup2()`的函数还包括`replace_fd()`。

对于flag`*_CLOEXEC`，在执行exec系列的时候，会有[`do_close_on_exec()`][15]行为，它最终也会调用`__put_unused_fd()`。

还有，诸如`exit()`类系统调用并不会逐个关闭文件描述符，它会通过执行`exit_files()`从而[`put_files_struct()`][13]。因为线程的存在，多个线程会共享同一个文件描述符，所以需要添加判断；若引用次数为零则直接释放`fdt`。

```c
void put_files_struct(struct files_struct *files)
{
    if (atomic_dec_and_test(&files->count)) {
        struct fdtable *fdt = close_files(files);
        /* free the arrays if they are not embedded */
        if (fdt != &files->fdtab)
            __free_fdtable(fdt);
        kmem_cache_free(files_cachep, files);
    }
}
```

### answer

因此，统计文件描述符变更的hook点：

1. `__alloc_fd()`中（针对`open()`类）
2. `__put_unused_fd()`内（针对`close()`）
3. `dup_fd()`中为有效的`struct file`增加引用次数时（针对`fork()`类）
4. `do_dup2()`的时候（针对`dup2() / dup3()`）
5. `close_files()`时（针对`exit()`）

## Reference

1. [Wikipedia: file descriptor][1]
2. [Linux doc: vfs.txt][2]
3. [The dentry Cache][3]
4. [阿里云系统组技术博客: The Aarray Data Structure][4]
5. [LWN.net: The XArray data structure][5]
6. [linux git commit: page cache: Rearrange address_space][6]
7. [VFS中的file，dentry和inode][7]
8. [知乎专栏: Linux 内核文件描述符表的演变][8]
9. [CSDN: struct files_struct和struct fdtable][9]
10. [UNIX环境高级编程][16]
11. [Linux环境编程——从应用到内核][17]
12. [Linux doc: fs.txt][18]
13. [Linux Interview Questions : Open Files / Open File Descriptors][19]

[1]: https://en.wikipedia.org/wiki/File_descriptor
[2]: https://www.kernel.org/doc/Documentation/filesystems/vfs.txt
[3]: https://www.halolinux.us/kernel-reference/the-dentry-cache.html
[4]: https://webcache.googleusercontent.com/search?q=cache:JujZbkQC2B8J:https://kernel.taobao.org/2018/05/The-XArray-data-structure/+
[5]: https://lwn.net/Articles/745073/
[6]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb797a8ee0ab4
[7]: https://bean-li.github.io/vfs-inode-dentry/
[8]: https://zhuanlan.zhihu.com/p/34280875
[9]: https://blog.csdn.net/metersun/article/details/80513702
[10]: https://elixir.bootlin.com/linux/v5.0/source/fs/file.c#L488
[11]: https://elixir.bootlin.com/linux/v5.0/source/fs/file.c#L552
[12]: https://elixir.bootlin.com/linux/v5.0/source/fs/file.c#L289
[13]: https://elixir.bootlin.com/linux/v5.0/source/fs/file.c#L413
[14]: https://elixir.bootlin.com/linux/v5.0/source/fs/file.c#L982
[15]: https://elixir.bootlin.com/linux/v5.0/source/fs/exec.c#L1303
[16]: https://book.douban.com/subject/25900403/
[17]: https://book.douban.com/subject/26820213/
[18]: https://www.kernel.org/doc/Documentation/sysctl/fs.txt
[19]: https://www.thegeekdiary.com/linux-interview-questions-open-files-open-file-descriptors/
