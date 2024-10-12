---
date: 2023-04-15 21:46:00
title: ffmpeg 命令笔记
tags:
  - "ffmpeg"
draft: false
---

这篇笔记用来记录一些常用的 ffmpeg 命令。

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

## 正文

ffmpeg 可以用来处理音视频，以下是一些常用的处理命令：

```bash
# 按照给定时间长度从头截取视频或音频，不做转码
$ ffmpeg -i input.mp3 -t 03:30 -c:v copy -c:a copy output.mp3

# 从某个时间开始，按照给定时间长度截取视频或音频，不做转码
$ ffmpeg -ss 01:00 -i input.mp3 -t 03:30 -c:v copy -c:a copy output.mp3
# 以上例子会从文件开始一分钟后截取三分半钟的片段，相当于新文件会是原有文件的 01:00 到 04:30 这段内容

# 提取音频，去掉视频
$ ffmpeg -i input.mp4 -c:a copy -vn output.mp4

# 提取视频，去掉音频
$ ffmpeg -i input.mp4 -c:v copy -an output.mp4

# 获取文件信息
$ ffprobe -v quiet -print_format json -show_format -show_streams input.mp4

# 制作音频部分的淡入和淡出效果
$ ffmpeg -i input.mp4 -af "afade=t=in:st=0:d=5,afade=t=out:st=210:d=5" output.mp4
# afade=t=in 代表 audio fade in 效果， st 代表开始的时间， d 代表持续时长
# 以上例子会在视频开始前 5 秒声音渐大， 210 秒到 215 秒声音渐小

# 制作视频部分的淡入和淡出效果
$ ffmpeg -i input.mp4 -vf "fade=in:0:3,fade=out:210:5" output.mp4
# 类似于音频的例子，但它会应用于视频部分

# 使用图片来生成一图流视频
$ ffmpeg -r 15 -f image2 -loop 1 -i input.jpg -i input.mp3 -s 1920x1080 -pix_fmt yuv420 \
    -t 210 -vcodec libx264 output.mp4
# -r 代表帧率，帧率越高画面越流畅，一图流视频的帧率无需过高
# -t 代表持续时间，在使用连续图片的情况下可以不使用 -loop 和 -t 参数，但单张图片需要使用 -loop 1 代表无限循环单张图片
# -f 代表使用 image2 来处理输入， -pix_fmt 代表图片的输入格式
# -s 代表分辨率， -vcodec 代表编码格式

# 音视频推流
$ ffmpeg -i input.mp4 -re -stream_loop -1 -c copy -f flv "rtmp://rtmp_endpoint"

# -re 代表 Read input at native frame rate 即以输入的原有帧率来读取
# -stream_loop -1 代表推流次数为 -1 ，也就是无限循环当前输入
```
