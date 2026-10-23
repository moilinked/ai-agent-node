# Plan & Execute vs ReAct：核心差异、架构设计与业务适用场景

在 AI Agent 开发中，Plan & Execute（规划-执行）和 ReAct（Reasoning + Acting）是两种典型的 Agent 决策与执行范式。

它们最本质的区别不是有没有规划能力，而是：

> ReAct 以当前环境反馈驱动下一步行动；Plan & Execute 以全局任务计划组织后续行动。

可以简单理解为：

- ReAct：边观察、边思考、边行动。 强调局部决策和实时适应。
- Plan & Execute：先规划整体任务，再逐步执行。 强调全局目标、任务拆解和过程控制。

但要注意：ReAct 也可以进行长期思考，Plan & Execute 也可以动态调整计划。两者不是严格对立的架构。

## 一、先看两种范式的执行过程

![ReAct 执行过程](https://oilinil.cn/api/files/2026/10/bc907653fa11f43ec81e6f6fbe76918d/plan-execute-vs-react-img1.webp)

![Plan & Execute 执行过程](https://oilinil.cn/api/files/2026/10/211f6b764e860f366ae10f4df827dfa9/plan-execute-vs-react-img2.webp)

ReAct 最初由 Yao 等人在 2022 年提出，强调推理与行动交替进行，让外部观察反馈影响后续决策。Plan & Execute 则强调将高层规划和具体执行分离。

## 二、核心差异对比

| 比较维度     | ReAct                          | Plan & Execute                   |
| ------------ | ------------------------------ | -------------------------------- |
| 决策方式     | 每轮根据上下文决定下一步       | 先规划总体步骤，再分步执行       |
| 规划粒度     | 主要是当前行动，可包含短期计划 | 显式的全局任务计划               |
| 环境适应性   | 通常较强                       | 取决于 Replan 设计               |
| 长期目标管理 | 依赖上下文与状态管理           | 显式计划便于跟踪                 |
| 工具使用     | 逐步探索、动态选择             | 围绕当前子任务执行               |
| 任务分解     | 可以动态形成                   | 通常在开始时形成                 |
| 并行执行     | 可扩展，通常以串行循环为基础   | 可通过 DAG 组织并行任务          |
| 模型使用     | 通常持续调用决策模型           | Planner 和 Executor 可用不同模型 |
| 可观测性     | 观察行动轨迹                   | 额外具备显式计划及步骤状态       |
| 主要风险     | 目标漂移、循环探索、调用过多   | 初始计划错误、过度规划、计划过时 |

这里有两个容易误解的地方。

第一，**Plan & Execute 不一定比 ReAct 更省 Token。** 如果规划和重规划频繁，可能增加调用成本；如果复杂任务能由小模型或确定性程序完成子步骤，则可能降低成本。LangChain 对不同版本 Plan & Execute 的介绍也体现了这种差异。

第二，**Plan & Execute 不一定更可靠。** 如果 Planner 一开始就错误理解任务，Executor 又机械执行，可能把早期错误放大。可靠性来自计划验证、执行反馈和重新规划，而不是来自“先规划”本身。

## 三、从决策机制看，二者究竟有什么不同？

### 1. ReAct：局部决策驱动

ReAct 的核心是根据当前状态选择下一步行动。

可以抽象为：

$$
a_t = \pi_\theta(a \mid g, h_t, o_t)
$$

其中：

- $g$：用户的任务目标。
- $h_t$：已有历史。
- $o_t$：最新环境观察。
- $a_t$：当前选择的行动。

其特点是：**每一步决策都可以充分利用最新反馈，而不必遵守某个事先确定的行动序列。**

例如，用户询问：

> 为什么我的 Go 服务 CPU 使用率突然升高？

ReAct Agent 的执行可能是：

```text
查询 CPU 指标
      ↓
发现某个实例 CPU 异常
      ↓
查询该实例的 pprof 数据
      ↓
发现 JSON 序列化占用较高
      ↓
检查对应请求日志
      ↓
发现某接口流量突增
      ↓
输出诊断结论与证据
```

这里有一个重要特点：

Agent 只有拿到 CPU 数据后，才知道应该查询 pprof；只有发现序列化热点后，才知道下一步该分析哪类请求。

如果强行事先确定完整路径，反而可能进行大量无用操作。

**因此，ReAct 更适合下一步行动高度依赖上一轮观察结果的任务。**

### 2. Plan & Execute：全局目标驱动

Plan & Execute 首先将用户目标拆解成一个任务计划：

$$
P = \operatorname{Planner}(g, c_0)
$$

例如：

$$
P = \{p_1, p_2, p_3, \ldots, p_n\}
$$

Executor 按照计划执行，并记录每个步骤的结果：

$$
r_i = \operatorname{Executor}(p_i, c_i)
$$

当发现计划无法继续执行时，可以生成更新后的计划：

$$
P' = \operatorname{Replan}(P, r_1, \ldots, r_i, g)
$$

例如，用户要求：

> 对比 Java、Go、Python 三种语言构建 AI Agent 的长期维护成本，并生成技术选型报告。

Planner 可以提前拆解为：

1. 定义比较维度。
2. 调查三种语言的 Agent 生态。
3. 对比并发、部署和工程成本。
4. 评估不同业务场景。
5. 汇总结论。
6. 生成最终报告。

这里多数子任务在开始时就能被识别。

甚至可以把 Java、Go、Python 的生态调查并行执行，等全部完成后再汇总。

因此，**Plan & Execute 更适合目标明确、任务可分解、步骤之间存在可管理依赖关系的复杂任务。**

## 四、适配什么业务场景？

### 推荐选型矩阵

| 业务场景                      | 推荐范式                   | 主要原因                       |
| ----------------------------- | -------------------------- | ------------------------------ |
| 智能客服动态排障              | ReAct                      | 下一步依赖用户回答及工具结果   |
| 系统故障根因分析              | ReAct                      | 调查路径无法提前完全确定       |
| 网页浏览和页面探索            | ReAct                      | 页面状态不断变化               |
| 数据库异常诊断                | ReAct                      | 需要不断验证假设               |
| 多来源市场研究报告            | Plan & Execute             | 易于按研究维度拆解             |
| 跨系统数据分析                | Plan & Execute             | 可定义步骤、依赖和验收条件     |
| 大型项目迁移                  | Plan & Execute + ReAct     | 全局任务明确，局部问题动态变化 |
| Coding Agent 修复 Bug         | ReAct 或混合               | 需要持续读取代码、测试并修正   |
| Coding Agent 开发完整 Feature | 混合                       | 需要全局规划和局部探索         |
| 多系统业务自动化              | 固定 Workflow + 局部 Agent | 已知步骤应优先确定性执行       |
| Deep Research                 | 混合                       | 研究计划明确，检索路径动态变化 |

下面选三个典型案例具体分析。

### 场景 A：Coding Agent 修复未知 Bug

倾向 ReAct 动态探索问题空间

![plan-execute-vs-react-img3](https://oilinil.cn/api/files/2026/10/2e25472358b736ad9cac68d29063d08a/plan-execute-vs-react-img3.webp)

这里最重要的不是预先规划十个步骤，而是让 Agent 依据真实测试结果不断缩小问题范围。

当然，Agent 可以有一个高层计划，例如“复现 → 定位 → 修复 → 测试”。这与 ReAct 并不冲突。

### 场景 B：大型系统版本迁移

倾向 Plan & Execute

长周期、多子任务、依赖明确

![plan-execute-vs-react-img4.webp](https://oilinil.cn/api/files/2026/10/910a5da15b35593774b2cb7bfea743cc/plan-execute-vs-react-img4.webp)

这里 Plan & Execute 的优势在于：

- 整体迁移目标清晰。
- 可以显式管理步骤依赖。
- 适合维护任务进度和 Checkpoint。
- 不同子任务可以独立验证。
- 部分步骤可以并行执行。

但修复具体的不兼容 API 时，子任务 Executor 仍然可以使用 ReAct。

### 场景 C：抽奖活动的领奖与发奖流程

假设业务流程是：

```text
验证中奖记录
      ↓
验证领奖资格
      ↓
锁定领奖状态
      ↓
创建发奖任务
      ↓
发送通知
      ↓
记录发放结果
```

**这个场景其实不应该优先使用 ReAct 或由 LLM 生成 Plan & Execute。**

因为业务流程已经确定，而且涉及：

- 状态一致性
- 数据库事务
- 幂等性
- 权限校验
- 奖品发放
- 失败重试

此时更合理的是使用确定性 Workflow / 状态机。

如果需要 AI，可以在特定节点增加 Agent，例如让 Agent 分析复杂的异常原因，或者辅助处理用户的非标准咨询。

这体现了一个很重要的选型原则：

> 不是所有多步骤业务流程都需要 Agent。固定规则优先使用确定性程序；只有需要语义理解、探索或自主决策的环节才引入 Agent。

## 五、真实工程中，更推荐混合架构

对于复杂的生产级 Agent，更倾向于：

**外层 Plan & Execute + 内层 ReAct + 独立 Verifier。**

这种架构既能保留长期计划，又能让具体步骤根据工具反馈自主探索。

![plan-execute-vs-react-img5](https://oilinil.cn/api/files/2026/10/c0e8f87d40f14b2b3a69e010baf2f72f/plan-execute-vs-react-img5.webp)

LangGraph 的 Plan-and-Execute 教程就采用了类似思路：用 ReAct Agent 执行当前计划步骤，然后通过 Replanner 决定继续、调整计划或结束任务。

[LangChain](https://github.langchain.ac.cn)


### 在 Harness 中应该如何实现？

建议明确分离四个模块。

| 模块      | 职责                         | 不能做什么             |
| --------- | ---------------------------- | ---------------------- |
| Planner   | 任务分解、依赖关系、验收标准 | 直接认定任务成功       |
| Executor  | 使用工具完成当前子任务       | 随意修改全局任务目标   |
| Verifier  | 根据真实工具结果检查完成条件 | 仅根据模型自述判断成功 |
| Replanner | 根据新证据调整剩余计划       | 忽略用户原始约束       |

执行任务时维护结构化状态：

```json
{
  "goal": "修复登录服务并通过测试",
  "plan_version": 2,
  "steps": [
    {
      "id": "step_1",
      "goal": "复现登录失败",
      "status": "completed"
    },
    {
      "id": "step_2",
      "goal": "定位根因",
      "status": "running"
    },
    {
      "id": "step_3",
      "goal": "修复并验证",
      "status": "pending"
    }
  ],
  "execution": {
    "max_tool_calls": 40,
    "remaining_tool_calls": 18
  }
}
```

这里全局 Plan 充当任务状态的一部分，ReAct 则负责当前子任务内的动态探索。

特别重要的是，**Replan 不应该在每次工具调用后无条件触发**，否则就退化成一个成本很高的多层 ReAct。

更合理的 Replan 触发条件包括：

- 当前步骤连续失败。
- 发现原计划依赖的前提不成立。
- 用户改变需求。
- 出现会影响后续步骤的新事实。
- 步骤执行超出预算。
- 关键验收条件发生变化。

## 六、性能和成本，哪个更优？

无法简单断言 Plan & Execute 比 ReAct 更快或更便宜。

假设一个任务需要 10 次工具调用。

两种执行成本的示意对比

| 项目           | ReAct          | Plan & Execute  |
| -------------- | -------------- | --------------- |
| 全局规划调用   | 0 次独立调用   | 1 次            |
| 工具间决策调用 | 约 10 次       | 取决于 Executor |
| 重新规划       | 可在循环中调整 | 0 至多次        |
| 并行处理       | 需要额外设计   | 可通过 DAG 调度 |

逻辑示例，非实测数据。不同框架可以合并模型调用或使用并行工具调用。

Plan & Execute 的潜在效率优势主要来自三点：

1. **模型分级**：全局规划使用强模型，简单步骤使用轻量模型甚至确定性程序。
2. **任务并行**：对于相互独立的子任务，通过 DAG 并行执行。
3. **上下文隔离**：Executor 只接收完成当前步骤必需的信息，不必反复携带整个任务的全部历史。

但如果任务本身只有两三步，规划调用很可能只会增加开销。

对于需要大量试错的任务，ReAct 也可能因为省去全局规划而更有效。

## 七、一个实用的架构选型判断方法

实际选型时，最重要的变量不是任务名称，而是两个属性：

- **任务可分解性**：能否在执行前合理确定主要步骤？
- **环境不确定性**：每一步的结果是否会显著改变后续行动？

![plan-execute-vs-react-img6](https://oilinil.cn/api/files/2026/10/ee9008c7f6f71e70f1db6668009f8b09/plan-execute-vs-react-img6.webp)

## 八、最终结论

ReAct 是一种局部闭环决策范式；Plan & Execute 是一种全局任务组织范式。

ReAct 擅长回答：

> 我现在观察到了什么？下一步应该做什么？

Plan & Execute 擅长回答：

> 为了完成整个目标，需要做哪些事情？目前进行到哪里？是否要调整计划？

从生产级 Agent 的角度，优先考虑三个层次：

- **确定性 Workflow**：控制必须按规则运行的业务步骤。
- **Plan & Execute**：管理复杂任务的全局目标、依赖和进度。
- **ReAct**：处理局部任务中的探索、工具反馈和动态决策。

如果要开发类似 Claude Code、Codex、Pi 的复杂 Coding Agent，不必在两种范式中二选一。更值得研究的是：**什么时候需要全局规划、什么时候允许局部自主探索，以及何时应该触发重新规划。**

这三个决策，往往比单纯选择 ReAct 或 Plan & Execute 更影响 Harness 的实际可靠性。

参考：[ReAct 原始论文](https://arxiv.org/abs/2210.03629?utm_source=chatgpt.com) · [LangChain Plan-and-Execute](https://www.langchain.com/blog/plan-and-execute-agents?utm_source=chatgpt.com) · [LangGraph 规划执行教程](https://github.langchain.ac.cn/langgraph/tutorials/plan-and-execute/plan-and-execute/?utm_source=chatgpt.com)