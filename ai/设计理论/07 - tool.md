```text
ToolSkillRuntime
│
├── tool_model              # 单个可执行能力
│   ├── id
│   ├── name
│   ├── namespace
│   ├── description
│   ├── input_schema
│   ├── output_schema
│   └── metadata
│
├── skill_model             # Tool + Instructions 的能力集合
│   ├── id
│   ├── name
│   ├── instructions
│   ├── tool_refs
│   └── resource_refs
│
├── tool_registry           # Tool 注册、发现与解析
│   ├── register
│   ├── unregister
│   ├── resolve
│   └── discover
│
├── skill_registry          # Skill 注册与激活
│   ├── register
│   ├── activate
│   ├── deactivate
│   └── discover
│
├── adapter_layer           # 外部能力统一适配
│   ├── function
│   ├── plugin
│   ├── mcp
│   ├── openapi
│   └── builtin
│
├── workbench               # 共享资源 / State 的 Tool 容器
│   ├── list_tools
│   ├── call_tool
│   ├── start
│   ├── stop
│   ├── reset
│   └── state
│
├── execution_engine        # Tool Call 执行主链
│   ├── resolver
│   ├── validator
│   ├── executor
│   ├── result_normalizer
│   └── error_mapper
│
├── execution_context       # 单次 Tool Call 上下文
│   ├── call_id
│   ├── agent_id
│   ├── session_id
│   ├── task_id
│   ├── cancellation_token
│   └── runtime_metadata
│
├── execution_policy        # 执行控制
│   ├── timeout
│   ├── retry
│   ├── concurrency
│   └── fallback
│
├── access_policy           # 能力访问控制
│   ├── exposure
│   ├── permission
│   └── capability_scope
│
├── state                   # Tool / Workbench 状态
│   ├── tool_state
│   ├── workbench_state
│   └── connection_state
│
├── lifecycle               # Runtime 生命周期
│   ├── initialize
│   ├── start
│   ├── reset
│   └── stop
│
└── observability           # Tool 调用追踪
    ├── trace
    ├── latency
    ├── error
    └── usage
```

# 1. Tool

```python
class ToolSpec:
    id: str
    name: str
    namespace: str
    description: str

    input_schema: dict
    output_schema: dict | None

    metadata: dict
```

统一接口：

```python
class BaseTool:

    async def invoke(
        self,
        arguments: dict,
        context: ToolExecutionContext
    ) -> ToolResult:
        ...

    async def stream(...):
        ...

    async def save_state(self) -> dict:
        ...

    async def load_state(self, state: dict):
        ...
```

统一：

```text
Python Function
MCP Tool
OpenAPI Function
Plugin Function
Builtin Tool
      ↓
    Tool
```

---

# 2. Skill

```python
class SkillSpec:
    id: str
    name: str
    description: str

    instructions: str

    tool_refs: list[str]
    resource_refs: list[str]

    metadata: dict
```

结构：

```text
Skill
│
├── instructions
├── tools
├── resources
└── activation
```

关系：

```text
Tool
→ 原子执行能力

Skill
→ 一组完成特定任务的 Tool + Instructions
```

---

# 3. Tool Registry

```python
class ToolRegistry:

    def register(tool): ...
    def unregister(tool_id): ...

    def get(tool_id): ...
    def resolve(name): ...

    def list(): ...
    def search(query): ...
```

索引：

```text
ToolRegistry
├── by_id
├── by_name
├── by_namespace
├── by_capability
├── by_tag
└── by_source
```

命名：

```text
namespace.tool_name
```

例如：

```text
github.search_code
filesystem.read_file
browser.navigate
knowledge.search
```

---

# 4. Skill Registry

```python
class SkillRegistry:

    def register(skill): ...
    def get(skill_id): ...

    def activate(skill_id): ...
    def deactivate(skill_id): ...

    def list(): ...
```

执行：

```text
Task
 ↓
Skill Selection
 ↓
Skill Activation
 ↓
Resolve Tool References
 ↓
Visible Tool Set
```

---

# 5. Function Adapter

```text
Python Function
      ↓
Signature Inspection
      ↓
Type Annotation
      ↓
Input Schema
      ↓
FunctionTool
      ↓
Tool Registry
```

支持：

```text
sync
async
stream
cancellation
typed arguments
typed return
```

---

# 6. Plugin Adapter

```text
Plugin
│
├── Function A
├── Function B
└── Function C
```

转换：

```text
Plugin
→ namespace

Function
→ Tool
```

例如：

```text
GitHubPlugin
├── search_code
└── fetch_file
```

转换：

```text
github.search_code
github.fetch_file
```

---

# 7. Workbench

