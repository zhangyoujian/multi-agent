# 任务拆解与分配、任务进度等相关文件存放路径

## （1）任务拆解文件（task_breakdown.json）

> 由`coordinator`在收到openclaw递交任务之后，将任务拆解成多个子任务，生成task_breakdown.json然后提交，成员通过 `git pull` 同步查看各自任务详情

### 模板条目
参考`tasks/task_breakdown_template.json文件`

### 记录文件
在tasks/task_breakdown.json中

## （2）各个角色进度日志（progress_log.md）

> `coordinator` 更新提交；成员通过 `git pull` 同步。

### 模板条目 (仅做参考)

| 日期时间 | 协同者 | 工作进度 |
|--------|---------|---------|
|YYYY-MM-DD HH:mm  | coordinator | 项目初始化，task_breakdown.json v1 已提交。|
|YYYY-MM-DD HH:mm  | researcher  | T1/T2 数据已写入 research_data/，写作可拉取。|
|YYYY-MM-DD HH:mm  | writer      |drafts/report_v1.md 已提交，请 reviewer 进行审阅。|
|YYYY-MM-DD HH:mm  | reviewer    |review_comments.md 已更新，阻塞项：…… |

###  记录文件
在tasks/progress_log.md中按照上述模板条目记录任务进度。

**重要**: `tasks/progress_log.md`只能追加,不能对已有条目修改



