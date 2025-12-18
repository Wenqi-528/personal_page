---
title: "NLP Assistant New"
date: 2025-12-17
draft: false
summary: 一个agent应用的落地档案
---


## 🌟 项目综述
在金融业务场景中，非技术人员（理财经理、风控分析师）面临着极高的取数门槛。传统的 Text-to-SQL 方案由于 LLM 的“幻觉”问题，直接生成的 SQL 往往存在字段名错误或逻辑陷阱。

本项目作为**智图 WHY 工具课程建设**的核心案例，主导设计了一套基于 **Function Call (工具调用)** 的混合驱动架构。该架构将 LLM 定位为“意图解析器”而非“代码生成器”，通过调用标准化的后端函数，确保了金融数据查询的 **100% 准确性** 与 **安全性**。



---

## 🛠️ 核心技术实现：从“幻觉生成”到“确定性调用”

### 1. Function Call 逻辑架构设计
我将查询过程拆解为三个关键环节，通过定义严格的 JSON Schema 约束模型输出：

* **语义解析层**：利用 LLM 识别用户问题中的核心实体（如：分行、产品代码）和指标意图（如：余额、逾期率）。
* **函数触发层**：模型不输出 SQL，而是输出一个结构化的 `JSON` 指令，调用我定义的 `get_financial_data()` 函数。
* **逻辑校验层**：后端代码接收到 JSON 参数后，通过预设的 SQL 模板库进行动态拼装，彻底杜绝了全表扫描等高风险操作。

### 2. 精细化的 Schema 定义 (PM 的核心产出)
为了确保 Function Call 的成功率，我主导设计了如下 Schema 结构，重点在于对枚举值（Enum）的严格控制：

```json
{
  "name": "query_bank_metrics",
  "description": "用于查询招商银行分行维度的业务指标数据",
  "parameters": {
    "type": "object",
    "properties": {
      "region": { "type": "string", "description": "分行名称，如：北京分行, 深圳分行" },
      "metric": { 
        "type": "string", 
        "enum": ["AUM", "活期存款", "贷款余额", "不良率"],
        "description": "业务核心指标名称" 
      },
      "time_frame": { "type": "string", "description": "时间区间，支持 YTD, Q3, M10 等格式" }
    },
    "required": ["region", "metric"]
  }
}