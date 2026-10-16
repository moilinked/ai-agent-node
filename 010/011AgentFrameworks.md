# 主流 Agent 框架

> 先建立「框架在整条 Agent 架构里站哪一层」的心智，再按 **TypeScript** 把 LangChain 跑通。  
> API 与包名以官方文档为准；本篇示例刻意简化，生产仍要补权限、幂等、Trace、Eval。

## 1. LangChain 核心概念与使用

### 1.1 它是什么、不是什么

**LangChain**（JS/TS）：用统一接口拼装 **模型 + 工具 + Prompt + 中间件**，快速得到可运行的 Agent harness。

| 说法                                | 更准确                                                               |
| ----------------------------------- | -------------------------------------------------------------------- |
| 「用了 LangChain = 有生产级 Agent」 | 否。它加速接线与原型；State / 权限 / 失败路径仍要你设计              |
| 「LangChain 替代自研 Runtime」      | 部分替代样板代码；核心循环机制不变（模型提议 → 执行工具 → 回灌消息） |
| 「只学 Python 版就行」              | 概念通，但包名、导入、`createAgent` 选项以 **JS 文档**为准           |

对照前端：更像「带约定的请求/编排库」，不是业务中台。  
对照架构笔记：默认落在 **Single Agent + Tools**；要 Graph 骨架、断点续跑、人机确认，往 **LangGraph** 走。

### 1.2 生态怎么拆（先分清再选型）

```text
LangChain   → createAgent：可配置的 Agent harness（模型循环 + 工具 + middleware）
LangGraph   → 低层编排：状态图、持久化、HITL、确定性节点 + Agent 节点
LangSmith   → Trace / Debug / Eval（建议从第一个 Demo 就开）
Deep Agents → 在 createAgent 上预装规划、子 Agent、文件类能力等（batteries-included）
```

| 需求                           | 更合适                                                     |
| ------------------------------ | ---------------------------------------------------------- |
| 边界清晰的工具型助手、先跑通   | **LangChain `createAgent`**                                |
| 业务流程要可控路径、恢复、审计 | **LangGraph**（或 createAgent + checkpointer，再演进到图） |
| 排障与回归                     | **LangSmith**                                              |
| 想少配、多开箱能力             | 看 Deep Agents；仍要理解其底层仍是 harness                 |

一句话：

> **Agent = Model + Harness。**  
> LangChain 的 `createAgent` 提供可裁剪的 harness；复杂「走到哪」交给 Graph。  
> Harness 的详细拆解见下一小节。

### 1.3 Harness 是什么

很多人第一次看到 *harness* 会懵：既不是模型，也不等于整套后端。先用三句话钉死，再拆边界。

1. **模型自己不会「做事」**：它只会根据窗口里的内容生成文本或 `tool_calls`。  
2. **Harness = 包在模型外面的脚手架**：负责把真实世界（工具、消息、策略、护栏）接到模型上，并驱动「调模型 → 执行工具 → 回写 → 再调」直到结束。  
3. **Agent 好不好用，很大程度是 task–harness fit**：同一模型，客服 harness 与长跑 coding harness 的配置会完全不同。

官方表述可以记：

> Harness 是围绕模型的 scaffolding，把模型连接到真实世界；  
> 其核心职责是：**在每一步把「对的上下文」交给模型。**

#### 1.3.1 用一张图分清几个词

