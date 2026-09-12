# Agent Harness

> 本文档目的：介绍 Agent Harness 的定义与词源、组件构成、middleware 在其中的角色与挂载关系，验证、故障恢复、自演进的实现手段，以及厂商差异维度。

## 一、定义

**Harness（挽具）**：包在模型外面、让模型能真正干活（或被真正测量）的工程系统。行业公式：

```
Agent = Model + Harness
```

模型负责"想"，harness 负责其余一切工程工作：任务拆解、工具执行、上下文管理、错误重试、结果闭环。

Anthropic 定义：*"the software scaffolding around a model: the loop, tools, context management, and guardrails that turn raw intelligence into a working agent."* 直白表述：Claude Code 是 harness，Claude 是里面的模型。

### 词源

- 本义"马具/挽具"——不替代动力，而是驾驭动力、把马力变成有用的功
- 电气工程：cable harness（线束），组织与保护电缆
- 软件测试：test harness（测试床），包裹被测代码、喂输入、收输出的自动化环境
- AI 领域的 harness 直接继承自 test harness

### 两个分支

| | Evaluation Harness（评估框架） | Agent Harness（智能体运行框架） |
|---|---|---|
| 出现时间 | 2021（EleutherAI 的 lm-evaluation-harness） | 2023-24 称 scaffold，2025-26 统一称 harness |
| 职能 | 加载任务集 → 格式化 prompt → 跑模型 → 评分 → 报告，测量模型 | 提供 loop、工具、上下文管理、权限，让模型在环境中行动 |
| 典型例子 | lm-evaluation-harness、SWE-bench 的运行器 | Claude Code、Codex CLI、DeepSeek Harness |

两者共享同一分工：**模型提供智能，harness 提供使用智能所需的一切其他东西**。

延伸阅读：[Harness工程.md](../../06-经典文章/Harness工程.md)（Lilian Weng，用验证回路做自我改进的 harness 工程）。

## 二、组件构成

| # | 组件 | 职能 | 比喻 |
|---|------|------|------|
| 1 | Agent Loop（工具调用循环，最核心） | 模型只发 `tool_use` 请求；harness 执行 → 回填 → 模型再决策，循环至完成 / 轮数上限 / 预算上限 | 脊柱 |
| 2 | 工具层 | Bash、Read/Write/Edit、Grep/Glob、WebSearch | 手 |
| 3 | 上下文管理 | compaction、记忆、文件系统 | 工作台 |
| 4 | 权限模型 | 直接执行 / 需确认 / 禁止 三档 | 缰绳 |
| 5 | 执行环境 / 沙箱 | 工具运行的 substrate，划定风险边界 | 围栏 |
| 6 | System Prompt 分层 | 固定可缓存部分 + 动态部分 | 岗位说明书 |
| 7 | 验证钩子 / Sub-agent | 后台校验执行结果，防"自我美化、虚报完成" | 质检 |

Agent Loop 即 ReAct 的 Thought-Action-Observation 循环的工程化（见 [ReAct.md](./ReAct.md)）；上下文管理详见 [上下文工程.md](./上下文工程.md)。

## 三、Middleware 在 Harness 中的角色

### 定位

Middleware 不是与 harness 并列的概念，也不是第 8 个组件，而是 harness 内部实现横切控制的机制：嵌在 Agent Loop 数据路径上的可插拔关卡（洋葱模型）：

```
请求 → [中间件1 [中间件2 [中间件3 → 模型调用 / 工具执行]]] → 响应反向穿回
```

每层在"进""出"两个时机可做四件事：**查看、改写、放行、拦截（短路）**。

### 四种角色

