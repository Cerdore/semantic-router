# CA-5 — Valkey last_accessed Never Updated on Retrieval

- **来源**: 代码分析 (unreported)
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: memory/valkey_store_helpers.go:42-62, 428

## 问题描述

`recordRetrieval` 更新 `access_count` (通过 `HINCRBY`) 和 `updated_at` (通过 `HSET`)，但**从未更新 `last_accessed`**。

架构约束: `last_accessed` 存储在 metadata JSON 块内部，不能原子更新。并发检索可能导致一个 goroutine 覆盖另一个的 `last_accessed` 值。

## 影响

`last_accessed` 在语义上具有误导性 — 它是创建时间戳，而非真正的最后访问时间。

## 复现

```bash
curl -X POST "$ROUTER_URL/v1/memory" -d '{"content":"test","user_id":"u1"}'
curl "$ROUTER_URL/v1/memory?user_id=u1"  # 注意 last_accessed
sleep 2
curl "$ROUTER_URL/v1/memory?user_id=u1"  # last_accessed 未改变
```

## 修复

将 `last_accessed` 提升为 Valkey 中的顶级 HSET 字段。

## 备注

-
