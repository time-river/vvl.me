---
title: linux下的访问控制之credentials
date: 2018-09-15 19:34:49
tags: ['linux', 'credentials']
---

## Overview
Credentials在Linux中用于访问控制（Access Control），基于*uid*、*gid*、*sid*，是Linux几种安全措施的一部分。同时，仅用于进程（task）中的Capabilities提供了更细化的权限控制机制。[1][1]

Note[2][2]：

* *uid* —— User Identifier，用户标识符，用于辨识用户。又分为*euid*、*ruid*、*suid*、*fsuid*。
  - *ruid* —— Real UID，真实用户ID，一般称之为*uid*。
  - *euid* —— Effective UID，有效用户ID。
  - *suid* —— Saved UID，暂存用户ID。
  - *fsuid* —— File System UID，文件系统用户ID。
* *gid* —— Group Identifier，用户组标识符，用户辨识用户组。也又分为*rgid*、*egid*、*sgid*、*fsgid*。因每个用户必须是一个组的成员，主组（*the primary group*）由组数据库中用户条目的数字*gid*标识。

在v4.12中，与进程访问控制有关的系统调用有以下18个：

| | |
| --: | --: |
| `getuid` | `setuid` | 
| `getgid` | `setgid` |
| `geteuid` |`getegid` |
| `setreuid` | `setregid` |
| `getresuid` | `setresuid` | 
| `getresgid` | `setresgid` |
| `getgroups` | `setgroups` |
| `getfsuid` | `getfsgid` | 
| `capget` | `catset` |

## Introduction

一些概念[4][4]：

- *Real user ID / Real group ID*：这些ID决定该进程的所有者是谁。
- *Effective user ID / Effective group ID*：内核利用这些ID决定进程对共享资源拥有怎样的访问权，比如：消息队列、共享内存和信号量。尽管大多数的UNIX系统使用这些ID决定文件的访问权，但Linux使用的是独有的*filesystem ID*。
- *Saved set-user-ID / Saved set-group-ID*：这两个ID在*set-user-ID*与*set-group-ID*程序执行后，保存相应的*effective ID*。因此，__一个*set-user-ID*程序的*effective user ID*可以在*real user ID*与*saved set-user-ID*之间来回切换，从而可以恢复/抛弃特权__。
- *Filesystem user ID / Filesystem group ID*：这些ID用于决定进程对文件与其他共享资源的访问权。进程无论何时更改*effective user/group ID*，内核也同时更改*filesystem user/group ID*。
- *Supplementary group IDs*：它是一组额外的group IDs，也用于文件、共享资源的访问控制。

Note[5][5][6][6][7][7]：
*Set-user-id / Set-group-id*区别于进程中的*saved set-user-ID / saved set-group-ID*，是文件上的概念。设置一个*Saved set-user-ID*的意义在于，在`execv`可执行文件之后，__如果可执行文件的*set-user-ID*位被设置了，进程的*effective user ID, saved set-user-ID*会设置成可执行文件所有者的*uid*__；*effective group ID*也有类似的操作。下面是内核中与此有关的具体代码：

```c
/*
 * fs/exec.c
 */
prepare_binprm
 - bprm_fill_uid
    if (mode & S_ISUID) {
      bprm->per_clear |= PER_CLEAR_ON_SETID;
        bprm->cred->euid = uid;
    }

    if ((mode & (S_ISGID | S_IXGRP)) == (S_ISGID | S_IXGRP)) {
        bprm->per_clear |= PER_CLEAR_ON_SETID;
        bprm->cred->egid = gid;
    }
 - security_bprm_set_creds
    new->suid = new->fsuid = new->euid;
    new->sgid = new->fsgid = new->egid;
```

> 举例说明：用户zzz执行了文件a.out，a.out的属主为hzzz且设置了set-user-ID位。现在本进程的real uid为zzz，effective uid = saved uid = hzzz。
>
> 进程执行了一会之后，突然想用zzz的权限访问一个文件，于是进程可能会调用setuid（zzz）， 此时检测进程的权限，进程的effective uid是hzzz，不是root，所以不能更改real uid（只有root才能更改real uid），所以只能设置effective uid，发现effective uid可以被设置为zzz（因为real uid是zzz），所以函数调用成功，只将effective uid设置成zzz。
>
> 现在进程访问完zzz的文件了，又想回到hzzz的环境中执行，所以有可能会调用setuid（hzzz），这次saved uid的作用就表现出来了，因为刚刚只是改变了effective uid, 而saved uid还保存着之前的effective uid，所以可以调用setuid（hzzz）来要回原来的权限。

描述*uid/gid*转换的有限状态自动机（FSA）在[Proceedings of the 11th USENIX Security Symposium][8]中的第10-12页有展示。