```text
┌──────────────────────── Runtime / 应用（更外层）────────────────────────┐
│  HTTP API、鉴权、租户、计费、队列、落库、前端 SSE …                       │
│                                                                        │
│   ┌────────────── Workflow / Graph（可选，管「走到哪」）──────────────┐   │
│   │  确定性节点：校验、写库、通知、分支                               │   │
│   │                                                                │   │
│   │   ┌──────────── Harness（管「这一个 Agent 节点怎么转」）─────────┐ │  │
│   │   │  system prompt / 消息组装 / 工具注册与执行 / middleware     │ │  │
│   │   │  循环控制、（可选）checkpointer、HITL 钩子                   │ │  │
│   │   │                                                           │ │ │
│   │   │              ┌──────── Model ────────┐                    │ │ │
│   │   │              │  推理、选工具、填参     │                    │ │ │
│   │   │              └───────────────────────┘                     │ │ │
│   │   └────────────────────────────────────────────────────────────┘ │ │
│   └──────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

| 概念                 | 回答的问题                                     | 不是什么                                       |
| -------------------- | ---------------------------------------------- | ---------------------------------------------- |
| **Model**            | 这一步生成什么、要不要调工具？                 | 不是执行器，也不保证副作用成功                 |
| **Agent Loop**       | 循环的机制形态（想→做→看→再想）                | 不是具体代码库名词；是抽象节奏                 |
| **Harness**          | 谁组装上下文、谁跑循环、谁执行工具、谁加护栏？ | 不是业务中台，也不自动等于 Graph               |
| **Runtime**          | 请求怎么进、会话怎么存、权限/观测怎么挂？      | 比 harness 更宽；harness 是其中核心一块        |
| **Workflow / Graph** | 整单业务「下一步去哪个节点」？                 | 可以内嵌 harness，但本身管路径不是管 tool loop |

对照前端（帮助迁移，不必一一对应）：

| Harness 里的东西            | 前端近似                                        |
| --------------------------- | ----------------------------------------------- |
| 调模型 + 工具循环           | 带重试的请求状态机（pending → calling → merge） |
| Tools + Schema              | 调用后端 API 的 client + 入参校验               |
| System prompt / 动态 prompt | 全局配置 + 按路由注入的策略                     |
| Middleware                  | 拦截器 / 中间件链（改请求、限流、打点）         |
| Checkpointer + `thread_id`  | 按会话 id 持久化的 store                        |

#### 1.3.2 Harness 里面通常有什么

按「从内到外」看一层层零件（LangChain 的 `createAgent` 大体覆盖这些）：

| 零件                             | 干什么                                                        | 缺失时的典型症状                       |
| -------------------------------- | ------------------------------------------------------------- | -------------------------------------- |
| **消息 / 上下文组装**            | 把 system、历史、工具结果放进窗口                             | 失忆、重复问、或窗口被脏历史撑爆       |
| **Tools**                        | 暴露可调能力；真正执行副作用                                  | 模型只会空谈；或乱调未授权接口         |
| **System Prompt / Skill 类指令** | 告诉模型怎么做、边界是什么                                    | 风格乱、步骤乱；**但写了也不等于鉴权** |
| **循环驱动**                     | 有 `tool_calls` 就执行再调模型，直到停                        | 你得自己写 while；或停不下来烧 Token   |
| **Middleware**                   | 在 before/after model、before/after tool 等钩子插入确定性逻辑 | 护栏只能靠「模型自觉」——不可靠         |
| **Checkpointer（可选）**         | 按 `thread_id` 存状态，多轮续跑                               | 每次 invoke 都像新会话                 |
| **停止与预算（应具备）**         | 步数/调用上限、超时、失败终止                                 | 空转、账单爆炸                         |

注意：上表最后一行，最小 Demo harness 往往很弱；**生产 harness 必须显式补上**，不能默认「框架已经帮你想好了」。

#### 1.3.3 最小 harness vs 厚 harness

```text
最小（createAgent 默认取向）
  model + tools + systemPrompt + 核心 tool loop
        │
        │  + middleware（摘要、重试、PII、HITL、限调用…）
        │  + checkpointer / 记忆 / 子 Agent…
        ▼
厚（如 Deep Agents 一类「预装意见」的 harness）
```

|      | 最小 harness（`createAgent`）  | 厚 harness（Deep Agents 等） |
| ---- | ------------------------------ | ---------------------------- |
| 优点 | 可裁剪、好理解、易按任务定制   | 开箱能力多，少从零拼         |
| 代价 | 生产能力要自己用 middleware 补 | 意见强，不合任务时要拆/改    |
| 何时 | 学习、定制业务 Agent           | 想快速接近「长任务助手」形态 |

选型口诀：**先保证 task–harness fit，再谈模型换新。**  
客服查单：窄工具 + 强确认 + 短上下文。长跑调研/写代码：摘要、待办、子 Agent、文件系统类能力更常见。

#### 1.3.4 Harness 不负责什么（避免期望错位）

1. **不负责业务「做对」的定义** — Success Criteria、是否真的退了款，要对照 DB / 工具回执。  
2. **不自动给你租户级鉴权** — Prompt 里的「禁止删除」可被忽略或注入绕过；鉴权在执行层。  
3. **不等于整条业务 Workflow** — 多步骤审批、断点续跑整张业务图，仍是 Graph / 状态机问题。  
4. **不是 Eval / 观测本身** — Trace、回归集通常靠 LangSmith 或自建；harness 最多打点方便接。

#### 1.3.5 和自研 Loop 的关系（读完应不再模糊）

自研时你手写的这段：

```text
while (未完成) {
  response = llm(messages, tools)
  if (有 tool_calls) 执行工具 → 把 ToolMessage 写回 messages
  else 返回最终回答
}
```

就是 **Agent Loop**。  
你在这段前后加的：拼 messages、注册 tools、校验参数、超时重试、截断上下文、写日志、存会话——合在一起就是 **Harness**。

因此：

- `createAgent(...)` ≈ 「框架帮你生成一个可配置 harness」  
- 换更强模型 ≠ 换更好 harness  
- 调不好 Agent，优先查：**上下文是否对、工具是否对、护栏是否在代码里**，而不是先怪「模型不够聪明」

### 1.4 安装与环境（TS）

官方要求（以文档为准）：

```bash
npm install langchain @langchain/core
# Node.js 22+

