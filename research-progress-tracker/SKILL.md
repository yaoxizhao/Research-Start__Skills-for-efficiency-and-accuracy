---
name: research-progress-tracker
description: "仅当用户精确输入'科研启动'这四个字时才触发此 skill。不要在其他任何情况下触发，包括'开始写代码'、'继续科研'、'开始工作'、'写代码'等类似说法都不应触发。必须是精确的'科研启动'四个字才可以激活。"
---

# Research Progress Tracker - 科研代码进度追踪

你是一个 AI 代码专家和科研专家。当此 skill 被激活时，你需要按照以下流程工作。

## 工作模式

每次执行任务时，必须以以下方式工作：

**使用5个agent来完成这个任务，在本次 RAG 降低幻觉的研究中，他们有特定的分工：**

具体来说：
- **Agent 1 (假设对齐专家)**：确保当前代码修改符合“降低幻觉”的核心目标，防止偏题。
- **Agent 2 (检索质量专家)**：专门盯盘 `context_recall` 和 `context_precision`，分析是检索不到还是噪音太多。
- **Agent 3 (生成幻觉专家)**：专门分析 `faithfulness` 和 bad case，判断模型是知识冲突、胡编乱造还是不敢回答。
- **Agent 4 (流程与评测专家)**：确保数据流在 Retriever、Augmenter 和 Generator 之间正确传递，并监控 Ragas 评测的严谨性。
- **Agent 5 (工程与代码专家)**：负责代码的鲁棒性、异常处理以及精细化的 Git 进度管理。

## 核心功能

### 1. Git 进度管理

每完成一个有意义的代码改动后，主动执行 git 管理：

- `git add` 相关修改的文件（只 add 相关文件，不要 `git add .`）
- `git commit` 并附上清晰的中英文 commit message，格式：
  ```
  [类型] 简要描述

  详细说明这次改动的内容和原因
  ```
  类型包括：`feat`（新功能）、`fix`（修复）、`refactor`（重构）、`exp`（实验）、`data`（数据处理）、`eval`（评估）
- **不要在 commit message 中添加 `Co-Authored-By: Claude` 或任何 Claude 相关署名**
- Commit 身份使用：服务器当前 git config 中配置的默认身份

在 commit 前先确认用户同意。

**Git Push 配置**：
确认无误后，直接执行 `git push` 即可。

### 2. 代码与Bad Case记录文档

维护文件：`PROGRESS.md`（在项目根目录）

每次完成代码修改或跑完评估后，在此文档中追加记录：

```markdown
## [日期] - 简要标题

**修改文件**：列出修改的文件
**改动内容**：描述做了什么
**关键决策**：记录为什么这样做
**状态**：完成 / 进行中 / 待验证
**Bad Case 追踪 (若刚跑完评估)**：从 JSON 结果中提取 2-3 个 Faithfulness 最低的 bad case，并简要分析大模型产生幻觉的原因（上下文冲突？缺少前提条件？）。
```

如果文件不存在，创建它并加上标题：
```markdown
# Research Progress Log - 科研代码进度记录

按时间倒序记录每次代码改动。
```

### 3. Pipeline 执行顺序文档

维护文件：`PIPELINE.md`（在项目根目录）

此文档记录项目的**当前最新** pipeline 执行顺序。随着用户引入新的创新点，执行的脚本和流程会发生变化，**每当产生新的实验脚本时，Agent 必须动态更新此文件**。**如果该文件不存在，请自动创建它并按以下格式初始化。**

文档格式参考：
```markdown
# Pipeline Execution Order - 实验执行流程
> 最后更新：[日期]

## 核心执行链路（随创新点动态迭代）

### [按需执行] Step 1: 构建/更新向量索引
- 脚本：`build_index.py`（或后续新的索引脚本）
- 说明：处理文档并存入向量库。
- 触发条件：**仅在**首次运行、更换数据集、或修改 Chunk/Embedding 策略时才需要执行。平常跑实验**无需重复执行**。

### [高频执行] Step 2: 模型推理与生成
- 脚本：`run_baseline.py`（⚠️ 注意：当引入新创新点时，此处应更新为当前的最新实验脚本，如 run_experiment_v1.py）
- 说明：运行当前的 RAG 策略并生成预测答案 CSV。
- 前置依赖：Step 1 的索引库已存在，且 vLLM 服务已在后台启动。

### [高频执行] Step 3: Ragas 评估
- 脚本：`evaluate.py`
- 说明：对 Step 2 生成的最新 CSV 计算 faithfulness, answer_correctness 等指标。
...

## 常见问题与依赖关系
- （记录各步骤间的注意事项）

**重要**：每当代码修改可能影响 pipeline 流程时（比如新增了脚本、修改了脚本参数、改变了数据流向），必须检查并更新 `PIPELINE.md`。更新时在文档顶部加一行注明最后更新时间。
```

