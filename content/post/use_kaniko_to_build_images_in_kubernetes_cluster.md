---
date: 2023-10-17 11:16:45
title: 使用 Kaniko 在 Kubernetes 集群中构建镜像
tags:
  - "kaniko"
  - "distroless"
  - "container"
  - "docker registry"
draft: false
---

这篇笔记用来记录如何使用 Kaniko 在 Kubernetes 集群中构建镜像。

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

## 使用 kaniko 构建镜像

kaniko 是 Google 开发的镜像构建工具，它的特点在于不需要守护进程，直接运行在容器或者 Kubernetes 集群。

我们可以编写相关的 yaml 文件直接应用到 Kubernetes 集群中。

``` yaml
# from https://github.com/GoogleContainerTools/kaniko
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--dockerfile=<path to Dockerfile within the build context>"
        - "--context=gs://<GCS bucket>/<path to .tar.gz>"
        - "--destination=<gcr.io/$PROJECT/$IMAGE:$TAG>"
      volumeMounts:
        - name: kaniko-secret
          mountPath: /secret
      env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /secret/kaniko-secret.json
  restartPolicy: Never
  volumes:
    - name: kaniko-secret
      secret:
        secretName: kaniko-secret
```

主要通过容器参数来定义构建内容，主要的参数信息参考：

* `--context` ：容器构建的上下文信息，一般就是源代码路径。
* `--dockerfile` ： dockerfile 文件所在路径，一般来说保持 dockerfile 在源代码目录并且命名为 `Dockerfile` 可以不需要这个参数。
* `--destination` ：镜像的命名相关信息。

可以看到这个 pod 还额外挂载了 secret ，这部分其实是用来存放拉取源码和构建完成后推送镜像的一些认证信息。

### 通过 hostPath 进行本地构建

以下是用来做本地测试时的 yaml 文件，需要注意只是用来测试 kaniko 的功能，在实际使用时应该确保 pod 可以正常调度到对应存放着源代码的节点上。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--context=/source/"
        - "--no-push"
        - "--destination=gcr.io/project/image:latest"
        - "--tar-path=/source/image.tar"
      volumeMounts:
        - name: source
          mountPath: /source
  restartPolicy: Never
  volumes:
    - name: source
      hostPath:
        path: /path/to/source/
```

新出现的参数信息参考：

* `--no-push` ：构建镜像完成后不进行推送。
* `--tar-path` ：通过指定路径将镜像通过 tar 文件进行保存。

虽然这里没有涉及 docker 镜像构建中的 `--build-arg` ，但是它在 kaniko 中同样支持，具体用法和 `docker build` 相似。

{{<details "使用 `--build-arg` 的 yaml 文件参考">}}

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      env:
      # for build arg with space
        - name: IFS
          value: ""
      args:
        - "--context=/source/"
        - "--no-push"
        - "--destination=gcr.io/project/image:latest"
        - "--tar-path=/source/image.tar"
        - "--build-arg=ARG_1=VALUE_1"
        - "--build-arg=ARG_2=VALUE_2"
        - "--build-arg=ARG_3='VALUE WITH SPACE'"
      volumeMounts:
        - name: source
          mountPath: /source
  restartPolicy: Never
  volumes:
    - name: source
      hostPath:
        path: /path/to/source/
```

{{</details>}}

编辑完成后直接通过 kubectl 启动 pod 来执行构建。

``` bash
# 保存 yaml 文件后就可以直接执行构建
kubectl apply -f kaniko.yaml

# 可以追踪构建过程
kubectl logs kaniko -f

# 构建完成后应该可以在对应目录找到 image.tar ，将它导入到 containerd 纳管
ctr -n k8s.io i import image.tar
ctr i ls
```

### 构建完成后推送到镜像仓库

这里会涉及本地仓库的搭建，测试环境可以使用 docker registry ，如果是生产环境可以考虑 harbor 。

{{<details "搭建 `docker registry` 的资源文件">}}

