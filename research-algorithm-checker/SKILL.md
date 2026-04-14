---
name: research-algorithm-checker
description: "仅当用户精确输入'科研启动-检查'这六个字时才触发此 skill。不要在其他任何情况下触发，包括'检查代码'、'检查算法'、'查bug'、'代码审查'等类似说法都不应触发。必须是精确的'科研启动-检查'才可以激活。"
---

# Research Algorithm Checker — RAG 代码审查 + 论文对齐

当此 skill 被激活时，你是一个严谨的 RAG 科研代码审查员。核心任务：**逐段读代码、逐段质疑、对比论文设计，找出 bug 和偏差**。

## 工作模式

**使用 5 个 agent 并行审查：**

- **Agent 1 (RAG 链路审查)**：追踪 Query → Embedding → ChromaDB → Context拼装 → vLLM生成 的完整数据流，确认衔接正确、无信息丢失。
- **Agent 2 (检索与边界异常)**：检查极端边界条件：检索返回空结果、索引越界、输入超长截断、API 速率限制等。
- **Agent 3 (Prompt 与防幻觉审查)**：审查 Prompt 模板：Context 是否隔离？指令是否严谨？是否防止了模型脱离上下文胡编？
- **Agent 4 (评测严谨性审查)**：核对 evaluate.py，确认 contexts/answer/ground_truth 没传错，Ragas 分数没有虚高或失真。
- **Agent 5 (论文对齐审查)**：**读取 `docs/INNOVATION_IDEAS.md`**，将代码实现与论文原始设计对比，找出偏差。检查代码是否真正验证了融合假设。

## 启动流程

1. **读取上下文**（不重复做环境检查，假设用户已通过 `科研启动` 确认环境）：
   - `docs/PROGRESS.md` — 了解上次进展
   - `docs/PIPELINE.md` — 当前 pipeline
   - `docs/ISSUES.md` — 已知问题（如果存在）
   - **`docs/INNOVATION_IDEAS.md`** — 从论文提取的融合方案
2. **代码库状态**：`git status && git log --oneline -5`
3. **向用户确认审查范围**：
   - "本次要检查哪些文件/函数？"
   - "是要找 Bug、查幻觉隐患，还是对照 INNOVATION_IDEAS 核对实现与论文设计的偏差？"
4. **启动 5 Agent 并行审查**

## 核心审查方法：逐段阅读、逐段质疑

1. **从入口开始，逐段阅读**：每次只读一个逻辑块
2. **每段读完后做三件事**：
   - **理解**：这段代码在干什么？
   - **判断**：干的这事对吗？有没有增加幻觉风险？
   - **结论**：对的 → 继续；有疑问 → 标记为潜在问题
3. **质疑要具体**：不是"可能有问题"，而是"这里把 Context 截断了，会导致核心前提丢失，诱发幻觉"
4. **对比论文设计**：每段代码对比 `docs/INNOVATION_IDEAS.md` 中的方案描述，确认实现没变形

**不跳步、不遗漏。**

## Issue 追踪

维护 `docs/ISSUES.md`，每发现一个问题，**逐条**向用户展示并询问分类：

> 1. [问题描述]
> 这个你要：**todo**（以后处理）/ **doing**（现在修）/ **discard**（舍弃）？

**规则**：
- discard 的问题以后不再重复提
- Issue ID 从 ISS-001 全局递增
- doing 完成后移到 Done，记录修复方式

## 审查结束

1. 更新 `docs/ISSUES.md`
2. 如果修了代码：git add 相关文件 → commit（不加 Claude 署名）→ 确认后 push
3. 更新 `docs/PROGRESS.md`
4. 给用户总结：发现了多少问题，多少 todo/doing/discard，整体代码质量评估
