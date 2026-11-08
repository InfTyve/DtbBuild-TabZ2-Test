# MT6765 内核云端构建（Vestel V TAB Z2 / EKVATOR）

把内核源码丢给 GitHub Actions 编译，产出可直接 `fastboot flash boot` 的 `boot.img`。

不需要本地装工具链，不需要高性能电脑。标准 runner（4 核 / 16 GB）约 30 分钟出包，公开仓库免费。

---

## 快速开始

1. 新建一个 GitHub 仓库，把本目录所有内容推上去（**公开仓库才免费用 Actions**）
2. 打开 `Actions` → `Build MT6765 Kernel` → `Run workflow`
3. 默认参数即可直接跑；跑完在页面底部 `Artifacts` 下载 `boot.img`

第一次建议先用默认的 Moto G22 源码树跑通流程，确认工具链和配置没问题，再换其他源码树。

---

## 仓库结构

```
.
├── .github/workflows/build.yml   # 构建流水线
├── configs/
│   ├── ekvator.config            # 完整内核配置（5402 行，从原厂 boot.img 的 IKCONFIG 提取，推荐）
│   └── ekvator_defconfig         # 精简版（4310 行，供 make xxx_defconfig 使用）
├── prebuilt/
│   ├── dtb                       # 本机设备树 107,563 字节（勿替换）
│   └── ramdisk.gz                # 本机 ramdisk（含 Magisk 补丁）
└── scripts/
    └── pack_bootimg.py           # boot.img v2 打包器，已通过原厂镜像 round-trip 验证
```

`pack_bootimg.py` 的验证结果：用原厂 kernel / ramdisk / dtb 重新打包，
与原厂 `boot.img` **逐字节完全一致，包括 header 里的 SHA1 id 字段**。
所以它的输出可以直接刷。

---

## 输入参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `kernel_repo` | Moto G22 开源树 | 内核源码地址 |
| `kernel_branch` | `main` | 分支或 tag |
| `config_file` | `ekvator.config` | `configs/` 下的文件名 |
| `clang_repo` | LineageOS r383902 | 原厂用的就是 clang 11.0.1，换版本容易踩坑 |
| `keep_vermagic_clean` | 开 | 关闭 `LOCALVERSION_AUTO`，否则 vermagic 会变成 `4.19.x-g<hash>` |
| `build_ccache` | 开 | 第二次构建快 2–3 倍 |

## 产物

- `boot.img` — 32 MiB 内对齐，可直接 `fastboot flash boot boot.img`
- `Image.gz` / `Image` — 裸内核，想自己用 magiskboot 打包时用

---

## 四个必须知道的问题

### 1. vermagic 一定会变，除非你用 4.19.127 源码

原厂：

```
4.19.127 SMP preempt mod_unload modversions aarch64
```

现成源码树的版本：Moto G22 是 4.19.191，Redmi 9A 项目是 4.19.325。
无论用哪个，编译出来的 vermagic 都不同，原厂的 `.ko` 会拒绝加载。

原厂只有 9 个 `=m` 模块，其中真正重要的是：

```
CONFIG_BATTERY_MT6359=m
CONFIG_SND_SOC_MT6359_ACCDET=m
```

**对策：把这两个改成 `=y` 内建**，消除对 `.ko` 的依赖。改完直接
`grep -E "BATTERY_MT6359|MT6359_ACCDET" configs/ekvator.config` 确认。

### 2. DTB 不要动

`prebuilt/dtb` 是 Vestel 自己的面板 / 触摸 / PMIC 参数，面板旋转角度是
`CONFIG_MTK_LCM_PHYSICAL_ROTATION="270"`（横屏平板）。换成别家的 DTB 直接黑屏。

所以构建流程**不编译 DTB**，只用这个二进制打包。

### 3. boot 分区 32 MiB，别撑爆

原厂：kernel 14.6 MB + ramdisk 1.3 MB + dtb 0.1 MB ≈ 16.2 MB，余量约 15 MB。

换成 4.19.325 加各种特性后体积会涨。**保持 `CONFIG_DEBUG_INFO=n`**，
否则光调试符号就能多出 100 MB。当前配置里 `CONFIG_DEBUG_INFO=y`，
第一次编译如果超 32 MB，把它关掉。

### 4. clang 版本别乱换

原厂 clang 11.0.1（r383902）。用更新的 clang 编 4.19 会遇到
`-fno-common`、integrated assembler 不完整等问题，需要额外 `KCFLAGS`。
流水线里已经加了 `KCFLAGS="-Wno-error"` 兜底，但优先用原厂同版本。

---

## 常见故障

| 现象 | 原因 | 处理 |
|---|---|---|
| `olddefconfig` 后配置项大量消失 | 源码树版本和配置不匹配 | 正常，olddefconfig 会补默认值；看日志确认关键项还在 |
| 编译报 `multiple definition` | 用了新 GCC | 确认走的是 clang，`CC="ccache clang"` |
| 生成的 boot.img 刷了不开机 | vermagic 不匹配导致 vendor 模块拒绝加载，或 DTB 被换 | 改内建、检查 dtb |
| 刷了黑屏 | DTB 用错，或内核 early boot 崩溃 | 用 `prebuilt/dtb`，先只换内核不换 ramdisk 试 |
| 分区写不下 | 内核太大 | 关 `DEBUG_INFO`，关 `KALLSYMS_EXTRA_PASS` |

---

## 关于 Android 16

这个仓库只解决"编译内核"这一件事。**编出内核 ≠ 能上 Android 16。**

Android 16 需要完整 ROM，缺的两块：

| 组件 | 来源 |
|---|---|
| device tree | `AOSPBuildHall/android_device_xiaomi_blossom`（Redmi 9A 家族，MT6765） |
| vendor blobs | 从你设备原厂固件提取，跑 `extract-files.sh` |

另外 Android 16 GSI 在这台设备上不可行：原厂 vendor 只有 HIDL
`composer@2.1~2.4`，而 Android 14 起 AOSP 已移除 HIDL composer 客户端，
只认 AIDL `composer3`，结果是系统能起来但永远黑屏。

---

## 许可

内核源码与配置来自 GPL-2.0 的 Linux 内核，本仓库的脚本与配置同样以
GPL-2.0 发布。`prebuilt/ramdisk.gz` 与 `prebuilt/dtb` 来自设备原厂固件。
