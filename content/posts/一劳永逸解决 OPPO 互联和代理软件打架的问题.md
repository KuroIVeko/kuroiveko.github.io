---
title: 一劳永逸解决 OPPO 互联和代理软件打架的问题
date: 2026-08-25T19:00:00
draft: false
tags:
  - oppo_connect
  - sing-box
  - mihomo
categories:
  - 分享
---


## 一、先说说这个破事

你开了 sing-box 或者 mihomo 的 TUN 模式，同时电脑上装了 **OPPO 互联**。然后连接数开始疯狂飙升，资源监视器里一堆莫名其妙的局域网连接，风扇嗡嗡响，网也变卡了。

**手动解法**是这样的：

1. `Win + R` 输入 `ncpa.cpl`，打开网络连接；
2. 右键那块叫 `singbox` 的网卡 → 属性 → **共享**；
3. 勾上「允许其他网络用户通过此计算机的 Internet 连接来连接」，在「家庭网络连接」下拉框里随便选一个，比如 **WLAN** → 确定；
4. 再进去一次，把勾**取消**掉 → 确定。

一开一关，世界安静了。行业黑话叫"踢一脚"。

**但是**——每次开机都得点一遍。一天下来点个七八回，属实有点侮辱人。

说真的，华为的多屏协同、小米的互联互通、甚至微软自家那个"手机连接"，开着 TUN 都相安无事。就 OPPO 互联，非要在局域网里用 mDNS 疯狂广播，一旦发现失败就无限重试，重试还不带退避的，直接把连接数堆爆。

别人家的互联是"你好，请问有设备在吗"，OPPO 这个是"你好？你好？你好？你好？你好？"——TUN 网卡那边不回话，它就问到天荒地老。

**最讽刺的是**，作妖的明明是 OPPO 互联，最后挨踢的却是 singbox。人家 sing-box 老老实实转发流量，什么都没干错，就因为长了一块 TUN 网卡，天天被摁在地上开关 ICS。属于是隔壁邻居半夜蹦迪，物业上门把你家电闸拉了。

没办法，OPPO 那边动不了，能动的只有自己这边。

---

## 二、部署（复制粘贴就完事了）

### 第一步：先看看你的网卡叫啥

**这步别跳**，跳了大概率踢错网卡。

随便开个 PowerShell，粘贴运行：
```powershell
$s = New-Object -ComObject HNetCfg.HNetShare
$s.EnumEveryConnection | ForEach-Object {
  $p = $s.NetConnectionProps($_)
  [pscustomobject]@{ Name=$p.Name; Device=$p.DeviceName; Status=$p.Status }
} | Format-Table -AutoSize
```

你会看到类似这样的东西：

```text
Name         Device                                   Status
----         ------                                   ------
以太网       Realtek PCIe GbE Family Controller            7
WLAN         Intel(R) Wi-Fi 6E AX210 160MHz                2
vgate0       Rust Wintun Tunnel Tunnel                     2
singbox      sing-tun Tunnel                               2
```

你要记两个名字：

- **公用侧**：你代理软件那块 TUN 网卡。sing-box 一般叫 `singbox`，mihomo / Clash Verge 一般叫 `Mihomo` 或 `Meta`。
- **专用侧**：也就是 GUI 里「家庭网络连接」下拉框要选的那个，下面以 `WLAN`为例。

如果和下面脚本里写的不一样，等会儿改一下。

### 第二步：管理员 PowerShell，整段粘贴

**注意是管理员权限**。开始菜单搜 PowerShell，右键"以管理员身份运行"。

