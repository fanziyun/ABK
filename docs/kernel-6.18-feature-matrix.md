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
| KernelSU 变体 | `kernelsu_variant` = Official / SukiSU / BakaSU | ⚠️ | 各变体仓无 6.18 专属分支，靠内核 API 兼容；`setuid_hook.c`、`security_add_hooks(struct lsm_id)`、`netlink_kernel_cfg.cb_mutex` 三处 6.12 专项修复已扩到 6.18 |
| SUSFS | `enable_susfs` | ❌ | `susfs4ksu` 最高只有 `gki-android16-6.12`，没有 `gki-android17-6.18`；a17 工作流把 `enable_susfs` 固定为 `false`，直连 `build.yml` 打开会报错退出 |
| ZRAM 增强 | `use_zram` | ✅ | `CONFIG_ZRAM`/`CONFIG_ZSMALLOC` 内建 + 删 `modules.bzl` 里的 zram.ko/zsmalloc.ko，与 6.12 一致 |
| ZRAM 完整算法 | `zram_full_algo` | ✅ | 6.18 与 6.12 同为"新式 backend"，走 `CONFIG_ZRAM_BACKEND_{LZO,LZ4,LZ4HC,ZSTD,DEFLATE,842}` |
| ZRAM 自定义算法 | `zram_extra_algos` | ✅（受限） | lzo/lz4/lz4hc/deflate/842/zstd 可用；lz4k、lz4kd、lz4k_oplus、zstdp 会在解析阶段被剔除并 warning |
| legacy LZ4K 补丁栈 | `zram_full_algo`/`zram_extra_algos` 内的 lz4k 系列 | ❌ | `SukiSU_patch` 的 `other/zram/zram_patch` 只有 5.10/5.15/6.1/6.6，6.12 与 6.18 都要跳过 |
| zstdp 可选后端 | 已废弃 | ❌ 不参与构建 | `zstdp_wrapper.c` 的 scomp 回调签名与 6.12+ crypto API 不兼容；A/B 实测 6.12.23/38/58 全部同因失败，与 6.18 无关。工作流现在无条件把 `zstdp` 从算法列表移除并 warning，`setup_zstdp.sh` 不再触发 |
| BBG 防格机 | `use_bbg` | ❌（6.18） | CI 实测：`security/baseband-guard/blkdev_helper.c:107` 调用 `blkdev_get_no_open()`，该 API 在 6.13 已被移除；6.12 同参可编过 → 上游 BBG 需适配 6.13+ |
| DDK LSM | `use_ddk` | ⚠️ | 仓库内 `ddk/patches/xingguang-ddk` 无 6.18 版本目录，需实测 Kconfig/LSM 接口 |
| NTsync | `use_ntsync` | ✅（带告警） | 6.18 内核自带 `drivers/misc/ntsync.c` 与 UAPI，跳过 base 补丁；`kernel_patches` 没有 `ntsync_compat_android17-6.18.patch`，缺失时给 `::warning` 并按原生实现继续 |
| Networking 增强（IPSet + BBR） | `use_networking` | ⚠️ | ipset/bbrv3 补丁上下文对 6.18 未验证 |
| KPM | `use_kpm`（SukiSU/BakaSU） | ⚠️ | `SukiSU_patch/kpm/patch_linux` 直接 patch `Image`，6.18 二进制布局需实测；目标目录已加 android17 |
| Re-Kernel | `use_rekernel` | ⚠️ | 依赖外部驱动源码，需实测 |
| 虚拟化支持 | `virtualization_support=on` | ⚠️ | 复用 `kernel-6.12` 的 SYSVIPC kABI 补丁 + 6.12+ packed 字段 fallback；`init_ipc_ns`/`put_ipc_ns` 导出补齐也扩到 6.18 |
| 一加 8E 支持 | `supp_op` | ❌ 不适用 | 本次只接 GKI 线：oneplus 系列工作流与一加机型清单（最高 android16/6.12）一律不动，a17 工作流也不提供 `supp_op` 开关 |
| 自定义源码构建 | `source_mode=custom_git` + `source_kernel_version=6.18` | ✅ | 映射表已加 `6.18 -> android17`，允许 `os_patch_level=lts` |
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

## 已知缺口 / 后续待办

- `ntsync_compat_android17-6.18.patch` 缺失：可基于 `ntsync_compat_android16-6.12.patch` 修上下文后补进 `WildKernels/kernel_patches`。
- `susfs4ksu` 出 `gki-android17-6.18` 分支后，只需去掉 a17 的 `enable_susfs: false` 与 `build.yml` 的拦截，其余链路已就绪。
- BBG / DDK / Networking / KPM / Re-Kernel / 虚拟化 的 6.18 结论必须由真实 CI 回填写回本表。
- zstdp 已按废弃处理，不再参与任何内核线的构建；如需复活，必须先修 `zram/zstdp/zstdp_wrapper.c` 的 scomp 回调签名。
- 一加（oneplus）线不纳入 6.18，等有 A17 机型再评估。
