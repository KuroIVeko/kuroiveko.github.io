---
title: "异地组网突然卡成 PPT：一次被运营商 UDP QoS 误伤的排查记录"
date: 2026-09-11T20:00:00+08:00
draft: false
tags: ["Tailscale", "Headscale", "sing-box", "网络", "QoS", "踩坑"]
categories: ["折腾"]
summary: "Headscale + Tailscale 组网用得好好的，某天突然卡到无法忍受。怀疑过 MTU，最后发现是运营商的 UDP QoS——而解决办法只有一行配置。"
---

## 起因：用得好好的，突然就卡了

我的异地组网方案是自建 Headscale 做控制面，客户端用的是 sing-box 自带的 Tailscale endpoint，家里的电脑和另一台机器（下文叫它 `nas`）之间走 Tailscale 直连。

一直以来都很稳，远程桌面、SSH、传文件都没什么问题。直到某一天，连 `nas` 突然变得奇卡无比：SSH 敲一个字母要等半天才回显，远程桌面一顿一顿的，像在看 PPT。

更诡异的是：**延迟看起来很正常**。

## 第一个怀疑对象：MTU

组网卡顿，第一反应就是 MTU。隧道套隧道，包太大被分片或者被黑洞丢掉，是异地组网的老生常谈了。

先看了一下以太网接口的 MTU：

```text
InterfaceAlias AddressFamily NlMtu
-------------- ------------- -----
以太网                     IPv6  1480
以太网                     IPv4  1500
```

IPv6 是 1480，有点奇怪但不算离谱（大概率是路由器 RA 下发的）。为了排除干扰，直接把以太网的 MTU 临时砍到 1280：

```powershell
netsh interface ipv4 set subinterface "以太网" mtu=1280 store=active
netsh interface ipv6 set subinterface "以太网" mtu=1280 store=active
```

> `store=active` 表示只在本次启动生效，重启或禁用/启用网卡后自动恢复，适合做实验。

结果：**该卡还是卡。**

## 用数据说话：到底是不是 MTU 的锅

先确认到 `nas` 的流量走哪张网卡：

```text
InterfaceAlias    : tailscale0
DestinationPrefix : 100.64.0.2/32
```

走的是 `tailscale0` 隧道，意料之中。然后做了三组测试。

### 1. 普通 ping：小包也在丢

```text
Ping statistics for 100.64.0.2:
    Packets: Sent = 20, Received = 12, Lost = 8 (40% loss),
Approximate round trip times in milli-seconds:
    Minimum = 49ms, Maximum = 61ms, Average = 53ms
```

**32 字节的小包，丢了 40%。** 延迟倒是稳稳的 50ms 出头。

这一条基本就把 MTU 排除了——MTU 问题的典型症状是**小包正常、大包出事**，32 字节的包再怎么也不会超过 MTU。

### 2. 带 DF 标志的 ping：分片行为正常

用 `ping -f -l <size>` 逐步加大包长：

```text
1000 => Reply from 100.64.0.2: bytes=1000 time=51ms TTL=128
1150 => Reply from 100.64.0.2: bytes=1150 time=52ms TTL=128
1180 => Packet needs to be fragmented but DF set.
1200 => Packet needs to be fragmented but DF set.
```

大包在本机就直接报 "needs to be fragmented"，而不是发出去以后石沉大海。这说明路径 MTU 发现是正常工作的，TCP 会自己把 MSS 调小，不会出现"连接建立了但传数据就卡死"的 PMTU 黑洞。

### 3. ping 本地网关：本地链路很干净

