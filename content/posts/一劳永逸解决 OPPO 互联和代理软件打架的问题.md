---
title: 一劳永逸解决 OPPO 互联和代理软件打架的问题
date: 2026-08-30T19:00:00
draft: false
tags:
  - oppo_connect
  - sing-box
  - mihomo
categories:
  - 分享
---
> 两分钟解决，不用装脚本。

---

## 一、症状

开了 sing-box 或者 mihomo 的 TUN 模式，同时装了 **OPPO 互联（O+Connect）**，然后：

- 连接数疯狂飙升，资源监视器里几千上万条莫名其妙的连接
- 风扇狂转，网络变卡
- 重启代理能好一会儿，过阵子又犯

网上流传的偏方是「给 TUN 网卡踢一脚」——在网络连接里把代理网卡的 Internet 连接共享开一下再关掉。有用，但治标不治本，每次开机都得来一遍。

---

## 二、真正的根因

打开设备管理器 → 网络适配器，你会看到一块叫 **`Rust Wintun Tunnel Tunnel`** 的东西，在网络连接里显示为 **`vgate0`**。

这块网卡是 **OPPO 互联自己装的**。

关键在于：**sing-box 的 TUN 网卡和它用的是同一个底层驱动（`wintun.sys`）**。两块适配器挂在同一个驱动上互相干扰，这才是连接数爆炸的源头。

查一下就知道了：

```powershell
Get-PnpDevice -Class Net | Where-Object InstanceId -like 'SWD\WINTUN\*' |
  Select-Object FriendlyName, InstanceId, Status
```
```text
FriendlyName              InstanceId                                        Status
------------              ----------                                        ------
Rust Wintun Tunnel Tunnel SWD\WINTUN\{EA35E09C-F10B-4A63-AFB0-7790F64CEB7F} OK
sing-tun Tunnel           SWD\WINTUN\{A8603457-C603-7694-FC37-1851837DFCEE} OK
```

看到没，两个都在 `SWD\WINTUN\` 下面。`sing-tun Tunnel` 只是 sing-box 给自己适配器起的显示名，底层跟 OPPO 那块是同一个驱动。

**最离谱的是**：我实测把 `vgate0` 禁用之后，OPPO 互联该传文件传文件、该投屏投屏，功能一切正常。也就是说——**它装了一块自己根本用不到的虚拟网卡，纯粹用来恶心人**。

所以解法不是去"踢"、去"刷"、去"重置"，而是**让这块网卡从一开始就别加载**。

---

## 三、解决方法（两步，两分钟）

### 第一步：开机不加载

**注意，光禁用 OPPO 互联的开机自启是没用的**，网卡照样会被装上。真正管用的是改服务。

1. `Win + R` 输入 `services.msc`
2. 找到 **`O+Connect Service`**
3. 双击 → 启动类型改成 **手动** → 停止 → 确定
```powershell
# 命令行版本，管理员 PowerShell
Set-Service -Name 'O+Connect Service' -StartupType Manual
Stop-Service -Name 'O+Connect Service' -Force
```

> 服务名如果对不上，用这个查：
> 
> ```powershell
> Get-Service | Where-Object { $_.DisplayName -like '*Connect*' -or $_.Name -like '*oplus*' } |
>   Select-Object Name, DisplayName, Status, StartType
> ```

改完重启，`vgate0` 就不会出现了。

### 第二步：临时用互联时也别让它装

改成手动之后，你还是可以随时打开 OPPO 互联用。这时候它会弹一个 **UAC 提权请求**——那就是它要装网卡。

**直接点「否」。**

拒绝之后互联照常连接、照常传文件，只是不会再装那块 Wintun 网卡。亲测无影响。

### 收尾：清掉已经装上的那块

如果现在设备管理器里还有 `vgate0`，删掉：
```powershell
# 管理员 PowerShell，把 GUID 换成你自己查到的
pnputil /remove-device "SWD\WINTUN\{EA35E09C-F10B-4A63-AFB0-7790F64CEB7F}"
```

或者设备管理器里右键 → 卸载设备。

**一个意外之喜**：删掉之后，OPPO 在**本次开机内不会再装回来**，只有重启系统才会重来。所以哪怕你忘了改服务，手动删一次也能顶到下次开机。

---

## 五、总结

**问题**：OPPO 互联会装一块它自己根本用不到的 Wintun 虚拟网卡（`vgate0`），和代理软件的 TUN 抢同一个驱动，导致连接数爆炸。

**解法**：

1. `services.msc` 里把 `O+Connect Service` 改成**手动**（光禁自启没用）
2. 用互联时如果弹 UAC，**点否**
3. 已经装上的用 `pnputil /remove-device` 删掉

**最大的教训**：我花了一周时间给症状写自动化脚本，却从来没花两分钟查过「这块碍事的网卡到底是谁装的」。

遇到问题先定位源头，再谈方案。不然你可能会像我一样，写七版脚本去治一个根本不该存在的病。

至于 OPPO 为什么要装一块自己都不用的网卡——只有天知道。