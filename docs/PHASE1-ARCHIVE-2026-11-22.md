# K8 macOS 阶段一存档（2026-11-22）

> **状态：阶段性结束，项目存档。** 后续如有新的突破（上游修复、硬件更换、系统升级等），再重新启用。
> 本文件是恢复现场的唯一入口：先读这里，再看 `FINAL-STATUS-2026-11-19.md` 的时间线。

## 1. 最终状态（最后一次实测：2026-11-22 12:04，25G83）

| 子系统 | 状态 | 关键证据 / 备注 |
|---|---|---|
| OpenCore | ✅ 1.0.8 **RELEASE** | `nvram …:opencore-version` = `REL-108-2026-09-27`；`Kernel→Add` 22 项（19 启用） |
| 启动 | ✅ 无需 U 盘，Timeout=5 | 内置盘 `macx`；启动到 SSH 约 15–20 s |
| GPU / 显示 | ✅ Radeon 780M，Metal 3 | PhxBringup 1.0.0 + PhxAccel 0.1（AuxKC）；`PhxFB: console: 2560x1600` |
| 帧同步 | ✅ vbl-timing | 60 fps 稳定，迟帧均值 ~85 µs、最大 ~1.5 ms |
| 残影 | ✅ dbuf（`debug.phxfb_dbuf=1`） | 与 `NVMTL_WS_WAIT_COMPLETION=1` 组合，用户确认不复现 |
| 功耗 / CPU | ✅ GFXOFF + CPPC | 空闲 socket **4.8–7 W**，`gfxoff=1`、`cppc_supported=1`、`gfx_max=2600` |
| xHCI | ✅ INTx（`WujieXhciIntx`） | 取代 XhciNudge 轮询；`kernel_task` ~0.9% |
| 音频 | ✅ VoodooHDA_BL 3.7.1 | 输出 + 模拟麦克风；codec 误报 Conexant |
| 蓝牙 | ✅ AX200 `8087:0029` | USBToolBox + UTBMap（XHC0 PRT5 internal）+ IntelBTPatcher 2.5.3（Tahoe）；`State: On`、真实 MAC、`THIRD_PARTY_DONGLE` |
| Wi-Fi | ✅ WPA2 可连 | `AirportItlwm` 2.4.0（laobamac fork）；WPA3-SAE 不支持 |
| 有线 | ⚠️ 能用但有 panic 风险 | RTL812xLucy 1.1.3；见 §3.1 |
| 容器 | ✅ Kiri + RyzenHV | `kern.hv_support=1`；compose/build 已验证；K8s 未测 |
| 日志脱敏 | ✅ | 公开仓序列号/MLB/UUID/ROM/ MAC/内网 IP 全部占位 |

## 2. 关键文件位置

### Mac 侧（本机）
```
（私有仓）/            私有仓（完整档案、证据、补丁、脚本）
  docs/FINAL-STATUS-2026-11-19.md         逐日时间线（最重要）
  docs/PHASE1-ARCHIVE-2026-11-22.md       本文件
  patches/utbmap/                         K8 UTBMap + 说明（蓝牙端口映射）
  patches/nvrm610/                        nvrm610.conf 三键 + 验证方法
  patches/k8-accel-vbl-timing-dbuf.patch  vbl-timing + dbuf 源码补丁
  patches/xhci-intx/ phxcppc/ voodoo/ hdmi-tools/
  evidence/                               崩溃/panic 原件
（公开仓）/                 公开仓（脱敏版）
（K8 Tune 仓）/                 K8 Tune（独立公开仓）
```

### K8 侧
```
ESP: /Volumes/NO NAME/EFI/OC/
  config.plist.bak-before-*               每个改动的备份（最新：P6/UTBMap/BTLEAdv）
  OpenCore.efi.debug-bak-*                旧 DEBUG 二进制（可回退）
/Users/Shared/gpu/backup/
  esp-OC-before-release-clean-20261121-143810.tar.gz   39 MB 整包
  aux.kc.*                                GPU AuxKC 备份
/Library/GPUBundles/nvmtl/nvrm610.conf    三键（见 patches/nvrm610/）
/Users/Shared/gpu/phxboot-off             存在=下次启动不进 GPU（安全阀）
/Users/Shared/gpu/hdmi/off                HDMI 实验关闭标记（保留）
```

### SMB 备份
```
SMB 备份（私有）\K8-phase1-2026-11-22\  本次存档
SMB 备份（私有）\K8-macOS-2026-11-19-installers\  安装盘/EFI 备份
```

## 3. 已知限制（不要再重复踩）