``` yaml
# docker registry 
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: registry
  name: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - image: registry:latest
        name: registry
        volumeMounts:
        - name: configfile
          mountPath: /etc/docker/registry/
      volumes:
        - name: configfile
          configMap:
            name: config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: config
data:
  config.yml: |+
    version: 0.1
    log:
      fields:
        service: registry
    storage:
      cache:
        blobdescriptor: inmemory
      filesystem:
        rootdirectory: /var/lib/registry
      delete:
        enabled: true
    http:
      addr: :5000
      headers:
        X-Content-Type-Options: [nosniff]
    health:
      storagedriver:
        enabled: true
        interval: 10s
        threshold: 3
---
apiVersion: v1
kind: Service
metadata:
    name: registry
spec:
  selector:
    app: registry
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

{{</details>}}

docker registry 搭建完成后，我们只需要对前面的本地构建所使用的 yaml 做出一些小修改就可以，主要还是 kaniko 的运行参数：

* 不再需要 `"--no-push"` 。
* 保存 tar 文件的参数 `"--tar-path=/source/image.tar"` 是可选项，按需选择保留或者去除。
* 将目标镜像 `"--destination=gcr.io/project/image:latest"` 修改为 `"--destination=registry:5000/project/image:latest"` 指向本地镜像仓库。
* 由于搭建的 docker registry 没有启用 https ，所以需要增加 `"--insecure"` 参数，这样 kaniko 才能正常进行镜像推送。

{{<details "修改后的 `kaniko` 具体实例">}}

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:latest
      args:
        - "--context=/source/"
        - "--destination=registry:5000/project/image:latest"
        - "--insecure"
      volumeMounts:
        - name: source
          mountPath: /source
  restartPolicy: Never
  volumes:
    - name: source
      hostPath:
        path: /path/to/source/
```

{{</details>}}

以下是一些可能用到的 docker registry api ：

``` bash
# 查看仓库中的镜像信息
curl "<registry:5000>/v2/_catalog"

# 根据仓库给出的镜像信息，可以查到某个镜像所有的 tags 
curl "<registry:5000>/v2/<project/image>/tags/list"

# 删除镜像时需要查询具体签名才能执行
# 使用 docker 进行镜像构建，可能需要使用下面的 header 进行查询
curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "<registry:5000>/v2/<project/image>/manifests/<latest>"
# 使用 kaniko 进行镜像构建，镜像使用的是 oci 标准格式
curl -I -H "Accept: application/vnd.oci.image.manifest.v1+json" "<registry:5000>/v2/<project/image>/manifests/<latest>"

# 取出查询结果的 header ，使用 Docker-Content-Digest 的签名进行删除
curl -X DELETE "<registry:5000>/v2/<project/image>/manifests/<sha256:3354a2f31bfbff0e6e92feb44290f6e261699a0b9ce8c4d390e6ced43588adbe>"

# api 返回成功只是进行了删除标记，并不代表空间回收， docker registry 还需要执行垃圾回收
# 垃圾回收需要在容器中执行
kubectl exec -it registry-7f654d74c6-mq2qt -- registry garbage-collect /etc/docker/registry/config.yml
```

## 使用 distroless 镜像

distroless 系列镜像是谷歌提供的精简镜像，它的安全性和镜像大小都是比较优秀的，可以用来作为基础镜像。

distroless 系列镜像主要有几个类型的基础镜像，应该根据这些特性来选出适合自己应用的类型。

distroless-static 是适用于静态编译运行的镜像，根据官方文档，它只包括了以下资源：

* ca-certificates
* A /etc/passwd entry for a root user
* A /tmp directory
* tzdata

所以在 distroless-static 作为基础镜像时，不再需要更新证书和时区内容，只需要传入对应的 TZ 变量就可以。

distroless-base 在 distroless-static 镜像的基础上增加了 glibc 和 openssl 的支持，还有 distroless-base-nossl 则是只支持 glibc ，这两个镜像有比较强的通用性。可以根据程序的库依赖要求来选择。

distroless-cc 则是在 distroless-base 的基础上支持 libgcc1 and its dependencies ，适用于 Rust 和 D 编写的程序。

另外这些镜像都已经移除了 shell 支持，不过 debug 镜像仍然在原有类型的镜像基础上支持了 busybox shell 。

目前我们正使用这些基础镜像开始替换 Alpine 镜像，在保持镜像轻量的同时提高安全性，使用 go 编写的程序都已经基本迁移完成。

以下是一个适用于 go 程序的 dockerfile 例子：

``` dockerfile
FROM golang:1.20 as build

WORKDIR /src
COPY . .
RUN go mod download \
    && CGO_ENABLED=0 go build -o /src/bin/app

FROM gcr.io/distroless/static-debian12

COPY --from=build /src/bin/app /app
ENTRYPOINT ["/app"]
```
