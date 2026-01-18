# Providers 模块

> 导航: [根目录](../../../../../CLAUDE.md) > [BYOK 运行时](../CLAUDE.md) > Providers

## 模块概述

LLM 提供商适配层，实现各 API 的请求构建、响应解析和流处理。

## 文件结构

| 文件 | 职责 | 行数 |
|------|------|------|
| `openai.js` | OpenAI Chat Completions API | ~253 |
| `openai-responses.js` | OpenAI Responses API | ~200 |
| `anthropic.js` | Anthropic Claude API | ~295 |
| `gemini.js` | Google Gemini API | ~250 |
| `http.js` | HTTP 请求工具 | ~80 |
| `sse.js` | SSE 流解析 | ~42 |
| `headers.js` | 请求头构建 | ~50 |
| `models.js` | 模型列表查询 | ~60 |
| `provider-util.js` | 共享工具函数 | ~80 |

## 支持的提供商类型

| 类型标识 | 提供商 | 入口文件 |
|---------|--------|---------|
| `openai_compatible` | OpenAI / Azure / 兼容 API | `openai.js` |
| `openai_responses` | OpenAI Responses API | `openai-responses.js` |
| `anthropic` | Anthropic Claude | `anthropic.js` |
| `gemini_ai_studio` | Google Gemini | `gemini.js` |

## 核心导出

### openai.js

```javascript
const {
  openAiCompleteText,        // 非流式补全
  openAiStreamTextDeltas,    // 流式文本
  openAiChatStreamChunks     // 流式聊天 (带 Tool Use)
} = require('./providers/openai');

// 非流式
const text = await openAiCompleteText({
  baseUrl: 'https://api.openai.com/v1',
  apiKey: 'sk-...',
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello' }],
  timeoutMs: 30000
});

// 流式
for await (const delta of openAiStreamTextDeltas({ ... })) {
  process.stdout.write(delta);
}
```

### anthropic.js

```javascript
const {
  anthropicCompleteText,
  anthropicStreamTextDeltas,
  anthropicChatStreamChunks
} = require('./providers/anthropic');

// 支持 Anthropic 特有功能
for await (const chunk of anthropicChatStreamChunks({
  baseUrl: 'https://api.anthropic.com',
  apiKey: 'sk-ant-...',
  model: 'claude-3-5-sonnet-20241022',
  messages,
  tools,          // Tool definitions
  systemPrompt,   // 独立 system prompt
  timeoutMs
})) {
  // chunk 包含 thinking, text, tool_use 等
}
```

### gemini.js

```javascript
const {
  geminiCompleteText,
  geminiStreamTextDeltas,
  geminiChatStreamChunks
} = require('./providers/gemini');

// Gemini 使用不同的 API 结构
const text = await geminiCompleteText({
  baseUrl: 'https://generativelanguage.googleapis.com/v1beta',
  apiKey: 'AIza...',
  model: 'gemini-2.0-flash',
  messages,
  timeoutMs
});
```

### sse.js

```javascript
const { parseSse } = require('./providers/sse');

// 解析 SSE 流
for await (const event of parseSse(response)) {
  // event: { event?: string, data: string }
  if (event.data === '[DONE]') break;
  const json = JSON.parse(event.data);
}
```

### http.js

```javascript
const {
  safeFetch,       // 带超时和错误处理的 fetch
  readTextLimit,   // 限制读取响应文本长度
  joinBaseUrl      // 拼接 baseUrl 和路径
} = require('./providers/http');

const resp = await safeFetch(url, options, {
  timeoutMs: 30000,
  abortSignal,
  label: 'OpenAI'  // 用于错误消息
});
```

## 依赖关系

```
openai.js / anthropic.js / gemini.js
  ├── http.js (网络请求)
  ├── sse.js (流解析)
  ├── headers.js (认证头)
  ├── provider-util.js (工具函数)
  └── ../core/augment-protocol.js (协议适配)

http.js
  └── ../infra/log.js (日志)

headers.js
  └── ../infra/util.js (工具)
```

## 请求头构建

### headers.js

```javascript
const {
  openAiAuthHeaders,    // Authorization: Bearer <key>
  anthropicAuthHeaders, // x-api-key: <key>, anthropic-version: ...
  geminiAuthHeaders,    // x-goog-api-key: <key>
  withJsonContentType   // 添加 Content-Type: application/json
} = require('./providers/headers');
```

## 错误处理

所有提供商模块统一错误处理模式：

```javascript
// HTTP 错误
if (!resp.ok) {
  throw new Error(`OpenAI ${resp.status}: ${await readTextLimit(resp, 500)}`);
}

// 响应格式错误
const text = json?.choices?.[0]?.message?.content;
if (typeof text !== 'string') {
  throw new Error('OpenAI 响应缺少 choices[0].message.content');
}
```

## 流式响应处理

### SSE 事件格式

```
event: message
data: {"id":"...","choices":[{"delta":{"content":"Hello"}}]}

data: [DONE]
```

### 解析流程

```javascript
for await (const ev of parseSse(resp)) {
  if (ev.data === '[DONE]') break;

  const json = JSON.parse(ev.data);
  const delta = json.choices?.[0]?.delta?.content;
  if (delta) yield delta;
}
```

## 注意事项

1. **API Key 安全**: 日志中使用 `redactKey()` 脱敏
2. **超时处理**: 所有请求必须设置 `timeoutMs`
3. **流中断**: 正确处理 `abortSignal` 取消请求
4. **速率限制**: 捕获 429 错误并提供友好提示
5. **兼容性**: OpenAI Compatible 支持 Azure、DeepSeek 等
