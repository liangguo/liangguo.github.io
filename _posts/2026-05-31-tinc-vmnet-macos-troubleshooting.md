---
layout: page
title: "macOS 26 上 tinc vmnet-host 模式的排查历程"
description: "记录在 macOS Tahoe 上排查 tinc VPN vmnet-host 单向连通问题的过程"
tags: [tinc, vpn, macos, vmnet]
categories: [networking]
---

# 缘起

我有几台 Mac 和 Linux 设备分布在不同地方, 一直用 tinc[1] 组建 mesh VPN 进行互联. 

tinc 在 Linux 上使用 TAP 设备实现 Layer 2 VPN, 一直运行得很好. 但在 macOS 上, TAP 设备依赖第三方内核扩展 (kext), 具体来说是 `tuntaposx` 这个项目提供的 tap.kext. Apple 从 macOS Catalina (10.15) 开始收紧了对第三方 kext 的限制, 需要关闭 SIP 才能加载; 到了 macOS Big Sur (11.0), Apple 正式废弃了 kext 机制, 推荐使用 System Extension 和 Network Extension 替代; 再到 Apple Silicon 的 Mac 上, 加载第三方 kext 更是需要降低安全等级到"Permissive Security", 步骤繁琐且每次系统更新后可能失效. 

所以换了新 Mac (macOS 26 Tahoe) 之后, 我就一直没有配置 tinc, 其他节点之间的 mesh 网络照常运行, 只是本机没有接入.

今天想把 tinc 重新设置起来. 搜索了一下, 发现 tinc 1.1-pre 分支已经加入了 vmnet.framework 的支持[2], 可以用 Apple 官方的 vmnet 框架实现 Layer 2, 不再需要 TAP kext. 于是想试一下，遇到的问题还蛮有趣的，是以为记。

# 配置

自行编译了带 vmnet 支持的 tinc, 配置文件 (`/usr/local/etc/tinc/do/tinc.conf`) 如下:

```
Name = beagle
ConnectTo = bcat
ConnectTo = gchk
ConnectTo = gcjp
Mode = switch
Port = 6656
DeviceType = vmnet-host
VmnetAddr = 10.100.6.6
VmnetNetmask = 255.255.255.0
```

`Mode = switch` 表示 Layer 2 交换模式, `DeviceType = vmnet-host` 使用 Apple vmnet 框架创建虚拟网络接口.

# 现象

tinc 启动后, macOS 创建了 `bridge100` 接口:

```
bridge100: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    inet 10.100.6.6/24
    member: vmenet0
```

从远端节点 ( 10.100.6.12) ping 本机, 能收到回复. 但从本机 ping 远端:

```
(base) liang@Beagle ~ % ping 10.100.6.12
PING 10.100.6.12 (10.100.6.12): 56 data bytes
ping: sendto: No route to host
```

errno 65, `EHOSTUNREACH`. 查看 `netstat -s` 中的 Opkts 计数器, 发现出站包数量不增加——数据包根本没有从 bridge100 发出去.

# 排查过程

## 1. 排除 tinc 连接问题

首先确认 tinc 的 meta connection 正常:

```
# tinc -n do dump connections
gchk at xxx.xxxx.xxx.xxx port 655 
```

控制平面没问题, 问题出在本机网络栈.

## 2. 排除路由问题

路由表中有到 10.100.6.0/24 的路由, 指向 bridge100. 尝试用 `IP_BOUND_IF` 强制绑定到 bridge100 发送 UDP 包, 仍然返回 EHOSTUNREACH. 说明不是路由问题, 而是内核拒绝从 bridge100 发出数据包.

## 3. vmnet-shared 模式测试

将 `DeviceType` 改为 `vmnet-shared`, 同样的问题. bridge100 出站被拒绝.

## 4. 发现 root 用户可以通

偶然发现用 root 用户 ping 是可以通的:

```
Beagle:do root# ping 10.100.6.12
PING 10.100.6.12 (10.100.6.12): 56 data bytes
64 bytes from 10.100.6.12: icmp_seq=0 ttl=64 time=2215.017 ms
64 bytes from 10.100.6.12: icmp_seq=2 ttl=64 time=250.369 ms
```

而普通用户:

```
(base) liang@Beagle ~ % ping 10.100.6.12
PING 10.100.6.12 (10.100.6.12): 56 data bytes
ping: sendto: No route to host
```

## 5. 定位到 Local Network Privacy

macOS 26 引入了更严格的 Local Network Privacy (LNP) 机制. 普通用户进程访问本地网络需要获得授权. root 用户不受此限制.

bridge100 是 vmnet 框架创建的虚拟桥接接口, 它被 macOS 视为一个独立的"本地网络". 终端应用 (iTerm/Terminal) 可能没有被授权访问这个网络, 导致普通用户的出站包被内核静默丢弃, 返回 EHOSTUNREACH.

# 解决方案

既然问题出在 Local Network Privacy 对 bridge100 的权限控制, 解决方法就是绕过 LNP:

1. 打开 **System Settings -> Privacy & Security -> Local Network**, 找到终端应用 (iTerm/Terminal), 授权访问本地网络.

2. 重启 iTerm，普通用户即可用访问 tinc 网络上的其他服务。

# AI 辅助排查

这次排查过程中, 我大量借助了 AI 助手. 坦白说, 如果没有 AI, 这个问题我可能要折腾好几天.

## AI 提出的猜想与验证

排查过程中, AI 先后提出了几个方向:

1. **bridge100 缺少 member 导致单向不通**: AI 分析 `ifconfig` 输出后, 认为 bridge 只有 vmenet0 一个 member, tinc 虚拟网卡没有被桥接进来. 实际验证发现 vmnet 模式下 bridge 的 member 就是这样的, 这个猜想不成立.

2. **vmnet 源 IP 检查限制**: AI 从 Apple 文档和论坛信息中找到 vmnet host/shared 模式有源 IP 过滤 ("Packets sent from a different IPv4 address are dropped by the system"), 怀疑 tinc 的非标准用法触发了这个限制. 这个方向有参考价值, 但最终不是导致问题的直接原因.

3. **改用 utun 模式**: AI 建议放弃 vmnet 改用 Mode=router + DeviceType=utun. 这是一个可行的备选方案, 但我更想让 Layer 2 跑起来.

4. **Local Network Privacy**: 当我发现 root 能通而普通用户不能通后, AI 迅速将问题定位到 macOS 26 的 LNP 机制. 这个判断是正确的.

此外, AI 帮我快速汇总了 Apple vmnet 头文件、QEMU vmnet 后端实现、以及 tinc vmnet 源码 (commit 6707f23) 中的关键信息, 省去了大量翻文档和读源码的时间.

# 感想

AI 非常强大，如果不是 AI 的帮助，我可能中途就放弃了。但是 AI 给的建议并不总是对的，从上面 AI 给的建议中就可以看到，有一些建议甚至走向了错误的方向。

本文由 AI 整理排查历程，并一起修改而成。

# 参考咨询
[1] https://www.tinc-vpn.org/

[2] https://github.com/gsliepen/tinc/commit/6707f23