# 按厂商再装集成，例如：
npm install @langchain/openai zod
# 或 @langchain/anthropic 等
```

环境变量示例（二选一即可先跑通）：

```bash
export OPENAI_API_KEY="..."
# 或 ANTHROPIC_API_KEY / GOOGLE_API_KEY 等
```

建议同时开 Trace：

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY="..."
```

模型标识常见写法：`"provider:model"`（如 `"openai:..."`、`"anthropic:..."`），或传入已构造的 Chat Model 实例。**具体模型名以官方 Integrations 为准**，笔记里用占位，避免抄死过时 ID。

### 1.5 最小可运行示例

（这就是在装配一个**最小 harness**：model + tool + systemPrompt，由 `createAgent` 驱动 loop。）

```ts
import { createAgent, tool } from "langchain";
import * as z from "zod";

const getWeather = tool(
  (input) => `It's always sunny in ${input.city}!`,
  {
    name: "get_weather",
    description: "Get the weather for a given city",
    schema: z.object({
      city: z.string().describe("The city to get the weather for"),
    }),
  },
);

const agent = createAgent({
  model: "openai:gpt-4.1-mini",
  tools: [getWeather],
  systemPrompt: "You are a helpful assistant. Be concise.",
});

const result = await agent.invoke({
  messages: [
    { role: "user", content: "What's the weather in San Francisco?" },
  ],
});

console.log(result.messages.at(-1));
```

读这段代码时抓住三点：

1. **`tool` + Zod schema**：给模型可调用的能力面（Function Calling 层）。  
2. **`createAgent`**：挂上 model / tools / systemPrompt，内部跑 tool loop。  
3. **`invoke({ messages })`**：输入是消息列表；输出状态里仍带完整 `messages`。

这是 Demo：无 `max` 步数之外的业务护栏、无真实 API、无权限。生产工具必须走你自己的鉴权与副作用控制。

### 1.6 核心概念清单

#### Model

- 负责推理与是否发起 tool call。  
- 用字符串或实例传入；换厂商时尽量只换 model，少改编排代码。  
- 模型「更强」≠ 工具更稳；超时、半成功、错误码仍在工具执行层。

#### Messages（对话状态的主载体）

一轮里常见角色：

| 角色             | 含义                                   |
| ---------------- | -------------------------------------- |
| `system`         | 行为约束（也可用 `systemPrompt` 注入） |
| `user`           | 用户输入                               |
| `assistant` / AI | 模型回复；可能带 `tool_calls`          |
| `tool`           | 工具执行结果，回灌给下一轮模型         |

`createAgent` 的状态里通常有 **`messages`（追加式）**：新消息往上加，不默认整表替换。  
这对应架构里的 **Conversation**；真正的任务进度（`goal` / `step` / `waiting_human`）若需要，应另建字段或外置 Task State，不要只靠聊天散文。

#### Tools

- 对模型：带 name / description / JSON Schema 的可选项。  
- 对应用：普通函数（可 async）；副作用、鉴权、幂等在这里做。  
- TS 侧用 **Zod** 定义入参最常见。  
- 工具名建议 `snake_case`，少空格与奇葩字符，兼容性更好。

#### System Prompt

- 塑造策略与边界，**不能替代**服务端鉴权。  
- 静态字符串用 `systemPrompt`；要按用户/租户动态变，用 **middleware**。

#### Middleware（可裁剪的 harness 扩展点）

挂在 loop 外围，例如：改 prompt、截断上下文、限制工具、重试、注入自定义 state。  
原则：**一块 middleware 管一件事**，按需组合，避免一上来堆成巨型 Agent 配置。

#### Checkpointer + `thread_id`（多轮 / 可恢复的入门）

无 checkpointer 时，每次 `invoke` 往往只看得见你这次传入的 messages。  
要「同一会话接着聊」，典型写法：

```ts
import { createAgent } from "langchain";
import { MemorySaver } from "@langchain/langgraph";

const agent = createAgent({
  model: "openai:gpt-4.1-mini",
  tools: [/* ... */],
  checkpointer: new MemorySaver(), // 进程内；生产换持久化实现
});

const config = { configurable: { thread_id: "user-123-session-1" } };

await agent.invoke(
  { messages: [{ role: "user", content: "旧金山天气？" }] },
  config,
);

