# 迷你主机第 0 步清单：编译 + 开机亮屏实测

> **2026-11-18 更新（必读）：**
> - 两个 kext 已编好发你了（PhxBringup.kext + PhxAccel.kext，hdmi-fb2 @ 50db1e1，Actions run 44210948721 全绿），**§1–§2 可以跳过**，直接走 §3 的替代捷径（下官方 v0.4.0 包，只换两个 kext）。完整 SHA 见 §3 末尾核对表（zip + 两个 kext 二进制各一份，先对 SHA 再往下走）。
> - **前置条件未满足**：本清单 §0 要求迷你主机已经能进 macOS Tahoe 26（OpenCore 正常工作）。K8 现在还是 Windows，装 macOS 是第 0 步之前另一步，本清单不含装机步骤——先装好 macOS 再走这份清单。
> - **版本口径（2026-11-18 晚，#30 对齐）**：装机目标系统改为 **26.7.3**（build 25G342）——#30 那台 UM790 Pro 在 26.7.3 + v0.4.0 上跑通了玻璃效果；`install.sh` 只要求 26.*，26.7.3 通过检查。§0 表里引的 nullarc 实测版（26.6.2、25G83）是上游文档原话，不动。
> - **新增 §6 C 路（NVRAM 持久性兜底，#30 实证）**：如果启动日志一直是 `not set`，不是握手没设，是变量重启就丢——把 `phxfb-boot` 写进 config.plist NVRAM → Add。
> - **2026-11-18 21:29 #30 机制已确认**：报告人回了三行成功日志（`taken` / `console: 1920x1080 row 7680 depth 32` / `panel published … vbl timer running`）与 config.plist 原文——head 0 在 HDMI 上接管 GOP 画面已实证，重启后玻璃还在、动态壁纸回来（切换其他壁纸会卡一会儿）。第 0 步不走 `phxfb2`（那是给第二块屏的，需要用户态工具点屏），只靠 head 0；B-11 仍是“再接一块屏”的情形，不影响单屏 head 0。
>
> 底座：用户 fork `izerok/nullarc`，分支 `hdmi-fb2`，尖提交 `50db1e1`（已核对 `git log`）。
> 依据：本清单的每一条命令都能追溯到下面引用的文件；凡是文档没写清的，列在 §9，不猜。
> 验收线（计划 §6.2）：屏幕从 OpenCore 菜单到登录界面全程有画面，登录后桌面正常显示。分辨率、睡眠、热插拔不计入第 0 步。
> 全程只读研究，未动代码、未发帖、未联系任何人。

---

## 0. 开始之前：先确认这几件事

| 确认项 | 为什么 | 出处 |
|---|---|---|
| 迷你主机已经能进 macOS Tahoe 26，且 OpenCore 正常工作 | 本清单只是在一台**已经能开机进 macOS** 的黑苹果上加显卡驱动，不是教你装黑苹果 | `docs/INSTALL.md` "What you need"：macOS Tahoe 26.6（在 26.6.2、25G83 上测过），已装好并用 OpenCore 启动 |
| macOS 版本是 26.x | `install.sh` 会拒绝非 26 的系统 | `install/install.sh` checks 段：`26.*` 才通过，否则报错 "nullarc is built for macOS Tahoe 26" |
| 显卡是 780M/760M（PCI `1002:15bf` 或 `1900`） | 其他 GPU 不支持 | `docs/INSTALL.md` "What you need"；`install.sh` 检查 PCI id `1002:15bf/15c8/1900/1901`（找不到只警告，不拦截） |
| 有备用盘/备用安装 + 一个能启动的备用 EFI U 盘 | 驱动要求全系统放宽 AMFI（FairPlay 视频会坏），且要反复开关机；主力系统一旦开不了机，排障很被动 | `docs/INSTALL.md` 开头警告；计划 §6.1 |
| 记下机器 exact 型号、macOS 版本、接口清单（几个 HDMI/DP/USB-C）、显示器型号和原生分辨率 | 每轮测试都要贴一遍，免得轮次之间对不上；K8 已确认：KTHIN K8 / H 255 / HDMI 4K160（KTC 27M1 Max）/ 目标 26.7.3（见 §9 第 10 条） | 计划 §6.1、§6.2；硬件报告 report__1.md |

---

## 1. 准备构建环境（都在 Mac 上做）

构建只能在 Mac 上进行：kext 是 x86_64 的 `-mkernel` 产物，Linux 上编不了。最顺手是在迷你主机本身上编（它进 macOS 后就是一台 x86_64 Mac），也可以在另一台 Mac 上编完再拷过去。

```sh
# 1. 克隆 fork（hdmi-fb2 分支）
git clone --branch hdmi-fb2 https://github.com/izerok/nullarc.git
cd nullarc

# 2. 装 Command Line Tools（README 要求），并取 MacKernelSDK
#    CI 用的就是这个办法：git clone --depth 1（.github/workflows/phxbringup-kext.yml）
git clone --depth 1 https://github.com/acidanthera/MacKernelSDK.git ~/gpu/MacKernelSDK

# 3. 准备 Vulkan-Headers（只给 Metal 插件构建用，见 §2）
#    src/nmplugin/build.sh 默认去 $HOME/gpu/Vulkan-Headers/include 找
```

注意：

