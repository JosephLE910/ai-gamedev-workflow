# Agentic Engineering 22 条技巧总结

> 来源：Matt Van Horn《Every Agentic Engineering Hack I Know》（2026年6月）
> 原文：https://x.com/mvanhorn/status/2061877533885473181

---

## 一、先规划，再动手（01-03）

| # | 技巧 | 要点 |
|---|------|------|
| 01 | 有想法先 `/ce-plan`，不直接开干 | 贴 issue、截图、Slack 讨论、设计稿；模糊时先 `/ce-brainstorm` |
| 02 | plan.md 是给 agent 看的，人扫一眼标题就行 | "Plans are for agents, you silly human." 计划是 leash，防 agent 抄近路 |
| 03 | 非代码工作也用同一条 loop | 战略文档、产品 spec、竞品分析、board update——先规划再执行 |

---

## 二、怎么把活高效喂给 agent（04-09）

| # | 技巧 | 要点 |
|---|------|------|
| 04 | 语音做主输入 | Monologue / Wispr Flow / Apple 听写；LLM 能补回含糊和卡壳 |
| 05 | 同时开 4-6 个 cmux 会话 | 一个写 plan、一个 build、一个做研究、一个修 bug，多线程调度 |
| 06 | 新终端标签页默认直达 Claude Code | 干掉 cd/手敲 claude 的摩擦 |
| 07 | 远程控制 + 邮箱入口 | `remoteControlAtStartup: true`；AgentMail 邮箱=新任务入口 |
| 08 | 跳过权限确认 | `bypassPermissions` + `skipDangerousModePermissionPrompt`；YOLO |
| 09 | Claude 管 plan，Codex 管 build | Claude: reasoning xhigh, fast off；Codex: reasoning xhigh, fast on |

---

## 三、agent 强不强，看你喂了多少上下文（10-15）

| # | 技巧 | 要点 |
|---|------|------|
| 10 | `/ce-plan` 前先跑 `/last30days` | 并行搜 Reddit/X/YouTube/GitHub/Web，站在社区最近 30 天经验上选型 |
| 11 | 会议原始 transcript 直接扔进去 | 别替模型总结；Granola 录音→原样喂→proposal 当晚发出 |
| 12 | 多 agent 同时跑，你负责给信号 | "agents supply volume, you supply taste"——人做 react-and-redirect |
| 13 | 视频走同一条 loop | HyperFrames：script.md → agent 渲染 MP4 |
| 14 | 笔记做成 agent 知识库 | Bear/Obsidian/gbrain/supermemory；本质是 Personal RAG |
| 15 | "随时随地工作"靠 Mac mini + 远程 | Mosh(抗差网) + tmux(抗断线) + Hermes/OpenClaw(自治远程) |

---

## 四、让 agent 走出终端，接管真实工作（16-20）

| # | 技巧 | 要点 |
|---|------|------|
| 16 | plan.md 给 agent，Proof 给同事 | 把 plan/spec 丢进 Proof 生成链接，同事可 inline comment，评论流回 agent loop |
| 17 | 任何做超过两次的事，写成 skill | 不让从零写，让 agent 先读跑通的 skill 再照着 scaffold |
| 18 | 开源贡献放进同一条 loop | /ce-plan + /ce-work → 数百个 PR 被合并进 Python/Go/OpenCV 等 |
| 19 | M5 Max + 64GB RAM 也扛不住 | 6 个 Claude + Codex 全天挂着；靠电池砖+Tesla充电器+禁止休眠 |
| 20 | Printing Press：给真实服务做 CLI | Agent Cookie 带登录态，agent 能操作网页服务（GitHub/Tesla/Instacart） |

---

## 五、最后提醒（21-22）

| # | 技巧 | 要点 |
|---|------|------|
| 21 | Agent 很容易上瘾 | "Building with agents is the greatest video game ever made." 要休息 |
| 22 | 这篇文章就是这么做出来的 | Talk → plan → build；已不用 IDE，不敲代码 |

---

## 总结 5 件事

1. Agentic Engineering = research → plan → build → review 默认流水线
2. 拉开差距的是**上下文**：截图、issue、会议录音、十年笔记稳定流进 agent
3. 人位置在上移：做调度，给信号、判断、品味、取舍
4. agent 拿到远程/邮箱/登录态/服务接口→从代码助手变成执行层
5. 越能 build 越要小心上瘾
