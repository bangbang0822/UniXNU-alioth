# 第一次上板：固件进到 DXE 后期，在 isense 之后同步异常

2026-11-13 00:32。设备 `c0152a34`（Redmi K40 / `M2012K11AC`，8 GiB / 256 GiB，已解锁）。
设备仓库 `30021c11` + 本次记录提交；固件检出 `FFT-infra/uefi-alioth`。

这是本仓库保留的**第一份固件上板日志**。此前 firmware 区只有一句"the UEFI firmware has been booted on the board, but no device log of that boot was kept"。

## 1. 怎么上去的

```
./unixnu build firmware -D ../UniXNU-alioth --uefi ../uefi-alioth
#   → Build/aliothPkg/DEBUG_CLANGPDB/FV/SILICIUM_UEFI.fd，3,145,728 字节（= 0x300000，与 [uefi_fd] size 一致）
./unixnu bootimg -D ../UniXNU-alioth --uefi ../uefi-alioth \
  --dtb ../uefi-alioth/Resources/DTBs/alioth.dtb --ramdisk none -o boot-fw-only.img
#   → 2,543,616 字节，magic ANDROID!
#   → sha256 24ecf30d22cd084c5fc83d7adfb8a417748b908ae5f7176f92d3e84fe46b01d1
adb reboot bootloader
<raw fastboot client> --boot boot-fw-only.img
```

没有 payload、没有 Data seed、没有 trust cache——这是纯固件镜像。`fastboot boot` 只把镜像读进 RAM，**不写任何分区**。

Slots 保持原样：`current-slot: b`，两槽 `successful:yes`。

## 2. 屏幕上的日志（原文）

```
Project Silicium for Xiaomi Redmi K40
Firmware Version 4.0 Build at 08:05:59 on ... 2026

Loading DxeCore at 0x009AA23000 EntryPoint=0x009AA242F4
ERROR: CPU Vectors region not present in cfg file !
Rpmh Sleep callback registration failed, Status = 0x8000000000000003
PM0: 30, PM1: 32, PM2: 33, PM3: 33, PM4: 42,
UFS INQUIRY ID: MICRON MT256GARAT4U31 0102
UFS Boot LUN: 2
UFS Sleep callback registration failed, Status = 0x8000000000000003
Installed Fat filesystem on 998FDE18
Overriding PIL cfg by caller
pil-SPSS Mount partition return Not Found
pil-SPSS Failed to get MDT size
pil-SPSS Failed to load metadata
SPSSDxe: SPSSDxe_LoadSPSS ProcessPilImageExt = Load Error
InitializeRealTimeClock: using default timezone/daylight settings
PmcFgSram_ReadBattID: BATT_ID_2 = 98529
PmcSchgSetChargerDefaultSetting Setting ICL = 500 mA
PmcSchgInitialChargerCheckup APSD done status: 1, Port Type = 0
QcomChargerDxe: ChargerPlatform_Init Enabled WDOG.
UsbConfigLibOpenProtocols: PMI version (0x20)
UsbConfigInit: UsbConfigInit, not start on port: 0, mode 0
APC1 IDDQ WC 0 , L2:0, CPU[0]:175, CPU[1]:175, CPU[2]:175, CPU[3]:175
APC1 Total 700
LoadSys TIME 37ms / tsens TIME 50ms / scm TIME 63ms
Load CPU 0..3, GPU 0, NPU 0
...
isense TIME 444ms
ISENSE TOTAL TIME 456ms

Synchronous Exception at 0x00000000989E7C24
```

截图见 `2026-11-13-first-firmware-boot.jpg`。

## 3. 读出来的东西

**硬件识别是对的。** `UFS INQUIRY ID: MICRON MT256GARAT4U31 0102` 与 2026-11-12 用 adb/fastboot 采到的 256 GB（`userdata` 230,839.98 MiB，19 个分区合计 238.1 GiB）互相印证；PMIC BATT_ID、APC1 IDDQ、tsens、scm 都跑完了。**固件卷尺寸、DTB、boot.img 几何、刷写路径四项全部验证通过。**

