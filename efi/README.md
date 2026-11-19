# EFI 说明（脱敏模板）

本目录只包含**文本**部分，二进制不在仓库内：

```
OC/config.plist     脱敏模板（SMBIOS 为占位符，ROM 全零）
OC/ACPI/*.dsl       ACPI 源文件（对应编译后的 .aml 需自行生成或从上游获取）
```

## 不在仓库里的内容与出处

| 内容 | 出处 |
|---|---|
| OpenCorePkg 1.0.8（BOOTx64.efi、OpenCore.efi、Drivers、Resources） | https://github.com/acidanthera/OpenCorePkg/releases |
| Lilu / VirtualSMC / AMFIPass / RestrictEvents / AppleMCEReporterDisabler / ForgedInvariant / NullEthernet / NVMeFix | https://github.com/acidanthera |
| XhciNudge.kext（二进制） | 来自 `bbbbRT` 的 EFI；行为分析见 `sfyyy/mechrevo-f7bsc-hackintosh` |
| PhxBringup.kext / PhxAccel.kext | `izerok/nullarc`，分支 `hdmi-fb2` @ `50db1e1`（见根 README 的 SHA） |
| AirportItlwm（Tahoe fork）、IntelBluetoothFirmware/IntelBTPatcher（Tahoe fork）、BlueToolFixup | `laobamac/itlwm`、`lshbluesky/IntelBluetoothFirmware`、`acidanthera/BrcmPatchRAM` |
| RTL812xLucy v1.1.3 | https://github.com/Mieze/RTL812xLucy |
| GenSMBIOS | https://github.com/corpnewt/GenSMBIOS |

## 使用前必须做的

1. **生成自己的 SMBIOS**：`GenSMBIOS` → `MacBookPro16,1` → 序列号 / MLB / UUID；`ROM` 填有线网卡 MAC
   （config 中当前为 `GENERATE-WITH-GenSMBIOS` 和全零，直接启动会失败）
2. 把上面列出的 kext 按 `Kernel → Add` 中的 `BundlePath` 放入 `OC/Kexts/`
3. ACPI：用 `iasl` 编译 `.dsl` 后放入 `OC/ACPI/`，并确认 `ACPI → Add` 条目齐全
4. 校验：`ocvalidate OC/config.plist`（OpenCore 1.0.8 对应版本）
5. `boot-args` 中的 `xhcinudge=4` 是 AMD Zen 4 机器的关键，不要删

## 关键条目速查

- `Kernel → Add`：`Lilu → AMFIPass → XhciNudge → 网络 kexts → VirtualSMC → PhxBringup`
- `NVRAM → Add → 7C436110-…`：`boot-args` 与 `csr-active-config=7F080000`
- `Misc → Security → SecureBootModel = Disabled`
- `Misc → Boot → LauncherOption = Disabled`（装完系统后再考虑改 `Full`）

## 补充说明（2026-11-19）

- 模板 `OC/config.plist` 已把 `Misc → Boot → Timeout` 设为 **5 秒**（原配置为 0，会无限等待按键）
- 模板 **不含** `-phxfboff`；它是 GPU 接管失败时的逃生开关，需要时临时加到 `boot-args`
- `SSDT-PLUG-ALT.dsl` 未使用（本机用 `SSDT-CPUR`），已从本目录移除
