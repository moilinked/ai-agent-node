# Multi-Agent

Multi-Agent 是当前 AI 应用开发中很重要的一种架构，但也是容易被过度设计的架构。

**不要因为任务复杂就使用 Multi-Agent，而应该在单 Agent 遇到明确的能力瓶颈时，再考虑引入多个 Agent。**

如果用一句话概括：

> Multi-Agent 通过让多个具有独立上下文、工具权限和职责的 Agent 协同工作，解决单个 Agent 难以高效处理的复杂任务。代价是更高的协调成本、状态管理复杂度，以及更难控制的错误传播。

本文从工程落地的角度展开，并结合 LLM Agent 的发展、常见架构和真实设计取舍，重点回答三个问题：它是什么、什么时候值得使用，以及怎样避免把系统做得复杂又不可靠。

## 一、先理解 Multi-Agent 到底是什么

### 1. Agent 与普通 LLM 调用有什么区别？

通常，一个 Agent 可以理解为：

$$
Agent = Model + Instructions + Tools + State + Control\ Loop
$$

其中：

- Model：负责理解任务、推理和决策。
- Instructions：定义角色、目标、约束和权限。
- Tools：让模型能够查询数据、操作文件、执行代码或调用 API。
- State：保存当前任务上下文、历史与执行状态。
- Control Loop：允许模型根据执行结果继续选择下一步操作。

普通 LLM 调用通常是输入一次、输出一次；Agent 则可以多次执行“推理 → 调用工具 → 观察结果 → 调整计划”。

Multi-Agent 在此基础上增加了多个独立 Agent 之间的协调。

### 2. Multi-Agent 与单 Agent 的区别

区别不在于调用了多少次模型，而在于是否存在多个具有独立执行职责的 Agent，以及这些 Agent 如何分工、协调和交换结果。

举个例子：单 Agent 先搜索、再写代码、最后运行测试，仍然是单 Agent；即使调用模型十次，也不自动成为 Multi-Agent。

但如果研究 Agent、编码 Agent、测试 Agent 分别维护自己的上下文，由调度机制分配任务并协调结果，这才是典型的 Multi-Agent。

### 3. 系统通常拆成哪三层？

最常见的架构并不是简单地让多个 Agent 互相聊天，而是把整个系统拆成三个部分：

1. 控制层（Control Plane）：谁负责决策、拆任务、调度和验收。
2. 执行层（Execution Plane）：哪些 Agent 真正完成研究、编码、测试等任务。
3. 状态层（State Plane）：任务状态、执行结果、共享上下文和中间产物存储在哪里。

理解这三层，基本就能弄清楚主管 Agent（Supervisor）和子 Agent（Worker）之间如何协作。

工程上的核心结论：

Supervisor 负责决定做什么，Worker 负责怎么做，Orchestrator（编排器）负责确保任务真正被执行、状态被记录、失败可以恢复。

Supervisor 不一定要直接管理所有底层通信。生产系统里，通常会让确定性的程序代码管理状态和生命周期，让 LLM 负责需要智能判断的任务规划和路由。

## 二、Multi-Agent 是如何发展起来的？

Multi-Agent 不是大语言模型诞生后才有的概念。传统多智能体系统早已用于分布式决策、机器人协同、交通调度和博弈研究。

LLM 带来的重要变化，是让 Agent 的决策、交流和任务分工可以更多地通过自然语言完成。

| 阶段                 | 主要特点                    | 关键变化                       |
| -------------------- | --------------------------- | ------------------------------ |
| 传统 MAS             | 规则、规划、博弈、分布式控制 | 研究多个自主主体的协作与竞争   |
| LLM Agent 兴起       | 模型推理 + 工具调用         | Agent 可以处理开放式任务       |
| LLM Multi-Agent 探索 | 角色分工、讨论、模拟组织    | 通过多个上下文进行协作         |
| 工程化阶段           | 状态持久化、工作流、权限、可观测性 | 从 Demo 转向可靠执行系统 |

今天工程实践中的关键问题已经从“多个 Agent 能不能协作”，逐渐转向“多个 Agent 协作能不能比简单方案带来可测量的收益”。

## 三、最常见的组织架构

这里要区分两个维度：组织结构决定谁领导谁，执行模式决定任务如何流动。

