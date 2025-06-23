---
date: 2025-06-10 11:14:00
title: 使用 Categraf 采集日志存储到 VictoriaLogs
tags:
  - "Openresty"
  - "Zstandard"
  - "Lua"
draft: false
---

使用 Categraf 采集日志存储到 VictoriaLogs 。

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

Categraf 内置的日志采集功能是从 DataDog Agent 中提取的，而 Datadog Agent 不支持自定义请求路径，所以在请求到达 VictoriaLogs 时需要通过 VMAuth 进行转换。

VMAuth 的配置文件：

``` bash
# vmauth.conf
unauthorized_user:
  url_map:
    - src_paths:
        - "/api/v2/logs"
        - "/api/v1/validate"
      url_prefix: "http://localhost:9428/insert/datadog/"
    - src_paths:
        - "/api/v1/series"
        - "/api/v2/series"
        - "/api/beta/sketches"
        - "/api/v1/validate"
        - "/api/v1/check_run"
        - "/intake"
        - "/api/v1/metadata"
      url_prefix: "http://localhost:9428/datadog/"
```

VMAuth 对应的 systemd service 文件：

```bash
[Unit]
Description=vmauth-prod
After=network.target

[Service]
Restart=on-failure
WorkingDirectory=/opt/vmutils
ExecStart=/opt/vmutils/vmauth-prod -auth.config=vmauth.conf

[Install]
WantedBy=multi-user.target
```

VictoriaLogs 对应的 systemd service 文件：

```bash
[Unit]
Description=victoria-logs-prod
After=network.target

[Service]
Restart=on-failure
WorkingDirectory=/opt/victorialogs/
ExecStart=/opt/victorialogs/victoria-logs-prod -storageDataPath=victoria-logs-data

[Install]
WantedBy=multi-user.target
```

在已经安装的 Categraf 中修改配置文件 `conf/logs.toml` ，参考以下配置：

```bash
[logs]
# 启用日志收集功能
enable = true
# 对应的 VMAuth 监听地址
send_to = "localhost:8427"
send_type = "http"
# topic 会在日志中以字段形式出现
topic = "flashcatcloud"
use_compress = false
send_with_tls = false
batch_wait = 5
run_path = "/opt/categraf/run"
open_files_limit = 100
scan_period = 10
frame_size = 9000

chan_size = 1000
pipeline = 4
kafka_version = "3.3.2"
batch_max_concurrence = 0
batch_max_size = 100
batch_max_contentsize = 1000000
producer_timeout = 10

sasl_enable = false
sasl_user = "admin"
sasl_password = "admin"
sasl_mechanism = "PLAIN"
sasl_version = 1
sasl_handshake = true

enable_collect_container = false

collect_container_all = false

[[logs.items]]
type = "file"
# 需要采集的日志文件
path = "/var/log/nginx/access.log"
# source 和 service 会在日志中以字段形式出现
source = "nginx"
service = "www.service.com"
```

重启 Categraf 后配置生效，这样就可以不用额外运行其他的日志收集器。
