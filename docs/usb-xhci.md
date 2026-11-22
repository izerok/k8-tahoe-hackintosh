# USB：xHCI INTx 与 USB-C/第二屏

> 原 README §7.9、§7.11。

### 7.9 xHCI INTx（取代 XhciNudge 轮询，2026-11-20）

AMD Phoenix 的 xHCI 只有一个 MSI 向量且只由 interrupter 0 触发，AppleUSBXHCI 的非等时事件在 interrupter 1，
所以 HID 事件会卡/只能用轮询。引入 15XA 笔记本线的 `WujieXhciIntx.kext`：

- 禁用 `XhciNudge.kext`，boot-args `xhcinudge` → `wujiexhci`
- 启动 45 秒后切 INTx；之前 2 ms 轮询兜底
- 实测 `debug.wujiexhci`、`wujiexhci_irqs` 递增、`spurious/storms=0`，键鼠正常；
  `kernel_task` 约 0.9%（XhciNudge 轮询时约 6–7%）
- 回退：`sysctl -w debug.wujiexhci=0` 即时回 MSI

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