- **没有任何文件钉住 Xcode / Command Line Tools / MacKernelSDK 的版本号。**CI 在 `macos-15` runner 上用最新 main 的 MacKernelSDK；Mesa 的 README 写的是 Tahoe 26.6.2 + CLT 16.2（`mesa-patches-phoenix/README.txt`）。构建机上的版本要记下来，编不过先对版本（见 §9 第 7 条）。
- `README.md` 与 `README_RU.md` 的构建段命令完全相同（只差注释语言，已逐行对过）。

---

## 2. 构建（在 Mac 上）

### 2.1 先跑冒烟测试（建议，不装机也能做）

```sh
make test
python3 tests/test_phx_profile.py
```

- 出处：`README.md` "Building and Testing"；`Makefile` 的 `test` 目标。本分支的测试共 7 个（含新增的 `test_phx_smu_pm`，`Makefile` 与主线的唯一差异就是多这一项，已用 `git diff` 核对）。
- ⚠️ **不要用 `make kext`。**`Makefile` 的 `kext` 目标调的是 `tools/build_kext.py`，编的是脚手架 `RDNA3Driver`，不是发货的两个 kext（计划 §3.2 已核对）。

### 2.2 编两个发货的 kext

```sh
# PhxBringup.kext（进 OpenCore EFI 的那个）
sh src/phxbringup/build.sh ~/gpu/MacKernelSDK
# 产物：src/phxbringup/build/PhxBringup.kext
# 成功时打印 "built .../PhxBringup.kext"

# PhxAccel.kext（进 AuxKC 的那个，含 PhxAMDFB 帧缓冲）
sh src/phxaccel/build.sh ~/gpu/MacKernelSDK
# 产物：src/phxaccel/build/PhxAccel.kext
# build.sh 末尾自动做 ad-hoc 签名（codesign --force --sign -）
```

- 出处：`README.md` "Building and Testing"；两个 `build.sh` 的用法行（`build.sh <MacKernelSDK>`）。
- 自查（照 CI 的做法，`.github/workflows/phxbringup-kext.yml` 的 "Check the bundle" 一步）：

```sh
K=src/phxbringup/build/PhxBringup.kext
file "$K/Contents/MacOS/PhxBringup"
lipo -archs "$K/Contents/MacOS/PhxBringup"   # 应只有 x86_64
plutil -lint "$K/Contents/Info.plist"
```

### 2.3 编 Metal 插件（二进制，不是整个 bundle）

```sh
cd src/nmplugin && ./build.sh
# 产物：./NVMTLDriver.new（单个 Mach-O 二进制）
# 默认去 $HOME/gpu/Vulkan-Headers/include 找 Vulkan 头文件；
# 也可以显式传：./build.sh /path/to/Vulkan-Headers/include
```

- 出处：`README.md`；`src/nmplugin/build.sh` 用法行（`build.sh [Vulkan-Headers include dir] -> ./NVMTLDriver.new`）。
- 注意：产物只是二进制，**不是 `NVMTLDriver.bundle` 目录**。装进 bundle 要靠 `tools/phxctl/plugdeploy.sh`（往已有的 bundle 里换二进制），而 bundle 的壳子只能取自发布包（见 §3）。分支里检入了一个现成的 `src/nmplugin/NVMTLDriver.new`（Mach-O 64 位 x86_64 bundle，约 1.5 MB，已用 `file` 核对），但它由哪次提交编出、是否与分支尖源码一致，仓库里没写——**别默认它就是分支尖编出的**，第 0 步优先自己编。

### 2.4 编 mesboot（有文档写明的命令）

```sh
clang -O2 -o mesboot tools/phxctl/mesboot.c -framework IOKit -framework CoreFoundation
# 须 root 运行，且 boot-arg 里要有 phxdebug=1
```

- 出处：`tools/phxctl/mesboot.c` 第 2 行头注释原文就是这条命令。
- 注意：`mesboot` 是 `phxboot.sh` 每启动时调的（`$G/phxctl/mesboot claim`，`tools/phxctl/phxboot.sh:53`），装机时它要放在 `/Users/Shared/gpu/phxctl/` 里（见 §4）。

### 2.5 phxctl：仓库里没有构建命令（缺口，见 §9）

- `tools/phxctl/phxctl.c` 里搜不到任何构建说明（已 grep 全文件）。它和 `mesboot.c` 是同类工具（都连 PhxBringup 的 user client），但仓库没给它的编译命令。
- 第 0 步的寄存器快照要靠它（见 §7）。先在 Mac 上试着编（与 mesboot 同类命令是否行得通，须实测验证），编不出就暂用 v0.4.0 发布包里的现成 `phxctl`（见 §3 的替代捷径）。

### 2.6 Mesa RADV（全源码路线才需要，最重的一块）

按 `mesa-patches-phoenix/README.txt`：

```sh
# 基线 Mesa f5cb8ee032adabef599ae892ec4e42d796b84da9
# 先打 Cobalt64 的 mesa-patches 0001..0009，再 git am 本目录的 0001..0007
# 构建环境：Tahoe 26.6.2、CLT 16.2、Python 3.14 venv（meson/ninja/mako/pyyaml/packaging）、
# glslangValidator 从源码编进 ~/gpu/prefix
meson setup build --buildtype=release -Dvulkan-drivers=amd -Dgallium-drivers= -Dplatforms= \
  -Dllvm=disabled -Dglx=disabled -Degl=disabled -Dgbm=disabled -Dopengl=false -Dgles1=disabled \
  -Dgles2=disabled -Dxmlconfig=disabled -Dvalgrind=disabled -Dlibunwind=disabled -Dzstd=disabled \
  -Dbuild-tests=false -Dvulkan-layers= -Dtools=
ninja -C build
# 产物：build/src/amd/vulkan/libvulkan_radeon.dylib
```

