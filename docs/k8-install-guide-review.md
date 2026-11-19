# K8 装机指南 v3 核对报告

- 核对对象：`k8-install-guide-2026-11-18 (3).md`（294 行，下文行号均指该文件）
- 核对时间：2026-11-18
- 方法：GitHub API / raw 原文 / 直接解析 `config.plist` / 静态阅读 OpCore-Simplify 与 nullarc 源码；能实测的只做 HEAD 与文件比对，不运行任何第三方工具

> **2026-11-18 晚更正**：§4 关于“stock v0.4.0 不能驱动 HDMI、K8 会黑屏”的结论**作废**。v5 的三行日志与 `PhxAMDFB::start()` 的 head 0 路径对得上：head 0 读 `getConsoleInfo()`，接管固件已点亮的输出（K8 的 HDMI），与 eDP 无关；`phxfb2` 是第二个头。详见文末 §9。

---

## 0. 一句话结论

**安装阶段 B 路（Firebat EFI 手工改）可用，但有 6 处会直接卡住、或按字面执行会误删 Windows。A 路（OpCore Simplify）在 K8 上不是"可能失败"，而是必然在兼容性检查处退出——当前版本对 780M 没有任何"继续"的分支。**

**GPU 阶段有一处上游文档与指南正面冲突：stock v0.4.0 不能驱动 HDMI/USB-C 外接显示，而 K8 没有内屏。指南里"#30 迷你主机 26.7.3 + v0.4.0 跑通玻璃"这句必须重新确认证据来源，否则第 0 步会在 K8 上黑屏。**

---

## 1. 核对为真的部分（可以放心，不用改）

| 指南说法 | 核对结果 | 证据 |
|---|---|---|
| macOS Tahoe 26.7.3，build 25G342 | ✅ 正确。2026-11-09 正式发布 | Apple 安全公告页（日文/新西兰）、CNMO、MacObserver 均记 25G342 |
| Tahoe 完整安装器不在 App Store，用 `softwareupdate` 拿 | ✅ 正确 | 多家 2026 年教程一致；OpCore/社区文档同 |
| InstallAssistant.pkg 链接与 18,388,869,343 字节 | ✅ 链接仍有效，`Content-Length` 完全一致 | `curl -sIL` 返回 HTTP 200 + 18,388,869,343 |
| OpCore-Simplify 7,779 star、BSD-3、未归档、支持 High Sierra→Tahoe | ✅ 基本一致（核对时 7,778 star / 741 fork，2026-11-08 最后推送） | GitHub API |
| OpCore 显卡表只到 Vega（Ryzen 1xxx–5xxx、7x30），780M 不在表内 | ✅ 正确，README 原文："AMD APU: The entire Vega Raven ASIC family (Ryzen 1xxx → 5xxx, 7x30 series)" | `README.md` 第 38 行 |
| Firebat EFI 仓库 main 头 `abec07c`（2026-10-19） | ✅ 完全一致：`abec07cea826d1bff96825cdce341c9afe15e4dd` | `git ls-remote` |
| Firebat `config.plist` 57,341 字节 | ✅ 一致 | raw 下载 |
| 27 个 AMD 内核补丁 | ✅ `Kernel → Patch` 正好 27 条 | 解析 config |
| 补丁是"algrey 那一套" | ⚠️ 不全是 algrey：还有 CaseySJ、Visual、Shaneee/Zormeister、laobamac。不影响使用，只是描述不准 | 补丁 Comment |
| 补丁适配 Tahoe | ✅ 有 `26.0+` 专用条目（如 `CaseySJ | probeBusGated | Disable 10 bit tags | 26.0+`） | config |
| Firebat 自带 boot-args 原文 | ✅ 逐字一致 | config |
| `amfi_get_out_of_my_way=1` 词对不上 `install.sh` | ✅ **完全正确**。`install.sh` 用 `case " $args " in *" $a "*)` 做空格分词全词匹配，`=1` 不会命中 `=0x1` | `install/install.sh` 第 37–40 行 |
| `csr-active-config` = 0x87F = `7F080000` | ✅ `fwgAAA==` 解码为 `7F 08 00 00` | config |
| `SecureBootModel = Disabled` | ✅ | config |
| SMBIOS `MacBookPro16,1`、序列号占位 `C02Z00000000` | ✅ 完全一致；MLB 也是占位 `C0200000000000000`，UUID 全 0，ROM 全 0 | config |
| NVRAM → Delete 已有 `boot-args`、`csr-active-config` | ✅ | config |
| 笔记本专用 kext 当前是开着的 | ✅ VoodooI2C（含 4 个插件）、VoodooI2CHID、VoodooPS2Controller（键盘插件）、HoRNDIS、RtWlanU 全为 `Enabled=true` | config |
| `SSDT-USB-Reset.aml` 开着、用途是禁 Firebat 内置摄像头 | ✅ 开着；README 原文确认是禁 Sonix 摄像头 | config + README |
| `AppleALC` / `NootedRed` / `RDNA3Driver` 本来就是关的 | ✅ 三者 `Enabled=false` | config |
| DeviceProperties 有 GPU 音频 `layout-id 7` | ✅ `PciRoot(0x0)/Pci(0x8,0x1)/Pci(0x0,0x6)` | config |
| nullarc v0.4.0 存在、参考机是 Firebat U6C H255 | ✅ `v0.4.0-alpha`，2026-11-09 发布（pre-release）；README 实测环境写的是 macOS 26.6.2 | GitHub Releases + README 第 29 行 |
| `install.sh` 检查 SIP、macOS 26、PhxBringup 已加载 | ✅ 与指南第 0 步的预期一致 | `install.sh` |

