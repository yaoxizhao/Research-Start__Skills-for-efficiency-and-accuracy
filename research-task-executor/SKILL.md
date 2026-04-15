---
name: research-task-executor
description: "仅当用户精确输入'科研启动-执行'这六个字时才触发此 skill。不要在其他任何情况下触发，包括'开始做'、'执行任务'、'帮我实现'等类似说法都不应触发。必须是精确的'科研启动-执行'才可以激活。"
---

# Research Task Executor — 论文方法实现 + 缝合创新

当此 skill 被激活时，你是科研实现引擎。核心任务分两步：

1. **实现论文方法**：把 INNOVATION_IDEAS.md 中已评估通过的论文方法，忠实实现为独立的 pipeline mode
2. **缝合创新**：在实现基础上，评估能否与已有 baseline 缝合，产出新的创新点

## 用户 Baseline 架构（启动时动态读取，不要硬编码）

启动时读取 `config.py` 获取当前配置，组装成架构概览：

```bash
python3 -c "import sys; sys.path.insert(0,'/data/zhaoyaoxi/rag_project'); import config as c; print(f'Dataset: {c.DATASET_NAME}, Embed: {c.EMBED_MODEL}, LLM: {c.LLM_MODEL}, Judge: {c.GLM_MODEL}, TopK: {c.TOP_K}, ChunkSize: {c.CHUNK_SIZE}, Workers: {c.RAGAS_MAX_WORKERS}')"
```

组装格式：
```
Query → {EMBED_MODEL} → ChromaDB 检索 top-{TOP_K} → 拼接 Context → {LLM_MODEL} 生成
数据集：{DATASET_NAME}
切块：chunk_size={CHUNK_SIZE}, chunk_overlap={CHUNK_OVERLAP}
评测：Ragas（{GLM_MODEL} 做 judge，{RAGAS_MAX_WORKERS} 并发）
硬件：2x RTX 3090（48GB 总显存，vLLM TP=2）
已注册 modes：[从 rag/pipeline.py available_modes() 读取]
```

## 启动流程

### 1. 读取上下文
- `docs/PROGRESS.md` — 上次进展
- `docs/PIPELINE.md` — 当前 pipeline
- `docs/EXPERIMENTS.csv` — 实验历史
- `docs/ISSUES.md` — 已知问题（如果存在）
- **`docs/INNOVATION_IDEAS.md`** — 从论文提取的融合方案（核心输入）
- **`docs/EXTENSION_GUIDE.md`** — 扩展规范（必须遵守的"只加文件，不改旧代码"原则）

### 2. 一致性检查（一次性）
```bash
# 检查 conda 环境
echo "Conda: $CONDA_DEFAULT_ENV"
# 检查索引是否存在
ls -la /data/zhaoyaoxi/rag_project/chroma_db/ 2>/dev/null | head-5
```
发现问题立即用 `⚠️` 警告用户。

### 3. 确认任务

**如果用户没有指定具体任务**，展示 INNOVATION_IDEAS.md 中评估通过但尚未实现的方案：

```markdown
## 待实现的论文方案

1. [方案A名称]（来自 CRAG 论文）— 融合位置：检索后 → 预计难度：低
2. [方案B名称]（来自 xxx 论文）— 融合位置：生成时 → 预计难度：中

请选择要实现的方案（输入序号），或描述你的具体需求。
```

**如果用户指定了任务**，直接进入可行性评估。

## 实现规划（必须执行的前置步骤）

选定方案后，**先规划再动手**。注意：论文方法的融合可行性已由「读论文」skill 评估过并记录在 INNOVATION_IDEAS.md 中，这里只做纯实现层面的规划。

| 规划维度 | 检查内容 |
|---|---|
| 论文方法核心逻辑 | 这个方法的核心步骤是什么？需要哪些模块？ |
| 需要新建哪些文件 | 列出具体文件（如 `rag/crag/evaluator.py`、`rag/crag/refiner.py`） |
| 是否遵守 EXTENSION_GUIDE | 只加文件不改旧代码？新模块独立？新 mode 已注册？ |
| 需要额外安装什么 | 是否需要新的 pip 包或模型权重 |

**规划结果呈现给用户，确认后再开始实现。**

---

## Phase 1：实现论文方法为独立 pipeline mode

**目标**：忠实还原论文的核心方法，注册为一个独立 mode，先跑出论文方法自身的指标。

### 研究阶段

