---
date: 2021-11-07 21:49:00
title: 硬盘知识拓展：总线适配器和磁盘阵列
tags:
  - "HDD"
  - "RAID"
  - "HBA"
draft: false
---

这篇笔记用来记录一些服务器磁盘管理的相关知识。

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

## 服务器磁盘管理

如今的家用计算机可能很多已经使用了 NVMe 协议的固态硬盘，但是一般情况下你仍然可以看到用于接入 SATA 硬盘的接口，它们是通过直接接入到主板南桥中的 SATA 控制器来扩展额外硬盘的。

服务器上的情况会略微不同，因为一般的 SATA 控制器只能接入有限数量的硬盘，通常是两个到四个这一量级。服务器动辄十几个以上的硬盘数量，只靠主板集成的 SATA 控制器是远远不够的，这时就需要用到 HBA 卡。

关于磁盘的总线控制芯片，市场上很多使用的是 Broadcom 的芯片和对应的扩展卡，如果是 Broadcom 生产的扩展卡，一般可以通过 `lspci` 查看到 `LSI` ， `AVAGO` 相关的字样，这是由于这家公司经过了多次并购的原因。

我们所说的 HBA 卡就是 Host Bus Adapter ，外观和使用上与网卡这一类的 PCI 扩展设备相似，接入这种卡的硬盘设备就不再经过南桥，而是通过它提供的 PCIe 通道和 CPU 进行数据交互，一般具有 12 Gb/s 和 24 Gb/s 的吞吐能力，比如使用 SAS3008 芯片的 LSI SAS 9300-8i 扩展卡就是 12 Gb/s ， HBA 卡使用的集成芯片一般是 SAS/SATA Storage I/O Controllers ，也就是只支持 SAS 和 SATA 接口的硬盘设备。但是近年来随着 NVMe 协议硬盘的使用越来越频繁，带有 Tri-Mode I/O Controller 的新一代 HBA 卡已经可以直接支持 PCIe/NVMe 协议的硬盘。

另一类带有 RAID 控制能力的扩展卡是 RAID Controller Card ，也就是常说的 RAID 卡，相较于 HBA 卡，它使用的芯片是 RAID-on-Chip ICs ，除了常规的 IO 控制之外还具有磁盘阵列的控制功能。可以认为是一种更智能的 HBA 卡，吞吐能力和 HBA 一样，具有 12 Gb/s 和 24 Gb/s 两种级别，也具有 Tri-Mode 类型的芯片。之前比较多见的 RAID 卡类型是使用 SAS3108 芯片的 MegaRAID SAS 9361-8i 。

除了原厂生成的扩展卡，很多服务器厂商使用 Broadcom 的芯片制作自己的扩展卡，比如 DELL 和 inspur 以及 Lenovo 等公司都有这样的操作，虽然最终制成的 RAID 卡名称各不相同，但一般情况下都可以使用 `megacli` 和 `storcli` 这两个工具通过命令行进行管理硬件 RAID 。

不过 SAS/SATA Storage I/O Controllers 这一类芯片可以通过特定的 firmware 来支持 RAID 功能，比如 SAS3008 IR 版本就可以支持 RAID0 和 RAID1 ，具有 IR 能力的较旧芯片 `LSI SAS-3 controllers` 系列可以使用 `sas3ircu` 工具通过命令行管理， `LSI SAS-2 controllers` 系列对应的是 `sas2ircu` 。

除了两种扩展卡外还有一种支持更大磁盘数量的扩展设备是 SAS expanders ，但这种扩展设备一般用于盘柜，它可以聚合大量的磁盘，并承担中转设备的责任，还需要配合服务器上的 HBA 卡来使用，因为 SAS expanders 不具有直接 IO 的能力。

一开始我对 SAS expanders 这种设备是有疑惑的，但有一种情况可以帮助理解这种设备的作用，前面所说的 LSI SAS 9300-8i 和 MegaRAID SAS 9361-8i 都具有相同的 8i 命名结尾，这是由于这两种卡上都有两个 SFF8643 的接口，每个 SFF8643 可以分出四个 SFF-8482 接口直连硬盘，所以 8i 的含义就是可以直连 8 个硬盘。我们可以看到更强大的 16i 和 24i 的扩展卡，不支持 Tri-Mode 一般由支持 4i 的 SFF8643 来组合扩展卡接口，支持 Tri-Mode 的一般由支持 8i 的 SFF8654 来组合扩展卡接口，这样可以支持更多数量的硬盘。

