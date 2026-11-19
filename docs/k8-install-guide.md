# K8 装机指南 — 在 KTHIN K8 上装 macOS Tahoe 26.6.2（原计划 26.7.3）

> **2026-11-19 实测更新（本仓库）**
> - 实际安装版本为 **26.6.2（25G83）**，不是本文标题里的 26.7.3；安装器、EFI 与实测步骤见根 README。
> - 安装阶段必须新增 **XhciNudge.kext + `xhcinudge=4`**：AMD Zen 4（Hawk Point）内核 USB 丢中断会让 `validate_root_image` 卡几十分钟，没有它装不上。
> - GPU 阶段：官方 nullarc v0.4.0 在 780M（`1002:1900`）上接管会卡死在内核早期；改用 `hdmi-fb2` fork 的两个 kext 后接管成功（详见根 README §3.3）。
> - **USB-C/DP 输出无信号**：0.4.0 只驱动 HDMI head 0（`0x53e4=0`、无 `phxfb1_hw`）；USB-C 口在 macOS 下仅数据可用。
> - **已知崩溃**：切换壁纸/截屏/录屏/共享屏幕会触发 nullarc 的窗口捕获缺陷（`MetalBacking::AbortWithTextureInfo`），WindowServer SIGABRT 后自动重启（黑屏几秒）。日常用静态壁纸，避免这些操作。
> - §3.8 关于 PhxBringup 的说法写严了：把它放进 EFI 并由 OpenCore 加载是正确的做法——`install.sh` 的前置检查要求它已加载。


- 机器：KTHIN 科偲 K8，Ryzen 7 255 / H255，Radeon 780M（1002:1900），BIOS 2.09
- 目标系统：macOS Tahoe **26.7.3**（build 25G342）——钉这个版本，因为 #30 那台迷你主机（UM790 Pro，7940HS）在 26.7.3 上跑通了玻璃效果，机制已确认：成功启动的三行日志（`taken` / `console: 1920x1080 row 7680 depth 32` / `panel published … vbl timer running`）与 config.plist 原文（`phxfb-boot` = Data `MDE=`，即 "01"，在 7C436110 下；NVRAM → Delete 里没有它，每次启动都重新注入）都已贴出，重启后玻璃还在、动态壁纸也回来了；nullarc 文档自己写的实测版是 26.6.2（25G83），那个范围也通
- EFI 只有一条路：Firebat U6C 的 EFI（同款 H 255 APU，按 K8 硬件改过）。OpCore Simplify 本来是 A 路，但 2026-11-18 晚核查源码后确认它在 K8 上直接退出（兼容性检查不认 780M），A 路已删，见 §3 开头
- 写于 2026-11-18；2026-11-18 晚按用户提醒改版：EFI 主路改为 OpCore Simplify 生成，Firebat EFI 退为备用；2026-11-18 晚按 #30 验证与用户要求再改版：目标系统 26.6.2 → 26.7.3（#30 对齐），补 NVRAM 持久性兜底；2026-11-18 晚按本地 AI 评审逐项核查后大改：OpCore Simplify A 路删除（源码核实会直接退出）、U 盘改 32GB、§5/§6 抹盘指令修正（方案 B 不得整盘抹）、§3.4 三 kext 改为新增条目并换 Tahoe fork 版、网卡换 RTL812xLucy、§7 补启动项、#30 表述降级；EFI 底座核对版本：`luuumi-dev/firebat-u6c-hackintosh-efi`，main 头 `abec07c`（2026-10-19）

## §0 这份指南是什么

**目标**：在 K8 上装好 macOS Tahoe，能进桌面。**只到进桌面为止。**

**GPU 加速不在本指南里**。显卡驱动是下一步，按第 0 步清单走（`minipc-step0-checklist-2026-11-15.md`，已发过你）。装系统这个阶段，显示走固件帧缓冲——没有加速也能显示，这点有实证：nullarc #30 那台迷你主机装完系统、没装任何驱动，显示器就是亮的。

