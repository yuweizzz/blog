---
date: 2021-07-03 15:00:00
title: 由 BIOS 传统启动向 UEFI 启动迁移
tags:
  - "Linux"
  - "UEFI"
draft: false
---

将 MBR+BIOS 启动的硬盘转换为 GPT+UEFI 启动。

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

现代设备中，采用 UEFI 启动的机器越来越多了，虽然在服务器中可以通过 CSM 来提供 Legacy BIOS 的支持，但是 UEFI 将会是未来使用的主流方式，所以有必要开始尝试使用 UEFI 启动。

## 将MBR转换为GPT

和 UEFI 启动模式紧密配合的分区方式是 GPT ，原有的 MBR 分区方式存在一定的局限性。而 GPT 为了保持兼容，使用了 LBA 0 扇区作为 MBR 保护扇区，这也使得 MBR 分区方式迁移到 GPT 分区方式成为可能。

将 MBR+BIOS 启动的硬盘转换为 GPT+UEFI 启动的第一步工作就是将 MBR 转换为 GPT 。

**进行分区操作时，一定要小心谨慎，做好数据备份，对根分区的错误操作可能是无法挽救的。**

```
# 首先观察根分区情况，这时可以使用 fdisk 或 parted
# 这个硬盘来自我的虚拟机

# fdisk /dev/sda
...
Command (m for help): p

Disk /dev/sda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0004dcb6

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048     3123199     1048576   83  Linux
/dev/sda3         3123200    20971519     8924160   83  Linux

# parted -l
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system     Flags
 1      1049kB  525MB   524MB   primary  xfs             boot
 2      525MB   1599MB  1074MB  primary  linux-swap(v1)
 3      1599MB  10.7GB  9138MB  primary  xfs
```

如果你的系统盘如上面展示的一样，最后一个分区用尽了所有剩余的硬盘扇区，这种情况下不建议进行分区方式的转换，因为根据 GPT 分区格式的定义， GPT 比 MBR 使用到了更多的硬盘起始扇区存放分区信息 (Primary GPT) ，而且还占用了硬盘末尾的部分扇区作为分区信息的备份 (Secondary GPT) 。

虽然很多分区工具在操作 msdos 硬盘时会保留一些起始扇区，默认从 1 MB 开始分区，以便后续兼容 GPT 格式，但一般不会对末尾扇区做保留。所以在转换 GPT 时，处于硬盘头部的分区通常可以无损转换，而处于硬盘尾部的分区会遭遇文件系统受损。

```
# 尝试对这个无法无损转换的硬盘进行操作
# gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: MBR only
  BSD: not present
  APM: not present
  GPT: not present


***************************************************************
Found invalid GPT and valid MBR; converting MBR to GPT format
in memory. THIS OPERATION IS POTENTIALLY DESTRUCTIVE! Exit by
typing 'q' if you don't want to convert your MBR partitions
to GPT format!
***************************************************************


Warning! Secondary partition table overlaps the last partition by
33 blocks!
You will need to delete this partition or resize it in another utility.

# 这里已经给出分区告警：检测到无效的 GPT 和有效的 MBR ，
# 分区工具将分区表自动转换为 GPT 格式，并且最后一个分区需要重新划分，
# 在退出分区工具时可以按 q 来取消这次转换

# 为了文件系统安全，记得通过按下 q 来退出。
```

虽然在我的虚拟机硬盘上无法直接实现转换，但是如果你的系统盘没有用尽所有的空间，或者最后一个分区用作交换分区时，分区方式就可以实现完美转换。

```
# 对可以无损转换的硬盘进行操作，原有分区信息大概如下
# 这是一块 msdos 分区的硬盘，未用尽所有扇区
# 这里为了实验删除了 swap 分区，实际上这一步可以省去，直接建立新分区和直接转换是没问题的
...
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         4196351   2.0 GiB     8300  Linux filesystem
   2         4196352       903086079   428.6 GiB   8300  Linux filesystem
   3       904110080       905134079   500.0 MiB   8200  Linux swap
...

# 直接通过 gdisk 进行操作
# gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.10

Partition table scan:
  MBR: MBR only
  BSD: not present
  APM: not present
  GPT: not present


***************************************************************
Found invalid GPT and valid MBR; converting MBR to GPT format
in memory. THIS OPERATION IS POTENTIALLY DESTRUCTIVE! Exit by
typing 'q' if you don't want to convert your MBR partitions
to GPT format!
***************************************************************


Command (? for help): d
Partition number (1-3): 3

Command (? for help): n
Partition number (3-128, default 3): 3
First sector (34-976773168, default = 904110080) or {+-}size{KMGTP}: 34
Last sector (34-2047, default = 2047) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): ef02
Changed type of partition to 'BIOS boot partition'

Command (? for help): n
Partition number (4-128, default 4): 4
First sector (904110080-976773168, default = 904110080) or {+-}size{KMGTP}: 
Last sector (904110080-976773168, default = 976773168) or {+-}size{KMGTP}: +500M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): ef00
Changed type of partition to 'EFI System'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sda.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot.
The operation has completed successfully.

# 执行 partprobe 可以得到新的分区信息如下：
...
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         4196351   2.0 GiB     8300  Linux filesystem
   2         4196352       903086079   428.6 GiB   8300  Linux filesystem
   3              34            2047   1007.0 KiB  EF02  BIOS boot partition
   4       904110080       905134079   500.0 MiB   EF00  EFI System
...
# 这里没有恢复 swap 分区，实际操作时可以按需求来恢复。
```

相比原有的三个分区，只需要新增 BIOS boot partition 和 EFI System 两个新分区即可，它们会在后续派上用场。

## 挂载 ESP