1. **控制点**：工具调用前做审批（human-in-the-loop）、权限校验、参数 schema 验证；模型输出后做敏感信息过滤、格式校验
2. **上下文守门人**：压缩历史、注入检索结果、隐藏无关工具定义——compaction 在 LangChain 架构里即是一个 middleware
3. **解耦者**：日志、追踪、缓存、重试、限流等横切关注点全部外置，核心循环只保留"调模型 → 执行工具 → 回填"最小逻辑
4. **可组合性来源**：关卡可堆叠、排序、单独启停；包裹顺序本身表达语义（"重试包住审批"与"审批包住重试"是两种语义）

边界：决策属于模型，实际执行属于工具，整体架构属于 harness——middleware 只做中介与控制。

### 与组件的关系：功能视图 × 机制视图

组件清单回答"harness 需要哪些能力"（功能视图）；middleware 回答"能力怎么装进去"（机制视图），两者正交。

- **主干**：Agent Loop 是 middleware 的宿主
- **关卡型能力**（上下文管理、权限执行点、Prompt 动态注入、验证钩子）：天然位于关卡位
- **底座型能力**（工具层、沙箱）：是承载物，middleware 只在通往它们的路上把关

```
┌─ Agent Loop ────────────────────────┐
│ [关卡: 上下文改写 / prompt 注入]     ← 组件 3、6
│      ↓                              │
│   模型调用                           │
│      ↓                              │
│ [关卡: 权限审批 / 参数校验]          ← 组件 4
│      ↓                              │
│   工具执行（跑在沙箱上）             ← 组件 2、5
│      ↓                              │
│ [关卡: 结果过滤 / 验证]             ← 组件 7
└─────────────────────────────────────┘
```

### 同一能力的不同挂载机制

| Harness | 机制 | 特点 |
|---|---|---|
| LangChain / LangGraph 1.0 | middleware | 拦截 agent loop 的请求/响应，可改写 state、短路、后处理 |
| Claude Code | hooks | 事件驱动（PreToolUse、PostToolUse 等生命周期事件） |
| NeMo Guardrails | rails | 对话级流程编排 |
| Microsoft Agent Framework | middleware + decorators | 官方将 middleware 列为 harness 构建块（审批、观测、有界循环） |

同一能力（compaction、审批等）在不同 harness 中选择不同挂载方式。功能生态位相同——都落在那几个"关卡位"上；组件清单稳定，挂载机制是各家选型。

## 四、验证、故障恢复与自演进

### 1. 验证钩子

对执行结果做事后客观校验的关卡，针对的失败模式是模型"自我美化、虚报完成"——模型既是生成者又是评估者，自评带自偏好偏差。

设计约束：校验者只消费工具执行结果（测试、lint、build 输出），不消费模型生成的文本；验证走独立于生成的通路。

挂载：Claude Code 用 hooks（PostToolUse / Stop 事件触发）；LangChain 用 post-response middleware；Claude Code 泄露架构中为后台分类器，只读工具执行结果。

### 2. 故障恢复

| 层 | 手段 | 处理的故障 |
|---|---|---|
| 模型层 | 错误作为 Observation 回填，模型调整策略重试 | 工具调用失败、参数错误 |
| 系统层 | retry + 指数退避、超时控制、有界循环（轮数/预算硬上限）、降级（上下文超窗触发 reactive compact、切换模型） | 网络抖动、限流、上下文超窗、死循环 |
| 状态层 | 会话持久化 + checkpoint，崩溃后恢复 | 进程中断、长任务交接 |
| 环境层 | 沙箱隔离 + 环境重置 | 失败波及范围控制 |

所有权：四层的机制都由 harness 提供；第一层的修复动作由模型完成（harness 只提供"错误回填"通道，修复靠模型智能），2-4 层是纯结构性容错（harness 独自吸收，模型无感知）。reactive compact 见 [上下文工程.md](./上下文工程.md)。

middleware 适用性边界：作用域为 per-request 的机制（retry、每轮持久化、降级触发）适合做成 middleware；作用域为 per-run（有界循环、checkpoint）或 per-environment（沙箱、重置）的机制由 loop 核心、运行时或环境承载。middleware 提供的是可组合性而非功能本身——同一机制内建（Claude Code）或 middleware 化（LangChain）功能等价。

