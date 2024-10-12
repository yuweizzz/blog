---
date: 2022-09-10 15:22:45
title: Kubernetes 实践笔记
tags:
  - "kubernetes"
draft: false
---

这篇笔记用来记录一些 Kubernetes 的相关知识，主要是 Kubernetes 实践过程中的相关内容。

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

## 安装 kubernetes

这里主要记录一些 kubernetes 初始化遇到的问题和解决方法。

首先是 `kubelet` ， `kubeadm` 和 `kubectl` 三个基本的软件，此外你还需要额外安装容器运行时，可以选择 `docker` ， `containerd` 或者 `crio` 等。

根据官方文档和实际运行情况来看，由于国内网络的问题，为了避免拉取镜像失败，最好把容器运行时配置中的 `pause` 镜像修改掉，以 `crio` 为例子：

```bash
$ cat /etc/crio/crio.conf
......
[crio.image]
pause_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
......

$ systemctl enable crio
$ systemctl reload crio
```

修改容器运行时配置并重启服务后，还需要启用内核模块和修改一些内核参数：

```bash
# 修改内核参数
$ echo 1 > /proc/sys/net/ipv4/ip_forward  # 不开启 ip forward 初始化会报错
# 持久化内核参数
$ echo net.ipv4.ip_forward = 1 > /etc/sysctl.d/kubernetes.conf

# 启用内核模块
$ modprobe br_netfilter  # 不启用 br_netfilter 初始化会报错
```

然后就可以使用 `kubeadm` 来初始化控制平面：

```bash
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

```bash
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

```bash
$ curl -L -O https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
# 修改 kube-flannel.yml 中 ConfigMap 的 Network 为需要的 podSubnet
$ kubectl apply -f kube-flannel.yml
```

安装完成后整个节点就会处于 Ready 状态，主节点安装完成。接下来如果要为这个集群添加工作节点，那么需要在工作节点上安装 `kubelet` ， `kubeadm` 和容器运行时，为了保证工作节点的正常运行，也应该开启对应的内核模块并修改内核参数，然后执行对应的命令即可：

```bash
# 主节点生成 token
$ kubeadm token create --print-join-command
# 生成的 token 可以在 secret 中找到，对应的 type 为 bootstrap.kubernetes.io/token
$ kubectl get secret -n kube-system

# 工作节点通过主节点提供的 token 加入集群
$ kubeadm join <api-server-endpoint> --token <discovery-token> --discovery-token-ca-cert-hash <discovery-token-ca-cert-hash>
```

## 允许控制平面节点调度

kubernetes 的控制平面节点默认是不允许调度的，它在一些低版本中也称为 Master 节点或主节点，可以通过 `kubectl describe nodes <name>` 看到主节点信息中带有污点 `node-role.kubernetes.io/control-plane:NoSchedule` ，可以通过去掉这部信息来允许主节点进行 Pod 调度。

```bash
# 假设主节点名称为 k8s-master
# 去除主节点禁止调度的污点
$ kubectl taint node k8s-master node-role.kubernetes.io/control-plane-

# 还原主节点禁止调度的污点
$ kubectl taint node k8s-master node-role.kubernetes.io/control-plane="":NoSchedule
```

## 修改 Pod 的 Host

有时候需要把一些特殊域名指定为固定的 IP 地址，可以通过 `hostAliases` 来实现，它相当于向 `/etc/hosts` 中添加了对应的信息。

```yaml
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

由于一开始没有启用这个模块，导致所有域名访问方式都超时失效，通过使用 `kubectl run dnsutils --image=mydlqclub/dnsutils:1.3 --command -- sleep 3600` 启动 Pod 来进行 `dig` 排查，结果返回了奇怪的报错信息： `reply from unexpected source: 10.244.0.22#53, expected 10.96.0.10#53` ，在查看官方 issue 之后，直接在对应节点上执行 `modprobe br_netfilter` 马上就解决了问题。

其实这里也侧面反映出了 service 的服务原理， Pod 所配置的 `/etc/resolv.conf` 是 kube-dns service 的 IP 地址 `10.96.0.10` ，所以 DNS 请求直接向这个地址发起请求，数据包正常从 Pod 内经过 kube-proxy 后发往 kube-dns 的任一 Pod 中，但是在数据包返回时，由于没有启用 `br_netfilter` 模块， nat 没有正常修改返回地址，导致最终返回到 Pod 中时，远端 IP 地址是 kube-dns 其中某一个 Pod IP ，被 Pod 认为不是正确的返回包直接丢弃，产生超时报错。

