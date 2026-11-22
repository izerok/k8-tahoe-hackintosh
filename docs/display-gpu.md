# 显示、GPU 与合成（含卡顿与 WindowServer 崩溃）

> 原 README §7.0–7.3、§7.13、§7.17；小节号保留便于对照。

### 7.0 空闲高频（nullarc #14）已修复（2026-11-20）

nullarc 的 `mesboot` 在建立 gfx 队列前没有写 `CP_MEC_DOORBELL_RANGE`，命令处理器永久报忙，PMFW 把 GFXCLK 钉在 2600 MHz，空闲也发热、风扇狂转。

修复：用 `izerok/nullarc` 的 **`gfx-idle` 分支**（commit `994fa0f`，"mesboot: set CP_MEC_DOORBELL_RANGE before the first gfx doorbell"）重新编译 `mesboot`，替换 `/Users/Shared/gpu/phxctl/mesboot` 后重启（只动用户态二进制，不改 kext/AuxKC）：

```bash
git clone --depth 1 --branch gfx-idle https://github.com/izerok/nullarc
clang -O2 -o mesboot nullarc/tools/phxctl/mesboot.c -framework IOKit -framework CoreFoundation
sudo cp mesboot /Users/Shared/gpu/phxctl/mesboot && sudo chown root:wheel $_ && sudo chmod 755 $_
```

实测：空闲 `debug.phx_pm_gfxclk` 从 **2600 → 800 MHz**（`mode=1`、上限 2600 也不影响空闲），接管与 Metal 3 正常。

### 7.1 已验证的显示配置（2026-11-19）

```bash
# 立即生效
sudo sysctl -w debug.nvaccel_surface_gpu_on=0

# /Library/GPUBundles/nvmtl/nvrm610.conf 追加（每个进程启动时读取，注销后生效）
NVMTL_WS_WAIT_COMPLETION=1
NVMTL_SCHEDULE_AFTER_SURFACE_WRITES=1
```

**2026-11-21 追加**：conf 还包含 `NVMTL_PIPELINE_THREADS=1`（WindowServer pipeline 崩溃规避，见 §7.17；
它只覆盖异步编译路径，不覆盖 CA 的同步调用）。

`debug.*` sysctl 每次重启会重置，需要开机自启（参考实现：`/usr/local/bin/phxdisplay.sh` + `/Library/LaunchDaemons/local.phxdisplay.plist`，`RunAtLoad` 执行 `sysctl -w debug.nvaccel_surface_gpu_on=0`）。

验证：`sysctl debug.nvaccel_surface_gpu_on` 为 0；插件二进制中存在上述两个 `NVMTL_*` 键；`debug.nvaccel_iop_fail` 保持 0。
实测结果：注销重登后残影**明显减轻但仍有残留**；再叠加 §7.2 的 `debug.phxfb_dbuf=1` 后，**用户确认残影消失**（2026-11-20）。

**开机显示依赖 EDID（2026-11-21 踩坑）**：PhxAMDFB 启动时沿用固件 framebuffer 的尺寸。若开机瞬间显示器
不在 HDMI 输入（例如输入源切到了 USB-C），固件会回退到 **3840×2160**（无 EDID），面板不支持就黑屏——
此时系统其实完全正常（SSH 可进、WindowServer 在跑），但**运行中切回 HDMI 不会重新读 EDID**，必须重启。
处理：确认显示器输入源=HDMI 后再重启。日志判据：`PhxFB: console: 2560x1600` 正常，`3840x2160` 异常。
注意 PhxAMDFB 不向 IOKit 发布 `IODisplayEDID`（ioreg 计数恒为 0），不要用它判断 EDID。

### 7.2 vbl-timing / dbuf 手工移植（2026-11-20）

参考项目的 `phxaccel-vbl-timing.patch` / `phxaccel-dbuf.patch` 对 fork 和 main `3497d87` 都冲突，已按 main 源码手工移植
（`patches/k8-accel-vbl-timing-dbuf.patch`）：

