# K8 Mini PC (Ryzen 7 H 255 / Radeon 780M) — macOS Tahoe 26.6.2 Hackintosh

> **阶段一存档（2026-11-22）**：项目阶段性结束，细节按主题拆分到 `docs/`。
> 现状、已知限制、恢复方法与重新启用清单见
> [docs/PHASE1-ARCHIVE-2026-11-22.md](docs/PHASE1-ARCHIVE-2026-11-22.md)。
> 若上游（nullarc / RTL8125 / AWDL）或硬件有突破，再继续更新。

在一台 **KTHIN 科偲 K8** 迷你主机上安装并跑通 macOS Tahoe 的完整记录：
OpenCore 引导、安装器 USB 丢中断修复、nullarc 780M 显卡接管，以及把桌面真正用起来踩过的坑。

> 本仓库是**脱敏后的公开版**：SMBIOS 序列号/MLB/UUID、网卡 MAC、内网地址、用户名均已替换为占位值；
> 第三方二进制（kext、EFI、安装包）不在仓库内，只给出出处与校验值。

![evidence](evidence/k8-desktop-2026-11-19-redacted.png)

## 先说结论：能用，但还不好用

现在这台机器可以完整跑起 macOS 桌面：GPU 接管、Metal 3、音频、蓝牙、Wi-Fi、容器都能用，
桌面合成稳定在 60 fps，空闲功耗降到 5–7 W。

**但距离"日常主力机"还有明显距离**——只解决了"可以用"，还没到"好用"：

- **稳定性没根治**：有线网卡驱动会触发内核 panic（已 3 次），WindowServer 会偶发崩溃（2 天 2 次），
  前者可能直接重启整机，后者会结束当前桌面会话、丢掉未保存的工作。
- **体验有硬伤**：设置页/墙纸面板 CPU 占用高、操作发涩；无 HiDPI；只有单 HDMI 输出；
  USB-C 不能出图也不能当扩展坞。
- **生态不完整**：AirDrop / Handoff / 接力不可用（Intel 网卡无 AWDL，蓝牙是第三方 dongle）；
  iPhone 与 Mac 之间只能靠局域网工具传文件。

所以这个仓库的定位不是"一台可以日常使用的黑苹果"，而是**把"能跑起来"的路径、参数和踩过的坑完整记录下来**，
让后来者少走弯路，也方便上游修复后快速复验。

## 状态速览（2026-11-22）

| 子系统 | 状态 | 说明 |
|---|---|---|
| 引导 | ✅ | OpenCore 1.0.8 RELEASE，内置盘独立启动，Timeout=5 |
| GPU / 显示 | ✅ | Radeon 780M，Metal 3；HDMI 2560×1600；桌面 60 fps |
| 帧同步 / 残影 | ✅ | vbl-timing：迟帧均值 ~85 µs、最大 ~1.5 ms；dbuf 双缓冲消残影 |
| 功耗 | ✅ | GFXOFF + CPPC：空闲 socket 5–7 W |
| 音频 | ✅ | VoodooHDA_BL：耳机/扬声器输出 + 模拟麦克风 |
| 蓝牙 | ⚠️ | AX200 `8087:0029` 可用（键鼠/耳机）；无 Handoff/AirDrop |
| Wi-Fi | ⚠️ | WPA2 / WPA2-WPA3 混合可连；WPA3-only 不支持 |
| 有线 | ⚠️ | 可用，但驱动存在 fq_codel panic 风险，大流量建议走 Wi-Fi |
| USB | ⚠️ | xHCI 走 INTx 稳定；USB-C 无视频（USB4 无 macOS 驱动） |
| 容器 | ✅ | Kiri + RyzenHV：compose / build 已验证；K8s 未测 |
| 睡眠/唤醒 | ⚠️ | 基础睡眠未验证；带 RyzenHV VM 的唤醒未测 |

## 离"日用"还差什么