接下来需要挂载 ESP ，也就是刚才新增的 EFI System 分区，它的全称是 EFI System Partition ，是预留给 UEFI 启动存放引导文件的分区。

``` bash
# 格式化 ESP ，注意它的文件系统是 FAT 格式
mkfs.vfat /dev/sda4
# 如果命令不存在，安装新的软件包后重试
yum install dosfstools

# /boot 分区已经预留了 efi 目录以供 UEFI 启动分区挂载
# 但我们现在使用的是 BIOS 启动，这个目录应该为空且没有挂载
mount /dev/sda4 /boot/efi
# 为了后续使用，这一步完成后应该手动更新到 fstab 中
```

## 重新安装BootLoader

由于我们转换了分区方式，原有的 BootLoader 也需要重新安装，现代 Liunx 系统基本都使用 grub2 作为 BootLoader 。

grub2 在安装时有两个重要镜像 boot.img 和 core.img ，它们协作完成系统的引导工作，其中 boot.img 会安装在 MBR 分区硬盘的第一个扇区中，它主要负责把程序执行权转交给 core.img 。

而 core.img 包含了基本功能和必要的模块，承担了引导系统内核的核心工作，一般会安装 MBR 分区硬盘的第一个扇区和第一个分区之间的空隙，这个位置通常称为 MBR gap ，这部分信息可以在 grub2 的[官方文档](https://www.gnu.org/software/grub/manual/grub/grub.html#BIOS-installation)找到。

由于我们将 MBR 转换为 GPT 格式，原有的 MBR gap 的位置会被 GPT 头所占据，按照 grub2 官方文档对 GPT 分区格式的使用指引，我们需要新的分区来替代 MBR gap ，这就是之前在分区时额外划分 BIOS Boot Partition 的原因。

``` bash
# 1.以 UEFI 引导为目标重装 grub2 ，需要提前挂载 ESP 分区
grub2-install --target=x86_64-efi --efi-directory=/boot/efi /dev/sda
# 此时应该可以在 ESP 分区， /boot/efi 子目录下找到 grubx64.efi 文件
# 如果执行有类似模块信息丢失的报错，安装新的软件包后重试
yum install grub2-efi-x64-modules

# 2.生成 grub.cfg
grub2-mkconfig -o /boot/grub2/grub.cfg
# 如果对比观察新生成的 grub.cfg 和原有的 grub.cfg ，
# 可以发现主要区别在于处理 MBR 的模块已经转换为处理 GPT 的模块。

# 3.值得注意的是，我们目前的系统并不处在 UEFI 启动模式，系统下没有 UEFI 变量
# 所以 grub2-mkconfig 还不能完全生成正确结果，需要手动修改一次
sed -i 's/linux16/linuxefi/g' /boot/grub2/grub.cfg
sed -i 's/initrd16/initrdefi/g' /boot/grub2/grub.cfg
# 如果没有对 grub.cfg 进行修改，系统仍然可以从 BIOS 模式启动，但无法从 UEFI 模式启动
```

到这里系统下的工作基本完成，下一步需要重启机器进入 BIOS setup 来修改启动模式。

## 修改启动模式和UEFI启动项

在机器重启自检过程中，根据提示按下按键就能进入 BIOS setup ，这个按键一般会是 Del 或者 F2 。

进入到 BIOS setup ，常规主板会在 Boot Mode 选项设置启动模式，在服务器平台这个设置可能在 CSM 选项中，我们只需要将 Legacy BIOS 修改为 UEFI ，然后保存退出即可。

第一次 UEFI 启动可能需要手动从 Boot Manager 引导，通过 Boot from File ，进入 ESP 选择对应的 efi 引导文件 grubx64.efi ，选择 efi 文件之后的界面会是熟悉的 grub2 内核选择界面，我们就可以正常进入系统了。

为了后续的使用方便，进入系统后我们还要修改 UEFI 启动项。

``` bash
# 安装 efibootmgr
yum install efibootmgr

# 查看系统启动项
efibootmgr [-v]
# -v 可以看到更详细的信息

# 禁用一些无用的启动项
efibootmgr -B -b [number]
# 启用需要的启动项目
efibootmgr -a -b [number]
# 修改启动顺序
efibootmgr -o [number,number,number...]

# 将系统添加到独立的启动项
efibootmgr -c -d /dev/sda -p 4 -L Grub2 -l /EFI/centos/grubx64.efi
# -c | --create  create new variable bootnum and add to bootorder
# -d | --disk disk  (defaults to /dev/sda) containing loader
# -l | --loader name  (defaults to "\EFI\centos\grub.efi")
# -L | --label label  Boot manager display label (defaults to "Linux")
# -p | --part part  partition containing loader (defaults to 1 on partitioned devices)
# 大概的用法如上，比较值得注意的是 -L 选项，使用 \ 会有异常，无法指向正确的路径
# 添加选项后最好要使用 efibootmgr -o 检查引导文件路径是否正确
```

到这里我们的系统已经完成由 MBR+BIOS 向 GPT+UEFI 的迁移工作。

## 总结

转换启动模式直接应用于生产环境的场景可能比较少，即使我们修改了启动模式可能也没有带来特别大的收益，不过这个过程可以让我们比较详细地了解 UEFI 和传统的 BIOS 启动模式的不同之处，扩展一下系统引导的相关知识，并且未来 UEFI 的使用场景可能会越来越多，提前接触也是好事。

这次实验还引出一个比较重要的问题：对于系统盘的分区划分，我个人觉得应该考虑把根分区安排在中间， swap 分区放在最后，这样分区的好处是遇到根分区扩容的场景或者类似于这次分区格式转换的场景会比较方便操作。