- 仓库里只有补丁和这份 README，没有一键脚本。工作量大，见 §3 的替代捷径。

### 2.7 第 0 步不需要编的东西

- **PhxSleepPolicy**（休眠 kext，`src/sleeppolicy/build.sh` 存在）：`install.sh` 不装它，发布包里也没有它；休眠不在第 0 步验收内。别为第 0 步编它。
- **不要动 `#14/#20` 之外的代码。**第 0 步是零改动实测（计划 §5.1）。

### 2.8 构建产物核对表

| 产物 | 位置 | 用途 |
|---|---|---|
| `PhxBringup.kext` | `src/phxbringup/build/PhxBringup.kext` | 进 EFI，见 §4 |
| `PhxAccel.kext` | `src/phxaccel/build/PhxAccel.kext` | 经 `install.sh` 进 AuxKC，见 §5 |
| `NVMTLDriver.new` | `src/nmplugin/NVMTLDriver.new` | 插件二进制，见 §3（不是 bundle） |
| `mesboot` | 编出的二进制 | 放进包的 `gpu/phxctl/`，见 §3 |
| `phxctl` | 待定（见 §2.5） | 放进包的 `gpu/phxctl/`，见 §3 |
| `libvulkan_radeon.dylib` | Mesa 构建产物 `build/src/amd/vulkan/libvulkan_radeon.dylib` | 放进包的 `PhxGPU/nvmtl/`，见 §3 |

---

## 3. 组装发布包布局（关键缺口：没有一键命令）

`install.sh` 只认**发布包布局**：它开头的检查段要求这些文件都在包根目录的相对位置上（`install/install.sh` checks 段）：

```
EFI/OC/Kexts/PhxBringup.kext
Library/Extensions/PhxAccel.kext
PhxGPU/NVMTLDriver.bundle
PhxGPU/nvmtl/libvulkan_radeon.dylib
gpu/phxboot.sh  gpu/full.sh  gpu/bringup5.sh  gpu/kinst.sh
gpu/phxctl/phxctl  gpu/phxctl/mesboot
gpu/local.phxboot.plist
gpu/fw/*.bin（至少 9 个）
```

但**仓库检出里没有这个布局**：`install.sh` 在 `install/` 下，kext 在 `src/*/build/` 下，脚本在 `tools/phxctl/` 下，没有 `PhxGPU/` 目录，也没有 `gpu/fw/`。而 `tools/release/build_release.sh` 是从**一台已经装好并跑着的机器**上抓现成文件拼包的（从 EFI 分区取 PhxBringup.kext、从 `/Library/Extensions` 取 PhxAccel.kext、从已装的 PhxGPU 目录取插件、从 `/Users/Shared/gpu/fw` 取固件），**不能从源码凭空拼出包**。所以必须手动组装。

```sh
# 在 Mac 上建一个包目录，逐项放进去（以 ~/nullarc-pkg 为例）
mkdir -p ~/nullarc-pkg/{EFI/OC/Kexts,Library/Extensions,PhxGPU/nvmtl,gpu/phxctl,gpu/fw}

# kext（§2 编出的）
cp -R src/phxbringup/build/PhxBringup.kext ~/nullarc-pkg/EFI/OC/Kexts/
cp -R src/phxaccel/build/PhxAccel.kext  ~/nullarc-pkg/Library/Extensions/

# 脚本（仓库里直接复制）
cp tools/phxctl/phxboot.sh tools/phxctl/full.sh tools/phxctl/bringup5.sh \
   tools/phxctl/kinst.sh tools/phxctl/local.phxboot.plist ~/nullarc-pkg/gpu/
cp mesboot ~/nullarc-pkg/gpu/phxctl/          # §2.4 编出的
cp phxctl  ~/nullarc-pkg/gpu/phxctl/          # §2.5，构建方式待定

# 安装器本身（放包根目录，install.sh 要求从包根目录用 sudo 跑）
cp install/install.sh install/uninstall.sh ~/nullarc-pkg/

# 固件（见下面的固件小节）
# cp .../*.bin ~/nullarc-pkg/gpu/fw/
```

**没解决的几项（组装时必须另想办法）：**

- `PhxGPU/NVMTLDriver.bundle`：源码只产出 `NVMTLDriver.new` 二进制，bundle 的壳子没有构建路径。`tools/phxctl/inst_plugin.sh` 的注释写的是装 "NVMTLDriver.bundle (NullMoth 2.4.1, unmodified)"——即壳子取自别处。`plugdeploy.sh` 也只是往**已有的** bundle 里换二进制。
- `PhxGPU/nvmtl/libvulkan_radeon.dylib`：须按 §2.6 编 Mesa；另需 `radeon_icd.json` 与 `nvrm610.conf`（`build_release.sh` 从已装系统复制；`inst_plugin.sh` 会现场生成 `radeon_icd.json`）。
- `gpu/fw/*.bin` 至少 9 个：`build_release.sh` 的 `FW` 变量列出 9 个文件名（`psp_13_0_4_toc.bin`、`gc_11_0_1_pfp.bin`、`gc_11_0_1_me.bin`、`gc_11_0_1_mec.bin`、`gc_11_0_1_mes_2.bin`、`gc_11_0_1_mes1.bin`、`gc_11_0_1_imu.bin`、`gc_11_0_1_rlc.bin`、`sdma_6_0_1.bin`）。`tools/fetch_test_firmware.sh` 只从 linux-firmware 下载其中 7 个（缺 `psp_13_0_4_toc.bin`、`gc_11_0_1_mes_2.bin`、`gc_11_0_1_mes1.bin`；多一个 `dcn_3_1_4_dmcub.bin`，那是 #27 后备路线换 DMCUB 固件用的）。缺的 3 个按文件名都在上游 linux-firmware 的 amdgpu 目录下（与该脚本用的镜像同源），但清单里这一步没逐个下载验证过——组装时逐个核对，凑不齐 9 个 `install.sh --check` 会直接失败。