```python
class Workbench:

    async def list_tools(self): ...

    async def call_tool(
        self,
        name,
        arguments,
        context
    ): ...

    async def start(self): ...
    async def stop(self): ...
    async def reset(self): ...

    async def save_state(self): ...
    async def load_state(self, state): ...
```

结构：

```text
Workbench
│
├── tools
├── connection
├── session
├── process
├── cache
└── state
```

负责：

```text
共享 Connection
共享 Session
共享 Process
共享 State
统一 Lifecycle
```

---

# 8. MCP Adapter

```text
MCP Server
    ↓
Transport
    ├── stdio
    ├── SSE
    └── Streamable HTTP
    ↓
MCP Session
    ↓
MCPWorkbench
    ↓
Tool Schema Normalize
    ↓
Tool Runtime
```

Tool：

```text
list_tools
call_tool
```

扩展：

```text
list_resources
read_resource

list_prompts
get_prompt
```

映射：

```text
MCP Tool
→ Tool Runtime

MCP Resource
→ Resource Adapter

MCP Prompt
→ Skill / Prompt Resource
```

---

# 9. OpenAPI Adapter

```text
OpenAPI Spec
     ↓
Operation Parser
     ↓
Input Schema
     ↓
HTTP Executor
     ↓
Tool
```

例如：

```text
GET /users/{id}
```

转换：

```text
users.get_user
```

---

# 10. Execution Context

```python
class ToolExecutionContext:
    call_id: str

    agent_id: str | None
    session_id: str | None
    task_id: str | None

    trace_id: str | None

    timeout: float | None
    cancellation_token: object | None

    credentials: dict
    metadata: dict
```

关系：

```text
Agent Runtime
      ↓
ExecutionContext
      ↓
Tool Runtime
```

---

# 11. Tool Resolution

```text
Tool Call
   ↓
Alias Resolution
   ↓
Namespace Resolution
   ↓
Version Resolution
   ↓
Tool Registry
   ↓
Executable Tool
```

接口：

```python
class ToolResolver:

    def resolve(
        self,
        tool_name: str,
        context: ToolExecutionContext
    ) -> BaseTool:
        ...
```

---

# 12. Argument Validation

```text
LLM Arguments
     ↓
JSON Parse
     ↓
Schema Validation
     ↓
Type Conversion
     ↓
Validated Arguments
```

处理：

```text
missing parameter
wrong type
unknown property
enum mismatch
nested schema error
```

---

# 13. Execution Engine

```python
class ToolExecutionEngine:

    async def execute(
        self,
        request: ToolCallRequest
    ) -> ToolResult:
        ...
```

主链：

```text
Tool Call
   ↓
Resolve
   ↓
Authorize
   ↓
Validate
   ↓
Build Context
   ↓
Execute
   ↓
Normalize
   ↓
Trace
   ↓
ToolResult
```

---

# 14. Tool Result

```python
class ToolResult:
    call_id: str
    tool_id: str

    status: str

    content: list
    artifacts: list

    error: dict | None

    metadata: dict
```

状态：

```text
success
error
cancelled
timeout
```

内容：

```text
text
json
image
file
resource
structured_data
```

---

# 15. Error Model

```python
class ToolError:
    code: str
    message: str
    category: str
    retryable: bool
    details: dict
```

类型：

```text
validation_error
permission_error
not_found
timeout
cancelled
connection_error
execution_error
remote_error
rate_limit
```

---

# 16. Exposure Policy

```text
Tool Registry
      ↓
Skill Activation
      ↓
Tool Discovery
      ↓
Exposure Policy
      ↓
Visible Tool Set
      ↓
Context Compiler
```

定义：

```text
Exposure
→ 模型能看到什么 Tool
```

---

# 17. Permission Policy

```text
Resolved Tool
      ↓
Permission Policy
      ↓
Allowed?
  ┌──────┴──────┐
  yes           no
  ↓             ↓
Execute    PermissionError
```

定义：

```text
Permission
→ Runtime 是否允许真正执行
```

---

# 18. Timeout / Retry / Concurrency

```text
ExecutionPolicy
│
├── timeout
│   └── maximum execution time
│
├── retry
│   ├── max_attempts
│   ├── backoff
│   └── retryable_errors
│
├── concurrency
│   ├── global_limit
│   ├── per_tool_limit
│   └── serialization_group
│
└── fallback
    ├── fallback_tools
    └── max_depth
```

---

# 19. State

```text
RuntimeState
│
├── tool_states
├── workbench_states
├── connection_states
└── runtime_metadata
```

Tool：

```text
save_state()
load_state()
```

Workbench：

```text
save_state()
load_state()
reset()
```

---

# 20. Lifecycle

```text
LOAD
 ↓
REGISTER
 ↓
INITIALIZE
 ↓
START
 ↓
DISCOVER
 ↓
EXECUTE
 ↓
SAVE / RESET
 ↓
STOP
```