**崩点是执行地址，不是数据地址。**（本节在 2026-11-14 00:40 更正，原文写的是"数据地址"。）`Silicon/Silicium/SiliciumPkg/Sec/AArch64/Exception.S` 的向量入口把 `elr_el1`/`elr_el2` 取到 `x1` 再传给 `SecCommonExceptionEntry`，所以 `ArchSec.c` 打印的那个值就是**故障 PC**。它落在 `Binaries/alioth/RawFiles/uefiplat.cfg` 登记的第 20 行区间里：

```
0x98900000, 0x03300000, "DXE Heap",  AddMem, SYS_MEM, SYS_MEM_CAP, Conv, WRITE_BACK_XN
```

即 `[0x98900000, 0x9BC00000)`，偏移 `0xe7c24`。DXE 驱动是从 FV（`0x9FC00000`）拷贝到分配内存里执行的，而分配内存正是这块 DXE Heap，所以这个 PC 说明**崩溃发生在某个已加载 DXE 驱动的代码里**，位置偏向堆区前段（约 950 KiB 处），即较早被加载的那批镜像之一。异常紧接 `ISENSE TOTAL TIME 456ms` 发生。

**崩溃与 payload 无关（2026-11-14 00:20 实测）。** 为排除"没有 ramdisk/payload 时 BDS 走空指针"这个假设，用同一份固件做了一张 `--ramdisk FILE` 的镜像（假 ramdisk 42 字节，sha256 `2d275950…`）并与 firmware-only 对照：

| 镜像 | `fastboot boot` | 屏幕结果 |
| --- | --- | --- |
| `boot-fw-only.img`（`--ramdisk none`） | OKAY | `Synchronous Exception at 0x00000000989E7C24` |
| `boot-fw-ramdisk.img`（假 ramdisk） | OKAY | **同一地址** |

两次都停在同一个 PC，所以这是固件自身在 DXE 里的确定性故障，与有没有 payload、与 UniXNU 内核都无关。修它不能等内核。

**USB 从未初始化。** `UsbConfigInit, not start on port: 0, mode 0`，这解释了为什么整轮上板在主机侧看不到任何 USB 设备——不是设备异常，是固件没起来。也意味着这一轮无法远程抓日志，只能看屏幕。

**电池低电量也会让 `fastboot boot` 被拒。** 2026-11-14 00:12 这个 bootloader 对两张镜像都返回 `boot -> FAIL`（客户端提示 `boot was refused`），`getvar:battery-voltage` 读回 `3400`。充到 `4055` 之后同一张镜像 `boot -> OKAY`。所以"boot 被拒"要先读电压，不要当成镜像问题。

## 4. 排除掉的怀疑：`CPU Vectors region not present`

这条报错最像元凶，但**排除了**：

- `Binaries/alioth/RawFiles/uefiplat.cfg:28` 有 `0x9FF8C000, 0x00001000, "CPU Vectors"`；
- 该文件 `:67` 写 `PlatConfigFileName = "uefiplatLA.cfg"`，而 `uefiplatLA.cfg` 在 **68 个声明了它的板子里一个都没有**，包括已知能跑的 cepheus。

所以这是这套 Mu-Silicium 的通用告警，不能当 alioth 的缺陷。

## 5. 仍然成立与新增的未完成项

- 崩因仍未定，但可观测性已经补上：`SecCommonExceptionEntry` 现在打印 `esr` / `far` / `el` / `sp` 四行信息，并用帧指针走一次最多 8 层的返回地址链（只打印落在固件可执行区间内的值）。下次上板一次就能区分"野指针读"（ESR 的数据中止位 + FAR）与"跳到坏代码"（非法指令/非法执行态），返回地址链用于对着 FV 的 map 定位到具体驱动。固件镜像 `boot-fw-esr.img`，sha256 `e4daa297…`（2026-11-14 01:15）。
- 屏幕是唯一输出通道：这台机器在 DXE 阶段没有 USB，`AdvancedLogDxe` 的缓冲也读不回来，所以每次上板都要人看屏幕/拍照。
- `pil-SPSS` 的三条失败（`Mount partition return Not Found` / `Failed to get MDT size` / `Failed to load metadata`）与 `SPSSDxe_LoadSPSS = Load Error` 是固件对 SPSS 分区的加载失败，当前不阻塞 DXE 推进，但同样未解决。
- `Rpmh` / `UFS` 的 sleep callback 注册均返回 `0x8000000000000003`（EFI_UNSUPPORTED），这与上一轮记录中的判断一致，未变化。
