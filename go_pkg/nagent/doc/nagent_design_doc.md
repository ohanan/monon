# nagent 技术设计文档

> **项目**：`github.com/ohanan/monon/go_pkg/nagent`
> **Go 版本**：1.26
> **参考**：[CloudWeGo Eino](https://github.com/cloudwego/eino)
> **更新**：2026-06-08

---

## 1. 背景与目标

`nagent` 是一个面向 Go 生态的 LLM Agent 开发框架，设计理念参考 Eino，核心目标：

- **简单事简单做**：用几行 Chain DSL 就能接入 LLM 对话
- **复杂事可实现**：用 Graph 构建有状态、有环路的 Agent 工作流
- **类型安全**：基于 Go 1.26 泛型，在编译期捕获类型错误
- **可扩展**：Provider / Tool / Memory 均面向接口，随时替换实现

---

## 2. 整体架构

```
┌────────────────────────────────────────────────────────┐
│                 Layer 3 · Agent Patterns               │
│         ReActAgent │ PlanExecuteAgent │ MultiAgent      │
├────────────────────────────────────────────────────────┤
│                 Layer 2 · Orchestration                │
│   Chain (线性 DSL)  │  Graph (有向图/有环)              │
│   Runner (token 级流式 + 并发执行)  │  StateGraph       │
├────────────────────────────────────────────────────────┤
│              Layer 1 · Component Abstractions          │
│  ChatModel │ Tool │ Retriever │ ChatTemplate           │
│  Message │ ToolCall │ StreamReader[T]                  │
└────────────────────────────────────────────────────────┘
        ↑ provider/    ↑ memory/    ↑ callback/
   （具体实现）    （会话记忆）    （可观测性）
```

---

## 3. 目录结构

```
nagent/
├── go.mod
│
├── component/                  # Layer 1：核心抽象接口
│   ├── message.go              # Message、Role、ToolCall、FunctionCall
│   ├── model.go                # ChatModel 接口
│   ├── tool.go                 # Tool 接口、ToolInfo、ToolSet、TypedTool[I,O]
│   ├── retriever.go            # Retriever 接口
│   ├── template.go             # ChatTemplate 接口
│   └── stream.go               # StreamReader[T]（泛型 token 流）
│
├── compose/                    # Layer 2：编排层
│   ├── graph/
│   │   ├── graph.go            # Graph[I,O]（无状态有向图，支持环路）
│   │   ├── state_graph.go      # StateGraph[I,O,S]（共享状态图）
│   │   ├── base_graph.go       # baseGraph[I,O,S]（内部共享核心逻辑）
│   │   ├── node.go             # nodeWrapper、节点类型枚举
│   │   ├── edge.go             # edgeRule、条件边路由
│   │   └── runner.go           # Runner[I,O]（Invoke + Stream）
│   └── chain/
│       └── chain.go            # Chain[I,O] Builder DSL（底层复用 Graph）
│
├── agent/                      # Layer 3：Agent 模式
│   ├── react/
│   │   └── agent.go            # ReActAgent（带环 Graph 实现）
│   └── plan_execute/
│       └── agent.go            # PlanExecuteAgent（Plan→Execute→Replan）
│
├── provider/                   # LLM 提供者（具体实现）
│   ├── openaicompat/
│   │   └── model.go            # 通用 OpenAI 兼容实现（HTTP + SSE）
│   ├── deepseek/
│   │   └── model.go            # DeepSeek（embed openaicompat，覆盖 BaseURL）
│   └── ollama/
│       └── model.go            # Ollama 本地（embed openaicompat，覆盖 BaseURL）
│
├── memory/                     # 会话记忆
│   ├── memory.go               # Memory 接口
│   └── inmemory/
│       └── store.go            # 线程安全内存存储
│
├── callback/                   # 可观测性
│   ├── handler.go              # CallbackHandler 接口（节点事件钩子）
│   └── logging/
│       └── handler.go          # 标准日志回调实现
│
└── examples/
    ├── simple_chat/            # Chain + DeepSeek 对话示例
    ├── react_agent/            # ReAct + 工具调用示例
    └── multi_agent/            # 多 Agent 协同示例
```

---

## 4. Layer 1：组件抽象（`component/`）

### 4.1 消息类型（`message.go`）

```go
type Role string

const (
    RoleSystem    Role = "system"
    RoleUser      Role = "user"
    RoleAssistant Role = "assistant"
    RoleTool      Role = "tool"
)

// Message 是框架内所有节点流转的核心数据单元
type Message struct {
    Role       Role
    Content    string
    ToolCalls  []ToolCall  // 模型请求工具调用时非空
    ToolCallID string      // Role == RoleTool 时标识对应的 ToolCall.ID
    Name       string      // 工具名（Role == RoleTool 时使用）
}

type ToolCall struct {
    ID       string
    Function FunctionCall
}

type FunctionCall struct {
    Name      string
    Arguments string // JSON 字符串
}
```

### 4.2 ChatModel 接口（`model.go`）

```go
// Option 用于运行时覆盖模型配置（温度、MaxTokens 等）
type Option func(*ModelConfig)

type ChatModel interface {
    // Generate 同步调用，返回完整回复
    Generate(ctx context.Context, msgs []Message, opts ...Option) (*Message, error)

    // Stream 流式调用，返回 token by token 的 StreamReader
    Stream(ctx context.Context, msgs []Message, opts ...Option) (*StreamReader[Message], error)

    // BindTools 返回绑定了工具集的新 ChatModel 实例（不修改原实例）
    BindTools(tools []Tool) ChatModel
}

// ModelConfig 汇聚所有模型运行时参数
type ModelConfig struct {
    Temperature *float64
    MaxTokens   *int
    TopP        *float64
    Stop        []string
}
```

### 4.3 Tool 接口（`tool.go`）

```go
// ToolInfo 描述工具的元信息，用于生成 LLM 的 function_calling schema
type ToolInfo struct {
    Name        string
    Description string
    ParamSchema []byte // JSON Schema（encoding/json 兼容）
}

type Tool interface {
    Info() ToolInfo
    Invoke(ctx context.Context, input string) (string, error)
}

// ToolSet 管理一组工具，支持按名称分发
type ToolSet struct {
    tools map[string]Tool
}

func NewToolSet(tools ...Tool) *ToolSet
func (ts *ToolSet) Invoke(ctx context.Context, call ToolCall) (string, error)
func (ts *ToolSet) Infos() []ToolInfo

// TypedTool 是类型安全的泛型工具包装，自动处理 JSON 序列化
// I：输入结构体（自动生成 JSON Schema）
// O：输出结构体（自动序列化为字符串返回）
type TypedTool[I, O any] struct {
    name        string
    description string
    fn          func(ctx context.Context, input I) (O, error)
}

func NewTypedTool[I, O any](name, desc string, fn func(ctx context.Context, input I) (O, error)) *TypedTool[I, O]
```

### 4.4 流式接口（`stream.go`）

```go
// StreamReader[T] 封装一个只读的泛型 channel，提供安全的 Recv/Close 语义
type StreamReader[T any] struct {
    ch  <-chan streamChunk[T]
    err error
}

type streamChunk[T any] struct {
    value T
    err   error
}

func NewStreamReader[T any](ch <-chan streamChunk[T]) *StreamReader[T]

// Recv 读取下一个 chunk；返回 io.EOF 表示流结束
func (r *StreamReader[T]) Recv() (T, error)

// Close 主动关闭，释放上游资源
func (r *StreamReader[T]) Close()
```

---

## 5. Layer 2：编排层（`compose/`）

### 5.1 内部共享核心（`base_graph.go`）

> `Graph[I,O]` 和 `StateGraph[I,O,S]` 的所有图结构逻辑均委托给 `baseGraph`，两者对外暴露不同的 Builder API。

```go
// 内部包级私有类型，不对外暴露
type baseGraph[I, O, S any] struct {
    nodes map[string]*nodeWrapper[S] // key → 节点
    edges map[string][]edgeRule      // from → 路由规则列表
    mu    sync.RWMutex
}

// 保留节点 key 常量
const (
    NodeStart = "__START__"
    NodeEnd   = "__END__"
)
```

### 5.2 无状态图（`graph.go`）

```go
// Graph[I,O] 适用于无共享状态的有向图（含环路）
// 节点之间通过输入/输出类型直接传递数据
type Graph[I, O any] struct {
    base *baseGraph[I, O, struct{}]
}

func NewGraph[I, O any]() *Graph[I, O]

// AddLambdaNode 添加普通函数节点
func (g *Graph[I, O]) AddLambdaNode(
    key string,
    fn func(ctx context.Context, in any) (any, error),
    opts ...NodeOption,
) error

// AddChatModelNode 添加 ChatModel 节点
// 该节点自动处理 Stream 模式下的 token 透传
func (g *Graph[I, O]) AddChatModelNode(
    key string,
    model component.ChatModel,
    opts ...NodeOption,
) error

// AddToolNode 添加工具执行节点
// 接收 []ToolCall，返回 []Message (role=tool)
func (g *Graph[I, O]) AddToolNode(
    key string,
    tools *component.ToolSet,
    opts ...NodeOption,
) error

// AddEdge 添加无条件边
func (g *Graph[I, O]) AddEdge(from, to string) error

// AddConditionalEdge 添加条件边
// router 返回路由 key，routes 映射 key → 目标节点
func (g *Graph[I, O]) AddConditionalEdge(
    from string,
    router func(ctx context.Context, out any) (string, error),
    routes map[string]string,
) error

// Compile 做静态拓扑分析并返回可执行的 Runner
func (g *Graph[I, O]) Compile(opts ...CompileOption) (*Runner[I, O], error)
```

**编译期检查项**：
- 所有节点均可从 `START` 到达
- `END` 节点可被到达
- 无孤立节点（无入边且非 START）
- 条件边的所有路由目标均已注册

### 5.3 有状态图（`state_graph.go`）

```go
// StateGraph[I,O,S] 适用于需要共享状态的场景（如多轮对话记忆、累积上下文）
// S 是用户自定义的状态结构体，在所有节点之间共享
type StateGraph[I, O, S any] struct {
    base         *baseGraph[I, O, S]
    stateFactory func() *S // 每次 Invoke 时创建新的 State 实例
}

func NewStateGraph[I, O, S any](stateFactory func() *S) *StateGraph[I, O, S]

// AddStateNode 节点函数额外接收 *S，可读写共享状态
// 并发节点访问 State 时通过 baseGraph 内的 sync.Mutex 保护
func (g *StateGraph[I, O, S]) AddStateNode(
    key string,
    fn func(ctx context.Context, in any, state *S) (any, error),
    opts ...NodeOption,
) error

// 其余 AddEdge / AddConditionalEdge / Compile 与 Graph[I,O] 相同
```

### 5.4 Runner（`runner.go`）

```go
type Runner[I, O any] struct {
    compiled *compiledGraph // 编译后的拓扑快照（不可变）
    opts     runnerOptions
}

// Invoke 同步执行，阻塞直到 END 节点完成
func (r *Runner[I, O]) Invoke(
    ctx context.Context,
    input I,
    opts ...RunOption,
) (O, error)

// Stream 流式执行
// - 非 ChatModel 节点：同步执行，完成后将结果传入下一节点
// - ChatModel 节点（最终输出节点）：token by token 直接写入返回的 StreamReader
func (r *Runner[I, O]) Stream(
    ctx context.Context,
    input I,
    opts ...RunOption,
) (*component.StreamReader[O], error)
```

**执行引擎策略**：

```
1. 图编译时生成"执行计划"（拓扑排序 + 并发组）
2. Invoke 时：
   - 按并发组依次调度，同组内节点用 errgroup 并发执行
   - 条件边在节点完成后评估，动态决定下一跳
   - 有环路时，循环节点重新入队，直到路由到 END
3. Stream 时：
   - 前置节点与 Invoke 一致（同步等待）
   - 最后的 ChatModel 节点调用 model.Stream()，将 channel tee 给：
     ① 内部聚合器（供后续节点使用，如有）
     ② 外部 StreamReader（供调用方消费）
```

### 5.5 Chain DSL（`chain/chain.go`）

Chain 是 Graph 的语法糖，专为线性管道设计。

```go
// Chain[I,O] 封装一个线性 Graph，提供流式 Builder API
type Chain[I, O any] struct {
    graph   *graph.Graph[any, any] // 内部图
    lastKey string
    seq     int
}

func New[I, O any]() *Chain[I, O]

// Pipe 追加一个处理节点（类型在运行时检查，编译期由泛型保障出入类型）
func (c *Chain[I, O]) Pipe(node ChainNode) *Chain[I, O]

// AppendChatModel 追加 ChatModel 节点（shorthand）
func (c *Chain[I, O]) AppendChatModel(model component.ChatModel) *Chain[I, O]

// Build 编译为 Runner，失败时 panic（Chain 场景下配置错误属于编程错误）
func (c *Chain[I, O]) Build() *graph.Runner[I, O]

// MustBuild 同 Build，返回 error 供调用方决策
func (c *Chain[I, O]) MustBuild() (*graph.Runner[I, O], error)
```

**使用示例**：

```go
runner := chain.New[[]component.Message, *component.Message]().
    AppendChatModel(deepseekModel).
    Build()

resp, err := runner.Invoke(ctx, []component.Message{
    {Role: component.RoleUser, Content: "你好，请介绍一下你自己"},
})
```

---

## 6. Layer 3：Agent 模式（`agent/`）

### 6.1 ReAct Agent（`agent/react/agent.go`）

ReAct（Reason + Act）是最常用的 Agent 模式，底层用带环 Graph 实现。

**执行流程**：

```
         ┌─────────────────────────────────────┐
         ↓                                     │
[START] → [model_node] → 有 ToolCall? → Yes → [tool_node]
                        ↓
                        No
                        ↓
                      [END]
```

```go
type Config struct {
    Model    component.ChatModel
    Tools    *component.ToolSet
    MaxIter  int    // 最大工具调用轮次，防止死循环，默认 10
    SystemPrompt string
}

type ReActAgent struct {
    runner *graph.Runner[[]component.Message, *component.Message]
}

func NewReActAgent(cfg Config) (*ReActAgent, error)

func (a *ReActAgent) Run(
    ctx context.Context,
    input string,
    opts ...graph.RunOption,
) (*component.Message, error)

func (a *ReActAgent) Stream(
    ctx context.Context,
    input string,
    opts ...graph.RunOption,
) (*component.StreamReader[component.Message], error)
```

**内部 Graph 构建（伪代码）**：

```go
func buildReActGraph(cfg Config) *graph.Graph[[]Message, *Message] {
    g := graph.NewGraph[[]Message, *Message]()

    modelWithTools := cfg.Model.BindTools(cfg.Tools)

    g.AddChatModelNode("model", modelWithTools)
    g.AddToolNode("tools", cfg.Tools)

    g.AddEdge(graph.NodeStart, "model")

    g.AddConditionalEdge("model",
        func(ctx context.Context, out any) (string, error) {
            msg := out.(*Message)
            if len(msg.ToolCalls) > 0 {
                return "has_tool_call", nil
            }
            return "finish", nil
        },
        map[string]string{
            "has_tool_call": "tools",
            "finish":        graph.NodeEnd,
        },
    )

    g.AddEdge("tools", "model") // 形成环路
    return g
}
```

---

## 7. Provider 层（`provider/`）

### 7.1 设计策略：openaicompat 作为通用底层

由于 DeepSeek 和 Ollama 均兼容 OpenAI API 格式，三个 Provider 共享同一套 HTTP + SSE 实现：

```
provider/openaicompat/model.go   ← 通用底层
       ↑ embed              ↑ embed
provider/deepseek/model.go    provider/ollama/model.go
（仅覆盖 BaseURL + 默认 Model）  （仅覆盖 BaseURL）
```

### 7.2 openaicompat 实现要点（`provider/openaicompat/model.go`）

```go
type Config struct {
    BaseURL string   // e.g. "https://api.deepseek.com/v1"
    APIKey  string
    Model   string   // 默认模型名
    Timeout time.Duration
}

type Model struct {
    cfg    Config
    client *http.Client
    tools  []component.ToolInfo // BindTools 后持有
}

func NewModel(cfg Config) *Model

// Generate 调用 /chat/completions（stream=false）
func (m *Model) Generate(ctx context.Context, msgs []component.Message, opts ...component.Option) (*component.Message, error)

// Stream 调用 /chat/completions（stream=true），解析 SSE data: {...} 帧
// 每个 SSE 帧中的 delta.content 作为一个 token 写入 StreamReader
func (m *Model) Stream(ctx context.Context, msgs []component.Message, opts ...component.Option) (*component.StreamReader[component.Message], error)

// BindTools 返回携带工具信息的新实例（不可变设计）
func (m *Model) BindTools(tools []component.Tool) component.ChatModel
```

**SSE 解析流程**：

```
HTTP Response Body (stream=true)
    │
    ├─ "data: {"choices":[{"delta":{"content":"Hello"}}]}\n\n"
    ├─ "data: {"choices":[{"delta":{"content":" world"}}]}\n\n"
    ├─ "data: [DONE]\n\n"
    │
    ↓ goroutine 逐行读取，写入 chan streamChunk[Message]
    ↓
StreamReader[Message].Recv() → 逐 token 返回
```

### 7.3 DeepSeek Provider（`provider/deepseek/model.go`）

```go
const (
    DefaultBaseURL = "https://api.deepseek.com/v1"
    DefaultModel   = "deepseek-chat"
)

type Config struct {
    openaicompat.Config        // embed，直接复用所有配置字段
}

func NewModel(apiKey string, overrides ...func(*Config)) *openaicompat.Model {
    cfg := Config{
        Config: openaicompat.Config{
            BaseURL: DefaultBaseURL,
            Model:   DefaultModel,
            APIKey:  apiKey,
        },
    }
    for _, o := range overrides {
        o(&cfg)
    }
    return openaicompat.NewModel(cfg.Config)
}
```

### 7.4 Ollama Provider（`provider/ollama/model.go`）

```go
const (
    DefaultBaseURL = "http://localhost:11434/v1" // Ollama OpenAI 兼容端点
)

func NewModel(modelName string, overrides ...func(*openaicompat.Config)) *openaicompat.Model {
    cfg := openaicompat.Config{
        BaseURL: DefaultBaseURL,
        APIKey:  "ollama", // Ollama 不校验，随意填
        Model:   modelName,
    }
    for _, o := range overrides {
        o(&cfg)
    }
    return openaicompat.NewModel(cfg)
}
```

---

## 8. 内存与回调

### 8.1 Memory 接口（`memory/memory.go`）

```go
type Memory interface {
    // Get 读取指定 session 的历史消息
    Get(ctx context.Context, sessionID string) ([]component.Message, error)

    // Append 追加新消息到指定 session
    Append(ctx context.Context, sessionID string, msgs ...component.Message) error

    // Clear 清空指定 session
    Clear(ctx context.Context, sessionID string) error
}
```

### 8.2 Callback 接口（`callback/handler.go`）

> Callback 是节点进度/事件的观测点，替代"节点级流式"（方案 B 决策）。

```go
type Event struct {
    NodeKey   string
    Type      EventType  // NodeStart | NodeEnd | NodeError
    Input     any
    Output    any
    Error     error
    Timestamp time.Time
    Duration  time.Duration // NodeEnd 时有效
}

type EventType int
const (
    EventNodeStart EventType = iota
    EventNodeEnd
    EventNodeError
)

type CallbackHandler interface {
    OnEvent(ctx context.Context, event Event)
}
```

**在 Runner 中注入 Callback**：

```go
runner.Invoke(ctx, input,
    graph.WithCallbacks(loggingHandler, tracingHandler),
)
```

---

## 9. 关键设计决策汇总

| 决策点 | 结论 | 理由 |
|--------|------|------|
| Graph 泛型 | `Graph[I,O]` + `StateGraph[I,O,S]` 分离 | API 清晰，内部 embed baseGraph 避免重复 |
| 流式粒度 | Token 级（透传最后节点的 SSE） | 用户体验优先；节点事件走 callback 层 |
| Provider 基础 | `openaicompat` 通用底层 | DeepSeek / Ollama 均兼容 OpenAI API，复用无额外成本 |
| 里程碑顺序 | Chain 跑通 → 再做 ReAct | 先夯实 DAG 执行，再引入有环图复杂度 |
| 工具类型安全 | `TypedTool[I,O]` 泛型包装 | 自动 JSON Schema 生成 + 反序列化，消除手写胶水代码 |
| BindTools | 返回新实例（不可变） | 线程安全，同一 Model 可绑定不同工具集并发使用 |

---

## 10. 实现阶段规划

| 阶段 | 内容 | 优先级 |
|------|------|--------|
| **Phase 1** | `component/`：所有接口 + 基础类型 | 🔴 必须 |
| **Phase 2** | `compose/graph/`：baseGraph、Graph、Runner（DAG + 条件边） | 🔴 必须 |
| **Phase 3** | `compose/chain/`：Chain Builder DSL | 🔴 必须 |
| **Phase 4** | `provider/openaicompat/` + `deepseek/` + `ollama/` | 🔴 必须 |
| 🏁 **里程碑** | **跑通 DeepSeek + Ollama 多节点对话 Chain** | — |
| **Phase 5** | `agent/react/`：带环 Graph + ReAct Agent | 🟡 重要 |
| **Phase 6** | `memory/inmemory/`：会话记忆 | 🟡 重要 |
| **Phase 7** | `callback/`：节点事件回调 + 日志实现 | 🟡 重要 |
| **Phase 8** | `compose/graph/state_graph.go`：StateGraph | 🟢 可选 |
| **Phase 9** | `agent/plan_execute/`：Plan-Execute-Replan | 🟢 可选 |
| **Phase 10** | `examples/`：three 个完整示例 | 🟢 可选 |

---

## 11. 技术选型

| 关注点 | 选型 | 理由 |
|--------|------|------|
| Go 版本 | 1.26 | 项目 go.mod 已指定 |
| HTTP 客户端 | `net/http`（标准库） | 无需引入第三方依赖 |
| SSE 解析 | 手写（`bufio.Scanner`） | SSE 协议简单，避免依赖 |
| JSON | `encoding/json` | 标准库，后期可换 `go-json` 加速 |
| 并发调度 | `golang.org/x/sync/errgroup` | 节点并发执行 + 错误聚合 |
| 测试 | `testing` + `github.com/stretchr/testify` | 断言 + Mock |
| 日志 | `log/slog`（标准库） | Go 1.21+ 结构化日志 |

---

## 12. 验证计划

### 自动化测试

```bash
go test ./component/...      # 接口契约单元测试
go test ./compose/graph/...  # 图拓扑分析 + Runner 执行
go test ./compose/chain/...  # Chain Builder + 线性流转
go test ./agent/react/...    # ReAct 循环（mock ChatModel）
go test ./provider/...       # Provider HTTP（httptest mock server）
```

### 手动里程碑验证

**里程碑（Phase 1~4）**：
```bash
cd examples/simple_chat
DEEPSEEK_API_KEY=sk-xxx go run .
# 期望：打印流式对话回复，token by token 输出到 stdout
```

**Phase 5 验证**：
```bash
cd examples/react_agent
DEEPSEEK_API_KEY=sk-xxx go run .
# 期望：Agent 调用"计算器"工具完成数学题，输出中间思考步骤
```
