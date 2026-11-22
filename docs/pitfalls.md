# 三个真正的坑与恢复手段

> 原 README §3、§6。

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

## 6. 恢复手段（重要）

| 手段 | 作用 |
|---|---|
| boot-arg `-phxfboff` | PhxAMDFB 不接管屏幕，回固件帧缓冲（**仓库模板 config 不含它**；救砖用的 U 盘 EFI 曾加过，需要时临时加到 boot-args 再启动） |
| `sudo touch /Users/Shared/gpu/phxboot-off` | 阻止下次启动 arm 接管 |
| `sudo nvram -d phxfb-boot` | 清除握手变量 |
| `sudo ./uninstall.sh` | 卸载 nullarc（恢复前先看官方 `docs/INSTALL.md`） |
| U 盘 EFI 启动 | 挂死时的救援路径（OpenCore 在 U 盘上） |
