# ReAct Agent

> 本文档目的：介绍 ReAct Agent 的核心思想与实现要点，重点说明 function calling 式实现中的循环判停、消息回填、思考模型行为等工程细节。

## 一、什么是 ReAct Agent

ReAct = Reasoning + Acting，Yao 等人 2022 年在论文《ReAct: Synergizing Reasoning and Acting in Language Models》中提出的 Agent 范式，让 LLM 把"推理"和"行动"交错进行。

### 核心循环

每一步产出三元组：

1. **Thought**：模型对当前状态做一步推理——下一步该做什么、为什么
2. **Action**：执行外部动作（搜索、查数据库、调 API、运行代码等），对应一次工具调用
3. **Observation**：动作返回的结果，作为下一步 Thought 的输入

循环直到模型不再调用工具，输出最终答案。

### 解决的问题

- **纯 CoT 推理**：模型只能凭记忆，遇到新事实会幻觉
- **纯 Acting**：模型不停调用工具但没有显式规划，容易跑偏或循环
- **ReAct**：让"推理"指导"行动选择"，让"观察"反过来修正"推理"，二者协同

## 二、两种实现路径

| 路径 | 实现方式 | 稳定性 | 适用场景 |
|------|---------|-------|---------|
| 经典文本解析式 | LLM 输出自由文本，正则解析 `Action: xxx[yyy]` | 脆弱 | 教学、不支持的弱模型 |
| **Function Calling 式（当前主流）** | LLM 直接返回结构化 `tool_calls` 字段 | 高 | 生产环境主流选择 |

主流模型厂商（OpenAI、Anthropic、Google、Qwen 等）已在 API 层原生支持工具调用，输出结构化 JSON。LangChain v0.1+ 推出的 `create_tool_calling_agent` 替代了旧版 ReAct 实现。本文后续内容聚焦 Function Calling 式。

## 三、Function Calling 式 ReAct 详解

### 工作原理

```
用户问题
   ↓
[LLM 推理] → content: "我需要先查天气"  (Thought)
              tool_calls: [{name: "get_weather", args: {city: "北京"}}]  (Action)
   ↓
[系统执行] → 调用 get_weather("北京") → "晴，25°C"  (Observation)
   ↓
[回填 ToolMessage] → 进入下一轮 LLM 推理
   ↓
循环直到 LLM 不再返回 tool_calls，输出最终答案
```

关键特征：**Thought 和 Action 不再混在一段文本里**，而是分别放在 `content` 和 `tool_calls` 两个字段中。

### 完整示例（LangChain + OpenAI）

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate

# 1. 定义工具（类型注解自动转 JSON Schema）
@tool
def get_weather(city: str) -> str:
    """获取指定城市的实时天气"""
    return f"【{city}】晴，25°C，湿度 65%"

@tool
def calculate(expression: str) -> str:
    """执行数学表达式计算"""
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"计算错误：{e}"

# 2. 初始化支持 function calling 的 LLM
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# 3. 准备 Prompt（必须包含 agent_scratchpad 占位符）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个能调用工具的助手，按需使用工具回答问题"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

# 4. 创建 Agent
tools = [get_weather, calculate]
agent = create_tool_calling_agent(llm, tools, prompt)

# 5. 封装为执行器
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10,
    handle_parsing_errors=True,
)

# 6. 运行
result = agent_executor.invoke({
    "input": "北京和上海的天气描述一样吗？如果不一样，两地湿度差多少？"
})
print(result["output"])
```

### 运行时输出示例

```
> Entering AgentExecutor chain...

Thought: 我需要先查北京和上海两地的天气
Action: get_weather
Action Input: {"city": "北京"}
Observation: 【北京】晴，25°C，湿度 65%

Thought: 还需要查上海的天气
Action: get_weather
Action Input: {"city": "上海"}
Observation: 【上海】多云，27°C，湿度 75%

Thought: 两地天气不同，需要计算湿度差 75 - 65
Action: calculate
Action Input: {"expression": "75 - 65"}
Observation: 10

