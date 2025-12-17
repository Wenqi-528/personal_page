---
title: "Fin-SQL-Optimizer: 金融 Text-to-SQL 的 CoT 蒸馏实践"
date: 2025-12-17
description: "深度对比补全法与推导法，构建 0 噪声的金融 SQL 推理数据集。"
# 这里的图片路径相对于 static 文件夹，建议放一张技术架构图
# featured_image: "/images/sql_project_cover.jpg" 
# 首页摘要手动控制（防止代码块溢出到首页）
summary: "通过端到端思维链（CoT）构造方案，配合工业级 SQL 执行沙箱校验，实现了从逻辑推导到结果验证的完整闭环，显著提升了模型在复杂金融场景下的 SQL 生成准确率。"
---

## 💡 项目核心动机

在处理金融领域的 **Text-to-SQL** 任务时，模型常面临“逻辑断层”：
1. **简单生成**：模型直接给出 SQL，但在处理复杂的跨表票据计算时，准确率不足。
2. **评估困境**：仅靠字符串比对无法判定 SQL 的正确性，必须回归到**数据执行层面**。

本项目通过 **端到端 CoT 构造方案** 与 **工业级 SQL 执行沙箱**，实现了一套从原始问答对到高性能蒸馏数据的自动化闭环。

---

## 🛠️ 技术深度解析

### 1. CoT 构造方案对比与选型
我深入研究了两套思维链构造方案，并基于**蒸馏需求**进行了选型：

| 维度 | 方案一：Post-hoc CoT | 方案二：End-to-End CoT (本项目采用) |
| :--- | :--- | :--- |
| **流程** | $Query + SQL \rightarrow CoT$ | $Query \rightarrow CoT + SQL$ |
| **因果性** | SQL 引导逻辑（可能马后炮） | **逻辑推导结果（因果关系强）** |
| **蒸馏价值** | 较低，难以迁移推理分布 | **极高，适合学生模型模仿思维模式** |
| **控制难度** | 简单 | 困难（需处理 SQL 错误与逻辑偏差） |

### 2. 自动化执行评测沙箱 (The Evaluation Sandbox)
为了解决方案二带来的质量波动，我开发了一套 Python 高级评测脚本，核心逻辑包括：

* **多维相似度量化**：
    $$Score = 0.5 \cdot RowSim + 0.3 \cdot ColSim + 0.2 \cdot MatchRatio$$
    突破了传统的完全相等匹配，能够识别出“字段重命名”或“排序差异”但结果正确的 SQL。
    


* **列拆分与子集匹配**：
    针对金融场景中常见的字段合并/拆分问题，脚本内置了 `advanced_column_split_match` 逻辑，能够通过组合搜索算法验证模型输出的字段集合。

* **健壮性设计**：
    - **多进程 Pool 隔离**：防止恶意或死循环 SQL 拖垮评测进程。
    - **自动 Checkpoint**：支持大规模数据集的断点续传。

---

## 📊 工程实践产出

* **数据利用率优化**：通过 `subset_matching` 和模糊匹配算法，将有效数据的挽救率提升了 **15%**。
* **训练集纯度**：自动过滤执行报错和逻辑断裂的样本，为模型蒸馏提供了近乎 0 噪声的训练语料。
* **性能监控**：实现了对 SQL 执行耗时、数值精度、容差范围（$1e^{-6}$）的全面监控。

---

## 💻 核心脚本架构展示

```python
# 脚本核心：多维相似度计算片段
def improved_similarity_calculation(answer_rows, output_rows):
    # 1. 执行高级行匹配 (Advanced Row Matching)
    is_match, match_info = advanced_row_matching(answer_rows, output_rows, match_threshold=0.5)
    
    # 2. 计算列相似度 (Column-wise Similarity)
    # ... (基于 Pandas 的高效列对齐算法)
    
    # 3. 加权得分计算
    similarity_score = float(0.5 * row_similarity + 0.3 * col_similarity + 0.2 * match_ratio)
    return similarity_score