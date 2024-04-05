---
title: linux网卡命名规则及问题解决
date: 2024-04-03 03:15:58
categories:
- linux
tags:
- 网卡
---

Linux 中网络接口的命名规则通常由 udev（内核设备管理器）管理，过去网卡的命名可能是 eth0、eth1 等，但现在 Linux 采用了一种更加可预测和稳定的命名方案，称为一致网络设备命名规范。
<!--more-->
# linix网卡命名

Linux内核为网络接口分配名称采用的是一种简单和直观的方式：一个固定的前缀和一个递增的序号。比如，内核使用`eth0`名称以标识启动后第一个加载的网络设备，第二个加载的设备名称是`eth1`，第三个是`eth2`，以此类推。。。如果用户想要在系统启动后添加一个新的网卡，那么内核也会按这个规则为它分配新的设备名称。

内核分配的网卡名称有一个隐患：每次系统启动时网络设备的加载顺序是不固定的（多数发生在为系统增加网卡，否则加载顺序基本固定），当系统重启时，内核可能会为因加载顺序为同一个网络设备分配一个与之前不同的名称，原本名称为eth0的网卡，可能经过一次系统重启后就变成了eth1。这样就会对部分涉及到网卡数据采集的应用程序产生影响，因此经开发者们商讨后，决定采用一致性网络设备命名规则

目前linux的主流操作系统采用一致网络设备命名（CONSISTENT NETWORK DEVICE NAMING）规范。dell开发了biosdevname方案，systemd v197版本中将dell的方案作了进一步的一般化拓展。目前的Centos既支持dell的biosdevname，也支持systemd的方案。

## net.ifnames和biosdevname

biosdevnane和net.ifnames是可以被设置的内核参数，默认是net.ifnames=1,biosdevname=0

**net.ifnames命名规范**

设备类型

| 格式 | 含义                  |
| ---- | --------------------- |
| en   | ethernet              |
| sl   | serial line IP (slip) |
| wl   | wlan                  |
| ww   | wwan                  |

设备位置

| 格式                                                         | 含义                                             |
| ------------------------------------------------------------ | ------------------------------------------------ |
| `b<number>`                                                  | BCMA bus core numbe                              |
| ` c<bus_id>  `                                               | CCW bus group name, without leading zeros [s390] |
| `o<index>[d<dev_port>]`                                      | on-board device index number                     |
| `s<slot>[f<function>][d<dev_port>]`                          | hotplug slot index number                        |
| `x<MAC>`                                                     | MAC address                                      |
| `[P<domain>]p<bus>s<slot>[f<function>][d<dev_port>]`         | PCI geographical location                        |
| `[P<domain>]p<bus>s<slot>[f<function>][u<port>][..][c<config>][i<interface>]` | USB port number chain           |

示例

**eth0**  经典的、不可预测的内核原生 ethX 命名

**eno1** 板载1号网卡，包含固件/BIOS 的名称为板载设备提供索引号

**ens33** BIOS 内置的 PCI-E 接口的网卡，包含固件/BIOS 提供的 PCI Express 热插拔插槽索引号的名称

**enp0s2** ethernet PCI接口位置：bus=2, slot=0，包含硬件连接器物理/地理位置的名称

**enx000ec6877201** mac地址为000ec6877201，包含接口 MAC 地址的名称

**wwp0s29f7u2i2**  4G modem

**wlp1s0** wlan PCI接口位置：bus=3, slot=0

## biosdevname

biosdevname本身是一个用于在Linux系统中生成网络设备名称的工具。它的作用是根据系统的BIOS信息为网络接口设备生成一个唯一的、可识别的名称。

| Device                          | Old Name     | New Name                                    |
| ------------------------------- | ------------ | ------------------------------------------- |
| Embedded network interface(LOM) | eth[0123...] | em[1234...]                                 |
| PCI card network interface      | eth[0123...] | p<slot>p<ethernet port>                     |
| Virtual function                | eth[0123...] | p<slot>p<ethernet port>_<virtual interface> |

**示例**
**em1** 板载网卡
**p3p4** pci网卡
**p3p4_1** 虚拟网卡

## systemd-udev

以centos8-stream为例，系统默认用于网卡设备重命名的服务是systemd-udevd，我们可以使用systemctl status systemd-udevd来查看目前该服务的状态。

