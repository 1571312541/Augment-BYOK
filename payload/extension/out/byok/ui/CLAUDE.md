# UI 模块

> 导航: [根目录](../../../../../CLAUDE.md) > [BYOK 运行时](../CLAUDE.md) > UI

## 模块概述

配置面板 WebView 实现，提供可视化的 BYOK 配置界面。

## 文件结构

| 文件 | 职责 | 行数 |
|------|------|------|
| `config-panel.js` | 面板主入口，WebView 创建与消息处理 | ~212 |
| `config-panel.html.js` | HTML 模板生成 | ~150 |
| `config-panel.css` | 样式表 | ~200 |
| `config-panel.webview.js` | WebView 端主逻辑 | ~180 |
| `config-panel.webview.render.js` | 渲染函数 | ~120 |
| `config-panel.webview.util.js` | WebView 端工具函数 | ~60 |

## 核心导出

### config-panel.js

```javascript
const { openConfigPanel } = require('./ui/config-panel');

// 打开配置面板
openConfigPanel(extensionContext);
```

## 架构设计

```
VS Code Extension Host              WebView (iframe)
┌─────────────────────┐            ┌─────────────────────┐
│  config-panel.js    │◄──────────►│  config-panel.      │
│  - createWebview()  │  postMessage│  webview.js         │
│  - createHandlers() │            │  - initApp()        │
│                     │            │  - handleMessage()  │
└─────────────────────┘            └─────────────────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────────┐            ┌─────────────────────┐
│  ConfigManager      │            │  webview.render.js  │
│  (config/config.js) │            │  - renderProviders()│
└─────────────────────┘            │  - renderRouting()  │
                                   └─────────────────────┘
```

## 消息协议

### Extension → WebView

```javascript
// 发送当前配置
vscode.postMessage({
  type: 'config',
  config: { providers: [...], routing: {...}, ... }
});

// 发送保存结果
vscode.postMessage({
  type: 'saveResult',
  success: true,
  message: '配置已保存'
});
```

### WebView → Extension

```javascript
// 请求加载配置
vscode.postMessage({ type: 'load' });

// 保存配置
vscode.postMessage({
  type: 'save',
  config: { providers: [...], routing: {...}, ... }
});

// 测试连接
vscode.postMessage({
  type: 'testConnection',
  providerId: 'openai-main'
});
```

## HTML 模板结构

### config-panel.html.js

```javascript
function generateHtml({ cspSource, cssUri, jsUri }) {
  return `<!DOCTYPE html>
<html>
<head>
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'none';
                 style-src ${cspSource};
                 script-src ${cspSource};">
  <link href="${cssUri}" rel="stylesheet">
</head>
<body>
  <div id="app">
    <section id="providers">...</section>
    <section id="routing">...</section>
    <section id="actions">...</section>
  </div>
  <script src="${jsUri}"></script>
</body>
</html>`;
}
```

## 样式设计

### config-panel.css

```css
/* VS Code 主题变量 */
:root {
  --vscode-font-family: var(--vscode-editor-font-family);
  --vscode-font-size: var(--vscode-editor-font-size);
}

/* 响应式布局 */
.provider-card {
  display: grid;
  grid-template-columns: 1fr 2fr;
  gap: 8px;
}

/* 表单控件 */
input, select {
  background: var(--vscode-input-background);
  color: var(--vscode-input-foreground);
  border: 1px solid var(--vscode-input-border);
}
```

## WebView 端逻辑

### config-panel.webview.js

```javascript
// 初始化
function initApp() {
  vscode.postMessage({ type: 'load' });

  window.addEventListener('message', (e) => {
    const msg = e.data;
    switch (msg.type) {
      case 'config':
        renderConfig(msg.config);
        break;
      case 'saveResult':
        showNotification(msg);
        break;
    }
  });
}

// 保存按钮
document.getElementById('save').onclick = () => {
  const config = collectFormData();
  vscode.postMessage({ type: 'save', config });
};
```

## 配置面板功能

### 提供商管理

- 添加/删除提供商
- 配置 API Key、Base URL、Model
- 测试连接功能

### 路由配置

- 按端点设置路由规则
- 选择默认提供商
- 配置 fallback 行为

### 高级设置

- 历史摘要启用/阈值
- 超时设置
- 调试模式开关

## 安全考虑

1. **CSP 策略**: 严格限制脚本和样式来源
2. **消息验证**: WebView 消息需验证来源
3. **API Key 显示**: 输入框 type="password"
4. **存储加密**: 依赖 VS Code SecretStorage（未来计划）

## 使用示例

### 注册命令

```javascript
// runtime/bootstrap.js
vscode.commands.registerCommand('augment-byok.openConfigPanel', () => {
  openConfigPanel(context);
});
```

### 面板生命周期

```javascript
const panel = vscode.window.createWebviewPanel(
  'byokConfig',
  'BYOK Configuration',
  vscode.ViewColumn.One,
  {
    enableScripts: true,
    retainContextWhenHidden: true,
    localResourceRoots: [extensionUri]
  }
);

panel.onDidDispose(() => {
  // 清理资源
});
```

## 注意事项

1. **主题适配**: 使用 VS Code CSS 变量确保深浅主题兼容
2. **状态保持**: `retainContextWhenHidden` 保持切换 tab 后状态
3. **资源路径**: 使用 `webview.asWebviewUri()` 转换本地路径
4. **错误提示**: 表单验证错误需清晰提示用户
