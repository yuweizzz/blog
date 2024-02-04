---
date: 2022-07-16 15:00:00
title: Nginx 实用技巧
tags:
  - "Nginx"
  - "OCSP"
  - "HSTS"
  - "CORS"
draft: false
---

这篇笔记用来 Nginx 在实际应用场景的一些使用技巧。

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

## Nginx 获取内置变量

Nginx 经常需要使用内置变量和请求相关的变量来构成配置文件。

Nginx 可以轻松获取 HTTP 请求头，使用 `$http_xxxx` 来获取，比如 `Host` 可以使用 `$http_host` 来获取， `User-Agent` 可以使用 `$http_user_agent` 来获取。

还有一些比较重要的内置变量：

* `$remote_addr` ：请求的来源 IP 地址，这里是指直接和当前 Nginx 交互的对端地址，所以它有可能只是源请求的一个代理地址。
* `$server_name` ： Nginx 虚拟主机中的 `server_name` 。
* `$request` ：原始的请求 url 。
* `$scheme` ：请求的协议，通常是 http 和 https 。
* `$request_method` ：请求的方法，也就是 HTTP 的请求方法 GET ， POST 等。
* `$request_uri` ：原始的请求路径和完整参数，是不可修改的。
* `$uri` ：来自 `$request_uri` 中的路径部分，在经过重写行为后可能会与原来不同。
* `$args` ：来自 `$request_uri` 中的参数部分。
> `$request_uri` 可以拆解为 `$uri$is_args$args` ，其中 `$is_args` 就是 uri 请求中的 `?` ，在实际请求没有参数的时候， `$is_args` 和 `$args` 都为空。
* `$host` ：以下出自官方文档的解释，可以看到取值优先级来源于以下三个值：请求 url 中的主机部分；请求头中 `Host` 的部分，同 `$http_host` ； Nginx 虚拟主机中匹配命中的 `server_name` 。
> $host：in this order of precedence: host name from the request line, or host name from the "Host" request header field, or the server name matching a request

和请求时间相关的变量，更多用于排查问题和性能分析：

* `$request_time` ：本地响应请求所使用的时间，从客户端读取第一个字节开始计时，单位精确到毫秒，这个值计算的时间已经包括了 `$upstream_response_time` 。
* `$upstream_response_time` ：使用代理的情况下，上游响应代理请求所使用的时间，单位精确到毫秒，这个值可以直观反应具体程序对请求的处理所需时间。
* `$time_local` ：输出当前时间值，通常用于日志记录。

## 使用 map 映射变量

`map` 是 Nginx 模块 ngx_http_map_module 提供的关键字，可以基于某个变量的现有值制定一定规则，从而设置新的变量。

``` bash
# 来自 Nginx 官方文档的示例
map $http_user_agent $mobile {
    default       0;
    "~Opera Mini" 1;
}

# 获取 HTTP 请求头 User-Agent 的值，如果使用正则匹配到 Opera Mini 则设置变量 mobile 为 1 ，其他情况默认为 0 
```

## 设置 json 格式日志

Nginx 原生的日志格式是列出了关键信息的单行文本，我们可以创建模拟 json 格式的日志写入规则。

``` bash
# 模拟 json 的日志格式
http {
    log_format main escape=json '{'
        '"datetime": "$time_local",'
        '"remote_addr": "$remote_addr",'
        '"http_host": "$http_host",'
        '"upstream_addr": "$upstream_addr",'
        '"request_method": "$request_method",'
        '"http_referer": "$http_referer",'
        '"status": "$status",'
        '"server_name": "$server_name",'
        '"request_uri": "$request_uri",'
        '"http_user_agent": "$http_user_agent",'
        '"http_x_forwarded_for": "$http_x_forwarded_for",'
        '"body_bytes_sent": "$body_bytes_sent",'
        '"upstream_response_time": "$upstream_response_time",'
        '"request_time": "$request_time"'
    '}';

    server {
        access_log logs/access.log main;
    }
}
```

## Nginx 日志轮转

