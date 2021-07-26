---
date: 2021-07-03 15:00:00
title: 常用 Linux 命令和 Shell 技巧记录
tags:
  - "Linux"
  - "Shell"
draft: false
---

这篇文档用以记录一些 Linux 下常用命令和一些实用的 Shell 技巧。

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

## Linux的归档和压缩命令

归档压缩是经常需要进行的操作，在 Linux 系统中基本上通过 tar 就可以应对大部分场景，某些特殊的场景可能会涉及 zip ， 7z ， rar 这些不同的压缩文件格式，它们需要使用额外的工具去操作，这里只介绍 tar 命令。

我们通常会使用 tar 对一系列文件进行归档打包，生成一个新的文件，但是新生成的打包文件我们可以自行决定是否压缩。所以归档打包和压缩不是同一概念，只是它们通常是紧密结合在一起的。

``` bash
# 归档实例：

# 打包目录
tar -cvf output.tar /mydir
# -c 代表创建打包文件
# -v 代表显示详细过程
# -f 命名生成的打包文件
# 这一般是打包文件的最小操作命令，将 /mydir 打包为 output.tar

# 打包文件并进行压缩
tar -cvzpf output.tar.gz /mydir
tar -cvjpf output.tar.bz2 /mydir
tar -cvJpf output.tar.xz /mydir
# -j 代表使用 bzip2 压缩
# -z 代表使用 gzip 压缩
# -J 代表使用 xz 进行压缩
# -p 代表使用原来的文件权限还原文件
# 这是带了压缩命令的打包文件，以上三种压缩格式都是比较常用的
# 命名可以任意，一般保持通用的命名规则

# 解包文件
tar -xvf output.tar -C /mydir
# -x 代表解压文件操作
# -f 代表进行操作的文件
# -C 用来指定解压的路径
# 这一般是解压文件的通用命令，可以不指定压缩格式，tar 会自动适配

# 指定的压缩格式解压文件
tar -xvJf output.tar.xz -C /mydir
# 可以指定对应的压缩格式
```

## 硬盘分区和格式化

硬盘需要进行分区和格式化之后才能使用，现阶段主流的硬盘分区方式是 GPT ，小容量的硬盘和系统盘还经常会使用 msdos 。常用的分区工具有 parted ， fdisk ， gdisk 这几个，文件系统格式化一般使用 mkfs 命令。

``` bash
# fdisk 只能支持 msdos 分区
# gdisk 在 fdisk 的基础上扩展 gpt 的功能
# 推荐使用 parted ，它拥有更全面的功能

# 查看已有分区信息
parted /dev/sda print     

# 以下命令具有一定危险，需要注意数据安全
# 设置分区表格式
parted /dev/sda mklabel [ gpt | msdos ]

# 新增分区
parted /dev/sda mkpart [ primary | extended | logical ] [ ext4 | xfs ] start end
# gpt 一般只用到 primary 分区，msdos 需要额外用到 extended 和 logical 分区
# 文件系统的标记一般可以不写
# 分区范围是必填项

# 删除某一分区， Number 代表分区编号
parted /dev/sda rm Number

# 格式化分区
mkfs.ext4 [-b size] [-L label] /dev/sda1
mkfs.xfs [-b size] [-L label] /dev/sda1
# 两种最常用的文件系统，参数都是相近的
# -b 用来设定最小区块大小，有1K，2K，4K
# -L 用来设置文件系统标签，用于挂载文件系统

# 速用脚本
parted /dev/sda -s mklabel gpt
parted /dev/sda -s mkpart primary 0 100%
# -s 用来屏蔽 parted 的交互信息
mkfs.xfs /dev/sda1
```

## curl的基本使用

curl 在 Linux 代替了浏览器的工作，经常用来调试接口。

``` bash
# 经典curl用例
curl -X POST -d '{"key":"value"}' -H 'Content-Type: application/json' http://....
# -X HTTP请求方法，通常有GET,POST,PUT,DELETE
# -d body，常用于POST方法的请求体
# -H header，定义HTTP请求头

# 这里列举其他常用的参数
# -b 用来设置cookie，常以 -b 'key1=value1;key2=value2' 的形式出现
# -k 用来跳过SSL验证，用于请求https协议
# -i 用来额外输出请求的响应头，响应体正常输出
# -I 使用这个参数后只输出请求的响应头，响应体不再输出
```

