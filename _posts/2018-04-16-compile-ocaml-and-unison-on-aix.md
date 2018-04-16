---
layout: page
title: "Compile OCaml and Unison on AIX"
description: ""
category:
tags: [ocaml, unison, aix, Linux]
categories: [aix]
---

# 缘起
我有两台机器, 一台AIX, 一台Linux, 2台机器距离比较远, 延迟能在200ms以上, 有时候需要在两个系统之间共享一些数据, 最常用的方法是使用ssh+rsync进行同步, 或者使用nfs, 使用同一份文件系统. 但ssh+rsync难于自动化, 每次变更需要手动处理. nfs则作为nfs client的机器每次文件操作就需要很长时间. 体验比较差. 

Unison File Synchronizer[1] 是一个比rsync更方便的同步工具, 并且能够定时进行同步, 或检测到文件变化后自动同步. 本文描述aix系统上unison的安装与使用. 

# 安装 OCaml

Unison使用ocaml编写, 在网上搜索, 没有搜到ocaml是否可以在aix上编译的信息, 官方网站也没有相关的说明. 经测试, ocaml是可以在aix上编译运行. 具体步骤如下:

1) 安装依赖的软件包gcc, gnu make, 如果已配置yum, 这两个软件可以使用yum安装, aix下yum的配置方式可以参考[2]. 安装完成后, gcc和make的版本如下:

```
# rpm -qa |egrep '(make|gcc)'
gcc-cpp-6.3.0-1.ppc
make-4.2.1-4.ppc
libgcc-6.3.0-1.ppc
gcc-6.3.0-1.ppc

```

2) 下载OCaml, 我下载的是4.06, 最新的稳定版, 下载地址为: 

```
http://caml.inria.fr/pub/distrib/ocaml-4.06/ocaml-4.06.0.tar.gz
```

下载完成后将下载到的文件解压. 

3) 编译OCaml

进入到解压的OCaml目录, 依次运行如下命令

```
./configure
gmake world 
gmake world.opt
gmake install
```

gmake install会将OCaml安装到/usr/local目录, 如需安装到其他目录, 可以设置`./configure`的参数. 

gnu make安装后的可执行文件是gmake, 而不是make, 需要注意. 

如果直接运行gmake, OCaml给出的提示如下: 

```
# gmake
Please refer to the installation instructions in file INSTALL.
If you've just unpacked the distribution, something like
        ./configure
        make world.opt
        make install
should work.  But see the file INSTALL for more details.

```

如果按照提示使用`make world.opt`编译, 则很多需要的命令会没有, 关于`make world`和`make world.opt`的差别, 可以参考OCaml安装文档[3].

# 安装unison 

Unison的下载地址为: 

```
https://github.com/bcpierce00/unison/archive/v2.51.2.tar.gz
```

下载完成后使用tar进行解压. 然后运行如下命令编译安装: 

```
export PATH=/usr/local/bin:$PATH
gmake 
gmake install
```

# 使用unison

同步2台机器上的一个目录可以使用如下命令:

```
unison <source dir> <dest dir> -auto -batch
```

例如: 

```
unison ssh://root@bcat/export/test test -auto -batch
```

定期运行unison可以使用`-repeat` 参数

```
unison <source dir> <dest dir> -auto -batch <sec> 

```

对于经常需要进行的unison同步, 可以设置unison配置文件, 一个简单的配置文件示例如下: 

```
# Unison preferences file
root = /home/liang/us
root = ssh://liang@bcat//y/home2/liang/us
repeat = watch
```

更高级的用法, 请参考unison官方使用手册[4]. 


# 其他

unison 只能和同版本的unison进行同步, 不同版本的unison之间无法进行同步. 

如果进行unison同步的两端为linux或windows, 可以使用unison fsmonitor模块, 这样只有在文件系统发生变化时才会发起同步, 减少资源消耗. 


[1] https://www.cis.upenn.edu/~bcpierce/unison/

[2] https://liangguo.github.io/aix/2018/01/22/install-yum-on-aix.html

[3] https://caml.inria.fr/distrib/ocaml-4.06/notes/INSTALL.adoc

[4] https://www.cis.upenn.edu/~bcpierce/unison/download/releases/stable/unison-manual.html