---

## 2. 必须改（P0）

### P0-1　U 盘要 32GB，不是 16GB（第 25 行）

Apple 官方页面（`support.apple.com/101578`）：**32GB 才"足够装下任何 macOS 安装器"，16GB 只够"大多数较早版本"**。Tahoe 的 InstallAssistant.pkg 已经是 17.1 GiB，`createinstallmedia` 之后还要更大，16GB 盘做不出来。

**改法**：`U 盘 16GB 以上` → `U 盘 32GB（Tahoe 安装器已超过 16GB）`。

---

### P0-2　A 路不是"可能被拒"，是必然退出（第 82、95、265、274 行）

静态阅读 OpCore-Simplify 当前 main（2026-11-08 推送）：

- `Scripts/compatibility_checker.py:168-183`：AMD 分支只认
  `15D8 / 15DD / 15E7 / 1636 / 1638 / 164C`（Vega APU）以及 `Vega 10` 等少数代号；
  **780M 的 `1900` 不在其中，"Phoenix/Hawk Point" 也不在 `AMDCodenames` 里**（该列表只有 Hainan/Oland/Cape Verde/Pitcairn/Tahiti/Bonaire/Hawaii/Tonga/Fiji 这些老 GCN dGPU）。
- 于是 `max_version = min_version = None`。
- `:232-245`：`gpu_compatibility == (None, None)` → 打印
  **"You cannot install macOS without a supported GPU."** → `exit_program()`。

也就是说：**流程根本走不到第 4 步选版本，更走不到 Build。**指南现在的写法（"780M 显示不支持，别管它，其他绿的就继续"）会让人以为能继续，实际是直接退出。

**改法（二选一）**：

1. **推荐**：把 A 路从"首选"降为"预期不可用"，B 路写成实际主路。第 3 步改成："工具会在兼容性检查处打印 `You cannot install macOS without a supported GPU.` 并退出，这是 780M 不在表内的必然结果；不要在这里耗时间，直接走 3B。"第 95 行的"如果……拒绝构建"改成"一定会拒绝"。
2. **想保留 A 路**：只改一行数据表，把 `1900` 加进 APU 白名单：
   ```python
   # Scripts/compatibility_checker.py，约第 183 行
   elif device_id in ("15D8", "15DD", "15E7", "1636", "1638", "164C", "1900"):
   ```
   这只会放开"能不能继续"的闸门，不会伪造显卡驱动；放开后必须同时做三件事（见 P0-3 与 P1-2）：版本号输 Darwin、**手动取消 NootedRed**、检查生成的 config 没有 `disable-gpu` 属性。

另外，指南第 274 行说"工具每次构建都会拉最新 OpenCore 和 kext"——它拉 AMD 补丁的地址是 `laobamac/AMD_Vanilla` 的 `patches.plist`（`Scripts/gathering_files.py:23`），不是 Dortania 原仓库。这点值得写进附录，因为 laobamac 这条线正是 Tahoe 的补丁来源，可信度反而更高。

