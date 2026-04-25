# 1454 — No Input Size Limit for Signal Evaluation (DoS)

- **来源**: https://github.com/vllm-project/semantic-router/issues/1454
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/classification

## 问题描述

Signal evaluation 延迟超线性增长:
- 10K chars → 20,856ms
- 25K chars → >30s (timeout)

多个大 prompt 会使 router 完全无响应（health endpoint 也停止）。

**代码位置**: `prepareSignalEvaluationInput()` (`req_filter_classification_signal.go:23`) — 无大小验证

## 安全影响

DoS 攻击可导致 router 服务完全不可用。

## 修复思路

1. 添加 `evaluationText` 大小上限
2. 超长输入截断或分块处理
3. 添加 per-request timeout

## 备注

-
