# 评审记录：G 路线批次 1（上游基线切换，已撤回）

> 页面标题与分支名（`baseline-upstream-bba3b578`）是历史名称：这一批最初想切到
> `UniXNU/UniXNU@bba3b578`，**第一轮评审的 BLOCKER 之后即撤回**（此后各轮评审的是缩减后的批次）；最终提交只做三件事——把两个 workflow 里
> 的 tools checkout 都钉到 `e0ee2ad9`，把 Review gate 写进 `AGENTS.md`，并把这次阻断的
> 记录与评审过程写进仓库。
> 本页按时间顺序保留各轮评审的结论与处置，不删除被否决的方案；各轮评审与处置节的标题说明它属于哪一轮。

- 日期：2026-11-18
- 各轮评审者（按时间，均为全新上下文的独立子 agent）：
  `e713285b-a3ac-4c7c-aa21-11f75236f488`、`efceeab2-2820-4546-a5d5-0c4c6a3e2f73`、
  `2a476f49-ba61-473d-a5f8-9cfeebbff3c8`、`3ef79e36-d06a-4ff4-903b-5bfe28b6c7ff`、
  `3cf4f28f-00e8-4d8e-b9c3-56fa3d35d9dd`（末轮为窄范围核验）
- 第一轮评审对象：`package.yml`、新增 `configs/components-extra.toml`、`porting.toml`、`AGENTS.md`
  （第一轮 findings 另对**未改动**的 `porting-record.yml` 提出 MINOR，指出它仍浮动在 `main`）
- 第二轮评审对象：缩减后的批次（`package.yml` 钉版本、**`porting-record.yml` 钉版本**、
  `AGENTS.md` Review gate、`porting.toml` finding、本页）
- 第三轮评审对象：同上，逐条核对第二轮 findings 是否修掉
- 各轮结论：**均为 CHANGES REQUESTED**；第一轮否决了切换本身，随后的各轮依次修正记录口径、
  剩余浮动与工具卫生，末轮只剩"正文里数轮次"这一处，故本页改为不再在正文计数。

## 第一轮评审：VERDICT 与 findings（原文摘要）

VERDICT: CHANGES REQUESTED

- **[BLOCKER]** `configs/components-extra.toml:26`（经 `package.yml` 注入）：`QcomXnuUsbDebug` 的
  `build[0]` 是 `{component}/scripts/build.py`，而上游 `bba3b578` 新增的 `kernel.preflight`
  （`scripts/unixnu/kernel.py:196-219（原文行号；准确范围是 196-222，见复核段）`，`cli.py:176-177` 每次 `build kernel` 调用）**不展开占位符**，
  直接把命令拼成 `<tree>/scripts/{component}/scripts/build.py`，该路径不可能存在；bootstrap 步骤
  第一次 `./unixnu build kernel` 会在编译前以 `Build inputs missing` 失败。fork 无 preflight，
  所以这是切换引入的回归。
- **[MAJOR]** `package.yml` 的 `repository: UniXNU/UniXNU` 没有 `ref`，实际是浮动 HEAD；
  supplement 只查存在不查重复，而 `unixnu` 对重复 id 直接抛错，上游一旦收下这两个组件就每次硬失败。
- **[MAJOR]** `porting.toml` 的 finding 把切换说成"只差两个组件"，但切换同时移动了输入：
  RamBlock `4be2d7e→00a6e5d`、`drivers/AdrenoMetal` gitlink `73d582f→8c7f950`、
  `gpu-metal-vulkan-inputs.json`、`patches/mesa` 23→24（新增
  `turnip-darwin-bo-finish-clears-under-lock.patch`），因此 2026-11-14 的镜像哈希不再对应同一输入。
