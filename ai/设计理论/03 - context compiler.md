# Context Compiler Design

```text
ContextCompiler
│
├── source_loader            # 收集所有候选上下文
│   ├── identity
│   ├── session_state
│   ├── conversation
│   ├── memory
│   ├── knowledge
│   └── tool_context
│
├── requirement_analyzer    # 判断本轮实际需要什么上下文
│   ├── task_intent
│   ├── entities
│   ├── topics
│   └── retrieval_requirements
│
├── conversation_context    # 管理当前对话上下文
│   ├── summary
│   ├── recent_messages
│   └── recall_history
│
├── retrieval_policy        # 从外部状态中召回相关信息
│   ├── memory_retrieval
│   ├── knowledge_retrieval
│   └── history_recall
│
├── compaction_policy       # 长上下文压缩
│   ├── trigger
│   ├── sliding_window
│   ├── recursive_summary
│   └── tool_result_compaction
│
├── selection_policy        # 候选上下文筛选与排序
│   ├── priority
│   ├── relevance
│   ├── recency
│   ├── importance
│   └── deduplication
│
├── budget_policy           # Token 预算分配
│   ├── model_window
│   ├── reserved_output
│   ├── mandatory_budget
│   └── dynamic_allocation
│
├── renderer                # ContextUnit → LLM Messages
│
└── metadata
    ├── compiler_version
    ├── policy_version
    └── metrics
```

## 1. Context Unit

所有进入 Compiler 的信息统一转换为 `ContextUnit`。

```python
class ContextUnit:
    id: str

    source: str
    type: str

    content: str
    token_count: int

    priority: float
    relevance: float
    recency: float
    importance: float

    mandatory: bool
    compressible: bool

    metadata: dict
```

类型：

```text
system
identity
session
task
summary
message
memory
knowledge
tool_result
runtime
```

作用：

```text
不同系统状态
    ↓
ContextUnit 标准化
    ↓
统一执行：
select
score
compact
budget
render
```

---

## 2. Source Loader

负责从 Runtime 各模块读取候选 Context。

```python
class SourceLoader:

    async def load(
        self,
        agent_state,
        session_state,
        user_input,
    ) -> list[ContextUnit]:
        ...
```

来源：

```text
Agent Identity
    ├── identity_core
    └── persona

Session State
    ├── active_task
    ├── task_progress
    └── runtime_state

Conversation
    ├── conversation_summary
    ├── recent_messages
    └── full_history_reference

Memory System
    ├── working
    ├── episodic
    ├── semantic
    ├── procedural
    └── profile

Knowledge System
    └── external knowledge

Tool Runtime
    ├── recent_tool_calls
    └── tool_results
```

边界：

```text
SourceLoader
→ 读取

Memory System / Session System
→ 存储

Context Compiler
→ 不负责底层持久化
```

---

## 3. Requirement Analyzer

判断当前这一轮需要什么 Context。

```python
class ContextRequirement:
    query: str

    task_intent: str | None

    entities: list[str]
    topics: list[str]

    needs_history: bool
    needs_memory: bool
    needs_knowledge: bool
    needs_tools: bool

    retrieval_queries: list[str]
```

示例：

```text
用户：
“继续昨天那个 Agent Memory 架构。”

↓

task_intent:
continue_previous_work

needs_history:
true

needs_memory:
true

retrieval_queries:
- Agent Memory architecture
- previous design
- yesterday discussion
```

运行链：

```text
User Input
+
Current Task
+
Session State
        ↓
RequirementAnalyzer
        ↓
ContextRequirement
```

---

## 4. Conversation Context

Conversation 不直接等于完整消息历史。

采用：

```text
ConversationContext
│
├── summary
├── recent_messages
└── recall_history
```

定义：

```text
summary
→ 已退出 Context Window 的历史压缩状态

recent_messages
→ 当前保留的原始消息

recall_history
→ 完整历史，可按需检索
```

