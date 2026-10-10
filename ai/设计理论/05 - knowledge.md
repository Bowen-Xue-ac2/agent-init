```text
KnowledgeSystem
│
├── source_registry        # 知识从哪里来
│   ├── source_id
│   ├── source_type
│   ├── uri
│   └── source_metadata
│
├── document_model         # 原始知识标准对象
│   ├── document
│   ├── version
│   ├── checksum
│   └── metadata
│
├── parsing_policy         # 如何把 Document 转成 Node / Chunk
│   ├── parser
│   ├── chunk_strategy
│   ├── chunk_size
│   └── overlap
│
├── knowledge_node         # Knowledge System 最小知识单元
│   ├── content
│   ├── metadata
│   ├── relationships
│   └── provenance
│
├── enrichment_policy      # 如何增强知识
│   ├── metadata_extraction
│   ├── entity_extraction
│   ├── relation_extraction
│   └── summary_extraction
│
├── embedding_policy       # 哪些知识进入向量空间
│   ├── model
│   ├── target_fields
│   └── version
│
├── graph_policy           # 如何建立知识图谱
│   ├── entities
│   ├── relationships
│   ├── claims
│   └── communities
│
├── storage_policy         # 知识保存位置
│   ├── document_store
│   ├── vector_store
│   ├── graph_store
│   └── index_store
│
└── metadata               # Knowledge System 管理信息
    ├── id
    ├── version
    ├── namespace
    └── tags
```

# 1. Source Registry

定义 Knowledge System 可以接收哪些外部知识源。

```python
class KnowledgeSource:
    source_id: str
    source_type: str
    uri: str
    metadata: dict
```

功能：

```text
source_id
→ 稳定知识源标识

source_type
→ file / url / api / database / repository / manual

uri
→ 原始资源位置

metadata
→ 来源级别描述信息
```

示例：

```yaml
source:
  source_id: sec_filings

  source_type: api

  uri: sec://company-filings

  metadata:
    domain: finance
    authority: primary
    language: en
```

完整入口：

```text
External Source
      ↓
SourceRegistry
      ↓
ReaderResolver
      ↓
Document
```

---

# 2. Document Model

所有原始知识进入系统后，先统一转换为 `Document`。

```python
class KnowledgeDocument:
    id: str
    source_id: str
    content: str

    content_type: str

    metadata: dict

    checksum: str
    version: str

    created_at: datetime
    updated_at: datetime
```

功能：

```text
Document
= Knowledge System 的原始知识对象

source_id
→ 回溯 Source Registry

checksum
→ 判断原始内容是否发生变化

version
→ 支持知识版本管理

metadata
→ 保存文档级属性
```

示例：

```yaml
document:
  id: doc_001
  source_id: sec_filings

  content_type: markdown

  metadata:
    company: example_corp
    report_type: 10-K
    year: 2026

  checksum: sha256:xxxx
  version: "1"
```

Document 不直接承担检索最小单位职责。

```text
Source
   ↓
Document
   ↓
Parsing
   ↓
KnowledgeNode
```

---

# 3. Parsing Policy

负责：

```text
Document
    ↓
结构识别
    ↓
Chunk / Node
```

```python
class ParsingPolicy:
    parser: str

    chunk_strategy: str

    chunk_size: int
    chunk_overlap: int

    preserve_structure: bool
```

支持：

```text
token_chunking
sentence_chunking
semantic_chunking
hierarchical_chunking
structure_aware_chunking
```

示例：

```yaml
parsing_policy:

  parser: markdown

  chunk_strategy: hierarchical

  chunk_size: 800
  chunk_overlap: 100

  preserve_structure: true
```

运行：

```text
Document
   ↓
ParserResolver
   ↓
StructureParser
   ↓
ChunkPolicy
   ↓
KnowledgeNode[]
```

其中 `chunk_size` 不应该成为固定系统常量，而是 Parsing Policy。

---

# 4. Knowledge Node

Node 是整个 Knowledge System 最重要的标准对象。

```python
class KnowledgeNode:
    id: str
    document_id: str

    content: str
    content_type: str

    metadata: dict

    relationships: list[NodeRelation]

    provenance: Provenance

    checksum: str
    version: str

    embedding: list[float] | None
```