await agent.invoke(
  { messages: [{ role: "user", content: "那明天呢？" }] },
  config, // 同一 thread_id → 续上历史
);
```

| 概念                | 作用                                                |
| ------------------- | --------------------------------------------------- |
| `thread_id`         | 会话/线程作用域，决定读哪份 checkpoint              |
| Checkpointer        | 把图/Agent 状态落下来（内存或 DB）                  |
| `context`（若使用） | 单次调用只读配置（如 `userId`），与 thread 历史分开 |

`MemorySaver` 适合本地；进程重启即丢。生产要可持久化的 checkpointer，并想清楚租户隔离。

### 1.7 一次 invoke 在干什么

```text
invoke(messages[, config])
        │
        ▼
  组装状态（含历史，若有 checkpointer）
        │
        ▼
  ┌──── model ────┐
  │  要不要 tool？ │
  └──────┬────────┘
         │ 有 tool_calls
         ▼
     执行 tools → 写回 Tool 消息
         │
         └── 再调 model ↺ 直到无 tool_calls 或触发停止条件
         │
         ▼
  返回更新后的 AgentState（含 messages）
```

工程上仍建议自问（与架构篇一致）：

- 有没有步数 / 成本上限？  
- 工具失败是重试、换工具，还是 `failed`？  
- 高风险副作用是否强制人工确认？  
- 「模型说完成了」是否对照过真实副作用 / DB？

### 1.8 旧概念怎么看（Chains / AgentExecutor）

| 旧 Impession                 | 现在怎么用                                               |
| ---------------------------- | -------------------------------------------------------- |
| `LLMChain` / 各种 Chain 拼装 | 了解「可组合 Runnable」即可；Agent 入门不先堆 Chain      |
| `AgentExecutor`              | 历史执行器；新项目优先 `createAgent`                     |
| LCEL（`\|` 管道）            | 仍可能在非 Agent 流水线里见到；与 createAgent 主线分开学 |

面试若被问到 Chain：能说清「早期组合抽象」，并落到「当前官方主推 Agent harness + LangGraph」。

### 1.9 常见误区

1. **把 Demo Agent 当生产：** 缺 State、权限、幂等、Trace、Eval。  
2. **工具面过大：** 几十个工具全塞给一个 Agent → 选错/漏调；应分组、路由或检索。  
3. **用 Prompt 代替鉴权：** Skill/System 里的「先确认再删」可被忽略或注入绕过。  
4. **Conversation = Task State：** 聊天历史续得上，不代表业务单「做到哪一步」可恢复。  
5. **死背 Python 示例：** TS 用 `createAgent({ ... })`、`systemPrompt`、Zod；以 [JS 文档](https://docs.langchain.com/oss/javascript/langchain/overview) 为准。  
6. **把 harness 理解成「另一个模型」或「整个后端」：** 它是模型外的脚手架；Runtime / Graph 比它更外或更宽。

### 1.10 建议学习顺序（官方）

1. [JS Overview](https://docs.langchain.com/oss/javascript/langchain/overview)  
2. [Install](https://docs.langchain.com/oss/javascript/langchain/install) / [Quickstart](https://docs.langchain.com/oss/javascript/langchain/quickstart)  
3. [Agents / createAgent](https://docs.langchain.com/oss/javascript/langchain/agents)  
4. [Tools](https://docs.langchain.com/oss/javascript/langchain/tools)  
5. [LangGraph Overview](https://docs.langchain.com/oss/javascript/langgraph/overview)（要上 Workflow 时）  
6. [LangSmith](https://docs.langchain.com/langsmith/home)  

本地练习建议：先 1 个只读工具跑通 loop → 再加 checkpointer 多轮 → 再故意制造工具超时看失败路径 → 最后才加第二个写操作工具（带确认）。

### 1.11 本节收束

1. **Agent = Model + Harness**：模型负责推理；harness 负责接线、循环、上下文与护栏扩展点。  
2. LangChain TS 入门 = **`createAgent`（最小 harness）+ `tool` + messages**，不是 Chain 博物馆。  
3. Harness 可配；**可控业务流程**仍要 Graph / 自研状态机思维。  
4. 框架帮你少写样板，**不帮你免除**权限、状态、观测与评测。  
5. 包版本与模型 ID 变化快：**以官方文档为准**，本笔记只钉概念与用法骨架。

---

## 2. 工具集成与自定义工具

> 工具是 Agent「能做事」的唯一出口：模型只提出调用，**副作用发生在你的函数里**。  

### 2.1 先分层：别把「工具」一词混用

对照架构笔记的工具分层：

```text
模型提出调用（Function Calling）
        ↓
LangChain tool 包装（name + description + Zod schema + 你的函数）
        ↓
