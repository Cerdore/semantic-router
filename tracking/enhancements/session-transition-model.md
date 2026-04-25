# Session Transition Missing PreviousModel for Chat Completions

- **来源**: 代码分析
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/session_transition.go:51

## 问题描述

"TODO: populate PreviousModel for Chat Completions once per-turn model history is available"

Session transition logic (cache warmth/performance estimation) 缺少 Chat Completions API 的模型历史。

## 影响

多轮聊天场景中模型选择可能次优。

## 备注

-