生产环境的 Nginx 日志应该定时切割和归档，否则可能会将存储空间占满，一般通过 `logrotate` 或者 `crontab` 完成这项工作。

``` bash
# Nginx 切割日志实例
$ cat /etc/logrotate.d/nginx
/usr/local/openresty/nginx/logs/*.log {
    daily                    # 每日执行一次日志轮转
    rotate 7                 # 历史日志保留 7 份
    missingok                # 忽略文件不存在的报错
    dateext
    dateformat -%Y%m%d%s     # 以固定的日期格式重命名历史日志
    compress
    delaycompress            # 启用日志压缩，并在下次轮转时，压缩上一份历史日志
    notifempty               # 空文件则不执行轮转
    sharedscripts            # 使用 *.log 说明目录可能存在多种日志，这个关键字用来声明所有日志都需要执行 postrotate 脚本 
    postrotate               # 轮转工作完成后需要执行脚本
        [ -e /usr/local/openresty/nginx/logs/nginx.pid ] && kill -USR1 `cat /usr/local/openresty/nginx/logs/nginx.pid`
    endscript
}
```

`logrotate` 的精细度最快只能每天执行一次，如果我们需要频繁地切割日志，可以将 `crontab` 和 `logrotate -v /etc/logrotate.d/nginx` 配合使用。

`postrotate` 的关键点在于 `kill -USR1 nginx.pid` 这个命令，它可以使 Nginx 重新打开日志文件。

## 代理 WebSocket

WebSocket 是基于 HTTP 协议的实时双向通讯协议，它属于应用层的协议。

在浏览器中， WebSocket 会以 `ws://` 的形式出现，基于 ssl 加密则会以 `wss://` 的形式出现，在 Nginx 中，所有的 WebSocket 请求只是带有特定请求头的普通 HTTP 请求。

WebSocket 的握手请求带有重要的两个请求头 `Upgrade: websocket` 和 `Connection: Upgrade` ，握手成功将会返回 `101 Switching Protocols` ，后续双方就可以使用 WebSocket 的方法互相通信。

通常 WebSocket 代理是这样做的：

``` bash
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    upstream webscoket {
        server 127.0.0.1:9000;
    }

    server {
        location /webscoket {
            proxy_pass http://webscoket;
            proxy_http_version 1.1;
            proxy_connect_timeout 60s;
            proxy_read_timeout 60s;
            proxy_send_timeout 60s;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
```

## 支持跨域访问 CORS

跨域访问是浏览器对 Web 资源访问的安全限制，通过特定的请求头响应来确认是否允许访问非同源的资源。

``` bash
# 简单的跨域请求
GET /resource HTTP/1.1         # 请求资源路径： /resource
Origin: http://web.domain.com  # 所在的请求页面： web.domain.com
Host: api.domain.com           # 完整请求： api.domain.com/resource
```

所有的跨域请求都需要带上 `Origin` 作为来源标识，此外可能还额外带有一些额外的请求头，通常会以 `OPTIONS` 方法进行预检测服务端是否响应跨域请求。

``` bash
# Preflight Request from MDN Docs

# Request
OPTIONS /doc HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:71.0) Gecko/20100101 Firefox/71.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Connection: keep-alive
Origin: https://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type

# Response
HTTP/1.1 204 No Content
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Max-Age: 86400
Vary: Accept-Encoding, Origin
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
```

比较重要的响应 Header 有：

* `Access-Control-Allow-Origin` ：允许跨域的来源，可以使用 `Access-Control-Allow-Origin: *` 代表公开资源。
* `Access-Control-Allow-Methods` ：允许跨域请求所使用的 HTTP 方法。
* `Access-Control-Allow-Headers` ：允许跨域请求所携带的 Header 。
* `Access-Control-Max-Age` ：预检跨域请求的缓存时间。
* `Access-Control-Allow-Credentials` ：允许跨域请求携带 cookies Header 信息，通过用于实现不同子域的 cookies 共享。当这个 Header 设置为 `true` 时， `Access-Control-Allow-Origin` 不允许设置为 `*` 。
* `Access-Control-Expose-Headers` ：允许浏览器额外获取的 Header ， `XMLHttpRequest` 对象的方法 `getResponseHeader()` 就可以获取到 `Access-Control-Expose-Headers` 中设置的 Header 。

