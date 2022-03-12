---
date: 2020-09-06 21:08:45
title: 使用 GitHub Actions 部署应用
tags:
  - "GitHub"
  - "GitHub Actions"
draft: false
---

通过 GitHub Actions 实现持续集成和持续部署。

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

## 前言

使用 GitHub Actions 之前，我们需要了解持续集成和持续部署的概念。

持续集成 (Continuous integration, CI) 和持续部署 (Continuous deployment, CD) 是随着 DevOps 的兴起而出现的开发工作流程，使用 CI 和 CD 可以省去重复工作，提高工作效率。

CI 指的是频繁地把代码更新提交到主干中，为了实现这一过程，代码的提交需要通过测试才行，而很多时候代码的构建和测试是重复的步骤，所以我们把构建环节和测试环节自动化，省去每次手动操作的麻烦。

CD 建立在 CI 的基础上，将成功集成的主干代码部署到生产环境中，这个过程同样有一些重复性工作，所以这一环节也可以自动化实现。

## 快速使用 GitHub Actions

GitHub Actions 是 GitHub 提供的一项服务，可以通过自定义 workflow 实现代码仓库的 CI 和 CD 工作流程，使用起来快速简单，完成满足个人项目的使用。

使用 GitHub Actions 大概有以下几个步骤：

1. 首先在 GitHub 上建立一个 repository 。
2. 进入你的 repository ，点击 Actions ，直接点击 Setup up this workflow ， workflow 可以选择创建一个新的 workflow ，也可以选择使用 GitHub 社区提供的各种编程环境的 workflow 模板。
3. 将 workflow 文件 commit 到分支中。

提交了 workflow 文件后， GitHub Actions 会根据定义的 workflow 执行相关操作，可以在 Actions 中查看 workflow 的运行结果和运行日志。

## 编写 workflow 文件

除了 GitHub 社区提供的各种编程环境的 workflow ，我们还可以自行编写 workflow 文件。

workflow 文件是使用 YAML 编写的配置文件，它会涉及以下列出的名词：

* workflow : 一个完整的 CI 和 CD 工作流程认为是一个 workflow ，每个 workflow 以后缀命名为 .yml 的文件保存在代码仓库的 .github/workflows 目录中。一个库允许拥有多个 workflow 。

* jobs : 一个 workflow 由一个或多个 job 构成，job 是可以自由命名的，比较常见的命名有 lint ， test ， build ， deploy 等。 job 默认是并行运行的，可以使用 needs 来规定依赖关系以实现顺序运行。

* step : 一个 job 由一个或多个 step 构成， step 是按照顺序关系执行的。 

* action : 一个 step 由一个或多个 action 构成， action 也是按照顺序关系执行的。 action 规定了细节工作，基本上由 shell 命令构成，可以自行编写 action 或使用 GitHub 社区提供的 action 。

YAML 语法比较简单，这里会跳过相关介绍，直接通过实例来学习 workflow 编写。

## python 自动测试实例

以下是 GitHub 社区提供的基于 python 环境的 CI 工作流程。

```
name: Python package

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.5, 3.6, 3.7, 3.8]

    steps:
      - uses: actions/checkout@v2
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v2
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install flake8 pytest
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
      - name: Lint with flake8
        run: |
          # stop the build if there are Python syntax errors or undefined names
          flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
          # exit-zero treats all errors as warnings. The GitHub editor is 127 chars wide
          flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
      - name: Test with pytest
        run: |
          pytest
```

可以看到，这个 workflow 比较简单，只定义了一个 job 。

在 jobs 被定义之前的 on 是 workflow 的关键字，用来定义这个 workflow 的运行触发条件。对于这个文件来说，在分支 master 上的 push 或 pull 动作都会激活这个 workflow 。除了 push 和 pull 两种触发方式，还可以使用 crontab 定时调度触发。

在 jobs 定义 step 之前，还有运行环境的相关定义。 runs-on 关键字用来定义 workflow 的主机工作环境，一般的工作环境有 Linux ， MacOS 和 Windows ，这里使用的是 ubuntu 最新版本。 matrix 是构建矩阵，常用于定义 jobs 运行在多个不同版本的工作环境，这里定义了不同的 python 解释器版本，包括了3.5，3.6，3.7，3.8，所以这个 workflow 会在不同的解释器版本下都运行一次。

主要部分 steps 列表定义了5个成员，在每个成员中， name 用来定义该 action 的名称， run 用来定义该 action 的具体内容， with 用来设置参数和变量， uses 用来调用社区公共 action ，它们是可以灵活组合的。

