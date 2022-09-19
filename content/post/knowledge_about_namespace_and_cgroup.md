---
date: 2022-05-13 20:44:45
title: 实现容器技术的基础：Namespace 与 Cgroup
tags:
  - "Container"
  - "Namespace"
  - "Cgroup"
draft: false
---

这篇笔记用来记录 Namespace 与 Cgroup 的相关知识。

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

容器的实现依赖于 Namespace 与 Cgroup ，它们是内核直接提供的功能，如果想要深入理解容器的运行机制，这是必须了解的知识点。

## Namespace

命名空间 Namespace 是内核用来隔离资源的一项技术，不同的进程所拥有的资源可以互相隔离，互不干扰。

容器本质上也是系统的进程，但是它拥有独立的命名空间，可以和系统原有的进程隔离。

在 Linux 中已经定义的命名空间有 8 种：

* User ： 用户和用户组
* PID ： 进程 ID
* Mount ： 挂载点
* Network ： 网络设备和网络协议栈
* UTS ： 主机名和域名
* IPC ： 进程间通信
* Time ： 系统时钟
* Cgroup ： cgroup 控制群组

在这些不同的命名空间种类中， `Time` 和 `Cgroup` 在高版本的内核中才得到实现，现有生产环境的流行内核版本大部分只实现了前六种类型。

``` bash
# CentOS 7.9
# Kernel Version 3.10.0
# 以 root 用户运行
$ whoami
root
$ lsns
        NS TYPE  NPROCS   PID USER   COMMAND
4026531836 pid      115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531837 user     115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531838 uts      115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531839 ipc      115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531840 mnt      111     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531856 mnt        1    18 root   kdevtmpfs
4026531956 net      115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532430 mnt        2  1010 root   /usr/sbin/NetworkManager --no-daemon
4026532439 mnt        1  1027 chrony /usr/sbin/chronyd

# 以低权级用户查看命名空间
$ whoami
mine
$ lsns
        NS TYPE  NPROCS   PID USER COMMAND
4026531836 pid        2  8945 mine bash
4026531837 user       2  8945 mine bash
4026531838 uts        2  8945 mine bash
4026531839 ipc        2  8945 mine bash
4026531840 mnt        2  8945 mine bash
4026531956 net        2  8945 mine bash
# 仔细对比使用 root 和使用低权级用户的命名空间，可以发现它们仍然在同一 namespace 中，但是低权级用户仅能查询到有限的信息
```

系统启动后默认会生成 init namespace ，系统进程基本都运行在 init namespace 中，后续的命名空间都和它是有关联的，大部分情况下是继承关系，某些特殊进程可能使用了一到两种额外的命名空间类型。

Linux 提供了用户层命令 `unshare` 和 `nsenter` 来操作进程的命名空间归属。

``` bash
# CentOS 7.9
# Kernel Version 3.10.0
# 当前系统可能对 user 命名空间有限制，一般有两种情况：
# 如果内核禁用了 user 命名空间，可能需要重新安装系统或者编译内核
# 如果编译选项正常，则可能是内核对命名空间数量做了限制
$ whoami
root
$ cat /boot/config-3.10.0-1160.el7.x86_64 | grep -i user_ns
CONFIG_USER_NS=y
$ cat /proc/sys/user/max_user_namespaces 
0

# 开放内核对命名空间的限制， unshare 才能正常创建 user 命名空间
$ echo 10 > /proc/sys/user/max_user_namespaces

# 模拟容器，创建一个隔离的进程
[root@localhost ~]# unshare --pid --user -r --uts --net --mount --ipc  --fork --mount-proc --propagation private /bin/sh
$ unshare \
> --mount --propagation private \
> --uts \
> --ipc \
> --net \
> --pid --fork --mount-proc \
> --user --map-root-user \
```

uts 和 ipc 基本上都是直接从原命名空间克隆一些必要的数据，后续和父空间互不干扰，比如 uts namespace 会克隆父空间的 `/proc/sys/kernel/hostname` 作为自己的 hostname 。

net 除了和父空间隔离外，会生成新的网络栈并默认生成独立的 lo 网络设备。

