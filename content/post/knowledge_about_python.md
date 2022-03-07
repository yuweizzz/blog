---
date: 2022-01-04 22:22:45
title: Python 知识笔记
tags:
  - "Python"
draft: false
---

这篇笔记用来记录一些 python 编程的相关知识。

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

## 创建python虚拟环境

在 python 项目中经常会因为依赖的版本不同而存在不兼容的问题，所以我们应该使用虚拟环境将不同项目的依赖隔离，否则混用的依赖可能会产生各种各样的运行问题。

在 python2 中经常使用 virtualenv 作为虚拟环境管理库，在 python3 中可以直接使用标准库的自带虚拟环境管理库 venv 。

``` bash
# python2 的 virtualenv 需要独立安装
$ pip install virtualenv

# 使用 python2 的 virtualenv 创建虚拟环境
$ virtualenv project

# 激活后可以获得独立的虚拟环境
$ source project/bin/activate
(project) $ python --version
Python 2.7.5
(project) $ pip list
Package    Version
---------- -------
pip        20.3.3
setuptools 44.1.1
wheel      0.36.2

# 退出虚拟环境
$ deactivate

# python3 的 venv 不需要额外安装
# 实际使用和 virtualenv 没有特别大的区别
$ python3 -m venv project
$ source project/bin/activate
(project) $ python --version
Python 3.6.8
(project) $ pip list
pip (9.0.3)
setuptools (39.2.0)
$ deactivate
```

## 依赖管理

创建虚拟环境后就可以直接安装需要的依赖，我的建议是除了虚拟环境管理库，其他依赖尽量不要在全局环境安装。

``` bash
# 更换镜像源
$ pip config set global.index-url https://mirrors.aliyun.com/pypi/simple/

# 安装全局依赖
$ pip install virtualenv

# 在虚拟环境中安装需要的包
(project) $ pip install <package>

# 将项目中的依赖导出
(project) $ pip freeze > requirements.txt

# 导入已定义的依赖列表
(project) $ pip install -r requirements.txt
```

## 字符串格式化

在 python 中对字符串格式化的操作比较频繁，这里做个备忘，记录一下相关的方法。

``` python
# format 在 python2 和 python3 中均可用
name = "python"
greet = "hi, this is {}".format(name)

# f-string 表达式在 python 3.6 以上版本可用
name = "python"
greet = f"hi, this is {name}"

# 可以对原有数据做格式化，主要通过 : 来定义输出格式
value = 16
# 十进制数字转为十六进制字符串
value_format = "{:x}".format(value)
value_format = f"{value:x}"
```

这里介绍的方法返回值都是字符串，比较适合用在格式化输出或者日志记录中。

## 字典的常用方法

字典在各类编程语言中非常常见，这里记录一下 python 中字典的常用方法。

``` python
Count = {"A": 10, "B": 20}

# 删除一个原有的键值对
Count.pop("A")

# 增加一个新的键值对
Count.update({"C": 30})

# 获取指定键的值，如果没有这个键则返回预设值
Count.get("A", 0)

# 遍历所有键值对
for each in Count.items():  # items 返回值是元组
    print(each)

for key, value in Count.items():  # 直接在循环中拆分元组
    print(key, value)
```

## 集合

在 python 中集合使用的频率可能会比较低，但它在某些场合非常好用。

集合最大的特点在于集合内的所有元素都是唯一的。

``` python
array = [1, 1, 2, 2, 3, 3]
print(set(array))
# print 将会输出 {1, 2, 3}
```

实际中，集合应该是通过 hash 来实现这个特性的。

由于列表是可变的对象，如果你想将任意嵌套列表转化为集合就会得到 unhashable type 的报错，因为它无法计算出确定的 hash 值。相反地，常规的数字，字符串是 hashable 的，并且 python 的元组也是 hashable 的。

我们可以利用集合元素的唯一性，来实现两部分数据的差异对比：

* 首先提取两部分数据的关键值，它应该像数据库记录的主键，作为这部分数据的最大特征值。
* 将它们分别转化为集合，并取得补集，或称为对称差集。
* 对这个对称差集做遍历，将它们分类归属。