- **[MINOR]** `porting-record.yml` 仍 checkout `FFT-infra/UniXNU@main`，与打包用的 catalog 不一致。
- **[MINOR]** "21 components" 不正确：include 17 + 依赖 3 = **20**。
- **[MINOR]** `AGENTS.md` 把评审记录指向仓库外路径，其他人无法照做；建议放进仓库内。
- **[MINOR]** 新增 `configs/components-extra.toml` 仍是未跟踪文件，`git commit -am` 会漏。
- **[MINOR]** `test -f` 失败无输出；`python3`/`tomllib` 未进 fail-fast。
- **[NOTE]** 高风险改动应先跑 `payload=false`，当时无 run 证据。
- **[NOTE]** 回滚点：本地 `main=e11f03f`，但 `origin/main=30021c1`（上游旧基线），回滚点不在 origin 上。
- **[NOTE]** "22 个 workaround 未删除"无法从仓库核实，但 diff 确实未删任何 workaround 代码。
- **[NOTE]** `Adreno650` 条目里的 revision 不是 device submodule 的 gitlink，resolver 优先 device 目录，
  该 pin 不生效，建议文件头说明。

## 作者独立复核

- **BLOCKER 成立**。直接读 `UniXNU/UniXNU@bba3b578` 的 `kernel.py`：
  `preflight()` 在 `scripts/unixnu/kernel.py:196-222`（196 起、222 止；219 是
  `raise Error('Build inputs missing...')`，220-222 是 `root_policies.prepare` 与日志），
  第 199 行取 `entry['build']` 原样，第 210 行
  `source = ctx.tree / (command if command.parts[0] == 'scripts' else Path('scripts') / command)`，
  对 `{component}/scripts/build.py` 得到 `tree/scripts/{component}/scripts/build.py`。
  与评审判断一致。
- **MAJOR（浮动 HEAD 与幂等）成立**：`actions/checkout` 不给 `ref` 时取默认分支 HEAD。
- **MAJOR（输入 delta）成立**：这是我在 finding 里漏写的部分。
- **第三轮补充核实**：`Adreno650` 的 build 是 `build-adreno650.py`，本仓库有这个脚本，tree 会把它
  复制进 `scripts/`，所以 preflight 对它是通过的。占位符阻断只由 `QcomXnuUsbDebug` 一条触发。

## 第一轮处置

| finding | 处置 |
| --- | --- |
| BLOCKER：preflight 不展开 `{component}` | **撤回本次基座切换**。批次 1 不再改基座，改为把现有基座**钉死到 `e0ee2ad9`**（即产出 2026-11-14 镜像的那个提交），并在 finding 里记下阻断的全部证据与上游文件行号。切换留到上游展开占位符、或本仓库提供 wrapper 之后 |
| MAJOR：没有 `ref` | 已加 `ref: e0ee2ad9a2654309480c8b0cc725d0dc09186cd4`，注释说明为什么钉这个提交 |
| MAJOR：supplement 非幂等 | 随切换一起撤回；`configs/components-extra.toml` 从本批次移除。批次 2 的幂等追加逻辑已单独放在 `k40-forensics/catalog-supplement.py`（仓库外，作者本机）（只追加缺失项、并列出应从上表删除的已存在项）；`k40-forensics/make-catalog-supplement.py`（仓库外，作者本机）只是生成器 |
| MINOR：`porting-record.yml` 仍 `ref: main` | 见**第二轮处置表**同名行：两个 workflow 现在钉同一个 SHA |
| MAJOR：输入 delta 未记录 | finding 已列出 RamBlock / AdrenoMetal gitlink / 两个 inputs.json / patches/mesa 23→24，并把结论写成条件句："从 bba3b578 出的镜像**不会**复现 2026-11-14 的哈希"（本批次仍用 e0ee2ad9，所以现有哈希仍然有效） |
| MINOR：21→20 | comment 与 finding 均改为"twenty（17 include + 3 依赖）" |
| MINOR：评审记录在仓库外 | 改为仓库内 `docs/reviews/<date>-<topic>.md`，`AGENTS.md` 与任务说明同步更新 |
| MINOR：未跟踪文件 | 该文件已从本批次移除 |
| MINOR：`test -f` 无输出 / tomllib 未进 fail-fast | supplement 步骤撤回，相关风险随之消失；`tomllib` 由 `unixnu` 自身导入，2026-11-14 的成功 run 已证明 runner 可用 |
| NOTE：缺 `payload=false` 证据 | 本批次仍触及 `.github/**`，因此在提交后**必须**跑一次 `payload=false` 才视为完成 |
| NOTE：回滚点不在 origin | 提交信息里写明回滚哈希 `e11f03f`；该提交在分支推送后即可从 fork 取到，在此之前只存在于本地 |
| NOTE：workaround 未删除 | 已核实：`package.yml` 相对 e11f03f 的 numstat 是 `+13/-3`（12 行注释 + `ref:` 行；删的 3 行是 fork pin 注释），`repository: FFT-infra/UniXNU` 是上下文行，未被删除 |
| NOTE：Adreno650 revision 不生效 | 该文件已移出本批次，批次 2 会在文件头写明 device 目录优先 |

