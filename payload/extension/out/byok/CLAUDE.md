[根目录](../../../../CLAUDE.md) > [payload](../../../) > [extension](../../) > [out](../) > **byok**

# BYOK 核心运行时模块

> BYOK (Bring Your Own Key) 的核心运行时代码，负责拦截 Augment 的 LLM 请求并路由到用户配置的 LLM 提供商。

## 模块职责

1. **请求拦截**: 通过 shim 拦截 `callApi` 和 `callApiStream` 调用
2. **路由决策**: 根据端点和配置决定 `byok|official|disabled` 模式
3. **协议转换**: 将 Augment 协议转换为各 LLM 提供商的 API 格式
4. **流式处理**: 支持 SSE 流式响应并转换回 Augment NDJSON 格式
5. **配置管理**: 从 VS Code globalState 读取和持久化配置

## 入口与启动

- **bootstrap.js**: 扩展激活时注入，注册 BYOK 命令，初始化配置
- **shim.js**: API 拦截层，处理 `maybeHandleCallApi` 和 `maybeHandleCallApiStream`

启动流程:
```
extension.js activate()
  -> bootstrap.install()
    -> 读取 globalState 配置
    -> 注册 VS Code 命令
    -> 返回原始 activate 继续执行
```

## 对外接口

### 导出的主要函数

**runtime/shim.js**:
- `maybeHandleCallApi({ endpoint, body, transform, ... })` - 同步 API 拦截
- `maybeHandleCallApiStream({ endpoint, body, transform, ... })` - 流式 API 拦截

**config/state.js**:
- `ensureConfigManager()` - 获取配置管理器单例
- `setRuntimeEnabled(ctx, enabled)` - 设置运行时开关

**core/router.js**:
- `decideRoute({ cfg, endpoint, body, runtimeEnabled })` - 路由决策

## 关键依赖与配置

### 内部模块依赖

| 子目录 | 文件数 | 职责 |
|-------|-------|------|
| `config/` | 3 | 配置管理、官方连接、状态 |
| `core/` | 20 | 核心逻辑、协议转换、路由 |
| `infra/` | 2 | 日志、工具函数 |
| `prompts/` | 12 | 各端点的提示词模板 |
| `providers/` | 9 | LLM 提供商适配器 |
| `runtime/` | 2 | 引导和 shim |
| `ui/` | 5 | 配置面板 WebView |

### 配置存储

- `augment-byok.config.v1` - 主配置（含 Key/Token）
- `augment-byok.runtimeEnabled.v1` - 运行时开关
- `augment-byok.historySummaryCache.v1` - 历史摘要缓存

## 数据模型

### 配置结构 (v1)

```javascript
{
  version: 1,
  official: { completionUrl, apiToken },
  providers: [{ id, type, baseUrl, apiKey, models, defaultModel, headers, requestDefaults }],
  routing: { defaultProviderId, rules: { [endpoint]: { mode, providerId, model } } },
  historySummary: { enabled, providerId, model, ... },
  telemetry: { disabledEndpoints: [] }
}
```

### 路由决策结果

```javascript
{
  mode: 'byok' | 'official' | 'disabled',
  endpoint: '/chat-stream',
  reason: 'rule' | 'model_override' | 'rollback_disabled',
  provider: { id, type, baseUrl, apiKey, ... },
  model: 'gpt-4o-mini'
}
```

## 测试与质量

- 无独立单元测试（运行时代码已是编译产物）
- 依赖构建期合约检查 (`tools/check/byok-contracts.js`)
- 依赖运行期手动测试 (`Self Test` 命令)

## 常见问题 (FAQ)

**Q: 为什么没有 TypeScript 源码?**
A: 运行时代码直接作为 JS 编写，类型边界用 `normalize/validate` 函数固定形状。

**Q: 如何添加新的 LLM 提供商?**
A: 在 `providers/` 目录添加新适配器，实现 `*CompleteText`、`*StreamTextDeltas`、`*ChatStreamChunks` 三个函数。

**Q: 如何调试?**
A: 使用 `infra/log.js` 的 `debug`/`info`/`warn` 函数，日志会输出到 VS Code 开发者工具。

## 相关文件清单

### 核心文件

- `runtime/bootstrap.js` - 扩展激活入口
- `runtime/shim.js` - API 拦截层（1200+ 行，最复杂）
- `core/router.js` - 路由决策
- `core/augment-chat.js` - Chat 协议处理
- `core/protocol.js` - Augment 协议构建
- `config/config.js` - 配置管理器

### Provider 适配器

- `providers/openai.js` - OpenAI 兼容 API
- `providers/openai-responses.js` - OpenAI Responses API
- `providers/anthropic.js` - Anthropic Claude API
- `providers/gemini.js` - Google Gemini API

### UI

- `ui/config-panel.js` - 配置面板主逻辑
- `ui/config-panel.webview.js` - WebView 消息处理
- `ui/config-panel.html.js` - HTML 模板

## 变更记录 (Changelog)

### 2026-01-18
- 初始化模块文档

