# K8 装机信息采集报告

- 采集时间：2026-11-18
- 系统：Windows 11 企业评估版 10.0.26200（UEFI 启动）
- 采集方式：WMI/CIM、PnP、注册表、EnumDisplaySettings、ACPI 固件表（VFCT）

## 0. 核对要点

| 项目 | 结果 |
|---|---|
| GPU Device ID | **1002:1900** ✅ 与预期一致 |
| VBIOS | 113-PHXGENERIC-001，已提取 ROM（16,896 B，校验通过） |
| BIOS | 2.09 / 2026-02-09（AMI，UEFI，SMBIOS 3.5） |
| 主板 | SMBIOS 为 Default string / Version 1.0；整机型号 K8 |
| 2.5G 网卡 | Realtek RTL8125 ×2 ✅ |
| 声卡 | **Conexant/Synaptics SN6140（14F1:1F87），不是 Realtek ALC** ⚠️ |
| Wi-Fi / 蓝牙 | Intel Wi-Fi 6 AX200（8086:2723）/ Intel BT（8087:0029） |
| USB | 4× AMD xHCI + 2× USB4 主机路由器 |
| 显示 | KTC 27M1 Max，**HDMI 3840×2160@160Hz**（最新复测） |
| 内存 | 40 GB（16 GB + 24 GB，DDR5-5600，非对称） |
| 硬盘 | Lexar NM710 1TB NVMe |
| 缺驱动 | 4 个 AMD 芯片组相关设备（14EC×2、15C7、AMDI0052） |

## 1. GPU / GPU-Z

- 型号：AMD Radeon 780M Graphics（RDNA3 核显，Ryzen 7 255/H255 内置）
- Device ID：**1002:1900**
  - 完整 PnP ID：`PCI\VEN_1002&DEV_1900&SUBSYS_80252014&REV_BA`
  - 子系统：2014:8025，Revision BA
- 驱动：32.0.13032.2001（2025-06-03，Advanced Micro Devices）
- VBIOS 版本：**113-PHXGENERIC-001**
  - 注册表 `HardwareInformation.BiosString` 读出的版本字符串
- VBIOS 内部标识：`ATOMBIOSBK-AMD VER022.012.000.029.000001`
- VBIOS 构建日期：2025-04-18 00:54:19
- ROM 备份：`VBIOS_780M_113-PHXGENERIC-001.rom`
  - 大小：16,896 字节（33 × 512）
  - SHA-256：`A81E1E22989753D1DE4326D0748C5AC182F77FBA63BC7271C56C08C48F05EB49`
  - 校验：8-bit 累加和 = 0；PCIR 签名存在；PCIR Vendor/Device = 1002:1900
  - 镜像大小字段：0x21 块 = 16,896 字节，与实际一致
  - PCIR code type = 0x00（legacy x86 镜像），image indicator = 0x80

说明：

- 本机没有安装 GPU-Z（Program Files、用户目录、开始菜单、卸载项均未找到）。
- 780M 是 APU，没有独立 VBIOS ROM 芯片；Windows 驱动使用的 VBIOS 来自 ACPI 的 **VFCT** 表。
- 这份 ROM 是从本机 VFCT 中原样提取的 1002:1900 镜像，和驱动实际使用的 VBIOS 一致；VFCT 原始表也一并保留。
- VFCT 表校验和 = 0；表长 17,028 字节，镜像偏移 0x68，长度 0x4200。
- 注意：该镜像是 VFCT 中的 legacy x86 VBIOS 部分（PCIR code type 0x00），不含主板 UEFI GOP 固件本身。若“后备路线”需要刷写/回滚，请确认工具是否接受 APU 的 VFCT 镜像。

## 2. 系统 / BIOS / 主板（msinfo32）

- 整机：Recoreqich K8（SMBIOS；对应科偲 KTHIN K8 系列）
- CPU：AMD Ryzen 7 255 w/ Radeon 780M Graphics，8 核 16 线程，3801 MHz
- BIOS：
  - 厂商：American Megatrends International, LLC.
  - 版本：**2.09**
  - 日期：**2026-02-09**
  - 模式：UEFI
  - SMBIOS：3.5
- 主板：
  - 制造商：Default string
  - 产品：Default string
  - 版本：Version 1.0
  - （SMBIOS 没有提供更具体的零售主板型号，整机型号为 K8）
- 操作系统：Windows 11 企业评估版 26200，安装日期 2026-10-07
- 完整报告：`msinfo32.txt`（1,092,912 字节）

## 3. 设备管理器 / 驱动清单

### 3.1 2.5G 网卡

- 型号：**Realtek Gaming 2.5GbE Family Controller ×2**（RTL8125）
- 硬件 ID：`PCI\VEN_10EC&DEV_8125&SUBSYS_812510EC&REV_05`
- 两个物理端口：
  - 端口 1（PCI 总线 1）：Realtek 驱动 `10.43.723.2020`（oem9.inf，2020-07-23），当前未连接
  - 端口 2（PCI 总线 3）：Microsoft 驱动 `1.0.0.14`（rtcx21x64.inf，2017-08-10），当前已连接，链路 2.5 Gbps
