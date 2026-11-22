# 安装、EFI 与 GPU 接管

> 原 README §4–§5。

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

- **删除**笔记本/台式无关项：`VoodooI2C`(+插件)、`VoodooI2CHID`、`VoodooPS2Controller`(+插件)、`HoRNDIS`、
  `RtWlanU`、`RtWlanU1827`、`Rtl89Driver`、`RealtekCardReader`、`RealtekBluetoothFirmware`、`SMCBatteryManager`、
  `SMCLightSensor`、`BrightnessKeys`、`NootedRed`、`RDNA3Driver`、`AppleALC`、`NVMeFix`、`DebugEnhancer`、
  `UTBDefault`、`SSDT-USB-Reset`（2026-11-21 已从 ESP 物理删除）
- **新增**：`PhxBringup.kext`（`MinKernel 25.0.0`）、有线网卡 `RTL812xLucy`、`AirportItlwm`(Tahoe fork)、
  `IntelBluetoothFirmware` + `IntelBTPatcher`（lshbluesky Tahoe fork）、`BlueToolFixup`、
  `WujieXhciIntx.kext`（xHCI INTx，§7.9）、`VoodooHDA_BL.kext`（音频，§7.5）、
  `PhxCPPC.kext`（CPU PM，§7.10）、`RyzenHV.kext`（Docker/Hypervisor 后端，§11）、`USBToolBox` + `UTBMap`（§7.14）
- **2026-11-21 变更（与当前模板一致）**：
  - **OpenCore 1.0.8 DEBUG → RELEASE**（`nvram …:opencore-version` = `REL-108-2026-09-27`），配置 45 → 22 项、
    `ocvalidate` 无问题
  - `IntelBTPatcher` **重新启用**（lshbluesky fork 2.5.3，`DTSDKName macosx26.2`）——Tahoe 蓝牙必需（§7.14）
  - `USBPorts.kext` 禁用（Tahoe 拒绝 v1.0 注入），改用 **`USBToolBox` + `UTBMap`**（§7.14）
  - `BTLEAdvFixup` 已装入但**禁用**（25G83 早期 panic，见 §7.14 踩坑记录）
  - `XhciNudge` 仍禁用（由 `WujieXhciIntx` 取代）
  - 保留 3 个禁用备用：`XhciNudge`、`USBPorts`、`BTLEAdvFixup`
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
