```text
RetrievalEngine
│
├── query_model            # 本轮检索请求的标准对象
│   ├── query
│   ├── filters
│   ├── namespaces
│   └── retrieval_context
│
├── query_policy           # 如何理解并处理 Query
│   ├── normalization
│   ├── rewrite
│   ├── decomposition
│   └── entity_extraction
│
├── retriever_registry     # 系统有哪些 Retriever
│   ├── semantic
│   ├── keyword
│   ├── metadata
│   ├── graph
│   └── memory
│
├── routing_policy         # 本轮使用哪些 Retriever
│   ├── route_strategy
│   ├── retriever_selection
│   ├── parallelism
│   └── fallback
│
├── retrieval_policy       # Retriever 的召回策略
│   ├── top_k
│   ├── threshold
│   ├── filters
│   └── depth
│
├── fusion_policy          # 多路召回如何合并
│   ├── normalization
│   ├── deduplication
│   ├── weighted_fusion
│   └── reciprocal_rank_fusion
│
├── scoring_policy         # 如何计算候选相关性
│   ├── semantic
│   ├── keyword
│   ├── recency
│   ├── importance
│   └── graph
│
├── rerank_policy          # 如何精排候选
│   ├── reranker
│   ├── rerank_top_k
│   ├── output_top_k
│   └── threshold
│
├── postprocess_policy     # 最终结果整理
│   ├── filter
│   ├── deduplicate
│   ├── diversity
│   └── reorder
│
└── metadata               # Retrieval Engine 管理信息
    ├── id
    ├── version
    ├── namespace
    └── tags
```

# 1. Query Model

所有检索请求首先转换为统一 `RetrievalQuery`。

```python
class RetrievalQuery:
    query: str

    filters: dict
    namespaces: list[str]

    query_embedding: list[float] | None

    entities: list[str]
    keywords: list[str]

    time_range: dict | None

    context: dict
```

功能：

```text
query
→ 原始检索文本

filters
→ metadata / type / source 等硬过滤条件

namespaces
→ 限制检索空间

query_embedding
→ Semantic Retriever 使用

entities
→ Graph Retriever 使用

keywords
→ Keyword Retriever 使用

time_range
→ 时间检索约束

context
→ 当前 Session / Task 提供的检索上下文
```

完整入口：

```text
Agent / Context Compiler
        ↓
RetrievalQuery
        ↓
QueryProcessor
        ↓
RetrievalEngine
```

---

# 2. Query Policy

Query Policy 决定原始查询如何转换成 Retriever 可以执行的请求。

```python
class QueryPolicy:
    normalization: bool

    rewrite: bool
    decomposition: bool

    keyword_extraction: bool
    entity_extraction: bool

    embedding: bool
```

运行：

```text
Raw Query
    ↓
Normalization
    ↓
Query Rewrite
    ↓
Keyword Extraction
    ↓
Entity Extraction
    ↓
Embedding
    ↓
RetrievalQuery
```

支持：

```text
direct_query
rewrite_query
multi_query
decomposed_query
entity_query
```

Query Policy 不负责：

```text
实际检索
结果排序
Context 构建
```

---

# 3. Retriever Base

所有 Retriever 必须实现统一接口。

```python
class BaseRetriever:

    def retrieve(
        self,
        query: RetrievalQuery
    ) -> list[RetrievalCandidate]:
        ...

    async def aretrieve(
        self,
        query: RetrievalQuery
    ) -> list[RetrievalCandidate]:
        ...
```

标准输出：

```python
class RetrievalCandidate:
    id: str

    content: str
    source_type: str
    source_id: str

    score: float | None

    metadata: dict

    provenance: dict

    retriever: str
```

因此：

```text
Vector Retriever
Keyword Retriever
Graph Retriever
Memory Retriever
Metadata Retriever
```

全部统一成：

```text
RetrievalQuery
      ↓
BaseRetriever
      ↓
RetrievalCandidate[]
```

---

# 4. Retriever Registry

Retriever Registry 管理 Retrieval Engine 可用的检索能力。

```python
class RetrieverRegistry:

    def register(name, retriever): ...

    def get(name): ...

    def list(): ...
```

