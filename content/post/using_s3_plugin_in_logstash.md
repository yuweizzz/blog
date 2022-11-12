---
date: 2022-08-15 15:00:00
title: Logstash S3 插件使用
tags:
  - "Logstash"
draft: false
---

这篇笔记用来记录如何使用 Logstash S3 插件。

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

为了减轻 Elasticsearch 集群存储空间的负担，需要将较旧的日志额外使用对象存储保存，以防不时之需。

以下 Logstash 配置可以在写入 Elasticsearch 集群的同时，将日志保存一份到对象存储中：

``` bash
# 消费 Kafka 消息
input {
    kafka {
        bootstrap_servers => ["localhost:9092"]
        topics_pattern => "logs-*"
        group_id => "logs-consumer-group"
        # 按照对应的 Kafka topic 分区来设置最佳，一般使用 3 个分区
        consumer_threads => 3
        codec => json
        # 从分区的最开始进行消费
        auto_offset_reset => "earliest"
        partition_assignment_strategy => "round_robin"
        # 将 Kafka 来源信息写入到 metadata 中，后续可以使用 %{[@metadata][kafka]} 来取需要的字段
        decorate_events => true
    }
}

# 同时写入到 Elasticsearch 集群和对象存储
output {
    elasticsearch {
        index => "%{[@metadata][kafka][topic]}-%{+YYYY.MM.dd}"
        hosts => ["http://localhost:9200"]
        user => "elastic"
        password => "elastic_password"
    }

    s3 {
        # 替换为对应的密钥信息
        "access_key_id" => "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
        "secret_access_key" => "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
        # 以腾讯云为例，假设对象存储访问地址为 https://my-cos.cos.ap-guangzhou.myqcloud.com
        # 拆分对应的桶名称 bucket 和访问后缀 endpoint
        "bucket" => "my-cos"
        # 如果是 AWS S3 可以省去 endpoint
        "endpoint" => "https://cos.ap-guangzhou.myqcloud.com"
        # 对应的 region
        "region" => "ap-guangzhou"
        # 存储目标路径
        "prefix" => "%{[@metadata][kafka][topic]}/%{+YYYY-MM-dd}"
        "codec" => "json_lines"
        "encoding" => "gzip"
        "validate_credentials_on_root_bucket" => "false"
        # 上传对象的存储类型，默认为标准存储 STANDARD
        "storage_class" => "STANDARD"
        # 当文件大小达到 500M 时， logstash 将消费完成的临时文件上传到对象存储
        "rotation_strategy" => "size"
        "size_file" => "524288000"
        "additional_settings" => {
            "force_path_style" => "true"
            "follow_redirects" => "false"
        }
    }
}
```

比较常用的 S3 对象存储类型有：
* `S3 Standard` ：标准存储，也是通用型的存储类型，对应 S3 插件 `storage_class` 中的 `STANDARD` 。
* `S3 Standard-IA` ：低频存储，用于访问量较低的存储类型，对应 S3 插件 `storage_class` 中的 `STANDARD_IA` 。
* `S3 One Zone-IA` ：单 Zone 的低频存储，它的 Availability Zones 比 `S3 Standard-IA` 少，对应 S3 插件 `storage_class` 中的 `ONEZONE_IA` 。
* `S3 Reduced Redundancy Storage` ：低冗余性，非关键性的高频访问存储类型，对应 S3 插件 `storage_class` 中的 `REDUCED_REDUNDANCY` 。

当使用 S3 插件作为 Logstash 的 `input` 时，需要将对象存储中的存储类型恢复为 `STANDARD` 才能正常读取，因为云厂商提供的对象存储服务通常会在一定的时间后将它们转化为归档存储。

以下 Logstash 配置可以将对象存储中日志文件还原到 Elasticsearch 集群：

``` bash
# 消费对象存储中的存储文件，输出到 Elasticsearch 集群
input {
    s3 {
        # 替换为对应的密钥信息
        "access_key_id" => "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
        "secret_access_key" => "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
        # 以腾讯云为例，假设对象存储访问地址为 https://my-cos.cos.ap-guangzhou.myqcloud.com
        # 拆分对应的桶名称 bucket 和访问后缀 endpoint
        "bucket" => "my-cos"
        # 如果是 AWS S3 可以省去 endpoint
        "endpoint" => "https://cos.ap-guangzhou.myqcloud.com"
        # 对应的 region
        "region" => "ap-guangzhou"
        "codec" => "json"
        # 消费完成后关闭输入，不等待新的文件产生
        "watch_for_new_files" => false
        # 可以只消费某些特定前缀的存储文件
        "prefix" => "logs-"
    }
}

output {
    elasticsearch {
        index => "logs-from-cos-%{+YYYY.MM.dd}"
        hosts => ["http://localhost:9200"]
        user => "elastic"
        password => "elastic_password"
    }
}
```
