---
date: 2020-02-29 21:23:45
title: Iptables 知识笔记
tags:
  - "iptables"
  - "Linux"
draft: false
---

这篇笔记用来记录 Linux 中 iptables 的相关知识。

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

## Iptables 的规则链

在 Linux 的网络协议栈中，通过内核组件 netfilter 来控制网络数据包的来往，对主机进行网络防护。而 iptables 是用来配置 netfilter 内核组件规则的用户层工具。

在 iptables 中，将所有网络数据包抽象为几条规则链，首先数据包会从外界流入，内核也会发出数据包，它们就区分两条规则链 `INPUT` 和 `OUTPUT` 。这里考虑的情况比较简单，相当于认为系统只是局域网络下的一个普通用户，但如果系统作为了局域网的网关时，两条规则链的管理能力是明显不够的。这时 `INPUT` 之前会诞生两条新的规则链 `PREROUTING` 和 `FORWORD` ，其中 `PREROUTING` 作用于路由前的数据包，即未分辨数据包发往何处， `FORWORD` 作用于需要转发的数据包，其实在 `PREROUTING` 中的数据包一般就只有进入内核，被转发或者丢弃这几种处理办法，正好对应了 `INPUT` 和 `FORWORD` 。

由于 `FORWORD` 的诞生，会额外在 `OUTPUT` 延申新的规则链 `POSTROUTING` ，从 `FORWARD` 中流出的数据包可以直接进入 `POSTROUTING` ，并且内核中 `OUTPUT` 的数据包也要经过 `POSTROUTING` 。

```text
+---------------+                             +---------------+
|               |                             |               |
|    INPUT      |                             |    OUTPUT     |
|               |                             |               |
+------+--------+                             +------+--------+
       ^                                             |
       |                                             |
       |                                             |
       |                +---------------+            |
       | ROUTING        |               |            |
       +--------------->+    FORWARD    +------------+
       |                |               |            |
       |                +---------------+            v
+------+--------+                             +------+--------+
|               |                             |               |
|  PREROUTING   |                             |  POSTROUTING  |
|               |                             |               |
+---------------+                             +---------------+
```

所以一共有 `INPUT` ， `OUTPUT` ， `PREROUTING` ， `FORWORD` ， `POSTROUTING` 五条规则链控制进出的网络数据包。在作为局域网络内单一主机时， `PREROUTING` ， `FORWORD` 和 `POSTROUTING` 是不起作用的，只由 `INPUT` 和 `OUTPUT` 负责本机的数据包管控。在作为网络入口或中转设备时， `PREROUTING` ， `FORWORD` 和 `POSTROUTING` 起到主要作用，`INPUT` 和 `OUTPUT` 起到辅助作用，如果想要将 Linux 作为网关来使用，还需要通过 `echo 1 > /proc/sys/net/ipv4/ip_forword` 启动内核的路由转发功能，这样上述的对应规则链才有意义。

## Iptables 的规则表

在前面的五条规则链的基础上，每条规则链之中具有相同功能的规则会划分到四张表中，四个表分别是 `filter` ， `mangle` ， `nat` 和 `raw` 。并且四个表有优先级之分，从高到低依次是 `raw` ， `mangle` ， `nat` ， `filter` ：

- raw 是跟踪数据表规则表，一般用来控制数据包的连接追踪机制。

- mangle 是修改数据标记位规则表，负责对数据包进行修改，并重新封装。

- nat 是地址转换规则表，负责地址转换和端口映射。

- filter 是过滤规则表，负责过滤，是使用最多的表。

在四张表中， `raw` 表比较特殊，它可以用于关闭数据的的追踪处理，因为 iptables 的数据追踪相关信息会记录在 `/proc/net/nf_conntrack` 中，并且 `/proc/sys/net/netfilter/nf_conntrack_max` 控制了数据包最大追踪数量，在流量过大时可能会出现 `ip_conntrack: table full, dropping packet` 这类日志，这时可以通过将这类数据包加入到 `raw` 表的规则中以关闭 iptables 对数据包状态的追踪，关闭了数据追踪的数据包就可以跳过后续低级别的规则表处理。

`nat` 表和 `filter` 表是使用频繁的规则表， `filter` 可以用来过滤攻击数据包，而 `nat` 在虚拟化网络和容器网络等场景都有相关的应用。 `mangle` 表则可以修改相关的数据包内容，复杂度更高。

## Iptables 的规则管理

在实际应用中，我们直接使用 iptables 添加需要的规则动作。

```bash
# 查看 iptables 规则
# 不指定 -t 选项，默认查询 filter 表的规则
$ iptables -L -n --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
2    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0
3    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
4    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22
5    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination
1    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
```

规则是以 IP 地址为基本匹配条件的，比较常见的响应动作有 `ACCEPT` ， `REJECT` ， `DROP` ， `LOG` ， `DNAT` ， `SNAT` ， `REDIRECT` 和 `MASQUERADE` 这几种，它们基本都出现在 `nat` 表和 `filter` 表中：

- `ACCEPT` ， `REJECT` ， `DROP` 是最基本的三种动作， `ACCEPT` 为接收， `REJECT` 为拒绝接收， `DROP` 为丢弃。

- `DNAT` ， `SNAT` ， `MASQUERADE` 和 `REDIRECT` 是 `nat` 表的专属动作，其中 `SNAT` 为源地址转换， `DNAT` 为目的地址转换， `REDIRECT` 为端口映射， `MASQUERADE` 为源地址转换的特殊形式，是一种动态转换源地址的 NAT ，它会自动获取地址作为 SNAT 的转换地址，而不是使用固定的地址进行转换。

- `LOG` 是用于日志记录的动作，它会在 `/var/log/message` 中记录一条信息，然后把数据包交由下一规则匹配。