```shell
[root@localhost ~]# systemctl status systemd-udevd.service
● systemd-udevd.service - udev Kernel Device Manager
   Loaded: loaded (/usr/lib/systemd/system/systemd-udevd.service; static; vendor preset: disabled)
   Active: active (running) since Mon 2024-04-01 00:48:50 CST; 2h 1min ago
     Docs: man:systemd-udevd.service(8)
           man:udev(7)
 Main PID: 579 (systemd-udevd)
   Status: "Processing with 12 children at max"
    Tasks: 1
   Memory: 19.8M
   CGroup: /system.slice/systemd-udevd.service
           └─579 /usr/lib/systemd/systemd-udevd
```

通过在虚拟机内查看打印信息，我们发现vmware的虚拟网卡被重命名了，

```shell
[root@localhost ~]# dmesg | grep ens160
[    2.354276] vmxnet3 0000:03:00.0 ens160: renamed from eth0
[   17.332767] IPv6: ADDRCONF(NETDEV_UP): ens160: link is not ready
[   17.339550] vmxnet3 0000:03:00.0 ens160: intr type 3, mode 0, 3 vectors allocated
[   17.339763] vmxnet3 0000:03:00.0 ens160: NIC Link is Up 10000 Mbps
```

注：**VMXNET3** 是一种 **VMware** 虚拟网卡（VNIC）类型。它基于 **博通 BCM 5719** 芯片开发，专为虚拟机在 **VMware** 平台上的网络通信而设计。

### 默认命名策略

默认情况下，systemd-udev会使用以下策略，采用支持的命名方案为接口命名：

- scheme 1、如果从BIOS中能够取到可用的板载网卡的索引号，则使用这个索引号命名，例如: eno1，如不能则尝试scheme2
- scheme 2、如果从BIOS中能够取到可以用的网卡所在的PCI-E热插拔插槽的索引号，则使用这个索引号命名，例如: ens1，如不能则尝试scheme 3
- scheme 3、如果能拿到设备所连接的物理位置信息，则使用这个信息命名，例如:enp2s0，如不能则尝试scheme 5
- scheme 4、使用网卡的MAC地址来命名，这个方法一般不使用。enx78e7d1ea46da
- scheme 5、传统的kernel命名方法，例如: eth0，这种命名方法的结果不可预知的，即可能第二块网卡对应eth0，第一块网卡对应eth1。

### rename流程

在centos8中， systemd命名网卡的规则根据以下6个配置文件
/lib/udev/rules.d/60-net.rules，
/lib/udev/rules.d/71-biosdevname.rules
/lib/udev/rules.d/75-net-description.rules
/lib/udev/rules.d/80-net-name-slot.rules
/lib/udev/rules.d/80-net-setup-link.rules
/lib/udev/rules.d/99-systemd.rules

- 第一步: /lib/udev/rules.d/60-net.rules

`60-net.rules`的作用是通过网络接口配置文件更改网络设备命名，它获取匹配网络设备MAC地址的配置文件，并将该配置文件中的设备名作为新设备名。
文件内容如下，当添加设备时，对于net子系统中任意非空设备驱动且type属性为1的设备，执行`/lib/udev/rename_device`程序，如果程序输出不为空，那么将设备名设置为程序输出

```
[root@localhost ~]# cat /lib/udev/rules.d/60-net.rules
ACTION=="add", SUBSYSTEM=="net", DRIVERS=="?*", ATTR{type}=="1", PROGRAM="/lib/udev/rename_device", RESULT=="?*", NAME="$result"
```

`/lib/udev/rename_device`是一个协助处理程序，它从`INTERFACE`环境变量中读取网络设备名称，随后通过`/sys/class/net/%s/address`获取MAC地址，如果`/etc/sysconfig/network-scripts/ifcfg-*`中有文件的`HWADDR`参数匹配到对应MAC地址，那么将输出该文件中`DEVICE`参数的值作为网卡名称

- 第二步:/lib/udev/rules.d/71-biosdevname.rules

`71-biosdevname.rules`一般在需额外安装的`biosdevname`软件包中，该规则的作用是通过从bios获取的设备文件名更改网络设备命名，它只会在`biosdevname`内核参数为1时才生效。主要是取SMBIOS中的`type 9 (System Slot)` 和 `type 41 (OnboardDevices Extended Information)`。
文件内容如下，当添加设备时，对于`net`子系统中任意名称为空（未重命名过），`type`属性为1且`DEVTYPE`环境变量为空的设备，如果`biosdevname`内核参数为1，那么执行`/sbin/biosdevname --smbios 2.6 --nopirq --policy physical -i %k`程序（%k是设备的内核名称），将其输出作为新名称。

