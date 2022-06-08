---
date: 2022-06-07 19:44:45
title: 符合容器化标准的容器运行时：Containerd
tags:
  - "Container"
  - "Containerd"
draft: false
---

这篇笔记用来记录 Containerd 的相关知识。

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

## 前言

containerd 是由 docker 公司贡献，符合 OCI 标准的容器进行时。

## containerd 和 docker 的关系

首先我们需要了解 [OCI](https://opencontainers.org/) 是什么，它的全称为 Open Container Initiative ，也就是开放容器提案，目前主要有 Runtime Spec 和 Image Spec ，Distribution Spec 几个定制标准，标准化使得各种第三方组件不再限定于某种容器运行时，而是可以自由运行在任何符合 OCI 标准的容器进行时，可以说 OCI 促进了容器技术的生态发展。

通常来说，容器进行时指的是管理容器生命周期的守护进程，比如 docker 就是最经典的容器进行时，我们只需要调用 docker 命令就可以启动或者停止容器。但实际上 docker 要做的工作不止这些，它还兼顾了镜像管理，镜像构建，容器网络和存储管理等工作，属于高层级的容器进行时。

containerd 也属于高层级的容器进行时，但相比于 docker 它更加精简，最关键地保留了管控容器生命周期的核心功能，在较新版本的 docker 中，管控容器生命周期的功能已经完全由 containerd 承担， docker 守护进程和 containerd 守护进程之间使用 grpc 进行通信。

containerd 守护进程管理着运行中的容器，与容器具体的交互功能则是由 runc 负责的， runc 是低底层的容器运行时接口，只要是符合 OCI 标准的容器镜像，就可以直接调用 runc 运行这个镜像。

但是 containerd 并不是直接调用 runc ，而是采用了垫片的方式，由 containerd-shim 去调用 runc ，而 containerd 只需要和 shim 交互，无需直接和更底层的 runc 交互，垫片机制在容器技术中很常见， Kubernetes 使用 dokcer 作为容器运行时也是通过 shim 来实现的。

```
#    containerd 和 docker 的关系

          docker daemon
                | grpc
                v
        containerd daemon
                | shim
                v
               runc
                |
                v
            container

```

## 直接通过 containerd cli 运行容器

由于 containerd 具备完善的容器生命周期能力，我们可以不再使用 docker cli ，直接使用 containerd 的 cli 工具 ctr 直接运行容器。

``` bash
# 安装 containerd 
$ yum install containerd
$ systemctl enable containerd
$ systemctl start containerd

# 拉取镜像
$ ctr i pull docker.io/library/busybox:latest
$ ctr i pull docker.io/library/nginx:alpine

# 生成容器
$ ctr c create docker.io/library/busybox:latest mybusybox 
$ ctr c ls
CONTAINER    IMAGE                               RUNTIME                           
mybusybox    docker.io/library/busybox:latest    io.containerd.runtime.v1.linux

# 启动容器
$ ctr t start -d mybusybox
$ ctr t ls
TASK         PID     STATUS    
mybusybox    2973    RUNNING
```

根据 OCI Runtime Spec ，正常的容器会经过以下几种运行状态：

```
# OCI Runtime Spec 

          init     ->    creating
                            |
            ^               v
     delete |            created
                            | start
                            v
         stopped   <-    running        <->      paused
                  kill             pause/resume

```

在 containerd 中，除了 container ，还引入了 task 的概念， start ， kill 和 delete 等动作是使用在 task 上的，对应 OCI Runtime Spec 定义的容器状态。而 container 只有 create 和 delete 两个动作，从表象上来看 task 在 containerd 中是实际运行的容器进程，而 container 是容器信息的声明。

``` bash
# task 支持多种动作
# pause 和 resume 就是冻结和恢复
$ ctr t ls
TASK         PID     STATUS    
mybusybox    2973    RUNNING

$ ctr t pause mybusybox
$ ctr t ls
TASK         PID     STATUS    
mybusybox    2973    PAUSED

$ ctr t resume mybusybox
$ ctr t ls
TASK         PID     STATUS    
mybusybox    2973    RUNNING

# kill 就是发送进程 signal ，一般用来发送终结信号
$ ctr t kill -s 9 mybusybox
# 2: SIGINT
# 9: SIGKILL
# 15: SIGTERM
$ ctr t ls
TASK         PID     STATUS    
mybusybox    2973    STOPPED

# 只有处于 STOPPED 状态才能被删除
$ ctr t delete mybusybox
$ ctr t ls
TASK    PID    STATUS

# exec 则是在容器中执行命令
$ ctr t exec -t --exec-id busybox-sh mybusybox sh
# -t 选项要求命令执行时分配终端
# --exec-id 选项用于命名这个 exec 进程
# 执行上述命令会生成基于 mybusybox task 的 sh 进程
```

## 增强 containerd 运行的容器

### 声明持久化挂载点

虽然 containerd 不具备 docker volumes 的功能，但是可以基于 mount 命名空间将宿主机上的某些文件目录映射到容器中。

``` bash
# 需要在 container creater 阶段声明 mount 信息
$ ctr c create docker.io/library/busybox:latest mybusybox --mount type=bind,src=/home/busybox,dst=/mnt,options=rbind:ro

$ ls /home/busybox/
fileA  fileB  fileC
$ ctr start -d mybusybox
# 进入容器并查询挂载目录
$ ctr t exec -t --exec-id busybox-sh mybusybox sh
/ # ls -l /mnt/
total 0
-rw-r--r--    1 root     root             0 Jun  6 13:23 fileA
-rw-r--r--    1 root     root             0 Jun  6 13:23 fileB
-rw-r--r--    1 root     root             0 Jun  6 13:23 fileC
```

### 使用 cni 扩展网络能力

containerd 可以基于 net 命名空间，配合 cni 工具构建网络模型。

cni 工具由 cnitool 和各个网络插件组成，一般使用 cnitool 读取定义了各种参数和插件的配置文件来生成网络接口，再将它们附加到 net 命名空间中。

``` bash
# 直接与宿主机共享网络栈
$ ctr c create --net-host docker.io/library/busybox:latest mybusybox
# 这样设置使得容器完全使用宿主机的网络，包括本地回环等所有网络接口
# --net-host 用于声明容器共享主机网络

# 使用 cni 扩展工具构建网络模型
# 使用 yum 安装 cni
$ yum install containernetworking-plugins

# cni 严重依赖环境变量提供运行时信息
# 配置文件路径声明，这一步通常可以省去，因为默认值就是 /etc/cni/net.d
$ export NETCONFPATH=/etc/cni/net.d
# 插件所在路径声明，如果 cnitool 执行失败，通常是没有指定这个路径导致的
$ ls /usr/libexec/cni/
bandwidth  firewall     host-local  macvlan  sample  tuning
bridge     flannel      ipvlan      portmap  sbr     vlan
dhcp       host-device  loopback    ptp      static
$ export CNI_PATH=/usr/libexec/cni/

# 使用 iproute2 创建 net 命名空间
$ ip netns add container-net

# 通过 conf 文件创建网络接口
$ cat /etc/cni/net.d/cni.conf
{
    "cniVersion": "0.4.0",
    "name": "cni",
    "type": "bridge",
    "bridge": "cni0",
    "isDefaultGateway": true,
    "forceAddress": false,
    "ipMasq": true,
    "hairpinMode": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.88.0.0/16"
    }
}
# 创建这个网络接口并附加到命名空间中
$ /usr/libexec/cnitool add cni /var/run/netns/container-net
# 查看已经生成的 cni0 网桥
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:0a:69:d0 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.7/24 brd 192.168.0.255 scope global dynamic ens33
       valid_lft 84564sec preferred_lft 84564sec
    inet6 fe80::20c:29ff:fe0a:69d0/64 scope link 
       valid_lft forever preferred_lft forever
3: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether b2:51:ec:c7:b0:39 brd ff:ff:ff:ff:ff:ff
    inet 10.88.0.1/16 brd 10.88.255.255 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::b051:ecff:fec7:b039/64 scope link 
       valid_lft forever preferred_lft forever
4: veth3adfd297@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master cni0 state UP group default 
    link/ether 96:4e:da:cb:0b:27 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::944e:daff:fecb:b27/64 scope link 
       valid_lft forever preferred_lft forever

# 声明并启动容器，查看容器内的网络信息
$ ctr c create docker.io/library/busybox:latest mybusybox --with-ns=network:/var/run/netns/container-net
$ ctr t start -d mybusybox
$ ctr t exec -t --exec-id busybox-sh mybusybox sh
/ # ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether ee:0f:03:a0:22:b7 brd ff:ff:ff:ff:ff:ff
    inet 10.88.0.2/16 brd 10.88.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ec0f:3ff:fea0:22b7/64 scope link 
       valid_lft forever preferred_lft forever

# 清除接口
$ /usr/libexec/cnitool del cni /var/run/netns/container-net
$ ip link delete cni0
$ ip netns delete container-net
```

以上信息描述了两种容器网络的实现方法，第一种非常简单，直接和主机共享网络栈，第二种网络模式则是由 cni 提供支持，和 docker 默认的网桥模式类似，更复杂的 cni 网络模式可以参阅[官方文档](https://github.com/containernetworking/cni/blob/spec-v1.0.0/SPEC.md)和各插件的[详细说明](https://www.cni.dev/plugins/current/main/)。

## 总结

目前 contained 已经可以直接作为 Kubernetes 的容器进行时，并且后续 Kubernetes 会减弱对 docker 的支持，在容器集群环境可以尝试直接使用 contained 替代 docker ，如果是单机环境则 docker 依旧是较好的选择。
