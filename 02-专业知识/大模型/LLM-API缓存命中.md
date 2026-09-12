# LLM API 缓存命中（Prompt Caching）

> 本文档目的：介绍 LLM API 缓存命中的机制、主流厂商方案、命中率影响因素与提高命中率的设计原则。

## 一、核心概念

大模型 API 按 token 计费，输入（prompt）和输出分别计价。缓存命中指：当本次请求的**输入前缀**与之前某次请求完全一致时，服务商直接复用已算好的结果，只对新内容重新计算，并按折扣价收费、降低首字延迟。

- **命中（cache hit）**：前缀匹配已有缓存，输入按命中价计价
- **未命中（cache miss）**：前缀有差异，从差异点起按全价重新计算，并写入缓存（write/create）

典型受益场景：

- **多轮对话**：历史消息每轮重复发送，命中后只付新增的一条
- **Agent / 工具调用循环**：固定 system prompt（工具定义、few-shot）每轮复用
- **RAG**：固定指令部分可复用，命中按折扣价计

## 二、技术原理

Transformer 生成回答分两阶段：

1. **prefill（预填充）**：并行读入整个 prompt，为每个 token 计算注意力 Key/Value，存入 **KV cache**。token 越多成本越高
2. **decode（逐 token 生成）**：一次生成一个 token，新 token 逐步追加进 KV cache

两次请求前缀一致时，第一次算好的 KV 直接复用，**跳过 prefill**，只对新 token 计算。命中价可以低到全价的 1/10，省下的正是这段重复计算。

**命中的前提是精确前缀匹配**：从第一个 token 起逐 token 完全一致。任何差异（空格、换行、顺序、内容变化）都会使差异点之后全部视为未命中。

## 三、主流厂商方案

| 维度 | OpenAI | Anthropic | DeepSeek | Gemini |
|------|--------|-----------|----------|--------|
| 名称 | Automatic Prompt Caching | Prompt Caching | 上下文硬盘缓存（Context Caching on Disk） | Context Caching（隐式 + 显式） |
| 启用方式 | 全自动，无需参数 | 显式，手动加 `cache_control` 断点 | 全自动，无需参数 | 隐式自动；显式手动创建 |
| 匹配粒度 | 前缀 ≥1024 tokens 起缓存 | 1024 tokens 块粒度 | 64 tokens 粒度 | 前缀匹配 |
| 写入成本 | 0.5× | **1.25×** | 免费 | 隐式免费；显式按存储时长计费 |
| 命中成本 | 0.125× | 0.1× | 约为未命中 1/10 量级 | 低于未命中 |
| TTL | 5–10 分钟滑动窗口 | 5 分钟 / 1 小时可选 | 尽力而为，可被随时清除 | 隐式约 20 分钟；显式可设更长 |
| usage 字段 | `prompt_tokens_details.cached_tokens` | `cache_creation_input_tokens` / `cache_read_input_tokens` | `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` | `cachedContentTokenCount` |
| 官方文档 | [prompt-caching](https://developers.openai.com/api/docs/guides/prompt-caching) | [prompt-caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) | [上下文硬盘缓存](https://api-docs.deepseek.com/zh-cn/guides/kv_cache/) | [context caching](https://ai.google.dev/gemini-api/docs/generate-content/caching) |

### OpenAI

- 全自动，API 无需任何改动；曾用名 prompt caching
- 前缀 ≥1024 tokens 自动缓存；命中 0.125×、写入 0.5×；缓存 5–10 分钟不活跃后失效
- 2025 年起提供 Semantic Caching（语义缓存，beta）与 get/set cache 企业 API，前者不要求字面前缀一致

### Anthropic

- 需在消息或 system 块上手动标注断点：

```json
{"type": "text", "text": "很长的系统提示...", "cache_control": {"type": "ephemeral"}}
```

- 从断点起按 1024-token 块缓存；一次请求最多约 4 个断点
- 首次写入按 1.25× 计费，短内容（<1024 tokens）开缓存不划算
- TTL 支持 5 分钟与 1 小时（`"cache_control": {"type": "ephemeral", "ttl": "1h"}`）；缓存可跨组织共享

### DeepSeek

- KV 缓存落硬盘，自动启用、写入免费
- 64-token 粒度；命中价约为未命中 1/10 量级（官方 2025-08 称"价格再降一个数量级"，当前价见[定价页](https://api-docs.deepseek.com/zh-cn/quick_start/pricing/)）
- 缓存容量尽力而为，负载高时可能被清空；命中与未命中 token 分开统计

### Gemini

- 隐式缓存：2.x/2.5 系列自动缓存重复前缀，约 20 分钟 TTL，无需配置
- 显式缓存：手动创建 `cachedContent`，TTL 可从小时级到数天，按"存储 token × 小时"计费，适合固定大语料反复使用

## 四、命中率为何不等于 100%

纯追加、前缀未变、间隔短时，历史部分设计上应接近全命中；实际命中率受以下因素影响：

| 因素 | 说明 |
|------|------|
| TTL 过期 | 两次请求间隔超过缓存保质期，缓存被清，下一次从头全量 miss |
| 容量淘汰 | 缓存是"尽力而为"优化，无 SLA；负载高峰可能被挤掉 |
| 前缀变动 | 命中要求从开头起的连续前缀一致；时间戳/检索结果等必变内容若放在前缀内，其后全部 miss |
| 多节点 | 缓存按机房/节点分布，负载均衡跨区域会掉命中；模型升级也会重置缓存 |
| 缓存阈值 | 前缀过短（如 OpenAI <1024 tokens）不进缓存 |
| 统计口径 | `cached_tokens` 只统计历史部分；新追加的输入是新 token，本就不算命中 |

## 五、提高命中率的设计原则

1. **稳定内容固定在最前**：system prompt、工具定义、few-shot 完全固定，包括格式与空白
2. **必变内容沉到最末**：每轮变化的 token（当前时间、随机扰动、检索结果、工具结果）放在 prompt 末尾；新输入本就要新算，放末尾不增加成本
3. **中间只做追加**：messages 数组只 append 不重写；不重排、不去重、不改写历史；工具调用结果按协议顺序追加，绝对不回填思考模型的 `reasoning_content`

其他做法：

- Anthropic 场景把动态内容放在 cache 断点**之后**，让断点前的块稳定复用
- 长对话压缩时采用摘要替代（摘要稳定后可进缓存）；压缩每 N 轮才一次，一次 miss 换后续继续命中是合理取舍
- 换模型、升级、首次请求、换节点必然全量 miss，属预算内成本
- 逐请求记录 `usage` 中的 hit/miss 字段，统计命中占比；历史部分频繁 miss 时检查 TTL 与前缀稳定性

## 六、使用注意事项

- 精确匹配脆弱：一个空格、一个换行、参数顺序变化都会使命中归零
- 短 TTL：低频请求（间隔超 TTL）基本享受不到缓存
- Anthropic 写入加价：小 prompt 强开缓存是负优化
- 隐私：提示词会暂存于服务商缓存基础设施，敏感语料需评估（见[Anthropic 安全性说明](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#security-and-privacy)）
- 计费口径：hit token 与 miss token 分开统计，混读 usage 会导致成本误判
- 价格参数随厂商调整，以各官方定价页为准