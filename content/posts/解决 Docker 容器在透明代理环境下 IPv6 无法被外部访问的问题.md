---
title: Docker 容器在透明代理环境下 IPv6 无法被外部访问的问题
date: 2025-09-20T14:00:00
draft: false
tags:
  - Linux开发
categories:
  - 教程
---
**前言**
>在折腾家庭服务器或 VPS 时，我们经常会遇到一种令人头秃的情况：宿主机本身的 IPv6 访问完全正常，但在 Docker 容器（例如 Lucky 反代、Nginx 等）内部的服务，明明监听了 IPv6 端口，外部却死活 TCPing 不通。
>特别是当你还在服务器上运行了 **Mihomo (Clash)** 等透明代理工具时，这个问题尤为明显。本文将分享一个基于 `ip6tables` 和策略路由的 Shell 脚本，完美解决 Docker 虚拟网桥下的 IPv6 直连问题。

## 🚫 问题现象

- 宿主机 IPv6 正常，可以 `curl -6 google.com`，外部也能 ping 通宿主机。
    
- Docker 容器部署在默认网桥 (`docker0`) 或自定义网桥（如 `br-lucky`）上。
    
- 容器端口已映射（例如 `-p 80:80`）。
    
- **症状**：外部通过 IPv6 访问容器端口超时（Time out），TCPing 失败。
    
- **环境**：Linux 服务器，运行了 Docker，且很可能运行了类似 Mihomo 的透明代理接管了流量。
    

## 🧐 原因分析

当你在服务器上运行透明代理时，代理程序通常会修改路由表或使用 TProxy/Tun 模式接管进出流量。

1. **入站流量**：外部请求通过 IPv6 到达宿主机网卡（如 `eth1`），被转发给 Docker 网桥，最终到达容器。
    
2. **出站流量（回包）**：容器处理完请求准备回包时，数据包从 Docker 网桥出来。
    
3. **路由冲突**：由于透明代理的规则（通常是策略路由），这个回包可能会被误判为需要“代理”的流量，或者被错误的路由表引导，导致它无法原路返回给外部请求者，从而造成连接中断。
    

我们需要做的，就是**给“回包”打上标记，让它绕过代理规则，强制走主路由表直连出去**。

## 🛠️ 解决方案：一键修复脚本

针对这个问题，我编写了一个 Bash 脚本。它主要做了两件事：

1. **打标（Marking）**：给从外部接口进来的连接打上标记（`0x114`），并确保 Docker 容器回应的数据包继承这个标记。
    
2. **策略路由（Policy Routing）**：告诉系统，凡是带有 `0x114` 标记的数据包，直接查 `main` 路由表，不要走代理的路由表。
    

### 脚本内容

创建一个名为 `fix_docker_ipv6.sh` 的文件：

Bash

```
#!/bin/bash

# 确保以 root 权限运行
if [ "$EUID" -ne 0 ]; then
  echo "请使用 sudo 运行此脚本"
  exit 1
fi

echo "正在配置 Mihomo/Docker IPv6 直连规则..."

# ===========================
# 1. 配置变量 (请根据实际情况修改)
# ===========================
# 外网物理网卡名称 (例如 eth0, eth1, enp3s0)
WAN_INTERFACE="eth1"
# 自定义 Docker 网桥名称 (Lucky 反代所在的网桥)
CUSTOM_BRIDGE="br-lucky"
# 标记值 (16进制)
FWMARK="0x114"

# ===========================
# 2. 路由策略 (Policy Routing)
# ===========================
# 清理旧规则 (避免重复)
ip -6 rule del fwmark $FWMARK lookup main pref 8900 2>/dev/null
# 添加新规则：凡是标记为 0x114 的 IPv6 包，强制查询 main 路由表
ip -6 rule add fwmark $FWMARK lookup main pref 8900

# ===========================
# 3. 定义 iptables 辅助函数
# ===========================
add_rule() {
    local chain=$1
    shift
    # check 检查规则是否存在，不存在则 append
    ip6tables -t mangle -C "$chain" "$@" 2>/dev/null
    if [ $? -ne 0 ]; then
        ip6tables -t mangle -A "$chain" "$@"
        echo "已添加规则: $chain $@"
    else
        echo "规则已存在，跳过: $chain $@"
    fi
}

# ===========================
# 4. 配置 Mangle 表规则
# ===========================

# [进站打标]
# 当数据包从外网接口进来时，给整个连接 (Connection) 打上标记
add_rule PREROUTING -i $WAN_INTERFACE -j CONNMARK --set-mark $FWMARK

# [Docker 查标]
# 当数据包从默认 Docker 网桥出来时，将连接的标记恢复到数据包上
add_rule PREROUTING -i docker0 -j CONNMARK --restore-mark

# [自定义网桥查标]
# 当数据包从 Lucky 网桥出来时，同样恢复标记
# 注意：即使网桥当前未启动，添加此规则也是安全的
add_rule PREROUTING -i $CUSTOM_BRIDGE -j CONNMARK --restore-mark

# [宿主机查标]
# 宿主机自身发出的包也尝试恢复标记 (用于本机访问)
add_rule OUTPUT -j CONNMARK --restore-mark

echo "IPv6 修复脚本执行完成。"
```

### 关键配置说明

- `ip -6 rule add fwmark 0x114 lookup main`：这是核心。它告诉 Linux 内核，只要看到标记是 `0x114` 的 IPv6 包，就忽略掉其他的路由表（比如 Clash 创建的路由表），直接去查主路由表，保证数据包能正确扔回给网关。
    
- `CONNMARK --set-mark`：在 PREROUTING 链（数据包刚进网卡时）记录标记。
    
- `CONNMARK --restore-mark`：在 Docker 容器回包时，把之前记录的标记“贴”回到回程的数据包上。
    

## 🚀 如何使用

1. **保存脚本**：将上面的代码保存为 `fix_docker_ipv6.sh`。
    
2. **修改网卡名称**：**务必**检查脚本中的 `WAN_INTERFACE="eth1"` 和 `CUSTOM_BRIDGE="br-lucky"` 是否与你的实际环境一致。可以使用 `ip addr` 查看你的网卡名称。
    
3. **赋予权限**：
    
    Bash
    
    ```
    chmod +x fix_docker_ipv6.sh
    ```
    
4. **运行脚本**：
    
    Bash
    
    ```
    sudo ./fix_docker_ipv6.sh
    ```
    

运行后，再次尝试从外部 IPv6 TCPing 你的 Lucky 端口，应该就能通了！

## 💡 持久化建议

重启服务器后，`iptables` 规则和路由策略通常会丢失。建议将此脚本加入开机自启：

- **方法一**：添加到 `/etc/rc.local` (如果系统支持)。
    
- **方法二**：创建一个简单的 systemd service。
    
- **方法三**：如果是 OpenWrt 或特定 NAS 系统，放在对应的“自定义脚本”设置中。
    

---

通过这个简单的脚本，我们利用 Linux 强大的网络标记功能，成功打通了 Docker 容器的 IPv6“任督二脉”，既保留了透明代理的功能，又实现了服务的直连访问。希望这对你有所帮助！