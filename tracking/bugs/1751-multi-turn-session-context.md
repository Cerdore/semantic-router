# 1751 — Multi-turn Follow-up Routing Uses Isolated Last-turn Classification

- **来源**: https://github.com/vllm-project/semantic-router/issues/1751
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: core/routing, extproc/classification

## 问题描述

`SelectionContext` 有 `SessionID`, `UserID`, `ConversationHistory` 字段，但 extproc 分类路径未一致填充它们。

Follow-up 如 "looks good, commit it" 在没有先前编码上下文的情况下路由到无关模型。

## 影响

- `selectModelFromCandidates`
- GMTRouter
- RL-driven selection

## 修复思路

需要通过分类管线传递 session ID。

## 备注

关联: #1439 (multi-turn model bouncing)
