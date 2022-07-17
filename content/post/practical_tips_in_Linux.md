---
date: 2021-07-23 15:00:00
title: 常用 Linux 命令和 Shell 技巧笔记
tags:
  - "Linux"
  - "Shell"
draft: false
---

这篇笔记用来记录 Linux 下常用命令和一些实用 Shell 技巧。

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

## Linux 的归档和压缩命令

归档压缩是经常需要进行的操作，在 Linux 系统中基本上通过 tar 就可以应对大部分场景，某些特殊的场景可能会涉及 zip ， 7z ， rar 这些不同的压缩文件格式，它们需要使用额外的工具去操作，这里只介绍 tar 命令。

我们通常会使用 tar 对一系列文件进行归档打包，生成一个新的文件，但是新生成的打包文件我们可以自行决定是否压缩。所以归档打包和压缩不是同一概念，只是它们通常是紧密结合在一起的。

``` bash
# 归档实例：

# 打包目录
$ tar -cvf output.tar /mydir
# -c 代表创建打包文件
# -v 代表显示详细过程
# -f 命名生成的打包文件
# 这一般是打包文件的最小操作命令，将 /mydir 打包为 output.tar

# 打包文件并进行压缩
$ tar -cvzpf output.tar.gz /mydir
$ tar -cvjpf output.tar.bz2 /mydir
$ tar -cvJpf output.tar.xz /mydir
# -j 代表使用 bzip2 压缩
# -z 代表使用 gzip 压缩
# -J 代表使用 xz 进行压缩
# -p 代表使用原来的文件权限还原文件
# 这是带了压缩命令的打包文件，以上三种压缩格式都是比较常用的
# 命名可以任意，一般保持通用的命名规则

# 解包文件
$ tar -xvf output.tar -C /mydir
# -x 代表解压文件操作
# -f 代表进行操作的文件
# -C 用来指定解压的路径
# 这一般是解压文件的通用命令，可以不指定压缩格式，tar 会自动适配

# 指定的压缩格式解压文件
$ tar -xvJf output.tar.xz -C /mydir
# 可以指定对应的压缩格式
```

## 硬盘分区和格式化

硬盘需要进行分区和格式化之后才能使用，现阶段主流的硬盘分区方式是 GPT ，小容量的硬盘和系统盘还经常会使用 msdos 。常用的分区工具有 parted ， fdisk ， gdisk 这几个，文件系统格式化一般使用 mkfs 命令。

``` bash
# fdisk 只能支持 msdos 分区
# gdisk 在 fdisk 的基础上扩展 gpt 的功能
# 推荐使用 parted ，它拥有更全面的功能

# 查看已有分区信息
$ parted /dev/sda print     

# 以下命令具有一定危险，需要注意数据安全
# 设置分区表格式
$ parted /dev/sda mklabel [ gpt | msdos ]

# 新增分区
$ parted /dev/sda mkpart [ primary | extended | logical ] [ ext4 | xfs ] start end
# gpt 不区分分区类型，这里会以 Name 代替， msdos 需要用到 primary ， extended 和 logical 分区
# 文件系统的标记一般可以不写
# 分区范围是必填项

# 删除某一分区， Number 代表分区编号
$ parted /dev/sda rm Number

# 格式化分区
$ mkfs.ext4 [-b size] [-L label] /dev/sda1
$ mkfs.xfs [-b size] [-L label] /dev/sda1
# 两种最常用的文件系统，参数都是相近的
# -b 用来设定最小区块大小，有1K，2K，4K
# -L 用来设置文件系统标签，用于挂载文件系统

# 速用脚本
$ parted /dev/sda -s mklabel gpt
$ parted /dev/sda -s mkpart primary 0% 100%  # 这里的 primary 为分区命名，可以自行定义
# -s 用来屏蔽 parted 的交互信息
$ mkfs.xfs /dev/sda1
```

## curl 的基本使用

curl 在 Linux 代替了浏览器的工作，经常用来调试接口。

``` bash
# 经典 curl 用例
$ curl -X POST -d '{"key":"value"}' -H 'Content-Type: application/json' http://....
# -X HTTP请求方法，通常有GET,POST,PUT,DELETE
# -d body，常用于POST方法的请求体
# -H header，定义HTTP请求头

# 这里列举其他常用的参数
# -b 用来设置 cookie ，常以 -b 'key1=value1;key2=value2' 的形式出现
# -k 用来请求 https 协议时跳过证书认证
# -i 用来额外输出请求的响应头，响应体正常输出
# -I 使用这个参数后只输出请求的响应头，响应体不再输出
# -x 用来设置代理服务器，用法为 -x/--proxy 127.0.0.1:8080
```