正常 Context：

```text
Conversation Summary
        +
Recent Message N-8
Recent Message N-7
...
Recent Message N
        +
Current User Message
```

完整历史：

```text
Full History
→ History Store
→ 不默认注入模型
```

---

## 5. Recent Message Window

近期对话保留原始内容。

```python
class RecentMessagePolicy:
    max_tokens: int
    min_turns: int
    preserve_tool_pairs: bool
```

规则：

```text
最新消息
→ 高优先级保留

完整 user / assistant turn
→ 尽量保持完整

tool_call + tool_result
→ 不拆分

超过预算
→ 最旧消息进入 Compaction
```

不要使用：

```python
messages[-20:]
```

而是：

```text
message boundary
+
token budget
+
tool dependency
```

共同决定窗口。

---

## 6. Conversation Compaction

对离开近期窗口的消息执行压缩。

```python
class CompactionPolicy:
    mode: str
    trigger_ratio: float
    sliding_window_ratio: float
    max_summary_tokens: int
```

推荐模式：

```text
mode:
sliding_window
```

生命周期：

```text
Recent Messages
      ↓
Context 接近阈值
      ↓
选择旧消息区间
      ↓
Compaction
      ↓
Conversation Summary
      +
Remaining Recent Messages
```

---

## 7. Summary State

Summary 使用结构化语义，不做普通自由摘要。

```python
class ConversationSummary:
    goals: list[str]

    progress: list[str]

    decisions: list[str]

    important_facts: list[str]

    artifacts: list[str]

    errors_and_fixes: list[str]

    current_state: str

    unresolved_items: list[str]

    lookup_hints: list[str]
```

功能：

```text
goals
→ 长期目标

progress
→ 已经完成什么

decisions
→ 已形成的架构 / 方案决策

important_facts
→ 关键事实、参数、约束

artifacts
→ 文件、路径、ID、URL 等引用

errors_and_fixes
→ 已发现问题及处理方式

current_state
→ 当前进行到哪里

unresolved_items
→ 尚未处理的问题

lookup_hints
→ 后续 Recall 使用的关键词
```

---

## 8. Recursive Summary

Summary 不无限堆叠。

采用：

```text
Summary V1
+
New Evicted Messages
        ↓
Summary V2
```

生命周期：

```text
M1 + M2 + M3
      ↓
Summary V1

Summary V1 + M4 + M5
      ↓
Summary V2

Summary V2 + M6
      ↓
Summary V3
```

最终 Context 始终维持：

```text
1 × Current Summary
+
N × Recent Messages
```

---

## 9. History Recall

Summary 无法保存所有细节。

完整历史通过 Recall 按需读取。

```python
class HistoryRecall:

    async def search(
        self,
        queries: list[str],
        top_k: int,
    ) -> list[ContextUnit]:
        ...
```

查询来源：

```text
current query
+
summary.lookup_hints
+
entities
+
task context
```

流程：

```text
Summary 提示存在历史信息
        ↓
RequirementAnalyzer
        ↓
HistoryRecall
        ↓
Relevant Historical Messages
        ↓
Context Compiler
```

因此：

```text
Summary
= continuity state

Recall
= detail recovery
```

---

## 10. Memory Retrieval

Context Compiler 调用 Memory System，不承担 Memory Storage。

```python
class MemoryRetrievalPolicy:
    enabled: bool
    top_k: int
    max_tokens: int
```

接口：

```python
memory_results = await memory_system.search(
    query=requirement.query,
    entities=requirement.entities,
    task=session.task,
)
```

返回：

```text
MemoryRecord[]
        ↓
ContextUnit[]
```

进入 Compiler 后统一执行筛选。

---

## 11. Memory Context 分层

Memory 进入 Context 分两类。

```text
Always-In-Context
│
├── critical profile
├── identity-related memory
└── persistent behavioral state

Retrieved
│
├── episodic
├── semantic
├── procedural
└── project memory
```

