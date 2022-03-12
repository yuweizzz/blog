---
date: 2021-08-03 22:00:00
title: 前端项目开发笔记
tags:
  - "Linux"
  - "npm"
  - "Node.js"
draft: false
---

这篇笔记用来记录一些前端项目开发过程中的常用操作。

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

## 搭建 Node.js 环境

Node.js 提供了二进制预编译的安装包，在[官网](https://nodejs.org/zh-cn/download/)直接下载到本地即可解压使用。

这里以 LTS 14.15.4 为例:

``` bash
$ curl -o /usr/local/src/node-v14.15.4-linux-x64.tar.xz 
    https://nodejs.org/download/release/v14.15.4/node-v14.15.4-linux-x64.tar.xz
$ tar -xvJf /usr/local/src/node-v14.15.4-linux-x64.tar.xz -C /usr/local/
$ echo 'export PATH="$PATH":/usr/local/node-v14.15.4-linux-x64/bin/' >> /etc/profile
```

如果只有某个用户需要使用 Node.js，可以把 PATH 变量写入到对应用户的 bash_profile 或 bashrc 中，这样可以起到一样的效果并且不会影响其他用户的环境变量。

## 修改 npm 镜像地址

npm 是 Node.js 的依赖包管理工具，类似于 python 的 pip ， npm 在国内的网络环境会有下载速度慢的现象，可以将镜像地址修改为国内的镜像地址来解决这个问题。 

``` bash
# 检查 npm 基本配置：
$ npm config list
; cli configs
metrics-registry = "https://registry.npmjs.org/"  # 默认的源镜像
scope = ""
user-agent = "npm/6.14.10 node/v14.15.4 linux x64"

; node bin location = /usr/local/node-v14.15.4-linux-x64/bin/node
; cwd = /usr/local/node-v14.15.4-linux-x64
; HOME = /root
; "npm config ls -l" to show all defaults.

# 修改为淘宝 npm 镜像以加快访问速度：
# 只修改当前用户的配置，非全局修改会生成 $HOME/.npmrc 
$ npm config set registry http://registry.npm.taobao.org/
$ cat ~/.npmrc  # 只影响当前用户
registry=http://registry.npm.taobao.org/

# 使用 -g 修改全局配置，全局修改会生成 $PREFIX/etc/npmrc 
$ npm -g config set registry http://registry.npm.taobao.org/ 
$ cat /usr/local/node-v14.15.4-linux-x64/etc/npmrc  # 生成于 node 安装目录下的 etc 目录中
registry=http://registry.npm.taobao.org/
```

## 初始化项目

下面简单记录了初始化前端项目的过程，可以提供后续参考。

生成 package.json 是一个项目的开始，无论使用的依赖包管理工具是 npm 或是 yarn 。我选用了 npm ，实际可以按照个人喜好来选择。

``` bash
$ mkdir myproject
$ cd myproject
$ npm init  # 交互式生成 package.json
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sensible defaults.

See `npm help init` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg>` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
package name: (myproject) 
version: (1.0.0) 
description: my project
entry point: (index.js) src/index.js
test command: node -v; npm -v;  # 这里是为了测试，实际使用时可以按需修改
git repository: 
keywords: 
author: me
license: (ISC) 
About to write to /tmp/myproject/package.json:

{
  "name": "myproject",
  "version": "1.0.0",
  "description": "my project",
  "main": "src/index.js",  # 通常会使用 src 来存放项目代码，实际使用时可以按需修改
  "scripts": {
    "test": "node -v; npm -v;"
  },
  "author": "me",
  "license": "ISC"
}


Is this OK? (yes) yes
```

生成 package.json 后，我们可以按照需要安装一些项目模块和开发模块工具。

``` bash
# 安装项目需要的模块
$ npm install <dependency> 
# 使用 -D 安装项目需要的开发模块工具
$ npm install -D webpack  # 打包工具
$ npm install -D webpack-cli
$ npm install -D eslint  # 语法检查工具

# 检查配置文件，可以看到新的依赖项已经被安装
$ cat package.json 
{
  "name": "myproject",
  "version": "1.0.0",
  "description": "my project",
  "main": "index.js",
  "scripts": {
    "test": "node -v; npm -v;"
  },
  "author": "me",
  "license": "ISC",
  "devDependencies": {
    "webpack": "^5.48.0",
    "eslint": "^7.32.0",
    "webpack-cli": "^4.7.2"
  }
}

# 安装完成后，可以在 node_modules 中找到对应模块目录
# 可直接执行脚本还会产生符号链接提供给项目使用，如下例子：
./node_modules/.bin/webpack-cli -> ../webpack-cli/bin/cli.js
```

虽然可以通过指明路径调用一些可用的脚本和命令，但我们一般把这些常用的命令写入到 package.json 的 scripts 中去，方便后续使用。

``` bash
# package.json 中 scripts 的基本使用：
# scripts 通常使用 build ， lint ， run 等常用命名来配置对应的一系列命令
# npm run 可以执行这些配置好的命令

# 测试之前已经配置的 scripts test ：
$ npm run test

> myproject@1.0.0 test
> node -v; npm -v

v14.15.4
7.20.3

# 后续使用会配置一些新的 scripts
```

下面是一些常用的开发模块，只用到了比较简单的场景，更详细的配置需要参考官方文档和其他优秀项目的用法。

使用 webpack 作为打包工具。

``` bash
# 配置 webpack ：
# 手工生成 webpack.config.js ，文件位于项目根目录
$ cat webpack.config.js
// start
const webpack = require('webpack');
const path = require('path');

const config = {
  entry: './src/index.js',  // 实际使用按需修改
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js'  // output 为打包输出目标
  }
};

module.exports = config;
// end
# 生成 webpack.config.js 后配置 webpack 打包命令到 npm scripts 
$ npm set-script "build" "webpack --mode production --config webpack.config.js"
# 添加完成后可以直接调用 npm run build 执行打包
```

使用 eslint 作为语法检查工具。

``` bash
# 配置 eslint ：
# 直接调用可执行脚本进行语法相关的配置初始化
$ ./node_modules/.bin/eslint --init
...
# 整个过程按照提示进行选择即可，一般会产生依赖安装，完成后会自动生成文件
Successfully created .eslintrc.json file in /tmp/myproject
...
# 配置 eslint 检查命令到 npm scripts
$ npm set-script "lint" "eslint src/"
# 添加后可以直接调用 npm run lint 执行语法检查
```