建议初始支持：

```text
semantic_retriever
keyword_retriever
metadata_retriever
graph_retriever
memory_retriever
```

后续可以增加：

```text
sql_retriever
web_retriever
repository_retriever
tool_result_retriever
```

但所有实现必须保持统一输出。

```text
RetrieverRegistry
       │
       ├── semantic
       ├── keyword
       ├── metadata
       ├── graph
       └── memory
```

---

# 5. Semantic Retriever

负责向量语义召回。

```python
class SemanticRetriever(BaseRetriever):

    vector_store: VectorStore

    top_k: int
    score_threshold: float | None
```

执行：

```text
Query
 ↓
Embedding
 ↓
Vector Search
 ↓
Metadata Filter
 ↓
Semantic Candidates
```

基础接口：

```python
vector_store.search(
    vector=query_embedding,
    top_k=top_k,
    filters=filters
)
```

Semantic Retriever 只负责：

```text
高召回语义搜索
```

不承担最终排序。

---

# 6. Keyword Retriever

负责 lexical / exact-match 类搜索。

```python
class KeywordRetriever(BaseRetriever):

    index: KeywordIndex

    top_k: int
```

可实现：

```text
BM25
TF-IDF
inverted index
exact term match
```

执行：

```text
Query
 ↓
Keyword Extraction
 ↓
Keyword Index
 ↓
Keyword Candidates
```

主要作用：

```text
专有名词
代码名称
数字
ID
缩写
精确术语
```

补偿纯 embedding retrieval 的召回缺陷。

---

# 7. Metadata Retriever

负责结构化字段检索。

```python
class MetadataRetriever(BaseRetriever):

    metadata_store: MetadataIndex
```

支持：

```text
source
author
document_type
namespace
category
topic
created_at
updated_at
importance
status
```

例如：

```yaml
filters:
  source_type: research_report
  domain: finance

  published_at:
    gte: 2026-01-01
```

执行：

```text
RetrievalQuery.filters
       ↓
MetadataIndex
       ↓
Filtered Candidate Set
```

Metadata Filter 可以同时作为其他 Retriever 的前置过滤条件。

---

# 8. Memory Retriever

Memory Retriever 是 Retrieval Engine 对 Memory System 的适配层。

```python
class MemoryRetriever(BaseRetriever):

    memory_store: MemoryStore

    memory_types: list[str]

    top_k: int
```

可检索：

```text
working
episodic
semantic
procedural
profile
```

执行：

```text
Query
 ↓
Memory Scope
 ↓
Vector / Metadata Search
 ↓
Time / Importance Weight
 ↓
Memory Candidates
```

Memory Retriever 不负责：

```text
记忆写入
记忆更新
冲突处理
遗忘
```

这些属于 Memory System。

---

# 9. Graph Retriever

Graph Retriever 负责关系结构检索。

```python
class GraphRetriever(BaseRetriever):

    graph_store: GraphStore

    sub_retrievers: list[BaseGraphRetriever]
```

建议至少支持：

```text
entity_retriever
vector_context_retriever
relation_retriever
path_retriever
synonym_retriever
structured_query_retriever
```

结构：

```text
GraphRetriever
│
├── EntityRetriever
├── VectorContextRetriever
├── RelationRetriever
├── PathRetriever
├── SynonymRetriever
└── StructuredQueryRetriever
```

运行：

```text
Query
 ↓
Entity Extraction
 ↓
Entity Resolution
 ↓
Graph Lookup
 ↓
Relation Expansion
 ↓
Path Expansion
 ↓
Graph Candidates
```

例如：

```text
OpenAI
  ↓ founded_by
Sam Altman
  ↓ invested_in
Company X
```

Graph Retrieval 的核心优势是：

```text
semantic similarity
          +
explicit relationships
```

---

# 10. Recursive Retrieval

Recursive Retrieval 用于分层索引与引用节点。

```python
class RecursiveRetrievalPolicy:
    enabled: bool

    max_depth: int
    max_nodes: int

    follow_retriever_refs: bool
    follow_node_refs: bool
```

节点可能不是最终知识，而是指向：

```text
another node
another retriever
another index
another query engine
```