很多 2u 的服务器都采用 12 Bay 的硬盘背板 backplane ，有一些服务器可能还扩展到 16 Bay ，但是这样的机器通常可以使用一张 8i 的扩展卡管理所有磁盘，是由于服务器的硬盘背板 backplane 集成了 SAS expanders 芯片，由它去中转 8i 能力之外的硬盘，这种情况应该可以帮助理解 SAS expanders 设备的存在意义。

## 磁盘阵列等级

前面所说的 RAID 就是磁盘阵列技术，全称为 Redundant Array of Independent Disks ，它是将多个物理磁盘组成一个逻辑磁盘的技术，可以提高磁盘的逻辑容量，带来更强的性能和数据冗余能力。

根据磁盘的数量可以创建不同的 RAID 等级，分别具有不同的性能表现和冗余能力提升，比较常用的有下面几个等级：

- RAID0 ：至少需要两个或两个以上的物理磁盘，逻辑容量为所有磁盘容量的总和，读写性能会增强，但是没有数据冗余能力，一个磁盘损坏则整个逻辑磁盘都会丢失数据。在写入数据时， RAID 控制器会将数据条带化，分别存储在不同的子磁盘中，读取时也需要同时获取不同子磁盘中的存储条带才能最终得到完整文件。
- RAID1 ：至少需要两个或两个以上的物理磁盘，逻辑容量为单个磁盘的容量，读写性能会增强，数据冗余能力会增强，容错能力取决于磁盘数，只要最终仍有一个磁盘正常，那么数据就是安全的。在写入数据时，所有子磁盘都是镜像写入的，各子磁盘的存储内容一致，在读取数据时效率会更高。
- RAID5 ：至少需要三个或三个以上的物理磁盘，逻辑容量为磁盘容量总和减去一个磁盘容量（n-1），最大的容错能力为一个磁盘。在写入数据时， RAID 控制器会将数据条带化并计算奇偶校验码，分别存储在不同的子磁盘中，读取效率和 RAID0 近似，同时可以通过校验码容忍单个磁盘的故障。
- RAID10 ：复合型 RAID 等级，先组合出多个 RAID1 逻辑盘，再将所有逻辑盘组合为 RAID0 逻辑盘，弥补了 RAID0 的数据冗余能力，但是继承了 RAID1 容量利用率低的问题。

在 Linux 中有软 RAID 的概念，但软 RAID 是由操作系统模拟 RAID 行为，性能比不上硬件 RAID ，生产环境中一般只用 RAID 卡或者主板集成的 RAID 控制器来实现 RAID 功能。

## RAID 组策略

RAID 卡一般会配置缓存模块和相应的电源模块，一些老式的 RAID 卡甚至还会使用外置电池，外置电池损坏是常有的事情，会有缓存丢失的风险，但现代的 RAID 卡一般使用内置的电源模块，提高了耐用性。

新建的 RAID 组可以设置不同的策略来达到性能目标：

写策略 Write Policy ：

- Write Through (WT) ：直写， RAID 组的写入 IO 请求会直接写入物理磁盘。 RAID0 推荐使用这种策略，可以实现最好的写入带宽。
- Write Back (WB) ：回写， RAID 组的写入 IO 请求会写入到 RAID 卡的缓存。这种策略会有掉电丢失数据的风险，所以有些 RAID 卡在缓存电源模块异常时会拒绝使用这种策略。

读策略 Read Policy ：

- No Read Ahead (NORA) ：不预读，RAID 组的读取 IO 请求会以 block 级别去处理，对随机读取友好。
- Read Ahead (RA) ：预读， RAID 组的读取 IO 请求会读取 block 所在的整个条带并进行缓存，对顺序读取友好。
- Adaptive Read Ahead (ADRA) : 自适应预读，由 RAID 卡根据读取 IO 请求决定使用 RA 或者 NORA 。

IO 策略 IO Policy ：

- Cached IO ：启用缓存，这里的缓存是 RAID 卡的读写缓存，类似于操作系统的缓存。
- Direct IO ：不启用缓存。

磁盘缓存策略 Disk Cache Policy ：

- Unchange ：保持磁盘原有的策略设置。
- Cached ：启用缓存，这里的缓存是磁盘自身的读写缓存。
- Direct ：不启用缓存。

很多情况下是根据自身的业务特性来设置读写策略的，读策略更多考量顺序读取和随机读取的比例，写策略在固态硬盘做子磁盘时基本都会选择直写，使用机械硬盘的 RAID5 和 RAID1 会选择回写， RAID0 可以选择直写。

至于 IO 策略一般都是禁用 RAID 缓存和磁盘缓存，禁用 RAID 缓存是由于操作系统已经拥有缓存机制，禁用磁盘缓存是由于它不像 RAID 卡缓存一样拥有自己的掉电保护机制。

## RAID 卡命令行工具使用

这里介绍用于 HBA 卡的 `sas3ircu` 。

