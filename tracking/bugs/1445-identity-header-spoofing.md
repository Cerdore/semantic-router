# 1445 — Identity Header Spoofing Without ext_authz

- **来源**: https://github.com/vllm-project/semantic-router/issues/1445
- **优先级**: P0
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: extproc/auth

## 问题描述

没有外部认证 provider 时，`x-authz-user-id` 和 `x-authz-user-groups` 直接从客户端请求 headers 读取，不做任何验证。

**代码位置**: `processor_req_body_memory.go:243-244`

```go
ctx.Headers[r.Config.Authz.Identity.GetUserIDHeader()]
```

## 安全影响

攻击者可以伪造任意用户身份，访问其他用户的 memory 数据、操纵个人化路由决策。

## 安全模式

与 #1443 同属一类问题：信任客户端提供的 header 而不验证来源。

## 修复思路

1. 强制要求 ext_authz 配置，拒绝无认证的请求
2. Envoy 层面剥离外部 identity headers
3. extproc 验证 header 是否来自可信的 ext_authz provider

## 备注

关联: #1443 (looper header injection), #1447 (已修复: authz-rbac 集成测试)
