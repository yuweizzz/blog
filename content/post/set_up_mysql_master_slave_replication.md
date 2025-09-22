---
date: 2025-07-22 14:45:48
title: 设置 mysql 主从复制
tags:
  - "mysql"
draft: false
---

基于已有运行的 mysql 主实例，搭建新的 mysql 从实例，设置主从复制。

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

目前线上运行着一个独立的 mysql 8.0 实例，需要新增一个只读的从实例来做分担主库的压力，同时作为主库的后备实例。这个实例已经开启了 binlog 日志，但是没有使用 GTID 模式。

新建的从实例的配置文件参考：

```bash
# mysql.conf
[mysqld]
datadir = /opt/mysql/
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
log-error = /var/log/mysql/error.log
bind-address = 0.0.0.0
default-time-zone = '+08:00'
log-bin = mysql-bin
# 这里的 server-id 应该使用和主实例不同的值
server-id = 2
# character-set-server = utf8mb4
# collation-server = utf8mb4_general_ci

binlog_format = ROW
slow_query_log = ON
slow_query_log_file = /var/log/mysql/mysqlslow.log
lower_case_table_names = 1
max_connections = 2000
wait_timeout = 86400
max_connect_errors = 100
max_allowed_packet = 1024m
# gtid_mode = ON
# enforce_gtid_consistency = ON
```

## 基于 binlog position 进行同步

首先对主实例进行备份：

```bash
mysqldump --single-transaction \
  --flush-logs \
  --master-data=2 \
  --databases db1 db2 \
  -u root -p > replica.sql
# --single-transaction 在单个事务中完成备份，避免锁表
# --master-data=2 以注释形式将 binlog 文件位置和对应点位记录到备份文件中
# --flush-logs 截断并重新写入新的 binlog

# 以下是一些没有用到的 mysqldump 选项，可以根据实际情况决定是否使用：
# --triggers 同时备份触发器
# --routines 同时备份存储过程和存储函数
# --events 同时备份事件
# --all-databases 备份所有的数据库

# 在数据库中新建用于同步的用户，添加对应的权限
create user 'replica'@'%' identified by 'replica_password';
grant replication client, replication slave on *.* to 'replica'@'%';
flush privileges;
```

在启动从实例之后，导入备份并设置主实例信息：

```bash
# 导入备份
source replica.sql;
# 重置当前 binlog ，在需要保留本地 binlog 的情况下，可以省略这一步
reset master;
# 设置主实例信息，其中的 binlog 文件位置和点位应该在 replica.sql 中获取
change master to
  master_host='10.0.0.1',
  master_user='replica',
  master_password='replica_password',
  master_log_file='mysql-bin.000100',
  master_log_pos=157;
# 启动同步
start slave;
# 将当前实例设置为只读
set global read_only=ON;

# 提升当前实例为独立节点
# stop slave;
# reset slave all;
# set global read_only=OFF;

# 从实例因为错误而停止同步时，可以根据实际情况决定是否跳过错误事件
# 以下操作可以跳过一次同步错误
# set gloabl sql_slave_skip_counter = 1;
# start slave;
```

因为目前的架构就是最简单的单主单从，直接使用 binlog 点位就可以实现同步。

## 基于 GTID 进行同步

在涉及到单主多从或者更复杂的情况下，一般会使用 GTID 模式，这样会需要所有的实例都开启 GTID 模式，通过配置文件中的 `gtid_mode` 和 `enforce_gtid_consistency` 进行控制。

首先对主实例进行备份：

```bash
mysqldump \
  --single-transaction \
  --flush-logs \
  --master-data=2 \
  --databases db1 db2 \
  --set-gtid-purged=on \
  -u root -p > replica.sql
# 对比前述的备份命令，额外设置 --set-gtid-purged=on 选项
# --set-gtid-purged=on 额外设置两个操作命令：
# 临时关闭 binlog ，所以本次 sql 导入时不会产生 binlog
# 设置 GTID_PURGED 变量，标记已经被删除了 binlog 的事务，这个值同时也是从实例应该开始同步的事务起点
```

在启动从实例之后，导入备份并设置主实例信息：

```bash
# 提前重置 binlog ，因为备份 sql 会涉及 GTID_PURGED 变量设置
reset master;
source replica.sql;
change master to
  master_host='10.0.0.1',
  master_user='replica',
  master_password='replica_password',
  master_auto_position=1;
# 不需要再额外指定 binlog 点位，通过 master_auto_position 自动定位
start slave;
set global read_only=ON;
```

这两种同步方式都使用到了 `reset master` ，需要注意这其实是非常危险的操作，由于我们是在全新实例上运行，所以使用这个命令影响不大，但是在已有数据的情况下，一定要慎重。
