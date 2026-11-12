# 上板单元的身份与 stock 固件版本：`c0152a34` 实测

2026-11-12。设备仓库 `30021c11`（改动前）。`adb` 只读采集，加一次 `fastboot getvar:all`；不刷写、不 dump 任何分区、不改槽。

## 1. 为什么记这一页

`porting.toml` 的 firmware 区此前写着：这台手机的 stock 固件（XBL/ABL、modem、ADSP 及其余 DSP 镜像）在本仓库里没有任何版本记录，也没有任何 dump，是"未完成的输入而不是已完成的输入"。本次用一台真实上板单元补上了其中可读的部分，并把仍然读不到的部分指名写清。

## 2. 单元身份

| 项 | 值 | 来源 |
| --- | --- | --- |
| 型号 | `M2012K11AC` / `alioth` / Redmi | `ro.product.model` |
| 序列号 | `c0152a34` | `ro.serialno` |
| SoC | SM8250（kona），`variant: SM8 UFS`，`hw-revision: 20001` | fastboot |
| `cpuid` | `0x701f5f43` | fastboot |
| 内存 | `MemTotal` 7,701,708 kB ≈ 7.34 GiB → 8 GiB SKU | `/proc/meminfo` |
| 存储 | 119 个分区，合计 238.1 GiB → 256 GB SKU | fastboot `partition-size:*` |
| 引导锁 | `unlocked: yes`、`verifiedbootstate=orange`、`ro.boot.vbmeta.device_state=unlocked` | fastboot / props |
| 当前槽 | `current-slot: b`，`slot-count: 2`，两槽 `successful:yes`、`retry 6/6` | fastboot |
| `vbmeta` | avb 1.1，digest `ab227220b6b7b2029d534d9e10e89f43796a2cbc78a2719bad47661ca6730f7a`，size 7808 | props |
| 其它 | `anti: 1`、`hwversion 9.9.0`、`bootreason reboot,factory_reset` | props / fastboot |

`cpuid 0x701f5f43` 是这台机器的真实芯片号：fastboot 的 `token` 解码为 `…alioth\x02\x04\x70\x1f\x5f\x43`，`alioth` 和该值在同一段里。`package.yml` 要求 secret `UNIXNU_UNIQUE_CHIP_ID` 恰好是 8 位十六进制（`^[0-9a-fA-F]{8}$`），与本值同格式，所以它是 `--unique-chip-id` 的首选候选；但 `procedure` 里用过 16 位的 `0xe835c44a614a1914`，说明 `payload.py` 也接受别的宽度，等价性需要一次构建来验证，不能从格式相同就直接断言。

## 3. stock 固件

| 项 | 值 |
| --- | --- |
| Android | 13（SDK 33），vendor SDK 30 |
| 版本 | HyperOS `OS1.0` / MIUI `V816`（code 816） |
| 构建 | `V816.0.10.0.TKHCNXM` |
| fingerprint | `Redmi/alioth/alioth:13/TKQ1.221114.001/V816.0.10.0.TKHCNXM:user/release-keys` |
| 分区级 incremental | odm / product / system / system_ext / vendor 全部 `V816.0.10.0.TKHCNXM` |
| 基线 | `ro.build.version.base_os = V816.0.6.0.TKHCNXM`（b 槽由 0.6.0 OTA 上来） |
| 安全补丁 | system `2024-03-01`，vendor `2024-04-01` |
| 构建时间 | `ro.build.date.utc` 1761736488 = 2025-10-29 |
| modem | `gsm.version.baseband = 2.5.c1-50.1a-14515.23-0527_1230_d46c7e8c13` |
| vendor 内核 | `Linux 4.19.157-perf-g92c089fc2d37`，2025-10-29 11:41:46 UTC，clang 10.0.7 (Android NDK) |

### 3.1 系统串和 vendor 分区对不上

同一次采集里两组字符串互相矛盾，记录时必须写明取哪一组：

```
ro.build.display.id          = TKQ1.221114.001 test-keys
ro.build.version.incremental = V14.0.1.0.TSZCNSZ          ← 另一个设备码的 MIUI 14 增量
ro.vendor.build.fingerprint  = Redmi/alioth/alioth:13/RKQ1.211001.001/V816.0.10.0.TKHCNXM
ro.build.version.base_os     = V816.0.6.0.TKHCNXM
ro.build.date.utc            = 1761736488                 ← 2025-10-29
```

`safety patch`、`odm/product/system/vendor` 的 incremental 和 vendor fingerprint 都指向 alioth 的 `V816.0.10.0.TKHCNXM`，但系统侧显示的是 `test-keys` 和一个非 alioth 的增量号，构建时间还比那个 HyperOS 版本晚一年。结论是：**本机跑的 system 镜像不是它自己 vendor 分区所署名的那份 stock 构建**，因此凡是描述板级固件的输入，只能取 vendor 分区那一组。

### 3.2 仍然读不到

- **XBL/ABL 版本串**：这个 bootloader 的 `fastboot getvar version-bootloader` 返回空串。
- **ADSP / CDSP / DSP 镜像版本**：`/vendor/firmware_mnt`、`/vendor/dsp`、`/vendor/firmware` 在非 root shell 下全部 `Permission denied`，且没有 root。
- **分区镜像本身**：一个都没 dump，本仓库也不保留任何固件分区副本。

