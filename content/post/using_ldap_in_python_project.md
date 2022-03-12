---
date: 2022-03-01 16:22:45
title: Python 项目接入 LDAP
tags:
  - "Python"
  - "LDAP"
draft: false
---

这篇笔记用来记录 LDAP 的相关知识以及如何在 Python 项目中接入 LDAP 。

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

## LDAP 的基本知识

LDAP 全称为 Light Directory Access Portocol ，它是基于 X.500 标准的轻量级目录访问协议，经常应用于企业的权限管控和登录认证，比如 AD 域控就是微软基于 LDAP 协议所开发的权限管控应用。

实际上 LDAP 主要是通过维护一个类似于文件目录结构的数据库，根据存储的记录条目提供给外部作为权限管控的数据来源。

由于涉及到 LDAP 有比较多的名词，在这里先给出常用的 LDAP 名词：

* DN ： Distinguished Name ，区分名称，它用来表示每条记录 entry 的唯一位置。
* DC ： Domain Component ，域名的组成部分，多个 DC 共同组成完整的域名。
* OU ： Organization Unit ，组织单元，组织是允许多级存在的。
* CN ： Common Name ，公共名称，也就是记录的名称。

entry 存储了实际的数据值，LDAP 使用 Objectclass 作为 entry 的基本存储对象，它和编程语言中的对象类似，在 Objectclass 内可以定义多个 Attributes 并且设定具体的值。

一条 entry 至少需要拥有一个 Objectclass ，所以获取一条 entry 时，就可以拿到它所定义的 Attributes ，再根据这些值去验证对应的权限。

entry 是通过 DN 来定位的，由于 LDAP 的存储结构是基于层级的，类似于文件目录，我们可以通过类比文件目录来加深对 DC ， OU 和 DN 的理解。

假设某个 LDAP 定义了三个域名： `A.com` ， `B.com` ， `C.com` ，并且下属均各有两个组织 `dev` 和 `prd` ，而我们在 `A.com` 的 `prd` 存放了 `fileA` ，那么整个 LDAP 存储信息会像下面这样：

``` bash
com
├── A
│   ├── prd
│   │   └── fileA
│   └── web
├── B
│   ├── prd
│   └── web
└── C
    ├── prd
    └── web
```

这时我们就可以使用 `cn=fileA,ou=prd,dc=A,dc=com` 这条 DN 来指向 `fileA` 这条 entry 。

可以看到， 在 DN 中 DC 就是对域名的分割，和常规的域名一致，顶级域处于第一位，域名级别越低，它在 DN 中出现的位置越靠前。

而 OU 和 CN 都是可以多级嵌套的，例子中没有体现，但实际中可以有 `cn=fileA,ou=prd,ou=web,dc=A,dc=com` 或者 `cn=fileA,cn=files,ou=prd,dc=A,dc=com` 这样的 DN 。多级 CN 和多级 OU 的顺序也类似于 DC ，都是级别越低，位置越靠前，我们可以自由地根据实际去定义 OU 或者 CN 。

一般来说， CN 出现在 DN 的首位，这里可以使用 uid 或者 sAMAccountName 去替代 CN 在 DN 中的位置，因为它们都是 entry 的 Attributes ，可以需要根据具体使用的 LDAP 服务器决定。

## 接入 LDAP 的代码思路

为了代码更具有通用性，方便接入到各式各样的系统中，认证过程通常可以这样设计：

1. 在 LDAP 中，在比较高等级的域设置一个只读权限的账号，只需要它可以获取下级记录的各项 Attributes 即可。
2. 使用只读账号连接 LDAP 服务器，做好查询准备。
3. 根据具体的 LDAP 设定和业务需求，确定需要使用的 Attributes 。
4. 当用户连接时，把用户的某部分信息作为 Attributes 的搜索条件，使用只读账号在对应域中找到这条记录后拿出对应的 DN 。
5. 使用这条 DN 和用户提供的密码测试连接是否正常，完成验证。

## 具体实现代码

使用以下代码前需要先安装依赖：

``` bash
$ pip install ldap3
```

实际代码内容：

``` python
from ldap3 import Server, Connection

class LDAP(object):
    def __init__(self):
        # 具体的 LDAP 服务器地址
        self.uri = "ldap://192.168.0.1:389"
        # 只读账号的 DN
        self.base_dn = "CN=Admin,ON=Users,DC=example,DC=com"
        # 只读账号的密码
        self.password = "xxxxxx"
        # 具体的搜索域
        self.search_dn = "OU=Accounts,OU=Users,DC=example,DC=com"
        # 初始化连接
        self.server = Server(self.uri)
        self.admin_connection_bind()

    def admin_connection_bind(self):
        self.admin_connection = Connection(
            self.server,
            user = self.base_dn,
            password = self.password,
            auto_bind = True
        )

    def target_connection_bind(self, dn, password):
        auth = Connection(
            self.server,
            user = dn,
            password = password
        )
        if auth.bind():
            auth.unbind()
            return True
        return False

    def search_Attributes(self, name):
        hit = self.admin_connection.search(
            search_base = self.search_dn,
            # 假设这里的 Attributes 为 sAMAccountName
            search_filter = f"(sAMAccountName={name})"
        )
        if hit:
            return self.admin_connection.entries[0].entry_dn
        return None

    def auth(self, user, password):
        dn = self.search_Attributes(user)
        if dn:
            return self.target_connection_bind(dn, password)
        return False

ldap_conn = LDAP()
print(ldap_conn.auth("user", "password"))
```

上述代码还有改进空间，不过已经可以覆盖大部分场景，只需要选定的 Attributes 实际值唯一即可。