---

### P0-3　A 路的版本号是 Darwin 号：Tahoe 26.7.3 = 输入 `25.7.3`（第 83 行）

`OpCore-Simplify.py:151-180` 的版本菜单是按 **Darwin** 内核号列出的（例如 `25. macOS Tahoe 26`），提示原文：

> To select a major version, enter the number (e.g., 19).
> To specify a full version, use the Darwin version format (e.g., 22.4.6).

Tahoe 26 = Darwin 25。输入 `26.7.3` 会被当成 Darwin 26（= macOS 27），超出 `25.99.99` 上限，函数会重新询问，什么也不会发生。

**改法**：`选 26.7.3` → `在版本菜单里输入 25.7.3（Darwin 号；输 25 就是 Tahoe 大版本）。不要输 26.7.3，工具会当成 Darwin 26 拒收。`

---

### P0-4　§6 第 5 步对"方案 B"是误删指令（第 233 行）

原文：

> 安装器里先打开**磁盘工具**，按 §5 把目标盘抹成 APFS + GUID

方案 A（新盘）没问题；**方案 B 是同一块盘上还有 Windows**，按字面"把目标盘抹掉"会把整块 Lexar NM710 抹掉，Windows 一起没了。§5 虽然写的是"新建一个 APFS 分区"，但 §6 这句是整盘动作。

**改法**：拆成两句：

- 方案 A：在磁盘工具里选中**新 NVMe 整盘** → 抹掉 → APFS / GUID。
- 方案 B：**绝对不要抹整盘**。先在 Windows 里用 `diskpart` 把压缩出来的空间建成一个**不格式化**的主分区（或直接留未分配），再在磁盘工具的图形视图里选中那块**空闲空间/新分区** → 分区 → `+` → 格式 APFS；只抹这个新分区。

---

### P0-5　§3B.4 的四个 kext：三个在 config 里根本没有条目；而且自带版本不适配 Tahoe（第 148–155 行）

核对 `Kernel → Add` 全表（33 条）：

| kext | 在 `EFI/OC/Kexts/` 里 | 在 `config.plist` 里 | 实际要做的 |
|---|---|---|---|
| `BlueToolFixup.kext` | ✅ 有 | ✅ 有条目，`Enabled=false` | 把 `Enabled` 改 `true` 即可 |
| `AirportItlwm.kext` | ✅ 有 | ❌ **完全没有条目** | 要**新增**一条 Kernel→Add |
| `IntelBluetoothFirmware.kext` | ✅ 有 | ❌ **完全没有条目** | 要**新增**一条 |
| `IntelBTPatcher.kext` | ✅ 有 | ❌ **完全没有条目** | 要**新增**一条 |

"把它们的 Enabled 改成 true"对后三个是不成立的——没有 Enabled 可改。

更关键的是版本：

- 自带的 `AirportItlwm.kext` 版本是 **2.3.0**，`BuildMachineOSBuild = 19H2026`（10.15 Catalina SDK 构建）。这是 Sonoma 时代的产物，**不是 Tahoe 构建**，在 26.x 上不会工作（机场框架 26 已变）。
- 自带的 `IntelBluetoothFirmware` / `IntelBTPatcher` 是 **2.4.0**（OpenIntelWireless 原版，已停更）。Tahoe 需要 **lshbluesky/IntelBluetoothFirmware 2.5.x** 分支（其 Release 明确写 "Add support for macOS Tahoe 26"，且 2.5.0-d3 修了 26.5 的 BLE 问题）。

**改法**：

1. Wi-Fi：把 `EFI/OC/Kexts/AirportItlwm.kext` **替换成 `laobamac/itlwm` 的 Tahoe Release 构建**（不是 OpenIntelWireless 原仓库，那个已停更）；然后在 Kernel→Add **新增**：
   ```
   BundlePath      AirportItlwm.kext
   ExecutablePath  Contents/MacOS/AirportItlwm
   PlistPath       Contents/Info.plist
   MinKernel       25.0.0
   Enabled         true
   ```
   （放在 Lilu 之后即可，和 VirtualSMC 的先后无所谓。）
   若加载后不认卡/卡住，社区实测还需要 `Kernel → Quirks → DisableIoMapper = true`（AppleVTD 关闭）；这是别人机器上的条件，K8 未验证。
