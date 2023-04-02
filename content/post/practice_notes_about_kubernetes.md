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