然后把下面**一整段**复制进去，回车。
```powershell
#requires -RunAsAdministrator

# 先清掉可能残留的进程
Stop-ScheduledTask -TaskName 'IcsToggle' -ErrorAction SilentlyContinue
Get-CimInstance Win32_Process -Filter "Name='powershell.exe'" |
  Where-Object { $_.CommandLine -like '*IcsToggle*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
Start-Sleep -Seconds 2

$Root = "$env:ProgramData\IcsToggle"
$Worker = Join-Path $Root 'toggle.ps1'
New-Item -ItemType Directory -Path $Root -Force | Out-Null

@'
$ErrorActionPreference = 'Stop'
$Root = "$env:ProgramData\IcsToggle"
$Log  = Join-Path $Root 'toggle.log'

# ==== 要改就改这里 ====
$PublicNames  = @('singbox','mihomo')   # 公用侧：TUN 网卡名
$PrivateName  = 'WLAN'                  # 专用侧：家庭网络连接
$ProcPattern  = 'sing-box|singbox|mihomo|clash|verge|flclash|karing|nekoray'
$WaitMax      = 300    # 最长等待代理+TUN 就绪的秒数
$PollSec      = 3      # 轮询间隔
$SettleSec    = 1      # 就绪后再晾多久才动手（电脑卡就调大）
$HoldSec      = 1      # ICS 打开后保持多久再关（电脑卡就调大）
$StepGapSec   = 1      # 公用侧和专用侧之间的间隔（电脑卡就调大）
# =====================

function Log($m) {
  try {
    "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')  $m" | Add-Content -Path $Log -Encoding utf8 -ErrorAction Stop
  } catch { }
}

try {
  if ((Test-Path $Log) -and (Get-Item $Log).Length -gt 200KB) {
    (Get-Content $Log -Tail 200) | Set-Content $Log -Encoding utf8
  }
} catch { }

function Get-Conns([string[]]$names) {
  $share = New-Object -ComObject HNetCfg.HNetShare
  $hits = @()
  foreach ($c in $share.EnumEveryConnection) {
    $p = $share.NetConnectionProps($c)
    if ($names -contains $p.Name) {
      $hits += [pscustomobject]@{ Conn = $c; Name = $p.Name }
    }
  }
  ,$hits
}

function Clear-AllSharing {
  $share = New-Object -ComObject HNetCfg.HNetShare
  $did = $false
  foreach ($c in $share.EnumEveryConnection) {
    try {
      $cfg = $share.INetSharingConfigurationForINetConnection($c)
      if ($cfg.SharingEnabled) {
        $n = $share.NetConnectionProps($c).Name
        $cfg.DisableSharing()
        Log "已关闭共享: $n"
        $did = $true
      }
    } catch { }
  }
  return $did
}

Log "===== 流程启动 (轮询 ${PollSec}s / 晾 ${SettleSec}s / 保持 ${HoldSec}s) ====="

try {
  # 确保 ICS 服务可用
  $svc = Get-Service SharedAccess -ErrorAction Stop
  if ($svc.StartType -eq 'Disabled') { Set-Service SharedAccess -StartupType Manual }
  if ((Get-Service SharedAccess).Status -ne 'Running') {
    Start-Service SharedAccess; Start-Sleep -Milliseconds 800
  }

  # 等代理进程 + TUN 网卡都就绪
  Log "等待代理和 TUN 就绪(最长 ${WaitMax}s，每 ${PollSec}s 查一次)"
  $deadline = (Get-Date).AddSeconds($WaitMax)
  $ready = $false
  while ((Get-Date) -lt $deadline) {
    $proc = Get-Process -ErrorAction SilentlyContinue | Where-Object { $_.ProcessName -match $ProcPattern }
    if ($proc -and (Get-Conns $PublicNames).Count -gt 0) { $ready = $true; break }
    Start-Sleep -Seconds $PollSec
  }
  if (-not $ready) { Log "超时：没等到代理和 TUN，撤了"; exit 0 }
  Log "已就绪"

  # 检查专用侧在不在
  $priv = Get-Conns $PrivateName
  if ($priv.Count -eq 0) {
    Log "警告：找不到专用侧连接 '$PrivateName'，本次跳过"
    exit 0
  }

  Log "晾 ${SettleSec}s 再动手..."
  Start-Sleep -Seconds $SettleSec

  # 动手前清干净
  if (Clear-AllSharing) { Start-Sleep -Seconds 3 }

  $share = New-Object -ComObject HNetCfg.HNetShare
  $pub = (Get-Conns $PublicNames)[0]

  # 顺序很关键：先公用侧(0)，再专用侧(1)
  $cfgPub = $share.INetSharingConfigurationForINetConnection($pub.Conn)
  $cfgPub.EnableSharing(0)
  Log "已设公用侧: $($pub.Name)"
  Start-Sleep -Seconds $StepGapSec

  $cfgPriv = $share.INetSharingConfigurationForINetConnection($priv[0].Conn)
  $cfgPriv.EnableSharing(1)
  Log "已设专用侧: $($priv[0].Name)"

  Log "保持 ${HoldSec}s..."
  Start-Sleep -Seconds $HoldSec

  # 关闭：先专用后公用
  $cfgPriv.DisableSharing()
  Log "已关闭专用侧: $($priv[0].Name)"
  Start-Sleep -Seconds 1
  $cfgPub.DisableSharing()
  Log "已关闭公用侧: $($pub.Name)"

  Log "===== 完成 ====="
}
catch {
  Log "错误: $($_.Exception.Message)"
  try { [void](Clear-AllSharing); Log "已兜底清理" } catch { Log "兜底失败: $($_.Exception.Message)" }
}
'@ | Set-Content -Path $Worker -Encoding utf8

$psExe = "$env:SystemRoot\System32\WindowsPowerShell\v1.0\powershell.exe"

# SYSTEM 身份，不需要登录，远程桌面场景也能用
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest

# 开机立即触发，无延时
$trigger = New-ScheduledTaskTrigger -AtStartup

$action = New-ScheduledTaskAction -Execute $psExe `
  -Argument "-NoProfile -NonInteractive -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$Worker`""