**替代捷径（推荐先走这条，工作量小得多）：**下载官方 `v0.4.0-alpha` 发布包，解开后只把其中 `PhxBringup.kext` 与 `PhxAccel.kext` 换成发你的 fork 版（或 §2.2 自编的），再跑 `install.sh`。理由（已核对）：`install.sh`、两个 `build.sh`、`phxboot.sh`、`kinst.sh`、`build_release.sh`、`docs/INSTALL.md`、`README.md` 在主线 `8aba872` 与 `hdmi-fb2` 尖 `50db1e1` 之间**完全没有差异**（`git diff` 已核对，只有 `Makefile` 多了一个测试）；且第 0 步不依赖的 fork 改动（`phxfb2` 第二帧缓冲、休眠、功耗）默认都是关闭的。发布包里的 `mesboot` 是主线版，fork 的 `mesboot.c` 改动（SDMA opt-in，默认关闭）不影响第 0 步。

⚠️ **install.sh 必须用包里的那份，别从 main 现拉**：当前 main 的 install.sh 与 v0.4.0 包里的已不同（2026-11-18 核对：main 把 PCI 列表里的 `1901` 换成了 `164e`，还多了 `launchctl enable system/local.phxboot` 两行）。捷径的"0 差异"只对主线 8aba872（v0.4.0 发布时的主线头）成立。

**核对（收到 kext 后先对 SHA，再往下走）：**

```sh
shasum -a 256 nullarc-hdmi-fb2-kexts.zip
# 应为 0b4b6c59a3c6f1a5553acf0a8774e9581f9a66356888721a5786cbfa5749b1fd
shasum -a 256 PhxBringup.kext/Contents/MacOS/PhxBringup
# 应为 dadc7cc625963aa69d02db68d50a897ce8fcf442759870f6cb8ae87d005c7a37
shasum -a 256 PhxAccel.kext/Contents/MacOS/PhxAccel
# 应为 f55a78d6b69879a63c3e2496ef8c21a4745e9b57b861b2c30f9c0c37f571877d
```

---

## 4. OpenCore 安装（`docs/INSTALL.md` §1 + `README.md` Installing）

```sh
# 1. 把 PhxBringup.kext 放进 EFI（用 §3 组装好的包里的那份）
#    目标：EFI/OC/Kexts/PhxBringup.kext

# 2. 把 SSDT-ALIB.aml 放进 EFI/OC/ACPI/，再在 config.plist → ACPI → Add 里加一条
#    （Path = SSDT-ALIB.aml，Enabled = true）——只拷进去不生效，OpenCore 只加载 Add 里列出的 SSDT
#    来源：hardware/acpi/SSDT-ALIB.aml（fork 里有，已核对存在）；上游 README 只写了"拷进去"，没提 Add 这一条，别漏了
#    第 0 步只接 HDMI 的话，它防的是 USB-C 电源事件，可以先不加；但加了就必须走 Add，不然等于没加
```

`config.plist` 改动（`docs/INSTALL.md` §1 原文）：

- **Kernel → Add**：加一条，`BundlePath` = `PhxBringup.kext`，`ExecutablePath` = `Contents/MacOS/PhxBringup`，`PlistPath` = `Contents/Info.plist`，`MinKernel` = `25.0.0`，`Enabled` = true。放在 Lilu 与 VirtualSMC **之后**。
- **NVRAM → Add → `7C436110-AB2A-4BBB-A880-FE41995C9F82`**：
  - `boot-args` 须包含：`amfi=0x80 amfi_get_out_of_my_way=0x1 -amfipassbeta phxdebug=1`（同时把 `AMFIPass.kext` 加进 Kernel → Add，见 `hardware/REQUIRED_KEXTS.md`）；
  - `csr-active-config` = `7F080000`；
  - 以上两键同时放进 **NVRAM → Delete**，新值才生效。
- **Misc → Security → SecureBootModel** = `Disabled`。

注意：

- `install.sh` 的检查只校验 `amfi=0x80`、`amfi_get_out_of_my_way=0x1`、`phxdebug=1` 三项，**不校验 `-amfipassbeta`**（`install/install.sh` checks 段的循环只列这三项）。但 `INSTALL.md` 要求写 `-amfipassbeta`，照写，别省。
- `amfi=0x80` 是全系统关库验证：FairPlay 视频（Apple TV+、Safari 看 Netflix）会坏，有些验签名的应用可能拒绝运行（`INSTALL.md` §1 原文）。
- 重启后验证 PhxBringup 已加载（`INSTALL.md` §1）：

```sh
ioreg -l | grep phx-build
# 必须打印一行，否则先别往下走
```

---

## 5. 运行安装器（`docs/INSTALL.md` §2 + `install/install.sh`）

在解开的发布包目录里（§3 组装好的那份），用 sudo 跑：

```sh
sudo ./install.sh --check          # 只检查，什么都不改
sudo ./install.sh --windowserver   # 安装；从第二次启动起 WindowServer 走 GPU
```

