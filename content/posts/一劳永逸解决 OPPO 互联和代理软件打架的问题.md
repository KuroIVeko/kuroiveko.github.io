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

你开了 sing-box 或者 mihomo 的 TUN 模式，同时电脑上装了 **OPPO 互联**。然后你发现电脑上的连接数开始疯狂飙升，资源监视器里一堆莫名其妙的局域网连接，风扇嗡嗡响，网也变卡了。

**手动解法**是这样的：

1. `Win + R` 输入 `ncpa.cpl`，打开网络连接；
2. 右键那块叫 `singbox` 的网卡 → 属性 → **共享**；
3. 勾上「允许其他网络用户通过此计算机的 Internet 连接来连接」→ 确定；
4. 再进去一次，把勾**取消**掉 → 确定。

一开一关，世界安静了。行业黑话叫"踢一脚"。

**但是**——每次开机要踢，重启代理要踢，从公司回家换个 Wi-Fi 还要踢。一天点个七八回，属实有点侮辱人。

说真的，华为的多屏协同、小米的互联互通、甚至微软自家那个"手机连接"，开着 TUN 都相安无事。就 OPPO 互联，非要在局域网里用 mDNS 疯狂广播，一旦发现失败就无限重试，重试还不带退避的，直接把连接数堆爆。

别人家的互联是"你好，请问有设备在吗"，OPPO 这个是"你好？你好？你好？你好？你好？你好？"——TUN 网卡那边不回话，它就问到天荒地老。

好在治它的办法有，就是麻烦。所以有了下面这个脚本。

---

## 二、部署（复制粘贴就完事了）

### 第一步：先看看你的网卡叫啥

**这步别跳**，跳了大概率踢错网卡（别问我怎么知道的，见第四节）。

随便开个 PowerShell，粘贴运行：
```powershell
$s = New-Object -ComObject HNetCfg.HNetShare
$s.EnumEveryConnection | ForEach-Object {
  $p = $s.NetConnectionProps($_)
  [pscustomobject]@{ Name=$p.Name; Device=$p.DeviceName; Status=$p.Status }
} | Format-Table -AutoSize
```

你会看到类似这样的东西：

```
Name         Device                                   Status
----         ------                                   ------
以太网       Realtek PCIe GbE Family Controller            7
WLAN         Intel(R) Wi-Fi 6E AX210 160MHz                2
vgate0       Rust Wintun Tunnel Tunnel                     2
singbox      sing-tun Tunnel                               2
```

找到你代理软件那一行，**记住最左边的 `Name`**。

- sing-box 一般叫 `singbox`
- mihomo / Clash Verge 一般叫 `Mihomo` 或 `Meta`

如果和下面脚本里写的不一样，等会儿改一下就行。

### 第二步：管理员 PowerShell，整段粘贴

**注意是管理员权限**。开始菜单搜 PowerShell，右键"以管理员身份运行"。

