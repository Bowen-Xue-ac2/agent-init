# Memory System Design

```text
MemorySystem
│
├── core_memory             # 高频、稳定、每轮可见的长期状态
│   ├── persona
│   ├── human
│   ├── project
│   └── memory_index
│
├── long_term_memory        # 大容量持久化记忆
│   ├── profile
│   ├── semantic
│   ├── episodic
│   └── procedural
│
├── write_policy            # 什么信息进入长期记忆
│   ├── extraction
│   ├── classification
│   ├── importance
│   └── conflict_resolution
│
├── retrieval_policy        # 如何找回记忆
│   ├── semantic
│   ├── keyword
│   ├── temporal
│   ├── importance
│   ├── entity
│   └── rerank
│
├── lifecycle_policy        # 如何长期维护记忆
│   ├── update
│   ├── merge
│   ├── supersede
│   ├── decay
│   ├── expiration
│   └── deletion
│
├── storage_policy          # 记忆如何持久化和建立索引
│   ├── metadata_store
│   ├── vector_store
│   ├── keyword_index
│   ├── entity_index
│   └── history_store
│
└── metadata                # Memory System 管理信息
    ├── id
    ├── version
    └── schema_version
```

## 1. Core Memory

保存需要持续进入 Agent Context 的高价值长期状态。

```python
class CoreMemory:
    persona: dict
    human: dict
    project: dict
    memory_index: dict
```

功能：

```text
persona
→ Agent 长期形成的工作方式、偏好、行为状态

human
→ 用户长期稳定信息

project
→ 当前项目长期有效的关键事实与约束

memory_index
→ 指向 Long-term Memory / Knowledge / Skill 的索引
```

Core Memory 特征：

```text
small
stable
high-value
always-visible
```

运行关系：

```text
Core Memory
      ↓
Context Compiler
      ↓
System Context
```

因此：

```text
Core Memory
≠ 普通向量数据库中的一条 Memory

Core Memory
= Agent Runtime 的持久上下文状态
```

---

## 2. Long-term Memory

负责大容量、按需检索的长期记忆。

```python
class LongTermMemory:
    profile: list[MemoryRecord]
    semantic: list[MemoryRecord]
    episodic: list[MemoryRecord]
    procedural: list[MemoryRecord]
```

四类记忆：

```text
profile
→ 用户 / Agent / 项目的稳定属性

semantic
→ 已形成的事实、知识、结论

episodic
→ 过去发生的事件、决定、经历

procedural
→ 从实践中形成的操作方法与经验
```

边界：

```text
working memory
→ Session / State

raw conversation
→ Recall Store

external documents
→ Knowledge System

long-term learned state
→ Memory System
```

---

## 3. Memory Record

所有长期记忆统一使用标准结构。

```python
class MemoryRecord:
    id: str

    type: MemoryType
    scope: MemoryScope

    content: str

    importance: float
    confidence: float

    entities: list[str]
    keywords: list[str]

    status: str

    created_at: datetime
    updated_at: datetime
    last_accessed_at: datetime | None
    access_count: int

    expires_at: datetime | None

    source_type: str
    source_id: str | None
    session_id: str | None

    version: int
    supersedes: str | None
```

Scope：

```python
class MemoryScope:
    user_id: str | None
    agent_id: str | None
    project_id: str | None
    session_id: str | None
```

作用：

```text
user scope
→ 某个用户长期记忆

agent scope
→ Agent 自身经验

project scope
→ 项目专属记忆

session scope
→ 特定会话产生的记忆来源
```

---

## 4. Memory Write Policy

决定什么信息应该进入长期记忆。

```python
class MemoryWritePolicy:
    extraction: ExtractionPolicy
    classification: ClassificationPolicy
    importance_threshold: float
    conflict_policy: ConflictPolicy
```

执行链：

```text
Conversation / Event
        ↓
Memory Write Gate
        ↓
Memory Extractor
        ↓
Memory Classifier
        ↓
Related Memory Search
        ↓
Conflict Resolver
        ↓
Memory Operation
```

