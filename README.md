# K8 Mini PC (Ryzen 7 H 255 / Radeon 780M) — macOS Tahoe 26.6.2 Hackintosh

在一台 **KTHIN 科偲 K8** 迷你主机上安装并跑通 macOS Tahoe 的完整记录：OpenCore 引导、安装器、USB 丢中断修复、以及 nullarc 780M 显卡接管。

> 本仓库是**脱敏后的公开版**：SMBIOS 序列号/MLB/UUID、网卡 MAC、内网地址、用户名均已替换为占位值；第三方二进制（kext、EFI、安装包）不在仓库内，只给出出处与校验值。完整私有归档见 `izerok/k8-tahoe-hackintosh-full`（不公开）。

![evidence](evidence/k8-desktop-2026-11-19-redacted.png)

---

## 1. 硬件与系统

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

## 2. 结果

| 项目 | 状态 |
|---|---|
| OpenCore 引导 / 安装 | ✅ 无需 U 盘，内置盘独立启动 |
| 有线网络 | ⚠️ RTL8125（`RTL812xLucy` 1.1.3）：下载大流量曾触发 fq_codel panic；已禁用 TSO/ASPM 与 pacing/L4S（§7.7） |
| Wi-Fi / 蓝牙 | ⚠️ Wi-Fi 驱动已加载（en3，当前未关联、走有线）；蓝牙未加载（AX200 的 USB 蓝牙设备未枚举） |
| 音频 | ✅ VoodooHDA_BL 3.7.1（`15e3` HDA）：耳机/扬声器输出 + 模拟麦克风；codec 被标成 Conexant |
| GPU 接管（head 0） | ✅ `PhxAMDFB on the panel: 1`，Metal 3 |
| USB / HID | ✅ AMD xHCI 走 INTx（WujieXhciIntx），不再 2–4 ms 轮询；kernel_task ~0.9% |
| CPU 电源 | ✅ PhxCPPC，EPP 128；空闲 socket ~5–21 W（core 约 1–2 W） |
| 玻璃/毛玻璃效果 | ✅ |
| 已知缺陷 | 无常规 HiDPI、仅 HDMI 单屏、USB-C/DP 无输出、偶发 WindowServer 毛玻璃管线崩溃（§7.3） |

---

## 3. 三个真正的坑（也是本仓库的重点）

### 3.1 安装器卡死在 `validate_root_image`

**现象**：OpenCore 日志完全正常（所有 kext 注入成功、57 秒完成 `EXITBS`），但内核在
`validate_root_image: validating root dmg /BaseSystem/BaseSystem.dmg` 一行卡几十分钟到一小时以上。
同一根 U 盘、同一个镜像在 Mac 上读取 164 MB/s、校验 9 秒完成。

**根因**：AMD Zen 4（Phoenix / Hawk Point）在 macOS 下会**丢失 xHCI 的 MSI 完成中断**。
每一笔 USB 批量传输都要等到超时才发现"其实完成了"，有效读速从 164 MB/s 跌到 **< 0.4 MB/s**。
症状还包括 `MODE_SENSE_06/10 failed`、`BOT reset completed with kIOReturnTimeout`、`Still waiting for root device`。

**修复**：`XhciNudge.kext`（映射 local APIC，每 N 毫秒向每个 xHCI 的 MSI 向量补一次中断）：

```
boot-args: ... phxdebug=1 xhcinudge
```

- `xhcinudge=<ms>` 是注入间隔，**单位毫秒**；`0` 关闭；默认 4
- 调成 `1` 时空闲 `kernel_task` 会到 30–40%，`4` 约 6.7%
- **不能移除**：`=0` 时表现为按键连发、鼠标极慢、大文件读取卡死
- **2026-11-20 起已被取代**：改用 `WujieXhciIntx` 的 INTx 模式（§7.9），
  `XhciNudge.kext` 禁用、boot-args 换成 `wujiexhci`；此段保留作背景与回退说明
- 来源：`bbbbRT` 的 EFI（仅二进制），经 `sfyyy/mechrevo-f7bsc-hackintosh` 的 `docs/known-issues.md` 逆向确认行为

### 3.2 纯 UEFI 固件的启动项问题

这块 BIOS **没有 CSM、没有 Above 4G、没有 AMD CBS/NBIO 菜单**，也无法手动添加启动项。
装完 macOS 后固件仍优先走残留的 `NVME:Windows Boot Manager`（Windows 已删，但 ESP 里 `Microsoft/` 目录还在）。

