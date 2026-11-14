# 2026-11-14: the first complete image built from a clean tree

Workflow `package`, run [43529232103](https://github.com/bangbang0822/UniXNU-alioth/actions/runs/43529232103),
`workflow_dispatch` on `a11c098`, `macos-26`, `payload=true`. Every one of the 28 steps
passed. This is the first run in which the whole chain — tree, verified-root policies,
bootstrap kernel, shipping kernel, firmware, payload and boot image — completed without
a hand-made artifact on the machine.

## What came out

| File | Bytes | sha256 |
| --- | ---: | --- |
| `boot.img` | 191,293,440 | `f201012d7a41477d10939f3d260e6be9bb03088bde857d8c78e6c44bc6364399` |
| `payload.img` (inside it) | 188,743,680 | `a24231e7a44ee5c62cc194628a4b859c78b2105cfc58866084855a8434b37be0` |
| `kernel.macho` (shipping) | 126,689,280 | `724da02089e89973e7cb3c38e18c778fc46ff8e9f77747d8318214bf4d355711` |
| `kernel.macho` (bootstrap) | 52,035,584 | `5fe8af111ce27313c31ba8537fca66163b31567dd3f52d90c994d2d9b611c53c` |

Geometry, measured on the downloaded file: `ANDROID!` magic, kernel region 2,546,907
bytes (BootShim plus the gzipped `SILICIUM_UEFI.fd`), ramdisk region 188,743,680 bytes
(the payload FAT, whose first sector carries an mtools MBR and which contains the
`NGPUADD2` addon). The K40's boot partition is 192 MiB (201,326,592 bytes), so the
image fits with 10,033,152 bytes to spare.

## What the run proves

The kernel links with all 29 archives, including the two verified-root ones and the
authenticated block transport, and the payload runtime is produced end to end:
Turnip 17,938,168 bytes, smoke shaders, the Rust bridge, the owner-signed Metal
bundle, the GPU addon, the trust bundle from `StaticTrustCache` plus
`Cryptex1,SystemTrustCache` with the addon's CDHashes appended, and a 224 GiB Data
seed.

## What it does not prove

Nothing here has run on the board. The only device-side result that exists is the
firmware crash of 2026-11-13, and a later measurement shows it is not about the
payload at all: a firmware-only image and one carrying a dummy ramdisk both stop at
`Synchronous Exception at 0x00000000989E7C24`. That address is the faulting PC (the
vector stub passes `elr_el1`/`elr_el2`), inside the DXE heap region the platform
declares, so the firmware fails in one of the earlier DXE images before any kernel
runs. Until that is fixed, this image cannot be exercised on the device.

## How it was made reproducible

Ten of the gaps this run closed were artifacts of one development machine rather than
of the repository: the two verified-root policies and their host library, the Metal
plugin's device sources, the Rust bridge archive, and the candidate kernel the two
archive builders bind against. Each now has a producer in the workflow, and the
verified-root tables and the Rust bridge are cached on their inputs, so a repeat costs
about twenty five minutes instead of an hour.
