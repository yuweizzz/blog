---
date: 2022-03-12 10:22:45
title: PowerDNS 调优
tags:
  - "Linux"
  - "DNS"
  - "PowerDNS"
  - "Lua"
draft: false
---

这篇笔记用来记录 PowerDNS 的性能优化。

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

## PowerDNS 的特点

PowerDNS 和 BIND 一样，都是 DNS 服务软件。

PowerDNS 比较大的特色在于它把权威解析和递归解析的能力拆解开来分为两个服务，负责权威解析的是 pdns authoritative server ，负责递归解析的是 pdns recursor server 。我认为这种拆解是有利的，它将不同的解析流量区分，排查问题时会更方便，对后续的性能瓶颈分析也是有利的。

PowerDNS 支持各种各样的后端，可以是文件系统，也可以是通用关系型数据库，虽然 BIND 可以通过 DLZ 来实现关系型数据库作为记录的存储后端，但从支持的后端多样性来看，明显是 PowerDNS 更丰富。

PowerDNS 内置 Web Server 实现了 API 支持和监控系统，可以进行实时数据监控和动态更新记录，而 BIND 虽然也有相关的统计信息输出，但是这部分功能明显逊色于 PowerDNS 。

PowerDNS 的搭建和配置都相对简单，这里只记录一些调优过程的思考。

## Authoritative Server 的性能调优

我们需要先了解 Authoritative Server 中重要的缓存种类：

- Packet Cache ：数据包缓存，可以无需做任何额外处理，直接响应查询请求的数据缓存。
- Query Cache ：执行后端查询后，后端查询到的数据库记录缓存。
- Negative Cache ：在 Query Cache 中，请求信息无法在后端查询到记录的数据缓存。

其实可以直接地认为是两种缓存， Packet Cache 是对请求回答数据的缓存， Query Cache 和 Negative Cache 都是对数据库记录的缓存，通常我们希望直接返回 Packet Cache ，这会是最快最节省资源的响应办法。如果确实无法直接命中，则应该优先在 Query Cache 部分寻找命中，可以节省对数据库的查询行为。在这一部分， Negative Cache 和常规 Query Cache 的命中都是同样的，只是这个请求是否能得到回答数据的区别而已。

关于缓存部分，性能优化的调节点是在于缓存时间和缓存条数，我们可以把默认的 Packet Cache 的缓存时间略微提高一些，它在配置文件中以 cache-ttl 出现，默认时间为 20s ，这个值可以调高至 60s 。

Query Cache 的缓存时间在配置文件中以 query-cache-ttl 出现，默认时间为 20s ，这个值也可以和 cache-ttl 一样调高到 60s 。

Negative Cache 的缓存时间在配置文件中以 negquery-cache-ttl 出现，默认时间为 60s ，由于它和 Query Cache 类似，可以保持和 query-cache-ttl 一致的 60s 。

要注意调节缓存时间能起到性能优化的前提是使用关系型数据库作为后端，如果是文件系统或者是基于内存的后端存储，这些缓存时间需要额外考量，甚至可以直接禁用缓存，因为它们的响应速度足够快，缓存反而会成为性能的拖累。

另一个性能优化的调节点在于对工作线程的调节，在 Authoritative Server 中有 receiver thread 和 distributor thread 的概念。

receiver thread 是用于接收请求的线程，它的线程数可以自由调节，但要达到优化性能，这个数量应该适中，最好是和 CPU 数量成倍数关系。它在配置中以 receiver-threads 出现，默认值是 1 。

distributor thread 是 receiver thread 接收请求后，用于处理这些请求的线程，主要担任查询工作。它在配置中以 distributor-threads 出现，默认值是 3 。

需要注意的是这里 distributor-threads 是每个 receiver thread 所关联的线程数量，也就是说一个 receiver thread 可以对应一个或多个 distributor thread ，这个值应该由使用的后端类型决定，如果只有单个关系型数据库作为后端，那么 distributor-threads 为 1 应该是最优的做法，但如果使用了多个后端数据库，设置较大的 distributor-threads 可以得到更好的性能。

然后是配置中的 reuseport 这个特性开关，它是通过启用内核的 SO_REUSEPORT 选项来使得多个套接字可以在同个端口监听，如果内核版本过低不支持 SO_REUSEPORT ，那么不管这个选项如何设置，它都是默认关闭的。