**修复**：

1. 删除内置 ESP 的 `EFI/Microsoft/`（Windows 残留，删除前确认 Windows 数据已备份）
2. BIOS → Boot → `Boot Option #1 = Hard Disk`（该通用项会加载第一块硬盘的 `\EFI\BOOT\BOOTX64.efi`）
3. 保留 U 盘 EFI 作为备用启动项

### 3.3 GPU：官方 nullarc v0.4.0 接管挂死，fork 版可用

**现象**：官方 `nullarc-0.4.0-alpha` 安装、`install.sh --check` 全绿，第一次自检启动正常；
**第二次接管启动**直接卡在内核早期（`AppleACPICPU: processor id=12 ... enabled`），网络不通、无 panic。

**修复**：换用 `hdmi-fb2` fork 的两个 kext（`izerok/nullarc`，分支 `hdmi-fb2` @ `50db1e1`）：

| 文件 | SHA-256 |
|---|---|
| `nullarc-hdmi-fb2-kexts.zip` | `0b4b6c59a3c6f1a5553acf0a8774e9581f9a66356888721a5786cbfa5749b1fd` |
| `PhxBringup.kext/Contents/MacOS/PhxBringup` | `dadc7cc625963aa69d02db68d50a897ce8fcf442759870f6cb8ae87d005c7a37` |
| `PhxAccel.kext/Contents/MacOS/PhxAccel` | `f55a78d6b69879a63c3e2496ef8c21a4745e9b57b861b2c30f9c0c37f571877d` |

替换后 `phx-build = 37`（官方为 34）、AuxKC 中 `PhxAccel` 大小 `0x1d1f2`（官方 `0x1b482`）。
接管成功的证据：

```
PhxAMDFB on the panel: 1
armed and released at 22:46:01; agdc 1
PhxFBWidth 2560 / PhxFBHeight 1600 / PhxFBRefreshMilliHz 60000
Metal 3
debug.nvaccel_iop_ok 持续增长
```

**2026-11-20 运行组合（更新）**：现运行 **fork `hdmi-fb2` build 37 的 PhxBringup + upstream main `3497d87` 编译的 PhxAccel**。main 的 PhxAccel 已按 `phxbringup-ptfix` 语义把页表 64→256、VRAM hi 1536→3840 MiB、GTT 512→2048 MiB 写入，并手工移植了 `vbl-timing` / `dbuf` 补丁（见 §7.2）；接管正常（`PhxAMDFB on the panel: 1`，Metal 3）。之所以保留 fork 的 PhxBringup：**main `3497d87` 没有 `debug.phx_pm_*` PM 代码**，K8 Tune 的 GPU 分档会失效；fork build 37 与 main PhxAccel 的 `phx-fb-*` 属性 ABI 兼容，实测正常。两套 kext 与 AuxKC 均已备份为回滚点。

---

## 4. 复现步骤

### 4.1 安装 U 盘

```bash
softwareupdate --list-full-installers
softwareupdate --fetch-full-installer --full-installer-version 26.6.2
# 若报 Code 203 "数码签名丢失或无效"，先查磁盘空间（下载缓存 + 解包需要 ~36 GB）
sudo diskutil eraseDisk JHFS+ "USB" GPT /dev/disk4
sudo "/Applications/Install macOS Tahoe.app/Contents/Resources/createinstallmedia" \
     --volume /Volumes/USB --nointeraction
```

> 原盘若是 MBR，`createinstallmedia` 不会创建 EFI 分区，OpenCore 无法启动——必须先 `eraseDisk ... GPT`。

### 4.2 EFI 要点

以 Firebat U6C 的 EFI 为底座（同款 H 255 APU），针对 K8 修改：

- **关闭**笔记本专用项：`VoodooI2C`(+插件)、`VoodooI2CHID`、`VoodooPS2Controller`(+插件)、`HoRNDIS`、`RtWlanU`、`SSDT-USB-Reset`
- **新增（2026-11-19）**：`XhciNudge.kext`（`MinKernel 21.0.0`）、`PhxBringup.kext`（`MinKernel 25.0.0`）、有线网卡 `RTL812xLucy`、`AirportItlwm`(Tahoe fork)、`IntelBluetoothFirmware`/`IntelBTPatcher`(Tahoe fork)、`BlueToolFixup`
- **2026-11-20 变更（与当前模板一致）**：
  - **禁用** `XhciNudge.kext` 与 `IntelBTPatcher.kext`（后者在 Tahoe 上 Hook 私有符号可致 Kernel Trap 14）
  - **新增** `WujieXhciIntx.kext`（xHCI INTx，§7.9）、`VoodooHDA_BL.kext`（音频，§7.5）、
    `PhxCPPC.kext`（CPU PM，§7.10）、`RyzenHV.kext`（Docker/Hypervisor 后端，§9）
  - boot-args 见 §3；完整启用列表以 `efi/OC/config.plist` 为准
