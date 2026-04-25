# Embedding Model Download Failure Handling Is Inconsistent

- **来源**: 代码分析 (关联 #1625, #1769)
- **优先级**: P2
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: classification

## 问题描述

模型下载在某些代码路径中可能静默失败（如 mmBERT on arm64）。无健康检查机制检测模型加载但产生垃圾结果（如所有查询 confidence=0）。Router 成功启动但实际不可用。

## 修复思路

1. 添加模型加载后健康检查
2. 验证分类结果不是全 confidence=0
3. 健康检查失败时降级或报警

## 备注

关联: #1625 (HF_TOKEN), #1769 (mmBERT arm64)
