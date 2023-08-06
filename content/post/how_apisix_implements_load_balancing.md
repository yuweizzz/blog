---
date: 2023-08-03 21:16:45
title: Apisix 的负载均衡实现
tags:
  - "apisix"
  - "openresty"
  - "lua"
draft: false
---

这篇笔记用来分析 Apisix 是如何实现负载均衡的。

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

## Route 和 Upstream

在 Apisix 中， `Route` 是最基础的资源对象，我们通过它来定义不同类型的请求，而 `Upstream` 是 `Route` 中请求的实际处理者， Apisix 会根据 `Upstream` 中的定义，对服务节点进行负载均衡，这一步也是这里想要分析的内容。

## 源码解析

熟悉 Openresty 的都应该知道它是根据请求的不同阶段来进行逻辑处理，同样地我们将 Apisix 中的 `access_by_lua_block` 和 `balancer_by_lua_block` 作为我们的分析入口。根据 `apisix/cli/ngx_tpl.lua` 可以找到这两个 block 分别对应了 `apisix.http_access_phase()` 和 `apisix.http_balancer_phase()` 这两个函数，所以我们需要在 `apisix/init.lua` 中的代码寻找这两个函数的定义。

我们可以看到 `apisix.http_access_phase()` 主要做了以下工作：

1. 创建上下文并且根据 ngx 中的信息写入到其中。
2. 根据上下文信息来匹配 `Route` 。
3. 匹配成功后，执行 `Route` 中定义的对应阶段中需要执行的插件。
4. 进行上游信息处理。

在第四步的进行上游信息处理中，具体工作交由函数 `handle_upstream()` 中来完成，可以看到其中有两个非常重要的调用：

* `set_upstream()` 根据 `Route` 找到对应的 `Upstream` ，这里 `set_upstream()` 是来自模块 `apisix/upstream.lua` 中的函数。
* `load_balancer.pick_server()` 会根据 `Upstream` 中的配置选出具体的上游节点，这里的 `load_balancer` 模块实际就是 `apisix/balancer.lua` 。

在 `handle_upstream()` 的最后阶段，选中的上游节点会写入到上下文 `api_ctx.picked_server` 之中，整个 `apisix.http_access_phase()` 过程也就基本结束了。

而 `apisix.http_balancer_phase()` 需要做的事情比较简单，因为它直接对应 `balancer_by_lua_block` 阶段，所以它的主要工作就是调用 `load_balancer.run()` ，也就是 `apisix/balancer.lua` 中的 `run()` 。

下面是 `apisix/balancer.lua` 的代码片段，主要有 `create_server_picker()` ， `pick_server()` 和 `run()` 三个函数的具体内容。