- **boot-args（当前模板）**：`keepsyms=1 msgbuf=1048576 revpatch=cpuname,sbvmm ipc_control_port_options=0 amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta phxdebug=1 wujiexhci rhv_enable_svm=1 rhv_allow_users=1 rhv_hv=1`（无 `-v`、无 `debug=0x100`——前者只是观感，后者会让 panic 停在黑屏；`wujiexhci` 见 §7.9，`rhv_*` 见 §9）
- **SMBIOS**：`MacBookPro16,1` + GenSMBIOS 生成序列号/MLB/UUID + `ROM = 有线网卡 MAC`
- `csr-active-config=7F080000`、`SecureBootModel=Disabled`、`DummyPowerManagement=true`、`ProvideCurrentCpuInfo=true`
- 仓库内 `efi/OC/config.plist` 为**脱敏模板**（SMBIOS 为占位符），`efi/OC/ACPI/*.dsl` 为 ACPI 源文件

### 4.3 安装与收尾

1. 后置 USB 口直插（不建议经扩展坞），OpenCore → Install macOS Tahoe
2. 磁盘工具：整盘抹 `APFS / GUID` → 安装
3. 装完把 U 盘 EFI 复制到内置盘 ESP：
   `diskutil mount disk0s1` → `ditto` → `diskutil unmount`
4. 删除 ESP 里的 `EFI/Microsoft/`，BIOS 启动项设为 `Hard Disk`

### 4.4 GPU（nullarc）

```bash
# 官方包 + fork 两个 kext
unzip nullarc-0.4.0-alpha.zip && cd nullarc-0.4.0-alpha
# 用 fork 版替换：
#   EFI/OC/Kexts/PhxBringup.kext
#   Library/Extensions/PhxAccel.kext
sudo ./install.sh --check
sudo ./install.sh --windowserver
# 打开自动登录（设置 → 用户与群组），否则登录窗可能黑屏
# 重启两次：第一次自检并 arm，第二次接管
```

若 `install.sh` 因 `PhxBringup.kext is not loaded` 拒绝执行（例如从 U 盘 EFI 启动），可单独发布 AuxKC：

```bash
sudo cp -R PhxAccel.kext /Library/Extensions/
sudo BK=/Users/Shared/gpu/backup /Users/Shared/gpu/kinst.sh
```

---

## 5. 验收命令

```bash
sw_vers                                   # 26.6.2 / 25G83
ioreg -l | grep phx-build                 # phx-build = 37
ioreg -rc PhxAMDFB -d0 | grep PhxFB       # PhxFBWidth/Height/RefreshMilliHz
system_profiler SPDisplaysDataType        # 780M / Metal 3
sysctl debug.nvaccel_iop_ok               # 随画面变化增长
tail -5 /var/log/phxboot.log              # done / armed and released
```

## 6. 恢复手段（重要）

| 手段 | 作用 |
|---|---|
| boot-arg `-phxfboff` | PhxAMDFB 不接管屏幕，回固件帧缓冲（**仓库模板 config 不含它**；救砖用的 U 盘 EFI 曾加过，需要时临时加到 boot-args 再启动） |
| `sudo touch /Users/Shared/gpu/phxboot-off` | 阻止下次启动 arm 接管 |
| `sudo nvram -d phxfb-boot` | 清除握手变量 |
| `sudo ./uninstall.sh` | 卸载 nullarc（恢复前先看官方 `docs/INSTALL.md`） |
| U 盘 EFI 启动 | 挂死时的救援路径（OpenCore 在 U 盘上） |

## 7. 已知问题与已验证的缓解

- **残影（2026-11-20 已解决）**：nullarc 0.4.0 的合成等待缺陷。§7.1 的 `surface_gpu_on=0` 单独只能减轻；
  叠加 §7.2 的 `debug.phxfb_dbuf=1` 后**用户确认残影消失**（双缓冲在 VUPDATE 翻转，旧帧不再被扫描到）
