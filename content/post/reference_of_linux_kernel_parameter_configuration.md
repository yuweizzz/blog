---
date: 2024-10-14 15:14:10
title: 常用 Linux 内核参数配置参考
tags:
  - "Linux"
  - "Kernel"
draft: false
---

这篇笔记用来记录一些常用 Linux 内核参数配置参考。

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

## net.ipv4.ip_forward

允许在不同网卡之间转发数据包，是路由转发的必要条件，默认值为 `0` ，即不开启路由转发。

非常需要修改的选项，比如容器环境就需要开启路由转发，否则容器的网络环境可能会有异常。

```bash
# 开启路由转发
# 直接修改内核参数，临时生效
echo 1 > /proc/sys/net/ipv4/ip_forward

# 通过 sysctl 持久化
echo net.ipv4.ip_forward = 1 >> /etc/sysctl.conf
sysctl -p
```

## net.ipv6.conf.all.disable_ipv6 和 net.ipv6.conf.default.disable_ipv6

是否禁用 IPv6 协议栈，默认值为 `0` ，即默认启用 IPv6 协议栈。

这个值应该根据自身的网络环境来设置，有时候为了减小复杂度，可能会直接禁用 IPv6 。

```bash
# 禁用 IPv6 协议栈
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
echo 1 > /proc/sys/net/ipv6/conf/default/disable_ipv6
```

## net.ipv4.conf.all.rp_filter 和 net.ipv4.conf.default.rp_filter

是否开启反向路径过滤，默认值为 0 ，即不开启反向路径过滤，各个发行版可能会有不同的调整。

具体的值定义如下：

```text
0 - No source validation.
    不做来源验证
1 - Strict mode as defined in RFC3704 Strict Reverse Path
    Each incoming packet is tested against the FIB and if the interface
    is not the best reverse path the packet check will fail.
    By default failed packets are discarded.
    严格模式，所有流入的数据包需要通过 FIB (Fowarding Information Base) 进行测试，
    如果反向路由和出口网卡不一致，那么将会校验失败，默认丢弃这些数据包
2 - Loose mode as defined in RFC3704 Loose Reverse Path
    Each incoming packet's source address is also tested against the FIB
    and if the source address is not reachable via any interface
    the packet check will fail.
    宽松模式，所有流入的数据包需要通过 FIB (Fowarding Information Base) 进行测试，
    如果反向路由可以通过任意网卡到达，则视为校验成功，否则视为校验失败，默认丢弃这些数据包
```

反向路由的校验过程实际上就是来源地址和目标地址互换后，进行路由以确认流出网卡是否和当前的入口网卡一致。在多网卡的情况下，非常容易发生出入网卡不一致的情况，还有一些网络应用比如 cilium 就需要将所有 `cilium_*` 名称的网卡关闭反向路径过滤。

```bash
# 禁用 rp_filter
# 如果只想在某些特定的网卡上禁用，只需要改变 all 和对应 interface 的配置参数
echo net.ipv4.conf.all.rp_filter = 0 >> /etc/sysctl.conf
echo net.ipv4.conf.eth0.rp_filter = 0 >> /etc/sysctl.conf
sysctl -p
```

## net.ipv4.neigh.default.gc_stale_time

通过 `ip neigh` 可以查看当前机器的 ARP 信息，其中 STALE 是代表这个 ARP 条目已经处于陈旧状态，而默认的 `net.ipv4.neigh.default.gc_stale_time` 的值是 `60` ，即处于 STALE 状态的超时时间是 60 秒。超过这个时间没有被引用，会根据当下的不同情况进入其他对应状态。

```bash
# 降低 ARP 缓存陈旧条目的轮转频率
echo net.ipv4.neigh.default.gc_stale_time = 120 >> /etc/sysctl.conf
sysctl -p
```

## kernel.pid_max

当内核分配的 pid 达到当前值时，会从最小值重新分配，并且内核不会分配大于或等于 pid_max 值的 pid 。内核的 pid_max 默认值为 32768 ，各个发行版可能会有不同的调整。

如果系统运行的进程数量达到这个限制值，可能无法产生新进程，最常见的表象是系统日志大量报错 `fork：Cannot allocate memory` ，无法登陆 SSH ，但实际内存使用量可能还没有到达上限。

```bash
# 修改进程数量上限
echo kernel.pid_max = 4194304 >> /etc/sysctl.conf
sysctl -p
```

## net.core.somaxconn

定义 `socket listen()` 的最大队列长度，一般也称作全连接队列，或者 `accept queue` ，此时 TCP 连接握手已经完成，等待上层应用调用 `accept()` 进行处理。默认值是 `4096` ，在一些低版本的内核中是 `128` 。这个值代表了对应 socket 能够处理的连接数量，用户层设置的值无法超过这个内核设置的值。

```bash
# 修改 socket 的全连接队列长度
echo net.core.somaxconn = 8192 >> /etc/sysctl.conf
sysctl -p
```

## net.ipv4.tcp_max_syn_backlog

定义每个 `listener` 处于 `SYN_RECV` 的队列长度，一般也称作半连接队列，或者 `syn queue` ，此时监听方已经接收到了客户端的 SYN 报文并且回复 ACK ，但需要继续等待客户端的 ACK 响应。

```bash
# 修改 TCP 半连接队列长度
echo net.ipv4.tcp_max_syn_backlog = 1024 >> /etc/sysctl.conf
sysctl -p
```

## net.netfilter.nf_conntrack_max 和 net.netfilter.nf_conntrack_buckets

net.netfilter.nf_conntrack_max 定义 conntrack 条目的最大值， net.netfilter.nf_conntrack_buckets 定义 conntrack 哈希表的大小。其中 net.netfilter.nf_conntrack_buckets 会由内核根据内存大小决定，而 net.netfilter.nf_conntrack_max 一般默认是 net.netfilter.nf_conntrack_buckets 这个值的 4 倍，避免出现过长的链表。

当系统日志出现大量 `nf_conntrack: table full, dropping packet` 的错误日志时，应该修改这两个值，允许存储更多的 conntrack 条目。

```bash
# 修改 conntrack 条目上限，这里的值仅供参考
echo net.netfilter.nf_conntrack_max = 1048576 >> /etc/sysctl.conf
echo net.netfilter.nf_conntrack_buckets = 262144 >> /etc/sysctl.conf
sysctl -p
```
