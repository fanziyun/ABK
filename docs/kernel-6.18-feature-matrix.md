# 内核 6.18（Android 17）功能可用性矩阵

本文记录 ABK 为 `android17-6.18` 新增构建链路后的功能可用性判断，作为 `.github/workflows/kernel-a17-6-18.yml`、`kernel-full-feature-matrix.yml` 的配套说明。

状态口径：

- ✅ 可用：静态链路完整，且与已在跑的 6.12 同路径（或上游分支/常量已核实）
- ⚠️ 待实测：工作流已接通，但依赖上游补丁/外部脚本对 6.18 的适配，需要真实构建回填结论
- ❌ 不可用：上游缺分支或补丁，工作流已显式拦截，不做静默降级

## 速查表

| 功能 | 输入开关 | 6.18 状态 | 依据 / 说明 |
|---|---|---|---|
| 内核源码 | `android_version=android17`、`kernel_version=6.18` | ✅ | AOSP `kernel/manifest` 存在 `common-android17-6.18`、`-2026-04`(6.18.20)、`-2026-06`(6.18.21)、`-2026-09`(6.18.32)、`-lts`(6.18.37) |
| 版本组合校验 | `sub_level` / `os_patch_level` | ✅ | `build.yml` 的 `KNOWN_KERNEL_PAIRS` 已登记 20/21/32 三条，非法组合会给出可用补丁级别 |
| GKI Kleaf/Bazel 构建 | — | ✅ | 与 6.12 同路径，`--lto=none`（6.12/6.18 都关 LTO），产物 `bazel-bin/common/kernel_aarch64/Image` |
| KernelSU 变体 | `kernelsu_variant` = Official / SukiSU / BakaSU | ✅ 三者均已实测 | BakaSU / Official / SukiSU 在 6.18.20、21、32 上各 3/3 通过；SukiSU 需给旧 pin 补 `mnt_namespace.seq` 的 6.18 门控（workflow 已内置），Official 的 KPM 按规则关闭 |
| SUSFS | `enable_susfs` | ❌ | `susfs4ksu` 最高只有 `gki-android16-6.12`，没有 `gki-android17-6.18`；a17 工作流把 `enable_susfs` 固定为 `false`，直连 `build.yml` 打开会报错退出 |
| ZRAM 增强 | `use_zram` | ✅（实测） | `CONFIG_ZRAM`/`CONFIG_ZSMALLOC` 内建；6.18 的模块清单在 `common/bazel/modules_private.bzl`（6.12 在 `common/modules.bzl`），两处都按 `$KERNEL_ROOT` 清理 |
| ZRAM 完整算法 | `zram_full_algo` | ✅（实测） | run `40890556268` 单版本验证：6.18 走 `CONFIG_ZRAM_BACKEND_{LZO,LZ4,LZ4HC,ZSTD,DEFLATE,842}`，配置步骤与编译均通过 |
| ZRAM 自定义算法 | `zram_extra_algos` | ✅（受限） | lzo/lz4/lz4hc/deflate/842/zstd 可用；lz4k、lz4kd、lz4k_oplus、zstdp 会在解析阶段被剔除并 warning |
| legacy LZ4K 补丁栈 | `zram_full_algo`/`zram_extra_algos` 内的 lz4k 系列 | ❌ | `SukiSU_patch` 的 `other/zram/zram_patch` 只有 5.10/5.15/6.1/6.6，6.12 与 6.18 都要跳过 |
| zstdp 可选后端 | 已废弃 | ❌ 不参与构建 | `zstdp_wrapper.c` 的 scomp 回调签名与 6.12+ crypto API 不兼容；A/B 实测 6.12.23/38/58 全部同因失败，与 6.18 无关。工作流现在无条件把 `zstdp` 从算法列表移除并 warning，`setup_zstdp.sh` 不再触发 |
| BBG 防格机 | `use_bbg` | ❌（6.18，暂不处理） | AOSP `android17-6.18` 的 `include/linux/blkdev.h` 已删除 `blkdev_get_no_open`/`blkdev_put_no_open`（6.12 仍在），而上游 Baseband-guard 的 `blkdev_helper.c:107/118`（AOSP `bd_meta_info` 分支）仍调用旧 API；上游 6.18 的 CI 覆盖的是 mainline LTS 的另一分支，不覆盖 AOSP GKI → 判定为上游缺口，本仓库暂不启用 |
| DDK LSM | `use_ddk` | ✅（实测） | `ddk/setup.sh` 在 6.18 上补丁全过，`CONFIG_XINGGUANG_DDK=y` 内建编译通过，三个子版本均成功 |
| NTsync | `use_ntsync` | ✅（带告警） | 6.18 内核自带 `drivers/misc/ntsync.c` 与 UAPI，跳过 base 补丁；`kernel_patches` 没有 `ntsync_compat_android17-6.18.patch`，缺失时给 `::warning` 并按原生实现继续 |
| Networking 增强（IPSet + BBR） | `use_networking` | ✅（实测） | IP_SET/IP6_NAT defconfig 注入 + bbrv3 补丁在 6.18 上应用与编译均通过 |
| KPM | `use_kpm`（SukiSU/BakaSU） | ✅（实测） | `SukiSU_patch/kpm/patch_linux` 对 6.18 Image 打补丁成功（`Image` → `oImage` 替换），BakaSU Stable 下三个子版本均通过 |
| Re-Kernel | `use_rekernel` | ✅（实测） | 外部 Re-Kernel 源码在 6.18 上集成并编译通过 |
| 虚拟化支持 | `virtualization_support=on` | ✅（实测） | 复用 `kernel-6.12` 的 SYSVIPC kABI 补丁 + 6.12+ packed 字段 fallback，`init_ipc_ns`/`put_ipc_ns` 导出补齐在 6.18 上编译通过 |
| 一加 8E 支持 | `supp_op` | ❌ 不适用 | 本次只接 GKI 线：oneplus 系列工作流与一加机型清单（最高 android16/6.12）一律不动，a17 工作流也不提供 `supp_op` 开关 |
| 自定义内核选项 | `custom_kernel_options` | ✅（实测） | run `40890556268` 传 `CONFIG_ZRAM_DEF_COMP_LZ4HC=y`，「配置内核选项」「应用自定义内核选项」两步均 success |
| 自定义源码构建 | `source_mode=custom_git` + `source_kernel_version=6.18` | ✅（静态） | 映射表已加 `6.18 -> android17`，允许 `os_patch_level=lts`；未跑真实私有仓 |
| 自定义外部模块 | `custom_external_modules` | ⚠️ 未测 | 需要外部模块仓；链路与 6.12 共用 |
| 辅助产物上传 | `upload_aux_artifacts` | ⚠️ 未测 | 仅 full-feature matrix / build.yml 暴露该开关，kernel-custom 入口没有 |
| Boot 镜像 / AnyKernel3 | — | ✅ | 与 6.12 同一打包链路（`boot.img`/`boot-gz.img`/`boot-lz4.img` + AVB footer） |
| KMI 标签 | — | ✅ | AOSP `android17-6.18` 的 `bazel/constants.scl` 里 `KMI_GENERATION=5`，本地版本串为 `6.18.x-android17-5-g<hash>-<bid>` |
| PR 抽样编译 | `kernel-pr-sampling.yml` | ✅ | 候选池 `sample-kernel-versions.py` 已加 6.18 三条，与 a17 matrix 严格一致（`cli/tests/test_kernel_random_sampling.py` 交叉校验） |