```bash
# 查看所有控制器
$ sas3ircu list
# 会输出所有可识别的扩展卡信息和对应 Index

# 根据 Index 查看某个控制器的磁盘信息和 RAID 信息
$ sas3ircu 0 display
# Enclosure 用来索引不同的 backplane
# Slot 用来索引 backplane 中的槽位

# IR 模式的 HBA 卡可以设置 RAID 组， IT 模式的无法使用 RAID 功能
# 假设 Enclosure Id 为 2 ，使用两块硬盘创建 RAID1 组
$ sas3ircu 0 create RAID1 max 2:0 2:1 noprompt
# 通过 volumeid 删除 RAID 组，假设 volumeid 为 100 ，实际可以使用 display 看到 volumeid
$ sas3ircu 0 deletevolume 100 noprompt

# 点亮指定硬盘的指示灯，在替换故障盘时非常有用，可能需要手动关闭
$ sas3ircu 0 locate 2:0 on
$ sas3ircu 0 locate 2:0 off

# 混合使用 RAID 组和直接使用物理盘可能会导致启动异常，可以根据需要指定启动设备
# 通过 volumeid 设置 RAID 组作为主启动项
$ sas3ircu 0 bootir 100
# 通过 volumeid 设置 RAID 组作为备选启动项
$ sas3ircu 0 altbootir 100

# 设置单盘作为主启动项
$ sas3ircu 0 bootencl 2:0
# 设置单盘作为备选启动项
$ sas3ircu 0 altbootencl 2:0
```

这里介绍用于 RAID 卡的 `megacli` 。

```bash
# 获取 RAID 卡的版本信息和型号
$ MegaCli64 -AdpALLInfo -a0

# 查看服务器 backplane 信息
$ MegaCli64 -EncInfo -a0
# 可以获取 Enclosure Device ID 和对应的可容纳磁盘数量 Number of Slots

# 查看 RAID 卡下属的磁盘信息和 RAID 组信息
$ MegaCli64 -PDList -a0
$ MegaCli64 -LDInfo -Lall -a0
$ MegaCli64 -CfgDsply -a0

# RAID 卡中的 PD 通常是以下的状态：
# Unconfigured Good ：新接入 RAID 卡的全新磁盘，可直接进行 RAID 配置
# Online ：已经被配置到 RAID 组中的正常磁盘
# Unconfigured Bad ：不能直接进行 RAID 配置的磁盘，不一定是故障的硬盘
# Offline ：已经被配置到 RAID 组中的离线磁盘，可能是手动下线或者磁盘故障导致
# Unconfigured Good (Foreign) ：带有未知 RAID 信息的磁盘
# JBOD ：不使用 RAID 功能的磁盘，使用和常规 HBA 卡一样的直通模式

# 由 Unconfigured Bad 或者 JBOD 转换到 Unconfigured Good
$ MegaCli64 -PDMakeGood -PhysDrv[2:0] -a0
# 类似于 HBA 卡，使用 Enclosure:Slot 来索引磁盘

# 使用 JBOD 模式，除了单盘直通外，还可以将多个磁盘合并为同一逻辑盘，不使用 RAID 条带化能力
$ MegaCli64 -PDMakeJBOD -PhysDrv[2:0] -a0
$ MegaCli64 -PDMakeJBOD -PhysDrv[2:0,2:1] -a0

# Foreign 意味着磁盘原先就带有 RAID 信息，可以选择导入或者删除
$ MegaCli64 -CfgForeign -Import -a0
$ MegaCli64 -CfgForeign -Clear -a0

# 添加 RAID 组
$ MegaCli64 -CfgLdAdd -r1 -PhysDrv[2:0,2:1] -Hsp[2:2] -a0
# -r 代表阵列等级，比如 -r0 代表 RAID0 ， -r5 代表 RAID5
# -Hsp 用来指定热备盘，它会在 RAID 组中某个磁盘故障时自动替换故障盘，实际中用得非常少

# 删除 RAID 组
$ MegaCli64 -CfgLdDel -L0 -a0
# -L 代表 RAID 组的 id

# 下线和上线 RAID 组中的某个磁盘
$ MegaCli64 -PDOffline -PhysDrv[2:0] -a0
$ MegaCli64 -PDOnline -PhysDrv[2:0] -a0

# 点亮指定硬盘的指示灯
$ MegaCli64 -PdLocate -start -PhysDrv[2:0] -a0
$ MegaCli64 -PdLocate -stop -PhysDrv[2:0] -a0

# 用于 RAID 组替换故障磁盘后，查看 RAID 组重建进度
$ MegaCli64 -PDRbld -ShowProg -PhysDrv[2:0] -a0
```
