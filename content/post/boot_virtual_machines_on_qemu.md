---
date: 2024-11-11 15:14:10
title: 通过 QEMU 运行虚拟实例
tags:
  - "Linux"
  - "QEMU"
draft: false
---

在 Linux 系统中通过 QEMU 运行虚拟实例。

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

## 编译 QEMU

```shell
# 安装依赖
apt install \
  gcc \
  make \
  python3-venv \
  python3-sphinx \
  python3-sphinx-rtd-theme \
  ninja-build \
  gcc-multilib \
  libglib2.0-dev \
  git \
  flex \
  bison

# 下载源码
wget https://download.qemu.org/qemu-9.1.1.tar.xz
tar -xvf qemu-9.1.1.tar.xz
cd qemu-9.1.1

# 设置编译选项，仅需要编译 aarch64-softmmu 和 x86_64-softmmu
./configure --enable-kvm --enable-debug --enable-werror --target-list="aarch64-softmmu,x86_64-softmmu"
make -j $(nproc)

# 检查编译后的可执行文件
ls build/qemu-system-*

# 这里省去安装步骤，如果需要长期保留使用可以执行安装
# make install
```

## 搭建桥接网络

当前 QEMU 宿主机是运行在一个 NAT 网络下的 VirtualBox 虚拟实例，在这台机器启动的 QEMU 实例，直接通过桥接连通到这个 NAT 网络，这样 NAT 网络下的 VirtualBox 虚拟实例和嵌套的 QEMU 实例就处于同一个子网下，可以直接互相通信。

```shell
# 手动添加网桥
ip link add name br0 type bridge
ip link set br0 up
# 非常重要的配置步骤，如果物理网卡不打开混杂模式，数据包会被丢弃，绑定网桥后会中断网络
# VirtualBox 则必须要在虚拟电脑中的网络配置开启混杂模式，单纯通过系统内命令行设置无效
ip link set dev eth0 promisc on
# 将物理网卡绑定到网桥上
ip link set dev eth0 master br0
# 通过 DHCP 获取网桥设备的 IP 地址
dhclient br0


# 添加 tap 设备并绑定到网桥上
ip tuntap add dev tap0 mode tap
ip link set tap0 up
ip link set dev tap0 master br0


# 持久化配置
cat /etc/network/interfaces
# --- START: /etc/network/interfaces ---
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

allow-hotplug eth0
# iface eth0 inet dhcp
iface eth0 inet manual

auto tap0
iface tap0 inet manual
  pre-up /sbin/ip tuntap add dev $IFACE mode tap || true
  up /sbin/ip link set dev $IFACE up
  post-down /sbin/ip link del dev $IFACE || true

auto br0 
iface br0 inet dhcp 
  bridge_ports eth0 tap0
  bridge_stp off
  bridge_fd 0
# --- END: /etc/network/interfaces ---
systemctl restart networking
```

## 运行实例

### x86

因为宿主机本身就是 x86 架构，所以可以使用 kvm 运行实例。

```shell
# 创建 qcow2 格式的镜像来充当系统盘
/opt/qemu-9.1.1/build/qemu-img create -f qcow2 /opt/x86.qcow2 10G

# 创建 x86 实例
/opt/qemu-9.1.1/build/qemu-system-x86_64 \
  --enable-kvm -smp 1 -m 1024M \
  -device nec-usb-xhci -device usb-kbd -device usb-tablet \
  -netdev tap,id=eth0,ifname=tap0,script=no,downscript=0 \
  -device virtio-net-pci,netdev=eth0,bootindex=0 \
  -drive if=none,file=/opt/x86.qcow2,id=hd0 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0,bootindex=1 \
  -nographic
```

