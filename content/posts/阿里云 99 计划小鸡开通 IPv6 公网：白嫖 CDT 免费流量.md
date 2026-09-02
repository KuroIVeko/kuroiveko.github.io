---

title: 阿里云 99 计划小鸡开通 IPv6 公网：白嫖 CDT 免费流量的完整踩坑记 
date: 2026-09-02 
tags:
- 阿里云
- IPv6
- CDT
- VPS
- 教程

---


阿里云 99 计划那台经济型 e 实例，默认只给 IPv4。想给它加个公网 IPv6，看起来是个小事——控制台点几下嘛。

实际折腾下来，踩了一路的坑：**四层配置缺一不可、计费方式选错白配、免费额度其实只有宣传的十分之一、安全组 v4 规则对 v6 完全无效**……

这篇把整个过程和所有坑记录下来。文末附一个**流量熔断脚本**，防止免费额度用超后被按 0.8 元/GB 扣费。

---

## 一、我为什么要折腾它的 IPv6

先交代一句动机，不感兴趣的可以直接跳到第二节。

我在这台小鸡上自建了 tailscale 的中继服务（headscale + DERP）。它的作用是帮我散落各地的设备互相打洞直连；**打不通洞的时候，流量就得绕道这台服务器中继转发**——慢、费流量，还占服务器带宽。

问题出在：**这台机器只有 IPv4**。而 tailscale 客户端探测自己网络环境的方式是"照镜子"——往中继服务器发个包，服务器告诉它"我看到你的来源地址是 XXX"。服务器没有 IPv6，客户端的 v6 包就压根发不到它，也就永远收不到 v6 的回音，于是判定"我没有 IPv6"，**连 v6 候选地址都不生成**。结果就是两台明明都有公网 v6、本可以直连的设备，只能老老实实走中继。

所以我需要给服务器加 IPv6——**不是为了让流量走 v6，而是为了让客户端"能照到 v6 的镜子"**，从而打通设备之间的 v6 直连。这个区别很重要，后面第六节的安全组设计整个建立在这一点上。

如果你是别的用途（比如给网站加 v6 访问、或者纯粹想有个 v6 玩玩），下面的步骤同样适用，只是安全组放行的端口按你自己的需求来。

---

## 二、先搞懂：阿里云的 IPv6 是四层结构

这是第一个大坑。阿里云的 IPv6 **不是一个开关**，而是四层嵌套，缺任何一层都不通：
```
① VPC（专有网络）开 IPv6 网段      ← 给整个虚拟网络分配一个 /56
   └─ ② 交换机（vSwitch）开 IPv6 网段  ← 从 /56 里切出一个 /64
       └─ ③ 弹性网卡分配 IPv6 地址      ← 具体机器的那个地址
           └─ ④ IPv6 网关开通公网带宽   ← 让这个地址能上公网
```

**最容易误解的地方**：做完 ①②③，你在机器里 `ip -6 addr` 已经能看到一个 2408 开头的地址了，看起来一切正常——

**但它只是内网地址。** 阿里云控制台会明确显示「网络类型：**私网**」，带宽 0 Mbps。

必须做第 ④ 步开通公网带宽，这个地址才真正能被外网访问。很多教程写到第三步就结束了，导致一堆人卡在"地址有了但 ping 不通"。

> 💡 前三层目前网上已有的教程很多，可以自行搜索操作，我们直接跳到第四层讲。

### 盘点清单

按顺序检查这几项，已完成的直接跳过：

|检查项|在哪看|
|---|---|
|CDT 是否启用、IPv6网关是否升级至 CDT 计费|`cdt.console.aliyun.com`|
|VPC 是否有 IPv6 网段|VPC 控制台 → 专有网络|
|交换机是否有 IPv6 网段|VPC 控制台 → 交换机 → 点进详情|
|网卡是否已分配 IPv6 地址|ECS 控制台 → 弹性网卡 → 详情 → 拉到底看「辅助私网 IP」|
|IPv6 网关是否存在、地址是否开通公网带宽|VPC 控制台 → 公网访问 → IPv6网关|
|安全组有没有 v6 规则|ECS 控制台 → 安全组 → 管理规则 → 入方向|

