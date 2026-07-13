---
date: 2026-04-10 10:00:00
title: 使用 fluent-bit 进行日志采集
tags:
  - "s3"
  - "fluent-bit"
  - "elasticsearch"
draft: false
---

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

这里介绍如何使用 fluent-bit 进行日志采集。

虽然 fluent-bit 可以使用自有格式的配置文件，但是为了通用性起见，这里一般使用 YAML 格式。

以下是 fluent-bit 可以使用的 YAML 格式配置文件：

```yaml
service:
  http_server: on
  http_listen: 0.0.0.0
  http_port: 2020
  hot_reload: on
  flush: 1
  log_level: info
  storage.path: /opt/fluent-bit/storage
  storage.max_chunks_up: 32
  storage.total_limit_size: 10G

multiline_parsers:
  - name: mylog
    type: regex
    rules:
      - state: start_state
        regex: '/^\d{4}/'
        next_state: cont
      - state: cont
        regex: '/^[\D]+/'
        next_state: cont

parsers:
  - name: myparser
    format: regex
    regex: '(?<date>^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3})\s+\[.*?\]\s+(?<level>[\w]+)\s+.*?\s+\#\s+\?\s+\-\s+\[(?<request_path>.*?)\][\s\S]+'

pipeline:
  inputs:
    - name: tail
      path: /var/logs/app-*.log
      read_from_head: false
      buffer_max_size: 5m
      storage.type: filesystem
      multiline.parser: mylog
      tag: app
      db: /opt/fluent-bit/storage/app.db
      inotify_watcher: false

  filters:
    - name: record_modifier
      match: "*"
      record:
        - hostname ${HOSTNAME}
    - name: lua
      match: "*"
      call: append_tag
      code: |
        function append_tag(tag, timestamp, record)
          new_record = record
          new_record["application"] = tag
          return 2, timestamp, new_record
        end
    - name: parser
      match: "*"
      key_name: log
      parser: myparser
      preserve_key: on
      reserve_data: on

  outputs:
    - name: stdout
      match: "*"
    - name: file
      match: "*"
      format: plain
    - name: s3
      match: "*"
      bucket: mybucket
      region: ap-southeast-1
      compression: gzip
      use_put_object: on
      # store_dir: /opt/fluent-bit/s3
      total_file_size: 200M
      upload_timeout: 20m
      retry_limit: 10
      s3_key_format: "/$TAG/%Y-%m-%d/%H:%M:%S-$UUID.json"
      # s3_key_format: '/$TAG[0]/$TAG[1]/%Y-%m-%d/%H:%M:%S-$UUID.gz'
      # s3_key_format_tag_delimiters: '.'
    - name: es
      match: "*"
      host: 192.168.1.1
      port: 9200
      index: app
      trace_error: on
      trace_output: on
      suppress_type_name: on
      tls: on
      tls.verify: off
      buffer_size: 5m
      http_api_key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

在了解各个配置项的定义内容后，可以根据自己的需要进行配置，以下是一些关键配置项的相关定义：

开启了 `http_server` 和 `hot_reload` 的情况下可以热重载，通过 `curl localhost:2020/api/v2/reload -X POST -d {}` 就可以执行重载。

在 tail 采集中， `buffer_max_size` 代表着最大的单行日志大小门限，如果采集内容超出这个限制会被丢弃。这里还使用了自定义的多行解析器，但是需要注意 `multiline.parser` 会有几种默认内置的解析器，包括 `Go` ，`Python` ，`Ruby` 和 `Java` ，自定义的多行解析器应该避免与他们重名。

tail 可以使用 `inotify_watcher` 关闭通过内核进行文件系统监控，这个特性是默认开启的，在大量日志写入的情况下，系统的 IO 占用率会被拉到非常高，可以通过关闭这个特性来降低。

`parser filter` 可以配合正则表达式，对指定的 `key_name` 字段的内容进行匹配，从而生成新的字段，其中 `preserve_key` 用来决定是否保留被指定的字段，而 `reserve_data` 用来决定是否保留其他未被指定的字段。

s3 可以使用 multipart upload 进行对象上传，在使用磁盘缓冲的情况下，可以节省非常多的内存资源，但是如果 multipart upload 上传中断，处理起来比较麻烦，这里是使用 put object 的上传方式，坏处是会占用 `total_file_size` 的内存空间，而且当你的输入管道越多，占用越高。所以可以根据具体情况来决定使用哪种上传方式。

s3 可以自定义上传的文件路径，通过 `s3_key_format_tag_delimiters` 设定 tag 的分隔符，就可以以数组的方式来读取分割后的 tag 值。参考注释的 `s3_key_format` 部分。

在 es 中， `suppress_type_name` 用来兼容没有 `_type` 字段的高版本 Elasticsearch ， `index` 则是写入的索引名称， tls 根据目标集群的配置来设置即可，这里开启了 tls 传输但是不做证书验证， `trace_*` 用来输出 Elasticsearch 的 API 返回值，可以用于排查错误。
