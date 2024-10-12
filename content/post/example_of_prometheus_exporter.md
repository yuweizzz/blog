---
date: 2024-03-31 13:26:00
title: Prometheus exporter 代码参考
tags:
  - "Prometheus"
  - "Go"
draft: false
---

这篇笔记主要记录如何自行编写 Prometheus exporter 。

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

本篇根据网络上找到的样板修改而来，首先通过 `go mod init myexporter` 来创建模块，具体的代码结构如下：

```bash
.
├── collector
│   └── http.go
├── go.mod
├── go.sum
└── main.go
```

`http.go` 代码内容如下：

```go
package collector

import (
 "github.com/prometheus/client_golang/prometheus"
 "net/http"
 "sync"
 "time"
)

type StatusCollector struct {
 requestDesc *prometheus.Desc
 mutex       sync.Mutex
}

func NewStatusCollector() prometheus.Collector {
 return &StatusCollector{
  requestDesc: prometheus.NewDesc(
   "http_request_status",
   "The Status Code of http request",
   nil,
   prometheus.Labels{"url": "https://www.baidu.com"}),
 }
}

func (n *StatusCollector) Describe(ch chan<- *prometheus.Desc) {
 ch <- n.requestDesc
}

func (n *StatusCollector) Collect(ch chan<- prometheus.Metric) {
 n.mutex.Lock()
 client := http.Client{
  Timeout: 5 * time.Second,
 }
 resp, err := client.Head("https://www.baidu.com")
 if err != nil {
  ch <- prometheus.MustNewConstMetric(n.requestDesc, prometheus.GaugeValue, 0)
  n.mutex.Unlock()
  return
 }
 ch <- prometheus.MustNewConstMetric(n.requestDesc, prometheus.GaugeValue, float64(resp.StatusCode))
 n.mutex.Unlock()
}
```

`main.go` 代码内容如下：

```go
package main

import (
 "myexporter/collector"
 "fmt"
 "github.com/prometheus/client_golang/prometheus"
 "github.com/prometheus/client_golang/prometheus/promhttp"
 "net/http"
)

func main() {
 r := prometheus.NewRegistry()
 r.MustRegister(collector.NewStatusCollector())
 handler := promhttp.HandlerFor(r, promhttp.HandlerOpts{})
 http.Handle("/metrics", handler)
 if err := http.ListenAndServe(":8080", nil); err != nil {
  fmt.Printf("Error occur when start server %v", err)
 }
}
```

代码编写完成后，直接通过 `go get` 和 `go build` 下载依赖模块和编译代码，就可以得到可执行文件。

建议通过 `CGO_ENABLED=0 go build -ldflags="-w -s" main.go` 进行编译，可以得到静态链接的可执行文件并且文件体积比较小。
