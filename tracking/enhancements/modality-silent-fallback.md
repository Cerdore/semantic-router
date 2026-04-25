# Modality Signal Silently Defaults to AR with 0 Confidence

- **来源**: 代码分析
- **优先级**: P3
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: classification/classifier_category_entropy.go:42

## 问题描述

记录 "BUG: unknown detection method" 但返回 `{Modality: "AR", Confidence: 0.0, Method: "error/unknown-method"}`。静默回退掩盖了配置错误。

## 修复思路

以更明显的方式暴露配置错误（如 metric 计数、health check warning）。

## 备注

-