我这台的情况是：前三层都已就绪，只缺 **CDT 升级** 和 **公网带宽**，安全组也只有孤零零一条 v6 规则。

---

## 三、CDT 免费额度的真相：不是 220GB，是 20GB

**CDT（云数据传输）** 是阿里云把多个产品的公网流量统一计费的服务，并附赠免费额度。

网上到处流传"CDT 白送 220GB/月免费流量"——**这是个误导性说法。**

真实规则是：

|地域类型|免费额度|
|---|---|
|**中国内地地域**|**20 GB / 月**|
|非中国内地地域|200 GB / 月|
|合计|220 GB / 月|

220 GB 是**两个地域池子的总和**。你的 99 小鸡如果在国内（杭州、上海、北京这些），**实际只有 20 GB/月**可用。那 200 GB 是给香港、新加坡这些海外地域的，你用不上。

其他要点：

- 只算**出方向**流量（服务器发出去的），入方向不计费
- 额度**按自然月重置**，月底不结转
- 额度是**账号级共享**的：同账号下所有 ECS、EIP、负载均衡的公网出流量**一起吃**这 20 GB
- 超出部分按**月累计阶梯价**计费，起步档在 0.8 元/GB 附近

20 GB 不算多。所以后面第七节的熔断脚本，不是我小题大做。

---

## 四、最大的坑：计费方式选错，CDT 白配

开通公网带宽时有两个计费方式，**选错了前面的 CDT 升级全白做**：

||按固定带宽计费|按使用流量计费 ✅|
|---|---|---|
|计费逻辑|按天扣费，用不用都扣|按每小时实际出方向流量|
|**CDT 能否抵扣**|❌ **抵扣不了**|✅ **被 CDT 接管**|
|适合谁|稳定高流量业务|低流量、想吃免费额度|

证据就写在 CDT 升级的确认弹窗里：

> "升级成功后，IPv6网关下所有**按流量计费**实例将实时生效。"

看到没，只对**按流量计费**的实例生效。

**而且这两种计费方式不能互相变更**——变配页面明确写着"按固定带宽计费和按使用流量计费不支持互相变更"。一开始就得选对，选错了只能删掉重开。

---

## 五、动手：三步开通

### 第 1 步：CDT 升级「IPv6网关」

打开 `cdt.console.aliyun.com`，往下拉找到「云产品升级至CDT计费」表格，找到 **IPv6网关** 那一行，点「去升级」→ 确定。

状态从「未升级」变成「已升级」就好了。

⚠️ **注意：升级不可回退**。阿里云明说"升级之后不支持回退"。不过这项本身**不产生任何费用**，它只决定流量走不走 CDT 统一出账，留着无害。

**顺序建议：先升级 CDT，再开通带宽。**

### 第 2 步：开通 IPv6 公网带宽

**路径**：VPC 控制台 → 左侧「公网访问 → IPv6网关」→ 点进你的网关 → **IPv6公网带宽** 页签 → 找到你的地址 → 右侧「**开通公网带宽**」

购买页这么填：

|配置项|选什么|
|---|---|
|流量|**按使用流量计费** ← 千万别选"按固定带宽计费"|
|计费方式|**按CDT计费**（选对上面那项后会自动高亮）|
|带宽|**按需选，我选了 1 Mbps**（默认 5，见下方讨论）|
|计费周期|按小时（固定的，改不了）|

点「立即创建」→ 确认订单页核对一遍 → 「立即开通」。

💰 **费用**：后付费，**点下去当场 0 元**，确认订单页的"应付费用"显示 `-`。不超额就一直是 0。

开通后 1~5 分钟生效，列表里会从「私网 0 Mbps」变成「公网 X Mbps / 后付费-按使用流量计费」。

### 带宽该选多大？先破除一个误解

很多人以为"把带宽设小点就不会超额"——**这是错的**。带宽上限只限制**瞬时速率**，不限制总量。算笔账：

|带宽|跑满一个月的流量|相对 20 GB 额度|
|---|---|---|
|100 Mbps|32 TB|1600 倍|
|10 Mbps|3.2 TB|160 倍|
|1 Mbps|324 GB|**16 倍**|

**哪怕只有 1 Mbps，理论上照样能烧穿额度。**

