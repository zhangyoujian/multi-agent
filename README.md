# multi-agent

基于 **OpenClaw** 的 **Git 驱动异步协同写作** 模板仓库：将任务拆解、研究、撰稿、审校拆成角色化智能体，通过同一 Git 仓库中的约定目录与通知文件协作，产出可追溯、可审计的报告类文稿（含 Markdown 与最终 PDF 等）。

## 目录组织

| 路径 | 作用 |
|------|------|
| `agents/` | 存放各角色Agent的配置（如coordinator.md、researcher.md、writer.md、reviewer.md），定义Agent的职责、工具权限、工作空间   | 
| `comments/` | 存放 reviewer 审稿意见(如review_comments.md)                                                                       |
| `docs/`   | 操作与协议说明，引导openclaw如何创建协同智能体                                                                         |
| `tasks/`  | 任务拆解与分配文件（如task_breakdown.json记录协调Agent拆解的子任务，progress_log.md跟踪各角色进度）                     |
| `research_data/` | researcher研究产出（数据与来源）。                                                                             |
| `drafts/` | 存放文稿草稿（Markdown/PDF），按章节或版本号命名（如chapter1_v1.md、report_final.md、report_final.docx                |
| `memory/` | 记录协作状态（如MEMORY.md存储项目整体进度、代码仓和其他项目共享数据）                                                   |


## 协作流水线

协作开始前：用户先提供 `ACCESS_TOKEN` 与仓库名称 `repo_name`。随后 **main** 创建远程项目仓，并保证仓库内容与当前模板仓一致。

当 **main** 收到写作需求后：先把 **仓库地址与访问 TOKEN** 下发给 **coordinator**，再由 **coordinator** 转发给 **researcher / writer / reviewer**。之后：**coordinator** 拆解任务并推送 → **researcher** 补充研究材料 → **writer** 撰写/迭代 → **reviewer** 审校并反馈 → **writer** 修订 → **coordinator** 确认完成并发布。

各角色在各自 OpenClaw 工作空间中持有**独立克隆的同一项目仓**副本，通过 **git add、push、pull** 协作；提交信息使用 **`[角色名]`** 前缀便于审计。**coordinator 不使用定时器**定期扫仓，仅在收到其他智能体或 main 的通知后更新进度与推送。

> 本 GitHub/Git 上的 **multi-agent 仓库**是**脚手架模板**；实际交付写作发生在 **main 为每次任务创建并初始化的项目仓**中。

## 文档索引

| 文档 | 说明 |
|------|------|
| **[docs/INSTRUCTION.md](docs/INSTRUCTION.md)** | **OpenClaw 必读**：如何根据 `agents/` 创建四个协同智能体、工作目录、Git 步骤与协同协议。 |
| [docs/GIT_HOOKS.md](docs/GIT_HOOKS.md) | Git Hooks 启用方式。 |

## 开源与许可

见仓库根目录 `LICENSE`（若有）。