``` lua
local function create_server_picker(upstream, checker)
    -- 根据 upstream 中定义的 type 选择具体的负载均衡算法
    local picker = pickers[upstream.type]
    if not picker then
        pickers[upstream.type] = require("apisix.balancer." .. upstream.type)
        picker = pickers[upstream.type]
    end

    if picker then
        local nodes = upstream.nodes
        local addr_to_domain = {}
        for _, node in ipairs(nodes) do
            if node.domain then
                local addr = node.host .. ":" .. node.port
                addr_to_domain[addr] = node.domain
            end
        end

        local up_nodes = fetch_health_nodes(upstream, checker)

        -- _priority_index 和 upstream 中定义的 priority 有关 
        if #up_nodes._priority_index > 1 then
            core.log.info("upstream nodes: ", core.json.delay_encode(up_nodes))
            -- 这里的 priority_balancer 就是 apisix/balancer/priority.lua
            local server_picker = priority_balancer.new(up_nodes, upstream, picker)
            server_picker.addr_to_domain = addr_to_domain
            return server_picker
        end

        core.log.info("upstream nodes: ",
                      core.json.delay_encode(up_nodes[up_nodes._priority_index[1]]))
        local server_picker = picker.new(up_nodes[up_nodes._priority_index[1]], upstream)
        server_picker.addr_to_domain = addr_to_domain
        return server_picker
    end

    return nil, "invalid balancer type: " .. upstream.type, 0
end

-- pick_server will be called:
-- 1. in the access phase so that we can set headers according to the picked server
-- 2. each time we need to retry upstream
local function pick_server(route, ctx)
    core.log.info("route: ", core.json.delay_encode(route, true))
    core.log.info("ctx: ", core.json.delay_encode(ctx, true))
    local up_conf = ctx.upstream_conf

    for _, node in ipairs(up_conf.nodes) do
        if core.utils.parse_ipv6(node.host) and str_byte(node.host, 1) ~= str_byte("[") then
            node.host = '[' .. node.host .. ']'
        end
    end

    local nodes_count = #up_conf.nodes
    if nodes_count == 1 then
        local node = up_conf.nodes[1]
        ctx.balancer_ip = node.host
        ctx.balancer_port = node.port
        node.upstream_host = parse_server_for_upstream_host(node, ctx.upstream_scheme)
        return node
    end

    local version = ctx.upstream_version
    local key = ctx.upstream_key
    local checker = ctx.up_checker

    ctx.balancer_try_count = (ctx.balancer_try_count or 0) + 1
    if ctx.balancer_try_count > 1 then
        if ctx.server_picker and ctx.server_picker.after_balance then
            ctx.server_picker.after_balance(ctx, true)
        end

        if checker then
            local state, code = get_last_failure()
            local host = up_conf.checks and up_conf.checks.active and up_conf.checks.active.host
            local port = up_conf.checks and up_conf.checks.active and up_conf.checks.active.port
            if state == "failed" then
                if code == 504 then
                    checker:report_timeout(ctx.balancer_ip, port or ctx.balancer_port, host)
                else
                    checker:report_tcp_failure(ctx.balancer_ip, port or ctx.balancer_port, host)
                end
            else
                checker:report_http_status(ctx.balancer_ip, port or ctx.balancer_port, host, code)
            end
        end
    end

    if checker then
        version = version .. "#" .. checker.status_ver
    end

    -- the same picker will be used in the whole request, especially during the retry
    -- 这里使用了 LRU 缓存，通过 server_picker 的复用达到节省资源的目的
    local server_picker = ctx.server_picker
    if not server_picker then
        server_picker = lrucache_server_picker(key, version,
                                               create_server_picker, up_conf, checker)
    end
    if not server_picker then
        return nil, "failed to fetch server picker"
    end

    -- 每个实现具体算法的 balancer 都具有 get() 方法
    local server, err = server_picker.get(ctx)
    if not server then
        err = err or "no valid upstream node"
        return nil, "failed to find valid upstream server, " .. err
    end
    ctx.balancer_server = server

    local domain = server_picker.addr_to_domain[server]
    local res, err = lrucache_addr(server, nil, parse_addr, server)
    if err then
        core.log.error("failed to parse server addr: ", server, " err: ", err)
        return core.response.exit(502)
    end

    res.domain = domain
    ctx.balancer_ip = res.host
    ctx.balancer_port = res.port
    ctx.server_picker = server_picker
    res.upstream_host = parse_server_for_upstream_host(res, ctx.upstream_scheme)

    return res
end

-- run 函数应该只用于 balancer_by_lua_block 阶段
function _M.run(route, ctx, plugin_funcs)
    local server, err

    -- 理论上在 access_by_lua_block 阶段已经拿到具体的上游节点
    if ctx.picked_server then
        -- use the server picked in the access phase
        server = ctx.picked_server
        ctx.picked_server = nil

        set_balancer_opts(route, ctx)

    else
        if ctx.proxy_retry_deadline and ctx.proxy_retry_deadline < ngx_now() then
            -- retry count is (try count - 1)
            core.log.error("proxy retry timeout, retry count: ", (ctx.balancer_try_count or 1) - 1,
                           ", deadline: ", ctx.proxy_retry_deadline, " now: ", ngx_now())
            return core.response.exit(502)
        end
        -- retry
        -- 如果在 access_by_lua_block 阶段获取上游节点失败，在这里显式调用 pick_server 重试
        server, err = pick_server(route, ctx)
        if not server then
            core.log.error("failed to pick server: ", err)
            return core.response.exit(502)
        end

        local header_changed
        local pass_host = ctx.pass_host
        if pass_host == "node" then
            local host = server.upstream_host
            if host ~= ctx.var.upstream_host then
                -- retried node has a different host
                ctx.var.upstream_host = host
                header_changed = true
            end
        end

        local _, run = plugin_funcs("before_proxy")
        -- always recreate request as the request may be changed by plugins
        if run or header_changed then
            balancer.recreate_request()
        end
    end

    core.log.info("proxy request to ", server.host, ":", server.port)

    local ok, err = set_current_peer(server, ctx)
    if not ok then
        core.log.error("failed to set server peer [", server.host, ":",
                       server.port, "] err: ", err)
        return core.response.exit(502)
    end

    ctx.proxy_passed = true
end
```

