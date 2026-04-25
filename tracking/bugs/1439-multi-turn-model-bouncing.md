# 1439 — Multi-turn Conversations Bounce Between Models

- **来源**: https://github.com/vllm-project/semantic-router/issues/1439
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: core/routing

## 问题描述

每轮对话独立评估；简单的 follow-up "commit it" 路由到更便宜的模型。每轮选择的模型仅取决于最后一条用户消息，而非对话上下文。

## 影响

多轮对话中模型跳跃，用户体验差。编码对话中途切换到非编码模型。

## 修复思路

1. 多轮对话使用 session affinity
2. 将对话历史纳入路由决策
3. 在 follow-up 检测到时不切换模型

## 备注

关联: #1751 (multi-turn session context)
