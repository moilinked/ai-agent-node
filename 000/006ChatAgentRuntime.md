# 关键技术栈入门

> 目标：自主研发一个最基础、能真正跑起来的 Chat Agent。  
> 不需要一开始就做 Multi-Agent、MCP、RAG、长期记忆——先把最小闭环做出来就够了。

## 先明确一点

**自研 Chat Agent ≠ 自研大模型。**

最现实的起点是：使用现成 LLM API，自己研发 Agent 的会话、状态、工具调用、Prompt、执行循环和应用层。

一个最基础的架构可以只有这些：

```text
┌─────────────┐
│   Chat UI   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Chat Server │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│     Agent Runtime   │
│                     │
│ System Prompt       │
│ Conversation State  │
│ Tool Calling        │
│ Agent Loop          │
└──────┬──────────────┘
       │
       ▼
┌─────────────┐
│     LLM     │
└─────────────┘
```

## 最少需要准备什么

| 模块          | MVP 阶段需要什么                    | 推荐程度 |
| ------------- | ----------------------------------- | -------- |
| LLM           | 一个支持 Chat / Tool Calling 的模型 | 必须     |
| Backend       | Python 或 TypeScript                | 必须     |
| Agent Runtime | 自己写一个简单 Agent Loop           | 必须     |
| Prompt        | System Prompt                       | 必须     |
| Conversation  | 保存最近的聊天历史                  | 必须     |
| UI            | 一个简单聊天页面                    | 推荐     |
| Tool          | 先做 1～2 个工具                    | 推荐     |
| Database      | SQLite / PostgreSQL                 | 后加     |
| RAG           | 暂时不用                            | 后加     |
| Memory        | 暂时只做 Session Memory             | 后加     |
| Multi-Agent   | 不需要                              | 暂时别做 |
| MCP           | 不需要                              | 暂时别做 |

真正的核心代码，其实就是这个循环：

```text
while True:

    # 1. 把当前聊天历史交给模型
    response = llm(messages, tools)

    # 2. 模型直接回答
    if response.type == "message":
        return response.text

    # 3. 模型想调用工具
    if response.type == "tool_call":
        result = execute_tool(
            response.tool_name,
            response.arguments
        )

        # 4. 把工具结果重新交给模型
        messages.append(response)
        messages.append(result)
```

这几十行逻辑，本质上已经是一个最基础的 Agent Runtime。

---

## 第一版建议只实现 6 个能力

### 1. Chat

用户发送：

```text
北京有什么好玩的？
```

Agent 能正常回复。

### 2. 上下文对话

用户：

```text
我叫小明。
```

然后：

```text
我叫什么？
```

Agent 能利用当前 Session 的聊天记录回答。

### 3. System Prompt

比如：

```text
You are an AI assistant.

Your goals:
- Answer user questions accurately.
- Use tools when necessary.
- Do not fabricate tool results.
- Ask for clarification only when necessary.
```

### 4. Tool Calling

第一版甚至只需要两个工具：

- `get_weather()`
- `calculator()`

用户：

```text
帮我查一下上海天气
```

流程：

```text
User
  ↓
Agent
  ↓
get_weather("Shanghai")
  ↓
Tool Result
  ↓
Agent
  ↓
最终回答
```

### 5. Agent Loop

Agent 不能只是：

```text
User → LLM → Response
```

而应该能够：

```text
User
 ↓
LLM
 ↓
Tool
 ↓
Observation
 ↓
LLM
 ↓
Response
```

### 6. Session State

最简单可以：

```text
sessions = {
    "user_123": [
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
    ]
}
```

第一版甚至不用数据库。

---

## 第一版建议做成

```text
Chat UI / CLI
      │
      ▼
   Go API
      │
      ▼
┌──────────────────────┐
│    Agent Runtime     │
│                      │
│  Prompt              │
│  Session State       │
│  Agent Loop          │
│  Tool Registry       │
│  Context Management  │
└──────────┬───────────┘
           │
           ▼
         LLM API
           │
      ┌────┴─────┐
      ▼          ▼
   Tool Call   Answer
      │
      ▼
 Local / API Tools
```

## 推荐技术栈

第一版尽量保持简单：

| 项          | 选择         |
| ----------- | ------------ |
| Language    | Go           |
| HTTP Server | net/http     |
| LLM Client  | 自己封装     |
| Data Format | JSON         |
| Session     | 内存 Map     |
| Tool System | Go interface |
| Config      | ENV          |
| Database    | 暂时不要     |
| Redis       | 暂时不要     |
| Vector DB   | 暂时不要     |
| MCP         | 暂时不要     |
| Multi-Agent | 暂时不要     |

甚至建议一开始不用复杂 Web Framework。

