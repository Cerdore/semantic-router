# Semantic Router Issue Tracking

> 常态化跟踪 vllm-project/semantic-router 的待修复 issues。
> 同步上游: `git checkout main && git fetch upstream && git merge upstream/main`
> 更新 tracking: `git checkout issue-tracking`

## 优先级矩阵

| 优先级 | 说明 |
|--------|------|
| P0 | 阻塞性安全漏洞，可导致系统被入侵 |
| P1 | 重要 bug / 安全风险，近期应修复 |
| P2 | 一般问题，排期修复 |
| P3 | 增强/优化，低优先级 |

---

## Active Bugs

### P0 — Security

| # | 文件 | 简述 |
|---|------|------|
| 1443 | [looper-header-injection.md](bugs/1443-looper-header-injection.md) | Looper header 可绕过所有安全插件 |
| 1445 | [identity-header-spoofing.md](bugs/1445-identity-header-spoofing.md) | 无 ext_authz 时身份 header 可伪造 |

### P1 — Core / Security / Routing

| # | 文件 | 简述 |
|---|------|------|
| 1452 | [feedback-api-no-auth.md](bugs/1452-feedback-api-no-auth.md) | Feedback API 无认证，可操纵 Elo 评分 |
| 1454 | [input-size-limit-dos.md](bugs/1454-input-size-limit-dos.md) | 无输入大小限制，可 DoS 攻击 |
| 1456 | [looper-breadth-no-limit.md](bugs/1456-looper-breadth-no-limit.md) | looper 广度无上限，成本爆炸风险 |
| 1769 | [mmbert-arm64-failure.md](bugs/1769-mmbert-arm64-failure.md) | mmBERT 在 arm64 上加载失败，分类全部返回 "other" |
| 1751 | [multi-turn-session-context.md](bugs/1751-multi-turn-session-context.md) | 多轮对话缺少 session 上下文 |
| 1500 | [cache-bypasses-rag-memory.md](bugs/1500-cache-bypasses-rag-memory.md) | 缓存命中绕过 RAG 和 memory 注入 |
| 1485 | [domain-classifier-underperforms.md](bugs/1485-domain-classifier-underperforms.md) | 域分类器实际准确率仅 78.2% |
| 1439 | [multi-turn-model-bouncing.md](bugs/1439-multi-turn-model-bouncing.md) | 多轮对话模型跳跃，缺少亲和性 |

### P1 — Code Analysis (unreported)

| # | 文件 | 简述 |
|---|------|------|
| CA-1 | [hnsw-selectlevel-broken.md](bugs/hnsw-selectlevel-broken.md) | HNSW 层级分配使用假随机数，图退化为线性搜索 |
| CA-2 | [hnsw-randfloat-broken.md](bugs/hnsw-randfloat-broken.md) | 混合缓存 randFloat() 仅 1000 个熵值 |
| CA-3 | [lora-double-init-race.md](bugs/lora-double-init-race.md) | ensureLoRAInitialized 损坏的双重检查加锁 |

### P2 — Code Analysis (unreported)

| # | 文件 | 简述 |
|---|------|------|
| CA-4 | [bestentry-tocctou-race.md](bugs/bestentry-tocctou-race.md) | bestEntry 快照 TOCTOU 索引过期 |
| CA-5 | [valkey-last-accessed-broken.md](bugs/valkey-last-accessed-broken.md) | Valkey last_accessed 从未更新 |
| CA-6 | [valkey-last-accessed-precision.md](bugs/valkey-last-accessed-precision.md) | last_accessed 精度不匹配 (秒 vs 毫秒) |

### P2 — Reported

| # | 文件 | 简述 |
|---|------|------|
| 1577 | [modality-signal-misclassified.md](bugs/1577-modality-signal-misclassified.md) | Website 将 Modality Signal 分类为 heuristic（已确认已修复） |

---

## Enhancements

| 文件 | 简述 |
|------|------|
| [pii-classifier-chunking.md](enhancements/pii-classifier-chunking.md) | PII 分类器长内容需要分块处理 |
| [authz-classifier-panic.md](enhancements/authz-classifier-panic.md) | Authz 分类器 panic 应改为 error |
| [mcp-tool-stub.md](enhancements/mcp-tool-stub.md) | MCP 工具调用未实现 (stub) |
| [gpu-detection-hardcoded.md](enhancements/gpu-detection-hardcoded.md) | GPU 检测硬编码返回 false |
| [session-transition-model.md](enhancements/session-transition-model.md) | Session transition 缺少 PreviousModel |
| [streaming-chunks-unbounded.md](enhancements/streaming-chunks-unbounded.md) | 流式 chunks 无界内存增长 |
| [modality-silent-fallback.md](enhancements/modality-silent-fallback.md) | Modality signal 未知方法静默回退 |
| [embedding-model-health-check.md](enhancements/embedding-model-health-check.md) | 模型下载失败缺少健康检查 |

---

## Upstream Follow

| 文件 | 简述 |
|------|------|
| [security-pattern-headers.md](upstream-follow/security-pattern-headers.md) | 安全模式分析：信任客户端 header |
| [fixed-bugs-verification.md](upstream-follow/fixed-bugs-verification.md) | 已修复 bugs 的回归验证清单 |
| [potentially-unfixed-edges.md](upstream-follow/potentially-unfixed-edges.md) | "已修复" bugs 中潜在未修复的边缘情况 |