设置多个 receiver thread 和开启 reuseport 的两者组合应该是最佳性能的工作方式，这样内核会将请求均衡到各个 receiver thread 中，获取比较好的性能表现。

最终的优化配置大概如下：

```bash
# 假设是 4 核机器，单个 MySQL 作为 backend
$ cat pdns.conf
cache-ttl=60
query-cache-ttl=60
negquery-cache-ttl=60
distributor-threads=1
receiver-threads=4
reuseport=yes
```

## Recursor Server 的性能调优

Recursor Server 同样有着多个缓存种类：

- Nameserver Speeds Cache ：对所有远端权威服务器的平均延迟时间的缓存。
- Negative Cache ：对无响应数据请求的缓存。
- Recursor Cache ：对递归过程一些公共记录信息的缓存。
- Packet Cache ：数据包缓存，可以无需做任何额外处理，直接响应查询请求的数据缓存。

在递归服务器中，各类缓存的 TTL 已经被默认设置为较高值，所以这部分并没有对它们做额外调节，更多的优化细节在于工作线程这一方面。

Recursor Server 的 threads 和 Authoritative Server 的 receiver threads 类似，是处理具体请求的线程，但它不负责具体的后端查询工作。

但在 Recursor Server 中还是有 distributor thread 的概念，它负责将请求分发到 thread 中，按照官方的说法，使用 distributor thread 可以提高缓存的命中率。但以实际测试情况来看，在原有 Packet Cache 命中率就很高的情况下，开启 distributor thread 会导致实际工作的 thread 负载不均衡，而缓存命中率只是略有提高。

所以实际使用中，较好的做法是禁用 distributor thread 和开启 reuseport 特性，由内核去把请求分配到 thread 中，并且使用 cpu-map 来把 thread 和具体的 CPU 绑定，有助于缓存的就近访问，提高响应的速度。

最终的优化配置大概如下：

```bash
# 假设是 4 核机器，单个 MySQL 作为 backend
$ cat pdns.conf
threads=4
pdns-distributes-queries=no
reuseport=yes
cpu-map=0=0 1=1 2=2 3=3
```

## Recursor Server 的 lua 扩展

powerdns 可以通过 lua 扩展脚本在查询的基础上实现更复杂的功能。

```bash
# 添加 lua 脚本扩展
$ cat pdns.conf
lua-dns-script /path/to/lua/script

# 动态重载 lua 脚本
rec_control reload-lua-script
```

powerdns 提供了多个查询钩子，可以在对应的查询阶段进行请求拦截并重写对应的回答动作，有以下几个钩子：

- ipfilter ：在查询数据包开始解析之前。
- gettag ：在查询数据包缓存之前。
- prerpz ：在应用响应策略之前。
- preresolve ：在查询逻辑工作开始之前。
- nodata, nxdomain ：在返回无数据结果和无域名结果之后。
- postresolve ：在查询逻辑工作结束之后。
- preoutquery ：在向权威服务器查询之前。
- policyEventFilter ：在响应策略命中之后。

由于存在着多阶段的钩子函数，所以实现扩展功能只需要重写对应的函数即可。

以下是一些参考用例，更多详细用法可以参考官方文档。