Memory Extractor：

```text
raw conversation
      ↓
facts
preferences
decisions
events
corrections
procedures
```

不要直接执行：

```text
conversation
→ vector store
```

而应该：

```text
conversation
→ extraction
→ structured memory candidate
→ persistence
```

---

## 5. Memory Classifier

对 Memory Candidate 进行类型、Scope 与价值判断。

```python
class MemoryCandidate:
    content: str

    type: MemoryType
    scope: MemoryScope

    importance: float
    confidence: float

    entities: list[str]

    source: MemorySource
```

分类：

```text
“用户偏好简洁输出”
→ profile

“项目使用 PostgreSQL”
→ semantic

“上次部署因为 migration 失败”
→ episodic

“部署前先执行 migration check”
→ procedural
```

判定结果：

```text
Candidate
   │
   ├── discard
   ├── long_term_memory
   └── core_memory_candidate
```

---

## 6. Conflict Resolver

Memory 写入不能使用简单 append。

```python
class MemoryConflictResolver:

    def resolve(
        self,
        candidate,
        existing_memories,
    ) -> MemoryOperation:
        ...
```

操作类型：

```python
MemoryOperation =
    ADD
    UPDATE
    MERGE
    SUPERSEDE
    DELETE
    NOOP
```

决策：

```text
没有相关记忆
→ ADD

完全重复
→ NOOP

同一事实，信息更完整
→ UPDATE

多条信息可以合并
→ MERGE

新事实替代旧事实
→ SUPERSEDE

明确失效
→ DELETE / EXPIRE
```

推荐：

```text
Memory A
    ↓ superseded_by
Memory B
```

而不是：

```text
Memory A
→ overwrite
```

这样可以保留历史状态。

---

## 7. Memory Retrieval Policy

负责从 Long-term Memory 找出当前任务相关内容。

```python
class RetrievalPolicy:
    semantic_weight: float
    keyword_weight: float
    temporal_weight: float
    importance_weight: float
    entity_weight: float
    confidence_weight: float

    candidate_k: int
    final_k: int
```

第一阶段：

```text
Query
 │
 ├── Semantic Search
 ├── Keyword / BM25
 ├── Entity Search
 ├── Temporal Filter
 └── Metadata Filter
        ↓
Candidate Set
```

第二阶段：

```text
score =
    semantic
  + keyword
  + recency
  + importance
  + entity_match
  + confidence
```

形式：

```python
score = (
    α * semantic_score
    + β * keyword_score
    + γ * temporal_score
    + δ * importance
    + ε * entity_score
    + ζ * confidence
)
```

第三阶段：

```text
Candidate Set
      ↓
Deduplication
      ↓
Conflict Collapse
      ↓
Reranker
      ↓
Top-K Memory
```

Memory Retrieval 只负责找出候选。

是否进入模型由 Context Compiler 决定：

```text
Memory Retrieval
        ↓
Relevant Memories
        ↓
Context Compiler
        ↓
Token Budget / Priority
        ↓
LLM Context
```

---

## 8. Entity Index

Entity 用于增强 Memory 检索。

```python
class MemoryEntity:
    id: str
    name: str
    type: str
    linked_memory_ids: list[str]
```

结构：

```text
OpenAI
  │
  ├── memory_21
  ├── memory_38
  └── memory_71
```

用途：

```text
Query
→ Entity Extraction
→ Entity Index
→ linked memories
```

因此：

```text
Entity
≠ Memory

Entity Index
= Memory Retrieval Index
```

后续可升级为：

```text
Entity Index
      ↓
Relation Index
      ↓
Memory Graph
```

---

## 9. Storage Policy

Memory System 是逻辑层，不绑定某一种数据库。

```python
class MemoryStorage:
    metadata_store: MetadataStore
    vector_store: VectorStore
    keyword_index: KeywordIndex
    entity_index: EntityIndex
    history_store: HistoryStore
```