`install.sh` 的检查项（checks 段，`--check` 会逐项打 ✓/✗）：

1. 必须用 sudo 跑；
2. macOS 26.*；
3. boot-arg 里有 `amfi=0x80`、`amfi_get_out_of_my_way=0x1`、`phxdebug=1`；
4. SIP 已放宽（`csrutil status` 不是 enabled，对应 `csr-active-config 7F080000`）；
5. `ioreg` 里有 `phx-build`（PhxBringup.kext 已加载）；
6. GPU PCI id 在 `1002:15bf/15c8/1900/1901` 之列（找不到只警告，不拦截）；
7. 包文件齐全（§3 列的那份清单）；
8. `gpu/fw/*.bin` 至少 9 个。

装到哪里（`install.sh` 与 `INSTALL.md` 的表格）：

| 内容 | 位置 |
|---|---|
| 开机 GPU 带起来（`phxboot.sh`、脚本、`phxctl`、`mesboot`、固件） | `/Users/Shared/gpu` |
| 每启动跑一次的 LaunchDaemon | `/Library/LaunchDaemons/local.phxboot.plist` |
| `PhxAccel.kext`，编进 AuxKC | `/Library/Extensions`（经 `kinst.sh` 用 `kmutil create -n aux` 发布，旧 KC 备份在 `/Users/Shared/gpu/backup`） |
| Metal 插件 `NVMTLDriver.bundle` 与 Mesa RADV `nvmtl/` | `/System/Library/CoreServices/CoreTypes.bundle/Contents/Library/PhxGPU`，`/Library/GPUBundles` 下是符号链接 |

- 不加 `--windowserver` 时，只有应用能用 GPU 的 Metal，WindowServer 继续用 CPU 画桌面；第 0 步必须用 `--windowserver`。
- **装完先开自动登录**（系统设置 → 用户与群组），再重启：GPU 路径下登录窗口可能黑屏（`INSTALL.md` §3 原文）。

---

## 6. 第 0 步开机流程（计划 §5.1）

```sh
# 开机前：只接一台显示器，先接 HDMI（USB-C 后面再说；HDMI 路径没有 EC/PD 参与，更简单）
```

**握手（二选一，区别写清楚）：**

- **A（计划 §5.1 原定，手动设）：开机前跑一次**

```sh
sudo nvram phxfb-boot=1
```

  - 作用：PhxAMDFB 在 `start()` 里读到这个 NVRAM 变量才接管屏幕，用完一次就删掉（`src/phxaccel/accel/phxfb.cpp:817-895`，已核对原文：`takeBootArming()` 读变量、删变量并强制同步）。值不重要，驱动只认有无（`takeBootArming()` 只查 `copyProperty` 是不是空），`1` 和 `01` 效果一样。
  - 代价：跳过了安装器的 GPU 自检启动——如果这台机器的 GPU 带起来本身就有问题，黑屏时分不清是“带起来失败”还是“接管失败”。
- **B（更稳妥的默认流程，不手动设）：**装完 `--windowserver` 后直接重启两次。第一次启动是 GPU 自检启动（`phxboot.sh` 跑完带起来、确认没问题后，`phxboot.sh:91` 行 `[ -e $G/phxboot-m2 ] && nvram phxfb-boot=1` 会自动为下一次启动设好握手）；第二次启动 PhxAMDFB 才接管屏幕。代价是多一次重启；好处是不跳过自检。
- **C（NVRAM 持久性兜底，2026-11-18 #30 实证新增）**：如果 A/B 走完、重启后的启动日志一直是 `NVRAM phxfb-boot not set`，先别反复设——查变量是不是重启就丢：新启动后跑 `nvram -p | grep phxfb-boot`，空的就是没存住（#30 那台 UM790 Pro 就是这样：OpenCore 不同步 NVRAM，macOS 里设了重启必丢；而且 `phxboot.sh:91` 的 `nvram phxfb-boot=1 && …` 失败是静默的，`nv_commit` 还把 stderr 丢了，所以“设了没生效”没有任何报错）。解法：把 `phxfb-boot` 直接写进 config.plist → NVRAM → Add → `7C436110-AB2A-4BBB-A880-FE41995C9F82`（就是 boot-args 所在的那一项），值填 `01`（驱动只认有无，`1` 也一样）；已确认可用的写法是 Data 型 `<data>MDE=</data>`（base64 解开就是 `01`，#30 原文如此），OpenCore 每次引导都会重新注入。写完重启一次，再看日志有没有 `NVRAM phxfb-boot taken`。
  - 代价：`takeBootArming()` 本来是“一次一用”（启动时读到就删掉并强制同步，万一某次启动在接管后 panic，下次会回到固件帧缓冲）；写进 config.plist 等于每次启动都 armed，这个安全网没了。**逃生开关**：`phxfb.cpp:894` 在接管之前先查 boot-arg `-phxfboff`，加了就直接回固件帧缓冲（`start()` 返回 false，驱动根本不接管）。如果接管后黑屏/循环开不了机，在 OpenCore 的 boot-args 里临时加 `-phxfboff` 启动一次，进系统后再把 config.plist 里 Add 的 `phxfb-boot` 删掉。
- 建议：第 0 步先走 **B**；如果 B 的第二次启动没亮，再试 **A** 手动设一次，以区分“接管没触发”还是“接管失败”；如果 A 设完重启后日志还是 `not set`，走 **C**。
- 注意：不用 `--windowserver` 装的话，`install.sh` 会删掉 `phxfb-boot`（`install/install.sh:103`），握手永远立不起来——第 0 步务必用 `--windowserver` 装。

