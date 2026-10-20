# 改进清单（对照代码逐条核实）

> 2026-09-23 建立。来源：外部给的「从半成品到真正可用」改进清单。
>
> **这份不是待办愿望单，是核实过的账。** 每一条都对着代码查了「做了没有 / 落在哪个文件」，
> 所以它同时回答两个问题：**还缺什么**，以及**别重复做什么**。
>
> 记号：✅ 已做 · ⚠️ 部分 · ❌ 缺口 · 🚫 有意不做（附理由）

---

## 总览

| 阶段 | ✅ | ⚠️ | ❌ | 🚫 |
|---|---|---|---|---|
| 一、稳固核心闭环 | 11 | 1 | 0 | 0 |
| 二、可观测性与用户接管 | 15 | 0 | 0 | 0 |
| 三、能力边界与可预期性 | 11 | 0 | 0 | 0 |
| 四、真实任务验收与文档 | 9 | 0 | 4 | 1 |

**一句话结论（2026-09-24 重新点数）：代码侧和文档侧的活都清完了。**
剩下的 4 个 ❌ 全卡在同一件事上 ——「**得有人真的用它**」：
真实任务验收、失败根因分析、早期用户试用，外加一个要录屏的动图。
唯一还挂着的 ⚠️（「成功率 ≥90% 没有量化基线」）同样要真跑一批才能量出来。

---

## 阶段一：稳固核心闭环

### 1.1 锁定最小可用闭环

- ✅ **最高频任务路径**：单文件/多文件修改 → 跑测试 → 汇报。提示词与工具集就是围绕它设计的
  （`loop-prompt.cjs`、`tools/registry.cjs`）
- ✅ **完整流程**：创建任务 → 工具调用 → 文件改动 → 测试执行 → 结果汇报
  （`task.cjs` + `tools/index.cjs` + `task-outcome.cjs` 认「跑没跑测试、过没过」）
- ⚠️ **成功率 ≥90%**：**2026-10-08 有了第一份量化基线**（10 条 Golden Tasks、29 次运行、
  总成功率 29/29）—— 但那是**任务集内**的成功率，不等于真实使用的成功率。
  完整数字见 [`Model Interaction Baseline.md`](Model%20Interaction%20Baseline.md) §3；
  任务定义见 [`Golden Tasks.md`](Golden%20Tasks.md)。「真实任务验收」那条仍见阶段四 4.1
- ✅ **非核心能力降级**：Model Router 默认关闭（`router.enabled = false`，CHANGELOG 0.24.0 起）；
  多任务并行是显式开的；MCP 要用户自己配服务器

### 1.2 失败处理：让失败成为一等公民

- ✅ **失败可识别**：`errors.cjs` **13 类**（`context_overflow` / `network` / `auth` / 文件不存在…），
  每类带「能不能自动恢复」+ 处置策略
- ✅ **失败可解释**：`task-diagnose.cjs` 出人话报告；`task.errors[]` 留原话
- ✅ **可重试自动重试**：`llm.cjs` 同供应商指数退避 → 换供应商；换之前明确告知（P1 表）
- ✅ **不可重试给用户选项**：权限确认框 + 任务横幅（继续 / 放弃）；`canAutoRecover` 对写操作返回 **false**
- ✅ **部分完成可断点续跑**：`task-resume.cjs` 的 `reopen()`；退出时标 `paused`，下次启动提示
- ✅ **失败不污染状态**：改动事务 `changeset.cjs`（写前快照、整批撤）；任务台账 `task-io.cjs` 原子写
- ✅ **台账记失败**：`task.errors[]` / `task.retries` / `task.pauseReason` / `budgetHit`
- ✅ **UI 区分状态**：`statusLanguage.ts` + `lifecycle.cjs` 13 相位；
  任务中心按 `running / paused / waiting_user / failed / completed / cancelled` **六组分**（`taskCenterModel.ts`）

---

## 阶段二：可观测性与用户接管

### 2.1 可观测性

- ✅ **当前状态**：`lifecycle.PHASES`（planning / executing / verifying / responding…）→ 状态栏 + 任务行
- ✅ **进度指示**：`taskProgress()` 出 N/M；`ProgressTimeline.tsx` 的计划勾选
- ✅ **最近动作**：`TaskConsole.tsx` 的 Tool Calls（步数 / 用时 / 成败）
- ✅ **异常提示**：`loopGuard.cjs` 认 A B A B 转圈（周期 2、重复 3 遍）→ 先改道，几次不听才交人；
  `task-notify.cjs` 出通知文案
- ✅ **台账带步骤计数**：`task.steps` + `checkpoints`
- ✅ **顶栏实时状态**：`layout/StatusBar.tsx`
- ✅ **转圈提示的具体措辞**（2026-09-24，v1.18.0）：那句「交给用户的话」以前硬编码在
  `loop-model.cjs` 里，只能跑满 12 轮才测得到 —— **等于没被钉住**。现在搬到
  `loop-guard.cjs` 的 `stopMessage()`，措辞有断言（「可能陷入重复执行」「连着几次还是
  转圈」「（继续 / 停止）」），界面那侧真机验过（「已暂停 · 检测到重复执行」+ 证据 + 两个按钮）

### 2.2 用户接管

- ✅ **暂停**：`chat:pause` + `controls.pauseRequested()`；Stop 全链路（AG-010）
- ✅ **修改方向**：`task-steering.cjs`（暂停状态下输入新指令，从当前检查点继续）
- ✅ **手动编辑**：`MessageEditor.tsx`；手动改过的文件在恢复时会被识别（见下条）
- ✅ **回滚**：检查点（AG-035 自动 checkpoint）+ 改动事务整批撤
- ✅ **终止**：任务中心可放弃（`task:purge` / `removeMany`）
- ✅ **暂停时显示检查点与已完成步骤**：任务详情里的检查点列表 + 计划勾选
- ✅ **回退：撤检查点之后的改动**（2026-09-24 加内核 / AG-045，2026-10-01 补界面入口 / AG-052）：
  `changeset:rollbackTo` 撤这个点**之后**的改动（判据是时间戳，界面请传 `at`）。整批撤（「撤销这些改动」）仍在，
  两者分工：一个回到某一步，一个整轮撤
  ⚠️ **名字别写成「回到某个检查点」** —— 它只撤「这个点之后**才第一次**被改的文件」，
  多轮反复改同一个文件时那个文件撤不动（快照只有改前镜像）。界面文案、确认框、
  审查面板的记录都按这个口径写销
- ✅ **手动改文件后重新读取**：`task.recovery` 算 `envChanged`，启动时提示
  「其中 N 条要动的文件已经变过了」（`useAppBootstrap.ts`），**不依赖缓存**（AG-019 避免重复读取是缓存，但有失效判定）

---

## 阶段三：能力边界与可预期性

### 3.1 能力边界

- ✅ **支持的任务类型**：README「任务类型矩阵」列了 12 类，各带成熟度与已知边界
- ✅ **不支持的任务类型**：同一张表里的「不做」两行 + Roadmap 的「不打算做」清单
- ✅ **README 的成熟度矩阵**（实验性 / 可用 / 稳定）—— 2026-09-23 补上
- ✅ **UI 中超出边界提前提示**（2026-09-24，v1.18.0）：`core/capability-bounds.cjs` 六条边界，
  **只提示不拦**（消息照发、活照跑）；模式写窄（「看看这个监控面板」不会误判成常驻监控）；
  提示里带上命中的原话，误报时用户一眼能判断。真机验过。自检组 `78-bounds`（20 项）
- ✅ **实验性能力默认关闭**：Model Router 默认关；插件要手动放目录；MCP 要手动配

### 3.2 可预期性

- ✅ **按任务类型预设推荐参数**（2026-09-24，v1.18.0）：`core/task-presets.cjs` 六种类型，
  各带温度 / 轮数 / 工具上限与**理由**；判定复用意图路由；认不出来就不给；用户调过全局预算就不插手。
  温度落对话级、预算落任务级（台账记 `preset`）。自检组 `76-task-presets`（33 项）
- ✅ **系统提示 / 工具描述 / 技能说明版本化**（2026-09-24，v1.18.0）：版本号 `prompt-stack/1`
  早就有、一直没人接；现在记进任务台账（`task.promptVersion`）、诊断里显示一行，
  并写清了「什么时候该 +1」（改文案 / 改层顺序 / 增删层）
- ✅ **Memory 注入逻辑明确**：`memory-recall.cjs` 按范围/类型/重要度/新鲜度/关键词打分，只注入前 N
- ✅ **重试策略讲全了**（2026-09-24）：见 [`架构.md`](架构.md) 的「重试与降级（讲全一次）」——
  读操作可自动重试、**写操作永不自动重试**、换供应商之前明确告知、上下文超限走压缩
- ✅ **台账记本次参数**：`task.model` / `task.models` / `task.budget` / `budgetHit` /
  `promptVersion` / `preset`
- ✅ **同一任务多次执行的一致性检查**（2026-09-24，v1.18.0）：`core/task-consistency.cjs`
  在诊断里摆出「上次跑同一目标」的差异（只跟更早的比；找不出可比对象就不写）。
  措辞是「对不上」不是「有问题」。自检组 `77-consistency`（19 项）

---

## 阶段四：真实任务验收与文档

### 4.1 真实任务验收

- ❌ **5–10 个真实任务 × 各跑 3 次** —— **需要人真的跑**，不是代码能补的。
  现成入口：`npm run acceptance`（真模型跑 **8 类**任务 T1–T8、按产物判成败，见
  `tools/acceptance-cases.mjs`；2026-09-30 更正：这里原写「5 类」，早已扩到 8 类）、
  `docs/试玩清单.md`（可扩成场景表）、两份验收核对（见下条）
- ❌ **失败根因分析** —— 同上（得先有真实失败样本）
- ✅ **内部验收清单**（2026-09-24）：现在有两份 —— `docs/验收核对-第16章.md`（31 项，有断言背书）
  与 [`docs/验收核对-通用.md`](验收核对-通用.md)（第 16 章之外 + 之后新增的能力 + 历史事故路径，
  30 项，标了哪些要上手）
- 🚫 **CI 里的端到端测试** —— `ci.yml` 注释里写明了**有意不做**：
  「不跑 `npm run package` —— 打便携版 exe 要 Windows + 几分钟，属于 Release 的事」。
  替代品是真机工具 `npm run shot:electron`（CDP 驱动打包应用），但目前要手动跑
- ❌ **早期用户试用** —— 流程问题

### 4.2 文档与上手

- ✅ **5 分钟快速开始**（2026-09-24）：README「快速开始」改成四步 ——
  下载 / 源码 / 填模型 / **跑通第一个任务**（点哪里、会看到哪四样东西、第一次拿什么目录试）
- ❌ **GIF / 短视频**：`docs/screenshots/` 有 22 张静态图，没有动图 ——
  **要录屏**（人做或另配工具链），代码补不了
- ✅ **核心概念文档**（2026-09-24）：新增 [`docs/核心概念.md`](核心概念.md) ——
  会话 / 任务 / 计划 / 检查点 / 改动事务 / 工具调用 / 权限 / 记忆 / 技能 / 项目 / 预算，一页讲清
- ✅ **FAQ**：README 有「FAQ」节 + 「已知问题」
- ✅ **故障排查文档**（面向用户）—— 2026-09-23 补上 `docs/故障排查.md`
- ✅ **平台支持矩阵** —— 2026-09-23 补上（见 README）
- ✅ **架构文档** —— 2026-09-23 补上 `docs/架构.md`
- ✅ **贡献指南**：根目录 `CONTRIBUTING.md`
- ✅ **版本发布与更新机制 / 数据迁移策略**（2026-09-24，面向用户）：
  新增 [`docs/更新与数据迁移.md`](更新与数据迁移.md) —— 怎么更新（保 `data/`）、备份与恢复、
  迁移规矩、加密与密钥绑定、怎么回退

---

## 本轮（2026-09-23）做了什么

| 项 | 产物 |
|---|---|
| 3.1 任务类型 × 成熟度矩阵 | README 新增「任务类型矩阵」节 |
| 4.2 平台支持矩阵 | README 新增「平台支持」节 |
| 4.2 架构文档 | 新增 `docs/架构.md` |
| 4.2 故障排查文档 | 新增 `docs/故障排查.md` |
| 本文档 | `docs/improvement-checklist.md`（就是这份） |
| 顺带 | 修掉 `ci.yml` 里滞后的「内核自检（1500+ 项）」→ 1811 |

## 2026-10-04 记（审计修复时**顺手发现、没修**的）

