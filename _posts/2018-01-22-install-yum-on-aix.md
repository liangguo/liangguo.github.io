---
layout: page
title: "Install yum on AIX"
description: ""
category:
tags: [yum,aix]
categories: [aix]
---

# 缘起
今天在一个aix的机器上安装了一下yum，做一个记录，备查。

# 升级 rpm
Yum需要rpm 4.9.1以上, 而aix 6.1自带的rpm版本较低, 需要下载高版本的rpm.rte, 下载地址如下:

```
https://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/INSTALLP/ppc/rpm.rte.4.9.1.3
```

下载完成后将下载文件放到某一目录中，然后运行`smitty installp` 进行升级. 

# 安装yum

aix中RPM软件包都安装在/opt目录下, 所以建议将/opt文件系统扩大到2G以上.

yum包含在AIX Linux tool box中，也可以使用yum bundle 进行安装, 由于直接使用Linux tool box 
的包进行安装，需要手动解决依赖关系，所以我下载了yum bundle，下载地址如下:

```
https://public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/ezinstall/ppc/yum_bundle_v1.tar
```

下载完成后使用tar进行解压，解压后进入对应目录，允许`rpm -ivh *` 安装目录下所有软件包.

# 配置yum

yum的配置文件为`/opt/freeware/etc/yum/yum.conf`, 因为我的aix服务器无法直接连接互联
网，所以我把linuxtoolbox全部下载下来放到了`/mnt/aix/aixtoolbox`目录下，最终的配置文件如下: 

```
[main]
cachedir=/var/cache/yum
keepcache=1
debuglevel=2
logfile=/var/log/yum.log
exactarch=1
obsoletes=1

[AIX_Toolbox]
name=AIX generic repository
baseurl=file:///mnt/aix/aixtoolbox/RPMS/ppc/
enabled=1
gpgcheck=0

[AIX_Toolbox_noarch]
name=AIX noarch repository
baseurl=file:///mnt/aix/aixtoolbox/RPMS/noarch/
enabled=1
gpgcheck=0


[AIX_Toolbox_61]
name=AIX 6.1 specific repository
baseurl=file:///mnt/aix/aixtoolbox/RPMS/ppc-6.1/
enabled=1
gpgcheck=0

```

AIX toolbox是随机光盘的一部分, 如果丢失，可以去如下ftp网站下载:

```
ftp://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/
```

配置完成后就可以像在Linux中一样使用rpm了，例如使用如下命令更新仓库信息:

```
yum update
```

使用如命令安装bash:

```
yum install bash
```

从此不用再烦心rpm的依赖关系了

# 其他

Linux toolbox提供了一个安装yum的脚本, 链接如下，但是我没有使用.

```
https://ftp.software.ibm.com/aix/freeSoftware/aixtoolbox/ezinstall/ppc/yum.sh
```

[1] https://public.dhe.ibm.com/aix/freeSoftware/aixtoolbox/ezinstall/ppc/README-yum

