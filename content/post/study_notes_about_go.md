---
date: 2022-05-09 19:44:45
title: Golang 学习笔记
tags:
  - "Go"
draft: false
---

这篇笔记用来记录一些 Golang 的相关知识。

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

## 创建 Go 项目

Go 的运行需要依赖于环境变量，以下是比较重要的环境变量：

* GOPATH ： Go 项目的存放路径，也就是整体的工作空间。
* GO111MODULE ： Go 的依赖管理系统开关。
* GOROOT ： Go 的安装路径，包括了命令行工具，标准库和文档等。
* GOPROXY ： Go 依赖包下载的代理地址，在国内的网络环境中非常重要，使用 `go env -w GOPROXY=https://goproxy.cn,direct` 直接更换即可。

在我开始使用 Go 语言的时候，主线版本是 1.16 ，所以此时 Go Module 已经非常成熟了，但网上仍存在大量关于 GOPATH 的相关资料，它同样和依赖管理密切相关，在这里首先需要明确的是 Go Module 是用来替代 GOPATH 的依赖管理模式。

使用 GOPATH 模式一般会有三个目录： `bin` ， `pkg` ， `src` ，其中所有项目的源代码都会在 `src` 目录中，这个目录下会以多级目录的形式维护各级依赖，而 `pkg` 用来存放编译过程的中间文件， `bin` 用来存放最终生成的可执行文件。

一般来说，我们创建新项目是最终目标是创建 main 包，并且经常需要在 main 包中导入依赖，但是这些依赖在 GOPATH 模式下，存放位置就只有 `$GOPATH/src` 和 GOROOT 中的标准库这两个位置，新项目仍需要处在 GOPATH 中，否则会找不到项目的依赖模块，就算可以找到对应的模块，也有着版本冲突的危险。

在使用 Go Module 的情况下，我们可以使用 `go mod` 来创建新项目和声明依赖，当前新项目的子目录就可以视为自身的子模块，在使用时就可以直接导入而不会在 GOPATH 中搜索并报错，这样就脱离了 GOPATH 的限制。如果是来自外部的模块，则需要使用 `go get package` 来导入，并且这些外部依赖的源代码会下载到 `$GOPATH/pkg/mod` 中，并在当前模块中执行严格的版本控制，这也是为什么使用 Go Module 进行依赖管理而 GOPATH 变量依旧重要的原因。

``` bash
# go mod 常用命令

# 创建项目
$ go mod init package
# 获取外部依赖
$ go get github.com/BurntSushi/toml
# 去除无用依赖
$ go mod tidy
# 修改依赖信息，例如修改依赖版本或者依赖包重命名
$ go mod edit -replace github.com/BurntSushi/toml=github.com/BurntSushi/toml@v1.1.0
# 等号前是修改前的信息，等号后是目标信息
```

使用 Go Module 的 Go 项目可以使用这两种结构：

* 在整个项目的根预留 `main.go` 作为总入口，使用子目录来区分各个功能模块。

``` bash
# 整体的项目结构如下
$ tree 
.
├── submoduleA
│   └── a.go
├── submoduleB
│   └── b.go
├── go.mod
└── main.go

# main.go 会是这样的：
...
package main

import module/submoduleA
import module/submoduleB

func main(){...}
...
# submodule 的编写会是这样：
...
package submoduleA

func A(){...}
...
```

* 将整个项目视为模块，额外创建目录用于制作 `main.go` 总入口。

``` bash
# 整体的项目结构如下
$ tree 
.
├── submoduleA
│   └── a.go
├── submoduleB
│   └── b.go
├── c.go
├── go.mod
└── cmd
    └── main.go

# main.go 和 submodule 的编写和前面基本一致
# 但 c.go 则直接属于 module ，它的编写会是这样：
...
package module

func C(){...}
...
```

某个目录存在 package main 的文件意味着当前目录下的 go 文件都是运行程序，这个目录允许存在多个 package main 文件，但不能存在其他 package 的 go 文件，并且这样的目录在任意模块中也不会被作为子模块导入。

## Golang 的数据类型

Golang 的数据类型和大多数编程语言相似，但有一些细微的区别。

这里先明确值类型和引用类型的区别：

* 值类型：变量直接存储数据。
* 引用类型：变量直接存储指针，再由指针指向实际存储的数据。

### 值类型数据

布尔类型 bool 是比较简单的数据类型，在 Golang 中以 `true` 和 `false` 作为直接关键字，但不能像 Python 直接将空值或者零值做为布尔值来推断，因为 Go 不支持隐式转换类型。

