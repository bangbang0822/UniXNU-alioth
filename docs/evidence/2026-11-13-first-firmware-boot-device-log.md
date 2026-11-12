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

**崩点是数据地址，不是代码地址。** `0x989E7C24` 落在 `Binaries/alioth/RawFiles/uefiplat.cfg` 自己登记的第 20 行区间里：

```
0x98900000, 0x03300000, "DXE Heap",  AddMem, SYS_MEM, SYS_MEM_CAP, Conv, WRITE_BACK_XN
```

即 `[0x98900000, 0x9BC00000)`，偏移 `0xe7c24`。异常在 `ISENSE TOTAL TIME 456ms` 之后立刻发生，说明是 isense 初始化之后解引用了一个堆上的坏指针。

**USB 从未初始化。** `UsbConfigInit, not start on port: 0, mode 0`，这解释了为什么整轮上板在主机侧看不到任何 USB 设备——不是设备异常，是固件没起来。也意味着这一轮无法远程抓日志，只能看屏幕。

## 4. 排除掉的怀疑：`CPU Vectors region not present`

这条报错最像元凶，但**排除了**：

- `Binaries/alioth/RawFiles/uefiplat.cfg:28` 有 `0x9FF8C000, 0x00001000, "CPU Vectors"`；
- 该文件 `:67` 写 `PlatConfigFileName = "uefiplatLA.cfg"`，而 `uefiplatLA.cfg` 在 **68 个声明了它的板子里一个都没有**，包括已知能跑的 cepheus。

所以这是这套 Mu-Silicium 的通用告警，不能当 alioth 的缺陷。

## 5. 仍然成立与新增的未完成项

- 崩因未定：只知道是 DXE Heap 上的野指针，需要能看 PC 的日志（串口或 Advanced Logger）才能定位到模块。
- `pil-SPSS` 的三条失败（`Mount partition return Not Found` / `Failed to get MDT size` / `Failed to load metadata`）与 `SPSSDxe_LoadSPSS = Load Error` 是固件对 SPSS 分区的加载失败，当前不阻塞 DXE 推进，但同样未解决。
- `Rpmh` / `UFS` 的 sleep callback 注册均返回 `0x8000000000000003`（EFI_UNSUPPORTED），这与上一轮记录中的判断一致，未变化。
