---
date: 2022-09-04 19:44:45
title: Lua 学习笔记
tags:
  - "Lua"
  - "Luarocks"
draft: false
---

这篇笔记用来记录一些 Lua 的相关知识。

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

lua 是一门高效的，易于嵌入其他应用程序的轻量级脚本语言。 lua 的高效快速主要是由于 lua 本身的底层设计以及非常高效的 Lua 解释器。这里主要会记录一些 lua 入门学习时一些遇到的问题。

## Lua String 处理

lua 的 string 类型数据是不可改变的，大部分情况下要依靠 string 库来处理。

``` lua
-- 字符串连接
var_a = "a"
var_b = "b"
var_c = "c"
print(var_a..var_b..var_c)
-- output:
-- abc

-- 字符串搜索
var_a = "a"
print(string.find(var_a, "a"))
-- output:
-- 1	1
-- 搜索成功则返回匹配字符的起止位置索引，失败则返回 nil 

-- 字符串匹配
print(string.match("  1 + 1  =2", "^%s*(.-)%s*=.*"))
-- output:
-- 1 + 1
print(string.match("  1 + 1  =2", "^%s*.-%s*=(.*)"))
-- output:
-- 2
-- 匹配成功则返回整个匹配的字符串，由于使用了括号进行子模式匹配，所以只返回了匹配部分，其中 .- 代表任意字符的非贪婪匹配
```

## Lua Table

lua table 是一种异于其他编程语言的数据类型，它是动态增长的字典类容器，同时具有数组的特性，如果仔细研究它的内部结构，可以发现它是有两者混合构成的。

``` lua
t = {a="A", "B", c="C", D}
print(t[0])    -- output: nil
print(t["a"])  -- output: A
print(t[1])    -- output: B
print(t["c"])  -- output: C
print(t[2])    -- output: nil
print(t[3])    -- output: nil

-- D 这个值是测试时疏漏了引号产生的错误，它会认为是一个值为 nil 的变量，相当于空洞
```

通过这个例子可以简单看出它们的混合特点，键值对和数组是互相独立的，需要使用明确的键取对应的值，数组值则使用数字索引，并且索引是从 1 开始而不是 0 ，要注意这里只是为了弄清楚它们的特性才使用了混合结构，实际中基本不会这样使用。

## Lua 模块和 Luarocks

常规的 lua 脚本需要依靠两个重要变量 `package.path` 和 `package.cpath` 来获取一些运行模块信息，它们主要区别在于模块是由 lua 语言实现还是由 c 语言实现。

``` bash
$ cat p.lua
print(package.path)
print(package.cpath)

$ lua p.lua
./?.lua;/usr/share/lua/5.1/?.lua;/usr/share/lua/5.1/?/init.lua;/usr/lib64/lua/5.1/?.lua;/usr/lib64/lua/5.1/?/init.lua
./?.so;/usr/lib64/lua/5.1/?.so;/usr/lib64/lua/5.1/loadall.so

# 可以通过指定环境变量 LUA_PATH 和 LUA_CPATH 来改变这两个值，其中 ;; 的第二个 ; 会被替换为原有的默认值
$ LUA_PATH="/tmp/?.lua;;" LUA_CPATH="/tmp/?.so;;" lua p.lua
/tmp/?.lua;./?.lua;/usr/share/lua/5.1/?.lua;/usr/share/lua/5.1/?/init.lua;/usr/lib64/lua/5.1/?.lua;/usr/lib64/lua/5.1/?/init.lua;
/tmp/?.so;./?.so;/usr/lib64/lua/5.1/?.so;/usr/lib64/lua/5.1/loadall.so;
```

我们如果想使用别人写好的 lua 脚本，通常需要自己去对应的发布地址下载并自行安装，但是 lua 也有类似于模块管理器一样的软件 luarocks ，使用它我们可以更方便管理我们需要用到的模块，也可以把我们自己写好的脚本提供给他人下载使用。

``` bash
# 使用 luarocks 下载安装 luasql
$ luarocks install luasql-mysql --server https://luarocks.cn MYSQL_INCDIR=/usr/include/mysql/ MYSQL_LIBDIR=/usr/lib64/mysql
# --server 可以指定 luarocks 服务器
# 可以通过环境变量来指定某些库文件位置，这些环境变量可以在对应模块的 rockspec 查找

# luasql-mysql rockspec
package = "LuaSQL-MySQL"
version = "2.3.5-1"
source = {
  url = "git://github.com/keplerproject/luasql.git",
  branch = "v2.3.5",
}
description = {
   summary = "Database connectivity for Lua (MySQL driver)",
   detailed = [[
      LuaSQL is a simple interface from Lua to a DBMS. It enables a
      Lua program to connect to databases, execute arbitrary SQL statements
      and retrieve results in a row-by-row cursor fashion.
   ]],
   license = "MIT/X11",
   homepage = "http://www.keplerproject.org/luasql/"
}
dependencies = {
   "lua >= 5.1"
}
external_dependencies = {
   MYSQL = {
      header = "mysql.h"
   }
}
build = {
   type = "builtin",
   modules = {
     ["luasql.mysql"] = {
       sources = { "src/luasql.c", "src/ls_mysql.c" },
       libraries = { "mysqlclient" },
       incdirs = { "$(MYSQL_INCDIR)" },
       libdirs = { "$(MYSQL_LIBDIR)" }
     }
   }
}

# 为了维持解释器环境不被污染，可以使用 --tree 指定模块的安装路径
$ luarocks install luasql-mysql --tree $(pwd)
# 如果不指定 --tree ，会根据当前用户决定安装路径， root 一般会安装到 /usr 这个路径中

# 删除已经安装的模块，必须指定 --tree
$ luarocks purge luasql-mysql --tree $(pwd)
```
