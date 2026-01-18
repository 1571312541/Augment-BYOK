# Core 模块

> 导航: [根目录](../../../../../CLAUDE.md) > [BYOK 运行时](../CLAUDE.md) > Core

## 模块概述

核心业务逻辑模块，负责请求路由、协议转换、聊天流处理和 Augment 原生数据结构适配。

## 文件结构

| 文件 | 职责 | 行数 |
|------|------|------|
| `router.js` | 路由决策逻辑 | ~45 |
| `protocol.js` | 通用协议转换 | ~156 |
| `augment-protocol.js` | Augment 协议常量与节点构造 | ~200 |
| `augment-struct.js` | 数据结构工具函数 | ~100 |
| `augment-chat.js` | 聊天流主入口 | ~150 |
| `augment-chat.openai.js` | OpenAI Chat 适配 | ~200 |
| `augment-chat.anthropic.js` | Anthropic Chat 适配 | ~180 |
| `augment-chat.gemini.js` | Gemini Chat 适配 | ~160 |
| `augment-chat.openai-responses.js` | OpenAI Responses API 适配 | ~180 |
| `augment-chat.shared.js` | Chat 共享工具 | ~80 |
| `augment-history-summary.js` | 历史摘要解析 | ~150 |
| `augment-history-summary-auto.js` | 自动历史摘要 | ~200 |
| `augment-history-summary-auto.abridged.js` | 精简历史摘要 | ~100 |
| `augment-node-format.js` | 节点格式化 | ~80 |
| `model-registry.js` | 模型注册表 | ~60 |
| `next-edit-fields.js` | 下一编辑字段解析 | ~100 |
| `next-edit-loc-utils.js` | 编辑位置工具 | ~80 |
| `next-edit-stream-utils.js` | 编辑流工具 | ~120 |
| `stream-guard.js` | 流错误守卫 | ~60 |
| `text-match.js` | 文本匹配工具 | ~80 |
| `tool-pairing.js` | Tool Use/Result 配对 | ~100 |
| `self-test.js` | 自检逻辑 | ~150 |
| `blob-utils.js` | Blob 处理 | ~40 |
| `diagnostics-utils.js` | 诊断工具 | ~60 |
| `number-utils.js` | 数字工具 | ~30 |
| `unicode-utils.js` | Unicode 处理 | ~40 |

## 核心导出

### router.js

```javascript
const { decideRoute } = require('./core/router');

// 路由决策
const decision = decideRoute({
  cfg,           // 配置对象
  endpoint,      // 端点名称
  body,          // 请求体
  runtimeEnabled // 运行时开关
});
// => { mode, endpoint, reason, provider, model }
```

### augment-protocol.js

```javascript
// 协议常量
const {
  REQUEST_NODE_TEXT,
  REQUEST_NODE_TOOL_RESULT,
  REQUEST_NODE_HISTORY_SUMMARY,
  RESPONSE_NODE_RAW_RESPONSE,
  RESPONSE_NODE_THINKING,
  RESPONSE_NODE_TOOL_USE,
  STOP_REASON_END_TURN,
  STOP_REASON_TOOL_USE_REQUESTED
} = require('./core/augment-protocol');

// 节点构造函数
const {
  rawResponseNode,
  toolUseStartNode,
  toolUseNode,
  thinkingNode,
  mainTextFinishedNode,
  tokenUsageNode,
  makeBackChatChunk
} = require('./core/augment-protocol');
```

### augment-chat.js

```javascript
const { augmentChatStreamChunks } = require('./core/augment-chat');

// 聊天流生成器
for await (const chunk of augmentChatStreamChunks({
  providerType,  // 'openai_compatible' | 'anthropic' | 'gemini_ai_studio'
  cfg,
  exchanges,
  newUserMessage,
  abortSignal
})) {
  // 处理 Augment 格式的 chunk
}
```

## 依赖关系

```
router.js
  └── ../config/state.js

augment-chat.js
  ├── augment-chat.openai.js
  ├── augment-chat.anthropic.js
  ├── augment-chat.gemini.js
  ├── augment-chat.openai-responses.js
  └── augment-chat.shared.js

augment-protocol.js
  └── augment-struct.js

augment-history-summary*.js
  ├── augment-protocol.js
  └── augment-struct.js
```

## 关键概念

### 路由模式 (Route Modes)

- `byok` - 使用 BYOK 配置的提供商
- `official` - 透传到 Augment 官方后端
- `disabled` - 返回空响应

### Augment 节点类型

**请求节点 (Request Nodes):**
- `text` - 用户文本消息
- `tool_result` - 工具执行结果
- `history_summary` - 历史摘要

**响应节点 (Response Nodes):**
- `raw_response` - 原始文本响应
- `thinking` - 思考过程
- `tool_use` - 工具调用请求

### 停止原因 (Stop Reasons)

- `end_turn` - 正常结束
- `tool_use_requested` - 请求工具调用
- `max_tokens` - 达到 token 限制

## 使用示例

### 完整聊天流处理

```javascript
const { decideRoute } = require('./router');
const { augmentChatStreamChunks } = require('./augment-chat');
const { makeBackChatChunk } = require('./augment-protocol');

async function handleChatStream(cfg, body) {
  const route = decideRoute({ cfg, endpoint: 'chat-stream', body, runtimeEnabled: true });

  if (route.mode !== 'byok') {
    return undefined; // 交给官方处理
  }

  const stream = augmentChatStreamChunks({
    providerType: route.provider.type,
    cfg: route.provider,
    exchanges: body.exchanges,
    newUserMessage: body.new_user_message,
    abortSignal
  });

  return stream;
}
```

## 注意事项

1. **协议兼容性**: 所有输出必须符合 Augment 前端期望的格式
2. **流错误处理**: 使用 `stream-guard.js` 确保流异常时优雅降级
3. **Tool Use 配对**: `tool-pairing.js` 确保 tool_use 和 tool_result 正确关联
4. **历史摘要**: 大上下文时自动启用历史摘要压缩
