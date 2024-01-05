---
date: 2021-05-30 15:00:00
title: Nginx 使用笔记
tags:
  - "Nginx"
draft: false
---

这篇笔记用来记录 Nginx 的编译安装和基本使用。

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

## 安装 Nginx

Nginx 是应用广泛的服务器软件，在 Nginx 的官方文档中可以看到， Nginx 可以应用在多种场景，包括基本 HTTP 服务器，代理服务器，邮件代理服务器。由于支持的特性非常多，所以安装的时候需要考虑的地方也相应增多了。

我们可以选择编译安装或者提供官网提供给对应发行版的软件包进行安装，使用软件包安装基本上只要解决软件源的问题就可以了，使用编译安装虽然比较麻烦，但是更适用于需要定制的软件特性的场景。

需要的源码包或者软件包的安装方式都可以在 Nginx 的[官网](http://nginx.org/)找到，在这里只介绍编译安装。

### Nginx 源码包结构

在下载源码包并进行解压后，可以看到几个关键的文件和目录：

* configure : 配置编译环境的脚本文件，执行这个脚本后会生成 Makefile 文件。 
* src : nginx 源代码目录。
* auto : configure 脚本需要调用 auto 中的各个脚本文件完成配置工作。

Nginx 源码包使用 autoconf 生成 configure 脚本，这种源码包的使用方式一般都是通过 configure 文件配置编译选项并生成 Makefile ，再通过 Makefile 进行编译和安装。

``` bash
# 由 autoconf 生成配置脚本的源码包编译时的一般步骤
$ ./configure
$ make 
$ make install
```

在 configure 执行时，我们可以选择是否编译某些功能特性，使用了 autoconf 的源码包都有这一特点。

在 Nginx 的编译安装时，我们可以选择编译某个模块来开启某个功能特性。其中， HTTP 服务模块是默认支持的，而邮件服务模块和传输层服务模块并不会默认支持，需要在编译时额外指定。

### 常用模块

Nginx 默认作为 HTTP 服务器工作，所以 HTTP 模块是 Nginx 的核心，但是这个模块可以强制禁用，方法是在 configure 执行时带上 `--without-http` 参数。

HTTP 服务模块包括以下几个常用模块，它们都是默认编译的：

* ngx_http_autoindex_module : 常用于下载服务器，启用这个模块可以自动建立下载目录页面。
* ngx_http_fastcgi_module/ngx_http_uwsgi_module/ngx_http_scgi_module : 对各类通用网关接口协议的支持模块，启用这些模块后可以对接其他程序的接口。
* ngx_http_gzip_module : 用于压缩响应结果，可以减小传输数据的体积。
* ngx_http_rewrite_module : 用于 url 重写和返回重定向。
* ngx_http_proxy_module : 用于设置代理。
* ngx_http_upstream_module : 用于设置负载均衡。

还有一个特殊的 HTTP 模块 `ngx_http_ssl_module` ，随着 https 的流行，它已经是不可或缺的模块，尤其是在生产环境中，但是它不是默认编译选项，在安装时需要额外指定。

更多的编译信息可以参照 Nginx [官方网站](http://nginx.org/en/docs/configure.html)。

### 编译安装

下面给出最小的编译参考例子。

``` bash
# rewrite 模块使用 prce 正则表达式，依赖 pcre 库
$ yum install pcre-devel pcre
# gzip 模块依赖 zlib 库
$ yum install zlib-devel zlib
# ssl 模块依赖 openssl 库
$ yum install openssl-devel openssl
# 添加 nginx 用户组
$ groupadd nginx
# 添加 nginx 用户
$ useradd nginx -g nginx -s /sbin/nologin -M
$ ./configure --prefix=/opt -user=nginx -group=nginx --with-http_ssl_module
# 不指定 prefix 则默认安装在 /usr/local 的 nginx 中
# 执行编译并安装
$ make && make install
# 直接启动服务
$ /opt/nginx/sbin/nginx 
```

## 使用 Nginx

Nginx 通过 nginx.conf 文件来配置各项服务。

### 基本配置实例

以下的例子是 Nginx 安装时提供的配置。

``` bash
$ cat nginx.conf
worker_processes  1;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

Nginx 是通过 master process 和 worker process 协同工作的，其中 master process 承担管控工作， worker process 承担具体请求处理工作。一般会设置多个 worker process 提高工作效率，数量通常是机器的硬件 CPU 核数。

在 conf 文件中的 worker_processes 是需要启动的 worker process 的数量， worker_connections 则是单个 worker process 负责承载的连接数量。

在 conf 文件中的 http 域用来配置具体的 HTTP 服务，它的下一级主要是通过 server 来划分不同的作用域。 http 域可以设置 HTTP 一些共有的服务配置，比如列出的 default_type 和 include ，它们主要定义 HTTP 服务的支持传输的类型相关内容， keepalive_timeout 设置长连接的超时时间。

单个 Nginx http 域可以配置多个 server ，这些 server 甚至可以使用相同的 listen 端口，但需要至少保持不同 server_name 。 server_name 用来配置域名， loaction 用来匹配 url ，它们共同组合定位资源。

error_page 用来导向 http 响应错误的返回页面，如果它放在 server 域中，则所有的 location 都会受到它的影响。

除了 HTTP 服务， Nginx 还支持邮件服务和传输层服务，可以在 conf 中配置 mail 域和 stream 域来定义相关服务细节，但这些服务需要额外编译对应的模块。

### loaction 实现资源匹配

location 通过 url 来匹配目标资源。

具体的匹配方式有 prefix string 和 regular expression ，也就是前缀匹配和正则匹配，还有比较特殊的完整匹配，可以参考 location 的语法规则。

> Syntax: `location [ = | ~ | ~* | ^~ ] url { ... }`

各个匹配符号的含义如下：

* `=` 用来代表完整匹配，只有 url 完全一致才会被这个 location 所匹配，这是最高优先级的规则。
* `^~` 表示从头开始匹配，开头的符号和普通正则的符号一致，这是前缀匹配的一种。
* 不带任何修饰符也是一种前缀匹配。
* `~` 和 `~*` 代表 location 采用正则匹配，带 `*` 不区分大小写，不带 `*` 则需要匹配大小写。

一个请求到达 Nginx 时的具体匹配过程由 server 中所有的 location 共同参与，一般的匹配规则如下：

1. 拿到请求 url ，进行解码，去除重复的 `/` 符号等规范化处理。
2. 对 url 进行精准匹配，如果匹配成功，立即返回结果并结束匹配过程。
3. 进行前缀匹配，如果成功匹配到带 `^~` 修饰符的最长匹配，立即返回结果并结束匹配过程。
4. 在第二步没有直接结束匹配的情况下，继续寻找最长匹配的前缀匹配，并临时存储这个匹配。
5. 由上至下逐一进行正则匹配。
6. 如果正则匹配成功，立即返回结果并结束匹配过程。
7. 如果正则匹配没有结果，则将存储的最长匹配作为最终结果返回，并结束匹配过程。

整个匹配过程主要复杂的地方在于前缀匹配，在第三个步骤时，带修饰符和不带任何修饰符两者没有优先之分，都会倾向寻找最长的匹配，但如果这个最长匹配是由 `^~` 修饰的，则直接返回这个匹配。不带修饰符的写法和带 `^~` 的写法的目标不能完全相同， Nginx 的配置检查将会无法通过，可以把一些更为详细的前缀匹配使用 `^~` 修饰，可以加快解析速度。

在匹配进入到某个 location 中后，匹配资源的工作就由关键字 alias 和 root 来定义，它们用来指向资源在服务器中的位置，一直以来，网上对 alias 和 root 的说法有很多，这里给出一些我的测试过的结论以供参考：

* root 和 index 已经被隐式定义在整个 server 域中，默认位置和前面给出的默认配置实例一致，如果不进行覆写，隐式地址为 Nginx 安装后自带的 html 目录， index 默认为目录下的 index.html 。
> 这个结果可以通过去除所有 location 和 server 的关键字定义，并在 html 下随意写入一个 index.html 文件得出。
* 在不配置任何错误定向页面时， Nginx 会使用默认的 404 和 50x 的页面。
* 如果你在 location 中重定义 root ，则这个 root 只能在当前 location 域中生效。这种配置方式除了要匹配 location ，还需要 url 在配置的 root 中准确命中资源才不会返回 404 错误。
> 使用这类配置的常见情况是 url 匹配了 location ，但是无法找到正确的资源路径，比如 `location /cn { root country/cn; }` 这种写法的寻找目录实际为 `country/cn/cn/` ，这是我个人在一开始使用 Nginx 时经常犯的错误，如果要在 location 中使用 root ，最好在 root 这个目录下拥有和 location 匹配条件一致的同名子目录。
* alias 只能位于 location 块中， root 可以放在 location 和 server 中， Nginx 自带的配置语法检查可以帮助我们避免这种错误。
> 对比 root 和 alias ， alias 的使用会更加灵活，它不会被匹配条件限定，可以自由地指定资源目录，而 root 已经在 url 匹配时隐式限定了固有的资源目录，所以推荐在 location 中使用 alias 。

### rewrite 实现重定向功能

rewrite 模块提供了灵活的重定向功能。

rewrite 模块提高了 rewrite 关键字， rewrite 只能放在 server ， location 或者自定义的 if 中，它的语法规则如下：

> Syntax: rewrite regex replacement [flag];

一个请求的重写过程一般有如下几步：

1. url 按照正常流程进行 location 匹配。
2. 进入到某个包含了 rewrite 指令的 location 中。
3. 直接执行 rewrite ，根据定义的 regex 执行 url replace 。
4. 根据 flag 决定替换后 url 的走向。

flag 定义的走向结果有以下几个：

* last ： 替换 url 之后终止后续的 rewrite 行为，直接返回替换后 url ，并使用它重新匹配 location 。
* break ： 替换 url 之后终止所有 rewrite 行为，包括 rewrite ， return ， if 等，然后使用替换后 url 在当前的 location 中匹配资源。
* redirect ： 302 临时重定向。
* permanent ： 301 永久重定向。

如果因为重写配置导致了替换后的 url 进行了多次匹配，匹配次数不能超过 10 次，否则会停止匹配并触发 500 Internal Server Error 的 http 响应。

rewrite 的最常见写法是 `rewrite ^/(.*)$ url/$1/ last ;` ，利用正则匹配式的括号取出需要的部分，再根据需要以变量的形式构造新的 url 并定义 flag 。

除了 rewrite 模块， try_file 关键字也提供了重定向的功能，但 try_file 是来自 ngx_http_core_module 的关键字。  

> Syntax: try_files file ... uri;

try_files 用于 location 中，可以指定多个 file ，它按顺序检查文件是否存在，并使用第一个找到的文件进行请求处理。

try_files 的一般写法是 `try_files urlA urlB urlC` ， urlA 将会直接在当前 location 的 root 或 alias 中寻找匹配资源，后续的 urlB 也是相似的做法，如果它们依次都无法找到资源，才会使用最后的 urlC 执行新一轮的 location 匹配，整个过程和 url 重写非常相似，可以看做是简化版本的 rewrite 。

### proxy 实现代理功能

proxy 模块提供了代理功能，代理实际上就是对请求的转发。

代理分为正向代理和反向代理，它们的工作本质是一样的，主要不同在于工作的场景，其中正向代理主要是面向用户提供代理服务，受保护的对象是用户，反向代理主要是面向服务端提供代理服务，受保护的对象是服务端。

一个普通的 http 正向代理配置如下：

``` bash
location / {
    # 特殊指定的 dns 服务器
    resolver 8.8.8.8;
    # 请求不做更改
    proxy_pass $scheme://$http_host$request_uri;
}
```

如果用户需要通过代理上网，需要手动设置代理服务器。

一个普通的 http 反向代理配置如下：

``` bash
location / {
    # 指向上游服务器，可以配合 upstream 使用
    proxy_pass        http://localhost;
    proxy_set_header  Host             $host;
    proxy_set_header  X-Real-IP        $remote_addr;
    proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
}
```

由于反向代理架构中，真正提供服务的是上游节点，所以代理服务器需要向真正提供服务的节点传递客户信息，要在请求转发时设置一些特殊的头部信息。

### upstream 实现负载均衡

upstream 提供负载均衡功能，一般配合 proxy 模块使用，可以将反向代理的目的地由一个节点扩展为一个带有负载均衡功能的上游集群。

一个普通的 http 反向代理和负载均衡配置如下：

``` bash
upstream servers {
    server server1.com;
    server server2.com;
    server server3.com;
}

location / {
    # 指向上游服务器，可以配合 upstream 使用
    proxy_pass        http://servers;
    proxy_set_header  Host             $host;
    proxy_set_header  X-Real-IP        $remote_addr;
    proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
}
```

upstream 用来定义上游集群， server 为集群内的具体节点，而且在这些 server 之间， upstream 还提供了多种负载均衡策略，现有的负载均衡策略如下：

* round-robin ：将请求轮询到负载集群中。
* least-connected ：基于最小连接数决定负载对象。
* ip-hash ：基于请求 IP 地址的散列算法。

具体的配置方法也非常简单，可以参考下面的例子：

``` bash
# 默认使用轮询策略，所有子节点没有优先级之分
upstream servers {
    server server1.com;
    server server2.com;
    server server3.com;
}
# 使用加权轮询策略， weight 越大，命中的次数将会越高
upstream servers {
    server server1.com weight=3;
    server server2.com weight=2;
    server server3.com;
}
# 使用最小连接数策略
upstream servers {
    least_conn;
    server server1.com;
    server server2.com;
    server server3.com;
}
# 使用 ip-hash 策略
upstream servers {
    ip_hash;
    server server1.com;
    server server2.com;
    server server3.com;
}
```

对于这些负载均衡策略，轮询策略比较简单，按进站的请求顺序，依次给到后端的节点，达到负载均衡的效果，但是默认的轮询策略比较笨，可以通过加权提高轮询的灵活性。

加权轮询可以实现根据权重分配负载， weight 越大，它就会被分配到更多的请求。

最小连接数策略是检测所有的上游节点，哪个节点上的连接数最小，就将请求发给这个节点。

比较特殊的是使用 hash 算法的策略， hash 策略的最大特点是可以使来源一致的请求固定到某一个上游节点中，达到保持会话的效果。

Nginx 有直接的 ip_hash 策略，它根据请求的 IP 计算 hash 值，通过 hash 值和上游节点数量进行取模，决定它所到达的上游节点。

除了 ip_hash ，还提供了 hash 关键字，可以自行选用字段作为计算 hash 值的对象。

> Syntax: hash key [consistent];

对比于 ip_hash ，它的好处在于可以自由选定字段，并且可以提供指定 consistent 关键字来支持一致性 hash 。

取模 hash 和一致性 hash 的差别是比较明显的，一致性 hash 更具灵活性，无论是新增或者减少上游节点，对整体服务影响都是比较小的，取模 hash 在上游节点的灵活增减上就没有优势，容易造成大规模的会话失效。而且根据 Nginx 的源码，在 ip_hash 策略中，只取了 IP 地址的前三段进行 hash 计算，在一些 IP 地址集中的环境下很可能造成负载不均衡，在使用时需要格外小心。

## 总结

就个人目前接触过的场景来说， Nginx 如果只是简单地代理一些后端服务还是比较简单的，这里只是简单介绍了基本的 Nginx 概念和常用的内容，更加细节的使用可能没有涉及到。

作为基础服务使用，平时可能更关注各个 location 和反向代理的功能，如果需要撑起一个比较庞大的后端结构，可以考虑使用 openresty 。