- 建议：干净安装后统一安装 Realtek 2.5GbE 驱动，避免一口用旧 Microsoft 驱动。

### 3.2 声卡

- **不是 Realtek ALC**。
- 板载 Codec：**Conexant/Synaptics SN6140（CX11880 系列）**
  - HDAUDIO ID：`HDAUDIO\FUNC_01&VEN_14F1&DEV_1F87&SUBSYS_14F103CF&REV_1001`
  - Vendor 14F1 = Conexant，Device 1F87 = SN6140/CX11880
- 当前驱动：Microsoft 通用 `High Definition Audio Device`（hdaudio.inf 10.0.26100.9818）
- 当前音频端点：
  - 活动输出（HDMI 复测）：`2 - 27M1 Max (AMD High Definition Audio Device)`；之前 DP 下为 `4 - 27M1 Max`
  - 板载耳机口：`耳机 (High Definition Audio Device)`，当前未插入/未 present
  - 麦克风：`麦克风 (High Definition Audio Device)`
- 另有 GPU 的 HDMI/DP 音频设备：`HDAUDIO\VEN_1002&DEV_AA01`，驱动 10.0.1.16（2020-04-01）
- 注意：如果装机清单按 Realtek ALC（例如 ALC897/ALC1220）准备驱动或 AppleALC 方案，这颗 Codec 对不上。

### 3.3 Wi-Fi

- 型号：**Intel(R) Wi-Fi 6 AX200 160MHz**
- 硬件 ID：`PCI\VEN_8086&DEV_2723&SUBSYS_00848086&REV_1A`
- 驱动：23.150.0.4（2025-06-12，Intel）
- 当前状态：WLAN 已启用但未连接（Disconnected）

### 3.4 蓝牙

- 型号：**Intel(R) Wireless Bluetooth**
- 硬件 ID：`USB\VID_8087&PID_0029`
- 驱动：23.90.0.8（2024-10-18，Intel）
- 说明：与 AX200 配套的蓝牙（USB 接口）

### 3.5 USB 控制器

- AMD USB 3.10 可扩展主机控制器（xHCI）×4：
  - `PCI\VEN_1022&DEV_15B9`
  - `PCI\VEN_1022&DEV_15BA`
  - `PCI\VEN_1022&DEV_15C0`
  - `PCI\VEN_1022&DEV_15C1`
  - 驱动：Microsoft USB xHCI 1.20（inbox）
- USB4 主机路由器 ×2：
  - `PCI\VEN_1022&DEV_1668`
  - `PCI\VEN_1022&DEV_1669`
  - 驱动：Microsoft USB4 主机路由器（inbox）
- 其他：USB4 根路由器、USB 3.0 根集线器等均为 Microsoft 通用驱动。

### 3.6 缺驱动设备（Device Manager problem 28）

| 设备 | 硬件 ID | 说明 |
|---|---|---|
| PCI 设备 ×2 | `PCI\VEN_1022&DEV_14EC` | AMD PCI / 芯片组设备 |
| PCI 加密/解密控制器 | `PCI\VEN_1022&DEV_15C7` | AMD 芯片组相关 |
| 未知 ACPI 设备 | `ACPI\AMDI0052` | AMD PPM Provisioning / GPIO |

→ 安装 **AMD Chipset Software（AMD 芯片组驱动）** 后，这 4 个设备应能全部识别。干净安装时这是必装项。

## 4. 显示 / EDID

### 4.1 HDMI 复测（2026-11-18 17:27，最新）

- 连接类型：**HDMI**（WMI `WmiMonitorConnectionParams`，VideoOutputTechnology = 5）
- 当前模式：**3840 × 2160 @ 160 Hz，32-bit**
  - EnumDisplaySettings（`ENUM_CURRENT_SETTINGS`）与 dxdiag 均显示 160Hz
  - dxdiag：`Current Mode: 3840 x 2160 (32 bit) (160Hz)`
- 活动显示器：**KTC 27M1 Max**
  - EDID PnP ID：`SKG27AC`（产品代码 0x27AC）
  - 活动实例：`DISPLAY\SKG27AC\5&1121e459&0&UID280_0`
  - 生产：2026 年第 1 周；尺寸 60 × 34 cm（27 英寸）
- 当前 HDMI 连接下可选的 3840×2160 刷新率：**50 / 59 / 60 / 119 / 120 / 144 / 160 Hz**
- 当前 HDMI 连接下可选的 2560×1440 刷新率：59 / 60 / 120 / 144 / 160 Hz
- EDID 支持范围：48–160 Hz，最大像素时钟 1480 MHz
- 备注：`Win32_VideoController` 的 `CurrentRefreshRate` 仍显示 59（WMI 缓存滞后，多次查询不变）；以 EnumDisplaySettings / dxdiag 的 **160Hz** 为准。dxdiag 的 Native Mode 是 3840×2160@59.997Hz（基础块详细时序），160Hz 来自扩展块/驱动。
- 显示器音频：HDMI 音频端点为 `2 - 27M1 Max (AMD High Definition Audio Device)`。

