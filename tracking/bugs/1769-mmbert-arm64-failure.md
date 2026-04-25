# 1769 — mmBERT Model Fails to Load on arm64

- **来源**: https://github.com/vllm-project/semantic-router/issues/1769
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: core/classification

## 问题描述

mmBERT unified embedding 模型在 arm64 (Apple Silicon/k3s) 上加载失败:
- 错误: `cannot find tensor _orig_mod.model.embeddings.tok_embeddings.weight`
- 所有 12 种 signal type 被跳过
- 每次分类返回 `category: "other", confidence: 0`
- BERT similarity 模型从同一目录加载正常 — 只有 mmBERT 失败

## 影响

arm64 平台上分类功能完全不可用，但 router 正常启动（无健康检查检测到模型产生垃圾结果）。

## 假设

arm64 candle 二进制文件编译时使用了与 amd64 不同的模型权重格式。

## 修复思路

1. 调查 arm64 candle 二进制文件的模型格式兼容性
2. 添加模型加载后的健康检查（验证分类结果不是全 confidence=0）
3. 加载失败时回退到 BERT similarity 模型

## 备注

关联: #1625 (已修复: HF_TOKEN 缺失时 router 崩溃)