带宽上限的真实作用是**把烧钱速度压到你能反应过来的程度**。1 Mbps 下，配合每 10 分钟跑一次的熔断脚本，两次检查之间最多跑 75 MB——基本不可能在一个周期内失控。

我的场景只需要 v6 做探测（发几个几十字节的小包），所以 1 Mbps 绰绰有余。**如果你要跑网站或者别的服务，按实际需要选，但建议别一上来就拉满。**

事后想改，用同一行的「变配带宽」，**不影响 IPv6 地址，也不产生额外费用**。

### 第 3 步：安全组放行 IPv6

**这是第二个大坑：**

> ⚠️ **安全组的 IPv4 规则和 IPv6 规则完全独立！**
> 
> 你现有的 v4 规则**不会**自动对 v6 生效，必须单独添加 v6 规则。

很多人开通完发现"ping 不通、端口不通"，就是卡在这——云上明明开通了，但安全组把 v6 全挡在外面。

**路径**：ECS 控制台 → 安全组 → 你的安全组 → 管理规则 → 入方向 → 增加规则

基础必备的两条：

|优先级|协议|端口|源|作用|
|100|**所有 ICMP-IPv6**|全部 (-1/-1)|`::/0`|ping6 + **PMTU 发现**|
|1|按你的服务|你的端口|`::/0`|你实际要暴露的服务|

> 🔴 **ICMPv6 一定要放行！** 不放行会导致 **MTU 黑洞**——表现为"能连上但传大文件卡死"，这种问题极难排查。IPv6 的路径 MTU 发现完全依赖 ICMPv6，不像 IPv4 还能靠分片兜底。

**关于"拒绝"规则**：阿里云安全组是**白名单**模型，没有放行规则的端口本来就不通。所以你**不需要**为了关闭某个端口专门加一条"拒绝"规则。

**填表时的两个小坑**：

1. 「访问来源」要先把下拉从 `IPv4` 切成 `IPv6`，再填 `::/0`，然后从自动补全里**点选**
2. 端口填在「访问目的(本实例)」那一栏——很容易手滑填到来源框里，来源框会把端口当成 IP 吞进去

### 第 4 步：验证

```bash
ip -6 addr show eth0 | grep inet6
ping6 -c3 2400:3200:1        # 阿里云公共 IPv6 DNS
```

看到 `scope global` 的 2408 开头地址 + ping6 0% 丢包，就成了。

> ⚠️ **如果你机器上跑着 sing-box / clash 等 TUN 模式代理**：
> 
> `curl -6 ifconfig.co` 会返回**代理服务器的出口 IP**，不是你自己的 v6 地址——因为 TUN 劫持了路由表。
> 
> **这不代表配置失败。** 判断 v6 通不通，用 `ip -6 addr` 和 `ping6`，别用 curl 测公网 IP，会被误导。

> 💡 如果 `ip -6 addr` 里没有地址：Ubuntu 需要在 netplan 开 dhcp6。编辑 `/etc/netplan/50-cloud-init.yaml`，在网卡下加 `dhcp6: true`，然后 `netplan apply`。

---

## 六、进阶：让 IPv6 只做探测，不做中继

这一节是我这个场景的特殊需求，**做网站的读者可以跳过**，但思路值得参考。

回到第一节的动机：**我需要 v6 的唯一目的是让客户端能"照镜子"探测自己的 v6 地址**，而不是让流量走 v6。

一开始我把中继端口（TCP 443）也在 v6 上放行了。后来意识到这是个自相矛盾的配置：

1. 客户端会通过 v6 连中继，**中继流量就走了 v6**，白白消耗宝贵的 20 GB
2. 更糟的是——**1 Mbps 被中继流量占满时，探测包会跟着丢**，反而破坏了 v6 存在的唯一意义

所以我把 v6 的 TCP 443 **删了**，最终只留两条 v6 规则：

|优先级|协议|端口|源|
|---|---|---|---|
|1|自定义 UDP|3478（探测端口）|`::/0`|
|100|所有 ICMP-IPv6|全部|`::/0`|

这样 v6 侧**从物理上只能做探测**，中继和控制平面全部回落到 IPv4。客户端走 v4 连中继完全不受影响。

**代价**：如果有**纯 IPv6 环境、完全没有 v4 出口**的客户端，它们会连不上。我这边没有这种设备，所以能接受。真遇到了把规则加回来就行。

