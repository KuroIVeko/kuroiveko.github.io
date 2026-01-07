---
title: WSL2 镜像网络 + Mihomo TUN + Docker Bridge 共存
date: 2025-11-15T10:35:00
tags:
  - Linux开发
categories:
  - 教程
draft: false
---
**前言**
>最近在折腾 WSL2 的 **Mirrored（镜像网络）** 模式。这个模式本该是完美的：WSL 和 Windows 共享 IP，端口互通，简直是开发者的福音。
>然而，当我开启 **Mihomo (Clash内核) 的 TUN 模式**，并把 Docker 容器（比如 Lucky、Alist）放入 **Bridge（虚拟网桥）** 网络后，噩梦开始了：
>* **现象**：Mihomo TUN 一开，外部（局域网/宿主机）访问容器端口直接断连。
>* **诡异点**：容器内部访问公网（翻墙）是正常的。
>* **更诡异点**：以前用 Host 模式好好的，一切换到 Bridge 模式就挂。
>折腾了一整天，试过改 DNS、改路由表 `route-exclude`、改 `strict-route`，统统无效。最后终于通过 `iptables` 打标找到了**终极解法**。

## 根本原因：Mihomo 的“强盗逻辑”与 Docker 的“身份危机”

问题的核心在于 **WSL2 镜像模式的网络栈是共享的**。

1.  **Mihomo 的 TUN 劫持逻辑**：
    Mihomo 开启 TUN 后，会下发一条极霸道的策略路由（`ip rule`）：
     `not from all iif lo lookup 2022`
     **翻译**：只要流量不是从 `lo`（本机回环）接口进来的，统统给我进代理隧道。

2.  **Docker Bridge 的尴尬**：
    * **Host 模式下**：容器共享宿主机网络，流量被内核视为“本机流量”（近似 `lo` 或 `eth0`），侥幸逃过 Mihomo 的劫持。
    * **Bridge 模式下**：容器流量是从 `br-xxxx`（虚拟网桥）出来的。在 Mihomo 眼里，这就是“外来流量”，直接被无脑劫持进 TUN。

3.  **为什么 `route-exclude` 没用？**
    Mihomo 的配置 `route-exclude-address` 排除的是**目的地 IP**。
    * **场景**：外部访问 Lucky -> Lucky 回包给外部。
    * **死结**：Mihomo 是在**入口**处（根据接口 `iif`）就进行了劫持，根本还没来得及看你的**出口**（目的地 IP）是不是在白名单里，数据包就已经进了黑洞。

## 解决方案：iptables 端口精准分流

既然 Mihomo 按“接口”一刀切，我们就用 Linux 底层的 `iptables` 按“端口”做精准手术。

**核心思路**：
利用 `mangle` 表，给特定服务端口（如 16601, 443, 3478）的数据包打上 **`fwmark`（防火墙标记）**。然后告诉 Linux 内核：“凡是带这个标记的包，直接走主路由，**严禁**进入 Mihomo 的代理表”。

### 终极一键脚本

这个脚本解决了以下痛点：
1.  **TCP 直连**：保证 Web 面板、HTTPS 服务能被外部访问。
2.  **UDP 直连**：保证 **STUN (3478)** 能探测真实 IP，保证 **Tailscale (41641)** 能 P2P 打洞。
3.  **自动持久化**：解决重启 WSL 规则丢失的问题。

#### 1. 创建脚本
在 WSL 中执行：
```bash
sudo tee /usr/local/bin/fix-lucky-net.sh <<'EOF'
#!/bin/bash

# ================= 配置区域 =================
# TCP 端口：Web面板、反代服务 (如 Lucky后台, 443)
TCP_PORTS="16601,8405,8443,443"

# UDP 端口：STUN探测、VPN打洞 (如 Tailscale)
# 注意：3478 必须直连，否则无法获取真实公网IP
UDP_PORTS="3478,41641"
# ===========================================

# 1. 清理旧规则（防止重复堆积）
ip rule del fwmark 0x1 lookup main priority 8000 2>/dev/null
iptables -t mangle -F PREROUTING
iptables -t mangle -F OUTPUT

# 2. 添加策略路由：见到 0x1 标记，直接查主表(main)，绕过代理
ip rule add fwmark 0x1 lookup main priority 8000

# 3. 【TCP】打标规则
if [ -n "$TCP_PORTS" ]; then
    # 外部流量进来 (PREROUTING)
    iptables -t mangle -A PREROUTING -p tcp -m multiport --sports $TCP_PORTS -j MARK --set-mark 0x1
    # 本机流量出去 (OUTPUT)
    iptables -t mangle -A OUTPUT -p tcp -m multiport --sports $TCP_PORTS -j MARK --set-mark 0x1
fi

# 4. 【UDP】打标规则 (关键！)
if [ -n "$UDP_PORTS" ]; then
    iptables -t mangle -A PREROUTING -p udp -m multiport --sports $UDP_PORTS -j MARK --set-mark 0x1
    iptables -t mangle -A OUTPUT -p udp -m multiport --sports $UDP_PORTS -j MARK --set-mark 0x1
fi

# 5. 内核参数修正 (防止开启 TUN 后回包因路径校验被丢弃)
sysctl -w net.ipv4.conf.all.rp_filter=0 >/dev/null 2>&1
sysctl -w net.ipv4.conf.eth0.rp_filter=0 >/dev/null 2>&1

echo "[$(date)] Network Rules Applied. TCP: $TCP_PORTS | UDP: $UDP_PORTS" >> /tmp/lucky-fix.log
EOF