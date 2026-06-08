# nagent —— 类 Eino 的 Go Agent 框架实现规划

## 背景

Eino 是 CloudWeGo 团队出品的 Go 生态 LLM 应用开发框架，核心思想是：
- **组件抽象**：统一的接口（ChatModel、Tool、Retriever 等）
- **编排层**：Chain（线性）+ Graph（有向图，支持环路）
- **Agent 工具包（ADK）**：ReAct、Plan-Execute、Human-in-Loop 等高级模式

`nagent` 将借鉴 Eino 的三层架构，在 Go 1.26 generics 的基础上实现一个类似的框架。

---

## 整体架构（三层）

```
┌─────────────────────────────────────────────────┐
│              Layer 3: Agent Patterns             │
│   ReActAgent | PlanExecuteAgent | MultiAgent     │
├─────────────────────────────────────────────────┤
│              Layer 2: Orchestration              │
│        Chain (线性) | Graph (有向图/有环)          │
│   Runner(流式/并发执行) | StateManager            │
├─────────────────────────────────────────────────┤
│           Layer 1: Component Abstractions        │
│  ChatModel | Tool | Retriever | ChatTemplate     │
│  Message | ToolCall | StreamReader               │
└─────────────────────────────────────────────────┘
```

---

## 目录结构规划

```
nagent/
├── go.mod
│
├── component/              # Layer 1: 核心组件抽象
│   ├── message.go          # Message、Role、ToolCall 类型定义
│   ├── model.go            # ChatModel 接口
│   ├── tool.go             # Tool 接口 & ToolSet
│   ├── retriever.go        # Retriever 接口
│   ├── template.go         # ChatTemplate 接口
│   └── stream.go           # StreamReader（泛型流式接口）
│
├── compose/                # Layer 2: 编排层
│   ├── graph/
│   │   ├── graph.go        # Graph[I,O,S] 泛型图核心（支持环路）
│   │   ├── node.go         # NodeInfo、节点注册
│   │   ├── edge.go         # Edge、条件边（branch）
│   │   ├── runner.go       # Runner：编译图、并发执行
│   │   └── state.go        # State 管理（StateGraph）
│   ├── chain/
│   │   └── chain.go        # Chain[I,O]（线性 DSL，底层用 Graph）
│   └── workflow/
│       └── workflow.go     # Workflow：DAG + 字段级映射
│
├── agent/                  # Layer 3: Agent 模式
│   ├── react/
│   │   └── agent.go        # ReAct（思考→工具调用→观察 循环）
│   ├── plan_execute/
│   │   └── agent.go        # Plan→Execute→Replan 模式
│   └── multiagent/
│       └── orchestrator.go # 多 Agent 协调
│
├── provider/               # 内置 LLM 提供者
│   ├── openaicompat/
│   │   └── model.go        # 兼容 OpenAI API 的通用实现（DeepSeek 等直接复用）
│   ├── deepseek/
│   │   └── model.go        # DeepSeek（embed openaicompat，仅覆盖默认 BaseURL）
│   └── ollama/
│       └── model.go        # Ollama 本地模型（兼容 OpenAI API，同样复用 openaicompat）
│
├── memory/                 # 会话记忆
│   ├── memory.go           # Memory 接口
│   └── inmemory/
│       └── store.go        # 内存存储实现
│
├── callback/               # 可观测性
│   ├── handler.go          # CallbackHandler 接口
│   └── logging/
│       └── handler.go      # 日志回调
│
└── examples/               # 示例
    ├── simple_chat/
    ├── react_agent/
    └── multi_agent/
```

---

## 各层详细设计

### Layer 1：组件抽象（`component/`）

#### `message.go`
```go
type Role string
const (
    RoleSystem    Role = "system"
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
    RoleTool      Role = "tool"
)

type Message struct {
    Role       Role
    Content    string
    ToolCalls  []ToolCall
    ToolCallID string   // RoleTool 时使用
    Name       string
}

type ToolCall struct {
    ID        string
    Function  FunctionCall
}

type FunctionCall struct {
    Name      string
    Arguments string // JSON
}
```

