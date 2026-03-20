# OpenClaw：根据 `agents/` 创建协同智能体（INSTRUCTION）

本文档面向 **OpenClaw 编排层 / 自动化脚本**：请 **严格依据** 本仓库 `./agents/` 下四个 Markdown 文件，为每个角色生成磁盘上的 **SOUL.md、TOOLS.md、USER.md**，注册 Agent，预装技能。
目标：专业分工、协同清晰、产出路径明确。

---
## 1. 总原则

1. **单一真相源**：每个角色的完整参数以 `./agents/<role>.md` 为准（`coordinator.md`、`researcher.md`、`writer.md`、`reviewer.md`）。
2. **目录隔离**：Agent 私有文件（SOUL/TOOLS/USER、私有 `memory/`、私有 `skills/`）**不得**写入仓库克隆目录；
3. **提交前缀**：各 Agent 在 `multi-agent/` git commit信息 必须以 `[coordinator]` / `[researcher]` / `[writer]` / `[reviewer]` 开头（与对应文件一致）。
---

## 2. OpenClaw 注册 Agent 步骤

### 2.1 确定注册名和配置元

| OpenClaw 注册名 | 配置源文件 | 工作目录 |
|-----------------|----------------------|-------------------|
| coordinator     | `agents/coordinator.md` | `~/openclaw-workspaces/coordinator/` |
| researcher      | `agents/researcher.md` | `~/openclaw-workspaces/researcher/` |
| writer          | `agents/writer.md` | `~/openclaw-workspaces/writer/` |
| reviewer        | `agents/reviewer.md` | `~/openclaw-workspaces/reviewer/` |

### 2.2 创建智能体（以 coordinator 为例）

```bash
# 1. 创建智能体
openclaw agents add coordinator

# 2. 克隆代码仓到工作空间
cd ~/openclaw-workspaces/coordinator/
git clone https://github.com/zhangyoujian/multi-agent.git

# 3. 为角色生成配置文件（参考 agents/coordinator.md）
#    - SOUL.md: 定义角色职责和行为准则
#    - TOOLS.md: 定义工具权限和可操作路径
#    - USER.md: 定义用户上下文和协作关系
```

### 2.3 预装技能

根据 `agents/<role>.md` 中的「Skills」列表，为每个 Agent 启用对应技能：

| 角色 | 核心技能 |
|------|----------|
| coordinator | sessions_send（任务通知）、cron（定时检查） |
| researcher | web_search、baidu-search（资料搜集） |
| writer | docx、markdown-converter（文档撰写） |
| reviewer | 无特殊技能（审校为主） |

---

## 3. 智能体间通信配置

### 3.1 通信工具

OpenClaw 提供以下工具实现智能体间通信：

| 工具 | 用途 | 场景 |
|------|------|------|
| `sessions_send` | 向指定会话发送消息 | 实时通知、任务分发 |
| `sessions_spawn` | 创建新的子代理会话 | 异步任务执行 |
| `subagents` | 管理子代理 | 查看/终止/引导子任务 |

### 3.2 配置示例

在 `openclaw.json` 中配置多智能体：

```json
{
  "agents": {
    "list": [
      {
        "id": "coordinator",
        "workspace": "~/openclaw-workspaces/coordinator"
      },
      {
        "id": "researcher", 
        "workspace": "~/openclaw-workspaces/researcher"
      },
      {
        "id": "writer",
        "workspace": "~/openclaw-workspaces/writer"
      },
      {
        "id": "reviewer",
        "workspace": "~/openclaw-workspaces/reviewer"
      }
    ],
    "default": "coordinator"
  },
  "bindings": [
    {
      "agentId": "coordinator",
      "match": { "channel": "webchat" }
    }
  ]
}
```

### 3.3 通信示例

**coordinator 通知 researcher 执行任务：**

```python
# 使用 sessions_send
sessions_send(
    sessionKey="agent:researcher:main",
    message="请根据 tasks/task_breakdown.json 搜集相关资料，输出到 research_data/ 目录"
)
```

**coordinator spawn 子代理执行任务：**

