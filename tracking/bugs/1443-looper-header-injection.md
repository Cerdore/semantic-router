# 1443 — Looper Header Injection Bypasses Security Plugins

- **来源**: https://github.com/vllm-project/semantic-router/issues/1443
- **优先级**: P0
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/security

## 问题描述

任何客户端都可以发送 `x-vsr-looper-request: true` 来跳过整个安全插件管线。Looper header 直接从客户端读取，没有来源验证。

**代码位置**: `processor_req_header.go:77-79`

## 安全影响

攻击者可以完全绕过 jailbreak 检测、PII 过滤、RBAC 权限检查等所有安全插件。

## 安全模式

与 #1445 同属一类问题：信任客户端提供的 header 而不验证来源。正确做法是由 Envoy proxy 层剥离/验证这些 header，但 extproc 代码本身不验证 header 来源。

## 修复思路

1. Looper header 应由内部 looper 组件签名，extproc 验证签名
2. 或通过 Envoy 配置剥离外部 `x-vsr-looper-request` header，只信任内部来源
3. 至少检查请求来源 IP/网络是否为内部 looper 服务

## 备注

关联: #1445 (identity header spoofing), #1452 (feedback API no auth)