## 更新镜像和回退

通过某个 deployment 作为实例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
      initContainers:
        - name: init
          image: busybox:1.28
          command: ["sh", "-c", "echo The init containers is running!"]
```

initContainers 和 containers 中的容器都可以更新镜像：

```bash
$ kubectl set image deployment/nginx-deployment init=busybox:1.29  nginx=nginx:1.16.1 --record=true
# set image 可以更新一个或多个镜像
# set image 可以操作的资源对象有：
#   pod (po), replication controller (rc), deployment (deploy), daemonset (ds), replica set (rs)
# 使用 --record=true 可以在 .metadata.annotations.kubernetes.io/change-cause 中记录本次更新镜像所使用的完整命令
```

更新镜像这类使得对应资源产生变化的操作可以通过 rollout 命令回退：

```bash
# undo 会回退到当前资源的前一版本
$ kubectl rollout undo deployment/nginx-deployment

# 可以查看详细的历史版本
$ kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment
# --record=true 会把改变资源的命令记录到 CHANGE-CAUSE 中，否则 CHANGE-CAUSE 应为 <none>
REVISION        CHANGE-CAUSE
1               <none>
2               <none>
...

# 根据详细的历史版本，可以指定需要回退到具体版本
$ kubectl rollout undo deployment/nginx-deployment --revision=1
```

## 安装 ingress controller

除了 service 对象资源， Kubernetes 还在这一基础上抽象出 ingress 的概念。

一般来说我们会基于 deployment 创建出 service ，再通过这些创建出来的 service 使得它们能够向集群外部提供服务，但现在我们不止满足于单纯向外提供服务，我们还想要控制流量的具体行为，这就是 ingress 这一对象存在的原因，它可以认为是对 service 对象中具体流量的控制规则。

那么如果想要使得 ingress 这一资源对象正常工作，我们就需要安装 ingress controller 到集群中。通常云厂商都会提供一些基于自身云服务开发的 ingress controller ，如果我们想要一些独立于云厂商之外的 ingress controller ，也有不少优秀的开源 ingress controller 实现，不过这里我们选择了官方维护的 ingress NGINX controller 。

安装 ingress NGINX controller 可以参考具体的[官方文档](https://kubernetes.github.io/ingress-nginx/)。

如果我们剖析具体的 YAML 文件，我们可以看到除了重要的 deployment 和 service 之外，主要是关于资源权限控制和自定义资源的相关定义。如果我们想要在自建的集群中使用，可以主要关注两个部分：

- 整体服务的可用性。
- service 种类。

由于默认的 deployment 是非常简单，启动后我们会发现这是单 pod 的 deployment ，基于实际上的考虑，我们可以选择将 deployment 修改为 daemonset 来增强容错能力，否则我们至少应该调整 deployment 的副本数来保证服务的可用性，此外还可能需要根据访问量来调节 Pod 资源，并且考虑 Pod Affinity 的相关问题。

而默认的 service 种类是 LoadBalancer ，它需要云厂商提供的负载均衡器来实现，并且这一过程可能要根据云厂商具体的实现文档，对已有的 service 进行某些修改才能正常工作，通常会是修改一些注释信息，同样地， ingress 也可能要做出一些对应的修改，才能正常在基于 LoadBalancer 的服务种类下正常工作。而如果是集群不是基于云服务实现的，那么我们可以选择 NodePort 类型的 service 或者将整个 deployment 运行在主机的网络栈中，这样做甚至可以将 service 省去，但这样做需要额外考虑负载均衡和可用性的相关问题。

## 通过指定 endpoints 的 service 访问外部服务

除了 `ExternalName` 类型的 service 可以实现访问集群外部服务，还可以通过指定 endpoints 来实现。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
---
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
  namespace: prod
subsets:
  - addresses:
      - ip: 10.10.10.10
    ports:
      - port: 8080
```

这里的 service 将会是 `ClusterIP` 类型，但是它的具体 endpoints 不使用 selector 进行选取，而是自行定义到具体服务入口，适用于那些运行在集群外但是在同个内网下的服务。
