---
date: 2021-03-30 21:00:00
title: 安装 Linux 图形界面
tags:
  - "Linux"
draft: false
---

这篇笔记主要介绍如何为 CentOS 安装图形界面。

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

## 图形界面的运行原理

Linux 各类发行版图形界面的理论支持就是大名鼎鼎的 X Window System ，它是基于 Server/Clinet 架构的一套复杂软件，主要的工作组件是 X Server 和 X Client 。

```
#  X Window System 架构:

+----------+        +----------+        +----------+
| Hardware |<------>| X Server |<------>| X Client |
+----------+        +----------+        +----------+
```

X Window System 的两个主要组件的工作内容如下：

* X Server：和硬件层面对接，从输入设备(键盘，鼠标)获取输入数据并告知 Client ，在输出设备(显示器)上绘制从 Client 获取的绘图数据。
* X Client：从 Server 获取到硬件的输入数据，处理它们得出对应的绘图数据返回给 Server 。

X Server 是相对固定的一套软件，和硬件交互的主要是各类硬件驱动的使用和管理。

而 X Client 就十分地自由，它可以是浏览器，办公软件，播放器等等。一个 X Client 可以认为是图形界面中的一个软件窗口。

我们经常听到的 KDE ， GNOME ， Xfce 是特殊的 X Client 套件，通常认为是窗口管理器，因为它们可以统筹管理同一界面下的多个 X Client 。

所以如果要为一台机器安装带图形界面的 Linux 系统，我们会需要安装 X Window System 组件以提供 X Server ，然后安装 KDE 这一类特殊的 X Client 套件，组成一套可用的图形界面。

## 传统Server/Clinet架构和X Window System架构的区别

### 传统Server/Clinet架构

在传统的 Server/Clinet 架构中，最经典应用应该是 HTTP 服务。

在 HTTP 服务中， Server 一般是运行在远端的 HTTP 服务器，比如 nginx 或者 Apache 。我们在 PC 端向 Server 发起请求， Server 在接受请求后响应并返回数据到 PC 端。

从使用者的视角来看， Client 很好理解，无疑就是自己使用的 PC 设备，而 Server 就是远端提供服务的设备。

### X Window System架构

当一台 Linux 系统的机器运行图形界面时，它的 X Server 和 X Client 都会运行在自己的本地机器上，这种应用情况在传统的 Server/Clinet 架构中比较少。

此外还有一种情况就是和 HTTP 服务类似， X Server 和 X Client 不在同一台机器上运行，而是通过网络通信，但这种 X Window System 应用情况会比较少。

```
#  运行在网络上的 X Window System 架构:


+----------+                                             +----------+
|  Input   |                                             |  Input   |
+----------+                                             +----------+
     |                                                        |
     |                 +---------------------+                |
     v                 |  remote workstation |                v
+----------+  Network  +----------+----------+  Network  +----------+ 
| X Server |<--------->| X Client | X Client |<--------->| X Server |
+----------+           +----------+----------+           +----------+
     |                 |     ...........     |                |
     |                 +---------------------+                |
     v                                                        v
+----------+                                             +----------+
|  Output  |                                             |  Output  |
+----------+                                             +----------+
```

在 X Window System 架构运行在网络上时， X Server 和 X Client 会通过网络传输数据。虽然 X Server 的服务对象依旧是 X Client ，但这时它们是分开在网络的两端的，站在使用者的角度上，远端不再是常见的 Server ，反而是各类 X Client ，它们会去完成软件的数据计算工作。而我们本地设备则成为了 X Server ，管理着本地硬件并且承担绘制图形的工作。

这就是 X Window System 比较特殊的地方，但它本质上还是遵循着 Server 为 Client 提供服务的原则。

## 使用 CentOS 图形界面

### 安装本地图形界面

在市面上的不同 Linux 发行版中，在个人用户中比较受欢迎的是 Ubuntu ，但是我个人习惯使用的 Linux 发行版是 CentOS ，它更多地用于服务器的场景，而且一般不会安装图形界面，所以需要额外去探究一下。

对于发行版的选择，我认为贴近自己的工作场景是比较重要的参考。 CentOS 是稳定的服务器系统，而 Ubuntu 使用的内核比较新，图形界面的支持比较丰富。此外还可以 Debian 或者 Fedora 这些发行版可供选择。

我个人推荐是 CentOS 或 Ubuntu 中任意一款，因为这些发行版的用户相对较多，各类支持和解决方案也相应地比其他发行版来的多。

如前面所提到的，我们需要安装 Server 和 Client 两个组件。