```lua
-- 以无响应域名结果的钩子为例，来自官方实例
nxdomainsuffix = newDN("com")
function nxdomain(dq)
    pdnslog("nxdomain called for: "..dq.qname:toString())
    if dq.qname:isPartOf(nxdomainsuffix) then
        dq.rcode = 0  -- 修改为正常应答
        dq:addAnswer(pdns.CNAME, "www.powerdns.org")
        dq:addAnswer(pdns.A, "1.2.3.4", 60, "www.powerdns.org")
        return true  -- return true 说明这个钩子函数生效，如果 return false 则这个钩子函数不生效
    end
    return false
end

-- 以查询逻辑工作开始之前的钩子为例，由官方实例改写
blockset = newDS()
blockset:add{"powerdns.org", "powerdns.com"}  -- 以列表形式添加多个域名
dropset = newDS()
dropset:add("pdns.org")  -- 添加单个域名
function preresolve(dq)
    -- 重写响应结果
    if blockset:check(dq.qname) then
        dq.variable = true      -- disable packet cache in any case
        if dq.qtype == pdns.A then
            dq:addAnswer(pdns.A, "1.2.3.4")
            return true
        end
    end
    -- 黑名单机制
    if dropset:check(dq.qname) then
        dq.appliedPolicy.policyKind = pdns.policykinds.Drop  -- 改写响应策略
        return false -- recursor still needs to handle the policy, 后续由定义策略处理
    end

    return false
end

-- 比较复杂的重写行为，可以在自建权威服务器的基础上，配合递归转发配置实现 dns 的内网劫持
-- 基本原理：查询请求由递归转发到自建域中，如果自建域不存在域名，再次转发到公网进行查询
rewriteset = newDS()
rewriteset:add("powerdns.org")
function nxdomain(dq)
    if rewriteset:check(dq.qname) then
        local dh = dq:getDH()
        -- 独立的处理函数 udpQueryResponse 允许发起新的 udp 查询，具体用法可以参考官方文档
        dq.followupFunction = "udpQueryResponse"
        dq.udpCallback = "gotdomaindetails"  -- 处理函数的回调函数
        dq.udpQueryDest = newCA("114.114.114.114:53")
        -- build_udp_package 需要自行实现，可以参考 lua-resty-dns ，工作内容为构建 dns 查询请求包
        dq.udpQuery = build_udp_package(dq.qname:toString(), dh:getID(), dh:getRD())
        return true
    end
    return false
end

function gotdomaindetails(dq)
    local dh = dq:getDH()
    -- 回调函数需要清除上一步的 dq 对象的赋值
    dq.followupFunction = ""
    dq.udpCallback = ""
    -- parse_response 需要自行实现，可以参考 lua-resty-dns ，工作内容为解析 dns 响应包
    local answers = parse_response(dq.udpAnswer, dh:getID())
    if not answers then
        return false
    end
    if answers.errcode then
        dq.rcode = answers.errcode
        return true
    end
    if answers then
        for _, ans in ipairs(answers) do
            dq:addRecord(ans.type, ans.address or ans.cname, ans.class, ans.ttl, ans.name)
        end
        -- 修改为正常应答
        dq.rcode = 0
        return true
    end
end

-- 自定义 metrics
new_metrics = getMetric("metrics")  -- 新增 metrics
metrics:inc()  -- 增加 metrics 计数，更多 metrics 方法可以参考官方文档
```

以下是附带了 `build_udp_package` 和 `parse_response` 函数实现的文件参考：

{{<details "完整的 `lua` 文件参考">}}