**开机与拍照：**

1. 重启（用 `sudo reboot` 或苹果菜单点重启，直接敲 `reboot` 会报 `Operation not permitted`）；每一步都拍照：OpenCore 菜单 → 啰嗦模式 → 登录界面（黑屏也要拍，注明停在哪一步）。
2. 预期（`INSTALL.md` §3）：第一次启动后 GPU 在开机后几分钟内起来；`--windowserver` 下第二次启动 WindowServer 走 GPU。
3. 进桌面后核对三项（`INSTALL.md` §3）：
   - `/var/log/phxboot.log` 末尾是 `done`；
   - 系统信息 → 图形卡/显示器显示 **AMD Radeon 780M**，Metal 支持：**Metal 3**；
   - `sysctl debug.nvaccel_iop_ok` 随画面变化增长（WindowServer 的帧由 GPU 画）。

```sh
tail -5 /var/log/phxboot.log
sysctl debug.nvaccel_iop_ok
```

---

## 7. 抓取清单（计划 §6.2；日志原文已在 fork 的 phxfb.cpp 里逐行核对）

**每轮必记（贴在日志开头）：**机器型号、macOS 版本、显示器型号、接的哪个口、boot-arg 清单。

**7.1 照片：**OpenCore 菜单 → 啰嗦模式 → 登录界面，各拍一张（黑屏也要拍，注明停在哪一步）。

**7.2 内核日志：**

```sh
log show --predicate 'process == "kernel"' --last boot | grep -i phx
tail -30 /var/log/phxboot.log
```

**7.3 重点看的 `PhxFB:` 行**（前缀 `PhxFB: `，`phxfb.cpp:29` 的 `PFBLOG` 定义；以下格式逐行核对过原文）：

| 日志行（格式） | 位置 | 意思 |
|---|---|---|
| `console: %ux%u row %u depth %lu at physical 0x%llx` | `phxfb.cpp:944` | GOP 画面找到了：分辨率、**每行字节数**（对照宽度算 pitch 是否对得上；RadeArc 在 8500G 上发现 pitch 字段 1024 对有效宽度 1920，原因未明——第 0 步要首先核对这个数） |
| `BAR index %u at 0x%llx: %u of %u words match the console (%u non-zero)` | `phxfb.cpp:932` | 画面在哪个 BAR 里 |
| `refresh: OTG0 %ux%u totals, %u frames in %llu us -> %u.%03u Hz` | `phxfb.cpp:321` | OTG0 在跑，刷新率测到了 |
| `refresh: OTG0 not running (ctl 0x%x, %u frames), 60 Hz assumed` | `phxfb.cpp:314` | OTG0 没在跑，默认 60 Hz（优雅降级，不失败） |
| `refresh: no MMIO view, 60 Hz assumed` | `phxfb.cpp:305` | 同上 |
| `panel published: %ux%u row %u at 0x%llx (vbl timer %s)` | `phxfb.cpp:989` | **head 0 接管成功** |
| `backlight: the PWM is not running (no MMIO view, or off), no brightness control` | `phxfb.cpp:959` | 预期出现，无害（迷你主机没有背光） |
| `NVRAM phxfb-boot taken (...)` | `phxfb.cpp:834` | 握手被取走了（接管触发了） |
| `NVRAM phxfb-boot not set: leaving the firmware framebuffer this boot` | `phxfb.cpp:895` | 握手没立起来（接管根本没触发） |
| `the console surface is in none of the GPU's BARs: leaving the firmware framebuffer` | `phxfb.cpp:937` | 接管失败：GOP 画面不在任何 BAR 里 |
| `no usable boot console (base 0x%lx %lux%lu depth %lu row %lu): leaving the firmware framebuffer` | `phxfb.cpp:908` | 接管失败：拿不到可用的开机控制台信息 |

**7.4 寄存器快照：**

```sh
# phxctl 在 /Users/Shared/gpu/phxctl/phxctl（install.sh 装过去的）
sudo /Users/Shared/gpu/phxctl/phxctl regs 0x53dc 0x53e4 0x36a3
sudo /Users/Shared/gpu/phxctl/phxctl gc
```

- `phxctl regs <dword>...` 接受**绝对 dword 偏移**，须 root（`tools/phxctl/phxctl.c` 第 4 行注释与 `main` 的 `regs` 分支已核对）；`phxctl gc` 打印已命名的 GC 状态寄存器表（`gc_regs[]`，`phxctl.c:102-120`）。
- ✅ **三个寄存器号已核实（2026-11-15 主代理复核，2026-11-18 评审再核）**：`0x53dc`（HDMI 热插拔）、`0x53e4`（DP 热插拔）、`0x36a3`（DMCUB `SCRATCH0`：2 = GOP 固件，3 = Linux DAL 固件）都是 dword 索引——`hdmi-fb2` 分支 `phxfb.cpp:330-338` 的注释与 `hwState()` 直接用它们（`r[0x53dc]`、`r[0x53e4]`、`r[0x36a3] == 3`），`src/amd/phx_dcn314_prog_regs.h:308` 也有 `kDmcubScratch0 = {0x036a3, ...}`。按清单里的命令直接用即可。**纠正**：上一版引的 `tools/hdmi/hdmi-on.sh`、`dp-on.sh`、`dmubboot.c` 在 `hdmi-fb2` @ 50db1e1 的 git 树里不存在（已逐项查过 376 个条目），出处已改成上面两个文件。