EDID 文件（HDMI）：

- 原始 EDID：`edid-now\EDID_SKG27AC_5_1121e459_0_UID280.bin`（384 字节，含 2 个扩展块）
- 易读副本：`EDID_KTC_27M1_Max_HDMI.bin`
- SHA-256：`854E32E242BE41C5F88BDB4D470BFCFB5E0334B2B2282B76598E10112F47BB8E`

### 4.2 之前的 DP 记录（2026-11-18 16:5x）

- 连接类型：WMI 枚举为 DisplayPort (External)
- 当时模式：3840 × 2160 @ 59 Hz，32-bit
- 活动实例：`DISPLAY\SKG27AC\5&1121e459&0&UID264_0`
- 当时 EDID：`edid\EDID_SKG27AC_5_1121e459_0_UID264.bin`；易读副本 `EDID_KTC_27M1_Max.bin`
- 说明：注册表中还有旧的 Philips 279P1（PHL0948）和其他 SKG 实例的 EDID 缓存，不是当前活动显示器。

## 5. 内存 / 硬盘

### 5.1 内存

- 物理总容量：**40 GB**（2 × SODIMM，DDR5-5600）
- Windows 可见：37.77 GiB（核显保留约 2 GiB）

| 插槽 | 容量 | 厂商/型号 | 速度 | 类型 |
|---|---|---|---|---|
| P0 CHANNEL A / DIMM 0 | 16 GB | SK Hynix `HMCG78AGBSA095N` | DDR5-5600 | SODIMM |
| P0 CHANNEL B / DIMM 0 | 24 GB | Micron/Crucial `CT24G56C46S5.M8C1` | DDR5-5600 | SODIMM |

- 注意：16 + 24 GB 为非对称双通道（flex mode），不是对称双通道；如果清单要求 32GB（16×2）或对称双通道，这一项不符。

### 5.2 硬盘

- 型号：**Lexar SSD NM710 1TB**
- 接口：NVMe（PCIe 4.0），GPT
- 容量：1,000,204,886,016 字节（约 931.5 GiB）
- 固件：`VF101C6F`
- 健康状态：Healthy / OK
- 分区：
  - C: NTFS，998,980,448,256 字节，可用 311,142,858,752 字节（约 289.8 GiB）
  - 另一个 NTFS 分区约 949 MiB（无盘符，恢复/引导相关），可用约 123 MiB

## 6. 文件清单

| 文件 | 说明 |
|---|---|
| `VBIOS_780M_113-PHXGENERIC-001.rom` | 780M VBIOS 镜像，16,896 字节 |
| `VFCT.bin` | ACPI VFCT 原始表，17,028 字节 |
| `EDID_KTC_27M1_Max_HDMI.bin` | 当前 HDMI 活动显示器 EDID（易读命名） |
| `edid-now\*.bin` | 本次 HDMI 复测导出的 EDID |
| `EDID_KTC_27M1_Max.bin` | 之前 DP 实例的 EDID（易读命名） |
| `edid\*.bin` | 注册表中所有显示器 EDID（含旧实例） |
| `dxdiag.txt` | dxdiag 报告（当前 HDMI 4K160） |
| `hardware.json` | 本次采集的完整原始数据（WMI/PnP/注册表/EDID 解析） |
| `msinfo32.txt` | msinfo32 系统信息完整报告 |
| `report.md` | 本报告 |

## 7. 备注 / 可能影响装机清单的点

1. **声卡不是 Realtek ALC**：实际为 Conexant/Synaptics SN6140（14F1:1F87），当前用 Microsoft 通用 HDA 驱动。若清单按 ALC 写，需要改。
2. **缺 AMD 芯片组驱动**：14EC ×2、15C7、AMDI0052 均为 problem 28；装 AMD Chipset Software 后应解决。
3. **当前 HDMI 已是 4K160**：显示器 KTC 27M1 Max，当前 HDMI 3840×2160@160Hz（dxdiag/EnumDisplaySettings 确认）；早先 DP 连接时为 4K60。
4. **连接类型**：当前 WMI 枚举为 HDMI（tech=5）；早先 DP 记录为 DisplayPort External。清单核对 HDMI 时按最新数据即可。
5. **内存为非对称 16+24 GB**：如果清单是 32GB 对称双通道，这一项不符。
6. **VBIOS 来源**：本机无 GPU-Z；780M 为 APU，无独立 ROM 芯片，ROM 从 ACPI VFCT 提取，校验通过。该镜像为 legacy x86 部分，UEFI GOP 在主板固件中。
