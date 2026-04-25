# CA-4 — bestEntry Snapshot TOCTOU Race with Expired Index

- **来源**: 代码分析 (unreported)
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: cache/inmemory_cache.go:613-648

## 问题描述

在 RUnlock 和 Lock 之间，bestIndex 可能因清理/逐出而失效:

```go
c.mu.RLock()
// ... 搜索时 bestIndex 有效
bestEntry = c.entries[bestIndex]  // 快照值
c.mu.RUnlock()                     // ← 窗口: 清理/逐出可移除条目

c.mu.Lock()
c.updateAccessInfo(bestIndex, bestEntry) // ← bestIndex 可能无效
c.mu.Unlock()
```

## 现有缓解

`updateAccessInfo` 有 RequestID 验证 + 线性搜索后备。防止 panic，但不能防止错误的条目更新 (极低概率)。已足够缓解。

## 修复

添加边界检查或重构为单次锁持有。

## 备注

已有缓解措施，优先级较低。
