[根目录](../CLAUDE.md) > **tools**

# 构建与补丁工具模块

> 负责下载上游 Augment VSIX、应用补丁、注入 BYOK 代码、打包最终产物的构建工具集。

## 模块职责

1. **构建**: 下载上游 VSIX，解包，overlay payload，repack
2. **补丁**: 修改 `extension.js` 和 `package.json`，注入 BYOK 逻辑
3. **检查**: 验证补丁完整性和合约
4. **报告**: 生成端点覆盖率报告

## 入口与启动

- **主入口**: `build/build-vsix.js` - 完整构建流程
- **分析入口**: `build/upstream-analyze.js` - 分析上游端点

```bash
# 构建 BYOK VSIX
npm run build:vsix

# 分析上游端点
npm run upstream:analyze

# 生成覆盖率报告
npm run report:coverage
```

## 对外接口

### build/build-vsix.js

完整构建流程:
1. 下载上游 VSIX
2. 解包到 `.cache/work/`
3. overlay `payload/extension/` 到解包目录
4. 应用各项补丁
5. 运行 guard 检查
6. repack 为新 VSIX

### 补丁模块

| 补丁文件 | 功能 |
|---------|------|
| `patch-augment-interceptor-inject.js` | 注入 augment-interceptor |
| `patch-extension-entry.js` | 注入 bootstrap 引导 |
| `patch-official-overrides.js` | 替换 completionURL/apiToken 读取逻辑 |
| `patch-callapi-shim.js` | 拦截 callApi/callApiStream |
| `patch-expose-upstream.js` | 暴露 toolsModel 等内部变量 |
| `patch-package-json-commands.js` | 添加 BYOK 命令到 package.json |
| `guard-no-autoauth.js` | 检查产物不含 autoAuth |

## 关键依赖与配置

### 外部依赖

- Node.js 20+
- Python 3 (用于 zip/unzip)
- 网络访问 VS Marketplace

### 工具库 (lib/)

- `fs.js` - 文件系统操作
- `http.js` - HTTP 下载
- `patch.js` - 补丁工具函数
- `run.js` - 子进程执行

## 数据模型

### upstream.lock.json

```json
{
  "upstream": { "version": "x.y.z", "url": "...", "sha256": "..." },
  "interceptorInject": { "file": "...", "sha256": "..." },
  "output": { "file": "...", "sha256": "..." },
  "generatedAt": "2026-01-18T..."
}
```

### upstream-analysis.json

```json
{
  "endpoints": [
    { "path": "/chat-stream", "kind": "callApiStream", "hasTransform": true }
  ],
  "stats": { "total": 71, "callApi": 40, "callApiStream": 31 }
}
```

## 测试与质量

- 构建期自动运行 `node --check` 语法检查
- 构建期自动运行 `byok-contracts.js` 合约检查
- CI 自动运行端点覆盖率检查（LLM 端点变化会 fail-fast）

## 常见问题 (FAQ)

**Q: 构建失败怎么办?**
A: 检查网络连接、Node.js/Python 版本，查看错误信息中的 patch needle 是否匹配。

**Q: 上游升级后构建失败?**
A: 可能是上游代码结构变化导致 patch needle 不匹配，需要更新补丁代码。

**Q: 如何保留中间产物调试?**
A: 设置 `AUGMENT_BYOK_KEEP_WORKDIR=1` 环境变量。

## 相关文件清单

```
tools/
  build/
    build-vsix.js        # 主构建脚本
    upstream-analyze.js  # 上游端点分析
  check/
    byok-contracts.js    # BYOK 合约检查
  lib/
    fs.js                # 文件系统工具
    http.js              # HTTP 工具
    patch.js             # 补丁工具
    run.js               # 进程执行
  patch/
    guard-no-autoauth.js              # autoAuth guard
    patch-augment-interceptor-inject.js
    patch-callapi-shim.js
    patch-expose-upstream.js
    patch-extension-entry.js
    patch-official-overrides.js
    patch-package-json-commands.js
  report/
    endpoint-coverage.js    # 覆盖率报告
    llm-endpoints-spec.js   # LLM 端点规格
```

## 变更记录 (Changelog)

### 2026-01-18
- 初始化模块文档

