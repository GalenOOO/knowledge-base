# LLM API 调用协议

> 本文档目的：介绍主流 LLM 生成接口协议（客户端 → 模型）的请求/响应格式与工具调用格式。

## 一、协议总览

生成接口协议都是 HTTP + JSON，流式走 SSE；差异集中在消息结构、鉴权、system 处理、工具调用和流式结束符。

| 协议 | 端点 | 鉴权 | 定位 |
|------|------|------|------|
| OpenAI Chat Completions | `/v1/chat/completions` | `Authorization: Bearer` | 事实标准，绝大多数厂商与本地推理框架兼容 |
| OpenAI Responses | `/v1/responses` | `Authorization: Bearer` | OpenAI 新一代统一接口，内置搜索/文件/计算机使用/MCP 工具 |
| Anthropic Messages | `/v1/messages` | `x-api-key` | Claude 官方协议，Claude Code、Cursor 等工具链原生使用 |
| Gemini generateContent | `generateContent` / `streamGenerateContent` | `?key=` 或 Bearer | 嵌套结构最深，多模态支持最强 |
| AWS Bedrock Converse | `/model/<id>/converse` | AWS SigV4 | 把 Claude、Llama、Mistral 等归一为统一格式 |
| Azure OpenAI | OpenAI 协议 + `api-version` | `api-key` | OpenAI 协议的变体，`deployment` 代替 `model` |
| 本地框架私有协议 | 如 Ollama `/api/chat`、llama.cpp `/completion` | 无 | 各自 JSON 格式，同时提供 `/v1` OpenAI 兼容端点 |

## 二、OpenAI Chat Completions

### 请求

```json
POST /v1/chat/completions
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system",    "content": "你是助手"},
    {"role": "user",      "content": "你好"},
    {"role": "assistant", "content": "之前的回复"},
    {"role": "user",      "content": "再问一句"}
  ],
  "temperature": 0.7,
  "max_tokens": 100,
  "stream": false
}
```

消息角色：`system` / `user` / `assistant` / `tool`。多轮对话把历史消息整体回传。

