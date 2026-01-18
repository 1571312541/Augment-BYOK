[根目录](../../../../../CLAUDE.md) > [payload](../../../../) > [extension](../../../) > [out](../../) > [byok](../) > **runtime**

# runtime/ - 运行时引导模块

> BYOK 的入口层，负责扩展激活时的注入和 API 拦截。

## 模块职责

1. **扩展注入**: 在 Augment 扩展激活时插入 BYOK 逻辑
2. **命令注册**: 注册 VS Code 命令 (enable/disable/config)
3. **API 拦截**: 拦截 `callApi` 和 `callApiStream` 调用
4. **路由分发**: 根据配置决定请求走 BYOK 还是官方

## 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `bootstrap.js` | 110 | 扩展激活入口 - 注入、命令注册 |
| `shim.js` | 1228 | API 拦截层 - 核心请求处理逻辑 |

## 启动流程

```
Augment extension.js activate()
    │
    ▼
bootstrap.install({ vscode, getActivate, setActivate })
    │
    ├── 包装原始 activate 函数
    ├── 读取 globalState 恢复 runtimeEnabled
    ├── 初始化 ConfigManager
    └── 注册 VS Code 命令
    │
    ▼
原始 activate(ctx) 继续执行
```

## 关键导出

### bootstrap.js

```javascript
// 安装 BYOK 到 Augment 扩展
install({ vscode, getActivate, setActivate }) → void
```

**注册的命令**:

| 命令 ID | 功能 |
|---------|------|
| `augment-byok.enable` | 启用 BYOK 运行时 |
| `augment-byok.disable` | 禁用 BYOK (回滚到官方) |
| `augment-byok.reloadConfig` | 重新加载配置 |
| `augment-byok.openConfigPanel` | 打开配置面板 |
| `augment-byok.clearHistorySummaryCache` | 清除历史摘要缓存 |

### shim.js

```javascript
// 同步 API 拦截
maybeHandleCallApi({
  endpoint,           // 端点路径
  body,               // 请求体
  transform,          // 响应转换函数
  timeoutMs,          // 超时时间
  abortSignal,        // 中止信号
  upstreamApiToken,   // 上游 Token (可选)
  upstreamCompletionURL // 上游 URL (可选)
}) → Promise<Result | undefined>

// 流式 API 拦截
maybeHandleCallApiStream({
  endpoint,
  body,
  transform,
  timeoutMs,
  abortSignal,
  upstreamApiToken,
  upstreamCompletionURL
}) → Promise<AsyncGenerator | undefined>
```

**返回值**:
- `undefined` → 不拦截，走官方
- `Result` / `AsyncGenerator` → BYOK 处理结果

## 支持的端点

### 同步端点 (`maybeHandleCallApi`)

| 端点 | 处理逻辑 |
|------|----------|
| `/get-models` | 合并 BYOK 模型到官方列表 |
| `/completion` | 代码补全 |
| `/chat-input-completion` | 输入补全 |
| `/edit` | 代码编辑 |
| `/chat` | 非流式对话 |
| `/next_edit_loc` | 下一编辑位置 |

### 流式端点 (`maybeHandleCallApiStream`)

| 端点 | 处理逻辑 |
|------|----------|
| `/chat-stream` | 流式对话 (Tool Use 支持) |
| `/prompt-enhancer` | Prompt 增强 |
| `/generate-conversation-title` | 对话标题生成 |
| `/instruction-stream` | 指令编辑流 |
| `/smart-paste-stream` | 智能粘贴流 |
| `/generate-commit-message-stream` | Commit 消息生成 |
| `/next-edit-stream` | 下一编辑建议流 |

## 官方服务注入

shim.js 会在 `/chat-stream` 和 `/chat` 请求中注入官方服务结果：

1. **Codebase Retrieval** (`agents/codebase-retrieval`)
2. **Context Canvas** (`context-canvas/list`)
3. **External Sources** (`get-implicit-external-sources`, `search-external-sources`)

## 依赖关系

```
bootstrap.js
    └── infra/log.js
    └── config/state.js
    └── ui/config-panel.js
    └── core/augment-history-summary-auto.js

shim.js
    └── infra/log.js, util.js
    └── config/state.js, official.js
    └── core/router.js, protocol.js, augment-chat.js, ...
    └── providers/openai.js, anthropic.js, gemini.js, ...
```

## 拦截决策流程

```
请求进入
    │
    ▼
检查 runtimeEnabled ──否──▶ 返回 undefined (官方处理)
    │是
    ▼
decideRoute() 路由决策
    │
    ├── mode="official" ──▶ 返回 undefined
    ├── mode="disabled" ──▶ 返回空响应
    └── mode="byok" ──▶ BYOK 处理
           │
           ▼
        选择 Provider
           │
           ▼
        调用 LLM API
           │
           ▼
        转换响应格式
           │
           ▼
        返回结果
```

## 变更记录

### 2026-01-18
- 初始化模块文档
