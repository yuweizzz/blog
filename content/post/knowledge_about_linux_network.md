---
date: 2023-11-09 20:00:00
title: Linux 网络协议知识笔记
tags:
  - "Linux"
  - "iproute2"
  - "iptables"
  - "openconnect"
draft: false
---

这篇笔记用来记录 Linux 系统下一些网络协议的相关知识。

<!--more-->

``` bash

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

## 路由管理

在 Linux 系统中一般通过 iproute2 对路由进行管理。

``` bash
$ ip route
default via 10.233.0.1 dev ens33 
10.233.0.0/16 dev ens33 proto kernel scope link src 10.233.0.2
```

虽然通过 `ip route` 大部分情况就可以满足基本的场景需求，但是可以使用多张路由表来实现更复杂的流量控制。

``` bash
# Linux 系统默认定义的路由表，其中 ip route 默认使用的就是 main ，也是大多数情况下默认使用的路由表
# /etc/iproute2/rt_tables 用来定义 route table 和 id 的映射关系
$ cat /etc/iproute2/rt_tables
#
# reserved values
#
255	local
254	main
253	default
0	unspec
#
# local
#
#1	inr.ruhep

# 路由表有优先级定义，一般来说 local 表的优先级应该是最高的
# 注意这里的数字表示的是 prio 而不是 table id
$ ip rule show
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default

# 添加新的路由表
$ echo "101 custom" >> /etc/iproute2/rt_tables  # 预先定义路由表的映射关系
$ ip rule add from 10.10.10.0/24 table custom prio 101
$ ip rule show
0:      from all lookup local
101:    from 10.10.10.0/24 lookup custom
32766:  from all lookup main
32767:  from all lookup default

# 可以不定义映射关系，直接通过 id 创建路由表
$ ip rule add from 10.10.11.0/24 table 102 prio 102
$ ip rule show
0:      from all lookup local
101:    from 10.10.10.0/24 lookup custom
102:    from 10.10.11.0/24 lookup 102
32766:  from all lookup main
32767:  from all lookup default

# 往新路由表中添加路由规则
# 当定义路由的 via 或者 src 时，指定的 IP 地址应该是本机持有的地址
$ ip route add default via 10.10.11.1 dev ens33 table 102        # 默认网关
$ ip route add 10.10.11.0/24 dev ens33 src 10.10.11.1 table 102  # 普通路由规则

# 查看添加规则后的路由表，不指定 table 则默认使用 main 表
$ ip route show table 102

# 删掉路由表
$ ip rule del table 102
$ ip rule del table custom  # 有映射关系的可以使用 id 或者名称
```

多张路由表可以用在多网络出口和特殊子网管理的场景，使用单独的路由表来控制对应的流量，在网络情况复杂的时候会很有用。

``` bash
# 通过 iptables 标记流量来指定处理的路由表，在容器网络管理方面经常使用
$ iptables -t mangle -A FORWARD -i ens33 -j MARK --set-mark 1
$ iptables -t mangle -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-A FORWARD -i ens33 -j MARK --set-xmark 0x1/0xffffffff

# 添加处理对应流量标记的路由表
$ ip rule add fwmark 1 table 103 prio 103
$ ip rule show 
0:      from all lookup local
101:    from 10.10.10.0/24 lookup custom
102:    from 10.10.11.0/24 lookup 102
103:    from all fwmark 0x1 lookup 103
32766:  from all lookup main
32767:  from all lookup default

