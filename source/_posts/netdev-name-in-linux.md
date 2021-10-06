---
title: Linux中的网卡重命名
date: 2021-06-13 17:43:32
tags: ["net", "linux"]
---

## Background

`$ ip link`能够显示网卡（netdev）的名称，曾经，它的前缀是*eth*、*wlan*，现在有的是*enp*，有的是*ens*，也有*eth*的。研究了下导致上述现象的原因。

## Histroy, and Systemd Strategy

### History

在Linux中的网卡驱动代码中，一个以太网设备的名称通常这么定义，比如Intel e1000[[1]]：

```c
// drivers/net/ethernet/intel/e1000/e1000_main.c
static int e1000_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
    ...
    strcpy(netdev->name, "eth%d");
	err = register_netdev(netdev);
    ...
}
```

`register_netdev()`会调用`dev_get_valid_name()`，判断`netdev->name`含有转义字符`%d`，因此动态地分配了index number用于唯一标识这个网卡。

> Note: 若`netdev->name`无转义字符`%d`，则其值即为网卡名称，不会动态地分配index number，此设备也只能存在一个该类型的网卡。

上述策略导致一个问题：特定网卡的名称依赖kernel启动过程中发现它的时机——完全相同的设备，同一位置（甚至同一个设备）的网络设备存在名称不同的可能。因此，Consistent Network Device Naming[[2]]的概念被提了出来。

有些发行版曾实现过一个策略[[4]]：将网卡名与mac地址绑定，在网络设备第一次出现时动态地生成udev规则，从而能够固定网卡的名称。但这策略也有其缺点：设备的mac地址不能发生变化，也依赖可写的root目录，并且还存在命名冲突的风险。

Dell实现了一个名为*biosdevname*[[3]]的程序实现了这类规则。它的原理：安装此程序后，会在*udev/rules.d*目录下添加一udev rule，udev在处理uevent事件的时候，匹配到此规则[[11]]，调用*biosdevname*，它根据特定的规范[[5]]重命名网卡。可在kernel参数中添加`biosdevname=0`禁用该功能。

### Systemd Strategy

Systemd实现了一套名为Predictable Network Interface Names[[8]]的机制用于解决网卡命名一致性的问题。其原理与*biosdevname*类似，它的命名规则、触发必要条件也是一条udev rule[[10]]，但规则是udev内置的。自v197始，该功能默认开启。

它的命名规则如下，序号越小优先级越高：

1. Names incorporating Firmware/BIOS provided index numbers for on-board devices (example: `eno1`)
2. Names incorporating Firmware/BIOS provided PCI Express hotplug slot index numbers (example: `ens1`)
3. Names incorporating physical/geographical location of the connector of the hardware (example: `enp2s0`)
4. Names incorporating the interfaces's MAC address (example: `enx78e7d1ea46da`)
5. Classic, unpredictable kernel-native ethX naming (example: `eth0`)

有三种方法禁用该功能[[6]][[7]]：

- kernel参数中添加`net.ifnames=0`
- 为网卡设置默认的策略：`ln -s /dev/null /etc/systemd/network/99-default.link`
- 为网卡手工创建自定义的名称，即在文件夹`/etc/systemd/network/`下创建后缀为`.link`[[9]]的文件

## Internal

netdev的重命名，在device shutdown的情况下，能够通过命令`# ip link set <DEVICE> name <NEW NAME>`完成，它调用的是netlink API。在kernel中，调用路径如下：

```md
-> sendmsg()
-> netlink_sendmsg()
-> rtnetlink_rcv()
-> netlink_rcv_skb()
-> rtnl_newlink() // callback is registered in `rtnetlink_init()`
-> do_setlink()
-> dev_change_name()
-> device_rename()
```

> Note: 运行`# ip link`，有时候会看到netdev下有*alias*字样，这种别名是通过`dev_set_alias()`来完成的。v5.5后，为网卡增加了alternative name[[11]]。

实际上systemd udev对网卡的重命名亦是通过调用netlink API来完成的。触发条件的规则[[10]]中有：

```md
IMPORT{builtin}="net_setup_link"
```

这指的是udev内置的一个方法：

```c
// src/udev/udev-builtin-net_setup_link.c
const UdevBuiltin udev_builtin_net_setup_link = {
        .name = "net_setup_link",
        .cmd = builtin_net_setup_link,
        .init = builtin_net_setup_link_init,
        .exit = builtin_net_setup_link_exit,
        .validate = builtin_net_setup_link_validate,
        .help = "Configure network link",
        .run_once = false,
};
```

至于网卡重命名，调用路径如下：

```md
# get name policy
-> builtin_net_setup_link_init() // in `src/udev/udev-builtin-net_setup_link.c`
   -> link_config_load()
      -> enable_name_policy() // read `net.ifnames` from kernel cmdline
      -> link_load_one()
         -> null_or_empty_path() // detect link file is whether `/dev/null` or not
         -> new() // parse link file

# rename
-> builtin_net_setup_link()
   -> link_config_apply()
      -> link_config_generate_new_name()
         -> sd_device_get_sysname()
      -> link_config_apply_alternative_names()
```

## Reference

- [systemd doc: PREDICTABLE_INTERFACE_NAMES][2]
- [RedHat doc: 11.9. DISABLING CONSISTENT NETWORK DEVICE NAMING][6]
- [Debian wiki: NetworkInterfaceNames][7]
- [Wikipedia: Consistent Network Device Naming][2]

[1]: https://github.com/torvalds/linux/blob/v5.12/drivers/net/ethernet/intel/e1000/e1000_main.c#L1200
[2]: https://en.wikipedia.org/wiki/Consistent_Network_Device_Naming
[3]: https://github.com/dell/biosdevname
[4]: https://wiki.debian.org/NetworkInterfaceNames#THE_.22PERSISTENT_NAMES.22_SCHEME
[5]: https://en.wikipedia.org/wiki/Consistent_Network_Device_Naming#Device_naming_rules
[6]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-disabling_consistent_network_device_naming
[7]: https://wiki.debian.org/NetworkInterfaceNames
[8]: https://github.com/systemd/systemd/blob/main/docs/PREDICTABLE_INTERFACE_NAMES.md
[9]: https://www.freedesktop.org/software/systemd/man/systemd.link.html
[10]: https://github.com/systemd/systemd/blob/v248/rules.d/80-net-setup-link.rules
[11]: https://github.com/dell/biosdevname/blob/v0.7.3/biosdevname.rules.in
[12]: https://lwn.net/Articles/794289/