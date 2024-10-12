---
date: 2024-04-18 15:44:45
title: 搭建 k3s 集群
tags:
  - "K3s"
  - "Cilium"
  - "Apisix"
draft: false
---

搭建 k3s 集群，同时使用 Cilium 和 APISIX Ingress Controller 。

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

## 安装 k3s

```bash
# 使用 etcd 作为配置中心
apt install etcd-server etcd-client

# 安装 k3s
# 禁用网络策略 --disable-network-policy
# 修改配置中心 --datastore-endpoint
# 启用内置 iptables 替换系统使用的版本 --prefer-bundled-bin
# 禁用组件 kube-proxy, flannel, traefik, metrics-server, servicelb
export INSTALL_K3S_EXEC='--datastore-endpoint=http://127.0.0.1:2379 --disable-network-policy --flannel-backend=none --disable-kube-proxy --disable=traefik --disable=servicelb --disable=metrics-server --prefer-bundled-bin'

# 禁用 metrics-server 可能会导致 HPA 不可用，应该根据项目情况来禁用这一选项。

# 执行安装
curl -sfL https://get.k3s.io | sh -
# 中国用户可以通过以下命令加速安装，避免出现下载失败的问题
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

因为后续会使用 cilium 替换默认的 flannel 网络插件，需要先使用 `--disable-network-policy` 和 `--flannel-backend=none` 禁止 k3s 自动安装网络插件。同时会通过 cilium 来彻底替换 kube-proxy ，所以会使用 `--disable-kube-proxy` 禁止 k3s 启动相关的组件。

traefik 的功能由 Apisix Ingress 接管， servicelb 的功能则通过 cilium 的 L2 Announcement 实现。

相应的 `--datastore-endpoint` 具体信息记录在 `/var/lib/rancher/k3s/server/db` ，而 k3s 集群创建的 local-path 类型的 PVC 则存储在 `/var/lib/rancher/k3s/storage` 。

通过修改 k3s registries 配置来解决 docker 被墙后的镜像拉取错误问题。

```bash
# 添加 /etc/rancher/k3s/registries.yaml
cat /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"

# 重启 k3s
systemctl restart k3s
```

## 安装 cilium

```bash
# 安装 helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sh -

# 安装 cilium
helm repo add cilium https://helm.cilium.io/
helm repo update
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm upgrade --install cilium cilium/cilium --version 1.15.1 \
    --create-namespace --namespace cilium \
    --set operator.replicas=1 \
    --set ipam.mode=cluster-pool \
    --set ipam.operator.clusterPoolIPv4PodCIDRList=10.0.0.0/16 \
    --set routingMode=native \
    --set autoDirectNodeRoutes=true \
    --set ipv4NativeRoutingCIDR=10.0.0.0/16 \
    --set bpf.masquerade=true \
    --set kubeProxyReplacement=true \
    --set loadBalancer.mode=dsr \
    --set loadBalancer.acceleration=native \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set installNoConntrackIptablesRules=true
```

cilium helm 参数可以参考以下信息：

- `operator.replicas` 的默认值是 2 ，这里使用 1 个副本就足够了。
- `ipam.mode` 默认就是 cluster-pool 模式，相应的要通过 `ipam.operator.clusterPoolIPv4PodCIDRList` 用来设置允许分配的 Pod CIDR 范围。
  > 设置之后可以通过 CiliumNode 查看具体的 CIDR 分配，具体节点上的 CIDR 由 cilium operator 分配。
- `routingMode` 使用 native 模式，也就是 cilium 的 Native-Routing 模式， native 模式的性能高于默认的 tunnel 模式， `ipv4NativeRoutingCIDR` 用于声明 Pod CIDR ， cilium 会认为这部分的流量是可以由系统网络直接路由，不需要使用 SNAT ， `autoDirectNodeRoutes` 允许 cilium 修改节点的路由信息，使得 Pod CIDR 可以被直接路由，适用于所有 worker 节点处在同一 L2 网络。
- `bpf.masquerade` 设置为 true 时，会使用 ebpf 来实现地址伪装，否则默认是通过 iptables 去实现的，主要针对 Pod 内部和集群外部之间的流量。
- `kubeProxyReplacement` 设置为 true 允许 cilium 使用内置功能替换 kube-proxy 。 `loadBalancer.mode` 设置为 dsr 是针对 NodePort 类型的 ebpf 增强，允许通过 ebpf 修改数据包，使得 Pod 不在当前节点的流量可以直接返回客户端，而不是通过 SNAT 进行跳转。 `loadBalancer.acceleration` 是优化选项，设置为 native 允许数据包通过 XDP 直接在网卡驱动部分处理，而不是系统内核，可以提高性能。
  > 虚拟机上的某些网卡驱动可能不支持 XDP ，这时 `loadBalancer.acceleration` 的 native 模式不可用，直接去掉即可。
- `installNoConntrackIptablesRules` 设置为 true 可以禁用流量的连接跟踪，也就是 iptables 提供的流量追踪功能，可以提高性能。
- `hubble.relay.enabled` 和 `hubble.ui.enabled` 是可观测性的相关部分，它们都是默认关闭的，这里重新启用了它们。其中 ui 是对应的图形界面， relay 是 hubble 的中继服务，用来沟通不同 hubble 实例和提供更多的 api 支持，开启 hubble 会略微降低一些性能。

关于性能优化的部分可以参考[官方文档](https://docs.cilium.io/en/stable/operations/performance/tuning/) ，关于 Host-Routing ， installNoConntrackIptablesRules 等性能优化的选项都可以找到对应说明。

正常安装后通过 cilium pod 查看安装信息和配置情况。

```bash
# 查看 cilium 状态
kubectl exec -it -n cilium cilium-5r2nh -- cilium status --verbose

