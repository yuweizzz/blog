---
date: 2024-04-03 19:44:45
title: openresty 学习笔记
tags:
  - "Lua"
  - "Openresty"
  - "Apisix"
draft: false
---

这篇笔记用来记录一些 Openresty 开发的相关知识。

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

## 定义 lua_package_path

在引入一系列自定义的 lua package 的时候比较有用，事实上大部分基于 openresty 的网关项目都是这样做的。

```lua
# nginx.conf
http {
    lua_package_path '/usr/local/src/lua-resty-http/?.lua;;';
    lua_package_cpath '/usr/local/src/lua-cjson/?.so;;';
}
```

## body_filter_by_lua_block 的流式处理

openresty 请求响应可能会经过多次 body_filter_by_lua_block 阶段。其中在 body_filter_by_lua_block 阶段的 `ngx.arg[1]` 和 `ngx.arg[2]` 参数分别代表响应内容和响应结束标记。

```lua
-- apisix brotli plugin
-- body_filter 函数会在 body_filter_by_lua_block 中执行
function _M.body_filter(conf, ctx)
    if not ctx.brotli_matched then
        return
    end

    local chunk, eof = ngx.arg[1], ngx.arg[2]
    if type(chunk) == "string" and chunk ~= "" then
        local encode_chunk = ctx.compressor:compress(chunk)
        ngx.arg[1] = encode_chunk .. ctx.compressor:flush()
    end

    if eof then
        ngx.arg[1] = ngx.arg[1] .. ctx.compressor:finish()
    end
end
```

eof 标记为 true 时 chunk 内容不一定为空，此时的 chunk 就是最后的响应内容。在某些响应体内容修改的场景中，可能需要将所有 `ngx.arg[1]` 内容进行合并后在处理，并且应该在 `header_filter_by_lua_block` 阶段将 `ngx.header.content_length` 置空。

```lua
-- 合并 ngx.arg[1] 可以参考 apisix hold_body_chunk 函数
-- apisix/core/response.lua
local arg = ngx.arg
function _M.hold_body_chunk(ctx, hold_the_copy)
    local body_buffer
    local chunk, eof = arg[1], arg[2]

    if not ctx._body_buffer then
        ctx._body_buffer = {}
    end

    if type(chunk) == "string" and chunk ~= "" then
        body_buffer = ctx._body_buffer[ctx._plugin_name]
        if not body_buffer then
            body_buffer = {
                chunk,
                n = 1
            }
            ctx._body_buffer[ctx._plugin_name] = body_buffer
        else
            local n = body_buffer.n + 1
            body_buffer.n = n
            body_buffer[n] = chunk
        end
    end

    if eof then
        body_buffer = ctx._body_buffer[ctx._plugin_name]
        if not body_buffer then
            return chunk
        end

        body_buffer = concat_tab(body_buffer, "", 1, body_buffer.n)
        ctx._body_buffer[ctx._plugin_name] = nil
        return body_buffer
    end

    if not hold_the_copy then
        -- flush the origin body chunk
        arg[1] = nil
    end
    return nil
end
```

## proxy_next_upstream 和 balancer_by_lua_block

proxy_next_upstream 是 ngx_http_proxy_module 提供的关键字，具体用法参考 `proxy_next_upstream error | timeout | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...;` ，默认定义是 `proxy_next_upstream error timeout;` 。它定义了是否在对应的状态下将请求发送到下一个 upstream ，实际上就是 nginx 的重试机制。

在 openresty 中，这部分可以配合 balancer_by_lua_block 来自定义上游的选择。但是只有在访问上游出现的错误符合所定义的条件时，才会再次进入 balancer_by_lua_block 执行阶段。

