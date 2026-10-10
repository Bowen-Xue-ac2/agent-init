```text
Orchestration
│
├── workflow_graph              # 定义工作流静态拓扑
│   ├── nodes                   # Agent / Task / Tool / Function / Subgraph
│   ├── edges                   # 固定控制流
│   ├── conditional_edges       # 条件分支
│   ├── entry_point             # 工作流入口
│   ├── finish_points           # 工作流出口
│   └── subgraphs               # 嵌套工作流
│
├── execution_engine            # 推动 Workflow 实际运行
│   ├── execution_loop          # schedule → execute → commit → next
│   ├── scheduler               # 判断当前可执行节点
│   ├── dispatcher              # 分发到 Agent / Tool / Task Runtime
│   ├── parallel_executor       # 并行执行节点
│   ├── fan_out                 # 单节点展开多分支
│   ├── fan_in                  # 多分支汇合
│   └── result_commit           # 执行结果写回 State
│
├── routing_policy              # 决定下一执行目标
│   ├── static                  # 固定 Edge
│   ├── conditional             # State / Result 条件路由
│   ├── selector                # 动态选择 Node / Agent
│   ├── handoff                 # Agent 主动转移控制权
│   └── fallback                # 默认回退路径
│
├── team_orchestrator           # Multi-Agent 协作
│   ├── participants            # Agent / Nested Team
│   ├── round_robin             # 固定轮转
│   ├── selector                # 动态 Speaker Selection
│   ├── swarm                   # Handoff 驱动协作
│   └── nested_team             # Team 嵌套
│
├── task_orchestrator           # Task 编排与分配
│   ├── task_definition         # Task 标准定义
│   ├── dependencies            # Task 依赖关系
│   ├── sequential              # 顺序任务
│   ├── hierarchical            # Manager / Worker
│   ├── delegation              # Agent 间任务委派
│   └── concurrent              # 并行任务
│
├── execution_context           # 单次 Run 的运行定位
│   ├── run_id
│   ├── session_ref
│   ├── state_ref
│   ├── current_node
│   ├── current_task
│   ├── current_agent
│   └── parent_run_id
│
├── lifecycle_policy            # Workflow 生命周期
│   ├── start
│   ├── interrupt
│   ├── resume
│   ├── cancel
│   └── terminate
│
├── termination_policy          # 判断执行何时结束
│   ├── graph_end
│   ├── task_complete
│   ├── condition_match
│   ├── max_steps
│   ├── max_turns
│   └── external_stop
│
├── failure_policy              # 执行失败后的控制流
│   ├── retry
│   ├── fallback
│   ├── continue
│   ├── fail_fast
│   └── error_route
│
├── orchestration_budget        # 控制编排执行规模
│   ├── max_steps
│   ├── max_turns
│   ├── max_parallelism
│   ├── max_nested_depth
│   ├── max_delegations
│   └── max_handoffs
│
├── event_bus                   # 编排运行事件
│   ├── run_events
│   ├── node_events
│   ├── task_events
│   ├── agent_events
│   ├── handoff_events
│   └── lifecycle_events
│
├── observability               # 执行链路记录
│   ├── execution_trace
│   ├── route_trace
│   ├── agent_transition
│   ├── task_trace
│   └── latency
│
└── metadata                    # Orchestration 管理信息
    ├── id
    ├── version
    ├── namespace
    └── tags
```

```text
WorkflowSpec
├── nodes
├── edges
├── entry_point
├── finish_points
└── subgraphs
```

```text
NodeSpec
├── id
├── type                  # agent | task | tool | function | team | subgraph
├── executor_ref
├── input_mapping
├── output_mapping
└── config
```

```text
EdgeSpec
├── source
├── target
├── type                  # static | conditional | handoff
└── condition_ref
```

```text
TaskSpec
├── id
├── description
├── expected_output
├── agent_ref
├── dependencies
├── context
└── config
```

```text
TeamSpec
├── id
├── participants
├── strategy              # round_robin | selector | swarm
├── termination_policy
├── max_turns
└── config
```

```text
ExecutionContext
├── run_id
├── workflow_id
├── session_ref
├── state_ref
├── current_node
├── current_task
├── current_agent
└── parent_run_id
```

```text
RouteDecision
├── source
├── destinations
├── selected_agent
├── route_type
├── payload
└── reason
```

```text
DelegationRequest
├── source_agent
├── target_agent
├── task
└── context
```