---

# 21. Observability

```python
class ToolTrace:
    call_id: str
    tool_id: str

    status: str

    attempts: int
    latency_ms: int

    metadata: dict
```

记录：

```text
resolution
validation
permission
execution
retry
fallback
latency
result
error
```

---

# 22. ToolRuntimeSpec

```python
class ToolRuntimeSpec:

    registry_policy
    skill_policy

    exposure_policy
    permission_policy

    timeout_policy
    retry_policy
    concurrency_policy
    fallback_policy

    lifecycle_policy
    observability_policy
```

配置：

```yaml
tool_runtime:

  metadata:
    id: default_tool_runtime
    version: "1.0"

  registry:
    namespace_required: true
    allow_alias: true

  skills:
    activation_strategy: dynamic
    max_active_skills: 8

  exposure:
    strategy: task_relevant
    max_visible_tools: 24

  execution:
    default_mode: async
    cancellation: true
    streaming: true

  timeout:
    default_seconds: 30

  retry:
    max_attempts: 2
    backoff: exponential

  concurrency:
    max_global_parallel: 8

  fallback:
    max_fallback_depth: 1

  lifecycle:
    auto_start_workbench: true

  observability:
    tracing: true
```

---

# 23. Runtime 装配

```text
tool_runtime.yaml
       ↓
ToolRuntimeSpec
       ↓
┌──────────────────────────────┐
│ Tool / Skill Runtime         │
│                              │
│ Tool Registry                │
│ Skill Registry               │
│ Adapter Layer                │
│ Workbench Manager            │
│ Execution Engine             │
│ Exposure Policy              │
│ Permission Policy            │
│ Lifecycle Manager            │
└──────────────────────────────┘
```

启动：

```text
RuntimeSpec
    ↓
Register Local Tools
    ↓
Load Skills
    ↓
Initialize Adapters
    ↓
Start Workbenches
    ↓
Discover Dynamic Tools
    ↓
Runtime Ready
```

---

# 24. 系统接口

```text
Agent Identity
      ↓
Capability Policy
      ↓
Tool Runtime
```

```text
Session / State
      ↓
Execution Context
      ↓
Tool Runtime
      ↓
ToolResult
```

```text
Tool Registry
      ↓
Exposure Policy
      ↓
Context Compiler
      ↓
Model Tool Schema
```

```text
Orchestration
      ↓
Tool Call
      ↓
Tool Runtime
      ↓
ToolResult
      ↓
Orchestration
```

---

# 25. 模块目录

```text
tools/
├── spec/
│   ├── tool
│   ├── skill
│   ├── result
│   ├── context
│   └── runtime
│
├── core/
│   ├── base_tool
│   ├── function_tool
│   └── stateful_tool
│
├── skills/
│   ├── skill
│   ├── activation
│   └── registry
│
├── registry/
│   ├── tool_registry
│   ├── namespace
│   └── discovery
│
├── execution/
│   ├── engine
│   ├── resolver
│   ├── validator
│   ├── executor
│   └── result_normalizer
│
├── workbench/
│   ├── base
│   ├── static
│   ├── dynamic
│   └── manager
│
├── adapters/
│   ├── function
│   ├── plugin
│   ├── mcp
│   ├── openapi
│   └── builtin
│
├── policy/
│   ├── exposure
│   ├── permission
│   ├── timeout
│   ├── retry
│   ├── concurrency
│   └── fallback
│
├── lifecycle/
├── state/
├── observability/
└── runtime/
```

最终职责：

```text
Tool
→ 定义原子能力

Skill
→ 组织能力集合

Tool Registry
→ 管理具体能力

Skill Registry
→ 管理能力组合

Adapter
→ 统一不同能力来源

Workbench
→ 管理共享资源与状态

Execution Engine
→ 执行 Tool Call

Exposure Policy
→ 决定模型看到什么 Tool

Permission Policy
→ 决定 Agent 能否执行 Tool

Execution Policy
→ 控制超时、重试、并行与 fallback

ToolResult
→ 统一执行结果

Lifecycle
→ 管理 Tool / Workbench 生命周期

Observability
→ 记录完整 Tool Call 路径

ToolRuntime
→ 装配并执行整个 Tool / Skill Runtime
```

最终主链：

```text
Task
 ↓
Skill Selection
 ↓
Tool Discovery
 ↓
Exposure
 ↓
Model Tool Schema
 ↓
Tool Call
 ↓
Resolve
 ↓
Authorize
 ↓
Validate
 ↓
Execute
 ↓
Normalize
 ↓
ToolResult
 ↓
Session / Context Compiler
 ↓
Agent Runtime
```