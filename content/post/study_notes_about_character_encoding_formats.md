---
date: 2023-09-26 15:22:00
title: 探究字符编码格式
tags:
  - "unicode"
  - "utf-8"
draft: false
---

一个由游戏汉化项目引起的思考。

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

基于日文开发的游戏无法正常在使用中文的 Windows 系统上显示文本。

如果你使用的是 Windows 操作系统，那么你可以在当前系统通过 `cmd` 或者 `PowerShell` 执行 `chcp` 命令，得到系统正在使用的活动代码页，活动代码页实际上就是 windows codepage ，中文的系统通常返回会是 936 ，也就是 CP936 。当你把系统语言设置为日文之后，就可以正常打开这些游戏，此时检查 codepage ，你会发现返回的代码应该是 932 ，也就是 CP932 。

codepage 是 Windows 系统用来转换编码格式的重要依据，在业务范围面向世界的 Windows 系统中，语言本地化是非常重要的环节，当使用了各种各样编码格式的文本要在 Windows 系统中能够以使用者熟悉的语言正常显示，就需要使用正确的编码格式来读取。

在这个过程中 Unicode 是一个不可忽视的编码标准。 Unicode 通过使用 `U+0000` 格式的码位来表示字符，这个格式可以覆盖非常大的数据范围，几乎足以包括了世界上所有语言中的字符。但是实际使用过程中很少直接使用对应的码位来表示字符，而是使用更高效的编码方式，也就是 utf-8 或者 utf-16 这两种。所以可以说 Unicode 是一个支持全部语言字符的通用性字符表，而 utf-8 和 utf-16 是基于这个字符表实现的更高效的表示方法。

实际上 Windows 系统也是通过 Unicode 来实现语言的多样化，在底层操作中使用 Unicode 可以避免很多问题，但是在本地化场景下，由于无法确定用户使用的语言，所以系统需要通过设置 codepage 来适应这个场景，这样当用户使用特殊的编码格式时，操作系统就可以通过 codepage 映射到 Unicode 中，保持底层操作的一致性。

## Unicode 字符编码

在这里我们只关注 utf-8 和 utf-16 两种编码格式，因为他们的使用频率最高。

首先是大小端的问题，因为硬件平台的差异，在处理多字节数据时，数据的高低字节读取顺序就很重要，应该保持写入时的顺序，否则读出来的数据就会截然不同。

以二进制数据 `0x1234` 为例子，此时高字节是 `0x12` ，而低字节是 `0x34` ，大小端的数据处理情况是这样的：

* 大端：高字节存放在内存低地址，低字节存放在内存高地址，假设从左到右表示内存地址增长，那么数据会是 `0x1234` 。
* 小端：高字节存放在内存高地址，低字节存放在内存低地址，假设从左到右表示内存地址增长，那么数据会是 `0x3412` 。

所以实际使用的 utf-16 还分为 utf-16 LE 和 utf-16 BE 两种，对应小端和大端的不同处理场景。

由于字节序会影响实际的读取，使用 utf-8 和 utf-16 编码格式的文件还需要通过字节顺序标记来记录对应的字节序，也就是在调整编码时经常会出现的 BOM 概念。这个标记在跨平台的时候非常重要，它是文件正常解码的关键。

utf-8 和 utf-16 的 BOM 会出现在文件开头，然后才是真正的文件内容， BOM 的具体内容是这样的：

* utf-8 ： `0xEFBBBF`
* utf-16 BE ： `0xFEFF`
* utf-16 LE ： `0xFFFE`

使用 utf-16 编码的文件必须带有 BOM ，否则会读取错误，而 utf-8 实际上可以不需要 BOM ，因为它是字节序无关的，但是使用 utf-8 编码格式的文件依然可以添加 BOM 作为标记。在 Windows 系统和 Linux 系统对 utf-8 BOM 的处理方式不同，在跨系统处理文件的时候可能要注意这个问题。

utf-8 编码可以使用一到四个字节来表示单个字符，英文字符只需要用到一个字节，而中文字符一般要用到三个字节，这样看起来中文字符或者使用多字节表示的字符似乎会受到大小端问题的影响，实际上 utf-8 虽然也用到了多字节，但是不同于 utf-16 这样的编码方式， utf-16 因为用到了两个字节和四个字节来表示字符，在使用两个字节的情况下相当于直接使用类似于 `U+0000` 的 Unicode 码位，此时字节序就会明显影响读取值所指向的具体码位。而 utf-8 在多字节的情况下，则需要通过第一个字节判断出具体的使用的字节数量，然后顺序读取后续的数据将它们当作整块数据来处理。

