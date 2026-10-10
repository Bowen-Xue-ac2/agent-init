# Session / State(这本质是进程管理的交互部分，但是针对不同OS，权限关系层不同都是需要重新考虑的，本架构只作为指导思想存在，其应用落地有高度不确定性)

## 1. 模块目标

负责 Agent 单次任务到长期会话期间的运行状态管理：

```text
create session
    ↓
load state
    ↓
execute
    ↓
apply writes
    ↓
checkpoint
    ↓
continue / interrupt / fail
    ↓
resume
    ↓
complete
```

核心职责：

- Session 生命周期管理
    
- State 数据管理
    
- Agent / Team 子状态管理
    
- 执行位置与任务状态管理
    
- Checkpoint 持久化与恢复
    

---

## 2. 核心对象模型

```text
Session
│
├── SessionMeta
├── SessionState
│   ├── SharedState
│   ├── ExecutionState
│   ├── AgentStates
│   └── OrchestratorState
│
├── CurrentCheckpoint
│
└── CheckpointHistory
```

---

## 3. Session

Session 是一次持续任务 / 对话 / workflow 的生命周期容器。

```python
Session:
    session_id: str
    status: SessionStatus

    current_run_id: str | None
    current_checkpoint_id: str | None

    created_at: datetime
    updated_at: datetime

    state_version: str
```

状态：

```text
CREATED
RUNNING
WAITING
FAILED
COMPLETED
CANCELLED
```

典型状态流：

```text
CREATED
   ↓
RUNNING
   ├── WAITING ──resume──→ RUNNING
   ├── FAILED  ──retry───→ RUNNING
   └── COMPLETED
```

---

## 4. 标识体系(绝对核心)

```text
session_id
    └── run_id
         └── checkpoint_id
              └── task_id
```

### session_id

持续 Session 的唯一身份。

```text
session_001
```

### run_id

一次具体 runtime 执行。

```text
session_001
├── run_001
└── run_002
```

Resume、retry 或新的用户输入都可以形成新的 run。

### checkpoint_id

某一执行时刻的状态快照。

```text
run_001
├── cp_001
├── cp_002
└── cp_003
```

### task_id

一次具体 Node / Tool / Agent 执行实例。

---

## 5. SessionState

Session 当前可恢复状态。

```python
SessionState:
    type: str
    version: str

    shared: SharedState
    execution: ExecutionState

    agent_states: dict[str, AgentState]
    orchestrator: OrchestratorState | None
```

`type + version` 用于：

```text
serialization
deserialization
schema validation
state migration
```

---

## 6. SharedState

Agent Workflow 的业务状态。

```python
SharedState:
    task: Any
    messages: list
    variables: dict
    artifacts: dict
    results: dict
```

作用：

```text
描述：
“当前系统知道什么”
```

更新规则：

```text
Node:

State
  ↓
execute
  ↓
Partial<State>
  ↓
Reducer / Channel
  ↓
New State
```

禁止 Node 直接覆盖整个 SessionState。

---

## 7. ExecutionState（冻结）快照

描述当前执行现场。

```python
ExecutionState:
    step: int

    current_node: str | None
    next_nodes: list[str]

    pending_tasks: list[TaskState]

    updated_fields: list[str]

    interrupts: list[InterruptState]

    status: ExecutionStatus
```

状态：

```text
READY
RUNNING
WAITING
FAILED
DONE
```

其核心问题是：

```text
现在执行到哪里？

下一步执行什么？

哪些任务已经完成？

哪些任务仍然 pending？
```

---

## 8. TaskState

记录具体执行单元。

```python
TaskState:
    task_id: str
    node_id: str

    status: TaskStatus

    attempt: int

    input_ref: str | None
    result: Any | None
    error: Any | None

    interrupts: list
```

状态：

```text
PENDING
RUNNING
INTERRUPTED
FAILED
COMPLETED
```

用于：

```text
retry
resume
parallel execution
failure recovery
debug
```

---

## 9. AgentState

每个 Agent 保存自己的内部可恢复状态。