执行：

```text
Initial Retrieval
       ↓
Candidate
       ↓
Reference?
   ┌───────┐
   │ no    │ → Result
   │ yes   │
   └───┬───┘
       ↓
Resolve Reference
       ↓
Secondary Retrieval
       ↓
Recursive Result
```

必须维护：

```text
visited_nodes
visited_retrievers
depth
```

避免：

```text
recursive loop
duplicate retrieval
unbounded expansion
```

---

# 11. Routing Policy

Router 决定本轮调用哪些 Retriever。

```python
class RoutingPolicy:

    strategy: str

    allow_multi_route: bool

    retriever_weights: dict

    fallback_retriever: str | None
```

可采用：

```text
rule_based
intent_based
classifier
llm_router
hybrid_router
```

例如：

```text
"上一次我说过什么？"
        ↓
Memory Retriever

"这份报告在哪一页？"
        ↓
Keyword + Metadata Retriever

"A 公司与 B 公司是什么关系？"
        ↓
Graph Retriever

"找到与这个概念最相关的内容"
        ↓
Semantic Retriever
```

多路：

```text
Query
  ↓
Router
  ↓
┌──────────┬──────────┬──────────┐
│ semantic │ keyword  │ graph    │
└──────────┴──────────┴──────────┘
              ↓
         Candidate Pool
```

---

# 12. Hybrid Retrieval

Hybrid Retrieval 不是单独 Retriever，而是一种执行策略。

```python
class HybridRetrievalPolicy:

    retrievers: list[str]

    parallel: bool

    candidate_limit: int
```

典型组合：

```text
semantic
+
keyword
+
metadata
+
time
+
importance
+
graph
```

执行：

```text
                  Query
                    ↓
           Retrieval Router
                    ↓
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Semantic      Keyword       Graph
       │            │            │
       └────────────┼────────────┘
                    ↓
               Candidate Pool
```

Hybrid Retrieval 的目标是：

```text
先提高 Recall
再进行 Ranking
```

而不是直接把所有结果交给 LLM。

---

# 13. Candidate Fusion

多 Retriever 返回结果后必须统一合并。

```python
class FusionPolicy:

    normalize_scores: bool

    deduplicate: bool

    strategy: str

    retriever_weights: dict
```

支持：

```text
weighted_sum
max_score
rank_fusion
reciprocal_rank_fusion
```

流程：

```text
Retriever A Results
Retriever B Results
Retriever C Results
        ↓
Score Normalization
        ↓
Deduplication
        ↓
Candidate Merge
        ↓
Fusion Score
```

同一 Node 被多个 Retriever 命中：

```text
semantic score = 0.82
keyword score  = 0.91
graph score    = 0.60
```

应合并成一个 Candidate，而不是三个结果。

---

# 14. Scoring Policy

第一阶段 Ranking 使用结构化评分。

```python
class ScoringPolicy:

    semantic_weight: float
    keyword_weight: float

    recency_weight: float
    importance_weight: float

    graph_weight: float

    source_weight: float
```

基础形式：

```text
retrieval_score =
      w_semantic   × semantic_score
    + w_keyword    × keyword_score
    + w_recency    × recency_score
    + w_importance × importance_score
    + w_graph      × graph_score
    + w_source     × source_score
```

例如：

```yaml
scoring:

  semantic_weight: 0.40
  keyword_weight: 0.15

  recency_weight: 0.15
  importance_weight: 0.15

  graph_weight: 0.10
  source_weight: 0.05
```

权重不是固定系统常量。

应该允许根据：

```text
query_type
memory_type
knowledge_domain
task
```

动态切换。

---

# 15. Time Scoring

时间不是简单 Metadata Filter。

可以同时参与排序。

```python
class TimeScoringPolicy:

    enabled: bool

    decay_function: str
    half_life: int | None

    timestamp_field: str
```

例如：

```text
recency_score = exp(-λ × age)
```

形成：

```text
semantic relevance
       +
time relevance
```

对于：

```text
新闻
Agent Episodic Memory
项目状态
运行日志
```

时间权重可以较高。

对于：

```text
数学理论
基础知识
长期用户偏好
```