| 项 | 说明 | 影响 | 状态 |
| --- | --- | --- | --- |
| 诊断包读「昨天」的日志 | `core/diagnostics.cjs` 的 `logFile()` 用 `new Date().toISOString().slice(0,10)`（**UTC** 日期）拼日志文件名，而应用自己写日志按**本地日期**命名（`core/log.cjs`）→ 东八区**凌晨 8 点之前**导出的诊断包，读的是**前一天**的日志文件 | 排查问题时看错日志（会误导）；不影响数据 | ✅ **2026-10-06 已修**：`logFile()` 改用 `log.dayStamp()`（本地日期，一个口径一处实现） |
| 前端第 4 份脱敏表 | `src/lib/dataPortExport.ts` 里自带一张「像不像密钥」的正则表（导出数据时用），与 `core/redact.cjs` 的模式表**不共享** | 与问题 8/9 同源：多条规则会漂移，改了主进程忘了前端 | ✅ **2026-10-06 已钉**：两边不合并（内核 CJS / 渲染层过 Vite，硬合会把内核拖进打包链），但加了 `dataPortExportSync.test.ts` 比正则体（键名逐字一致 + 自由文本候选是内核子集）。原来注释里写的「自检组 69」**根本不存在** —— 这条契约此前没人守 |
| 三份原子写 | `core/session-crypto.cjs` 自己那份 `writeAtomic`（加密迁移用）与新的 `core/safe-write.cjs` 重复。**没并**：`session*.cjs` 是硬禁区 | 同一段逻辑三个地方（`session-crypto` / `safe-write` / 各自的调用点） | ✅ **2026-10-07 已收敛**（用户授权后动 `session-crypto.cjs`）：实际是 **4 份**（不止三份）—— `safe-write`（共享版，**只有它有**退避重试）+ `session-crypto` / `projects` / `schedule-store` 各写的本地副本（**都没重试**）。现在三处一律走 `safe-write.writeAtomic`。自检 `118-safe-write` 加钉子：**全内核只允许一个 `writeAtomic` 实现**。「没顺手动的」：`profile.cjs` 存头像（二进制拷贝）/ `rotate-file.cjs`（轮转）/ `memory.cjs`（旧文件改名）语义不同，各自留着。真机验过：会话加密往返 23 行 + `projects.json` / `schedules.json` 写入 |
| 三个文件贴在 300 行上 | `src/types/models-extra.ts`、`scripts/selftest/groups/04-reliability.mjs`、`src/components/settings/tabs/SecurityTab.tsx`（第 3 批想在那页加一句提示，加了就 303 ✗ —— 提示改挂到 `ProviderPanel`）；另有 `electron/core/config-normalize.cjs` 现在**正好 300** | 下次动它们时会先撞红线，得顺手拆 | ✅ 2026-10-06 核过：**正好 300 行**的其实是 9 个（不止这里列的三个）——还有 `Composer.tsx` / `TaskRow.tsx` / `McpTab.tsx` / `models-extra.ts` / `safety.ts` / `config-defaults.cjs` / `errors.cjs`。拆分是纯机械活且有回归风险，留给专门一批（拆法见 `.github/skills/split-file`） |
| `browse` 的三个兄弟不过网络策略 | `browse-click` / `browse-elements` / `browse-type` 操作的是**已打开**的页面。问题 20 只把策略接在 `browse` 的 URL 上；跳转归 `navigation-policy`（`general.browserNavigation`）。要接得先说清「按哪个 URL 判」（目标链接 / 当前页） | 「设置里禁止了网络，Agent 却还能点开一个链接」——取决于页面自己怎么跳 | ✅ **2026-10-07 已接**：口径定为「按跳转的**目标 URL** 判，只接最强档（`deny` / 命中禁止名单），且优先于导航策略的 `allow`」—— `navigation-policy.decide` 多收一个「目标是否被网络策略明确禁止」（`defaultNetDeny`，与 `browse` 同一份判据），`window.open` 那条路也按它过滤。**`ask` / `allow` 档一个字没改**（在 `will-navigate` 里没法弹卡，接 `ask` 只能二选一，都比现状差）。「这次不做什么」记在 [`安全模型.md`](安全模型.md) §8。自检 `09-browser`（+6 项）。`browse_nav`（后退 / 前进不触发 `will-navigate`）仍是缺口 |
| 规模授权收窄后的多一次往返 | 第 3 批把授权收窄成「只认紧接着拦截那次 ask、且问题里有规模词 + 数字」。模型若在拦截后先去读了文件再问，就得再问一次（安全方向的代价） | 多一张卡片；不会误授权 | 发现于 2026-10-04，**先看着**（实测发现模型常跑偏，再考虑把窗口放宽） |

## 2026-10-04 记（第 3 批做完，已销账的三条）

| 项 | 结果 |
| --- | --- |
| `looksSecret` 修好但零调用 | ✅ 已销：`memory-schema.looksLikeSecret` 直接就是它（三处判据收敛成一处） |
| `task-notes.addCommand` 的 command/result 未脱敏 | ✅ 已销：写盘前过 `redact`（**先脱敏再截断**），`exitOk` 仍用原文判 |
| `run_shell` 与 `browse` 的 ask 口径不一致 | ✅ 已销：共用 `_shared.decideNetAsk` —— 上层会问就交给它、不会问就拒（`browse` 不再「放行 + 记日志」） |

## 2026-10-04 记（第 4 批做完，新发现的两条）

| 项 | 说明 | 影响 | 状态 |
| --- | --- | --- | --- |
| 两处单文件轮转逻辑 | `log-actions.rotateIfNeeded`（6MB，先有）与新的 `data-retention.rotateFile`（token 指标用）是同一套做法。**没并**：不动已验过的代码（本批约束） | 同一段逻辑两个地方，改一处容易忘另一处 | ✅ **2026-10-06 已收敛**到 `electron/core/rotate-file.cjs`，两边都改调它 |
| 审计问题 18 / 19 一直没排进批次 | ① 外链打开无 scheme 白名单（两条 `shell.openExternal` 出口 + 渲染层正则放行 `file:`，顺带终端链接是死路径）；② 主窗口无 CSP | 两者都是审计说的「建议修」但**没进任何一批**（每批清单里都没有），不是被否掉 | ✅ **第 5 批已做**（外链白名单 + CSP，见 `审计修复进度.md` 第 5 批） |

## 还没做（按建议顺序）

0. ~~**A2 规模预检的「文件数」闸门**（2026-10-03 记，2026-10-04 第 3 批又核了一遍）~~
   → ✅ **2026-10-06 已接上**：新增 `electron/core/scale-files.cjs`，口径是**下界**
   （只数工作目录 + 每个子目录两层、到上限就停；读不出来给 **null** 而不是 0）——
   只在「递归 / 批量」时才估。`scale.cjs` 的第四条硬拦（`scaleMaxFiles`）从
   「恒不成立、只能挂在源码里假装存在」变成真的会拦；自检 `109-scale-config` 里
   对应那条从「筛查」升级成**行为断言**（真数一个目录）。
   · 界面上**仍然不给旋钮**（刻意）：估算只会低估，当旋钮会给人「很准」的错觉；
     它是内部门槛，用审计里的真实命中率调。
   · 校准口径（原样保留）：每次触发都写审计 `extras.scale` + 一行 `[规模预检]` 日志；
     跑一段时间后统计「预估 ≥ 300s 的操作实际触发多少次 / 实际跑多久」，用来校准
     `scaleHardSeconds` 的默认值（现在是拍的 120 秒 —— 「不靠拍，靠数据」）。