## 第二轮评审：VERDICT 与 findings（原文摘要）

评审者 `efceeab2`，对象为缩减后的批次。结论同样是 **CHANGES REQUESTED**：

- **[MAJOR]** finding 与 workflow 注释把占位符阻断写成"两条 catalog 都受影响"，实际**只有
  `QcomXnuUsbDebug`**：`Adreno650` 的 `build` 是 `build-adreno650.py`，本仓库有这个脚本
  （`scripts/build-adreno650.py`），`tree.py` 会把 device scripts 复制进 `<tree>/scripts`，
  所以 preflight 能通过。阻断成立，但范围要改。
- **[MAJOR]** `porting-record.yml` 仍 `ref: main`，而 `package.yml` 已钉 `e0ee2ad`：记录检查读的是
  自己 checkout 的 `components.toml`，fork 的 main 一动，检查验证的 catalog 就和构建不是同一个。
  另外第一轮那条关于本文件的 MINOR 在处置表里没有对应行。
- **[MINOR]** 行号不精确：preflight 是 `scripts/unixnu/kernel.py:196-222`（219 只是
  `raise Error('Build inputs missing...')`，220-222 是 `root_policies.prepare` 与日志）；
  `cli.py:176-177` 正确。
- **[MINOR]** "2026-11-14 的哈希不再描述同一输入"是现在时，而本批次钉的正是 e0ee2ad，所以那些哈希
  对当前构建**仍然有效**；应改为"从 bba3b578 构建才不会复现"。
- **[MINOR]** 处置表里"删了 3 行注释与 1 行 `repository`"不准：diff 只删 3 行注释，
  `repository:` 是上下文行。
- **[NOTE]** 批次 2 的解除条件要写全：幂等 supplement（或上游补齐两条）+ 占位符展开（或 wrapper）。
- **[NOTE]** 本页只有第一轮 verdict，且范围写的是旧范围；按新规则本页自身也会阻断提交，
  需追加第二轮结论与缩减后的范围。
- **[NOTE]** "把 fork 远端命名为 `mine`" 并未执行（`git remote -v` 只有 origin），应改成
  "分支推送后回滚点才可达"。
- **[NOTE]** 批次 2 的生成脚本仍写 "21 components" 且非幂等；finding 里"追加是合法 TOML 且能存活
  fetch/tree"这一断言已无产物可复核，需改写或加限定。
- **[NOTE]** `porting.toml` 旧 diagnostics finding 说上游 catalog 是 25 个组件，新 finding 说
  bba3b578 是 28 个：两者可能都对但属于不同 revision，记录里应写明。
- **[NOTE]** 页面标题与分支名仍写 bba3b578，而提交只钉 e0ee2ad，需要一句话说明。

## 第二轮处置