对于 user ， unshare 提供了 `-r/--map-root-user` 来实现 root 用户映射，低权级用户可以通过使用这个选项成为新命名空间中的 root 用户，获取最高权限。

对于 pid ，unshare 提供了 `--fork` 和 `--mount-proc` 扩展选项，值得注意的是这里无论带什么选项， unshare 这个进程不会加入新的 pid 命名空间：

* `--mount-proc` 可以使得隔离进程和父空间的所有进程信息隔离开来，子空间的 pid 将重新从 1 开始衍生，但父空间仍然可以监控到这部分信息。如果不带 `--mount-proc` 选项，则父空间的 pid 信息会被子空间继承。
* `--fork` 则可以生成子进程来运行指定程序，如果使用上面的例子，你将会得到 `unshare` 进程和作为子进程的 `/bin/sh` ，其中 `unshare` 进程属于父 pid 命名空间，子进程 `/bin/sh` 新生成的子 pid 命名空间。如果不带 `--fork` 将无法生成新的 pid 命名空间，生成的 `/bin/sh` 仍然在原有的命名空间，并且它的生命周期极短，命令运行完成后进程就不再分配内存了。

mount 命名空间是最复杂的一种，它使用 shared subtrees 运行机制，允许在不同 mount 命名空间之间自动，受控地传播 mount 事件和 unmount 事件。简单来说，我们通过设置某个命名空间的 propagation 属性，决定这个命名空间中挂载动作和卸载动作是如何传播到其他命名空间。

propagation 有 `private|shared|slave|unchanged` 这几种，最常用的是 private ，它默认从父空间继承挂载节点，后续不同空间的挂载动作和卸载动作互不影响，在 docker 的官方文档中，有这么一段话： `Volumes use rprivate bind propagation, and bind propagation is not configurable for volumes.` ，说明了卷是基于 mount 命名空间来实现挂载特性的。

```bash
# 在运行容器的情况下查看现有的命名空间
# 使用 containerd 作为容器运行时
$ ctr i pull docker.io/library/busybox:latest
$ ctr c create docker.io/library/busybox:latest busybox
$ ctr t start -d busybox
$ ctr t ls
TASK         PID      STATUS    
busybox    61830    RUNNING
$ lsns
        NS TYPE  NPROCS   PID USER   COMMAND
4026531836 pid      114     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531837 user     115     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531838 uts      114     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531839 ipc      114     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531840 mnt      112     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026531856 mnt        1    18 root   kdevtmpfs
4026531956 net      114     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 22
4026532430 mnt        1   972 chrony /usr/sbin/chronyd
4026532446 mnt        1 61830 root   sh
4026532447 uts        1 61830 root   sh
4026532448 ipc        1 61830 root   sh
4026532449 pid        1 61830 root   sh
4026532451 net        1 61830 root   sh
```

从实际运行的容器来看，基本可以验证容器是拥有独立命名空间的进程这一说法。

## Cgroup

cgroup 是内核用来控制系统资源的一项技术，它可以基于 cgroup 进行资源分配管控，主要针对硬件层面的资源，只需要将进程纳入 cgroup 就可以管控进程的资源。

根据前面所述，容器本质上也是系统的进程，那么它也可以交由 cgroup 进行资源控制。

``` bash
$ ls /sys/fs/cgroup/
total 0
drwxr-xr-x. 5 root root  0 Apr 14 22:13 blkio
lrwxrwxrwx. 1 root root 11 Apr 14 22:13 cpu -> cpu,cpuacct
lrwxrwxrwx. 1 root root 11 Apr 14 22:13 cpuacct -> cpu,cpuacct
drwxr-xr-x. 5 root root  0 Apr 14 22:13 cpu,cpuacct
drwxr-xr-x. 3 root root  0 Apr 14 22:13 cpuset
drwxr-xr-x. 5 root root  0 Apr 27 21:01 devices
drwxr-xr-x. 3 root root  0 Apr 14 22:13 freezer
drwxr-xr-x. 3 root root  0 Apr 14 22:13 hugetlb
drwxr-xr-x. 5 root root  0 Apr 14 22:13 memory
lrwxrwxrwx. 1 root root 16 Apr 14 22:13 net_cls -> net_cls,net_prio
drwxr-xr-x. 3 root root  0 Apr 14 22:13 net_cls,net_prio
lrwxrwxrwx. 1 root root 16 Apr 14 22:13 net_prio -> net_cls,net_prio
drwxr-xr-x. 3 root root  0 Apr 14 22:13 perf_event
drwxr-xr-x. 5 root root  0 Apr 14 22:13 pids
drwxr-xr-x. 5 root root  0 Apr 14 22:13 systemd
```

