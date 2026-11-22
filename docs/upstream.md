# 参考项目与上游跟进

> 原 README §7.6、§7.12。

### 7.6 780M 笔记本线 vs 迷你主机线：还可推进什么

- **笔记本线（Wujie 14 Pro / 15XA）**：USB-C DP 第二屏（`phxfb2=1`/`phxfb3=1` + `tools/hdmi` 的 DMUB/DP 训练）；
  `WujieXhciIntx` 用 INTx 取代轮询；CPU PM/CPPC；快速静音启动；HiDPI；BTLEAdvFixup/ForbiddenItlwm（需 BT 硬件）
- **迷你主机线（F7BSC）**：vbl-timing、dbuf、4 个 NVMTL 补丁、ptfix 池上限、gfx-idle mesboot、thermal 脚本、§7.1 绕法
- **K8 已采纳**：vbl-timing/dbuf/NVMTL/ptfix/gfx-idle/thermal、VoodooHDA_BL、GFXOFF、gpucg、禁用 IntelBTPatcher/BT daemon
- **下一阶段优先级**：
  1. USB-C/HDMI 第二屏（高价值/高风险，需要 K8 VBIOS + DMUB 固件 + 不回读 PhxBringup；K8 USB-C 是否 DP Alt Mode 未验证）
  2. `WujieXhciIntx`（中高风险，需替换 XhciNudge，失败会丢键鼠）
  3. CPU PM/CPPC（中风险，可能与 DummyPowerManagement 冲突）
  4. 快速启动/去 `-v`（低风险，纯观感）

### 7.12 上游跟进：删 WAIT_INTERNAL、surface-marks 评估（2026-11-21）

跟踪 `sfyyy/mechrevo-f7bsc-hackintosh` 4 个新提交后的取舍：

- **已采纳（conf 部分）**：从 `nvrm610.conf` 删除 `NVMTL_WS_WAIT_INTERNAL=1`（上游 `ba64fa0`）。
  上游实测 WindowServer 连每次 CA 离屏绘制都在等 GPU，投屏时屏幕 27 → 53 帧。K8 删除后
  CVDisplayLink 3×5 s 采样 300 cb、avg 16.67 ms、max 17.7 ms、0 个 >20 ms（与删除前一致），
  **用户确认残影未复发**（dbuf 仍在），备份 `nvrm610.conf.bak-before-waitinternal-rm-*`。
  配套的 `nvmtl-ws-capture-nowait.patch`（采集合成不等 GPU）K8 无投屏场景，未装。
- **缓行**：`e1eea3c` surface-marks（B-58 根因修复，mark 从单 bit 改递增编号）——上游标注"尚未装机验证"；
  K8 的残影已由 §7.2 dbuf + §7.1 绕法解决，且它与我们已装的 4 个 NVMTL 补丁同文件，等上游装机验证后再评估。
  注：dbuf（显示扫描层）与 marks（合成等待层）是**不同层的两个问题**，互补而非替代。
- **认知更新**：780M 无 VCN 硬件编解码（上游 `931e5eb`）——腾讯会议/共享屏幕高 CPU 是 CPU 软编，
  等上游 VCN 队列计划；设置页/墙纸 tab 卡顿是 `SCHEDULE_AFTER_SURFACE_WRITES=1`（防花屏）+
  非 256B 对齐 surface 私有图像拷贝的**架构已知代价**，上游同样存在，无需行动；
  CoreImage 朝向/usage 补丁（`fdf98c6`）K8 未遇症状，按需。