### 3. 自演进

模型权重不变；改进对象是 harness 自身——运行时产生的经验改变 harness，使下一次运行更强。

描述一个演进动作需要两根轴：**对象轴**（动的是哪个部件）与**深度轴**（改动以什么形式存在、离核心多近）。二者正交。

#### 对象轴：harness 的三个部件层

| 层 | 是什么 | 例子 |
|---|---|---|
| 内容层 | 流经 harness 的数据 | CLAUDE.md、记忆、踩坑记录 |
| 能力层 | 可装卸的能力单元 | skill、插件、工具 |
| 机制层 | 结构与旋钮——又分参数侧与代码侧 | 参数侧：工程化 system prompt、工具 schema、策略配置；代码侧：loop 逻辑、挂载机制、上下文管理实现 |

生产实践的三个通道（对象轴视角）：

| 通道 | 动的部件 | 门控 | 效果 |
|---|---|---|---|
| 规则/记忆写回 | 内容层 | 全自动 | 新约束下次会话生效 |
| Skill/插件固化 | 能力层 | 审批装卸 | 流程固化为可复用单元，运行时装卸 |
| Eval 反馈回路 | 机制层（参数侧为主） | 验证门控 | 指标驱动迭代 |

#### 深度轴：改动形式与门控

```
文本 → 配置 → 代码 → 协议/权重
浅 ────────────────→ 深
门控：全自动 → 审批 → 验证门控 → 人主导
```

- 深度越高，越接近核心，自我指涉风险越大（harness 是控制器，改坏控制器即无物可修）
- **配置与代码是当前的分界线**——生产实践中，配置侧可自动改（自演进），代码侧主要由人改（架构演进）。这是实践分界，非本质分界：研究侧已在推这条线
- 演进趋势是分界线持续右移：硬编码 → 可配置 → 可学习，自演进的作用面随之扩大

门控与「对象 × 深度」的组合对应：

| 改动 | 深度 | 当前门控 |
|---|---|---|
| 写回规则/记忆 | 文本 | 全自动 |
| 装卸 skill/插件 | 配置 | 审批 |
| 调策略参数（压缩阈值、schema） | 配置 | 现产品由人配置；自动调参见于研究（MCE） |
| 改 harness 代码（loop、挂载机制、上下文主路线） | 代码 | 人主导、版本为单位（Cordis 微内核变更仍经仓库级修改） |

#### 研究的进攻路线：深度轴上的阶梯

研究沿深度轴由浅入深推进。每级都是「对象 × 深度」的组合：

| 阶梯 | 对象 × 深度 | 代表工作 |
|---|---|---|
| prompt | 机制层参数 × 文本 | Self-Harness（system prompt 修改） |
| 工具与验证 | 能力层 / 机制层 × 配置与代码 | Self-Harness（工具包装器、验证步骤注入、规划模板更新） |
| 上下文与记忆 | 内容层 / 机制层 × 配置 | ACE（上下文当持续进化的 playbook）、MCE |
| 工作流结构 | 机制层 × 代码 | AFlow / ADAS（搜索工作流结构） |
| harness 代码 | 机制层代码侧 × 代码 | Meta-Harness / AHE / DGM（harness 当代码搜索与改写） |
| 协议与权重联合 | 超出三层（harness 接口、模型侧） | HarnessX / AgentFactory |

两点实测结论：

1. **验证与自检机制收益最大**——模型默认"写完即完成"，验证回路是最便宜的补丁。LangChain 仅改 harness 使 Terminal-Bench 2.0 从 Top 30 升至 Top 5，主因是注入 build-verify 循环与完成前校验 middleware；DGM 涌现出 peer-review
2. **能力层出现自生成方向**——Alita 从成功轨迹生成 MCP 工具，抽象为参数化原语入工具库，按需检索复用，将 pass@k 转化为 pass@1

