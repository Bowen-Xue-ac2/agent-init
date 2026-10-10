# Agent 整体架构、生命周期与功能需求

## 结论 / TL;DR

整个 Agent 由八个模块构成：**Identity 定义主体，Session/State 保存现场，Orchestration 控制流程，Context Compiler 生成本轮上下文，Memory 管长期经验，Knowledge 管外部知识，Retrieval 统一召回，Tool/Skill 执行动作**。

核心闭环：

```text
定义与注册 → 启动与装配 → 创建/恢复 Session → 编排任务
→ 检索 Memory/Knowledge → 编译 Context → 模型推理
→ Tool/Agent 执行 → State 提交与 Checkpoint
→ 输出结果 → Memory 沉淀 → 继续/暂停/恢复/结束
```

---

## 1. 八模块理想结构

```text
Agent Platform
├── 1. Agent Identity
│   ├── AgentSpec：metadata / identity_core / persona
│   ├── Policies：prompt / model / capability / delegation
│   ├── Registry / Factory
│   └── IdentityCompiler / PersonaStore
│
├── 2. Session / State
│   ├── Session：session_id / run_id / status
│   ├── SessionState：shared / execution / agents / orchestrator
│   ├── Channel / Reducer / PendingWrite
│   └── Checkpoint / CheckpointStore / Resume / Retry
│
├── 3. Context Compiler
│   ├── SourceLoader / RequirementAnalyzer
│   ├── Conversation：recent / summary / recall
│   ├── Selection：normalize / score / deduplicate / compact
│   ├── BudgetAllocator
│   └── Renderer / CompiledContext
│
├── 4. Memory
│   ├── Core Memory：persona / human / project / index
│   ├── Long-term：profile / semantic / episodic / procedural
│   ├── Write：extract / classify / conflict-resolve / persist
│   ├── Recall / History / Storage
│   └── Lifecycle：reflect / consolidate / promote / decay / expire
│
├── 5. Knowledge
│   ├── SourceRegistry / ReaderResolver
│   ├── Ingestion：normalize / deduplicate / parse / chunk
│   ├── Enrichment / Provenance / Embedding / Graph
│   ├── Document / Vector / Graph / Index Stores
│   └── ChangeDetector / Incremental Update
│
├── 6. Retrieval
│   ├── QueryProcessor / Router / RetrieverRegistry
│   ├── Retrievers：semantic / keyword / metadata / memory / graph
│   ├── Recursive Expansion / Fusion / Scoring / Rerank
│   ├── Postprocess / Cache / Budget
│   └── RetrievalResult / RetrievalTrace
│
├── 7. Tool / Skill Runtime
│   ├── ToolRegistry / SkillRegistry / Exposure
│   ├── Adapters：function / plugin / MCP / OpenAPI
│   ├── Workbench / Connection / State
│   ├── Resolve / Authorize / Validate / Execute / Normalize
│   └── Timeout / Retry / Concurrency / Fallback / Trace
│
└── 8. Orchestration
    ├── WorkflowGraph：node / edge / condition / subgraph
    ├── Engine：schedule / route / dispatch / fan-out / fan-in / commit
    ├── Task / Team / Delegation / Handoff
    ├── Lifecycle / Failure / Termination / Budget Policies
    └── EventBus / Trace / OrchestrationResult
```

### 模块职责与边界

| 模块 | 唯一职责 | 主要输入 | 主要输出 | 不负责 |
|---|---|---|---|---|
| Identity | 定义 Agent 是谁、用什么模型、拥有什么能力 | AgentSpec、Persona | Runtime Agent 配置 | 保存任务现场 |
| Session/State | 保存可恢复的执行事实 | 输入、局部更新、任务结果 | State、Checkpoint | 决定执行顺序 |
| Context Compiler | 生成本轮唯一模型上下文 | Identity、State、RetrievalResult、历史、工具定义 | CompiledContext | 执行工具、持久化知识 |
| Memory | 管理运行产生的长期经验 | 对话、行为、结果、反馈 | MemoryRecord、Persona Candidate | 管理外部文档知识 |
| Knowledge | 建设可检索的外部知识资产 | 文件、API、仓库等来源 | Document、Node、Index、Graph、Provenance | 查询排序与 Prompt 编排 |
| Retrieval | 从 Memory/Knowledge 找到本轮候选信息 | Query、任务上下文、索引 | RetrievalResult、Trace | 修改 Session、分配最终 Token |
| Tool/Skill | 暴露并安全执行外部能力 | ToolCall、ExecutionContext | ToolResult、Artifact、Trace | 决定工作流路由 |
| Orchestration | 控制任务、节点和 Agent 的执行顺序 | WorkflowSpec、State、执行结果 | RouteDecision、OrchestrationResult | 成为第二份 State 或 Context |

### 依赖原则