2. 蓝牙：换成 lshbluesky 的 2.5.x（IntelBluetoothFirmware + IntelBTPatcher + 原有 BlueToolFixup），三条都要新增/打开。并且需要：
   - NVRAM → Add（`7C436110-…-FE41995C9F82`）里补 `bluetoothExternalDongleFailed = 00`、`bluetoothInternalControllerInfo = 000000000000000089653A552EFD`（Firebat 的 config 只在 Delete 里有这两个键，Add 里没有）；
   - USB 端口映射（蓝牙是 `8087:0029` USB 设备），没映射前很可能根本不出现——指南 §9 已把 USB 映射排到装机后，那么 **BT 就应该写成"装机后再弄"**。
3. 把这条写清楚：**不要打开自带 2.3.0 的 AirportItlwm**，否则轻则不加载，重则启动卡住。

---

### P0-6　§7 少了"把内置盘做成启动项"这一步（第 250–252 行）

Firebat 的 config 里：

- `Misc → Boot → LauncherOption` = **`Disabled`**
- `EFI/OC/Drivers` 里**没有** `Bootstrap.efi`
- `UEFI → Drivers` 只有 `HfsPlus.efi`、`OpenRuntime.efi`、`ResetNvramEntry.efi`

这意味着：把 EFI 拷到内置盘 ESP 后，固件 NVRAM 里**不会自动出现**指向它的启动项。拔掉 U 盘后能不能进 OpenCore，取决于主板会不会把内置盘 `\EFI\BOOT\BOOTx64.efi` 自动枚举成 "UEFI OS"——[待验证]，不能当成必然。

**改法**（任选，推荐第 1 个）：

1. 在第一次从 U 盘启动**之前**，把 `Misc → Boot → LauncherOption` 改成 **`Full`**。OpenCore 启动时会自己往 BIOS NVRAM 写一条名为 `OpenCore` 的启动项；之后即使拔盘、换启动顺序也能找回来。
2. 拷完 EFI 后在 BIOS 里手动加：Boot → Add Boot Option → 选内置盘 → 路径 `\EFI\BOOT\BOOTx64.efi`，命名 `OpenCore`。
3. 进 macOS 后 `sudo bless --mount /Volumes/EFI --setBoot --file /Volumes/EFI/EFI/BOOT/BOOTx64.efi`（部分主板不认，作为兜底）。

第 251 行"拔掉 U 盘，重启，确认……"应该改成"先按上面建好启动项，再拔盘验证"。

---

## 3. 高优先级补充（P1）

### P1-1　SMBIOS 的 ROM 还是全 0（第 172–177 行）

config 里 `PlatformInfo → Generic → ROM` = `AAAAAAAA`（6 字节全 0）。指南只让填序列号、MLB、UUID，**没提 ROM**。ROM 全 0 会让 iCloud/App Store/iMessage 的风控直接拒绝。

**改法**：加一行——`ROM` 填本机有线网卡的 MAC（6 字节，十六进制），例如 `PlatformInfo → Generic → ROM = <你的 en0 MAC>`。GenSMBIOS 不生成这个。

### P1-2　A 路生成后必须多查三项（第 85–91 行）

即使按 P0-2 放开了闸门，OpCore 还会：

- `Scripts/kext_maestro.py:433-436`：**只要检测到 AMD "Integrated GPU"，就无条件加入 `NootedRed`**。NootedRed 是 Vega 核显驱动，780M（RDNA3）不能用，和 nullarc 也会打架。生成后必须在 kext 列表里把它取消/在 config 里 `Enabled=false`。
- 可能给 780M 加 `disable-gpu` 设备属性（README 说它会"禁用不支持/未使用的 PCI 设备"）。必须确认 `DeviceProperties` 里**没有**针对 `1002:1900` 的 `disable-gpu`。
- boot-args 里若出现 `-amd_no_dgpu_accel` / `-radvesa`（工具对"AMD dGPU + OCLP"会加），第 0 步前要删掉。

### P1-3　EFI 从 Windows 到 Mac 的搬运没写（第 33–36 行）

A 路在 K8 的 Windows 上生成 EFI，但 §2 是在 Mac 上做 U 盘。中间少一步：怎么把 `Results/EFI` 弄到 Mac。

