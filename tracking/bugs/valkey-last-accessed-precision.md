# CA-6 — last_accessed Precision Mismatch (seconds vs milliseconds)

- **来源**: 代码分析 (unreported)
- **优先级**: P3
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: memory/valkey_store_helpers.go:428, 517

## 问题描述

- `last_accessed` 以**秒**精度存储: `memory.LastAccessed.Unix()`
- 读取为 `time.Unix(int64(lastAccessed), 0)` — 纳秒部分始终为 0
- 所有其他时间戳使用 `UnixMilli()`，精度为毫秒

一秒内的多次访问产生相同的 `last_accessed` 值。

## 修复

统一使用毫秒精度。

## 备注

低优先级，美观性问题。
