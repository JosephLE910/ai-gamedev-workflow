# 代码工坊 · 协作蓝图

[代码工坊](https://github.com/JosephLE910/CodeWorkshop) 的多智能体协作体系核心文档。

## 这是什么

把"规划者 / 执行者 / 人"三方协作的整套 AI 开发流程抽象成一份**可复用蓝图**——新项目拷贝即可复用，不绑定特定 AI 工具或引擎。

## 文件

| 文件 | 是什么 |
|---|---|
| `AI_ONBOARDING.md` | 接入前门：任何 AI / 队友 / 工具先读这份 |
| `WORKFLOW.md` | 多智能体协作蓝图（架构全貌） |
| `PLANNER_RULES.md` | 规划者开工规则 |
| `EXECUTOR_RULES.md` | 执行者开工规则 |
| `LESSONS.md` | 统一经验库（横跨 UE/Unity/Blender 三项目，按工种分类索引） |
| `notes/` | 从经典书提炼的读书笔记（输入层） |

## 快速接入

1. 新项目 `git clone` 或拷贝这几份文件到你的 `<repo>/shared/`
2. 让 AI 先读 `AI_ONBOARDING.md`，再按角色读 `PLANNER_RULES.md` 或 `EXECUTOR_RULES.md`
3. 照 `WORKFLOW.md` §5 部署清单补齐项目特定部分

## 治理

- 这个仓库是 **主本 canonical**，仅由工坊主理人助手维护
- 各项目从主本拷贝后各自本地迭代
- 通用流程改进走「衍生本 → 提炼 → 回提主本」回路
