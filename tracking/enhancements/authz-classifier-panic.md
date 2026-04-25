# Authz Classifier Panic on Unexpected Subject Kind

- **来源**: 代码分析
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: classification/authz_classifier.go:157

## 问题描述

```go
panic("this is a bug: NewAuthzClassifier should have rejected this at startup")
```

如果配置重载引入了 `NewAuthzClassifier` 未验证的新 subject kind，服务器会崩溃。

## 影响

配置热重载可能导致服务崩溃。

## 修复思路

将 `panic()` 改为 logged error + safe fallback。

## 备注

-
