---
title: 在VMware Player中启用UEFI模式
date: 2018-03-31 14:26:40
tags: [vmware, virtualization]
---

## 背景

几周前用QEMU的时候，发现[QEMU user manual](https://qemu.weilnetz.de/doc/qemu-doc.html)上面有这么一行，有`uefi`这一选项：

```md
-smbios type=0[,vendor=str][,version=str][,date=str][,release=%d.%d][,uefi=on|off]

    Specify SMBIOS type 0 fields
```

昨晚，打算给别人展示使用UEFI磁盘的分区情况，发现VMware Player中的VM并非UEFI方式启动的，于是打算找找有没有相关的资料展示怎么在VMware Player中使用UEFI。

## 可行性

VMware Player相比于VMware Workstation，功能实在是少得可怜，但毕竟Management tools与Hypervisor相互分离，猜测作为Non-commercial的Player与Workstation不大可能在Hypervisor上有所区别，只是一些Management tools的区别。

举个例子：libvirt作为QEMU的管理工具，为了方便地给QEMU创建的VM分配IP，会在创建的网桥中开启一个小型的DHCP。而VMware Player的网络配置中有NAT模式，它的VM的IP等也是自动配置的，猜测也会有DHCP服务存在。果然在`C:\ProgramDataVMware\`下发现了`vmnetdhcp.conf`，添加了下面的配置，便成功为MAC地址为`00:50:56:C0:00:08`的网络接口分配固定的IP。

```conf
host VMnet8 {
    hardware ethernet 00:50:56:C0:00:08;
    fixed-address 192.168.80.1;
    option domain-name-servers 0.0.0.0;
    option domain-name "";
    option routers 0.0.0.0;
}
```

搜到的资料提示修改`*.vmx`，打开后发现是配置信息，类似于libvirt的`*.xml`。但仅仅添加`firmware = "efi"`还是不够的，因为默认的BIOS版本太老了，还得修改BIOS版本。

## 方法

在安装目录下找到`x64`文件夹（比如我的是：`C:\Program Files (x86)\VMware\VMware Player\x64`），ROM名称有`BIOS.440.ROM, EFI32.ROM, EFI64.ROM`。在`*.vmx`文件中添加如下信息：

```vmx
bios440.filename = "BIOS.440.ROM"
firmware = "efi"
efi32.filename = "EFI64.ROM"
```

如果是32-bit UEFI，需要把`EFI64.ROM`变成`EFI32.ROM`。

## 参考资料

- [VMWare Player with UEFI](https://blog.gulivert.ch/vmware-player-with-uefi-bios/)
- [http://pete.akeo.ie/2011/06/extracting-and-using-modified-vmware.html](http://pete.akeo.ie/2011/06/extracting-and-using-modified-vmware.html)