时间衰减可以很弱。

---

# 16. Importance Scoring

Importance 来自 Memory / Knowledge 对象本身。

```python
class ImportancePolicy:

    field: str

    normalize: bool

    weight: float
```

例如：

```yaml
candidate:
  semantic_score: 0.81

  recency_score: 0.72

  importance_score: 0.95
```

最终：

```text
high relevance
+
high importance

→ 排名提升
```

Importance 不应该由 Retrieval Engine 写入。

```text
Memory / Knowledge System
→ 产生 importance

Retrieval Engine
→ 消费 importance
```

---

# 17. Rerank Policy

Reranker 负责对召回结果进行第二阶段精排。

统一接口：

```python
class BaseReranker:

    def rerank(
        self,
        query: str,
        candidates: list[RetrievalCandidate],
        top_k: int
    ) -> list[RetrievalCandidate]:
        ...
```

支持：

```text
cross_encoder
embedding_reranker
llm_reranker
external_reranker
```

建议运行：

```text
Retrieve Top 50
      ↓
Fusion Top 30
      ↓
Rerank Top 20
      ↓
Final Top 8
```

避免：

```text
Vector Search Top 8
→ 直接进入 Context
```

Reranker 可以按 Query 类型动态开启。

---

# 18. Postprocessing

Rerank 后进入最后整理阶段。

```python
class PostprocessPolicy:

    score_threshold: float | None

    deduplicate: bool

    diversity: bool

    context_reorder: bool
```

执行：

```text
Ranked Candidates
        ↓
Threshold Filter
        ↓
Duplicate Removal
        ↓
Diversity Control
        ↓
Context Reorder
        ↓
RetrievalResult
```

Postprocessor 可以处理：

```text
重复 Node
同一文档过度占比
低分结果
相邻 Chunk 合并
结果顺序优化
```

但不负责：

```text
最终 Prompt 排版
Context token 分配
Context compression
```

这些属于 Context Compiler。

---

# 19. Retrieval Budget

Retrieval 必须受到 Budget 控制。

```python
class RetrievalBudget:

    max_retrievers: int

    recall_top_k: int
    fusion_top_k: int
    rerank_top_k: int
    final_top_k: int

    max_recursive_depth: int
    max_graph_depth: int

    max_candidates: int

    max_latency_ms: int | None
```

例如：

```yaml
budget:

  max_retrievers: 4

  recall_top_k: 30
  fusion_top_k: 30

  rerank_top_k: 20
  final_top_k: 8

  max_recursive_depth: 2
  max_graph_depth: 2

  max_candidates: 100
```

Budget 控制：

```text
检索广度
检索深度
计算成本
Rerank 成本
响应延迟
```

---

# 20. Retrieval Result

Retrieval Engine 最终返回统一对象。

```python
class RetrievalResult:

    query: RetrievalQuery

    candidates: list[RetrievalCandidate]

    route: list[str]

    metadata: dict
```

Candidate：

```python
class RetrievalCandidate:

    id: str

    content: str

    source_id: str
    source_type: str

    metadata: dict
    provenance: dict

    score: float

    score_breakdown: dict

    retrievers: list[str]
```

示例：

```yaml
candidate:

  id: node_001

  score: 0.91

  score_breakdown:
    semantic: 0.88
    keyword: 0.72
    recency: 0.94
    importance: 0.85
    graph: 0.60

  retrievers:
    - semantic
    - keyword

  provenance:
    document_id: doc_001
    section: risk_factors
```

---

# 21. Retrieval Pipeline

Retrieval Engine 的核心执行链：

```python
class RetrievalPipeline:

    def retrieve(self, request):

        query = query_processor.process(request)

        routes = router.route(query)

        candidates = execute_retrievers(
            routes,
            query
        )

        candidates = recursive_expand(candidates)

        candidates = fuse(candidates)

        candidates = score(candidates)

        candidates = rerank(candidates)

        candidates = postprocess(candidates)

        return RetrievalResult(
            query=query,
            candidates=candidates,
            route=routes,
        )
```

完整流程：