> 这个思路可以推广：**想清楚你到底需要 v6 干什么，只开那个端口。** v6 地址是公开可路由的，不像 NAT 后面的 v4 有天然屏障，开一个端口就是真的暴露在全球互联网上。

---

## 七、防超额：流量熔断脚本

### 为什么需要

**阿里云没有"流量到额自动断网"的开关。** 云监控只能告警，能触发的动作限于发通知、弹性伸缩、函数计算回调——想真的断，得自己实现。

而且云监控这边还有个限制：**「IPv6公网带宽」产品的监控指标全是速率类**（流入/流出带宽、带宽使用率、包速率……），**没有"本月累计流量"这种指标**。所以你没法用云监控对"已经用了多少 GB"设告警，只能对"带宽是不是持续走高"做早期预警。

真要防超额，还得靠本机脚本。

### 为什么不用 vnstat

我最初想用 `vnstat`，但它有个致命问题：

> **vnstat 按接口统计，不区分 IPv4 / IPv6。**

大部分机器 v4 流量远大于 v6，用 vnstat 会把 v4 算进去，导致严重误判、提前误杀。

**正确做法是用 `ip6tables` 的计数器**，挂在 mangle 表的 POSTROUTING 链上，只数公网 v6 出站流量，并排除：

- 自己的 VPC 内网网段（内网互访不计费）
- 链路本地 `fe80::/10`（邻居发现）
- 组播 `ff00::/8`

这样才精确对应 CDT 的计费口径，而且能连 docker 容器的流量一起抓到。

### 工作原理

```
【计量】mangle/POSTROUTING -o eth0 ──→ V6METER 链
                                        ├─ fe80:/10   → RETURN（不计）
                                        ├─ ff00:/8    → RETURN（不计）
                                        ├─ 自己VPC网段  → RETURN（不计）
                                        └─（无 target 的计数规则）← 读它的 bytes

【熔断】filter/OUTPUT + FORWARD ──→ V6BLOCK 链（平时空的，不影响任何流量）
                                     超额时填入：
                                     ├─ ICMPv6      → RETURN（永不封！）
                                     ├─ 内网/本地    → RETURN
                                     └─ DROP        ← 公网 v6 出站全断
```

**特性**：

- cron 每 10 分钟检查一次，累计到状态文件
- 到阈值自动封禁，**月初自动清零并解封**
- **重启后计数器归零会被自动识别**（判断 `当前值 < 上次值` 即为重置）
- **封禁状态在重启后会被重新加固**
- **ICMPv6 永不封**——封了会造成 PMTU 黑洞，而它的流量小到可以忽略

### 脚本

保存为 `/usr/local/sbin/v6guard.sh`，`chmod 700`。

