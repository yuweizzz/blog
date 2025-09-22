---
date: 2022-08-13 15:00:00
title: 搭建轻量级服务器
tags:
  - "Linux"
  - "IPv6"
  - "Rsync"
draft: false
---

这篇笔记用来记录将玩客云主机作为家用服务器的搭建经历。

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

## 购买主机和系统刷入

玩客云主机是迅雷出品的智能硬件，它会使用客户闲置的计算资源，同时给予可提现的玩客币作为回报，类似于比特币的挖矿行为，但是现在玩客币交易受限，所以被囤积挖矿的二手玩客云主机可以用较低的价格入手，在拼多多上只要 50 块钱左右就可以买到。

主机的硬件配置为 4 核 1.5G ARM 架构 CPU ， 1GB RAM 和 8GB eMMC ROM ，千兆网卡和两个 USB 2.0 接口以及 HDMI 接口，作为轻量级内网服务器完全是足够的。

刷机资源在网上也是非常之多，可以在恩山论坛或者 B 站都可以找到。

刷机过程大概需要如下步骤：

1. 拆开主机外壳，在短接电路板上的触点的情况下，使用 USB 线连接电脑刷入底包。
2. 完成后使用制作完成的启动 U 盘，调用脚本将系统数据写入到 eMMC ROM 中。

这里涉及了一些硬件嵌入式开发的知识，但是单从 Linux 系统的角度上看， eMMC ROM 就是作为系统盘的块设备，执行 U 盘中的安装脚本只是跳过了常规的系统安装步骤，直接将完整的系统数据写入 eMMC ROM 中，后续主机就可以从 eMMC ROM 中的系统启动。

最终使用的镜像为 Armbian 20.12 Buster with Linux 5.9.0-rc7-aml-s812 。

## 修改网络环境

现有的环境下家庭宽带申请公网 ip 地址比较困难，最终决定直接使用 IPv6 来实现公网访问。

### 组网方案

原有的网络设定是光猫直接使用路由模式，路由器接入光猫千兆口作为子设备，上网设备直接接入路由器，这是比较普遍的家庭宽带组网方案。

新的组网方案使用了光猫桥接和路由器拨号的组合模式。

光猫桥接部分需要设置的东西比较少，只需要将光猫的因特网接口从路由模式更改为桥接模式即可，桥接后光猫主要承担光电转换的工作。

路由器拨号部分需要修改路由器的 wan 口设置，使用原有光猫宽带账号进行 pppoe 拨号上网，同时启用 IPv6 wan ，并且设置 wan 口向运营商上游索取 IPv6 地址的方式，这样 wan 口就可以拿到运营商分配的 IPv6 地址，它通常是 64 位掩码的。

在拿到运营商分配的 IPv6 地址之后，wan 口设置打开 prefix delegation 和 Rapid-commit ， lan 口设置关闭 DHCPv6 功能，开启 IPv6 radvd 服务。

完成了上述设置后，内网的支持 IPv6 的所有设备就都拥有了理论上全球可通信的 IPv6 地址。

### IPv6 理论知识

在 IPv6 中主要有 SLAAC 和 DHCPv6 两种地址获取方式，其中 DHCPv6 可以继续细分为 stateless 和 stateful 方式，以实际测试的情况来看，路由器使用任一方式都可以正常拿到 IPv6 地址。

SLAAC 的一些相关名词的详细信息如下：

- SLAAC ：全称为 Stateless Address AutoConfiguration ，也就是无状态自动地址配置，它是 IPv6 的一大特色，可以为接入 IPv6 网络的设备自动配置 IPv6 地址。
- NDP ：全称为 Neighbor Discovery Protocol ，也就是邻居发现协议， SLAAC 就是由这个协议提供的功能实现。
- RA ：全称为 Router Solicitation ，也就是路由协定请求，是由 NDP 实现的报文。
- RS ：全称为 Router Advertisement ，也就是路由通告，同样是由 NDP 实现的报文，作为 RA 报文的响应。

一般的 SLAAC 过程是这样的：假设我们的 IPv6 网络已经架设完成，那么新设备接入网络时会根据 NDP 协议定义向网络内的路由器发起 RS 协商请求，等待来自路由器的 RA 响应，最重要的响应信息其实是 IPv6 的子网前缀，新设备再按照这个信息生成自身的 IPv6 地址，成功接入网络，完成地址配置。

DHCPv6 则是类似于传统 DHCP 服务的 IPv6 实现，同时拥有一些 IPv6 独有的功能，比如前缀代理 prefix delegation 就是 DHCPv6 的特色功能，简称为 DHCPv6-PD 。而 Rapid-commit 是 DHCPv6 的功能选项，用来支持 DHCP 地址快速分配。