除了在业务端实现跨域请求 Header 响应，还可以在 Nginx 中设置。

``` bash
location / {
    add_header 'Access-Control-Allow-Origin' $http_origin;
    add_header 'Access-Control-Max-Age' '86400';
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin, User-Agent, Cache-Control, X-Requested-With';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

## 通过 SNI 分发不同证书

Nginx 可以通过 SNI 获取到请求的域名信息，我们可以根据这个信息分发对应的证书，适合用于同站点多域名的情况。

``` bash
# 需要确认 nginx 版本是否支持 SNI
$ nginx -V
......
TLS SNI support enabled
......

$ cat nginx.conf
......
# 变量 ssl_server_name 可以获取 ssl 握手时请求的域名，只需要将实际签发的证书按照对应域名命名即可
server {
    listen 443 ssl;
    server_name $ssl_server_name;
    ssl_certificate $ssl_server_name.crt;
    ssl_certificate_key $ssl_server_name.pri;
    ......
}
......
```

## 使用 TLS 会话复用优化 https

TLS 握手成功后会在服务端创建 session 并返回 session ID 给到客户端，这样在首次 TLS 握手成功后，后续的通信就可以直接通过 session 复用省去密钥协商过程，提高响应效率。

在 nginx 中通过使用 ssl_session_cache 关键字来启用 TLS session 。

``` bash
$ cat nginx.conf
......
server {
    listen 443 ssl;
    server_name $ssl_server_name;
    ssl_certificate $ssl_server_name.crt;
    ssl_certificate_key $ssl_server_name.pri;
    
    # 使用 shared 会由所有工作进程共享 session ， 1m 代表共享空间的内存大小为 1MB
    ssl_session_cache shared:SSL:1m;
    # ssl_session_tickets 是 tls 复用的另一种机制
    # 它不同于 session ID ，这种机制由客户端保存来 session ticket ，在会话复用时由服务端认证 ticket 来决定能否复用会话
    ssl_session_tickets on;
    # ssl_session_ticket_key 可以自行指定或者保持注释由 nginx 使用默认随机数
    # ssl_session_ticket_key random.key;
    # ssl_session_timeout 代表会话的有效时间， 5m 代表有效时间为 5 分钟
    ssl_session_timeout 5m;
    ......
}
......
```

## 开启 HSTS

为了将访问 HTTP 的 80 端口流量导向 HTTPS 的 443 端口，通过会额外配置一个虚拟主机用于重定向。

``` bash
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    # Enable HSTS 
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
}
```

在第一次访问 80 端口时，如果服务成功将你重定向到了 443 端口，并且添加了 `Strict-Transport-Security` 的 Header ，那么在浏览器在下次访问这个域名的服务时，会强制使用 HTTPS 进行访问，如果 Header 添加了 `includeSubDomains` 则会连同服务的子域名一起保护。

开启了 HSTS 后想降级使用 HTTP 是比较麻烦的，可能需要修改浏览器的 HSTS 相关设置，此外 `Strict-Transport-Security` 还可以添加 `preload` 指令，这样浏览器第一次访问时会直接强制 HTTPS 而不是经过一次重定向后再强制，要正常使用 `preload` 指令需要自行将域名注册到 Chromium 项目的 HSTS preload list 并修改服务端配置，成功注册到 HSTS preload list 后如果想降级使用 HTTP 可能会导致服务无法访问。

## 开启 OCSP Stapling

OCSP 全称 Online Certificate Status Protocol ，也就是在线证书状态协议，在请求端浏览器获取到服务端证书后，可以向证书签发机构调用接口检验证书的时效性。

OCSP Stapling 是基于 OCSP 的 TLS 协议扩展，和 OCSP 不同的是，服务端预先通过缓存 OCSP 结果，将这些信息一步到位返回给请求端浏览器，省去浏览器调用证书签发机构验证接口的时间。

``` bash
# 以 github.com 为测试站点，首先获取证书链
$ echo | openssl s_client -showcerts -connect www.github.com:443 2>/dev/null | sed -n '/BEGIN/,/END/p'
-----BEGIN CERTIFICATE-----
MIIFajCCBPCgAwIBAgIQBRiaVOvox+kD4KsNklVF3jAKBggqhkjOPQQDAzBWMQsw
CQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMTAwLgYDVQQDEydEaWdp
Q2VydCBUTFMgSHlicmlkIEVDQyBTSEEzODQgMjAyMCBDQTEwHhcNMjIwMzE1MDAw
MDAwWhcNMjMwMzE1MjM1OTU5WjBmMQswCQYDVQQGEwJVUzETMBEGA1UECBMKQ2Fs
aWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZyYW5jaXNjbzEVMBMGA1UEChMMR2l0SHVi
LCBJbmMuMRMwEQYDVQQDEwpnaXRodWIuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0D
AQcDQgAESrCTcYUh7GI/y3TARsjnANwnSjJLitVRgwgRI1JlxZ1kdZQQn5ltP3v7
KTtYuDdUeEu3PRx3fpDdu2cjMlyA0aOCA44wggOKMB8GA1UdIwQYMBaAFAq8CCkX
jKU5bXoOzjPHLrPt+8N6MB0GA1UdDgQWBBR4qnLGcWloFLVZsZ6LbitAh0I7HjAl
BgNVHREEHjAcggpnaXRodWIuY29tgg53d3cuZ2l0aHViLmNvbTAOBgNVHQ8BAf8E
BAMCB4AwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMIGbBgNVHR8EgZMw
gZAwRqBEoEKGQGh0dHA6Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFRMU0h5
YnJpZEVDQ1NIQTM4NDIwMjBDQTEtMS5jcmwwRqBEoEKGQGh0dHA6Ly9jcmw0LmRp
Z2ljZXJ0LmNvbS9EaWdpQ2VydFRMU0h5YnJpZEVDQ1NIQTM4NDIwMjBDQTEtMS5j
cmwwPgYDVR0gBDcwNTAzBgZngQwBAgIwKTAnBggrBgEFBQcCARYbaHR0cDovL3d3
dy5kaWdpY2VydC5jb20vQ1BTMIGFBggrBgEFBQcBAQR5MHcwJAYIKwYBBQUHMAGG
GGh0dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBPBggrBgEFBQcwAoZDaHR0cDovL2Nh
Y2VydHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0VExTSHlicmlkRUNDU0hBMzg0MjAy
MENBMS0xLmNydDAJBgNVHRMEAjAAMIIBfwYKKwYBBAHWeQIEAgSCAW8EggFrAWkA
dgCt9776fP8QyIudPZwePhhqtGcpXc+xDCTKhYY069yCigAAAX+Oi8SRAAAEAwBH
MEUCIAR9cNnvYkZeKs9JElpeXwztYB2yLhtc8bB0rY2ke98nAiEAjiML8HZ7aeVE
P/DkUltwIS4c73VVrG9JguoRrII7gWMAdwA1zxkbv7FsV78PrUxtQsu7ticgJlHq
P+Eq76gDwzvWTAAAAX+Oi8R7AAAEAwBIMEYCIQDNckqvBhup7GpANMf0WPueytL8
u/PBaIAObzNZeNMpOgIhAMjfEtE6AJ2fTjYCFh/BNVKk1mkTwBTavJlGmWomQyaB
AHYAs3N3B+GEUPhjhtYFqdwRCUp5LbFnDAuH3PADDnk2pZoAAAF/jovErAAABAMA
RzBFAiEA9Uj5Ed/XjQpj/MxQRQjzG0UFQLmgWlc73nnt3CJ7vskCICqHfBKlDz7R
EHdV5Vk8bLMBW1Q6S7Ga2SbFuoVXs6zFMAoGCCqGSM49BAMDA2gAMGUCMCiVhqft
7L/stBmv1XqSRNfE/jG/AqKIbmjGTocNbuQ7kt1Cs7kRg+b3b3C9Ipu5FQIxAM7c
tGKrYDGt0pH8iF6rzbp9Q4HQXMZXkNxg+brjWxnaOVGTDNwNH7048+s/hT9bUQ==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIEFzCCAv+gAwIBAgIQB/LzXIeod6967+lHmTUlvTANBgkqhkiG9w0BAQwFADBh
MQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3
d3cuZGlnaWNlcnQuY29tMSAwHgYDVQQDExdEaWdpQ2VydCBHbG9iYWwgUm9vdCBD
QTAeFw0yMTA0MTQwMDAwMDBaFw0zMTA0MTMyMzU5NTlaMFYxCzAJBgNVBAYTAlVT
MRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxMDAuBgNVBAMTJ0RpZ2lDZXJ0IFRMUyBI
eWJyaWQgRUNDIFNIQTM4NCAyMDIwIENBMTB2MBAGByqGSM49AgEGBSuBBAAiA2IA
BMEbxppbmNmkKaDp1AS12+umsmxVwP/tmMZJLwYnUcu/cMEFesOxnYeJuq20ExfJ
qLSDyLiQ0cx0NTY8g3KwtdD3ImnI8YDEe0CPz2iHJlw5ifFNkU3aiYvkA8ND5b8v
c6OCAYIwggF+MBIGA1UdEwEB/wQIMAYBAf8CAQAwHQYDVR0OBBYEFAq8CCkXjKU5
bXoOzjPHLrPt+8N6MB8GA1UdIwQYMBaAFAPeUDVW0Uy7ZvCj4hsbw5eyPdFVMA4G
A1UdDwEB/wQEAwIBhjAdBgNVHSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwdgYI
KwYBBQUHAQEEajBoMCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5kaWdpY2VydC5j
b20wQAYIKwYBBQUHMAKGNGh0dHA6Ly9jYWNlcnRzLmRpZ2ljZXJ0LmNvbS9EaWdp
Q2VydEdsb2JhbFJvb3RDQS5jcnQwQgYDVR0fBDswOTA3oDWgM4YxaHR0cDovL2Ny
bDMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0R2xvYmFsUm9vdENBLmNybDA9BgNVHSAE
NjA0MAsGCWCGSAGG/WwCATAHBgVngQwBATAIBgZngQwBAgEwCAYGZ4EMAQICMAgG
BmeBDAECAzANBgkqhkiG9w0BAQwFAAOCAQEAR1mBf9QbH7Bx9phdGLqYR5iwfnYr
6v8ai6wms0KNMeZK6BnQ79oU59cUkqGS8qcuLa/7Hfb7U7CKP/zYFgrpsC62pQsY
kDUmotr2qLcy/JUjS8ZFucTP5Hzu5sn4kL1y45nDHQsFfGqXbbKrAjbYwrwsAZI/
BKOLdRHHuSm8EdCGupK8JvllyDfNJvaGEwwEqonleLHBTnm8dqMLUeTF0J5q/hos
Vq4GNiejcxwIfZMy0MJEGdqN9A57HSgDKwmKdsp33Id6rHtSJlWncg+d0ohP/rEh
xRqhqjn1VtvChMQ1H3Dau0bwhr9kAMQ+959GG50jBbl9s08PqUU643QwmA==
-----END CERTIFICATE-----

