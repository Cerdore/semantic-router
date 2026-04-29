# Stream Computing & Database Inspired Routing Ideas

**Status:** Brainstorming / Ideation
**Author:** @chenxiansen.cxs
**Date:** 2026-04-28
**Related:** `routing-hint-and-plan-binding.md`, `routing-hint-and-plan-binding-evaluation.md`

## Overview

在已有的 Hint Injection + Plan Binding 提案基础上，借鉴流计算和数据库领域的成熟思想，发掘更多可落地的路由增强能力。本文档是脑暴产物，每条 idea 独立可评估，部分可合并为一个完整的 "Routing Intelligence Layer"。

---

## Idea 1: Circuit Breaker — 模型健康状态机

**灵感来源:**
- **DB 连接池健康检查** — 连接池定期探活，坏连接踢出，恢复后重新加入
- **流计算 CEP** — 复合事件模式检测（"3 次超时 + 延迟飙升 → 降级"）
- **Netflix Hystrix / Resilience4j** — 经行业验证的断路器模式

**已有基础设施:**
- `WindowedMetricsManager` 已在收集 per-model 的 error rate、P50/P95/P99 latency、queue depth
- `LatencyAwareSelector` 已展示了"用实时延迟数据影响路由"的模式
- `DependencyHealth` Prometheus gauge 已存在

**缺失:** 没有任何消费这些窗口指标来影响路由决策的代码。健康检查仅限启动时。

**设计概要:**

```
Model Health State Machine:

     ┌──────────┐     errors > threshold      ┌──────────┐
     │  CLOSED  │ ──────────────────────────→  │   OPEN   │
     │ (正常路由) │                             │ (停止路由) │
     └──────────┘                             └──────────┘
           ↑                                        │
           │                                  cooldown expired
           │                                        │
           │          ┌──────────────┐              │
           └──────────│  HALF_OPEN   │←─────────────┘
                      │ (探测流量恢复) │
                      └──────────────┘
```

**配置:**

```yaml
routing:
  circuitBreakers:
    - model: "deepseek-v31"
      errorRateThreshold: 0.05        # 5% 错误率触发
      latencyP99Threshold: "10s"      # 或 P99 > 10s 触发
      consecutiveFailures: 5          # 或连续 5 次失败触发
      openDuration: "60s"             # OPEN 状态持续多久
      halfOpenMaxRequests: 10         # HALF_OPEN 时允许多少探测请求
      halfOpenSuccessRate: 0.8        # 探测成功率门槛恢复 CLOSED
      windowSize: "1m"                # 评估窗口
```

**关键决策:**
- 断路器在 `SelectionContext` 层工作：OPEN 状态的模型从 `CandidateModels` 中移除
- HALF_OPEN 时：允许有限请求通过，用于探测恢复
- 结合 CEP 模式：不只单阈值，可定义复合条件（"error rate > 5% AND P99 > 10s"）
- 断路器状态变化作为 event 写入 replay store，便于事后分析

**v2 展望:**
- 自适应阈值（基于历史基线的动态阈值，而非静态配置）
- 部分降级（不停止全部流量，降级到 `route_weights=healthy:70,degraded:30`）

---

## Idea 2: Cost-Based Optimizer (CBO) — 多维代价模型

**灵感来源:**
- **PostgreSQL/MySQL 查询优化器** — `cost = cpu_cost + io_cost + network_cost`，用表统计信息（直方图、MCV）做选择性估计
- **Oracle Optimizer** — `optimizer_mode` (ALL_ROWS, FIRST_ROWS), `dbms_stats` 统计信息管理
- **EXPLAIN PLAN** — 展示优化器为什么选择了某个执行计划

**已有基础设施:**
- `ModelPricing` 已有 per-model 定价（PromptPer1M, CompletionPer1M）
- `SelectionContext` 已有 `CostWeight` 和 `QualityWeight` 字段
- `HybridSelector` 已有 `applyCostAdjustment()` 给便宜模型加分
- `SessionTurnCost` Prometheus histogram 已在追踪每轮开销
- `WindowedMetricsManager` 有 P50/P95/P99 延迟
- `LatencyAwareSelector` 有 TTFT/TPOT 数据