DHCPv6-PD 可以认为是划分下属子网并进行前缀下发的服务，在前述的拨号过程中，由于已经开启了 wan 口的 PD 功能，它就会作为客户端，向运营商获取子网前缀，然后在这个子网前缀允许的范围内再次划分子网，向下属的 lan 口进行派发。

DHCPv6-PD 的意义在于路由自动协商，在路由器开启 PD 之后，运营商是 DHCPv6-PD 服务端，路由器是运营商服务端的直接客户端，路由器下联设备是运营商服务端的间接客户端，当路由器拿到运营商分配的子网前缀，它继续下发到设备的子网前缀信息会自动添加到运营商服务端的静态路由中，这也是子网设备能够全球访问的原因，如果路由器不开启 PD ，则 wan 口可以正常获取到 IPv6 但不进行子网前缀下发，内网设备只能通过保留回环进行 IPv6 通信，这时 lan 口的 DHCPv6 服务就显得无关紧要，因为它分配的网络只在本地局域网可用，不会通告给运营商，公网环境没有到达这个子网的路由。

IPv6 radvd 服务用于辅助 SLAAC 地址获取，但是需要注意不开启这个服务设备仍然可以正常获取 IPv6 ，它的存在意义在于它会主动向网络链路中的设备广播 RA 报文，这样在 pppoe 重新拨号后， IPv6 地址的子网前缀改变，所有内网设备也会收到 RA 报文并重新配置地址，否则只能等待设备主动发起 RS 或者 DHCP 才能获取到新的 IPv6 地址。

前面还讲到 DHCPv6 细分为 stateless 和 stateful 方式，实际上是这两种方法主要是网络参数的获取差异，其中 stateful 方式的 IPv6 地址， DNS 地址等网络参数都从 DHCP 服务器获取，而 stateless 方式的 IPv6 地址是从 RA 通告报文中获取子网前缀并自行生成， DNS 地址等网络参数仍然从 DHCP 服务器获取，决定使用何种获取方式完全由 RA 报文的标识决定，在使用硬件路由的情况下是很难直接改变的。

### 实际表现

我的个人 Win10 电脑在网络环境设置完成后，在默认设置开启了 DHCP 的情况下，拿到了三个 IPv6 地址：

- 第一个地址是通过 SLAAC 获取的地址， SLAAC 一般会使用网卡 mac 地址或者网络接口的随机 ID 来组合子网前缀生成 IPv6 地址， Win10 使用了随机 ID 和子网前缀来生成。
- 第二个地址是通过 DHCP 获取的地址， DNS 服务器地址也是在这一步获取的，由于我所有的内网设备只有 Win10 开启了 DHCP ，所以这个地址是 IPv6 局域网子网前缀下的第一个 ip 。
- 第三个地址是随机生成的临时地址，这是 IPv6 为了隐私安全设定的标准，会基于 SLAAC 的地址额外生成临时地址，用于对外主动通信，它的使用优先级是最高的。

我的玩客云 Linux 设备在网络环境设置完成后，拿到了一个 IPv6 地址：

- 唯一的地址是通过 SLAAC 获取的，使用了网卡 mac 地址和子网前缀来生成。

在 Linux 中， IPv6 地址的生成方式是依赖于内核参数的：

- `net.ipv6.conf.eth0.autoconf` ：参数为 0 代表禁用 SLAAC ，参数为 1 代表启用 SLAAC 。
- `net.ipv6.conf.eth0.accept_ra` ：参数为 0 代表不接收 RA 报文，参数为 1 代表允许接收 RA 报文，参数为 2 代表允许接收和转发 RA 报文，这个参数对软路由能否启用 IPv6 协议栈非常关键。
- `net.ipv6.conf.eth0.addr_gen_mode` ：参数为 0 代表使用网卡 mac 地址作为 SLAAC 的地址后缀，参数为 1 代表使用网络接口的随机 ID 作为 SLAAC 的地址后缀。
- `net.ipv6.conf.eth0.use_tempaddr` ：参数为 0 代表不生成临时地址，参数为 1 代表使用并生成临时地址。

在我使用的 Armbian 中， `accept_ra` 参数为 1 ， `autoconf` 参数为 1 ， `addr_gen_mode` 参数为 0 ， `use_tempaddr` 参数为 0 ，所以最终只获取到一个由 SLAAC 生成的 IPv6 地址。