| 架构                | 核心机制            | 适合场景         | 主要风险         |
| ------------------- | ------------------- | ---------------- | ---------------- |
| Supervisor–Worker   | 主管分发并验收      | 综合任务、研究、开发 | 主管成为瓶颈     |
| Hierarchical        | 多级主管递归拆解    | 超大型复杂任务   | 层级过深、信息损失 |
| Pipeline            | 固定顺序传递结果    | 审批、数据处理   | 前序错误向后传播 |
| Parallel            | 多 Agent 并行执行   | 多源研究、独立模块 | 结果冲突、资源竞争 |
| Handoff             | Agent 之间移交控制权 | 客服、专业领域切换 | 上下文丢失       |
| Peer / Blackboard   | 平级或共享任务板协作 | 分布式异步系统   | 状态一致性复杂   |

具体形态可以这样理解。

1. Supervisor / Worker（中心主管型）

一个主管 Agent 拆解任务，调用多个专业子 Agent，最后汇总结果。

适合：研究报告、复杂代码开发、数据分析。通常是最容易落地的起点。

风险：主管成为瓶颈。拆解、路由和验收都经过同一个 Supervisor，子任务一多，它的上下文、延迟和 Token 会先被打满，Worker 空等。

例子：商品详情页同时交给前端、后端、测试三个 Worker。三份结果都已经返回，主管仍要逐份读完再决定下一步，整体耗时卡在这一轮推理上。

2. Hierarchical（多级主管型）

总主管 → 领域主管 → 执行 Agent，支持递归分配任务。

适合：大型软件项目、长周期自主任务、需要多团队协调的系统。

风险：层级过深、信息损失。每一层都会改写任务，约束和验收标准逐层变薄，底层看到的目标和上层已经不是同一件事。

例子：总主管要求「详情页必须对齐购物车 API，不得改后端」。领域主管转成「做详情页前端」，执行 Agent 没看到这条约束，直接改了接口字段。

3. Sequential / Pipeline（流水线型）

Agent A 的产物作为 Agent B 的输入，再由 Agent C 完成下一环节。

适合：需求分析 → 开发 → 测试 → 部署等相对固定的流程。

风险：前序错误向后传播。下游默认上游产物可信，很少回头核对前提，错误会一路进入后面的环节。

例子：需求 Agent 把 SKU 规格写错，开发按错误规格实现，测试只核对「实现与需求文档一致」，错误规格进入部署。

4. Parallel（并行执行）

多 Agent 并行执行彼此独立的子任务。

适合：多源研究、独立模块。

风险：结果冲突、资源竞争。各自上下文独立，汇总时结论可能对不上；若同时写同一份文件或同一条数据，后完成的会覆盖先完成的。

例子：两个研究 Agent 同时查竞品价格，一个取官网价，一个取促销价，汇总时互相矛盾；两边又都在写同一份报告，后写入的版本盖掉了先写入的版本。

5. Peer-to-Peer / Handoff（平级协作型）

没有一直掌控全局的主管，Agent 根据任务需要把控制权交给另一个 Agent。

适合：客服分流、专业领域切换、不同 Agent 轮流主导的会话。

风险：上下文丢失。移交时如果只传一句自然语言，上一个 Agent 已确认的事实、约束和未完成事项到不了下一个 Agent。

例子：客服 Agent 已经确认要退的是订单 A，移交给售后 Agent 时只说「用户要退货」。售后按最近一笔订单 B 开了退货单。

6. Blackboard / Event-Driven（共享状态型）

多个 Agent 围绕共享任务板、事件总线、数据库协作，完成一个子任务后触发其他 Agent。

适合：异步任务、长时间运行、高并发系统。

风险：状态一致性复杂。多个 Agent 读写同一块任务板，没有版本和任务所有权时，会重复领取，或把别人尚未完成的状态写成完成。

例子：编码 Agent 还在改详情页，测试 Agent 已经从任务板上读到「待测试」并把它标成通过。后面的发布 Agent 被这个事件触发，拿到的是半成品。

这几种模式并不互斥。比如一个 Hierarchical 系统，内部可以同时使用 Pipeline 和共享状态事件总线。实践中最常见的是混合架构。例如 Supervisor 创建一个 DAG，独立任务并行执行，依赖任务按照 Pipeline 串行推进。

OpenAI Agents SDK 明确区分两种重要方式：`Agents as tools` 让主管继续掌控最终回复；`Handoffs` 则让专业 Agent 接管会话。

## 四、什么时候应该使用 Multi-Agent？