``` python
# 假设有以下两个数据集
Old = {"A": 10, "B": 20, "C": 30}
New = {"B": 100, "C": 30, "D": 40}

# 拿到数据集的所有重要特征值组合为元组
Old_tuples = [("A", 10), ("B", 20), ("C", 30)]
New_tuples = [("B", 100), ("C", 30), ("D", 40)]

# 集合化后取补集
Old_set = set(Old_tuples)
New_set = set(New_tuples)
Sysmmetric_Difference = Old_set ^ New_set
# Sysmmetric_Difference: {('D', 40), ('A', 10), ('B', 100), ('B', 20)}

# 遍历数据
for i in Sysmmetric_Difference:
    if i in Old_tuples:
        print("Old Data: ", i)
    elif i in New_tuples:
        print("New Data: ", i)

# 实际输出：
# New Data:  ('D', 40)
# Old Data:  ('B', 20)
# Old Data:  ('A', 10)
# New Data:  ('B', 100)

# 这里还不能完全确定部分数据，可以多做一次特征筛选
Indexs = [ key for key, _ in Sysmmetric_Difference ]
# Counter 可以统计列表的元素出现次数
from collections import Counter
Both = [ key for key, value in dict(Counter(Indexs)).items() if value > 1 ]
for key, value in Sysmmetric_Difference:
    if key in Both and (key, value) in Old_set:
        print("Before Changed Data: ", (key, value))
    elif key in Both and (key, value) in New_set:
        print("After Changed Data: ", (key, value))
    elif (key, value) in Old_set and (key, value) not in New_set:
        print("Old Data: ", (key, value))
    elif (key, value) in New_set and (key, value) not in Old_set:
        print("New Data: ", (key, value))

# 实际输出：
# New Data:  ('D', 40)
# Before Changed Data:  ('B', 20)
# Old Data:  ('A', 10)
# After Changed Data:  ('B', 100)
```

以上只是一个理想化的计算过程，实际中应该要结合目标场景来使用。

## 迭代器和生成器

这是对使用 map 函数引发异常的探究。

``` python
def add(x):
    return x + 1

List = [1, 2, 3, 4, 5]
Map = map(add, List)

print(list(Map))
print(list(Map))

# 在使用 python3 的情况下：
# 第一次输出为 [2, 3, 4, 5, 6]
# 第二次输出为 []
```

在 python 中，很多数据类型都是可迭代的，比如列表就是最经典的可迭代对象，可以使用 `isinstance(Object, Iterable)` 来判断对象是否是可迭代的， `Iterable` 需要从 `collections` 中导入。

实际中，一个对象如果是可迭代的，那么它需要实现 `__iter__` 方法或者 `__getitem__` 方法。

其中 `__getitem__` 方法常见于通过索引取值的数据类型，而 `__iter__` 是可迭代对象优先使用的方法，一般它会返回迭代器对象 iterator object 。

迭代器对象的作用是遍历可迭代对象。它自身需要实现 `__iter__` 和 `__next__` 两种方法。

其中 `__iter__` 还是用于返回迭代器对象，通常就是这个 iterator object 本身， `__next__` 则是迭代器对象用来获取下一元素的方法。

在 python2 中， map 函数返回是一个列表，但在 python3 中， map 函数返回是一个 map 对象。虽然列表和 map 对象都是可迭代对象，但是实际中 map 对象是迭代器对象，所以在上面的测试用例中， map 对象经历两次迭代，第二次迭代时 `__next__` 已经无法继续获取下一个元素，取值为空。

除了可迭代对象和迭代器，还会使用到生成器 generator object ，实际中它就是迭代器对象的子类。

我们一般通过关键字 `yield` 定义生成器函数。

``` python
def gen():
    yield "step1"
    yield "step2"
    yield "step3"

print([each for each in gen()])
# print 将会输出 ['step1', 'step2', 'step3']
```