Thought: 已得答案，可以输出
Final Answer: 两地天气不同——北京晴 25°C 湿度 65%，上海多云 27°C 湿度 75%，湿度差 10%。

> Finished chain.
```

## 四、关键工程问题

### 问题 1：怎么判断执行完成？

**唯一可靠判据**：本轮 LLM 响应中**不再包含 `tool_calls`**（或为空数组）。

```python
# 通用判停逻辑
if not response.tool_calls:
    # 模型自己决定不再调用工具 → 循环结束
    final_answer = response.content
    break
else:
    # 还有动作要执行 → 继续
    for tc in response.tool_calls:
        result = execute_tool(tc)
        ...
```

**辅助判停（防御性）**：

- `max_iterations`：硬上限，防死循环（一般 5-10）
- 重复检测：连续 N 次调用相同工具 + 相同参数 → 强制停止
- 工具错误回灌后仍无进展 → 强制停止

**注意**：不要用 "content 中是否包含 Final Answer" 来判停。Function calling 式不强制模型输出 `Final Answer:` 字样，判停信号完全来自 `tool_calls` 字段。

### 问题 2：循环中每个步骤完成后，messages 要添加哪些信息？

OpenAI 协议规定 messages 必须**成对**回填：

**模型返回 tool_calls 时**——把原始 assistant 消息整体追加（包含 content + tool_calls）：

```python
messages.append({
    "role": "assistant",
    "content": response.content,        # Thought 文本，可能为空
    "tool_calls": response.tool_calls,  # 结构化工具调用
})
```

**工具执行完毕后**——每个 tool_call 对应一条 `role: "tool"` 消息，必须带 `tool_call_id`：

```python
for tc in response.tool_calls:
    result = execute_tool(tc)
    messages.append({
        "role": "tool",
        "tool_call_id": tc.id,        # 必填，关联上一步的 tool_call
        "content": str(result),        # Observation，必须是字符串
    })
```

**完整一轮后的 messages 序列**：

```
[system]      系统提示
[user]        用户原始问题
[assistant]   content="需要查天气"  + tool_calls=[查北京天气]   ← Thought + Action
[tool]        content="晴 25°C"      ← Observation 1
[assistant]   content="继续查上海"  + tool_calls=[查上海天气]   ← Thought + Action
[tool]        content="多云 27°C"    ← Observation 2
[assistant]   content="湿度差 10%"  + tool_calls=[]              ← Final Answer
```

**关键约束**：

- 每个 `tool_calls` 中的 id 必须有对应的 `role: "tool"` 消息，否则下一轮 API 报错
- `tool_calls` 和 `role: "tool"` 必须成对相邻出现
- 思考模型的 `reasoning_content` **不进 messages**（见问题 4）

### 问题 3：未完成时，每个步骤的输出是否都包含工具调用？

**是的**。在 ReAct 循环中：

- **未完成** → `tool_calls` 必非空。模型必须调工具才能获取新信息，否则循环无意义。
- **完成** → `tool_calls` 必为空。此时 `content` 才是最终答案。

中间某轮 `content` 可能为空字符串（模型只想调工具不想说话），这是合法的：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [{...}]
}
```

但有经验的实现会在 system prompt 中要求"每次调用工具前简述理由"，让 `content` 成为可读的 Thought 日志，便于调试。

**特殊情况——并行工具调用**：模型可能在一条 assistant 消息里 `tool_calls` 数组包含多个元素，对应多个 `role: "tool"` 消息。ReAct 不强制串行，并行调用更高效。

### 问题 4：思考模型应用在 ReAct 中如何思考？

#### 思考发生的位置

思考模型在**每次 LLM 调用**内部都做深度思考，相当于在原 ReAct 的 Thought 之外多了一层"思考的思考"：

```
非思考模型每轮：
  [一次推理] → content (Thought) + tool_calls (Action)

思考模型每轮：
  [隐藏草稿区长链思考]  ← reasoning_content，可能几千 token
       ↓
  → content (精炼后的 Thought 或空) + tool_calls (Action)
```

#### 各家思考模型输出对比

