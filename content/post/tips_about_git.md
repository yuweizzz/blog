---
date: 2020-07-23 22:00:00
title: 常用 Git 命令笔记
tags:
  - "Git"
draft: false
---

这篇笔记用来记录一些常用的 Git 命令。

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

## 前言

使用 Git 可以有条理地管理代码，尤其是在大型项目和团队开发的场合，熟练使用 Git 是开发人员的基本功。

想要熟练使用 Git 的最好办法是了解 Git 的基本原理和实践。

## 快速了解 Git 术语

首先了解几个基本概念：

- 仓库 (repository) ：仓库是一个被 Git 管控的目录。
- 分支 (branch) ：每个仓库一般有一个默认分支 master/main ，并且可以拥有多个分支。
- 提交 (commit) ：可以视为整个代码仓库的不同版本，提交会推进仓库的版本更新。

## 创建并配置本地仓库

```bash
cd repository
git init
```

这样 repository 目录就是一个 Git 本地仓库，后续用来添加项目代码文件。

初始化仓库后，仓库会带有一些默认的配置，我们可以根据需要去查询或者修改。

```bash
# config -l用来查看仓库属性
# 如果要使用 github 的远程仓库，至少应该为仓库配置用户名和邮箱
# --global 和 --local 可以用来设定属性是 git 全局属性还是单一仓库属性
$ git config -l
$ git config [--global|--local] user.name "your name"
$ git config [--global|--local] user.email "your@email.com"
```

## 转换仓库中文件的状态

下面是一个 Git 仓库实例，基本覆盖了文件状态之间的转换：

```bash
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#   modified:   fileA
#
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#   (commit or discard the untracked or modified content in submodules)
#
#   modified:   fileB
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#   fileC

```

这里引入了 Git 仓库**工作区**和**暂存区**的概念：

- 上面的仓库包含了多种文件，FileA ，FileB ，FileC 处于同一个工作区，工作区 (workspace) 可以简单认为是仓库所在目录，其中存放了各种不同状态的文件。

- 暂存区 (index) 是一个抽象概念，它的信息会保存在实际文件 (.git/index) 中，我们修改过的文件，就可以暂存 (staged) 到暂存区中，也是 Changes to be committed 这一部分，它的下一步操作通常会是提交 (commit) 。

下面进入实例的分析：

- Changes to be committed 列出的是已经被暂存的 FileA ，它已经由 not staged 转换为 staged 。下一步它可以提交到分支中，或者使用 reset 重置掉这次 modified 。
- Changes not staged for commit 列出的是在工作区中被修改过，但是尚未被 staged 的 FileB 。下一步它可以使用 add 更新到暂存区，或者使用 checkout 丢弃本次修改，恢复到上次 add 的暂存区版本或 commit 保存的版本，恢复的版本由最近的一次的修改情况决定。
- Untracked 的 FileC 目前是不受 Git 管制的，但是可以通过 add 转换状态。

以下是实际命令的使用过程：

### 添加和删除文件

添加文件和暂存文件都使用了 add 命令，删除文件使用 rm 命令。

通过删除文件，我们可以把文件排除出工作区 (untracked) ，如果是某些特定种类的文件需要排除，推荐使用 .gitignore 。

```bash
$ git add <file>
# add 用于 tracked 和 staged

$ git rm [-f] [--cached] [--] <file>
# rm 用来移除文件
# rm 带 -f 选项会删除工作区和暂存区文件，即本地文件也会被删除
# rm 带 --cached 选项会删除暂存区文件，工作区文件不会被改变
# -- 选项用来把操作限制在当前分支
```

### 文件的重置

需要丢弃不想要的修改，可以使用 reset 命令或者 checkout 命令。

```bash
$ git reset HEAD <file>
# reset 指令涉及 git 的实现原理，HEAD 是指向当前分支最新 commit 的指针
# 这里只能用于回退暂存区文件，后面会有涉及这一步的解释

$ git checkout -- <file>
# checkout 其实是用于分支的命令，-- 用来把操作限制在当前分支
# 这里用于将文件恢复到上次 add 的暂存区版本或 commit 保存的版本
# 这个命令会改变实际文件，有一定风险
```

## 提交到新版本

文件修改，重置，暂存一系列操作，都是为了将代码修改到我们认可的程度，然后 commit 到分支中，产生新的版本库，一次或多次的 commit 可以视为一个新的版本库。

```bash
$ git commit -m <msg>
# 提交暂存区的文件到版本库， -m 为提交的说明信息

$ git commit --amend [<msg>]
# 修改上次 commit ，可以添加提交暂存区文件并修改上次 commit 说明信息
```

这样所有 staged 的文件就会被提交到一个新的版本库中了。我们还会得到一个 commit ID ，这是非常重要的依据，后续版本库的管理都是基于 commit ID 来实现的。

带 --amend 的 commit 指令可以在一次提交后，追加位于暂存区的文件变动。使用这个命令后只会有一个提交，本次带 --amend 的提交将代替上一次提交的结果。如果暂存区是干净的，可以使用这个命令修改上次 commit 的说明信息。

## 对 commit 进行检查

在经过一定的开发时间后，仓库必然会产生很多 commit ，可以使用 log 和 diff 进行检查。

