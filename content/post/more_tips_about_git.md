---
date: 2020-08-13 22:00:00
title: Git 命令进阶使用笔记
tags:
  - "Git"
  - "GitHub"
draft: false
---

这篇笔记用来记录一些 Git 命令的进阶使用和部分原理知识。

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

在本地环境的 Git 使用熟练后，我们会使用到一些更复杂的功能，协助我们更好地工作。

## 了解 Git 原理

在使用更复杂的功能前，简单了解一下 Git 的原理会对后续使用有帮助。

Git 本质上是一个内容寻址的文件系统，它会为对象计算一个索引值，并凭借这个唯一值去获取对象内容。 Git 对象一共有三种：

* blob object
* tree object
* commit object

当仓库初始化并提交第一次 commit ，Git 会根据仓库情况自动创建这些对象：

* blob object 对应着文件，用来保存代码的详细内容。
* tree object 对应着文件目录，用来保存目录结构和对应目录下的 blob 对象， tree 对象可以拥有子 tree 对象和 blob 对象，但是 blob 对象只能拥有一个父 tree 对象。
* commit object 对应着 commit ，用来保存本次 commit 对应的 tree object 和 parent commit object ，此外还会保存 commit 的用户配置和注释信息。


每个 commit object 和其他 object 的关系一般是这样的：

```
                                          +-------------+
                                       +->+ blob object |
                                       |  +-------------+
 +---------------+    +-------------+  |  +-------------+
 | commit object +--->+ tree object +---->+ blob object |
 +---------------+    +-------------+  |  +-------------+
                                       |  +-------------+   +-------------+
                                       +->+ tree object |-->+ blob object |
                                          +-------------+   +-------------+
```

每一次 commit 会产生新的 commit object ，通过各自的 parent commit object 可以形成一条 commit 链。

```
     father              children             children              
+---------------+    +---------------+    +---------------+
| commit object +<---+ commit object +<---+ commit object |
+---------------+    +---------------+    +---------------+
        |                    |                    |
        v                    v                    v
 +-------------+      +-------------+      +-------------+
 + tree object +      + tree object +      + tree object +
 +-------------+      +-------------+      +-------------+
        |                    |                    |
        v                    v                    v
       ...                  ...                  ...
```

通过整个 commit 链，我们就可以追溯整个仓库的所有历史变动。

## branch 的工作原理

在前面的使用过程中，我们接触了分支 branch 的概念了，也知道每个 Git 仓库会自动创建一个 master/main 分支，而它本质上是一个指向 commit object 的指针。

虽然 Git 会默认创建 master/main 分支，但是对比其他分支，它并没有优先级的区别，只是在使用习惯上的差别。我们一般以 master/main 为主线，其他分支为辅来进行开发。

```
                                            +------------+
                                            | new branch |
                                            +-----+------+
                                                  |
     father              children                 v
+---------------+    +---------------+    +-------+-------+
| commit object +<---+ commit object +<---+ commit object |
+---------------+    +---------------+    +-------+-------+
                                                  ^
                                                  |
                                              +---+----+    +------+
                                              | master +<---+ HEAD |
                                              +--------+    +------+
```

分支在 Git 中的更底层描述是引用 reference ，一般简写为 refs 。经常和 refs 联系在一起的还有 HEAD 的这个概念，通过前面的使用我们知道 HEAD 用来指向当前分支的最新 commit ，这样的描述和 refs 非常接近，当分支切换时， HEAD 会自动指向切换后分支的最新 commit 。但实际上， HEAD 只是 refs 的符号引用，我们将通过实例来说明。 

``` bash
# 以任意一个仓库为例，可以看到当前仓库有两个分支
$ git branch
* branchA
  main

# 查看仓库已拥有的 refs
$ ls .git/refs/heads/
branchA  main

# 查看 refs 指向的 object 类型
$ git cat-file -t `cat .git/refs/heads/main`
commit  # 指向的正是 commit

# 查看 refs 指向的 object 的具体内容
$ git cat-file -p `cat .git/refs/heads/main`
# 命令应该会输出某个 commit 的具体信息，我们可以通过对比 git log 得到它是当前分支的最新 commit 

# 查看 HEAD 的内容
$ cat .git/HEAD 
ref: refs/heads/branchA
# 可以看到它是 refs 的符号引用
```

branch ， refs ， HEAD 三者之间有一个共同点，那就是它们的本质都是用来指向 commit object ， refs 是 branch 的低级原语，而 HEAD 是 refs 的符号引用，它是为了使用方便引出的概念，每次进行 branch 切换， HEAD 也会自动更新。

## 使用 remote 功能

除了在本地设备上使用之外， Git 更强大的地方在于远端功能的支持，我们可以把仓库托管到云端，这个云端可以是 GitHub 或者自建的 GitLab 平台，这样我们就可以随时获得最新版本的代码仓库，使得团队协作开发更方便。