```lua
# openresty balancer_by_lua_block doc
http {
    upstream backend {
        server 0.0.0.1;   # just an invalid address as a place holder

        balancer_by_lua_block {
            local balancer = require "ngx.balancer"

            -- well, usually we calculate the peer's host and port
            -- according to some balancing policies instead of using
            -- hard-coded values like below
            local host = "127.0.0.2"
            local port = 8080

            local ok, err = balancer.set_current_peer(host, port)
            if not ok then
                ngx.log(ngx.ERR, "failed to set the current peer: ", err)
                return ngx.exit(500)
            end
        }

        keepalive 10;  # connection pool
    }

    server {
        # this is the real entry point
        listen 80;

        location / {
            # make use of the upstream named "backend" defined above:
            proxy_pass http://backend/fake;
        }
    }

    server {
        # this server is just for mocking up a backend peer here...
        listen 127.0.0.2:8080;

        location = /fake {
            echo "this is the fake backend peer...";
        }
    }
}
```

进入 balancer_by_lua_block 阶段后可以通过 `ngx.balancer.set_more_tries` 来设置新增的重试次数，会在原有的重试次数上增加。所以应该额外设置 proxy_next_upstream_tries 或者 proxy_next_upstream_timeout ，否则可能出现无限重试的情况。而 `ngx.balancer.get_last_failure` 可以拿到进入本次 balancer_by_lua_block 阶段的原因以及状态码。

## Luajit string buffer

Luajit 提供了一个高效的 string buffer 模块，可以实现 FIFO 的字符缓存队列。

```lua
str_buffer = require("string.buffer")

strings = "abcdefghij"
-- new 方法可以指定给定的内存空间大小，但无论是否给定，后续内存空间都会自动适应增长
-- set 方法可以直接设置数据内容
buffer = str_buffer.new():set(strings)

print(buffer:get(5))
-- get 方法会将数据从队列中取出
-- out:
-- abcde
print(buffer:tostring())
-- tostring 方法可以输出当前队列内容，并且不修改原有队列
-- out:
-- fghij
print(buffer:get(5))
-- out:
-- fghij

-- 取完所有数据，后续的 get 和 tostring 都会返回 ""
if buffer:get(1) == "" then
    print([[get ""]])
    --- out:
    --- get ""
end

-- buffer 可以复用
buffer:put("12345")
print(buffer:get(5))
-- out:
-- 12345

-- reset 方法可以将 buffer 置空，相当于取出所有数据，但内存空间不会释放
buffer:reset()
-- free 方法可以主动释放内存空间
buffer:free()
```

## ngx.re module

在 openresty 中，正则相关的处理一般使用 ngx.re 模块。这个模块提供的函数和 Lua string 非常接近，同样都有 find ， match ， gmatch ， sub ， gsub 这几个函数。

```lua
location /find {
    content_by_lua_block {
        local regex = [[\d+]]
        -- find 同样是找到匹配式在目标字符串中的起止位置
        local from, to, err = ngx.re.find("hello, 1234", regex, "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        if from then
            ngx.say("from ", from, " to ", to)
        else
            ngx.say("not matched")
        end
    }
}

location /match {
    content_by_lua_block {
        local regex = [[\d+]]
        -- match 函数的返回值 m 是 table 类型
        -- 其中 m[0] 是匹配字符串，而是否存在 m[1] 和 m[2] 等字符串则取决于 regex 是否使用子匹配捕获
        local m, err = ngx.re.match("hello, 1234", regex, "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        if m then
            -- out:
            -- 1234
            ngx.say(m[0])
        else
            ngx.say("not matched")
        end
    }
}

location /gmatch {
    content_by_lua_block {
        local regex = [[\d]]
        -- gmatch 返回迭代器函数，每个迭代返回值和 match 一致
        local iterator, err = ngx.re.gmatch("hello, 1234", regex, "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        for each in iterator do
            -- out:
            -- 1
            -- 2
            -- 3
            -- 4
            ngx.say(each[0])
        end
    }
}

location /sub {
    content_by_lua_block {
        local regex = [[\d]]
        -- sub 函数会执行一次符合匹配式的字符串替换
        -- gsub 函数则会替换所有符合匹配式的字符串
        -- n 都用来表示匹配次数
        -- 没有成功匹配到替换内容， replaced 会原样输出
        local replaced, n, err = ngx.re.sub("hello, 1234", regex, "a", "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        -- out:
        -- hello, a234
        ngx.say(replaced)

        local replaced, n, err = ngx.re.gsub("hello, 1234", regex, "a", "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        -- out:
        -- hello, aaaa
        ngx.say(replaced)

        -- 使用子匹配替换
        local regex = [[(\d)(\d)]]
        local replaced, n, err = ngx.re.gsub("hello, 1234", regex, "$2$1", "jo")
        if err then
            ngx.log(ngx.ERR, "error: " .. err)
            return
        end
        -- out:
        -- hello, 2143
        ngx.say(replaced)
    }
}
```

