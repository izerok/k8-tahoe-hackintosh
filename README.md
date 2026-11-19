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
| 有线网络 | ✅ RTL8125（RTL812xLucy） |
| Wi-Fi / 蓝牙 | ⚠️ 驱动已注入，未做完整实测 |
| GPU 接管（head 0） | ✅ `PhxAMDFB on the panel: 1`，Metal 3 |
| 玻璃/毛玻璃效果 | ✅ |
| 已知缺陷 | 残影、WindowServer CPU 偏高、无常规 HiDPI、仅 HDMI 单屏 |

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
boot-args: ... phxdebug=1 xhcinudge=4
```

- `xhcinudge=<ms>` 是注入间隔，**单位毫秒**；`0` 关闭；默认 4
- 调成 `1` 时空闲 `kernel_task` 会到 30–40%，`4` 约 6.7%
- **不能移除**：`=0` 时表现为按键连发、鼠标极慢、大文件读取卡死
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
- **新增**：`XhciNudge.kext`（`MinKernel 21.0.0`）、`PhxBringup.kext`（`MinKernel 25.0.0`）、有线网卡 `RTL812xLucy`、`AirportItlwm`(Tahoe fork)、`IntelBluetoothFirmware`/`IntelBTPatcher`(Tahoe fork)、`BlueToolFixup`
- **boot-args**：`-v keepsyms=1 debug=0x100 msgbuf=1048576 revpatch=cpuname,sbvmm ipc_control_port_options=0 amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta phxdebug=1 xhcinudge=4`
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

- **残影**：nullarc 0.4.0 的合成等待缺陷。已验证缓解（见 §7.1）：残影**明显减轻但未完全消失**
- **撕裂 / 67 ms 卡顿**：需要 fork 的 `phxaccel-dbuf` / `phxaccel-vbl-timing` 补丁；官方 0.4.0 的 kext **没有** `debug.phxfb_dbuf`
- **WindowServer CPU 偏高**：官方插件每帧开销大；参考项目的 `nvmtl-*` 性能补丁可降到 25–40%
- **有线网卡 panic**：`RTL812xLucy` v1.1.3 在 fq_codel 断言（`classq_fq_codel.c:1127`）下崩溃过一次，panic 栈直接指向该 kext（`RTL8125::outputStart`），属驱动侧 bug；暂未复现
- **WindowServer 崩溃（黑屏几秒/回登录）**：任何**窗口捕获**都会触发 nullarc 的 Metal 缺陷——`MetalBacking::AbortWithTextureInfo` → `MetalIOSurfaceBacking::PrepareForUse` → `CaptureSurfaceMetal::Populate` → `WS::Capture::create_iosurface_for_window_list`，WindowServer 以 SIGABRT 退出。实测触发场景：**切换壁纸**（系统设置预览会抓窗口 + 动态壁纸还在解码视频）、截屏、录屏、共享屏幕。WindowServer 会自动重启，开启自动登录可自动回到桌面；**日常用静态壁纸、避免在 GPU 路径下做窗口捕获**。
- **无 HiDPI**：PhxAMDFB 只报告固件 GOP 的单模式
- **仅 HDMI 单屏**：0.4.0 不驱动外接 PHY
- **锁屏也会触发同一缺陷（2026-11-20 新增）**：锁屏界面会做一次窗口捕获，同样命中 `MetalBacking::AbortWithTextureInfo`；更糟的是崩溃后重启的 WindowServer 可能主线程卡死，被 `userspace_watchdog_timeout` 杀掉（实测 40 秒无响应），表现为持续黑屏。`phxboot.sh` 检测到上一次启动未完成会自动创建 `phxboot-off` 关闭 GPU，需删除该文件并重启才能恢复加速。**日常不要锁屏/截屏/录屏/共享屏幕**；可在 系统设置 → 锁屏 中把"需要密码"设为"永不"、屏保设为"永不"，避免误触。
- **USB-C / DP 外接显示器无信号**：nullarc 0.4.0 不驱动外接显示 PHY，只靠固件 GOP 在 **HDMI head 0** 上点亮。实机验证：`system_profiler` 只报一个 `Internal` 显示；`debug.phxfb1_hw` 不存在（head 1 未实现）；DP 热插拔寄存器 `0x53e4` = `0x00000000`。USB-C 口在 macOS 下仅数据可用，第二块屏要等驱动支持 `phxfb2`。
- **显存 2 GB**：由 BIOS 的 UMA Frame Buffer 分配决定；2560×1600 够用，想上 4K 建议调大
- **显示器尺寸误报**：关于本机里"42 英寸"是 PhxAMDFB 报的尺寸，实际为 27 英寸，不影响使用
- **OpenCore 为 DEBUG 版**（`Misc → Debug → Target = 67`），稳定后可换 RELEASE 或把 Target 调小
- **选单等待**：模板 config 已把 `Timeout` 设为 5 秒（原为 0 = 无限等待）

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
实测结果：注销重登后残影**好很多但仍存在**——这是官方 0.4.0 kext 的极限，进一步改善需要 fork 版 `PhxAccel` 的 dbuf/vbl 补丁。

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
