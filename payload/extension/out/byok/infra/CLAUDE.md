[根目录](../../../../../CLAUDE.md) > [payload](../../../../) > [extension](../../../) > [out](../../) > [byok](../) > **infra**

# infra/ - 基础设施模块

> 提供日志记录和通用工具函数，被所有其他模块依赖。

## 模块职责

1. **日志系统**: 带前缀和敏感信息脱敏的日志输出
2. **字符串处理**: 规范化、验证、转换
3. **端点解析**: URL 和路径处理
4. **BYOK Model ID 解析**: `byok:providerId:modelId` 格式解析

## 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `log.js` | 131 | 日志系统 - 脱敏、截断、分级输出 |
| `util.js` | 94 | 通用工具函数 |

## 关键导出

### log.js

```javascript
// 日志函数（自动脱敏 API Key、Token）
debug(...args)  // 仅 AUGMENT_BYOK_DEBUG=1 时输出
info(...args)   // 常规信息
warn(...args)   // 警告
error(...args)  // 错误

// 文本脱敏（手动调用）
redactText(text) → string
```

**脱敏规则**:
- `Bearer xxx` → `Bearer ***`
- `ace_xxx` → `ace_***`
- `sk-ant-xxx` → `sk-ant-***`
- `sk-xxx` → `sk-***`

**省略字段** (避免日志过长):
- `prefix`, `suffix`, `selected_code`
- `blobs`, `chat_history`, `nodes`
- `tool_definitions`, `rules`

### util.js

```javascript
// 字符串规范化
normalizeString(v) → string           // trim + 空值检查
requireString(v, label) → string      // 必需字符串，否则抛错

// 端点处理
normalizeEndpoint(endpoint) → string  // "/chat-stream" 格式

// Token 处理
normalizeRawToken(token) → string     // 移除 "Bearer " 前缀，提取环境变量值

// BYOK Model ID 解析
parseByokModelId(modelId, opts?) → { providerId, modelId } | null
// 示例: "byok:openai:gpt-4o" → { providerId: "openai", modelId: "gpt-4o" }

// 转换辅助
safeTransform(transform, raw, label) → any  // 安全执行转换函数

// 异步生成器
emptyAsyncGenerator() → AsyncGenerator  // 空流

// ID 生成
randomId() → string  // UUID 或回退格式
```

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `AUGMENT_BYOK_DEBUG` | `0` | 设为 `1` 启用 debug 日志 |

## 日志配置

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_LOG_STRING_BYTES` | 4000 | 单个字符串最大字节数 |
| `MAX_LOG_DEPTH` | 6 | 对象递归深度限制 |
| `MAX_LOG_ARRAY` | 40 | 数组最大元素数 |

## 使用示例

```javascript
const { debug, info, warn, error } = require("./infra/log");
const { normalizeString, parseByokModelId } = require("./infra/util");

// 日志（自动脱敏）
info("请求配置", { apiKey: "sk-xxx123456" });
// 输出: [Augment-BYOK] 请求配置 { apiKey: "[redacted]" }

// 字符串处理
const ep = normalizeEndpoint("/chat-stream?foo=bar");  // "/chat-stream"

// Model ID 解析
const parsed = parseByokModelId("byok:anthropic:claude-3-5-sonnet-20241022");
// { providerId: "anthropic", modelId: "claude-3-5-sonnet-20241022" }
```

## 依赖关系

```
log.js
    └── (无外部依赖)

util.js
    └── (无外部依赖)
```

**被依赖**: 所有其他模块

## 变更记录

### 2026-01-18
- 初始化模块文档