- **撕裂 / 67 ms 卡顿**：**67 ms 已修复**（`debug.phxfb_timing`，见 §7.2）；撕裂的 `dbuf` 已移植，默认关闭，`sudo sysctl -w debug.phxfb_dbuf=1` 开启，待快速滚动/拖动的视觉验收
- **WindowServer CPU 偏高**：官方插件每帧开销大；参考项目的 `nvmtl-*` 性能补丁可降到 25–40%
- **有线网卡 panic**：`RTL812xLucy` v1.1.3 在 fq_codel 断言（`classq_fq_codel.c:1127`）下崩溃过一次，panic 栈直接指向该 kext（`RTL8125::outputStart`），属驱动侧 bug；暂未复现
- **窗口捕获崩溃（已修复，2026-11-20）**：根因是 PhxBringup 每进程 GPU 页表 2 GiB 上限 + 显存池偏小。按 `phxbringup-ptfix` 语义把页表 64→256 块（≈8 GiB）、VRAM hi 1536→3840 MiB、GTT 512→2048 MiB（单块 1024）后，截屏/录屏/锁屏不再崩，锁屏动态壁纸正常；`vkfill` 可申请总量 2032 → **6128 MB**。若复现，内核日志会出现 `PhxBringup: GPU page tables of context N full`。
- **Vision/OCR 闪退（已修复，2026-11-20）**：插件加载 `nn-hide` 补丁（`patches/nvmtl-nn-hide.patch`），让 Core ML/Vision 的推理栈看不到 GPU、改在 CPU 推理，程序自身 Metal 仍用 780M；QQ 扫码、Shottr OCR 不再崩溃。
- **无 HiDPI**：PhxAMDFB 只报告固件 GOP 的单模式
- **仅 HDMI 单屏**：0.4.0 不驱动外接 PHY
- **锁屏（2026-11-20 复测：正常）**：同一池上限修复后，锁屏/解锁正常，动态壁纸在锁屏界面正常显示；早先的 watchdog 卡死不再复现。
- **USB-C / DP 外接显示器无信号**：nullarc 0.4.0 不驱动外接显示 PHY，只靠固件 GOP 在 **HDMI head 0** 上点亮。实机验证：`system_profiler` 只报一个 `Internal` 显示；`debug.phxfb1_hw` 不存在（head 1 未实现）；DP 热插拔寄存器 `0x53e4` = `0x00000000`。USB-C 口在 macOS 下仅数据可用。新 main `3497d87` 增加了动态 **HDMI** 第二头（`debug.phxfb_hdmi`），但 **USB-C/DP 仍未驱动**（无 `phxfb1_hw`、`0x53e4=0`），第二块屏仍需等 `phxfb2`。
- **显存 2 GB**：由 BIOS 的 UMA Frame Buffer 分配决定；2560×1600 够用，想上 4K 建议调大
- **显示器尺寸误报**：关于本机里"42 英寸"是 PhxAMDFB 报的尺寸，实际为 27 英寸，不影响使用
- **音频（2026-11-20 已解决）**：Tahoe 移除了 `AppleHDA`，但笔记本线（Wujie 14 Pro）的
  `VoodooHDA_BL.kext` 3.7.1 直接匹配 AMD `15e3` HDA。经 OpenCore `Kernel → Add` 注入后，
  系统出现“Conexant (Unknown): Headphones”默认输出和“Microphone (Analog)”输入，`afplay` 正常。
  注意：Tahoe 会拒绝未签名 kext 的运行时加载，必须走引导注入；同时已禁用 `IntelBTPatcher.kext`
  （Tahoe 上 Hook 私有符号可致早期 Kernel Trap）
- **OpenCore 为 DEBUG 版**（`Misc → Debug → Target = 67`），稳定后可换 RELEASE 或把 Target 调小
- **选单等待（2026-11-20 已验证）**：`Timeout=5` + `ShowPicker=true`；实机 OpenCore 日志
  `OCB: Showing menu...`（04.319 s）→ `OCB: Should boot from 1. macx`（09.399 s），间隔 **5.08 秒**，
  无需按键自动进入 `macx`

### 7.0 空闲高频（nullarc #14）已修复（2026-11-20）

nullarc 的 `mesboot` 在建立 gfx 队列前没有写 `CP_MEC_DOORBELL_RANGE`，命令处理器永久报忙，PMFW 把 GFXCLK 钉在 2600 MHz，空闲也发热、风扇狂转。

