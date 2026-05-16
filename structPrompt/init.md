我想通过AI学习一个开源项目https://github.com/skygazer42/GustoBot
我想要AI分析整个项目的架构，核心方法逻辑，规划学习路线等，一步一步指导我学习这个项目，生成项目md文档供我学习。

重点学习agent相关的内容，例如多agent  human in loop，function call， 记忆管理，上下文管理， rag， 检索，召回，重排等 ,MCP SKILLS Langchian Langgraph workflow, retrieve VectorDB (请你再额外补充其他关键技术点)
以及本项目的核心技术点。

重点关注以下内容
## 一、 RAG 核心与高级优化 (RAG Architecture & Optimization)

### 1. 检索前 (Pre-Retrieval)

- 如何根据业务场景选择最合适的 **Chunking (切片)** 策略？（如 Semantic Chunking 与 Recursive Character 的差异）
    
- 在高并发场景下，如何设计多租户 (Multi-tenant) 的向量隔离方案？
    
- **Query Transformation**: 如何实现多维度的 Query 重写 (Rewrite)、分解 (Decompose) 和回退 (Step-back) 以提升检索召回率？
    

### 2. 检索中与检索后 (Retrieval & Post-Retrieval)

- **Hybrid Search**: 向量检索与关键词检索（如 BM25）的权重比例如何动态调优？
    
- **Rerank 模型**: 为什么要引入重排序？它解决的是 Embedding 模型的什么局限性？
    
- **GraphRAG**: 在处理非结构化文档或需要全局总结的任务时，GraphRAG 相比传统向量 RAG 的核心优势和落地难点是什么？
    
- 如何解决检索内容中的 **“Lost in the Middle”**（长上下文中间信息丢失）问题？
    

### 3. 评估与工程化

- 如何建立 RAG 的黄金测试集？**RAGAS** 或 **TruLens** 的评估维度（忠实度、相关性、答案相关性）在实际业务中如何闭环？
    
- 面对海量非结构化数据（如 PDF 里的表格和图片），你的多模态解析方案是什么？
    

---

## 二、 LangGraph 与复杂状态机 (Agentic Workflows)

### 1. LangGraph 核心机制

- **State Management**: LangGraph 相比 LangChain 原生的 LCEL Chain，在处理状态保存和循环逻辑时有哪些本质区别？
    
- **Cycles & Recursion**: 如何在图中防止 Agent 陷入死循环？如何设置合理的递归上限（Recursion Limit）？
    
- **Checkpointer**: 深入解释 LangGraph 的持久化层机制，如何实现 **"Time Travel" (状态回溯与调试)**？
    

### 2. 任务编排与控制

- **Human-in-the-loop (HITL)**: 如何在 LangGraph 节点间插入人工干预点（如：断点审批、状态注入）？
    
- **Conditional Edges**: 如何设计复杂的逻辑分支，确保 Agent 能根据上一个节点的输出准确选择 Tool 或结束任务？
    
- **Parallelism**: 如何在 LangGraph 中实现多个 Node 的并发执行？
    

---

## 三、 Agent 核心能力与工具集成 (Agentic Reasoning)

### 1. 规划与反思 (Planning & Reflection)

- **ReAct** 模式的优缺点是什么？在什么场景下你会选择 **Plan-and-Execute** 或 **Self-Reflection** 模式？
    
- 当 LLM 生成的 Plan 过长时，如何保证每一步执行的准确性？
    
- 如何优化 Agent 的 **Tool Selection (工具选择)**？当 Tool 数量超过 50 个时，如何处理上下文超出或模型混淆问题？
    

### 2. 记忆系统 (Memory)

- **Long-term vs. Short-term Memory**: 如何设计 Agent 的记忆架构？什么时候该用向量数据库，什么时候该用关系型数据库（如 PostgreSQL）存储对话上下文？
    
- **Summary Memory**: 如何动态总结历史对话，以在有限的 Token 窗口内保留尽可能多的关键信息？
    

---

## 四、 生产环境与工程挑战 (Production & Scaling)

### 1. 性能与延迟

- 如何平衡 Agent 的多步推理（Reasoning Loop）带来的高延迟与用户体验？
    
- **Streaming**: 如何在 LangChain/LangGraph 中实现中间步骤（Intermediate Steps）的流式输出？
    

### 2. 稳定性与可靠性

- **Hallucination Control**: 如何利用 **Guardrails** (如 NeMo Guardrails) 或自研逻辑来拦截非法或幻觉输出？
    
- **Fallback 机制**: 当主模型（如 GPT-4o）调用失败或速率受限时，如何设计降级方案？
    
- 如何处理工具调用过程中的解析错误（Output Parsing Errors）？


每个md的最后，如果当前模块的内容可以让我动手debug，实操，请提供相应的实操教程，如最小可运行示例等
各个md应当拆分合适，md的数量不要太多也不要太少，应该按照逻辑，模块编写

综上请给出提示词，以便于AI能高效理解并完成我的需求