```bash
$ git log
# 查看 commit 历史记录，可以看到简要的提交信息

$ git show <object>
# 用来查看 commit 的具体变动，默认为最近一次 commit 的详细信息
# 可以通过指定 commit id 来查看不同 commit

$ git reflog
# 类似于 bash history，它会输出 commit 操作的历史记录，可以作为版本回退和误操作恢复的依据

$ git diff <commit> <commit> [--] [<path>...]
# 用来查看不同 commit 之间的所有文件的变动，可以指定单个文件

$ git diff --cached [<commit>] [--] [<path>...]
# 除了用于 commit 外，还可以用来查看暂存区和工作区的文件变动
```

## 回退 commit

如果需要撤消 commit ，恢复原有的代码版本，需要用到 reset 指令。

在 reset 命令中，我们经常会看到 HEAD 的使用，它用来指向当前分支的最新 commit ，所以最新 commit 的上一个版本使用 HEAD^ 表示，上上一个版本使用 HEAD^^ 表示，这是一种便捷写法，使用对应 commit ID 也是一样的效果。

```bash
# reset 的使用参数
$ git reset [--mixed | --soft | --hard | --merge | --keep] [-q] [<commit>]
# --mixed               reset HEAD and index
# --soft                reset only HEAD
# --hard                reset HEAD, index and working tree
# --merge               reset HEAD, index and working tree
# --keep                reset HEAD but keep local changes
# -q, --quiet           be quiet, only report errors
```

可以看到 reset 有五种工作模式，但我在使用的时候一般只用到 mixed ， soft 和 hard 三种：

- mixed ：默认的工作模式，这个模式会影响 commit 和暂存区保存的文件变动。
- soft ：这个模式的影响对象是 commit ，所以实际效果看起來就只有 commit 的消失，也就是 HEAD 指向了更旧的 commit 。 commit 回退后，文件变化会存放在暂存区。
- hard ：这个模式下， commit ，工作区，暂存区的文件变动都会丢失。

在前面的实例中 `git reset HEAD <file>` 就是 mixed 模式的应用，它用来回退 staged 文件到 not staged 的状态。因为指定了 HEAD ，所以文件变化还是留存在工作区，暂存区被清空， commit 无变化。

soft 模式是最安全的，它不会影响实际文件，而 hard 模式是最为危险，在没有确定你需要丢弃你新写的代码，最好不要使用。

此外还有一个值得注意的地方， reset 命令可以应用于单个文件，但仅限于 mixed 模式， hard 模式和 soft 模式均无法对单文件作用。

如果需要撤销 commit ，但不具体恢复到某个原有的代码版本，需要用到 revert 指令。

revert 的使用方法类似于 reset ，但它不会改变 commit 的历史记录，而是在最新 commit 的基础上，将所指定 commit 的文件变动还原，并将这个过程提交为一个新的 commit 。

```bash
# revert 的使用参数
$ git revert <commit>
# 执行后将会产生新的 commit 的提交，默认将以 revert 信息作为 commit 信息
```

## 分支管理

简单项目可能没有使用到多分支，但如果是大型项目和团队开发项目，一般都会有多分支，多分支的好处在于开发过程中，各分支之间互不干扰，代码维护起来方便简单。

分支创建，分支删除和分支合并是分支管理比较常用的操作，其中分支合并应该是多分支场景最重要的操作，当不同分支上的开发进行到一定程度时，可以将所涉及的文件变化都合并到同个分支中，然后这个分支就拥有了最新版本的文件。

```bash
# git branch 会列出当前已有分支，并以 * 标识目前所在的分支
$ git branch
* master
  branchA
  branchB

# 创建一个新的分支。
$ git branch <branch name>

# 创建和切换到目的分支。
$ git checkout -b <branch name>
# -b 切换到目的分支，如果目的分支不存在，则会创建它后再执行切换

# 删除一个分支
$ git branch [ -d | -D ] <branch name>
# -d 删除分支时还会检查分支是否已经合并，如果未合并，那么删除不会执行
# -D 和 -d 的区别在于不检查分支的合并情况，直接执行删除

# 合并分支
# 需要先切换到旧的分支中
$ git checkout <old branch name>
# 然后将新改动的分支合并到旧的分支中
$ git merge <new branch name>
# 最理想的情况是 Fast-forward ，如果不同分支中的改动有冲突，则需要手动解决后才能合并
```

## 工作区改动入栈暂存

如果我们需要临时切换到其他分支，而不想对现有工作区进行暂存和提交，希望它保持原状，可以使用 stash 来将工作区入栈保存。

```bash
$ git stash
# 保存暂存区和工作区的文件改动

$ git stash list
# 显示入栈暂存的情况

$ git stash pop
# 恢复已入栈的文件改动到工作区，默认会把工作区和暂存区的改动都恢复到工作区
$ git stash pop --index
# 在默认的基础上，除了工作区的文件改动，还会将原来暂存区的改动恢复到暂存区

$ git stash drop [<stash>]
# 删除一个已入栈的工作区变动，如果不指定具体的 stash ，则默认删除最新入栈的 stash 。

$ git stash clear
# 删除所有已入栈的工作区变动
```

## 标签管理

标签 tag 可以认为是一个特殊的分支，我们通过给分支中的某个 commit 打上标签，用来作为代码的版本信息管理依据。

```bash
$ git tag -l
# 列出历史 tag 信息

$ git tag v1.0
# 新增标签

$ git tag -d v1.0
# 删除已有标签

$ git push origin v1.0
# 将已有标签同步到远程仓库

$ git push origin :v1.0
# 将远程仓库中的标签删除
```

## 结语

这篇笔记基本覆盖了本地使用 Git 时的可能会使用的命令，它们的复杂度并不高，我们只需要记忆这些实际应用中使用率比较高的命令，更复杂的命令可以在需要时再查阅。
