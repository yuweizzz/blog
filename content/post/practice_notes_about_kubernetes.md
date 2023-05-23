---
date: 2022-09-10 15:22:45
title: Kubernetes 实践笔记
tags:
  - "kubernetes"
draft: false
---

这篇笔记用来记录一些 Kubernetes 的相关知识。

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

## 安装 kubernetes

这里主要记录一些 kubernetes 初始化遇到的问题和解决方法。

首先是 `kubelet` ， `kubeadm` 和 `kubectl` 三个基本的软件，此外你还需要额外安装容器运行时，可以选择 `docker` ， `containerd` 或者 `crio` 等。

根据官方文档和实际运行情况来看，由于国内网络的问题，为了避免拉取镜像失败，最好把容器运行时配置中的 `pause` 镜像修改掉，以 `crio` 为例子：

``` bash
$ cat /etc/crio/crio.conf
......
[crio.image]
pause_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
......

$ systemctl enable crio
$ systemctl reload crio
```

修改容器运行时配置并重启服务后，还需要启用内核模块和修改一些内核参数：

``` bash
# 修改内核参数
$ echo 1 > /proc/sys/net/ipv4/ip_forward  # 不开启 ip forward 初始化会报错
# 持久化内核参数
$ echo net.ipv4.ip_forward = 1 > /etc/sysctl.d/kubernetes.conf

# 启用内核模块
$ modprobe br_netfilter  # 不启用 br_netfilter 初始化会报错
```

然后就可以使用 `kubeadm` 来初始化控制平面：

``` bash
# 直接调用 init 命令来进行初始化
$ kubeadm init \
  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
  --pod-network-cidr 10.244.0.0/16 \
  --service-cidr 10.96.0.0/16 \
  --cri-socket unix:///var/run/crio/crio.sock \
  --v 5

# 生成默认配置文件并进行修改，然后根据配置文件进行初始化
$ kubeadm config print init-defaults --component-configs KubeProxyConfiguration,KubeletConfiguration > config.yaml
$ kubeadm init --config config.yaml
# config.yaml 中需要修改的地方有：
# InitConfiguration.localAPIEndpoint.advertiseAddress 需要修改为主机 IP 地址
# InitConfiguration.nodeRegistration.name 按照需要修改为对应主机名
# InitConfiguration.nodeRegistration.criSocket 需要修改为正确的容器运行时监听地址，相当于 --cri-socket
# ClusterConfiguration.imageRepository 需要修改为国内的镜像地址，相当于 --image-repository
# ClusterConfiguration.networking.serviceSubnet 需要修改为 service 网络范围，相当于 --service-cidr
# ClusterConfiguration.networking.podSubnet 需要修改为 pod 网络范围，相当于 --pod-network-cidr
# KubeProxyConfiguration.mode 可以按照需要修改为 ipvs ，否则默认使用 iptables
```

初始化完成后，可以使用 `kubectl` 来查看集群状态：

``` bash
# 复制证书信息到当前用户工作目录下
$ cp /etc/kubernetes/admin.conf $HOME/.kube/config
$ kubectl get node
$ kubectl get pods -A

# 直接声明 KUBECONFIG 变量指定证书信息
$ export KUBECONFIG=/etc/kubernetes/admin.conf
$ kubectl get node
$ kubectl get pods -A
```

这时节点仍处于 NotReady 状态，因为网络插件还没有安装，可以选择需要的网络插件进行安装，这里以 `flannel` 为例子：

``` bash
$ curl -L -O https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# 修改 kube-flannel.yml 中 ConfigMap 的 Network 为需要的 podSubnet
$ kubectl apply -f kube-flannel.yml
```

安装完成后整个节点就会处于 Ready 状态，主节点安装完成。接下来如果要为这个集群添加工作节点，那么需要在工作节点上安装 `kubelet` ， `kubeadm` 和容器运行时，为了保证工作节点的正常运行，也应该开启对应的内核模块并修改内核参数，然后执行对应的命令即可：

``` bash
# 主节点生成 token
$ kubeadm token create --print-join-command
# 生成的 token 可以在 secret 中找到，对应的 type 为 bootstrap.kubernetes.io/token
$ kubectl get secret -n kube-system

# 工作节点通过主节点提供的 token 加入集群
$ kubeadm join <api-server-endpoint> --token <discovery-token> --discovery-token-ca-cert-hash <discovery-token-ca-cert-hash>
```

## 允许控制平面节点调度

kubernetes 的控制平面节点默认是不允许调度的，它在一些低版本中也称为 Master 节点或主节点，可以通过 `kubectl describe nodes <name>` 看到主节点信息中带有污点 `node-role.kubernetes.io/control-plane:NoSchedule` ，可以通过去掉这部信息来允许主节点进行 Pod 调度。

``` bash
# 假设主节点名称为 k8s-master
# 去除主节点禁止调度的污点
$ kubectl taint node k8s-master node-role.kubernetes.io/control-plane-

# 还原主节点禁止调度的污点
$ kubectl taint node k8s-master node-role.kubernetes.io/control-plane="":NoSchedule
```

## 修改 Pod 的 Host

有时候需要把一些特殊域名指定为固定的 IP 地址，可以通过 `hostAliases` 来实现，它相当于向 `/etc/hosts` 中添加了对应的信息。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostaliases-pod
spec:
  restartPolicy: Never
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
  containers:
  - name: cat-hosts
    image: busybox:1.28
    command:
    - cat
    args:
    - "/etc/hosts"
```

上述文件是官方文档给出的实例，如果是 `pod` 则应该将 `hostAliases` 添加到 `.spec.hostAliases` 这个字段中，而在 `deployment` 中则应该添加到 `.spec.template.spec.hostAliases` 这个字段中。

## br_netfilter 的重要作用

在 Pod 中通过 service 的名称来访问对应的服务时， `br_netfilter` 起到极为重要的作用，建议把这个模块写入到模块开机启动配置 `/etc/modules-load.d/modules.conf` 当中。

由于一开始没有启用这个模块，导致所有域名访问方式都超时失效，所以就选择使用 dnsutils 镜像进行 `dig` 排查，结果返回了奇怪的报错信息： `reply from unexpected source: 10.244.0.22#53, expected 10.96.0.10#53` ，在查看官方 issue 之后，直接在对应节点上执行 `modprobe br_netfilter` 马上就解决了问题。

其实这里也侧面反映出了 service 的服务原理， Pod 所配置的 `/etc/resolv.conf` 是 kube-dns service 的 IP 地址 `10.96.0.10` ，所以 DNS 请求直接向这个地址发起请求，数据包正常从 Pod 内经过 kube-proxy 后发往 kube-dns 的任一 Pod 中，但是在数据包返回时，由于没有启用 `br_netfilter` 模块， nat 没有正常修改返回地址，导致最终返回到 Pod 中时，远端 IP 地址是 kube-dns 其中某一个 Pod IP ，被 Pod 认为不是正确的返回包直接丢弃，产生超时报错。

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

## Service type

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