**结论先说**：EFI 只有一条路——Firebat U6C 的 EFI，同款 H 255 APU，改几处就能用（关笔记本专用项、加 Wi-Fi/蓝牙、补有线网卡驱动、**改两个启动参数**、注册启动项）。**OpCore Simplify 不能用**：我读了它的源码，`compatibility_checker.py` 的 AMD 分支不认 780M（Phoenix/1900），会打印 `You cannot install macOS without a supported GPU.` 然后直接退出，根本到不了选版本那步——不是"显示不支持但能继续"，是死路。所以别下它，直接走 §3 的 Firebat 手工改。

**提醒**：已编好的两个 kext（PhxBringup + PhxAccel，hdmi-fb2 @ 50db1e1）的下载链接 **2026-11-20 过期**（是我发你的那个外链过期，不是 GitHub 上的构建产物过期），还没下载的话先下载并对一下 SHA（`0b4b6c59…b1fd`），别等过期了再重编。

**可信度分级**：[已核] = 我从 EFI 仓库原文或你的硬件报告里核对过；[通用] = AMD 黑苹果通用做法，Dortania 指南同款；[待验证] = K8 上还没人试过，按实机结果为准。

---

## §1 准备清单

| 东西 | 说明 |
|---|---|
| U 盘 32GB | 做 macOS 安装盘，装完就没用了。26.7.3 安装器 17.1 GiB，标称 16GB 的盘只有约 14.9 GiB 可用，装不下（Apple 官方也写 32GB 才够任何安装器） |
| 一台 Mac | 下载安装器、做 U 盘、改 EFI 配置 |
| 第二块 NVMe（推荐） | K8 有双 M.2 槽；新盘装 macOS，Windows 完全不动。没有新盘就看 §5 方案 B |
| Windows 备份 | 动盘之前先备份。你的报告里还没确认备份——**装之前先做** |
| 显示器接 HDMI | 开机前就把 HDMI 线插好（你的显示器已确认 HDMI 4K160 通） |
| 有线键盘鼠标 | 安装过程用 USB 的就行 |

**下载这样**（都在 Mac 上操作）：

1. Firebat EFI：https://github.com/luuumi-dev/firebat-u6c-hackintosh-efi —— 点绿色的 Code → Download ZIP，解压后得到 `EFI` 文件夹。
2. 有线网卡驱动 `RTL812xLucy.kext`：GitHub `Mieze/RTL812xLucy`，最新 Release v1.1.3（`LucyRTL8125Ethernet` 已于 2026-11-08 宣布 EOL，别下那个）。
3. Wi-Fi/蓝牙 fork 版：`laobamac/itlwm` 最新 Release（v2.4.0-alpha，取其中的 `AirportItlwm.kext`）和 `lshbluesky/IntelBluetoothFirmware` 最新 Release（v2.5.3，取其中的 `IntelBluetoothFirmware.kext` 与 `IntelBTPatcher.kext`）。注意：上游 OpenIntelWireless 的 itlwm 停在 v2.3.0（只声明支持到 Sonoma）、IntelBluetoothFirmware 停在 v2.4.0，Tahoe 支持只在 fork 里有，别下上游的。
4. `GenSMBIOS`（GitHub `corpnewt/GenSMBIOS`，最新 Release）。

---

## §2 在 Mac 上做安装 U 盘

**不要从 App Store 下安装器。** 苹果现在不把 Tahoe 完整安装器放 App Store，用命令行拿：

```bash
# 先看有哪些完整安装器可选
softwareupdate --list-full-installers

# 下 26.7.3（下完在 /Applications 里出现“安装 macOS Tahoe.app”）
softwareupdate --fetch-full-installer --full-installer-version 26.7.3
```

**做启动 U 盘**（把 `MyUSB` 换成你的 U 盘卷名；这步会抹掉 U 盘）：

