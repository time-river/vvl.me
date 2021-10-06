---
title: x86架构下Linux image的组成、早期启动、kexec重启等
date: 2021-03-17 23:37:35
tags: ["x86", "linux"]
---

## Background

这段时间的工作涉及x86 CPU的早期启动，参与了几个与此有关的bugfix，涉及到的问题：

1. 在没有日志信息的情况下，怎么利用寄存器的状态确定CPU运行到了哪种状态
2. smp init / shutdown的细节
3. `INIT, SIPI`对CPU的影响
4. kexec-tools载入image时的内存布局
5. image early bootstrap时的内存布局

## construction of AMD64 Linux image

64—bit的x86 Linux，编译会生成这几个文件：

```md
- vmlinux
- arch/x86/boot/compressed/{vmlinux.bin, vmlinux.bin.gz, vmlinux}
- arch/x86/boot/{setup.elf, setup.bin, vmlinux.bin, bzImage}
```

64-bit的x86 CPU因历史原因，从8086 16-bit的real mode开始启动，CPU复位后执行的第一行代码亦在16-bit的real mode，通过一系列步骤，从real mode转换成protected mode，再进入32-bit mode，最后进入64-bit的long mode。

位于源码目录下的**vmlinux**，是含有elf header、debuginfo等信息的运行着的kernel的原始文件，它运行在long mode，它由除了arch/x86/boot目录下的x86内核代码编译生成。基于它使用objcopy能够生成**arch/x86/boot/compressed/vmlinux.bin**，再经过压缩生成**arch/x86/boot/compressed/vmlinux.bin.gz**，使用mkpiggy生成piggy.S。它的入口在**arch/x86/kernel/head_64.S**中的`startup_64`。