**缺失:** 代价是静态配置（`CostWeight`），不是从实时数据动态计算的。没有 EXLAIN ROUTE 能力。没有多维代价的联合建模。

**设计概要:**

```
Route Cost = w₁ × NormalizedLatency(model, category)
           + w₂ × NormalizedPrice(model)
           + w₃ × (1 - HistoricalAccuracy(model, category))
           + w₄ × (1 - HealthScore(model))
           + w₅ × CapacityPenalty(model)        ← 接近并发上限时惩罚
```

- 每个模型的 "统计信息" 定期刷新（类似 DB 的 `ANALYZE`）
- 延迟来自 live 窗口指标，准确率来自用户反馈，价格来自配置
- 用户可通过 hint 调整权重：`/*+ VSR: cost_weights=latency:0.6,price:0.4 */`

**"EXPLAIN ROUTE" 能力:**

```json
// GET /v1/routing/explain?query=write a quick sort
{
  "candidates": [
    {"model": "deepseek-v31", "cost": 0.23, "latency_ms": 450, "price_per_1k": 0.002,
     "accuracy": 0.92, "health": 0.99},
    {"model": "qwen3-32b",    "cost": 0.31, "latency_ms": 380, "price_per_1k": 0.004,
     "accuracy": 0.88, "health": 1.0},
    {"model": "gpt-4o-mini",  "cost": 0.45, "latency_ms": 520, "price_per_1k": 0.001,
     "accuracy": 0.85, "health": 1.0}
  ],
  "selected": "deepseek-v31",
  "reason": "最低综合代价 (0.23), 延迟和价格的加权最优"
}
```

**v2 展望:**
- "EXPLAIN ROUTE" 交互式 UI dashboard
- 代价模型的在线学习（用实际结果反馈修正代价估计）

---

## Idea 3: Shadow Routing & A/B Verification（SPM 风格）

**灵感来源:**
- **Oracle SQL Plan Management (SPM)** — 新执行计划先以 "unaccepted" 进入，shadow 执行并与 accepted plan 对比，只有证明更优才 promote
- **Oracle 23ai Real-Time SPM** — foreground verification：在决策时双路执行，异步比较
- **DB Plan Baseline Evolution** — 自动 evolve baseline

**已有基础设施:**
- 无。搜索结果中 "shadow" 全文搜索返回零结果。

**缺失:** 完全没有流量镜像、dark launch、A/B 测试的基础设施。

**设计概要:**

```
Shadow Route Flow:

Request ──→ 主路由 (accepted binding/model)
         │
         └──→ Shadow 路由 (candidate) ──→ 异步比较 ──→ Promote/Reject

比较维度 (Compound Improvement Ratio):
  CIR = (candidate_latency / baseline_latency)
      × (candidate_cost / baseline_cost)
      × (baseline_accuracy / candidate_accuracy)
  Promote when CIR < 0.85 (candidate 综合改善 > 15%)
```

**配置:**

```yaml
routing:
  shadowExperiments:
    - name: "deepseek-v31-migration"
      baseline: "qwen3-32b"            # 当前使用的模型
      candidate: "deepseek-v31"        # 待验证的模型
      trafficPercent: 10               # 10% 流量走 shadow
      verificationWindow: "24h"        # 验证周期
      improvementThreshold: 0.15       # 综合改善 > 15% 才 promote
      metrics: ["latency", "cost", "accuracy"]
      autoPromote: false               # 手动确认或自动 promote
```

**关键决策:**
- Shadow 请求的结果记录到 replay store，但不返回给用户
- 用户感知的是 baseline 模型的响应
- 比较在异步完成，不阻塞主请求
- `trafficPercent` 需要一致性哈希（同一 user/session 稳定分流）

**v2 展望:**
- "Foreground SPM" — 同时发两个请求，先返回的给用户，晚返回的用于比较
- 自动 promote/reject 基于统计显著性（t-test 而非简单阈值）

