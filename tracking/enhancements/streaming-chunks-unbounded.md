# Streaming Chunks Unbounded Memory Growth

- **来源**: 代码分析
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/request_context.go:65-67

## 问题描述

`StreamingChunks []string` 和 `StreamingContent string` 无大小限制地累积所有 chunk。超长流式响应（如代码生成）可能导致无界内存增长。

## 现有缓解

典型 SSE 响应大小有限，但无显式上限。

## 修复思路

添加 `StreamingChunks` 大小上限。

## 备注

-