#### `model.go`（ChatModel 接口）
```go
type ChatModel interface {
    Generate(ctx context.Context, msgs []Message, opts ...Option) (*Message, error)
    Stream(ctx context.Context, msgs []Message, opts ...Option) (*StreamReader[Message], error)
    BindTools(tools []Tool) ChatModel
}
```

#### `tool.go`（Tool 接口）
```go
type Tool interface {
    Info() ToolInfo       // 名称、描述、参数 JSON Schema
    Invoke(ctx context.Context, input string) (string, error)
}

// 类型安全的泛型工具包装
type TypedTool[I, O any] struct { ... }
```

#### `stream.go`（泛型流式）
```go
type StreamReader[T any] struct { ... }
func (r *StreamReader[T]) Recv() (T, error)
func (r *StreamReader[T]) Close()
```

---

### Layer 2：编排层（`compose/`）

#### Graph 核心（`compose/graph/graph.go`）

```go
// S = State 类型（有状态图用），I = 输入，O = 输出
type Graph[I, O any] struct {
    nodes map[string]*nodeWrapper
    edges map[string][]edgeRule
}

func NewGraph[I, O any]() *Graph[I, O]

// 添加普通函数节点
func (g *Graph[I, O]) AddNode(key string, fn func(ctx context.Context, in any) (any, error)) error

// 添加 ChatModel 节点
func (g *Graph[I, O]) AddChatModelNode(key string, model component.ChatModel) error

// 添加工具节点
func (g *Graph[I, O]) AddToolNode(key string, tools []component.Tool) error

// 普通边
func (g *Graph[I, O]) AddEdge(from, to string) error

// 条件边（分支路由）
func (g *Graph[I, O]) AddConditionalEdge(from string, router func(ctx context.Context, out any) (string, error), routes map[string]string) error

// 编译为可执行的 Runner
func (g *Graph[I, O]) Compile(ctx context.Context, opts ...CompileOption) (*Runner[I, O], error)
```

**关键设计**：
- `START` / `END` 为保留节点 key
- 图编译时做拓扑检查（检测死循环非法、孤立节点等）
- 支持 `StateGraph`：每个节点接收/修改共享 State，通过 `sync.Mutex` 保证并发安全

#### Runner（`compose/graph/runner.go`）

```go
type Runner[I, O any] struct { ... }

func (r *Runner[I, O]) Invoke(ctx context.Context, input I, opts ...Option) (O, error)
func (r *Runner[I, O]) Stream(ctx context.Context, input I, opts ...Option) (*component.StreamReader[O], error)
```

**执行策略**：
- BFS/拓扑排序决定执行顺序
- 无依赖关系的节点并发执行（`errgroup`）
- 支持 Interrupt/Resume（通过 ctx + channel）

#### Chain DSL（`compose/chain/chain.go`）

```go
chain := chain.New[[]Message, *Message]().
    Pipe(templateNode).
    Pipe(modelNode).
    Pipe(parserNode).
    Build()
```

底层直接复用 Graph，仅是更简单的 Builder API。

---

### Layer 3：Agent 模式（`agent/`）

#### ReAct Agent（`agent/react/agent.go`）

```go
type ReActAgent struct {
    model  component.ChatModel
    tools  component.ToolSet
    maxIter int
}

// 内部循环：
// 1. model.Generate(messages)
// 2. 如果有 ToolCall → 执行工具 → 把结果加入 messages → 回到 1
// 3. 没有 ToolCall → 返回最终答案
```

实现为 Graph 中的**带环**节点：
- `model_node` → 条件边 → `tool_node`（有 ToolCall）
- `model_node` → 条件边 → `END`（无 ToolCall）
- `tool_node` → `model_node`（循环）