## 怎么实测

单版本：`Actions → 内核构建 - Android 17 (6.18) → Run workflow`，勾选要验的功能（如 `use_zram=true` + `zram_extra_algos=lz4,zstd`）。

批量功能矩阵：`kernel-full-feature-matrix.yml`，`android_version=android17`、`kernel_version=6.18`，一次跑完 ZRAM/BBG/DDK/NTsync/Networking/KPM/Re-Kernel/虚拟化的组合。

观察点：

1. 日志应出现 `内核版本组合校验通过: 6.18.20-android17-2026-04`（或 21/32）。
2. `strings bazel-bin/common/kernel_aarch64/Image | grep 'Linux version'` 应输出 `6.18.x-android17-5-g...-ab...`。
3. ZRAM：刷入后 `cat /sys/block/zram0/comp_algorithm`，方括号内为当前算法，列表应含已启用的 backend。
4. KPM：patch 成功后 `Image` 被 `oImage` 替换，日志无 `KPM image patch failed`。
5. SUSFS：a17 工作流里没有该开关；若手工调 `build.yml` 传 `enable_susfs=true`，应在校验步骤即失败（预期行为）。

## CI 实测记录（2026-10-29）

| 轮次 | 对象 | 结果 | 结论 |
|---|---|---|---|
| 1 | 6.18 全功能 | 失败于「构建信息摘要」 | zram 算法过滤命令转义写错 → 已修 |
| 2 | 6.18 全功能 + 基线 | 失败于编译期 bazel 解析 | 6.18 把 `kmi_symbol_list_strict_mode` 写成多行 select，按行 sed 砍首行 → 已改为整块删除 |
| 3 | 6.12 对照（同参） | 三个子版本全失败 | `zstdp_wrapper.c` scomp 回调类型不兼容 → zstdp 判定为废弃，已从构建剔除 |
| 4 | 6.18 单版本 | 失败于 BBG | `blkdev_get_no_open` 在 6.13 被移除 → BBG 在 6.18 不可用 |
| 5 | 6.18 单版本（BBG 关） | 失败于模块 staging | 内建 zram 后 kleaf 仍要求 zram.ko；6.18 的模块清单在 `bazel/modules_private.bzl`，且脚本需按 `$KERNEL_ROOT` 定位 → 已修 |
| 6 | 6.18 单版本（6.18.32 / 2026-09） | ✅ 全链路成功 | run `40861562304`：编译、KPM、Boot 镜像、Bundle、artifact 全部成功 |
| 7 | 6.18 矩阵（20 / 21 / 32，BakaSU） | ✅ 三条全绿 | run `40865373665`：三个子版本的编译、KPM、打包、上传均 success |
| 8 | 6.18 矩阵（Official，KPM 关） | ✅ 三条全绿 | run `40875146331` |
| 9 | 6.18 矩阵（SukiSU，首轮） | ❌ 三条全失败 | `mnt_namespace.seq` 在 6.18 被移除，Stable pin 未适配 |
| 10 | 6.18 矩阵（SukiSU，补门控后） | ✅ 三条全绿 | run `40881944723` |
| 11 | 6.18 单版本（zram_full_algo + custom_kernel_options） | ✅ 全链路成功 | run `40890556268`：新式全算法 backend、内核选项注入、KPM、Boot、Bundle 均通过 |

> 结论：6.18 的 GKI 链路本身是通的；当前不可用的是 SUSFS（上游无分支）、BBG（用了 6.13 已删的 blkdev API）、zstdp（已废弃）。其余功能在 BakaSU Stable + 三个子版本上实测通过。

## 已知缺口 / 后续待办

- `ntsync_compat_android17-6.18.patch` 缺失：可基于 `ntsync_compat_android16-6.12.patch` 修上下文后补进 `WildKernels/kernel_patches`。
- `susfs4ksu` 出 `gki-android17-6.18` 分支后，只需去掉 a17 的 `enable_susfs: false` 与 `build.yml` 的拦截，其余链路已就绪。
- BBG / DDK / Networking / KPM / Re-Kernel / 虚拟化 的 6.18 结论必须由真实 CI 回填写回本表。
- zstdp 已按废弃处理，不再参与任何内核线的构建；如需复活，必须先修 `zram/zstdp/zstdp_wrapper.c` 的 scomp 回调签名。
- 一加（oneplus）线不纳入 6.18，等有 A17 机型再评估。