Node 应承担：

```text
content
→ 实际知识内容

metadata
→ 可过滤、可解释的知识属性

relationships
→ 与其他知识单元之间的结构关系

provenance
→ 回到原始来源

embedding
→ 向量表示
```

基础关系：

```python
class NodeRelation:
    relation_type: str
    target_id: str
```

建议至少支持：

```text
SOURCE
PREVIOUS
NEXT
PARENT
CHILD
```

形成：

```text
Document
   │
   ├── Node A
   │      ├── PREVIOUS
   │      └── NEXT
   │
   └── Node B
          ├── PARENT
          └── CHILD
```

---

# 5. Metadata System

Metadata 不只是附加字段，而是 Knowledge System 的重要控制面。

```python
class KnowledgeMetadata:
    source: dict
    structural: dict
    temporal: dict
    domain: dict
    semantic: dict
```

建议分类：

```text
source_metadata
→ source / author / url / file

structural_metadata
→ section / page / heading / position

temporal_metadata
→ publish_time / effective_time / updated_at

domain_metadata
→ domain / category / topic

semantic_metadata
→ keywords / summary / concepts
```

示例：

```yaml
metadata:

  source:
    source_id: sec_001
    authority: primary

  structural:
    section: risk_factors
    page: 37

  temporal:
    published_at: 2026-03-01

  domain:
    domain: finance
    topic: credit_risk

  semantic:
    keywords:
      - liquidity
      - debt
```

同时定义：

```python
class MetadataPolicy:
    embed_fields: list[str]
    llm_fields: list[str]
    filter_fields: list[str]
```

即：

```text
metadata
   ├── 哪些进入 embedding
   ├── 哪些进入 LLM Context
   └── 哪些允许 Retrieval Filter
```

---

# 6. Provenance

所有知识对象必须能够追溯来源。

```python
class Provenance:
    source_id: str
    document_id: str

    locator: dict

    parent_node_ids: list[str]

    extraction_method: str | None
```

示例：

```yaml
provenance:

  source_id: annual_report_2026

  document_id: doc_001

  locator:
    page: 42
    section: liquidity_risk

  parent_node_ids: []

  extraction_method: document_chunk
```

派生知识：

```text
Source
  ↓
Document
  ↓
Node
  ↓
Entity
  ↓
Relationship
  ↓
Community Report
```

必须能够反向：

```text
Community Report
      ↓
Entity / Relationship
      ↓
Node
      ↓
Document
      ↓
Source
```

因此 provenance 不是 citation formatting。

它属于 Knowledge Data Model。

---

# 7. Enrichment Policy

负责把原始 Node 转换成更丰富的知识对象。

```python
class EnrichmentPolicy:
    metadata_extraction: bool
    keyword_extraction: bool
    summary_extraction: bool

    entity_extraction: bool
    relation_extraction: bool
    claim_extraction: bool
```

运行：

```text
KnowledgeNode
      ↓
EnrichmentPipeline
      │
      ├── MetadataExtractor
      ├── KeywordExtractor
      ├── SummaryExtractor
      ├── EntityExtractor
      ├── RelationExtractor
      └── ClaimExtractor
```

注意两种产物不同：

```text
Metadata / Summary / Keywords
→ Node Enhancement

Entity / Relationship / Claim
→ Derived Knowledge
```

不要把所有 enrichment 结果都塞回 metadata。

---

# 8. Embedding Policy

Embedding 是知识表示策略，而不是 Node 本体。

```python
class EmbeddingPolicy:
    model: str

    target_fields: list[str]

    dimensions: int | None

    version: str
```

示例：

```yaml
embedding_policy:

  model: text-embedding-model

  target_fields:
    - node.content
    - entity.description
    - community_report.summary

  version: v1
```

运行：

```text
Knowledge Object
      ↓
EmbeddingPolicy
      ↓
EmbeddingModel
      ↓
Vector
      ↓
VectorStore
```

支持多个对象分别 embedding：

```text
Node
→ content embedding

Entity
→ description embedding

CommunityReport
→ summary embedding
```

Embedding Model 更新时：

```text
embedding_policy.version change
        ↓
detect affected objects
        ↓
reembed
        ↓
rebuild vector index
```

