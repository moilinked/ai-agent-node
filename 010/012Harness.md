# Agent Harness：从 LLM 到自主执行系统的核心架构

在 AI Coding 领域，Harness 是理解 Claude Code、Codex、Pi 等 Coding Agent 架构差异的关键概念之一。

如果将 LLM 比作大脑，那么 Harness 就是让大脑能够感知环境、使用工具、执行任务、保留状态并控制行为的软件运行系统。

但这个比喻还不够准确。从工程角度看：

> Agent Harness 是围绕 LLM 构建的一套运行时控制系统（Runtime Control System），负责组织模型推理、上下文、工具执行、会话状态、权限、反馈与任务生命周期，使原本只能生成内容的模型成为能够持续执行任务的 Agent。

OpenAI 在介绍 Codex Agent Loop 时，明确将 Agent Loop 描述为协调用户、模型和工具执行的核心逻辑。VS Code 的 Agent Harness 文档也采用了相近的定义。

先给出三个核心结论：

1. Model 决定如何推理和选择行动，Harness 决定如何组织、执行和约束这些行动。
2. 相同的模型，搭配不同 Harness，可能表现出明显不同的 Agent 能力。
3. Harness 的价值不只是调用工具，而是保障长时间、多步骤任务能够可靠执行。

## 一、Harness 在 AI Agent 中处于什么位置？

先看一张简化的架构图。

