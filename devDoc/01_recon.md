# Phase 1 · 项目侦察报告（Recon Report）

> 本报告对应仓库 `graph-rag-agent`（GitHub: 1517005260/graph-rag-agent），当前工作树状态对应 `master` 分支头 `4296b7c Update start.md`。所有结论都可在源码中验证，相关锚点形式为 `路径:行号`。

---

## 1. 一句话定位与目标

**一句话定位**：一个以中文教育/政务文档为对象、用 **LangGraph + Neo4j + OpenAI 兼容 LLM** 复现 GraphRAG 并融合 DeepSearch 推理与 Plan-Execute-Report 多智能体写报告的 **可解释 RAG 教学/研究型项目**。

- **目标场景**：私域知识库问答（默认主题为「华东理工大学学生管理」，见 `graphrag_agent/config/settings.py:63-88`），同时给出 5 种不同复杂度的 Agent 让用户对比。
- **目标用户**：① 想从零跟着代码学 GraphRAG / Multi-Agent 工程的工程师；② 想对接私域知识库快速搭一个问答系统的研究者；③ 想做检索/答案质量评测对比的研究人员（项目自带 20+ 评估指标和 `evaluate_all_agents.py`）。
- **项目官方亮点**（`readme.md:111-119`）：复现 GraphRAG、与 DeepSearch 融合、多 Agent、20+ 评估指标、增量更新、实体消歧/对齐、思考过程可视化。

---

## 2. 技术栈清单

### 2.1 运行时与语言

- Python `>=3.11`（`pyproject.toml:5`，但 `assets/start.md` 写的是 `conda create -n graphrag python==3.10`，README 和 start 文档不完全一致，**留作 issue**）。
- 主要语言：Python，注释/文档/UI 大量中文。

### 2.2 框架与第三方库（`requirements.txt`）

| 类型 | 关键依赖 | 版本 |
|---|---|---|
| Agent 编排 | `langgraph` | 0.3.18 |
| LLM 适配 | `langchain` / `langchain_core` / `langchain_community` / `langchain_openai` / `langchain_neo4j` | 0.3.x |
| 监控 | `langsmith` | 0.3.18 |
| 图数据库客户端 | `langchain_neo4j`，`graphdatascience` | 0.4.0 / 1.12 |
| 向量库（本地） | `faiss-cpu` | 1.11.0（仅用于 cache 向量索引，主向量索引在 Neo4j） |
| 嵌入与重排 | `sentence_transformers`，`OpenAIEmbeddings` | 4.1.0 |
| 中文 NLP | `jieba`，`hanlp` | 0.42.1 / 2.1.1 |
| 文档解析 | `PyPDF2`，`python-docx`，`textract`，`lxml` | — |
| 服务端 | `fastapi`，`uvicorn`，`sseclient-py`，`schedule` | 0.115.11 / 0.29.0 |
| 前端 | `streamlit`，`pyvis`（图谱可视化），`matplotlib` | 1.42.2 / 0.3.2 |
| 工具 | `rich`（终端美化）、`tqdm`、`psutil`、`shutup` | — |

`pyproject.toml` 是占位（没有 dependencies），真正的依赖在 `requirements.txt`。

### 2.3 模型与服务