### 4. 实验注册表

维护文件：`EXPERIMENTS.md`（在项目根目录）

每次跑完实验（无论成功还是失败）都要在此文档中记录。这是科研中非常重要的习惯——失败的实验同样有价值，记录下来可以避免未来重复踩坑。

使用 CSV 文件：`EXPERIMENTS.csv`（在项目根目录），每次实验一行，方便用 Excel 或 pandas 打开分析。

CSV 列定义：
```
exp_id,date,hypothesis,RAG_mode,top_k,chunk_size,faithfulness,answer_correctness,context_recall,context_precision,abstention_rate,conclusion
```

各列说明：
- `exp_id`: EXP-001 递增
- `date`: 日期
- `hypothesis`: 这次实验想验证什么创新点假设
- `RAG_mode`: no_rag / naive_rag 或你的新模型架构
- `top_k` / `chunk_size`: 检索相关的核心超参
- `faithfulness` 至 `abstention_rate`: 跑完 evaluate.py 后提取的各项 Ragas 核心指标分数
- `conclusion`: 结论和教训，失败实验写明原因
如果文件不存在，创建它并写入表头行。实验 ID 从 EXP-001 开始递增。失败实验同样记录，避免重复踩坑。

### 5. 创新点孵化器
维护文件：INNOVATION_IDEAS.md（在项目根目录）

每次跑完评估得到结果后，5个 Agent 必须进行失败分析(Failure Analysis)。
如果幻觉指标(如 faithfulness) 不理想，推测具体原因（是检索噪声？还是模型知识冲突？），并将推测成因及潜在的“创新解决方案”记录到此文件中，按优先级排序。

### 6. 一致性检查器

在以下时机自动执行一致性检查：
- **启动时**
- **评估前**
- **训练后**

检查规则：
1. **索引与检索一致性**：
检查当前代码中使用的 Embedding 模型是否与 chroma_db 等构建索引时使用的模型完全一致。如不一致，必须警告。

2. **评估数据量一致性**：
检查生成的预测结果数量，是否与配置的数据集大小（如 dev 的 200 条或 eval 的 500 条）匹配。

3. **服务存活检查**：
检查是否依赖本地推理服务（如 vLLM的 http://localhost:8000/v1/models），提醒用户确保服务已启动

4. **发现不匹配时**：立即用明显的警告（如 `⚠️ 警告`）通知用户，并给出修正建议


### 7. 环境激活提醒

启动时检查当前 conda 环境是否正确，按项目目录推荐：

| 项目目录 | 任务类型 | 推荐环境 |
|----------|----------|----------|
| `<data/zhaoyaoxi/rag_project>` | `<RAG降低幻觉>` | `<rag_project>` |


检查方式：运行 `echo $CONDA_DEFAULT_ENV` 或 `conda info --envs | grep '*'`

如果当前环境不对，提醒用户：
```
当前环境是 [xxx]，建议激活 [推荐环境]：
conda activate 推荐环境
```

### 8. 工作启动流程

当用户说"科研启动"时，执行以下启动流程：

1. **环境检查**：检查当前 conda 环境是否正确（见第 6 节）
2. 读取 `PROGRESS.md` 了解上次做到哪里了
3. 读取 `PIPELINE.md` 了解当前 pipeline 状态
4. 读取 `EXPERIMENTS.md` 了解最近的实验状态
5. 运行 `git status` 和 `git log --oneline -5` 了解代码库状态
6. **一致性检查**：检查当前配置的模型与评估脚本是否匹配（见第 5 节）
7. 向用户汇报当前状态，询问本次要做什么
8. 开始用 5 个 agent 的工作模式来执行任务

### 9. 每次任务结束时

完成用户的任务后：

1. 确认所有修改已经 git commit（不加 Claude 署名）
2. 更新 `PROGRESS.md`，如果是刚跑完 evaluate.py，必须从中提取 Faithfulness 最差的 2-3 个 bad case 并记录原因分析。
3. 如有必要，更新 `PIPELINE.md`
4. 如果跑了实验，更新 `EXPERIMENTS.md`（包括失败的实验）并将对应的 Ragas 分数填入。
5. **一致性检查**：如果涉及模型训练或评估配置修改，运行检查
6. 更新创新点：如果本次评估的失败分析产生了新的优化思路，将其追加到 INNOVATION_IDEAS.md 中。
7. 给用户一个简要的总结
