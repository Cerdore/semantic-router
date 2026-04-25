# CA-2 — Hybrid Cache randFloat() Has Only 1000 Entropy Values

- **来源**: 代码分析 (unreported)
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: cache/hybrid_cache_hnsw.go:168-170

## 问题描述

```go
func randFloat() float64 {
    return float64(time.Now().UnixNano()%1000) / 1000.0
}
```

仅 1,000 个可能值 (0.000 to 0.999)，通过 `selectLevelHybrid` 中的循环调用暴露模式。序列确定且熵极低。

## 修复

用 `math/rand/v2` 替换。

## 备注

与 CA-1 同根因，修复方式相同。