在使用远端功能之前，需要确认 GitHub 或者自建的 GitLab 平台的账号鉴权是否完成，可以参考之前的[配置 GitHub 免密认证的方法](https://yuweizzz.github.io/post/authenticating_to_github/)和[通用的账号配置方法](https://yuweizzz.github.io/post/tips_about_git/#%E5%88%9B%E5%BB%BA%E5%B9%B6%E9%85%8D%E7%BD%AE%E6%9C%AC%E5%9C%B0%E4%BB%93%E5%BA%93)。

```
# 本地仓库和远端仓库的基本关系:

# for new repository:

       full         remote add       empty
+------------------+--------->+-------------------+
| local repository |          | remote repository |
+------------------+<---------+-------------------+
       empty          clone           full

# for existing repository:

     new commit        push       
+------------------+--------->+-------------------+
| local repository |          | remote repository |
+------------------+<---------+-------------------+
                       pull        new commit
```

在本地仓库和远端仓库的交互中，出现了几个新的概念： clone ， pull 和 push ，它们同时也是 Git 的子命令：

* clone ： 用来将远端仓库复制到本地。

* pull ： 用来更新本地仓库，将远端仓库的新变动拉取到本地。

* push ： 用来更新远端仓库，将本地仓库的新变动推送到远端。

我们通过具体实际操作来熟悉它们。

``` bash
# 查看已配置的远程仓库
$ git remote -v
origin  git@github.com:username/repository.git (fetch)
origin  git@github.com:username/repository.git (push)
# 每个本地仓库允许配置多个远程仓库，默认的远程仓库会是 origin 


# 将远端仓库下载到本地
# 远端仓库的地址信息一般可以直接在对应网页复制
$ git clone git@github.com:username/repository.git
# clone 可以指定明确的远端仓库中任一分支，不指定默认为 master/main
$ git clone -b branchA git@github.com:username/repository.git


# 将本地仓库关联到远端仓库并推送更新
# 将现存的本地仓库关联到全新的远程仓库
$ git remote add origin git@github.com:username/repository.git
# 设置分支关联并推送本地仓库的 commit 到远端仓库中
$ git push -u origin master
# -u 是关键的参数，它将 origin/master 分支和本地的 master 分支关联，它们才能使用 pull 和 push 
# 这一步可以根据情况使用下面命令进行替代
# 使用 checkout 创建全新分支并设置上游分支
$ git checkout -b mybranch origin/mybranch
# 使用 set upstream 设置本地分支的上游分支
$ git checkout mybranch
$ git branch -u/--set-upstream-to origin/mybranch
# 完成后，后续直接使用 git push 即可
$ git push


# 拉取远端仓库领先于本地仓库的 commit 同时合并到本地仓库的当前分支中
$ git pull
# 一般情况下它的效果会相当于 git pull origin master:master ， git pull 正常工作的前提是 remote 和 upstream 已经正常设置
# 除了这种默认情况，也可以显式声明使用其他远端仓库和需要拉取合并的分支

# 实际上 pull 是通过 fetch 和 merge 协助完成的，可以通过下面命令来替代
# 这个步骤用来查看上游分支设置
$ git branch -vv
* main 3f14e1e [origin/main] Update README.md
# fetch 指定了 origin master ，所以它会拉取 origin/master 这个分支
$ git fetch origin master
# 将 origin/master 分支 merge 到当前分支，也就是 master 分支
$ git merge origin/master
```

## 使用 submodule 功能

submodule 可以让你的代码仓库使用其他现有的仓库作为子模块，这样我们可以轻松引用别人的仓库作为我们自己代码仓库的一部分。

它的特色在于二者的文件管控是分离的，子模块在父仓库的 git 管控中显示为一个对象，但是它拥有独立的 git 目录。这样父仓库和子仓库就可以拥有各自 commit ，但同时又可以正确地关联二者之间的从属关系。

``` bash
# 为现有仓库添加子模块
$ git submodule add git@github.com:username/repository.git subrepository
# add 会将远端仓库 repository 克隆到本地 subrepository 目录中，并命名子模块为 subrepository 

# 在父仓库的管控域查看新增的子模块
$ git status
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

    new file:   .gitmodules
    new file:   subrepository  # 可以看到，父仓库将子模块视为一个文件对象

# 进入子模块的管控域，可以看到它和普通的仓库是相同的
$ cd subrepository
# 可以看到子模块的最后一次 commit
$ git show
# 可以看到子模块的 commit log
$ git log
```

如果引用了一个由他人维护开发的仓库作为自己仓库的依赖，这种情况是比较常见的，而且我们可能需要对这个仓库作一些特殊定制，比如这个仓库中静态博客的主题就做了部分定制，这种情况下需要做一些额外设置。

```bash
# 首先在 Github 上自行 fork 需要使用到的第三方仓库，这一步直接在 Github 或者 GitLab 上点击目标仓库的 fork 即可

# 进入本地父仓库，自行添加 fork 的仓库作为子模块
$ git submodule add git@github.com:username/repository.git

# 由于这个子模块已经 fork 到我们自己的仓库中，所以可以无顾忌地做自定义的修改，它应该只会被你的父仓库所使用
# 和普通仓库一样进行修改操作
$ cd repository
$ vi xxxx
$ git add xxxx
$ git commit -m "add file xxxx"

# 接下来进行额外仓库的关联设置，帮助我们后续使用
# 首先检查原有配置
$ git remote -v
origin  git@github.com:username/repository.git (fetch)
origin  git@github.com:username/repository.git (push)
# 目前只指向了 fork 的远端仓库，我们把实际的仓库来源额外添加到配置中
$ git remote add source git@github.com:another_username/repository.git
# 再次检查配置，此时应该出现新的配置项
$ git remote -v
origin  git@github.com:username/repository.git (fetch)
origin  git@github.com:username/repository.git (push)
source  git@github.com:another_username/repository.git (fetch)
source  git@github.com:another_username/repository.git (push)

# 此时我们的子模块已经被自行修改过一次，但是源仓库有了新的 commit ，我们需要把它同步过来
# 直接拉取来源仓库并直接合并，这个过程有可能需要修复版本冲突
# 如果不需要修复冲突，这个命令会自动 merge 成为新的 commit 
$ git pull source master

# 将新的变动同步到自己的 fork 仓库中，这一步不是必须的，可以按情况执行
$ git push origin master

# 回到父仓库，子模块会被记录为拥有新变动，父仓库需要自行提交 commit
```

这个操作同时也适用于普通仓库，进行额外关联配置后，虽然需要执行的操作变多了，但是对后续维护更方便。