## curl 结合 bash 变量使用

curl 可以结合 bash 变量，实现更灵活的 url 请求。

``` bash
# 结合 bash 变量的简单实例
$ keyA=AAA
$ keyB=BBB
$ curl -X POST -H 'Content-Type: application/json' -d '{"keyA": "'"$keyA"'","keyA": "'"$keyB"'"}' http://....
```

可以看到，这个过程使用了大量的 `'` 和 `"` ，可以将其细分为五块：

* `'{"keyA": "'`
* `"$keyA"`
* `'","keyA": "'`
* `"$keyB"`
* `'"}'`

其中 `'` 包围的文本不会被 bash 转义，所以 `{}` 会以源文本的格式保留而不被转义。而变量总是以 `"$keyA"` 的形式出现，保证它被 bash 正确转义并获取变量内容。所以文本块就是 `'` 包围的部分和变量转义后的组合文本，注意 `"$keyA"` 和任意 `'` 包围的部分之间不能有空格，这样它们就会被认为是完整的 `-d` 选项的内容，发起请求时就不会报错了。

## 使用 sed 快速去除无用字符

sed 可以很简单快速去除空白行，首尾空白字符，注释行。

``` bash
# 删除空白行
$ sed '/^$/d'

# 删除行左边的空白字符
$ sed 's/^[ \t]*//g'

# 删除行右边的空白字符
$ sed 's/[ \t]*$//g'

# 删除注释行
$ sed '/^#/d'
# 可能要配合删除左侧空白字符使用

# 实现上述操作基于正则表达式：
# $ 代表行尾
# ^ 代表行首
# [] 整体代表一个字符，[] 里面是字符范围
# 在上面的操作中，[ \t] 用来匹配 ' ' 或 '\t' 任意一种
# * 表示一个或多个，可以结合确定的单个字符或者字符范围使用
```

## 快速生成随机强密码

``` bash
# 随机生成含有特殊字符，数字，大小写字母的强密码
# head -c 可以设置密码长度
$ cat /dev/urandom | tr -dc '[:graph:]' | head -c 24; echo

# 字符集由 tr -dc 决定，可以按需指定
# 只需数字
$ cat /dev/urandom | tr -dc '0-9' | head -c 24; echo
# 只需大小写字母
$ cat /dev/urandom | tr -dc 'a-zA-Z' | head -c 24; echo
```

## 快速转换进制

``` bash
# 将数值快速转换成目标进制表示

# 十进制转十六进制
$ printf "%x" 12345

# 十进制转八进制
$ printf "%o" 12345

# 十进制转科学计数
$ printf "%e" 12345
```

## 在 shell 中使用数组

bash 4 原生支持一维数组，在某些情况下可能会使用到这种数据结构。

``` bash
#!/bin/bash
# 使用之前需要先声明数组
declare -a array
declare -A Array
# -a 声明的数组是普通的一维数组， index 只能是0,1,2,...
# -A 声明的数组是关联数组，类似于 python 的字典， index 可以自由定义
# 推荐使用 -A ，因为关联数组的兼容性比较好，它可以模拟普通数组，
# 而普通数组无法实现关联数组的特性

# 数组赋值
array=(1 2 3 4)
Array=([A]=1 [B]=2 [C]=3 [D]=4)

# 或者直接定义 index 和 value
Array["index"]=vaule

# 遍历 index
# 使用 * 和 @ 均可
for i in ${!array[*]}; do echo $i; done;
for i in ${!array[@]}; do echo $i; done;

# 遍历 value ，和 index 类似
for i in ${array[*]}; do echo $i; done;
for i in ${array[@]}; do echo $i; done;

# 获取数组的元素个数
echo ${#array[*]};
echo ${#array[@]};
# 如果指定了确切的 index ，将会获取对应 value 的长度
echo ${#array["index"]};
```

## bash 变量操作符

bash 变量支持操作符，可以基于原有值做一些快速变换。

```bash
$ var=file.name.sh
# 修改第一个匹配字符 . 为 _
$ echo ${var/./_}
file_name.sh

# 修改所有匹配字符 . 为 _
$ echo ${var//./_}
file_name_sh

# 将表达式从左到右进行匹配，删除匹配的部分，可以使用通配符
$ echo ${var#*.}
name.sh

# 从左到右进行匹配的贪婪模式
$ echo ${var##*.}
sh

# 将表达式从右到左进行匹配，删除匹配的部分，同样可以使用通配符
$ echo ${var%.*}
file.name

# 从右到左进行匹配的贪婪模式
$ echo ${var%%.*}
file
```

