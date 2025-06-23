---
date: 2025-05-10 11:14:00
title: 在 Openresty 中使用 Zstandard 算法实现 Compression dictionary transport
tags:
  - "Openresty"
  - "Zstandard"
  - "Lua"
draft: false
---

通过 Zstd-ffi 在 Openresty 中实现 Compression dictionary transport 。

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

## 前言

[Compression Dictionary Transport](https://datatracker.ietf.org/doc/draft-ietf-httpbis-compression-dictionary/) 是一项比较新的 HTTP 传输协定，和以往的压缩传输相比，比较大的特点是使用了共享压缩字典。

## 基本实现

目前有两种压缩算法支持共享压缩字典，分别是 Brotil 和 Zstandard 。它们都可以指定一些固定文本内容作为压缩字典进行文件压缩，解压文件时也需要通过字典进行解压。这样可以显著降低压缩后的文档大小，在网络传输过程是非常有优势的，但是对应的交换条件是字典进行需要额外传输。

具体协议内容已经由 Compression Dictionary Transport 指定，包括了字典传输，和已经压缩的响应内容如何关联字典。目前 Chromium 的 version 117 及以上版本已经实现了对这个协议的支持。

除了客户端之外，服务端也需要相应的支持，这里可以参考我之前的[实现仓库](https://github.com/yuweizzz/compression-dictionary-transport)，这个例子是最小实现，在投入生产环境时应该根据具体情况进行修改。

关于字典的指定，一般分为动态字典和静态字典。

浏览器会根据 `Use-As-Dictionary` 的 Header 内容决定，将已响应的内容作为字典进行存储，并在向服务端请求时，是否要求服务端传输对应的字典压缩内容。

这里所说的静态字典，典型应用就是响应内容的增量更新。比如某个 JavaScript 脚本的版本更新。它可以把旧版本当作字典，由于旧版本已经存储到浏览器中，所以请求新版本时，字典传输过程就可以节省，只需要传输压缩后的内容。

而动态字典一般需要在服务端先通过训练生成新的字典文件，当服务端的文件变动时，一般需要进行新的字典训练，所以称为动态字典，它可以充当所有文件的字典。

Brotil 和 Zstandard 都有对应的字典训练工具，但是 Zstandard 提供的训练工具会生成的字典会带有 ID 和 Entropy Tables ，而 Brotil 生成的字典是纯文本，在 Compression Dictionary Transport 中使用时需要使用纯文本字典，所以 Zstandard 字典还需要进一步处理，这些工具可以在前述的实现仓库中下载。

```bash
# 通过 zstd 训练字典，然后通过 zstd_dictionary_converter 去掉特定信息
zstd --train content/* -o zstd_dict.dat
zstd_dictionary_converter -i zstd_dict.dat -o dict.dat

# 通过 brotli 训练字典
brotli_dictionary_generator dict.dat content/*
```

## 测试

这里以静态字典为例。

```bash
# base64 of sha256 sum:
# jquery-2.2.4.min.js -> :BbhdlvQf/xTY9gja0Dq3HiwQF8LaCRTXxZKRutelT44=:

# sha256 sum
# jquery-3.7.1.min.js -> fc9a93dd241f6b045cbff0481cf4e1901becd0e12fb45166a8f17f95823f0b1a

# use jquery-2.2.4.min.js as dictionary
curl -k -H "sec-fetch-dest: document" \
  -H "accept-encoding: dcz" \
  -H "available-dictionary: :BbhdlvQf/xTY9gja0Dq3HiwQF8LaCRTXxZKRutelT44=:" \
  'https://localhost:4444/jquery-3.7.1.min.js' -o file

# HTTP/2 200
# server: openresty/1.25.3.2
# date: Tue, 20 May 2025 06:59:10 GMT
# content-type: application/javascript; charset=utf-8
# last-modified: Fri, 18 Oct 1991 12:00:00 GMT
# etag: "28feccc0-155ed"
# use-as-dictionary: match="/jquery-*.js", match-dest=("document")
# content-encoding: dcz
# vary: Accept-Encoding, Available-Dictionary

# decompress
zstd -d -D /var/www/jquery-2.2.4.min.js file -o jquery-3.7.1.min.js
sha256sum jquery-3.7.1.min.js /var/www/jquery-3.7.1.min.js

# fc9a93dd241f6b045cbff0481cf4e1901becd0e12fb45166a8f17f95823f0b1a  jquery-3.7.1.min.js
# fc9a93dd241f6b045cbff0481cf4e1901becd0e12fb45166a8f17f95823f0b1a  /var/www/jquery-3.7.1.min.js
```
