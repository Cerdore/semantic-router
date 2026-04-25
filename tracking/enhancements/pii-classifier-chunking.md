# PII Classifier Needs Chunking for Long Content

- **来源**: 代码分析
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: classification/classifier_pii_ops.go:131

## 问题描述

代码注释: "TODO: classifier may not handle the entire content, so we need to split the content into smaller chunks"

PII 分类器处理全文而不分块；可能在超长 prompt 中静默漏掉 PII 或完全失败。

## 影响

长输入的 PII 检测假阴性。

## 修复思路

实现内容分块逻辑，分批送入 PII 分类器。

## 备注

-
