# 1456 — No Limits on Looper Breadth Schedule or modelRefs Count

- **来源**: https://github.com/vllm-project/semantic-router/issues/1456
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: core/looper

## 问题描述

- `breadth_schedule` 是 `[]int`，无上限验证
- `breadth_schedule: [32, 16, 8]` = 每个请求 57 次后端调用
- 10 个 modelRefs + confidence 算法 = 最多 10 次顺序调用
- 无 per-user 成本追踪

## 安全影响

恶意配置可能导致单请求产生大量后端调用，成本爆炸。

## 修复思路

1. 验证 `breadth_schedule` 长度和元素值上限
2. 限制 `modelRefs` 数量
3. 添加 per-request 总调用次数上限
4. 添加 per-user 成本追踪

## 备注

-
