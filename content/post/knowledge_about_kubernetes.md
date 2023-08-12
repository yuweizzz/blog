---
date: 2022-09-11 15:22:45
title: Kubernetes 知识笔记
tags:
  - "kubernetes"
draft: false
---

这篇笔记用来记录一些 Kubernetes 的相关知识，主要是 Kubernetes 基本原理的相关内容。

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

## Pod 的 DNS 策略

根据官网，目前 Kubernetes 支持以下特定 Pod 的 DNS 策略，设定字段为 `.spec.dnsPolicy` ：

* `Default` ： Pod 从运行所在的节点继承 `/etc/resolv.conf` ，但是这种策略并不是 Pod 默认的 DNS 策略。
* `ClusterFirst` ：使用集群提供的 DNS 服务，通常使用 CoreDNS 作为集群 DNS ， `ClusterFirst` 才是 Pod 默认的 DNS 策略。
* `ClusterFirstWithHostNet` ：对于以 hostNetwork 方式运行的 Pod ，应该将它的 DNS 策略显式设置为 `ClusterFirstWithHostNet` 。
* `None` ：允许 Pod 忽略集群中的 DNS 设置并使用 `.spec.dnsConfig` 作为 DNS 设置。

使用 `Default` 策略时， Pod 中的 `/etc/resolv.conf` 将和宿主机中的 `/etc/resolv.conf` 完全一致，这时就无法通过 service name 域名的方式去访问内部服务。

使用 `ClusterFirst` 策略时，可以突破 `Default` 策略的限制，在 Pod 中直接通过 service name 域名去访问集群中 service 类型的资源，这时 Pod 中的 `/etc/resolv.conf` 一般会是如下的内容：

``` bash
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

可以看到 `search` 把搜索域限定在集群中，所以我们就可以直接访问内部服务了，不跨命名空间的 service 访问将在 `default.svc.cluster.local` 域中搜索，因为这部分内容取自 `default` 命名空间中的 Pod ，不同命名空间中的 Pod 将会自动跟随当前所在空间；跨命名空间的 service 访问将在 `svc.cluster.local` 域中搜索，但没有显式给出命名空间，所以要由使用者自动指定 service name 加 namespace name 的域名。

而 `options` 中的 `ndots` 是用来判断是否需要使用 `search` 查找域的凭据，如果给定域名中的 `.` 小于 `ndots` 定义的数值，则会先在所有 `search` 定义的域中查询，在这些查询都失败后再直接使用给定域名进行查询。当访问内部服务时，不论是否跨命名空间，给定域名是 `service-name` 或 `service-name.namespace-name` 这种形式，它们都会被 `ndots` 命中并执行 `search` 查找。

使用 `ClusterFirstWithHostNet` 策略则是在 `.spec.hostNetwork` 值为 `true` 时所需要使用的关键字，不使用这个策略，则 `.spec.hostNetwork` 值为 `true` 的 Pod 无法访问集群内部服务。

使用 `None` 策略可以参考官方给出的配置参考，定义自己需要的配置：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 192.0.2.1
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

## Service 类型

Kubernetes 的 sevice 资源有四种类型，它决定了集群服务是如何暴露的，设定字段为 `.spec.type` ：

### ClusterIP

`ClusterIP` 是默认和最常见的服务类型，成功创建资源后集群会自动分配服务的 IP 地址，这个 IP 地址和 Pod 网段是隔离的，它的网络通信依赖于 kube-proxy 组件。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc
```

通常会将 Pod 或者 Deployment 资源加上 label ，然后使用 Service selector 来选中这些资源，这样的 Service 就可以将请求均衡发送到 Pod 集合中。

### NodePort

`NodePort` 是 `ClusterIP` 的扩展类型。除了集群内部的服务 IP 地址，还会在集群中所有节点的对应端口进行监听，并代理到各自节点中的 Pod 中，实现外部访问集群内服务。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```

在 yaml 文件中， port 是 Pod 暴露服务的端口， targetPort 是 Service 暴露服务的端口，两者可以保持一致，由 targetPort 跟随 port 即可， nodePort 可以显式指定，也可以不进行指定并由集群自动分配，通常采取第二种方式，来避免手动指定的端口和已有的服务产生冲突。自动分配的范围由 apiServer 中的 `--service-node-port-range` 参数指定，默认值是 `30000-32767` 。

### LoadBalancer

`LoadBalancer` 是基于云厂商提供服务实现的服务类型，它基于 `NodePort` 进行扩展，和云厂商的负载均衡器绑定，是更高级的外部访问集群内服务实现方式。

可以认为这里 `NodePort` 是隐式实现的，并且云厂商还应该提供负载均衡器，它会对集群中所有节点代理，实现负载均衡。这样外部访问不再直接访问集群节点而是通过负载均衡器进行代理转发。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

在创建 LoadBalancer Service 时，通常只需要指定 `LoadBalancer` 作为 Service 类型即可，关于 port 的设置和 `nodePort` 类型相似。创建资源完成后会将负载均衡器信息报告在 status 中。

### ExternalName

`ExternalName` 是不直接关联 Pod 的服务类型，它实际是通过创建 CNAME 记录，来实现 Service 访问映射到其他域名。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

从官方文档中的实例可以非常明确这个服务类型的作用，它创建了一个内部服务，在提供给其他服务的同时，它本身并未在集群中运行，而这个服务只是在内部 DNS 创建 CNAME 记录，当其他服务访问时，实际上会通过 CNAME 访问到集群外部。也就是在访问 `my-service.prod.svc.cluster.local` 时，实际相当于访问 `my.database.example.com` 。

除了映射外部域名，它也经常用于将其他命名空间的服务映射为本地命名空间的服务，方便内部服务的域名访问规范。
