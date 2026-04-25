# MCP Tool Invocation Not Implemented (Stub)

- **来源**: 代码分析
- **优先级**: P3
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/req_filter_rag_mcp.go:44

## 问题描述

代码注释: "TODO: Implement MCP tool invocation when MCP client is available"

MCP-based RAG 工具是 stub；任何依赖 MCP tool call 的决策会静默 no-op。

## 修复思路

实现 MCP client 集成。

## 备注

-