**7.5 ioreg：**

```sh
ioreg -l | grep phx-build            # PhxBringup 在位
ioreg -rc PhxAMDFB -d0               # 帧缓冲是否起来
ioreg -l | grep -i "PhxFBWidth\|PhxFBHeight\|PhxFBRefreshMilliHz"
```

- 属性名已核对：`PhxFBWidth` / `PhxFBHeight` / `PhxFBRefreshMilliHz` 是 kext 自己设的属性（`phxfb.cpp:289-290`），即 GOP 实际给的模式。
- `sysctl debug.phxfb1_hw`：只有 head 1 存在时才有；第 0 步只用 head 0，预期没有，不用管。

---

## 8. 结果判定与恢复手段

**通过（A 档，计划 §6.2）：**屏幕从 OpenCore 菜单到登录界面全程有画面，且登录后桌面正常显示。分辨率、睡眠、热插拔都不计入 A 档。

**四种可能结果与分支（计划 §5.1，推断）：**

| 结果 | 说明 | 下一步 |
|---|---|---|
| 亮屏 + 桌面正常 + 分辨率原生 | 最理想，A 档和 B 档一次过 | 只剩显示睡眠打磨（计划 §5.4） |
| 亮屏 + 桌面正常 + 分辨率偏低（如 1024×768） | head 0 接管成功，GOP 选的模式不好 | 计划 §5.2（模式修复） |
| 开机过程有画面，进桌面后黑屏 | 引导阶段 GOP 在工作，kext 接管后管线停了（可能原因：PhxBringup 上电过程动了 DCN 状态，或 GOP 用的不是管线 0） | 先排障（§7.4 的寄存器快照），再看计划 §5.3 后备路线 |
| 全程黑屏（连 OpenCore 菜单都没有） | 固件 GOP 根本没点亮外接，或点的口不对——这是固件/BIOS 问题，不是驱动问题 | 换口、换线、进 BIOS 看显示输出设置；与驱动无关 |

**恢复手段（`docs/INSTALL.md` "When something goes wrong"）：**

```sh
# 手动关 GPU（下次启动走固件帧缓冲）
sudo touch /Users/Shared/gpu/phxboot-off
# 再打开
sudo rm /Users/Shared/gpu/phxboot-off
# 卸载
sudo ./uninstall.sh
```

- 开不了机时：在 OC 里禁用 PhxBringup（或用备用 EFI U 盘启动），进系统后跑卸载器；卸载后还要从 OC 里删 PhxBringup.kext 与相关 boot-arg。
- 注意 `phxboot.sh` 自带保护：一次没跑完的启动会被下一轮检测到并自动关 GPU、建 `/Users/Shared/gpu/phxboot-off`；WindowServer 在 GPU 上 90 秒内崩 3 次会回固件帧缓冲并重启（`tools/phxctl/phxboot.sh` 头注释与 `m2_fail`）。

---

## 9. 待确认问题（文档没写清的，不猜）

1. **`install.sh` 不能直接从仓库检出运行。**它只认发布包布局（§3 的清单），而 `tools/release/build_release.sh` 只能从一台已装好的机器拼包，不能从源码凭空拼包。§3 的手动组装步骤是按 `install.sh` 的检查清单反推的，**未经实测**。替代捷径：官方 v0.4.0 发布包 + 只替换两个 kext（§3 末尾）。
2. **`phxctl` 没有仓库内构建命令。**`mesboot` 有（头注释里写明了命令），`phxctl.c` 里没有。须在 Mac 上验证编译方式，或暂用发布包里的现成 `phxctl`。第 0 步的寄存器快照依赖它，编不出会卡住 §7.4。
3. **`phxctl regs` 的三个寄存器号（0x53dc / 0x53e4 / 0x36a3）已核实。**主代理复核：它们是 dword 索引，`hdmi-fb2` 分支 `phxfb.cpp:330-338`（注释与 `hwState()`）与 `src/amd/phx_dcn314_prog_regs.h:308`（`kDmcubScratch0 = {0x036a3, ...}`）均直接使用，按 §7.4 的命令用即可。（2026-11-18 纠正：上一版引的 `tools/hdmi/hdmi-on.sh`、`dp-on.sh`、`dmubboot.c` 在该分支的 git 树里不存在，已删。）
4. **`NVMTLDriver.bundle` 的壳子没有源码构建路径。**源码只产出 `NVMTLDriver.new` 二进制；`plugdeploy.sh` 只往已有 bundle 里换二进制。须取自发布包（`inst_plugin.sh` 注释称其为 "NullMoth 2.4.1, unmodified"）。分支里检入的 `src/nmplugin/NVMTLDriver.new` 是已编好的二进制，来源未注明，别默认它与分支尖源码一致。
5. **Mesa RADV 构建最重且无一键脚本。**仓库里只有补丁与 `mesa-patches-phoenix/README.txt`（Mesa 基线 + Cobalt64 9 个补丁 + Phoenix 7 个补丁 + meson/ninja + 从源码编 glslangValidator + Python 3.14 venv）。可考虑先用发布包里的 `libvulkan_radeon.dylib`（见 §3 替代捷径）。
6. **固件缺 3 个文件——已核实都在上游（2026-11-15 主代理复核）。**`tools/fetch_test_firmware.sh` 只下 7 个，缺 `psp_13_0_4_toc.bin`、`gc_11_0_1_mes_2.bin`、`gc_11_0_1_mes1.bin`（`install.sh` 要求 `gpu/fw/*.bin` 至少 9 个）。三个文件在该脚本的首选镜像（GitLab `kernel-firmware/linux-firmware`，`/-/raw/main/amdgpu/`）上逐个测过，均为 HTTP 200，可直接下载：