你的业务 API / DB / 第三方 SDK（真正鉴权、事务、副作用）
```

| 层级             | 职责                                        | 常见错误                            |
| ---------------- | ------------------------------------------- | ----------------------------------- |
| Function Calling | 稳定产出 `name + arguments`                 | 指望模型在散文里「自己写 JSON」更稳 |
| `tool(...)`      | 给模型可发现的能力面，并在 harness 里被执行 | 把原始宽 API 原样暴露给模型         |
| 业务 API         | 权限、幂等、审计                            | 只在 description 里写「请先确认」   |

一句话：

> **Tool 应比原始 API 更窄、更任务化**；权限永远在执行层，不在 Prompt。

### 2.2 定义工具：`tool` + Zod

```ts
import { tool } from "langchain";
import * as z from "zod";

const getOrder = tool(
  async ({ orderId }) => {
    // 这里打你的内部 API；示例简化
    return {
      orderId,
      status: "paid",
      shipped: false,
    };
  },
  {
    name: "get_order",
    description:
      "按订单号查询订单状态。仅用于已登录用户查询自己的订单，不用于退款。",
    schema: z.object({
      orderId: z.string().describe("订单号，例如 A1"),
    }),
  },
);
```

写法要点：

| 项            | 建议                                                                       |
| ------------- | -------------------------------------------------------------------------- |
| `name`        | `snake_case`；少空格与特殊字符，跨厂商兼容更好                             |
| `description` | 写清「何时用 / 何时不用」；近似工具要能区分                                |
| `schema`      | 必填、类型、`.describe()` 帮助模型填参                                     |
| 返回值        | `string`（给人读）或 **结构化 object**（给模型抽字段）；生产优先结构化摘要 |

挂到 Agent：

```ts
const agent = createAgent({
  model: "openai:gpt-4.1-mini",
  tools: [getOrder],
  systemPrompt: "查单时必须调用 get_order，结论要带订单号与状态。",
});
```

### 2.3 异步工具

工具的 handler 可以写成 `async`：查库、打 HTTP、投递队列等 I/O 都需要 `await`，这很常见。

```ts
const searchDocs = tool(
  async ({ query }) => {
    const res = await fetch(`https://example.com/search?q=${encodeURIComponent(query)}`);
    if (!res.ok) {
      return { ok: false, error: `upstream_${res.status}` };
    }
    const data = (await res.json()) as { hits: { title: string }[] };
    return {
      ok: true,
      // 入窗前先瘦身
      titles: data.hits.slice(0, 5).map((h) => h.title),
    };
  },
  {
    name: "search_docs",
    description: "搜索内部文档标题，返回最多 5 条。",
    schema: z.object({
      query: z.string().describe("检索关键词"),
    }),
  },
);
```

工程底线（框架不会替你做完）：

- **超时**：`AbortSignal` / 自己的 timeout，避免工具把整轮 Agent 挂死  
- **取消**：上层 `invoke` 被取消时，工具应能中止出站请求（Go 里对应 `context.Context`）  
- **不要在工具里睡死等人工**：人工确认应进 `waiting_human` / HITL 状态机，而不是 `await prompt()` 堵死事件循环  

### 2.4 错误如何回灌（让模型能改策略）

工具失败时，常见有两种处理：

1. **在工具内捕获，返回结构化错误**（多数业务首选）→ 变成 ToolMessage 内容，模型可换参/换工具/问用户  
2. **抛出异常** → 由 harness / middleware 的错误策略处理（重试、转成错误消息、或中止）

推荐返回「模型可读、可分支」的错误，而不是堆栈全文：

```ts
const createRefund = tool(
  async ({ orderId, amount }) => {
    try {
      // await refundApi(...)
      return { ok: true, refundId: "R1" };
    } catch (e) {
      const code = (e as { code?: string }).code ?? "unknown";
      // 可重试 vs 不可重试要写清楚，避免模型盲着重试
      return {
        ok: false,
        code,
        retryable: code === "timeout" || code === "503",
        message: "退款创建失败",
      };
    }
  },
  {
    name: "create_refund",
    description: "创建退款单。金额必须与可退余额一致。",
    schema: z.object({
      orderId: z.string(),
      amount: z.number().positive(),
    }),
  },
);
```

| 错误类型           | 建议回灌给模型的信息      | 模型侧期望动作             |
| ------------------ | ------------------------- | -------------------------- |
| 参数不合法         | 缺哪个字段 / 范围         | 改 arguments 或问用户      |
| 超时 / 5xx         | `retryable: true`         | 有限次重试或换工具         |
| 业务拒绝（不可退） | `retryable: false` + 原因 | 向用户解释并结束该路径     |
| 权限不足           | 明确 forbidden            | **不要**改去调更高权限工具 |

注意：

- 「返回错误字符串」≠「任务失败结束」；loop 可能继续。要硬停，需在 harness / middleware / 外层 Graph 设终止条件。  
- LangChain 里更细的 tool 重试策略，多通过 **middleware** 配置；细节以官方 [Tools](https://docs.langchain.com/oss/javascript/langchain/tools) / Agents 文档为准。

### 2.5 Context / 配置注入（别让模型填 userId）

`thread_id` 管会话历史；**`context` 管本次调用的只读配置**（用户身份、租户、locale 等）。  
身份类参数应来自网关鉴权后注入，**不要**设计成「请模型传入 `userId`」。

```ts
import { createAgent, tool } from "langchain";
import * as z from "zod";