**⚠️ 用之前改两个地方**：`VPC6` 改成你自己的 VPC IPv6 网段（在 VPC 控制台能看到），`LIMIT_GB` 按需调整。
```bash
#!/usr/bin/env bash
# v6guard - IPv6 公网出流量计量 + 超额熔断
set -u

IFACE="eth0"
VPC6="2408:xxxx:xxxx:xxxx::/56"   # ← 改成你自己的 VPC IPv6 网段
LIMIT_GB="18"                      # ← 阈值，20GB 额度建议设 18（见下方说明）
STATE="/var/lib/v6guard/state"
LOG="/var/log/v6guard.log"
MCHAIN="V6METER"
BCHAIN="V6BLOCK"

LIMIT_BYTES=$(( LIMIT_GB * 1024 * 1024 * 1024 ))
mkdir -p "$(dirname "$STATE")"

log() { echo "$(date '+%F %T') $*" >> "$LOG"; }

ensure_meter() {
  ip6tables -t mangle -n -L "$MCHAIN" >/dev/null 2>&1 || {
    ip6tables -t mangle -N "$MCHAIN"
    ip6tables -t mangle -A "$MCHAIN" -d fe80::/10 -j RETURN
    ip6tables -t mangle -A "$MCHAIN" -d ff00::/8  -j RETURN
    ip6tables -t mangle -A "$MCHAIN" -d "$VPC6"   -j RETURN
    ip6tables -t mangle -A "$MCHAIN"                       # 纯计数规则
    log "meter chain created"
  }
  ip6tables -t mangle -C POSTROUTING -o "$IFACE" -j "$MCHAIN" 2>/dev/null || {
    ip6tables -t mangle -A POSTROUTING -o "$IFACE" -j "$MCHAIN"
    log "meter hook installed"
  }
}

read_counter() {
  ip6tables -t mangle -L "$MCHAIN" -vxn 2>/dev/null \
    | awk 'NF==8 && $1 ~ /^[0-9]+$/ {print $2}' | tail -1
}

ensure_bchain() {
  ip6tables -n -L "$BCHAIN" >/dev/null 2>&1 || { ip6tables -N "$BCHAIN"; log "block chain created"; }
  for c in OUTPUT FORWARD; do
    ip6tables -C "$c" -o "$IFACE" -j "$BCHAIN" 2>/dev/null || \
      ip6tables -I "$c" 1 -o "$IFACE" -j "$BCHAIN"
  done
}

block_on() {
  ensure_bchain
  ip6tables -F "$BCHAIN"
  ip6tables -A "$BCHAIN" -p ipv6-icmp -j RETURN     # 保住 NDP / PMTU
  ip6tables -A "$BCHAIN" -d fe80::/10 -j RETURN
  ip6tables -A "$BCHAIN" -d ff00::/8  -j RETURN
  ip6tables -A "$BCHAIN" -d "$VPC6"   -j RETURN
  ip6tables -A "$BCHAIN" -j DROP
}

block_off() { ensure_bchain; ip6tables -F "$BCHAIN"; }
is_blocked() { ip6tables -S "$BCHAIN" 2>/dev/null | grep -q -- '-j DROP'; }
human() { awk -v b="$1" 'BEGIN{printf "%.2f GiB", b/1073741824}'; }

load_state() {
  MONTH=""; TOTAL=0; LAST=0
  [ -f "$STATE" ] && . "$STATE"
  TOTAL=${TOTAL:-0}; LAST=${LAST:-0}
}
save_state() { printf 'MONTH=%s\nTOTAL=%s\nLAST=%s\n' "$MONTH" "$TOTAL" "$LAST" > "$STATE"; }

cmd="${1:-check}"
case "$cmd" in
  check)
    ensure_meter; ensure_bchain; load_state
    NOW_MONTH=$(date +%Y-%m)
    if [ "$MONTH" != "$NOW_MONTH" ]; then
      log "new month $NOW_MONTH (was ${MONTH:-none}) - reset + unblock"
      MONTH="$NOW_MONTH"; TOTAL=0; LAST=0; block_off
    fi
    CUR=$(read_counter)
    [ -z "$CUR" ] && { log "ERROR: cannot read meter counter"; exit 1; }
    if [ "$CUR" -ge "$LAST" ]; then DELTA=$(( CUR - LAST ))
    else DELTA="$CUR"; log "counter reset detected (cur=$CUR < last=$LAST)"; fi
    TOTAL=$(( TOTAL + DELTA )); LAST="$CUR"; save_state
    if [ "$TOTAL" -ge "$LIMIT_BYTES" ]; then
      if is_blocked; then block_on
      else block_on; log "LIMIT REACHED $(human $TOTAL) >= ${LIMIT_GB}GiB - IPv6 egress BLOCKED"; fi
    else
      is_blocked && { block_off; log "under limit again - unblocked"; }
    fi
    ;;
  status)
    ensure_meter; load_state
    CUR=$(read_counter); CUR=${CUR:-0}
    if [ "$CUR" -ge "$LAST" ]; then LIVE=$(( TOTAL + CUR - LAST )); else LIVE=$(( TOTAL + CUR )); fi
    REM=$(( LIMIT_BYTES - LIVE )); [ "$REM" -lt 0 ] && REM=0
    echo "month     : ${MONTH:-unset}"
    echo "used      : $(human $LIVE)   (limit ${LIMIT_GB} GiB)"
    echo "remaining : $(human $REM)"
    if is_blocked; then echo "state     : BLOCKED"; else echo "state     : ok"; fi
    ;;
  unblock) block_off; log "manual unblock"; echo "unblocked" ;;
  block)   block_on;  log "manual block";   echo "blocked" ;;
  reset)
    ensure_meter; load_state
    MONTH=$(date +%Y-%m); TOTAL=0; LAST=$(read_counter); LAST=${LAST:-0}
    save_state; block_off; log "manual reset"; echo "counter reset"
    ;;
  *) echo "usage: $0 {check|status|block|unblock|reset}"; exit 1 ;;
esac
exit 0
```