补这两项需要 root shell 或镜像 dump（`dd`/9008），不是 `getprop` 能解决的。

## 4. 分区几何（fastboot `getvar:all`）

`system`、`vendor`、`product`、`odm`、`system_ext` 都不是独立分区，都在 `super` 内；本板**没有** `cache`、没有 `preload`。

| 分区 | 字节 | MiB |
| --- | ---: | ---: |
| `userdata` | 242,053,263,360 | 230,839.98 |
| `super` | 9,126,805,504 | 8,704 |
| `cust` | 1,073,741,824 | 1,024 |
| `modem_a` / `modem_b` | 469,762,048 | 448 |
| `exaid` | 402,653,184 | 384 |
| `boot_a` / `boot_b` | 201,326,592 | 192 |
| `rescue` / `rawdump` | 134,217,728 | 128 |
| `vendor_boot_a` / `_b` / `minidump` | 100,663,296 | 96 |
| `logo` / `dsp_a` / `dsp_b` / `persist` / `persistbak` / `logdump` | 67,108,864 | 64 |
| `dtbo` / `mdtp` / `vm-linux` / `splash` / `mtdblk` / `spunvm` | 33,554,432 | 32 |
| `metadata` / `devinfo` / `oops` | 16,777,216 | 16 |
| `xbl_a` / `xbl_b` / `hyp_a` / `hyp_b` | 8,388,608 | 8 |
| `xbl_config_a` / `_b` | 8,343,552 | 7.96 |
| `tz_a` / `tz_b` | 4,194,304 | 4 |
| `misc` | 4,194,304 | 4 |
| `abl_a` / `abl_b` / `uefisecapp_a` / `_b` | 2,097,152 | 2 |
| `keymaster_a` / `_b` / `bluetooth_a` / `_b` | 1,048,576 | 1 |
| `aop_a` / `aop_b` | 524,288 | 0.5 |
| `vbmeta*` | 131,072 | 0.12 |

`boot` 每槽 192 MiB，与上游"payload 打进 boot 镜像、`fastboot flash boot_b`"的做法相容。`max-download-size` 805,306,368 (768 MiB)。

## 5. 硬件清点（与移植直接相关）

| 子系统 | 实测 |
| --- | --- |
| 存储 | `1d84000.ufshc`，`ro.boot.boot_devices = soc/1d84000.ufshc` |
| GPU | `3d00000.qcom,kgsl-3d0` + `3da0000.qcom,kgsl-smmu`（Adreno 650） |
| 显示 | `ae00000.qcom,mdss_mdp`，`card0-DSI-1`，1080×2400，440 dpi，60/90/120 Hz，HDR 类型 1–4，峰值 420 nits、HBM 900 nits |
| 触控 | 活动设备 `fts_ts`，`CONFIG_TOUCHSCREEN_FOCALTECH=y`，SPI；`CONFIG_TOUCHSCREEN_SYNAPTICS_TCM=y` 同时编入但未实例化。与上游 FocalTech FT5652 一致 |
| 触觉 | `aw8697_haptic`，`CONFIG_INPUT_AW8697_HAPTIC=y` |
| 光感 | `tcs3701`（ams AG） |
| Wi-Fi | `b0000000.qcom,cnss-qca6390`；`1c00000.qcom,pcie`、`1c10000.qcom,pcie` |
| IPA | `1e00000.qcom,ipa` |
| 控制台 | `ttyMSM0` |
| USB | `a600000.dwc3` |
| 相机 | imx582 wide / imx355 ultra / s5k5e9yx04 macro / s5k3t2 front（`persist.vendor.camera.*.info`） |
| 指纹 | `ro.boot.fpsensor = fpc` |

## 6. 采集方式（含一处工具坑）

```
adb shell getprop                    # 46 KB，全量
adb shell cat /proc/meminfo /proc/cpuinfo /proc/version
adb shell ls -l /dev/block/by-name/  # 119 个分区到 sda/sdd/sde/sdf 的映射
adb pull /proc/config.gz             # 173 KB 解压后的内核配置
adb reboot bootloader                # ← 只为了 getvar:all
<raw fastboot client> getvar:all     # 273 行
<raw fastboot client> reboot
```

**坑**：一旦给 `VID_18D1&PID_D00D` 绑了 libwdi 生成的 WinUSB 驱动（`usb_driver/Android.inf`，`pnputil /add-driver ... /install`），SDK 的 `fastboot.exe`（37.0.1）就永远停在 `< waiting for any device >`，因为这个 INF 把自己的 `DeviceInterfaceGUIDs` 写成 `{8462A650-C1FA-478E-BD89-1D2A1D8DC9E4}`，而不是 Android Bootloader Interface 的 `{f72fe0d4-cbcb-407d-8814-9ed673d0dd6b}`。绕法是直接用 pyusb 讲 fastboot 协议（`k40-forensics/fastboot_raw.py`），端点 = interface 0 的 `0x01`/`0x81`。设备本身是健康的，`Status: OK` 且 `adb` 恢复正常。

## 7. 原始采集

`k40-forensics/raw-20261112-210056/`：`getprop-all.txt`、`getprop-flat.txt`、`summary.txt`、`partitions.json`、`fastboot-getvar-all.txt`、`fastboot-partitions.json`、`kernel-config.txt`、各 `proc-*` 与 `sys-*` 原始输出。脚本 `k40-forensics/collect-k40.ps1` 可对任意单元重跑。