---

## Idea 4: CEP-Based Anomaly Detection — 复合事件模式检测

**灵感来源:**
- **Flink CEP / Esper** — 复杂事件处理引擎，"A followed by B within 5s, without C in between"
- **DLACEP (EDBT 2024)** — Deep Learning accelerated CEP，小模型识别退化前兆
- **Session Windows** — 自动将相关异常分组为 "incident"

**已有基础设施:**
- `WindowedMetricsManager` 有完整的环形缓冲区 + 多时间窗口 + 百分位计算
- `ModelErrorRate`, `ModelLatencyP50/P95/P99` 已作为 Prometheus 指标发出
- 延迟缓存 (`TPOTCache`, `TTFTCache`) 有实时 per-model 数据

**缺失:** 没有任何异常检测消费这些指标。没有 incident 自动聚合。

**设计概要:**

```
CEP 模式定义:

pattern: "degradation_spiral"
  - ModelLatencyP95(model=X) > threshold FOR 2 consecutive 1-minute windows
  - AND ModelErrorRate(model=X) > 0.01
  - WITHIN 5 minutes
  → ACTION: setModelHealth(X, DEGRADED) + alert

pattern: "flash_crash"
  - ModelErrorRate(model=X) > 0.5 FOR 1 window
  - OR 5 consecutive timeouts
  - WITHIN 30 seconds
  → ACTION: setModelHealth(X, OPEN) + immediate circuit breaker trip

pattern: "slow_drift"
  - ModelLatencyP50(model=X) INCREASING by > 20% OVER 1 hour
  - AND ModelRequestsWindowed(model=X) STABLE (±5%)
  → ACTION: create incident + notify oncall

pattern: "cost_runaway"
  - SessionTurnCost(session=S) > budget × 2
  - WITHIN 1 request
  → ACTION: auto-demote session to lower-cost models
```

**运营商 DSL:**

```yaml
routing:
  anomalyPolicies:
    - name: "degradation_spiral"
      pattern:
        - metric: ModelLatencyP95
          condition: "> 2 * baseline_p95"
          forWindows: 2
          window: "1m"
        - metric: ModelErrorRate
          condition: "> 0.01"
          forWindows: 1
          window: "1m"
      within: "5m"
      action:
        type: "circuit_breaker_open"
        reason: "degradation spiral detected"
```

**Session Window Incident Grouping:**
- 相关异常事件自动分组为一个 incident（session gap = 2 分钟无新异常）
- Incident 关闭时自动生成摘要（受影响模型、持续时间、峰值指标）
- 为未来的 DLACEP 留数据接口（历史 incident 用于训练前兆检测模型）

---

## Idea 5: Resource Governor — 请求分级与资源隔离

**灵感来源:**
- **Oracle Resource Governor** — classifier function → consumer group → resource plan (CPU/Memory/IO caps)，自动 consumer group switching
- **Oracle DBRM Scheduler** — 按时间窗口切换 resource plan
- **DB Workload Management** (Teradata, Redshift) — 请求队列、优先级、并发控制

**已有基础设施:**
- `RateLimitResolver` 有 user/group/model 级别的 RPM/TPM 限制
- `RateLimitContext` 有 `UserID`, `Groups`, `Model`, `TokenCount`
- `LatencyAwareSelector` 有 queue depth 感知基础

**缺失:** 没有请求分类、优先级队列、"降级"（demotion）、按时间窗口切换路由策略的能力。

**设计概要:**

```
请求分类 pipeline:

Request → Classify (user_tier, category, estimated_complexity, priority)
        → Workload Group (production / batch / free_tier / internal)
        → Resource Plan (model_pool, concurrency_limit, timeout)
        → Model Selection (within allowed pool)

Automatic Demotion:
  如果 session 累计 cost > budget → 降级到 cheaper pool
  如果单次请求 latency > deadline → cancel + fallback 到更快模型
```

**配置:**