现在的 DNS 设置仍然是由 IPv4 的 DHCP 服务配置的，因为 SLAAC 并不会配置 DNS 信息，如果想要在 Linux 中使用 DHCPv6 自动配置 DNS 信息，应该使用命令 `dhclient -6 <interface>` 主动拉起 DHCPv6 客户端，这样 DNS 设置就会由 DHCPv6 客户端接管配置。在 Armbian 中，和 DNS 配置相关的文件是 `/etc/resolv.conf` 和 `/etc/resolvconf/run/resolv.conf` ，它们之间应该通过软链接相关联。

## 配置硬盘休眠

由于主机有 USB 2.0 接口，并且家里有一块闲置的西数 500GB SATA 硬盘，所以就将这块硬盘作为主机的外置存储。因为在实际使用过程中并不会经常地读写硬盘，所以我希望硬盘在平时可以处于休眠状态，并且在真正读写工作完成后继续休眠。

在一开始时优先想到使用 `hdparm` 工具来设置硬盘的待机模式：

```bash
# 使用 hdparm 设置硬盘的待机模式
$ hdparm -y /dev/sda
$ hdparm -C /dev/sda
# -y 可以使硬盘强制进入待机模式，不应该在系统盘上使用
# -C 可以查看目前硬盘状态，硬盘一般会处于 active 或者 standby
# -Y 可以使硬盘强制进入睡眠状态，尽量不要使用

# 待机状态的硬盘可以被硬盘写入事件唤醒，但是睡眠状态的硬盘未经实测，很可能无法直接被硬盘写入事件唤醒
```

但是实际使用过程中却发生了错误，经过资料查阅之后得到大概的结论如下：

- hdparm 工具是通过 kernel 的 libata 子系统和 IDE 子系统来和硬盘交互，达到读取或者设置硬盘参数的目的。
- 玩客云主机使用 USB2.0 接口连接硬盘，系统实际上通过 UAS (USB Attached SCSI) 协议来和硬盘进行交互。

```bash
# 查看运行时的系统模块，可以看到 uas 正在被使用
$ lsmod | grep uas
uas                    20480  0

# 查询设备信息
$ udevadm info -q all -p /sys/block/sda
P: /devices/platform/soc/c90c0000.usb/usb1/1-1/1-1:1.0/host0/target0:0:0/0:0:0:0/block/sda
N: sda
L: 0
S: disk/by-path/platform-c90c0000.usb-usb-0:1:1.0-scsi-0:0:0:0
S: disk/by-id/usb-WDC_WD50_00AVDS-61U7B1_7F833EEF5DC0-0:0
E: DEVPATH=/devices/platform/soc/c90c0000.usb/usb1/1-1/1-1:1.0/host0/target0:0:0/0:0:0:0/block/sda
E: DEVNAME=/dev/sda
E: DEVTYPE=disk
E: MAJOR=8
E: MINOR=0
E: SUBSYSTEM=block
E: USEC_INITIALIZED=11132604
E: ID_VENDOR=WDC_WD50
E: ID_VENDOR_ENC=WDC\x20WD50
E: ID_VENDOR_ID=152d
E: ID_MODEL=00AVDS-61U7B1
E: ID_MODEL_ENC=00AVDS-61U7B1\x20\x20\x20
E: ID_MODEL_ID=1337
E: ID_REVISION=0508
E: ID_SERIAL=WDC_WD50_00AVDS-61U7B1_7F833EEF5DC0-0:0
E: ID_SERIAL_SHORT=7F833EEF5DC0
E: ID_TYPE=disk
E: ID_INSTANCE=0:0
E: ID_BUS=usb
E: ID_USB_INTERFACES=:080650:080662:
E: ID_USB_INTERFACE_NUM=00
E: ID_USB_DRIVER=usb-storage
E: ID_PATH=platform-c90c0000.usb-usb-0:1:1.0-scsi-0:0:0:0
E: ID_PATH_TAG=platform-c90c0000_usb-usb-0_1_1_0-scsi-0_0_0_0
E: ID_PART_TABLE_UUID=3f36f79e
E: ID_PART_TABLE_TYPE=dos
E: DEVLINKS=/dev/disk/by-path/platform-c90c0000.usb-usb-0:1:1.0-scsi-0:0:0:0 /dev/disk/by-id/usb-WDC_WD50_00AVDS-61U7B1_7F833EEF5DC0-0:0
E: TAGS=:systemd:
```