- **LLM**：`ChatOpenAI`，模型与 base_url 完全可配（`graphrag_agent/models/get_models.py:25-32`），实测可用 `gpt-4o`、`deepseek-chat (20241226)`（README 中标注兼容性）。
- **Embeddings**：`OpenAIEmbeddings`（`graphrag_agent/models/get_models.py:22-24`），默认 `text-embedding-3-large`。
- **代理/路由**：推荐通过 [One-API](https://github.com/songquanpeng/one-api) 或 `new-api` 走 OpenAI 兼容协议（`.env.example:5`，`assets/start.md`）。
- **图数据库**：Neo4j 5.22.0 + APOC + GDS（`docker-compose.yaml`）。GDS 用于 Leiden / SLLPA 社区检测。
- **Streaming**：项目自述「伪流式」（`CLAUDE.md` 与 `graphrag_agent/agents/base.py:115-164`：先生成全文再切句下发，**没有用 `astream_events` 真流式**，全局 grep `astream_events` 为空）。

### 2.4 部署形态

- **三层架构**：① 核心包 `graphrag_agent/` ② `server/`（FastAPI）③ `frontend/`（Streamlit）。三层都通过 `.env` 与 `graphrag_agent/config/settings.py` 配置。
- **Docker**：只有 Neo4j 用 docker-compose 启动；One-API / 后端 / 前端均裸跑。

---

## 3. 顶层目录树（每个目录一句话职责）

```
graph-rag-agent/
├── graphrag_agent/                       # 核心包：所有图谱、检索、Agent、缓存、评估代码
│   ├── agents/                           # 五种 Agent + multi_agent 子包（Plan-Execute-Report）
│   │   └── multi_agent/                  # 子图编排栈：core/planner/executor/reporter/tools/integration
│   ├── cache_manager/                    # 两级缓存（session-aware + global）+ 向量相似缓存
│   ├── community/                        # Neo4j GDS 社区检测（Leiden / SLLPA）与 LLM 摘要
│   ├── config/                           # settings.py / neo4jdb.py / prompts/（9 个分类提示词文件）
│   ├── evaluation/                       # 评估框架：core/metrics/evaluators/preprocessing/test
│   ├── graph/                            # 图谱构建：extraction/processing(消歧+对齐)/indexing/structure
│   ├── integrations/build/               # 一键全量 / 增量构建脚本（KGBuilder→IndexCommunity→ChunkIndex）
│   ├── models/                           # LLM / Embeddings 获取与 token 计数
│   ├── pipelines/ingestion/              # 多格式文件读取 + 中文分块（jieba/HanLP）
│   └── search/                           # local/global/hybrid/naive/chain_exploration + tool 注册表
│       └── tool/reasoning/               # 深度研究内核：thinking/evidence/kg_builder/validator 等
├── server/                               # FastAPI 服务端：routers + services + utils + server_config
├── frontend/                             # Streamlit 前端：app.py + components + utils
├── test/                                 # 流式/非流式 / Hotpot 多智能体评测脚本
├── training/                             # 可选：GRPO 训练实体抽取模型 + 微调 embedding（`grpo.ipynb`）
├── evaluation/test (in graphrag_agent)   # 评估测试集与 README，已涵盖所有 Agent 评测
├── datasets/                             # 数据集（默认空，readme.md 占位）
├── files/                                # 默认知识源（4 个学校 PDF/DOC + txt 文件夹）
├── assets/                               # 文档图片、`start.md` 快速开始
├── structPrompt/                         # 项目说明文档（未在代码中引用）
├── server / frontend / test / training/  # 顶层独立目录，便于分别部署
├── docker-compose.yaml                   # 仅 Neo4j
├── .env.example                          # 详细分类的运行参数（数百项）
├── readme.md / CLAUDE.md / AGENTS.md     # 项目说明 / Claude Code 项目说明 / Agent 协作指引
└── pyproject.toml / requirements.txt / setup.py
```

每个核心子目录都自带 `readme.md`（共 14 个，见 `find ... -name "readme.md"`），可作为二级文档来源。

---

## 4. 入口文件 / 启动方式 / 主流程调用链

### 4.1 三大入口

| 入口 | 启动命令 | 入口文件 |
|---|---|---|
| 知识图谱构建 | `python graphrag_agent/integrations/build/main.py` | `graphrag_agent/integrations/build/main.py:55-61` |
| 增量更新 | `python graphrag_agent/integrations/build/incremental_update.py --once / --daemon` | `graphrag_agent/integrations/build/incremental_update.py:475` (main) |
| FastAPI 后端 | `python server/main.py` | `server/main.py:32-33` |
| Streamlit 前端 | `streamlit run frontend/app.py` | `frontend/app.py:45-47` |
| 终端测试 | `python test/search_without_stream.py` / `search_with_stream.py` | `test/` |
| 评估 | `python graphrag_agent/evaluation/test/evaluate_all_agents.py …` | 同上 |

### 4.2 构建期主流程（Build pipeline）

`KnowledgeGraphProcessor.process_all` (`integrations/build/main.py:20-53`) 依次：

```
0. connection_manager.drop_all_indexes()                  # 清旧索引
1. KnowledgeGraphBuilder.process()  -> build_graph.py:36  # 文件→分块→实体抽取→Neo4j 写入
2. IndexCommunityBuilder.process()                        # 实体向量索引 + Leiden/SLLPA 社区 + LLM 社区摘要
3. ChunkIndexBuilder.process()                            # Chunk 向量索引（依赖前两步）
```

每一步背后的核心组件：

- 文件读取：`pipelines/ingestion/file_reader.py:14`（TXT/PDF/MD/DOC/DOCX/CSV/JSON/YAML，递归扫描）。
- 中文分块：`pipelines/ingestion/text_chunker.py:7` `ChineseTextChunker`（jieba + 句末检测，`CHUNK_SIZE=500, OVERLAP=100`，可配）。
- 实体/关系抽取：`graph/extraction/entity_extractor.py:16` `EntityRelationExtractor`（LLM + ChatPromptTemplate + 本地 pickle 缓存 + `concurrent.futures` 并发，`MAX_WORKERS=4`）。
- 图结构写入：`graph/structure/struct_builder.py:10` `GraphStructureBuilder`（批量 `MERGE` + 并发 chunk 处理）。
- 实体消歧：`graph/processing/entity_disambiguation.py:15` `EntityDisambiguator`（字符串召回 → 向量重排 → NIL 检测）。
- 实体对齐：`graph/processing/entity_alignment.py:12` `EntityAligner`（同 canonical_id 下冲突检测与合并）。
- 社区检测：`community/detector/leiden.py:7`（GDS Leiden + 内存自适应参数）、`community/detector/sllpa.py`。
- 社区摘要：`community/summary/leiden.py:1` `community/summary/sllpa.py:1`（LLM 生成结构化摘要）。

### 4.3 问答期主流程（Inference pipeline）

#### ① FastAPI → AgentManager → Agent

```
HTTP POST /chat            server/routers/chat.py:13
       ↓
process_chat / process_chat_stream    server/services/chat_service.py
       ↓
agent_manager.get_agent(agent_type, session_id)   server/services/agent_service.py:34
       ↓
agent.ask(...) / agent.ask_stream(...)            BaseAgent in graphrag_agent/agents/base.py
```

`AgentManager` 维护 `{agent_type}:{session_id}` 粒度的实例池（`server/services/agent_service.py:48-56`），用 `threading.RLock` 保证并发。

#### ② 五种 Agent 的内部流程

所有 Agent 都继承自 `BaseAgent`（`graphrag_agent/agents/base.py:21-781`），主要骨架：

```
ask(query) →
  ├── 全局缓存 (GlobalCacheKeyStrategy)
  ├── 快速缓存 check_fast_cache（高质量）
  ├── 会话缓存 ContextAwareCacheKeyStrategy
  └── miss → graph.stream(inputs, config)         # LangGraph 主流程
                      ↓
        START → agent (_agent_node, 抽关键词→bind_tools→llm)
              → tools_condition
              → retrieve (ToolNode) → generate / reduce / END
```

各 Agent 的差异在 `_setup_tools` 与 `_add_retrieval_edges`：

| Agent | 工具 | 路由特点 |
|---|---|---|
| `NaiveRagAgent` (`naive_rag_agent.py:14`) | `NaiveSearchTool.get_tool()` | retrieve → generate（最简单） |
| `GraphAgent` (`graph_agent.py:26`) | `LocalSearchTool` + `GlobalSearchTool.search` | retrieve → 条件路由 → generate 或 reduce |
| `HybridAgent` (`hybrid_agent.py:20`) | `HybridSearchTool.get_tool()` + `get_global_tool()` | retrieve → generate |
| `DeepResearchAgent` (`deep_research_agent.py:21`) | `DeepResearchTool` / `DeeperResearchTool` 的 thinking/stream/exploration/reasoning_analysis 工具 | retrieve → generate；额外提供 `explore_knowledge`、`detect_contradictions`、`analyze_reasoning_chain` |
| `FusionGraphRAGAgent` (`fusion_agent.py:24`) | **绕过 LangGraph**，直接委托给 `MultiAgentFacade` | 见下 |

#### ③ Plan-Execute-Report 流程（FusionGraphRAGAgent）

```
FusionGraphRAGAgent.ask(query)               agents/fusion_agent.py:39
     ↓ MultiAgentFacade.process_query        multi_agent/integration/legacy_facade.py:34
     ↓ MultiAgentOrchestrator.run            multi_agent/orchestrator.py:108
     ├── Plan：BasePlanner.generate_plan     multi_agent/planner/base_planner.py:127
     │      ├── Clarifier.analyze
     │      ├── TaskDecomposer.decompose → TaskGraph(拓扑 DAG)
     │      ├── PlanReviewer.review → PlanSpec
     │      └── _ensure_reflection_task（按配置追加 reflection 节点）
     ├── Execute：WorkerCoordinator.execute_plan   executor/worker_coordinator.py:77
     │      ├── 三个执行器：RetrievalExecutor / ResearchExecutor / ReflectionExecutor
     │      ├── 串行 (_execute_sequential) / 并行 (_execute_parallel, ThreadPoolExecutor)
     │      └── 反思自动重试 _handle_reflection_retry
     └── Report：BaseReporter.generate_report      reporter/base_reporter.py:136
            ├── OutlineBuilder.build_outline → ReportOutline
            ├── 选择 traditional / Map-Reduce 写作（阈值 MA_MAPREDUCE_THRESHOLD=20）
            │     - Map: EvidenceMapper.map_evidence_batch（可并行）
            │     - Reduce: SectionReducer.reduce（tree/collapse/refine 三种策略）
            │     - Assemble: ReportAssembler.assemble
            ├── ConsistencyChecker.check（默认开启）
            ├── _append_evidence_annex 追加证据附录
            ├── CitationFormatter.format_references
            └── report_id 级缓存（_save_report_cache, 按 evidence_fingerprint）
```

数据流主类型：

- `PlanSpec` / `TaskGraph` / `TaskNode` (`multi_agent/core/plan_spec.py:67-419`)
- `PlanExecutionSignal`（拓扑序）(`plan_spec.py:409-419`)
- `PlanExecuteState`（LangGraph 总状态，`multi_agent/core/state.py:163-268`）
- `ExecutionRecord` / `ToolCall` / `ReflectionResult` (`multi_agent/core/execution_record.py:14-238`)
- `RetrievalResult` 统一证据格式（`multi_agent/core/retrieval_result.py:99-217`），所有检索工具通过 `search/retrieval_adapter.py` 适配为该格式。

---

## 5. 能力清单 ✅/❌

下面对照原始**重点技术点清单**逐项打勾，括号内是定位锚点。

### A. RAG 架构与高级优化

| 能力 | 状态 | 证据 / 位置 |
|---|---|---|
| Pre-Retrieval：Chunking 策略 | ✅ **递归字符 / 中文句末** | `pipelines/ingestion/text_chunker.py:7-292`，基于 jieba 分词 + 句末检测，但**没有 Semantic / Parent-Document / Sentence-Window**。 |
| 多租户向量隔离 | ❌ 未实现 | 全局只有一个 Neo4j 向量索引 `vector`（`config/settings.py:273`），无 namespace/租户字段。 |
| Query Rewrite / Decompose / HyDE / Step-back | 🟡 **部分**：澄清+任务分解、假设生成 | Clarifier 重写 (`planner/clarifier.py:53`)；TaskDecomposer (`planner/task_decomposer.py:36`)；`HypothesisGeneratorTool`（hypothesis_tool.py，列在 `tool_registry.EXTRA_TOOL_FACTORIES`），DeeperResearchTool 内部还有 `_hypothesis_cache`（`deeper_research_tool.py:365-366`）。**未实现典型 HyDE / Step-back 提示式重写**。 |
| Hybrid Search（向量+BM25） | ❌ **不是 BM25**，是「图结构+向量」 | `HybridSearchTool` (`search/tool/hybrid_tool.py:30`) 是 LightRAG 风格的局部细节 + 全局主题双级检索，结合实体跳数与社区，**没有 BM25**。全仓 grep `BM25 / bm25` 无结果。 |
| Multi-Query Retriever | ❌ 未实现 | 无 `MultiQueryRetriever`。 |
| MMR vs Similarity | 🟡 仅 Similarity | `Neo4jVector.similarity_search`，没有 `max_marginal_relevance` 调用。 |
| Rerank（Cohere/BGE） | 🟡 **仅在实体消歧** | `entity_disambiguation.vector_rerank` (`entity_disambiguation.py:61-98`) 做字符串召回后向量重排，**不是 Cohere/BGE**，只是 cosine。检索路径里没有外置 reranker。 |
| 「Lost in the Middle」缓解 | ✅ Map-Reduce + 一致性检查 | Reporter 的 Map-Reduce 模式（`base_reporter.py:397-555`）就是为长上下文整合设计的；额外有 `ConsistencyChecker` 二次校验。 |
| Contextual Compression | ❌ 未实现 | 全仓无 `ContextualCompressionRetriever`。 |
| GraphRAG | ✅✅ 项目核心 | `community/`、`graph/`、`search/local_search.py`、`search/global_search.py` 全部完整复现 GraphRAG 论文流水线。 |
| 多模态 RAG | ❌ 仅文本 | `file_reader.py:14-409` 只解析文本/表格，**PDF 图片、OCR、表格结构化均不处理**。 |
| 评测（Faithfulness/Context Precision/Answer Relevancy） | ✅ **自研一套**，不是 RAGAS/TruLens | `evaluation/metrics/`：`FactualConsistency`、`ResponseCoherence`、`RetrievalPrecision`、`RetrievalUtilization`、`ChunkUtilization`、`SubgraphQualityMetric`、`ReasoningCoherence`、`KnowledgeGraphUtilizationMetric` 等共 17 个指标类（`evaluation/metrics/*.py`）。 |
| 黄金测试集 | 🟡 | `evaluation/test/evaluate_all_agents.py:13-20` 支持外部传入 `questions_file` / `golden_answers_file`，自带的 `datasets/` 是空的占位。 |

### B. LangGraph 与复杂状态机

| 能力 | 状态 | 证据 |
|---|---|---|
| State 管理 / Cycles / Recursion | ✅ | `BaseAgent._setup_graph` (`base.py:81-113`) 用 `StateGraph` 和 `tools_condition`；`recursion_limit` 来自 `AGENT_SETTINGS["default_recursion_limit"]=5`（`config/settings.py:294`）。 |
| Checkpointer / Time Travel | 🟡 仅 `MemorySaver` | `base.py:41 self.memory = MemorySaver()`，**没有持久化到 PG/Redis**，所以没有真正的「Time Travel」。 |
| Human-in-the-Loop / interrupt | ❌ 未实现 | 无 `interrupt_before / interrupt_after / dynamic_interrupt`。澄清是通过返回 `clarification.questions` 让上层决定，**不是 LangGraph interrupt**。 |
| Conditional Edges / Router | ✅ | `GraphAgent._grade_documents` (`graph_agent.py:112-169`) 是典型路由；Tool 选择由 `bind_tools + tools_condition` 处理。 |
| Parallelism / Send / Map-Reduce | 🟡 **未使用 LangGraph Send**，是手写并发 | `WorkerCoordinator._execute_parallel` (`worker_coordinator.py:150-256`) 用 `ThreadPoolExecutor`；Reporter Map 阶段同样 (`base_reporter.py:502-555`)。无 `Send` API。 |
| Subgraph | ❌ 未实现 | 全仓 grep `add_subgraph` 无结果；多智能体编排其实**绕过 LangGraph**（`FusionGraphRAGAgent.graph = _GraphShim()`，`fusion_agent.py:31`）。 |

### C. Agent 核心能力

| 能力 | 状态 | 证据 |
|---|---|---|
| ReAct / Plan-and-Execute / Reflexion | ✅ 三种都有 | Naive/Hybrid/Graph 是 ReAct 风格；DeepResearchAgent 用 `thinking` 多回合思考-搜索循环；FusionGraphRAGAgent 是 Plan-Execute-Report；`ReflectionExecutor`（`executor/reflector.py:30`）+ `_handle_reflection_retry` 提供 Self-Reflection。 |
| Tool Selection（>50） | ❌ 不存在 | 现有工具不到 10 个（`TOOL_REGISTRY` + `EXTRA_TOOL_FACTORIES` 共 9，`search/tool_registry.py:20-34`），没有 RAG-over-Tools。 |
| Multi-Agent 拓扑（Supervisor/Swarm/...） | ✅ **Hierarchical/Supervisor 混合** | Orchestrator 是 Supervisor，Planner/Worker/Reporter 三段流水线；无 Swarm/Network。 |
| 记忆：Short-term / Long-term | 🟡 | Short-term：`MemorySaver` + 两级 CacheManager（`cache_manager/manager.py:13-394`）；Long-term：**没有真正跨会话长期记忆**，仅靠全局缓存（同 query 命中）和 Neo4j 知识图谱。 |
| Function Calling（OpenAI tools / 结构化输出 / JSON mode） | 🟡 | Agent 主流程用 `llm.bind_tools` (`base.py:215`)；Multi-Agent 大量子组件用「prompt 让 LLM 输出 JSON → `parse_json_text`」自定义解析（`multi_agent/tools/json_parser.py:29-72`），**未使用 `with_structured_output` / Pydantic Parser**。 |

### D. MCP / Skills / 上下文工程

| 能力 | 状态 | 证据 |
|---|---|---|
| MCP Server/Client | ❌ 完全没有 | 全仓 grep `mcp` 无结果。 |
| Skills | ❌ | 无 SKILL.md / `claude_skills` 等。 |
| Context Engineering | 🟡 | Prompt 都集中在 `graphrag_agent/config/prompts/`（agent/executor/graph/planner/qa/reasoning/reporter/search 八个分类），有清晰的分层；通过 `MA_SECTION_MAX_CONTEXT_CHARS=800` 等控制注入长度。但没有动态优先级压缩。 |

### E. 生产环境与工程挑战

| 能力 | 状态 | 证据 |
|---|---|---|
| 可观测性：LangSmith / Langfuse / OTel | 🟡 **仅 LangSmith**，且只有一处 | `search/tool/local_search_tool.py:4` `from langsmith import traceable`；`.env.example` 第 188-199 行的 `LANGSMITH_*` 是可选环境变量。无 Langfuse / OpenTelemetry。 |
| 流式输出 | 🟡 **伪流式** | `BaseAgent._stream_process` (`base.py:115-164`)：先 `_generate_node_async` 生成完整答案再按句号切分 yield，**不是 token-level 真流式**，也没有 `astream_events`。 |
| Hallucination 防控（Guardrails/NeMo/结构化校验/自洽性投票） | 🟡 | `AnswerValidator` (`search/tool/reasoning/validator.py:3`) + `AnswerValidationTool` (`search/tool/validation_tool.py:11`)；`ConsistencyChecker` (`reporter/consistency_checker.py:28`) 做答案-证据一致性检查；`_default_validation`（`cache_manager/manager.py:374-389`）做关键词覆盖度兜底。无 NeMo Guardrails、无自洽性投票。 |
| Fallback / RateLimiter / 重试 | 🟡 **轻量级** | `LeidenDetector._execute_fallback_leiden` 做参数降级（`community/detector/leiden.py:43`）；`entity_extractor.py` 有 `retry` 装饰器 (`graph/core` 提供)；**未使用 LangChain `with_fallbacks` / `RateLimiter`**。 |
| Output Parsing Errors | 🟡 | 多智能体用 `parse_json_text`：找 `{...}` 块 → `json.loads`，失败抛 `ValueError`（`multi_agent/tools/json_parser.py:29-72`）。无 `OutputFixingParser / RetryWithErrorOutputParser`。 |
| 成本与延迟 | ✅ **缓存层做得最完善** | 两级 `CacheManager` + 向量相似缓存（`cache_manager/vector_similarity/`）、`get_fast`/`check_fast_cache` 快速路径、report_id 级缓存与 evidence_fingerprint 重用（`base_reporter.py:761-852`）。无 Anthropic Prompt Cache 接入。 |
| 安全（Prompt Injection / 脱敏 / 工具沙箱） | ❌ 未涉及 | 无任何 prompt injection 防御、PII 脱敏、工具沙箱。 |
| 测试：LLM-as-a-Judge / 回归 / A/B | ✅ LLM-as-Judge | `evaluation/metrics/llm_metrics.py:329 LLMGraphRagEvaluator`（用 LLM 评分 5 维度），`evaluate_all_agents.py` 支持多 Agent 共指标横向对比；无显式 A/B。 |

---

## 6. 项目里**确实存在 vs 确实不存在**的能力一览（速查表）

### ✅ 确实存在

- **GraphRAG 全流程**（实体抽取 → Neo4j → 社区检测 → Map/Reduce 全局检索）
- **5 种 Agent + 多智能体 Plan-Execute-Report 编排**
- **实体消歧 + 实体对齐**（业界少见的开源实现）
- **增量更新机制**（`incremental_update.py --daemon` + `file_registry.json`）
- **两级智能缓存 + 向量相似缓存**
- **DeepResearch 双工具（标准 + 增强版）**（`DeepResearchTool` / `DeeperResearchTool`，后者带社区感知、动态 KG 构建、推理链追踪、矛盾检测）
- **Chain of Exploration**（`search/tool/reasoning/chain_of_exploration.py:9`）
- **完整 FastAPI + Streamlit 前后端 + 图谱可视化**
- **20+ 评估指标 + LLM-as-Judge**
- **Map-Reduce 长文档生成（3 种 Reduce 策略：tree/collapse/refine）**
- **PlanSpec 拓扑排序 + 并行执行 + 反思自动重试**
- **report_id + evidence_fingerprint 级 Reporter 缓存**

### ❌ 确实不存在（避免学习时踩空）

- **BM25 / Ensemble Retriever / Cohere/BGE Reranker / MMR / Multi-Query / ContextualCompression / HyDE / Step-back**（任何典型 RAG 高级检索优化均未引入）
- **LangGraph `Send` API / Subgraph / interrupt / dynamic interrupt**
- **真正的 token 级流式（`astream_events`）**
- **MCP / Skills / 任何外部协议适配**
- **多租户向量隔离 / PII 脱敏 / Prompt Injection 防御 / 工具沙箱**
- **Langfuse / OpenTelemetry / 多模型 fallback / 显式 RateLimiter**
- **Long-term Memory（vector / Postgres / Redis）**
- **多模态 RAG（图片/表格结构化）**
- **持久化 checkpointer（仅 `MemorySaver`）**

---

## 7. 阅读源码时易踩的坑（侦察期间发现）

1. **`fusion_agent.py` 绕过 LangGraph**：`self.graph = _GraphShim()` (`fusion_agent.py:30-31`)，新人会以为它走 LangGraph 主图，实际所有工作交给 `MultiAgentFacade`。
2. **Streamlit 前端调用 `/chat`，对深度研究**：默认 timeout=120s 太短，`CLAUDE.md` 明确说需要把 `frontend/utils/api.py` 的 `timeout=120` 注释掉。
3. **构建顺序强约束**：`Entity Index 必须在 Chunk Index 之前`，否则 `ChunkIndexBuilder` 会找不到 vector property，主流程已在 `integrations/build/main.py:28-44` 串好，**不要单独跑 step 3**。
4. **Python 版本不一致**：`pyproject.toml` 要 `>=3.11`，README 又叫 `python==3.10`，实际依赖以 3.10 经过测试。学习时建议跟 README。
5. **embedding 模型选择**：`config/settings.py:179 CACHE_EMBEDDING_PROVIDER` 默认 `sentence_transformer`，而 `.env.example` 里写的是 `openai`，**两者不一致**，以 env 为准。
6. **多智能体的「并行」并非 LangGraph**：是 `ThreadPoolExecutor`，靠 `MA_WORKER_EXECUTION_MODE=parallel` 切换（`config/settings.py:341-348`）。
7. **DeepResearchTool 1220 行 / DeeperResearchTool 2134 行**：是项目最重的两个文件，强烈建议放到学习路径的中后段。
8. **prompts 集中管理**：8 个 `*_prompts.py`（`config/prompts/`）才是真正的「业务逻辑」，比代码本身更关键，要逐个读。

---

## 8. 无法访问 / 未发现的资源（如需我处理请告知）

- **`datasets/`** 仅有 `readme.md` 占位，没有真实问答数据集；如需复现评估，需自带 QA 数据。
- **README 链接的「deepwiki」/ `zread.ai`** 是外站，未读取，**如需我抓取其总览作为辅助阅读，请确认是否允许联网**。
- **`training/grpo.ipynb`** 是可选训练入口（GRPO + embedding 微调），未细读；**如需纳入学习路线请告知**。

---

## 9. 一句话总结（送给学习者）

> **这是一个把 GraphRAG 学术流水线 + 多 Agent 编排做到工程级别完整度的中文项目，但在「检索高级优化」和「生产可观测/安全」两端是空白；非常适合用来学 GraphRAG / LangGraph / 多 Agent 模式，但不要把它当「生产级 RAG 框架最佳实践」来背。**

---

**Phase 1 完成。请确认是否进入 Phase 2（学习路线规划）。如有以下需要先告知：**

1. 是否把 `training/grpo.ipynb` 纳入学习路线（否则按可选附录处理）
2. 是否需要联网抓取 `deepwiki.com/1517005260/graph-rag-agent` 或 `zread.ai` 作为补充
3. 你的优先目标更偏向 **「学 GraphRAG 原理」** 还是 **「学 LangGraph + 多 Agent 工程」**，二者在后续 12+ 篇文档的权重会不同