```text
Orchestration ──控制──→ Agent / Task / Tool / Subgraph
Session/State ──提供──→ 唯一可恢复执行现场
Knowledge ──建索引──→ Retrieval ←──检索源── Memory
Retrieval ──候选──→ Context Compiler ←── Identity / State / History / Tool Schema
Context Compiler ──唯一上下文──→ Model
Model ──决策──→ Orchestration / Tool Runtime
所有执行结果 ──局部写入──→ Reducer → Checkpoint
```

---

## 2. 完整生命周期链路

### 2.1 定义、注册与启动

```text
DEFINE
AgentSpec + WorkflowSpec + 各模块 Spec
    ↓
VALIDATE
Schema、版本、引用、权限、预算、工作流拓扑校验
    ↓
REGISTER
Agent / Workflow / Knowledge / Retriever / Tool / Skill Registry
    ↓
BIND
模型、存储、索引、适配器、Workbench、CheckpointStore
    ↓
START
启动 Tool Workbench，发现动态工具，装载 Memory/Knowledge 索引
    ↓
READY
发布可用版本
```

### 2.2 一次任务的主链

```text
1. RECEIVE
用户输入 / 外部事件 / ResumeCommand
    ↓
2. SESSION
创建或加载 Session → 创建 run_id → 恢复最新 Checkpoint
    ↓
3. ORCHESTRATE
加载 Workflow → Scheduler 选择可执行节点 → Router 决定目标
    ↓
4. BUILD QUERY
RequirementAnalyzer 从输入、目标和 State 生成检索需求
    ↓
5. RETRIEVE
Query 标准化 → 路由 → Memory/Knowledge 多路召回
→ 扩展 → 去重/融合 → 评分 → 重排 → RetrievalResult
    ↓
6. COMPILE CONTEXT
装载 Identity/Persona、会话历史、State、检索结果、工具定义
→ 去重/选择/压缩 → Token 分配 → CompiledContext
    ↓
7. REASON
ModelRouter 选择模型 → LLM 生成答复、ToolCall、Handoff 或 Delegation
    ↓
8. ACT
Tool：Resolve → Authorize → Validate → Execute → Normalize
Agent：创建子任务 → 调度目标 Agent/Team → 汇总结果
    ↓
9. COMMIT
结果转为 Partial State Update → Channel/Reducer
→ PendingWrite → ExecutionState/TaskState 更新 → Checkpoint
    ↓
10. CONTROL
检查中断、失败、终止条件和预算
├── 未完成：回到 3
├── 等待：进入 WAITING
├── 可重试：从稳定 Checkpoint 重跑 Task
└── 完成：生成 OrchestrationResult
    ↓
11. RESPOND
输出文本、结构化结果和 Artifact，记录 Trace
    ↓
12. LEARN
运行事件 → Memory 提取/分类/去重/冲突处理/存储
→ 必要时形成 Persona Candidate
    ↓
13. COMPLETE
提交最终 Checkpoint，Session 标记 COMPLETED / FAILED / CANCELLED
```

### 2.3 暂停、恢复与失败

```text
Interrupt
→ 持久化 PendingWrite
→ 创建 Checkpoint
→ Session = WAITING

ResumeCommand
→ 新建 run_id
→ 加载 Checkpoint / TaskState / AgentState / OrchestratorState
→ 注入恢复值
→ Session = RUNNING
→ 从可重启节点继续

Failure
→ 记录 Error 与 attempt
→ 按 FailurePolicy 选择 retry / fallback / error_route / fail_fast
→ Retry 只从最近稳定 Checkpoint 恢复，不依赖进程调用栈
```

### 2.4 长期维护与演化

```text
Knowledge：Observe Source → ADD / UPDATE / DELETE → 局部重建 Node/Embedding/Graph/Index
Memory：Reflect → Consolidate → Promote/Demote → Decay → Expire/Delete
Persona：Memory Pattern → Candidate → 冲突合并 → Persona Update
Identity：职责/权限/模型策略变化 → 新 AgentSpec Version → 校验 → 发布
Tool：Save/Reset State → Stop Workbench → 升级 Adapter/Schema
Session：按保留策略归档 Checkpoint、Trace 与 Artifact
```

---

## 3. Agent 功能性需求

### 3.1 配置与装配

- **FR-01**：系统必须支持八模块独立 Spec、Schema 校验、版本化和注册。
- **FR-02**：系统必须校验跨模块引用，包括 Agent、Workflow、Tool、Retriever、Knowledge Domain 与存储绑定。
- **FR-03**：系统必须按 AgentSpec 装配 Identity、模型、能力、Persona 与权限，不以 Prompt 代替运行时权限控制。
- **FR-04**：系统必须支持组件替换与多实现绑定，且不改变上层接口。

### 3.2 会话与状态

