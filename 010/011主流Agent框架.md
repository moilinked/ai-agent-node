# 主流 Agent 框架

> 先建立「框架在整条 Agent 架构里站哪一层」的心智，再按 **TypeScript** 把 LangChain 跑通。  
> API 与包名以官方文档为准；本篇示例刻意简化，生产仍要补权限、幂等、Trace、Eval。

## 学习目标

读完「LangChain 核心概念与使用」应能回答：

1. LangChain / LangGraph / LangSmith 各解决什么，和「自研 Runtime」什么关系  
2. **Harness 是什么**：和 Model / Agent Loop / Runtime / Graph 怎么划界；为何说 Agent = Model + Harness  
3. 当前 TS 入门路径为何是 `createAgent`，而不是旧的 Chain / AgentExecutor  
4. Model、Messages、Tools、System Prompt、Middleware、Checkpointer 各自干什么  
5. 一次 `invoke` 对应怎样的 Agent Loop；多轮会话靠什么续上历史  

---

## 本篇目录（规划）

1. **LangChain 核心概念与使用（TS）** ← 本节  
2. 工具集成与自定义工具（加深 `tool` / 权限 / 上下文）— 待补  
3. LangGraph 与可控 Workflow — 待补  
4. 其他框架对照（按需）— 待补  

> **纠错（相对旧提纲）：**  
> 「Agent Executor 与 Chains」曾是主路径，现已不是 TS 入门主线。见到大量 `LLMChain` / `AgentExecutor` 教程，当作历史即可；新代码优先 `createAgent` +（复杂时）LangGraph。

---

## 1. LangChain 核心概念与使用（TypeScript）

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

| 概念 | 回答的问题 | 不是什么 |
| ---- | ---------- | -------- |
| **Model** | 这一步生成什么、要不要调工具？ | 不是执行器，也不保证副作用成功 |
| **Agent Loop** | 循环的机制形态（想→做→看→再想） | 不是具体代码库名词；是抽象节奏 |
| **Harness** | 谁组装上下文、谁跑循环、谁执行工具、谁加护栏？ | 不是业务中台，也不自动等于 Graph |
| **Runtime** | 请求怎么进、会话怎么存、权限/观测怎么挂？ | 比 harness 更宽；harness 是其中核心一块 |
| **Workflow / Graph** | 整单业务「下一步去哪个节点」？ | 可以内嵌 harness，但本身管路径不是管 tool loop |

对照前端（帮助迁移，不必一一对应）：

| Harness 里的东西 | 前端近似 |
| ---------------- | -------- |
| 调模型 + 工具循环 | 带重试的请求状态机（pending → calling → merge） |
| Tools + Schema | 调用后端 API 的 client + 入参校验 |
| System prompt / 动态 prompt | 全局配置 + 按路由注入的策略 |
| Middleware | 拦截器 / 中间件链（改请求、限流、打点） |
| Checkpointer + `thread_id` | 按会话 id 持久化的 store |

#### 1.3.2 Harness 里面通常有什么

按「从内到外」看一层层零件（LangChain 的 `createAgent` 大体覆盖这些）：

| 零件 | 干什么 | 缺失时的典型症状 |
| ---- | ------ | ---------------- |
| **消息 / 上下文组装** | 把 system、历史、工具结果放进窗口 | 失忆、重复问、或窗口被脏历史撑爆 |
| **Tools** | 暴露可调能力；真正执行副作用 | 模型只会空谈；或乱调未授权接口 |
| **System Prompt / Skill 类指令** | 告诉模型怎么做、边界是什么 | 风格乱、步骤乱；**但写了也不等于鉴权** |
| **循环驱动** | 有 `tool_calls` 就执行再调模型，直到停 | 你得自己写 while；或停不下来烧 Token |
| **Middleware** | 在 before/after model、before/after tool 等钩子插入确定性逻辑 | 护栏只能靠「模型自觉」——不可靠 |
| **Checkpointer（可选）** | 按 `thread_id` 存状态，多轮续跑 | 每次 invoke 都像新会话 |
| **停止与预算（应具备）** | 步数/调用上限、超时、失败终止 | 空转、账单爆炸 |

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

| | 最小 harness（`createAgent`） | 厚 harness（Deep Agents 等） |
| - | ----------------------------- | ---------------------------- |
| 优点 | 可裁剪、好理解、易按任务定制 | 开箱能力多，少从零拼 |
| 代价 | 生产能力要自己用 middleware 补 | 意见强，不合任务时要拆/改 |
| 何时 | 学习、定制业务 Agent | 想快速接近「长任务助手」形态 |

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
  // 换成你账号可用的模型字符串，或 ChatOpenAI 等实例
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

（下列零件都可以看成 harness 的可配置部件；总览见 §1.3。）

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

## 2. 工具集成与自定义工具（待补）

要点预告：Zod schema、异步工具、错误如何回灌、context / 配置注入、危险工具与确认、结果截断入窗。

---

## 3. LangGraph 与可控 Workflow（待补）

要点预告：Node / Edge / State、与 `createAgent` 的分工、Persistence、HITL。