修复：用 `izerok/nullarc` 的 **`gfx-idle` 分支**（commit `994fa0f`，"mesboot: set CP_MEC_DOORBELL_RANGE before the first gfx doorbell"）重新编译 `mesboot`，替换 `/Users/Shared/gpu/phxctl/mesboot` 后重启（只动用户态二进制，不改 kext/AuxKC）：

```bash
git clone --depth 1 --branch gfx-idle https://github.com/izerok/nullarc
clang -O2 -o mesboot nullarc/tools/phxctl/mesboot.c -framework IOKit -framework CoreFoundation
sudo cp mesboot /Users/Shared/gpu/phxctl/mesboot && sudo chown root:wheel $_ && sudo chmod 755 $_
```

实测：空闲 `debug.phx_pm_gfxclk` 从 **2600 → 800 MHz**（`mode=1`、上限 2600 也不影响空闲），接管与 Metal 3 正常。

### 7.1 已验证的显示配置（2026-11-19）

```bash
# 立即生效
sudo sysctl -w debug.nvaccel_surface_gpu_on=0

# /Library/GPUBundles/nvmtl/nvrm610.conf 追加（每个进程启动时读取，注销后生效）
NVMTL_WS_WAIT_COMPLETION=1
NVMTL_SCHEDULE_AFTER_SURFACE_WRITES=1
```

`debug.*` sysctl 每次重启会重置，需要开机自启（参考实现：`/usr/local/bin/phxdisplay.sh` + `/Library/LaunchDaemons/local.phxdisplay.plist`，`RunAtLoad` 执行 `sysctl -w debug.nvaccel_surface_gpu_on=0`）。

验证：`sysctl debug.nvaccel_surface_gpu_on` 为 0；插件二进制中存在上述两个 `NVMTL_*` 键；`debug.nvaccel_iop_fail` 保持 0。
实测结果：注销重登后残影**明显减轻但仍有残留**；再叠加 §7.2 的 `debug.phxfb_dbuf=1` 后，**用户确认残影消失**（2026-11-20）。

### 7.2 vbl-timing / dbuf 手工移植（2026-11-20）

参考项目的 `phxaccel-vbl-timing.patch` / `phxaccel-dbuf.patch` 对 fork 和 main `3497d87` 都冲突，已按 main 源码手工移植
（`patches/k8-accel-vbl-timing-dbuf.patch`）：

- **vbl-timing（67 ms 卡顿）**：`debug.phxfb_timing`（默认 1）让 `getTimingInfoForDisplayMode` 返回
  `kIODetailedTimingValid`，并把 `measureRefresh()` 提前到 `IOFramebuffer::start()` 之前。
  K8 上 OTG 实测总量 `2720x1481` **小于** 2560x1600 surface（固件缩放），参考补丁会因 `fVTotal <= fHeight`
  提前返回、无法进入 VBL timing 模式；移植版在 totals 不覆盖 surface 时补标准 blanking。
  验证：`ioreg -rc PhxAMDFB` 出现 `IOFBCurrentPixelClock = 268627210`；CVDisplayLink 5 秒采样
  **300 callbacks、out avg 16.667 ms、0 个 >20 ms**；修复前为 16.781 ms 步进并反复出现 **66.7 ms**（4 帧）空档。
- **dbuf（撕裂）**：`debug.phxfb_dbuf` 默认 0；`N > 1` 表示试运行 N 秒后自动回到单缓冲。
  实测 `=30`：`state=1`、30 秒 1027 flips、`wait_timeouts=0`、自动回退 `homes=1`；
  常开 `=1` 后 CVDisplayLink 仍稳定。**用户确认开启后残影消失**（2026-11-20），已把 `debug.phxfb_dbuf=1`
  写入 `/usr/local/bin/phxdisplay.sh`（开机重试直到 OID 可用）持久化。临时关闭：`sudo sysctl -w debug.phxfb_dbuf=0`。
- 驱动侧另有 `debug.phxfb_dbuf_state` / `_flips` / `_waits` / `_wait_timeouts` / `_wait_max_us` 可观测。

### 7.3 偶发 WindowServer 崩溃（毛玻璃管线）与剩余卡顿

- 2026-11-20 14:48 出现一次黑屏：`WindowServer` SIGSEGV（`EXC_BAD_ACCESS 0x212`），launchd 随即重启；
  崩溃栈为 `CA::OGL::BlurState::narrow_blur` → `CA::OGL::GaussianBlurFilter::render` →
  `MetalContext::draw` → `nvmtl_vk_pipeline_create_rt` → `radv_CreateGraphicsPipelines`