这个 workflow 的具体 step 完成了以下工作：

1. 调用公共 action checkout@v2 ，我们可以在公共模板看到这个 action 的大量使用，它的工作内容是把 workflow 所在的仓库代码拉取到当前 runs-on 的工作环境中。
2. 调用公共 action setup-python@v2 ，用来设置 python 的工作环境。
3. 自定义 action ，升级 pip ，并且安装测试所需要的模块和代码仓库的依赖。
4. 自定义 action ，使用 flake8 模块来检查代码语法和编写规范。
5. 自定义 action ，使用 pytest 模块来进行单元测试。

除了 python 之外，社区还有对各类开发语言的 workflow 支持，可以根据自己的需要选用。

## Hexo 自动构建部署实例

在很多情况下需要使用自定义 workflow ，以下是我使用过的工作流程，略微比 python 自动测试的 workflow 复杂，用来自动构建并部署静态博客。

```
name: deploy blog
on:
  push:
    branches:
      - master
jobs:
  build:
    name: Build and deploy blog
    env:
      MY_SECRET   : ${{secrets.commit_secret}}
      USER_NAME   : username
      USER_EMAIL  : username@mail.com
      PUBLISH_DIR : ./public
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [10.x]

    steps:
      - uses: actions/checkout@v1
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}
      - name: generate key and identify host 
        run: |
          mkdir ~/.ssh/
          echo "$MY_SECRET" | tr -d '\r'  > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
      - name: install package and build
        run: |
          npm install
          npm run build
      - name: commit and push to remote repository
        run: |
          cd $PUBLISH_DIR
          git init
          git config --local user.name $USER_NAME
          git config --local user.email $USER_EMAIL
          git remote add origin your_remote_repository
          git add --all
          message=$(date '+%Y-%m-%d %H:%M:%S')
          git commit -m "$message UTC"
          git push -f origin master 
          echo deploy complete.
```

workflow 使用 env 关键字在 job 中定义环境变量，在这个文件中，除了明文的环境变量，还使用了 Github 仓库中的 Actions secrets 来设置环境变量。

Actions secrets 是在每个 Github 仓库都可以进行设置，用来存放敏感信息的键值对。在代码仓库的 Settings/Secrets 中添加新的键值对之后，就可以在 workflow 文件中使用`${{secrets.Name}}`进行引用。

Actions secrets 在 workflow 工作过程中是保密的，可以在 workflow 运行日志中观察到这些值由`*`号所保护。

这个 workflow 的具体 step 完成了以下工作：

1. 调用公共 action checkout@v1 ，拉取仓库代码到当前 runs-on 的工作环境中。
2. 调用公共 action setup-node@v1 ，用来设置 nodejs 的工作环境。
3. 自定义 action ，从 Actions secrets 读取密钥内容写入到文件中，然后认证 GitHub 远程主机。这一步 action 是为了实现免密推送，具体的实现方法可以参考之前的[文章](https://yuweizzz.github.io/post/authenticating_to_github/)。为了使这一步 action 执行成功，需要在 Github 账户上传公钥文件，并且把对应的私钥文件添加到仓库的 Settings/Secrets 中。
4. 自定义 action ，安装 nodejs 的依赖包并进行构建，生成静态文件。
5. 自定义 action ，将代码仓库关联到 GitHub 上的运程仓库，做好配置之后 commit 并且 push 代码仓库。

因为在最后一步的 action 中关联的远程仓库设置了 GitHub Pages 服务，所以在 push 之后，应用也就部署完成了。可以看到，整个过程其实是集合了我们平时编译部署需要执行的命令，模拟了一次完整的部署过程。这个 workflow 的最后一步处理还不是很完美，会把原有的 commit 信息破坏，改进后的版本可以参考这个 blog 源代码仓库的 [workflow](https://github.com/yuweizzz/Blog/blob/main/.github/workflows/main.yml) 。

## 总结

GitHub Actions 服务可以很方便实现 CI 和 CD 的工作流程。

Github 社区提供了大量的 workflow 模板，在我个人的角度看，它们大部分时候不适合直接在自己项目中使用，但对我们编写自己的工作流程是非常有用的，比较好的办法是直接引用社区模板，再进行适配修改。

workflow 在使用过程中免不了出错和异常，如果其中一个环节出现错误，整个执行过程就会被中断，这时可以通过 Actions 中的执行日志进行排查。

这篇文章涉及到的工作流程比较简单，有很多 workflow 关键词没有被使用，更多信息可以参考 GitHub Actions 的[官方文档](https://docs.github.com/en/actions)。
