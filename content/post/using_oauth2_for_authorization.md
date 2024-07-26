---
date: 2024-07-11 11:00:00
title: 通过 OAuth2 进行授权
tags:
  - "Kubernetes"
  - "OAuth2"
  - "OpenID Connect"
draft: false
---

通过 OAuth2 实现单点登陆和应用授权。

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

## 前置条件

以下内容全部运行于 kubernetes 环境。

## 安装 OpenID Connect Provider

这里使用 dex 作为 OpenID Connect Provider 并且通过 helm 进行安装。

``` bash
$ helm repo add dex https://charts.dexidp.io
$ helm upgrade --install dex dex/dex --create-namespace --namespace dex
$ helm upgrade dex dex/dex --namespace dex --values dex.yaml
```

`dex.yaml` 的主要内容是 dex 的配置文件，可以在这个文件中对资源进行更多定义。

``` yaml
image:
  tag: v2.38.0

config:
  issuer: http://dex.lan/
  storage:
    type: memory

  oauth2:
    skipApprovalScreen: true
    responseTypes: [ "code", "id_token", "token" ]
    alwaysShowLoginScreen: false

  connectors:
  - type: ldap
    id: ldap
    name: ldap
    config:
      host: openldap-service.openldap-namespace:1389
      insecureNoSSL: true
      bindDN: cn=users,dc=example,dc=com
      bindPW: password
      usernamePrompt: Username
      userSearch:
        baseDN: cn=users,dc=example,dc=com
        filter: "(objectClass=person)"
        username: cn
        idAttr: uid
        emailAttr: mail
        nameAttr: name
        preferredUsernameAttr: cn

  staticClients:
  - id: oauth2-proxy
    redirectURIs:
    - 'http://app.lan/oauth2/callback'
    name: 'oauth2-proxy'
    secret: XltaNeGaLcEZTnDUMTsiXDYH
```

`v2.39.1` 版本的 dex 似乎无法正常实现 `skipApprovalScreen` 配置，降级到 `v2.38.0` 版本使用，可以参考这个 [issue](https://github.com/dexidp/dex/issues/3540) 。

具体的用户数据从 openldap 中获取，可以参考[这里](https://github.com/yuweizzz/devops-tools/tree/master/openldap)快速搭建 openldap 。

## 安装 OAuth2 proxy

应用本身不具备 oauth2 的认证功能的情况下，需要额外部署应用来实现这部分功能。

``` bash
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
helm install oauth2-proxy oauth2-proxy/oauth2-proxy --create-namespace --namespace oauth2-proxy
helm upgrade oauth2-proxy oauth2-proxy/oauth2-proxy --namespace oauth2-proxy --values oauth2-proxy.yml
```

`oauth2-proxy.yml` 的主要内容如下：

``` yaml
config:
  clientID: "oauth2-proxy"
  clientSecret: "XltaNeGaLcEZTnDUMTsiXDYH"
  configFile: |-
    redirect_url = "http://app.lan/oauth2/callback"
    provider = "oidc"
    oidc_issuer_url = "http://dex.lan/"
    upstreams = [ "http://app-service.app-namespace:8080" ]
    email_domains = [ "*" ]
    cookie_secure = false
    cookie_secret = "XLgkh_0hkftnJdtGY3L9YEZpBzp5ITtlwWQ4Vuf-Q-U="
    proxy_websockets = true
```

这里的 upstream 是集群中已有的服务，除了 dex 之外，也可以选择不同的 OpenID Connect Provider ，根据 oauth2-proxy 的官方文档，使用 dex 作为 provider 时需要关闭 `cookie_secure` 。

可以看到这里用到了两个域名 `dex.lan` 和 `app.lan` ，其中 `dex.lan` 是专门留给 dex 对外服务使用的，因为在用户没有经过登陆授权时，需要导向到 dex 进行用户登陆，所以这个服务需要对外暴露。而 `app.lan` 则是实际应用的入口，外部流量经由它代理到具体的上游服务。