- 这是 **NVMTL/RADV 创建毛玻璃（backdrop blur）渲染管线时的驱动段错误**，不是 `dbuf`/`vbl-timing`：
  当时 `phxfb_dbuf_wait_timeouts=0`、`nvaccel_iop_fail=0`，显示翻转正常
- 缓解：`defaults write com.apple.universalaccess reduceTransparency -bool true`（下次登录生效），
  减少系统毛玻璃图层与 blur 管线；撤销：把 `true` 改成 `false`
- 卡顿量化：简单 60 fps 动画 58.8 fps、无 >40 ms 空档；3 个毛玻璃页面 56.9 fps，
  约 10–13% 的帧间隔为 33 ms，WindowServer CPU 27–46%。残余卡顿是 **blur 合成 CPU 开销**，
  `dbuf`/`phx_pm_mode`/`surface_gpu_on` 的 A/B 都在噪声内；67 ms 空档已由 §7.2 修掉

### 7.4 蓝牙：AX200 的 USB 蓝牙设备未枚举

Wi-Fi 侧 `pci8086,2723`（AX200）存在且 `AirportItlwm` 已加载；蓝牙应为 USB `8087:0029`，但 IORegistry
全树没有该设备，`IntelBluetoothFirmware` 不加载，`system_profiler` 只显示假的 `BCM_4350C2`（UART，State Off）。

已启用 `USBPorts.kext`（K8 xHCI map，含 15b9/15ba 的 HS05 internal 端口）并重启验证：键盘/鼠标正常，
但 XHC3/XHC4 的 internal USB2 `PRT1`（connector 9）仍为空。判断不是 kext 或端口映射问题，而是
**BT 模块 USB 未供电/未接线，或 BIOS 关闭了 Bluetooth**；下一步是 BIOS 检查、彻底断电冷启动，
或使用 USB 蓝牙 dongle。

### 7.5 音频与空载功耗（2026-11-20）

- **音频**：`VoodooHDA_BL.kext` 3.7.1（Wujie 14 Pro 笔记本线）经 OpenCore `Kernel → Add` 注入，
  `IOPCIPrimaryMatch 0x15e31022` 命中 K8 的 `AZAL`；输出/麦克风正常，codec 名误报为 Conexant
- **GFXOFF**：`debug.phx_pm_gfxoff=1`（fork PhxBringup build 37 的 PM）让 PMFW 空闲时给 GPU 断电；
  空闲 socket 47 W → 27 W，`gfxoff_state=1`，有活动时自动唤醒
- **时钟门控**：`gpucg.sh all`（GC 11.0.1 的 MGCG/CGCG/CGLS，15XA 笔记本线）再降到 **23.4 W**，
  温度 54.5 → **50.1 °C**；GPU 仍 800 MHz、Metal 3、WindowServer 正常
- 持久化：`debug.phx_pm_gfxoff=1` 在 `k8tuned.sh`；`gpucg.sh all` 在 `phxdisplay.sh` 开机重试执行
- 蓝牙虽无硬件，但**不要禁用 `bluetoothd`**：试过 `launchctl disable system/com.apple.bluetoothd` 后，
  `ContextStoreAgent`/`logd`/`launchd` 会以 56%/36%/136% CPU 自旋重试，空闲功率反而升到 40 W；
  已 `launchctl enable` + `bootstrap system /System/Library/LaunchDaemons/com.apple.bluetoothd.plist` 恢复。
  它每 8–9 分钟 EXC_GUARD 重启一次的代价远小于这个自旋。

### 7.6 780M 笔记本线 vs 迷你主机线：还可推进什么

- **笔记本线（Wujie 14 Pro / 15XA）**：USB-C DP 第二屏（`phxfb2=1`/`phxfb3=1` + `tools/hdmi` 的 DMUB/DP 训练）；
  `WujieXhciIntx` 用 INTx 取代轮询；CPU PM/CPPC；快速静音启动；HiDPI；BTLEAdvFixup/ForbiddenItlwm（需 BT 硬件）