const listMyOrders = tool(
  async ({ limit }, config) => {
    const userId = config.context.userId; // 来自 invoke，而不是模型胡填
    // return await db.orders.find({ userId }).limit(limit)
    return { userId, orderIds: ["A1", "A2"], limit };
  },
  {
    name: "list_my_orders",
    description: "列出当前登录用户的最近订单。",
    schema: z.object({
      limit: z.number().int().min(1).max(20).default(5),
    }),
  },
);

const contextSchema = z.object({
  userId: z.string(),
  tenantId: z.string(),
});

const agent = createAgent({
  model: "openai:gpt-4.1-mini",
  tools: [listMyOrders],
  contextSchema,
});

await agent.invoke(
  { messages: [{ role: "user", content: "我最近有哪些订单？" }] },
  {
    configurable: { thread_id: "sess-001" },
    context: { userId: "u_42", tenantId: "t_1" }, // 服务端注入
  },
);
```

|          | `context`                | `thread_id` / checkpointer |
| -------- | ------------------------ | -------------------------- |
| 变不变   | 单次 invoke 只读配置     | 跨轮持久化的对话/图状态    |
| 典型内容 | userId、tenant、权限角色 | messages、任务中间态       |
| 安全     | 绝不可被模型随意覆盖     | 要做租户隔离，防串会话     |

工具第二参在文档里也可能体现为 runtime / `ToolRuntime`（含 `toolCallId`、`writer` 等）。字段名以当前 JS SDK 为准；**模式不变：配置与身份从运行时进，不从模型参数进。**

### 2.6 危险工具与确认（执行层，不是文案层）

| 操作类型               | 默认策略          |
| ---------------------- | ----------------- |
| 只读查询               | 可自动            |
| 低风险可回滚修改       | 可自动 + 审计日志 |
| 发邮件 / 外部提交      | 草稿或确认        |
| 退款 / 删数据 / 改权限 | **强制确认**      |

三种常见落地（由严到松）：

1. **根本不注册写工具**：Agent 只产出「退款意图」结构化结果，由后端工作流/人工点执行。最稳。  
2. **HITL middleware / Graph 中断**：模型选了 `create_refund`，执行前暂停，等人批过再跑。  
3. **两步工具**：`prepare_refund`（只算方案）→ 人确认 → `commit_refund`（真正副作用）。

错误示范：

```text
system: 退款前必须先征得用户同意。
```

这只是希望模型听话，**可被忽略或注入绕过**。正确做法是：未带确认态时，写工具直接 `return { ok:false, code:"need_confirmation" }` 或根本不执行。

### 2.7 结果截断与结构化入窗

工具结果会进入 messages，占上下文预算。典型翻车：

- 把整页 HTML / 全表 dump 回模型  
- 日志 + 堆栈原样回灌  
- 多工具并行，每路都回大 JSON  

建议约定：

```ts
function toToolObservation(data: unknown, maxChars = 2000): string {
  const text = typeof data === "string" ? data : JSON.stringify(data);
  if (text.length <= maxChars) return text;
  return text.slice(0, maxChars) + `\n...[truncated, total=${text.length}]`;
}
```

更好的做法是 **先结构化再截断**：

- 列表：只留 id / 标题 / 状态，限制 N 条  
- 详情：只留本任务决策所需字段  
- 大文档：返回「摘要 + 引用 id」，需要时再 `get_doc_chunk`  

对照前端：列表接口返回 table row，不把整颗 ORM 实体塞进 props。

### 2.8 返回值形态（选型）

| 返回                   | 适用                               |
| ---------------------- | ---------------------------------- |
| `string`               | 简单说明、短答案                   |
| `object`               | 需要模型读字段、做分支（推荐默认） |
| 多模态 content blocks  | 截图等（模型需支持）               |
| `Command`（LangGraph） | 工具要直接改图状态时；入门可后看   |

默认优先：**小而稳的 object**（含 `ok` / `code` / 业务字段），少返回散文小说。

### 2.9 工具面治理（集成时的默认纪律）

1. **先有可治理 API，再包成 Tool**（更窄、任务化）。  
2. **同 Agent 工具不宜过多**；近似工具要么合并，要么用路由/子 Agent 拆开。  
3. **读多写少**：写工具单独名单 + 确认策略。  
4. **每个工具有超时、错误码、幂等键**（尤其支付/退款）。  
5. **观测**：name、arguments（脱敏）、latency、ok/code、token 占用——否则无法 Eval「工具是否选对」。

### 2.10 自研 Runtime 对照（学完可搬走）

用 Go / 自研时，不必绑 LangChain，但应对齐同一契约：

| LangChain           | 自研时至少要有                      |
| ------------------- | ----------------------------------- |
| `name` + Zod schema | JSON Schema / 校验库                |
| `description`       | 给模型的工具说明                    |
| tool 函数           | 带 timeout 的出站调用               |
| ToolMessage 回灌    | 把结果写回 messages                 |
| `context.userId`    | 从 `context.Context` / 请求态取身份 |
| 截断                | 入窗前 DTO / max bytes              |

### 2.11 本节收束

1. 工具 = 模型的行动出口；**执行与鉴权在你的代码里**。  
2. TS 落地骨架：`tool` + Zod +（可选）async + 结构化返回。  
3. 错误要「可分支」地回灌；身份走 **context**，不走模型参数。  
4. 危险操作靠执行层确认 / 不注册写工具；Prompt 只是辅助。  
5. 结果必须 **瘦身再入窗**，否则上下文污染会拖垮整个 harness。

练习建议：只读 `get_order` → 故意返回 `retryable` 错误看模型是否乱重试 → 再加 `create_refund` 但默认拒绝未确认 → 最后给大 JSON 做截断对比 Trace。

---

## 3. LangGraph 与可控 Workflow

> LangGraph 是**低层编排运行时**：用状态图把确定性步骤和 Agent 步骤画在同一张图里。  
> 它管「走到哪、如何暂停/恢复」；不替代模型与工具本身。API 以 [LangGraph JS 文档](https://docs.langchain.com/oss/javascript/langgraph/overview) 为准。

### 3.1 它解决什么

纯 `createAgent` 适合边界清晰的 tool loop。一旦出现：

- 多步业务路径要可枚举、可测  
- 断点续跑、失败从节点恢复  
- 退款等操作必须人工确认后再继续  
- 确定性校验/写库与 LLM 判断交替出现  

就更适合 **Graph / Workflow**。LangGraph 正是这一层的常见实现。

对照架构主线：**Workflow outside, Agent inside**——图钉路径，节点里再挂 harness / LLM。

```text
StateGraph（确定性骨架）
  ├─ 普通 Node：校验、查库、写库、发通知（代码说了算）
  ├─ Agent / LLM Node：理解、选工具、生成（模型说了算）
  ├─ Edge / Condition：下一步去哪（规则或状态字段）
  └─ Checkpointer：状态落盘，可 interrupt / resume
