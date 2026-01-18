# Prompts 模块

> 导航: [根目录](../../../../../CLAUDE.md) > [BYOK 运行时](../CLAUDE.md) > Prompts

## 模块概述

提示词模板与构建模块，为每个 LLM 端点定义专门的 System Prompt 和消息格式化逻辑。

## 文件结构

| 文件 | 对应端点 | 职责 |
|------|---------|------|
| `common.js` | 通用 | 共享工具函数与模板片段 |
| `completion.js` | `/completion` | 代码补全提示词 |
| `chat-input-completion.js` | `/chat-input-completion` | 聊天输入补全 |
| `edit.js` | `/edit` | 代码编辑提示词 |
| `instruction-stream.js` | `/instruction-stream` | 指令流处理 |
| `next-edit-stream.js` | `/next-edit-stream` | 下一编辑流 |
| `next-edit-loc.js` | `/next_edit_loc` | 编辑位置预测 |
| `smart-paste-stream.js` | `/smart-paste-stream` | 智能粘贴 |
| `prompt-enhancer.js` | `/prompt-enhancer` | 提示词增强 |
| `commit-message-stream.js` | `/generate-commit-message-stream` | Git 提交消息生成 |
| `conversation-title.js` | `/generate-conversation-title` | 对话标题生成 |

## 核心导出

### common.js

```javascript
const {
  truncate,           // 截断文本到指定长度
  fmtSection,         // 格式化代码块区域
  buildSystem,        // 构建 System Prompt
  extractCodeContext, // 提取代码上下文
  MAX_CONTEXT_CHARS   // 最大上下文字符数
} = require('./prompts/common');
```

### 端点提示词模块

每个端点模块导出类似结构：

```javascript
// 以 completion.js 为例
const { buildCompletionPrompt } = require('./prompts/completion');

const { system, messages } = buildCompletionPrompt({
  prefix,         // 光标前代码
  suffix,         // 光标后代码
  language,       // 编程语言
  filePath,       // 文件路径
  recentFiles     // 最近编辑的文件
});
```

## 提示词设计原则

### 1. 上下文优先

```javascript
// 提取周围代码上下文
const context = extractCodeContext({
  before: prefix,
  after: suffix,
  maxChars: 8000
});
```

### 2. 语言感知

```javascript
// 根据语言调整提示词
const langHint = language
  ? `You are completing ${language} code.`
  : 'You are completing code.';
```

### 3. 输出格式约束

```javascript
// 明确输出格式要求
const formatInstruction = `
Output ONLY the code to insert.
Do NOT include any explanation.
Do NOT wrap in markdown code blocks.
`;
```

## 端点-提示词映射

| 端点 | System Prompt 要点 |
|------|-------------------|
| `/completion` | 代码补全，只输出代码，无解释 |
| `/chat-input-completion` | 聊天输入自动补全建议 |
| `/edit` | 代码编辑，遵循用户指令修改 |
| `/instruction-stream` | 按指令生成代码，流式输出 |
| `/next-edit-stream` | 预测下一处需要编辑的位置和内容 |
| `/next_edit_loc` | 仅返回编辑位置，JSON 格式 |
| `/smart-paste-stream` | 智能适配粘贴内容到当前上下文 |
| `/prompt-enhancer` | 增强用户输入的提示词质量 |
| `/generate-commit-message-stream` | 根据 diff 生成规范提交消息 |
| `/generate-conversation-title` | 根据对话内容生成简洁标题 |

## 使用示例

### 代码补全

```javascript
const { buildCompletionPrompt } = require('./prompts/completion');

const { system, messages } = buildCompletionPrompt({
  prefix: 'function fibonacci(n) {\n  if (n <= 1) return n;\n  return ',
  suffix: '\n}',
  language: 'javascript',
  filePath: 'src/math.js'
});

// system: "You are a code completion assistant..."
// messages: [{ role: 'user', content: '...' }]
```

### 提交消息生成

```javascript
const { buildCommitMessagePrompt } = require('./prompts/commit-message-stream');

const { system, messages } = buildCommitMessagePrompt({
  diff: 'diff --git a/src/index.js...',
  recentCommits: ['fix: typo', 'feat: add login']
});

// 生成符合 Conventional Commits 规范的消息
```

## 依赖关系

```
common.js (基础工具)
  │
  ├── completion.js
  ├── chat-input-completion.js
  ├── edit.js
  ├── instruction-stream.js
  ├── next-edit-stream.js
  ├── next-edit-loc.js
  ├── smart-paste-stream.js
  ├── prompt-enhancer.js
  ├── commit-message-stream.js
  └── conversation-title.js
```

## 注意事项

1. **Token 预算**: 提示词需考虑模型 token 限制，使用 `truncate()` 控制长度
2. **格式一致性**: 输出格式约束需与 `core/protocol.js` 的解析逻辑匹配
3. **多语言支持**: 提示词本身使用英文，确保各模型兼容性
4. **敏感信息**: 不在提示词中硬编码任何敏感配置