$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -MultipleInstances IgnoreNew -StartWhenAvailable `
  -RestartCount 2 -RestartInterval (New-TimeSpan -Minutes 2) `
  -ExecutionTimeLimit (New-TimeSpan -Minutes 10)

Register-ScheduledTask -TaskName 'IcsToggle' -Action $action -Trigger $trigger `
  -Principal $principal -Settings $settings `
  -Description '开机后等代理就绪，配置 ICS 共享保持一下再关闭' -Force | Out-Null

Write-Host "`n[OK] 装好了（公用侧 singbox → 专用侧 WLAN）" -ForegroundColor Green
Write-Host "[i ] 流程：开机立即启动 → 每 3s 查一次代理 → 就绪后开 ICS → 保持 1s → 关" -ForegroundColor Yellow
Write-Host "[..] 试跑一次，约 20 秒..." -ForegroundColor Cyan
Start-ScheduledTask -TaskName 'IcsToggle'; Start-Sleep -Seconds 25
Get-Content "$Root\toggle.log" -Tail 20
```

### 第三步：看结果

跑完会自动试一次。如果代理正开着，日志大概长这样：
```text
===== 流程启动 (轮询 3s / 晾 1s / 保持 1s) =====
等待代理和 TUN 就绪(最长 300s，每 3s 查一次)
已就绪
晾 1s 再动手...
已设公用侧: singbox
已设专用侧: WLAN
保持 1s...
已关闭专用侧: WLAN
已关闭公用侧: singbox
===== 完成 =====
```

看到 `===== 完成 =====` 就成了，**以后再也不用管**。

如果网卡名和脚本里写的对不上，用记事本打开 `C:\ProgramData\IcsToggle\toggle.ps1`，改 `$PublicNames` 和 `$PrivateName` 两行，下次开机生效。

### 关于那几个等待时间

我这台机器实测**全设成 1 秒就够了**，而且开大反而不好——ICS 开着的那段时间连接数还在涨，等得越久涨得越多，纯属自己给自己添堵。

但保留了可调，因为有些电脑启动慢、ICS 服务响应慢，1 秒可能不够。如果你的日志里出现 `对象已存在` 之类的报错，把 `$StepGapSec` 和 `$SettleSec` 往上调（比如 3 或 5）再试。

---

## 三、不想要了怎么卸

管理员 PowerShell，两行搞定：
```powershell
Unregister-ScheduledTask -TaskName 'IcsToggle' -Confirm:$false
Remove-Item "$env:ProgramData\IcsToggle" -Recurse -Force
```

不留注册表垃圾，不留后台进程，不留任何痕迹。

---

## 四、六次翻车实录（这节才是精华）

最终方案看着简单，其实是撞了六回墙才撞出来的。写下来给后来人省点事。

### 翻车一：模糊匹配踢错了网卡

一开始图省事，用正则匹配网卡，关键词里塞了 `wintun` 和 `tunnel`，还同时匹配网卡名和驱动名。结果日志给我来了一句：
```text
2026-08-25 08:21:35  已踢: vgate0
```

**踢错人了。**

因为 `vgate0` 的驱动名是 `Rust Wintun Tunnel Tunnel`，两个关键词全中，而且它在枚举顺序里排在 `singbox` 前面，脚本一命中就 `break`，真正该踢的反而躲过一劫。

**教训**：Wintun 是通用隧道驱动，WireGuard、各种网关客户端都在用，**靠驱动名认人纯属自找麻烦**。改成只认网卡名、精确比对（`-contains` 而不是 `-match`），再装十个 VPN 也不会误伤。

顺带一提，误踢一次没后果。就是有点对不起 vgate0，它比 singbox 还无辜。

### 翻车二：踢两下不够，会死灰复燃

第二版改成开机踢两下，间隔 20 秒。当场测试完美，用了没几天发现——**过一会儿它又犯了**。

于是加码到每 10 秒踢一次、连踢 7 次、覆盖整整一分钟，想用密度压死它。结果还是会复燃。

**教训**：这条路方向就错了。"快速开关"本质上只是让接口闪一下，刷新得不够彻底，次数堆得再多也是同一个无效动作重复七遍。

### 翻车三：热点方案很好用，但它需要登录

后来发现一个更管用的偏方：**开一下 Windows 移动热点再关掉**，比单踢一块网卡彻底得多。

写成脚本后确实一次见效。但这条路上又连摔两跤：

- **`System.WindowsRuntimeSystemExtensions` 找不到** —— 热点走 WinRT 的 `NetworkOperatorTetheringManager`，在 PowerShell 5.1 里必须先 `Add-Type -AssemblyName System.Runtime.WindowsRuntime`，否则那个类型压根不加载。我漏了这行。
- **绕道绕进坑里** —— 漏了上面那行之后，我误判成"PowerShell 7 不支持 WinRT"，于是手搓了个轮询 `IAsyncOperation.Status` 的替代方案。结果 PS 5.1 根本不投影这个属性，取出来是 `$null`，先报"操作超时"，加了 `.ToString()` 之后升级成"不能对 Null 值调用方法"。**绕路之前先确认自己绕的方向对不对。**

修好之后热点方案跑通了，但致命问题来了：**WinRT 热点 API 必须在用户会话下运行，SYSTEM 身份调用必然失败**，任务只能用 `AtLogOn` 触发。

我平时开机是不登录的，直接远程桌面连进去——**结果就是 RDP 还没连上，机器就已经炸了**，脚本压根没机会跑。

**教训**：写开机自动化之前，先想清楚"开机"和"登录"是两码事。需要用户会话的 API，在无人值守场景下等于不存在。

### 翻车四：不是踢一脚，是要"切状态"

绕回 ICS 方案后（它 SYSTEM 身份能跑，不需要登录），换了个思路：

**不再是"开一下立刻关"，而是真的打开、保持一小会儿、再关掉。**

**教训**：从"快速开关"到"状态切换"，是真正有效的那一下。之前一直在优化次数和频率，其实方向不对——**问题不在于踢得够不够多，而在于有没有真的完成一次状态切换**。

顺带踩了个小坑：改完参数立刻手动重测，日志里出现了两轮不同的时长。原因是上一次的 PowerShell 进程还在自己的 sleep 里没退出，**脚本参数是进程启动时读进内存的，改文件不影响正在跑的进程**。改参数前先 `Stop-ScheduledTask`。

### 翻车五：以为下拉框可以空着

之前一直只设公用侧，没管「家庭网络连接」那个下拉框。加上专用侧之后，日志开始刷 `对象已存在 (0x80071392)`，重试三次全挂。

我先怀疑是移动热点占了专用侧的位置，查了一圈——所有连接的共享状态全是 `False`，热点两块虚拟网卡都是 Disconnected，注册表里连 `SharedAccess\Parameters` 键都不存在。**干干净净，没有任何残留。**

于是我又得出一个错误结论："专用侧不用设，Windows 会自己配，那个下拉框本来就该是空的。"

结果被当场纠正：**不选是真的不生效**。

### 翻车六：顺序反了（真正的元凶）

既然手动能成，那就是调用序列的问题。写了个前台对照实验，一步步打印状态：
```text
--- 试 EnableSharing(0) on singbox ---
公用侧 OK
singbox Enabled=True  Type=0
WLAN    Enabled=False Type=0