- **FR-05**：系统必须以 `session_id → run_id → checkpoint_id → task_id` 标识完整执行链。
- **FR-06**：系统必须保存 SharedState、ExecutionState、TaskState、AgentState 与 OrchestratorState。
- **FR-07**：节点只能提交局部更新；系统必须通过 Channel/Reducer 合并并以 PendingWrite 保证顺序和并行隔离。
- **FR-08**：系统必须支持 Checkpoint、暂停、恢复、重试、回滚与 Fork；恢复后不得依赖原进程栈。

### 3.3 编排与模型执行

- **FR-09**：系统必须支持顺序、条件、并行、依赖图、子图及 fan-out/fan-in 工作流。
- **FR-10**：系统必须支持单 Agent、Round Robin、Selector、Handoff、Delegation、Manager/Worker 与 Nested Team。
- **FR-11**：每一步必须执行调度、路由、分发、结果提交、中断检查、终止检查和预算检查。
- **FR-12**：系统必须支持最大 step、turn、并行数、嵌套深度、委派数和 handoff 数限制。
- **FR-13**：系统必须按策略选择模型，并返回文本、结构化输出、工具调用或控制流决策。

### 3.4 上下文与检索

- **FR-14**：Context Compiler 必须是进入模型的唯一上下文出口。
- **FR-15**：系统必须汇集 Identity、Persona、近期历史、摘要、State、Memory、Knowledge、Tool Schema 与 Tool Result。
- **FR-16**：系统必须执行需求分析、标准化、去重、评分、选择、压缩、Token 预算和渲染，并保留来源信息。
- **FR-17**：Retrieval 必须支持语义、关键词、元数据、Memory 与 Graph 多路检索，以及路由、融合、评分、重排和 fallback。
- **FR-18**：Retrieval 只返回候选与 Trace，不得修改 Session；Context Compiler 有权再次筛选、压缩或拒绝候选。

### 3.5 Memory 与 Knowledge

- **FR-19**：Memory 必须支持 profile、semantic、episodic、procedural 与 core memory 的写入、检索、更新、历史和删除。
- **FR-20**：Memory 写入必须包含提取、分类、Scope、重要度、去重、冲突处理和版本记录。
- **FR-21**：系统必须支持反思、合并、晋升、降权、过期，并将身份相关模式转为可审查的 Persona Candidate。
- **FR-22**：Knowledge 必须支持来源注册、读取、标准化、去重、解析、分块、增强、溯源、Embedding、Graph 与索引。
- **FR-23**：Knowledge 必须基于 Hash/版本识别新增、修改、删除和未变化内容，并执行增量更新。
- **FR-24**：Knowledge 与 Memory 必须逻辑隔离；前者管理外部知识，后者管理运行经验。

### 3.6 Tool / Skill 执行

- **FR-25**：系统必须统一接入 Function、Plugin、MCP 与 OpenAPI 工具，并通过 namespace/version 解析。
- **FR-26**：工具执行必须经过 Resolve、Authorize、Argument Validate、Execute、Normalize 和 Trace。
- **FR-27**：系统必须区分 Exposure Policy 与 Permission Policy：可见不等于可执行。
- **FR-28**：系统必须支持 Tool/Workbench 生命周期、状态保存恢复、流式输出、取消、超时、重试、并发限制与 fallback。
- **FR-29**：ToolResult 必须统一表达 success、error、cancelled、timeout，以及 text、JSON、file、image、resource 和 artifact。

### 3.7 输出、事件与可观测性

- **FR-30**：系统必须统一记录 run、node、task、agent、route、retrieval、context、model、tool、state commit 和 lifecycle 事件。
- **FR-31**：所有 Trace 必须可由 `session_id/run_id/task_id/call_id` 关联，并记录耗时、尝试次数、状态与错误类别。
- **FR-32**：系统必须输出最终结果、任务结果、Artifact、最终 State 引用与 Trace 引用。
- **FR-33**：错误必须结构化并标识类别、可重试性、原因与关联执行单元。

---

## 4. 核心状态机与系统不变量

```text
Session：CREATED → RUNNING ↔ WAITING → COMPLETED
                         └──────→ FAILED / CANCELLED

Task：PENDING → RUNNING → COMPLETED
                  ├────→ INTERRUPTED
                  └────→ FAILED → RETRY(RUNNING)

Workflow：CREATED → RUNNING ↔ PAUSED → COMPLETED
                           └──────→ FAILED / CANCELLED / TERMINATED
```

系统必须始终满足：

1. **Checkpoint 是恢复依据，SessionState 是执行事实源，Trace 不是状态源。**
2. **Identity Core 只经版本升级改变；普通交互只更新 Memory/Persona。**
3. **Knowledge 建知识、Memory 管经验、Retrieval 找信息、Context Compiler 决定入模内容。**
4. **Orchestration 只控制流程；业务状态必须通过 Reducer 提交。**
5. **工具暴露、运行时授权、执行结果和副作用必须分别记录并可追溯。**
