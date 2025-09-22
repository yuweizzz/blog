---
date: 2024-05-09 16:14:45
title: 使用 tekton 在 kubernetes 集群中运行流水线
tags:
  - "Tekton"
  - "Kubernetes"
draft: false
---

这篇笔记用来记录 tekton 在 kubernetes 集群中的基本使用。

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

## 安装 tekton

```bash
# install tekton pipeline
curl https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml -o pipeline.yaml
kubectl apply -f pipeline.yaml

# install tekton dashboard
curl https://storage.googleapis.com/tekton-releases/dashboard/latest/release-full.yaml -o dashboard.yaml
kubectl apply -f dashboard.yaml

# install tekton triggers
curl https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml -o triggers.yaml
curl https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml -o interceptors.yaml
kubectl apply -f triggers.yaml
kubectl apply -f interceptors.yaml
```

## 使用 task 和 taskrun 运行单次任务

最基本的运行资源是由 task 和 taskrun 组成的，比如我们可以通过 tekton hub 提供的 git-clone task ，来执行单次代码克隆。

{{<details "`git-clone.yaml`">}}

```yaml
# git-clone v0.7
# https://hub.tekton.dev/tekton/task/git-clone
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
  labels:
    app.kubernetes.io/version: "0.7"
  annotations:
    tekton.dev/pipelines.minVersion: "0.29.0"
    tekton.dev/categories: Git
    tekton.dev/tags: git
    tekton.dev/displayName: "git clone"
    tekton.dev/platforms: "linux/amd64,linux/s390x,linux/ppc64le,linux/arm64"
spec:
  description: >-
    These Tasks are Git tasks to work with repositories used by other tasks
    in your Pipeline.

    The git-clone Task will clone a repo from the provided url into the
    output Workspace. By default the repo will be cloned into the root of
    your Workspace. You can clone into a subdirectory by setting this Task's
    subdirectory param. This Task also supports sparse checkouts. To perform
    a sparse checkout, pass a list of comma separated directory patterns to
    this Task's sparseCheckoutDirectories param.
  workspaces:
    - name: output
      description: The git repo will be cloned onto the volume backing this Workspace.
    - name: ssh-directory
      optional: true
      description: |
        A .ssh directory with private key, known_hosts, config, etc. Copied to
        the user's home before git commands are executed. Used to authenticate
        with the git remote when performing the clone. Binding a Secret to this
        Workspace is strongly recommended over other volume types.
    - name: basic-auth
      optional: true
      description: |
        A Workspace containing a .gitconfig and .git-credentials file. These
        will be copied to the user's home before any git commands are run. Any
        other files in this Workspace are ignored. It is strongly recommended
        to use ssh-directory over basic-auth whenever possible and to bind a
        Secret to this Workspace over other volume types.
    - name: ssl-ca-directory
      optional: true
      description: |
        A workspace containing CA certificates, this will be used by Git to
        verify the peer with when fetching or pushing over HTTPS.
  params:
    - name: url
      description: Repository URL to clone from.
      type: string
    - name: revision
      description: Revision to checkout. (branch, tag, sha, ref, etc...)
      type: string
      default: ""
    - name: refspec
      description: Refspec to fetch before checking out revision.
      default: ""
    - name: submodules
      description: Initialize and fetch git submodules.
      type: string
      default: "true"
    - name: depth
      description: Perform a shallow clone, fetching only the most recent N commits.
      type: string
      default: "1"
    - name: sslVerify
      description: Set the `http.sslVerify` global git config. Setting this to `false` is not advised unless you are sure that you trust your git remote.
      type: string
      default: "true"
    - name: crtFileName
      description: file name of mounted crt using ssl-ca-directory workspace. default value is ca-bundle.crt.
      type: string
      default: "ca-bundle.crt"
    - name: subdirectory
      description: Subdirectory inside the `output` Workspace to clone the repo into.
      type: string
      default: ""
    - name: sparseCheckoutDirectories
      description: Define the directory patterns to match or exclude when performing a sparse checkout.
      type: string
      default: ""
    - name: deleteExisting
      description: Clean out the contents of the destination directory if it already exists before cloning.
      type: string
      default: "true"
    - name: httpProxy
      description: HTTP proxy server for non-SSL requests.
      type: string
      default: ""
    - name: httpsProxy
      description: HTTPS proxy server for SSL requests.
      type: string
      default: ""
    - name: noProxy
      description: Opt out of proxying HTTP/HTTPS requests.
      type: string
      default: ""
    - name: verbose
      description: Log the commands that are executed during `git-clone`'s operation.
      type: string
      default: "true"
    - name: gitInitImage
      description: The image providing the git-init binary that this Task runs.
      type: string
      default: "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.29.0"
    - name: userHome
      description: |
        Absolute path to the user's home directory. Set this explicitly if you are running the image as a non-root user or have overridden
        the gitInitImage param with an image containing custom user configuration.
      type: string
      default: "/tekton/home"
  results:
    - name: commit
      description: The precise commit SHA that was fetched by this Task.
    - name: url
      description: The precise URL that was fetched by this Task.
  steps:
    - name: clone
      image: "$(params.gitInitImage)"
      env:
        - name: HOME
          value: "$(params.userHome)"
        - name: PARAM_URL
          value: $(params.url)
        - name: PARAM_REVISION
          value: $(params.revision)
        - name: PARAM_REFSPEC
          value: $(params.refspec)
        - name: PARAM_SUBMODULES
          value: $(params.submodules)
        - name: PARAM_DEPTH
          value: $(params.depth)
        - name: PARAM_SSL_VERIFY
          value: $(params.sslVerify)
        - name: PARAM_CRT_FILENAME
          value: $(params.crtFileName)
        - name: PARAM_SUBDIRECTORY
          value: $(params.subdirectory)
        - name: PARAM_DELETE_EXISTING
          value: $(params.deleteExisting)
        - name: PARAM_HTTP_PROXY
          value: $(params.httpProxy)
        - name: PARAM_HTTPS_PROXY
          value: $(params.httpsProxy)
        - name: PARAM_NO_PROXY
          value: $(params.noProxy)
        - name: PARAM_VERBOSE
          value: $(params.verbose)
        - name: PARAM_SPARSE_CHECKOUT_DIRECTORIES
          value: $(params.sparseCheckoutDirectories)
        - name: PARAM_USER_HOME
          value: $(params.userHome)
        - name: WORKSPACE_OUTPUT_PATH
          value: $(workspaces.output.path)
        - name: WORKSPACE_SSH_DIRECTORY_BOUND
          value: $(workspaces.ssh-directory.bound)
        - name: WORKSPACE_SSH_DIRECTORY_PATH
          value: $(workspaces.ssh-directory.path)
        - name: WORKSPACE_BASIC_AUTH_DIRECTORY_BOUND
          value: $(workspaces.basic-auth.bound)
        - name: WORKSPACE_BASIC_AUTH_DIRECTORY_PATH
          value: $(workspaces.basic-auth.path)
        - name: WORKSPACE_SSL_CA_DIRECTORY_BOUND
          value: $(workspaces.ssl-ca-directory.bound)
        - name: WORKSPACE_SSL_CA_DIRECTORY_PATH
          value: $(workspaces.ssl-ca-directory.path)
      script: |
        #!/usr/bin/env sh
        set -eu

        if [ "${PARAM_VERBOSE}" = "true" ] ; then
          set -x
        fi


        if [ "${WORKSPACE_BASIC_AUTH_DIRECTORY_BOUND}" = "true" ] ; then
          cp "${WORKSPACE_BASIC_AUTH_DIRECTORY_PATH}/.git-credentials" "${PARAM_USER_HOME}/.git-credentials"
          cp "${WORKSPACE_BASIC_AUTH_DIRECTORY_PATH}/.gitconfig" "${PARAM_USER_HOME}/.gitconfig"
          chmod 400 "${PARAM_USER_HOME}/.git-credentials"
          chmod 400 "${PARAM_USER_HOME}/.gitconfig"
        fi

        if [ "${WORKSPACE_SSH_DIRECTORY_BOUND}" = "true" ] ; then
          cp -R "${WORKSPACE_SSH_DIRECTORY_PATH}" "${PARAM_USER_HOME}"/.ssh
          chmod 700 "${PARAM_USER_HOME}"/.ssh
          chmod -R 400 "${PARAM_USER_HOME}"/.ssh/*
        fi

        if [ "${WORKSPACE_SSL_CA_DIRECTORY_BOUND}" = "true" ] ; then
           export GIT_SSL_CAPATH="${WORKSPACE_SSL_CA_DIRECTORY_PATH}"
           if [ "${PARAM_CRT_FILENAME}" != "" ] ; then
              export GIT_SSL_CAINFO="${WORKSPACE_SSL_CA_DIRECTORY_PATH}/${PARAM_CRT_FILENAME}"
           fi
        fi
        CHECKOUT_DIR="${WORKSPACE_OUTPUT_PATH}/${PARAM_SUBDIRECTORY}"

        cleandir() {
          # Delete any existing contents of the repo directory if it exists.
          #
          # We don't just "rm -rf ${CHECKOUT_DIR}" because ${CHECKOUT_DIR} might be "/"
          # or the root of a mounted volume.
          if [ -d "${CHECKOUT_DIR}" ] ; then
            # Delete non-hidden files and directories
            rm -rf "${CHECKOUT_DIR:?}"/*
            # Delete files and directories starting with . but excluding ..
            rm -rf "${CHECKOUT_DIR}"/.[!.]*
            # Delete files and directories starting with .. plus any other character
            rm -rf "${CHECKOUT_DIR}"/..?*
          fi
        }

        if [ "${PARAM_DELETE_EXISTING}" = "true" ] ; then
          cleandir
        fi

        test -z "${PARAM_HTTP_PROXY}" || export HTTP_PROXY="${PARAM_HTTP_PROXY}"
        test -z "${PARAM_HTTPS_PROXY}" || export HTTPS_PROXY="${PARAM_HTTPS_PROXY}"
        test -z "${PARAM_NO_PROXY}" || export NO_PROXY="${PARAM_NO_PROXY}"

        /ko-app/git-init \
          -url="${PARAM_URL}" \
          -revision="${PARAM_REVISION}" \
          -refspec="${PARAM_REFSPEC}" \
          -path="${CHECKOUT_DIR}" \
          -sslVerify="${PARAM_SSL_VERIFY}" \
          -submodules="${PARAM_SUBMODULES}" \
          -depth="${PARAM_DEPTH}" \
          -sparseCheckoutDirectories="${PARAM_SPARSE_CHECKOUT_DIRECTORIES}"
        cd "${CHECKOUT_DIR}"
        RESULT_SHA="$(git rev-parse HEAD)"
        EXIT_CODE="$?"
        if [ "${EXIT_CODE}" != 0 ] ; then
          exit "${EXIT_CODE}"
        fi
        printf "%s" "${RESULT_SHA}" > "$(results.commit.path)"
        printf "%s" "${PARAM_URL}" > "$(results.url.path)"
```

