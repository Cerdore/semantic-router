# Potentially Unfixed Edge Cases in "Fixed" Bugs

- **来源**: 代码审查
- **发现日期**: 2026-04-25

## 5.1 — Streaming Cache Fix (#1555) — Potential Silent Failure

- `cacheStreamingResponse()` 在 `finalizeStreamingResponse()` 中调用
- 如果 `reconstructStreamingResponse` 失败，只记录 warning，响应仍发送
- 缓存失败可能直到缓存命中率下降才被发现

## 5.2 — Cross-user Cache Fix (#1448) — Missing Cache Migration

- `ScopeQueryToUser()` 改变了缓存键格式
- 如果存在修复前的缓存条目（无用户隔离），这些条目仍可跨用户访问
- 无缓存迁移或失效机制

## 5.3 — enable_thinking Fix (#858) — May Need New Model Validation

- 修复显式为 Qwen3/DeepSeek 设置 `enable_thinking: false`
- 未来的推理模型如果未被添加到列表，可能有同样问题
- 无通用机制处理未知推理模型

## 5.4 — Cache Key Normalization with Empty UserID (#1448 follow-up)

- **File**: `cache/cache.go:66-68`
- `ScopeQueryToUser()` 在 `userID == ""` 时返回原查询（"向后兼容"）
- 如果 ext_authz 配置错误且 userID 为空，缓存隔离静默失效
- 用户隔离实际禁用时无 warning 日志