```bash
sudo /Applications/Install\ macOS\ Tahoe.app/Contents/Resources/createinstallmedia --volume /Volumes/MyUSB
```

**如果 `softwareupdate` 那条命令拿不到 26.7.3**，直接下 26.7.3 的 InstallAssistant.pkg，苹果官方链接：https://swcdn.apple.com/content/downloads/02/21/142-54128-A_OWMUBNYT6S/xlorkwl2msc8dgn34ayr8dk2jm8xka68z3/InstallAssistant.pkg（18,388,869,343 字节，约 17.1 GiB）——双击运行，安装器同样进 /Applications，之后 `createinstallmedia` 不变。前面下过的 26.6.2 的包和 26.7.2 的包（18,384,624,402 字节）都不用了，改下这个。

**把 EFI 放进 U 盘**：

1. 挂载 U 盘的 EFI 分区。终端里跑 `diskutil list` 找到 U 盘的 EFI 分区（一般是 `diskN s1`，类型 EFI），然后：

   ```bash
   sudo diskutil mount diskNs1
   ```

2. 把 §3 得到的 `EFI` 文件夹整个复制到 U 盘 EFI 分区的根目录。完成后分区里应该只有 `EFI` 一个文件夹，里面是 `BOOT` 和 `OC`。

---

## §3 EFI：Firebat EFI 手工改 [已核]

只有这一条路。**OpCore Simplify 已删**：2026-11-18 晚读了它的源码（main 头 `9aed67e`），`Scripts/compatibility_checker.py:148-182` 的 AMD 分支不认 780M（Phoenix/1900），`232-244` 行打印 `You cannot install macOS without a supported GPU.` 后 `sys.exit(0)` 硬退出，而且这步在选版本之前——不是"显示不支持但能继续"，是根本走不到选版本。它的版本菜单确是 Darwin 号（Tahoe 对应 25.7.3），但对 K8 没意义。别下它。

Firebat 是笔记本，K8 是迷你主机，有些东西要动。**改配置用 ProperTree**（Mac 上免费的 plist 编辑器），**不要用文本编辑器直接改 config.plist**，格式一坏 OpenCore 就不认。

另注意一项（第 0 步时再说）：NVRAM 持久性兜底（#30 实证）——第 0 步的握手变量 `phxfb-boot` 是靠 `nvram` 命令或 `phxboot.sh` 写的，有些机器的 OpenCore 不同步 NVRAM，重启就丢，表现为启动日志一直是 `not set`。如果第 0 步走到那一步，按清单 §6 的 C 路把 `phxfb-boot` 写进 config.plist 的 NVRAM → Add；现在不用提前加。

### 3.1 留着不动的 [已核]

- 27 个 AMD 内核补丁（algrey 的那套），全开着，别碰
- `Lilu.kext`、`AMFIPass.kext`、`VirtualSMC.kext`、`AppleMCEReporterDisabler.kext`、`ForgedInvariant.kext`、`RestrictEvents.kext`、`NullEthernet.kext`
- SSDT：`SSDT-EC.aml`、`SSDT-CPUR.aml`、`SSDT-USBX.aml`、`SSDT-RMNE.aml`
- `csr-active-config`：`0x87F`（就是第 0 步要的 `7F080000`，已经对上了，别改）
- `SecureBootModel`：`Disabled`（已经是关的，别改）
- DeviceProperties 里给 GPU 音频的 `layout-id 7`，留着
- NVRAM → Delete 里已经有 `boot-args` 和 `csr-active-config`，留着（新值要生效就靠它）

### 3.2 必须改的：两个启动参数 [已核]

Firebat 自带的 boot-args 是：

```text
-v keepsyms=1 debug=0x100 msgbuf=1048576 revpatch=cpuname,sbvmm ipc_control_port_options=0 amfi=0x80 amfi_get_out_of_my_way=1 -amfipassbeta
```

有两处要改，改完应该是：

```text
-v keepsyms=1 debug=0x100 msgbuf=1048576 revpatch=cpuname,sbvmm ipc_control_port_options=0 amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta phxdebug=1
```