```text
Raw Query
    ↓
Query Processor
    ↓
RetrievalQuery
    ↓
Router
    ↓
┌───────────────────────────────────┐
│                                   │
▼            ▼           ▼          ▼
Semantic   Keyword     Memory      Graph
│            │           │          │
└────────────┴───────────┴──────────┘
                 ↓
          Candidate Pool
                 ↓
       Recursive Expansion
                 ↓
         Score Normalize
                 ↓
          Deduplicate
                 ↓
              Fusion
                 ↓
             Scoring
                 ↓
             Rerank
                 ↓
          Postprocess
                 ↓
        RetrievalResult
```

---

# 22. Fallback

Retriever 可能没有结果。

因此定义：

```python
class FallbackPolicy:

    retry_without_filters: bool

    broaden_query: bool

    fallback_retrievers: list[str]

    max_attempts: int
```

流程：

```text
Retrieval
    ↓
Enough Candidates?
 ┌──────┴──────┐
 yes           no
 │             │
Result      Relax Filter
               ↓
           Query Rewrite
               ↓
        Fallback Retriever
```

Fallback 不应该无限循环。

---

# 23. Retrieval Cache

高频检索可以增加轻量缓存。

```python
class RetrievalCache:

    query_hash: str

    retrieval_spec_version: str

    result: RetrievalResult

    expires_at: datetime
```

Cache Key：

```text
query
+
filters
+
namespace
+
retrieval policy version
```

避免：

```text
同一 Query
→ 每次重新 embedding
→ 每次重新 vector search
→ 每次重新 rerank
```

但 Memory / Knowledge 更新后应能够触发失效。

---

# 24. Observability

Retrieval Engine 必须能够解释：

```text
为什么找到了这个结果
为什么这个结果排在这里
```

建议记录：

```python
class RetrievalTrace:

    query_id: str

    selected_retrievers: list[str]

    route_reason: dict

    candidate_counts: dict

    score_trace: dict

    rerank_trace: dict

    latency: dict
```

例如：

```text
query
 ↓
semantic: 30 candidates
keyword: 18 candidates
graph: 7 candidates
 ↓
dedup: 41 candidates
 ↓
fusion: 30 candidates
 ↓
rerank: 10 candidates
 ↓
final: 6 candidates
```

Observability 主要服务：

```text
debug
evaluation
weight tuning
retrieval quality analysis
```

---

# 25. Evaluation

Retrieval Engine 应独立做 Retrieval Evaluation。

```python
class RetrievalEvaluation:

    recall_at_k
    precision_at_k
    hit_rate

    mrr
    ndcg

    latency
```

重点评估两个阶段：

```text
Recall Stage
→ 正确知识有没有被找出来

Ranking Stage
→ 正确知识有没有排在前面
```

即：

```text
Retriever Evaluation
        +
Reranker Evaluation
```

不要只测试最终 Agent Answer。

---

# 26. Retrieval Lifecycle

```text
QUERY
收到检索请求
    ↓
PROCESS
Query 标准化
    ↓
ROUTE
选择 Retriever
    ↓
RECALL
多路召回
    ↓
EXPAND
Recursive / Graph 扩展
    ↓
FUSE
候选合并
    ↓
SCORE
统一评分
    ↓
RERANK
精排
    ↓
POSTPROCESS
整理结果
    ↓
RETURN
输出 RetrievalResult
```

---

# 27. 完整 RetrievalSpec

最终配置不要散落在 Retriever 实现中。

```python
class RetrievalSpec:

    metadata: RetrievalMetadata

    query_policy: QueryPolicy

    routing_policy: RoutingPolicy

    retrieval_policy: RetrievalPolicy

    recursive_policy: RecursiveRetrievalPolicy

    fusion_policy: FusionPolicy

    scoring_policy: ScoringPolicy

    rerank_policy: RerankPolicy

    postprocess_policy: PostprocessPolicy

    budget: RetrievalBudget

    fallback_policy: FallbackPolicy
```

完整配置：

