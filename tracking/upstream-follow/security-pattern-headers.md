# Security Pattern: Trusting Client-Supplied Headers Without Origin Validation

- **来源**: 安全审查
- **发现日期**: 2026-04-25

## 模式描述

代码库中存在反复出现的模式：**信任客户端提供的 header 而不验证来源**。三个相关 bug 共享同一根因：

1. **#1443** — Looper header (`x-vsr-looper-request`) 绕过所有插件
2. **#1445** — Identity headers (`x-authz-user-id`, `x-authz-user-groups`) 可伪造
3. **#1452** — Feedback API 无认证

## 修复模式

Envoy proxy 层必须在 header 到达 extproc server 前剥离/验证这些 header。但 extproc 代码本身不验证 header 来源。

## 建议

- 审查所有从请求 header 读取的内部 header
- 添加 header provenance 验证中间件
- 文档化 Envoy 配置要求

## 相关文件

- `processor_req_header.go:77-79` (#1443)
- `processor_req_body_memory.go:243-244` (#1445)
