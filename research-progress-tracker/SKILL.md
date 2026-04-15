---
name: research-progress-tracker
description: "仅当用户精确输入'科研启动'这四个字时才触发此 skill。不要在其他任何情况下触发，包括'开始写代码'、'继续科研'、'开始工作'、'写代码'等类似说法都不应触发。必须是精确的'科研启动'四个字才可以激活。"
---

# Research Progress Tracker — 状态同步 + 方向决策

当此 skill 被激活时，你是科研项目的"仪表盘"，快速汇报当前状态并推荐下一步方向。

## 启动流程（一次性完成，不要启动子 agent，直接在主对话中用工具完成）

按顺序执行以下步骤，全部完成后统一汇报：

### 1. 环境检查
```bash
echo "Conda: $CONDA_DEFAULT_ENV" && echo "Python: $(python3 --version)"
```
如果环境不是 `rag_project`，提醒用户 `conda activate rag_project`。

### 2. 读取项目状态
依次读取以下文件（不存在则跳过）：
- `docs/PROGRESS.md` — 上次做到哪里了
- `docs/PIPELINE.md` — 当前实验流程
- `docs/EXPERIMENTS.csv` — 实验历史和指标
- `docs/ISSUES.md` — 已知问题（如果存在）
- **`docs/INNOVATION_IDEAS.md`** — 从论文中提取的融合方案（pdf-reader 生成的）

### 3. 代码库状态
```bash
git status && echo "---" && git log --oneline -5
```

### 4. 汇报格式

按以下结构向用户汇报：

```markdown
## 当前状态

**环境**：rag_project / Python 3.12
**Git**：[分支名]，[有无未提交改动]
**已注册 pipeline modes**：[从 rag/pipeline.py 读取注册表，列出所有 mode]
**上次进度**：[PROGRESS.md 最近一条的摘要]

## 实验指标

| Exp | 模式 | faithfulness | answer_correctness | 结论 |
|-----|------|-------------|-------------------|------|
| 最近3条实验... ||||

## 创新点待办

[读取 INNOVATION_IDEAS.md，列出所有未实现的融合方案]
- 方案A：xxx（来自 CRAG 论文）— 未实现
- 方案B：xxx（来自 xxx 论文）— 未实现
- 缝合方案C：xxx + xxx — 未实现

## 建议下一步

基于当前状态，建议：
1. [具体建议，如"方案A可行性最高，建议用 `科研启动-执行` 去实现"]
2. [如果有新论文待分析，建议用 `读PDF`]
```

## 方向推荐逻辑

汇报完成后，根据以下优先级推荐下一步：

1. **如果 INNOVATION_IDEAS.md 有未实现方案** → 推荐 `科研启动-执行`，并指出哪个方案最适合先做
2. **如果有新论文在 /data/zhaoyaoxi/paper/ 下还没分析** → 推荐 `读PDF` 先分析
3. **如果上次实验结果不理想** → 推荐分析 bad case 或 `科研启动-检查` 审查代码
4. **如果代码有未提交改动** → 提醒用户先 commit

## 文档更新（仅在本次会话中做了实际工作时执行）

如果用户在本次"科研启动"后实际执行了工作（写代码/跑实验），在任务结束时更新：
- `docs/PROGRESS.md` — 追加本次工作记录
- `docs/EXPERIMENTS.csv` — 如果跑了实验
- `docs/PIPELINE.md` — 如果改了流程

Git commit 由 `科研启动-执行` skill 负责，本 skill 不做 git 操作。