直接用 `net/http` 就足够。

等项目变大以后再决定要不要引入 Gin、Echo、Fiber、Chi 等。

---

## Go 项目结构

可以这样设计：

```text
chat-agent/
│
├── cmd/
│   └── server/
│       └── main.go
│
├── internal/
│   │
│   ├── agent/
│   │   ├── agent.go
│   │   ├── loop.go
│   │   └── prompt.go
│   │
│   ├── llm/
│   │   ├── client.go
│   │   └── types.go
│   │
│   ├── tools/
│   │   ├── tool.go
│   │   ├── calculator.go
│   │   └── weather.go
│   │
│   ├── session/
│   │   ├── store.go
│   │   └── memory.go
│   │
│   └── api/
│       └── chat.go
│
├── pkg/
│
├── go.mod
└── README.md
```

这里最重要的是：

- `agent/`
- `llm/`
- `tools/`
- `session/`

这四层不要混在一起。

### 1. LLM 层

首先不要让 Agent 直接依赖某一家模型厂商。

定义自己的接口：

```go
type LLM interface {
    Chat(
        ctx context.Context,
        req ChatRequest,
    ) (*ChatResponse, error)
}
```

你的 Agent 只知道：

```text
LLM.Chat()
```

而不知道底下是：

- OpenAI
- Anthropic
- Gemini
- Local Model

以后换模型就很简单。

例如：

```go
type ChatRequest struct {
    Messages []Message
    Tools    []ToolDefinition
}

type ChatResponse struct {
    Content   string
    ToolCalls []ToolCall
}
```

### 2. Message

自己定义统一的数据结构：

```go
type Message struct {
    Role       string
    Content    string
    ToolCallID string
}
```

例如：

```go
messages := []Message{
    {
        Role:    "system",
        Content: "You are a helpful AI assistant.",
    },
    {
        Role:    "user",
        Content: "北京今天天气怎么样？",
    },
}
```

以后无论换哪个 LLM Provider，都先转换成你的内部 Message。

这一层非常重要。

不要让 OpenAI SDK Types 污染整个项目。

最好：

```text
你的领域模型
      ↓
Adapter
      ↓
Provider SDK/API
```

### 3. Tool 系统

Tool 是你第一个 Agent 最值得自己设计的东西。

定义：

```go
type Tool interface {
    Name() string

    Description() string

    Execute(
        ctx context.Context,
        args json.RawMessage,
    ) (string, error)
}
```

例如 Calculator：

```go
type CalculatorTool struct{}

func (t *CalculatorTool) Name() string {
    return "calculator"
}

func (t *CalculatorTool) Description() string {
    return "Perform mathematical calculations"
}

func (t *CalculatorTool) Execute(
    ctx context.Context,
    args json.RawMessage,
) (string, error) {

    // parse arguments
    // calculate
    // return result

    return "42", nil
}
```

这样你的 Agent 不需要知道 Tool 的具体实现。

### 4. Tool Registry

然后做一个工具注册中心：

```go
type ToolRegistry struct {
    tools map[string]Tool
}
```

例如：

```go
func NewToolRegistry() *ToolRegistry {
    return &ToolRegistry{
        tools: make(map[string]Tool),
    }
}
```

注册：

```go
registry.Register(&CalculatorTool{})
registry.Register(&WeatherTool{})
```

调用：

```go
tool, ok := registry.Get(toolName)

if !ok {
    return errors.New("tool not found")
}

result, err := tool.Execute(ctx, args)
```

以后你加：

- `search_web`
- `query_database`
- `send_email`
- `read_file`
- `write_file`
- `run_code`

Agent 层完全不用改。

### 5. Session State

第一版直接：

```text
map[string][]Message
```

就够了。

不过由于 Go 有并发，需要加锁：

```go
type MemoryStore struct {
    mu       sync.RWMutex
    sessions map[string][]Message
}
```

例如：

```go
func (s *MemoryStore) Get(sessionID string) []Message {
    s.mu.RLock()
    defer s.mu.RUnlock()

    return s.sessions[sessionID]
}
```

保存：

```go
func (s *MemoryStore) Append(
    sessionID string,
    msg Message,
) {
    s.mu.Lock()
    defer s.mu.Unlock()

    s.sessions[sessionID] = append(
        s.sessions[sessionID],
        msg,
    )
}
```

第一版：

```text
MemoryStore
```

后面可以替换成：

- Redis
- PostgreSQL
- MongoDB

Agent 完全不用改。

### 6. 最核心：Agent Loop

这才是真正的 Agent。

伪代码大概：