使用时需要注意 `#` 和 `%` 必须是从左或从右的完整匹配，或者是方向一致的通配符匹配才能正确删除字符。

关于这两个定义符的记忆，我建议直接看键盘， `#` 在 `$` 左边， `%` 在 `$` 右边，把它们指向 `$` 则暗合它们的开始匹配方向。

## awk 内置函数 split()

``` bash
#!/bin/bash
# awk 内置函数 split() 的用法：

# awk: split(string_to_split,array,IFS)
# 第一个参数是想要进行分割的字符串
# 第二个参数是分割完成后的变量
# 第三个参数用来指定分割符

# 实例
split("A;B;C;D",array,';')
# 执行完成后，array=['A','B','C','D']
# 值得注意的是 array 是 awk 的内部变量
```

## awk 导入外部数据

``` bash
#!/bin/bash
# awk 可以导入外部数据，通常会用来导入 shell 变量进行进一步处理
# 灵活利用导入可以实现多样的数据处理

# 导入字符串并生成关联数组：
echo 'begin' | awk -v foreign='1#A#2#B#3#C' \
'BEGIN{
    split(foreign, dict, "#");
    for(i=1;i<=length(dict);i+=2){
        result[dict[i]]=dict[i+1]
    }
}
{
    for (i in result){
        print result[i];
    }
}'

# -v 可以导入外部数据为 awk 内部变量
# 这个例子将字符串 '1#A#2#B#3#C' 转换为字典 ([1]=A [2]=B [3]=C)
```

## 简单的循环用例

在 Linux 中经常需要批量执行命令，需要一些限制避免占用大量资源，下面提供一些简单的循环用例。

``` bash
#!/bin/bash
# 限制同一时间内循环的进程数量

# 总运行次数，数据源存放在 file 中
number=$(cat file | wc -l) 
# 单次并发进程数量控制
size=20

begin=1
end=$size

until [ $begin -gt "$number" ]
do
    for i in $(sed -n "$begin","$end"p file)
    do
    {
        # 主循环体，数据取自于 file
    }&
    done 

    begin=$( expr $end + 1 )
    end=$( expr $end + $size )
    wait
    # 单次并发会在这里阻塞，直到所有进程结束
    # 所以这个用例需要在循环体中加入超时控制
done


# 循环读取文件内容
# 文件内容为多行数据，每行数据格式为 X Y Z
while read A B C
do
    echo $A,$B,$C
done < file
```

## 使用 rpm 进行软件包操作

rpm 是红帽软件包工具，是红帽系发行版使用的基本软件管理工具。除了安装卸载软件，它可以提供一些额外的功能。

``` bash
# 直接安装已有的 rpm 包
$ rpm -ivh package.rpm  
# -i 用来表示安装
# -v 用来显示详细信息
# -h 用来显示安装进度

# 卸载已有软件
$ rpm -evh package
# -e 用来表示卸载

# 升级已有软件
$ rpm -Uvh package.rpm
# -U 用来表示升级软件，这个选项会保留原有配置文件

# 查询软件是否安装
$ rpm -qa | grep package
# 这是我个人常用的查询方法，用来查询所有已经安装的软件然后再抓取需要的信息

# 查看软件安装的路径和文件位置
$ rpm -ql package.rpm

# 查询目标文件是由哪个软件所安装
$ rpm -qf filename
```

## 使用 yum 下载 rpm 包

有时候会有下载 rpm 包的需求，可以通过 yum 实现。

``` bash
$ yum install package --downloadonly --downloaddir=/your/dir
# 可以只下载 rpm 包而不进行安装
# 不指定下载目录，则下载后文件保存在 /var/cache/yum/ 下的子目录中
# 如果 --downloadonly 运行失败，可能需要自行安装这个插件
```

## 配置 yum 源

yum 是 Fedora 和 RedHat 以及 SUSE 中使用的软件包管理器，它的使用比直接使用 rpm 更加方便简单。

``` bash
# yum 的全局配置
$ cat /etc/yum.conf 
[main]
cachedir=/var/cache/yum/$basearch/$releasever  # 指定缓存软件包和依赖信息的目录
keepcache=0  # 指定缓存是否会在使用后保留
debuglevel=2  # 指定日志信息的详细级别
logfile=/var/log/yum.log  # 指定生成日志的位置
exactarch=1  # 指定是否使用单一架构的软件包
obsoletes=1  # 指定是否允许更新比较陈旧的软件包
gpgcheck=1  # 指定进行安装包签名检查
plugins=1  # 指定是否可以使用插件
installonly_limit=5  # 指定允许保存的内核软件包数量
bugtracker_url=http://bugs.centos.org/set_project.php?project_id=23&ref=http://bugs.centos.org/bug_report_page.php?category=yum
distroverpkg=centos-release  #指定用来获取系统的发行版信息的软件
```

