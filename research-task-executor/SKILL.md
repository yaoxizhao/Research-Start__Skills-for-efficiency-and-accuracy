---
name: research-task-executor
description: "仅当用户精确输入'科研启动-执行'这六个字时才触发此 skill。不要在其他任何情况下触发，包括'开始做'、'执行任务'、'帮我实现'等类似说法都不应触发。必须是精确的'科研启动-执行'才可以激活。"
---

# Research Task Executor — 从融合方案到实验结果的完整执行

当此 skill 被激活时，你是科研实现引擎。核心任务：**读取 INNOVATION_IDEAS.md 中的融合方案，评估可行性，实现代码，跑实验，给出指标对比。**

## 用户 Baseline 架构（必须了解）

```
Query → BAAI/bge-m3 Embedding → ChromaDB 检索 top-k → 拼接 Context → Qwen2.5-7B 生成回答
数据集：BeIR/fiqa（金融QA，57,600 docs）
评测：Ragas（DeepSeek API 做 judge）
硬件：2x RTX 3090（48GB）
```

## 启动流程

### 1. 读取上下文
- `docs/PROGRESS.md` — 上次进展
- `docs/PIPELINE.md` — 当前 pipeline
- `docs/EXPERIMENTS.csv` — 实验历史
- `docs/ISSUES.md` — 已知问题（如果存在）
- **`docs/INNOVATION_IDEAS.md`** — 从论文提取的融合方案（核心输入）

### 2. 一致性检查（一次性）
```bash
# 检查 vLLM 是否在运行
curl -s http://localhost:8000/v1/models | python3 -m json.tool 2>/dev/null | head -3 || echo "vLLM 未启动"
# 检查 conda 环境
echo "Conda: $CONDA_DEFAULT_ENV"
# 检查索引是否存在
ls -la /data/zhaoyaoxi/rag_project/chroma_db/ 2>/dev/null | head -5
```
发现问题立即用 `⚠️` 警告用户。

### 3. 确认任务

**如果用户没有指定具体任务**，展示 INNOVATION_IDEAS.md 中的待实现方案：

```markdown
## 待实现的融合方案

1. [方案A名称]（来自 CRAG 论文）— 融合位置：检索后 → 预计难度：低
2. [方案B名称]（来自 xxx 论文）— 融合位置：生成时 → 预计难度：中

请选择要实现的方案（输入序号），或描述你的具体需求。
```

**如果用户指定了任务**，直接进入可行性评估。

## 可行性评估（必须执行的前置步骤）

选定方案后，**先评估再动手**：

| 评估维度 | 检查内容 |
|---|---|
| 需要改哪些文件 | 列出具体文件和改动范围 |
| 需要额外安装什么 | 是否需要新的 pip 包或模型权重 |
| 显存够不够 | vLLM 占了多少，新模块需要多少 |
| 预计时间 | 实现多久 + 跑实验多久 |
| 有无现成参考 | GitHub 上有没有类似实现 |

**评估结果呈现给用户，确认后再开始实现。** 如果资源不够或时间太长，建议用户换方案。

## 工作模式

**使用 5 个 agent 并行实现：**

- **Agent 1 (RAG 架构师)**：把控整体架构，确保新模块（如 Query Rewrite、Reranker、文档过滤）能插入现有 pipeline。
- **Agent 2 (检索工程师)**：专注 ChromaDB 读写、Chunking 策略、检索优化。
- **Agent 3 (生成与 Prompt 工程师)**：vLLM 接口交互、Prompt 模板、幻觉约束。
- **Agent 4 (评测数据流专家)**：确保输出 CSV 格式满足 evaluate.py 要求（contexts, answer, ground_truth 字段齐全）。
- **Agent 5 (工程规范专家)**：代码质量、异常处理、Git 管理。

## 实现原则

1. **最小完整方案**：只实现当前方案需要的东西，不重构已有代码
2. **先研究再动手**：用 agent 读现有代码，理解架构后再写
3. **参考源码**：如果 INNOVATION_IDEAS.md 中记录了论文的 GitHub 地址，**必须先用 agent 读取该仓库的核心代码**，参考其实现方式和代码结构，再动手写。不要闭门造车。
4. **RAG 链路自检**：输出前按 Query→检索→拼装→生成→输出 走一遍
5. **标注假设**：对不确定的部分明确标注"假设"和"未验证"

### 源码参考流程

当 INNOVATION_IDEAS.md 中的方案有 GitHub 源码时：

1. **克隆仓库到临时目录**：
   ```bash
   git clone --depth 1 <GitHub_URL> /tmp/ref_<repo_name>
   ```
2. **用 agent 读取核心文件**：找到与融合方案相关的核心模块（通常是 model.py、retriever.py、pipeline.py 等），读取关键实现逻辑
3. **提取关键设计**：
   - 核心数据结构
   - 关键函数签名和调用方式
   - 超参数默认值
   - 可以直接复用的工具函数
4. **参考但不照搬**：理解其设计思路，适配到自己的 baseline 架构中，不要直接复制粘贴（避免 license 问题）

## 任务结束

代码实现完成后：

1. **Git 管理**：git add 相关文件 → commit（不加 Claude 署名）→ 确认后 push
2. **更新文档**：
   - `docs/PROGRESS.md` — 追加本次工作记录
   - `docs/PIPELINE.md` — 如果流程变了，更新执行步骤
   - `docs/INNOVATION_IDEAS.md` — 更新方案状态为"已实现，待评测"
3. **告知用户如何跑实验**：输出具体的运行命令，用户自己执行。例如：

```markdown
## 代码已完成，请手动执行以下步骤：

### 1. 启动 vLLM（如果还没启动）
bash serve_model.sh

### 2. 运行实验
python run_baseline.py --mode <新模式名>

### 3. 运行评测
python evaluate.py --input results/<最新CSV>

### 4. 评测完后回来告诉我结果
把评测输出贴给我，我帮你分析指标和 bad case。
```

4. **总结**：给用户简要的完成总结 + 运行命令