1. **阅读论文原文**（如果 INNOVATION_IDEAS.md 中有 markdown 路径）：
   ```bash
   ls /data/zhaoyaoxi/paper/markdown/ | grep -i "<论文名关键词>"
   ```
   找到对应的 markdown 文件，通读论文的方法部分（通常是 Section 3 或 Methodology）

2. **参考源码**（如果 INNOVATION_IDEAS.md 中记录了 GitHub 地址）：
   ```bash
   git clone --depth 1 <GitHub_URL> /tmp/ref_<repo_name>
   ```
   用 agent 读取核心文件，提取：
   - 核心数据结构和算法流程
   - 关键函数签名
   - 超参数默认值
   - 可以直接复用的工具函数

3. **阅读现有代码架构**：用 agent 读取 `rag/pipeline.py`、`rag/retriever.py`、`rag/augmenter.py`、`rag/generator.py`，理解扩展点

### 实现阶段

**分工明确：主对话写代码，agent 只做只读分析。**

以下 5 个角色是思考角度的划分，不是 5 个独立 agent：
- **角度 1 (RAG 架构)**：把控整体架构，确保新模块能插入现有 pipeline
- **角度 2 (检索工程)**：ChromaDB 读写、Chunking 策略、检索优化
- **角度 3 (生成与 Prompt)**：vLLM 接口交互、Prompt 模板、幻觉约束
- **角度 4 (评测数据流)**：确保输出 CSV 格式满足 evaluate.py 要求（contexts, answer, ground_truth 字段齐全）
- **角度 5 (工程规范)**：代码质量、异常处理、Git 管理

### 文件组织规则（重要）

每个论文方法的**实现代码**放在 `rag/` 下的独立子文件夹中，但 **pipeline mode 的注册统一在 `rag/pipeline.py`** 中完成。

**目录结构示例**：
```
rag/
├── __init__.py
├── pipeline.py        ← 所有 mode 的注册中心（基类 + 注册表，只追加注册）
├── retriever.py       ← baseline 检索器（已有，不改动）
├── augmenter.py       ← baseline prompt 构建器（已有，不改动）
├── generator.py       ← baseline 生成器（已有，不改动）
│
├── crag/              ← CRAG 论文方法的实现模块
│   ├── __init__.py
│   ├── evaluator.py   ← 检索结果评估器
│   └── refiner.py     ← 知识精炼模块
│
├── self_rag/          ← Self-RAG 论文方法的实现模块（未来）
│   ├── __init__.py
│   ├── ...
│
└── crag_self_rag/     ← 缝合方案的实现模块
    ├── __init__.py
    └── ...
```

**注册方式（全在 `rag/pipeline.py` 底部追加）**：
```python
# Phase 1：论文方法 — 直接在工厂函数中编排流程
@register_pipeline("crag")
def _create_crag(collection_name=None, **kwargs):
    from rag.crag.evaluator import CRAGEvaluator
    from rag.crag.refiner import CRAGRefiner
    from rag.retriever import Retriever
    from rag.augmenter import Augmenter
    from rag.generator import Generator

    # 在工厂函数或 Pipeline 子类中编排流程
    ...

# Phase 2：缝合方案 — 组合已有子文件夹的模块
@register_pipeline("crag_self_rag")
def _create_crag_self_rag(collection_name=None, **kwargs):
    from rag.crag.evaluator import CRAGEvaluator
    from rag.self_rag.xxx import SelfRAGModule
    ...
```

**规则**：
- 子文件夹只放**功能模块**（evaluator、refiner、augmenter 等），不放 pipeline 编排逻辑
- 所有 mode 的编排和注册**统一在 `rag/pipeline.py` 底部追加**
- Phase 1 的论文方法 → `rag/<方法名>/` 子文件夹
- Phase 2 的缝合方案 → `rag/<方法A>_<方法B>/` 子文件夹（额外模块），注册仍在 `rag/pipeline.py`
- 每个子文件夹必须有 `__init__.py`
- **禁止在 `rag/` 根目录下创建新的 `.py` 文件**

### 实现原则

1. **遵守 EXTENSION_GUIDE "只加文件，不改旧代码"**：
   - 在 `rag/<方法名>/` 子文件夹中新建所有模块文件
   - 在 `rag/pipeline.py` 底部用 `@register_pipeline` 注册新 mode（只添加，不修改已有注册）
   - **禁止修改**：`run_baseline.py`、`rag/retriever.py`、`rag/augmenter.py`、`rag/generator.py`、已有的 pipeline 注册