```python
# 使用 sessions_spawn
sessions_spawn(
    task="分析最新行业报告，提取关键数据到 research_data/summary.md",
    runtime="subagent",
    agentId="researcher",
    mode="run"  # 一次性任务
)
```

**查看活跃的子代理：**

```python
subagents(action="list")
```

---

## 4. Git 协作流程

### 4.1 标准操作流程

```bash
# 进入代码仓目录
cd ~/openclaw-workspaces/<role>/multi-agent

# 拉取最新
git pull --rebase

# 查看状态
git status

# 添加允许的文件
git add <允许的路径>

# 提交（必须带角色前缀）
git commit -m "[coordinator] 更新任务进度"

# 推送
git pull --rebase
git push
```

### 4.2 各角色可操作路径

| 角色 | 可写 | 只读 |
|------|------|------|
| coordinator | `tasks/**`、`memory/**` | `drafts/**`、`research_data/**`、`comments/**` |
| researcher | `research_data/**` | `drafts/**`、`comments/**` |
| writer | `drafts/**` | `research_data/**`、`comments/**` |
| reviewer | `comments/**` | `drafts/**`、`research_data/**` |

---

## 5. 智能体内部协作规范

### 5.1 任务流转

```
coordinator 拆解任务 → 通知 researcher → researcher 搜集资料 → 
writer 撰写初稿 → reviewer 审校 → writer 修订 → coordinator 确认发布
```

### 5.2 进度同步

- **coordinator** 维护 `tasks/progress_log.md`，记录各角色完成状态
- 所有角色通过 `git pull` 获取最新进度
- 每次状态更新后必须 `git commit` + `git push`

### 5.3 消息通知规范

| 场景 | 发送方 | 接收方 | 消息内容 |
|------|--------|--------|----------|
| 任务分配 | coordinator | researcher | 任务ID、要求、截止时间 |
| 资料完成 | researcher | coordinator | 产出文件路径、数据摘要 |
| 稿件完成 | writer | coordinator | 稿件版本、待审校章节 |
| 审校完成 | reviewer | writer | 修改意见、问题列表 |

### 5.4 审计追溯

- 所有 Git 提交必须带 `[角色名]` 前缀
- Commit message 示例：
  - `[coordinator] 创建任务拆解 task_breakdown.json`
  - `[researcher] 添加2025年市场规模数据`
  - `[writer] 完成第一章初稿`
  - `[reviewer] 提出15条审校意见`

---

## 6. 定时任务配置（可选）

coordinator 可配置定时检查任务：

```bash
# 创建定时任务：每5分钟检查一次代码仓更新
openclaw cron add \
  --name "check-updates" \
  --schedule "*/5 * * * *" \
  --session-target isolated \
  --payload '{"kind":"agentTurn","message":"检查代码仓更新并同步进度"}'
```

---

## 7. 禁止行为（全局）

- 在 `multi-agent/` 内放置 **SOUL.md、TOOLS.md、USER.md** 或各 Agent 私有 **memory/skills**
- 越权修改其他角色独占路径（见各 `agents/*.md`）
- 提交信息不带 **正确 `[角色名]`** 前缀
- 未经授权删除或覆盖他人已提交的文件

---

## 8. 执行顺序

1. 阅读仓库 `agents/*.md` 了解各角色配置
2. 按第 2 节创建四个智能体并克隆代码仓
3. 为每个 Agent 生成 `SOUL.md / TOOLS.md / USER.md`
4. 按第 3 节配置智能体间通信
5. 重启 Gateway：`openclaw gateway restart`

---

## 9. 验收清单

- [ ] 四个工作目录均存在，且各有独立 `multi-agent/` 克隆
- [ ] 每个目录下 **SOUL.md / TOOLS.md / USER.md** 与 `agents/<role>.md` 一致
- [ ] 四个 Agent 均已 `openclaw agents add` 注册
- [ ] 各角色预装 Skills 与 `agents/<role>.md` 列表一致
- [ ] `sessions_send` / `sessions_spawn` 工具可用
- [ ] 测试：coordinator 能成功向 researcher 发送消息