| finding | 处置 |
| --- | --- |
| MAJOR：占位符只影响 QcomXnuUsbDebug | `package.yml` 注释与 finding 都改为"只有 QcomXnuUsbDebug 的 build 行以占位符开头；Adreno650 用 build-adreno650.py，preflight 能过" |
| MAJOR：`porting-record.yml` 仍浮动 | 已钉到同一个 `e0ee2ad9a2654309480c8b0cc725d0dc09186cd4`，并加注释说明两个 workflow 必须同 catalog；处置表补了这一行 |
| MINOR：行号 | 统一改为 `scripts/unixnu/kernel.py:196-222` 与 `cli.py:176-177` |
| MINOR：时态 | 改为条件句"从 bba3b578 构建才不会复现 2026-11-14 的哈希" |
| MINOR：workaround 行措辞 | 改为 `+13/-3`、删的是 3 行注释、`repository:` 是上下文行 |
| NOTE：批次 2 条件 | finding 与结论都写成"幂等 supplement（或上游补齐）+ 占位符展开（或 wrapper）+ 先跑 kernel-only" |
| NOTE：本页缺第二轮 | 即本节 |
| NOTE：`mine` 远端 | 改为"分支推送后回滚点才可达"，不再声称已配置远端 |
| NOTE：生成脚本与断言 | 幂等追加逻辑放进 `k40-forensics/catalog-supplement.py`，`make-catalog-supplement.py` 只做生成且已改 20 个组件；finding 里**删除了**"追加是合法 TOML / 能存活 fetch/tree"的断言（本批次没有 supplement 产物可复核），批次 2 再补证据 |
| NOTE：25 vs 28 | finding 里写明"旧 finding 记的是另一个 revision；bba3b578 实测 28" |
| NOTE：页名与分支名 | 页首加说明：标题与分支名是历史名称，本批只钉 e0ee2ad |

## 第三轮评审：VERDICT 与 findings（原文摘要）

评审者 `2a476f49`，逐条核对第二轮 findings。**无 BLOCKER / MAJOR**，结论仍为
CHANGES REQUESTED，全部是记录自述与工具卫生：

- **[MINOR]** 处置表声称 finding 里新增了"追加本身是合法 TOML；是否被覆盖留待批次 2"，
  但 finding 里并不存在这句（实际只是删掉了旧断言）。
- **[MINOR]** "幂等追加"被记在 `make-catalog-supplement.py` 名下，但那是生成器；幂等追加器是
  `catalog-supplement.py`，且生成器头注释仍断言 `package.yml` 会 append。
- **[MINOR]** `+10/-3` 应为 `+13/-3`（numstat 实测）。
- **[MINOR]** 作者复核段仍写 `196-219`，应为 `196-222`（219 只是 raise）。
- **[NOTE]** 页首"两件事"与结论"三件"不一致；两轮"评审对象"清单都漏了 `porting-record.yml`。
- **[NOTE]** 第一轮处置表缺 `porting-record.yml` 那条 MINOR 的处置行。
- **[NOTE]** `AGENTS.md` 里 `scripts/porting-record.py` 未说明是 tools checkout 的脚本。
- **[NOTE]** 本页仍是未跟踪文件，`git commit -am` 会漏。
- **[NOTE]** 批次 2 的两个脚本在仓库外、生成器含绝对路径与内嵌 GitHub token，干净克隆无法复现。
- **[NOTE]** `porting-record.yml` 头注释仍把切回条件写成"once it is upstream"（只提 catalog）。

## 第三轮处置

| finding | 处置 |
| --- | --- |
| MINOR：finding 新句不存在 | 处置表改为"**删除了**该断言，批次 2 再补证据"，与 finding 实际内容一致 |
| MINOR：appender 记错文件 | 处置表点名 `catalog-supplement.py`；生成器头注释同步改正（不再声称 `package.yml` 会 append） |
| MINOR：`+13/-3` | 第一轮与第二轮处置表都已改正 |
| MINOR：`196-219` | 作者复核段改为 `196-222`，并补一句第三轮的核实结论 |
| NOTE：两件事/三件事 | 页首改为"三件事"，并把 `porting-record.yml` 加进第二轮评审对象，并在第一轮对象后注明该文件当时未改动 |
| NOTE：第一轮表缺行 | 补一行，指向第二轮处置表 |
| NOTE：`porting-record.py` 路径 | `AGENTS.md` 改为"tools checkout 的 `tools/scripts/porting-record.py`" |
| NOTE：本页未跟踪 | 提交前 `git add` 全部五个文件，并用 `git show --stat` 复核五项都在提交里（只 add 本页会得到一个只有记录的提交） |
| NOTE：批次 2 脚本在仓库外 | 从生成器里去掉绝对路径与内嵌 token（改为读取参数/环境变量）；算法写进本页下方的批次 2 条件说明，不再依赖本机文件 |
| NOTE：头注释 only-upstream | `porting-record.yml` 头注释改为"catalog 两条 + 占位符展开"两个条件，并指向 finding |