然后把下面**一整段**复制进去，回车。
```powershell
#requires -RunAsAdministrator
$Root = "$env:ProgramData\ProxyIcsKick"
$Worker = Join-Path $Root 'kick.ps1'
New-Item -ItemType Directory -Path $Root -Force | Out-Null

@'
param([switch]$Boot)

$ErrorActionPreference = 'Stop'
$Root = "$env:ProgramData\ProxyIcsKick"
$Log  = Join-Path $Root 'kick.log'
$Lock = Join-Path $Root 'last.stamp'

# ==== 要改就改这里 ====
$AdapterNames  = @('singbox','mihomo')   # 你的网卡名，填第一步看到的
$ProcPattern   = 'sing-box|singbox|mihomo|clash|verge|flclash|karing|nekoray'
$CooldownSec   = 15
$SettleSec     = 3
$HoldMs        = 1500
$BootWaitMax   = 300
$BootPollSec   = 10
$BootKickTimes = 2
$BootKickGap   = 20
# =====================

function Log($m) {
  $tag = if ($Boot) { '[BOOT]' } else { '[NET ]' }
  "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')  $tag $m" | Add-Content -Path $Log -Encoding utf8
}
if ((Test-Path $Log) -and (Get-Item $Log).Length -gt 200KB) {
  (Get-Content $Log -Tail 200) | Set-Content $Log -Encoding utf8
}

function Ensure-IcsService {
  $svc = Get-Service SharedAccess -ErrorAction Stop
  if ($svc.StartType -eq 'Disabled') { Set-Service SharedAccess -StartupType Manual }
  if ((Get-Service SharedAccess).Status -ne 'Running') {
    Start-Service SharedAccess; Start-Sleep -Milliseconds 800
  }
}

function Get-TunConnections {
  $share = New-Object -ComObject HNetCfg.HNetShare
  $hits = @()
  foreach ($c in $share.EnumEveryConnection) {
    $p = $share.NetConnectionProps($c)
    if ($AdapterNames -contains $p.Name) {
      $hits += [pscustomobject]@{ Conn = $c; Name = $p.Name }
    }
  }
  ,$hits
}

function Invoke-Kick {
  $share = New-Object -ComObject HNetCfg.HNetShare
  $hits = Get-TunConnections
  if ($hits.Count -eq 0) { return $false }
  foreach ($h in $hits) {
    try {
      $cfg = $share.INetSharingConfigurationForINetConnection($h.Conn)
      if ($cfg.SharingEnabled) { $cfg.DisableSharing(); Start-Sleep -Milliseconds 500 }
      $cfg.EnableSharing(0)
      Start-Sleep -Milliseconds $HoldMs
      $cfg.DisableSharing()
      Log "已踢: $($h.Name)"
    }
    catch { Log "踢 $($h.Name) 失败: $($_.Exception.Message)" }
  }
  return $true
}

try {
  Ensure-IcsService

  if ($Boot) {
    Log "启动模式开始，最长等待 ${BootWaitMax}s"
    $deadline = (Get-Date).AddSeconds($BootWaitMax)
    $ready = $false
    while ((Get-Date) -lt $deadline) {
      $proc = Get-Process -ErrorAction SilentlyContinue | Where-Object { $_.ProcessName -match $ProcPattern }
      if ($proc -and (Get-TunConnections).Count -gt 0) { $ready = $true; break }
      Start-Sleep -Seconds $BootPollSec
    }
    if (-not $ready) { Log "超时：没等到代理和 TUN，撤了"; exit 0 }

    Start-Sleep -Seconds 5
    for ($i = 1; $i -le $BootKickTimes; $i++) {
      Log "第 $i/$BootKickTimes 次"
      [void](Invoke-Kick)
      if ($i -lt $BootKickTimes) { Start-Sleep -Seconds $BootKickGap }
    }
    Set-Content -Path $Lock -Value (Get-Date).Ticks
    Log "启动模式完成"
  }
  else {
    if (Test-Path $Lock) {
      $age = (Get-Date) - (Get-Item $Lock).LastWriteTime
      if ($age.TotalSeconds -lt $CooldownSec) { exit 0 }
    }
    $proc = Get-Process -ErrorAction SilentlyContinue | Where-Object { $_.ProcessName -match $ProcPattern }
    if (-not $proc) { Log "代理没开，不折腾"; exit 0 }

    Start-Sleep -Seconds $SettleSec
    Set-Content -Path $Lock -Value (Get-Date).Ticks
    if (-not (Invoke-Kick)) { Log "没找到匹配的网卡，跳过" }
  }
}
catch {
  Log "错误: $($_.Exception.Message)"
}
'@ | Set-Content -Path $Worker -Encoding utf8

$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest
$psExe = 'powershell.exe'
$baseArg = "-NoProfile -NonInteractive -ExecutionPolicy Bypass -WindowStyle Hidden -File `"$Worker`""

$bootTriggers = @()
$t1 = New-ScheduledTaskTrigger -AtStartup;  $t1.Delay = 'PT60S'; $bootTriggers += $t1
$t2 = New-ScheduledTaskTrigger -AtLogOn;    $t2.Delay = 'PT30S'; $bootTriggers += $t2

$bootAction = New-ScheduledTaskAction -Execute $psExe -Argument "$baseArg -Boot"
$bootSettings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -MultipleInstances IgnoreNew -StartWhenAvailable -RestartCount 2 -RestartInterval (New-TimeSpan -Minutes 2) `
  -ExecutionTimeLimit (New-TimeSpan -Minutes 15)

Register-ScheduledTask -TaskName 'ProxyIcsKick-Boot' -Action $bootAction -Trigger $bootTriggers `
  -Principal $principal -Settings $bootSettings `
  -Description '开机后等 TUN 就绪，连踢两脚' -Force | Out-Null

$cls = Get-CimClass -Namespace root/Microsoft/Windows/TaskScheduler -ClassName MSFT_TaskEventTrigger
$evt = New-CimInstance -CimClass $cls -ClientOnly
$evt.Enabled = $true
$evt.Subscription = '<QueryList><Query Id="0" Path="Microsoft-Windows-NetworkProfile/Operational"><Select Path="Microsoft-Windows-NetworkProfile/Operational">*[System[EventID=10000]]</Select></Query></QueryList>'

$netAction = New-ScheduledTaskAction -Execute $psExe -Argument $baseArg
$netSettings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -MultipleInstances IgnoreNew -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 2)

Register-ScheduledTask -TaskName 'ProxyIcsKick-Net' -Action $netAction -Trigger $evt `
  -Principal $principal -Settings $netSettings `
  -Description '网络一变就踢一脚' -Force | Out-Null

Unregister-ScheduledTask -TaskName 'ProxyIcsKick' -Confirm:$false -ErrorAction SilentlyContinue

Remove-Item (Join-Path $Root 'last.stamp') -Force -ErrorAction SilentlyContinue
Write-Host "`n[OK] 装好了" -ForegroundColor Green
Write-Host "[..] 试跑一次，等 40 秒..." -ForegroundColor Cyan
Start-ScheduledTask -TaskName 'ProxyIcsKick-Boot'; Start-Sleep -Seconds 40
Get-Content "$Root\kick.log" -Tail 12
```

### 第三步：看结果

跑完会自动试踢一次。如果你代理正开着，40 秒内应该能看到：
[BOOT] 已踢: singbox
[BOOT] 已踢: singbox
```

