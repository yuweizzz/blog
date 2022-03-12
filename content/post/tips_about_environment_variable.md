---
date: 2020-03-29 20:41:45
title: 配置 Linux 环境变量
tags:
  - "Linux"
  - "Environment Variable"
draft: false
---

这篇笔记用来记录配置 Linux 环境变量的相关知识。

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

## 背景

最近经常需要自行编译安装一些开源软件，它们有着各自的运行方式，经常需要定义一些环境变量，那么在何处定义这些变量成了需要探究的问题。

## 设置即时生效的环境变量

我们可以从最常用的环境变量 `PATH` 开始，它是寻找解释器或者命令的默认路径，以 `:` 分割不同路径的形式出现。在这个变量定义的路径中，可执行文件可以被快速搜索并直接执行，而且路径的顺序是有意义的，越靠前的搜索优先级越高。经常可以在一些软件的使用文档中看到 `export PATH=/path/to/bin:$PATH` 的类似用法。

除了 `export` 之外，还有 `source` 也可以用来设置一些环境变量， python 的虚拟环境 venv 就用到了 `source` 来设置一些环境变量，实际上 `export` 和 `source` 都是 bash 的内置命令。

``` bash
# 以下出自 bash 的相关文档
$ man bash
...
SHELL BUILTIN COMMANDS
...
       export [-fn] [name[=word]] ...
       export -p
              The supplied names are marked for automatic export to the environment of subsequently executed commands.
              If the -f option is given, the names refer to functions.If no names are given, or if the -p option is
              supplied, a list of all names that are exported in this shell is printed.  The -n option causes the export
              property to be removed from each name.  If a variable name is followed by =word, the value of the variable 
              is set to word.  export returns an exit status of 0 unless an invalid option is encountered, one of the 
              names is not a valid shell variable name, or -f is supplied with a name that is not a function.
...
        .  filename [arguments]
       source filename [arguments]
              Read and execute commands from filename in the current shell environment and return the exit status of 
              the last command executed from filename.  If filename does not  contain a slash, file names in PATH are 
              used to find the directory containing filename.  The file searched for in PATH need not be executable.
              When bash is not in posix mode, the current directory is searched if no file is found in PATH.  If the 
              sourcepath option to the shopt builtin command is turned off, the  PATH is not searched.If any arguments 
              are  supplied, they become the positional parameters when filename is executed.  Otherwise the positional 
              parameters are unchanged.The return status is the status of the last command exited within the script 
              (0 if no commands are executed), and false if filename is not found or cannot be read.
...
```

从文档可以看到， `export` 被明确定位用来快速设置环境变量。而 `source` 则要特殊一些，可以看到它是对文件级进行操作的，实际上它并不明确涉及任何变量改动。但是它们有一个相同点在于它们的生效范围都是当前 shell 。所以当你使用了 `export` 设置了一些变量，或者使用 `source` 执行了某个文件，那么它们涉及的变量改动就影响了当前 shell ，起到了改变环境变量的效果。

## 设置可持久化的环境变量

由于前面所述的方法只对当前 shell 生效，那么持久方案也值得我们额外关注。

这里只讨论 Linux 环境下的 bash ，其他类型的 shell 和 bash 可能略有差别，但它们的机制应该是相近的。

### shell 的属性

bash 的机制比表面看上去的复杂得多，它隐藏了很多的属性设置。可以根据我们比较关系的属性，来简单划分 shell 的种类。

影响环境变量的属性有比较关键的两个，分别是登录相关的属性和交互相关的属性。

根据登录相关的属性分为登录式 login shell 和非登录式 non-login shell ，根据交互相关的属性分为交互式 interactive shell 和非交互式 non-interactive shell ，将两种属性组合起来可以将 shell 分为四个种类。

``` bash
# 以下结果由直接登录系统后的 shell 中执行得出

# 是否为登录式 shell
$ shopt | grep login
login_shell          on

# 是否为交互式 shell
$ echo $-
himBH

# 以脚本文件执行检查
$ cat test.sh
shopt | grep login
echo $-
$ bash test.sh
login_shell          off
hB
```

关于登录属性的判断比较直观，关于交互属性则需要通过特殊变量去获取， `-` 变量可以获取 shell 的部分设置选项，每个字母代表一种属性，其中 `i` 正是关于交互属性的字段。 在 `-` 变量包含的各字段意义如下：

``` bash
$ man bash
...
OPTIONS
       -l        Make bash act as if it had been invoked as a login shell (see INVOCATION below).
                 # 以登录模式启动 shell
       -i        If the -i option is present, the shell is interactive.
                 # 以交互模式启动 shell
...
SHELL BUILTIN COMMANDS
       set [--abefhkmnptuvxBCEHPT] [-o option-name] [arg ...]
       set [+abefhkmnptuvxBCEHPT] [+o option-name] [arg ...]
              Without options, the name and value of each shell variable are displayed in a format that can be reused 
              as input for setting or resetting the currently-set variables.Read-only variables cannot be reset.
              In posix mode, only shell variables are listed.  The output is sorted according to the current locale.
              When options are specified, they set or unset shell attributes.  Any arguments remaining after option 
              processing are treated as values for the positional  parameters and are assigned,
              in order, to $1, $2, ...  $n.  Options, if specified, have the following meanings:
...
              -h      Remember the location of commands as they are looked up for execution.
                      This is enabled by default.
                      # 缓存执行过的二进制命令的路径，以便下次调用省去搜索时间
              -B      The shell performs brace expansion (see Brace Expansion above).  This is on by default.
                      # 允许使用 shell 的花括号扩展
              -m      Monitor mode.  Job control is enabled.  This option is on by default for interactive shells on 
                      systems that support it (see JOB CONTROL above).  Back‐ground processes run in a separate process
                      group and a line containing their exit status is printed upon their completion.
                      # 允许作业控制和后台运行
              -H      Enable !  style history substitution.  This option is on by default when the shell is interactive.
                      # 允许快速索引存在历史记录中的命令
...
```