```shell
[root@localhost ~]# cat /lib/udev/rules.d/71-biosdevname.rules
SUBSYSTEM!="net", GOTO="netdevicename_end"
ACTION!="add",    GOTO="netdevicename_end"
NAME=="?*",       GOTO="netdevicename_end"
ATTR{type}!="1",  GOTO="netdevicename_end"
ENV{DEVTYPE}=="?*", GOTO="netdevicename_end"

# kernel command line "biosdevname={0|1}" can turn off/on biosdevname

IMPORT{cmdline}="biosdevname"
ENV{biosdevname}=="?*", ENV{UDEV_BIOSDEVNAME}="$env{biosdevname}"

# ENV{UDEV_BIOSDEVNAME} can be used for blacklist/whitelist

# but will be overwritten by the kernel command line argument

ENV{UDEV_BIOSDEVNAME}=="0", GOTO="netdevicename_end"
ENV{UDEV_BIOSDEVNAME}=="1", GOTO="netdevicename_start"

# off by default

GOTO="netdevicename_end"

LABEL="netdevicename_start"

# using NAME= instead of setting INTERFACE_NAME, so that persistent

# names aren't generated for these devices, they are "named" on each boot.

SUBSYSTEMS=="pci", PROGRAM="/sbin/biosdevname --smbios 2.6 --nopirq --policy physical -i %k", NAME="%c"  OPTIONS+="string_escape=replace"

LABEL="netdevicename_end"
```

- 第三步： /lib/udev/rules.d/75-net-description.rules

`75-net-description.rules` 中的规则让 udev 通过检查网络接口设备，根据device属性填写网卡的属性命名，有些设备属性可能处于未定义状态，一个网卡通常同时具有多个维度的名称，systemd在选取的时候，按照有先后次序，使用先命中的，顺序可以简单理解为（eno1-ens1-enp1）

```shell
[root@localhost ~]# udevadm info /sys/class/net/ens160 | grep NAME
E: ID_NET_NAME=ens160
E: ID_NET_NAME_MAC=enx000c290e5d36
E: ID_NET_NAME_PATH=enp3s0
E: ID_NET_NAME_SLOT=ens160
```

当添加网络设备时，调用`net_id`内置程序获取设备信息并设置内部环境变量，对于usb类型的网络设备，调用`usb_id`以及`hwdb`内置程序获取设备信息并设置内部环境变量，对于pci类型的网络设备，根据一些内核中的设备属性以及`hwdb`内置程序设置内部环境变量，文件内容如下

```shell
# do not edit this file, it will be overwritten on update

ACTION=="remove", GOTO="net_end"
SUBSYSTEM!="net", GOTO="net_end"

IMPORT{builtin}="net_id"

SUBSYSTEMS=="usb", IMPORT{builtin}="usb_id", IMPORT{builtin}="hwdb --subsystem=usb"
SUBSYSTEMS=="usb", GOTO="net_end"

SUBSYSTEMS=="pci", ENV{ID_BUS}="pci", ENV{ID_VENDOR_ID}="$attr{vendor}", ENV{ID_MODEL_ID}="$attr{device}"
SUBSYSTEMS=="pci", IMPORT{builtin}="hwdb --subsystem=pci"

LABEL="net_end"
```

- 第四步： /lib/udev/rules.d/80-net-name-slot.rules

如果在60-net.rules ，71-biosdevname.rules这两条规则中没有重命名网卡，且内核未指定net.ifnames=0参数，则udev依次尝试使用以下属性值来命名网卡，如果这些属性值都没有，则网卡不会被重命名。

```shell
ID_NET_NAME_ONBOARD
ID_NET_NAME_SLOT
ID_NET_NAME_PATH
```

上边的71-biosdevname.rules 是实际执行biosdevname的policy
75-net-description.rules和80-net-name-slot.rules实际执行step1,2,3

- 第五步 /lib/udev/rules.d/80-net-setup-link.rules

`80-net-setup-link.rules`的作用是通过一些udev内置程序获取网络设备的信息并设置为环境变量，用于后续步骤。