实际中常见的用法是在循环中使用关键字 `yield` ，变相实现写入多个 `yield` 的效果，这样生成器就可以多次产生返回值。

在生成器函数中，当遍历语句执行到 `yield` 关键字时会产生返回值并结束本次步进，下一轮遍历则从 `yield` 关键字之后的语句开始执行，如果直接调用生成器函数会返回对应的生成器对象，这里的工作方式实际上就是 `__iter__` 和 `__next__` 的组合。

## 使用supervisor值守python进程

使用 python 作为后台服务的时候，如果希望它可以起到类似于 systemd 具体服务的效果，可以使用 supervisor 。

supervisor 是 python 编写的进程管理服务，使用 client/server 架构，可以方便监测进程状态，并提供值守功能，其中 server 是 supervisord ，它负责监听挂载的进程状态和客户端的请求信号， client 是 supervisorctl ，用于向服务端发送控制信号，使用 supervisor 的好处在于程序的启动变得简洁，并且自带值守功能，不用担心进程意外死亡导致的服务停摆。

``` bash
# 推荐使用 yum 安装 supervisor 
$ yum install supervisor

# supervisord 部分没有具体需要可以保持不变
$ cat /etc/supervisord.conf 
# 这里是具体进程的配置目录
...
[include]
files = supervisord.d/*.ini
...
# 可以看到具体的配置范例
...
;[program:theprogramname]
;command=/bin/cat                        ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s           ; process_name expr (default %(program_name)s)
;numprocs=1                              ; number of processes copies to start (def 1)
;directory=/tmp                          ; directory to cwd to before exec (def no cwd)
;umask=022                               ; umask for process (default None)
;priority=999                            ; the relative start priority (default 999)
;autostart=true                          ; start at supervisord start (default: true)
;autorestart=true                        ; retstart at unexpected quit (default: true)
;startsecs=10                            ; number of secs prog must stay running (def. 1)
;startretries=3                          ; max of serial start failures (default 3)
;exitcodes=0,2                           ; 'expected' exit codes for process (default 0,2)
;stopsignal=QUIT                         ; signal used to kill process (default TERM)
;stopwaitsecs=10                         ; max num secs to wait b4 SIGKILL (default 10)
;user=chrism                             ; setuid to this UNIX account to run the program
;redirect_stderr=true                    ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path                  ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB             ; max logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10               ; max of stdout logfile backups (default 10)
;stdout_capture_maxbytes=1MB             ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false             ; emit events on stdout writes (default false)
;stderr_logfile=/a/path                  ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB             ; max logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10               ; max of stderr logfile backups (default 10)
;stderr_capture_maxbytes=1MB             ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false             ; emit events on stderr writes (default false)
;environment=A=1,B=2                     ; process environment additions (def no adds)
;serverurl=AUTO                          ; override serverurl computation (childutils)
...
```

以下是其中非常重要的几个配置项：

* autorestart ：决定进程状态异常后是否会自动重启，这是最重要的值守功能，一般需要开启。
* autostart ：决定进程是否跟随 supervisord 启动，一般需要开启，这样启动 supervisord 服务就相当于直接启动我们自定义的程序。
* environment ：进程的环境变量配置。
* command ：启动时的具体命令，如果是 python 进程，可以使用项目虚拟环境的绝对路径来指明对应的解释器。
* directory ：程序的具体工作目录。

在安装 supervisor 后，只要写入对应的 ini 文件到 `/etc/supervisord.d/` 目录中，就可以使用 `supervisorctl` 启动 ini 文件定义的 program ，或者在定义文件后直接重启整个 supervisord 服务 ，同样可以启动我们需要的 program 。

``` bash
# 定义新的 program 后需要重载
$ supervisorctl reload
# 或者直接重启 supervisord
$ systemctl reload supervisord

# 使用 supervisorctl 操作具体 program 支持 start/stop/restart 三个动作
# 操作所有 program
$ supervisorctl start all
# 操作某个 program 需要使用指明具体名称，这个名称对应 ini 文件中定义的 program:theprogramname
$ supervisorctl start theprogramname
```