而不需要重新 ingestion 原始资料。

---

# 9. Graph Policy

Graph 是 Knowledge System 的派生知识层。

```python
class GraphPolicy:
    entity_types: list[str]

    extract_relationships: bool
    extract_claims: bool

    entity_resolution: bool
    community_detection: bool
```

## Entity

```python
class Entity:
    id: str

    name: str
    type: str
    description: str

    source_node_ids: list[str]
```

## Relationship

```python
class Relationship:
    id: str

    source_entity_id: str
    target_entity_id: str

    relation_type: str

    description: str

    source_node_ids: list[str]
```

## Claim

```python
class Claim:
    id: str

    subject_id: str

    statement: str
    status: str | None

    valid_from: datetime | None
    valid_to: datetime | None

    source_node_ids: list[str]
```

运行：

```text
KnowledgeNode[]
      ↓
Entity / Relationship Extraction
      ↓
Subgraphs
      ↓
Entity Resolution
      ↓
Graph Merge
      ↓
Knowledge Graph
```

之后：

```text
Knowledge Graph
      ↓
Community Detection
      ↓
Community
      ↓
CommunityReport
```

## Community

```python
class Community:
    id: str

    level: int

    entity_ids: list[str]
    relationship_ids: list[str]
```

## Community Report

```python
class CommunityReport:
    community_id: str

    summary: str
    findings: list[str]

    source_node_ids: list[str]
```

最终形成：

```text
Node
 ↓
Entity
 ↓
Relationship
 ↓
Community
 ↓
CommunityReport
```

---

# 10. Storage Policy

Storage 只负责持久化，不承担知识逻辑。

```python
class StoragePolicy:
    document_store: str
    vector_store: str
    graph_store: str
    index_store: str
```

示例：

```yaml
storage_policy:

  document_store: postgres

  vector_store: qdrant

  graph_store: neo4j

  index_store: postgres
```

职责分离：

```text
DocumentStore
→ Document / Node / Metadata

VectorStore
→ Vector + vector metadata

GraphStore
→ Entity / Relationship / Community

IndexStore
→ 索引结构与索引状态
```

Runtime 通过统一接口访问：

```text
StorageContext
     │
     ├── document_store
     ├── vector_store
     ├── graph_store
     └── index_store
```

---

# 11. Indexing

Indexing 的职责是把 Knowledge Objects 转换成 Retriever 可读取的数据结构。

```python
class IndexPolicy:
    vector_index: bool
    keyword_index: bool
    metadata_index: bool
    graph_index: bool
```

形成：

```text
KnowledgeNode
   ├── VectorIndex
   ├── KeywordIndex
   └── MetadataIndex

Entity / Relationship
   └── GraphIndex
```

边界：

```text
Knowledge System
→ 建立 Index

Retrieval Engine
→ 查询 Index
```

因此：

```text
vector search 算法
hybrid search
reranking
time weighting
importance weighting
```

不属于 Knowledge System。

---

# 12. Ingestion Pipeline

整个 Knowledge System 的核心执行链。

```python
class KnowledgePipeline:

    def ingest(self, source):

        document = load(source)

        document = normalize(document)

        if deduplicate(document):
            return

        nodes = parse(document)

        nodes = enrich(nodes)

        nodes = attach_provenance(nodes)

        vectors = embed(nodes)

        graph = build_graph(nodes)

        persist(
            document=document,
            nodes=nodes,
            vectors=vectors,
            graph=graph,
        )
```

完整执行：

```text
External Source
       ↓
SourceRegistry
       ↓
ReaderResolver
       ↓
Document
       ↓
Normalization
       ↓
Hash / Dedup
       ↓
Parser
       ↓
ChunkPolicy
       ↓
KnowledgeNode[]
       ↓
Metadata / Enrichment
       ↓
Provenance Binding
       ↓
┌─────────────────────────┐
│                         │
▼                         ▼
Embedding              Graph Extraction
│                         │
▼                         ▼
VectorStore            GraphStore
│                         │
└───────────┬─────────────┘
            ↓
        Index Build
            ↓
     Knowledge Ready
```

---

# 13. Incremental Update

Knowledge System 不应该每次完整 rebuild。