```sh
base=https://gitlab.com/kernel-firmware/linux-firmware/-/raw/main/amdgpu
for f in psp_13_0_4_toc.bin gc_11_0_1_mes_2.bin gc_11_0_1_mes1.bin; do
  curl -fsSL -o ~/nullarc-pkg/gpu/fw/$f "$base/$f"
done
```

注意：GitHub 的 `torvalds/linux-firmware` 镜像上这三个文件查不到（404），是镜像残缺，别用它核对。
7. **无固定工具链版本。**仓库与 CI 均未钉 Xcode / CLT / MacKernelSDK 版本（CI 用 `macos-15` + 最新 main 的 MacKernelSDK；Mesa README 写 CLT 16.2、Tahoe 26.6.2）。构建机上的版本须记录在案。
8. **PhxSleepPolicy 与第 0 步无关。**它有自己的 `src/sleeppolicy/build.sh`，但 `install.sh` 不装它；休眠不在验收内，别为第 0 步编它。
9. **fork 的 CI 已能编 kext（2026-11-18 更新）。**`izerok/nullarc` 新增 `.github/workflows/build-kexts.yml`（commit fd51863），Actions run 44210948721 全绿：PhxBringup.kext + PhxAccel.kext 均已编出并验过（x86_64 Mach-O、adhoc 签名）。第 0 步不用在本地编，直接用发你的那份；§1–§2 仅作备用参考。原先的 `ci.yml` 只是 Linux 单元测试，不编 kext。
10. **用户机器信息（2026-11-18 已确认）：**KTHIN 科偲 K8 迷你电脑，Ryzen 7 H 255（80W），Oculink 口 + USB4，双 2.5G 网口（RTL8125），双 DDR5 槽（40GB，16+24），BIOS 2.09（2026-02-09），Wi-Fi AX200；显示器 KTC 27M1 Max，HDMI 3840×2160@160Hz；macOS 目标 26.7.3；现有 Windows 可动。双 M.2 是用户口述，硬件报告里只确认了一条 Lexar NM710 1TB，未逐项确认。仍未知：BIOS 里的显示输出设置、Linux Live USB 能否启动（只影响 §5.3 后备路线）。
11. **v0.4.0 的 NVMTLDriver.bundle + fork 的 PhxAccel 混用 ABI 未验。**#30 的成功是间接证据（他用的是 v0.4.0 整包）。如果 Metal 应用异常（闪退、花屏、Metal 3 不认），先换 fork 自编的 NVMTLDriver.new（§2.3 + plugdeploy.sh）或整包换 fork 版，再排障；别先动 head 0 的接管逻辑。

---

## 出处（核对过的文件）

- `README.md`（构建命令、安装步骤、组件表）；`README_RU.md`（构建段命令相同，已逐行对过）。
- `Makefile`（测试目标；`kext` 目标用 `tools/build_kext.py` 编脚手架）；`docs/INSTALL.md`（安装全流程、检查项、恢复手段）。
- `install/install.sh`（检查项、安装位置、phxboot-m2 逻辑）；`install/uninstall.sh`。
- `src/phxbringup/build.sh`、`src/phxaccel/build.sh`、`src/nmplugin/build.sh`（三条构建命令与产物路径）。
- `src/phxaccel/accel/phxfb.cpp`（`PFBLOG` 各行原文：:29、:305、:314、:321、:330-338（寄存器注释与 `hwState()`）、:834、:894（`-phxfboff` 逃生开关）、:895、:908、:932、:937、:944、:959、:989；属性 `PhxFBWidth/Height/RefreshMilliHz` 在 :289-290）。
- `tools/phxctl/phxboot.sh`（握手自动重设 :91、保护逻辑、日志位置）；`tools/phxctl/kinst.sh`（AuxKC 发布）；`tools/phxctl/plugdeploy.sh`（插件二进制装进已有 bundle）；`tools/phxctl/phxctl.c`（`regs` 用法：绝对 dword 偏移，须 root；`gc` 表在 :102-120）；`tools/phxctl/mesboot.c`（头注释第 2 行为构建命令）。
- `tools/release/build_release.sh`（发布包从已装好的机器拼包；`FW` 变量列出 9 个固件文件名）。
- `tools/fetch_test_firmware.sh`（只下 7 个固件文件）；`mesa-patches-phoenix/README.txt`（Mesa 构建命令与环境）。
- `hardware/REQUIRED_KEXTS.md`（Lilu、VirtualSMC、RestrictEvents、AMFIPass；Sample.plist）；`hardware/acpi/SSDT-ALIB.aml`（fork 里存在）。
- `.github/workflows/phxbringup-kext.yml`（CI 只编 PhxBringup.kext，macos-15 + 最新 MacKernelSDK main）。
- `git diff origin/main HEAD`（主线 `8aba872` 与 `hdmi-fb2` 尖 `50db1e1` 之间：`install.sh`、两个 `build.sh`、`phxboot.sh`、`kinst.sh`、`plugdeploy.sh`、`build_release.sh`、`INSTALL.md`、`README.md` 均无差异；只有 `Makefile` 多了一个测试目标 `test_phx_smu_pm`）。
- 计划：`~/workspace/goals/780m-nullarc/hidden_files/minipc-plan-2026-11-15.md` §5.1（第 0 步）、§6（测试计划与待提供输入）。