```yaml
retrieval:

  metadata:
    id: default_retrieval
    version: "1.0"
    namespace: agent

    tags:
      - hybrid
      - memory
      - knowledge

  query_policy:
    normalization: true
    rewrite: false
    decomposition: false

    keyword_extraction: true
    entity_extraction: true

    embedding: true

  routing_policy:
    strategy: hybrid

    allow_multi_route: true

    fallback_retriever: semantic

  retrieval_policy:
    top_k: 30
    threshold: null

  recursive_policy:
    enabled: true

    max_depth: 2
    max_nodes: 50

    follow_retriever_refs: true
    follow_node_refs: true

  fusion_policy:
    normalize_scores: true
    deduplicate: true

    strategy: reciprocal_rank_fusion

  scoring_policy:
    semantic_weight: 0.40
    keyword_weight: 0.15

    recency_weight: 0.15
    importance_weight: 0.15

    graph_weight: 0.10
    source_weight: 0.05

  rerank_policy:
    reranker: cross_encoder

    rerank_top_k: 20
    output_top_k: 8

  postprocess_policy:
    score_threshold: 0.30

    deduplicate: true
    diversity: true

    context_reorder: true

  budget:
    max_retrievers: 4

    recall_top_k: 30
    fusion_top_k: 30

    rerank_top_k: 20
    final_top_k: 8

    max_recursive_depth: 2
    max_graph_depth: 2

    max_candidates: 100

  fallback_policy:
    retry_without_filters: false
    broaden_query: true

    fallback_retrievers:
      - semantic
      - keyword

    max_attempts: 2
```

---

# 28. Runtime 装配流程

```text
retrieval.yaml
      ↓
RetrievalSpec Loader
      ↓
Schema Validation
      ↓
RetrieverRegistry
      ↓
┌────────────────────────────────┐
│ Retrieval Runtime              │
│                                │
│ Query Processor                │
│ Router                         │
│ Retriever Registry             │
│ Recursive Retrieval            │
│ Fusion                         │
│ Scoring                        │
│ Reranker                       │
│ Postprocessor                  │
└────────────────────────────────┘
```

执行：

```text
RetrievalRuntime.retrieve(query)
        │
        ├── QueryProcessor.process()
        │
        ├── Router.route()
        │
        ↓
Retriever Execution
        │
        ├── SemanticRetriever
        ├── KeywordRetriever
        ├── MetadataRetriever
        ├── MemoryRetriever
        └── GraphRetriever
        ↓
Candidate Pool
        ↓
Recursive Retrieval
        ↓
Fusion
        ↓
Scoring
        ↓
Reranker
        ↓
Postprocessor
        ↓
RetrievalResult
```

---

# 29. 与 Knowledge System 的接口

Knowledge System 建立知识与索引。

Retrieval Engine 消费索引。

```text
Knowledge System
      │
      ├── DocumentStore
      ├── VectorStore
      ├── KeywordIndex
      ├── MetadataIndex
      └── GraphStore
              ↓
       Retrieval Engine
```

接口：

```python
class KnowledgeRetrievalAdapter:

    def vector_search(...): ...

    def keyword_search(...): ...

    def metadata_search(...): ...

    def graph_search(...): ...

    def get_node(...): ...
```

边界：

```text
Knowledge System
→ 建知识 / 建索引

Retrieval Engine
→ 查询知识 / 排序知识
```

---

# 30. 与 Memory System 的接口

```text
Memory System
      │
      ├── working
      ├── episodic
      ├── semantic
      ├── procedural
      └── profile
              ↓
        MemoryRetriever
              ↓
       Retrieval Engine
```

Memory System 提供：

```text
memory content
embedding
timestamp
importance
metadata
memory_type
```

Retrieval Engine 负责：

```text
semantic match
time weighting
importance weighting
fusion
rerank
```

边界：

```text
Memory System
→ 管理记忆生命周期

Retrieval Engine
→ 找出当前 Query 最需要的记忆
```

---

# 31. 与 Context Compiler 的接口

Retrieval Engine 输出候选知识。

```text
RetrievalResult
      ↓
Context Compiler
      │
      ├── selection
      ├── token allocation
      ├── compression
      ├── ordering
      └── formatting
      ↓
Model Context
```

Retrieval Engine 不负责：

```text
最终 context token budget
system prompt
conversation history
tool result placement
prompt formatting
```

