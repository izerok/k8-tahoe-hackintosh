# 蓝牙（AX200 8087:0029）

> 原 README §7.4、§7.14。§7.4 是历史误判，保留作排错记录。

### 7.4 蓝牙：AX200 的 USB 蓝牙设备未枚举（历史，已由 §7.14 修复）

> **2026-11-21 更新：本节结论是错的，保留作排错记录。** 真正的根因是 `USBPorts.kext` 在 Tahoe
> 被内核拒绝（bundle-id 版本冲突），端口映射从未生效；且 Tahoe 必须加载 IntelBTPatcher。
> 完整修复与验证见 **§7.14**。

当时的观察：Wi-Fi 侧 `pci8086,2723`（AX200）存在且 `AirportItlwm` 已加载；蓝牙应为 USB `8087:0029`，
但 IORegistry 全树没有该设备，`IntelBluetoothFirmware` 不加载，`system_profiler` 只显示假的
`BCM_4350C2`。当时误判为「USB 未供电/未接线或 BIOS 关闭」。

### 7.14 蓝牙：AX200 `8087:0029` 在 Tahoe 上的双层修复（2026-11-21）

**症状**：`system_profiler` 只有伪造的 `BCM_4350C2`（UART、State Off），USB 树里根本没有
Intel 蓝牙设备；Windows 下同一硬件正常。

**根因 1 — 端口映射从未生效**：`USBPorts.kext` 的 bundle id 与系统
`com.apple.driver.AppleUSBHostMergeProperties` 相同，而系统预链版本是 v1.2，注入的 v1.0 被拒：

```
Refusing new kext com.apple.driver.AppleUSBHostMergeProperties, v1.0: already have prelinked v1.2
```

于是 macOS 只按 ACPI 建端口，XHC0 的 `PRT5`（`_UPC` 判为不可连接）被跳过——而它正是蓝牙所在口
（Windows `pnpBluetooth`：`USB\VID_8087&PID_0029\...&0&5`，同 root hub 上另有键鼠）。

**修复 1**：改用 `USBToolBox.kext` + 自建 `UTBMap.kext`（独立 bundle id），XHC0 七口映射为：
1–4 Type-A(3)、**5 Internal(255)**、6–7 Type-A(3)；XHC1 两口 Type-C(9)。
重启后 `Bluetooth USB Host Controller@61500000` 出现且 `FirmwareLoaded=Yes`。

**根因 2 — Tahoe 需要 IntelBTPatcher**：设备枚举、固件下载都正常后，bluetoothd 仍报
`Invalid chipset 0` 并循环退出（`Configuring bluetooth device: ... [0x61500000]` 之后）。
原版 IntelBTPatcher 的 max kernel 只到 Sequoia；需用 **lshbluesky fork 2.5.3（`DTSDKName=macosx26.2`）**。

**修复 2**：启用该 `IntelBTPatcher.kext`。

**验证**

```
Address: <redacted>      State: On        Chipset: THIRD_PARTY_DONGLE
Firmware Version: v8641 c8641            Transport: USB      LMPv 11 (5.2)
bluetoothd: 单进程、0% CPU、无 "Invalid chipset"、无重启循环
已加载: IntelBTPatcher 2.5.3 / BlueToolFixup 2.7.3 / USBToolBox 1.2.0 / IntelBluetoothFirmware 2.5.3
```

**注意**：`THIRD_PARTY_DONGLE` 表示由 BlueToolFixup 接管，Handoff/AirDrop 等 Apple 连续性功能
不可用（第三方蓝牙固有限制），普通键鼠/耳机/GATT 正常；BT MAC 属硬件标识，本仓不记录。

**踩坑记录（2026-11-21）**：曾尝试 Wujie 14 Pro 参考 EFI 里的 `BTLEAdvFixup.kext` 0.9.3 +
boot-arg `-btleadv` 来改善 BLE 广播，在 **26.6.2 / 25G83** 上直接早期 panic 并自动重启（无限重启）。
**社区文档（ForbiddenItlwm `docs/INSTALL.md`）给出真正原因**：`BTLEAdvFixup` 与 `IntelBTPatcher`
hook 同一个函数，**Lilu 遇到双插件 hook 会直接 panic**——当时两者同时启用才是 panic 主因；
正确组合是 `IntelBluetoothFirmware + BlueToolFixup + BTLEAdvFixup`，**必须关闭 IntelBTPatcher**，
由 BTLEAdvFixup 接管它的职责。

另外 `BTLEAdvFixup` 的补丁模式是从具体 bluetoothd 提取的字节，Tahoe 版只覆盖 **26.7.1 (25G241)、
26.7.2 (25G332)、26.7.3 (25G342)**；25G83 不在其中，模式不匹配时它**不会 panic，只是不生效**
（要支持 25G83 需从本机 bluetoothd 重新提取四个 site 并重编：`BTLEAdvPatches.hpp` + `tools/build_btleadv.py`）。
同时因为本机 boot-args 移除了 `debug=0x100`，panic 不留屏、也不留报告（DiagnosticReports 无新增、
NVRAM 无 panic 标记）——排障时应临时加回 `debug=0x100`。已在 `Kernel→Add` 中禁用并回滚。

**另注**：iPhone 能搜到主机但配对失败属正常——macOS 只对 HID/音频等 profile 开放配对，
`MobilePhone` 类设备会被忽略（日志里体现为 `Device lost: ... DvT MobilePhone`，无任何 pairing/SMP 事件）；
iPhone↔Mac 的 Apple 连续性配对走 BLE+iCloud 身份，第三方蓝牙芯片不支持。验收请用蓝牙耳机/键鼠。