| 改动 | 为什么 |
|---|---|
| `amfi_get_out_of_my_way=1` → `=0x1` | 第 0 步的 `install.sh` 是按**空格分隔的完整词**匹配的，它只认 `amfi_get_out_of_my_way=0x1` 这个词。Firebat 写的 `=1` 数值上一样，但词对不上，`--check` 会直接报缺失。我读过 `install.sh` 的检查代码确认过这一点 |
| 末尾加上 `phxdebug=1` | `install.sh` 检查的三项之一，Firebat 自带的 boot-args 里没有。现在加上，省得第 0 步时再回来改 |

### 3.3 关掉的：笔记本专用的东西 [已核]

在 config.plist → Kernel → Add 里，把这些的 Enabled 改成 `false`：

| 关掉 | 原因 |
|---|---|
| `VoodooI2C.kext` 及其全部插件 | 触摸板，K8 没有 |
| `VoodooI2CHID.kext` | 同上 |
| `VoodooPS2Controller.kext` 及其插件 | PS/2 键盘，K8 用 USB 键盘 |
| `HoRNDIS.kext` | 安卓 USB 共享网络，Firebat 的东西 |
| `RtWlanU.kext` | Firebat 的 USB 无线网卡，K8 没有 |

SSDT 里把 `SSDT-USB-Reset.aml` 也关掉（ACPI → Add 里 Enabled 改 `false`）：它是用来禁 Firebat 内置摄像头的，K8 没有这个摄像头。`SSDT-ALS0.aml`（光线传感器）可以留着，无害。

### 3.4 Wi-Fi / 蓝牙：三个要新增条目，一个改开关 [已核]

**先纠正上一版的一个错误**：上一版写"把这四个的 Enabled 改成 `true`"，但 config.plist 里只有 `BlueToolFixup.kext` 有条目（Enabled=False），`AirportItlwm`、`IntelBluetoothFirmware`、`IntelBTPatcher` 在 Kernel → Add 里根本没有条目——没条目就无从改起，必须**新增条目**。顺序都排在 `Lilu.kext` 之后、`VirtualSMC.kext` 之前。

| kext | 怎么做 | 版本（重要） |
|---|---|---|
| `BlueToolFixup.kext` | 已有条目，把 Enabled 改成 `true` | 自带 2.7.3，可用 |
| `AirportItlwm.kext` | **新增** Kernel → Add 条目，Enabled=`true`；先把 §1 下的 fork 版拷进 `EFI/OC/Kexts/`（替换自带版） | 自带 2.3.0 是上游最新，但上游只声明支持到 Sonoma；**必须换 `laobamac/itlwm` 的 v2.4.0-alpha** |
| `IntelBluetoothFirmware.kext` | **新增** Kernel → Add 条目，Enabled=`true`；先把 §1 下的 fork 版拷进 `EFI/OC/Kexts/`（替换自带版） | 自带 2.4.0 是上游最新，但上游无 Tahoe 声明；**必须换 `lshbluesky/IntelBluetoothFirmware` 的 v2.5.3**（简介明确写 including Tahoe 26） |
| `IntelBTPatcher.kext` | **新增** Kernel → Add 条目，Enabled=`true`；先把 §1 下的 fork 版拷进 `EFI/OC/Kexts/`（替换自带版） | 与上一行同一份 fork（v2.5.3） |

注意：上游 OpenIntelWireless 的 itlwm 停在 v2.3.0、IntelBluetoothFirmware 停在 v2.4.0，Tahoe 支持只在 fork 里有——"去 GitHub 下最新的"如果下的是上游，等于原地踏步。

### 3.5 加上的：有线网卡驱动 [已核]