# cilium 实际上也使用到了流量标记，以下输出是来自一台运行了 cilium 的系统
$ ip rule show
9:	from all fwmark 0x200/0xf00 lookup 2004
100:	from all lookup local
32766:	from all lookup main
32767:	from all lookup default
```

## MTU 和 MSS

在使用 openconnect 搭建 VPN 服务时，比较有意思的地方就是 tunnel 的 MTU 设定。

``` bash
# 搭建 VPN 服务时用到的防火墙配置，其中 10.10.10.0/24 是 VPN 服务所使用的网段
$ iptables -t filter -A FORWARD -s 10.10.10.0/24 -j ACCEPT
$ iptables -t filter -A FORWARD -o vpns+ -j ACCEPT
$ iptables -t filter -A FORWARD -i vpns+ -j ACCEPT
$ iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
# 有一些说法指出防火墙还需要配置下面这条规则
$ iptables -A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

# 连接 VPN 后查看新建立的 tunnel
$ ip tuntap show
tun0: tun
$ ip addr show tun0
27: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN group default qlen 500
    link/none
    inet 10.10.10.101/32 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::ec6a:a1a5:de04:9aa3/64 scope link flags 800
       valid_lft forever preferred_lft forever
```

openconnect 使用 tun 设备来连接客户端和服务端， tun 设备是 Linux 系统下的虚拟三层网络设备，可以看到这里使用的 MTU 值是 1472 。

MTU 是网络层的概念，它定义了网络传输中允许数据包通过的最大值，这个值的计算已经包括数据包标头在内，而 TCP MSS 是除去标头部分后 TCP 协议传输实际数据内容的最大值，是传输层的概念。因为标准以太网接口的 MTU 是 1500 bytes ，而 TCP 协议标头和 IP 协议标头都是 20 bytes ，所以标准以太网下对应的 TCP MSS 就是 1460 bytes 。

MTU 和 TCP MSS 的重要区别是，当数据包超过目标设备的 MTU 则会导致数据的分包或者丢弃，这种情况会降低传输效率。而在 TCP 协议的传输过程中，双方可以进行 MSS 通告，然后根据双方提供的 MSS 值中的最小值确定连接的具体 MSS 值。

一般来说， VPN 软件可能会添加一些自定义的数据包标头，所以为了保险起见，在创建 tun 设备时，会尽量使用低于 1500 的 MTU 值，这样当隧道中的数据包驮载到以太网设备中传输时，尽量做到保持数据包大小在标准以太网接口 MTU 之内。这一点可以参考 gre tunnel 的实现， gre tunnel 经常使用 1476 的 MTU 值，因为它需要 24 bytes 作为数据包标头。在 gre tunnel 的数据包中，包括了均为 20 bytes 的 TCP 协议标头和 IP 协议标头，这样它的 MSS 应该设置为 1436 bytes 是最合理，当数据包从 tunnel 中发出时，还需要添加 24 bytes 的数据包标头，最后总共就是 1500 bytes 的数据包格式。更多的详细信息可以参考思科的[官方文档](https://www.cisco.com/c/zh_cn/support/docs/ip/generic-routing-encapsulation-gre/25885-pmtud-ipfrag.html)。

那么根据推断， openconnect 的数据包标头应该是 28 bytes ，其中 20 bytes 应该是标准 IP 协议的标头， 而 8 bytes 应该是自定义标头，如果使用的是 ipv6 协议，这个总值应该是 48 bytes ，这里可以通过在 openconnect 服务端显式设置 MTU 值来验证，在连接过程中你可以看到对应的 HTTP header 已经包含了这部分信息，其中 `X-CSTP-Base-MTU` 就是服务端的 tunnel 预设值，一般会使用低于 1500 的 MTU 值，而 `X-CSTP-MTU` 就是去除标头后的实际 MTU 值，也就是最后 tunnel 实际使用的 MTU 值。

防火墙的 `--clamp-mss-to-pmtu` 一般是用来协调使用不同 MTU 值的网络接口传输过程中的 TCP MSS ，在 openconnect 中，服务端的流量出入口通常都是同个网卡，这个设置实际上是可以省去的，如果网络环境比较复杂可能才需要这个设置。此外除了自动调整，也可以将 TCP MSS 设定为一个固定值，通过 `iptables -A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1400` 就可以实现。