```go
func (a *Agent) Run(
    ctx context.Context,
    sessionID string,
    input string,
) (string, error) {

    a.sessions.Append(
        sessionID,
        Message{
            Role:    "user",
            Content: input,
        },
    )

    for step := 0; step < a.maxSteps; step++ {

        messages := a.sessions.Get(sessionID)

        resp, err := a.llm.Chat(
            ctx,
            ChatRequest{
                Messages: messages,
                Tools:    a.toolDefinitions(),
            },
        )

        if err != nil {
            return "", err
        }

        // 没调用工具，说明已经得到答案
        if len(resp.ToolCalls) == 0 {

            a.sessions.Append(
                sessionID,
                Message{
                    Role:    "assistant",
                    Content: resp.Content,
                },
            )

            return resp.Content, nil
        }

        // 执行工具
        for _, call := range resp.ToolCalls {

            tool, ok := a.tools.Get(call.Name)

            if !ok {
                continue
            }

            result, err := tool.Execute(
                ctx,
                call.Arguments,
            )

            if err != nil {
                result = err.Error()
            }

            a.sessions.Append(
                sessionID,
                Message{
                    Role:       "tool",
                    Content:    result,
                    ToolCallID: call.ID,
                },
            )
        }
    }

    return "", errors.New(
        "maximum agent steps exceeded",
    )
}
```

这个其实就是你自己的：

**Go Agent Runtime V0.1**

核心只有：

```text
LLM
 ↓
Tool Call?
 ↓
Yes ──→ Execute Tool
         ↓
      Observation
         ↓
         LLM

No
 ↓
Final Answer
```

### 7. Agent 本身

可以设计成：

```go
type Agent struct {
    llm      LLM
    tools    *ToolRegistry
    sessions SessionStore

    systemPrompt string
    maxSteps     int
}
```

这样 Agent 核心依赖非常清晰：

```text
Agent
 │
 ├── LLM
 ├── Tools
 ├── Session
 └── Prompt
```

### 8. API 层

例如：

```text
POST /api/chat
```

请求：

```json
{
  "session_id": "user-123",
  "message": "帮我计算 128 * 39"
}
```

响应：

```json
{
  "message": "128 × 39 = 4992"
}
```

Go：

```go
type ChatRequest struct {
    SessionID string `json:"session_id"`
    Message   string `json:"message"`
}

type ChatResponse struct {
    Message string `json:"message"`
}
```

Handler：

```go
func (h *ChatHandler) Chat(
    w http.ResponseWriter,
    r *http.Request,
) {

    var req ChatRequest

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(
            w,
            "invalid request",
            http.StatusBadRequest,
        )
        return
    }

    result, err := h.agent.Run(
        r.Context(),
        req.SessionID,
        req.Message,
    )

    if err != nil {
        http.Error(
            w,
            err.Error(),
            http.StatusInternalServerError,
        )
        return
    }

    json.NewEncoder(w).Encode(
        ChatResponse{
            Message: result,
        },
    )
}
```

---

## 第一版最终架构

建议做到这里就停：

```text
                    Client
                      │
                      ▼
                  Go HTTP API
                      │
                      ▼
                 Chat Handler
                      │
                      ▼
                ┌───────────┐
                │   Agent   │
                └─────┬─────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
         LLM        Session     Tools
          │           │           │
          │       Memory Map   Registry
          │                       │
          │                ┌──────┴──────┐
          │                ▼             ▼
          │           Calculator      Weather
          │
          └──────────── Agent Loop
```

这已经是一个完整的 Chat Agent MVP。

---

## 建议的研发顺序

```text
Day 1 / V0
Go CLI
 ↓
LLM 对话

V1
 ↓
System Prompt
 ↓
多轮 Context

V2
 ↓
Tool Interface
 ↓
Calculator Tool

V3
 ↓
完整 Agent Loop
 ↓
最大 Steps

V4
 ↓
net/http API
 ↓
Session ID

V5
 ↓
Web Chat UI

V6
 ↓
PostgreSQL / Redis
 ↓
持久化 Session

V7
 ↓
Streaming
 ↓
SSE / WebSocket

V8
 ↓
RAG / Search / MCP
```

特别建议：不要一开始用 LangGraph 一类的 Agent 框架思维去套 Go。先自己实现 LLM abstraction + Tool Registry + Session Store + Agent Loop，可能几百行核心代码就够了。

等做到后面，会非常自然地理解为什么需要：

- State Machine
- Graph
- Planner
- Memory
- MCP
- Multi-Agent
- Tracing
- Evals

而不是先学一堆框架概念。

---

## 第一阶段目标（收束）

用 Go 写一个不依赖 Agent Framework 的 Chat Agent Runtime，支持：

- 多轮对话
- Tool Calling
- Agent Loop
- Session

这是一个非常适合作为自研 Agent 起点的范围。