装 cron：

```bash
( crontab -l 2>/dev/null | grep -v v6guard; \
  echo '*/10 * * * * /usr/local/sbin/v6guard.sh check >/dev/null 2>&1' ) | crontab -
```

日常用法：

```bash
v6guard.sh status     # 看本月用量 / 剩余 / 状态
v6guard.sh unblock    # 手动解封
v6guard.sh block      # 手动封（测试用）
v6guard.sh reset      # 计数清零
tail -f /var/log/v6guard.log
```

输出长这样：

month     : 2026-09
used      : 0.42 GiB   (limit 18 GiB)
remaining : 17.58 GiB
state     : ok
```

> 💡 **阈值为什么设 18 而不是 20**：阿里云的 20 GB 大概率是**十进制**（20 × 10⁹ 字节 = 18.6 GiB）。设 18 GiB 留了点余量，宁可早断不要晚断。

### 写这个脚本踩的三个坑

如果你要改这个脚本，这三个坑能省你不少时间。

**坑 1：`ip6tables: multiple -d flags not allowed`**

一开始想用一条规则搞定排除：

```bash
ip6tables -t mangle -A POSTROUTING -o eth0 \
  ! -d "$VPC6" ! -d fe80:/10 ! -d ff00:/8 ...
```

报错。**ip6tables 不接受多个 `-d`，哪怕是否定形式（`! -d`）也不行。**

解法是改用**自定义链 + 多条 RETURN 规则**做排除。意外收获是代码反而更清晰了——封禁/解封变成了对一条链的 flush/fill。

**坑 2：awk 字段数判断，一直读不到计数**

原本用 `awk '$3=="all" && $0 !~ /RETURN/'` 取字节数，结果日志反复报"读不到计数器"。

原因是 `ip6tables -L -vxn` 的输出有两个反直觉的地方：

1. **prot 列显示的是 `0`，不是 `all`**
2. **无 target 的计数规则那一行少一个字段**（target 列是空的）

实测：

NF=9 |    4    288 RETURN  0  --  *  *  :/0  fe80:/10 |   ← RETURN 行
NF=8 |   48   8120         0  --  *  *  :/0  :/0      |   ← 计数行，target 为空！
```

解法：改用 `awk 'NF==8 && $1 ~ /^[0-9]+$/ {print $2}' | tail -1`

**调试技巧**：用 `awk '{print NR": NF="NF" |"$0"|"}'` 把每行字段数打出来，一眼看清。

**坑 3：脚本退出码始终是 1**

功能都正常，但 `echo $?` 总返回 1，cron 会当成执行失败。

原因是 `is_blocked` 这个函数是 case 分支的**最后一条语句**，未封禁时返回 1，就成了整个脚本的退出码。

解法：脚本末尾加一行 `exit 0`。

### 验证熔断真的有效

别信"应该能用"，实测一下：
```bash
# 1. 手动封禁
v6guard.sh block

# 2. ICMPv6 应该仍然通（验证 PMTU 保护生效）
ping6 -c2 2400:3200:1

# 3. TCP 应该被挡（不用 curl，避免被 TUN 代理干扰）
timeout 4 bash -c 'exec 3<>/dev/tcp/2400:3200:1/53' && echo 通了 || echo 被挡了

# 4. 看 DROP 计数器有没有涨
ip6tables -L V6BLOCK -vxn | tail -1

# 5. 解封，确认恢复
v6guard.sh unblock
```

我这边实测：封禁后 ping6 正常、TCP 失败、DROP 计数器涨了 24 包 3760 字节，解封后 TCP 立即恢复。

---

## 八、配个云监控告警（可选）

脚本是保险丝，告警是耳目，建议都配上。

**路径**：云监控 → 报警规则 → 创建报警规则