{{</details>}}

添加 task 后，需要通过 taskrun 对这个 task 进行引用，除了 taskrun 之外还需要配置其他的 resource 和 Git clone 密钥。

```yaml
# taskrun.yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: git-clone-
spec:
  taskRef:
    name: git-clone
  params:
    - name: url
      value: git@github.com:yuweizzz/devops-tools.git
  workspaces:
    - name: output
      persistentVolumeClaim:
        claimName: output-pvc
      subPath: repo
    - name: ssh-directory
      secret:
        secretName: deploy-ssh-credentials
```

执行以下步骤：

```bash
# 用来存放代码内容的 pvc 资源
$ cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: output-pvc
spec:
  resources:
    requests:
      storage: 100Mi
  volumeMode: Filesystem
  storageClassName: local-path
  accessModes:
    - ReadWriteOnce

# id_rsa
# 可以直接使用已有密钥，如果需要重新生成则要在 git 服务端重新添加新生成的私钥
$ ssh-keygen -t rsa -C "Deploy Key" -P "" -f id_rsa

# config
# ssh config 配置文件可以管理具体的连接主机和对应密钥
$ cat config
Host gitlab
    HostName 192.168.2.1
    Port 22
    User git
    IdentityFile ~/.ssh/id_rsa

# 加载资源，其中 ssh config 可以不生成，但 id_rsa 是必须的
kubectl apply -f pvc.yaml
kubectl apply -f git-clone.yaml
kubectl create secret generic deploy-ssh-credentials --from-file=id_rsa=id_rsa --from-file=config=config

# 执行 taskrun ，使用的是 create 而不是 apply
kubectl create -f taskrun.yaml
```