把 §1 下载的 `RTL812xLucy.kext`（v1.1.3）复制到 `EFI/OC/Kexts/` 里，然后在 config.plist → Kernel → Add 里加一条，Enabled 设 `true`。K8 的两个 2.5G 口都是 RTL8125，就靠它。别用 `LucyRTL8125Ethernet.kext`——作者已于 2026-11-08 宣布它 EOL，继任者就是 RTL812xLucy（超集，Tahoe 已实测）。

### 3.6 序列号：必须重新生成 [已核]

Firebat 的 config.plist 里 SMBIOS 是 `MacBookPro16,1`，但序列号是占位符（`C02Z00000000` 这种）。**占位序列号不能用**，进系统后 Apple ID 相关服务会出问题。

用 GenSMBIOS 生成一套新的：

```bash
# 在 GenSMBIOS 目录里跑，按提示选 MacBookPro16,1，生成 3 个
python3 GenSMBIOS.py
```

把生成的三样填进 config.plist → PlatformInfo → Generic：

- `SystemSerialNumber`（序列号）
- `MLB`（主板序列号）
- `SystemUUID`（新生成一个 UUID，终端跑 `uuidgen` 就行）

型号保持 `MacBookPro16,1` 不变。

### 3.7 音频：先不动 [已核]

- `AppleALC.kext` 在 Firebat 的 config 里本来就是关着的，**保持关闭**。K8 的声卡是 Conexant SN6140，AppleALC 没有对应的 layout，打开也没用。
- 板载音频是装机后再处理的事（备选是 VoodooHDA），不影响安装。
- GPU 的 HDMI/DP 音频（`1002:AA01`）要等 GPU 驱动装好后再看，也不在本指南里。

### 3.8 千万别做的事

- **不要往 EFI 里加 `PhxBringup.kext` / `PhxAccel.kext`**。Firebat 的 EFI 里本来就没有这两个，GPU 驱动是装完系统后用 `install.sh` 装的（第 0 步清单里有），不是放 EFI 里的。第 0 步还会往 EFI 里加 `PhxBringup.kext` 和 `SSDT-ALIB.aml`，那是清单 §4 的事，现在不用加。
- **不要开 `NootedRed.kext`**。那是给 Vega 核显（5800H 那种）用的，K8 的 780M 用不上，config 里本来就是关着的。
- **不要开 `RDNA3Driver.kext`**。那是没写完的脚手架，config 里本来就是关着的。

---

## §4 K8 的 BIOS 设置

开机按 **Del** 进 BIOS（启动菜单一般是 F7 或 F11，以屏幕提示为准 [待验证]）。找到下面这些项，按表里改；菜单名字可能和写的略有出入，以意思为准：

| 设置 | 改成 | 说明 |
|---|---|---|
| CSM / Compatibility Support Module | Disabled（关） | [通用] |
| Secure Boot | Disabled（关） | [通用] 不关 OpenCore 起不来 |
| Above 4G Decoding | Enabled（开） | [通用] AMD 黑苹果必开 |
| Fast Boot | Disabled（关） | [通用] |
| XHCI Handoff | Enabled（开） | [通用] |
| Serial / COM Port | Disabled（关） | [通用] 有就关 |
| 性能模式 | 保持 80W | [已核] 你的 H 255 是 80W 性能模式，别动 |
| TPM / fTPM | 不动 | [通用] 不用特意开关 |

改完保存退出（一般是 F10）。

---

## §5 磁盘方案

### 方案 A（推荐）：第二块 NVMe 装 macOS

K8 有双 M.2 槽，买块 NVMe 插进空槽，macOS 全装在新盘上。**Windows 一个字节都不动**，最干净。安装时在磁盘工具里只抹新盘就行。

### 方案 B：同一块盘双系统

没有新盘就用这个。先在 Windows 里把 C 盘压缩出至少 100GB 空闲（你的 C 盘现在剩约 290GB，够分），然后在 macOS 安装器的磁盘工具里，把空出来的空间新建一个 APFS 分区装 macOS。