| 问题 | 对日常的影响 | 现状 / 规避 |
|---|---|---|
| RTL8125 `fq_codel` panic | 大流量下载可能整机重启 | sysctl 缓解已验证无效；大流量走 Wi-Fi，等驱动 1.1.4 |
| WindowServer `radv` 崩溃 | 桌面会话重启，丢未保存工作 | 2 天 2 次；`NVMTL_PIPELINE_THREADS` 覆盖不到；可开"降低透明度" |
| 设置页/墙纸面板 CPU 高 | 面板内滚动/点选发涩 | nullarc 架构代价（CA 图层准备 + Metal 绘制），上游未修 |
| AirDrop / Handoff | 不能与 iPhone 直接互传/接力 | 无 AWDL；用 LocalSend / PairDrop 替代 |
| USB-C 无视频 | 不能接第二块屏、不能当扩展坞 | USB4 router 无驱动；需 USB DisplayLink 适配器 |
| 无 HiDPI | 字偏小、锐度一般 | PhxAMDFB 只报固件 GOP 单模式 |
| 蓝牙第三方 | 无 Apple 连续性功能 | Intel 卡 + BlueToolFixup，芯片层限制 |
| 睡眠/唤醒 | 长期挂机需手动关机 | 需实测（尤其带 VM） |

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/PHASE1-ARCHIVE-2026-11-22.md](docs/PHASE1-ARCHIVE-2026-11-22.md) | 存档总览、已知限制、恢复手册、重新启用清单 |
| [docs/build.md](docs/build.md) | 安装 U 盘、EFI 要点、GPU 接管、验收命令（原 §4–5） |
| [docs/pitfalls.md](docs/pitfalls.md) | 三个真正的坑、恢复手段（原 §3、§6） |
| [docs/display-gpu.md](docs/display-gpu.md) | 显示配置、vbl-timing/dbuf、卡顿、WindowServer 崩溃（§7.0–7.3/7.13/7.17） |
| [docs/bluetooth.md](docs/bluetooth.md) | AX200 蓝牙双层修复（§7.4/§7.14） |
| [docs/network.md](docs/network.md) | RTL8125、Wi-Fi WPA3、AirDrop（§7.7/§7.15/§7.16） |
| [docs/usb-xhci.md](docs/usb-xhci.md) | xHCI INTx、USB-C/第二屏（§7.9/§7.11） |
| [docs/audio-power.md](docs/audio-power.md) | 音频与空载功耗（§7.5） |
| [docs/cpu-pm.md](docs/cpu-pm.md) | AMD CPPC / EPP（§7.10） |
| [docs/boot.md](docs/boot.md) | OpenCore 与快速启动（§7.8） |
| [docs/upstream.md](docs/upstream.md) | 参考项目与上游跟进（§7.6/§7.12） |
| [docs/docker-kiri.md](docs/docker-kiri.md) | Docker / Kiri（§11） |
| [docs/evidence.md](docs/evidence.md) | 证据清单（§8） |
| [docs/license.md](docs/license.md) | 脱敏与许可（§9） |
| [docs/hardware-report.md](docs/hardware-report.md) | 硬件报告（Windows 侧 msinfo32/dxdiag 摘要） |
| [docs/k8-install-guide.md](docs/k8-install-guide.md) | 安装指南原文 |
| [docs/minipc-step0-checklist.md](docs/minipc-step0-checklist.md) | 第 0 步 GPU 清单 |

## 仓库结构

```
.
├── README.md                 本页：定位、状态、索引
├── docs/                     主题文档（上表）+ 存档说明 + 安装指南
├── efi/OC/config.plist       脱敏后的 OpenCore 配置模板
├── evidence/                 桌面截图与系统信息（已脱敏）
├── patches/                  源码补丁与参考配置
│   ├── k8-accel-vbl-timing-dbuf.patch
│   ├── nvrm610.conf
│   ├── UTBMap-k8-Info.plist
│   └── ...
└── THIRD_PARTY_NOTICES.md    第三方组件与许可
```

## 硬件与系统

| 部件 | 规格 |
|---|---|
| 机器 | KTHIN 科偲 K8（迷你主机，BIOS 2.09 / AMI 2.22.0059，纯 UEFI） |
| CPU | AMD Ryzen 7 H 255（8C/16T，Zen 4 Hawk Point，80W） |
| 核显 | Radeon 780M，PCI `1002:1900` |
| 内存 | 40 GB DDR5-5600（16+24） |
| 存储 | Lexar NM710 1TB（NVMe，整盘安装 macOS） |
| 网卡 | 2× RTL8125 2.5G 有线；Intel AX200（Wi-Fi/BT） |
| 显示 | HDMI 2560×1600@60（固件 GOP 点亮的单一模式） |
| 系统 | **macOS Tahoe 26.6.2（build 25G83）**，OpenCore 1.0.8 |

## 快速开始

1. 装机：按 [docs/build.md](docs/build.md) 做 U 盘与 EFI（注意安装器卡死与启动项两个坑）。
2. 装机前先读 [docs/pitfalls.md](docs/pitfalls.md)。
3. 装完对照 [存档文档](docs/PHASE1-ARCHIVE-2026-11-22.md) 的状态表与恢复手册验收。
4. 想日常用：先把 [docs/network.md](docs/network.md) 的 Wi-Fi 配好（有线大流量有 panic 风险）。

## 证据

- `evidence/k8-desktop-2026-11-19-redacted.png`：GPU 接管后的桌面（序列号/机器名/用户名已打码）
- `evidence/system-info.txt`：`sw_vers`、GPU、`PhxFB*`、`phxboot.log`、`phx-build`、AuxKC 加载记录
- 详细列表见 [docs/evidence.md](docs/evidence.md)

## 脱敏与许可

- SMBIOS（序列号/MLB/UUID）与 ROM/MAC 在 config 中为 `GENERATE-WITH-GenSMBIOS` / 全零；请用 GenSMBIOS 生成自己的
- 内网地址替换为 `192.0.2.x`（RFC 5737 文档网段），用户名替换为 `user`
- 第三方组件版权归各自作者，见 `THIRD_PARTY_NOTICES.md`；本仓库自有文字与配置以 MIT 发布
- 仅供学习研究；黑苹果与实验性 GPU 驱动有损坏系统/数据风险，操作前做好备份

## 致谢

- **Acidanthera**：OpenCore、Lilu、VirtualSMC、AMFIPass、RestrictEvents 等
- **Firebat U6C EFI**（`luuumi-dev`）：本机 EFI 的底座
- **`sfyyy/mechrevo-f7bsc-hackintosh`**：同代 7840H 迷你主机参考；`XhciNudge` 行为分析与 Tahoe USB/GPU 经验
- **`bbbbRT`**：`XhciNudge.kext` 原始二进制
- **`izerok/nullarc`**（`hdmi-fb2` 分支）：780M 加速与 fork 版 kext
- **Cobalt64 / nullmoth**：RDNA3 macOS 驱动研究
- **Mieze**：RTL812xLucy；**OpenIntelWireless / laobamac / lshbluesky**：AX200 驱动 fork