```python
class ChangeDetector:
    source_hash: str
    document_hash: str
    node_hash: str
```

支持：

```text
ADD
UPDATE
DELETE
UNCHANGED
```

运行：

```text
Source Scan
    ↓
Hash Compare
    ↓
┌───────────────┐
│ unchanged     │ → skip
│ added         │ → ingest
│ modified      │ → update
│ deleted       │ → remove
└───────────────┘
```

修改：

```text
Document changed
      ↓
Affected Nodes
      ↓
Rechunk if necessary
      ↓
Re-enrich
      ↓
Re-embed
      ↓
Update Graph
      ↓
Update Index
```

而不是：

```text
Document changed
→ rebuild whole knowledge base
```

---

# 14. Knowledge Lifecycle

```text
INGEST
知识进入系统
    ↓
NORMALIZE
转换成统一 Document
    ↓
PARSE
Document → Node
    ↓
ENRICH
补充 Metadata / Entity / Relation
    ↓
EMBED
生成向量表示
    ↓
GRAPH
建立结构知识
    ↓
INDEX
建立 Retrieval Index
    ↓
STORE
持久化
    ↓
OBSERVE
监测源变化
    ↓
UPDATE
增量更新
```

操作接口：

```python
class KnowledgeService:

    def add(...): ...
    def update(...): ...
    def upsert(...): ...
    def delete(...): ...

    def reindex(...): ...
    def reembed(...): ...
    def rebuild_graph(...): ...
```

---

# 15. 完整 KnowledgeSpec

最终配置不要散落在各 Runtime 模块中。

```python
class KnowledgeSpec:
    metadata: KnowledgeMetadata

    sources: list[KnowledgeSource]

    parsing_policy: ParsingPolicy

    enrichment_policy: EnrichmentPolicy

    embedding_policy: EmbeddingPolicy

    graph_policy: GraphPolicy

    storage_policy: StoragePolicy

    index_policy: IndexPolicy
```

完整配置：

```yaml
knowledge:

  metadata:
    id: financial_knowledge
    version: "1.0"
    namespace: finance

    tags:
      - finance
      - research

  sources:
    - source_id: filings
      source_type: api
      uri: sec://filings

  parsing_policy:
    parser: auto
    chunk_strategy: hierarchical
    chunk_size: 800
    chunk_overlap: 100
    preserve_structure: true

  enrichment_policy:
    metadata_extraction: true
    keyword_extraction: true
    summary_extraction: false

    entity_extraction: true
    relation_extraction: true
    claim_extraction: false

  embedding_policy:
    model: embedding-model-v1

    target_fields:
      - node.content
      - entity.description
      - community_report.summary

    version: v1

  graph_policy:
    entity_types:
      - person
      - company
      - product
      - event

    extract_relationships: true
    extract_claims: false

    entity_resolution: true
    community_detection: true

  storage_policy:
    document_store: postgres
    vector_store: qdrant
    graph_store: neo4j
    index_store: postgres

  index_policy:
    vector_index: true
    keyword_index: true
    metadata_index: true
    graph_index: true
```

---

# 16. Runtime 装配流程

```text
knowledge.yaml
      ↓
KnowledgeSpec Loader
      ↓
Schema Validation
      ↓
KnowledgeRegistry.register()
      ↓
┌──────────────────────────────┐
│ Knowledge Runtime            │
│                              │
│ Source Registry              │
│ Parsing Pipeline             │
│ Enrichment Pipeline          │
│ Embedding Binding            │
│ Graph Binding                │
│ Storage Binding              │
└──────────────────────────────┘
```

执行 ingestion：

```text
KnowledgeRuntime.ingest(source_id)
        │
        ├── SourceRegistry.get()
        │
        ├── ReaderResolver.resolve()
        │
        ├── ChangeDetector.check()
        │
        ↓
KnowledgePipeline
        │
        ├── document normalize
        ├── parsing
        ├── chunking
        ├── metadata enrichment
        ├── provenance binding
        ├── embedding
        ├── graph extraction
        └── indexing
        ↓
StorageContext
        │
        ├── DocumentStore
        ├── VectorStore
        ├── GraphStore
        └── IndexStore
        ↓
Knowledge Ready
```

---

# 17. 与 Retrieval Engine 的接口

