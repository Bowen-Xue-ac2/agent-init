# Agent Identity Design

```text
AgentIdentity
│
├── identity_core          # Agent 是谁：稳定身份
│   ├── role
│   ├── mission
│   ├── responsibilities
│   └── behavioral_rules
│
├── persona                # Agent 如何表现：可演化身份
│   ├── working_style
│   ├── preferences
│   ├── learned_traits
│   └── relationship_context
│
├── prompt_policy          # 如何将身份编译进模型上下文
│   ├── system_preset
│   ├── version
│   └── variant
│
├── model_policy           # Agent 使用什么模型运行
│   ├── model
│   ├── reasoning
│   └── generation_config
│
├── capability_policy      # Agent 被允许做什么
│   ├── tools
│   ├── delegation
│   └── permissions
│
└── metadata               # Identity 管理信息
    ├── id
    ├── name
    ├── version
    └── tags
```

## 1. Identity Core

稳定身份定义，不随普通对话自动变化。

```python
class IdentityCore:
    role: str
    mission: str
    responsibilities: list[str]
    behavioral_rules: list[str]
```

功能：

```text
role
→ Agent 的专业角色 / 职能定位

mission
→ Agent 长期目标，替代 CrewAI 的 goal

responsibilities
→ Agent 明确承担的职责范围

behavioral_rules
→ 工作方式、判断原则、输出原则
```

示例：

```yaml
identity_core:
  role: Senior Financial Research Agent

  mission: >
    Research financial markets and produce evidence-based analysis.

  responsibilities:
    - collect information
    - validate sources
    - compare conflicting evidence
    - produce structured conclusions

  behavioral_rules:
    - distinguish facts from inference
    - preserve source provenance
    - prefer primary sources
```

---

## 2. Persona

用于存放可随着长期交互逐渐形成的身份状态。

对应 Letta 的 `persona` 思路。

```python
class Persona:
    working_style: list[str]
    preferences: dict[str, str]
    learned_traits: list[str]
    relationship_context: dict[str, str]
```

示例：

```yaml
persona:
  working_style:
    - concise
    - evidence-first
    - structured

  preferences:
    explanation_depth: medium
    output_style: engineering

  learned_traits:
    - prefers architecture before implementation

  relationship_context:
    user_role: project_owner
```

生命周期：

```text
初始 Persona
    ↓
Runtime Interaction
    ↓
Memory Event
    ↓
Persona Update Candidate
    ↓
Memory Conflict / Merge
    ↓
Updated Persona
```

`Identity Core` 与 `Persona` 分离：

```text
Identity Core
= 稳定定义

Persona
= 长期运行形成的动态身份
```

---

## 3. Prompt Policy

Identity 本身不直接等于 System Prompt。

Prompt Policy 决定如何把 Identity 编译成模型可消费的上下文。

```python
class PromptPolicy:
    system_preset: str
    version: str
    variant: str | None
```

示例：

```yaml
prompt_policy:
  system_preset: research_agent
  version: v1
  variant: standard
```

支持：

```text
research_agent + standard
research_agent + coding
research_agent + long_context
research_agent + subagent
```

对应运行链：

```text
Identity
+
PromptPolicy
+
Runtime Context
        ↓
Context Compiler
        ↓
System Context
```

---

## 4. Identity Compiler

专门负责将结构化 Identity 编译成 Prompt。

```python
class IdentityCompiler:

    def compile(
        self,
        identity_core,
        persona,
    ) -> str:

        return render(
            role=identity_core.role,
            mission=identity_core.mission,
            responsibilities=identity_core.responsibilities,
            behavioral_rules=identity_core.behavioral_rules,
            persona=persona,
        )
```

输出结构：

```text
<identity>
Role:
...

Mission:
...

Responsibilities:
...

Behavioral Rules:
...

Current Persona:
...
</identity>
```

进入完整 Context Compiler：

```text
System Constitution
        +
Agent Identity
        +
Persona
        +
Relevant Memory
        +
Knowledge
        +
Task State
        +
Tool Context
        ↓
Context Compiler
        ↓
LLM Messages
```

---

## 5. Model Policy

模型属于运行策略，不属于人格文本。

```python
class ModelPolicy:
    model: str
    reasoning: str | None
    temperature: float | None
    max_tokens: int | None
```

示例：

```yaml
model_policy:
  model: gpt-5.6
  reasoning: high
  temperature: 0.2
```

运行链：

```text
AgentSpec.model_policy
        ↓
ModelRouter
        ↓
Provider / Model
        ↓
LLM Runtime
```

允许后续实现：

```text
Agent A → GPT
Agent B → Claude
Agent C → Local Model
```

而不改变 Identity。

---

## 6. Capability Policy

描述 Agent 实际拥有的 Runtime 权限。

```python
class CapabilityPolicy:
    tools: list[str]
    delegation: DelegationPolicy
    permissions: list[str]
```

### Delegation

```python
class DelegationPolicy:
    enabled: bool
    allowed_agents: list[str]
    max_depth: int
```

示例：

```yaml
capability_policy:

  tools:
    - web_search
    - knowledge_search

  permissions:
    - knowledge.read
    - memory.read

  delegation:
    enabled: true
    allowed_agents:
      - data_agent
      - coding_agent
    max_depth: 2
```