|配置项|值|
|---|---|
|产品|**IPv6公网带宽**|
|监控指标|**流出带宽使用率**|
|紧急|连续 5 个周期 ≥ 90%|
|警告|连续 5 个周期 ≥ 50%|

如前所述，这个产品没有累计流量指标，只能拿带宽使用率做早期预警。真实用量看 `v6guard.sh status`。

**两个注意点**：

1. 首次进入会弹窗要求创建服务关联角色 `AliyunServiceRoleForCloudMonitor`，不授权建不了规则
2. ⚠️ **告警发给"云账号报警联系人"，用的是账号绑定的手机/邮箱**。如果你账号没绑邮箱（阿里云会在控制台顶部提示），**告警根本发不出去**。去账号中心补上

---

## 九、常见问题

### Q：IPv6 地址会变吗？删了公网带宽再开回来会不会换地址？

**不会。**

阿里云文档原文是"删除IPv6公网带宽后，该IPv6地址的**公网通信能力中断**"——注意措辞是"通信能力中断"，不是"地址释放"。

因为**地址**和**公网带宽**是两个独立资源：地址是绑在弹性网卡上的基础资源，公网带宽只是挂在它上面的增值服务。删带宽只是把网络类型从「公网」改回「私网」，地址本身纹丝不动。

**不会变的操作**：重启实例、停机再启动、删除公网带宽后重开、变配带宽大小、改安全组

**会变的操作**：

|操作|后果|
|---|---|
|控制台手动删除 IPv6 地址后重新分配|新地址随机生成|
|删除 / 重建弹性网卡|地址随网卡消失|
|释放 ECS 实例|全没了|
|关闭交换机或 VPC 的 IPv6 再重开|网段可能重新分配，地址必变|

一句话：**只要不碰"关闭IPv6"按钮、不删网卡、不释放实例，地址就是稳定的。**

### Q：重启后 `ip -6 addr` 空了，是地址被回收了吗？

**不是。** 阿里云文档提到某些系统（CentOS 8、Ubuntu 18/20）的网卡配置可能被 cloud-init 重置，导致系统里"丢失"IPv6。

**云上的地址还在**，只是操作系统没配上，重新走 DHCPv6 拿回来还是同一个。检查 netplan 配置即可。

### Q：想彻底关掉不玩了，怎么停止计费？

VPC 控制台 → IPv6网关 → 点进网关 → IPv6公网带宽页签 → 「**删除公网带宽**」

删完回到「私网」，计费立即停止，地址保留，以后想恢复重新开通即可。

**CDT 那个升级关不掉**（不支持回退），但它本身不产生费用，留着无害。

### Q：为什么我开通了却 ping 不通？

按这个顺序排查：

1. **安全组有没有放行 ICMPv6？**（最常见原因，v4 规则不管 v6）
2. 控制台看网络类型是不是「公网」，不是「私网」
3. 机器里 `ip -6 addr` 有没有 `scope global` 的地址
4. 你**本地网络**有没有 IPv6？很多家宽和公司网络没有 v6，你 ping 不通是自己这边的问题。可以用 [test-ipv6.com](https://test-ipv6.com/) 测一下，或者找个有 v6 的地方测

### Q：99 计划的机器带宽本来就小，加 IPv6 会不会挤占 IPv4 的带宽？

不会。IPv6 公网带宽是**独立计费、独立限速**的，跟你实例本身的 IPv4 公网带宽是两回事。

---

## 十、总结

整套流程走下来，关键点就这么几条：

1. **IPv6 是四层结构**，前三层配好只是有了内网地址，必须开「公网带宽」才能上公网
2. **CDT 免费额度国内只有 20 GB/月**，不是宣传的 220 GB
3. **计费方式必须选「按使用流量」**，选「按固定带宽」的话 CDT 抵扣不了，前面白配
4. **安全组 v4 和 v6 规则完全独立**，v4 规则对 v6 一点用没有
5. **ICMPv6 一定要放行**，否则 MTU 黑洞等着你
6. **带宽上限不是流量熔断器**，想防超额得自己上脚本
7. **想清楚你要 v6 干什么，只开需要的端口**——v6 地址是全球可路由的，没有 NAT 保护

最后提醒一句：v6 地址暴露在公网上，不像 NAT 后面的 v4 有天然屏障。开端口前想清楚，能不开的别开。