## 第四轮评审：VERDICT 与 findings（原文摘要）

评审者 `3ef79e36`，窄范围核对第三轮的 10 条。结论仍为 CHANGES REQUESTED，无 BLOCKER/MAJOR：
10 条里 8 条确认修掉，两条只修了一半，另有两处引用未标注。

- **[MINOR]** 第一轮"评审对象"清单仍无 `porting-record.yml`，与页内引用它的一轮 findings 和处置行
  矛盾，而处置行却声称"加进了两轮清单"。
- **[MINOR]** 撤回轮次被写成三种：页首"三轮评审后撤回"、"第一轮否决了切换本身"、
  finding 里"withdrawn after two independent reviews"。
- **[MINOR]** 提交检查清单只要求复核本页，而当时 index 里只有本页，四个改动文件仍未暂存。
- **[NOTE]** 第一轮原文引用的 `196-219` 没有行内标注。
- **[NOTE]** `catalog-supplement.py` 记名时未说明在仓库外；"批次 2 算法"只描述追加器，漏了生成器。

## 第四轮处置

| finding | 处置 |
| --- | --- |
| MINOR：第一轮清单 | 第一轮对象后加括注：该文件当时未改动，但已进第一轮 findings |
| MINOR：撤回轮次 | 页首改为"**第一轮评审的 BLOCKER 之后即撤回**（此后各轮评审的是缩减后的批次）"，正文不再声明轮次数字；`porting.toml` 的 finding 写作 "withdrawn after the first of four independent reviews; the later three reviewed the reduced batch" |
| MINOR：提交范围 | 检查清单改为"五个文件一起 add，用 `git show --stat` 复核五项" |
| NOTE：`196-219` | 引用处加括注"原文行号；准确范围是 196-222，见复核段" |
| NOTE：路径与生成器 | 两件工具都标注"仓库外，作者本机"；结论补上生成器的职责与完整算法 |
## 末轮评审（窄范围核验）

评审者 `3cf4f28f` 只核对上一轮的 5 条与是否引入新矛盾：**5 条全部修掉**，唯一新问题是
本页仍在正文里数轮次（页首已按"此后各轮"改写，而另外四处仍写"三轮"）。

处置：本页**不再在正文里声明轮次数字**——每节标题说明它属于哪一轮，评审者清单逐一点名，
这样后续再加一轮也不会产生新的自相矛盾。


## 结论

批次 1 缩减为三件互不耦合的事：**两个 workflow 的 tools checkout 都钉到 `e0ee2ad9`**、
Review gate 写进仓库、把这次阻断的测量与各轮评审写进记录。
切换本身转为独立批次，解除条件是：幂等 catalog supplement（或上游补齐两条）**且**
上游展开 `{component}` 占位符（或本仓库提供 wrapper），并以 `payload=false` 通过为先决条件。
批次 2 的算法写在这里，重做时不依赖本机文件：先用 fork 的 `components.toml` 摘出
`QcomXnuUsbDebug` 与 `Adreno650` 两个 `[[component]]` 块生成增量文件（逐字复制，不改字段）；再把
每个块的 `id` 与目标 `components.toml` 比对，只追加缺失项，并把已存在的项打印出来要求从增量
文件中删除；追加后重新解析目标文件，确认两个 `id` 都在。