先判断三个条件：**上下文隔离、任务并行、专业能力差异。**这是 Multi-Agent 最有价值的三个驱动因素。

### 场景 A：任务可以有效并行

例如要对 20 个竞品做市场研究，每个竞品的资料收集彼此独立。

单 Agent 依次研究需要承担所有等待时间；多个研究 Agent 可以分组并行。

但注意：多 Agent 并不必然更快。它要看可并行比例和额外协调成本。

$$
T_{\text{multi}} \approx T_{\text{split}} + \max(T_1, \ldots, T_n) + T_{\text{merge}}
$$

如果拆解、启动与合并成本大于节省的执行时间，多 Agent 反而更慢。

### 场景 B：单 Agent 上下文被严重污染

一个 Agent 如果同时处理海量检索结果、代码日志、历史决策与测试报告，模型可能难以稳定关注当前任务。

多个 Agent 可以隔离上下文：研究 Agent 只看资料，编码 Agent 只看代码，测试 Agent 只看验收标准与产物。

不过，上下文隔离不等于信息越少越好。如果子 Agent 缺少必须共享的设计约束，最终反而会集成失败。

### 场景 C：不同任务需要不同能力或权限

例如：

- 数据研究 Agent 只拥有只读数据库权限。
- 编码 Agent 可以修改独立工作区。
- 测试 Agent 可以运行测试，但不能部署。
- 发布 Agent 只有收到审批后才能操作生产环境。

这里分 Agent 不只是为了智能分工，更是为了权限隔离和降低执行风险。

### 场景 D：任务结果可被独立验证

例如代码是否通过测试、数据是否满足 Schema、报告引用是否存在，均可以建立明确验收机制。

这种任务更容易分发给多个 Agent，因为每个子任务具有清晰的完成条件。

反过来，如果每个 Worker 完成后都需要 Supervisor 重新读取全部上下文才能判断是否正确，那么并行协作的价值会明显下降。

## 五、什么时候不应该使用 Multi-Agent？

| 场景                   | 更合适的方案                 | 原因                 |
| ---------------------- | ---------------------------- | -------------------- |
| 简单问答、摘要、分类   | 普通 LLM 调用                | 多 Agent 只增加复杂度 |
| 单一代码文件修改       | 单 Agent + Tools             | 工作范围足够集中     |
| 步骤完全固定           | 确定性 Workflow              | 不需要多个模型反复决策 |
| 强依赖串行推理         | 单 Agent 或分阶段 Workflow   | 难以并行             |
| 极度重视延迟和 Token 成本 | 单 Agent 优先             | 多次推理与同步有开销 |
| 共享资源冲突严重       | 串行执行或资源锁             | 并发不一定提高吞吐量 |

特别需要警惕一个误区：

**角色数量增加，不代表智能水平增加。**

把一个任务拆成“规划 Agent、思考 Agent、写作 Agent、检查 Agent”，看起来很完整，但如果这些角色重复阅读同一份上下文、做相似的判断，可能只是在重复消耗 Token。

多 Agent 相比单 Agent 可能使用约 3～10 倍 Token。这个范围不是通用常数，但足以说明协调开销不能忽略。

## 六、Supervisor 到底怎样给 Worker 分发任务？

以一个真实的例子来解释：

用户提出：

> 帮我开发一个商品详情页，支持图片切换、SKU 选择、加入购物车，并完成测试。

主管 Agent 不应该直接把这整段话复制给每个 Worker，而应该先建立任务依赖关系。