```

| | LangChain `createAgent` | LangGraph |
| - | ----------------------- | --------- |
| 抽象高度 | 现成 Agent harness | 低层编排（自己画图） |
| 默认形态 | 模型 ⇄ 工具循环 | 任意 Node/Edge 拓扑 |
| 强项 | 快速跑通单 Agent | 可控路径、持久化、HITL、长任务 |
| 关系 | 底层常建在 LangGraph 上 | 可单独用；节点里也可再嵌 Agent |

不必二选一：简单助手用 `createAgent`；业务流程变复杂再把关键路径抽成 Graph，或在 Graph 的某个节点里调用 Agent。

### 3.2 Node / Edge / State

| 概念 | 含义 | 例子（退货） |
| ---- | ---- | ------------ |
| **State** | 全图共享、可持久化的数据 | `orderId`、`withinWindow`、`messages`、审批结果 |
| **Node** | 一步工作单元（函数） | `loadOrder`、`policyCheck`、`askHuman`、`createRefund` |
| **Edge** | 走完 A 之后去哪 | `loadOrder` → `policyCheck` |
| **Condition** | 按 State 分支 | 可退 → 退款节点；不可退 → 说明原因后结束 |
| **START / END** | 图的入口与出口 | — |

最小示意（概念级；以当前 SDK 为准）：

```ts
import {
  StateGraph,
  StateSchema,
  MessagesValue,
  START,
  END,
} from "@langchain/langgraph";
import { z } from "zod";

const State = new StateSchema({
  messages: MessagesValue,
  orderId: z.string().optional(),
  withinWindow: z.boolean().optional(),
});

const loadOrder = async (state: typeof State.State) => {
  // 确定性：查库，不调用 LLM
  return { orderId: state.orderId, withinWindow: true };
};

const graph = new StateGraph(State)
  .addNode("load_order", loadOrder)
  .addEdge(START, "load_order")
  .addEdge("load_order", END)
  .compile();