**改法**：加一句——用一个 FAT32 小 U 盘/移动硬盘把 `EFI` 文件夹拷到 Mac；或者干脆在 Windows 上用 OpCore 官方 README 推荐的 **UnPlugged** 直接做安装盘（`corpnewt/UnPlugged`），整条 A 路可以完全不用 Mac。

### P1-4　`phxfb-boot` 写进 NVRAM → Add 有反作用（第 90 行）

`phxboot.sh` 的失效保护是：M2 失败时 `nvram -d phxfb-boot` 再重启，回到固件帧缓冲（`m2_fail()`，`tools/phxctl/phxboot.sh:31-35`）。如果 OpenCore 的 `NVRAM → Add` 里长期放着 `phxfb-boot = 1`，**每次开机都会重新武装**，`m2_fail` 删了也白删 → 可能变成黑屏重启循环。

**改法**：兜底只在确认 NVRAM 完全不持久时用，并且写成"一次性"：先 `sudo nvram phxfb-boot=1; sudo nvram IONVRAM-FORCESYNCNOW-PROPERTY=phxfb`（和 phxboot 自己一样）；实在不行才用 Add，且**成功进桌面一次后就从 config 里删掉**。

### P1-5　有线网卡驱动选错代（第 36、159 行）

指南让下 `LucyRTL8125Ethernet.kext`。Mieze 已经出了继任者 **RTL812xLucy**，README 原文：

> Support for AppleVTD (Tahoe included), but also works without it.
> The driver has been successfully tested with Tahoe, Sequoia and Monterey…
> RTL812xLucy ist als Ersatz für LucyRTL8125Ethernet gedacht, muss dieser Treiber entfernt, bzw. in OC deaktiviert werden.

即：**Tahoe 用 `Mieze/RTL812xLucy`，并且要把 LucyRTL8125Ethernet 移除/禁用**。K8 的两个 RTL8125 正好在支持列表（8125A/B/BP/CP/D）。

**改法**：§1 与 §3B.5 把 `LucyRTL8125Ethernet` 换成 `RTL812xLucy`（GitHub `Mieze/RTL812xLucy`，最新 Release），其余步骤不变。

### P1-6　OpenCore 版本可能偏旧

`EFI/OC/OpenCore.efi` 二进制里能读到 `1.0.0` 字样（不能 100% 断定就是 1.0.0，但 README 徽章也只写 `1.0.x`）。OC 1.0.0 是 2024 年的版本，boot Tahoe 26 建议用当前最新 1.0.x。

**改法**：B 路开始前，用 OpenCore 官方 `OpenCorePkg` 的最新 Release 覆盖 `BOOTx64.efi` / `OpenCore.efi` / `Drivers/OpenRuntime.efi`（kext 不动），或至少用 `ocvalidate` 确认现有 config 能被新版接受。这条标 [待验证]，不影响先试。

### P1-7　NullEthernet + SSDT-RMNE 可能抢占 en0

K8 有真实 RTL8125，但 Firebat EFI 同时开了 `NullEthernet.kext` + `SSDT-RMNE.aml`（给没有内置网卡的机器造一个假 en0）。两者同时存在时，真网卡可能变成 en1，iServices 用的 MAC 反而是假的。

**改法**：先不动；如果装完 iCloud/App Store 异常，先把 `NullEthernet` 和 `SSDT-RMNE` 关掉再看。

### P1-8　其它小项

- **双 M.2**（第 28 行）：硬件报告只记录了一块 Lexar NM710，没有确认第二个 M.2 槽。装机前开机箱/看手册确认，[待验证]。
- **时间**：macOS 把 RTC 当 UTC，Windows 当本地时间，双系统会差 8 小时。§9 可加一条：装完在 Windows 里把 `RealTimeIsUniversal=1`，或在 macOS 里 `sudo systemctl`……（macOS 无此命令）→ 用 Windows 注册表方案。
- **BIOS**：Firebat config 已经是 `ResizeAppleGpuBars=-1`、`ResizeGpuBars=-1`，所以 BIOS 里 ReBAR 开着也能启动；指南只要保证 **Above 4G Decoding = Enabled** 即可，不必额外关 ReBAR。若启动卡在 PCI，再回头关 ReBAR。
- **§8 表格**里"OpCore 里 780M 显示不支持……其他项是绿的就继续构建"这一行按 P0-2 重写。

---

## 4. 重点：GPU 阶段与 "#30 迷你主机" 证据冲突

### 4.1 上游文档的硬事实