# 以下是命令的输出结果，已经省略了部分内容
KubeProxyReplacement:   True   [ens192   192.168.10.100 fe80::20c:29ff:fe89:c031 (Direct Routing)]
Cilium:                 Ok   1.15.1 (v1.15.1-a368c8f0)
# Host Routing 在内核支持 ebpf 和使用 KubeProxyReplacement 等条件下会使用 BPF ，默认是 Legacy
Host Routing:           BPF
# bpf.masquerade 需要设置为 true ，否则此处使用 iptables
Masquerading:           BPF   [ens192]   10.0.0.0/16 [IPv4: Enabled, IPv6: Disabled]
# 开启 Hubble
Hubble:                  Ok   Current/Max Flows: 4095/4095 (100.00%), Flows/s: 11.27   Metrics: Disabled
KubeProxyReplacement Details:
  # kubeProxyReplacement 需要设置为 true
  Status:                 True
  Socket LB:              Enabled
  Socket LB Tracing:      Enabled
  Socket LB Coverage:     Full
  Devices:                ens192   192.168.10.100 fe80::20c:29ff:fe89:c031 (Direct Routing)
  # loadBalancer.mode 需要设置为 dsr
  Mode:                   DSR
    DSR Dispatch Mode:    IP Option/Extension
  Backend Selection:      Random
  Session Affinity:       Enabled
  Graceful Termination:   Enabled
  NAT46/64 Support:       Disabled
  # 如果 loadBalancer.acceleration 设置为 native ，此处应该会被启用
  XDP Acceleration:       Disabled
```

使用 cilium 的 L2 Announcement 可以在内网环境下直接实现 LoadBalancer 类型的 service 。这个功能对 cilium 的版本有一定要求，并且必须使用 kubeProxyReplacement 模式。

```bash
# 可以在之前的步骤中直接开启，也可以在安装 cilium 之后重新启用
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm upgrade cilium cilium/cilium \
    --namespace cilium \
    --reuse-values \
    --set l2announcements.enabled=true
```

启用相关配置后，需要新增两个 cilium crd ，其中 CiliumLoadBalancerIPPool 设定的 CIDR 范围必须是内网可以路由的网段。而 CiliumL2AnnouncementPolicy 则是用来设置 L2 Announcement 具体生效的集群节点。

```yaml
apiVersion: "cilium.io/v2alpha1"
kind: CiliumL2AnnouncementPolicy
metadata:
  name: policy
spec:
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
  interfaces:
    - ens192
  externalIPs: true
  loadBalancerIPs: true
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "lb-ip-pool"
spec:
  cidrs:
    - cidr: "192.168.20.0/24"
```

同时新增一个可供测试的 service 资源。

```yml
apiVersion: v1
kind: Service
metadata:
  name: hubble-ui
  namespace: cilium
spec:
  type: LoadBalancer
  selector:
    k8s-app: hubble-ui
  ports:
    - port: 8081
      targetPort: 8081
```

此时新增的 service 应该带有 EXTERNAL-IP 并且它处在我们赋予的 CiliumLoadBalancerIPPool 范围内，此时子网内的其他设备应该可以直接访问这个 service 。同时也会增加具体的 Lease 资源，可以通过 `kubectl get leases -n cilium` 检查。

## 安装 APISIX Ingress Controller

这里使用的是无需 etcd 的集成模式。

```bash
git clone --depth 1 --branch 1.7.0 https://github.com/apache/apisix-ingress-controller.git ingress-apisix-1.7.0
cd ingress-apisix-1.7.0
kubectl apply -k samples/deploy/crd/v1/
kubectl apply -f samples/deploy/composite.yaml
```