cgroup filesystem 是使用 cgroup 时内核态与用户态沟通的重要节点，默认路径一般为 `/sys/fs/cgroup` ，目前 cgroup 有 v1 和 v2 两个版本，这里使用的是 cgroup v1 ，如果想要使用 cgroup v2 ，应该通过 `/proc/filesystems` 查看当前内核是否支持。

在这里我们可以看到 cgroupfs 将各种资源划分到多个子系统，主要有 cpu ， memory ， net ， blk 这几种资源，它们对应的控制器名称在后续创建 cgroup 是比较重要的。

### 使用 libcgroup 创建 cgroup

使用 libcgroup 创建 cgroup 并不是特别推荐的做法，但是可以帮助我们深入了解如何定义和控制 cgroup 。

如前文所述，我们需要通过 cgroup filesystem 来和内核沟通，虽然可以在 `/sys/fs/cgroup` 中直接创建新的 cgroup ，但是我们选择重新挂载 cgroupfs 来学习具体的工作过程。

``` bash
# 挂载 cgroup fs
# 以挂载两种子系统为例

# 首先创建目录，实际中可以自由命名
$ mkdir -p /mnt/cgroups/cpuset
$ mkdir -p /mnt/cgroups/memory

# 挂载 cpuset 和 memory 子系统
$ mount -t cgroup -o cpuset cgroup_cpuset /mnt/cgroups/cpuset
$ mount -t cgroup -o memory cgroup_memory /mnt/cgroups/memory

# 可以查询到新挂载的 cgroup fs ，对应目录下也会自动创建相应的控制器
$ mount -l
......
cgroup_memory on /mnt/cgroups/memory type cgroup (rw,relatime,seclabel,memory)
cgroup_cpuset on /mnt/cgroups/cpuset type cgroup (rw,relatime,seclabel,cpuset)
......

# 卸载子系统
$ umount /mnt/cgroups/memory
$ umount /mnt/cgroups/cpuset
```

自行挂载的 cgroup fs 在机器重启后会自动失效，如果确实需要使用 libcgroup ，推荐做法是将需要挂载的子系统和 cgroup 一并写入 `/etc/cgconfig.conf` 并启动 cgconfig 服务，就可以使自定义 cgroup 保持重启系统生效。

在挂载子系统完成后，就可以在对应的子系统中需要创建需要的 cgroup 。