## 运行 apisix 测试用例

关于如何搭建测试环境，这里引用 apisix 仓库文档 `docs/en/latest/building-apisix.md` 作为参考：

> 1. 安装 `perl` 的包管理器 [cpanminus](https://metacpan.org/pod/App::cpanminus#INSTALLATION)。
> 2. 通过 `cpanm` 来安装 [test-nginx](https://github.com/openresty/test-nginx) 的依赖：
>
>    ```shell
>    sudo cpanm --notest Test::Nginx IPC::Run > build.log 2>&1 || (cat build.log && exit 1)
>    ```
>
> 3. 将 `test-nginx` 源码克隆到本地：
>
>    ```shell
>    git clone https://github.com/openresty/test-nginx.git
>    ```
>
> 4. 运行以下命令将当前目录添加到 Perl 的模块目录：
>
>    ```shell
>    export PERL5LIB=.:$PERL5LIB
>    ```
>
>    你可以通过运行以下命令指定 NGINX 二进制路径：
>
>    ```shell
>    TEST_NGINX_BINARY=/usr/local/bin/openresty prove -Itest-nginx/lib -r t
>    ```
>
> 5. 运行测试：
>
>    ```shell
>    make test
>    ```

正常设置环境变量后，可以通过 `prove -Itest-nginx/lib -r t/plugin/file.t` 指定要运行的测试文件。其中 `TEST_NGINX_BINARY=/usr/local/bin/openresty` 可以通过主动声明 openresty 的可执行路径 `export PATH=/usr/local/openresty/nginx/sbin:$PATH` 替换，无须每次运行时再次声明。

测试过程经常会涉及一些资源创建和变更，以下是控制平面的一些常用的 API 收集：

```bash
# 在 admin key 不再使用默认值，而是运行时生成后，需要使用这个命令获取
admin_key=$(yq '.deployment.admin.admin_key[0].key' conf/config.yaml | sed 's/"//g')

# 创建 ssl 资源
curl http://localhost:9180/apisix/admin/ssls/1 \
-H "X-API-KEY: $admin_key" -X PUT -d '
{
    "cert" : "'"$(cat t/certs/apisix.crt)"'",
    "key": "'"$(cat t/certs/apisix.key)"'",
    "sni": "test.com",
    "client": {
        "ca": "'"$(cat t/certs/apisix.crt)"'",
        "depth": 1
    }
}'

# 创建 route 资源
curl http://localhost:9180/apisix/admin/routes/1 \
-H "X-API-KEY: $admin_key" -X PUT -d '
{
    "uri": "/*",
    "plugins": {
        "gzip": {}
    },
    "upstream": {
        "nodes": {
            "httpbin.org": 1
        },
        "type": "roundrobin"
    }
}'

# 创建 stream route 资源
curl http://localhost:9180/apisix/admin/stream_routes/1 \
-H "X-API-KEY: $admin_key" -X PUT -d '
{
    "server_addr": "192.168.1.88",
    "server_port": 9200,
    "upstream": {
        "nodes": {
            "192.168.1.99:9200": 1
        },
        "type": "roundrobin"
    }
}'

# 创建 consumer 资源
curl http://localhost:9180/apisix/admin/consumers \
-H "X-API-KEY: $admin_key" -X PUT -d '
{
    "username": "foo",
    "plugins": {
        "basic-auth": {
            "username": "foo",
            "password": "bar"
        }
    }
}'

# 创建关联 consumer 的 route 资源
curl http://localhost:9180/apisix/admin/routes/2 \
-H "X-API-KEY: $admin_key" -X PUT -d '
{
    "uri": "/*",
    "plugins": {
       "basic-auth": {}
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "httpbin.org": 1
        }
    }
}'
```