等待 taskrun 执行完成，在正常成功的情况下对应 Pod 应该处于 Completed 状态。

## 使用 pipeline 和 pipelinerun 运行流水线任务

pipeline 是基于 task 编排组成的流水线，而 pipelinerun 是 pipeline 的具体执行，就像 task 与 taskrun 之间的关系一样。

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: my-pipeline
spec:
  params:
    - name: git-url
  workspaces:
    - name: source-code
    - name: ssh-directory
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
      workspaces:
        - name: output
          workspace: source-code
        - name: ssh-directory
          workspace: ssh-directory
    - name: check
      taskSpec:
        steps:
          - name: ls-file
            image: cgr.dev/chainguard/busybox:latest
            script: ls $(workspaces.source-code.path)
      runAfter:
        - clone
```

这个 pipelinerun 中的 output workspace 不再使用固定的 PersistentVolumeClaim 资源，而是声明了 Template 并在每次运行时遵循这个 Template 来创建新的 PersistentVolumeClaim 资源。而 Secret 则延用了原先的 deploy-ssh-credentials 。

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: clone-and-check-
spec:
  params:
    - name: git-url
      value: git@github.com:yuweizzz/devops-tools.git
  pipelineRef:
    name: my-pipeline
  taskRunTemplate:
    serviceAccountName: default
  timeouts:
    pipeline: 1h0m0s
  workspaces:
    - name: source-code
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 50Mi
          storageClassName: local-path
          volumeMode: Filesystem
    - name: ssh-directory
      secret:
        secretName: deploy-ssh-credentials
```

