# OpenClaw：根据 `agents/*` 创建协同智能体（INSTRUCTION）

本文档面向 **OpenClaw 编排层 / 自动化脚本**：请 **严格依据** 本仓库 `./agents/` 下四个 Markdown 文件，为每个角色生成磁盘上的 **SOUL.md、TOOLS.md、USER.md**，注册 Agent，预装技能。
目标：专业分工、协同清晰、产出路径明确。

---
## 1. 总原则

1. **单一真相源**：每个角色的完整参数以 `./agents/<role>.md` 为准（`coordinator.md`、`researcher.md`、`writer.md`、`reviewer.md`）。
2. **目录隔离**：Agent 私有文件（SOUL/TOOLS/USER、私有 `memory/`、私有 `skills/`）**不得**写入仓库克隆目录；
3. **提交前缀**：各 Agent 在新代码仓 `<repo_name>/` git commit信息 必须以 `[coordinator]` / `[researcher]` / `[writer]` / `[reviewer]` 开头（与对应文件一致）。
4. **写作任务门禁（强约束）**：在用户**尚未明确提出写作需求**前，`main` **严格禁止**向 `coordinator` 发送 `writing_task`（或等价写作任务消息）。
5. **远程项目仓由 coordinator 创建**：模板克隆自 [zhangyoujian/multi-agent](https://github.com/zhangyoujian/multi-agent.git)；`ACCESS_TOKEN` 由 main 在创建智能体后引导用户提供并先发给 coordinator，再由 coordinator 分发给其他智能体；各角色将 TOKEN 写入**本机工作区** `MEMORY.md`（**不**提交到项目仓）。
---

## 2. OpenClaw 注册 Agent 步骤

### 2.1 确定注册名和配置元

| OpenClaw 注册名 | 配置源文件 | 工作目录 |
|-----------------|----------------------|-------------------|
| coordinator     | `agents/coordinator.md` | `~/openclaw-workspaces/agents/coordinator/` |
| researcher      | `agents/researcher.md` | `~/openclaw-workspaces/agents/researcher/` |
| writer          | `agents/writer.md` | `~/openclaw-workspaces/agents/writer/` |
| reviewer        | `agents/reviewer.md` | `~/openclaw-workspaces/agents/reviewer/` |

备注: 这里的`~/openclaw-workspaces`指的是当前openclaw的实际工作目录

### 2.2 创建智能体

```bash
# 1. 创建智能体
openclaw agents add coordinator
openclaw agents add researcher
openclaw agents add writer
openclaw agents add reviewer

# 3. 为角色生成配置文件（参考 agents/**）对应角色的配置文件
#    - SOUL.md: 定义角色职责和行为准则
#    - TOOLS.md: 定义工具权限和可操作路径
#    - USER.md: 定义用户上下文和协作关系

```

### 2.3 预装技能

根据 `agents/<role>.md` 中的「Skills」列表，为每个 Agent 启用对应技能：

| 角色 | 核心技能  |
|------|----------|
| coordinator | sessions_send（任务通知与凭据下发） |
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
> 本项目的智能体协作仅使用 `sessions_send`（不使用子代理创建/执行任务）。

### 3.2 配置示例

在 `openclaw.json` 中配置多智能体：
** 重要 **: 务必配置会话超时时间，设置timeoutSeconds为600秒，以防止会话过期。

```json
{
  "agents": {
    "list": [
      {
        "id": "coordinator",
        "workspace": "~/openclaw-workspaces/agents/coordinator",
        "timeoutSeconds": 600
      },
      {
        "id": "researcher", 
        "workspace": "~/openclaw-workspaces/agents/researcher",
        "timeoutSeconds": 600
      },
      {
        "id": "writer",
        "workspace": "~/openclaw-workspaces/agents/writer",
        "timeoutSeconds": 600
      },
      {
        "id": "reviewer",
        "workspace": "~/openclaw-workspaces/agents/reviewer",
        "timeoutSeconds": 600
      }
    ],
    "default": "coordinator"
  },
  "tools": {
    "sessions": {
      "visibility": "all"
    },
    "agentToAgent": {
      "enabled": true,
      "allow": [
        "main",
        "coordinator",
        "researcher",
        "writer",
        "reviewer"
      ]
    }
  }
}
```

### 3.3 通信示例

**创建智能体后，main 引导用户提供 ACCESS_TOKEN 并下发给 coordinator：**

```python
# main 将用户提供的 ACCESS_TOKEN 发给 coordinator，供后续在托管平台创建远程仓与推送
sessions_send(
    sessionKey="agent:coordinator:main",
    message='{"kind":"access_token","access_token":"<ACCESS_TOKEN>","project_id":"<project_id_optional>"}'
)
```

**main 向 coordinator 下达写作任务（用户已明确提出写作需求；建仓由 coordinator 完成）：**

```python
# 前置条件：用户已明确提出写作需求
# coordinator 收到后：克隆 https://github.com/zhangyoujian/multi-agent.git → 按主题命名新远程仓 →
# 拷贝模板全部文件到新目录 → git init → 推送到远程 → 再生成 tasks/*、memory/MEMORY.md 并通知其他智能体
sessions_send(
    sessionKey="agent:coordinator:main",
    message='{"kind":"writing_task","project_id":"<project_id>","theme":"<topic>","requirements":"<requirements>","deadline":"<deadline>"}'
)
```

### 3.4 通信权限约束
- main 角色只能给coordinator发送消息，不能直接通知其他角色
- coordinator 角色可以向所有角色发送消息
- 其他角色只能接收来自 coordinator 的消息， 不能接受来自 main 角色的消息
- 其他角色只能发送给 coordinator，不能发送给 main 角色
---

## 4. Git 协作流程

### 4.1 标准操作流程

```bash
# 进入代码仓目录
cd ~/openclaw-workspaces/agents/<role>/<repo_name>

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
| researcher  | `research_data/**`      | `tasks/**`、`memory/**`                        |
| writer      | `drafts/**`             | `tasks/**`、`research_data/**`、`comments/**` 、`memory/**`|
| reviewer    | `comments/**`           | `tasks/**`、`drafts/**`、`memory/**`                       |

---

## 5. 智能体内部协作规范

### 5.1 任务流转

```
coordinator 拆解任务 → 通知 researcher → researcher 搜集资料 → 
writer 撰写初稿 → reviewer 审校 → writer 修订 → coordinator 确认发布
```

### 5.2 进度同步

- **coordinator** 维护 `tasks/progress_log.md`，记录各角色完成状态（**仅**在收到其他智能体的会话通知后拉取、更新并推送）。
- 所有角色在收到 **coordinator** 通知后，使用 `git pull` 更新**当前项目协作仓**获取最新进度。
- 每次状态更新后必须 `git commit` + `git push`（各角色各自负责的目录）。
- **ACCESS_TOKEN**：创建智能体后由 main 引导用户提供并发给 **coordinator**；coordinator 建仓并推送后，再将 **repository_url + access_token** 发给 researcher / writer / reviewer。各角色将 TOKEN 写入**本机工作区** `~/openclaw-workspaces/agents/<role>/MEMORY.md`（持久化），**禁止**将 TOKEN 写入项目仓内被 Git 跟踪的文件。

### 5.3 消息通知规范

| 场景 | 发送方 | 接收方 | 消息内容 |
|------|--------|--------|----------|
| 凭据收集 | 用户 → main | — | 创建智能体后，main 引导用户提供 `ACCESS_TOKEN` |
| 凭据下发 | main | coordinator | `access_token`（及可选 `project_id`） |
| 写作任务 | main | coordinator | 主题、要求、截止（**仅**在用户已提出写作需求后）；coordinator 随后克隆模板、创建远程仓、拷贝文件、`git init`、推送 |
| 凭据转发 | coordinator | researcher / writer / reviewer | `repository_url`、`access_token`、`default_branch` |
| 资料收集 | coordinator | researcher | 收集资料主题、收集资料要求、完成时间 |
| 资料收集完成 | researcher | coordinator | 收集资料完成，待开始初稿 |
| 撰写稿件 | coordinator | writer | 参考资料完成初稿 |
| 初稿完成 | writer | coordinator | 待审校稿件 |
| 审校稿件 | coordinator | reviewer | 稿件版本、待审校章节 |
| 审校完成 | reviewer | coordinator | 修改意见、问题列表 |
| 修改稿件 | coordinator | writer | 根据修改意见修改稿件 |
| 终稿完成 | writer | coordinator   | 待发布稿件 |
| 发布稿件 | coordinator | main | 稿件已完成 |

### 5.4 审计追溯

- 所有 Git 提交必须带 `[角色名]` 前缀
- Commit message 示例：
  - `[coordinator] 创建任务拆解 task_breakdown.json`
  - `[researcher] 添加2025年市场规模数据`
  - `[writer] 完成第一章初稿`
  - `[reviewer] 提出15条审校意见`

---

## 6. coordinator 调度方式（事件驱动）

- coordinator 在收到 **main** 下发的 **`access_token`** 时，将 TOKEN **持久写入**本机 `~/openclaw-workspaces/agents/coordinator/MEMORY.md`（不触发建仓）。
- coordinator **仅在**收到 **main** 的 **`writing_task`** 后执行**建仓、首推、任务拆解**；或在收到 **researcher / writer / reviewer** 的 `sessions_send` 完成通知后，对项目仓执行 pull → 核对交付物 → 更新 `tasks/progress_log.md` / 协作用 `memory/MEMORY.md` → commit → push → 按需通知下一角色。

---

## 7. 禁止行为（全局）

- 在 `<repo_name>/` 内放置 **SOUL.md、TOOLS.md、USER.md** 或各 Agent 私有 **memory/skills**
- 越权修改其他角色独占路径（见各 `agents/*.md`）
- 提交信息不带 **正确 `[角色名]`** 前缀
- 未经授权删除或覆盖他人已提交的文件
- 禁止各角色违背`3.4 通信权限约束` 和 `5.3 消息通知规范` 彼此发送消息
- **用户未提出写作需求时，禁止 `main` 给 `coordinator` 下达写作任务**
- **禁止将 `ACCESS_TOKEN` 写入项目协作仓内被跟踪的文件**（应写入各角色工作区私有 `MEMORY.md`）
---

## 8. 执行顺序

1. 阅读仓库 `agents/*.md` 了解各角色配置
2. 按第 2 节注册四个智能体；**researcher / writer / reviewer** 可待 coordinator 下发 **repository_url + access_token** 后再在各自工作空间执行 `git clone`（或使用凭据更新已有 `<repo_name>//` 远程）
3. 为每个 Agent 生成 `SOUL.md / TOOLS.md / USER.md`
4. 按第 3 节配置智能体间通信
5. 重启 Gateway：`openclaw gateway restart`
6. main 引导用户提供 `ACCESS_TOKEN`，并通过 `sessions_send` 发给 **coordinator**（可写入 coordinator 工作区私有 `MEMORY.md` 的提示由 coordinator 执行）
7. 用户**未提出写作需求前**，**禁止**向 coordinator 发送 `writing_task`
8. 用户提出写作需求后，main 向 coordinator 发送 `writing_task`
9. **coordinator** 克隆 `https://github.com/zhangyoujian/multi-agent.git`，按主题命名并**新建远程仓**，拷贝模板全部文件，`git init` 后推送到远程；再生成 `tasks/task_breakdown.json` 与 `tasks/progress_log.md`，并将 `repository_url + access_token` 转发给 researcher / writer / reviewer
10. researcher / writer / reviewer 将 TOKEN 写入各自工作区 **`MEMORY.md`**，再 `git clone` 项目仓协作

---

## 9. 验收清单

- [ ] 四个 Agent 工作目录均存在；**researcher / writer / reviewer** 在收到 coordinator 下发的 **repository_url** 后，各有独立 `<repo_slug>/` 克隆（该远程仓由 **coordinator** 创建并初始化）
- [ ] 每个目录下 **SOUL.md / TOOLS.md / USER.md** 与 `agents/<role>.md` 一致
- [ ] 四个 Agent 均已 `openclaw agents add` 注册
- [ ] 各角色预装 Skills 与 `agents/<role>.md` 列表一致
- [ ] `sessions_send` 工具可用
- [ ] `coordinator`与**researcher / writer / reviewer**等智能体之间通信正常
- [ ] 智能体会话超时时间是否配置成600


## 10. 注意事项

- 引导用户在创建智能体后提供代码托管平台的 **ACCESS_TOKEN**；由 **main** 发给 **coordinator**，再由 **coordinator** 分发给其他智能体；各角色将 TOKEN 写入工作区私有 **`MEMORY.md`**，**禁止**写入项目仓内被跟踪文件。
- 每个写作项目对应 **coordinator 新建并推送的独立远程仓**：克隆 [模板仓库](https://github.com/zhangyoujian/multi-agent.git) → 按主题命名远程仓 → 拷贝全部模板文件 → `git init` → 推送。
- 本仓库为**模板**；运行时协作以 **coordinator 创建的项目仓**为准。