`create_server_picker()` 用于生成具体的 `balancer` 并将它返回，可以看到它是以 `Upstream` 信息作为参数的，其中 `type` 则是具体指明需要调用哪个算法来生成 `balancer` ； `run()` 函数则是根据 `access_by_lua_block` 和 `balancer_by_lua_block` 的具体情况来执行连接到上游的动作；而 `pick_server()` 更是贯穿整个过程，不止在 `access_by_lua_block` 过程中被调用，也在 `run()` 中在重试阶段被调用，它的具体工作内容是这样的：

1. 拿到 `Route` 中的 `Upstream` ，并执行相关的上下文写入。
2. 如果 `Upstream` 中只定义一个上游节点，则直接返回，否则将检查请求的重试情况和各个上游节点的健康状态。
3. 尝试拿到对应的 `balancer` 并进行节点选择。
4. 选择成功，则执行相关的上下文写入。

可以看到，处理过程最后都由 `set_current_peer()` 函数来结束，它会连接到前面已经选出的上游节点，这样整个负载均衡的过程就基本结束了。

## 负载均衡算法

根据 Apisix 的源代码，我们可以看到有以下不同的 `balancer` ：

* `apisix/balancer/roundrobin.lua` ：对应的是 `round-robin` ，即轮询算法。
* `apisix/balancer/chash.lua` ：对应的是 `consistent hash` ，即一致性哈希算法。
* `apisix/balancer/least_conn.lua` ：对应的是 `least connected` ，即最小连接数算法。
* `apisix/balancer/ewma.lua` ：对应的是 `EWMA` ，即指数加权移动平均算法。
* `apisix/balancer/priority.lua` ：这个 `balancer` 应该是基于 `Upstream` 中自定义的优先级来实现的算法。

仔细观察所有实现具体算法的 `balancer` 可以发现，除去比较特殊的 `priority.lua` 之外，其他的 `balancer` 都实现了 `_M.new()` 的实例化函数以及 `get()` ， `after_balance()` 和 `before_retry_next_priority()` 三个方法，这相当于这些模块都是 `balancer` 作为不同算法的具体实现，而根据 `apisix/balancer.lua` 中的代码，在实际中决定启用哪一种 `balancer` 是由 `Upstream` 的 `type` 决定的。

关于负载均衡的算法，我们可以对比 Nginx 的负载均衡实现，原生的 Nginx 支持 `round-robin` ， `least connected` 和 `ip-hash` 三种，而 Apisix 也基本支持了这三种算法，还额外支持了 `EWMA` 算法。它是一种利用响应时间进行负载均衡的算法，它会根据上游节点的上次响应时间，和两次访问时间的间隔加上特定的权重来计算节点得分，最后选择得分最高的节点。

Apisix 中的 `round-robin` 算法是基于 Openresty 的 `resty.roundrobin` 模块来实现的，而 `chash` 算法是基于 `resty.chash` 实现的，它是通过 `Upstream` 来指定 hash key 的，而 Nginx 的 `ip-hash` 则是基于请求的 IP 地址作为 hash key 的，此外还可以使用 `hash key consistent` 的配置关键字来实现指定 hash key 的 chash 算法。

`round-robin` 算法是比较简单的，除了轮询之外，还可以支持带有权重的轮询，而 `chash` 算法则是根据根据 hash key 来计算 hash 值，后端节点们不再是传统的线性空间，而是看做一个环，实际节点会和环上的虚拟节点对应，通过计算得出的 hash 值落到环中，先命中虚拟节点而后得出实际节点，这种算法除了能够较好地均衡请求之外，还不会因为后端节点的增减引起大量的请求波动，因为实际节点变动后，这个实际节点对应的虚拟节点会被分散到就近的实际节点，不会像传统 hash 算法一样命中失败。

`least connected` 算法是基于上游节点的负载情况来选择的，将会选择连接数最小的节点作为请求的上游节点，这样做的最大好处是均衡性比较强。根据 Apisix 的源码我们可以看到它是基于二叉树来实现的。根据源代码 `apisix/balancer/least_conn.lua` ，可以看到 `balancer` 创建了一个基于节点权重的最小堆，这是通过将 `1 / weight` 作为排序依据来实现的，然后在每次请求到达时，通过 `heap.peek()` 取出节点，然后更新对应的排序值，实际上的更新后排序值就是 `请求值 / weight` ，这样就可以实现基于最小连接数的负载均衡。