关系：

```text
Memory Store
    │
    ├── Critical Context
    │       ↓
    │   Always Include
    │
    └── Long-tail Memory
            ↓
        Retrieval
            ↓
        Relevant Context
```

避免：

```text
全部 Memory
→ System Prompt
```

---

## 12. Knowledge Retrieval

Knowledge 与 Memory 分离。

```python
class KnowledgeRetrievalPolicy:
    top_k: int
    rerank: bool
    preserve_provenance: bool
```

运行：

```text
ContextRequirement
        ↓
KnowledgeSystem.search()
        ↓
Candidate Documents
        ↓
Rerank
        ↓
ContextUnit[]
```

ContextUnit metadata：

```python
metadata = {
    "source_id": source_id,
    "document_id": document_id,
    "chunk_id": chunk_id,
    "provenance": provenance,
}
```

---

## 13. Tool Context

Tool Result 只在其仍然对当前推理有价值时进入 Context。

```python
class ToolContextPolicy:
    max_result_tokens: int
    preserve_recent_results: bool
    compress_large_results: bool
```

Tool Result 状态：

```text
active
→ 当前任务仍依赖

important
→ 后续推理仍需要

archived
→ 已进入 summary / state

expired
→ 不再进入 context
```

大 Tool Output：

```text
Raw Tool Result
      ↓
超过 Token Limit
      ↓
Extract / Compress
      ↓
保留：
- result
- key values
- identifiers
- errors
- status
```

---

## 14. Selection Policy

统一处理所有非 mandatory Context。

```python
class SelectionPolicy:
    priority_weight: float
    relevance_weight: float
    recency_weight: float
    importance_weight: float
```

评分：

```text
score =
    priority_weight   × priority
  + relevance_weight  × relevance
  + recency_weight    × recency
  + importance_weight × importance
```

进一步加入 Token 成本：

```text
context_value =
score / token_count
```

优先保留：

```text
high relevance
+
high importance
+
low token cost
```

---

## 15. Priority Layer

建议 Context Priority：

```text
P0
├── system instructions
├── current user input
└── critical identity

P1
├── active task state
├── session constraints
└── active tool results

P2
├── recent messages
├── relevant memory
└── critical knowledge

P3
├── conversation summary
├── additional knowledge
└── historical recall

P4
└── low-value auxiliary context
```

`P0` 默认：

```python
mandatory = True
```

---

## 16. Deduplication

多个系统可能返回同一事实。

```python
class ContextDeduplicator:

    def deduplicate(
        self,
        units: list[ContextUnit],
    ) -> list[ContextUnit]:
        ...
```

检测：

```text
exact match
semantic overlap
same source_id
same memory_id
same historical message
```

优先保留：

```text
higher confidence
higher relevance
newer version
better provenance
```

---

## 17. Token Budget

模型窗口由 Compiler 统一管理。

```python
class ContextBudget:
    context_window: int

    reserved_output: int
    safety_margin: int

    available_input: int
```

计算：

```text
available_input =
context_window
- reserved_output
- safety_margin
```

例如：

```text
context_window     = 128000
reserved_output    = 16000
safety_margin      = 4000

available_input    = 108000
```

---

## 18. Budget Allocation

采用动态优先分配，而非固定比例。

```text
available_input
      ↓
mandatory
      ↓
identity / task / session
      ↓
recent messages
      ↓
relevant memory
      ↓
knowledge
      ↓
tool context
      ↓
historical recall
```

接口：

```python
class BudgetAllocator:

    def allocate(
        self,
        units: list[ContextUnit],
        budget: ContextBudget,
    ) -> list[ContextUnit]:
        ...
```

Overflow：

```text
超过预算
    ↓
drop low-priority
    ↓
compress compressible unit
    ↓
reduce retrieval top-k
    ↓
compact conversation
```

---

## 19. Context View