``` bash
# 创建 cgroup

# 注意需要切换到新挂载的子系统目录下，否则 cgroup 会默认创建到 /sys/fs/cgroup 下的子系统
$ cd /mnt/cgroups/memory
# cgcreate 的创建语法为 -g <controller_name>:<custom_name> ，控制器和子系统对应，cgroup 命名则无限制
$ cgcreate -g memory:mem

# 在对应目录下直接 mkdir 也可以创建 cgroup ，下面的语句等效于 cgcreate -g memory:mem
$ cd /mnt/cgroups/memory
$ mkdir mem

# 一些其它控制器的例子
$ cd /mnt/cgroups/cpuset
$ cgcreate -g cpuset:custom_cpuset

# 查看创建的 cgroup
$ ll /mnt/cgroups/memory
total 0
-rw-r--r--. 1 root root 0 May 13 21:36 cgroup.clone_children
--w--w--w-. 1 root root 0 May 13 21:36 cgroup.event_control
-rw-r--r--. 1 root root 0 May 13 21:36 cgroup.procs
-r--r--r--. 1 root root 0 May 13 21:36 cgroup.sane_behavior
drwxr-xr-x. 2 root root 0 May 13 22:06 mem  # 新增的 cgroup
-rw-r--r--. 1 root root 0 May 13 21:36 memory.failcnt
--w-------. 1 root root 0 May 13 21:36 memory.force_empty
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.failcnt
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.limit_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.max_usage_in_bytes
-r--r--r--. 1 root root 0 May 13 21:36 memory.kmem.slabinfo
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.tcp.failcnt
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.tcp.limit_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--. 1 root root 0 May 13 21:36 memory.kmem.tcp.usage_in_bytes
-r--r--r--. 1 root root 0 May 13 21:36 memory.kmem.usage_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.limit_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.max_usage_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.memsw.failcnt
-rw-r--r--. 1 root root 0 May 13 21:36 memory.memsw.limit_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.memsw.max_usage_in_bytes
-r--r--r--. 1 root root 0 May 13 21:36 memory.memsw.usage_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.move_charge_at_immigrate
-r--r--r--. 1 root root 0 May 13 21:36 memory.numa_stat
-rw-r--r--. 1 root root 0 May 13 21:36 memory.oom_control
----------. 1 root root 0 May 13 21:36 memory.pressure_level
-rw-r--r--. 1 root root 0 May 13 21:36 memory.soft_limit_in_bytes
-r--r--r--. 1 root root 0 May 13 21:36 memory.stat
-rw-r--r--. 1 root root 0 May 13 21:36 memory.swappiness
-r--r--r--. 1 root root 0 May 13 21:36 memory.usage_in_bytes
-rw-r--r--. 1 root root 0 May 13 21:36 memory.use_hierarchy
-rw-r--r--. 1 root root 0 May 13 21:36 notify_on_release
-rw-r--r--. 1 root root 0 May 13 21:36 release_agent
-rw-r--r--. 1 root root 0 May 13 22:00 tasks

# 删除 cgroup ，它的全部进程会移动到其父群组
$ cgdelete -g memory:mem
```

创建完 cgroup 后，我们就可以调整资源参数，并将进程加入到 cgroup 中使它们受到对应的资源约束。

``` bash
# 设置 cgroup 参数同样需要注意路径的问题
$ cd /mnt/cgroups/memory
$ cgset -r memory.limit_in_bytes=2M mem

# 将进程运行于 cgroup 中
$ cgexec -g memory:mem top

# 将已有进程加入到 cgroup 中
$ cgclassify -g memory:mem <pid>

# 查看对应 cgroup 中运行的进程
$ cat /mnt/cgroups/memory/mem/tasks
```

### 使用 systemd 创建 cgroup

使用 systemd 来生成 cgroup 是更合适的做法。

systemd 提供了 slice ， service 和 scope 几个资源等级，其中 slice 是最上层的等级，默认情况下，系统会创建四种 slice ：

* -.slice ： 根 slice 。
* system.slice ： 所有系统 service 的默认位置。
* user.slice ： 所有用户会话的默认位置。
* machine.slice ： 所有虚拟机和 Linux 容器的默认位置。

我们可以通过 `systemd-cgls` 来查看 systemd 生成的 cgroup ，可以看到最小单位是 service 和 scope 。其中 service 是用户最常接触的 systemd 资源，一般通过 `.service` 文件定义具体的运行细节，然后交由 systemd 管控，而 scope 更多地用于用户进程，一般用户的交互式 shell 进程都由 scope 管理，并处于 user.slice 分片这个层级之中。

我们需要关注 `/sys/fs/cgroup/systemd` 这个目录，进入这个目录就可以看到标准的 cgroup 控制文件和下属的 slice ，再深入 slice 则是各自下属的 service 和 scope ， `/sys/fs/cgroup/systemd` 实际上就是一个 cgroup ，而 slice ， service 和 scope 就是 systemd 的子 cgroup 。但是由 systemd 管控的 cgroup 比较特殊，不会直接暴露它所拥有的控制器。

使用 systemd 生成 cgroup 我们无需关心 cgroupfs ，直接使用 systemd 提供的命令操作即可，并且后续都不应该直接操作这些 cgroup 。