---

## 实现阶段规划

| 阶段 | 内容 | 优先级 |
|------|------|--------|
| **Phase 1** | `component/`：Message、ChatModel、Tool、StreamReader 接口 | 🔴 必须 |
| **Phase 2** | `compose/graph/`：Graph/baseGraph、Runner（DAG + 条件边） | 🔴 必须 |
| **Phase 3** | `compose/chain/`：Chain DSL（基于 Graph） | 🔴 必须 |
| **Phase 4** | `provider/openaicompat/` + `provider/deepseek/` + `provider/ollama/` | 🔴 必须 |
| 🏁 **里程碑** | **跑通 DeepSeek + Ollama 多节点对话 Chain** | — |
| **Phase 5** | `agent/react/`：带环 Graph + ReAct Agent | 🟡 重要 |
| **Phase 6** | `memory/`：会话记忆接口 + 内存实现 | 🟡 重要 |
| **Phase 7** | `callback/`：可观测性回调（节点事件钩子） | 🟡 重要 |
| **Phase 8** | `compose/graph/state.go`：StateGraph（共享状态） | 🟢 可选 |
| **Phase 9** | `agent/plan_execute/`：Plan-Execute-Replan | 🟢 可选 |
| **Phase 10** | `examples/`：simple_chat、react_agent、multi_agent | 🟢 可选 |

---

## 开放问题（需要你确认）

> [!NOTE]
> **✅ 问题 1：Graph 泛型参数设计 — 已确认方案 A**
>
> `Graph[I, O]`（无状态）+ `StateGraph[I, O, S]`（有状态）分离暴露，
> 内部用 `baseGraph[I, O, S]` 共享核心逻辑（embed），避免重复代码。

> [!NOTE]
> **✅ 问题 2：流式输出粒度 — 已确认方案 B**
>
> `Runner.Stream()` 返回 `StreamReader[O]`，直接透传最后 ChatModel 节点的 token by token 输出。
> 节点进度/事件通过 `callback/` 层的钉子承接，不展宽 Stream API 表面。

> [!NOTE]
> **✅ 问题 3：优先实现的 Provider — 已确认**
>
> - `provider/openaicompat/`：兼容 OpenAI API 的通用底层实现
> - `provider/deepseek/`：embed openaicompat，仅覆盖 BaseURL，开箱即用
> - `provider/ollama/`：Ollama 本地模型，同样复用 openaicompat（Ollama 支持 OpenAI 兼容接口）

> [!NOTE]
> **✅ 问题 4：第一个里程碑 — 已确认方案 A**
>
> Phase 1~4：打通 `component` 接口 → `Graph` DAG 执行 → `Chain` DSL → DeepSeek/Ollama Provider，
> 跑通一个完整的多节点对话 Chain 作为里程碑。
> ReAct（带环路 Graph）作为 Phase 5 在里程碑之后自然递进。

---

## 技术选型

| 关注点 | 选型 | 理由 |
|--------|------|------|
| Go 版本 | 1.26 | 项目 go.mod 已指定，可用最新泛型特性 |
| HTTP 客户端 | `net/http` (标准库) | 无需引入依赖 |
| JSON | `encoding/json` | 标准库，配合 `go-json` 可选加速 |
| 并发 | `golang.org/x/sync/errgroup` | 节点并发执行 |
| 测试 | `testing` + `testify` | 标准 + 断言库 |

---

## 验证计划

### 自动化测试
- `go test ./component/...` — 接口单元测试
- `go test ./compose/...` — Graph 拓扑、Runner 执行逻辑
- `go test ./agent/...` — ReAct 循环单元测试（mock model）

### 手动验证
- 跑通 `examples/simple_chat`：构造 Chain，调用 OpenAI 返回一段对话
- 跑通 `examples/react_agent`：ReAct Agent 使用「计算器」工具完成数学题