![multi-agent-img-1](https://oilinil.cn/api/files/2026/10/4dec5b03a9cbedf87960ed1caf83b6a4/multi-agent-img-1.webp)

### 1. 常见的三种分发机制

| 机制       | 怎么分发                                     | 特点                         |
| ---------- | -------------------------------------------- | ---------------------------- |
| Tool Call  | Supervisor 调用 `frontendAgent(ctx, task)`   | 简单，同步等待结果           |
| Task Queue | Supervisor 创建任务，Worker 从队列领取       | 异步、并发、可重试           |
| Handoff    | Supervisor 交出控制权，由目标 Agent 继续执行 | 适合专业 Agent 接管会话      |

最简单的实现是把 Worker 包装成 Tool：

```go
frontendResult, err := frontendAgent(ctx, task)
if err != nil {
    return err
}
backendResult, err := backendAgent(ctx, task)
if err != nil {
    return err
}
```

如果两个任务独立，可以并行：

```go
// errgroup 等待两边都结束；任一失败则取消另一边并返回错误。
// 依赖 golang.org/x/sync/errgroup。
g, gctx := errgroup.WithContext(ctx)

var frontendResult, backendResult Result
g.Go(func() error {
    var err error
    frontendResult, err = frontendAgent(gctx, frontendTask)
    return err
})
g.Go(func() error {
    var err error
    backendResult, err = backendAgent(gctx, backendTask)
    return err
})
if err := g.Wait(); err != nil {
    return err
}
```

但这里有一个很重要的工程问题：

**并行运行多个 Agent，不代表它们自动知道彼此正在做什么。**

它们的上下文可能完全独立，必须通过明确的任务输入、共享存储或事件协议来协调。

### 2. 分发时，Supervisor 需要传哪些内容？

推荐使用结构化的 Task Contract，而不是一句自然语言指令。

```json
{
  "task_id": "T1",
  "parent_task_id": "T0",
  "assigned_to": "frontend_agent",
  "objective": "实现商品详情页前端",
  "inputs": {
    "prd": "artifact://prd-v1",
    "api_spec": "artifact://cart-api"
  },
  "dependencies": [],
  "constraints": [
    "使用 React",
    "不得修改后端 API"
  ],
  "acceptance_criteria": [
    "SKU 选择正常",
    "加入购物车交互正常"
  ],
  "deadline": null,
  "max_attempts": 3
}
```

其中最重要的是四个字段：`objective`、`inputs`、`constraints` 和 `acceptance_criteria`。

任务目标告诉 Worker 要做什么，输入提供上下文，约束限定权限，验收标准定义什么时候算完成。

如果没有验收标准，Worker 很容易返回一句“已经实现”，但实际上没有可检查的交付物。

## 七、主管 Agent 和子 Agent 如何同步状态？

这是 Multi-Agent 最容易设计错误的地方。

**需要区分三个完全不同的概念：任务状态、执行上下文和交付产物。**

| 类型          | 保存什么                            | 谁来维护              |
| ------------- | ----------------------------------- | --------------------- |
| Task State    | pending、running、completed、failed | Orchestrator / 数据库 |
| Agent Context | Agent 的指令、对话记录、工作记忆    | Agent Runtime         |
| Artifacts     | 代码、文档、测试报告、研究结果      | 文件系统 / 对象存储   |

不建议把这三种东西全部塞进一个共享聊天记录里。

### 1. 状态如何流转？

Worker 任务状态机：

```text
Pending
   ↓
Queued
   ↓
Running
   ↓
Completed
   ↓
Accepted
```

交互示意：生产系统还需处理 blocked、failed、cancelled、timed_out 及重试状态。

这里需要区分 `completed` 和 `accepted`：

- `completed`：Worker 认为自己的工作完成了。
- `accepted`：Supervisor 或验证程序确认交付物符合要求。

这两个状态不能混为一谈。

### 2. Worker 怎样报告进度？

通常有三种同步方式。

方式 A：同步返回

主管调用 Worker，然后等待 Worker 完成。

```go
result, err := worker.Run(ctx, task)
if err != nil {
    return err
}
```

优点是实现简单，适合几秒到几十秒的子任务。

缺点是主管无法通过这个返回值直接得知执行过程中的详细进度，除非 Runtime 提供流式事件。

方式 B：事件推送

Worker 每完成一个阶段，就向事件总线发送状态事件。

```json
{
  "event_id": "evt-102",
  "type": "task.progress",
  "task_id": "T1",
  "attempt": 1,
  "status": "running",
  "progress": {
    "stage": "implementing_sku_selector",
    "completed_steps": 3,
    "total_steps": 5
  },
  "timestamp": "2026-10-27T01:00:00Z"
}
```

典型流程是：

![multi-agent-img-2](https://oilinil.cn/api/files/2026/10/bdc8e1ca628b5f1d10933ceb7ba51dc4/multi-agent-img-2.webp)

这里 Worker 不一定要直接和 Supervisor 通信，它只需要把事件交给 Orchestrator。

这是一种非常重要的解耦。

方式 C：主管主动轮询

Supervisor 或调度程序定期查询任务状态：

```http
GET /tasks/T1
```

返回：

```json
{
  "task_id": "T1",
  "status": "running",
  "attempt": 1,
  "last_heartbeat": "2026-10-27T01:00:00Z",
  "artifact_refs": []
}
```

适合简单的远程执行服务。缺点是存在轮询延迟和额外请求。

实际项目中经常采用 **事件驱动 + 数据库持久化 + 必要时轮询恢复** 的组合。

### 3. 状态应该放在哪里？

对于有多个并发 Worker 的生产系统：

![multi-agent-img-3](https://oilinil.cn/api/files/2026/10/43bf04b863f013acdc817c76f27ca1a7/multi-agent-img-3.webp)

数据库保存的是任务的权威状态（Source of Truth）。

Redis 或消息队列可以用来传递事件、安排执行，但消息队列里的某条消息本身不应成为唯一的任务状态记录。

尤其不要只靠 Supervisor 的上下文窗口记住哪些 Worker 完成了工作。上下文可能被压缩、截断，服务实例也可能重启。

## 八、状态同步不能完全依赖 LLM

假设主管同时启动三个 Worker：

![multi-agent-img-4](https://oilinil.cn/api/files/2026/10/1279c13566a06012fb82563fb1bda252/multi-agent-img-4.webp)

这里有几个必须由代码保证、而不是让 LLM 自行猜测的规则。

### 1. 状态更新需要原子性

例如：

```sql
UPDATE tasks
SET status = 'running',
    worker_id = 'worker-01',
    version = version + 1
WHERE task_id = 'T1'
  AND status = 'queued'
  AND version = 3;
```

只有更新成功的 Worker 才拥有任务执行权。

这可以防止两个 Worker 同时领取同一个任务。

### 2. 事件需要防重复处理

消息队列可能重复投递消息。

因此每个事件需要 `event_id`，每次执行需要 `attempt_id`，写入结果时要校验当前执行是否仍然有效。

否则可能出现这样的情况：

- Worker A 超时。
- 系统把任务交给 Worker B 重试。
- Worker B 完成任务。
- Worker A 之后又返回旧结果，覆盖新结果。

这是实际系统中很常见的竞态条件。

### 3. 任务依赖需要明确

例如：

```go
func canRun(task Task, store TaskStore) bool {
  for _, depID := range task.Dependencies {
    if store.Get(depID).Status != "accepted" {
      return false
    }
  }
  return true
}
```

T3 必须等待 T1、T2 都通过验收。

不要让主管 Agent 根据聊天记录判断“前端好像已经完成，可以开始测试了”。

### 4. 共享资源必须隔离

如果多个 Coding Agent 同时修改同一份文件，可能互相覆盖。

常见做法是给每个 Worker 分配独立工作区、Git 分支或文件修改权限，最后通过 Merge / Review 机制合并。

这与状态同步同样重要，因为**任务状态一致，不代表产物一致**。

## 九、Supervisor 和 Worker 到底交换什么消息？

推荐把消息协议控制在几种明确类型之内。

| 消息             | 方向                      | 用途               |
| ---------------- | ------------------------- | ------------------ |
| `task.assign`    | Supervisor → Worker       | 分配任务           |
| `task.started`   | Worker → Orchestrator     | 确认开始           |
| `task.progress`  | Worker → Orchestrator     | 汇报进度           |
| `task.blocked`   | Worker → Supervisor       | 请求决策或补充信息 |
| `task.completed` | Worker → Supervisor       | 提交交付物         |
| `task.failed`    | Worker → Orchestrator     | 失败报告           |
| `task.cancel`    | Supervisor → Worker       | 取消任务           |
| `task.accepted`  | Supervisor → Orchestrator | 验收通过           |

这里的箭头表示逻辑上的消息接收方，实际通信可以统一经过 Orchestrator 或事件总线。

一个比较完整的 Worker 返回结果可以设计成：

```json
{
  "task_id": "T1",
  "attempt_id": "T1-A1",
  "status": "completed",
  "summary": "商品详情页前端功能已实现",
  "artifacts": [
    {
      "type": "git_commit",
      "uri": "git://repo/commit/abc123"
    }
  ],
  "validation": {
    "tests_passed": 12,
    "tests_failed": 0
  },
  "issues": [],
  "next_actions": [
    "等待购物车 API 集成"
  ]
}
```

建议 Supervisor 只读取这个结构化结果、必要的日志摘要，以及相关交付物。

不要把所有 Worker 的完整思考过程、工具调用记录和对话历史都无差别地塞回 Supervisor。这会显著增加上下文和 Token 成本，也容易带来相互干扰。

## 十、Supervisor 应该什么时候重新参与？

不建议 Supervisor 每收到一条进度消息，就进行一次 LLM 推理。

更合理的方式是使用事件触发决策：

- 任务完成

  主管验收结果，决定继续、重做或调整后续计划。

- 任务阻塞

  Worker 无法自行解决，主管决定补充信息、调整任务或转交其他 Agent。

- 任务需要重规划

  例如某个 API 不可用，原有任务依赖关系已经不成立。

- 需要审批或关键决策

  例如生产环境发布、敏感数据操作或费用超预算。

心跳、一般性进度、常规重试等行为，直接由 Orchestrator 处理即可。

## 十一、工程实践中最容易忽略的问题

| 问题         | 典型后果                    | 推荐解决方式                 |
| ------------ | --------------------------- | ---------------------------- |
| 任务拆分不合理 | Worker 相互等待、重复工作 | 按上下文和依赖拆分，使用 DAG |
| 上下文传递丢失 | 子 Agent 理解错目标       | 结构化任务契约 + 产物引用    |
| 状态不一致   | 任务重复执行或错误覆盖      | 数据库权威状态 + 版本控制    |
| 错误相互传播 | 多个 Agent 一起得出错误结论 | 独立验证与外部证据           |
| 无限委派     | Token 和执行时间失控        | 最大递归深度与预算上限       |
| 权限过大     | 误删、误改、越权操作        | 最小权限 + 人工审批          |
| 共享资源冲突 | 并发文件覆盖、数据竞争      | 隔离工作区、锁或事务         |
| 缺少可观测性 | 无法定位失败原因            | Trace、事件日志、成本统计    |

其中有三个问题尤其重要。

### 1. Agent 之间达成一致，不等于结果正确

假设 Research Agent 提供了错误数据，Analysis Agent 基于这份数据推理，Review Agent 又只检查逻辑是否通顺。

最后三个 Agent 都认为结果正确，但它们实际上共享了同一个错误前提。

因此需要独立验证机制，例如外部事实检查、测试用例、Schema 校验和来源交叉验证。

### 2. Worker 不能无限创建下级 Agent

如果 Supervisor 可以创建 5 个 Worker，每个 Worker 又创建 5 个子 Agent，那么三层结构就可能产生 125 个末端 Agent。

生产系统至少要设置 `max_depth`、`max_parallel_agents`、`token_budget`、`tool_call_limit`、`timeout` 和 `retry_limit` 等限制。

并且子任务预算应该从父任务预算中分配，而不是无限增加总预算。

### 3. 不能把自然语言聊天当作可靠的任务状态管理

例如 Worker 回复“我已经完成 80%”，这只是自我报告，不一定对应可验证的进度。

更可靠的是记录实际里程碑、工具执行结果和交付产物。

这里有一个重要原则：

**让 LLM 负责需要智能判断的工作，让确定性程序负责必须严格正确的工作。**

例如任务如何拆解可以交给 Supervisor，但任务是否已超时、是否重复执行、是否满足依赖条件，最好由程序明确判断。

## 十二、如何从零开始构建，以及不同规模怎么选型？

### 1. 四个阶段

1. 阶段一：建立 Single-Agent 基线

   先让一个 Agent 能够完成端到端任务，并记录成功率、耗时、Token、工具调用次数及人工干预次数。

   没有基线，就无法证明 Multi-Agent 是否真正提高了性能。

2. 阶段二：只增加一个明确的子 Agent

   例如把独立资料检索交给 Research Agent，主管只接收结构化结果。比较增加 Agent 前后的质量与成本。

3. 阶段三：引入任务依赖和持久化状态

   当出现多个并行任务、异步执行和故障恢复需求时，再引入 DAG、状态数据库、消息队列、幂等处理及心跳机制。

4. 阶段四：形成可观测、可评估的系统

   持续记录端到端成功率、P50/P95 延迟、每个成功任务的成本、失败恢复率和安全事件。只有当收益明确时才继续扩大 Agent 数量。

### 2. 不同规模的系统，应该选择什么架构？

| 规模            | 推荐架构                      | 状态机制                |
| --------------- | ----------------------------- | ----------------------- |
| 2～3 个 Agent   | Supervisor + Agents as Tools  | 运行时状态 / 会话       |
| 4～10 个 Agent  | Supervisor + DAG Orchestrator | 持久化任务状态          |
| 10～30 个 Agent | 分层 Supervisor + Worker Pool | 数据库 + 消息队列       |
| 大规模异步任务  | Hierarchical + Event-Driven   | 事件总线 + 持久化状态机 |

这些数字只是选型参考，不是技术上的硬阈值。任务复杂度、运行时长、失败成本通常比 Agent 数量更重要。

### 3. 复杂任务的推荐架构

从零开发一个能够处理复杂任务、支持多个 Worker 并发、长期运行且需要故障恢复的 Multi-Agent 系统：

> Supervisor + DAG Orchestrator + Worker Pool + Persistent State

各自职责如下：

| 组件                       | 最核心的职责                               |
| -------------------------- | ------------------------------------------ |
| Supervisor（LLM）          | 任务拆解、选择 Agent、动态重规划、语义验收 |
| Orchestrator（代码）       | DAG 调度、生命周期、并发、超时、重试       |
| Worker（Agent）            | 执行专业任务、产出结果、汇报异常           |
| State Store（数据库）      | 保存权威任务状态、执行版本和事件记录       |
| Artifact Store（文件存储） | 保存代码、文档、测试报告等实际产物         |

这个架构最关键的设计原则是：

**LLM 决定任务应该如何推进，但不要让 LLM 成为任务执行状态的唯一事实来源。**

同时还有一个更深层的设计选择：Supervisor 是否允许 Worker 再创建新的 Worker？

如果允许，就从单层 Manager / Worker 模式演化为真正的 Hierarchical Multi-Agent。这会引入子任务所有权、递归深度限制、资源预算继承、取消传播和跨层状态聚合等问题。

这也是简单的 Multi-Agent Demo 与可投入生产的 Multi-Agent 系统之间最重要的架构分界之一。

## 十三、如何判断 Multi-Agent 是否真正比单 Agent 更好？

不要只评估“结果看起来更完整”。建议至少比较以下指标：

| 维度     | 核心指标                      |
| -------- | ----------------------------- |
| 质量     | 端到端任务通过率              |
| 速度     | P50、P95 完成耗时             |
| 成本     | 每个成功任务的 Token / API 成本 |
| 可靠性   | 超时率、失败率、恢复成功率    |
| 协作效率 | 任务交接失败率、重复工作量    |
| 安全性   | 越权操作、审批绕过、安全事件  |

最值得关注的通常是单位成功任务成本。

因为某个 Multi-Agent 系统即使单次请求成本较高，只要成功率显著提升、人工返工大幅减少，仍然可能在业务上更划算。

## 十四、框架怎么选？

| 框架或方案            | 适合的主要需求                                 |
| --------------------- | ---------------------------------------------- |
| OpenAI Agents SDK     | Agents as Tools、Handoff、工具集成、运行时编排 |
| LangGraph             | 显式状态图、复杂流程、条件分支和持久化         |
| AutoGen               | 多 Agent 消息协作、实验和研究                  |
| 自研 Orchestrator     | 需要深度集成业务系统、权限和内部任务引擎       |

可以参考以下设计：

- [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)：适合显式状态图、条件路由、持久化及可恢复工作流。
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)：支持 Manager、Agents as Tools、Handoff 和运行时编排。
- [Microsoft AutoGen](https://microsoft.github.io/autogen/)：适合多 Agent 消息协作及团队式交互。

对于大多数工程团队，更建议从现成 Agent SDK 加少量自有编排代码开始，而不是一开始就自研整个 Multi-Agent Runtime。

## 十五、Multi-Agent 的真正价值在哪里？

未来更有价值的 Multi-Agent 系统，不一定拥有更多 Agent，而是拥有三个能力：

第一，**能够识别哪些任务应该拆分。** 不是默认并发，而是根据依赖、上下文和成本动态决策。

第二，**能够可靠协作。** 包括任务所有权、状态一致性、权限隔离、冲突处理和异常恢复。

第三，**能够证明协作有价值。** 每增加一个 Agent，都能够用成功率、速度、成本或安全性指标解释为什么需要它。

这也带来一个很重要的架构取向：

> Multi-Agent 不应该被设计成一个由很多 AI 角色组成的虚拟公司，而应该被设计成一个拥有明确责任边界、可验证交付物和可靠运行时的分布式任务执行系统。