![AI Agent 的整体架构](https://oilinil.cn/api/files/2026/10/161c13a7383871841071ec76abf371af/agent-1.webp)

这里最重要的是将三个组件分开。

| 组件        | 核心职责                           | 举例                               |
| ----------- | ---------------------------------- | ---------------------------------- |
| Model       | 推理、生成文本、提出工具调用       | Codex 模型、Claude                 |
| Harness     | 管理运行循环、工具调用、状态和权限 | Codex Harness、Pi Runtime          |
| Environment | 执行命令和保存实际文件状态         | 本地操作系统、Docker、远程 Sandbox |

OpenAI 的 Agents API 架构也明确区分 Harness、Environment 和 Application Server：Harness 控制 Agent 循环和 Session，Environment 提供实际计算和文件执行能力，Application Server 负责产品集成。

这解释了一个常见问题：

模型本身不会直接修改文件，也不会直接执行 Shell。

它可以生成类似这样的结构化工具调用请求：

```
{
  "tool": "read_file",
  "arguments": {
    "path": "src/auth.ts"
  }
}
```

Harness 接收请求、检查参数和权限，然后调用真实工具。工具返回的文件内容再被 Harness 写入模型上下文，模型才能继续推理。

因此，Harness 是模型意图与真实世界操作之间的桥梁。

## 二、Harness 与 Agent Loop 有什么区别？

这两个概念经常被混用，但从架构设计上可以作一个实用区分：

Agent Loop 是核心执行循环；Harness 是包含 Agent Loop 及其周边运行机制的完整系统。

### 1. 最简单的 Agent Loop

```
User Task
   ↓
Build Prompt
   ↓
Call LLM
   ↓
Model Response
   │
   ├── Final Answer → Stop
   │
   └── Tool Call
          ↓
      Execute Tool
          ↓
      Collect Result
          ↓
      Update Context
          ↓
      Call LLM Again
```

它的核心形式可以写成：

$$
s_{t+1} = F(s_t, a_t, o_t)
$$

其中：

- st：当前 Agent 状态，包括上下文、任务进度等。
- at：模型在当前状态下选择的行动。
- ot：工具或环境返回的观察结果。
- F：Harness 定义的状态更新逻辑。

模型负责提出行动：

$$
a_t \sim \pi_\theta(a \mid c_t)
$$

Harness 则负责构造 ct、执行允许的行动、处理观察结果并更新状态。

当然，实际模型生成的是 Token 序列，工具调用也是以特定格式编码的输出。这里是为了描述 Agent 系统而进行的抽象。

### 2. 一个最小 Harness 的伪代码

```ts
async function runAgent(task: string) {
  const session = createSession(task);

  while (!session.finished) {
    // 1. 构造模型上下文
    const context = buildContext(session);

    // 2. 调用模型
    const response = await model.generate({
      messages: context.messages,
      tools: toolRegistry.schemas()
    });

    // 3. 保存模型输出
    session.append(response);

    // 4. 判断是否需要工具调用
    if (response.toolCalls.length === 0) {
      session.finish(response.text);
      break;
    }

    // 5. 执行工具
    for (const call of response.toolCalls) {
      const result = await toolRegistry.execute(call);
      session.appendToolResult(result);
    }

    // 6. 下一轮继续推理
  }

  return session.result;
}
```

这段代码已经具备基础的 Agent 行为。

但是它还不是一个可靠的生产级 Harness。

它至少缺少：

- 超时、取消与错误恢复。
- 工具调用的参数和权限校验。
- Context Window 超限处理。
- Session 持久化。
- 最大执行步数和成本预算。
- 工具结果校验。
- 事件日志与 Trace。
- 人工审批和任务终止管理。

也就是说，实现 Agent Loop 很容易，实现可靠的 Harness 才是真正困难的部分。

## 三、生产级 Harness 的八大核心模块

我把 Harness 拆成八个逻辑模块。这不是某个产品的标准源码目录，而是一种方便开发和架构评审的功能分解。

### 1. Model Adapter：模型适配层

负责将不同模型的 API 差异统一为 Harness 可以理解的接口。

例如：

```
                 Model Adapter
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
     OpenAI API    Anthropic API   本地模型
          │            │            │
       Codex         Claude       Qwen 等
```

主要职责包括：

- 统一模型输入和输出结构。
- 处理 Streaming。
- 解析 Tool Calls。
- 处理不同 Provider 的错误与重试。
- 记录 Token Usage。
- 管理模型超时、上下文长度和参数限制。

需要特别注意：统一 API 并不意味着不同模型的语义能力完全一致。

例如不同模型对并行 Tool Calls、结构化输出、推理内容以及 Context Cache 的支持可能不同。适配器必须保留必要的能力差异，而不能简单地把所有行为强行转换成同一种格式。

[Pi](https://github.com/NicheCeo/pi-mono) 的 `pi-ai` 就承担了多模型统一调用的角色。

### 2. Context Engine：上下文管理引擎

这是影响 Coding Agent 实际表现的重要模块。

模型每次推理都需要收到 Context，但不代表应该把整个历史记录原封不动地传给模型。

Harness 需要决定：

$$
C_t = \operatorname{Assemble}(I, H_t, M_t, R_t, E_t, B)
$$

其中：

- $I$：System Instructions 与项目指令。
- $H_t$：当前会话历史。
- $M_t$：长期记忆及任务状态。
- $R_t$：检索得到的相关资料。
- $E_t$：环境观察和工具返回。
- $B$：上下文 Token 预算。

真正需要解决的问题包括：

| 问题           | Harness 的处理机制           |
| -------------- | ---------------------------- |
| Context 超限   | Compaction、截断、摘要       |
| 工具输出过长   | 分段读取、压缩、外部存储     |
| 重要约束丢失   | 将约束保留在持久状态         |
| 历史事实冲突   | Scope、Time、Provenance 校验 |
| 检索结果无关   | Context Selection、Reranking |
| Token 成本过高 | 缓存、去重、预算控制         |
| 长期任务恢复   | Checkpoint 与状态重建        |

例如，Agent 第一次读取某个文件：

```text
read("src/auth.ts") → 1200 行
```

第二次需要定位函数时，如果再次发送全部 1200 行，可能浪费大量 Token。

更合理的策略是保留文件引用、摘要和已确定的关键信息，需要时重新读取原始文件。

但如果某段代码即将修改，又不能只依赖压缩后的摘要，因为摘要可能遗漏关键细节。

Context Engine 的本质是信息选择与状态管理，而不只是 Token 截断。

Codex 的官方架构说明也明确描述了对不断增长的上下文进行压缩的机制，并强调通过稳定 Prompt 前缀提高 Prompt Caching 效率。

### 3. Tool Runtime：工具执行系统

Tool Calling 通常包含四个层次：

![一次工具调用的生命周期](https://oilinil.cn/api/files/2026/10/4f9461990a99a1d5b4ed4529dfc718b2/agent-2.webp)

这里最容易出现的安全误区是：

> 模型决定调用工具，所以模型也可以决定工具是否安全。

实际上，模型只能提出请求，真正的权限判断必须由可信运行时代码执行。

例如，模型发出：

```json
{
  "tool": "shell",
  "arguments": {
    "command": "rm -rf /important-data"
  }
}
```

Harness 应该先执行策略校验，并根据系统权限规则拒绝该命令或要求人工审批。

即使 Prompt 写着“不要删除文件”，也不应将它当成可靠的强制安全边界。

另外，MCP Server 提供的工具并不意味着自动获得 Codex Shell Sandbox 的保护。OpenAI 的 Codex 技术说明特别指出，外部 MCP 工具需要分别落实其自身的安全控制。

### 4. Session Manager：会话与状态管理

Agent Session 不等于一次模型调用。

一个 Coding Agent 任务可能经历几十轮模型推理和上百次 Tool Call。

因此，Harness 至少需要维护：

```text
Session
├── Session ID
├── Task Goal
├── Constraints
├── Conversation Events
├── Model Outputs
├── Tool Calls & Results
├── Context Checkpoints
├── Runtime Status
├── Approval Requests
└── Final Result
```

这里应当区分三种状态：

| 状态类型           | 包含什么               | 应如何保存              |
| ------------------ | ---------------------- | ----------------------- |
| Conversation State | 消息、工具调用历史     | 事件日志或消息存储      |
| Agent State        | 计划、进度、待解决问题 | 结构化状态与 Checkpoint |
| Environment State  | 文件、Git、进程、容器  | 真实执行环境或快照      |

这三者不能混为一谈。

**恢复会话历史，并不代表执行环境也已经恢复。**

例如 Harness 从数据库恢复了“测试服务已启动”的消息，但对应 Docker 容器可能早已停止。

因此恢复时还需要重新校验实际环境状态。

### 5. Policy Engine：权限与安全控制

Policy Engine 决定 Agent 被允许做什么。

例如：

- 是否可以读取项目外文件。
- 是否允许联网。
- 是否可以安装依赖。
- 是否可以执行破坏性命令。
- 是否允许调用外部系统。
- 是否需要用户确认。

这些限制必须在实际执行层生效。

一般可以分成：

**Prompt-level Constraints**：告诉模型应该遵守什么规则。

**Runtime-level Enforcement**：即使模型不遵守，系统也会拒绝不合法操作。

两者都很重要，但后者才是安全边界。

### 6. Orchestration：任务与子 Agent 编排

简单 Harness 是一个 Agent Loop。

复杂 Harness 可能管理多个 Agent：

```text
                 Parent Agent
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       Explorer     Coder       Reviewer
          │           │           │
          └───────────┼───────────┘
                      ↓
              Integration / Verify
```

Harness 需要负责：

- 创建子 Agent。
- 控制并发数量。
- 分配上下文与工具权限。
- 收集执行结果。
- 处理中断、失败和超时。
- 合并结果，避免文件修改冲突。

但请注意：**Subagent 和 Multi-Agent 不是 Harness 的必要组成部分。**

像 Pi 这样以极简为目标的 Harness，可以不默认提供复杂的多 Agent 编排，而让开发者按需扩展。

### 7. Verification：结果验证

Agent 执行了某个行动，不等于行动达到了用户目标。

例如：

```text
工具执行成功：文件写入完成
                      ≠
任务完成：Bug 修复成功
```

生产级 Coding Harness 可以设置独立验证流程：

```text
Code Changed
    ↓
Compile / Typecheck
    ↓
Unit Tests
    ↓
Integration Tests
    ↓
Acceptance Criteria
    ↓
Verified Result
```

不同工具对自动验证的覆盖程度不同，并非所有 Harness 默认都有完整的 Verifier。

尤其要避免让同一个模型仅通过自我评价，就认定任务成功。

### 8. Observability：运行追踪与可观测性

传统后端主要关注：

- 请求延迟。
- 错误率。
- CPU、内存。
- 服务吞吐量。

Agent 还需要关注：

- 每次模型调用及其 Token 使用量。
- 工具调用轨迹。
- Agent Loop 迭代次数。
- Context Compaction 次数。
- 重试与失败原因。
- 人工审批次数。
- 是否达到了可验证的任务目标。

这里的关键工具通常是 Execution Trace（执行轨迹）。

一条完整 Trace 应该能够回答：

> Agent 为什么读取这个文件？为什么选择修改这个函数？工具返回了什么？何时开始偏离正确方向？最终为什么报告成功？

没有 Trace，很难系统性改进 Agent Harness。

## 四、用真实的 Coding Agent 任务理解 Harness

假设我们使用相同的 Codex 模型，让两套不同的 Harness 执行同一个任务：

> 修复登录接口返回 401 的 Bug，并确保测试通过。

### 一个简单 Harness 的执行

```text
User Task
    ↓
LLM: 我认为 JWT 有问题
    ↓
Tool: 修改 JWT 代码
    ↓
LLM: 已经修复
    ↓
Finish
```

这套执行流程有几个问题：

- 没有充分读取证据。
- 没有验证原因。
- 没有运行测试。
- 没有确认最终状态。

### 一个更完善的 Harness 的执行

```text
User Task
    ↓
Load project instructions
    ↓
Read related code
    ↓
Run failing tests
    ↓
Collect evidence
    ↓
LLM proposes change
    ↓
Validate tool operation
    ↓
Edit code
    ↓
Run tests
    ├── Failed → return evidence → retry
    └── Passed
          ↓
     Check task constraints
          ↓
     Inspect changed files
          ↓
     Verified Result
```

第二套 Harness 并没有改变 Codex 模型的权重。

它改变的是：

1. 模型能获得哪些信息。
2. 模型能够执行哪些行动。
3. 如何处理工具反馈。
4. 什么情况下需要重新推理。
5. 什么情况下才允许报告任务完成。

这就是 Harness Engineering 的核心价值之一：通过设计运行环境、执行约束和反馈机制，改善 Agent 的实际任务表现。

OpenAI 在 2026 年 2 月的 Harness Engineering 实践文章中也强调了环境可理解性、可执行的架构约束和反馈循环的重要性。

详细文章可以查看：[harness-engineering](https://openai.com/index/harness-engineering/)

## 五、Claude Code、Codex、Pi 的 Harness 有什么区别？

这是理解 Harness 最直观的方式。

三者都具有 Agent Loop、模型适配、Context 管理和工具执行能力，但工程设计哲学不同。

| 维度         | Claude Code                   | Codex                           | Pi                              |
| ------------ | ----------------------------- | ------------------------------- | ------------------------------- |
| 设计倾向     | 功能丰富的 Coding Agent       | 可靠的软件工程执行系统          | 极简、可扩展的 Harness          |
| Agent Loop   | 内置                          | 内置                            | 轻量内核                        |
| Context 管理 | 自动压缩、项目上下文等        | 自动压缩、项目上下文等          | Session Tree、可定制压缩        |
| 默认工具     | 完整编码工具集                | 完整编码工具集                  | 主要是 read、write、edit、bash  |
| 扩展机制     | Hooks、Skills、MCP、Subagents | Skills、MCP、SDK、App Server 等 | TypeScript Extensions、SDK、RPC |
| 安全机制     | 权限与审批机制                | Sandbox 与审批机制              | 依赖具体配置及扩展              |
| 主要吸引力   | 开箱即用的开发体验            | 工程执行、安全边界与集成        | Harness 可编程程度高            |

其中，Claude Code 更注重成熟的交互式开发体验；Codex 在 Harness、执行环境和客户端之间采用明确的架构边界；Pi 则强调保持内核小巧，把差异化行为交给 Extensions。

可以参考：
- [unlocking-the-codex-harness](https://openai.com/index/unlocking-the-codex-harness/)
- [pi-mono](https://github.com/fivewillow/badlogic-pi-mono/blob/main/packages/coding-agent/README.md)
- [unrolling-the-codex-agent-loop](https://openai.com/index/unrolling-the-codex-agent-loop/)

### 1. Claude Code：功能丰富、配置驱动

Claude Code 提供相对完整的编码工具、项目指令、Hooks、Skills 和子 Agent 机制。

你通常不必重新设计整个 Agent Loop，就能通过配置改变不少行为。

适合希望快速得到成熟 Coding Agent，而不是从头研究运行时细节的开发者。

### 2. Codex：重视执行控制与可集成性

Codex 的一个值得关注的设计是将 Harness 与客户端和实际执行环境区分开。

![Codex 架构概念示意](https://oilinil.cn/api/files/2026/10/003f87e464917d64af912accdd5659f7/agent-3.webp)

OpenAI [官方介绍](https://openai.com/index/unlocking-the-codex-harness/)中，Codex App Server 为 CLI、IDE 等不同产品界面提供一个可集成的通信层，使它们可以使用底层 Codex Harness 的能力。

需要区分：Codex App Server 是一种集成接口，并不等同于 Agent Loop 本身。

### 3. Pi：Harness 作为可编程产品

Pi 的设计特别适合研究 Harness，因为它主动减少默认功能，将更多控制权暴露给开发者。

其架构可以概括为：

```text
Pi Coding Agent
     │
     ├── CLI / TUI
     ├── Session Management
     ├── Skills / Extensions
     │
     └── Pi Agent Core
              │
              ├── Agent Loop
              ├── Tool Execution
              └── State & Events
                       │
                       └── Pi AI
                            └── Model Providers
```

Pi 使用统一的模型接口处理不同 Provider，由 Agent Core 管理工具调用与状态，再由 Coding Agent 层提供终端体验和扩展机制。

这也解释了为什么 Pi 可以与 Codex 模型结合。

## 六、Pi + Codex 和原生 Codex，究竟改变了什么？

假设使用相同的 Codex 模型：

![Pi、Codex对比](https://oilinil.cn/api/files/2026/10/935a8487835173aeb8e343d6443e7dc8/agent-4.webp)

两者的模型可以相同，但下面这些变量会变化：

| 变量                 | 为什么影响 Agent                 |
| -------------------- | -------------------------------- |
| System Prompt        | 改变模型对角色、工具与任务的理解 |
| Tool Schema          | 改变模型可选择的动作空间         |
| Context Construction | 改变模型当前掌握的事实           |
| Tool Results         | 改变模型能够观察到的环境信息     |
| Compaction           | 改变长任务中保留与丢失的信息     |
| Execution Policy     | 改变哪些行动实际能够执行         |
| Error Recovery       | 改变模型遇到失败之后能否继续     |
| Completion Policy    | 改变何时结束任务                 |

因此，即使两个系统使用相同模型，也可能执行出不同的轨迹：

$$
\tau = \{s_0, a_0, o_0, s_1, a_1, o_1, \ldots, s_T\}
$$

Harness 改变状态构建方式和执行反馈，因此可以改变整条轨迹 $\tau$。

最终结果：

$$
P(\text{Success} \mid M, H_1, T) \neq P(\text{Success} \mid M, H_2, T)
$$

其中 $M$ 是模型，$H$ 是 Harness，$T$ 是任务。

这个不等式表示两者的成功概率可能不同，并不意味着某个 Harness 必然更好。

**对研究 AI Coding 而言，Pi + Codex 与原生 Codex 的对比，本质上就是 Harness 设计的对比实验。**

## 七、Harness、Framework、Skill、MCP 和 Workflow 的边界

这几个概念也经常混淆。

| 概念            | 是什么                  | 主要解决的问题         |
| --------------- | ----------------------- | ---------------------- |
| Model           | 推理模型                | 下一步生成什么         |
| Harness         | Agent 运行控制系统      | 如何持续运行并执行     |
| Agent Framework | 构建 Agent 的开发工具包 | 如何高效开发 Agent     |
| Workflow        | 任务执行流程            | 按什么步骤组织工作     |
| Skill           | 可复用的操作知识与指令  | 具体任务应该怎样做     |
| MCP             | 工具与上下文连接协议    | 如何接入外部能力       |
| Sandbox         | 隔离执行环境            | 在哪里安全地执行       |
| Memory          | 状态与知识存储机制      | 哪些信息可以跨步骤保留 |

它们的关系并不是完全平级的。

例如：

- 你可以使用 Framework 开发 Harness。
- 一个 Harness 可以运行多个 Workflow。
- 一个 Workflow 可以调用不同 Skill。
- MCP 可以提供 Harness 所需的工具接口。
- Harness 可以通过 Sandbox 执行危险或不可信代码。
- Memory 可以为 Harness 提供会话和长期上下文。

$$
P(\text{Success} \mid M, H_1, T) \neq P(\text{Success} \mid M, H_2, T)
$$

只有把模型推理、工具反馈和状态变化组织成持续的执行过程，才进入 Harness 的核心职责范围。

还有一个重要区别：

### Harness vs Workflow Engine

Workflow Engine 通常擅长执行预定义步骤和依赖关系。

例如：

```text
A: 获取代码
    ↓
B: 构建项目
    ↓
C: 执行测试
    ↓
D: 部署
```

Agent Harness 允许模型根据当前环境，在运行时动态选择行动。

例如：

```text
发现测试失败
    ↓
模型决定下一步
    ├── 阅读日志
    ├── 搜索相关代码
    ├── 修改某个文件
    └── 请求用户帮助
```

两者可以结合：Workflow Engine 控制确定性流程，Harness 管理需要模型自主选择的步骤。

同样，LangChain、Spring AI、Eino、Agents SDK 等属于开发工具或框架，它们可以提供 Harness 的部分甚至大量运行功能。无需从零实现所有模块才能拥有 Harness。

## 八、如果从零开发 Harness，应该如何设计？

### 1. 核心架构

```text
agent-runtime/
├── model/
│   ├── ModelClient
│   └── ModelRegistry
├── context/
│   ├── ContextBuilder
│   ├── Compactor
│   └── MemoryRetriever
├── agent/
│   ├── AgentLoop
│   ├── AgentState
│   └── AgentScheduler
├── tools/
│   ├── ToolRegistry
│   ├── ToolExecutor
│   └── ToolValidator
├── session/
│   ├── SessionStore
│   └── CheckpointManager
├── policy/
│   ├── PermissionChecker
│   └── ApprovalManager
├── events/
│   └── EventBus
├── verification/
│   └── ResultVerifier
└── observability/
    ├── Tracing
    └── Metrics
```

这是概念性的模块设计，具体实现还需要依情况而定。

### 2. Agent Loop 不应只是无限 while 循环

推荐使用显式状态机。

![Agent Runtime State Machine]()

状态机示意。审批、异常和恢复路径在实际实现中需要显式定义。

推荐至少包含：

`INITIALIZED`、`THINKING`、`TOOL_PENDING`、`WAITING_APPROVAL`、`EXECUTING`、`VERIFYING`、`COMPLETED`、`FAILED`、`CANCELLED`。

复杂任务还可以增加 `PAUSED`、`RECOVERING` 和 `COMPacting` 等状态。

这样做的优势在于：任务执行到某个阶段突然失败，系统能够知道应该恢复到哪里，而不必依靠模型猜测当前状态。

### 3. 采用事件日志与 Checkpoint

Harness 的每次状态变化都可以产生事件：

```json
{
  "session_id": "session-123",
  "event_id": "event-42",
  "type": "tool_execution_completed",
  "tool": "run_tests",
  "status": "failed",
  "exit_code": 1,
  "duration_ms": 2500
}
```

当 Agent 失败时，可以通过事件日志：

- 重建执行轨迹。
- 分析首次出现错误的步骤。
- 恢复会话。
- 评估不同版本 Harness 的表现。
- 生成工具使用和成本指标。

对文件修改、支付、部署等有副作用的动作，还需要幂等机制和实际状态核对。事件重放不应盲目重复执行所有工具调用。

### 4. 必须具备的终止机制

一个 Agent 即使模型一直想继续执行，也不能无限运行。

例如：

```yaml
limits:
  max_steps: 80
  max_tool_calls: 150
  max_wall_time_seconds: 1800
  max_cost_usd: 3.00

recovery:
  retry_transient_errors: true
  max_retries_per_tool: 2

completion:
  require_verification: true
```

这些是示例配置，不是通用最佳参数。

Harness 还必须能区分：

- 模型自然完成任务。
- 任务通过验证。
- 超出预算而停止。
- 工具执行失败。
- 等待用户审批。
- 用户主动取消。

尤其不能将“模型不再调用工具”直接等同于“任务成功”。

## 九、Harness 对幻觉和 Memory 冲突有什么作用？

### 1. 幻觉：Harness 无法消灭，但可以阻断传播

假设模型错误地说：

> 我已经执行测试，所有测试通过。

如果 Harness 没有执行记录，那么就不能将这一判断标记为经过验证的事实。

正确做法是：

```text
Model Claim
    ↓
Check Tool Execution Log
    ↓
Check Test Exit Code
    ↓
Check Test Report
    ↓
Verified / Unverified / Failed
```

也就是说，Harness 应该通过独立证据验证模型声明，而不是把模型声明本身当作事实。

### 2. Memory Conflict：Harness 应管理记忆的可信边界

这个能力应当放在 Harness 的 Context / Memory 管理层，而不只依靠 Prompt。

例如：

```text
Memory A：项目使用 Java
Memory B：计划迁移 Go
Memory C：仓库实际仍使用 Java
```

Harness 可以：

1. 根据实体、项目和时间检索记忆。
2. 识别“迁移计划”和“当前事实”的区别。
3. 查询当前代码仓库。
4. 将有效事实组装进入模型上下文。
5. 保留历史事实与来源。

因此，优秀的 Context Engine 不只是把信息交给模型，而是尽可能保证模型收到的信息具有正确的作用域、时效性和来源。

## 十、如何评价 Harness 的好坏？

不要仅使用“模型写代码厉不厉害”来衡量 Harness。

建议关注以下指标：

| 指标                    | 衡量目标                   |
| ----------------------- | -------------------------- |
| Task Success Rate       | 实际任务完成率             |
| Verified Success Rate   | 经过外部验证的成功率       |
| Tool Error Rate         | 工具调用错误率             |
| Recovery Rate           | 失败后的恢复能力           |
| Context Retention       | 长任务中关键约束的保留程度 |
| Token per Task          | 每个任务的 Token 消耗      |
| Execution Latency       | 任务端到端耗时             |
| False Completion Rate   | 错误宣称成功的比例         |
| Human Intervention Rate | 需要人工介入的频率         |
| Security Violation Rate | 违反执行安全约束的比例     |

其中一个值得特别关注的指标是：

$$
 \text{Efficiency} = \frac{\text{Verified Successful Tasks}} {\text{Total Cost}} 
$$

成本可以按 Token、美元或执行时间分别衡量，不能混为一个单位。

评价 Harness 时，还应控制模型版本、采样参数、任务集、工具权限以及执行环境。否则很容易将 Model 的能力差异误判为 Harness 的差异。

## 十一、Harness Engineering：为什么这个方向越来越重要？

这里还要区分两个术语：

**Agent Harness** 指负责 Agent 运行的实际软件系统。

**Harness Engineering** 则是围绕 Agent 的运行环境、工具、上下文、约束、反馈与验证机制进行设计和优化的工程实践。

OpenAI 在 [Harness Engineering: Leveraging Codex in an Agent-First World](https://openai.com/index/harness-engineering/) 一文中介绍了一个实践方向：工程师更多地设计可供 Agent 理解和验证的环境，通过架构约束、测试和反馈循环支持 Agent 完成开发工作。

传统软件开发主要是：

```text
开发者 → 编写代码 → 执行程序
```

Agent 驱动的软件开发逐渐增加一层：

```text
开发者 → 设计任务、工具、约束与反馈
                    ↓
                 Harness
                    ↓
                  Model
                    ↓
             执行、验证、迭代
```

因此，Harness Engineering 是一个跨越模型应用、系统设计、运行时、状态管理和软件可靠性的领域。

## 十二、推荐的源码与学习路径

### 第一阶段：Codex Agent Loop

阅读 [Unrolling the Codex Agent Loop](https://openai.com/index/unrolling-the-codex-agent-loop/)。

重点理解 Prompt Construction、Tool Calls、Streaming、Context Compaction、Prompt Caching。

[Codex 源码](https://github.com/openai/codex)

### 第二阶段：Pi 的极简设计

阅读 Pi 的 Agent Core、Model Adapter、Tool Runtime 和 Extensions。

重点理解：一个非常小的 Agent Loop 如何通过 Extensions 演变为复杂 Coding Agent。

[Pi 项目仓库](https://github.com/badlogic/pi-mono)

### 第三阶段：Runtime 与执行环境解耦

阅读 [Unlocking the Codex Harness](https://openai.com/index/unlocking-the-codex-harness/)，再研究 Sandbox、审批和 Session 恢复。

重点是让 Harness 不依赖某一个 UI、某一个容器或某一种模型。

## 最后总结

如果要记住 Harness 最重要的几个关系，可以用下面这个公式：

$$
\boxed{ \text{Agent System} = \text{Model} + \text{Harness} + \text{Tools / Environment} }
$$

而 Harness 的核心可以进一步拆成：

$$
\boxed{ \text{Harness} = \text{Loop} + \text{Context} + \text{Execution Control} + \text{State} + \text{Feedback} }
$$

这只是架构层面的概括，不是严格的数学定义。

对于 AI Coding Agent，我认为最值得深入研究的不是“怎么写一个 while 循环调用 LLM”，而是以下三个问题：

- **Context Engineering**：如何让模型每次推理都获得足够准确、相关且未失效的信息？
- **Execution Engineering**：如何让模型选择的行动安全、可靠、可取消、可恢复？
- **Feedback Engineering**：如何让工具返回、测试结果和环境状态真正推动 Agent 改正错误，而不是不断放大错误？

**最终，Model 影响 Agent 能做出多聪明的判断；Harness 影响这些判断能否转化为可靠、可控、可验证的实际工作。**
