# Steering 与 Follow-up

> 本文档目的：介绍 agent 运行中追加用户消息的注入机制——steering、follow-up 与 nextTurn 的定义、注入切点、区别、循环结构定位与业界实现。

## 一、定位：运行中的用户消息注入

Agent 执行长任务（读代码 → 修代码 → 跑测试）期间，用户追加输入有三种意图：

| 意图 | 说法 |
|------|------|
| 纠偏 | 「方向走偏了，现在就改」 |
| 接续 | 「做完之后顺便再做这件事」 |
| 预置 | 「下次让你做 X 时，记得带上 Y」 |

传统交互只有两条路：等 agent 跑完再发——纠偏迟到，错误方向的工作已经做完；abort 后重发——丢弃已积累的上下文，从零开始。

两类机制的共同底座：**消息队列 + agent loop 在固定切点 poll**。消息在切点追加到消息历史末尾（append-only），随整条历史发给模型——用注入替代中断。Agent Loop 本身见 [Harness.md](./Harness.md)。

## 二、Steering（转向）

**定义**：turn 进行中注入、改变当前任务方向的用户消息。

- **切点**：当前 turn 的工具调用结算后、下一次 LLM 调用前——消息在下一次 assistant 响应前 push 到消息末尾，自然插入对话流的当前点
- **效果**：agent 在同一个 run 内调整方向，已积累的上下文与已完成的工作全部保留，无需 abort 重来
- **队列语义**（pi）：每个切点默认只取一条（one-at-a-time），余下等下个切点；连续多次打断时可切换为一次清空（all），打包成一次干预

典型话术五类（gemini-cli 文档）：

| 类型 | 示例 |
|------|------|
| 纠正路径 | "Actually, the utilities are in src/common/utils." |
| 跳过步骤 | "Skip the unit tests for now and just focus on the implementation." |
| 补充上下文 | "The User type is defined in packages/core/types.ts." |
| 重新定向 | "Stop searching the codebase and start drafting the plan now." |
| 处理歧义 | "Use the existing Logger class instead of creating a new one." |

## 三、Follow-up（跟进）

**定义**：turn 即将结束时取出、作为新一轮 turn 启动的接续任务。

- **切点**：模型不再产出工具调用、agent 即将结束（agent_end）时，poll follow-up 队列
- **效果**：链式任务自动化——提前排队「改完 bug 后提交代码」，agent 完成当前任务后自动接续，无需等用户回来再发
- **机制**：follow-up 消息取出后作为下一轮的 steering 消息注入、启动新 turn——底层机制与 steering 复用，切点不同

**来源**：机制（队列、切点 poll、注入）由 harness 提供；往队列放消息的主体有四类：

| 主体 | 机制 | 系统 |
|------|------|------|
| 用户 | pi TUI 键位显式排队（Enter 发 steer、Alt+Enter 发 follow-up）；SDK 宿主应用编程调用 `session.followUp(text)`（OpenClaw 即此类集成）；Claude Code 忙时排队，当前 turn 结束后运行（queued user prompt） | pi、Claude Code |
| 扩展 | agent_end 后 extension handler 调 `steer()` / `followUp()` 入队，驱动追加轮次——pi 会话层 `_handlePostAgentRun()` 的第三个检查项 | pi |
| harness 自动 | 后台工作完成的唤醒（后台 shell run_in_background、Monitor、后台 subagent、scheduled wakeup）；定时到点（`/loop` → CronCreate：到期任务低优先级入队，只在空闲时触发）；异步工具结果晚到（async tool calling：应用晚交结果触发续跑） | Claude Code、OpenAI |
| 外部系统 | MCP server 主动向会话推送消息（Channels） | Claude Code |

用户怎么让一条忙时消息走 steer 还是 follow-up：pi 用键位显式选择；Happier 的 pending queue 用设置分类（排队消息可编辑、排序，待 agent 空闲送达）。

触发条件：**工作完成于 turn 结束之后**。turn 内完成的前台工具结果作为 Observation 回填当前 turn；跨 turn 的完成事件每件生成一条——消息历史 append-only，晚到的结果无法插入历史，只能以新消息追加。