全局配置基本上不用做修改，更多情况下只需要自行添加需要的镜像源。

``` bash
# 参考具体的镜像源配置
$ cd /etc/yum.repos.d
$ cat CentOS-Base.repo 
[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

#rerpmleased updates 
[updates]
name=CentOS-$releasever - Updates
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
```

在 /etc/yum.repos.d 目录中以 repo 结尾的文件会被识别为镜像源的配置文件，一个镜像源配置文件内可以配置多个 Repository ，每个 Repository 的唯一标识用 [xxxx] 表示，它们的具体内容如下：

* name 是对 Repository 的描述。
* enable 规定对应 Repository 是否启用，可以使用这个选项屏蔽 Repository 。
* mirrorlist 和 baseurl 规定了 Repository 的地址。

    * baseurl 是指向 Repository 的 repodata 目录的地址，里面存放了软件包和他们的依赖关系。它可以指向本地和云端，本地一般以 file:// 来指定，云端可以使用 http，ftp 等工具。
    * mirrorlist 是 baseurl 的一种集合形式，可以说 mirrorlist 指向的是一系列的 baseurl ，配合 fastestmirror 插件能找到响应速度最快的 Repository 。      

* gpgcheck 和 gpgkey 是对软件包的签名检查的相关选项。

    * gpgcheck 规定是否进行签名检查。
    * gpgkey 指定了签名检查的合法签名数据源。

现有国内网络环境下，原始镜像速度很慢，可以使用公开的国内镜像源，它们大部分提供了相关的配置方法，如果是自建的镜像源，那么至少需要自行配置 name 和 baseurl 才能正常使用。

## 使用 cpio 打开 rpm 文件

rpm 包可以视为一个特殊的归档文件，如果需要提取这个档案中的内容，一般通过 rpm2cpio 和 cpio 去实现。

``` bash
# 常用的实例命令
$ rpm2cpio package.rpm | cpio -divm

# 以下是 cpio 比较重要的参数
# -i/--extract 展开文件
# -o/--create 生成文件
# -v/--verbose 显示详细过程
# -d/--make-directories 在需要时建立目录
# -m/--preserve-modification-time 保留更改时间。
```

## 更新文件到 initramfs 镜像中

在 Linux 系统中，我们一般可以在 `/boot` 目录中找到 initramfs 镜像文件，它们是引导系统的必要组件。如果需要使用到的系统命令没有编译到镜像中，可以使用 dracut 命令进行镜像更新。

``` bash
# 使用 file 命令可以看到 CentOS 7.4 的 initramfs 镜像是由 gunzip 压缩的文件
# initramfs-3.10.0-693.el7.x86_64.img: gzip compressed data, from Unix, 
# last modified: Sat Mar 16 15:51:25 2019, max compression

# 以常规思路去处理这个文件
# 首先需要更改命名后缀，否则 gunzip 无法识别
$ cp initramfs-3.10.0-693.el7.x86_64.img /tmp/initramfs-3.10.0-693.el7.x86_64.img.gz
# 使用 gunzip 解压
$ gunzip -d initramfs-3.10.0-693.el7.x86_64.img.gz
# 使用 file 命令可以看到解压后是 cpio 归档文件
# initramfs-3.10.0-693.el7.x86_64.img: ASCII cpio archive (SVR4 with no CRC)
# 进一步解包归档文件
$ cpio -divm < initramfs-3.10.0-693.el7.x86_64.img
# 到这一步可以完全提取出镜像内的文件

# 使用专用命令去查看镜像信息，比较推荐使用这种用法
$ lsinitrd /tmp/initramfs-3.10.0-693.el7.x86_64.img
# 可以看到基础的镜像信息和镜像内的具体文件信息

# 更新系统命令文件到 initramfs 中
$ dracut -v -I '/usr/sbin/xfsdump /usr/sbin/xfsrestore' -f [initramfs.img]
# -v 显示详细过程
# -I 需要添加到镜像中的文件列表，不同文件以空格隔开
# -f 强制覆写原有镜像，这个选项一般放在最后或者镜像文件名之前，否则可能会报错
# 这里可以省略镜像名称，会默认使用 /boot/initramfs-kernel_version.img
# 有需要可以在命令最后加上镜像文件名

# 这里添加了 xfs 文件系统的备份和还原命令，它们是需要额外安装的
# 执行 dracut 完成后可以使用 lsinitrd 检查是否成功
```