```yaml
routing:
  resourcePlans:
    - name: "business_hours"
      schedule: "0 9 * * 1-5"         # cron, 工作日 9am
      duration: "9h"
      groups:
        - name: "production"
          match:
            userTiers: ["enterprise", "pro"]
          modelPool: ["deepseek-v31", "gpt-4o"]  # premium models
          concurrencyLimit: 100
          requestTimeout: "30s"

        - name: "free_tier"
          match:
            userTiers: ["free"]
          modelPool: ["qwen3-8b", "gpt-4o-mini"]  # budget models
          concurrencyLimit: 20
          requestTimeout: "60s"

    - name: "night_batch"
      schedule: "0 18 * * 1-5"
      duration: "15h"
      groups:
        - name: "all"
          modelPool: ["qwen3-8b", "deepseek-v31"]  # 夜间放宽
          concurrencyLimit: 500

  demotionPolicies:
    - condition: "session_cost > monthly_budget * 0.8"
      action: "switch_group"
      targetGroup: "throttled"
    - condition: "request_latency > deadline"
      action: "cancel_and_fallback"
```

**关键设计点:**
- Classifier function 在信号评估前运行（类似 binding 的早期介入点）
- Resource plan 切换动作写 replay store（审计追踪）
- 与 rate limiter 互补：rate limiter 负责 "allow/deny"，resource governor 负责 "route to which tier"

---

## Idea 6: Dynamic Routing Table — 流式配置热更新

**灵感来源:**
- **Flink Dynamic Table** — 同一实体既可视为快照（table）也可视为 changelog（stream）
- **Stream-Table Duality** — INSERT/UPDATE/DELETE 语义的配置变更流
- **Kappa Architecture** — 一切皆流，无批/流分离

**已有基础设施:**
- Plan Binding 已支持通过 YAML 配置注册 binding（持久化）
- 配置热加载机制（config file watch）
- `CanonicalRoutingLoader` 已能解析 routing 配置

**缺失:** 配置变更需要 YAML + CI/CD 部署。无 API 驱动的动态 binding 注册/撤销。

**设计概要:**

```
REST API for Dynamic Binding Management:

POST   /admin/v1/bindings              # 创建 binding（即时生效）
DELETE /admin/v1/bindings/{name}        # 删除 binding
GET    /admin/v1/bindings              # 列出所有 binding
GET    /admin/v1/bindings/{name}/stats # binding 命中统计
PATCH  /admin/v1/bindings/{name}       # 部分更新（enable/disable, 改 priority）

POST   /admin/v1/bindings/{name}/expire # 立即过期

Config changelog stream:
  - 所有变更追加到 replay store 的 binding_change_log
  - 重启时回放 changelog，重建当前 effective binding set
  - 支持 "config as code" 和 "API override" 双模式
```

**关键设计点:**
- 配置的双向融合：YAML 声明基础配置 + API 动态覆盖（API 变更优先级高于 YAML）
- 重启持久化：API 创建的 binding 写入 changelog，重启后回放
- Config changelog 作为 replay store 的一个 stream，支持时间旅行查询
- `x-vsr-config-version` response header 让客户端知道当前生效的配置版本

**v2 展望:**
- Conditional binding (基于时间/负载/事件的自动 apply/expire)
- 配置的 git-ops 集成（API 变更 → 自动 PR 回 YAML）

---

## Idea 7: Incremental Materialized View — 路由统计增量刷新

**灵感来源:**
- **Oracle MV FAST REFRESH** — 仅应用 delta（MV log），无需全量重算
- **RisingWave** — 流式物化视图，维护持续 streaming execution DAG
- **PostgreSQL REFRESH MATERIALIZED VIEW CONCURRENTLY**

**已有基础设施:**
- `WindowedMetricsManager` 的环形缓冲区在每次 `UpdateInterval` (10s) 重算
- `TPOTCache` / `TTFTCache` 用 EMA (α=0.3) 做增量更新
- 多个 selector 有自己的 stats 结构