保存状态与发送给模型的内容分离。

```python
class ContextView:

    messages: list[LLMMessage]

    async def get_messages(self):
        return self.messages
```

关系：

```text
Persistent History
Session State
Memory
Knowledge
Tool Results
      │
      ↓
Context Compiler
      │
      ↓
Context View
      │
      ↓
LLM
```

因此：

```text
Stored State
≠
Model-visible State
```

---

## 20. Renderer

所有 Context 最终统一由 Renderer 转换。

```python
class ContextRenderer:

    def render(
        self,
        units: list[ContextUnit],
    ) -> list[LLMMessage]:
        ...
```

推荐最终结构：

```text
SYSTEM
│
├── system constitution
└── agent identity

RUNTIME STATE
│
├── active task
└── session state

MEMORY
│
├── persistent profile
└── retrieved memory

CONVERSATION
│
├── conversation summary
├── recalled history
└── recent messages

KNOWLEDGE
└── retrieved knowledge

TOOLS
└── active tool results

CURRENT TURN
└── current user message
```

---

## 21. Compiled Context

Compiler 最终只返回一个标准结构。

```python
class CompiledContext:
    messages: list[LLMMessage]

    token_count: int

    included_units: list[str]
    dropped_units: list[str]

    memory_ids: list[str]
    knowledge_ids: list[str]
    recalled_message_ids: list[str]

    summary_version: str | None

    metadata: dict
```

LLM Runtime 只消费：

```python
compiled_context.messages
```

---

## 22. Compiler 主流程

```python
class ContextCompiler:

    async def compile(
        self,
        agent_state,
        session_state,
        user_input,
        model_config,
    ) -> CompiledContext:

        requirement = self.requirement_analyzer.analyze(
            session_state,
            user_input,
        )

        base_units = await self.source_loader.load(
            agent_state,
            session_state,
            user_input,
        )

        history = await self.conversation_context.resolve(
            session_state,
            requirement,
        )

        memory = await self.memory_retriever.retrieve(
            requirement
        )

        knowledge = await self.knowledge_retriever.retrieve(
            requirement
        )

        tool_context = self.tool_selector.select(
            session_state.tool_results
        )

        units = normalize(
            base_units,
            history,
            memory,
            knowledge,
            tool_context,
        )

        units = self.deduplicator.deduplicate(units)

        units = self.selector.score(units)

        budget = self.budget_manager.create(model_config)

        selected = await self.budget_allocator.allocate(
            units,
            budget,
        )

        messages = self.renderer.render(selected)

        return CompiledContext(
            messages=messages,
            ...
        )
```

---

## 23. Compile Request

```python
class CompileRequest:
    agent_id: str
    session_id: str

    user_input: str

    model_config: ModelConfig

    task_state: dict | None
```

调用：

```python
context = await context_compiler.compile(
    agent_state=agent_state,
    session_state=session_state,
    user_input=user_input,
    model_config=model_config,
)
```

---

## 24. Runtime 集成

```text
User Input
    ↓
SessionRuntime
    ↓
ContextCompiler
    │
    ├── load sources
    ├── analyze requirement
    ├── retrieve
    ├── compact
    ├── select
    ├── allocate tokens
    └── render
    ↓
CompiledContext
    ↓
LLM Runtime
    ↓
Tool / Delegation Loop
    ↓
Response
    ↓
Session State Update
    ↓
Memory Events
```

---

## 25. Context 生命周期

```text
COLLECT
读取候选 Context
    ↓
ANALYZE
识别本轮需求
    ↓
RETRIEVE
召回 Memory / Knowledge / History
    ↓
NORMALIZE
转换 ContextUnit
    ↓
SELECT
评分 / 去重 / 排序
    ↓
COMPACT
压缩超长 Context
    ↓
BUDGET
Token 分配
    ↓
RENDER
构造 LLM Messages
    ↓
EXECUTE
模型推理
    ↓
UPDATE
更新 Session / History
```

