# 评审记录：移植上游 2026-11-09→13 的功能改动（路线 2）

- 日期：2026-11-19
- 评审者：独立子 agent `424e7cd5-5782-4338-bfcb-289e40119032`（全新上下文，未与作者会话共享）
- 对象：分支 `port-upstream`（从 `main`=`200e223` 拉出）的 17 个 path
- 结论：**CHANGES REQUESTED → APPROVE**

## 这一批做什么

把 `FFT-infra/UniXNU-alioth@e4f9243b`（2026-11-13）相对本分支的功能改动**逐文件取上游版本**移植过来，
只在 `board.toml` 里替换 `[payload] kernel_args`：

- 启动参数 47→50：五个去 `sheng-` 前缀（`vsync`/`gpu-addon`/`fb-bgra`/`fb-gpu`；
  `pmgr` 的改名早在本分支 `1377237` 已落地，本批只补读者）、`sheng-autoreset` 600→240、
  新增 `diag=1`、`log-gpu=1`、`metal-archive-fallback=1`；
- `package/BootLog.c`：遥测表从"只打印前 6 行"（全是内存计数）改为**打印全部 48 行**有文本的行，
  每 10 秒一次 —— 这是"shader 轮次读不到任何读数"的直接原因；
- `package/GpuProbe.cpp`、`LinkStubs.c`、`LegacyDisplayProbe.cpp`、`ShengPlatformPower.cpp`、
  `BootDiagExport.c`：参数改名与别名回退；
- `scripts/metallib.py`（新，暂无 recipe 调用）、`scripts/make-gpu-addon-alioth.sh`（对齐 pin 的打包器接口）；
- `configs/prime-list-ios.txt`（168 条本板 `.metallib` 路径）、`configs/wallpaper-bundle/**`、
  `patches/xnu/zzz-native-wallpaper-bundle.patch` + `series`；
- `drivers/AdrenoMetal`、`drivers/ShengHidInput` 子模块 gitlink 随动到上游 SHA。

## 第一轮：CHANGES REQUESTED

- **[BLOCKER]** `porting.toml` 的 `power` 区把一条 finding 写进了 `evidence` 数组（裸字符串），
  而记录校验器对每条 evidence 调 `item.get('kind')` → `Porting record` 直接崩。
  根因：`power` 区的 `findings` 是**单行数组**，插入脚本越过它。
- **[MAJOR]** 认为 `make-gpu-addon-alioth.sh` 传了打包器不认的 `--wallpaper-bundle`。
- **[MAJOR]** 子模块 gitlink 未随动，`diag`/`log-gpu`/`metal-archive-fallback` 与 BootLog 6→48 的
  实际效果无法核对。
- 4 条 MINOR（display finding 描述错文件、configuration finding 过度声明读者、命令行字节数过期、
  壁纸补丁 mount 偏移越界）+ 2 条 NOTE。

## 处置

| 项 | 处置 |
| --- | --- |
| BLOCKER（evidence 裸字符串） | 字符串移回 `findings`（现 6 条），`evidence` 恢复为唯一一条表；**实证**：`porting-record.py check` → `alioth: 15 structured areas checked`（此前 AttributeError） |
| MAJOR（`--wallpaper-bundle`） | **驳回**：拉 pin 原文核对，`FFT-infra/UniXNU@e0ee2ad9` 的 `make-gpu-addon.py` 含该 flag（用法串 + 解析，共 10 处命中），并按 `wallpaper-bundle/` 前缀打包；评审的 0 命中来自比 pin 旧的本地副本。评审随后**撤回**该 MAJOR 并自行复核确认 |
| MAJOR（gitlink） | 已 bump 到 `8459f908…` / `b5b925b8…`，`git diff up/main -- drivers/*` 为空；插件侧效果仍未验证，记录未写成已验证 |
| MINOR（display） | 改写：该文件带的是 `fb-gpu` 别名回退与 `surfaceWhite`，`dpu-scanout` 改名在参数表 |
| MINOR（configuration 过度声明） | 按读者逐个给证据：`gpu-addon`（base patch:301）、`pmgr`、`fb-gpu` 有读者；其余六个开关两边都无读者，写成"前瞻性" |
| MINOR（字节数） | 987 → **913**（实测 895→913） |
| MINOR（mount 偏移） | packaging finding 注明最远 `0xD1C` 仍在同一 4 KiB 页、顺序不重叠，余量降至 ~0x2E4 |
| NOTE（metallib.py 无调用者） | gpu finding 写明"为 addon 批次准备" |

## 第二轮：APPROVE，余两条措辞

作者在提交前按其意见收紧：

- "six names lost their sheng- prefix" → **five**，并写明 `pmgr` 的改名早于本批、本批只补读者；
- "those three really did run under names nothing looked for" → **two**（`gpu-addon`、`pmgr`）；
  明确 `fb-gpu` 原本两边配对，属正常改名，不得记成修复。

## 结论

代码侧与 `up/main` 逐字一致、`kernel_args` 50 条零差异、无删除无夹带、回滚点 `200e223`。
未验证项（插件二进制对三个新开关与 48 行遥测的实际效果）在记录里保持"未验证"状态。