从物理层面上看，整体硬件直接表现为通过 USB 物理接口直接连接了 SATA 硬盘设备，但从协议层面上看，由于块设备挂载于 Linux 系统的 SCSI 总线，所以在使用 `hdparm` 下达命令时，系统需要使用 SAT (SCSI / ATA Translation) 将 ATA 命令转换为 SCSI 命令格式，经过 SCSI 总线，最终由 UAS 驱动模块将具体的电信号传达给硬盘，但是主机的 USB 接口芯片可能由于设计或固件版本等原因导致驱动无法正常传输某些转换后的 ATA 命令，这就是直接使用 `hdparm` 命令报错的原因。

原生的 SCSI 命令可以正常被 UAS 驱动传达给硬盘，所以使用 `sdparm` 来设置硬盘的待机模式，它是可以直接发送 SCSI 命令的交互工具。

```bash
# 使用 sdparm 设置硬盘的待机模式

# 查看硬盘是否允许进入待机模式
$ sdparm -l --get STANDBY /dev/sda
    /dev/sda: WDC WD50  00AVDS-61U7B1     0508
STANDBY       1  [cha: y, def:  1, sav:  1]  Standby_z timer enable
# 当前 sav 值允许硬盘进入待机模式
# 当前 def 值说明默认情况下允许硬盘进入待机模式

# 查看硬盘进入待机模式的定时器
$ sdparm -l --get SCT /dev/sda
    /dev/sda: WDC WD50  00AVDS-61U7B1     0508
SCT           6000  [cha: y, def:18000, sav:6000]  Standby_z condition timer (100 ms)
# 当前定时器时间为 6000 * 100 ms 即 10 min
# 默认定时器时间为 18000 * 100 ms 即 30 min
# 到达定时时间值则硬盘进入待机模式

# 允许硬盘进入待机模式
sdparm -l --save --set STANDBY=1 /dev/sda
# 调节待机模式定时器时间为 5 min
sdparm -l --save --set SCT=3000 /dev/sda
```

经过上述设置后，硬盘会在停止读写活动的 5 min 之后变为待机状态，因为我的硬盘盒有工作指示灯，可以明显观察到空闲时硬盘的工作灯处于绿色状态，有别于工作时的红色状态。

## 通过 rsync 同步文件

由于有一些原有文件存放在 Win10 电脑上想要同步到 Armbian 主机中，所以使用了 rsync 来同步文件，这样一来 Windows 是 rsync 服务端，而 Armbian 是 rsync 客户端。

在 Windows 端使用 cwRsync 搭建服务端，可以直接使用 `rsync.exe` 通过命令行启动服务，具体配置和启动方法如下：

```bash
# 以下命令在 Windows cmd.exe 下执行：
> cd cwrsync  # 切换到工作目录
> type rsyncd.conf  # 查看配置文件
use chroot = false
strict modes = false
hosts allow = *
log file = rsyncd.log
pid file = rsyncd.pid
port = 873

[data]  # 按需命名，在客户端中使用
path = /cygdrive/c/Files/Downloads  # 在 Windows 路径中为 C:\Files\Downloads
read only = true
transfer logging = yes
lock file = rsyncd.lock
auth users = user  # 客户端连接服务端需要的认证用户
secrets file = rsync.password  # 用户密码存放的文件

> type rsync.password
user:user

# 启动 rsyncd 时 rsync.password 和 rsyncd.conf 均需在同一目录下
> rsync.exe --daemon --config=rsyncd.conf

# 停止 rsyncd 时，找到进程号并杀死进程，清理 pid 文件以便下次启动
> type rsyncd.pid
1188
> taskkill /PID 1188 /F
> del rsyncd.pid

# 以下命令在 Linux shell 下执行：
# Linux 客户端连接 rsyncd 服务端
$ rsync -rvu rsync://user@192.168.1.2/data /mnt/
# 执行命令会将 C:\Files\Downloads 中的文件会递归同步到 /mnt 中

# 更复杂的同步用法要配合其他子选项，这是一个支持断点传输的用例：
$ rsync -rv --progress --partial --append-verify rsync://user@192.168.1.2/data /mnt/

# 具体的选项意义如下：
# -r 表示同步过程将递归项目下的所有目录
# -v 用于显示整个传输过程的运行信息
# --progress 用于显示传输过程中各文件的传输进度
# --partial 表示保留未完全传输完成的文件区块
# --append-verify 会利用 --partial 中保留的文件区块，实现断点传输
# -u/--update 与 --append-verify 都是优化传输的选项
# -u 会根据文件命名和文件时间戳只更新不同的部分，是相对于整个文件而言
# --append-verify 可以使得部分传输完成的文件块进行续传，不过它和 -u 选项不能同时使用
```
