# Loop Engineering 实践指南

> 来源：[腾讯技术工程](https://mp.weixin.qq.com/s/YqIyL7uW4EV2r5HLDW7wcA)
> 作者：eliqiao
> 提出者：Addy Osmani（Google 工程师）

## 一句话摘要

Loop Engineering 是继 Prompt Engineering、Context Engineering 之后的"AI 编程第三次革命"。核心理念：让人从循环内的操作者变成循环之上的监督者——你定义"做什么"和"何时算完成"，AI 自己决定"怎么做"和"下一步是什么"。

## ReAct vs Loop Engineering

| 维度 | ReAct | Loop Engineering |
|---|---|---|
| 关注层次 | 单次任务的执行过程 | 跨任务的编排与调度 |
| 状态管理 | 依赖上下文窗口内记忆 | 外置到文件/数据库，每次迭代全新上下文 |
| 停止条件 | 模型自己判断"做完了" | 独立评估器验证可度量条件 |
| 验证机制 | 自我检查（同一模型） | 对抗验证（不同模型/独立评估器） |
| 运行周期 | 单次对话 | 可持续数小时甚至数天 |

**类比**：ReAct 是工人的工作方式（砌墙-检查-修），Loop Engineering 是项目经理（编排进度-质检-返工）。

Loop Engineering 不是替代 ReAct，而是在其上增加编排层：ReAct 是 Inner Loop，Loop Engineering 是 Outer Loop。

## 核心架构

### 五阶段循环

**Discover → Plan → Execute → Verify → Iterate**

- **Discover**：自动读取 CI 失败、issue、代码审查等信号
- **Plan**：分解目标为具体步骤（温度适中，避免过早收敛）
- **Execute**：执行代码编辑与工具调用（幂等、可回滚）
- **Verify**：通过测试、lint、类型检查等客观信号验证
- **Iterate**：失败则自动修复重试；成功则进入下一任务

### 状态外置哲学

所有状态存储在外部系统（文件/数据库），不依赖模型上下文窗口。每次迭代从全新上下文开始，从持久化文件中读取状态。

### 六要素

| 要素 | 作用 |
|---|---|
| 自动化 | 提供循环心跳，按计划或事件触发 |
| 工作树 | Git worktree 为每个 Agent 创建独立工作目录 |
| 技能（SKILL.md）| 固化项目知识，避免每次冷启动 |
| 连接器（MCP）| 打通 issue 系统、CI 等真实工具链 |
| 子智能体 | 写代码与检查代码分离，形成对抗验证 |
| 状态文件 | 记录进度，支撑断点续跑 |

## 在 CodeBuddy 中的实现

| Loop Engineering 要素 | CodeBuddy 实现 |
|---|---|
| 自动化 | `/goal`、`/loop`、Automations |
| 工作树 | Git worktree + Team 模式 |
| 技能 | Skills 机制 |
| 连接器 | MCP 协议 |
| 子智能体 | Task 工具 + Team 模式 |
| 状态文件 | Memory、CODEBUDDY.md、Rules |

### `/goal` — 条件驱动的持续工作

设置可验证的完成条件，跨多轮自动工作直到满足：

```
/goal all tests in test/auth pass and the lint step is clean
/goal all tests pass or stop after 30 turns
```

每轮结束后独立小模型评估器判断条件是否满足（三态：ok/not-yet/impossible）。评估器与执行 Agent 不同模型 → 天然对抗验证。

### `/loop` — 时间驱动的循环

按时间间隔重复执行，适合监控巡检：

```
/loop 3m 检查流水线是否跑完
/loop 30m 运行单元测试，有失败告诉我
```

### Automations — 跨会话定时任务

持久化定时任务（Recurring cron 规则 / Once 定时触发），不随会话消失。

## 与我们的体系的关系

我们的工作流已经覆盖了 Loop Engineering 的多个要素：

| 我们的实践 | 对应的 Loop 要素 |
|---|---|
| 🔒 lock 层放 plan + review，🔧 work 层放实现 | 对抗验证（规划者 ≠ 执行者） |
| PLANNER_EXCHANGE.md 跨规划者传递意图 | 状态外置（不是靠上下文窗口记） |
| PROJECT_STATE.md 记录「做过什么」「当前在哪」 | 状态外置（断点续跑） |
| git worktree 隔离执行者 | 工作树并行隔离 |
| PLANNER_RULES / EXECUTOR_RULES | Skills 固化协作知识 |

**可以借鉴的**：
- `/goal` 条件驱动循环：适合"跑通所有测试 → 修 → 再跑"类任务
- 对抗验证理念：执行者改完代码后，用不同 prompt 让 AI 审查（已在 PLANNER_RULES 里）
