# 1577 — Website Classifies Modality Signal as Heuristic

- **来源**: https://github.com/vllm-project/semantic-router/issues/1577
- **优先级**: P2
- **状态**: resolved (已确认修复)
- **发现日期**: 2026-04-25
- **涉及模块**: docs/website

## 问题描述

Website 将 Modality Signal 分类为 heuristic 而非 learned。

## 复现结果

全部 9 步验证均确认已修复。代码、文档、sidebar 配置一致: modality 是 learned signal。

## Root Cause

Bug 在 issue 提交时确实存在。由 commit `29afc8cf` 修复，该 commit 将 signal 文档重组为 heuristic/learned 子目录。

## 结论

Issue 可以关闭。

## 备注

详见 `.claude/issue-1577-reproduction.md`