`luuumi-dev/nullarc` README（main，2026-11-18）第 44 行：

> **Not yet** | WIP | **External display PHYs (HDMI / Type-C DP via DCN 3.1.4 in progress)**, hardware VCN 4.0.2 video decode.

`KNOWN_ISSUES.md` B-11：

> **Симптомы**: Внешние мониторы не выводят изображение.
> **Причина**: `PhxAMDFB` настроен только на встроенную eDP-панель. Дополнительные DCN-пайплайны и PHY не сконфигурированы.

即：**v0.4.0 / main 的 PhxAMDFB 只会点 eDP 内屏，HDMI 和 USB-C DP 都不输出。** 参考机是 Firebat U6C 笔记本（2560×1600 eDP）。

### 4.2 HDMI 头是哪来的

`izerok/nullarc` 的 `hdmi-fb2` 分支（不是 main）在 **2026-11-15** 才加入：

- `e15ab67` PhxAMDFB: a second head for the HDMI port (**boot-arg `phxfb2=1`**)
- `024a5ac` display power state parameter for the panel, display sleep for the HDMI head
- `50db1e1` debug.phxfb1_hw also reports the USB-C port (DP sink present, DIG2 on)

而 `phxfb.cpp:71-74` 的注释写得很清楚：

> The kext writes no display register for it either: **a user-space tool lights the port** (DMUB firmware, pixel clock, OTG1, DIG1, HUBP1 scanning this head's buffer) and then tells the framebuffer the mode and the monitor's EDID through `debug.phxfb1_config`.

也就是说，K8 要走外接显示，需要同时满足：**hdmi-fb2 构建 + `phxfb2=1` + 一个用户态点屏工具**。三者缺一，HDMI 不亮。

### 4.3 因此 "#30 26.7.3 + v0.4.0 玻璃" 需要重新确认

- v0.4.0-alpha 发布于 **2026-11-09**；
- HDMI 第二头的提交是 **2026-11-15**，晚于 release；
- stock v0.4.0 的 PhxAMDFB 只认 eDP；UM790 Pro 没有 eDP。

**结论**：如果 #30 真的是一台 UM790 Pro、真的看到了 GPU 合成的玻璃效果，那它用的**一定不是 stock v0.4.0**，而是 `hdmi-fb2` 一类的构建（并且开了 `phxfb2=1`、跑了点屏工具）。指南第 4 行把它写成 "26.7.3 + v0.4.0"，会让人以为拿官方 v0.4.0 就能复现——在 K8 上大概率是装完 PhxAccel 后**黑屏**。

**改法**：

1. 第 4 行改成"26.7.3 + nullarc（**hdmi-fb2 构建 / `phxfb2=1`**）"，并在括号里注明"不是官方 v0.4.0 release"。
2. §0 的"GPU 不在本指南"保持，但加一句警告：K8 只有 HDMI/DP，没有 eDP；官方 v0.4.0 的加速输出只针对内屏，装 PhxAccel 前必须准备好回滚（保留 U 盘、知道 `phxboot-off` 开关、或先用 `phxfb2` 不打开的方式启动）。
3. 第 16 行"已编好的两个 kext……SHA `0b4b6c59…b1fd`"：**先确认这两个 kext 是从哪条分支编的**。如果是 v0.4.0 release 的 `PhxBringup.kext-0.4.0-alpha.zip`（24.3 KB）→ 没有 HDMI 头；如果是 hdmi-fb2 的自编构建 → 才可能点 HDMI。SHA 对不上就说明拿错了。
4. 顺带：GitHub Release 的资产**不会过期**（`nullarc-0.4.0-alpha.zip` 53.2 MB、`PhxBringup.kext-0.4.0-alpha.zip` 24.3 KB）。指南说"2026-11-20 过期"的应该是某个外部分享链接——那种链接过期后，官方 release 仍然可下，不用慌；但也正因如此，外部链接里的 kext 来源最需要核对。

---

## 5. 可直接照抄的改法清单（按行号）

| 行 | 现在 | 改成 |
|---|---|---|
| 25 | `U 盘 16GB 以上` | `U 盘 32GB（Tahoe 安装器已超 16GB）` |
| 4 | `26.7.3 + v0.4.0` | `26.7.3 + nullarc hdmi-fb2 构建（phxfb2=1），不是官方 v0.4.0 release` |
| 82 | `780M 会显示不支持，别管它……其他项是绿的就继续` | `工具会直接打印 "You cannot install macOS without a supported GPU." 并退出；这是 780M 不在表内的必然结果，直接走 3B` |
| 83 | `选 26.7.3，也就是它默认的最新版` | `输入 Darwin 号 25.7.3（Tahoe 26 = Darwin 25）；输 26.7.3 会被当成 macOS 27 拒收` |
| 90 | `按清单 §6 的 C 路把 phxfb-boot 写进 NVRAM → Add` | `优先用 sudo nvram phxfb-boot=1 + IONVRAM-FORCESYNCNOW；不要长期放在 NVRAM→Add（会绕过 phxboot 的 m2_fail 自救，可能黑屏重启循环）` |
| 95 | `如果工具因为 780M 拒绝构建` | `工具必然拒绝（见上）；除非按附录 A 改一行白名单，否则直接走 3B` |
| 148–155 | `把它们的 Enabled 改成 true` | `BlueToolFixup 只需打开；AirportItlwm / IntelBluetoothFirmware / IntelBTPatcher 在 config 里没有条目，要新增；且自带 AirportItlwm 2.3.0 与 IntelBT 2.4.0 不适配 Tahoe，先换成 laobamac / lshbluesky 的 Tahoe 构建` |
| 159 | `LucyRTL8125Ethernet.kext` | `RTL812xLucy.kext（Mieze，Tahoe 实测；并禁用 LucyRTL8125Ethernet）` |
| 172–177 | 只填序列号/MLB/UUID | 加 `ROM`（6 字节 MAC，不能全 0） |
| 233 | `把目标盘抹成 APFS + GUID` | `方案 A 抹新盘；方案 B 绝不抹整盘，只抹新建的空闲分区` |
| 250–252 | 直接拔 U 盘验证 | 先 `LauncherOption=Full` 或 BIOS 手动加启动项，再拔盘 |
| 265 | `其他项是绿的就继续构建` | 按 P0-2 改写 |

---

## 6. 附录 A：如果一定要走 A 路（可选）

1. 解压 OpCore-Simplify。
2. 打开 `Scripts/compatibility_checker.py`，在 AMD 分支的 APU 元组里加 `"1900"`（约第 183 行，见 P0-2）。
3. 正常跑 `OpCore-Simplify.bat` → 导出硬件报告 → 兼容性检查应能过（780M 会显示成兼容的 Vega APU）。
4. 版本菜单输入 **`25.7.3`**。
5. 在 kext 配置菜单里**取消 `NootedRed`**；若工具加了 `WhateverGreen` 可以留（无害）。
6. 生成后核对：`Kernel → Add` 无 NootedRed；`DeviceProperties` 无 `disable-gpu`；boot-args 无 `-amd_no_dgpu_accel`/`-radvesa`；Wi-Fi kext 是 `AirportItlwm-laobamac`（工具自带这个数据集，min Darwin 24.2、max 25.99）。
7. SMBIOS 是工具新生成的（序列号/MLB/UUID 有效），但 `ROM` 仍需自己填 MAC。

风险：这是改工具的数据表，不是官方支持路径；工具后续更新可能改变行为。如果不想碰源码，就走 B。

---

## 7. 未能核实 / 需要你确认的

| 项 | 状态 |
|---|---|
| `0b4b6c59…b1fd` 那两个预编译 kext 本体 | 未拿到文件，无法比对 SHA；**请确认分支来源（v0.4.0 还是 hdmi-fb2）** |
| #30 那台 UM790 Pro 的具体构建、是否 `phxfb2=1`、用什么工具点 HDMI | 无法从公开仓库确认；与 v0.4.0 文档冲突，需你回忆/翻记录 |
| K8 的第二个 M.2 槽 | 硬件报告未记录 |
| BIOS 里 80W 性能模式、启动菜单键 F7/F11/Del | 未实测 |
| K8 上 RTL812xLucy / laobamac Wi-Fi / Intel BT 的实际表现 | 未实测，只能按上游与社区报告标注 |
| OpenCore 具体版本 | 二进制有 `1.0.0` 字样，无法 100% 断定 |

---

## 8. 证据索引

- OpCore-Simplify：`github.com/lzhoang2801/OpCore-Simplify`，main @ 2026-11-08；关键文件
  `Scripts/compatibility_checker.py:168-245`、`OpCore-Simplify.py:140-180`、`Scripts/kext_maestro.py:433-436`、`Scripts/gathering_files.py:23`、`Scripts/datasets/kext_data.py:131-145,229-240`
- Firebat EFI：`github.com/luuumi-dev/firebat-u6c-hackintosh-efi`，main `abec07cea826d1bff96825cdce341c9afe15e4dd`；`EFI/OC/config.plist`（57,341 B）
- nullarc：`github.com/luuumi-dev/nullarc`，`v0.4.0-alpha`（2026-11-09）、README 第 29/44 行、`KNOWN_ISSUES.md` B-11、`install/install.sh:37-40`、`tools/phxctl/phxboot.sh:31-35,94`
- HDMI 分支：`github.com/izerok/nullarc` `hdmi-fb2`，`e15ab67` / `024a5ac` / `50db1e1`（2026-11-15），`src/phxaccel/accel/phxfb.cpp:71-74`
- Apple：`support.apple.com/101578`（32GB U 盘）；`support.apple.com/ja-jp/149719`（26.7.3，2026-11-09）
- InstallAssistant：`swcdn.apple.com/.../InstallAssistant.pkg` → HTTP 200，18,388,869,343 B
- Wi-Fi：`github.com/laobamac/itlwm`（Tahoe 26.x AirportItlwm）；社区实测需 `DisableIoMapper=true`
- 蓝牙：`github.com/lshbluesky/IntelBluetoothFirmware`（2.5.x，Tahoe）
- 有线：`github.com/Mieze/RTL812xLucy`（Tahoe 实测，替代 LucyRTL8125Ethernet）

---

## 9. 2026-11-18 晚更正：head 0 接管的是固件点亮的输出，不是 eDP

v5 提供的三行日志与代码逐字对应：

| 日志 | 代码 | 含义 |
|---|---|---|
| `taken` | `phxfb.cpp:834` `NVRAM phxfb-boot taken (removed and committed)` | kext 消费 arming 变量并强制提交。`takeBootArming()`（`:819-839`）只检查变量**存在**，所以 Data `MDE=`（"01"）与字符串 "1" 等价 |
| `console: 1920x1080 row 7680 depth 32` | `:944` | `getConsoleInfo(&v)` 读到固件控制台 1080p/32bpp，head 0 直接采用这个模式与 surface |
| `panel published … vbl timer running` | `:989` | head 0 发布，VBL 定时器运行 |

`start()`（`:903-912`）只要求：固件控制台存在、32bpp、宽高/行宽合法、surface 能在 GPU 的某个 BAR 里找到；它**不区分 eDP / HDMI**。所以在 K8 上，BIOS 点亮的 HDMI 控制台就是 head 0 的“panel”，单显示器**不需要** `phxfb2`。README 的 “External display PHYs WIP” 与 B-11 说的是**额外的** DCN 管线/PHY（第二、第三个显示器），不是固件已经点亮的那个。日志里的 1920×1080 也说明它不是 Firebat 的 2560×1600 内屏，而是外接显示器——与 UM790 Pro 的说法一致。

仍然成立的部分：

- `phxfb2=1` 是第二个头（`phxfb.cpp:71-74`），需要用户态工具点屏（fork 的 `tools/` 里没找到这个工具）；K8 单显示器用不到，**不要加**。
- 这组证据证明的是“PhxAMDFB 接管了显示”，**不证明 Metal**。玻璃效果在软件合成下也能画出来；要确认 GPU 合成，看 `debug.nvaccel_iop_ok`、`mtlprobe` 或拖动窗口帧率。
- K8 特有风险：如果固件控制台不是 32bpp，或 surface 不在 GPU BAR 里，日志会打 `no usable boot console` / `the console surface is in none of the GPU's BARs`，PhxAMDFB 会留在固件帧缓冲。第 0 步先搜这两句。
- 逃生开关：boot-arg **`-phxfboff`**（`phxfb.cpp:894`，在 arming 之前判断），比改 `phxfb2` 直接。
- `phxfb-boot` 放进 NVRAM → Add 的副作用：kext 每次启动会删掉并强制提交；Add 在变量不存在时写入，所以每次启动都会重新武装——GPU 起不来时可能反复重试。成功进桌面后可以移除 Add 项。