#### 通用改进回路

各方法形态各异，但共享同一条回路：

```
rollout → 错误分析 → 提案 → 回归验证 → merge
          (归因到可编辑面)  (无回归、净正向才采纳)
```

差异在每一环的机制：搜索算法、元智能体、进化、RL。

可编辑面本身是设计对象。AHE 把它文件化为七类正交组件——system prompt、工具表示、middleware、skill、sub-agent 配置、长期记忆等；只读面冻结，防 reward hacking。

#### 完全体：RSI

自演进的完全体是 RSI（递归自我改进，Recursive Self-Improvement）：系统用当前智能改进"产生智能的机制"，形成复利闭环——更好的 harness → 更强的 agent → 更能改进 harness。

RSI 的作用面有三个：模型权重（self-play、合成数据、持续学习）、训练流水线、部署系统（harness）。harness 自演进是其中工程上最可行的先导路径：权重不可动时，改进 harness 便宜、可验证、立即生效。

分界线两侧均已有研究原型：

- 配置侧：Self-Harness 使三个固定模型在 Terminal-Bench 2.0 上 held-out pass rate 全部提升——40.5%→61.9%、23.8%→38.1%、42.9%→57.1%
- 代码侧：DGM 自改 80 轮，SWE-bench 从 20.0% 升至 50.0%

生产演进已是半自动：LangChain 以 trace 分析 skill 聚合失败轨迹、生成 harness 改动，人只审核采纳环节。验证钩子是自演进的安全阀。故障恢复使 harness 持续运转；自演进使 harness 越用越好。

#### 有效性边界

等算力对照实验（TerminalBench 2.1，加 UnitTest）中，HarnessEvolution 的 pass@1 为 75.8，低于 ParallelSampling 的 86.0 与 SequentialRefinement 的 91.8；且学到的改动任务特定性强、held-out 泛化弱。部分"自进化收益"可能只是多次尝试的算力红利。

适用条件：有提升空间、成败依赖工具与设计的任务。对模型能力本身不足的任务收益有限。

## 五、厂商差异

组件清单层面各厂商趋同（七大组件即最小可用 agent）；差异在匹配方式而非功能有无。

| 差异维度 | 内容 | 例子 |
|---|---|---|
| 模型协同设计（最根本） | harness 与自家模型联合设计，是模型训练分布的一部分——工具 schema 训练期适配、prompt 分层与缓存断点耦合；直接迁移到其他模型效果显著下降 | Claude Code 内置工具无需推理时完整描述；并行调用与规划行为依赖后训练对齐 |
| 挂载机制选型 | 同一能力选不同安装方式，决定扩展模型与生态开放度 | Claude Code 内建+hooks；LangChain 万物皆 middleware；DeepSeek Harness 的 Cordis 微内核一切皆插件、运行时热插拔 |
| 上下文策略侧重 | 同一问题选不同主路线 | Claude Code 大窗口+compaction；Codex CLI 记录时截断；Cursor 外置文件化 |
| 开放边界 | 开源/闭源、产品形态、扩展面 | DeepSeek 开源 vs Claude Code 闭源；CLI / SDK / IDE |

各家优化的是「自家模型 × 自家 harness」的乘积，harness 单独不构成可迁移的竞争力。

## 六、关键认知

1. **能力是「模型 + harness」的联合属性**：同一模型换一套 harness（重试策略、工具 schema、上下文组装方式、步数上限），表现差异巨大。评测换模型时 harness 必须是常量，否则测得的是两者的混合物
2. **范式演进**：Prompt Engineering → Context Engineering → Harness Engineering——瓶颈从"模型聪不聪明"转向"驾驭模型的系统好不好"
3. **易混淆项**：Harness.io 是 DevOps CI/CD 商业平台，与本概念无关；scaffold 是 agent harness 的同义旧词
