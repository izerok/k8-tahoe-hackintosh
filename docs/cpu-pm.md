# CPU 电源管理（CPPC/EPP）

> 原 README §7.10。

### 7.10 AMD CPPC / EPP（2026-11-20）

用 nullarc PR #21 的 `PhxCPPC.kext`（MSR CPPC），`/usr/local/bin/cppc.sh` 开机设
`debug.phx_cppc_enable=1`、`max/min=-2`、`epp=128`、`apply=1`：

- 空闲 core power **27 W → 1–2 W**，socket **35 W → ~5 W**，温度 51 → 41–45 °C
- 4×`yes` 满载 socket ~36 W，结束后自动回落
- 查看：`sysctl debug.phx_cppc_dump`；EPP 0=性能 … 255=省电
