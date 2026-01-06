---
title: Docker Bridge 模式 IPv6 “不踢不动”的陈年顽疾
date: 2025-11-30T04:22:00
tags:
  - Linux开发
categories: 教程
draft: false
---
**前言**：
> 在 Docker 的世界里，IPv6 就像是一个被后妈虐待的孩子。
> 如果你尝试在自定义 Bridge 网络下开启 IPv6，你大概率会遇到这个极其离谱的现象：
> 容器刚启动时网络是通的，但过了一会儿就断了。
> 你在宿主机 `ping` 它一下，它就醒几秒，不踢它它就装死（FAILED 状态）。
> 本文将带你彻底根治这个“富贵病”，并顺便批判一下 Docker 烂到骨子里的 IPv6 实现。

## 1. 为什么 Docker 的 IPv6 会“装死”呢？

在标准的 IPv6 协议中，设备之间靠 **NDP（邻居发现协议）** 来寻找对方。当宿主机想给容器发包时，会广播问一句：“谁是 `fd00:1::2`？”

**Docker 的罪状有三：**
1. **网桥太“蠢”**：Docker 创建的 Linux 网桥经常漏掉组播包，导致容器听不见宿主机的召唤。
2. **邻居表老化**：Linux 内核默认只缓存几十秒邻居记录。一旦记录失效，宿主机再次发起探测，如果探测包在虚拟网桥里丢了，邻居表就会变成 `FAILED`。
3. **缺乏保活机制**：Docker 并没有为 Bridge 模式提供稳定的 NDP Proxy 或是保活代理。

**结论**：在 Docker 这种烂网桥环境下，动态邻居发现是不可靠的。
![屏幕截图 2026-01-05 164623.png](https://img.suyuri.com/my-blog-images/2026/01/05/695b7bd351289.jpeg)

---

## 2. 终极解法：强制“永久绑定” (Permanent Binding)

既然动态发现不靠谱，我们就把容器的 IP 和 MAC 地址直接**焊死**在宿主机的邻居表里。

我们要达到的效果是：**将邻居表状态从 `REACHABLE`（动态）强制提升为 `PERMANENT`（永久）**。一旦变为永久，内核就不会再去发送探测包，连接将永远保持激活。

为了不让手动操作累死人，我们需要一个自动化守护进程。

---

## 3. 落地实施方案

### 第一步：固定你的网桥与网段

在 `docker-compose.yml` 中，必须固定网桥名字，方便脚本识别。
```yaml
networks:  lucky:    enable_ipv6: true
    driver: bridge
    driver_opts:      com.docker.network.bridge.name: br-lucky # 固定网桥名
    ipam:      config:        - subnet: fd00:1:/80 # 你的自定义网段
          gateway: fd00:1:1
```
          
### 第二步：编写“守护神”脚本

创建 /usr/local/bin/docker-ipv6-monitor.sh，这个脚本会每 5 秒巡逻一次，发现新容器立刻将其邻居表状态“按死”在永久状态。
```
#!/bin/bash

# 配置你指定的网桥名
BRIDGE_NAME="br-lucky"

echo "Docker IPv6 Guardian started (Polling Mode)..."

while true; do
    # 确保网桥存在
    if [ ! -d "/sys/class/net/$BRIDGE_NAME" ]; then
        sleep 5
        continue
    fi

    # 扫描属于该网段的容器 IP 和 MAC
    docker ps -q | xargs -r docker inspect --format '{{range .NetworkSettings.Networks}}{{.GlobalIPv6Address}} {{.MacAddress}}{{end}}' | grep "^fd00:1" | while read -r ip mac; do
        
        if [ ! -z "$ip" ] && [ ! -z "$mac" ]; then
            # 核心绝杀：强制写入永久邻居表 (nud permanent)
            ip -6 neigh replace "$ip" lladdr "$mac" dev "$BRIDGE_NAME" nud permanent
        fi
    done

    sleep 5
done

```

### 第三步：解决 Systemd 的“水土不服”

为了防止脚本因为格式或环境问题无法自启，执行以下命令进行“大清洗”：
```
# 1. 洗掉可能存在的 Windows 换行符
sudo sed -i 's/\r$//' /usr/local/bin/docker-ipv6-monitor.sh
# 2. 确保 Shebang 正确
sudo sed -i '1c #!/bin/bash' /usr/local/bin/docker-ipv6-monitor.sh
# 3. 赋予权限
sudo chmod +x /usr/local/bin/docker-ipv6-monitor.sh
```
### 第四步：创建系统服务 (Systemd)

创建 /etc/systemd/system/docker-ipv6-guardian.service：
```
[Unit]
Description=Docker IPv6 Neighbor Table Guardian
After=docker.service
Requires=docker.service

[Service]
Type=simple
# 显式调用 bash 防止 203 错误
ExecStart=/bin/bash /usr/local/bin/docker-ipv6-monitor.sh
Restart=always
RestartSec=5
Environment="PATH=/usr/bin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

[Install]
WantedBy=multi-user.target
```
## 4. 运行与验证
```
sudo systemctl daemon-reload
sudo systemctl enable --now docker-ipv6-guardian
```
运行后，输入 ip -6 neigh show | grep fd00，你会看到满屏的 **PERMANENT**。  
这时候，你可以试着把容器晾在那一个小时，然后再去访问，你会发现它**秒通**！再也不需要宿主机去踢它那一脚了。

---

希望本文的“暴力美学”解法，能给同样被折磨的你节省一点宝贵的睡眠时间。