# 可以看到一共有两张证书，第一张证书是 github.com 站点本身使用的证书，第二张证书是签发机构的证书
# 将第一张证书保存为 github.pem ，第二张证书单独保存为 issuer.pem
# 证书链如果有两张以上的证书，那么 issuer.pem 应该保持中间证书应该在前，根证书应该在后

# 查询 OCSP 接口
$ openssl x509 -noout -ocsp_uri -in github.pem
http://ocsp.digicert.com

# 进行 OCSP 校验
$ openssl ocsp -issuer issuer.pem -cert github.pem -text -no_nonce -url http://ocsp.digicert.com
# 获得到完整的 OCSP 响应，可以看到结果是验证通过的，并且带有有效时间信息和 CA 签名
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha1
          Issuer Name Hash: 2B1D1E98CCF37604D6C1C8BD15A224C804130038
          Issuer Key Hash: 0ABC0829178CA5396D7A0ECE33C72EB3EDFBC37A
          Serial Number: 05189A54EBE8C7E903E0AB0D925545DE
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: 0ABC0829178CA5396D7A0ECE33C72EB3EDFBC37A
    Produced At: Nov 10 13:30:57 2022 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: 2B1D1E98CCF37604D6C1C8BD15A224C804130038
      Issuer Key Hash: 0ABC0829178CA5396D7A0ECE33C72EB3EDFBC37A
      Serial Number: 05189A54EBE8C7E903E0AB0D925545DE
    Cert Status: good
    This Update: Nov 10 13:15:02 2022 GMT
    Next Update: Nov 17 12:30:02 2022 GMT

    Signature Algorithm: ecdsa-with-SHA384
         30:64:02:30:68:1e:bf:ce:94:74:e2:b8:70:10:7e:f0:66:0c:
         b1:d7:c2:1f:1d:4a:32:a0:fc:58:f9:6b:a7:8f:e7:73:b9:25:
         57:17:33:bf:fc:58:df:d6:e9:e2:6a:0f:5b:22:69:9a:02:30:
         28:88:61:cb:10:d8:71:27:80:08:38:cc:54:d5:c8:2a:46:e0:
         ee:4d:7b:97:3b:5b:a1:90:ab:34:26:10:30:0b:81:01:a5:b4:
         0c:f4:68:6d:eb:0b:ea:da:fa:63:4c:eb