看到这个就成了，**以后再也不用管**。开机自动踢两下，换 Wi-Fi 自动踢一下，插拔网线自动踢一下。

如果第一步看到的网卡名和脚本里的 `@('singbox','mihomo')` 对不上，用记事本打开 `C:\ProgramData\ProxyIcsKick\kick.ps1`，改那一行就行，改完立刻生效，不用重装。

---

## 三、不想要了怎么卸

管理员 PowerShell，三行搞定，删得干干净净：
```powershell
Unregister-ScheduledTask -TaskName 'ProxyIcsKick-Boot' -Confirm:$false
Unregister-ScheduledTask -TaskName 'ProxyIcsKick-Net' -Confirm:$false
Remove-Item "$env:ProgramData\ProxyIcsKick" -Recurse -Force
```

不留注册表垃圾，不留后台进程，不留任何痕迹。

---

## 四、我踩过的坑（你可以直接绕开）

最开始我图省事，用模糊匹配找网卡，关键词里加了 `wintun` 和 `tunnel`。结果日志给我来了一句：
2026-08-25 08:21:35  已踢: vgate0
```

**踢错人了。**

因为 `vgate0` 的驱动名叫 `Rust Wintun Tunnel Tunnel`，两个关键词全中，而且它排在 `singbox` 前面，脚本一命中就停，真正该踢的反而躲过一劫。

Wintun 是个通用的隧道驱动，WireGuard、各种网关客户端都用它，靠驱动名认人纯属自找麻烦。所以现在的版本改成**只认网卡名，而且精确匹配**，多装几个 VPN 也不会误伤。

顺带一提，误踢一次没什么后果，脚本是开了 ICS 立刻关掉，状态会还原。

---

## 五、几个常见疑问

**Q：重启还在吗？系统更新会不会没？**

在的。计划任务和脚本都存在系统级目录，重启、睡眠唤醒、Windows 打补丁、甚至升级大版本都保留。只有重装系统或者"重置此电脑"才需要重新装一遍。

**Q：会不会一直有个进程挂后台占内存？**

不会。平时是零进程，只有被触发的那十几秒才跑一下 PowerShell，跑完自动退出。

**Q：会弹 UAC 吗？**

不会。任务以 SYSTEM 身份运行，全程静默，你甚至察觉不到它在干活。

**Q：我在网络连接里看到 `192.168.137.1`，是脚本搞的吗？**

先跑一下这个：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object IPAddress -like '192.168.137.*' |
  Select-Object InterfaceAlias, IPAddress
```

如果显示的是 `本地连接* 2` 这种名字，**那不是脚本干的**，是 Windows「移动热点」用的虚拟网卡，天生自带这个 IP，跟你装不装这脚本没关系。无视它。

只有出现在 `以太网`、`WLAN` 这种真网卡上才需要管，但本脚本不会导致这种情况。

**Q：日志在哪？**

`C:\ProgramData\ProxyIcsKick\kick.log`。想看的话：
```powershell
Get-Content "$env:ProgramData\ProxyIcsKick\kick.log" -Tail 20
```

`[BOOT]` 是开机踢的，`[NET ]` 是网络变化踢的。日志超过 200KB 自动裁剪，不会撑爆硬盘。

**Q：日志说「没找到匹配的网卡」怎么办？**

回第一步，重新看你的网卡叫什么，然后改 `kick.ps1` 里的 `$AdapterNames`。

**Q：日志说「代理没开，不折腾」？**

那说明代理确实没开，正常现象。如果代理明明开着还这么说，说明你的进程名不在识别列表里，`Get-Process | Select ProcessName` 查一下真实进程名，补进 `$ProcPattern` 就行。

**Q：踢了还是有问题？**

打开 `kick.ps1`，把 `$BootKickTimes = 2` 改成 `3`，或者把 `$HoldMs = 1500` 改成 `3000`（ICS 开着的时间长一点）。改完立即生效。

---

## 六、总结

一句话：**让 Windows 自己在合适的时机替你去点那两下鼠标。**

开机踢两脚，网络一变再踢一脚，剩下的时间它就安静躺着不占任何资源。装一次，忘掉它。

至于 OPPO 什么时候能修好这个 bug——我建议你还是先用脚本吧。