```text
Ping statistics for 192.168.31.1:
    Packets: Sent = 20, Received = 20, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

网卡计数器也看了，收发错误、丢弃全是 0，千兆全双工。本机和本地链路没毛病。

### 丢包长什么样

每 0.5 秒 ping 一次，1000 字节，连续 60 次，`.` 是成功、`X` 是丢包：

```text
..XX......X....X..XXXX..XX.X..X..X.X....X.X..X.....X.X.X.XXX
```

23/60，约 38%，而且分布相当随机，没有固定周期。

再从 `nas` 那边反向 ping 回来——**情况一模一样**：延迟好看，丢包惨烈。

### 小结

| 现象 | 说明 |
|---|---|
| 小包也丢 40% | 不是 MTU 的问题 |
| 大包能正确报分片 | 没有 PMTU 黑洞 |
| 本地网关 0 丢包 | 本机、网卡、局域网都没问题 |
| 两端对称丢包，延迟稳定 | 路是通的，但包被人"挑着扔" |

延迟稳定 + 大量随机丢包 + 只影响隧道，这个组合太眼熟了：**运营商对 UDP 做了 QoS。** Tailscale 底层是 WireGuard，WireGuard 全程跑 UDP，首当其冲。

MTU 改回去，换个方向查。

## 找轮子：UDP 被限速有什么现成方案

搜了一圈，别人踩过的坑和验证过的方案大概是这些：

### udp2raw / phantun：把 UDP 伪装成 TCP

- [udp2raw](https://github.com/wangyu-/udp2raw)：老牌工具，FakeTCP 模式会模拟三次握手和 seq/ack，防火墙看来是 TCP，本质还是 UDP（不重传、无拥塞控制）。
- [phantun](https://github.com/dndx/phantun)：Rust 写的后起之秀，每个包只多 12 字节头（udp2raw 是 44 字节），性能更好。

但是有个硬伤：**它们很难直接套在 Tailscale 上。** Tailscale 的数据面（magicsock）自己管理 UDP socket、自己打洞、自己选路，外部工具截不到它的底层流量。有人为此[专门折腾过](https://blog.3gxk.net/archives/p2pzhi-lian-xu-ni-zu-wang-yu-jian-yun-ying-shang-udpxian-su-de-jie-jue-fang-an)，最后放弃了 udp2raw。phantun 的成功案例（比如[这篇](https://cloud.tencent.com/developer/article/2153908)）都是给裸 WireGuard 用的，而且至少需要一端有公网 IP。

### 自建 DERP + 强制中继

Tailscale 体系里唯一原生的 TCP 通道就是 DERP 中继（走 HTTPS/443）。在国内 VPS 上自建 `derper`，加到 Headscale 的 DERP map 里，然后在客户端设置环境变量：

```bash
TS_DEBUG_ALWAYS_USE_DERP=true
```

强制所有流量走中继，UDP QoS 就管不着了。代价是多绕一跳，带宽受限于 VPS。

### 换组网工具：EasyTier

[EasyTier](https://github.com/EasyTier/EasyTier) 从 v2.5.0 开始实验性地支持 FakeTCP 隧道（致谢里写着 udp2raw 和 phantun），同时支持 TCP / WS / WSS / QUIC / WG 等多种传输。想要"组网 + 伪 TCP"一体化的话，这是目前最现成的轮子，代价是得放弃 Headscale。

### 最简单的：换端口到 UDP 443

这是[这篇 OpenWrt 调优文章](https://www.cnblogs.com/zqingyang/p/19243450)和[这篇](https://log.ozo.ooo/posts/tailscale-randomize-port/)里实测有效的办法：

> Tailscale 默认监听 UDP 41641，容易被识别为 P2P 流量。改成 UDP 443，运营商会把它当成 QUIC（HTTP/3）网页流量，而 QUIC 流量运营商是不敢随便限的——限了 YouTube 和一堆 App 都得卡。

成本最低，先试这个。

## 解决：一行配置

普通 tailscaled 改端口是加 `--port=443` 参数。但我的客户端是 **sing-box 自带的 Tailscale endpoint**，查了一下[官方文档](https://sing-box.sagernet.org/configuration/endpoint/tailscale/)，从 **sing-box 1.14.0** 开始有一个 `listen_port` 字段：

> The UDP port to listen on for WireGuard and peer-to-peer traffic.

改配置：

```json
{
  "endpoints": [
    {
      "type": "tailscale",
      "tag": "tailscale-out",
      "auth_key": "<your-auth-key>",
      "control_url": "https://headscale.example.com",
      "system_interface": true,
      "listen_port": 443
    }
  ]
}
```

改之前可以先用 `sing-box check -c config.json` 确认你的版本认这个字段，顺便看一眼本地 UDP 443 有没有被别的程序占用。

重启 sing-box。

**如丝般顺滑。**

SSH 秒回显，远程桌面不卡了，前后对比简直两个世界。

> 严格来说，本地监听端口不等于公网上的端口——家里路由器做 NAT 时可能会把源端口改掉。如果只改本地端口没效果，可以在路由器上把 UDP 443 转发到这台机器，或者开启 UPnP / NAT-PMP 让 Tailscale 自己申请映射。我这里是直接改完就好了，运气不错。

## 复盘：为什么会被限？

### 运营商在抓的是"特征"，不是"你"

这两年运营商在严查家宽跑 PCDN（把家里上行带宽租出去赚钱）。PCDN 的流量特征大概是：

- 家宽 IP
- 非标准端口
- 长时间、持续地对外跑 UDP

而 Tailscale 直连的流量，**恰好完美符合这组特征**。规则并不知道你是在连自己的 NAS 还是在跑 PCDN，特征对上了就一起限。

所以——**我们是被误伤的正常用户。** 同样容易中招的还有 ZeroTier、裸 WireGuard、Moonlight/Parsec 串流、部分游戏联机。

改成 443 的本质，是去掉了"非标准端口"这个最显眼的特征，让流量看起来像普通的 QUIC 访问。

### 为什么手机蜂窝网络连过去一点都不卡？

后来发现，手机用蜂窝网络、同样通过 sing-box 的 Tailscale 连 `nas`，一直都很流畅。这更说明 QoS 不是一刀切，而是有逻辑的。可能的原因有两个：

1. **手机大概率根本没走 UDP 直连。** 蜂窝网络基本都在运营商级 NAT（CGNAT）后面，而且多是对称型 NAT，Tailscale 很难打洞成功，会回落到 DERP 中继——DERP 走的是 TCP 443，UDP QoS 自然管不到。可以在另一端用 `tailscale status` 看手机那一行是 `relay "xxx"` 还是 `direct IP:port`。
2. **家宽和移动网络的管控策略本来就不是一套。** 防 PCDN 的规则主要针对家宽上行，移动网络按流量计费，有自己的管控方式。

## 443 也会被限吗？

有可能，但概率低很多：

- **DPI 可以识别**：WireGuard 的包头格式很固定，和真 QUIC 长得不一样。运营商真想做深度包检测，端口伪装是骗不过去的。
- **按流量模式判断**：家宽对家宽、长时间大流量 UDP，端口是什么可能都不重要。
- **按 IP 限**：有人反馈过大流量 UDP 被电信整个 IP 拉进小黑屋，换什么端口都没用。

所以留好退路，按成本从低到高：

1. 443 也变慢了 → 试试 `"listen_port": 0`，每次启动随机选端口。
2. 还是不行 → 自建国内 DERP + `TS_DEBUG_ALWAYS_USE_DERP=true`，全程 TCP。
3. 实在不行 → 换 EasyTier，用 FakeTCP 从包头层面伪装成 TCP。
4. 另外也可以打客服说明自己是正常用户、没跑 PCDN，要求解除限速——有人成功过，看地区看运气。

## 总结

- **异地组网突然卡，先别急着怪 MTU。** 用 ping 小包测一下丢包：小包都在丢，那就不是 MTU 的事。
- **延迟稳定 + 大量随机丢包 + 本地链路干净**，基本可以锁定运营商 UDP QoS。
- **Tailscale 很难套 udp2raw / phantun**，别在这条路上浪费时间。
- **最便宜的解法是把监听端口改成 UDP 443**，sing-box 1.14.0+ 用 `listen_port`，tailscaled 用 `--port=443`。
- 被限了不代表你做错了什么，你只是长得像坏人。

## 参考

- [P2P虚拟组网时遇见运营商UDP限速的解决方案 - 竹影流浪](https://blog.3gxk.net/archives/p2pzhi-lian-xu-ni-zu-wang-yu-jian-yun-ying-shang-udpxian-su-de-jie-jue-fang-an)
- [OpenWrt 下 Tailscale 性能调优全记录 - 博客园](https://www.cnblogs.com/zqingyang/p/19243450)
- [Tailscale 端口被封？开启端口随机化恢复直连速度](https://log.ozo.ooo/posts/tailscale-randomize-port/)
- [TailScale 组网时 IPv6 遇到 UDP QOS - V2EX](https://www.v2ex.com/t/1122948)
- [突破运营商 QoS 封锁，WireGuard 真有"一套"！- 腾讯云](https://cloud.tencent.com/developer/article/2153908)
- [sing-box Tailscale endpoint 文档](https://sing-box.sagernet.org/configuration/endpoint/tailscale/)
- [phantun](https://github.com/dndx/phantun) / [udp2raw](https://github.com/wangyu-/udp2raw) / [EasyTier](https://github.com/EasyTier/EasyTier)
