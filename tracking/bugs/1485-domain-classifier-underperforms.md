# 1485 — Default Domain Classifier Underperforms on Unseen Data

- **来源**: https://github.com/vllm-project/semantic-router/issues/1485
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: research/classification

## 问题描述

mmBERT-32K 域分类器:
- 在 MMLU (14,042 独立问题) 上得分 **78.2%**
- 之前在 MMLU-Pro test split 上显示 ~82% — 但模型在该 split 上训练过（膨胀）
- 实际准确率低于声称值

## 影响

路由决策基于不准确的域分类，模型选择可能错误。

## 修复思路

1. 在独立 holdout 集上重新评估
2. 考虑微调或替换分类器
3. 更新文档中的准确率声明

## 备注

-