``` go
// from Standard library unicode/utf8
func DecodeRune(p []byte) (r rune, size int) {
	n := len(p)
	if n < 1 {
		return RuneError, 0
	}
	// as 和 first 都是常量
	// const as = 0xF0
	// first 是记录了 utf-8 第一个字节编码情况的数组
	p0 := p[0]
	x := first[p0]
	// 判断是否使用英文字符，即使用单个字节的情况
	if x >= as {
		// The following code simulates an additional check for x == xx and
		// handling the ASCII and invalid cases accordingly. This mask-and-or
		// approach prevents an additional branch.
		mask := rune(x) << 31 >> 31 // Create 0x0000 or 0xFFFF.
		return rune(p[0])&^mask | RuneError&mask, 1
	}
	// 使用多字节的情况
	// 通过第一个字节判断具体使用的字节数量
	sz := int(x & 7)
	accept := acceptRanges[x>>4]
	if n < sz {
		return RuneError, 1
	}
	b1 := p[1]
	if b1 < accept.lo || accept.hi < b1 {
		return RuneError, 1
	}
	// 使用两个字节的情况
	if sz <= 2 { // <= instead of == to help the compiler eliminate some bounds checks
		return rune(p0&mask2)<<6 | rune(b1&maskx), 2
	}
	b2 := p[2]
	if b2 < locb || hicb < b2 {
		return RuneError, 1
	}
	// 使用三个字节的情况
	if sz <= 3 {
		return rune(p0&mask3)<<12 | rune(b1&maskx)<<6 | rune(b2&maskx), 3
	}
	b3 := p[3]
	if b3 < locb || hicb < b3 {
		return RuneError, 1
	}
	// 使用四个字节的情况
	return rune(p0&mask4)<<18 | rune(b1&maskx)<<12 | rune(b2&maskx)<<6 | rune(b3&maskx), 4
}
```

这是用来测试的 utf-16 LE 编码转换为 utf-8 编码的代码：

``` go
package main

import (
	"bufio"
	"os"

	"golang.org/x/text/encoding/unicode"
	"golang.org/x/text/transform"
)

func main() {
	filename := os.Args[1]
	file, err := os.Open(filename)
	if err != nil {
		panic(err)
	}

	f, err := os.Create("./convert_" + filename)
	defer f.Close()
	w := bufio.NewWriter(f)

	scanner := bufio.NewScanner(transform.NewReader(file, unicode.UTF16(unicode.LittleEndian, unicode.UseBOM).NewDecoder()))
	for scanner.Scan() {
		line := append(scanner.Bytes(), []byte("\n")...)
		w.Write(line)
	}
	w.Flush()
}
```

这里其实是通过 utf-16 解码之后默认使用 utf-8 编码来实现的。如果想转换其他编码格式还需要重新定义编码器，使用新的编码器将 Scan 得到的数据转换为 string 类型写入。

## 常见语言的编码格式

GB2312 是比较早的中文字符编码，而后由于 Unicode 的制定和汉字收录拓展的关系，发展出了 GBK 编码格式，它向下兼容 GB2312 ，也基本实现了对当时版本 Unicode 收录汉字范围的覆盖，现代 Windows 系统使用的 CP936 就是 GBK 编码格式。

Shift JIS 则是日文字符编码，也就是 CP932 所使用的编码格式。

Big5 是繁体中文字符编码，对应的 codepage 是 CP950 ，多用于港澳台地区，而 GBK 主要是简体中文字符。

``` python
# 不同编码格式的读写测试

# 通过二进制编辑器来读取文件内容
raw = """
c4e3 bac3
"""
raw = raw.replace("\n", "")
raw = raw.replace(" ", "")
bins = b''.join([ bytes.fromhex(raw[i:i+2]) for i in range(len(raw))[::2] ])
# 以 GBK 编码格式来解码，可以输出实际的文本内容
bins.decode("gbk")
# 输出乱码可能是小端模式的显示问题，尝试互换位置再进行解码
(b''.join([ bytes.fromhex(raw[i+2:i+4] + raw[i:i+2]) for i in range(len(raw))[::4] ])).decode("gbk")
# 计算字节数量，通过十六进制显示
hex(int(len(raw)/2))

# 以 Shift JIS 编码格式来解码，可以用来解码一些在中文系统下的日文乱码文件
bins.decode("shift-jis")

# 指定某个编码格式将文本内容进行编码，通过十六进制显示
''.join([ f"{i:x}" for i in "文本内容".encode("gbk")])
```
