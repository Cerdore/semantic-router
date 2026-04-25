# CA-1 — HNSW selectLevel() Uses Broken Randomness, Graph Degenerates to Linear Search

- **来源**: 代码分析 (unreported)
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: cache/inmemory_cache.go:1138-1141, cache/hybrid_cache_hnsw.go:168-170

## 问题描述

`time.Now().UnixNano()%1000000/1000000.0` 用作假随机数。批量插入时所有节点获得相同层级，HNSW 图坍缩为单层。

搜索复杂度从 O(log n) 退化为 O(n)。

## 复现

1. 部署 router 配置 `use_hnsw: true`
2. 快速连续发送 100+ 语义不同的请求
3. 对比 FindSimilar 延迟中位数与 `use_hnsw: false`
4. 实测加速比 ~1.0x (预期 ~5-10x)

## 修复

```go
// 使用 math/rand/v2 (Go 1.22+)
func (h *HNSWIndex) selectLevel() int {
    r := rand.Float64()
    if r == 0.0 {
        r = 1e-9
    }
    return int(-math.Log(r) * h.ml)
}
```

## 备注

简单一行修复，巨大的性能影响。关联 Bug CA-2 (同根因)。