文件内容如下，当添加网络设备时，调用`path_id`内置程序获取设备信息，随后调用`net_setup_link`内置程序，如果先前没有重命名设备，且内置程序输出的`ID_NET_NAME`环境变量不为空，那么将该环境变量的值设置为新设备名称。

```shell
# do not edit this file, it will be overwritten on update

SUBSYSTEM!="net", GOTO="net_setup_link_end"

IMPORT{builtin}="path_id"

ACTION!="add", GOTO="net_setup_link_end"

IMPORT{builtin}="net_setup_link"

NAME=="", ENV{ID_NET_NAME}!="", NAME="$env{ID_NET_NAME}"

LABEL="net_setup_link_end"
```

`net_setup_link`内置程序会读取`/usr/lib/systemd/network/99-default.link`配置文件，从而决定最后网络设备应该使用的名称：

```shell
[Link]
NamePolicy=kernel database onboard slot path
AlternativeNamesPolicy=database onboard slot path
MACAddressPolicy=persistent
```

该文件的`NamePolicy`参数标记了网络设备命名的优先级顺序：内核名称 > udev硬件数据库中记录名称 > onboard名称(`ID_NET_NAME_ONBOARD, eno`) > slot名称(`ID_NET_NAME_SLOT, ens`) > path名称(`ID_NET_NAME_PATH, enp`)。

另外，`AlternativeNamesPolicy`则记录了网络设备的别名，如果有该配置项，那么应用程序也可以通过别名来访问网络设备。

# 修改网卡命名的场景

## 取消一致网络设备命名

1. 修改grub2启动参数，在GRUB_CMDLINE_LINUX的中加上"net.ifnames=0 biosdevname=0"的参数

   `vi /etc/default/grub`

   ```shell
   GRUB_CMDLINE_LINUX="find_preseed=/preseed.cfg auto noprompt priority=critical locale=en_US" net.ifnames=0 biosdevname=0”
   ```

2. 重新加载到启动中
   
   对于 Debian 的 Ubuntu/Mint：
   
   ```text
   sudo update-grub
   ```
   
   Centos/RHEL
   
   ```text
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   ```
   
3. 重新对网卡配置文件进行命名（网卡文件全部重命名，顺便修改配置文件NAME、DEVICE的名称）
   `mv /etc/sysconfig/network-scripts/ifcfg-enp0s3 /etc/sysconfig/network-scripts/ifcfg-eth0`

4. reboot重启生效

## 基于MAC地址固定网络设备名称

通过udev设备管理器，我们可以很方便的更改以及定制网络设备命名规则，比如如果想要基于MAC地址固定某个网络设备的名称，那么可以创建`/etc/udev/rules.d/70-persistent-net.rules文件，内容如下：

```shell
SUBSYSTEM=="net",ACTION=="add",ATTR{address}=="ac:1f:6b:84:85:03",ATTR{type}=="1",NAME="eth0"
```

其中`ATTR{address}`表示MAC地址，`ATTR{type}=="1"`表示是`Ethernet`类型。修改后重启生效

## ubuntu18修改网卡名称

ubuntu18默认根据 /lib/udev/rules.d/目录下的80-net-setup-link.rules文件定义的规则。如果要更改规则，需要先将文件80-net-setup-link.rules从/lib/udev/rules.d目录复制到/etc/udev/rules.d目录。因为/etc/udev/rules.d目录下规则的优先级高于/lib/udev/rules.d目录，识别网卡并命名时，会优先从/etc/udev/rules.d目录下寻找规则文件。将ID_NET_NAME改成ID_NET_SLOT即可

```shell
# do not edit this file, it will be overwritten on update

SUBSYSTEM!="net", GOTO="net_setup_link_end"

IMPORT{builtin}="path_id"

ACTION=="remove", GOTO="net_setup_link_end"

IMPORT{builtin}="net_setup_link"

NAME=="", ENV{ID_NET_NAME}!="", NAME="$env{ID_NET_NAME}"

LABEL="net_setup_link_end"
```

# 虚拟机网卡元数据


# 参考文档

[centos7中的网卡一致性命名规则、网卡重命名方法 - Noway11 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zyd112/p/8143464.html)

[Linux网络设备命名规则简介 - frankming - 博客园 (cnblogs.com)](https://www.cnblogs.com/frankming/p/17535560.html)

[linux网卡命名规则与修改方法 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/671719163)