**缺失:** 窗口指标的更新是全量重算而非增量。复杂指标（embedding drift、聚类凝聚力）没有物化视图概念。

**设计概要:**

```
指标分类:

FAST REFRESH 指标 (O(1) 增量更新):
  - request_count: new_count = old_count + 1
  - avg_latency:  new_avg = old_avg + (value - old_avg) / n
  - error_rate:   rolling counter
  - token_usage:  cumulative sum
  → 每次请求后立即更新，无需等 10s tick

COMPLETE REFRESH 指标 (周期性批量重算):
  - P50/P95/P99 latency (需要排序)
  - embedding drift (需要全量 embedding 距离计算)
  - cluster cohesion (需要聚类算法)
  → 在 background goroutine 中按 interval 重算
```

**关键设计点:**
- 每个指标声明自己的 refresh capability（`FAST` 或 `COMPLETE`）
- FAST 指标在请求完成回调中立即更新，确保实时性
- COMPLETE 指标复用现有 `WindowedMetricsManager` 的定时重算
- 消费者（selector、circuit breaker、CEP）不关心指标是 FAST 还是 COMPLETE refresh

---

## Idea 8: Routing Time Travel & Audit Trail（WAL 风格）

**灵感来源:**
- **DB Write-Ahead Log (WAL)** — 所有变更先写日志再执行，支持 crash recovery + point-in-time recovery
- **Oracle Flashback Query** — `SELECT ... AS OF TIMESTAMP` 查询历史状态
- **DB Audit Trail** — 完整的操作审计链

**已有基础设施:**
- Replay store 已记录每次路由决策
- `SessionID` / `TurnIndex` 已存在
- Streaming changelog (Idea 6) 可复用

**缺失:** 没有 "为什么在这个时间点做了这个路由决策" 的可查询历史。没有 point-in-time recovery 能力。

**设计概要:**

```
Routing Decision WAL Entry:
{
  "timestamp": "2026-04-28T14:23:01.123Z",
  "request_id": "req_abc123",
  "session_id": "sess_xyz",
  "category": "code_generation",
  "selected_model": "deepseek-v31",
  "candidates": ["deepseek-v31", "qwen3-32b", "gpt-4o-mini"],
  "candidate_scores": {"deepseek-v31": 0.87, "qwen3-32b": 0.72, "gpt-4o-mini": 0.45},
  "active_bindings": ["code_to_deepseek"],
  "active_circuit_breakers": [],
  "cost": {"estimated": 0.0023, "actual": 0.0025},
  "signals": {"complexity": "medium", "domain": "code"},
  "decision_reason": "cost_based_optimizer min_cost(0.23)"
}
```

**Time Travel 查询:**

```sql
-- 为什么昨天 14:00 的请求路由到了 qwen 而非 deepseek？
SELECT * FROM routing_wal
WHERE timestamp BETWEEN '2026-04-27 13:55' AND '2026-04-27 14:05'
  AND selected_model = 'qwen3-32b'
  AND category = 'code_generation'
```

**关键设计点:**
- WAL 写入不阻塞路由决策（异步写入，fire-and-forget）
- Schema 设计需向前兼容（JSON 格式，新增字段用 omitempty）
- 支持 point-in-time replay：给定时间点，重建当时的有效配置状态
- 与 Idea 6 的 config changelog 结合：WAL(路由决策) + Changelog(配置变更) = 完整的审计能力

---

## Cross-Cutting: 模块依赖关系