**抹盘格式（只对方案 A）**：方案 A（全新 NVMe）才在磁盘工具里选目标盘 → 抹掉 → 格式 **APFS**、方案 **GUID 分区图**。**方案 B 千万别这么干**：对整块盘点"抹掉"会清掉全盘所有分区，Windows 一起没。方案 B 只在空出来的空间上**新建一个 APFS 分区**（上面那段的做法），连抹都不需要。

---

## §6 安装流程

1. **HDMI 线开机前就插好**，显示器开着。安装 U 盘插 K8 背面 USB 口（别用前置扩展口）。
2. 开机，按启动菜单键选 U 盘的 **UEFI** 启动项（名字里带 UEFI 的那个，别选成不带 UEFI 的）。
3. 出现 OpenCore 选单（黑底白字列表），选 **Install macOS Tahoe**，回车。
4. 开始跑代码（`-v` 已经开了，会刷屏，正常）。几分钟后应该出现 macOS 安装器界面。**这时显示没有加速，分辨率可能不是 4K，颜色也可能发灰——都正常**，只要能看见、鼠标能动就行。
5. 安装器里先打开**磁盘工具**：方案 A（新 NVMe）把新盘抹成 APFS + GUID；方案 B 只在空出来的空间上新建一个 APFS 分区，**不要点整盘抹掉**（会连 Windows 一起抹）。然后关掉磁盘工具。
6. 点**安装 macOS**，选刚抹好的盘，开始装。中间会重启好几次。
7. **每次重启都会回到 OpenCore 选单**：如果列表里有 `macOS Installer` 就选它；装到后面会出现你新装的 macOS 卷（名字是你起的），就选那个。别选成 Windows。
8. 进到 macOS 设置助理（选地区、建账户那套），按提示走完，进桌面。

**检查点**（到哪一步卡住，就拍下屏幕发我）：

- [ ] OpenCore 选单出现了吗
- [ ] 选完 Install 之后，安装器界面出现了吗
- [ ] 磁盘工具里认到目标盘了吗（新 NVMe / 分好的 APFS 分区）
- [ ] 安装进度条走完了、重启后出现新 macOS 卷了吗
- [ ] 设置助理走完、进桌面了吗

---

## §7 装完收尾

1. **把 EFI 从 U 盘搬到内置盘**：进 macOS 后，挂载内置盘的 EFI 分区（终端 `diskutil list` 找 EFI 分区，`sudo diskutil mount diskNs1`），把 U 盘 EFI 分区里的 `EFI` 文件夹整个复制过去。
2. **注册启动项**：Firebat 的 config 里 `Misc → Boot → LauncherOption` 是 `Disabled`，且 Drivers 里没有 `Bootstrap.efi`，所以内置盘的 EFI 不会自动出现在固件启动列表里。改法：把 `LauncherOption` 改成 `Full`（重启后 OpenCore 会把自己注册进 NVRAM 启动项），或者开机进 BIOS 手动加一条指向 `EFI/OC/OpenCore.efi` 的启动项。注意：这步不做也不一定开不了机——`EFI/BOOT/BOOTx64.efi` 还在，多数固件能经默认路径兜底启动；但注册了才稳。
3. 拔掉 U 盘，重启，确认不插 U 盘也能进 OpenCore 选单、进 macOS。
4. **下一步就是第 0 步清单**：按 `minipc-step0-checklist-2026-11-15.md` 走——装 nullarc 的 v0.4.0 包、换上已编好的两个 kext、设 `phxfb-boot`、开机看日志。GPU 的事全在那份清单里。boot-args 里的 `phxdebug=1` 和 `=0x1` 在 §3.2 里已经处理好了，`install.sh --check` 不会在启动参数上卡住。如果启动日志一直是 `NVRAM phxfb-boot not set`，先看清单 §6 的 C 路（NVRAM 持久性兜底），别反复重装。

---

## §8 常见问题