---

## 26. Context 与 State 边界

```text
Agent State
├── identity
├── persona
└── persistent config

Session State
├── task state
├── conversation state
└── runtime state

Memory System
├── episodic
├── semantic
├── procedural
└── profile

Knowledge System
└── external knowledge

        ↓

Context Compiler

        ↓

Model Context
```

定义：

```text
Agent State
→ Agent 当前是什么

Session State
→ 当前任务进行到哪里

Memory System
→ Agent 曾经知道什么

Knowledge System
→ 外部世界有什么信息

Context Compiler
→ 当前这一轮模型应该看到什么
```

---

## 27. Metadata / Metrics

```python
class ContextCompilerMetadata:
    compiler_version: str
    policy_version: str
```

每轮记录：

```python
class ContextMetrics:
    total_tokens: int

    system_tokens: int
    identity_tokens: int
    session_tokens: int
    summary_tokens: int
    recent_message_tokens: int
    memory_tokens: int
    knowledge_tokens: int
    tool_tokens: int

    retrieved_memory_count: int
    retrieved_knowledge_count: int
    recalled_message_count: int

    dropped_units: int

    compaction_triggered: bool
```

用于定位：

```text
为什么 Memory 没进入模型？
为什么 Context 爆掉？
为什么历史内容丢失？
为什么某个 Tool Result 仍然占用 Context？
```

---

# 28. 完整 ContextCompilerSpec

```python
class ContextCompilerSpec:

    requirement_policy: RequirementPolicy

    conversation_policy: ConversationPolicy

    retrieval_policy: RetrievalPolicy

    compaction_policy: CompactionPolicy

    selection_policy: SelectionPolicy

    budget_policy: BudgetPolicy

    render_policy: RenderPolicy
```

配置：

```yaml
context_compiler:

  conversation:
    mode: sliding_window
    preserve_recent_turns: 8

  retrieval:
    memory:
      enabled: true
      top_k: 8

    knowledge:
      enabled: true
      top_k: 6

    history:
      enabled: true
      top_k: 6

  compaction:
    trigger_ratio: 0.75
    summary_mode: recursive
    preserve_lookup_hints: true

  selection:
    priority_weight: 1.0
    relevance_weight: 1.0
    recency_weight: 0.4
    importance_weight: 0.6

  budget:
    reserved_output: 16000
    safety_margin: 4000

  render:
    preserve_provenance: true
```

---

# 29. 模块目录

```text
context/
│
├── compiler
│   ├── compiler.py
│   └── models.py
│
├── source
│   ├── loader.py
│   └── normalizer.py
│
├── requirement
│   └── analyzer.py
│
├── conversation
│   ├── recent_window.py
│   ├── summary.py
│   └── recall.py
│
├── retrieval
│   ├── memory.py
│   ├── knowledge.py
│   └── history.py
│
├── compaction
│   ├── planner.py
│   ├── summarizer.py
│   └── tool_compactor.py
│
├── selection
│   ├── scorer.py
│   ├── deduplicator.py
│   └── selector.py
│
├── budget
│   ├── token_counter.py
│   └── allocator.py
│
├── render
│   └── renderer.py
│
└── observability
    └── metrics.py
```

最终职责：

```text
SourceLoader
→ 收集候选信息

RequirementAnalyzer
→ 判断本轮需要什么

ConversationManager
→ 管理 summary / recent / recall

Retriever
→ 拉取 Memory / Knowledge / History

Compactor
→ 控制长期上下文增长

Selector
→ 判断什么值得进入 Context

BudgetAllocator
→ 控制 Token 使用

Renderer
→ 转换为模型消息

ContextCompiler
→ 编排以上模块并输出唯一 Context View
```

整个模块最终只解决一个问题：

```text
All Available State
        ↓
Context Compiler
        ↓
Best Context For This Turn
        ↓
LLM
```