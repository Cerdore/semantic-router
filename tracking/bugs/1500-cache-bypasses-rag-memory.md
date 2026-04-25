# 1500 — Cache Hits Bypass RAG and Memory Injection Pipeline

- **来源**: https://github.com/vllm-project/semantic-router/issues/1500
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: core/cache

## 问题描述

管线顺序:
1. `applyRateLimitAndCacheChecks()` (cache READ)
2. `executeRAGPlugin()`
3. `prepareRequestForModelRouting()` (memory retrieval)

缓存命中后立即返回；RAG 上下文和用户 memory 从未注入。用户得到有效的通用答案，只是没有个性化。

## 影响

正确性问题 — 非安全漏洞，但影响用户体验和个性化质量。

## 修复思路

1. 在缓存查找之前移动 RAG/memory 注入
2. 或将 RAG/memory 上下文纳入缓存键
3. 至少对缓存命中结果做 RAG/memory 后处理

## 备注

-