### profile 和 rc

bash 使用了 profile 和 rc 来完成一些 bash 环境的设置工作，环境变量的设置会在这一步完成。

``` bash
# 以下出自 bash 的相关文档
$ man bash
...
FILES
       /etc/profile
              The systemwide initialization file, executed for login shells
       /etc/bash.bash_logout
              The systemwide login shell cleanup file, executed when a login shell exits
       ~/.bash_profile
              The personal initialization file, executed for login shells
       ~/.bashrc
              The individual per-interactive-shell startup file
       ~/.bash_logout
              The individual login shell cleanup file, executed when a login shell exits
       ~/.inputrc
              Individual readline initialization file
...
INVOCATION
...
       When bash is invoked as an interactive login shell, or as a non-interactive  shell  with  the
       --login option, it first reads and executes commands from the file /etc/profile, if that file
       exists.  After reading that file, it looks for ~/.bash_profile,  ~/.bash_login,  and  ~/.pro‐
       file,  in  that  order, and reads and executes commands from the first one that exists and is
       readable.  The --noprofile option may be used when the  shell  is  started  to  inhibit  this
       behavior.

       When  a login shell exits, bash reads and executes commands from the files ~/.bash_logout and
       /etc/bash.bash_logout, if the files exists.

       When an interactive shell that is not a login shell is started, bash reads and executes  com‐
       mands from ~/.bashrc, if that file exists.  This may be inhibited by using the --norc option.
       The --rcfile file option will force bash to read and execute commands from  file  instead  of
       ~/.bashrc.

       When  bash is started non-interactively, to run a shell script, for example, it looks for the
       variable BASH_ENV in the environment, expands its value if it appears  there,  and  uses  the
       expanded  value  as the name of a file to read and execute.  Bash behaves as if the following
       command were executed:
              if [ -n "$BASH_ENV" ]; then . "$BASH_ENV"; fi
       but the value of the PATH variable is not used to search for the file name.
...
```

加载 profile 和 rc 是由 shell 属性决定的，从 bash 文档可以得出，登录式 shell 启动时会加载 profile 系列文件，非登陆交互式 shell 启动时会加载 rc 系列文件。

根据上述文档，只要是登录式 shell ，不论是否带有交互属性，它的显式预加载链为 `/etc/profile --> ~/.bash_profile` ，可以通过 `bash -i -l -x` 进行验证，读取整体链条文件可以知道完整的预加载链为：

``` bash
/etc/profile    -->    ~/.bash_profile
      |                       |
      V                       V
/etc/profile.d/*.sh       ~/.bashrc
                              |
                              V
                         /etc/bashrc
                              |  is non-login shell  # 当前是 login shell ，这一步实际中不执行
                              V
                      /etc/profile.d/*.sh
```

如果是非登陆交互式 shell ，它的显式预加载链为 `~/.bashrc` 。通过 `bash -i -x` 或者 `bash -x` 进行验证，效果是一致的，完整的预加载链为：

``` bash
~/.bashrc
     |
     V
/etc/bashrc
     |  is non-login shell
     V
/etc/profile.d/*.sh
```

现在只剩最后一种 shell ，非登录式非交互式 shell 听上去不可思议，但它确实是存在的，其实在前面运行 shell 的属性检查脚本 `bash test.sh` 的那个 shell 就是非登录式非交互式 shell ，它并不会加载任何 profile 或者 rc ，但它会从父进程继承环境变量，并且执行完文件中的命令这个 shell 就消亡了。

## 选择合理的设置文件

综合上面来看，我们可以在两个地方设置持久的环境变量，那就是 `/etc/profile.d/*.sh` 和 `~/.bashrc` 。

将变量写在 `/etc/profile.d/*.sh` 是 bash 本身推荐的做法，但是如果仔细阅读过 `/etc/bashrc` 和 `/etc/profile` 的代码你会知道，它们对 `/etc/profile.d/*.sh` 的处理是一致的，只有对应的用户拥有 `/etc/profile.d/` 中的文件可读权限，那么这个脚本才会被执行。如果有时候自行写入了 `/etc/profile.d/` 中的文件却无法正常加载变量，很可能就是权限导致的问题。

将变量写在常用用户的 `~/.bashrc` 我个人认为是更好的办法，它的好处在于方便隔离，各个用户之间不会互相影响。

实际使用需要基于实际场景来设置，如果是公共性质比较强的环境变量，写在 `/etc/profile.d/*.sh` 中更方便，如果是一些个人环境和调试使用的环境变量，写在 `~/.bashrc` 更好。

## 题外话

由于本文涉及了 `source` ，为了更好理解，这里记录了一些常用的脚本执行方式，隐藏在它们背后的实际调用方式如下：

* `./xx.sh` 要求文件有执行权限，它直接执行了文件，实际上生成新的非登录非交互式 shell 去执行文件内容，执行后 shell 消亡。
* `source xx.sh` 没有文件权限要求，它会在当前 shell 执行并返回执行结果。
* `. xx.sh` 执行方式等同于 `source xx.sh` 。
* `. ./xx.sh` 执行方式等同于 `source xx.sh` ，加上 `./` 是为了指明文件路径。
* `bash xx.sh` 没有文件权限要求，执行方式等同于 `./xx.sh` 。
* `bash ./xx.sh` 执行方式等同于 `./xx.sh` ，加上 `./` 是为了指明文件路径。
