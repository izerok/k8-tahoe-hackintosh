# 音频与空载功耗

> 原 README §7.5。

### 7.5 音频与空载功耗（2026-11-20）

- **音频**：`VoodooHDA_BL.kext` 3.7.1（Wujie 14 Pro 笔记本线）经 OpenCore `Kernel → Add` 注入，
  `IOPCIPrimaryMatch 0x15e31022` 命中 K8 的 `AZAL`；输出/麦克风正常，codec 名误报为 Conexant
- **GFXOFF**：`debug.phx_pm_gfxoff=1`（fork PhxBringup build 37 的 PM）让 PMFW 空闲时给 GPU 断电；
  空闲 socket 47 W → 27 W，`gfxoff_state=1`，有活动时自动唤醒
- **时钟门控**：`gpucg.sh all`（GC 11.0.1 的 MGCG/CGCG/CGLS，15XA 笔记本线）再降到 **23.4 W**，
  温度 54.5 → **50.1 °C**；GPU 仍 800 MHz、Metal 3、WindowServer 正常
- 持久化：`debug.phx_pm_gfxoff=1` 在 `k8tuned.sh`；`gpucg.sh all` 在 `phxdisplay.sh` 开机重试执行
- 蓝牙现已可用（§7.14），但**不要禁用 `bluetoothd`**：试过 `launchctl disable system/com.apple.bluetoothd` 后，
  `ContextStoreAgent`/`logd`/`launchd` 会以 56%/36%/136% CPU 自旋重试，空闲功率反而升到 40 W；
  已 `launchctl enable` + `bootstrap system /System/Library/LaunchDaemons/com.apple.bluetoothd.plist` 恢复。
  它每 8–9 分钟 EXC_GUARD 重启一次的代价远小于这个自旋。