### 3.1 RTL8125 `fq_codel` panic（内核级，未解）
三次同栈 panic（`fq_codel_dq → RTL8125::outputStart`），缓解（TSO/ASPM 关、`pacing=0`、`l4s=0`）**证明无效**。
`debug=0x100` 已移除（panic 自动重启）。**规避：大流量走 Wi-Fi；有线只作 SSH/应急；等 Mieze 1.1.4。**

### 3.2 WindowServer 崩溃（`radv_CreateGraphicsPipelines`，上游未修）
两次同栈（11-20 14:48、11-21 17:45）。CA 并发调用**同步** pipeline 接口 → NVMTL → radv。
`NVMTL_PIPELINE_THREADS=1` 只覆盖异步队列，**无效**（已写入 conf 但不要当修复）。
**复发时：开「降低透明度」，或向上游报 issue（同步路径串行化）。**

### 3.3 AirDrop 不可用
`AirportItlwm` 无 AWDL（无 `awdl0`，`AWDL REALTIME is inactive`）。社区方案 ForbiddenItlwm 只支持 26.7.x。
**替代：LocalSend / PairDrop。**

### 3.4 USB-C 无视频、无 USB
K8 的 USB-C 走 USB4 host router（`1022:1668/1669`），macOS 无驱动（`Thunderbolt/USB4: No hardware found`，
Type-C `ConnectCount=0`）。**唯一路线：USB DisplayLink 适配器**（驱动 17.1 支持 Intel + Tahoe 26）。

### 3.5 设置页/墙纸面板 CPU 高（上游架构代价）
60 fps 不丢帧，但 WindowServer CPU 空闲 ~33%、墙纸面板 ~63%（CA 图层准备 + Metal 绘制）。
**不要试图用提频解决**（GPU busy 0%，瓶颈在 CPU）。

### 3.6 其他
- 蓝牙是 `THIRD_PARTY_DONGLE`：Handoff/AirDrop/接力不可用；iPhone 手动配对系统不支持
- BTLEAdvFixup 0.9.3 在 25G83 上**早期 panic**（与 IntelBTPatcher 双 hook + 版本不匹配），保持禁用
- 睡眠/带 RyzenHV VM 的唤醒未测
- Kiri K8s 单节点未测（compose/build 已验证）

## 4. 紧急恢复手册

| 症状 | 处理 |
|---|---|
| 开机黑屏 | 确认显示器输入源=HDMI 再重启（开机无 EDID 会退回 4K framebuffer） |
| 无限重启（早期 panic） | U 盘 EFI 启动内置 `macx` → 关闭最近加入的 kext / 还原 `config.plist.bak-*` → 重启 |
| GPU 异常 / 想禁用 GPU | `sudo touch /Users/Shared/gpu/phxboot-off` 后重启；恢复 `rm` 后再重启 |
| 显示花屏/撕裂 | 检查 `debug.phxfb_dbuf=1`、`NVMTL_WS_WAIT_COMPLETION=1` 是否在 conf（`od -c`） |
| SSH 连不上 | 用 `<hostname>.local` 重新解析（Wi-Fi/有线切换会换 IP） |
| 蓝牙消失 | 确认 `USBToolBox` + `UTBMap` 启用、`IntelBTPatcher` 启用、UTBMap 中 XHC0 PRT5=255 |
| Wi-Fi 连不上 | 看 airportd 日志 `security=`：`wpa3-sae` 不支持，路由器改 WPA2/混合 |

## 5. 重新启用项目时的检查清单

1. K8 上电，`ssh <user>@<K8-IP>` 或 mDNS 名；确认 `uptime`、`sw_vers -buildVersion`
2. `od -c /Library/GPUBundles/nvmtl/nvrm610.conf` 确认三键（历史上丢过 `WAIT_COMPLETION`）
3. `kmutil showloaded` 确认 kext 数（19 启用）、蓝牙/Wi-Fi/音频
4. 看 `/Library/Logs/DiagnosticReports` 是否有新的 WindowServer/panic
5. 关注上游：Mieze `RTL812xLucy` 1.1.4、nullarc 同步 pipeline 修复、ForbiddenItlwm 26.6 支持
6. 有突破时：先在本仓开分支记录假设 → 小步验证 → 更新 `FINAL-STATUS` 与本文件

## 6. 阶段一时间线索引

- 2026-11-18：装机、EFI 改造
- 11-19：GPU 接管（PhxBringup）、DSDT/kext 初版
- 11-20：vbl-timing + dbuf、音频、xHCI INTx、CPPC/GFXOFF、快速启动
- 11-21：设置页卡顿定位、蓝牙修复（映射 + BTPatcher）、Wi-Fi/WPA3 定位、AirDrop 结论、OpenCore RELEASE、kext 精简、WindowServer 崩溃分析
- 11-22：清缓存 + `NVMTL_PIPELINE_THREADS=1`（验证生效但覆盖不到 WS）、阶段存档
