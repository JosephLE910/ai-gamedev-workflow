# 执行者规则 Executor Rules

> 🏛️ **顶层主本（CANONICAL MASTER）· 仅由工坊主理人助手维护**（配套 `shared/WORKFLOW.md` §6）。
> 新项目从本主本拷贝到 `<repo>/shared/EXECUTOR_RULES.md`；通用条款（git 分层 / plan 修改权 / 代码质量 / 省 token）改进走 WORKFLOW §6 回提主本，别在项目副本里改。
>
> 执行 Agent 接到 plan 后先读 `shared/AI_ONBOARDING.md`，再读此文件。两边（Trae / Claude Code / Codex 等）通用。与 `PLANNER_RULES.md` 对等；架构全貌见 `shared/WORKFLOW.md`。
>
> ⚠️ **项目特定部分待填**（标 `<待定>`）：拷到项目后，由规划者按本项目类型/引擎/工具链替换「资源串行锁」「编辑器/工具链」「桥接」「日志」四节（占位符 `<repo>`=项目根、`<exec>`=执行者 worktree、`<RESOURCE_LOCK>`=锁文件路径）。其余是通用规则，可直接用。

## 工作目录（worktree 隔离）

> **分布式协作**：若你在自己的 clone / 自己机器上工作，开工前先 `git pull` 拿最新 `shared/`；完事后 `git push`。分支名建议带 plan、任务或 owner，避免和别的队友/AI 撞。下面的 worktree 规则主要用于同一台机器上多 Agent 隔离。

执行者与规划者**不共享 working tree**，各在自己的目录：
- **Claude Code 执行者**：独立 worktree `<exec>`（规划者常驻 `<repo>` main，共享同一 `.git`，互不抢分支）
- **Trae 执行者**：按 Trae 侧自己的目录
- 下方命令一律用 `$(pwd)` 自适应 —— **先 `cd` 到你的项目根目录**再执行

要点：
- 读规划者最新 plan：`git show main:plans/XX.md`（objects 共享，立即可见，无需 pull）
- 首次进入新 worktree 要重新编译/重装依赖（构建产物不随 worktree 复制）

## 完成与 Merge（人验认可）

执行者完成后**不直接 merge**。最终判定权在**人**。

**测试分工**（关键）：
- 🤖 **执行者**：功能正确性的**自动化**验证 —— 读状态/数据 log，证明"功能对不对"。**这套调试方法不变**。
- 🎮 **人**：玩法/手感/体验的验证 —— 亲自跑。判断"好不好/对不对味"。**AI 给不了这个判断，执行者不替代、不靠截图猜**。

流程：
1. **执行者**：完成功能 + 自动化验证（log/数据证明功能对）+ push + §执行经验
2. **人验**：实际操作，体验手感/玩法 ★
3. 人给反馈 → 执行者改 → 人再验（迭代循环）
4. 人满意 + 规划者审查（`git diff` 初版vs终版 + 提炼经验）→ **规划者** `merge --no-ff`

```bash
git checkout main
git merge --no-ff plan/XX-short -m "Merge plan/XX: <一句话>"
```

## Git 分支规则

**所有工作必须在独立分支上进行，包括代码、文档、配置、调研摘要。**
禁止在 main 上做任何改动，无论是否 commit。

```bash
# 1. 切到最新 main
git checkout main

# 2. 创建分支（命名规则：plan/XX-简短描述）
git checkout -b plan/01-bootstrap

# 3. 开工前读取 main 上的最新规划文件（规划者在 main 上维护 plan）
git show main:plans/XX-short-name.md

# ... 改代码、构建、跑、验 ...

# 4. 提交（用具体文件列表，禁止 git add -A；commit 总结要写清楚）
git add <具体文件...>
git commit -m "Plan XX: <一句话说清改了什么/达成什么>"

# 5. 推送（无 remote 就只本地 commit）
git push -u origin plan/01-bootstrap
```

禁止行为：
- 禁止 `git add -A` 或 `git add .`
- 禁止 force push 到 main/master
- 禁止在 main 分支上做任何改动（含文档、调研摘要、配置文件）
- 禁止 `git checkout main` 后直接编辑文件
- push 后不要创建 PR/MR，直接告诉用户

**Commit 总结最低纪律**：
- 标题一句话说清“改了什么 / 达成什么”，禁止 `update` / `fix` / `wip` / `改了点东西` 这类空话。
- 复杂改动写正文：为什么、影响范围、回归风险、未解决项。
- 关联 plan 编号；深入踩坑仍写回 plan §执行经验，commit 不替代执行经验。
- 提交前确认没有误带本机绝对路径、密钥、`.env`、构建产物或大二进制中间物。

## Plan 修改权（分层）

规划者只出**初版** plan。执行者可根据实际情况**直接改 plan**，不必照初版死磕——你在一线，实现细节你最懂（避免不专业指挥专业）。但分两层：

🔧 **实现层（自由改）**：方案、参数、代码结构、执行步骤顺序、技术选型细化
→ 直接改 plan 文件，在 §执行经验 说明改了什么、为什么。规划者 merge 时 diff 吸取经验。