完整结构：

```text
MemoryStorage
│
├── metadata_store
│   └── MemoryRecord 主数据
│
├── vector_store
│   └── embedding + semantic search
│
├── keyword_index
│   └── BM25 / exact search
│
├── entity_index
│   └── entity → memory_ids
│
└── history_store
    └── Memory mutation history
```

Embedding 单独作为 Adapter：

```python
class Embedder:
    def embed(self, text: str) -> list[float]:
        ...
```

允许：

```text
Memory System
    ↓
Embedder Adapter
    ↓
OpenAI / Voyage / Local Embedding
```

不改变 Memory 数据模型。

---

## 10. Memory History

所有长期 Memory 修改都保留变更记录。

```python
class MemoryVersion:
    id: str
    memory_id: str

    event: str

    old_content: str | None
    new_content: str | None

    created_at: datetime

    actor_id: str | None
```

History：

```text
ADD
UPDATE
MERGE
SUPERSEDE
DELETE
PROMOTE
DEMOTE
```

结构：

```text
memory:v1
   ↓ UPDATE
memory:v2
   ↓ SUPERSEDE
memory:v3
```

作用：

```text
audit
rollback
temporal reasoning
conflict analysis
memory evolution
```

---

## 11. Memory Lifecycle Policy

负责 Memory 的长期维护。

```python
class MemoryLifecyclePolicy:
    decay_enabled: bool
    expiration_enabled: bool

    consolidation_interval: str
    reflection_interval: str

    promotion_threshold: float
    deletion_threshold: float
```

生命周期：

```text
Experience
    ↓
Memory Candidate
    ↓
Long-term Memory
    ↓
Repeated / Important / Stable
    ↓
Core Memory
```

反向：

```text
Core Memory
    ↓ low utility
Long-term Memory
    ↓ obsolete
Superseded / Expired
    ↓
Archive / Delete
```

维护模块：

```text
MemoryMaintenance
│
├── reflection
│   # 从经历中提取长期模式
│
├── consolidation
│   # 多条相近记忆合并
│
├── conflict_scan
│   # 检查矛盾记忆
│
├── promotion
│   # Long-term → Core
│
├── demotion
│   # Core → Long-term
│
├── decay
│   # 长期降低低价值记忆权重
│
├── expiration
│   # 时间型记忆失效
│
└── defragmentation
    # 重组 Memory 结构和索引
```

---

## 12. Core Memory Promotion

Core Memory 不应该由普通写入流程随意修改。

独立建立 Promotion Policy：

```python
class PromotionPolicy:
    min_importance: float
    min_confidence: float
    min_repetition: int
    require_stability: bool
```

流程：

```text
Long-term Memory
      ↓
Repeated Retrieval
      +
High Importance
      +
High Confidence
      +
Stable Across Time
      ↓
Core Memory Candidate
      ↓
Promotion
```

例如：

```text
“用户本轮要求简短回答”
→ episodic / temporary

“用户长期持续要求工程化简洁输出”
→ profile

持续稳定后
→ Core Memory.human
```

---

## 13. Recall System

Recall 与 Memory 必须分开。

```text
Recall
= 找回过去发生过什么

Memory
= 保存从过去提炼出的长期状态
```

结构：

```text
Conversation
     │
     ├──→ Recall Store
     │
     └──→ Memory Extraction
                 ↓
          Memory System
```

Recall Record：

```python
class RecallRecord:
    session_id: str
    message_id: str

    role: str
    content: str

    created_at: datetime
```

因此：

```text
“用户三个月前具体说过什么？”
→ Recall

“用户长期偏好什么？”
→ Memory
```

---

## 14. 与 Session / State 的接口

Memory 不承担 Runtime State。

```text
Session / State
├── current_task
├── current_step
├── variables
├── pending_actions
├── checkpoints
└── execution_state
```

Memory：

```text
Memory
├── profile
├── semantic
├── episodic
└── procedural
```

