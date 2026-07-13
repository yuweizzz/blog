---
date: 2026-04-10 10:00:00
title: 使用 vector 进行日志采集
tags:
  - "s3"
  - "vector"
  - "victorialogs"
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

vector 是日志采集工具的新秀，性能确实很优越，但是代价是资源消耗高，在使用它之前应该权衡自己的业务日志量和计算资源是否适配。

以下是 vector 可以使用的 toml 格式配置文件：

```toml
timezone = "local"
data_dir = "/var/lib/vector"

[sources.app_log]
type = "file"
include = ["/var/logs/app-*.log"]
# 允许最大的行大小为 5MB
max_line_bytes = 5242880

# 跨行合并
[sources.app_log.multiline]
# 首行正则
start_pattern = '^\d{4}'
mode = "continue_through"
# 子行正则
condition_pattern = '^[\D]+'
timeout_ms = 1000

# 从 S3 存储桶取回日志，需要配置带有 sqs 队列的存储桶
# [sources.from_aws_s3]
# type = "aws_s3"
# region = "ap-southeast-1"
# auth.access_key_id = "xxxxxxxxxxxxxxxxxxxx"
# auth.secret_access_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
# sqs.queue_url = "https://sqs.ap-southeast-1.amazonaws.com/000000000001/applog_queue"
# decoding.codec = "json"

[transforms.remap_applog]
inputs = ["app_log"]
type = "remap"
source = '''
  # 默认的消息体存储在 .message 之中，这里使用日志内容覆写，只单独提取时间和应用名字段
  pattern = r'^(?P<date>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\.\d{3})\s+(?P<application>[\w]+)\s+(?P<message>[\s\S]+)'
  parsed_message, err = parse_regex(.message, pattern)
  if err == null {
    . = merge(., parsed_message)
  } else {
    .format_log_error = err
  }
  # 将时间戳带上时区信息进行转换
  parsed_timestamp, err = parse_timestamp(.date, format: "%Y-%m-%d %H:%M:%S.%f", timezone: "Asia/Shanghai")
  if err == null {
    .date = parsed_timestamp
  } else {
    .format_timestamp_error = err
  }
  # 删除无用的字段
  del(.useless)
'''

# 写入到 VictoriaLogs
[sinks.to_vlog]
inputs = ["remap_applog"]
type = "http"
uri = "http://localhost:9428/insert/jsonline?_stream_fields=host&_msg_field=message&_time_field=date"
compression = "gzip"
encoding.codec = "json"
framing.method = "newline_delimited"
healthcheck.enabled = false

# 写入到 s3 存储桶
[sinks.to_aws_s3]
inputs = ["remap_applog"]
type = "aws_s3"
region = "ap-southeast-1"
storage_class = "STANDARD"
auth.access_key_id = "xxxxxxxxxxxxxxxxxxxx"
auth.secret_access_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
bucket = "bucket"
encoding.codec = "json"
buffer.type = "disk"
# 使用 1Gb 的磁盘缓冲空间
buffer.max_size = 1073741824
# 当缓冲空间不足时，阻塞消息写入
buffer.when_full = "block"
# 每 15 分钟执行一次 batch
batch.timeout_secs = 900
# 单次 batch 最大使用 500MB
batch.max_bytes = 524288000
compression = "gzip"
content_encoding = "gzip"
content_type = "application/json"
filename_append_uuid = true
filename_extension = "json"
filename_time_format = "%H:%M:%S"
force_path_style = true
key_prefix = "/{{ application }}/%F/"
```

单行采集和多行匹配的机制，和其他文件采集工具没有特别大的差别，比较特殊的是数据处理的部分， vector 使用了独特的 `Vector Remap Language` ，也就是 `VRL` 进行数据处理，这里使用到的 `VRL` 适用于最简单的场景，使用正则表达式取出相应的字段，同时进行时区转换。

关于数据传输的目标端的部分，大部分常用的解决方案都支持，包括 elasticsearch ，消息队列，或者是 s3 以及各种云厂商的云存储服务。

在数据输出时， s3 有比较特殊处理机制，就是通过 [buffer 和 batch](https://vector.dev/docs/reference/configuration/sinks/aws_s3/#buffers-and-batches) 进行数据缓冲， vector 会将处理后的数据存放在 buffer 中，在达到一个 batch 的量值再进行上传，这里可以优化的地方是 buffer 应该尽量放在磁盘中，并且调大存储空间，相应的 batch 的触发值可以调整成较长时间和较大值，这样发送到 s3 时就不会特别细碎，但是单次数据量太大也不太合适，这样回放数据时可能非常吃资源，示例的配置文件使用了压缩，最终数据大小基本维持在 200 MB 以下。

使用 s3 进行数据回放时， `sqs.queue_url` 是必填选项，因为 vector 的机制是从通知队列获取到新上传的对象信息，再根据这些信息从 s3 中进行下载。首先需要在 sqs 中创建队列，在 IAM 设置时，需要允许对应的 s3 bucket 向 sqs 中发送消息，然后需要在 s3 bucket 的属性中，创建事件通知的规则，只需要对象创建事件即可。
