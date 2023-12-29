---
date: 2023-12-23 15:00:00
title: Debian 系统实用技巧笔记
tags:
  - "Linux"
  - "Debian"
draft: false
---

这篇笔记用来记录 Debian 系统中的一些实用技巧。

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

## 配置 Debian 源

dpkg 和 apt 是 Debian 系列 Linux 发行版的软件包管理器，其中 dpkg 是比较底层的软件包工具，而 apt 则是更高层级的管理工具，两者之间的关系类似于 rpm 和 yum 的关系。

``` bash
# 修改 Debian 10 buster 镜像源
$ echo 'deb http://mirrors.ustc.edu.cn/debian buster main contrib non-free
deb http://mirrors.ustc.edu.cn/debian buster-updates main contrib non-free
deb http://mirrors.ustc.edu.cn/debian buster-backports main contrib non-free
deb http://mirrors.ustc.edu.cn/debian-security/ buster/updates main contrib non-free' \
  > /etc/apt/sources.list

# 更新软件列表
$ apt update
```

### 配置不稳定镜像源获取最新版本软件

为了能够使用一些版本比较高的软件，有时候可能需要将镜像源配置为不稳定版本，在 Debian 中不稳定版本的镜像源代号为 sid ，这个镜像源通常包括了一些最新版本但是还不稳定的软件。

``` bash
# 添加 Debian sid 镜像源
$ echo deb https://mirrors.aliyun.com/debian/ sid main contrib non-free non-free-firmware \
  > /etc/apt/sources.list.d/sid.list

# 更新软件列表
$ apt update
```

### 配置镜像源的优先级

虽然可以通过添加 sid 镜像源来获取最新软件，但是这个操作是有一定危险性的，所以在添加 sid 源后千万不要使用 `apt upgrade` 更新所有软件，最好在安装完需要的某部分软件后，把 sid 源的优先级降低。

``` bash
# 添加镜像源的优先级配置
$ cat /etc/apt/preferences.d/sid 
Package: *
Pin: release o=Debian,a=unstable,n=sid
Pin-Priority: 50

# 更新软件列表
$ apt update

# 默认的镜像源优先级是 500 ，越大的优先级会被优先使用
# 以下输出是一台调整过优先级的 Debian 11 系统的运行信息，可以看到 sid 源的优先级已经被降低
$ apt-cache policy
Package files:
 100 /var/lib/dpkg/status
     release a=now
  50 https://mirrors.aliyun.com/debian sid/non-free-firmware amd64 Packages
     release o=Debian,a=unstable,n=sid,l=Debian,c=non-free-firmware,b=amd64
     origin mirrors.aliyun.com
  50 https://mirrors.aliyun.com/debian sid/non-free amd64 Packages
     release o=Debian,a=unstable,n=sid,l=Debian,c=non-free,b=amd64
     origin mirrors.aliyun.com
  50 https://mirrors.aliyun.com/debian sid/contrib amd64 Packages
     release o=Debian,a=unstable,n=sid,l=Debian,c=contrib,b=amd64
     origin mirrors.aliyun.com
  50 https://mirrors.aliyun.com/debian sid/main amd64 Packages
     release o=Debian,a=unstable,n=sid,l=Debian,c=main,b=amd64
     origin mirrors.aliyun.com
 500 http://deb.debian.org/debian bullseye-updates/main amd64 Packages
     release v=11-updates,o=Debian,a=oldstable-updates,n=bullseye-updates,l=Debian,c=main,b=amd64
     origin deb.debian.org
 500 http://security.debian.org/debian-security bullseye-security/main amd64 Packages
     release v=11,o=Debian,a=oldstable-security,n=bullseye-security,l=Debian-Security,c=main,b=amd64
     origin security.debian.org
 500 http://deb.debian.org/debian bullseye/main amd64 Packages
     release v=11.8,o=Debian,a=oldstable,n=bullseye,l=Debian,c=main,b=amd64
     origin deb.debian.org
```

### 一些重要软件的独立镜像源

这里会记录一些重要软件的独立镜像源，以便需要时能够快速安装。

``` bash
# jenkins Debian LTS release
$ curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  > /usr/share/keyrings/jenkins-keyring.asc
$ echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" \
  > /etc/apt/sources.list.d/jenkins.list
$ apt update
$ apt install jenkins

# docker-ce
$ curl -fsSL https://download.docker.com/linux/debian/gpg \
  > /usr/share/keyrings/docker-ce-keyring.asc
$ echo "deb [arch="$(dpkg --print-architecture)" signed-by=/usr/share/keyrings/docker-ce-keyring.asc] \
  https://download.docker.com/linux/debian "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" \
  > /etc/apt/sources.list.d/docker-ce.list
$ apt update
$ apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# openresty
$ curl -fsSL https://openresty.org/package/pubkey.gpg \
  > /usr/share/keyrings/openresty-keyring.asc
$ echo "deb [signed-by=/usr/share/keyrings/openresty-keyring.asc] http://openresty.org/package/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" openresty" \
  > /etc/apt/sources.list.d/openresty.list
$ apt update
$ apt install openresty

# mysql
$ wget https://dev.mysql.com/get/mysql-apt-config_0.8.15-1_all.deb
# mysql-apt-config 实际上就是安装对应的 apt 镜像源和签名信息的步骤
# 0.8.15 版本的 mysql-apt-config 可以下载安装 mysql 5.7 和 mysql 8.0
# 新版本的 mysql-apt-config 只能下载安装 mysql 8.0
$ dpkg -i mysql-apt-config_0.8.15-1_all.deb
# 安装完成后会跳出图形界面，可以自行选择需要的镜像源，这里选择的是 mysql 5.7
# 配置完成后想要修改的话可以通过以下命令重新打开
$ dpkg-reconfigure mysql-apt-config
$ apt update
$ apt install mysql-community-server
```

## 调整网卡配置

Debian 系统使用的网卡配置文件是 `/etc/network/interfaces` 和 `/etc/network/interfaces.d/*` ，一般会在这里调整网卡的 IP 地址获取行为。

``` bash
# 默认的 /etc/network/interfaces 配置文件
$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens33
iface ens33 inet dhcp

# 调整网卡为静态 IP 地址后的 /etc/network/interfaces 配置文件
$ cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens33
auto ens33
iface ens33 inet static
address 192.168.1.2/23
gateway 192.168.1.1
dns-nameservers 192.168.1.1
```

## vim 兼容性问题

在 Debian 的一些版本中默认安装的编辑器是 vim-tiny ，可能会出现比如上下左右方向键变成 ABCD 的字符输入，退格键不能正常删除字符，或者文本粘贴出现异常缩进，可以通过修改 `vimrc` 来解决。

``` bash
$ cat ~/.vimrc
"解决文本粘贴出现异常缩进"
set paste
"关闭兼容模式，解决上下左右方向键变成 ABCD 的字符输入"
set nocompatible
"解决退格键不能正常删除字符"
set backspace=2
```
