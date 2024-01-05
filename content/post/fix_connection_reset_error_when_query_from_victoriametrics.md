---
date: 2023-12-15 10:00:00
title: 解决使用 VictoriaMetrics 时的连接重置问题
tags:
  - "VictoriaMetrics"
  - "Go"
draft: false
---

这篇笔记用来记录如何解决使用 VictoriaMetrics 过程中查询端出现 connection reset by peer 报错的问题。

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

在使用 VictoriaMetrics 作为时序数据源时，偶尔发现查询端的日志中存在 connection reset by peer 的报错。

在发现这个问题后，我第一时间检查了 VictoriaMetrics 的日志，但是对应的日志中并没有关于连接重置的错误日志，所以这应该不是 VictoriaMetrics 服务本身在请求过程中发生问题导致的连接重置。

由于服务端的信息比较少，所以只能从查询端入手排查，根据查询端的代码，整个查询过程中共用一个封装好的查询结构体，深扒一下代码，原来这个结构体是基于 `http.Client` 来实现的。

由于 VictoriaMetrics 和查询端都是 Go 语言编写的，所以尽量查找一些 Go 在网络相关方面的错误案例，很快就有了线索，那就是 `http.Client` 的连接管理问题。

在 `http.Client` 结构体中的 `Transport` 部分维护着一个连接池，实际上可以通过 `DisableKeepAlives` 选项来禁用长连接，不过在查询端中为了提高效率，并没有使用这个选项，还是选择使用这个连接池来执行具体的查询请求。

因为 VictoriaMetrics 默认的关闭空闲连接的时间是 `1m` ，而查询端是周期性发起查询请求，其中大部分都是 `30s` 查询一次，这样就有可能正好撞上 VictoriaMetrics 的空闲连接关闭时间，此时上游已经关闭连接，而查询端仍然向这个连接发起查询请求，从而产生 connection reset by peer 的错误。

要解决这个问题有两个方法，一个是完全禁用长连接，这样可以从根本解决问题，但是要修改查询端的代码，而且这样就不能复用连接资源了，另一个方法是控制上游的连接超时时间，使得长连接在适当的时间回收。

这里选择的是第二个方法， VictoriaMetrics 提供了控制连接超时时间的选项 `-http.idleConnTimeout` ，将这个时间从默认的 `1m` 调整到 `100s` 后，就可以避免大量查询周期为 `30s` 的请求产生错误。这个时间的调节应该根据实际情况出发，不论是选择更长或者更短的空闲等待时间都是可以的，只要能够避开查询端的请求周期就没问题。但还是推荐尽量使用更长的时间，这样可以节省发起连接的资源损耗。