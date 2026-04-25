# 1452 — Feedback API Accepts Unauthenticated Requests

- **来源**: https://github.com/vllm-project/semantic-router/issues/1452
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: api/feedback

## 问题描述

`POST /api/v1/feedback` 在 8080 端口上没有认证。攻击者可以提交数千条虚假反馈来改变所有用户的 Elo 评分。

## 安全影响

Elo 评分直接影响使用 Elo/RL-driven/GMTRouter 时的模型选择。攻击者可操纵路由决策，将请求导向特定模型。

## 修复思路

1. 为 feedback API 添加认证中间件
2. 验证请求来源（Envoy sidecar vs 外部）
3. 添加 rate limiting

## 备注

关联: #1443, #1445 — 同为认证/授权缺失类安全问题