| 现象 | 先查什么 |
|---|---|
| 开机直接进 Windows，看不到 OpenCore 选单 | Secure Boot 是不是真关了；启动顺序里 U 盘是不是 UEFI 项；U 盘 EFI 分区里 `EFI/BOOT/BOOTx64.efi` 在不在 |
| 选完 Install 后黑屏 | HDMI 线是不是开机前就插好了；换 USB-C DP 口试一次；拍下 `-v` 最后停住的那几行发我 |
| 安装器里磁盘工具看不到盘 | 目标盘是不是 NVMe 正常识别；方案 B 的话，Windows 里压缩卷是不是真分出来了 |
| 重启后又回到 Windows | 启动顺序：把装 macOS 的盘（或 U 盘）放第一；或者开机手动按启动菜单键选 |
| 跑代码时卡住不动超过 10 分钟 | `-v` 已经开了，拍下停住的最后几行发我，别硬重启超过两次 |
| 进桌面后没有网络 | 有线网：检查 `RTL812xLucy.kext` 是不是加进 config 且 Enabled；无线：AirportItlwm 是不是 fork 的 v2.4.0-alpha（上游 2.3.0 不支持 Tahoe） |

---

## §9 已知未解决 / 待验证

这些都不影响安装，装完再处理：

- **板载音频 SN6140**：AppleALC 无对应 layout，备选 VoodooHDA，装机后再试
- **USB 端口映射**：K8 专用的端口表还没做，装机后用 USBToolBox 重新 mapping
- **睡眠 / 唤醒**：迷你主机上没验证过，先别测
- **Wi-Fi / 蓝牙**：AirportItlwm + Intel 蓝牙那套是按 AX200 配的，但 K8 上还没实测过
- **GPU 驱动**：不在本指南里，走第 0 步清单
- **BIOS 启动菜单键**（F7/F11/Del）：以实机屏幕提示为准

---

## 附：核对来源

- OpCore Simplify（`lzhoang2801/OpCore-Simplify`）：2026-11-18 晚读 main 头 `9aed67e` 源码后确认不能用——`compatibility_checker.py` 的 AMD 分支不认 780M（Phoenix/1900），打印 `You cannot install macOS without a supported GPU.` 后 `sys.exit(0)` 硬退出，且在选版本之前；A 路已删。版本菜单是 Darwin 号（Tahoe 对应 25.7.3），对 K8 无意义
- Firebat EFI 内容（kext 清单与开关状态、SSDT 清单、boot-args 原文、SMBIOS、27 个 AMD 补丁、quirks）：2026-11-18 从 `luuumi-dev/firebat-u6c-hackintosh-efi` main 头 `abec07c` 的 `config.plist`（57,341 字节）与文件树原文逐项核对
- boot-args 检查逻辑：2026-11-18 读 `izerok/nullarc` hdmi-fb2 分支 `install/install.sh` 原文——按空格分隔的完整词匹配 `amfi=0x80`、`amfi_get_out_of_my_way=0x1`、`phxdebug=1` 三项；Firebat 自带的是 `amfi_get_out_of_my_way=1`（词对不上）且缺 `phxdebug=1`
- SSDT-ALIB 与第 0 步的 EFI 改动：`minipc-step0-checklist-2026-11-15.md` §4（PhxBringup.kext 进 EFI、SSDT-ALIB.aml 进 EFI/OC/ACPI、boot-args 与 csr 要求）
- K8 硬件（RTL8125×2、SN6140、AX200、4× xHCI + USB4、BIOS 2.09、40GB 内存、Lexar NM710 1TB、C: 剩约 290GB、HDMI 4K160）：用户 2026-11-18 发的硬件报告 `report__1.md`
- 安装器阶段无加速也能显示：nullarc #30 迷你主机实证（装完系统、没装驱动，显示器是亮的）
- BIOS 设置、磁盘分区、安装流程：AMD 黑苹果通用做法（Dortania 指南同款）
- macOS 安装器获取方式（`softwareupdate --fetch-full-installer`）：2026-11-18 已核对，App Store 已无 Tahoe 完整安装器
