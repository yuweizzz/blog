---
date: 2024-11-28 10:13:10
title: 替换 VirtualBox 内置网卡 ROM
tags:
  - "PXE"
  - "VirtualBox"
draft: false
---

替换 VirtualBox 内置网卡的 ROM ，解决 iPXE 功能缺失问题。

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

以下内容出自这个[文档](https://frankchang.me/2017/06/27/replace-virtualbox-pxe-rom/)。

> VirtualBox 中網路卡預設的 BootROM 是使用 iPXE，但是這個預設的 iPXE 只有少少的功能，例如在無碟開機所需要的 iSCSI 功能沒有編進去。當然我們可以利用 DHCP + TFTP 送 iPXE Image 做 PXE chainloading，但是實務上就會遇到一些問題，最主要是無法確定現在用的究竟是哪個 iPXE，因為 user-class 通通都是 "iPXE"。

最近在研究 PXE 的相关问题，在 VirtualBox 面临的最大问题是无法辨别 iPXE 是否为我们自行编译的版本。比较好的办法就是彻底禁用原生的 iPXE ，而 VirtualBox 提供的 VirtualBox Extension Pack 中就带有 Intel 网卡的 BootROM ，可以在[官方网站](https://www.virtualbox.org/wiki/Downloads)下载。

下载之后可以直接打开执行安装，默认会安装到 VirtualBox 的同级路径中，然后就可以通过命令进行替换，参考以下步骤，需要注意以下步骤是在 Windows 系统下执行的：

```powershell
# 查看虚拟实例的 name/uuid
C:\Program Files\Oracle\VirtualBox>VBoxManage.exe list vms

# VBoxManage Usage
C:\Program Files\Oracle\VirtualBox>VBoxManage.exe setextradata
Usage - Set a keyword value that is associated with a virtual machine or configuration:

  VBoxManage setextradata <global | uuid | vmname> <keyword> [value]

# 通过 uuid 执行对应实例的 ROM 替换
C:\Program Files\Oracle\VirtualBox>VBoxManage.exe setextradata b108481a-68dc-4067-9367-763515aaad70 VBoxInternal/Devices/pcbios/0/Config/LanBootRom "c:\Program Files\Oracle\VirtualBox\ExtensionPacks\Oracle_VirtualBox_Extension_Pack\PXE-Intel.rom"
```