- **迷你主机线（F7BSC）**：vbl-timing、dbuf、4 个 NVMTL 补丁、ptfix 池上限、gfx-idle mesboot、thermal 脚本、§7.1 绕法
- **K8 已采纳**：vbl-timing/dbuf/NVMTL/ptfix/gfx-idle/thermal、VoodooHDA_BL、GFXOFF、gpucg、禁用 IntelBTPatcher/BT daemon
- **下一阶段优先级**：
  1. USB-C/HDMI 第二屏（高价值/高风险，需要 K8 VBIOS + DMUB 固件 + 不回读 PhxBringup；K8 USB-C 是否 DP Alt Mode 未验证）
  2. `WujieXhciIntx`（中高风险，需替换 XhciNudge，失败会丢键鼠）
  3. CPU PM/CPPC（中风险，可能与 DummyPowerManagement 冲突）
  4. 快速启动/去 `-v`（低风险，纯观感）

### 7.7 RTL8125 fq_codel panic（2026-11-20）

Chrome 点击下载按钮后整机挂死，panic：

```
assertion failed: now >= *pkt_enq_time
classq_fq_codel.c:1127
fq_codel_dq → fq_if_dequeue → ifnet_dequeue → RTL8125::outputStart
com.insanelymac.RTL812xLucy 1.1.302
```

与 2026-11-19 22:11 的 panic 同栈，是网卡驱动 bug，不是 GPU/音频。已做：
- 网卡 kext `Driver Parameters`：`enableTSO4/6=false`、`enableAspmL0s/L1=false`
- 开机守护 `local.nicfix`：`net.classq.fq_codel.enable_pacing=0`、`enable_l4s=0`
- boot-args 去掉 `debug=0x100`（panic 自动重启，不再黑屏空转）

验证：`ifconfig en1` options 从 `46b` 变 `40b`（无 TSO）；100 MB 单流 + 2×100 MB 并行下载未复现。
用户复测 Chrome 下载也**不再崩溃**。若仍复现：禁用 RTL8125，改用 AX200 Wi-Fi 或 USB 网卡。

### 7.8 快速启动（2026-11-20）

- boot-args 去掉 `-v`（保留 `keepsyms=1`、`msgbuf`）
- `Misc.Debug.Target=1`（只 console，不向 ESP 写 `opencore-*.txt`）、`AppleDebug=false`、`LogModules=""`
- `ocvalidate 1.0.8` 校验：No issues found
- 实测重启后 SSH 约 30 秒回来（之前 45–55 秒），ESP 不再新增 OpenCore 日志；`Timeout=5` 不变

### 7.9 xHCI INTx（取代 XhciNudge 轮询，2026-11-20）

AMD Phoenix 的 xHCI 只有一个 MSI 向量且只由 interrupter 0 触发，AppleUSBXHCI 的非等时事件在 interrupter 1，
所以 HID 事件会卡/只能用轮询。引入 15XA 笔记本线的 `WujieXhciIntx.kext`：

- 禁用 `XhciNudge.kext`，boot-args `xhcinudge` → `wujiexhci`
- 启动 45 秒后切 INTx；之前 2 ms 轮询兜底
- 实测 `debug.wujiexhci`、`wujiexhci_irqs` 递增、`spurious/storms=0`，键鼠正常；
  `kernel_task` 约 0.9%（XhciNudge 轮询时约 6–7%）
- 回退：`sysctl -w debug.wujiexhci=0` 即时回 MSI

### 7.10 AMD CPPC / EPP（2026-11-20）

用 nullarc PR #21 的 `PhxCPPC.kext`（MSR CPPC），`/usr/local/bin/cppc.sh` 开机设
`debug.phx_cppc_enable=1`、`max/min=-2`、`epp=128`、`apply=1`：

- 空闲 core power **27 W → 1–2 W**，socket **35 W → ~5 W**，温度 51 → 41–45 °C
- 4×`yes` 满载 socket ~36 W，结束后自动回落
- 查看：`sysctl debug.phx_cppc_dump`；EPP 0=性能 … 255=省电

### 7.11 USB-C / HDMI 第二屏：K8 是 USB4 DP tunneling，macOS 无驱动

K8 只有 **1× HDMI 2.1**；USB-C 在 Windows 下能出视频，但走的是 **USB4 DP tunneling**：
Windows `msinfo32` 有 AMD USB4 host router `1022:1668`/`1022:1669`，`usb4hostrouter` 运行中，
另有 `displaymux`、`UCM-TCPCI`，不是原生 DP Alt Mode。