连接：

```text
Session Runtime
      ↓ event
Memory Write Pipeline
```

以及：

```text
Memory Retrieval
      ↓
Context Compiler
      ↓
Session Runtime
```

---

## 15. 与 Knowledge System 的接口

```text
Knowledge System
= 外部资料

Memory System
= Agent 经历形成的长期状态
```

示例：

```text
Apple 10-K
→ Knowledge

“此前分析认为 Apple 服务业务利润贡献正在提高”
→ Memory
```

Memory 保存引用：

```python
class MemorySource:
    source_type: str
    source_id: str
    locator: str | None
```

例如：

```text
source:
  source_type: knowledge
  source_id: apple_10k_2026
  locator: section_4
```

避免直接复制大型 Knowledge 内容进入 Memory。

---

# 16. Memory Service

统一对 Runtime 暴露 API。

```python
class MemoryService:

    async def add(
        self,
        event,
        scope,
    ) -> MemoryWriteResult:
        ...

    async def search(
        self,
        query,
        scope,
        types=None,
        top_k=10,
    ) -> list[MemorySearchResult]:
        ...

    async def get(
        self,
        memory_id,
    ) -> MemoryRecord:
        ...

    async def update(
        self,
        memory_id,
        patch,
    ) -> MemoryRecord:
        ...

    async def delete(
        self,
        memory_id,
    ) -> None:
        ...

    async def history(
        self,
        memory_id,
    ) -> list[MemoryVersion]:
        ...

    async def promote(
        self,
        memory_id,
        block,
    ) -> None:
        ...

    async def reflect(
        self,
        scope,
    ) -> ReflectionResult:
        ...

    async def consolidate(
        self,
        scope,
    ) -> ConsolidationResult:
        ...
```

---

# 17. 完整 MemorySpec

最终统一配置：

```python
class MemorySpec:
    metadata: MemoryMetadata

    core_memory: CoreMemoryConfig
    long_term_memory: LongTermMemoryConfig

    write_policy: MemoryWritePolicy
    retrieval_policy: RetrievalPolicy
    lifecycle_policy: MemoryLifecyclePolicy

    storage_policy: StoragePolicy
```

配置示例：

```yaml
memory:

  metadata:
    id: default_memory
    version: "1.0"
    schema_version: "1"

  core_memory:
    enabled: true

    blocks:
      - persona
      - human
      - project
      - memory_index

  long_term_memory:
    types:
      - profile
      - semantic
      - episodic
      - procedural

  write_policy:
    extraction: llm
    importance_threshold: 0.5
    deduplicate: true
    conflict_resolution: true

  retrieval_policy:
    semantic_weight: 0.40
    keyword_weight: 0.15
    temporal_weight: 0.15
    importance_weight: 0.15
    entity_weight: 0.10
    confidence_weight: 0.05

    candidate_k: 50
    final_k: 10

  lifecycle_policy:
    reflection: true
    consolidation: true
    decay: true
    expiration: true

  storage_policy:
    metadata_store: postgres
    vector_store: pgvector
    keyword_index: postgres_fts
    history_store: postgres
```

---

# 18. Runtime 装配流程

```text
memory.yaml
      ↓
MemorySpec Loader
      ↓
Schema Validation
      ↓
┌───────────────────────────┐
│ Memory Runtime            │
│                           │
│ Core Memory               │
│ Long-term Memory          │
│ Retrieval Engine          │
│ Write Pipeline            │
│ Lifecycle Manager         │
└───────────────────────────┘
```

写入一轮：

```text
Runtime Event
      ↓
MemoryService.add()
      ↓
Extractor
      ↓
Classifier
      ↓
RelatedMemoryRetriever
      ↓
ConflictResolver
      ↓
Writer
      │
      ├── Metadata Store
      ├── Vector Store
      ├── Keyword Index
      ├── Entity Index
      └── History Store
```

检索一轮：

