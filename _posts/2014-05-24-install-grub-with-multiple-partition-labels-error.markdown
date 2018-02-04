---
layout: page
title:  "Install grub with multiple partition labels error"
comments: true
date:   2014-05-24 01:57:25
categories: [debian]
---

今天在试图将`root文件系统`迁移到一块空白磁盘时，遇到了一个安装`grub`的错误:


	root@bcat:~# grub-install /dev/sdc
	Installing for i386-pc platform.
	grub-install: warning: Attempting to install GRUB to a disk with multiple partition labels.  This is not supported yet..
	grub-install: warning: Embedding is not possible.  GRUB can only be installed in this setup by using blocklists.  However, blocklists are UNRELIABLE and their use is discouraged..
	grub-install: error: will not proceed with blocklists.

非常奇怪的一个问题，在网上乱找，发现[这个][1]帖子，猛然想起这个磁盘曾经被不小心写
入过ISO文件。于是按照帖子的内容执行了dd

	dd if=/dev/zero of=/dev/sdc seek=1 count=2047 bs=1b

这个命令会在磁盘的开始写入1M的zero，由于Linux下使用`fdisk`进行分区时会空出前2048个块，
所以执行这个命令不会影响现有的数据，执行完毕后，grub可以顺利安装

	root@bcat:~# grub-install /dev/sdc
	Installing for i386-pc platform.
	Installation finished. No error reported.
	root@bcat:~#  


[1]: http://lilydjwg.is-programmer.com/tag/%E6%95%B0%E6%8D%AE%E6%81%A2%E5%A4%8D