```lua
local sub = string.sub
local char = string.char
local rshift = bit.rshift
local lshift = bit.lshift
local byte = string.byte
local band = bit.band
local gsub = string.gsub
local find = string.find
local match = string.match
local insert = table.insert
local concat = table.concat


local DOT_CHAR = byte(".")
local ZERO_CHAR = byte("0")
local COLON_CHAR = byte(":")

local IP6_ARPA = "ip6.arpa"

local TYPE_A      = 1
local TYPE_NS     = 2
local TYPE_CNAME  = 5
local TYPE_SOA    = 6
local TYPE_PTR    = 12
local TYPE_MX     = 15
local TYPE_TXT    = 16
local TYPE_AAAA   = 28
local TYPE_SRV    = 33
local TYPE_SPF    = 99

local CLASS_IN    = 1

local SECTION_AN  = 1
local SECTION_NS  = 2
local SECTION_AR  = 3

local resolver_errstrs = {
    "format error",     -- 1
    "server failure",   -- 2
    "name error",       -- 3
    "not implemented",  -- 4
    "refused",          -- 5
}

local function _encode_name(s)
    return char(#s) .. s
end

local function build_udp_package(qname, id, no_recurse, opts)
    local qtype

    if opts then
        qtype = opts.qtype
    end

    if not qtype then
        qtype = TYPE_A  -- A record
    end

    local ident_hi = char(rshift(id, 8))
    local ident_lo = char(band(id, 0xff))

    local flags
    if no_recurse then
        -- print("found no recurse")
        flags = "\0\0"
    else
        flags = "\1\0"
    end

    local nqs = "\0\1"
    local nan = "\0\0"
    local nns = "\0\0"
    local nar = "\0\0"
    local typ = char(rshift(qtype, 8), band(qtype, 0xff))
    local class = "\0\1"    -- the Internet class

    local name = gsub(qname, "([^.]+)%.?", _encode_name) .. '\0'

    if byte(qname, 1) == DOT_CHAR then
        return nil, "bad name"
    end

    return  ident_hi .. ident_lo .. flags .. nqs .. nan .. nns .. nar .. name .. typ .. class

end

local function _decode_name(buf, pos)
    local labels = {}
    local nptrs = 0
    local p = pos
    while nptrs < 128 do
        local fst = byte(buf, p)

        if not fst then
            return nil, 'truncated';
        end

        -- print("fst at ", p, ": ", fst)

        if fst == 0 then
            if nptrs == 0 then
                pos = pos + 1
            end
            break
        end

        if band(fst, 0xc0) ~= 0 then
            -- being a pointer
            if nptrs == 0 then
                pos = pos + 2
            end

            nptrs = nptrs + 1

            local snd = byte(buf, p + 1)
            if not snd then
                return nil, 'truncated'
            end

            p = lshift(band(fst, 0x3f), 8) + snd + 1

            -- print("resolving ptr ", p, ": ", byte(buf, p))

        else
            -- being a label
            local label = sub(buf, p + 1, p + fst)
            insert(labels, label)

            -- print("resolved label ", label)

            p = p + fst + 1

            if nptrs == 0 then
                pos = p
            end
        end
    end

    return concat(labels, "."), pos
end

local function parse_section(answers, section, buf, start_pos, size,
                             should_skip)
    local pos = start_pos

    for _ = 1, size do
        -- print(format("ans %d: qtype:%d qclass:%d", i, qtype, qclass))
        local ans = {}

        if not should_skip then
            insert(answers, ans)
        end

        ans.section = section

        local name
        name, pos = _decode_name(buf, pos)
        if not name then
            return nil, pos
        end

        ans.name = name

        -- print("name: ", name)

        local type_hi = byte(buf, pos)
        local type_lo = byte(buf, pos + 1)
        local typ = lshift(type_hi, 8) + type_lo

        ans.type = typ

        -- print("type: ", typ)

        local class_hi = byte(buf, pos + 2)
        local class_lo = byte(buf, pos + 3)
        local class = lshift(class_hi, 8) + class_lo

        ans.class = class

        -- print("class: ", class)

        local byte_1, byte_2, byte_3, byte_4 = byte(buf, pos + 4, pos + 7)

        local ttl = lshift(byte_1, 24) + lshift(byte_2, 16)
                    + lshift(byte_3, 8) + byte_4

        -- print("ttl: ", ttl)

        ans.ttl = ttl

        local len_hi = byte(buf, pos + 8)
        local len_lo = byte(buf, pos + 9)
        local len = lshift(len_hi, 8) + len_lo

        -- print("record len: ", len)

        pos = pos + 10

        if typ == TYPE_A then

            if len ~= 4 then
                return nil, "bad A record value length: " .. len
            end

            local addr_bytes = { byte(buf, pos, pos + 3) }
            local addr = concat(addr_bytes, ".")
            -- print("ipv4 address: ", addr)

            ans.address = addr

            pos = pos + 4

        elseif typ == TYPE_CNAME then

            local cname, p = _decode_name(buf, pos)
            if not cname then
                return nil, pos
            end

            if p - pos ~= len then
                return nil, format("bad cname record length: %d ~= %d",
                                   p - pos, len)
            end

            pos = p

            -- print("cname: ", cname)

            ans.cname = cname

        elseif typ == TYPE_AAAA then

            if len ~= 16 then
                return nil, "bad AAAA record value length: " .. len
            end

            local addr_bytes = { byte(buf, pos, pos + 15) }
            local flds = {}
            for i = 1, 16, 2 do
                local a = addr_bytes[i]
                local b = addr_bytes[i + 1]
                if a == 0 then
                    insert(flds, format("%x", b))

                else
                    insert(flds, format("%x%02x", a, b))
                end
            end

            -- we do not compress the IPv6 addresses by default
            --  due to performance considerations

            ans.address = concat(flds, ":")

            pos = pos + 16

        elseif typ == TYPE_MX then

            -- print("len = ", len)

            if len < 3 then
                return nil, "bad MX record value length: " .. len
            end

            local pref_hi = byte(buf, pos)
            local pref_lo = byte(buf, pos + 1)

            ans.preference = lshift(pref_hi, 8) + pref_lo

            local host, p = _decode_name(buf, pos + 2)
            if not host then
                return nil, pos
            end

            if p - pos ~= len then
                return nil, format("bad cname record length: %d ~= %d",
                                   p - pos, len)
            end

            ans.exchange = host

            pos = p

        elseif typ == TYPE_SRV then
            if len < 7 then
                return nil, "bad SRV record value length: " .. len
            end

            local prio_hi = byte(buf, pos)
            local prio_lo = byte(buf, pos + 1)
            ans.priority = lshift(prio_hi, 8) + prio_lo

            local weight_hi = byte(buf, pos + 2)
            local weight_lo = byte(buf, pos + 3)
            ans.weight = lshift(weight_hi, 8) + weight_lo

            local port_hi = byte(buf, pos + 4)
            local port_lo = byte(buf, pos + 5)
            ans.port = lshift(port_hi, 8) + port_lo

            local name, p = _decode_name(buf, pos + 6)
            if not name then
                return nil, pos
            end

            if p - pos ~= len then
                return nil, format("bad srv record length: %d ~= %d",
                                   p - pos, len)
            end

            ans.target = name

            pos = p

        elseif typ == TYPE_NS then

            local name, p = _decode_name(buf, pos)
            if not name then
                return nil, pos
            end

            if p - pos ~= len then
                return nil, format("bad cname record length: %d ~= %d",
                                   p - pos, len)
            end

            pos = p

            -- print("name: ", name)

            ans.nsdname = name

        elseif typ == TYPE_TXT or typ == TYPE_SPF then

            local key = (typ == TYPE_TXT) and "txt" or "spf"

            local slen = byte(buf, pos)
            if slen + 1 > len then
                -- truncate the over-run TXT record data
                slen = len
            end

            -- print("slen: ", len)

            local val = sub(buf, pos + 1, pos + slen)
            local last = pos + len
            pos = pos + slen + 1

            if pos < last then
                -- more strings to be processed
                -- this code path is usually cold, so we do not
                -- merge the following loop on this code path
                -- with the processing logic above.

                val = {val}
                local idx = 2
                repeat
                    local slen = byte(buf, pos)
                    if pos + slen + 1 > last then
                        -- truncate the over-run TXT record data
                        slen = last - pos - 1
                    end

                    val[idx] = sub(buf, pos + 1, pos + slen)
                    idx = idx + 1
                    pos = pos + slen + 1

                until pos >= last
            end

            ans[key] = val

        elseif typ == TYPE_PTR then

            local name, p = _decode_name(buf, pos)
            if not name then
                return nil, pos
            end

            if p - pos ~= len then
                return nil, format("bad cname record length: %d ~= %d",
                                   p - pos, len)
            end

            pos = p

            -- print("name: ", name)

            ans.ptrdname = name

        elseif typ == TYPE_SOA then
            local name, p = _decode_name(buf, pos)
            if not name then
                return nil, pos
            end
            ans.mname = name

            pos = p
            name, p = _decode_name(buf, pos)
            if not name then
                return nil, pos
            end
            ans.rname = name

            for _, field in ipairs(soa_int32_fields) do
                local byte_1, byte_2, byte_3, byte_4 = byte(buf, p, p + 3)
                ans[field] = lshift(byte_1, 24) + lshift(byte_2, 16)
                            + lshift(byte_3, 8) + byte_4
                p = p + 4
            end

            pos = p

        else
            -- for unknown types, just forward the raw value

            ans.rdata = sub(buf, pos, pos + len - 1)
            pos = pos + len
        end
    end

    return pos
end


local function parse_response(buf, id, opts)
    local n = #buf
    if n < 12 then
        return nil, 'truncated';
    end

    -- header layout: ident flags nqs nan nns nar

    local ident_hi = byte(buf, 1)
    local ident_lo = byte(buf, 2)
    local ans_id = lshift(ident_hi, 8) + ident_lo

    -- print("id: ", id, ", ans id: ", ans_id)

    if ans_id ~= id then
        -- identifier mismatch and throw it away
        return nil, "id mismatch"
    end

    local flags_hi = byte(buf, 3)
    local flags_lo = byte(buf, 4)
    local flags = lshift(flags_hi, 8) + flags_lo

    -- print(format("flags: 0x%x", flags))

    if band(flags, 0x8000) == 0 then
        return nil, format("bad QR flag in the DNS response")
    end

    if band(flags, 0x200) ~= 0 then
        return nil, "truncated"
    end

    local code = band(flags, 0xf)

    -- print(format("code: %d", code))

    local nqs_hi = byte(buf, 5)
    local nqs_lo = byte(buf, 6)
    local nqs = lshift(nqs_hi, 8) + nqs_lo

    -- print("nqs: ", nqs)

    if nqs ~= 1 then
        return nil, format("bad number of questions in DNS response: %d", nqs)
    end

    local nan_hi = byte(buf, 7)
    local nan_lo = byte(buf, 8)
    local nan = lshift(nan_hi, 8) + nan_lo

    -- print("nan: ", nan)

    local nns_hi = byte(buf, 9)
    local nns_lo = byte(buf, 10)
    local nns = lshift(nns_hi, 8) + nns_lo

    local nar_hi = byte(buf, 11)
    local nar_lo = byte(buf, 12)
    local nar = lshift(nar_hi, 8) + nar_lo

    -- skip the question part

    local ans_qname, pos = _decode_name(buf, 13)
    if not ans_qname then
        return nil, pos
    end

    -- print("qname in reply: ", ans_qname)

    -- print("question: ", sub(buf, 13, pos))

    if pos + 3 + nan * 12 > n then
        -- print(format("%d > %d", pos + 3 + nan * 12, n))
        return nil, 'truncated';
    end

    -- question section layout: qname qtype(2) qclass(2)

    --[[
    local type_hi = byte(buf, pos)
    local type_lo = byte(buf, pos + 1)
    local ans_type = lshift(type_hi, 8) + type_lo
    ]]

    -- print("ans qtype: ", ans_type)

    local class_hi = byte(buf, pos + 2)
    local class_lo = byte(buf, pos + 3)
    local qclass = lshift(class_hi, 8) + class_lo

    -- print("ans qclass: ", qclass)

    if qclass ~= 1 then
        return nil, format("unknown query class %d in DNS response", qclass)
    end

    pos = pos + 4

    local answers = {}

    if code ~= 0 then
        answers.errcode = code
        answers.errstr = resolver_errstrs[code] or "unknown"
    end

    local authority_section, additional_section

    if opts then
        authority_section = opts.authority_section
        additional_section = opts.additional_section
        if opts.qtype == TYPE_SOA then
            authority_section = true
        end
    end

    local err

    pos, err = parse_section(answers, SECTION_AN, buf, pos, nan)

    if not pos then
        return nil, err
    end

    if not authority_section and not additional_section then
        return answers
    end

    pos, err = parse_section(answers, SECTION_NS, buf, pos, nns,
                             not authority_section)

    if not pos then
        return nil, err
    end

    if not additional_section then
        return answers
    end

    pos, err = parse_section(answers, SECTION_AR, buf, pos, nar)

    if not pos then
        return nil, err
    end

    return answers
end


rewriteset = newDS()
rewriteset:add("powerdns.org")
function nxdomain(dq)
    if rewriteset:check(dq.qname) then
        local dh = dq:getDH()
        dq.followupFunction = "udpQueryResponse"
        dq.udpCallback = "gotdomaindetails"
        dq.udpQueryDest = newCA("114.114.114.114:53")
        dq.udpQuery = build_udp_package(dq.qname:toString(), dh:getID(), dh:getRD())
        return true
    end
    return false
end


function gotdomaindetails(dq)
    local dh = dq:getDH()
    dq.followupFunction = ""
    dq.udpCallback = ""
    local answers = parse_response(dq.udpAnswer, dh:getID())
    if not answers then
        return false
    end
    if answers.errcode then
        dq.rcode = answers.errcode
        return true
    end
    if answers then
        for _, ans in ipairs(answers) do
            dq:addRecord(ans.type, ans.address or ans.cname, ans.class, ans.ttl, ans.name)
        end
        dq.rcode = 0
        return true
    end
end
```

{{</details>}}

## 参考文档

- [powerdns authoritative performance](https://doc.powerdns.com/authoritative/performance.html)
- [powerdns recursor performance](https://docs.powerdns.com/recursor/performance.html)
- [powerdns recursor lua scripting](https://docs.powerdns.com/recursor/lua-scripting/index.html)
