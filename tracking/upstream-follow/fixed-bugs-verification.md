# Fixed Bugs — Regression Verification Checklist

> 已确认修复的 bugs，需在新版本发布时回归验证。

## Verified Fixed (code confirmed)

| # | 简述 | 验证方式 | 验证日期 |
|---|------|---------|---------|
| 1766 | Envoy config host_rewrite + duplicated /v1 | 代码确认 | 2026-04-25 |
| 1555 | Streaming responses never written to cache | 代码确认 | 2026-04-25 |
| 1448 | Cache no user isolation (cross-user leak) | 代码确认 | 2026-04-25 |
| 929 | getRecommendedModel returns invalid names | 代码确认 | 2026-04-25 |
| 858 | enable_thinking not set for Qwen3/DeepSeek | 代码确认 | 2026-04-25 |
| 913 | Streaming + cache SSE broken | 代码确认 | 2026-04-25 |
| 1126 | Jailbreak/PII plugins global scope | 代码确认 | 2026-04-25 |
| 1577 | Modality signal misclassified as heuristic | 9-step repro | 2026-04-25 |

## Closed (unverified)

| # | 简述 | 备注 |
|---|------|------|
| 1447 | Authz-rbac integration test raw headers | |
| 1625 | Router crash on HF_TOKEN not set | |
| 1729 | Memory integration test flaky | |
| 1309 | Not enough PII types error | |
| 1314 | Unable to use complexity signal | |
| 1405 | External model config not picked up | |
| 952 | Same semantics different classification | |
| 953 | Cache fails with qwen3 embedding | |
| 940 | Cache hits reuse response.id | |
| 928 | embedding_model ignored, still uses BERT | |
| 1590 | CI failing on production stack | |
| 1598 | make agent-bootstrap fails on Homebrew Python | |
| 1599 | Feature request template missing label | |
| 1600 | CONTRIBUTING.md missing requirements.txt | |
| 1603 | AI-gateway progressive stress 500s | |
| 1481 | ML Setup page missing from nav | |
| 1364 | build-router-onnx failed | |
| 1307 | vllm-sr status incorrect | |
| 1199 | Router fails missing PII mapping file | |
| 1162 | Language signal missing from dashboard | |
| 1062 | test-pii-classifier fails | |
| 1050 | CUTLASS submodule not downloaded | |
| 1020 | Envoy initialization failure | |
| 980 | Fix response-api E2E profile | |
| 971 | ToolsDB handler 500 instead of 404 | |
| 964 | Typo "Intall" in docs | |
| 958 | Docs index page display error | |
| 918 | Cannot open shared object file | |
| 866 | Enable kv-cache in Dynamo | |