🔒 **锁定层（改前协商）**：目标、验收标准、带 ⚠️ 的保护门槛（如 Step 0 gate）
→ 要改先在 `shared/PLANNER_EXCHANGE.md` 说明理由，等规划者/用户确认。
→ **禁止为"让测试通过"而放宽验收标准**（否则验收失去意义）。

## 代码质量（实现层自由改 ≠ 可以欠债）

🔧 实现层你自由改，但**抄近路不等于免费**。代码健康度是验收的第三维，分工：**规划者审、你来改**（见 `WORKFLOW.md`「§3.8 代码质量与重构」）。

- 抄了近路 / 留了重复 / 临时硬编码 → 在 **§执行经验注明**，让规划者判是顺手清还是记债，别闷着。
- 规划者审 diff 提的**小清理**（重命名/抽函数/去重）→ 你在**同分支** merge 前顺手改。
- **大重构**（拆上帝类/重组共享组件/换架构）→ 别在功能 plan 里顺手做（混了难审难回滚），单开 `REFACTOR` plan。
- 改**共享组件/类**（多处复用的）时尤其留意：你这分支看不到全图重复，规划者会兜底审。

## 多 AI 接力 / 接手他人未完成工作

接手一条已被别人或另一个 AI 改过的分支时，先稳住基线，别在混乱上叠混乱。

- **先识别最后已验证基线**：看 `git log`、plan §执行经验、测试记录，找最后一个通过实测/自动验的提交或状态。
- **识别当前未验证改动**：用 `git status`、`git diff --stat`、最近 commit 摘要看清哪些是假设、哪些已验。
- **不要连叠未验证修复**：一个 bug 一次只改一个假设，改完立刻验证，再提交下一步。没验证的“修复”叠起来会难审、难回滚、难定位。
- **需要回退时先保存**：用临时分支、stash、patch 等保存未验证工作；除非用户明确允许丢弃，不要用会直接抹掉别人工作的破坏性回退。
- **不要手改构建中间产物排查问题**：`Intermediate/`、`Build/`、`Library/`、`Temp/`、打包产物、手工注入包内容等通常是产物不是源。修源码/脚本/配置，然后干净重建。

## 资源串行锁  `<待定>`

> 若本项目有单机独占的重资源（编辑器/GPU/端口/设备）→ 用文件锁串行；没有则删除本节。
> 分布式 clone 下，每台机器的本地资源通常互不抢；真正要串行的是共享/难合并资源归属（场景、预制体、数据表、设计稿、部署目标等），动前在 `PLANNER_EXCHANGE.md` 登记。

```bash
# 拿锁（最多等 30min）→ 用资源 → 释放，循环要短
elapsed=0
while [ -f <RESOURCE_LOCK> ] && [ $elapsed -lt 1800 ]; do
    sleep 30; elapsed=$((elapsed + 30))
done
touch <RESOURCE_LOCK>
if [ $elapsed -ge 1800 ]; then echo "ERROR: lock timeout"; exit 1; fi

# --- 构建 / 运行 / 验证 ---

rm <RESOURCE_LOCK>
```

拿锁 → 用资源 → 释放，循环要短。不要锁着重资源做不相干的事。

## 编辑器 / 工具链  `<待定>`

> 项目类型确定后，替换为本项目的构建/运行/截图命令。例：
> - UE：`open *.uproject` + `Build.sh ... Mac Development`
> - Unity：编辑器路径 + batchmode 命令
> - Web/服务：`npm run build` / `npm run dev` / 端口

```bash
# <构建命令待填>
# <运行/启动命令待填>
```

## 桥接 / 自动化  `<待定>`

> 若用桥接（如 SoftUEBridge / Unity MCP）驱动运行时验证，在此写命令与端口。否则删除本节。

## 日志  `<待定>`

> 写本项目可靠的日志/遥测通道（哪种 log 可信、哪种不可靠），调试前先查 `LESSONS.md §DEBUG`。

## 经验库

读 `shared/LESSONS.md` **只读** §{工种标签} + §DEBUG 两章，别整文件加载（LESSONS 在涨，整读是浪费）。用顶部分类表跳到对应 §。

**领域实现卡住时**，查输入层提炼笔记 `CodeWorkshop/ai-gamedev-workflow/notes/`（不够再翻 `Books/` 原书）——前人经验，别硬猜。见 `WORKFLOW.md`「§4.2 输入层」。

若项目有 `PIPELINE.md` 或 `<DOMAIN>-<DOMAIN>-MESSAGE.md`，跨工种/跨仓库交付和接入前先读对应文件；交付后把位置、规格、遗留写回公告板。

## 省 Token 纪律（读取 / 输出 / 日志）

> 成本大头是"反复搬进上下文的背景"，不是你打的字。下面几条每次都省。

- **先定位再读**：用 `grep`/`rg` 找到行号，再用范围读（offset/limit），不要整文件读进来。
- **引用带完整路径**：`Source/.../Foo.cpp:142`，别只写文件名让 AI 全项目搜。
- **构建/测试 log 只取关键段**：`... 2>&1 | grep -iE 'error|warning|fail' | head -50`，别把几千行输出灌进上下文。
- **审查/对比用 `--stat` 起步**：`git diff --stat` 看全貌，再定向看某文件的 hunk。
- **二进制资产不展开**：查改动看路径/大小，不读字节。
- **输出给结论**：要 diff/步骤/结论就直说，别复述问题、别长篇铺垫。