--- 试 EnableSharing(1) on WLAN ---
专用侧 OK
最终 singbox Enabled=True Type=0
最终 WLAN    Enabled=True Type=1
```

**先公用后专用，两步全 OK。**

而我脚本里写的是先专用后公用，还煞有介事地加了注释解释"反过来 Windows 可能自己挑一个专用侧，就不受控了"——纯属凭直觉编的，一做实验就打脸。

原因其实不难理解：公用侧先启用，NAT 实例才被创建出来，专用侧才有东西可挂。反过来单独声明一个没有配对公用侧的专用侧，系统无从建立，就报"对象已存在"。这也解释了为什么最开始截图里下拉框是空的——**它要等公用侧启用之后才有内容**。

**教训**：COM 这种没文档、报错信息还含糊的接口，**先手动跑一遍验证调用序列，别照着直觉写**。我在这个坑上浪费的时间，比写整个脚本还多。而且中间还因为查询结果"太干净"反推出了一个更离谱的结论，差点把方向带偏。

---

## 五、几个常见疑问

**Q：需要登录吗？我开机就挂着远程连的。**

不需要。这版是 SYSTEM 身份 + `AtStartup` 触发，开机就跑，没人登录也照跑。这正是我最后放弃热点方案的原因。

**Q：重启还在吗？系统更新会不会没？**

在的。计划任务和脚本都存在系统级目录，重启、Windows 打补丁、甚至升级大版本都保留。只有重装系统或者"重置此电脑"才需要重新装。

注意 `AtStartup` 只认真正的冷启动和重启，**睡眠唤醒、休眠恢复不算**。如果你平时是合盖睡眠而不是关机，任务本来就不会跑，日志不更新是正常的。

**Q：会不会一直有个进程挂后台占内存？**

不会。平时零进程，开机跑那十几秒才有个 PowerShell，跑完自动退出。

**Q：会弹 UAC 吗？**

不会。SYSTEM 身份静默运行，你察觉不到它在干活。

**Q：ICS 开着那一秒会不会影响上网？**

WLAN 在那一秒会被临时占用（分到 `192.168.137.1`），关闭后自动恢复。如果你是有线上网，完全无感；如果你正用 Wi-Fi，可能会闪断一下。

**Q：切 Wi-Fi、插拔网线要不要也触发？**

实测基本不会犯，所以最终版只保留了开机触发，够用就行。硬要加也行，用事件日志 `Microsoft-Windows-NetworkProfile/Operational` 的 10000 事件做触发器，但我建议先别加，能少一个定时炸弹是一个。

**Q：我在网络连接里看到 `192.168.137.1`，是脚本搞的吗？**

先跑一下：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object IPAddress -like '192.168.137.*' |
  Select-Object InterfaceAlias, IPAddress
```