Retrieval 的：

```text
final_top_k
```

只是候选上限。

Context Compiler 仍可以：

```text
再次筛选
截断
压缩
不采用
```

---

# 32. 与 Session / State 的接口

Session 可以为 Retrieval 提供当前任务上下文。

```text
Session State
      ↓
Retrieval Context
      │
      ├── task_id
      ├── current_goal
      ├── active_entities
      ├── active_namespace
      └── temporal_context
      ↓
Retrieval Engine
```

但 Retrieval Engine 不修改 Session 主状态。

只返回：

```text
RetrievalResult
RetrievalTrace
```

---

# 33. 模块边界

最终建议形成：

```text
retrieval/
├── spec/
│   ├── query
│   ├── candidate
│   ├── result
│   ├── query_policy
│   ├── routing_policy
│   ├── retrieval_policy
│   ├── fusion_policy
│   ├── scoring_policy
│   ├── rerank_policy
│   └── budget
│
├── query/
│   ├── processor
│   ├── normalizer
│   ├── rewriter
│   ├── decomposer
│   ├── keyword_extractor
│   └── entity_extractor
│
├── retrievers/
│   ├── base
│   ├── semantic
│   ├── keyword
│   ├── metadata
│   ├── memory
│   └── graph
│
├── graph/
│   ├── entity
│   ├── vector_context
│   ├── relation
│   ├── path
│   ├── synonym
│   └── structured_query
│
├── routing/
│   ├── router
│   └── policy
│
├── recursive/
│   ├── resolver
│   ├── traversal
│   └── dedup
│
├── fusion/
│   ├── normalizer
│   ├── deduplicator
│   ├── weighted
│   └── rrf
│
├── scoring/
│   ├── semantic
│   ├── keyword
│   ├── time
│   ├── importance
│   ├── graph
│   └── composite
│
├── reranker/
│   ├── base
│   ├── cross_encoder
│   ├── embedding
│   └── llm
│
├── postprocess/
│   ├── filter
│   ├── diversity
│   ├── merge
│   └── reorder
│
├── cache/
│
├── evaluation/
│
├── observability/
│
├── registry/
│
└── runtime/
```

最终职责：

```text
RetrievalSpec
→ 定义 Retrieval Engine

QueryProcessor
→ 将用户 Query 编译成 RetrievalQuery

RetrieverRegistry
→ 注册检索能力

Router
→ 决定调用哪些 Retriever

SemanticRetriever
→ 向量语义召回

KeywordRetriever
→ 关键词 / BM25 召回

MetadataRetriever
→ 结构字段检索

MemoryRetriever
→ 检索 Agent Memory

GraphRetriever
→ 检索实体关系知识

RecursiveRetriever
→ 沿索引 / Node / Retriever 引用继续检索

FusionEngine
→ 合并多路候选

ScoringEngine
→ 计算 semantic / time / importance 等综合分

Reranker
→ 第二阶段精排

Postprocessor
→ 整理最终候选

RetrievalBudget
→ 控制检索成本与深度

RetrievalTrace
→ 记录完整检索路径

RetrievalRuntime
→ 装配并执行整个 Retrieval Pipeline
```

最终系统关系：

```text
                  ┌─────────────────┐
                  │   Memory System │
                  └────────┬────────┘
                           │
                           │
┌─────────────────┐        │
│ Knowledge System│────────┤
└─────────────────┘        │
                           ▼
                  Retrieval Engine
                           │
                           │ RetrievalResult
                           ▼
                  Context Compiler
                           │
                           ▼
                     Agent Runtime
```

进一步压缩：

```text
Knowledge System
→ 建知识

Memory System
→ 管记忆

Retrieval Engine
→ 找知识 / 找记忆

Context Compiler
→ 决定哪些信息进入模型

Agent Runtime
→ 使用最终 Context 执行任务
```

最终 Retrieval 主链：

```text
Query
  ↓
Process
  ↓
Route
  ↓
Recall
  ↓
Recursive / Graph Expand
  ↓
Fuse
  ↓
Score
  ↓
Rerank
  ↓
Postprocess
  ↓
RetrievalResult
  ↓
Context Compiler
```