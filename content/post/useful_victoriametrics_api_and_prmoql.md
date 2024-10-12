---
date: 2023-12-16 10:00:00
title: VictoriaMetrics API 和常用的 PromQL
tags:
  - "VictoriaMetrics"
  - "Prometheus"
draft: false
---

这篇笔记用来记录一些用到的 VictoriaMetrics API 和常用的 PromQL 。

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

## VictoriaMetrics API

这里主要指的是 VictoriaMetrics 单机版所使用的 API 。

后续内容均使用以下 metrics 作为例子：

```text
http_requests_total{code="200",handler="query",instance="localhost:9090",job="prometheus",method="get"}  1
http_requests_total{code="200",handler="query_range",instance="localhost:9090",job="prometheus",method="get"}  0
```

查询相关的 API 有即时查询和范围查询两种：

```bash
# 即时查询
curl http://localhost:8428/prometheus/api/v1/query \
  -d 'query=http_requests_total'

# 范围查询
curl http://localhost:8428/prometheus/api/v1/query_range \
  -d 'query=http_requests_total' \
  -d 'start=-1d' \
  -d 'step=1h'
```

即时查询一般会返回最新的数据样本，但是在指明某些时间节点时，如果没有直接命中对应的数据样本，会返回距离该时间最近的数据节点。

范围查询和即时查询相似，但是它可以返回多个数据样本，并且同样会尽量返回某些接近的数据样本来替换一些缺失数据。

所以两者返回的数据都不一定是实际时间对应的数据，可能是一个替换值。

删除相关的 API ：

```bash
curl http://localhost:8428/api/v1/admin/tsdb/delete_series \
  -d 'match[]=http_requests_total'
```

该操作会将整个指标都直接删除，并且占用的存储空间不会立即释放。

## 常用的 PromQL

这里会介绍一些比较常用的 PromQL 。

### label_replace

label_replace 的具体用法可以参考这个公式： `label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)` 。

主要用来替换某些标签内容，实际用例参考：

```text
before:
http_requests_total{code="200",handler="query",instance="localhost:9090",job="prometheus",method="get"}  1
http_requests_total{code="200",handler="query_range",instance="localhost:9090",job="prometheus",method="get"}  0

expr:
label_replace(http_requests_total{instance="localhost:9090"},"instance","$1","job","(.*)")

after:
http_requests_total{code="200",handler="query",instance="prometheus",job="prometheus",method="get"}  1
http_requests_total{code="200",handler="query_range",instance="prometheus",job="prometheus",method="get"}  0
```

### increase

increase 用来计算区间向量的增长量，以区间向量的第一个元素和最后一个元素进行计算，实际用例参考：

```text
before:
http_requests_total{code="200"} 100375 @1708045914.967
http_requests_total{code="200"} 100377 @1708045924.967
http_requests_total{code="200"} 100378 @1708045934.967
http_requests_total{code="200"} 100379 @1708045944.967
http_requests_total{code="200"} 100381 @1708045954.967
http_requests_total{code="200"} 100381 @1708045964.967

expr:
increase(http_requests_total{code="200"}[1m])

after:
http_requests_total{code="200"} 7
```

increase 存在数据外推现象，所以实际使用的计算数据可能不是简单的区间向量的第一个元素和最后一个元素直接运算，而会通过外推后的值来平衡计算。

### rate 和 irate

rate 和 irate 都可以用来计算区间向量中数据的增长率，但是 rate 是以区间向量的第一个元素和最后一个元素进行计算，而 irate 是以区间向量的最新两个元素进行计算： `rate(http_requests_total{code="200"}[10m])` 和 `irate(http_requests_total{code="200"}[10m])` 。

rate 存在数据外推现象，它基本上是基于 increase 的计算值再和时间进行计算。而 irate 只会取最近的两个元素进行计算，所以不会存在数据外推。

### group_left 和 group_right

group_left 和 group_right 是用于向量匹配的关键字，允许不同向量之间的多对一或者一对多的匹配。

参考公式：

```text
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

实际用例参考：

```text
before:
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120

expr:
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m

after:
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

在上述例子中，我们需要先明确在计算表达式中，多的对象指的是 `method_code:http_errors:rate5m` ，而一的对象指的是 `method:http_requests:rate5m` 。它是由向量处于 group_left 或者 group_right 的左右位置决定的。

on 或者 ignoring 是对多所指的对象而言的， ignoring 用来忽略这个标签， on 用来匹配这个标签。由于这个例子中， code 标签被忽略，所以使用 method 标签进行匹配。

group_left 实际上还可以声明额外的标签，以允许使用一对象中的标签覆盖最终结果。由于这个例子中，只有单一的 method 标签，可以通过下面这里扩展例子来理解：

```text
before:
method_code:http_errors:rate5m{method="get", code="500", url="endpoint/500"}  24
method_code:http_errors:rate5m{method="get", code="404", url="endpoint/404"}  30
method_code:http_errors:rate5m{method="put", code="501", url="endpoint/501"}  3
method_code:http_errors:rate5m{method="post", code="500", url="endpoint/500"} 6
method_code:http_errors:rate5m{method="post", code="404", url="endpoint/404"} 21

method:http_requests:rate5m{method="get", url="endpoint"}  600
method:http_requests:rate5m{method="del", url="endpoint"}  34
method:http_requests:rate5m{method="post", url="endpoint"} 120

expr:
method_code:http_errors:rate5m / on(method) group_left(url) method:http_requests:rate5m

after:
{method="get", code="500", url="endpoint"}  0.04            //  24 / 600
{method="get", code="404", url="endpoint"}  0.05            //  30 / 600
{method="post", code="500", url="endpoint"} 0.05            //   6 / 120
{method="post", code="404", url="endpoint"} 0.175           //  21 / 120
```
