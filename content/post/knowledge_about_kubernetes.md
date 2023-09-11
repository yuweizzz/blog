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

## Kubernetes 身份认证与鉴权

在 Kubernetes 中， Pod 和其他的工作负载资源是我们最经常接触的资源对象，通常来说它们是直接和集群外部交互的，但是当它们需要进行集群内部交互，比如最常见的与 ApiServer 进行交互的时候，就会涉及到身份认证与鉴权，在 Kubernetes 中是一般使用基于 RBAC 的鉴权方式来实现。

在这个认证体系中，比较重要的是 `ServiceAccount` 资源对象，它在每个新的 `namespace` 创建时都会随之默认生成，实际上也就是这个命名空间中的资源向 ApiServer 请求时默认使用的身份，当然除了这个默认对象之外也可以自行创建。

``` yaml
# 来自 ingress nginx controller manifest 中的 ServiceAccount
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.8.2
  name: ingress-nginx
  namespace: ingress-nginx

---
# 来自 Kubernetes 官方文档，带有 imagePullSecrets 的 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
imagePullSecrets:
  - name: myregistrykey
```
 
`ServiceAccount` 中的 `automountServiceAccountToken` 参数可以控制是否将 `ServiceAccount` 对应生成的 token 挂载到具体的工作负载中，一般会在容器的 `/run/secrets/kubernetes.io/serviceaccount/` 路径下，但是 token 的具体内容信息跟所在 Kubernetes 集群的版本有关。 `ServiceAccount` 还可以通过 `imagePullSecrets` 参数来关联 `Secret` 类型资源，使用了这种关键字的 `ServiceAccount` 就会具有特定的镜像拉取认证信息。

具体的 `ServiceAccount` 可以通过工作负载资源对象中的 `serviceAccountName` 参数来具体指定，否则工作负载默认使用创建命名空间时自动创建的 `ServiceAccount` 。

但是只凭借 `ServiceAccount` 是不够的，可以说它只是确认了请求方究竟是谁的问题，也就我们说的身份认证。而对于请求方实际具有的权限则是通过 `Role` ， `RoleBinding` ，`ClusterRole` 和 `ClusterRoleBinding` 来定义。

这种方式就是基于 RBAC 的鉴权方式，通过具体身份绑定了定义具体权限的角色，这样请求方认证身份后就拥有对应的权限。

{{<details "`Role` 和 `ClusterRole` 具体实例">}}

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.8.2
  name: ingress-nginx
  namespace: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resourceNames:
  - ingress-nginx-leader
  resources:
  - leases
  verbs:
  - get
  - update
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.8.2
  name: ingress-nginx
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - nodes
  - pods
  - secrets
  - namespaces
  verbs:
  - list
  - watch
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - discovery.k8s.io
  resources:
  - endpointslices
  verbs:
  - list
  - watch
  - get
```

{{</details>}}

根据 Kubernetes 的官方文档， `Role` 和 `ClusterRole` 都是用来定义具体权限规则的鉴权资源，但是 `Role` 只能用于定义某个命名空间范围内的资源对象，而 `ClusterRole` 可以定义甚至整个集群作用域范围内的资源对象，可以看到它们的具体内容都是根据不同的 apiGroups 来定义对应资源的访问方法，一般只要结合两者就可以覆盖所有资源对象的具体访问权限。

在创建 `Role` 和 `ClusterRole` 之后，就完成了对角色具体权限的定义，这时只需要将这个角色和具体用户绑定，就可以赋予这个用户这些已经定义的权限，这一步需要用到的是 `RoleBinding` 和 `ClusterRoleBinding` 的资源对象。

``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.8.2
  name: ingress-nginx
  namespace: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.8.2
  name: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress-nginx
subjects:
- kind: ServiceAccount
  name: ingress-nginx
  namespace: ingress-nginx
```

在完成上述资源的定义之后，就可以使用对应的工作负载中的 token 来和 ApiServer 进行交互，很多实际使用的云原生服务都用到了这些资源。
