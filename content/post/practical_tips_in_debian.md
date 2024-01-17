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

## sshd 安全加固

sshd 基本上是 Linux 系统必定会运行的服务，一般会增加一些相关的安全配置。

``` bash
# 修改 sshd 服务的配置
$ cat /etc/ssh/sshd_config
# 禁止 root 用户登陆
PermitRootLogin no

# 禁用通过用户密码验证登陆
PasswordAuthentication no

# 启用用户私钥验证登陆
PubkeyAuthentication yes
# 验证私钥的公钥来源文件，一般不作修改
# AuthorizedKeysFile .ssh/authorized_keys

# AllowUsers 和 DenyUsers 可以限制用户登陆，其中 Deny 的优先级高于 Allow
# 除了用户还可以限制登陆来源 IP ，来源 IP 可以是具体 host 或者 CIDR
AllowUsers userA@192.168.0.0/24 userA@10.0.0.1
DenyUsers userB
```

### sshd 算法禁用

sshd 用到的一些算法可能已经不安全，所以需要手动禁用。

``` bash
# 查看默认支持的加密算法
$ ssh -Q cipher
# 查看默认支持的 mac 算法
$ ssh -Q mac
# 查看系统支持的密钥交换算法
$ ssh -Q kex

# 大部分情况下使用的 sshd 配置是默认的，可以先进行查询
$ sshd -T | grep -i cipher
$ sshd -T | grep -i macs
$ sshd -T | grep -i kexalgorithms

# 在原有查询的基础上修改 sshd 服务的配置
# 如果是禁用某些算法，直接把它去掉即可
# 如果是新增某些算法，则不应该超出 ssh -Q 查询到的对应算法范围
# 以下是 Debian 11 的默认 sshd 服务配置，可以在这个基础上修改
$ cat /etc/ssh/sshd_config
Ciphers chacha20-poly1305@openssh.com,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
MACs macs umac-64-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha1-etm@openssh.com,umac-64@openssh.com,umac-128@openssh.com,hmac-sha2-256,hmac-sha2-512,hmac-sha1
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256

# 一些已知的弱算法
Ciphers:
3des-cbc
aes128-cbc
aes192-cbc
aes256-cbc
blowfish-cbc
cast128-cbc

MACs:
hmac-md5
hmac-md5-96
hmac-md5-etm@openssh.com
hmac-md5-96-etm@openssh.com
umac-64-etm@openssh.com
umac-64@openssh.com

KexAlgorithms:
diffie-hellman-group-exchange-sha1
diffie-hellman-group1-sha1

# 修改后需要重启 sshd 服务，可以通过命令来验证是否成功
$ nmap --script ssh2-enum-algos -sV -p 22 127.0.0.1
$ ssh -o Ciphers=aes128-cbc root@127.0.0.1
$ ssh -o KexAlgorithms=diffie-hellman-group-exchange-sha1 root@127.0.0.1
```

### 启用 sftp

sshd 服务默认就会启动 sftp 服务，这里主要是对某些用户做一些额外的 sftp 适配。

``` bash
# 增加 sftp 用户并修改密码
$ useradd sftp -s /sbin/nologin -M
$ passwd sftp

# 修改 sshd 服务的配置
$ cat /etc/ssh/sshd_config
Subsystem sftp internal-sftp
Match User sftp
ForceCommand internal-sftp
X11Forwarding no
AllowTcpForwarding no
PasswordAuthentication yes
ChrootDirectory /data/sftp

# 创建 sftp 数据目录
$ mkdir -p /data/sftp
$ chown root:root /data/sftp
$ mkdir -p /data/sftp/sftp
$ chown sftp:sftp /data/sftp/sftp
$ chmod 750 /data/sftp/sftp
# 使用 sftp 用户登陆后会被限制在 /data/sftp 中，并且只能修改 /data/sftp/sftp 下的文件
```

## 增加 history 时间戳

``` bash
# 追加环境变量到 /etc/profile.d/history_timestamp.sh 文件中
$ cat /etc/profile.d/history_timestamp.sh
export HISTTIMEFORMAT="%F %T `whoami` "
```

## 修改 Linux 进程数限制

用户直接执行 `ulimit -a` 就可以看到对应的自身受到的进程相关限制，可以通过修改 `/etc/security/limits.conf` 来改变这个设置。

``` bash
$ cat /etc/security/limits.conf
# nproc 代表进程数量限制
# nofile 代表打开文件数量限制
user soft nproc 65536
user hard nproc 65536
user soft nofile 65536
user hard nofile 65536

# /etc/security/limits.conf 修改后需要确认 pam_limits.so 动态库正常加载
# 确认以下两个配置存在 required pam_limits.so
$ cat /etc/pam.d/sshd
$ cat /etc/pam.d/login
session required pam_limits.so
```

## 启用时间同步服务

Debian 一般会默认启用这个服务，这里的主要是为了修改 ntp 服务器地址。

``` bash
$ vi /etc/systemd/timesyncd.conf
[Time]
NTP=ntp.aliyun.com ntp1.aliyun.com
FallbackNTP=0.debian.pool.ntp.org 1.debian.pool.ntp.org 2.debian.pool.ntp.org 3.debian.pool.ntp.org

# 配置检查
$ timedatectl show-timesync --all
# 重启服务
$ systemctl restart systemd-timesyncd.service
```