```
                    ┌──────────────────────────┐
                    │  WAL / Audit Trail (Idea 8)│ ← 记录一切
                    └──────────┬───────────────┘
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    ▼                          ▼                          ▼
┌──────────────┐   ┌──────────────────┐   ┌──────────────────────┐
│ CEP Anomaly  │   │ Circuit Breaker  │   │ Shadow Routing +     │
│ Detection    │──→│ (Idea 1)         │   │ A/B Verification     │
│ (Idea 4)     │   │                  │   │ (Idea 3)             │
└──────────────┘   └────────┬─────────┘   └──────────┬───────────┘
                            │                        │
                            ▼                        ▼
               ┌──────────────────────────────────────────┐
               │        Model Selection Layer              │
               │  ┌──────────────────────────────────┐    │
               │  │ Cost-Based Optimizer (Idea 2)     │    │
               │  │ Resource Governor (Idea 5)        │    │
               │  │ Incremental MV Stats (Idea 7)     │    │
               │  └──────────────────────────────────┘    │
               └──────────────────────────────────────────┘
                            │
                            ▼
               ┌──────────────────────────────────────────┐
               │  Dynamic Routing Table (Idea 6)           │
               │  + Plan Binding (existing proposal)       │
               │  + Hint Injection (existing proposal)     │
               └──────────────────────────────────────────┘
```

---

## 优先级建议

| 优先级 | Idea | 理由 |
|--------|------|------|
| **P0** | Circuit Breaker (1) | 生产必备。凌晨3点模型故障必须自动处理，不可依赖人工。代码库零基础设施，但从 WindowedMetricsManager 开始构建成本可控 |
| **P0** | Cost-Based Optimizer (2) | 直接提升核心路由决策质量。已有 70% 基础设施（pricing + latency + CostWeight），主要是整合工作 |
| **P1** | CEP Anomaly Detection (4) | Circuit Breaker 的 "智能版"。单独阈值不够，需要复合模式检测。与 Idea 1 有天然协同 |
| **P1** | Shadow Routing (3) | 安全迁移的必备能力。基础设施从零开始，但 SPM 模式经 25+ 年验证，方向正确 |
| **P2** | Resource Governor (5) | 多租户场景刚需。与已有 RateLimitResolver 互补 |
| **P2** | Dynamic Routing Table (6) | 运维体验质的提升。但需要 API 鉴权 + 审计 + 回滚，工程量大 |
| **P3** | WAL / Time Travel (8) | 审计和调试的 nice-to-have，但对路由核心价值间接 |
| **P3** | Incremental MV Stats (7) | 性能优化，不影响功能面。可在系统负载瓶颈出现时再做 |

---

## 与已有提案的关系

已有提案 `routing-hint-and-plan-binding.md` 的核心洞察是：**优化器再强，也需要人工干预的逃生舱。** 本文档的 ideas 在另一个维度上发力：**让优化器本身更强。**

| 维度 | Hint + Binding | 本文 Ideas |
|------|---------------|-----------|
| 核心问题 | 路由可控性（逃生舱） | 路由智能性（自动驾驶） |
| 手段 | 声明式覆盖（人工指定） | 自动适应（数据驱动） |
| 时效性 | 持久规则 + 临时覆盖 | 实时感知 + 自动响应 |
| 类比 | 数据库 Hint + SPM Binding | 数据库 CBO + 自动调优 |

两者互补：Hint/Binding 是方向盘（人控），本文 ideas 是更好的引擎（自动驾驶）。完整的 Routing Intelligence Layer = Hint/Binding + CBO + Circuit Breaker + Shadow Routing + CEP。

---

## Open Questions

1. **Circuit Breaker 和 CEP 是否合并为一个 "Health & Anomaly Engine"？** 断路器关注单个模型的 binary 状态，CEP 关注复合模式 + 趋势。合并可减少概念数量，但实现复杂度更高。
2. **Shadow Routing 的结果是否返回给用户？** 如果 shadow 模型更好，用户却收到 baseline 的较差结果，这在体验上有问题。是否可选 "return best of both"？
3. **CBO 的代价权重由谁来设定？** 运维人员很难合理设定 `w₁:w₂:w₃:w₄`。是否从用户隐式反馈中学习这些权重？
4. **Dynamic Routing Table 的 API 变更是否需要 approval workflow？** 如果提供 API 即时生效，是否需要内置审批流程（类似 Oracle SPM 的 plan verification）？
5. **Idea 1-4 是否应该合并为一个统一的 "Adaptive Routing Engine"？** 它们都消费相同的窗口指标、影响相同的模型选择流程。单独实现可能造成逻辑碎片化。
