# GPU Detection Not Implemented (Hardcoded false)

- **来源**: 代码分析
- **优先级**: P3
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: apiserver/route_model_info.go:361

## 问题描述

`GPUAvailable: false` 硬编码，注释: "TODO: Implement GPU detection"

Model info API 返回不正确的 GPU 可用性状态。

## 影响

下游依赖 GPU 可用性信息的组件收到错误数据。

## 备注

-