Response verify OK
github.pem: good
	This Update: Nov 10 13:15:02 2022 GMT
	Next Update: Nov 17 12:30:02 2022 GMT
```

在 Nginx 中可以通过 ssl 模块中的相关指令来开启 OCSP 支持。

``` bash
# 开启 OCSP 支持
http {
    server {
        listen              443 ssl;
        keepalive_timeout   70;

        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         AES128-SHA:AES256-SHA:RC4-SHA:DES-CBC3-SHA:RC4-MD5;
        ssl_certificate     /usr/local/nginx/conf/cert.pem;
        ssl_certificate_key /usr/local/nginx/conf/cert.key;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 10m;

        ssl_stapling on;
        resolver 8.8.8.8;
    }
}
# 使用上述配置时 cert.pem 包含完整的证书链
# 以前面的 github.com 为例， cert.pem 需要是 showcerts 所展示的两张证书

http {
    server {
        listen              443 ssl;
        keepalive_timeout   70;

        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         AES128-SHA:AES256-SHA:RC4-SHA:DES-CBC3-SHA:RC4-MD5;
        ssl_certificate     /usr/local/nginx/conf/cert.pem;
        ssl_certificate_key /usr/local/nginx/conf/cert.key;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 10m;

        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /usr/local/nginx/conf/chain.pem;
        resolver 8.8.8.8;
    }
}
# 使用上述配置用来应对 OCSP 响应返回包括对应证书的情况
# 指定 ssl_trusted_certificate 会用于校验返回的证书， chain.pem 就是除了本身站点证书之外的证书链
# 配置异常可能会出现下面的报错：
OCSP_basic_verify() failed (SSL: error:27069076:OCSP routines:OCSP_basic_verify:signer certificate not found)