Runtime 转换：

```text
delegation.enabled
        ↓
ToolResolver
        ↓
DelegateAgentTool
        ↓
Runtime.invoke(target_agent)
```

因此：

```text
Prompt：
“你可以委托任务”

不负责权限控制。

Runtime：
allowed_agents
max_depth
permissions

负责真正的执行控制。
```

---

## 7. Metadata

Identity 本身必须可寻址、版本化和注册。

```python
class IdentityMetadata:
    id: str
    name: str
    version: str
    tags: list[str]
```

示例：

```yaml
metadata:
  id: financial_researcher
  name: Financial Researcher
  version: "1.0"
  tags:
    - finance
    - research
```

Runtime 使用：

```text
AgentRegistry
    │
    ├── financial_researcher:v1
    ├── coding_agent:v3
    └── planner:v2
```

---

# 8. 完整 AgentSpec

最终建议不要单独创建巨型 `Identity` 类，而是：

```python
class AgentSpec:
    metadata: IdentityMetadata

    identity: IdentityCore
    persona: Persona

    prompt_policy: PromptPolicy
    model_policy: ModelPolicy
    capability_policy: CapabilityPolicy
```

完整配置：

```yaml
agent:

  metadata:
    id: financial_researcher
    name: Financial Researcher
    version: "1.0"
    tags:
      - finance
      - research

  identity:
    role: Senior Financial Research Agent

    mission: >
      Produce reliable financial research based on
      validated evidence.

    responsibilities:
      - research
      - evidence validation
      - synthesis

    behavioral_rules:
      - distinguish fact from inference
      - preserve source provenance
      - report conflicting evidence

  persona:
    working_style:
      - concise
      - structured
      - evidence-first

    preferences: {}

    learned_traits: []

    relationship_context: {}

  prompt_policy:
    system_preset: financial_research
    version: v1
    variant: standard

  model_policy:
    model: gpt-5.6
    reasoning: high

  capability_policy:
    tools:
      - web_search
      - knowledge_search

    permissions:
      - memory.read
      - knowledge.read

    delegation:
      enabled: true
      allowed_agents:
        - data_agent
      max_depth: 2
```

# 9. Runtime 装配流程

```text
agent.yaml
    ↓
AgentSpec Loader
    ↓
Schema Validation
    ↓
AgentRegistry.register()
    ↓
┌─────────────────────────────┐
│ Runtime Agent Instance      │
│                             │
│ Identity                    │
│ Persona                     │
│ Model Binding               │
│ Capability Binding          │
└─────────────────────────────┘
```

执行一轮：

```text
Runtime.invoke(agent_id, task)
        │
        ├── AgentRegistry.get()
        │
        ├── Memory.retrieve(persona/profile)
        │
        ├── ModelRouter.resolve()
        │
        ├── ToolResolver.resolve()
        │
        ↓
ContextCompiler
        │
        ├── system preset
        ├── identity
        ├── current persona
        ├── memory
        ├── task state
        └── tool definitions
        ↓
LLM
        ↓
Tool / Delegation Loop
        ↓
Result
        ↓
Memory / Persona Update Events
```

# 10. Identity 生命周期

```text
DEFINE
AgentSpec 定义
    ↓
REGISTER
注册到 AgentRegistry
    ↓
COMPILE
Identity → Context
    ↓
EXECUTE
Runtime 执行
    ↓
OBSERVE
记录行为与事件
    ↓
UPDATE
更新 Persona
    ↓
VERSION
必要时升级 Identity Core Version
```

两种更新路径必须分开：

```text
普通长期交互
        ↓
Persona Update

Agent 职责 / Mission / 权限发生变化
        ↓
AgentSpec Version Update
```

# 11. 与 Memory System 的接口

Identity 不承担完整记忆功能。

只保留与身份相关的长期状态：

```text
Identity System
    │
    └── Persona

Memory System
    ├── working
    ├── episodic
    ├── semantic
    ├── procedural
    └── profile
```

推荐关系：

```text
Memory events
      ↓
Identity-related extraction
      ↓
Persona Candidate
      ↓
Conflict / Importance / Confidence
      ↓
Persona Store
```

因此：

```text
“用户喜欢表格”
→ profile memory

“Agent逐渐形成先验证来源再分析的工作习惯”
→ persona

“Agent必须进行来源验证”
→ identity.behavioral_rules
```

# 12. 模块边界

最终建议形成：

```text
agent/
├── spec
│   ├── identity
│   ├── persona
│   ├── model_policy
│   ├── prompt_policy
│   └── capability_policy
│
├── registry
│
├── factory
│
└── lifecycle

context/
└── identity_compiler

prompt/
└── prompt_registry

model/
└── model_router

capability/
├── tool_resolver
└── delegation_runtime

memory/
└── persona_store
```

最终职责边界：

```text
AgentSpec
→ 定义 Agent

AgentRegistry
→ 找到 Agent

AgentFactory
→ 创建 Agent

IdentityCompiler
→ 把身份变成上下文

PromptRegistry
→ 管理 system prompt

ModelRouter
→ 选择模型

CapabilityRuntime
→ 控制工具 / delegation

PersonaStore
→ 保存可演化身份

Runtime
→ 把所有模块组合并执行
```