CentOS 如果没有安装过图形界面，需要先安装 X Window System ，它提供了 X Server 的功能。

还需要安装 X Client ，可以选择只安装个人需要的 X Clinet，但一般会都会安装主流的桌面套件提供完整的支持，我的个人选择的桌面套件是 xfce 。

``` bash
#  前置的软件安装工作：

$ yum group install 'x window system' -y
$ yum group install xfce -y  # 这里可选择其他套件：GNOME Desktop or KDE Plasma Workspaces
# 安装完成后，切换到图形界面的两种方式：
$ init 5
$ systemctl isolate graphical.target

# 安装完成后，可以设置默认启动环境为图形界面
$ systemctl get-default  # 默认应该为多用户界面 
multi-user.target
$ systemctl set-default graphical.target
$ reboot  # 修改设置后重启生效
```

### 运行时分析

如果安装没有报错，我们在执行切换命令后就会自动切换到图形界面。

> 在某些版本中，可能会出现无法立即切换图形的情况，解决办法一般是先切换为原来的运行等级，在终端界面会产生文本模式的会话，选择接受 licence 后再尝试切换。 

为了更好地了解运行的原理，我们可以抓取进程信息进行分析，看看服务是如何运行的。

``` bash
#  切换图形界面后的进程信息：

$ ps aux | grep X
root       1002  0.0  0.2 225840  4812 ?        Ss   16:55   0:00 /usr/bin/abrt-watch-log -F Backtrace 
  /var/log/Xorg.0.log -- /usr/bin/abrt-dump-xorg -xD
root       1597  0.5  2.4 340104 49876 tty1     Ssl+ 16:56   0:01 /usr/bin/X :0 -background none -noreset 
  -audit 4 -verbose -auth /run/gdm/auth-for-gdm-VsIE6S/database 
  -seat seat0 -nolisten tcp vt1  # X server 的进程详情
root       2060  0.0  0.0 112812   948 pts/1    S+   17:00   0:00 grep --color=auto X
```

如果我们已经启动了图形界面，那么这台机器上必然已有一个 X 进程，这时我们需要关注一个环境变量 DISPLAY ，它为所有的 X Client 提供 X Server 的通信地址。

这里有个地方比较有意思，如果在图形界面下的终端打印这个变量，可以看到格式为 hostname:displaynumber.screennumber 的值，但是如果你切换到命令行界面并打印这个变量，会发现结果为空。那是因为这个变量在命令行界面是无意义的，而它对图形界面运行的 Client 却是不能缺少的，如果没有它， Server 和 Client 之间就无法通信，所以新的 Client 都会继承这个环境变量。

关于 DISPLAY 这个环境变量，我们可以在 X Server 的文档中查到相关的信息：

* hostname 如果是为空，那么它默认会使用本地 X Server 中的最高效通信方式 Unix Socket ，实际使用中这个值经常留空。
* displaynumber 用来界定显示器和输入设备的集合，用于区分不同的显示界面，是不能为空的。
* screennumber 用来界定某个集合中不同的显示器，比如多显示器使用主副屏的情况。考虑到现实情况中，用户通常只有一个显示器， screennumber 留空则默认 0 。

所以可以看到实际中的 DISPLAY 一般是类似于 :0 ， :1 ， :10 这样的值，系统默认启动的 X 进程 DISPLAY 就是 :0 ，符合前面所描述的使用习惯。此外 displaynumber 一般还会作为这个 X Server 监听的端口号偏移量。 

X Server 的默认监听端口是 6000 ，如果是指定了 DISPLAY 的 X Server ，那么除了端口已被占用的情况，一般会在 6000 的 displaynumber 偏移端口上进行监听，当指定 DISPLAY 为 :1 会监听在 6001 端口，指定 DISPLAY 为 :10 会监听在 6010 端口，这个部分可以通过 netstat 查看系统的监听情况。 

再说回系统默认启动的 X 进程，我们可以在这个进程参数中看到 DISPLAY 的值为 :0 。如果依前面所述，此时应该可以查询到监听在 6000 端口的 X 进程。但是这时我们通过 netstat 检查端口，会发现这个端口并不会被占用。出现这种现象的原因是系统默认启动的 X 进程带有 -nolisten tcp 参数，禁用了 tcp 监听功能。

如果我们自行启动一个 X 进程，它默认是不带 -nolisten tcp 参数的。我们可以自行定义 DISPLAY 变量，例如 X :1 ，就可以捕捉到监听在 6001 端口的 X 服务。

### 基于网络运行图形界面