# 先行存储 OCSP 响应结果
$ openssl ocsp -issuer chain.pem -cert cert.pem -text -no_nonce -url `openssl x509 -noout -ocsp_uri -in cert.pem` -respout resp.der
# 使用 ssl_stapling_file 指令后无需动态调用 OCSP 接口，使用存储结果进行响应
http {
    server {
        listen              443 ssl;
        keepalive_timeout   70;

        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         AES128-SHA:AES256-SHA:RC4-SHA:DES-CBC3-SHA:RC4-MD5;
        ssl_certificate     /usr/local/nginx/conf/cert.pem;
        ssl_certificate_key /usr/local/nginx/conf/cert.key;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 10m;

        ssl_stapling on;
        ssl_stapling_file /usr/local/nginx/conf/resp.der;
    }
}

# 验证是否服务端开启 OCSP stapling
$ echo | openssl s_client -status -connect www.github.com:443 2>/dev/null | grep 'OCSP Response'
# 出现 OCSP Response Status: successful (0x0) 则说明服务端开启了 OCSP stapling
```

## Nginx 常用配置选项

### ngx_http_core_module 配置

* `client_header_buffer_size 1k;` ：这个配置用来定义请求的 Header 缓冲区大小，如果 Header 内容大于这个值，会使用 `large_client_header_buffers` 的配置。
* `large_client_header_buffers 4 8k;` ：这个配置用来限制请求 URL 和 Header 字段大小，虽然定义了多个缓冲区，但是 URL 和单个 Header 不能超过单个缓冲区的最大限制，而且总大小应该保持在多个缓冲区总和内，否则都会返回请求错误。

### ngx_http_proxy_module 超时时间配置

* `proxy_connect_timeout 60s;` ：这个配置用来定义请求与上游建立连接的超时时间。
* `proxy_read_timeout 60s;` ：这个配置用来定义读取连接的超时时间，是指相邻两次读操作之间的最长时间间隔，达到超时时间连接将会关闭。
* `proxy_send_timeout 60s;` ：这个配置用来定义写入连接的超时时间，和 `proxy_read_timeout` 的定义类似，只不过它是基于写操作的。

### ngx_http_proxy_module 缓冲设置

启用代理缓冲在一定程度上可以提高访问的效率。其中 `proxy_buffering` 是启用代理缓冲的开关，而 `proxy_buffer_size` 是基本的内存缓冲区大小，和 `proxy_buffering` 是否启用无关。

根据 nginx 的官方文档，不启用 `proxy_buffering` ， nginx 会同步性地获取上游响应并返回给客户端，每次获取的响应大小被限制在 `proxy_buffer_size` 中，通常情况下是 4k ，也就是一个内存页的大小，会根据运行平台而变。

如果使用 `proxy_buffering` ，那么如下关键字就可以被有效使用：

* `proxy_buffers 8 4k|8k;` ：定义缓冲区的数量和字节大小。
* `proxy_busy_buffers_size 8k|16k;` ：在上游响应未完全读取的情况下，当缓冲的内容超过这个定义的大小时，就开始向客户端返回数据。
* `proxy_temp_file_write_size 8k|16k;` ：内存缓冲区大小不足时，单次写入磁盘缓冲文件的字节大小。
* `proxy_max_temp_file_size 1024m;` ：内存缓冲区大小不足时，最大可用的磁盘缓冲文件的字节大小。

使用 `proxy_buffering` ，缓冲区将由 `proxy_buffer_size` 和 `proxy_buffers` 共同构成，并且 `proxy_busy_buffers_size` 默认是这两个值中单个缓冲区的两倍大小，在整体缓冲区容量不足的情况下，需要设置 `proxy_temp_file_write_size` 和 `proxy_max_temp_file_size` 的容量大小才能使用磁盘缓冲。

## acme 自动更新证书

以下是通过使用 acme.sh 的 webroot 模式申请免费证书的步骤。

``` bash
# 通过 80 端口提供认证服务接口
server {
    listen 80;
    server_name _;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/;
        try_files $uri =404;
    }
}
```

修改完 nginx 配置后，参考的执行命令如下：

``` bash
# 安装 acme.sh
$ curl https://get.acme.sh | sh
# 选择 CA 登陆账号
$ acme.sh --register-account --server letsencrypt -m your_email@email.com
# 通过 webroot 模式签发证书
$ acme.sh --issue -d your.domain.com -w /var/www/ --server letsencrypt
# 签发成功后将证书安装到对应的 nginx 目录下
$ acme.sh --install-cert \
  -d your.domain.com \
  --key-file /path/to/cert.key \
  --fullchain-file /path/to/cert.pem \
  --reloadcmd "systemctl reload nginx"
```