```

要点：

- **State 要有契约**：节点读写约定字段；上一步散文不要直接污染下一步。  
- **能写死的边就写死**：退货窗口、权限校验别丢给模型「自由发挥」。  
- **Agent 只出现在需要开放判断的节点**：例如「用户这句话是咨询还是要退款」。

对照前端：State ≈ 多步向导的 store；Edge ≈ 路由表；Node ≈ 每一步的提交/校验逻辑。

### 3.3 和 `createAgent` 怎么分工

| 场景 | 更合适 |
| ---- | ------ |
| 单轮/短任务、工具少、路径事先画不全 | `createAgent` |
| 多阶段业务、要审计「走过哪些节点」 | LangGraph |
| 只要多轮聊天记忆 | `createAgent` + checkpointer 往往够 |
| 要「执行前暂停等人批」且状态可恢复 | LangGraph + checkpointer + `interrupt` |
| 节点内仍需要 tool loop | Graph 节点里调用 `createAgent`，或把 tool 循环做成子图 |

经验顺序：

```text
先 createAgent + 硬护栏
  → 流程与恢复要求上来后，把骨架迁到 StateGraph
  → 高风险边插入 HITL
  → 需要跨会话长期偏好再用 Store（与 thread 内 checkpoint 分开）
```

### 3.4 Persistence（可恢复的前提）

LangGraph 常见两套持久化：

| | **Checkpointer** | **Store** |
| - | -------------- | --------- |
| 存什么 | 某一 `thread_id` 上的图状态快照 | 跨线程的业务型 KV（偏好、事实等） |
| 用途 | 续聊、HITL、故障恢复、time travel | 长期记忆、跨会话共享 |
| 调用 | `compile({ checkpointer })` + 每次带 `thread_id` | `compile({ store })`，节点内读写 |

```ts
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver(); // 进程内；生产换 Postgres 等
const app = graph.compile({ checkpointer });

const config = { configurable: { thread_id: "refund-order-A1" } };

await app.invoke({ messages: [{ role: "user", content: "我要退 A1" }] }, config);
// 同一 thread_id 再次 invoke → 从 checkpoint 续上，而不是空白开局
```

注意：

- **没有 checkpointer，就没有可靠的 interrupt/resume。**  
- `MemorySaver` 重启即丢；生产用持久化实现，并做租户隔离。  
- `thread_id` 是指针：换 id = 新线程；复用 id = 续跑同一条。  
- Checkpoint 会随步数增长，长会话要考虑裁剪/保留策略（见官方 Persistence 文档）。

### 3.5 HITL（Human-in-the-loop）

模式：图跑到关键点 → `interrupt(payload)` 暂停并落盘 → 人审 → 用同一 `thread_id` + `Command({ resume })` 继续。

```ts
import { interrupt, Command } from "@langchain/langgraph";

async function approveRefund(state: /* State */) {
  const decision = interrupt({
    type: "approve_refund",
    orderId: state.orderId,
    amount: state.amount,
  });
  // resume 传入的值会成为 interrupt() 的返回值
  return { approved: Boolean(decision) };
}

// 首次：跑到 interrupt 停下
const config = { configurable: { thread_id: "refund-A1" } };
const paused = await app.invoke({ orderId: "A1", amount: 99 }, config);
// paused.__interrupt__ 里是给前端/人工看的 payload

// 人批准后：同一 thread 续跑
await app.invoke(new Command({ resume: true }), config);
```

约束（踩坑点）：

1. **必须有 checkpointer + 稳定 `thread_id`。**  
2. resume 后，**含 `interrupt` 的节点会从头再执行**；`interrupt` 之前若有副作用，要幂等或放进可安全重跑的封装。  
3. 不要用 `try/catch` 把 `interrupt` 产生的中断吞掉（除非按文档再抛出）。  
4. Prompt 里写「先问人」≠ HITL；未 resume 前写工具不应真正执行高风险副作用。

常见落地：退款确认、发邮件前审阅、删数据审批、把模型草稿改完再继续。

### 3.6 图上的节奏（再钉一次）

```text
确定性 Node（鉴权/校验）
    → Agentic Node（理解意图 / 选工具）
    → 确定性 Node（写库 / 审计）
    →（可选）interrupt 等人
    → 确定性 Node（真正副作用）
    → END
```

相对「一个大 Agent 从头逛到尾」：路径可测、状态可恢复、人工点可插入。  
相对「纯 BPM 没有模型」：在开放理解处仍保留 LLM。

包名与 `StateSchema` 写法随版本会变；机制不变——**State + Node + Edge + Checkpointer +（可选）interrupt**。细节以官方 Overview / Persistence / Interrupts 为准。