成功执行命令后，搭配之前写过的 [PXE 安装工具](https://github.com/yuweizzz/go-pxe-installer)，很快就可以得到一台全新的可用实例。

具体的参数含义参考：

- `--enable-kvm` 是最重要的选项，允许使用 kvm 模块进行加速，这样虚机运行速度可以接近原生系统。
- `-smp` 用来指定虚机的 CPU 核数， `-m` 用来指定虚机的内存大小。
- `-device` 用来创建虚机中的设备， `-drive` 或者 `-netdev` 用来关联宿主机中和对应虚机的相关设备。
- `-device nec-usb-xhci -device usb-kbd -device usb-tablet` 用来设置 USB 设备。
- `-netdev tap,id=eth0,ifname=tap0,script=no,downscript=0` 指定了宿主机中的 tap 设备作为虚机的实际关联设备，具体的 iface 是我们前面创建的 tap0 ，而通过自定义的 id ，可以和 `-device virtio-net-pci,netdev=eth0,bootindex=0` 进行关联，在虚机中，它是一个挂载于 pci 总线上的网络设备，并且指定了启动顺序。
- `-device virtio-scsi-pci -device scsi-hd,drive=hd0,bootindex=1` 这里是关于硬盘设备的配置，这里使用了 `virtio-scsi-pci` 作为中间设备，具体的硬盘设备会作为 scsi 子设备连接到 scsi 总线，这样更加接近物理服务器的架构，而硬盘设备就是前面创建的 qcow2 镜像，通过 `-drive if=none,file=/opt/x86.qcow2,id=hd0` 指定。
- `-nographic` 用来关闭图形界面，虚拟机的输出信息会自动重定向到标准输出中。

如果想要使用 UEFI 模式则可以参考以下步骤：

```shell
# 创建 qcow2 格式的镜像来充当系统盘
/opt/qemu-9.1.1/build/qemu-img create -f qcow2 /opt/x86.qcow2 10G

# 安装 UEFI 固件，其中固件内容可以复用，而每个实例的 EFI 变量应该独立保存
apt install ovmf
ls /usr/share/OVMF/OVMF_CODE.fd
cp /usr/share/OVMF/OVMF_VARS.fd /opt/OVMF_VARS.fd

# 创建 UEFI 模式下的 x86 实例
/opt/qemu-9.1.1/build/qemu-system-x86_64 \
  --enable-kvm -smp 1 -m 1024M \
  -drive "if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd" \
  -drive "if=pflash,format=raw,file=/opt/OVMF_VARS.fd" \
  -device nec-usb-xhci -device usb-kbd -device usb-tablet \
  -netdev tap,id=eth0,ifname=tap0,script=no,downscript=0 \
  -device virtio-net-pci,netdev=eth0,bootindex=0 \
  -drive if=none,file=/opt/x86.qcow2,id=hd0 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0,bootindex=1 \
  -nographic         
```

具体的执行命令和 BIOS 模式下接近，只有额外指定了 UEFI firmware 的区别。

对于启动顺序，这里都设置了网络优先启动，在没有系统的情况下，需要先通过 PXE 启动安装系统，等待安装完成后可以将其改为硬盘优先，或者完全去掉关于启动顺序的配置，交由默认的 BIOS 配置决定。

UEFI 模式下除了手动指定启动顺序之外，还可以交由 EFI 变量来控制。如果 EFI 变量缺失或者启动顺序异常，可以参考[这里](https://yuweizzz.github.io/post/convert_legacy_bios_to_uefi/#%E4%BF%AE%E6%94%B9%E5%90%AF%E5%8A%A8%E6%A8%A1%E5%BC%8F%E5%92%8C-uefi-%E5%90%AF%E5%8A%A8%E9%A1%B9)来重新修改。

此外还有一些 QEMU 运行时的错误问题，参考以下解决方法：

- 运行时出现 `access denied` 的网络报错，需要执行 `echo allow br0 > /opt/qemu-9.1.1/build/qemu-bundle/usr/local/etc/qemu/bridge.conf` ，如果是已经 `make install` 的情况下，则需要执行 `echo allow br0 > /etc/qemu/bridge.conf` 。
- 运行时出现 BIOS 相关文件缺失的报错，可以尝试添加 `-L /opt/qemu-9.1.1/pc-bios/` 来解决报错。

### arm64

由于是异构平台，实际上是没有办法使用 kvm 进行加速的，并且没有找到 arm64 的可用 BIOS 固件，这里使用的是 UEFI 模式。

```shell
# 创建 qcow2 格式的镜像来充当系统盘
/opt/qemu-9.1.1/build/qemu-img create -f qcow2 /opt/aarch64.qcow2 10G

# 下载 UEFI 固件，使用的是 Linaro Releases 编译发布的开源 UEFI firmware
wget https://releases.linaro.org/components/kernel/uefi-linaro/16.02/release/qemu64/QEMU_EFI.fd

# 填充 UEFI 固件，满足 qemu 的 arm64 pflash 大小限制
truncate -s 64M /opt/arm64_vars.fd
truncate -s 64M /opt/arm64_code.fd
dd if=/opt/QEMU_EFI.fd of=/opt/arm64_code.fd conv=notrunc

# 创建 UEFI 模式下的 arm64 实例
/opt/qemu-9.1.1/build/qemu-system-aarch64 \
  -M virt -cpu cortex-a76 \
  -smp 1 -m 1024M \
  -drive "if=pflash,format=raw,readonly=on,file=/opt/arm64_code.fd" \
  -drive "if=pflash,format=raw,file=/opt/arm64_vars.fd" \
  -device nec-usb-xhci -device usb-kbd -device usb-tablet \
  -netdev tap,id=eth0,ifname=tap0,script=no,downscript=0 \
  -device virtio-net-pci,netdev=eth0,bootindex=0 \
  -drive if=none,file=/opt/aarch64.qcow2,id=hd0 \
  -device virtio-scsi-pci -device scsi-hd,drive=hd0,bootindex=1 \
  -nographic
```

具体的命令配置和 x86 的没有特别大的区别，这里只是使用 virt 作为 Machine type 以及 CPU model 则选用了 Cortex-A 系列的 CPU ，实际上 x86 也可以做到指定对应的 Machine type 和 CPU model ，不过一般保持默认值就足够了。