| 模型类型 | 思考输出位置 | 是否进 messages | 是否计费 |
|---------|------------|--------------|---------|
| gpt-4o / gpt-4o-mini | 无思考 | — | — |
| OpenAI o1 / o3 | 隐藏，API 不返回 | — | 计费但不返回 |
| DeepSeek-R1 | `reasoning_content` 字段 | 不进 messages | 计费 |
| Qwen3-thinking / QwQ | `reasoning_content` 字段 | 不进 messages | 计费 |

**核心原则**：思考模型的 `reasoning_content` **永远不进 messages**。原因：

1. **协议要求**：OpenAI 协议中 assistant 消息只接受 `content` 和 `tool_calls`，`reasoning_content` 不是标准字段，回灌会被 API 拒绝。
2. **模型设计**：思考过程是模型"打草稿"的过程，下一轮调用时模型会重新思考，不需要历史草稿作为输入。
3. **上下文成本**：思考内容通常几千 token，回填会迅速膨胀上下文。
4. **意图清晰**：思考内容是模型的内部状态，不应对外可见，也不应影响下一轮的推理路径。

#### 思考模型 vs 非思考模型在 ReAct 中的差异

| 维度 | 非思考模型 | 思考模型 |
|------|----------|---------|
| 每轮 LLM 调用 | 直接输出 content + tool_calls | 先长链思考，再输出 content + tool_calls |
| Thought 可见性 | content 即为 Thought，可能为空 | content 可能为空或精炼，思考在 reasoning_content |
| 推理深度 | 浅，一步推理 | 深，可在单轮内做多次推演 |
| 单轮延迟 | 低 | 高（思考消耗时间） |
| 总轮数 | 多（需要多步推理才能决策） | 少（单轮内可能直接决策） |
| Token 消耗 | 低 | 高（思考内容计费但不进 messages） |
| 工具选择准确率 | 一般 | 高（思考充分） |
| 循环次数 | 多 | 少 |
| 日志可读性 | content 即是 Thought 日志 | content 可能为空，调试时需要查看 reasoning_content |

#### 思考模型在 ReAct 中的优势

1. **更准的工具选择**：思考过程能权衡多个候选工具，减少误选
2. **更好的参数构造**：能在思考中预演工具调用，发现参数错误
3. **更早的终止判断**：思考中能判断"信息是否足够"，避免无效的多轮调用
4. **更强的错误恢复**：工具返回错误时，思考模型能在草稿中分析原因并调整策略

#### 实践建议

1. **不要强制思考模型输出 Thought**：思考模型的 content 可能自然为空，这是合理的，强制输出反而打断其思考节奏
2. **`max_iterations` 可适当降低**：思考模型单轮决策能力强，3-5 轮通常足够
3. **日志单独存储 reasoning_content**：调试用，但不回灌进 messages
4. **成本控制**：思考模型 token 消耗高 3-10 倍，复杂任务才用，简单路由用普通模型
5. **系统提示要简洁**：思考模型已经会自己推理，过多约束反而干扰思考

## 五、局限性

- 依赖 LLM 自我反思能力，弱模型容易循环或跳步
- Thought 显式输出会消耗大量 token
- 错误的 Observation 可能误导后续推理，缺乏回溯机制
- 长链路时上下文膨胀
- 思考模型在 ReAct 中总成本可能 3-10 倍于普通模型

## 六、地位与演化

ReAct 是后续 Agent 框架的基石范式之一，LangChain、LlamaIndex、AutoGPT 等都内置了 ReAct 实现或变体。即使现在主流已转向 function calling + planning 的混合范式，ReAct 的 Thought-Action-Observation 仍是理解 Agent 工作原理的最小骨架。

从 ReAct 出发的演化方向：

- **反思机制**：失败时让模型自检（Reflexion）
- **规划层**：先拆任务再 ReAct（Plan-and-Execute）
- **记忆层**：加短期/长期记忆（MemGPT 思路）
- **多 Agent**：多个 ReAct Agent 协作（AutoGen、CrewAI）
- **图编排**：LangGraph 用状态图建模更复杂的 Agent 流程