```text
ContextCompiler requests memory
        ↓
MemoryService.search()
        ↓
Scope Filter
        ↓
Semantic / Keyword / Entity
        ↓
Temporal + Importance Scoring
        ↓
Reranker
        ↓
Relevant Memory
        ↓
Context Compiler
```

---

# 19. Memory 生命周期

```text
OBSERVE
Runtime 产生 Conversation / Event
        ↓
EXTRACT
提取 Memory Candidate
        ↓
CLASSIFY
判断类型 / Scope / Importance
        ↓
RESOLVE
处理重复 / 冲突 / 更新
        ↓
STORE
持久化 + 建立索引
        ↓
RETRIEVE
任务运行时动态召回
        ↓
REFLECT
长期分析经验
        ↓
CONSOLIDATE
合并、压缩、结构化
        ↓
PROMOTE / DECAY
升级 Core 或降低权重
        ↓
EXPIRE / DELETE
淘汰失效 Memory
```

---

# 20. 与 Agent Identity 的接口

```text
Agent Identity
    │
    └── Persona

Memory System
    │
    ├── Profile Memory
    ├── Semantic Memory
    ├── Episodic Memory
    └── Procedural Memory
```

关系：

```text
Runtime Experience
        ↓
Memory Extraction
        ↓
Profile / Semantic / Episodic / Procedural
        ↓
Identity-related Pattern Detected
        ↓
Persona Candidate
        ↓
Persona Store
```

区分：

```text
“用户喜欢工程化输出”
→ profile memory

“Agent 长期形成 evidence-first 工作方式”
→ persona

“Agent 必须验证来源”
→ identity.behavioral_rules
```

---

# 21. 模块边界

```text
memory/
├── spec/
│   ├── models
│   ├── types
│   ├── scope
│   └── policies
│
├── core/
│   ├── blocks
│   └── promotion
│
├── write/
│   ├── extractor
│   ├── classifier
│   ├── resolver
│   └── writer
│
├── retrieval/
│   ├── semantic
│   ├── keyword
│   ├── temporal
│   ├── entity
│   ├── scorer
│   └── reranker
│
├── lifecycle/
│   ├── reflection
│   ├── consolidation
│   ├── decay
│   ├── expiration
│   └── defragmentation
│
├── storage/
│   ├── metadata_store
│   ├── vector_store
│   ├── keyword_index
│   ├── entity_index
│   └── history_store
│
├── recall/
│   └── recall_store
│
├── service/
│   └── memory_service
│
└── factory/
    └── memory_factory
```

最终职责：

```text
MemorySpec
→ 定义 Memory System

MemoryService
→ Memory 对外统一接口

MemoryExtractor
→ 从 Experience 提取记忆

MemoryClassifier
→ 判断类型 / Scope / 价值

ConflictResolver
→ ADD / UPDATE / MERGE / SUPERSEDE

MemoryWriter
→ 执行持久化

RetrievalEngine
→ 搜索和排序 Memory

CoreMemory
→ 保存持续进入 Context 的高价值状态

LifecycleManager
→ Reflection / Consolidation / Decay / Expiration

RecallStore
→ 保存和检索原始对话历史

Storage Adapters
→ 管理实际数据库和索引

ContextCompiler
→ 决定哪些 Memory 最终进入模型
```

最终运行关系：

```text
                 Agent Runtime

Session / State ────────┐
                        │
Memory System ──────────┼──→ Context Compiler ───→ LLM
                        │
Knowledge System ───────┤
                        │
Tool Results ───────────┘


Memory System

Core Memory
    ↑
    │ Promotion / Demotion
    ↓
Long-term Memory
    ├── Profile
    ├── Semantic
    ├── Episodic
    └── Procedural
          ↑
          │
    Write Pipeline
          ↑
Conversation / Runtime Events

Recall Store
    └── Raw Conversation History

Lifecycle Manager
    ├── Reflection
    ├── Consolidation
    ├── Conflict Resolution
    ├── Decay
    ├── Expiration
    └── Defragmentation
```