``` bash
# 使用 systemd-run 创建临时 cgroup
$ systemd-run --unit=toptest --slice=test top -b
# 可以通过带 --scope 则生成 scope 等级的 systemd 资源，不带则默认是 service 等级
# 不指定 slice 则默认是 system.slice

# 生成的 unit 直接受 systemctl 管制
$ systemctl status toptest
● toptest.service - /usr/bin/top -b
   Loaded: loaded (/run/systemd/system/toptest.service; static; vendor preset: disabled)
  Drop-In: /run/systemd/system/toptest.service.d
           └─50-Description.conf, 50-ExecStart.conf, 50-Slice.conf
   Active: active (running) since Sat 2022-05-14 01:17:06 CST; 6min ago
 Main PID: 3708 (top)
   CGroup: /test.slice/toptest.service
           └─3708 /usr/bin/top -b
# 临时 systemd unit 在运行完成后会自动消亡
```

而如果是需要长期使用的 cgroup ，应该定义为 service 资源，并将需要的配置参数写入到 `.service` 文件，存放路径应为 `/usr/lib/systemd/system/` ，服务类型的软件一般都会自动将它们的服务定义文件安装到这个目录，这样就可以和 `systemd-run` 生成的 unit 一样，直接使用 `systemctl` 来操作。

``` bash
# 设置 systemd unit 的参数
# systemd 对 cgroup 参数做了封装，具体使用需要参考具体文档

# 设置 service 的参数
$ systemctl set-property sshd MemoryLimit=50M --runtime
# --runtime 允许临时改变现有的参数，重启系统或者删除运行时配置后参数会恢复原有设置
# 设置的参数会写入到 /run/systemd/system/sshd.service.d/ 下的配置文件中

# 不带 --runtime 会写入到独立的配置文件中，重启系统后也能生效
$ systemctl set-property sshd MemoryLimit=50M
$ cat /etc/systemd/system/sshd.service.d/50-MemoryLimit.conf 
[Service]
MemoryLimit=52428800

# 还可以通过手动修改 service 文件实现，和不带 runtime 效果一致
$ cat /etc/systemd/system/sshd.service.d/50-MemoryLimit.conf >> /usr/lib/systemd/system/sshd.service

# 新增参数和删除参数需要修改或者移除相关配置文件，然后重载服务
$ systemctl daemon-reload
$ systemctl restart sshd

# 如果是由 systemd-run 启动的资源，使用 --runtime 的意义不大，重启后都会失效，但依然可以实时改变参数
$ systemctl set-property toptest MemoryLimit=10M --runtime
$ systemctl status toptest
● toptest.service - /usr/bin/top -b
   Loaded: loaded (/run/systemd/system/toptest.service; static; vendor preset: disabled)
  Drop-In: /run/systemd/system/toptest.service.d
           └─50-Description.conf, 50-ExecStart.conf, 50-MemoryLimit.conf, 50-Slice.conf
   Active: active (running) since Sat 2022-05-14 17:19:07 CST; 17s ago
 Main PID: 2661 (top)
   Memory: 0B (limit: 10.0M)
   CGroup: /test.slice/toptest.service
           └─2661 /usr/bin/top -b
```

现有常用的容器进行时中， docker 默认会在 `/sys/fs/cgroup` 的各个子系统中自行维护 cgroup ，但可以通过修改 cgroup driver 来托管到 systemd 中，而 containerd 也存在类似的驱动设置。

在 docker 使用 `"exec-opts": ["native.cgroupdriver=systemd"]` 改变 cgroup driver 的情况下，新运行的容器会作为 system.slice 中的 scope 运行，如果使用默认的 cgroupfs 的情况下，会如前面所述，在 `/sys/fs/cgroup` 的各个子系统中创建 cgroup ，而不是托管在 systemd 中。

## 结语

总结来看，容器依赖内核功能来实现隔离能力， namespace 主要面向信息隔离， cgroup 主要面向资源隔离管控，在实际使用中，我们无需细致去关心它们的技术细节，但是了解它们的实现原理对后续学习是比较有帮助的。