## 使用 trigger 触发 pipelinerun

tekton trigger 可以提供相关的 web 接口，使得外部服务可以通过这个接口来触发对应的 pipelinerun 。

具体的 tekton trigger 由 TriggerTemplate ， TriggerBinding 以及 EventListener 三个部分组成。其中 TriggerTemplate 和 pipeline 定义密切相关，基本上是在 pipeline 的基础上进行额外包装，而 TriggerBinding 则是规定了具体的 JSON payload 格式，应该要与 TriggerTemplate 中的参数结构对应。最终由 EventListener 对外提供 web 接口，接收来自外部的请求。

此外还可以通过 Interceptors 对请求内容进行一些校验和转换处理，但这里暂时只用到了简单的参数映射，所以没有用上 Interceptors 。

```yaml
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: my-triggertemplate
spec:
  params:
    - name: git-url
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: my-pipeline-run-
      spec:
        pipelineRef:
          name: my-pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
        workspaces:
          - name: source-code
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce
                resources:
                  requests:
                    storage: 50Mi
                storageClassName: local-path
                volumeMode: Filesystem
          - name: ssh-directory
            secret:
              secretName: deploy-ssh-credentials
---
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: my-triggerbinding
spec:
  params:
    - name: git-url
      value: $(body.git-url)
---
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: my-eventlistener
spec:
  serviceAccountName: my-tekton-triggers-sa
  triggers:
    - bindings:
        - ref: my-triggerbinding
      template:
        ref: my-triggertemplate
```

因为执行过程会触发资源创建，所以需要赋予特定的 rbac 权限， tekton triggers 安装时默认生成了相关的 ClusterRole ，我们只需要创建对应的 ServiceAccount 来引用它们就可以了。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-tekton-triggers-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-eventlistener-rolebinding
subjects:
  - kind: ServiceAccount
    name: my-tekton-triggers-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-roles
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-eventlistener-clusterrolebinding
subjects:
  - kind: ServiceAccount
    name: my-tekton-triggers-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-clusterroles
```

{{<details "具体 ClusterRole 的 YAML 文件参考">}}

```yaml
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-triggers
  name: tekton-triggers-eventlistener-roles
rules:
  - apiGroups:
      - triggers.tekton.dev
    resources:
      - eventlisteners
      - triggerbindings
      - interceptors
      - triggertemplates
      - triggers
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - tekton.dev
    resources:
      - pipelineruns
      - pipelineresources
      - taskruns
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - serviceaccounts
    verbs:
      - impersonate
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-triggers
  name: tekton-triggers-eventlistener-clusterroles
rules:
  - apiGroups:
      - triggers.tekton.dev
    resources:
      - clustertriggerbindings
      - clusterinterceptors
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - list
      - watch
```

{{</details>}}

因为没有主动声明 Type 的 EventListener 默认会以 ClusterIP 类型的 Service 创建，所以我们可以直接通过命令行执行 `curl http://<CLUSTER-IP>:<PORT> -d '{"git-url": "git@github.com:yuweizzz/devops-tools.git"}'` 就可以触发 pipelinerun 。