```python
AgentState:
    type: str
    version: str

    agent_id: str

    model_context: dict
    local_state: dict
```

例如：

```text
agent_states
├── planner
│   ├── model_context
│   └── local_state
│
├── researcher
│   ├── model_context
│   └── local_state
│
└── executor
    ├── model_context
    └── local_state
```

其中：

```text
model_context
= 当前 Agent 自己的 LLM conversation/context

local_state
= 当前 Agent 独有的运行变量
```

---

## 10. OrchestratorState

保存 Multi-Agent 调度器自身状态。

```python
OrchestratorState:
    type: str
    version: str

    current_agent: str | None
    previous_agent: str | None

    current_turn: int

    plan: Any | None

    custom: dict
```

不同 orchestration 模式可扩展：

```text
RoundRobin
└── next_agent_index

Selector
└── previous_agent

Swarm
└── current_agent

Planner
├── plan
├── current_step
└── completed_steps
```

---

## 11. State Channel / Reducer

State 字段不直接接受任意覆盖。

每个字段定义：

```python
StateChannel:
    key
    value_type
    update_type

    get()
    update()
    checkpoint()
    restore()
```

典型策略：

```text
messages
→ append / merge

current_task
→ last value

results
→ dict merge

events
→ append

counter
→ arithmetic reducer
```

数据路径：

```text
Node output
   ↓
State Update
   ↓
Channel / Reducer
   ↓
State
```

---

## 12. PendingWrite

Task 执行期间产生、尚未形成完整 checkpoint 的状态更新。

```python
PendingWrite:
    checkpoint_id: str

    task_id: str
    task_path: str

    sequence: int

    channel: str
    value: Any
```

生命周期：

```text
Checkpoint N
     ↓
Task execution
     ↓
PendingWrite
     ↓
PendingWrite
     ↓
merge
     ↓
Checkpoint N+1
```

作用：

```text
partial execution recovery
parallel task isolation
write ordering
crash recovery
```

---

## 13. Checkpoint

Checkpoint 表示某个确定执行时刻的完整可恢复现场。

```python
Checkpoint:
    checkpoint_id: str
    session_id: str
    run_id: str

    parent_checkpoint_id: str | None

    state_version: str

    state: SessionState

    step: int
    source: CheckpointSource

    created_at: datetime
```

`source`：

```text
INPUT
EXECUTION
UPDATE
FORK
RESUME
```

形成链：

```text
cp_001
   ↓
cp_002
   ↓
cp_003
   ↓
cp_004
```

支持：

```text
resume
rollback
time travel
fork
debug
```

---

## 14. Checkpoint Metadata

执行控制信息与业务 State 分离。

```python
CheckpointMetadata:
    step: int
    source: str

    run_id: str

    parent_id: str | None

    updated_fields: list[str]

    created_at: datetime
```

---

## 15. Interrupt

主动暂停 Workflow。

```python
InterruptState:
    interrupt_id: str

    task_id: str
    node_id: str

    value: Any

    created_at: datetime
```

运行逻辑：

```text
Node
 ↓
interrupt()
 ↓
persist pending writes
 ↓
checkpoint
 ↓
Session = WAITING
```

典型用途：

```text
human approval
missing input
external event waiting
manual review
```

---

## 16. Resume

Resume 是一次新的 runtime execution。

```python
ResumeCommand:
    interrupt_id: str
    value: Any
```

流程：

```text
ResumeCommand
      ↓
load checkpoint
      ↓
restore state
      ↓
restore pending task
      ↓
inject resume value
      ↓
re-enter node
      ↓
continue execution
```

Node 应按照：

```text
restartable execution
```

设计，而不是依赖 Python stack continuation。

---

## 17. Retry

Retry 复用最近稳定 checkpoint。

```text
task failed
    ↓
record error
    ↓
retry allowed?
   / \
 yes  no
 ↓     ↓
load   session failed
state
 ↓
task attempt +1
 ↓
execute
```

`attempt` 属于 TaskState，而非业务 State。