1. ~~**危险度层：递归删盘根的写法漏了一半**（2026-10-03 做 A2 第 4 步时顺手撞出来，没动它）~~
   → ✅ **2026-10-03 已修**（`core/risk-targets.cjs` + 自检组 `61-destructive`）：
   根因是**判据钉错了地方** —— 旧的两条 PowerShell 正则要求盘根恰好在**最后一个参数**，
   顺序一换（`-Recurse C:\ -Force`）、加引号（`"C:\"`）就掉出 critical；`rm -rf /`
   自己一条都没匹配上（而注释里拿它当参照）。修法：**先归一化再判目标**，位置 / 引号 / 额外开关无关。
   · 已修复（原来是 high，现在 critical）：`rm -rf /` · `rm -rf C:\` · `rm -rf "/"` · `rm -rf C: -Confirm:$false`
     · `del /s /q C:\` · `rd /s /q C:\Windows` · `Remove-Item` 的四种写法 · `sudo` / `bash -c` 包裹的 ·
     · 不带 `-r` 的 `rm -f /`（原来只判 medium → 会被静默放行）·
     家目录（`~` / `$env:USERPROFILE` / `C:\Users\<名字>`）**及其下的一级标准目录**（`~/Documents` 等）·
     系统目录及其子树（`C:\Windows` / `/usr` / `/var` …）· 所有用户的父目录（`C:\Users` / `/home`）·
     间接写法（`find / -delete`、`echo / | xargs rm -rf`、脚本里写死的 `rmtree('E:/')`）
   · 曾漏过的 28 条原样记在 `踩坑记录.md`；实测口径：78+ 条里 0 漏判 / 0 误判
   · 反向锁：`MUST_NOT_CRITICAL`（17 条，相对路径 / 再深一层 / 名字像但不是 / 跨段）
   · ⚠️ **仍未覆盖的**（以后真踩到再补，别以为全收了）：没白名单的变量（`${HOME}`、`%HOMEPATH%`、
     `$env:LOCALAPPDATA`）当目标时**不判**（保守）—— 变量看不见就不猜；`/tmp` 与 `/var/tmp`
     这类 scratch 位置**故意**不算（但对 `/var` 整棵子树是算的，这是取舍，见 `安全模型.md` §3）。
2. ~~**回退到指定检查点**（2.2）~~ → ✅ **2026-09-24 已做（v1.12.0，AG-045）**：
   `changeset:rollbackTo` + `changeset-rollback.cjs`，撤销检查点**之后**的改动。
   判据是时间戳（`checkpoint.at` vs `files[].at`），界面请传 `at`（下标会随 50 条截断挪位）。
   自检组 `66-rollback-to`（43 项）
3. ~~**提示词/工具描述版本化**（3.2）~~ → ✅ **2026-09-24 已做（v1.18.0）**：
   `prompt-stack.cjs` 的 `VERSION` 早就有，缺的是**没人接**。现在
   `buildPromptContext` 把它交出去、`loop.cjs` 记进任务台账（`task.promptVersion`），
   诊断报告里显示一行「提示词：prompt-stack/1」。同一版只写一次盘；
   中途换过版本的记进 `promptVersions`。自检钉在 `06-prompt-state` 组。
4. ~~**按任务类型预设参数**（3.2）~~ → ✅ **2026-09-24 已做（v1.18.0）**：
   `core/task-presets.cjs`——六种类型（闲聊/查资料/写代码/多步执行/创作/分析）各有
   温度与**为什么这么推荐**。两条设计：判定复用意图路由的规则（不另写一套关键词）；
   **认不出来就不给建议**（兜底成 chat 是「没法判断」不是「判断出是闲聊」）。
   温度落在**对话级**（`threadSettings.temperature`，界面在「设置 → 对话」里选）。
   ⚠️ **2026-09-29 改**：预设**不再预填预算**（原来给每个任务塞「轮数 60 / 工具 120」，
   用户看到撞上限却查不出数字哪来的）；预算默认全不限，只在任务卡片上由用户自己填。
   自检组 `76-task-presets`。
5. ~~**一致性检查**（3.2）~~ → ✅ **2026-09-24 已做（v1.18.0）**：
   `core/task-consistency.cjs`——同一会话里找目标最像的**更早**那条任务，
   比结论、测试结果、改动范围、计划步数，把差异摆出来（诊断报告里一段）。
   措辞是「对不上」而不是「有问题」：差异大不等于做错了。
   找不出可比对象就一个字都不写。自检组 `77-consistency`（19 项）。
6. ~~**转圈提示的文案守卫**（2.1）~~ → ✅ **2026-09-24 已做（v1.18.0）**：
   那句话以前硬编码在 `loop-model.cjs`，只能跑满 12 轮才测到，**等于没被钉住**。
   现在搬到 `loop-guard.cjs` 的 `stopMessage()`，自检直接断言完整措辞
   （「可能陷入重复执行」「连着几次还是转圈」「（继续 / 停止）」）并钉住
   「循环真的调它」。界面那一侧用真机探针验过：造一条 loop-paused 任务，
   界面上出现「已暂停 · 检测到重复执行」+ 证据 + 继续/停止按钮。
7. **真实任务验收**（4.1）—— **2026-10-08 做了第一轮**：
   10 条 Golden Tasks（定义见 [`Golden Tasks.md`](Golden%20Tasks.md)）在独立实例上跑了 29 次，
   29/29 到达 `completed`，数字进了 [`Model Interaction Baseline.md`](Model%20Interaction%20Baseline.md)。
   这**不等于**「真实使用的成功率」—— 任务集是我设计的，比真实场景窄。
   下一步（要么你来、要么授权我）：把 `npm run acceptance` 加 `docs/试玩清单.md`
   扩成 5–10 个**真实**场景、各跑 3 次，失败的那几条顺带就是根因分析的样本。
   （另一个 ❌ 是 GIF/短视频 —— 要录屏，代码补不了。）
8. ~~**界面上超边界任务的提前提示**（3.1）~~ → ✅ **2026-09-24 已做（v1.18.0）**：
   `core/capability-bounds.cjs`——六条边界（常驻后台 / 云同步 / 多人协作 / 手机端 /
   跑一整天 / 跨仓库 / 碰生产），**只提示不拦**：消息照发、活照跑。
   模式写得很窄（「看看这个监控面板」不算常驻监控），因为误报一次用户就再也不看这类提示了。
   提示里带上命中的原话，误报时他一眼能判断。真机验过。
   自检组 `78-bounds`（20 项）。

## 2026-09-24 补做（改进清单 A 组前三项）

| 项 | 产物 | 自检组 |
|---|---|---|
| **回退到指定检查点**（AG-045） | `changeset-rollback.cjs` + `changeset:rollbackTo` | `66-rollback-to`（43 项） |
| **上条的界面入口 + 干跑预览 + 中文失败原因**（AG-052） | `checkpointRollbackApi.ts` + `rollback/CheckpointRollbackBar.tsx` + `RollbackRecord.tsx` | `98-rollback-preview`（40 项）+ 真机 `tmp/verify-rollback-ui.cjs`（19 项，**该脚本已不在仓库**） |
| **Artifact 内核落盘 + 版本化**（AG-046） | `core/artifact.cjs` + `handlers/artifact.cjs` + `artifactApi.ts` | `67-artifact` |
| **Skill 权限声明**（AG-047） | `core/skill-permissions.cjs` + 设置页展示 | `68-skill-perms` |

## 2026-09-24 · 第二轮（可预期性 B 组 + 文档三件套 + 启动页）

| 项 | 产物 | 验证 |
|---|---|---|
| 提示词版本进台账（②-1） | `task.promptVersion` + 诊断一行 | 自检组 `06-prompt-state` |
| 按任务类型预设参数（②-2） | `core/task-presets.cjs` + 设置 → 对话里的「温度」 | `76-task-presets`（真机验过） |
| 一致性检查（②-3） | `core/task-consistency.cjs`（诊断里一段） | `77-consistency` |
| 转圈文案守卫（②-4） | `loop-guard.stopMessage()` | `42-loopguard`（真机验过） |
| 超边界提前提示（②-5） | `core/capability-bounds.cjs` | `78-bounds`（真机验过） |
| **启动页跟着「真的就绪」收尾** | `bootFrames.bootExitAt` + `BootSequence` 的 `ready` | `bootFrames.test.ts`（真机量过：**10.5s → ~2.5s**） |
| 文档三件套 | `docs/核心概念.md` / `docs/更新与数据迁移.md` / README「快速开始」重写 | —— |
| 通用验收清单 | `docs/验收核对-通用.md`（30 项） | —— |
| 顺带修掉 | `data/tasks` 里 647 条测试残留（挪去备份）+ `env.mjs` 的 `disposeTasks()` | 五个自检组改用 |

## 2026-10-03 · 用 Harbor 做白板工具那一轮暴露的（1 / 3 / 4 已做，只剩 2）

> 这一节起初**只记账、没动手**（四条都不是「顺手改一下」的东西）。2026-10-07 用户授权后：
> **1（规模闸门按实际规模）、3（自动化测试默认动作）、4（测试脚本可靠性）已落地**；
> 只剩 **2（长任务上下文断层）** 仍是 ❌ —— 那是架构级改动（要 Personal Context 那层稳定事实），
> **结论是「先不做」**。
>
> 记号沿用上面：⚠️ 部分 · ❌ 缺口 · 🚫 有意不做。

### ✅ 1. 规模闸门按「命令形态」拦，不按「实际规模」（P4-1）—— 已闭合

**现象**：Agent 要清理 2 个临时文件 + 列一层目录，被规模闸门拦下要求确认。
**根因**：判据是**命令形态**（看到 `-Recurse` / `rm -rf` 就拦），不是**实际规模**
（这个目录是 3 MB 还是 3 GB）。A2 刚做完的「重操作规模确认」第一次真实使用就撞上了。
**方向**（两选一，还没定）：
- 先做**小范围探测**（只数工作目录前两层 / 读目录项数），拿到量再决定拦不拦；
  代价是探测本身有开销，而且要处理「探测失败」的退路；
- 或者只拦**预估超过阈值**的，不拦所有递归操作 —— 但预估本身就得先探测（回到上一条）。

**关联**：`scaleMaxFiles`（原来基本算不出来）是同一个缺口的两半。校准口径已经建好：审计
`extras.scale` + 一行 `[规模预检]` 日志，攒一段时间统计「预估 ≥ 300s 的操作实际触发多少次 / 实际跑多久」。

> **进展（2026-10-07 已闭合）**：文件数**已经能估**（`core/scale-files.cjs`，2026-10-06 接通）。
> 2026-10-07 按用户批准修掉了判据：`core/scale.cjs` 拦之前先探**目标路径的实际规模**
> （新增 `probeSize`：**有界递归**，数到 `scaleMaxFiles` 或目录上限就停）——
> 探得到且不大 → 不拦（降 note）；探得到但很大 → 照旧拦；探不到 → 照旧拦（保守方向）。
> 走的是上上面「方向」的第一条。
> 修它动了权限行为（**硬禁区 10**），已用户批准；「这次不做什么」记在 [`安全模型.md`](安全模型.md) §8。
> 验证：自检组 `128-scale-target`（替身 + 真探测器两路）。

### ❌ 2. 长任务上下文断层（架构级）

**现象**：长任务里对话被裁剪之后，Agent 要**重新侦察**一遍（再读一遍文件、再列一遍目录）。
**根因**：裁剪只保留最近若干条消息，没有「跨轮次稳定的项目事实」这一层。
**方向**：需要 Personal Context 那种系统（把「这个仓库是什么/关键约定/当前进度」放在裁剪之外
且可增量更新）。这是架构级改动，不是补丁 —— **先不做**。
**关联**：`core/compact.cjs` 现在只做「压缩最近对话」，没有跨轮次的稳定层。

### ✅ 3. 自动化测试不是默认动作（2026-10-07 已做 · 用户授权，最保守版）

**现象**：用 Harbor 做新项目时，「跑测试」是**想到了才做**，不是流程强制的。
**根因**：内核有「跑没跑测试」的判读（`core/task-outcome.cjs`），但没有**默认动作**：
新项目里没有测试脚本时，没人提醒「要不要先建一个最小验证」。
**当时的顾虑（现在写进了文案）**：不能变成「每个任务都逼着跑测试」—— 那会把闲聊任务也拖下水。
**落地（保守版，与 `SCALE_RULE` 同一套做法）**：`core/prompt-rules.cjs` 加一条 `TEST_RULE`
（**不跟任何开关**）—— 改了代码就跑到绿、项目里没测试就先建一个最小的。**尺度写死在文案里**：
末一句明说「纯问答 / 只读 / 闲聊**不适用**」。没动内核行为（不加门禁、不拦工具），
只是把「动手前先想到验证」写进系统提示。自检 `131-test-prompt`（7 项）。

### ✅ 4. 测试脚本自身的可靠性（2026-10-07 已做 · 用户授权）

**现象**：2026-10-03 这一夜，我自己的真机脚本连着三轮报的是**脚本的错**而不是产品的问题：
`function(){}` 漏 `return`（判据恒 undefined）、模板字符串里的 `\n` 变成真换行（页面里报
`SyntaxError: missing /`）、写在字符串里的 `require('./x')` 被自检 12 组当成真引用、
清理脚本按命令行匹配时把**调用它的自己**杀掉、假模型 server 不关导致 node 不退出、
环境变量在 `cmd`/PowerShell 两层里传不进去。每一轮都要人肉分辨「这次是真的还是脚本又错了」。
**落地**：`tools/shot/cdp.mjs`（已有的共享库）补齐可靠性三件套 —— `checkExpression`
（发出去之前先编译，语法错当场拦）、`evaluate`（页面异常只留第一行 + 带 label；返回 undefined
时警告「是不是漏了 return」）、`makeCleanup`（退出倒序关子进程 / 连接）、`findPageTarget`
（连不上端口报错带「怎么办」）。`tools/shot-electron.mjs` 删掉自带的那份 CDP 客户端改用共享库；
新增 `tools/cdp.mjs`（真机探针 CLI，把散在 `tmp/` 的一次性脚本收成仓库内一等工具）。
自检 `130-cdp-helpers`（13 项，纯逻辑）。**`tmp/` 里那几十个旧脚本没清**（它们是 gitignore 的
一次性产物），以后按新工具写。

### 顺带记：自检组加了文件但忘了注册 = 静默不跑

2026-10-03 新加 `scripts/selftest/groups/111-confirm-notify.mjs` 时，本地跑完发现
总数没变 —— `selftest.mjs` 的 `GROUPS` 是**显式清单**，光放文件进去不会跑，也不报错。
已补：`12-requires.mjs` 里加了「`groups/` 下每个组都挂在清单上」的断言（`*-parts.mjs` 除外）。

## 2026-10-07 · 架构收敛（外部计划 —— **已全部落地**）

> 来源：外部「Harbor Agent 1.30+ 短期维护与架构收敛计划」。
> **完整规格**（对象关系 / 每项要动哪些文件 / 怎么验）在 [`Runtime领域模型.md`](Runtime领域模型.md) ——
> 这里只记一笔，不重复明细（本项目「同一件事写两份」栽过跟头）。
>
> 四项（Decision 统一 / Action Model / Event Contract / Risk-Scale 拆分）**都触发**
> `AGENT.md` 第 3 节的硬禁区（改 3+ 文件 / 动内核关键文件 / 改数据结构 / 动安全模型），
> 所以当初是「规格不是待办：逐项批准后再动手」。
>
> 对代码核过、**已完成别重做**的三项：防 Task 膨胀（已升格成硬禁区 11）、
> Subagent 只读（`core/subagent.cjs`）、测试数据隔离（`data/selftest-data` / `data/unit-test-data`）。
>
> **进展（2026-10-07）**：4.1 Decision 统一、4.3 Event Contract（`core/event-types.cjs` 一处清单
> + 自检 `126`）、4.2 Action Model **全 4 步**（第 4 步「Permission 收到 Action 上」做的是**保守版**：
> 只把各层门的**结论**回填到 `Action.permission`，判据与触发一字未改 —— 见 `安全模型.md` §8）、
> 4.4(b)(c)、P4-1（规模闸门按实际规模）**均已落地**。
> 四项收敛**全部完成**；详见 `Runtime领域模型.md`。



# 另外一台电脑（非开发机器）的 1.31.0 版本的 Harbor 检查问题记录

- 记录时间：2026-10-04（初版 05:50 ／ 深度检测 05:55 ／ 第 3 轮深究 06:05 ／ 第 4 轮深究 06:15 ／ 第 5 轮深究 06:30 ／ 第 6 轮深究 07:00 ／ 第 7 轮插件/MCP ／ 第 8 轮文件范围/密钥 ／ 第 9 轮生图/项目规则 ／ 第 10 轮定时任务 ／ **第 11 轮 window/tray 与收尾 06:20 ／ 第 12 轮前端 dist 面 06:35 ／ 第 13 轮配置数据面、提示词装配、搜索、记忆链、tool-output 落盘 06:55**）
- 检查对象：`I:\Harbor`（便携版安装目录，非源码仓库；程序源码以可读文本放在 `resources\app\electron\`，前端为已构建的 `resources\app\dist\`）
- 检查方式：只读核对 —— 版本 / 权限开关 / 备份与日志保留策略；第 2 轮源码逐处读证；第 3 轮追加安全面（导航策略 / preload / 凭证加密 / 会话加密 / 脱敏链）读证 + 正则实测；第 4 轮追加磁盘增长面（缓存 / 会话 / 崩溃转储 / 前端脱敏 / 空壳备份界面）读证；第 5 轮追加 IPC 入参面（139 通道逐一过筛：backup/session 名字穿越、超时杀进程树、shell 绕文件范围）读证；本轮同时清除了所有撤销/更正条目；**第 6 轮追加渲染层与网络面（外链 scheme 校验 / CSP / 网络策略接线 / 权限请求处理器）读证，前端 dist 为压缩产物、用本机 python 脚本扫描上下文取证**；第 11 轮追加 window/tray IPC 面（渲染层可达的窗口/托盘动作与退出路径）读证；第 12 轮全量扫描前端 dist 业务面（危险 API / 存储 / 粘贴拖放 / webview / iframe / 硬编码 URL / 遥测痕迹）取证；第 13 轮追加配置数据面（`data/config.json`、`credentials.json`、`capabilities.json` 实测值与源码默认值比对）与提示词装配面（prompt-stack / loop-prompt / prompt-env / context-builder）、搜索面（search / search-cache）、记忆链（memory-store / memory-recall）、tool-output 落盘面读证
- 说明：本文件汇总所有核对发现，含**已确认**、**待核实**、**搁置**三类，避免后续误用。

---

## 〇、环境基线（核对事实，非问题）

| 项目 | 实际情况 |
|---|---|
| 应用版本 | 1.21.0（与发布页最新一致，**无需升级**） |
| 内核 | Electron 44.3.0 |
| 构建时间 | 2026-10-03（commit b095f13b） |
| 权限档 | **完全访问（full）** |
| 备份 | 1 份（2026-10-04 05:17，`auto`）；保留策略：留最近 10 份，每 24 小时自动一次 |
| 日志占用 | 整个 `data/logs` 约 2.0MB（第 4 轮实测，仍在增长） |

`data\` 各子目录实测占用（本轮用脚本逐个目录求和，字节）：

| 目录 | 占用 | 谁在管清理 |
|---|---|---|
| chromium | 20.8 MB | Electron 自管，不必干预 |
| logs | 2.00 MB | **无人清理** ❌ |
| sessions | 1.27 MB（单文件已达 2.4 MB） | 保留是设计（会话要留）✅ |
| events | 1.50 MB（第 4 轮实测） | **无人清理** ❌ |
| audit | 0.90 MB | 30 天清理 ✅ |
| tasks | 24 KB | 任务台账，按会话增删 ✅ |
| changesets | 3 KB | 保留 50 份 ✅ |
| cache | 近 0（`prompt-diag/` 空） | **无兜底** ⚠️（问题 10） |
| crash | 近 0 | **无兜底** ⚠️（问题 11，潜在） |
| 其余（errors/stats/backups…） | 近 0 | 各有兜底 ✅ |
| 根目录 `debug.log` | 2.7 KB | **无兜底**（问题 12，仅记录） |

---

## 一、已确认的问题

### 问题 1：`data/logs` 里「按天的旧文件」永不删除 —— 只增不减

- **现象**：`data/logs` 下四类文件都没有「按天保留」清理：
  - 主日志 `2026-10-04.log`：一天一个，**无体积上限、无天数清理**（`core/log.cjs` 全文只有 `appendFileSync`，无 prune）；
  - 动作流水 `actions-*.jsonl`：单文件封顶 6MB（`core/log-actions.cjs:47` 有轮转），但**每天的旧文件永久保留**；
  - 工具输出 `tool-output-*.log`：每次大输出落一个，**无清理**；
  - `token-metrics.jsonl`：见问题 4。
- **证据**：全库搜「清理」相关（`prune`/`retention`/`KEEP`/`cleanup`/`rotate`/`purge`/`sweep`）—— 只有 errors(30 天)、audit(30 天)、backup(10 份)、changeset(50 份)、stats(90 天)、memory 过期项有兜底，**`logs` 一个都没有**。
- **影响**：当前不痛（约 2MB）。长期是**唯一会无限增长**的目录之一，随时间线性累积。
- **建议**：给 `data/logs` 增加与 `errors` 完全相同的「保留 30 天」清理（`error-retention.cjs` 可直接照抄）。改动轻。
- **状态**：✅ 已确认，**建议修**。

### 问题 2：「完全访问」档下，配置里设的 `ask` 策略不生效

- **现象**：配置中 `shellPolicy` 的 medium（构建 / 装依赖 / 改文件 / 联网等）与 high 档均设为 `ask`，但实际不弹窗。
- **证据**：权限判定只在权限档「需要确认」时才真正弹窗；当前 **full**，medium / high 被跳过，永远不问。
- **实际效果**：除非命中「会毁掉系统」的最高风险档，**其余操作一律不问、直接执行**。
- **讽刺点**：项目自身注释写明「最忌讳『声明了但不生效』」，当前设置恰是「配置写了 ask、实际不生效」。
- **建议**：不在逻辑、在**提示** —— 选 full 档时界面应明示「medium / high 不再询问」。
- **状态**：✅ 已确认，**建议加提示**。

### 问题 3：`data/events/` 无任何保留策略

- **现象**：`data/events/<日期>.jsonl` 一天一个文件，**只追加，从不清理**。
- **证据**：`core/events.cjs` 全文只有 `appendFileSync`（第 139 行），无 prune / 无 KEEP_DAYS；全库搜清理关键字也没有任何一处指向 `data/events`。
- **影响**：**比主日志增长更快** —— 一天就 1.20MB（`2026-10-03.jsonl`），且只增不减。
- **建议**：并入问题 1 的修复 —— 同一段「保留 N 天」逻辑，同时覆盖 `logs` 与 `events`。
- **状态**：✅ 已确认（本轮新增），**建议修**。

### 问题 4：`data/logs/token-metrics.jsonl` 无上限、无轮转

- **现象**：每请求一行指标，全写进**同一个文件**，永不分割、永不清理。
- **证据**：`core/token-metrics.cjs:46` 只 `appendFileSync`，全文无大小上限、无轮转、无清理。
- **影响**：这是唯一一个「连 6MB 轮转都没有」的日志文件 —— 会单文件无限变大（读它的 `summarize` 是**全量读入内存**，第 90-91 行，文件越大越慢）。
- **建议**：并入问题 1，或至少给它加与 `actions` 相同的轮转。
- **状态**：✅ 已确认（本轮新增），**建议修**。

### 问题 5：备份清单里的「记忆」指向过时路径 —— 记忆永远不会被备份

- **现象**：备份清单写的是 `memory.md`，但**真实记忆已经迁到 `memory.json`**。
- **证据**：
  - `core/backup.cjs:31`：ITEMS 里是 `{ name: 'memory.md', dir: false }`；
  - `core/memory-schema.cjs:39-41`：`filePath()` 返回 `data/memory.json`；
  - `core/memory.cjs:56`：老 `memory.md` 在启动时会被**改名**为 `memory.md.migrated`；
  - 实测 `I:\Harbor\data\` 根下**没有** `memory.md`，也没有 `memory.json`（当前记忆为空）。
- **后果**：`create()` 第 149 行 `if (!fs.existsSync(source)) continue` **静默跳过** —— 备份照报「成功」，只是少了记忆这一项；`restore()` 同样用 ITEMS，**恢复也恢复不到记忆**。
- **定性**：这正是项目自己最忌讳的「声明了但不生效」—— 备份说明里写着「备配置、记忆、对话、技能」，实际只备了其中的配置/对话/技能。
- **建议**：把 ITEMS 的 `memory.md` 改成 `memory.json`；顺带考虑把「跳过不存在的项」改成**记一条日志**，别静默。
- **状态**：✅ 已确认（本轮新增，**本轮最严重**）。

### 问题 6：`scaleMaxFiles` 配置项永不生效（文件数阈值是死条件）

- **现象**：配置里有「文件数 ≥ 阈值就硬拦」这一维，但**任何输入都不可能触发它**。
- **证据**：
  - `core/scale.cjs:124`：`files: null /* 预检不数文件 —— 数一遍本身就是它要拦的那种操作 */`（**写死 null**）；
  - `core/scale.cjs:151`：硬拦条件之一是 `(estimate.files !== null && estimate.files > maxFiles)` —— 因为上一行恒为 null，**这个条件恒为 false**；
  - `scaleMaxFiles` 全库唯一被「读」的地方是 `scale.cjs:57`，读了只喂给这个死条件。
- **佐证**：`core/scale-config.cjs:19` 注释自己承认 ——「scaleMaxFiles　能算出文件数时 ≥ 它就硬拦（**现在算不出来**，见下）」。
- **定性**：属于「已知未完成」被留在配置里。界面/配置里能改它，但改多少都不影响行为 —— 又一个「声明了但不生效」。
- **建议**：二选一 —— 要么在界面标注「暂未生效」（同问题 2 的做法），要么补上「读目录数文件」的实现。别让它长期当摆设。
- **状态**：✅ 已确认（本轮新增），**建议至少加标注**。

### 问题 7：规模授权判据过宽 —— 任何一次「答了」都会解锁规模授权

- **现象**：规模闸门拦下一次操作后，只要对话里**任意一次** `ask_user` 被用户回答，上一次被拦的规模操作就被记成「已授权」，之后**本对话内同类不再问**。
- **证据**：
  - `core/tools/scale-gate.cjs:95-104` `noteAsked()`：被调用就把 `kind` 加进 `grants`；
  - `core/tools/ask_user.cjs:213-221`：判定条件只是 `Array.isArray(reply?.answers) && reply.answers.length > 0`（**用户答了**），不看问的是不是规模问题；
  - `core/tools/scale-gate.cjs:232-235`：`granted()` 为真就直接放行（返回 `level: 'ok'`）。
- **后果**：模型在规模拦截后，如果先问了**一个无关问题**（比如「要什么风格」）并被回答，那次规模拦截就顺带解锁了。
- **与注释矛盾**：`scale-gate.cjs:90-91` 注明「范围含糊时**宁可下次再问一遍**」，但实现是「答过即本对话内永不问」—— **注释与代码不一致**。
- **建议**：把 `noteAsked` 的触发收窄到「刚被拦下之后那一次 ask」，或至少记下「被回答的问题是否与规模相关」。
- **状态**：✅ 已确认（本轮新增），**建议修**（严重度中：限于本对话内、且需用户实际点过回答）。

### 问题 8：记忆的「像不像密钥」判定比脱敏规则窄 —— 7 类密钥能写进记忆

- **现象**：密钥过滤（`memory-schema.cjs:48-49` 的 `SECRET_LIKE`）只认 5 种形态，比脱敏模块 `redact.cjs` 的规则窄一大截。
- **实测证据**（第 3 轮用本机 python 复现两套正则对撞，13 个样本）：**7 类密钥被记忆放行、而脱敏都能识别** ——
  `github_pat_…`、`AIza…`(Google)、`xoxb-…`(Slack)、`hf_…`(HuggingFace)、JWT(`eyJ….….…`)、`api_key=xxx`、`password: xxx`。
- **为什么后果不轻**：记忆**每轮都注入上下文**（`memory-store.cjs:16-17` 注释自己写的），且 `memory.json` **在备份范围内**（见问题 5）—— 漏一条密钥等于「每轮都泄露 + 落进备份」。这与 `redact.cjs:4`「Secret 不许离开凭证库」、`memory-store.cjs:75`「密钥一律不记」两条自我承诺都冲突。
- **根因**：项目里「像不像密钥」这套判断**存在三份实现**（`redact.cjs` 宽 / `memory-schema.cjs` 窄 / `diagnostics.cjs` 更窄，见问题 9），没有单一真相。
- **建议**：让 `memory-schema.cjs` 改用 `redact.cjs` 导出的判定（如 `redact.looksSecret(content)`），删掉自己那份 `SECRET_LIKE`。
- **状态**：✅ 已确认（第 3 轮新增，**实证**），**建议修**。

### 问题 9：诊断包自写脱敏规则（未复用 redact），且 `mask()` 是死代码

- **现象**：
  - `core/diagnostics.cjs:44-48` 的 `scrubLogLine` **自写**一套正则（只认 `sk-` / `Bearer ` / `api_key=`），**没有复用** `redact.cjs`；
  - `core/diagnostics.cjs:36` 的 `mask()` **全库零调用者**（第 3 轮搜 `mask(` 仅命中定义处），而该文件 `:10` 注释仍对外承诺「只报长度和前后几位」。
- **严重度：低**（已核实，**非实际泄露**）—— 诊断包读的日志**写入时已过宽规则 `redact`**（`log.cjs:62` / `log-actions.cjs:80` / `audit.cjs:85,90` / `error-observer.cjs:224` 四处都接了），日志里本无明文；`scrubLogLine` 只是「又窄了一层的第二道兜底」。
- **真正的风险**：两套规则会**漂移** —— 哪天改了 `redact` 而忘了这里，第二道就形同虚设。`redact.cjs:53` 自称「唯一对外主入口」，此处未照做。
- **建议**：`scrubLogLine` 直接调 `redact.redact(line)`；`mask()` 要么接上调用，要么删掉（当前 `configSummary` 只显示「已配置」，比 mask 更安全，留着反而误导）。
- **状态**：✅ 已确认（第 3 轮新增），**建议修**（低优先）。

> **第 4 轮补强（根因更清楚了）**：全库核对，「像不像密钥」这套判断**至少有 4 份实现**，宽窄不一：
> 1. `core/redact.cjs`（**最宽**，导出 `looksSecret()`，自称「唯一对外主入口」）；
> 2. `core/memory-schema.cjs:48` 的 `SECRET_LIKE`（**窄**，只有 `sk-` / `gh*_` / `AKIA` / 私钥块 / `Bearer`）；
> 3. `core/diagnostics.cjs:46`（**更窄**，只有 `sk-` / `Bearer` / `api_key=`，见问题 9）；
> 4. **前端** `resources\app\dist\assets\SettingsModal-*.js` 里的 `Kr` 正则组（**较宽**，连 `github_pat_` / `AIza` / `xox[baprs]-` / `hf_` / JWT 都认，用于界面显示与导出脱敏）。
>
> 更说明问题的是：**同一个仓库里已有正确样板** —— `core/schedule-store.cjs:108` 用的是**并集**
> （`SECRET_LIKE.test(sample) || redact.looksSecret(sample)`），而 `core/memory-store.cjs:76/151` **只用了窄的那半**。
>
> **第 13 轮补强（脱敏覆盖缺口又一实证）**：实测本机会话文件 `sess_muswe29o9bjdl.jsonl`，`read_file` 读 `credentials.json` 时返回的 **safeStorage 密文**（`djEw…==` base64）被**原样落进会话**（全文 6 处，工具结果与消息序列化里都有）——`redact` 的精确名单登记的是**解密后的明文**，模式层也不认 `djEw` 前缀密文形态，serializeLine 因此放行。实际风险低（DPAPI 密文脱离本机本账户不可解，攻击者同账户时读 credentials.json 本身更方便），但属于「脱敏覆盖承诺 vs 实际」的又一处不一致，与本文「凭证相关数据应过 redact」的自述不符。修法可并入「redact 做唯一真相」——把凭证库登记值的**磁盘密文形态**也纳入替换名单。
> 即：不是「没想到」，是**记忆这条链漏接了更宽的那道**。修法照 `schedule-store` 抄一行即可。

### 问题 10：`data/cache/prompt-diag/` 会话诊断快照只写不清 —— 删会话后成孤儿

- **现象**：`core/context-diag.cjs` 每轮对话会把上下文分层诊断快照写到 `data/cache/prompt-diag/<sessionId>.json`（每会话只留最后一份）。但**全库没有任何地方删它**。
- **证据**：搜 `chatCache` / `prompt-diag` 全库，只有 `context-diag.cjs:76`（写）与 `paths.cjs:58`（目录定义）两处；`session-write.cjs:132-147` 的 `remove` / `removeAll` **只删 `sessions/*.jsonl`**，不碰快照。
- **影响**：每用过一个会话就留一个小 JSON（几 KB 量级），**永不清理**；「删除全部对话」「删单条会话」都不回收。当前该目录为空（尚未触发），属**潜在线性累积**。
- **建议**：删会话时一并删对应快照（`remove` / `removeAll` 里加一行）；或给 `data/cache` 加一个「清理无主快照」的兜底。
- **状态**：✅ 已确认（第 4 轮新增），**建议修**（低）。

### 问题 11：`data/crash/`（崩溃转储）无保留策略

- **现象**：`main.cjs:56` 把崩溃转储目录指到 `data/crash`（`app.setPath('crashDumps', DIRS.crash)`）。转储文件落这里后**无任何清理**。
- **证据**：`DIRS.crash` 只在 `paths.cjs:61` 定义、`ensureDirs` 建目录、`main.cjs:56` 指路径 —— 全库没有针对它的 prune / 保留策略。
- **影响**：崩溃转储单个可达数 MB，且崩溃时往往连着产生。当前目录**为空**（至今未崩过），属**潜在**风险。
- **建议**：参考 `error-retention.cjs` 加一个「保留最近 N 份 / 总量上限」。低优先。
- **状态**：✅ 已确认（第 4 轮新增），**潜在，建议观察**。

### 问题 12：安装根目录的 `debug.log` 无人管理

- **现象**：`I:\Harbor\debug.log`（2.7 KB，20 行，时间戳 2026-09-25 ~ 09-28），内容全是 Chromium crashpad 的 `registration_protocol_win.cc:108 CreateFile: 系统找不到指定的文件。(0x2)`。它由 Electron/Chromium 写在**安装根目录**，应用没有接管（`logs` 路径虽被指到 `data/logs`，但这个文件仍留在根）。
- **证据**：`main.cjs:55` 只设了 `app.setPath('logs', DIRS.logs)`；根目录 `debug.log` 不在 `data/` 下，不受任何保留策略约束。
- **影响**：**极小**（频次低、体积小）。但它是唯一一个「落在 `data/` 之外、又不是安装文件、还会增长」的文件 —— 与项目自我承诺的「任何数据都落在 `<根>/data` 下」略有出入（好在不在 C 盘）。
- **建议**：可忽略；若在意，可加 `--disable-logging` 或把日志重定向到 `data/logs`。
- **状态**：✅ 已确认（第 4 轮新增），**仅记录，不建议改**。

### 问题 13：首启「空壳备份」被界面当成有效备份，且恢复它可能清空当前会话/技能

- **现象**：首次启动时自动备份先跑（`main.cjs:244` 在窗口创建后调 `maybeAuto()`），此时 `config.json` / `memory.json` 尚未落盘 → `create()` 对不存在的项**静默跳过**（`backup.cjs:149`），只留下 `sessions/`、`skills/` 两个**空目录**。这份「空壳」会一直留在备份列表里。
- **界面呈现（第 4 轮已读实前端 `SettingsModal-*.js` 的备份面板 `Ur`）**：
  - 每条显示「时间 + 来源徽标（自动/手动/恢复前）+ `Br(size)` + `items.join(', ')`」，**没有「内容为空 / 不完整」的任何提示**；
  - 空壳因此显示成一份**正常备份**（如「~0 B · sessions, skills」），并带可点的「恢复」按钮。
- **真正的风险（恢复逻辑）**：`backup.cjs:214-221` 恢复时对每个 ITEM **先 `rmSync(当前)` 再 `cpSync(备份)`**。空壳里 `sessions` / `skills` 目录**存在但为空** → 从它恢复会**删掉当前全部会话与技能**、换成空目录；而 `config.json` / `memory.md` 因备份里不存在被跳过（不动）。即「恢复到首启空壳」= **清空对话和技能**。
- **叠加问题 5**：因备份清单还写着 `memory.md`（应为 `memory.json`），记忆在任何一次恢复里都恢复不到。
- **建议**：① 备份时对「一个用户项都没拷到」的情况**不出备份**（或标记为「不完整」）；② 界面显式标注空/不完整备份并禁用/二次确认其「恢复」；③ 顺手修问题 5 的路径。
- **状态**：✅ 已确认（第 4 轮，由「待核实 1」升级而来），**建议修**（触发概率低，但后果重）。

### 问题 14：`backup:restore` / `backup:remove` 的 `name` 参数无路径校验 —— `..` 可穿越备份目录

- **现象**：`core/backup.cjs` 的 `restore(name)`（:206-208）与 `remove(name)`（:242-246）都是 `path.join(backupRoot(), name)` 后直接操作，**全库无任何校验**；`handlers/extras.cjs:163-164` 原样 `String(name)` 传入。
- **后果**：`backup:remove('..\\..\\xxx')` = **递归删除备份目录外的任意目录**（`rmSync recursive`）；`restore('..\\..\\xxx')` 会把外部目录内容拷进 `data/`。
- **对照（同一个坑、同项目已修过两处）**：`core/artifact.cjs:43-45` 有 `safeId` 白名单，注释自己写「不卡的话 `remove('../../sessions')` 就能删到别处去」；`core/skills.cjs:239` 的 remove 有 `path.dirname` 校验。**backup 漏了同一道。**
- **可达性（已核实）**：webview 被 `navigation-policy.cjs:81` 剥离 preload（`delete webPreferences.preload` + 强制沙箱），外部网页拿不到 `window.workbench`；正常界面只传 `list()` 正则过滤后的名字。**需渲染层被注入才可利用** —— 低概率，但属「防御网漏了一角」。
- **建议**：restore/remove 入口加与 `list()` 同款校验 `/^\d{8}-\d{6}(-\d+)?$/`，一行。
- **状态**：✅ 已确认（第 5 轮新增），**建议修**（低概率，修法一行）。

### 问题 15：session 通道的 id 无校验 —— `..` 可穿越 sessions 目录

- **现象**：`core/session-io.cjs` 的 `fileFor(id) = path.join(DIRS.sessions, \`${id}.jsonl\`)` **无任何校验**；`handlers/session.cjs` 的 load / append / updateMeta / remove 全部 `String(id)` 直传。
- **后果（比问题 14 面更宽，读写删三向）**：`session:remove('..\\..\\...')` 可删任意单个 `.jsonl` 文件；`session:append('..\\..\\...')` 可**向任意路径写** `.jsonl` 内容（会话消息落盘到目录外）；`session:load` 可读任意 `.jsonl`。
- **可达性**：同问题 14 —— 需渲染层被注入；正常界面传的是 `newId()` 生成的 `sess_xxx`。
- **建议**：`fileFor` 入口加白名单 `^sess_[a-z0-9]+$`（与 `newId` 生成格式一致），一处修完四个通道。
- **状态**：✅ 已确认（第 5 轮新增），**建议修**（低概率，修法一行）。

### 问题 16：超时与终端中止只杀 cmd.exe 外壳，孙进程成孤儿 —— `killTree` 没接到这两条链

- **现象**：`core/abort.cjs:8-11` 自己记录过历史教训①「`child.kill()` 在 Windows 上只杀直接子进程，cmd 死了以后孙子会变孤儿继续跑」，并为此写了 `killTree`（`taskkill /T /F`，:32-50）—— 但它**只接到了「模型 Stop」一条链**（`run_shell.cjs:122-131`）。
- **没接的两条**：
  - 模型侧**超时**：`run_shell.cjs:143` 把超时交给 `exec` 内置 `timeout` —— Node 内部只杀 cmd.exe；
  - 终端侧**超时 + 中止**：`shell.cjs:185-193` 超时 timer、`shell.cjs:239-261` abort/abortAll 都是 `child.kill()` —— 同样只杀 cmd.exe。
- **后果**：命令超时后提示「已终止」，**实际孙进程还在后台跑**（继续占资源 / 写文件 / 联网）；终端 Stop 同理。与「超时已终止」这一用户可见承诺不符。
- **建议**：两处超时/中止统一改走 `killTree`（照抄 `abort.cjs` 的 taskkill 写法）。
- **状态**：✅ 已确认（第 5 轮新增），**建议修**（后果真实，修法现成）。

### 问题 17：`run_shell` 不受文件范围（fileScope=workspace）约束 —— 边界可用一条命令绕过

- **现象**：`_shared.cjs:34-54` 的 `resolvePath` 对所有文件工具做 `capability.check`（工作目录外要授权）；而 `run_shell.cjs:114` 的 `cwd = args.cwd ? String(args.cwd) : ctx.workdir` **没有**任何 capability 校验；`permission.cjs:23-81` 只在工具抛 `PERMISSION_REQUIRED` 时介入，shell 不抛。
- **后果**：fileScope=workspace（默认档）下，「模型不能随便动工作目录外」的承诺可用一条命令绕过 —— `type C:\Users\...\.ssh\id_rsa` 属 low 风险、不联网、无规模特征，**全程无任何拦截**（full 档本来就不限，无差别）。
- **定性**：文件工具与 shell 工具的边界不一致。shell 无法彻底按路径沙箱（命令里可以写任意路径），但 **cwd 参数至少该过一道 `capability.check`**，并把越界访问记进审计 —— 现在连「留痕」都没有。
- **建议**：① cwd 先过 `capability.check`（复用 `_shared.resolvePath` 的写法）；② 审计里记录实际 cwd。是否给命令正文也加路径检查，需作者拍板。
- **状态**：✅ 已确认（第 5 轮新增），**建议修**（严重度中低，只影响 workspace 档用户）。

> **第 6 轮补注（同源缺口，影响更小）**：`handlers/pty.cjs:19` 的 `pty:start` 同样**原样收渲染层 cwd**（`payload.cwd || getWorkdir()`），无校验、无 capability 检查。但 PTY 是**用户自己敲命令**的终端面板，不是模型工具 —— 仅记录，不建议单独修。

---

### 问题 18：外链打开没有 scheme 白名单 —— 聊天里的 `file://` / 自定义协议链接会被系统打开（官方清单点名的「shell.openExternal 与不可信内容」）

- **现象**：主窗口 `setWindowOpenHandler`（`main.cjs:116-119`）对**任何** `window.open` / `target="_blank"` 一律 `shell.openExternal(url)` 后拒绝建窗 —— **不验 scheme**（对照：webview 的弹窗是**全拒**，`navigation-policy.cjs:114`；主窗口却是外抛给系统）。
- **渲染层的放行面（前端 dist 压缩产物，python 扫描取证）**：
  - markdown 链接校验正则 `et = /^(https?:|file:|\/|\.\/|\.\.\/|#|mailto:)/i`（`MessageList-*.js`）—— **显式放行 `file:`**，`[点我](file:///C:/Windows/system.ini)` 会渲染成可点链接；
  - 图片语法放行 `file:` 与 `data:image/`；搜索结果**引用链接**（`e.url`）渲染成 `<a target="_blank">` 时**完全无校验**。
- **后果（需用户点击，故中低危）**：点击 `file:///` 链接 → `shell.openExternal` → ShellExecute → 系统默认程序**打开本地文件**（如浏览器/阅读器，属信息暴露）；搜索来源的恶意 `ms-settings:` / `smb:` / 任意注册协议 → 拉起对应系统应用。
- **顺带发现的死路径**：`BottomPanel-*.js` 里终端链接的打开流程是 `confirm()` 后 `window.open()`（**无参**）再 `e.location.href = t` —— 而主进程恒返回 deny，`window.open()` 永远返回 null，走到 `console.warn('Opening link blocked')` —— **终端里点链接实际打不开**（功能性 bug，与上面「打开得太多」互为镜像）。
- **建议**：① `openExternal` 前加协议白名单（`http:`/`https:`/`mailto:`，照抄 `navigation-policy.decide` 的思路）；② `et` 正则去掉 `file:`；③ 引用链接渲染前过同一白名单；④ 终端链接 flow 改为 `window.open(t)`（让 handler 收到真实 URL）或走 IPC 外抛。
 - **修法依赖（第 12 轮修正）**：建议①并非单点 —— 外抛出口有**两处**：`main.cjs:117` 的 `setWindowOpenHandler`（`window.open`/`target=_blank` 路径）与聊天 markdown 链接的**另一条** `shell.openExternal` 路径（渲染层点击 markdown 链接走 IPC 外抛、主进程直接打开，不经过 window.open handler），**两处都要加白名单**；且 `setWindowOpenHandler` 已是「全收单点」（所有 `window.open` 一律 deny 后外抛），前端 BottomPanel 的 confirm+`opener=null` 实为**死路径**（永远拿不到新窗口引用，第 12 轮实证）。
- **状态**：✅ 已确认（第 6 轮新增），**建议修**。

### 问题 19：主窗口没有 CSP —— Electron 官方安全清单第 7 条缺失

- **现象**：`resources\app\dist\index.html` **无 CSP meta 标签**；`main.cjs:125` 用 `loadFile` 加载（没有响应头可设）；全 dist 扫描**无任何 `Content-Security-Policy` 字样**；主进程也没有对主窗口 session 设 CSP 头。
- **为什么值得修**：主窗口渲染层里 `window.workbench`（preload 桥）暴露**全部 139 个通道**，其中 `backupRemove` / `sessionRemove` / `auditClear` / `memoryClear` / `configReset` / `statsReset` 等**不弹确认**。当前消息渲染是自定义 tokenizer（节点类型表只有 text/code/link/image 等 16 种，**无 html 类型、无 `dangerouslySetInnerHTML`**），尚无活 XSS 通道 —— 所以这是**加固项，不是活漏洞**：一旦未来引入任何渲染漏洞，没有 CSP 时攻击面直接等于整个桥。
- **建议**：`index.html` 加 CSP meta，如 `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: file:; connect-src 'self'`（按实际需要收放）。
- **状态**：✅ 已确认（第 6 轮新增），**建议修**（加固，改动一行）。

### 问题 20：网络策略管不到 Agent 自己的 browse / search —— `net-policy` 的 `webview` 分支是死代码

- **现象**：`netPolicy.decide()` 全库只有两个调用者 —— `core/tools/run_shell.cjs:58`（kind:`shell`）与 `core/mcp-network-guard.cjs:117`（kind:`mcp`）；而 `net-policy.cjs:101-109` 里**为 `kind:'webview'` 写好的分支没有任何调用者**。
- **browse 工具的实际把关**（`core/tools/browse.cjs:23-41`）：`rejectInternal` **只验 http/https scheme**，localhost/内网**显式放行**（注释自述是刻意），并声称「真正的拦截交给导航策略」—— 但 `will-navigate` **不触发于程序化导航**（webview 的 `src` 赋值 / loadURL），所以工具驱动的打开**导航策略也管不到**。`search_web.cjs` 同样无任何 net-policy 检查。
- **后果**：`security.network.mode=deny` 或 `denyHosts` 配了也**拦不住 browse 访问 `http://127.0.0.1` / 内网任意 http(s)**，且正文会被读回模型上下文（注入→内网探测→内容外泄给模型的完整链路）。`describePolicy()`（`config-security.cjs:20` 自述「能管到的：应用自己发起的网络请求」）**与实现不符** —— 又是这个项目自己最忌讳的「声明了但不生效」。
- **建议**：`browse.run()` 里调 `netPolicy.decide({ kind:'webview', target:url, ctx })`，deny 抛错、ask 走确认（分支已写好，**接线即可**）；`search_web` 视后端同样接入或说明豁免理由。
- **补充（第 6 轮后端核查）**：`core/search.cjs:160` 的 `custom` 后端用用户在设置里填的 URL 模板直接 `fetchText`，**同样无 net-policy、无 scheme 校验** —— `deny` 档对它无效。与 browse 的区别：请求去向由用户配置固定，模型只能影响 `{query}` 替换部分，风险低一档；建议保存时至少提示「指向内网的模板不受网络策略保护」。模型 API（`llm.cjs` baseUrl）与生图属同性质用户配置出口，不接 net-policy 是**合理设计**（本地 Ollama `127.0.0.1` 场景），不列为问题。
- **状态**：✅ 已确认（第 6 轮新增），**建议修**（严重度中）。

### 问题 21：无 `setPermissionRequestHandler` —— webview 网页的摄像头 / 麦克风 / 地理位置等权限**默认自动批准**

- **现象**：全库（主进程 + handlers + core）**没有一处** `setPermissionRequestHandler` / `setPermissionCheckHandler`；内嵌浏览器的 webview 用 `partition="persist:agent-browser"`（`RightPanelHost-*.js`），该 session 同样没设处理器。
- **依据**（Electron 官方安全清单第 5 条原文）：*"By default, Electron will automatically approve all permission requests unless the developer has manually configured a custom handler."*
- **后果**：用户用 browse 打开的**任意网页**（多数是 https 安全上下文）可以不经任何询问拿到摄像头 / 麦克风 / 通知 / 地理位置等权限 —— 隐私风险，且第三方页面的通知会与本应用自己的任务通知混在一起。
- **建议**：`app.whenReady` 后对 `session.fromPartition('persist:agent-browser')` 设 `setPermissionRequestHandler`，**默认全拒**（或仅放行通知），符合「Agent 只读网页」的定位。
- **状态**：✅ 已确认（第 6 轮新增），**建议修**。

### 问题 22：`updateMeta` 整篇重写 —— 非原子覆盖 + 读不出的坏行被静默永久抹掉

- **现象**：改标题 / 改模式 / 改 workdir 时，`session-write.cjs:110` 走 `readLines → writeLines` 整篇重写，而 `session-io.cjs:91` 的 `writeLines` 是**直接 `fs.writeFileSync` 覆盖**（先截断再写）。同项目 `session-crypto.cjs:195` 已有现成的 `writeAtomic`（临时文件 + rename，注释自述「中途崩了不会留下半个文件」），但**只用在加密迁移里，日常写路径没复用**。
- **后果一（非原子）**：长会话文件可达数 MB，`updateMeta` 重写期间崩溃 / 断电 / 杀进程 → 文件截断，截断点之后的会话内容全部丢失。追加式写「崩了只丢最后一行」的保证在这里失效。
- **后果二（静默抹行，更隐蔽）**：`readLines`（`session-io.cjs:54`）对解不开 / JSON 坏行是**跳过并计数**（读取容错，本意正确）；但 `updateMeta` 拿这个「已跳行」的列表直接 `writeLines` 写回 —— 被跳过的行**永久删除**，用户只得到一条 `log.warn`，界面无提示。触发场景正是代码自己承认的「换了 Windows 账户、或把 data/ 拷到别的机器」：用户换环境后随手改个标题，解不开的那些行就永远没了。
- **建议**：`updateMeta` 不必整篇重写 —— meta 恒在第一行，读原文件只替换首行、其余原样保留，并复用 `writeAtomic` 落盘（现成函数，接线即可）。改动轻。
- **状态**：✅ 已确认（第 6 轮新增），**建议修**（严重度中：概率低，但一旦触发是用户数据不可逆丢失）。

### 问题 23：插件权限闸是「自声明」—— 不声明权限的插件在只读档 / 禁写 / 禁网约束下照跑，且代码运行在主进程

- **现象**：`plugin-tool.cjs:31-35` 以插件 **manifest 自声明**的 `permissions.network / permissions.write` 判定「有没有副作用」；不声明（或声明不全）→ `risky=false`。于是三道闸全部以自声明为前提而失效：
  - `:42` `allowWrite === false`（用户这轮说「只分析，不修改」）只拦**声明了 write** 的插件；
  - `:46` `allowNetwork === false` 只拦**声明了 network** 的插件；
  - `:50` 只读档只拦 `risky`（=自声明有副作用）的插件。
- **为什么这不是空想**：插件代码经 `require(entryPath)` **在主进程里执行**（`plugin-tool.cjs:13-14` 自述「插件本身在宿主进程里 require，它能用 node:fs 之类，真正沙箱化是 P2」；`plugins.cjs:139` 自述「P0 记录但不强制」）—— 主进程里可以 require 应用的 credentials / config 模块、读写任何文件、任意联网、摸 ipcMain，对应用内数据等于全权。且插件**不经过路径授权**（`runWithPathPermission` 只包内置工具，`index.cjs:248`）。
- **后果**：用户把权限档设成「只读」、或本轮说了「只分析不修改」，一个 manifest 没写（或谎写）权限的插件照样执行、照写文件、照联网，**审计里还是 ok:true**；界面提示「（无副作用）」与事实相反。第三方插件只需在 manifest 里少写两行即可绕过全部三档。
- **缓解（事实层面）**：插件没有安装入口（handlers / main.cjs 全库无插件管理），只能由用户手工放入 `data/plugins/`；只读档下模型也写不了新插件（write_file 被拦）。所以触发前提是「已装了一个谎报权限的插件」—— 概率低，但信任边界的承诺（三档权限）对它不成立。
- **建议**：fail-closed —— 未声明权限的插件一律按「有副作用」处理（readonly 拦、ask 确认、allowWrite/allowNetwork 同样适用）；首次加载插件时向用户做一次信任确认并落审计；真正沙箱化（独立进程 + 权限 API）是 P2。
- **顺带（轻观察，不算问题）**：`plugin-tool.cjs:64` 用 `if (!approval)` 真值判断，与 `index.cjs:59-62` 注释里的 fail-closed 铁律（`approval !== true`，曾因传 `{ok:false}` 被当真值同意）风格不一致；当前 `ctx.confirm` 包装层（`chat-confirm.cjs:117`）保证只返回布尔，所以**现在不构成漏洞** —— 但若未来 confirm 返回形状变化，这里会静默变松，建议顺手统一。
- **状态**：✅ 已确认（第 7 轮新增），**建议修**（严重度中低：需已装谎报插件；修法一行 fail-closed）。

### 问题 24：系统通知出口零脱敏 —— 命令原文与任务结论首行直进 OS 通知

- **现象**：`handlers/notify.cjs:57` 的 `notify()` 只做长度截断（标题 120、正文 400），**不过 `redact`**。两个内容源都是未经脱敏的内存原文：
  - `core/confirm-notice.cjs:56` `permissionAsk` 取确认卡 summary 第一行 —— run_shell 的 summary 是 `执行命令：${command.slice(0,120)}`（`run_shell.cjs:174`），**命令原文**（模型完全可能在命令里内嵌 token，如 `curl -H "Authorization: Bearer sk-…"`）；
  - `core/task-notify.cjs:76` `firstLine(task.result)` 取任务结论首行 80 字 —— `task.finish`（`task.cjs:141`）写 result 时**只有 `slice(0,4000)`、无脱敏**，模型复述的密钥会原样进台账再进通知。
- **为什么值得修**：同一项目对日志（`log.cjs:62`）、审计（`audit.cjs:85`）、会话（`serializeLine`）、导出（渲染层双脱敏+拒绝落盘）、诊断包（日志二道）**全部执行脱敏**，且 `redact.cjs` 文件头自述「日志、会话、备份、诊断包、导出、审计、错误信息——都要先过这里」；系统通知是**唯一零脱敏的对外出口**。OS 通知会留在 Windows 通知历史里，屏幕共享/录屏/投屏时直接暴露。
- **建议**：`notify()` 里对 title/body 各过一遍 `redact()`（一行改动）；顺带 `task.cjs:141` 写 result 时也过 `redact`（台账 args 已用 `scrubLight`，result 漏了）。触发前提是「窗口不在前台才发系统通知」，所以是低概率 × 低-中后果。
- **状态**：✅ 已确认（第 9 轮新增），**建议修**（改动一行；严重度低-中）。

### 问题 25：`assistant.model` 指向不在 provider 模型列表里的模型（第 13 轮新增）

- **现象**：`data/config.json` 实测 —— DeepSeek provider 声明 `models: ['deepseek-flash', 'deepseek-v4-pro']`，但 `assistant.model` 仍是 `deepseek-chat`（`config-defaults.cjs:116` 的默认值），**不在该列表里**；而 `config-defaults.cjs:104` 的默认模型列表本是 `['deepseek-chat', 'deepseek-reasoner']` —— 说明此机配置被 1.21.0 迁移或手改过模型列表，却没同步 `assistant.model`。另一台同源机器（Geekspace 档）模型列表（`gpt-5.6-sol` 等）与默认预设也不同，同类漂移。
- **为什么值得修**：`config-normalize.cjs:163` 对 `assistant.model` 只做 `str()` 兜底，**不校验「模型是否在该 provider 列表内」**；后果是模型选择 UI 与实际请求模型脱节（下拉显示 flash/v4-pro，实跑 deepseek-chat），若 API 端停用旧模型名则请求直接 4xx。
- **建议**：normalize 时若 `assistant.model` 不在对应 provider 的 models 列表，回落到列表第一项并在日志留痕（低危，迁移问题居多）。
- **状态**：✅ 已确认（第 13 轮新增），严重度**低**（DeepSeek 官方目前仍受理 deepseek-chat，当前能跑），建议修。

### 问题 26：`tool-output-*.log` 落盘零脱敏（第 13 轮新增）

- **现象**：`core/tools/_shared.cjs:117` `truncateWithLog()` 在 run_shell 输出超过 8000 字符时，把**命令输出原文**（`src`，未经任何处理）`writeFileSync` 直写 `data/logs/tool-output-<时间戳>.log`。`run_shell.cjs:167` 传入的 `parts` 是 stdout+stderr 原文。**全链无 `redact` 调用**。
- **实测**：`data/logs` 现有 21 个 tool-output 文件（2026-10-04 一天内生成），单个最大 1.4MB，合计约 2.3MB。内容即命令输出全文（本次检查自己的 findstr/dir 输出也在其中）。
- **为什么值得修**：`redact.cjs` 文件头自述「日志、会话、备份、诊断包、导出、审计、错误信息——都要先过这里」，且 log.cjs / log-actions / audit 四处都接了；tool-output 同处 `data/logs` 却是**零脱敏落盘口**。命令输出里完全可能出现密钥（`type config.json`、`echo %API_KEY%`、API 响应含 token、`findstr` 命中凭证行等场景）。这也修正问题 24 的表述 —— 系统通知不是「唯一零脱敏」的口，落盘面还有这一个。
- **建议**：`truncateWithLog` 写盘前对 `src` 过一遍 `redact()`（一行改动）。与问题 1 的保留策略一并修（同一处代码）。
- **状态**：✅ 已确认（第 13 轮新增），严重度**中**（脱敏覆盖网的破口；触发条件=输出>8000 字符，实际每天都会发生多次）。

### 问题 27：`memory.json` 非原子写 + 损坏静默归零（第 13 轮新增）

- **现象**：`memory-store.cjs:51-54` `persist()` 用 `fs.writeFileSync` 直接整篇覆写，**非原子**（对比会话层有 `writeAtomic`、changeset 层有原子写，这里没有）；且 `load()`（:37-49）JSON 解析失败时 `catch` **静默返回空 items**，不告警、不保留损坏文件 —— 之后任何一次 `add`/`touch`/`update` 都会把空列表写回，**损坏文件被覆盖，全部记忆永久消失且无任何痕迹**。
- **叠加风险**：问题 5 已证备份漏掉记忆（路径写错，备份里没有 memory.json）→ 记忆的兜底只有磁盘上这一份非原子写的文件，一旦在写盘瞬间崩溃/断电，无备份可恢复。
- **触发概率**：低（写盘窗口毫秒级，但 `touch()` 每轮注入后都写盘，触发机会每轮都有）；**后果**：中（用户长期积累的偏好记忆全丢，且静默）。
- **建议**：`persist()` 换成先写临时文件再 rename 的原子写（项目内 `writeAtomic` 已有样板）；`load()` 解析失败时保留 `.corrupt-<时间戳>` 副本并记 error 日志。
- **状态**：✅ 已确认（第 13 轮新增），严重度**低概率×中后果**，建议修。

---

## 二、搁置（属于设计取舍，非缺陷）

### 搁置 1：权限档选择（完全访问 vs 需要确认）

- **性质**：产品设计选择题，**不是 bug**。
  - **完全访问**：省事；代价即问题 2 —— 写文件、装依赖、联网都不询问。
  - **需要确认**：这些操作会询问，多几次弹窗，换来「每次动手前都可见」。
- **结论**：需用户权衡「省事」与「可见性」后拍板。

### 搁置 2：「聊久了会忘事」

- **性质**：属设计取舍，读配置无法证实或证伪。
- **结论**：搁置，不作为问题。

### 搁置 3（第 3 轮）：改动快照（changeset）存文件原文、不脱敏 —— 属设计必需

- **性质**：`core/changeset.cjs:146` 把被改文件**改动前的完整内容**存进 `data/changesets/<id>/files/`，全程不脱敏。
- **为什么不是缺陷**：`changeset.cjs:13` 注释写明存的是「**原文**」—— 回滚必须一字不差地还原，脱敏了就没法回滚。**刻意取舍**。
- **风险边界**：`data/changesets` **不在备份范围内**（备份只含 配置/记忆/会话/技能），不随备份外泄；只有用户主动把整个 `data/` 拷走才会带出。
- **结论**：搁置，不作为问题。

### 搁置 4（第 7 轮）：技能 permissions 声明不强制

- **性质**：`skills.cjs` 文件头（1–23 行）与 `buildPromptSection` 都**明确自述**「permissions 是声明，不是强制，拦不住任何一次工具调用」，真实闸门在工具层权限档 —— 代码诚实、文档一致，**不属「声明了不生效」**。
- **风险边界**：技能正文（SKILL.md）会被模型 read_file 后照做，属「用户手写的长期指令」，信任边界是用户自己；技能也只能用户手工创建（create 有目录名清洗、remove 有穿越校验）。
- **结论**：搁置，不作为问题。若未来想让声明变强制，需要工具层按「当前生效技能」收紧 —— 作者已在注释里写清，属待办而非缺陷。

---

## 三、逐轮核查后「确认无问题」的面

> 记这一节是为了**避免误以为到处是坑**：以下面均逐处读过源码，确认接线正常 / 设计正确，**无需改动**。

- **导航策略** `navigation-policy.cjs`：确实接上了（`main.cjs:110`，`getMode` 读 `general.browserNavigation`）；协议检查在 `allow` 短路**之前**（`:41` 先于 `:43`），`file://` / `javascript:` / `data:` 一律拦；webview 两层加固。
- **启动清扫** `boot-cleanup.cjs`：接上了（`main.cjs:236` 建窗口前扫遗留任务、`:275` 退出清理）。
- **凭证加密** `credentials.cjs`：真用 Electron `safeStorage`（Windows 下即 DPAPI）；纯 Node 环境退化为明文时**如实标记** `backend='plain'`、`available=false`，不假装加密。
- **会话加密** `session-crypto.cjs`：AES-256-GCM 逐行封装、**默认关**（避免静默重写全部会话）、迁移前先备份、原子写、幂等、拿不到密钥**抛错不回落明文**。
- **落盘脱敏（第一道）**：`log.cjs:62`、`log-actions.cjs:80`、`audit.cjs:85/90`、`error-observer.cjs:224` 都接了宽规则 `redact`。
- **危险命令识别** `risk-targets.cjs`：归一化后按 腿1动作 / 腿2目标 / 腿3绕行 三层判定，含真机漏判记录（曾 46 条漏 28）与自检组 `61-destructive` 锁定。
- **preload 桥** `preload.cjs`：白名单式 `contextBridge`，**未**暴露整个 `ipcRenderer`。
- 轻观察（**不算问题**）：`credentials.remove()` 的 `redact.forget` 只在 `plain` 后端下有效（`credentials.cjs:155`）；加密后端下实际 forget 的是密文、而登记进脱敏表的是明文，故旧值会**多留**在内存脱敏表里直到重启 —— 方向是「多打码」，偏安全一侧。
- **导出链**（第 4 轮新增）：脱敏在**渲染层**完成（`Xr/Zr` 按 `Kr` 正则组打码 + `$r` 残留检查），**有未打码的疑似密钥就拒绝落盘**；主进程 `export:saveText` 只负责写用户选定的文件。**合规**。
- **内存缓存** `cache.cjs` / `file-cache.cjs`（第 4 轮新增）：条数 + 总字节（默认 8 MB）+ 单条上限（256 KB）三道闸，`evict` 逻辑正确（刚 put 的永不立删、覆盖时先扣旧字节）；文件缓存额外用 mtime 失效，写操作显式 invalidate 兜住 Windows mtime 精度问题。**设计正确**。
- **数据体检** `data-doctor.cjs`（第 4 轮新增）：只读、不修不删，分 error / warn 两级，坏 JSON 当问题报而非抛错，会读尾巴而非整篇（`TAIL_BYTES=64KB`）。查的是**引用完整性**（不是保留策略），**设计正确、无需改**。
- **会话写链** `session-io/session-write.cjs`（第 4 轮新增）：追加式 JSONL；`serializeLine` 是**唯一序列化入口**，先字段级 `scrubStorage` → 再模式级 `redact` → **最后**才 `sealLine`（顺序正确，先加密就看不见内容了）；`readLines` 对坏行容错跳过。**设计正确**。（单会话文件可达数 MB 属用户数据、**保留是设计**；只是 `updateMeta` 需整篇重写、`readLines` 每次全量读入，长对话下属可接受的开销 —— 整篇重写的**原子性**与「坏行被跳过后再写回会静默删除」是例外，见问题 22。）
- **IPC 包装层与浏览器往返**（第 5 轮新增）：`register-handlers.cjs` 把 `ipcMain.handle` 统一包一层（成败/耗时进动作流水、失败进观察哨，139 通道全覆盖）；`preload.cjs` 是**命名方法白名单**（`window.workbench.xxx`），未暴露通用 invoke；webview 加固会 `delete webPreferences.preload` —— **外部网页拿不到任何桥**；`handlers/browser.cjs` 的往返按 requestId 配对、正文在**主进程**清洗（`browser-text.cjs`）、45s 超时。**设计正确**。
- **消息 markdown 渲染链**（第 6 轮新增，前端压缩产物经 python 扫描取证）：`MessageList` 是**自定义 tokenizer → React 元素**，节点类型只有 text / code / link / image / table 等 16 种，**无 html 类型、无 `dangerouslySetInnerHTML`、无 `innerHTML` 赋值** —— 模型输出 / 文件内容渲染成 DOM 时不存在 HTML 直通。**设计安全**。（配套风险在问题 18/19 —— 链接 scheme 与外层缺 CSP，属外围加固，不是渲染链本身。）
- **内嵌浏览器隔离**（第 6 轮新增）：webview 用独立分区 `persist:agent-browser`（`RightPanelHost`）、`sandbox/contextIsolation/nodeIntegration=no` 全开、preload 被主进程剥离（见上一条）；主窗口 `webPreferences` 是 `sandbox + contextIsolation + nodeIntegration:false` 标准组合。**基线正确**。
- **网络栈选择** `http.cjs`（第 6 轮新增）：`net.fetch`（读系统代理，探活用）/ `fetchStream` 走 undici（避免流卡死拖死主线程）的分工有真机案例支撑。**设计正确**。
- **无自动更新**：便携版设计，全库无 autoUpdater —— 属产品取舍，不列问题。
- **MCP 执行链**（第 7 轮新增）：工具名 `mcp__*` 在工具咽喉**按写操作对待**（`index.cjs:64-110`）：只读档拒、ask 档确认、`approval !== true` 才放行（fail-closed）；返回值统一标「外部进程内容不可信」；`mcp-connection.cjs` 环境变量默认只给白名单最小集（PATH/SystemRoot 等）、`inheritEnvironment` 时过滤密钥名（`isSecretEnvKey`）、调用超时 + 中断信号一路带到子进程、进程退出释放全部挂起请求、网络守卫在启动前拦「声明不给网络却联网」的服务器。**设计正确、覆盖完整**。
- **技能系统**（第 7 轮新增）：`skills.cjs` 从头到尾**如实声明**「permissions 是声明不是强制、拦不住任何工具调用」，真实闸门在工具层权限档；frontmatter 解析容错（坏文件不搞垮整个列表）、权限写错时挂在 `permissionError` 上界面标红并提示词写明「声明无效」；create 清洗目录名、remove 校验直接子目录防穿越。**设计诚实、无需改**（声明不强制见搁置 4）。
- **文件范围执行点**（第 8 轮新增）：`capability.cjs` 用 `realpath`（往上找最近存在祖先再拼回）破软链接/目录联接绕过前缀比较、敏感路径表 12 条（.ssh/.env/浏览器 Profile/凭据文件）、once 授权用后即销、session 级 12 小时过期、永久授权可撤、清单上限 200 条；`risk-gate.cjs` fail-closed（没确认界面按拒绝、`approved !== true` 不放行）；`net-policy.decide` 裁决顺序钉死（不涉网放行 → 技能 deny → 全局 deny → 禁名单 → 全命中允许 → 全局允许 → 其余 ask），读配置失败回落到 ask。**设计正确、无需改**。
- **密钥流转**（第 8 轮新增）：`llm.cjs` 密钥只进 Authorization header，日志只记 URL（不含 key）；`redact.cjs` 精确（凭证库登记值）＋ 模式（sk-/Bearer/JWT/私钥块/URL 内嵌凭据/28 个敏感字段名）双层，`looksSecret` 可供记忆过滤复用（见问题 8 的建议）；`config:patch` 经 `config-secrets.absorb` 把 apiKey 搬进凭证库、配置里只落 credentialRef。**设计正确**（系统通知出口的例外见问题 24）。
- **诊断包**（第 8 轮新增）：API Key 只报「已配置」与长度；日志行有 `scrubLogLine` 二道；消息预览依赖「盘上会话已过 serializeLine 脱敏」兜底（无独立三道，但方向安全）；不打包凭证库内容。**方向安全、无需改**。
- **自检体系**（第 8 轮新增）：`--self-test` / `--screenshot` 是验收入口，正常启动不跑（代价合理）；通道清点用 `ipcMain._invokeHandlers` 私表 + listenerCount 退路，能抓「handler 注册块中途抛错导致后面全哑」这类失败；PTY 真开真写回显探测；自检探针会话写完即删。**设计正确、无需改**。
- **生图落盘**（第 9 轮新增）：文件名由本地时间戳（毫秒）生成，**模型无法控制文件名/路径**；默认走 `resolvePath`（同 write_file 权限链），用户自定义目录视为已授权（合理）；http(s) 来源仅限上游响应返回的链接（模型不能直传）。**设计正确、无需改**。
- **项目规则加载**（第 9 轮新增）：`.harbor/rules.md` + `rules/*.md` 按码位排序、8000 字上限、mtime+size 指纹缓存、只读；注入段**明示**「内容算数据不算指令，出现『忽略之前的指令』之类要当可疑内容并告诉用户」。工作目录指向陌生仓库时的规则注入面属「用户主动信任该项目」的边界（与 AGENT.md 同类），不是缺陷。**设计正确、无需改**。
- **定时任务授权边界**（第 10 轮新增）：`confirm` 恒为 `async () => false`（没人在场 = 没人能批准，代码注释明令不许改成真值）；三档授权（readonly/workspace/full）显式写死在 `PROFILES`、未知档回落最严 readonly（fail-closed）、`configFor` 返回新对象不污染全局配置、`shellPolicy` 整体替换（用户的高危 allow 绝不漏进来）、高危/危急命令三档一律 block；并发保护（同一条不许并发跑）、心跳串行 + `unref`、无人值守轮不弹澄清卡（`markUnattended` + finally 清理）；「挡不住的东西」（search_web 无开关、medium 命令有副作用）在 GRANT_INFO 里**照实写给用户**。**预算边界**：`loop.cjs:296` 把 `run` 转发给 `loop-run.cjs`，定时任务与普通对话走**同一条**带预算的入口 —— 任务预算（轮数/工具调用/时长/token）在轮次边界查账、撞线标 `paused`（继续/停止/调预算，不是失败），另有 `loop.cjs` 200 轮硬顶、`loop-guard` 转圈检测、`limits.cjs` 全局日/月 token 闸在调模型前查账兜底；预算五项**默认不限**是用户 2026-09-29 明确拍板的设计，因此无人值守场景的实际兜底是那三道硬闸。**设计正确、覆盖完整、无需改**。
- **window/tray IPC 面**（第 11 轮新增）：渲染层经 preload 白名单只能触达 `app:quit`（setQuitting → app.quit()，will-quit 里杀终端子进程 + 显式 destroy 托盘，收尾有序）、`app:showWindow`（restore+show+focus）、`window:titleBar`（只喂 setTitleBarOverlay 配色，非法值被 try/catch 吞掉并回退默认色，不是 HTML 注入面）三条窗口通道；托盘实例仅主进程持有（tray.cjs），菜单只有「显示窗口/退出」两项，**没有任何渲染层通道可操作托盘**、也没有「隐藏到托盘」类 IPC；关窗行为由 `general.minimizeToTray` 配置控制（关窗只 hide、真正退出走托盘菜单）；sandbox + contextIsolation 下渲染层拿不到裸 ipcRenderer，未暴露通道不可达。**设计正确、无需改**。
- **前端 dist 业务面**（第 12 轮新增，全量 32 个资产文件 python 扫描取证）：`innerHTML`/`dangerouslySetInnerHTML`/`eval`/`new Function`/`document.write`/`postMessage`/`atob` 等命中**全部是 React/Fluent/xterm/zod 框架内部代码**，业务代码零命中；`fetch` 仅 3 处（脚本预取库 + 解析用户粘贴的引用列表字符串，无任意 URL 请求面）；localStorage 仅做旧键名迁移（`codex-workbench:* → personal-agent:*`），**不存任何密钥/凭证**；**无** onPaste/clipboardData/dataTransfer/iframe/webview 元素/裸 `ipcRenderer`/`process.env`/`nodeIntegration` 命中；硬编码 URL 仅模型端点预设（openai/deepseek/moonshot）、schema 命名空间、bing.com 默认搜索引擎 —— **前端代码零遥测、零密钥字样、无 redact 副本（脱敏收敛在主进程）、无 node 能力痕迹**。终端链接打开流程的死路径性质已并入问题 18（第 12 轮实证）。**设计正确、无需改**。

---

## 四、结论与优先级

按「现在就能修 + 确实是缺口 + 改动轻」排序：

1. **问题 5 —— 备份漏掉记忆（memory.md → memory.json）**：一行路径改错，代价最大（用户以为有备份、其实没有记忆）。**优先修。**
2. **问题 8 —— 记忆密钥过滤太窄（已实证 7 类漏网）**（第 3 轮新增，第 4 轮补强）：改用 `redact.looksSecret` 即可，改动轻（`schedule-store.cjs:108` 已有并集样板）；后果是漏网密钥「每轮注入 + 落进备份」。**与问题 5 并列优先。**
3. **问题 13 —— 首启空壳备份可被「恢复」成清空会话/技能**（第 4 轮新增）：触发概率低（要用户去点恢复那份首启备份），但后果重（删掉全部对话与技能），且与问题 5 叠加。建议与问题 5 一并修。
4. **问题 1 / 3 / 4 —— `logs` + `events` + `token-metrics` 无清理**：三处同一类缺口，可复用 `error-retention.cjs` 的「保留 30 天」逻辑一次修完。`events` 增长最快（一天约 1.5 MB，实测仍在涨）。
5. **问题 9 —— 多套脱敏规则并存 + 死代码 `mask()`**（第 3 轮新增，第 4 轮补强为「≥4 份实现」）：根因同问题 8，一并按「`redact` 做唯一真相」修；严重度低（无实际泄露）。
6. **问题 16 —— 超时/终端中止只杀 cmd.exe 外壳，孙进程成孤儿**（第 5 轮新增）：后果真实（「已终止」后命令还在跑），修法现成（照抄 `abort.cjs` 的 `killTree` 接到超时与终端中止两条链）。
7. **问题 17 —— `run_shell` 绕开文件范围档**（第 5 轮新增）：workspace 档的边界承诺可用一条命令绕过；至少 cwd 该过 `capability.check` 并留痕。
8. **问题 20 —— 网络策略管不到 browse/search，`webview` 分支是死代码**（第 6 轮新增）：`security.network` 设了 deny/denyHosts 也拦不住模型浏览内网/localhost 并把内容读回上下文；修法是接线（分支已写好）。严重度中。
9. **问题 18 —— 外链打开无 scheme 白名单**（第 6 轮新增）：`shell.openExternal` 不验协议 + markdown 正则放行 `file:`，聊天里的本地文件链接可被系统打开；顺带终端链接打开流程是死的（功能性 bug）。修法在渲染层正则与 main.cjs 各加一行。
10. **问题 21 —— webview 权限请求默认自动批准**（第 6 轮新增）：网页可未经询问拿摄像头/麦克风/通知/地理位置（Electron 官方默认行为，官方清单第 5 条）；修法：给 `persist:agent-browser` session 设处理器默认全拒。
11. **问题 19 —— 主窗口无 CSP**（第 6 轮新增）：加固项（当前渲染链无活 XSS），一行 meta；防的是未来任何渲染漏洞直接摸到全桥。
12. **问题 22 —— `updateMeta` 整篇重写：非原子 + 静默抹坏行**（第 6 轮新增）：改标题时崩溃/断电会截断会话文件；某行解不开时（换机器/换账户）改个标题就永久删掉该行。修法现成（只替换首行 + 复用 `writeAtomic`），后果是用户数据不可逆丢失。
13. **问题 23 —— 插件权限闸是自声明**（第 7 轮新增）：manifest 谎报/漏报权限即可让插件在只读档、禁写、禁网约束下照跑（代码在主进程，全权），审计还记 ok。修法一行 fail-closed（未声明按有副作用），首次加载加信任确认。
14. **问题 24 —— 系统通知出口零脱敏**（第 9 轮新增）：run_shell 命令原文与任务结论首行直进 OS 通知（留在通知历史、屏幕共享可见），是唯一零脱敏的对外出口；修法一行（`notify()` 里 title/body 过 `redact`，顺带 task.result 落盘也过）。
15. **问题 6 —— `scaleMaxFiles` 是死配置**：影响小，但属「声明了不生效」，建议至少加界面标注。
16. **问题 7 —— 规模授权判据过宽**：限本对话内，建议收窄触发条件。
17. **问题 2 —— full 档下 ask 不生效**：逻辑无需改，补界面提示即可。
18. **问题 14 / 15 —— backup 与 session 的 name/id 无校验、`..` 可穿越**（第 5 轮新增）：低概率（需渲染层被注入），修法各一行（正则白名单）；同项目 artifact/skills 已有样板，属「防御网漏角」。
19. **问题 10 —— `data/cache/prompt-diag/` 孤儿快照**（第 4 轮新增）：删会话时一并清即可，改动一行；属潜在线性累积。
20. **问题 11 —— `data/crash/` 无保留策略**（第 4 轮新增）：潜在，建议照 `error-retention` 加清理；低优先。
21. **问题 12 —— 安装根目录 `debug.log`**（第 4 轮新增）：体积极小，**仅记录、不建议改**。
22. **问题 25 —— `assistant.model` 不在 provider 模型列表**（第 13 轮新增）：严重度低，见第一节。
23. **问题 26 —— `tool-output-*.log` 落盘零脱敏**（第 13 轮新增）：命令输出原文直写 `data/logs` 不过 redact，一天已 21 个文件约 2.3MB；与问题 24 同类破口、与问题 1 同一处代码，建议一并修。
24. **问题 27 —— `memory.json` 非原子写 + 损坏静默归零**（第 13 轮新增）：与问题 5（备份漏记忆）叠加后记忆无任何兜底；修法照项目内 writeAtomic 样板。
25. **搁置 1 / 2 / 3 / 4（权限档、记忆取舍、改动快照、技能声明不强制）**：设计选择题 / 设计必需，待用户拍板或已归档。
26. **第三节「确认无问题」清单**：导航策略 / 启动清扫 / 凭证加密 / 会话加密 / 第一道脱敏 / 危险命令识别 / preload / 导出链 / 内存缓存 / 数据体检 / 会话写链（updateMeta 例外见问题 22）/ IPC 包装层与浏览器往返 / **消息 markdown 渲染链、内嵌浏览器隔离、网络栈选择、网络出口（第 6 轮：无遥测、无更新检查、模型 API 不接 net-policy 属合理设计）/ MCP 执行链、技能系统（第 7 轮）/ 文件范围执行点、密钥流转、诊断包、自检体系（第 8 轮）/ 生图落盘、项目规则加载（第 9 轮）/ 定时任务授权边界（第 10 轮）/ window/tray IPC 面（第 11 轮）/ 前端 dist 业务面（第 12 轮）／ 配置数据面（第 13 轮：凭证 safeStorage 加密生效、授权台账留痕；会话里出现凭证密文原文的覆盖缺口见问题 9 第 13 轮补强）／ 提示词装配面（第 13 轮：层序稳定区在前省缓存、注入边界声明在 safety 层、模板与失败摘要均限长）／ 搜索面（第 13 轮：超时 20s、结果截断 400 字、数量钳制、缓存 10 分钟 TTL+字节闸）／ 记忆检索链（第 13 轮：范围隔离、注入上限、SECRET_LIKE 过滤，落盘原子性见问题 27）** —— **无需改动**。

---

*本记录仅为只读核对所得，未对 `I:\Harbor` 做任何修改。*