在现实使用中，为用户提供远程桌面连接一般使用基于 RFB 协议的 VNC 或者基于 RDP 协议的 Xrdp ，其实可以直接使用 X Window System 来实现。

在 X Server 和 X Client 同在一台机器上时，它们可以通过 UNIX socket 来实现通信。如果要实现 X Server 和 X Client 远程通信，则需要通过 TCP socket 来实现。

远程通信还需要解决安全问题， X Window System 自身提供多种安全认证的机制，但是最安全的方式是使用 OpenSSH X11 Forwarding 。

依据前面的理论，要基于网络运行图形界面，本地机器需要启动 X Server 服务，远端机器提供 X Client ，从远端的 X Client 到本地的 X Server 要使用 X11 Forwarding 来解决安全问题。 X11 Forwarding 实际上就是端口转发。我们可以先进行实践，再进行分析探究。

以下是需要在本地机器和远端机器进行的工作，完成后可以直接启动远程图形界面：

---

* 远端需要完成的工作

首先需要为远端机器安装 X Client ，可以自行选择安装需要的桌面套件或者只安装需要的 X Client ，然后开启 SSH daemon 的 X11 Forwarding 支持。

``` bash 
$ yum group install xfce -y  # 安装 X Client 
$ vi /etc/ssh/sshd_config 
......
X11Forwarding yes  # 开启 X11 Forwarding 
......
$ systemctl restart sshd  # 开启后重启 daemon 使配置生效
```

我们平时在使用 ssh 去登录远程机器时，可能会出现 WARNING! The remote SSH server rejected X11 forwarding request. 的报错，这就说明远程机器的 X11 Forwarding 功能被禁用了。如果要在网络上运行图形界面，那么 X Client 所在的远程机器必须打开这个功能。

---

* 本地需要完成的工作

为了不与现有的图形环境冲突，启动一个新的 X 进程。

``` bash
$ X :1 &  # 指定 DISPLAY 运行 X ，这里 DISPLAY 可以任意取值，尽量避免端口冲突
```

然后设置 DISPLAY 环境变量，这个是必须的步骤，它会作为 ssh 设置端口转发的依据。

设置完成后直接发起带 X11 Forwarding 的 ssh 登录请求，在登录成功后就可以启动远程 X Client 了。

``` bash
$ export DISPLAY=:1  # 这里的值和前面 X 的 DISPLAY 值需要保持一致，这个变量设置错误会导致转发出错
$ ssh -X user@remotehost  # -X 启动 X11 Forwarding 功能，远端此时应该已经开启 X11Forwarding yes
......  # 进入远程连接
[user@remotehost ~]$ startxfce4  # 启动 xfce 桌面，此时会在本地 X Server 中显示远程的 xfce 桌面
[user@remotehost ~]$ firefox  # 启动单一的 X Client 而不是桌面套件，此时会在本地 X Server 中开启 firefox 浏览器
```
---

### 运行时分析

按照以上的步骤，可以建立起两台机器之间的 X Window System 通信。下面进行简单的分析：

* 在 X Server 端所在的机器，如果启动 X 进程而且不关闭 tcp 监听，通过 netstat 抓取到的端口号为 DISPLAY 的 6000 加上 displaynumber 。如果启动的 X 进程关闭了 tcp 监听，就无法抓取到监听端口，作为替换的结果是你可以追踪到一个打开的 Unix Socket 。

* 在本地使用 startxfce4 启动图形界面后，使用命令行可以输出得到 DISPLAY 值为 6000 加上 X11DisplayOffset 。 X11DisplayOffset 和 X11Forwarding 一样定义在 X Client 所在的机器的 /etc/ssh/sshd_config 中。

我们可以知道大概的运行过程是这样：在设置了 DISPLAY 环境变量的 shell 启动 ssh -X 会话后，这个会话启动时会把 DISPLAY 的值进行端口映射，远程的映射信息是在远程的 /etc/ssh/sshd_config 中设置的。 ssh 连接成功就意味着端口转发的建立，在远程机器上的 X Client 并不感知这个端口已经被转发，它会正常地进行通信，并被本地的 X Server 显示。

## 总结

X Window System 是 Linux 上运行图形界面的基础，这个是使用图形界面必要了解的理论部分。

在本地使用图形界面比较简单，大部分情况下可以之间安装对应软件即可之间使用。比较特殊还是在远程运行图形界面，需要进行一些配置的修改才能成功，但是 X Window System 的性能比较一般，对图形界面有更高的要求一般不使用。

所以这篇笔记适用于最小安装的本地环境或者简单的内网环境。

