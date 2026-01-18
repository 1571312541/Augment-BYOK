[根目录](../../../../../CLAUDE.md) > [payload](../../../../) > [extension](../../../) > [out](../../) > [byok](../) > **config**

# config/ - 配置管理模块

> 负责 BYOK 配置的加载、验证、持久化和运行时状态管理。

## 模块职责

1. **配置持久化**: 使用 VS Code `globalState` 存储配置
2. **配置验证**: 规范化用户输入，确保配置结构合法
3. **状态管理**: 维护运行时开关和工具定义缓存
4. **官方连接**: 提供上游 Augment 服务的连接信息

## 文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `config.js` | 361 | 配置管理器核心 - 默认值、规范化、CRUD |
| `state.js` | 53 | 全局运行时状态 - 开关、工具定义缓存 |
| `official.js` | 29 | 官方 Augment 连接信息获取 |

## 关键导出

### config.js

```javascript
// 默认配置结构
defaultConfig() → {
  version: 1,
  official: { completionUrl, apiToken },
  providers: [{ id, type, baseUrl, apiKey, models, defaultModel, headers, requestDefaults }],
  routing: { defaultProviderId, rules: { [endpoint]: { mode, providerId, model } } },
  historySummary: { enabled, providerId, model, ... },
  telemetry: { disabledEndpoints: [] }
}

// 配置管理器工厂
createConfigManager(opts?) → ConfigManager
```

### state.js

```javascript
// 全局状态对象
state = {
  installed: boolean,              // BYOK 是否已安装
  vscode: object,                  // VS Code API 引用
  extensionContext: object,        // 扩展上下文
  runtimeEnabled: boolean,         // 运行时开关
  configManager: ConfigManager,    // 配置管理器实例
  lastCapturedToolDefinitions: [], // 上游工具定义缓存
  lastCapturedToolDefinitionsAtMs: number,
  lastCapturedToolDefinitionsMeta: object
}

// 运行时开关控制
setRuntimeEnabled(ctx, enabled) → Promise<boolean>

// 工具定义捕获
captureAugmentToolDefinitions(toolDefinitions, meta) → boolean
getLastCapturedToolDefinitions() → { toolDefinitions, capturedAtMs, meta }

// 配置管理器单例
ensureConfigManager(opts?) → ConfigManager
```

### official.js

```javascript
// 获取官方连接信息
getOfficialConnection() → { completionURL: string, apiToken: string }

// 默认上游 URL
DEFAULT_OFFICIAL_COMPLETION_URL = "https://api.augmentcode.com/"
```

## 配置存储键

| 键名 | 用途 |
|------|------|
| `augment-byok.config.v1` | 主配置 (providers, routing, historySummary) |
| `augment-byok.runtimeEnabled.v1` | 运行时开关 (boolean) |

## 路由模式

| 模式 | 行为 |
|------|------|
| `byok` | 使用用户配置的 LLM 提供商 |
| `official` | 透传到 Augment 官方服务 |
| `disabled` | 返回空响应，不发送请求 |

## 依赖关系

```
config.js
    └── infra/log.js (debug, warn)
    └── infra/util.js (normalizeEndpoint, normalizeString)

state.js
    └── config.js (createConfigManager)

official.js
    └── state.js (ensureConfigManager)
    └── infra/util.js (normalizeString, normalizeRawToken)
```

## 使用示例

```javascript
// 获取当前配置
const cfg = ensureConfigManager().get();

// 保存配置
await ensureConfigManager().saveNow(newConfig, "user_edit");

// 检查运行时状态
if (state.runtimeEnabled) {
  // BYOK 模式处理
}

// 获取官方连接
const { completionURL, apiToken } = getOfficialConnection();
```

## 变更记录

### 2026-01-18
- 初始化模块文档