2. **忠实还原论文**：先不要做任何"改进"或"缝合"，严格按论文描述实现核心方法
3. **最小完整方案**：只实现当前方案需要的东西，不重构已有代码
4. **RAG 链路自检**：输出前按 Query→检索→处理→拼装→生成→输出 走一遍
5. **标注假设**：对不确定的部分明确标注"假设"和"未验证"

### Phase 1 完成标准

- 论文方法已注册为独立 pipeline mode（如 `crag`）
- 代码能通过 `python run_baseline.py --mode <新模式名>` 运行
- 输出 CSV 格式正确（contexts, answer, ground_truth 字段齐全）

---

## Phase 2：缝合创新

**前提**：Phase 1 已完成，论文方法已有独立 pipeline mode。

**目标**：评估论文方法能否与已有 baseline（naive_rag 等）或已实现的其他创新点缝合，产生新的创新点。

### 缝合评估

对比以下维度，判断缝合价值：

| 评估维度 | 说明 |
|---|---|
| 方法互补性 | 两个方法作用于不同环节（如一个改检索、一个改生成）→ 互补性强 |
| 叠加增益 | 组合后预期能否比单独使用任一方法更好？ |
| 学术故事线 | 能否讲出一个完整的"动机→方法→实验"逻辑链？ |
| 实现复杂度 | 缝合是否只需编排两个已有模块，还是需要额外的适配逻辑 |
| 资源开销 | 缝合后显存/推理时间是否仍可接受 |

### 缝合实现

如果评估通过：

1. **新建组合子文件夹**：创建 `rag/<method_a>_<method_b>/`（如 `rag/crag_self_rag/`），内含 `__init__.py` 和 `pipeline.py`
2. **编排调用顺序**：在缝合文件夹的 `pipeline.py` 中，import Phase 1 和已有 baseline 的模块，按合理的顺序串联
3. **注册新 mode**：如 `crag_self_rag`，在 `rag/pipeline.py` 底部注册，import 路径为 `from rag.crag_self_rag.pipeline import CRAGSelfRAGPipeline`
4. **两个原模块不动**：缝合只是组合调用，不修改任何已有实现

### 缝合方案输出

向用户展示缝合方案：
```markdown
## 缝合方案：[吸引人的名称]

**组合**：[方法A] + [方法B]
**新 mode 名**：[如 crag_rag]
**学术故事**：[2-3 句话说明为什么这个组合有意义]
**实现方式**：[说明如何编排两个模块]
**预期效果**：[对比单独使用各方法的预期提升]
**实现成本**：[低/中/高]
```

---

## 任务结束

代码实现完成后：

1. **Git 管理**：git add 相关文件 → commit（不加 Claude 署名）→ 确认后 push
2. **更新文档**：
   - `docs/PROGRESS.md` — 追加本次工作记录
   - `docs/PIPELINE.md` — 如果流程变了，更新执行步骤
   - `docs/INNOVATION_IDEAS.md` — 更新方案状态为"已实现，待评测"
3. **告知用户如何跑实验**：输出具体的运行命令，用户自己执行：

```markdown
## 代码已完成，请手动执行以下步骤：

### 1. 启动 vLLM（如果还没启动）
bash serve_model.sh

### 2. 先跑论文方法独立实验
python run_baseline.py --mode <论文方法mode>

### 3. 如果有缝合方案，跑缝合实验
python run_baseline.py --mode <缝合mode>

### 4. 逐个评测
python evaluate.py --input results/<最新CSV>

### 5. 评测完后回来告诉我结果
把评测输出贴给我，我帮你记录和分析。
```

4. **总结**：给用户简要的完成总结 + 运行命令

## 实验结果记录

当用户贴回评测结果后：

1. **解析结果**：从用户贴的 ragas JSON 或终端输出中提取 faithfulness、answer_correctness、context_recall、context_precision、abstention_rate
2. **追加到 `docs/EXPERIMENTS.csv`**：新增一行，格式遵循 CSV 表头（exp_id 递增）
3. **更新 `docs/INNOVATION_IDEAS.md`**：方案状态从"已实现，待评测"改为"已评测"，附上关键指标
4. **更新 `docs/PROGRESS.md`**：追加本次实验记录和指标摘要
5. **给用户简要分析**：
   - 论文方法 vs naive_rag baseline 的指标对比
   - 缝合方案 vs 单独方法的指标对比
   - bad case 提示