**关于`setuid`可以简单记为：在没有特权的情况下，`euid / fsuid`可以通过`setuid`设置成`ruid`或`suid`。**

## Pre-internal

### User namespaces

A new approach to user namespaces[3][3]梗概：

容器可以被看做一种轻量级的虚拟化技术。因为与宿主机共享内核，所以比真正的虚拟机运行效率更高。但必须提供一种机制，把全局可见的资源封装进命名空间中，对容器展现只属于自己的那部分资源（比如进程ID、文件系统、网络接口）的视图。

用户命名空间（user namespaces）可以被认为是user/group ID以及相关权限的封装，它允许容器的所有者进程（不一定为root用户）在容器内部以root的身份运行，同时将容器内用户与系统的其余部分隔离。那么，同一个进程怎样才能在不同的上下文中有不同的*uid*呢？

Eric Biederman提交了一组patch解决了这个问题。这组patch中定义了两种新的数据类型`kuid_t`（Kernel UID）与`kgid_t`（Kernel GID）。Kernel UID用于描述进程在内核中的身份，而不管它在容器中可能采用的任何*uid*；它是用于大多数特权检查的值；并且进程并没有办法知道它的值。

为了kernel ID与user ID、group ID在不同的命名空间中做转换，除了知道Kernel ID之外，还需要知道特定的namespace ID，因此有了下面的一组用于*uid*的转换函数（*gid*同样存在）：

```c
    kuid_t make_kuid(struct user_namespace *from, uid_t uid);
    uid_t from_kuid(struct user_namespace *to, kuid_t kuid);
    uid_t from_kuid_munged(struct user_namespace *to, kuid_t kuid);
    bool kuid_has_mapping(struct user_namespace *ns, kuid_t uid);
```

在Kernel与user ID、group ID之间建立映射是一种特权操作，需要`CAP_SETUID, CAP_SETGID`标志。

## Internal

### `set*uid`

#### `setuid(uid)`

```md
kuid <- make kuid using uid and namespace
if process has CAP_SETUID priviledge; then:
    uid = euid = suid = fsuid = kuid
else if kuid == old_uid || kuid == old_suid; then:
    fsuid = euid = kuid
else:
    goto error
```

#### `setreuid(ruid, euid)`

```md
kruid <- make kruid using ruid and namespace
keuid <- make keuid using euid and namespace
if kruid != old_uid && kruid != old_euid 
     && keuid != old_uid && keuid != old_euid && keuid != old_suid
     && process has no CAP_SETUID priviledge; then:
    uid = kruid
    euid = suid = fsuid = keuid
else:
    goto error
```

#### `setresuid(ruid, euid, suid)`

```md
kruid <- make kruid using ruid and namespace
keuid <- make keuid using euid and namespace
ksuid <- make ksuid using ruid and namespace
if process has no CAP_SETUID
     && kruid != old_uid && kruid != old_euid && kruid != old_suid
     && keuid != old_uid && keuid != old_euid && keuid != old_suid
     && ksuid != old_uid && ksuid != old_suid && ksuid != old_suid; then:
    goto error
else:
    uid = kruid
    euid = fsuid = keuid
    suid = ksuid
```

### `set*gid`

the same as `set*uid`.

## Reference

- [Linux doc: Credentials in Linux#task-credentials][1]
- [Wikipedia: 用户ID][2]
- [LWN.net: A new approach to user namespaces][3]
- [Linux Programmer's Manual: credentials - process identifiers][4]
- [setuid和seteuid][5]
- [set-user-id (suid), set-group-id (sgid), saved-suid 筆記][6]
- [深刻理解——real user id, effective user id, saved user id in Linux][7]
- [Proceedings of the 11th USENIX Security Symposium][8]

[1]: https://www.kernel.org/doc/html/v4.17/security/credentials.html#task-credentials "Linux doc: Credentials in Linux#task-credentials"
[2]: https://zh.wikipedia.org/wiki/%E7%94%A8%E6%88%B7ID "Wikipedia: 用户ID"
[3]: https://lwn.net/Articles/491310/ "LWN.net: A new approach to user namespaces"
[4]: http://man7.org/linux/man-pages/man7/credentials.7.html "Linux Programmer's Manual: credentials - process identifiers"
[5]: https://lengzzz.com/note/archive-20140117 "setuid和seteuid"
[6]: https://kenlosolid.blogspot.com/2010/11/set-user-id-suid-set-group-id-sgid.html "set-user-id (suid), set-group-id (sgid), saved-suid 筆記"
[7]: https://blog.csdn.net/fmeng23/article/details/23115989 "深刻理解——real user id, effective user id, saved user id in Linux"
[8]: https://www.usenix.org/legacy/event/sec02/full_papers/chen/chen.pdf "Proceedings of the 11th USENIX Security Symposium"