## 使用sed快速去除无用字符

sed 可以很简单快速去除空白行，首尾空白字符，注释行。

``` bash
# 删除空白行
sed '/^$/d'

# 删除行左边的空白字符
sed 's/^[ \t]*//g'

# 删除行右边的空白字符
sed 's/[ \t]*$//g'

# 删除注释行
sed '/^#/d'
# 可能要配合删除左侧空白字符使用

# 实现上述操作基于正则表达式：
# $ 代表行尾
# ^ 代表行首
# [] 整体代表一个字符，[] 里面是字符范围
# 在上面的操作中，[ \t] 用来匹配 ' ' 或 '\t' 任意一种
# * 表示一个或多个，可以结合确定的单个字符或者字符范围使用
```

## 在shell中使用数组

bash 4 原生支持一维数组，在某些情况下可能会使用到这种数据结构。

``` bash
# 使用之前需要先声明数组
declare -a array
declare -A Array
# -a 声明的数组是普通的一维数组，index只能是0,1,2,...
# -A 声明的数组是关联数组，类似于python的字典，index可以自由定义
# 推荐使用 -A ，因为关联数组的兼容性比较好，它可以模拟普通数组，
# 而普通数组无法实现关联数组的特性

# 数组赋值
array=(1 2 3 4)
Array=([A]=1 [B]=2 [C]=3 [D]=4)

# 或者直接定义index和value
Array["index"]=vaule

# 遍历index
# 使用 * 和 @ 均可
for i in ${!array[*]}; do echo $i; done;
for i in ${!array[@]}; do echo $i; done;

# 遍历value，和index类似
for i in ${array[*]}; do echo $i; done;
for i in ${array[@]}; do echo $i; done;

# 获取数组的元素个数
echo ${#array[*]};
echo ${#array[@]};
# 如果指定了确切的index，将会获取对应value的长度
echo ${#array["index"]};
```

## awk内置函数split()

``` bash
# awk内置函数split()的用法：
awk: split(string_to_split,array,IFS)
# 第一个参数是想要进行分割的字符串
# 第二个参数是分割完成后的变量
# 第三个参数用来指定分割符

# 实例
split("A;B;C;D",array,';')
# 执行完成后，array=['A','B','C','D']
# 值得注意的是array是awk的内部变量
```

## awk导入外部数据

``` bash
# awk可以导入外部数据，通常会用来导入shell变量进行进一步处理
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

# -v 可以导入外部数据为awk内部变量
# 这个例子将字符串'1#A#2#B#3#C'转换为字典([1]=A [2]=B [3]=C)
```

## 简单的循环用例

在 Linux 中经常需要批量执行命令，需要一些限制避免占用大量资源，下面提供一些简单的循环用例。

``` bash
# 限制同一时间内循环的进程数量

# 总运行次数，数据源存放在file中
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
        # 主循环体，数据取自于file
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

## 使用yum下载rpm包

有时候会有下载 rpm 包的需求，可以通过 yum 实现。

``` bash
yum install package --downloadonly --downloaddir=/your/dir
# 可以只下载 rpm 包而不进行安装
# 不指定下载目录，则下载后文件保存在 /var/cache/yum/ 下的子目录中
# 如果 --downloadonly 运行失败，可能需要自行安装这个插件
```

## 使用cpio打开rpm文件

rpm 包可以视为一个特殊的归档文件，如果需要提取这个档案中的内容，一般通过 rpm2cpio 和 cpio 去实现。

``` bash
# 常用的实例命令
rpm2cpio package.rpm | cpio -divm

# 以下是 cpio 比较重要的参数
# -i/--extract 展开文件
# -o/--create 生成文件
# -v/--verbose 显示详细过程
# -d/--make-directories 在需要时建立目录
# -m/--preserve-modification-time 保留更改时间。
```