整型 int 细分为有符号类型 int8 ， int16 ， int32 ， int64 和各自对应的无符号类型 uint8 ， uint16 ， uint32 ， uint64 一共 8 种类型。这里对整形数据的定义和 C 语言是基本一致的，实际代码编写过程中通常直接使用 int 类型和 uint 类型，而 int 的具体位数一般在编译时确定，在现代操作系统中通常会是 32bits 或者 64bits ，但即使是在 64bits 操作系统中 int 的编译结果通常会是 32bits 。在需要强关联某个具体 bits 类型时可以通过显式声明来实现，这种情况常见于内核开发，普通应用比较少。

浮点型 float 则有 float32 和 float64 两种，具体用法类似于 int 。

还有两种特殊类型 byte 和 rune ，其中 byte 是 int8 的别名，rune 是 int32 的别名，这两种类型用来表示数据不是数值类型，应该直接以二进制格式对待。

字符类型 string 是更为特殊的数据类型，它的本质是只读的 byte slice ，所以 string 是可以遍历的，但只能得到 byte 类型的数据，由于字符编码格式的不同，直接使用 len 测试 string 长度会出问题，因为 UTF-8 编码的中文字符占用三个 byte 的存储空间，而 UTF-8 编码和 ascii 编码的英文字符都只需要占用一个 byte 的存储空间，两个中文字符的计算结果会是 6 而不是 2 。所以字符类型应该使用对应的库去处理，减少类型转换，如果确实需要使用类型转换，纯中文字符，纯英文字符和混合中英字符可以使用 rune 类型，纯英文字符还可以额外使用 byte 类型。

array 在 Go 中是值类型，声明时以 `[n]T` 的形式出现，它必须在声明时指定长度，可以通过类似 `Array := [...]int{12, 78, 50}` 的形式自动推断长度，并且后续这个长度是不可变的。

struct 也是值类型，是由其他类型组合形成的复合数据类型。

### 引用类型数据

slice ， map 和 channel 是引用类型数据。

slice 称为切片，和 array 类似，一般从 array 截取得到 slice 。但和 array 不同， slice 是长度可变的，声明时以 `make([]T, n)` 的形式出现，声明后仍然可以申请内存空间。

map 则是映射，类似于字典，声明时以 `make(map[T]T)` 的形式出现。

``` golang
// array
string_array := [3]string{"1", "2", "3"}

// slice
var string_slice = []string{"1", "2", "3"}
string_slice := []string{"1", "2", "3"}
string_slice := make([]string, 10)

// map
map_expamle := map[string]int{"a": 3, "b": 4}
```

### 函数传参

在 Golang 中，所有函数的传参都是传值，在函数中的形参是实参的副本。

如果是值类型作为参数，这种结论非常容易理解，那就是形参在函数中的任何修改并不会影响实参，因为它们在内存中的实际位置并不相同。

如果是引用类型作为参数，通常会造成比较大的迷惑，我们可以逐一分析：

``` golang
package main

import "fmt"

func changeArrayByPointer(a *[3]int) {
    a[0] = 10
}

func changeArray(a [3]int) {
    a[0] = 10
    fmt.Println("in func changeArray():", a)
}

func changeSlice(s []int) {
    s[0] = 10
}

func changeMap(m map[string]int) {
    m["a"] = 10
}

func main() {
    a := [3]int{1, 2, 3}
    b := [3]int{1, 2, 3}
    s := []int{1, 2, 3}
    m := map[string]int{"a": 1, "b": 2}
    changeArrayByPointer(&a)
    fmt.Println("a:", a)
    changeArray(b)
    fmt.Println("b:", b)
    changeSlice(s)
    fmt.Println("s:", s)
    changeMap(m)
    fmt.Println("m:", m)
}
// output:
// a: [10 2 3]
// in func changeArray(): [10 2 3]
// b: [1 2 3]
// s: [10 2 3]
// m: map[a:10 b:2]
```

这里是一个非常典型的例子，首先我们可以看到，对于值类型 array ，当它们作为函数参数时，并无法改变实参，但是通过使用指针变量作为函数参数时，确实成功改变了实参，而对于引用类型 map 和 slice ，由于它们隐式使用了指针变量作为函数参数，所以最终实参也被成功改变。

这里比较颠覆传统的思路是 struct 类型，要注意它实际是值类型，如果想要通过函数改变结构体的内容，直接将 struct 作为参数是不行的，应该这样做：

``` go
package main

import "fmt"

type A struct {
    a string
    b string
}

func changeA(s *A) {
    s.a = "A"
    s.b = "B"
}

func main() {
    s := &A{a: "a", b: "b"}
    fmt.Println(s)
    changeA(s)
    fmt.Println(s)
}
// output:
// &{a b}
// &{A B}
```

它必须和 array 类型一样，显示地声明函数参数就是一个指针变量，否则直接使用结构体作为函数参数是无法成功改变实参的。