- **vbl-timing（67 ms 卡顿）**：`debug.phxfb_timing`（默认 1）让 `getTimingInfoForDisplayMode` 返回
  `kIODetailedTimingValid`，并把 `measureRefresh()` 提前到 `IOFramebuffer::start()` 之前。
  K8 上 OTG 实测总量 `2720x1481` **小于** 2560x1600 surface（固件缩放），参考补丁会因 `fVTotal <= fHeight`
  提前返回、无法进入 VBL timing 模式；移植版在 totals 不覆盖 surface 时补标准 blanking。
  验证：`ioreg -rc PhxAMDFB` 出现 `IOFBCurrentPixelClock = 268627210`；CVDisplayLink 5 秒采样
  **300 callbacks、out avg 16.667 ms、0 个 >20 ms**；修复前为 16.781 ms 步进并反复出现 **66.7 ms**（4 帧）空档。
- **dbuf（撕裂）**：`debug.phxfb_dbuf` 默认 0；`N > 1` 表示试运行 N 秒后自动回到单缓冲。
  实测 `=30`：`state=1`、30 秒 1027 flips、`wait_timeouts=0`、自动回退 `homes=1`；
  常开 `=1` 后 CVDisplayLink 仍稳定。**用户确认开启后残影消失**（2026-11-20），已把 `debug.phxfb_dbuf=1`
  写入 `/usr/local/bin/phxdisplay.sh`（开机重试直到 OID 可用）持久化。临时关闭：`sudo sysctl -w debug.phxfb_dbuf=0`。
- 驱动侧另有 `debug.phxfb_dbuf_state` / `_flips` / `_waits` / `_wait_timeouts` / `_wait_max_us` 可观测。

### 7.3 偶发 WindowServer 崩溃（毛玻璃管线）与剩余卡顿

- 2026-11-20 14:48 出现一次黑屏：`WindowServer` SIGSEGV（`EXC_BAD_ACCESS 0x212`），launchd 随即重启；
  崩溃栈为 `CA::OGL::BlurState::narrow_blur` → `CA::OGL::GaussianBlurFilter::render` →
  `MetalContext::draw` → `nvmtl_vk_pipeline_create_rt` → `radv_CreateGraphicsPipelines`
- 这是 **NVMTL/RADV 创建毛玻璃（backdrop blur）渲染管线时的驱动段错误**，不是 `dbuf`/`vbl-timing`：
  当时 `phxfb_dbuf_wait_timeouts=0`、`nvaccel_iop_fail=0`，显示翻转正常
- 缓解：`defaults write com.apple.universalaccess reduceTransparency -bool true`（下次登录生效），
  减少系统毛玻璃图层与 blur 管线；撤销：把 `true` 改成 `false`
- 卡顿量化：简单 60 fps 动画 58.8 fps、无 >40 ms 空档；3 个毛玻璃页面 56.9 fps，
  约 10–13% 的帧间隔为 33 ms，WindowServer CPU 27–46%。残余卡顿是 **blur 合成 CPU 开销**，
  `dbuf`/`phx_pm_mode`/`surface_gpu_on` 的 A/B 都在噪声内；67 ms 空档已由 §7.2 修掉

### 7.13 设置页/墙纸 tab 卡顿：A/B 与采样结论（2026-11-21）

删除 `WAIT_INTERNAL` 后用户确认残影未复发，但设置页/墙纸 tab 仍比其它程序卡。做了 A/B + 采样：

- **A/B 删 `NVMTL_WS_WAIT_COMPLETION=1`**：WS 主渲染线程从 CPU 忙转为阻塞在 Vulkan fence
  （`cnd_wait`/`__vk_sync_wait_many`），系统稳定。但它是 §7.1 防残影绕法的核心，**已恢复**，
  备份 `nvrm610.conf.bak-before-waitcompletion-test-*`
- **采样定位**：Settings 冷启动时 WS 热点是 `nvmtl_translate` 的 SPIR-V 解析/分配
  （rspirv parse/alloc 系列）——WindowServer 重启后首次加载 160 MB default.metallib 需全量翻译，
  结果被 "library key memo"（整库哈希）缓存，**只有 WS 重启后第一次开设置才卡**；
  插件 aircache（.ll 级）工作正常（SSV 侧 1300+ 条持久化），与此无关
- **GPU 与卡顿无关**：合成瓶颈在 CPU 翻译，GPU busy 长期 0%、gfxclk 800 MHz——
  提频（K8 Tune 1600/2600）对设置页卡顿无效
- **最终构成（三层，均为 nullarc 架构已知项）**：
  1. WS 冷启动后首次全量 SPIR-V 翻译（一次性，memo 缓存后消失）
  2. `SCHEDULE_AFTER_SURFACE_WRITES=1` 防花屏代价（设置页 UI 由各进程自绘）
  3. 非 256B 对齐 surface 的每帧私有图像拷贝（墙纸 tab 缩略图多）
