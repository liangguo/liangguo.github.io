---
layout: page
title: "Install Audacity ffmpeg plugin on Mac OSX"
description: ""
category:
tags: [audacity,ffmpeg]
categories: [misc]
---

# 缘起

最近, 我开始使用`audacity`进行音频编辑, audacity的安装比较简单,只需要运行`brew install audacity`就可以安装, 但默认安装后, 不能读取mp4等视频文件, 有时候需要抓取视频文件中的音频, 就得先用ffmpeg转换成mp3或ogg等格式, 再在audacity中进行处理. 这样稍微有点麻烦. 

audacity支持ffmpeg插件, 安装插件后, 就可以直接打开视频文件, 并自动转换成音频. 本以为安装插件会非常简单, 没想却折腾了好几天, 好在最终解决了, 本文记录我在解决该问题中的各种尝试.

# 官方方法

官方文档提示去网站[1]下载ffmpeg插件, 但是悲催的是访问该网站报`Error 1020`错误, 通过google cache可以看到文件的内容, 但是无法下载附件, 在网上翻了很久很久, 都没有找到替代的网址. 

# 自行编译

首先想尝试变异ffmpeg, 生成需要的包, 但是没有找到编译的手册, 使用ffmpeg官方提供的版本, 或者brew安装的版本均无法使用. 

其次尝试编译audacity, audacity关于ffmpeg参数`audacity_use_ffmpeg`提供了3个选项, `loaded, linked, off`, 尝试了loaded和linked, 均无法装载系统的 ffmpeg库. 

似乎找不到路了, 搁置数日. 

# 柳暗花明

audacity的开发工作迁移到了github, 在github的issue中搜索, 搜到了一个issue [2], 提供了ffmepg编译好的库, 以及编译脚本, 下载测试, 发现可以直接使用, 顺利解决问题. 感谢lllucius提供. 

# 吐槽

和网站[1] 一样, audacity官网, 以及论坛, 访问均报`Error 1020`错误, 好在可以通过google cache查看页面内容, 不知道去那里可以报bug. 

# 感怀

好久不写blog, 上一篇日志还是3年前的2018年了, 这两年来, 琐事越来越多, 能用来研究自己喜欢东西的时间越来越少, 甚憾. 

[1] https://lame.buanzo.org/
[2] https://github.com/audacity/audacity/pull/741