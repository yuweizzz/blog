---
date: 2023-01-10 15:22:45
title: 硬件测试工具使用笔记
tags:
  - "fio"
  - "stream"
draft: false
---

这篇笔记用来记录硬件测试工具 fio 和 stream 的使用。

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

## 硬盘性能测试工具 fio

fio 是比较常用的硬盘性能测试工具，它可以模拟不同读写场景并得到对应的 IOPS 和硬盘带宽，延迟等性能参数。

### io engine

使用 fio 需要指定 io engine ，实际上就是指定读写存储设备需要使用到的系统调用库，测试硬盘性能一般会使用 sync 或者 libaio ，其中 sync 使用的是 Linux 中基本的 IO 系统调用库，而 libaio 是 Linux 原生异步 IO 库。

### io pattern

fio 可以指定具体的读写模式来模拟现实场景，比较常用的有下面几个：

- read 顺序读
- write 顺序写
- randread 随机读
- randwrite 随机写
- readwrite 顺序读写
- randrw 随机读写

一般来说，机械硬盘的强项是顺序读写，而随机读写是机械硬盘的弱项；固态硬盘的强项是随机读写，顺序读写虽然表现也不错但是并不占优势，因为固态硬盘的价格比机械硬盘更高。

像是常规的硬盘测试可以是裸盘测试或是基于文件系统测试，通常会结合顺序读写和随机读写来考量硬盘的综合性能。

可以将如下测试过程作为参考，这是基于裸盘进行测试的：

顺序写满 -> 256k 顺序写 -> 256k 顺序读 -> 4k 顺序写 -> 4k 顺序读 -> 随机写满 -> 4k 随机写 -> 4k 随机读

顺序读写一般会以比较大的块大小去操作，通常可以测试出比较高的带宽数据，但是随机读写一般会使用 4k 或者 16k 这些比较小且贴合操作系统的内存页大小去操作，这里更重要的数据应该是 IOPS ，同时也是衡量它们性能的重点。

### fio option

下面是一些重要的 fio 命令参数：

- `-name` 可以命名当前进行的测试任务。
- `-filename` 用来指定测试对象，如果是类似于 `/dev/sdb` 这样的设备则说明测试对象是裸盘，如果是类似于 `/home/file.img` 这样的文件路径则说明测试对象是基于文件系统的。如果是裸盘测试很有可能把原有的文件系统写坏，使用时要注意数据安全。
- `-ioengine` 用来指定进行测试的 IO 函数库，也就是前述的 sync 或者 libaio 。
- `-rw` 用来指定具体的读写模式，也就是前述的 io pattern 。
- `-direct` 决定使用 buffered IO 或者是 non-buffered IO ，其中 buffered IO 是大多数操作系统的默认 IO 方式，数据会先被内核从硬盘中读取到缓冲中，再由用户层将内核缓冲作为第一层来读取，使用 `-direct=1` 可以屏蔽内核缓冲的使用，使用 non-buffered IO 来直接读取硬盘，大部分情况下应该使用 non-buffered IO 进行测试。
- `-iodepth` 主要作用于异步 IO 库，也就是 libaio ，它可以决定并发性地发起多少个 IO 请求，这个选项在使用 buffered IO 和同步 IO 库时基本不起作用。
- `-bs` 用来指定读写的 IO 块大小。
- `-size` 用来指定读写内容的大小，在读写给定大小内容完成后就会结束任务，如果不指定 `-size` 还会根据 `-runtime` 决定任务的运行时间。
- `-numjobs` 和 `-thread` 是控制测试任务的并发方式，如果使用 `-thread` 则使用单进程多线程的方式来运行，如果不使用 `-thread` 则使用多进程的方式来运行，而 `-numjobs` 则用来控制线程数量，在使用并发的情况下，可以使用 `-group_reporting` 来综合所有执行结果以输出测试报告。

参考命令如下：

```bash
# 顺序读
$ fio -name=read -filename=/dev/sdb -rw=read -ioengine=libaio -direct=1 -iodepth=32 -bs=256k -size=8G \
  -numjobs=4 -thread -group_reporting

# 顺序写
$ fio -name=write -filename=/dev/sdb -rw=write -ioengine=libaio -direct=1 -iodepth=32 -bs=256k -size=8G \
  -numjobs=4 -thread -group_reporting

# 随机写
$ fio -name=randwrite -filename=/dev/sdb -rw=randwrite -ioengine=libaio -direct=1 -iodepth=32 -bs=4k -size=8G \
  -numjobs=4 -thread -group_reporting

# 随机读
$ fio -name=randread -filename=/dev/sdb -rw=randread -ioengine=libaio -direct=1 -iodepth=32 -bs=4k -size=8G \
  -numjobs=4 -thread -group_reporting
```