- 当前 conf 组合（dbuf=1 + WAIT_COMPLETION + SCHEDULE_AFTER，无 WAIT_INTERNAL）为已知最优

### 7.17 WindowServer 崩溃：radv_CreateGraphicsPipelines（2026-11-21）

**症状**：桌面突然消失回到登录界面（WindowServer 重启），系统本身不重启、SSH 正常、无内核 panic。

**崩溃报告**（`WindowServer-*.ips`，EXC_BAD_ACCESS / SIGSEGV）：
```
libvulkan_radeon  radv_CreateGraphicsPipelines
NVMTLDriver       nvmtl_opaque_CreateGraphicsPipelines
NVMTLDriver       nvmtl_vk_pipeline_create_impl
NVMTLDriver       nvmtl_vk_pipeline_create_rt
NVMTLDriver       -[NVMTLDevice newRenderPipelineStateWithDescriptor:error:]
QuartzCore        CA::OGL::MetalContext::create_pipeline_state
```
同一栈出现 2 次（11-20 14:48 高斯模糊、11-21 17:45）。上游 main 仍为 `3497d87`，
release 仍 `v0.4.0-alpha`，nullarc issues 无对应报告；BUGS.md 只有 B-12（首次特效卡顿）与 B-22（缓存被误关）。

**分析（决定了哪些手段有用）**
- `NVMTL_PIPELINE_THREADS`（官方旋钮）只限制**异步**接口
  `newRenderPipelineStateWithDescriptor:completionHandler:` 的编译队列宽度（1 = 旧的串行行为）；
- 崩溃栈走**同步**接口 → `nvmtl_vk_pipeline_create_rt` → radv，**不经过该队列**，也未见串行锁；
  CA 的 `dispatch_async_pipeline` 会并发调用同步接口。
- 因此该旋钮对 WindowServer 这条崩溃路径**大概率无效**，不能当修复。

**已做的规避**
1. conf 增加 `NVMTL_PIPELINE_THREADS=1`（验证方法：用带 `NVMTL_LOG_PATH` 的**异步**探针，
   日志出现 `pipeline compile: 1 asynchronous builds at a time`；同步接口不会建队列，所以 WindowServer
   的日志里看不到这一行）
2. 清空 nvmtl `aircache`/`spvcache`、Mesa shader cache、root 与 SSV 侧 `aircache`（自动重建，
   首次启动会略慢）
3. 复核并补回丢失的 `NVMTL_WS_WAIT_COMPLETION=1`（防残影键，曾从 conf 中消失；建议每次重启 `od -c` 复核）

**若再犯**：开启「降低透明度」减少 blur pipeline 创建（此前实测有效），或由上游在同步路径串行化 / 修 radv 竞态。

## 其他显示相关修复与限制（原 §7 引言）

- **窗口捕获崩溃（已修复，2026-11-20）**：根因是 PhxBringup 每进程 GPU 页表 2 GiB 上限 + 显存池偏小。
  按 `phxbringup-ptfix` 语义把页表 64→256 块（≈8 GiB）、VRAM hi 1536→3840 MiB、GTT 512→2048 MiB
  （单块 1024）后，截屏/录屏/锁屏不再崩，锁屏动态壁纸正常；`vkfill` 可申请总量 2032 → 6128 MB。
  若复现，内核日志会出现 `PhxBringup: GPU page tables of context N full`。
- **Vision/OCR 闪退（已修复，2026-11-20）**：插件加载 `nn-hide` 补丁（`patches/nvmtl-nn-hide.patch`），
  让 Core ML/Vision 的推理栈看不到 GPU、改在 CPU 推理，程序自身 Metal 仍用 780M；QQ 扫码、Shottr OCR 不再崩溃。
- **无 HiDPI**：PhxAMDFB 只报告固件 GOP 的单模式。
- **锁屏（2026-11-20 复测：正常）**：同一池上限修复后，锁屏/解锁正常，动态壁纸在锁屏界面正常显示。
- **显存 2 GB**：由 BIOS 的 UMA Frame Buffer 分配决定；2560×1600 够用，想上 4K 建议调大。
- **显示器尺寸误报**：关于本机里"42 英寸"是 PhxAMDFB 报的尺寸，实际为 27 英寸，不影响使用。