如果显示 `本地连接* 2` 这种名字，**那不是脚本干的**，是 Windows 移动热点用的虚拟网卡，天生自带这个 IP。无视它。

**Q：日志在哪？**

`C:\ProgramData\IcsToggle\toggle.log`：
```powershell
Get-Content "$env:ProgramData\IcsToggle\toggle.log" -Tail 20
```

日志超过 200KB 自动裁剪，不会撑爆硬盘。

**Q：日志说「没找到匹配的网卡」或「找不到专用侧连接」？**

回第一步重新看网卡名，改 `toggle.ps1` 里的 `$PublicNames` / `$PrivateName`。

**Q：日志说「超时：没等到代理和 TUN」？**

代理没在 5 分钟内起来，或者进程名不在识别列表里。`Get-Process | Select ProcessName` 查一下真实进程名，补进 `$ProcPattern`。

**Q：日志报「对象已存在 (0x80071392)」？**

先手动清一遍残留：

```powershell
$s = New-Object -ComObject HNetCfg.HNetShare
$s.EnumEveryConnection | ForEach-Object {
  $c = $s.INetSharingConfigurationForINetConnection($_)
  if ($c.SharingEnabled) { $c.DisableSharing() }
}
"已清理"
```

还报的话，把 `$StepGapSec` 和 `$SettleSec` 调大到 3 或 5，给 ICS 服务多点反应时间。

**Q：想手动测一下？**

```powershell
Stop-ScheduledTask -TaskName IcsToggle
Start-ScheduledTask -TaskName IcsToggle
Start-Sleep 25
Get-Content "$env:ProgramData\IcsToggle\toggle.log" -Tail 15
```

先 Stop 是为了避免上一次的进程还没退出导致日志串味（见翻车四）。

---

## 六、总结

一句话：**开机后等代理起来，把 ICS 的公用侧设成 TUN 网卡、专用侧设成 WLAN，保持一秒，再关掉。**

关键有两点：一是要完整地切一次状态而不是快速闪一下；二是**先公用后专用**，顺序反了直接报错。

我绕了六圈才把这两件事弄明白，其中一半时间浪费在凭直觉写 COM 调用顺序上。

至于为什么最后是 singbox 挨揍——毕竟能改的只有自己家的配置，改不了别人家的代码。至于 OPPO 什么时候能修好这个 bug，我建议你还是先用脚本吧。