## 内存性能测试工具 stream

stream 是内存带宽性能的测试工具，主要通过内存运算操作来计算出机器的内存带宽性能。

它的原理是通过在内存中划定三个数组，通过函数对不同数组内数据进行计算来得出内存的带宽性能，根据 stream 的设计，我们可以自行指定这三个数组的大小，并且这个大小应该超过 CPU 的三级缓存大小，这样测试时才不会受到 CPU 缓存的影响，推荐是 4 倍的三级缓存大小。

实际使用可以参考下面的过程：

```bash
# 获取机器的 CPU 信息
$ lscpu | grep -E 'Socket|L3'
Socket(s):           1
L3 cache:            16384K
# Socket(s) 可以得到机器有多少个物理 CPU ，一般服务器是单个，两个或者四个 CPU
# L3 cache 可以得到单个物理 CPU 的三级缓存大小，整台机器的三级缓存大小应该是 Socket(s) * L3 cache Size

# 下载源代码
$ wget http://www.cs.virginia.edu/stream/FTP/Code/stream.c

# 编译需要确定 STREAM_ARRAY_SIZE 编译参数
$ gcc -O3 -fopenmp -DSTREAM_ARRAY_SIZE=8500000 -DNTIMES=10 stream.c -o stream
# -O3 指定编译优化级别为最高
# -fopenmp 启用多核支持，这样 stream 会默认以核心数量来启动线程
# -DNTIMES 用来改变 stream 的运行次数， stream 默认会多次执行计算并取出最好的运行结果，默认值为 10
# -DSTREAM_ARRAY_SIZE 就是我们需要指定的数组大小，默认值为 10000000
# 如果 STREAM_ARRAY_SIZE 默认值已经大于通过机器的三级缓存总和计算的结果，则可以直接保持这个默认值

# 源代码部分参考：
/* -------------------------------------------------- */
#ifndef STREAM_TYPE
#define STREAM_TYPE double
#endif

static STREAM_TYPE      a[STREAM_ARRAY_SIZE+OFFSET],
                        b[STREAM_ARRAY_SIZE+OFFSET],
                        c[STREAM_ARRAY_SIZE+OFFSET];
/* -------------------------------------------------- */
# 可以看到 STREAM_ARRAY_SIZE 默认使用 double 类型
# 假设 Socket(s)=1 和 L3 cache Size=16384K ，则 STREAM_ARRAY_SIZE 的计算公式为：
# 16384(KB) * 1024 * 1 * 4 / 8(B) = 8388608 -> 8500000
# 假设 Socket(s)=1 和 L3 cache Size=40M ，则 STREAM_ARRAY_SIZE 的计算公式为：
# 40(MB) * 1024 * 1024 * 1 * 4 / 8(B) = 20971520 -> 21000000

# 执行 stream
$ ./stream
# 以下是在一台云服务器上的运行结果：
-------------------------------------------------------------
STREAM version $Revision: 5.10 $
-------------------------------------------------------------
This system uses 8 bytes per array element.
-------------------------------------------------------------
Array size = 8500000 (elements), Offset = 0 (elements)
Memory per array = 64.8 MiB (= 0.1 GiB).
Total memory required = 194.5 MiB (= 0.2 GiB).
Each kernel will be executed 10 times.
 The *best* time for each kernel (excluding the first iteration)
 will be used to compute the reported bandwidth.
-------------------------------------------------------------
Number of Threads requested = 4
Number of Threads counted = 4
-------------------------------------------------------------
Your clock granularity/precision appears to be 1 microseconds.
Each test below will take on the order of 3318 microseconds.
   (= 3318 clock ticks)
Increase the size of the arrays if this shows that
you are not getting at least 20 clock ticks per test.
-------------------------------------------------------------
WARNING -- The above is only a rough guideline.
For best results, please be sure you know the
precision of your system timer.
-------------------------------------------------------------
Function    Best Rate MB/s  Avg time     Min time     Max time
Copy:           28577.0     0.005012     0.004759     0.005315
Scale:          28416.1     0.004947     0.004786     0.005157
Add:            29771.7     0.007201     0.006852     0.007542
Triad:          29348.9     0.007279     0.006951     0.007677
-------------------------------------------------------------
Solution Validates: avg error less than 1.000000e-13 on all three arrays
-------------------------------------------------------------
```
