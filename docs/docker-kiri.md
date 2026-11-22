# Docker / Kiri（RyzenHV 后端）

> 原 README §11。

## 11. Docker / Kiri（2026-11-20）

K8 上可以在 AMD Ryzen 黑苹果里跑容器：**RyzenHV**（`OverlordDev/ryzen-hypervisor.framework`，MIT）
内核扩展用 AMD-V/SVM 实现了 Apple `Hypervisor.framework` 的后端（`kern.hv_support` 原本为 0，
这是 Kiri/Docker Desktop/VirtualBox 等所有走该框架的 VM 应用在 AMD 上起不来的共同根因）。

- RyzenHV 0.7.8 **native 模式**：kext 经 `hv_support_kext` 接口注册为 hypervisor provider，
  boot-args `rhv_enable_svm=1 rhv_allow_users=1 rhv_hv=1`；`rhv-info hv` 报 SVM/NPT/TPR shadow 全部就绪
- `kern.hv_support` 每次重启后回 0，由 `rhv autostart install` 的 LaunchDaemon 置 1
- **Kiri 0.2.2**（走 Hypervisor.framework 的容器引擎）：装 RyzenHV 后从
  `VM did not boot: this Mac does not support virtualization` 变为完全可用——
  `hello-world`、`nginx -p 8080:80` + `curl` 200、`ps/rm` 均正常
- Docker Desktop 4.97.0 验证可运行（RyzenHV native 模式无需 shim），本机已卸载、由 Kiri 覆盖需求
- **`/var/run/docker.sock` 默认不存在**（Docker Desktop 才创建它）：`docker load -i xxx.tar` 之类
  直接连 `/var/run/docker.sock` 的命令会报 `no such file or directory`。
  修复：`sudo kiri setup --docker-socket`，创建 `/var/run/docker.sock → ~/.kiri/run/docker.sock`
  （只影响 docker-py/Testcontainers 这类忽略 context 的工具；`kiri`/`docker` CLI 走 context 不受影响）
- 实测 `docker load` 169 MB 镜像包 → 4 个 tag 全部载入；容器运行、端口发布、出站网络均正常
- 已知边界：RyzenHV 实验性（无 entitlement 检查、睡眠/唤醒未验证、Tahoe 非上游实测版本）；
  compose/build/K8s（`kiri k8s start`）尚未测试；容器内 netlink 操作（如防火墙规则）因权限不足失败，属容器沙箱正常行为