---

## 18. CheckpointStore

统一持久化接口。

```python
class CheckpointStore:

    get(session_id, checkpoint_id=None)

    put(checkpoint)

    put_writes(checkpoint_id, writes)

    get_writes(checkpoint_id)

    list(session_id)

    delete(...)
```

实现：

```text
CheckpointStore
├── MemoryCheckpointStore
├── SQLiteCheckpointStore
└── PostgresCheckpointStore
```

---

## 19. 推荐持久化模型

### sessions

```text
session_id
status
current_run_id
current_checkpoint_id
state_version
created_at
updated_at
```

### checkpoints

```text
session_id
checkpoint_id
parent_checkpoint_id
run_id
step
source
state
metadata
created_at
```

### checkpoint_writes

```text
session_id
checkpoint_id
task_id
task_path
sequence
channel
value
```

### agent_states

可选择：

```text
直接嵌入 checkpoint
```

或：

```text
独立存储大型 AgentState
```

---

## 20. Runtime 与 State 边界

必须保持：

```text
Session State
≠ Runtime Context
≠ Memory
```

### Session State

```text
当前任务可恢复状态
```

例如：

```text
task
messages
plan
current_node
pending_tasks
agent_states
```

### Runtime Context

仅属于当前 run：

```text
user_id
db connection
tool registry
stream writer
execution metadata
heartbeat
run control
```

不直接进入业务 State。

### Memory

跨 Session 信息：

```text
profile
semantic memory
episodic memory
procedural memory
```

由 Memory System 独立负责。

---

## 21. 完整执行链

```text
User Input
    ↓
SessionManager
    ↓
load/create Session
    ↓
load latest Checkpoint
    ↓
restore SessionState
    ↓
Runtime
    ↓
Scheduler
    ↓
Task
    ↓
Agent / Tool / Node
    ↓
Partial State Update
    ↓
Channel / Reducer
    ↓
PendingWrite
    ↓
apply writes
    ↓
ExecutionState update
    ↓
Checkpoint
    ↓
CheckpointStore
    ↓
next task
```

暂停：

```text
execute
 ↓
interrupt
 ↓
checkpoint
 ↓
WAITING
```

恢复：

```text
ResumeCommand
 ↓
load checkpoint
 ↓
restore task/state
 ↓
RUNNING
```

---

## 22. 最终目录

```text
session/
├── models/
│   ├── session.py
│   ├── state.py
│   ├── task.py
│   ├── agent_state.py
│   ├── orchestrator_state.py
│   ├── checkpoint.py
│   └── interrupt.py
│
├── state/
│   ├── channel.py
│   ├── reducer.py
│   └── update.py
│
├── checkpoint/
│   ├── base.py
│   ├── memory.py
│   ├── sqlite.py
│   └── postgres.py
│
├── runtime/
│   ├── lifecycle.py
│   ├── restore.py
│   ├── resume.py
│   └── retry.py
│
└── manager.py
```

---

# 最终职责定义

```text
Session
→ 管理一次持续任务的生命周期

SessionState
→ 当前所有可恢复状态的根对象

SharedState
→ 保存 Workflow 共享业务状态

ExecutionState
→ 保存当前执行位置与调度状态

TaskState
→ 保存单个执行任务状态

AgentState
→ 保存各 Agent 的内部上下文

OrchestratorState
→ 保存 Multi-Agent 调度进度

Channel / Reducer
→ 控制 State Update 如何合并

PendingWrite
→ 保存尚未形成完整 checkpoint 的中间更新

Checkpoint
→ 固化完整可恢复执行现场

CheckpointStore
→ 持久化 checkpoint 与 writes

Interrupt
→ 将运行状态安全转换为 WAITING

Resume
→ 从 checkpoint 恢复并重新进入执行

Retry
→ 从稳定状态重新执行失败 Task
```

整体抽象：

```text
Session
    ↓
State
    ↓
Execution
    ↓
Writes
    ↓
Checkpoint
    ↓
Resume / Retry / Continue
```