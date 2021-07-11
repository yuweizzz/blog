---
date: 2021-07-05 21:08:45
title: 在网页上添加Live2D看板娘
tags:
  - "Hugo"
  - "Live2D"
draft: false
---

在 Hugo 静态网站添加 Live2D 看板娘。

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

在没有建设自己的站点之前，见到别人站点上的 Live2D 看板娘觉得很有趣，所以在搭建了自己的站点后，也想要在自己的站点实现。

## Hugo站点自定义

实现网页添加 Live2D 看板娘需要先对站点进行适配改造。

Jekyll ， Hexo 和 Hugo 这一类静态网站生成器的工作方式都是类似的，它们大都是以 Markdown 语法编写网页的主体，通过不同的模板组合来渲染静态页面，这种工作方式使得自定义非常方便，一般只需要修改或者添加模板就可以达到自定义的效果了。

自定义过程中为了避免修改过多的模板，把需要进行的工作全部放在 Javascript 中是个不错的选择。本次自定义也遵循这样的做法，基本上只需要额外引用 Javascript 文件就可以了，在 Hugo 中，负责加载 Javascript 的部分一般存放在 Partial Templates 中。

以下片段出自 Hugo 的[官方文档](https://gohugo.io/templates/partials/])：

```
Partial templates—like single page templates and list page templates—have a specific lookup order.
However, partials are simpler in that Hugo will only check in two places:

1.layouts/partials/*<PARTIALNAME>.html
2.themes/<THEME>/layouts/partials/*<PARTIALNAME>.html

This allows a theme’s end user to copy a partial’s contents into a file of the same name for further customization.
```

根据上述文档，我们可以复制并修改主题中的 Partial Templates 来达到自定义的效果，并且可以不改动主题原有的模板文件。

以下是具体的操作步骤：

``` bash
# 修改 config.toml 
# 使用变量控制自定义部分
$ vi config.toml
.....  # 新增变量
[params]
custom_js = "js/custom_live2d.js"  
.....


# 复制原有主题文件到高查找等级的目录中
$ cp themes/<THEME>/layouts/partials/script.html layouts/partials/script.html


# 修改 layouts/partials/script.html
$ vi layouts/partials/script.html
.....  # 直接将下列部分添加到script.html中
{{ if .Site.Params.custom_js -}}
<script type="text/javascript" src="{{ $.Site.Params.custom_js | absURL }}"></script>
{{- end }}
.....


# 将完成之后的 Javascript 文件放到 static/js 中即可
# 后续生成站点就会以新模板工作
```

关于 Partial Templates 修改的部分，可以选择直接写入 Javascript 引用标签，这里使用带变量控制的语法块是为了后续的维护方便。模板语法和 python 的 jinja2 非常相似，有过使用经验应该很容易理解。

到这里 Hugo 需要改动的地方就完成了，接下来需要做的是将 Live2D 模型资源和对应的 Javascript 文件正确引用。

## Live2D资源的应用

在网络上已经有很多开箱即用的相关应用了，但是它们不契合我的喜好，于是我在已有资源的基础上修改，改进为我喜欢的效果。

首先需要了解的是现有的模型资源分类，现有的 Live2D 模型资源有 Cubism 2 ， Cubism 4 和 Cubism 3 三个版本，其中 3 和 4 之间是互相兼容，所以基本上可以认为只有两个版本。 2 和 3 之间还是有很大区别的，在个人项目中我只使用到了 Cubism 3 。

``` bash
# Cubism 3 Model 基本结构：
.
├── Name.moc3
├── Name.model3.json  # 最重要的入口文件
├── Name.physics3.json
├── motions
│   ├── Action1.motion3.json
│   ├── Action2.motion3.json
│   └── Action3.motion3.json
└── textures
    ├── texture_00.png
    ├── texture_01.png
    └── texture_02.png
```

模型文件大都是 json 格式，其中 model3.json 是最为重要的入口文件，很多模型的行为都是由它定义的，后续也需要修改这个文件来实现一些自定义需求。

现在我们拥有了模型，如果要在网页上实现模型的应用，还需要对应模型版本的 framework 的支持。在这一块我们只能使用 Live2D 出品公司原有的框架，或者使用由大神整理过的资源，我直接使用了 [pixi-live2d-display](https://github.com/guansss/pixi-live2d-display) 插件，在这个插件中， pixijs 是整个项目的重要部分，它是一个强大的前端 2D 动画渲染引擎，和 Cubism framework 很好地结合在一起，所以我们不必去了解底层的 api ，可以直接使用这个插件为我们提供的 api ，这也是我选择这个插件的原因。

在我的个人项目中，主要使用了原生 pixijs 的 api 以及 pixi-live2d-display 中提供的 api ，通过数据绑定和原生 api 来构建主体功能，并且对模型资源进行了部分修改，使得运行时的模型支持交互动作，主要是在 model3.json 文件中修改 motion 相关的部分。有兴趣可以参考[源码](https://github.com/yuweizzz/CustomLive2D)。

最后一步是将整个项目通过 webpack 打包为单个 Javascript 文件，并将打包后的 Javascript 文件和 Live2D 模型资源安置到 Hugo 的 static 目录中就大功告成了。

## 结语

总结来说，自定义站点这部分比较简单，在 Live2D 应用花费了比较多的时间，因为我是个前端新手，对 Javascript 的应用还只停留于通过简单的 DOM 操作和基础函数调用。但是经过这个项目，让我感受到了 Javascript 的迅速发展，它对于有后端经验的人还是比较友好的。

在实际的使用过程，网络是重要的因素，如果经常会出现模型加载异常，可以考虑使用 CDN 资源。