实测用 QiRanChenk/nullarc `hdmi-fb2`（`phxfb2=1`/`phxfb3=1`）+ K8 VBIOS
（`113-PHXGENERIC-001`）+ linux-firmware `dcn_3_1_4_dmcub.bin` + 15XA `tools/hdmi`：
`0x53dc=0x12`（HDMI）、`0x53e4=0`（DP/USB-C 无 HPD），`dmubboot aux hpd` 返回 “nothing”，
DP 训练无法开始。已回滚到 main vbl/dbuf Accel；工具快照在私有 `patches/hdmi-tools/`。
替代：USB **DisplayLink** 转 HDMI/DP 适配器（走 USB 数据，不需要 USB4/DP tunneling），
或换带 macOS 支持的 DP/USB4 的机器。

## 8. 证据

- `evidence/k8-desktop-2026-11-19-redacted.png`：GPU 接管后的桌面（序列号/机器名/用户名已打码）
- `evidence/system-info.txt`：`sw_vers`、GPU、`PhxFB*`、`phxboot.log`、`phx-build`、AuxKC 加载记录

## 9. 脱敏与许可

- SMBIOS（序列号/MLB/UUID）与 ROM/MAC 在 config 中为 `GENERATE-WITH-GenSMBIOS` / 全零；请用 GenSMBIOS 生成自己的
- 内网地址替换为 `192.0.2.x`（RFC 5737 文档网段），用户名替换为 `user`
- 第三方组件版权归各自作者，见 `THIRD_PARTY_NOTICES.md`；本仓库自有文字与配置以 MIT 发布
- 仅供学习研究；黑苹果与实验性 GPU 驱动有损坏系统/数据风险，操作前做好备份

## 10. 致谢

- **Acidanthera**：OpenCore、Lilu、VirtualSMC、AMFIPass、RestrictEvents 等
- **Firebat U6C EFI**（`luuumi-dev`）：本机 EFI 的底座
- **`sfyyy/mechrevo-f7bsc-hackintosh`**：同代 7840H 迷你主机参考；`XhciNudge` 行为分析与 Tahoe USB/GPU 经验
- **`bbbbRT`**：`XhciNudge.kext` 原始二进制
- **`izerok/nullarc`**（`hdmi-fb2` 分支）：780M 加速与 fork 版 kext
- **Cobalt64 / nullmoth**：RDNA3 macOS 驱动研究
- **Mieze**：RTL812xLucy；**OpenIntelWireless / laobamac / lshbluesky**：AX200 驱动 fork

## 11. Docker / Kiri（2026-11-20）

K8 上可以在 AMD Ryzen 黑苹果里跑容器：**RyzenHV**（`OverlordDev/ryzen-hypervisor.framework`，MIT）
内核扩展用 AMD-V/SVM 实现了 Apple `Hypervisor.framework` 的后端（`kern.hv_support` 原本为 0，
这是 Kiri/Docker Desktop/VirtualBox 等所有走该框架的 VM 应用在 AMD 上起不来的共同根因）。

- RyzenHV 0.7.8 **native 模式**：kext 经 `hv_support_kext` 接口注册为 hypervisor provider，
  boot-args `rhv_enable_svm=1 rhv_allow_users=1 rhv_hv=1`；`rhv-info hv` 报 SVM/NPT/TPR shadow 全部就绪
- `kern.hv_support` 每次重启后回 0，由 `rhv autostart install` 的 LaunchDaemon 置 1
- **Kiri 0.2.2**（走 Hypervisor.framework 的容器引擎）：装 RyzenHV 后从
  `VM did not boot: this Mac does not support virtualization` 变为完全可用——
  `hello-world`、`nginx -p 8080:80` + `curl` 200、`ps/rm` 均正常
- Docker Desktop 4.97.0 验证可运行（RyzenHV native 模式无需 shim），本机已卸载、由 Kiri 覆盖需求
- **`/var/run/docker.sock` 默认不存在**（Docker Desktop 才创建它）：`docker load -i xxx.tar` 之类
  直接连 `/var/run/docker.sock` 的命令会报 `no such file or directory`。
  修复：`sudo kiri setup --docker-socket`，创建 `/var/run/docker.sock → ~/.kiri/run/docker.sock`
  （只影响 docker-py/Testcontainers 这类忽略 context 的工具；`kiri`/`docker` CLI 走 context 不受影响）
- 实测 `docker load` 169 MB 镜像包 → 4 个 tag 全部载入；容器运行、端口发布、出站网络均正常
- 已知边界：RyzenHV 实验性（无 entitlement 检查、睡眠/唤醒未验证、Tahoe 非上游实测版本）；
  compose/build/K8s（`kiri k8s start`）尚未测试；容器内 netlink 操作（如防火墙规则）因权限不足失败，属容器沙箱正常行为
