# 网络：有线、Wi-Fi 与 AirDrop

> 原 README §7.7、§7.15、§7.16。

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
用户复测 Chrome 下载也**不再崩溃**。若仍复现：禁用 RTL8125，改用 AX200 Wi-Fi 或 USB 网卡
（注意 Wi-Fi 只支持 WPA2 / WPA2-WPA3 混合，见 §7.15；WPA3-only 环境需先改路由器）。

### 7.15 Wi-Fi：WPA3-SAE 纯模式无法关联（2026-11-21）

**症状**：AX200 Wi-Fi（`AirportItlwm` 2.4.0，laobamac Tahoe fork）能扫描到十几个 AP，但连接自家网络一直失败，
菜单栏无明确报错。

**airportd 日志定位**：

```
ASSOC: network ... security=wpa3-sae ...
Failed to associate to Wi-Fi network <redacted> on interface en3
Association failed ... com.apple.wifi.apple80211API.error Code=-3900 "tmpErr"
```

**原因**：laobamac 的 `AirportItlwm` README 明确写明支持范围：

- ✅ WPA2-Personal（PSK/AES）
- ✅ WPA2/WPA3 transition（混合模式，PMF 可选）
- ❌ **WPA3-only（SAE）/ 强制 PMF / MLO / AWDL**

日志里同一请求内 `PRIVATE MAC` 先成功创建、失败后才有 `FAILED to create private MAC ((null))`，
那条是清理阶段的噪声，不是失败原因。

**处理**

1. 路由器无线安全改为 **WPA2/WPA3 混合** 或 **WPA2-PSK (AES)**（最直接）
2. 或改用 WPA2 的 2.4G SSID / 访客网络
3. 路由器不可改时：手机热点、USB 网络共享，或继续有线（en1 正常）
4. 必须用 WPA3 的话只能换 macOS 原生支持的 Broadcom 卡（Tahoe 上仍需 OCLP root patch）

**备注**：这与 USB 端口映射、蓝牙修复均无关；扫描、驱动加载、固件都正常（`AirportItlwm 2.4.0_7204c69 fw 68.01d30b0c.0`）。

### 7.16 AirDrop：需要 AWDL，当前驱动不支持（2026-11-21）

**现象**：Wi-Fi 正常连上（WPA2）后，iPhone 与主机互相仍看不到/无法 AirDrop。

**原因**：AirDrop 的发现与传输走 **AWDL**（Apple Wireless Direct Link），需要网卡驱动创建并维护
`awdl0` 接口。`AirportItlwm`（含 laobamac Tahoe fork 2.4.0）**不实现 AWDL**：

- `ifconfig -a` 里 **没有 `awdl0`**
- airportd 日志持续报 `AWDL REALTIME is inactive`
- 上游 README 明确写：AWDL / AirDrop / MLO 不支持

**社区现状**：目前唯一给 Intel 卡补 AWDL 的是 **ForbiddenItlwm**（含 BTLEAdvFixup），
但它只支持 **26.7.1 (25G241) / 26.7.2 (25G332) / 26.7.3 (25G342)**；本机 **26.6.2 (25G83) 不在支持范围**
（驱动在该 build 上会关闭 AirDrop、只作普通 Wi-Fi 用）。升级到 26.7.x 是唯一软件路径，但本机 GPU 栈
（PhxAccel/AuxKC）与小版本强相关，升级需重做，风险高。

**可选方案**

| 方案 | 说明 |
|---|---|
| LocalSend / PairDrop 等局域网工具 | 跨平台、走普通 TCP，最快可用；iPhone 装 App 即可 |
| 升级到 26.7.x + ForbiddenItlwm | 社区实测 AirDrop 可用（AX211/AX210），但需重做 GPU 栈，风险高 |
| 换 Broadcom 卡（BCM94360 系） | 原生 AirDrop/Handoff；Tahoe 仍需 OCLP root patch，与现有 GPU 补丁叠加需谨慎 |