```text
Handoff
├── source_agent
├── target_agent
├── payload
└── reason
```

```text
InterruptEvent
├── run_id
├── node_id
├── payload
└── reason
```

```text
OrchestrationResult
├── run_id
├── status
├── output
├── final_node
├── final_agent
├── task_results
├── state_ref
└── trace_ref
```

```text
Run Request
    ↓
Workflow Load
    ↓
ExecutionContext
    ↓
State Load
    ↓
Scheduler
    ↓
Router
    ↓
Dispatcher
    ↓
┌──────────┬──────────┬──────────┬──────────┐
│ Agent    │ Task     │ Tool     │ Subgraph │
└──────────┴──────────┴──────────┴──────────┘
    ↓
Collect Result
    ↓
Commit State
    ↓
Interrupt Check
    ↓
Termination Check
    ↓
Next Step
    ↓
OrchestrationResult
```

```text
Multi-Agent
│
├── Round Robin
│   └── A → B → C → A
│
├── Selector
│   └── Context → Agent Selection
│
├── Handoff
│   └── Agent A → Agent B
│
├── Delegation
│   └── Agent A → Task → Agent B
│
└── Nested Team
    └── Team → Agent / Team
```

```text
Task Orchestration
│
├── Sequential
│   └── Task A → Task B → Task C
│
├── Parallel
│   └── Task A → [Task B, Task C]
│
├── Dependency Graph
│   └── dependencies → scheduler
│
└── Hierarchical
    └── Manager → Workers → Manager
```

```text
Lifecycle
│
├── created
├── running
├── paused
├── resumed
├── completed
├── failed
├── cancelled
└── terminated
```

```text
orchestration/
├── spec/
│   ├── orchestration
│   ├── workflow
│   ├── node
│   ├── edge
│   ├── task
│   ├── team
│   ├── context
│   └── result
│
├── workflow/
│   ├── graph
│   ├── compiler
│   └── validator
│
├── execution/
│   ├── engine
│   ├── scheduler
│   ├── dispatcher
│   ├── fan_out
│   ├── fan_in
│   └── commit
│
├── routing/
│   ├── router
│   ├── static
│   ├── conditional
│   ├── selector
│   └── handoff
│
├── team/
│   ├── base
│   ├── round_robin
│   ├── selector
│   ├── swarm
│   └── nested
│
├── task/
│   ├── executor
│   ├── dependency
│   ├── sequential
│   ├── hierarchical
│   └── delegation
│
├── lifecycle/
│   ├── controller
│   ├── interrupt
│   ├── resume
│   ├── cancel
│   └── termination
│
├── policy/
│   ├── execution
│   ├── routing
│   ├── team
│   ├── task
│   ├── failure
│   └── budget
│
├── events/
│   ├── event
│   └── bus
│
├── adapters/
│   ├── state
│   ├── agent
│   ├── tool
│   └── context
│
├── observability/
│
├── evaluation/
│
└── runtime/
```

```text
LangGraph
├── StateGraph            → Workflow Graph
├── Node / Edge           → NodeSpec / EdgeSpec
├── Conditional Edge      → Conditional Router
├── Pregel                → Execution Engine
├── Interrupt             → Lifecycle Control
└── Subgraph              → Nested Workflow
```

```text
AutoGen
├── RoundRobinGroupChat   → Round Robin
├── SelectorGroupChat     → Selector
├── Swarm                 → Handoff Team
├── HandoffMessage        → Handoff
├── TerminationCondition  → Termination Policy
└── Nested Team           → Team Composition
```

```text
CrewAI
├── Crew                  → Team / Task Container
├── Task                  → TaskSpec
├── Sequential Process    → Sequential Task
├── Hierarchical Process  → Manager / Worker
├── Manager Agent         → Hierarchical Orchestrator
├── Delegation            → DelegationRequest
└── Flow                  → Workflow Structure
```

```text
Agent Identity
→ 定义执行主体

Session / State
→ 保存执行状态

Orchestration
→ 控制执行顺序与协作关系

Context Compiler
→ 构造本轮模型上下文

Retrieval / Memory / Knowledge
→ 提供上下文信息

Tool / Skill Runtime
→ 提供执行能力

Agent Runtime
→ 执行 Agent 推理与动作
```

```text
Orchestration
=
Workflow Graph
+
Execution Engine
+
Routing
+
Task Coordination
+
Multi-Agent Coordination
+
Lifecycle Control
+
Termination
```