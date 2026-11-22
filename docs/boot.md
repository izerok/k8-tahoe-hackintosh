# OpenCore 与快速启动

> 原 README §7.8。

### 7.8 快速启动（2026-11-20，2026-11-21 更新）

- **OpenCore 1.0.8 已换 RELEASE**（原 DEBUG）：`OpenCore.efi` 868 KB → 627 KB、`BOOTx64.efi` 53 KB → 25 KB，
  启动到 SSH 约 15–20 s；旧 DEBUG 二进制保留为 `*.debug-bak-*` 可回退
- boot-args 去掉 `-v`（保留 `keepsyms=1`、`msgbuf`）
- `Misc.Debug.Target=1`（只 console，不向 ESP 写 `opencore-*.txt`）、`AppleDebug=false`、`LogModules=""`
- `ocvalidate 1.0.8` 校验：No issues found
- 实测重启后 SSH 约 30 秒回来（之前 45–55 秒），ESP 不再新增 OpenCore 日志；`Timeout=5` 不变
