---
date: 2025-03-23 15:06:00
title: 在 Ocserv 中使用双因素身份验证
tags:
  - "ocserv"
  - "otp"
  - "mfa"
draft: false
---

在 Ocserv 认证过程中添加双因素身份验证。

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

一般的 Ocserv VPN 服务使用的是用户密码验证，可以额外添加一层双因素身份验证来增加系统安全性。

使用用户密码验证的配置：

```bash
# /etc/ocserv/ocserv.conf
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
```

额外添加一层双因素身份验证非常简单，只需要修改认证配置，其他的原有配置不受影响：

```bash
# /etc/ocserv/ocserv.conf
auth = "plain[passwd=/etc/ocserv/ocpasswd,otp=/etc/ocserv/users.oath]"
```

后续的用户 otp 信息都需要添加到 `/etc/ocserv/users.oath` 文件中。以 `testuser` 用户为例，除了本身在 `/etc/ocserv/ocpasswd` 有对应的用户密码信息之外，通过以下命令来添加 otp 配置：

```bash
echo "HOTP/T30 testuser - $(head -c 16 /dev/urandom | xxd -c 256 -ps)" >> /etc/ocserv/users.oath
```

此时用户登陆 VPN 并输入用户密码后，会要求其输入 otp 验证码。所以还需要将密钥信息分发给对应的用户，让他们在自己的 otp 应用中绑定。

```bash
# 以下命令只是示例，实际中这里应该使用 /etc/ocserv/users.oath 中的密文信息
echo "0x$(head -c 16 /dev/urandom | xxd -c 256 -ps)" | xxd -r -c 256 | base32
```

生成的 base32 编码字符串就是密钥，以 Google Authenticator 为例，用户需要通过输入设置密钥，选择基于时间的密钥类型，然后自行添加保存，以便后续使用。

参考文档：

- [mod-authn-otp wiki](https://github.com/archiecobbs/mod-authn-otp/wiki/UsersFile)。
- [openconnect vpn how-to guides](https://docs.openconnect-vpn.net/recipes/ocserv-2fa/)。