### 响应（非流式）

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "你好"},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 10, "completion_tokens": 20, "total_tokens": 30}
}
```

`finish_reason` 取值：`stop` / `length` / `tool_calls`。

### 流式

SSE，每个数据块：

```text
data: {"choices": [{"delta": {"content": "你"}}]}
```

结束符为 `data: [DONE]`。思维链模型通过非标准扩展字段 `reasoning_content` 返回思考过程（DeepSeek 等同样如此），该字段不进 messages。

### 工具调用

工具声明（`parameters` 为 JSON Schema）：

```json
{
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "查询城市天气",
      "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"]
      }
    }
  }]
}
```

模型返回（`arguments` 是 **JSON 字符串**）：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc",
        "type": "function",
        "function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

结果回传：把含 `tool_calls` 的 assistant 消息原样回传，再用 `role: "tool"` 附上结果（每条 `tool_call_id` 对应一条 tool 消息）：

```json
{
  "messages": [
    {"role": "user", "content": "北京天气？"},
    {"role": "assistant", "content": null,
     "tool_calls": [{"id": "call_abc", "type": "function", "function": {"name": "get_weather", "arguments": "{\"city\":\"北京\"}"}}]},
    {"role": "tool", "tool_call_id": "call_abc", "content": "晴，25°C"}
  ]
}
```

## 三、OpenAI Responses

### 请求

```json
POST /v1/responses
{
  "model": "gpt-5",
  "input": [
    {"role": "user", "content": "你好"}
  ],
  "instructions": "系统提示词",
  "tools": [{"type": "web_search_preview"}]
}
```

消息字段为 `input`，系统提示用 `instructions`。

### 响应

`output` 是**类型化 item 数组**：

```json
{
  "id": "resp_xxx",
  "object": "response",
  "output": [
    {"type": "message", "role": "assistant",
     "content": [{"type": "output_text", "text": "你好"}]},
    {"type": "function_call", "name": "xxx", "arguments": "{}"}
  ],
  "usage": {"input_tokens": 10, "output_tokens": 20}
}
```

### 流式

SSE 事件驱动（如 `response.output_text.delta`），无 `[DONE]`。

### 工具调用

function 工具是**扁平结构**（无 `function` 嵌套）：

```json
{
  "tools": [{
    "type": "function",
    "name": "get_weather",
    "description": "查询城市天气",
    "parameters": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}
  }]
}
```

模型返回 `output` 中的 `function_call` item；执行后把 `function_call_output` 追加回 `input`：

```json
{
  "input": [
    {"role": "user", "content": "北京天气？"},
    {"type": "function_call", "call_id": "fc_123", "name": "get_weather", "arguments": "{\"city\":\"北京\"}"},
    {"type": "function_call_output", "call_id": "fc_123", "output": "晴，25°C"}
  ]
}
```

内置工具（`web_search_preview`、`file_search`、`computer_use_preview`、`mcp_servers`）同样走 `tools`，仅 `type` 不同。

## 四、Anthropic Messages

### 请求

`system` 独立成字段，`content` 可以是字符串或类型化块数组（`text` / `image` / `tool_use` / `tool_result`）：

```json
POST /v1/messages
{
  "model": "claude-sonnet-4-5",
  "system": "你是助手",
  "messages": [
    {"role": "user", "content": "你好"}
  ],
  "max_tokens": 100,
  "stream": true
}
```

### 响应

`content` 为块数组：

```json
{
  "id": "msg_xxx",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "text", "text": "你好"}
  ],
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 10, "output_tokens": 20}
}
```

`stop_reason` 取值：`end_turn` / `max_tokens` / `tool_use`。

### 流式

SSE 事件类型驱动（`message_start` → `content_block_start` → `content_block_delta` → `message_delta` → `message_stop`），无 `[DONE]`。

### 工具调用

工具声明（`input_schema` 为 JSON Schema）：

```json
{
  "tools": [{
    "name": "get_weather",
    "description": "查询城市天气",
    "input_schema": {
      "type": "object",
      "properties": {"city": {"type": "string"}},
      "required": ["city"]
    }
  }]
}
```

模型返回 `content` 中的 `tool_use` 块（`input` 是**对象**，非字符串）：

```json
{
  "content": [
    {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "北京"}}
  ],
  "stop_reason": "tool_use"
}
```

结果回传：**以 `role: "user"` 回传** `tool_result`：

```json
{
  "messages": [
    {"role": "user", "content": "北京天气？"},
    {"role": "assistant", "content": [
      {"type": "tool_use", "id": "toolu_01", "name": "get_weather", "input": {"city": "北京"}}
    ]},
    {"role": "user", "content": [
      {"type": "tool_result", "tool_use_id": "toolu_01", "content": "晴，25°C"}
    ]}
  ]
}
```

## 五、Gemini generateContent

### 请求

```json
POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=API_KEY
{
  "contents": [
    {"role": "user", "parts": [{"text": "你好"}]}
  ],
  "systemInstruction": {"parts": [{"text": "你是助手"}]},
  "generationConfig": {"temperature": 0.7, "maxOutputTokens": 100}
}
```

多模态时多个 `part`：`{"inlineData": {"mimeType": "image/png", "data": "base64..."}}`。

### 响应

```json
{
  "candidates": [
    {"content": {"role": "model", "parts": [{"text": "你好"}]}, "finishReason": "STOP"}
  ],
  "usageMetadata": {"promptTokenCount": 10, "candidatesTokenCount": 20}
}
```

### 流式

`streamGenerateContent`，SSE，每个 chunk 是一个候选块，无 `[DONE]`。

### 工具调用

工具声明，包一层 `functionDeclarations`：

```json
{
  "tools": [{
    "functionDeclarations": [{
      "name": "get_weather",
      "description": "查询城市天气",
      "parameters": {"type": "object", "properties": {"city": {"type": "string"}}, "required": ["city"]}
    }]
  }]
}
```

模型返回 `parts` 中的 `functionCall`：

```json
{
  "candidates": [{
    "content": {"role": "model", "parts": [
      {"functionCall": {"name": "get_weather", "args": {"city": "北京"}}}
    ]},
    "finishReason": "STOP"
  }]
}
```

结果回传：以 `role: "user"` 用 `functionResponse` part：

```json
{
  "contents": [
    {"role": "user", "parts": [{"text": "北京天气？"}]},
    {"role": "model", "parts": [{"functionCall": {"name": "get_weather", "args": {"city": "北京"}}}]},
    {"role": "user", "parts": [{
      "functionResponse": {"name": "get_weather", "response": {"result": "晴，25°C"}}
    }]}
  ]
}
```

## 六、其它协议要点

- **AWS Bedrock Converse**：请求结构接近 OpenAI（`messages` + `content` 数组），鉴权用 AWS SigV4。
- **Azure OpenAI**：OpenAI 协议 + `api-version` 参数，`deployment` 代替 `model`。
- **Ollama `/api/chat`**：请求 `{"model", "messages", "stream"}`，响应 `{"message", "done"}`；同时提供 `/v1` OpenAI 兼容端点。
- **llama.cpp `/completion`**：请求 `{"prompt", "stream"}`（非 messages 结构），响应 `{"content", "stop"}`；同时提供 `/v1` OpenAI 兼容端点。

## 七、协议横向对比

| 维度 | OpenAI Chat | OpenAI Responses | Anthropic | Gemini |
|------|-------------|------------------|-----------|--------|
| 消息字段 | `messages[]` | `input[]` | `messages[]` | `contents[]`/`parts` |
| 系统提示 | messages 里的 role | `instructions` | 独立 `system` | `systemInstruction` |
| 鉴权 | Bearer | Bearer | `x-api-key` | `?key=` 或 Bearer |
| 输出结构 | `choices[].message` | `output[]` item | `content[]` 块 | `candidates[].parts` |
| 流式结束符 | `[DONE]` | 事件 | 事件（`message_stop`） | 无 |
| 思维链 | `reasoning_content`（扩展） | `reasoning` item | `thinking` 块 | `thought` part |

## 八、工具调用横向对比

| 维度 | OpenAI Chat | OpenAI Responses | Anthropic | Gemini |
|------|-------------|------------------|-----------|--------|
| 工具 schema 字段 | `function.parameters` | `parameters`（扁平） | `input_schema` | `functionDeclarations[].parameters` |
| 模型返回 | `message.tool_calls[]` | `output[]` 的 `function_call` | `content[]` 的 `tool_use` 块 | `parts[]` 的 `functionCall` |
| 参数格式 | JSON 字符串 | JSON 字符串 | 对象 | 对象 |
| 结果回传角色 | `role: "tool"` | item `function_call_output` | `role: "user"` | `role: "user"` |
| 结果回传字段 | `tool_call_id` | `call_id` | `tool_use_id` | `functionResponse` |
| 多个工具并行 | 支持 | 支持 | 支持 | 支持 |

## 九、使用注意事项

- 工具描述一律用 JSON Schema；模型只负责"选工具 + 填参数"，**真正执行始终在客户端**
- 工具执行结果必须回传，模型才能基于结果继续生成（Agent 循环成立的前提）
- 各协议的参数格式不同：OpenAI 系为 JSON 字符串，Anthropic / Gemini 为对象
- 流式结束符不统一：OpenAI Chat 为 `[DONE]`，Anthropic / Responses 为事件，Gemini 无
- 思维链字段各异，且一律不进 messages 回传