**arch/x86/boot/compressed/**目录下的相关代码是将CPU从32-bit的protected mode转换成64-bit的long mode，在这里将会启用分页(paging)机制、解压真正的64-bit内核代码，内核代码的地址随机化(kaslr)也会在这里完成。它的入口在**arch/x86/boot/compressed/head_64.S**中的`startup_32`。该目录下的一系列文件(含有piggy.S)，会被编译链接成**arch/x86/boot/compressed/vmlinux**。

**arch/x86/boot/**目录下的代码将CPU从real mode转换成32-bit的protected mode。它的起始地址在**arch/x86/boot/header.S**中的`_start`。这里做了些早期资源的探测与初始化，比如内存`detect_memory`、显示`set_video`等，这些工作集中在在main.c文件中的`main()`。该目录下的文件被编译生成**arch/x86/boot/{setup.elf, setup.bin}**，其中setup.bin是由setup.elf经objcopy得到。

> Note: `$cat <dir>/.<file>.cmd`可以看到<file>是怎样生成的，比如`$ cat arch/x86/boot/.bzImage.cmd`

## early boot: process, status and debug option

### boot process

具体来讲，kernel的早期启动路径如下：

```md
# files in arch/x86/boot 
_start (header.S)
-> main (main.c)
-> protected_mode_jump (pmjump.S)
-> 0x66, 0xea __BOOT_CS:in_pm32 (pmjump.S, enter 32-bit protected mode)

# files in arch/x86/boot/compressed
-> startup_32 (head_64.S, enable paging)
-> startup_64 (head_64.S, enter long mode)
-> extract_kernel (misc.c, uncompress kernel to `LOAD_PHYSICAL_ADDR`, default 0x1000000)

# files in arch/x86/kernel
-> startup_64 (head_64.S)
-> secondary_startup_64 (head_64.S)
-> initial_code (head_64.S, here is `x86_64_start_kernel`)
-> x86_64_start_kernel (head64.c)
```

> Note:
> real mode至32-bit protected mode的跳转代码这样写的：
>
> ```asm
>          # Transition to 32-bit mode
>         .byte   0x66, 0xea              # ljmpl opcode
> 2:      .long   in_pm32                 # offset
>         .word   __BOOT_CS               # segment
> ```
>
> 这是一个PE = 0的far jump，查阅Intel手册Volume 2 3.2中有关JMP指令的描述知道，在跳转过程中会将`%CS`置`__BOOT_CS`(`0x10`)，因小端序，所以`__BOOT_CS`放在了`in_pm32`的后面，这三行相当于`jmp cs:ip`

### boot status

Linux对x86规定了启动协议[[1]]，规定了早期启动时的寄存器状态与内存布局。`struct setup_header`定义了与内核启动有关的信息，其作为setup.elf一部分嵌在内核image中，位于`_start`靠前的512个字节内，在载入内核前由grub等loader动态地修改其值。在16-bit模式下，成员`cmd_line_ptr`指定了内核启动参数cmdline的起始地址；`ramdisk_image`是initramfs的起始地址；`hdr.code32_start`是real mode跳转到protected mode代码的跳转地址，即`startup_32`的地址。在64-bit模式下，`cmd_line_ptr + ext_cmd_line_ptr<<32`组成cmdline的地址，`ramdisk_image + ext_ramdisk_image<<32`组成initramfs的起始地址，其中`ext_{cmd_line_ptr, ramdisk_image}`为`struct boot_params`的成员。

准确地说，`struct boot_params`中的地址是相对地址，以image的载入地址作为偏移的起点。启动协议规定了三类boot protocol：16-bit、32-bit、64-bit。对于UEFI等现代BIOS，内核可以直接从32-bit模式启动，`startup_32`指定了内核的起始地址，为`0x0`；64-bit的BIOS能够直接启动long mode内核，起始地址是`0x200`。无论32-bit还是64-bit模式的启动，`struct boot_params`的所有内容都被预先设置好；16-bit模式仅部分被要求设置，比如cmdline与initramfs。

> Note: 这里的`0x0, 0x200`应该是相对偏移，即若arch/x86/boot/compressed/vmlinux的代码段载入地址为`addr`，则`startup_32`地址为`addr+0x0`，`startup_64`地址为`addr+0x200`。

在内核的启动过程中，能够从寄存器的状态推断出当前运行的kernel处于哪种模式。x86的复位向量是`0xfffffff0`，指向ROM，执行到内核的real mode时，`%cs`由bootloader设置，`%ds, %es, %ss`的值与`%cs`相同，CR0寄存器的`PE`位(bit 0)为0。
32-bit的protected mode的`%cs`是`__BOOT_CS`(`0x10`)，对应的GDT descriptor(从0计数的第2个)`D/B`(bit 22)是1，`L`(bit 21)是0，CR0中的`PE`(bit 0)是1。它有两个起点：一个是写在pmjump.S中的由real mode跳转至，跳转后首先将`%ds, %es, %fs, %gs, %ss`设置为`__BOOT_DS`(`0x18`)，设置`ltr`为`__BOOT_TSS`；另一个是32-bit内核arch/x86/boot/compressed/vmlinux的起始地址`startup_32`，进入至该地之后`%ds, %es, %ss`被设置为`__BOOT_DS`(`0x18`)，跳转long mode之前置`ltr`为`__BOOT_TSS`。载入新的DGT，其`L`(bit 21)为1，置CR0的`PG`(bit 31)为1启用paging后，后跳转到long mode。跳转进入long mode时更新`%cs`的值为`__KERNEL_CS`(`0x10`)，之后第一件事是清零`%ds, %es, %ss, %fs, %gs`。

> Note: arch/x86/boot/compressed目录下的代码中存在不少`BP_`前缀的变量，它的定义在编译中生成，定义在arch/x86/kernel/asm-offsets.c中，`BP`是`boot_params`的缩写，`BP_xxx`即`boot_params.xxx`。

### debug option

对于x86早期启动阶段的日志信息，可以使用cmdline `earlyprintk=ttyS0 debug`控制串口日志信息的打印，与串口相关的配置信息在arch/x86/boot/early_serial_console.c[[2]]中，在real mode[[3]]与protected mode[[4]]中都能看到`console_init()`的身影。

## smp init, kexec reboot

### smp init

多核CPU上，第一个启动的CPU称为BSP(bootstrap processor)，其他的CPU称为AP(application processor)。自BSP跳转进入arch/x86/kernel/head_64.S中的`startup_64`后，arch/x86/boot目录下的代码不再使用。BSP唤醒AP的调用路径如下：

```md
smp_init (kernel/smp.c)
-> bringup_nonboot_cpus (kernel/cpu.c, sequence wake up AP)
-> cpu_up (kernel/cpu.c)
-> _cpu_up (kernel/cpu.c)
-> cpuhp_up_callbacks (kernel/cpu.c)
-> cpuhp_invoke_callback (kernel/cpu.c)
-> cpuhp_hp_states[CPUHP_BRINGUP_CPU].startup.single (kernel/cpu.c)
-> bringup_cpu (kernel/cpu.c, callback)
-> __cpu_up (arch/x86/include/asm/smp.h)
-> smp_ops.cpu_up (arch/x86/kernel/smp.c)
-> native_cpu_up (arch/x86/kernel/smpboot.c, callback)
-> do_cpu_up (arch/x86/kernel/smpboot.c)
   - start_ip = real_mode_header->trampoline_start
   - early_gdt_descr.address = (unsigned long)get_cpu_gdt_rw(cpu)
   - initial_code = (unsigned long)start_secondary
   - initial_stack  = idle->thread.sp
-> wakeup_cpu_via_init_nmi (arch/x86/kernel/smpboot.c, for apic has no wakeup_secondary_cpu method)
-> wakeup_secondary_cpu_via_init (arch/x86/kernel/smpboot.c)
```

AP被BSP通过顺序的`INIT INIT SIPI(Startup IPI)`三个IPI中断(inter processor interrupts)唤醒，在`do_cpu_up`中指定了AP的起始地址`real_mode_header->trampoline_start`。AP的初始化路径如下：

```md
# files in arch/x86/realmode/rm
trampoline_start (trampoline_64.S, 16-bit real mode)
-> startup_32 (trampoline_64.S, 32-bit protected mode)

# files in arch/x86/kernel
-> startup_64 (head_64.S, 64-bit long mode)
-> secondary_startup_64 (head_64.S, initial_code has been start_secondary)
-> start_secondary (arch/x86/kernel/smpboot.c)
```

> Note: 在arch/x86/realmode目录下，前缀`pa_`与`BP_`类似，它定义在编译时生成的realmode/rm/realmode.lds中，前缀`pa_xxx`即symbol `xxx`。

BSP在进入`do_cpu_up`逐个唤醒AP时，与AP之间有类似同步的操作，通过变量`cpu_initialized_mask, cpu_callout_mask`实现，AP的相关代码在`wait_for_master_cpu`(arch/x86/kernel/cpu/common.c)中。可为Linux启用dynamic debug 参数`dyndbg="file smpboot.c +p; file smp.c +p"`获取详细的smp init日志信息。

### kexec reboot

Linux提供了两个系统调用`kexec_load, kexec_file_load`[[5]]，能够令OS不经BIOS重启。kexec-tools[[6]]利用这两个接口实现了相关功能，参数`-l`使用的是`kexec_load`，`-s`使用的是`kexec_file_load`，他们的区别主要体现在long mode内核的引导代码上：`-l`的引导阶段代码使用的是kexec-tools源码目录下的purgatory，`-q`使用的是内核源码中的arch/x86/realmode。此外，`kexec_file_load`自v5.4起能够通过`CONFIG_KEXEC_SIG`支持security boot。对64-bit的bzImage，默认情况下，kexec—tools使用的32-bit的启动协议，跳转入口是`startup_64`。

使用`-l`载入内核，kexec-tools至少会塞入用于内核引导的purgatory、启动协议要求的boot params与cmdline，以及bzImage三段数据，还可以塞入可选的initramfs。`kexec_load`参数`entry`指向bzImage入口的物理地址，`struct struct kexec_segment`中的`buf, bufsz`指向用户态地址空间及其大小，`mem, memsz`指向将要载入用户态数据的物理地址。在kexec-tools的代码中，因为purgatory是预先编译好的二进制数据，因此存在很多`elf_rel_{get, set}_symbol()`的相关代码，他们用于动态更新purgatory诸如bzImage的入口、boot params等数据。kexec-tools `-l`存在参数`--console-serial, --console-vga`用于CPU显示运行在purgatory阶段时的相关debug信息。

`kexec_load, kexec_file_load`的区别仅仅在载入bzImage的方式，一个是以二进制数据方式，一个是通过读取传入的文件名方式，其余步骤皆相同，即相当于把kexec-tools中对bzImage的解析从用户态搬到了内核态、使用的是内核提供的purgatory (arch/x86/realmode)。他们的核心是对`struct kimage`的填充：`control_code_page`保存着kexec reboot过程中所需的页表；对`-l, -f`这类`KEXEC_TYPE_DEFAULT`类型的bzImage载入，因其目标物理地址空间不存在连续的区域能够载入数据，只能零散地存放在目标区域，因此需要提供一页swap区域用于在kexec reboot时将零散的数据集中起来，这就是`swap_page`的作用；至于`-p`参数指定的类型是`KEXEC_TYPE_CRASH`，它使用的是由内核启动参数`crashkernel`预留出的一篇区域，因此不需要`swap_page`。

kexec-tools的`-e`参数用于内核的重启，入口在`reboot(LINUX_REBOOT_CMD_KEXEC)`，在reboot过程中，会触发以此触发`device_shutdown`、`migrate_to_reboot_cpu`与`machine_shutdown`，最后调用到`machine_kexec` (arch/x86/kernel/machine_kexec_64.c)，进入`relocate_kernel` (arch/x86/kernel/relocate_kernel_64.S)后，确保启用paging、protected mode的情况下跳转进入purgatory。在`relocate_kernel`过程中，`swap_pages`函数即利用swap page将分散的数据集中起来。

自`migrate_to_reboot_cpu`后的步骤都是BSP的行为，BSP在调用`machine_shutdown`时令AP进入halt (`hlt`指令)状态，调用路径如下：

```md
# BSP
machine_shutdown (arch/x86/kernel/reboot.c)
-> native_machine_shutdown (arch/x86/kernel/reboot.c)
-> stop_other_cpus (x86/include/asm/smp.h)
-> native_stop_other_cpus (arch/x86/kernel/smp.c)
   - apic_send_IPI_allbutself(REBOOT_VECTOR) (vec 0xf8)
   - apic_send_IPI_allbutself(NMI_VECTOR)

# AP
REBOOT_VECTOR
-> smp_reboot_interrupt (arch/x86/kernel/smp.c)
 or sysvec_reboot (from v5.6)
-> stop_this_cpu (arch/x86/kernel/process.c)
-> native_halt (arch/x86/include/asm/irqflags.h)

NMI_VECTOR
-> smp_stop_nmi_callback (arch/x86/kernel/smp.c)
-> stop_this_cpu
```

## x86 initialization

Intel手册Volume 3 9.1 INITIALIZATION OVERVIEW详细地描述了CPU初始化的细节，涵盖power-up, RESET, INIT三类事件的处理器状态、处理器自检、model与stepping信息、第一条指令的执行地址信息。

## Reference

- [Linux doc: x86 boot][1]
- [StackExchange: What is the difference between the following kernel Makefile terms: vmLinux, vmlinuz, vmlinux.bin, zimage & bzimage?][7]
- [Linux Inside: Kernel Boot Process][8]
- [Intel® 64 and IA-32 Architectures Software Developer Manuals][9]

[1]: https://github.com/torvalds/linux/blob/master/Documentation/x86/boot.rst
[2]: https://github.com/torvalds/linux/blob/v5.11/arch/x86/boot/early_serial_console.c#L148
[3]: https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L140
[4]: https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c#L370
[5]: https://man7.org/linux/man-pages/man2/kexec_load.2.html
[6]: https://github.com/horms/kexec-tools
[7]: https://unix.stackexchange.com/questions/5518/what-is-the-difference-between-the-following-kernel-makefile-terms-vmlinux-vml
[8]: https://0xax.gitbooks.io/linux-insides/content/Booting/
[9]: https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html