**计划任务不占 follow-up**。plan（如 TodoWrite 写入的任务列表）是上下文里的数据，不是队列里的消息：任务推进靠内层循环在同一 run 内跨多个 turn 连续完成，零 follow-up——plan 的作用正是让内层循环不停。仅当任务被外包给后台 subagent 时，其完成事件才按上述触发条件生成 follow-up：n 项全在主线程做则 0 条，全部外包则 n 条。

**非消息续跑不是 follow-up**。pi 会话层 `_handlePostAgentRun()` 在 agent 停下后依次检查三件事：可重试的瞬时错误（auto-retry）、上下文超限（auto-compact 后重跑）、扩展在 agent_end 阶段排队的新消息。前两种命中时调 `agent.continue()` 空转续跑——不入队、不追加消息，是会话层直接驱动的追加轮次；只有第三种走 follow-up 消息通道。

**分层规律**：模型是唯一不能直接入队的主体——没有 followUp 工具，只能发起后台任务、定时器，借 harness 之手在未来生成 follow-up。四类主体与模型的关系：注入的执行权永远在 harness，其余主体只有排队权（与上下文工程的决策权梯度同构，见 [上下文工程.md](./上下文工程.md)）。

## 四、区别

| | Steering | Follow-up |
|---|---|---|
| 时机 | turn 进行中（工具调用之间） | turn 结束时，触发新 turn |
| 意图 | 纠偏、改方向、补约束 | 追加、接续下一个任务 |
| 循环位置 | 内层循环内消费 | 外层循环消费 |
| 对当前任务 | 改变正在做的事 | 不打断，只接续 |

## 五、循环结构中的位置

调用层级（从内到外）：**模型推理**（一次 LLM 调用）→ **turn**（一次推理 + 若干工具执行）→ **run**（一次 agentLoop 执行，一条用户消息引发的全部工作）。

双层循环（pi 的 agentLoop）：

```
用户输入 → run
  ├─ 内层循环（ReAct 循环）：turn → turn → …，直到模型不再产出工具调用
  │    ↑ steering 在 turn 边界注入，改变当前任务方向
  ├─ 内层退出（agent 本要结束）→ poll follow-up 队列
  └─ 有排队 → 注入新 turn、重入内层；队列空 → run 结束
```

- **内层循环**：驱动当前任务的工具调用链。单次用户输入本来就会触发多次模型推理——每个工具回合一次；模型零工具调用直接回答时才恰好一次
- **外层循环**：follow-up 的作用层——不在推理层多调几次，而在 run 层让本该结束的继续
- **自动 follow-up 可链式触发**：每个后台完成事件各生成一个 wake-up、各触发一轮续跑；scheduled wakeup 使 agent 定时自我续跑。续跑链需要兜底——AgentBox 以 15s 无新 turn 的 grace + backgroundTaskTimeoutMs（默认 30 分钟）约束，防止 run 永不结束

第三个同类 API：**nextTurn(msg)**——在下一次用户新 prompt 之前注入，用于预置用户级上下文。三者共享同一套消息队列 + 切点 poll，按注入切点区分语义。

## 六、业界实现

| 实现 | 机制 |
|------|------|
| Claude Code | 实时 steering：异步双缓冲队列，agent 工作期间注入新指令、约束或重定向，无需重启循环 |
| gemini-cli | Model Steering（实验性，默认关闭）：运行中键入提示，CLI 用快速模型生成一句确认，下一轮次将提示连同内置指令注入，模型重新评估当前计划并调整 |
| OpenAI Responses API | Mid-turn steering：响应进行中经 WebSocket 发送补充指令，模型中途吸收修正与需求变化 |
| pi | steer / followUp / nextTurn 三个 API；TUI 键位 Enter=steer、Alt+Enter=followUp；steering 在 turn 边界消费，follow-up 在 agent_end 消费 |

核心思想一致：把「等 agent 跑完再说话」变成「在安全切点把话插进对话流」，用注入替代中断。