Knowledge System 不负责决定查询结果排序。

对 Retrieval 暴露标准对象：

```python
class KnowledgeAPI:

    def get_document(id): ...

    def get_node(id): ...

    def get_nodes(ids): ...

    def get_entity(id): ...

    def get_neighbors(entity_id): ...

    def get_provenance(id): ...

    def resolve_source(id): ...
```

关系：

```text
Knowledge System
      │
      ├── DocumentStore
      ├── VectorStore
      ├── MetadataIndex
      └── GraphStore
              ↓
       Retrieval Engine
              │
              ├── semantic search
              ├── keyword search
              ├── metadata filter
              ├── graph traversal
              ├── hybrid fusion
              └── rerank
```

因此：

```text
Knowledge System
= 建知识

Retrieval Engine
= 找知识
```

---

# 18. 与 Context Compiler 的接口

Retrieval 返回的是知识对象。

```text
RetrievalResult[]
      ↓
Context Compiler
      │
      ├── select
      ├── deduplicate
      ├── compress
      ├── allocate token budget
      └── format provenance
      ↓
LLM Context
```

Knowledge System 不负责：

```text
top-k selection
token budgeting
context compression
prompt ordering
```

这些属于 Context Compiler / Retrieval Engine。

---

# 19. 与 Memory System 的边界

```text
Knowledge System
→ 外部世界知识

Memory System
→ Agent 运行产生的长期/短期记忆
```

例如：

```text
公司年报
→ Knowledge

研究论文
→ Knowledge

GitHub Repository
→ Knowledge

用户告诉 Agent 自己喜欢简洁输出
→ Memory

Agent 上一次执行失败
→ Episodic Memory
```

但二者可以使用相同基础设施：

```text
Embedding Model
Vector Store
Metadata Filtering
```

逻辑模型仍然分离。

---

# 20. 模块边界

最终建议形成：

```text
knowledge/
├── spec/
│   ├── source
│   ├── document
│   ├── node
│   ├── parsing_policy
│   ├── enrichment_policy
│   ├── embedding_policy
│   ├── graph_policy
│   ├── storage_policy
│   └── index_policy
│
├── source/
│   ├── registry
│   ├── reader
│   └── resolver
│
├── ingestion/
│   ├── pipeline
│   ├── normalizer
│   ├── deduplicator
│   └── change_detector
│
├── parsing/
│   ├── parser_registry
│   ├── chunker
│   └── structure_parser
│
├── enrichment/
│   ├── metadata
│   ├── summary
│   ├── entity
│   ├── relationship
│   └── claim
│
├── provenance/
│   ├── lineage
│   └── source_ref
│
├── embedding/
│   ├── embedder
│   └── cache
│
├── graph/
│   ├── extraction
│   ├── entity_resolution
│   ├── community
│   └── report
│
├── storage/
│   ├── document_store
│   ├── vector_store
│   ├── graph_store
│   └── index_store
│
├── indexing/
│   ├── vector_index
│   ├── keyword_index
│   ├── metadata_index
│   └── graph_index
│
├── registry/
│
└── lifecycle/
```

最终职责：

```text
KnowledgeSpec
→ 定义 Knowledge System

SourceRegistry
→ 管理知识来源

ReaderResolver
→ 读取不同知识源

IngestionPipeline
→ 执行知识进入流程

Parser / Chunker
→ Document → KnowledgeNode

EnrichmentPipeline
→ 提取 Metadata / Entity / Relation / Claim

Provenance
→ 保存知识来源链

EmbeddingRuntime
→ 建立向量表示

GraphRuntime
→ 建立实体关系知识

StorageContext
→ 管理持久化后端

Indexer
→ 建立 Retriever 可消费索引

KnowledgeRegistry
→ 注册不同 Knowledge Domain

KnowledgeRuntime
→ 将所有模块组合并执行
```

最终系统关系：

```text
External Knowledge
       ↓
Knowledge System
       ↓
Structured Knowledge Assets
       │
       ├── Documents
       ├── Nodes
       ├── Metadata
       ├── Embeddings
       ├── Entities
       ├── Relationships
       └── Provenance
       ↓
Retrieval Engine
       ↓
Context Compiler
       ↓
Agent Runtime
```