顺序是 iptables 的工作准则，在各个表中会按照规则的顺序进行匹配，匹配成功就执行对应的动作，后面的规则不会被使用，除了特殊动作 `LOG` 。如果流过某个规则链的所有表格都没有对应的匹配规则，就会按照这个规则链的默认策略来响应。

```bash
# iptables 具体使用例子

# 新增规则，新规则将会是表中最后一条规则
$ iptables -t filter -A INPUT -s 192.168.1.222 -j REJECT
# 插入规则，新规则的序号将会是 1
$ iptables -t filter -I INPUT -s 192.168.1.222 -j  REJECT
# 使用具体序号插入规则
$ iptables -t filter -I INPUT 4 -s 192.168.1.222 -j REJECT
# 通过详细条件匹配规则并进行删除
$ iptables -t filter -D INPUT -s 192.168.1.222 -j REJECT
# 通过规则序号匹配规则并进行删除
$ iptables -t filter -D INPUT  4
# 通过规则序号匹配规则并进行规则内容替换
$ iptables -t filter -R INPUT 4 -s 192.168.1.222 -j REJECT
# 清除所有规则
$ iptables -F
# 设置规则链的默认策略
$ iptables -t filter -P INPUT DROP
# 输出规则表中的所有规则
$ iptables -t filter -S
# 将当前所有规则保存到文件中
$ iptables-save > /etc/sysconfig/iptables
# 使用文件中保存的规则覆盖当前所有规则
$ iptables-restore < /etc/sysconfig/iptables


# 通过 iptables 设置端口转发
# 源地址为 192.168.1.222:9999 ，目标地址为 192.168.1.233:10000 ，则应该在 192.168.1.222 上添加转发规则
$ iptables -t nat -A PREROUTING -p tcp -m tcp --dport 9999 -j DNAT --to-destination 192.168.1.233:10000
$ iptables -t nat -A POSTROUTING -d 192.168.1.233 -p tcp -m tcp --dport 10000 -j SNAT --to-source 192.168.1.222
```

### Iptables 规则的多种匹配条件

在 iptables 中规则是以 IP 地址为基本匹配条件的，其中 `-s` 表示源地址，`-d` 表示目的地址，除了 IP 地址， iptables 还有很多匹配条件，包括端口，传输协议，数据包状态等。比如可以使用 `-p` 指定传输协议；使用 `-i` 指定流入数据的网卡接口；使用 `-o` 指定流出数据的网卡接口。

以上还是比较单一的匹配条件，可以通过扩展模块来加强这些单一条件，常用的模块有 iprange ，string ，time ，limit ，connlimit ，state 等。

- iprange 模块用来指定一定范围内的 IP 地址，具体用例为 `-m iprange --src-range 192.168.1.20-192.168.1.100` 和 `-m iprange --dst-range 192.168.1.127-192.168.1.146` 。

- string 模块可以指定匹配报文中的字符串，具体用例为 `-m string --algo bm --string 'match string'` ，其中 --algo 用来指定匹配算法，一般是 bm 或 kmp ， --string 用来指定要匹配的字符串。

- time 模块可以匹配数据包的时间，通过时间进行对应匹配动作，具体用例为 `-m time --timestart 09:00:00 --timestop 18:00:00` 和 `-m time --weekdays 1,2,3,4,5` 。

- limit 模块可以限制报文的入站速率，通过时间单位来限制某种数据包的入站数量。一般用法是 `-m limit --limit 100/{ second | minute | hour | day } --limit-burst 5` 。这个模块使用了令牌桶算法， --limit 是令牌的生成速率， --limit-burst 是令牌桶的上限，默认为 5 ，它是用来限制峰值的数值，会在令牌被使用后按照给定的速率填充。

- connlimit 模块可以限制 IP 地址到本机的连接数量，比如 `-m connlimit --connlimit-above 3 -p tcp --dport 22` 可以限制单个 IP 地址到本机的 SSH 链接只能在 3 个以内，如果不指定 IP ，对所有 IP 都会进行限制； `-m connlimit --connlimit-above 3  --connlimit-mask 24` 会限制整个网段的连接数量， 24 代表掩码的位数，可以把一个网段内所有 IP 连接限制在 3 个以下。

- state 模块可以监控数据包的状态，通过状态进行对应匹配动作。这个状态并不是 tcp 协议的连接状态，而是独有的一种状态分类，对各种协议均有效。它一共有 `NEW` ， `ESTABLISHED` ， `INVALID` ， `RELATED` ， `UNTRACKED` 五种状态：
其中 `NEW` 用来代表第一次到达本机的数据包； `ESTABLISHED` 用来代表已经通过一次 iptables 的数据包，比如本机向外发出请求，外部响应发回的数据包； `RELATED` 用来代表数据包不是主动入站或出站，而是通过其他连接发起的数据包，有时我们连接一项服务，这些服务主动发起新连接就属于这种状态； `INVALID` 用来代表无法确定状态的数据包，一般会被直接丢弃； `UNTRACKED` 用来代表无法追踪的数据包。具体用例为 `-m state --state NEW,ESTABLISHED` 。

- tcp 模块可以专门处理 tcp 协议的数据包，可以通过 --tcp-flags 来筛选 tcp 报头一些特有的标志位，比如发起连接的 SYN ，断开连接的 FIN 。具体用例 `-m tcp --tcp-flags ALL SYN` 可以用来匹配发起 tcp 连接的报文，即第一次握手。而 `-m tcp --tcp-flags ALL SYN,ACK` 可以用来匹配第二次握手的报文，它以筛选出报文中同时带有 SYN 和 ACK 的标志为匹配条件。
