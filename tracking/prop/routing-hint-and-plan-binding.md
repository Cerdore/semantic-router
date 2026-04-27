# Routing Hint Injection & Plan Binding

**Status:** Proposal  
**Author:** @chenxiansen.cxs  
**Date:** 2026-04-27  

## 1. Motivation

### 1.1 问题

vLLM Semantic Router 目前的决策链路完全是全自动的：

```
Request → Signal Evaluation → Decision Engine → Model Selection → Model
```

用户或运维人员**没有任何声明式的手段**来影响路由决策。整个系统像一个没有逃生舱的自动驾驶——它在大多数情况下表现得很好，但在一些关键场景下，你无法说"这次请按我说的做"。

### 1.2 缓存不能解决这个问题

Semantic Cache 解决的是**性能问题**（相同 query 不重复计算），而非**正确性/可控性问题**：

| 维度 | Semantic Cache | Hint & Binding |
|------|---------------|----------------|
| 目标 | 减少延迟和计算成本 | 确保路由决策的正确性 |
| 机制 | "这个 query 我见过"→复用结果 | "这类 prompt 必须走模型 X" |
| 时效性 | 随缓存过期而失效 | 持久或按需生效 |
| 模型故障 | 无法处理（缓存旧结果不可用） | 立即绕行到备用模型 |

路由决策和缓存是两个正交的维度。而且，错误的缓存结果会固化错误的路由决策。

### 1.3 真实场景

**场景 A：渐进式模型迁移**  
团队正在把代码类流量从 `qwen3-32b` 迁移到 `deepseek-v31`，希望先让 10% 流量走新模型观察效果。目前没有机制支持这种灰度。

**场景 B：成本护栏**  
CFO 要求闲聊类请求不得使用高成本模型。尽管信号系统能识别闲聊，但 Model Selection 可能在多个候选间轮转，偶尔仍选中贵模型。需要声明式地锁死。

**场景 C：外部系统集成**  
CI/CD 的 E2E 测试需要验证 Gemini 兼容性。测试代码只想加一个 HTTP header，不想为临时需求改 router 配置、重启服务。

**场景 D：紧急降级**  
凌晨 3 点，某个模型大量超时。SRE 不想改 YAML + 走 CI/CD 部署。希望打一个临时规则，五分钟内将所有数学类请求切到备用模型。

### 1.4 数据库领域的先例

数据库领域用 25+ 年证明了这类机制的必要性：

| 概念 | 数据库 | LLM Router |
|------|--------|------------|
| Hint 注入 | Oracle `/*+ INDEX(t idx) */`, pg_hint_plan | `x-vsr-route-hint: model=X` |
| Plan Binding | TiDB `CREATE BINDING`, Oracle SPM | `planBindings` YAML 配置 |
| 基线管理 | Oracle SQL Plan Baseline | Binding 历史 + 回滚 |

核心洞察：**优化器再强，也需要人工干预的逃生舱。**（因为优化器不掌握业务语义，且紧急场景等不起）

---

## 2. Design

### 2.1 核心概念

Hint（临时、请求级）+ Plan Binding（持久、注册级），覆盖不同时效性需求：

```
┌──────────────────────────────────────────────────────────┐
│                    Request Arrives                        │
│                          │                                │
│          ┌───────────────▼───────────────┐               │
│          │  1. Plan Binding 匹配？        │  ← 运维注册   │
│          │     (registry-level, 持久)     │               │
│          │     支持过期时间，到期自动失效  │               │
│          └───────────────┬───────────────┘               │
│                          │ 未匹配                         │
│          ┌───────────────▼───────────────┐               │
│          │  2. HTTP Header 有 hint？     │  ← 客户端注入  │
│          │     (request-level, 临时)      │               │
│          └───────────────┬───────────────┘               │
│                          │ 未匹配                         │
│          ┌───────────────▼───────────────┐               │
│          │  3. x-vsr-enable-route-hints  │               │
│          │     且 prompt 有 /*+ VSR: */？│  ← 用户注解    │
│          │     (需 header 显式开启)       │               │
│          └───────────────┬───────────────┘               │
│                          │ 无 hint                        │
│          ┌───────────────▼───────────────┐               │
│          │  正常 Signal→Decision 流程     │               │
│          └───────────────────────────────┘               │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Hint 类型

| 类型 | 语法 | 行为 |
|------|------|------|
| **强制模型** | `model=X` | 跳过信号评估+决策引擎，直接路由到 X |
| **强制决策** | `decision=X` | 跳过信号评估，使用决策 X 的 ModelRefs |
| **软偏好** | `prefer_model=X` | 保持正常流程，偏置 Model Selection |
| **模型排除** | `exclude_models=A,B` | 从候选集中移除指定模型 |
| **信号注入** | `signal=type/name` | 注入虚拟信号（安全信号不可注入） |

### 2.3 Hint 类型与架构层级的精确映射

vLLM Semantic Router 的核心链路是一条四层管道：

```
HTTP Request
  │
  ▼
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐     ┌──────────┐
│   Signal      │ ──→ │   Decision       │ ──→ │   Model           │ ──→ │  Route   │
│   Evaluation  │     │   Engine         │     │   Selection       │     │  Output  │
│   (17 种分类器)│     │   (Rule Tree 匹配)│     │   (Registry 选择) │     │  (模型名) │
└──────────────┘     └──────────────────┘     └───────────────────┘     └──────────┘
```

五种 hint 类型恰好在这四层管道的不同深度"切入"：

```
                      signal=type/name
                           │
                           ▼
          ┌────────────────────────────┐
          │   Stage 2: Signal Evaluation│  ← 注入虚拟信号，继续走后续流程
          └──────────┬─────────────────┘
                     │
                decision=X
                     │
                     ▼
          ┌────────────────────────────┐
          │   Stage 3: Decision Engine │  ← 跳过信号评估，直接指定 decision
          └──────────┬─────────────────┘
                     │
          exclude_models / allow_models
                     │
                     ▼
          ┌────────────────────────────┐
          │   Stage 4b: Model Selection│  ← 过滤候选模型列表
          └──────────┬─────────────────┘
                     │
                   model=X
                     │
                     ▼
          ┌────────────────────────────┐
          │   Stage 4: Route Output    │  ← 跳过全部，直接指定最终模型
          └────────────────────────────┘
```

| Hint 类型 | 干预深度 | 跳过 | 保留 | 数据依赖 |
|-----------|:--------:|------|------|---------|
| `signal=type/name` | **最浅** | 无 | Signal → Decision → Selection → Route | 注入到 `SignalResults.MatchedXXXRules` |
| `exclude_models` | 浅 | 无 | Signal → Decision → Selection(filtered) | 过滤 `SelectionContext.CandidateModels` |
| `allow_models` | 浅 | 无 | Signal → Decision → Selection(filtered) | 同上 |
| `decision=X` | **中** | Signal | Decision(指定) → Selection → Route | 查 `config.Decisions[name]`，构造 `DecisionResult` |
| `model=X` | **最深** | Signal+Decision+Selection | Route(直接指定) | 查包含 X 的 decision + authz 验证 |

两种维度正交：**Hint 类型**回答"做什么"（干预哪一层），**注入来源**回答"谁说的"（Binding > Header > Prompt 优先级）。

### 2.4 Plan Binding

```yaml
routing:
  planBindings:
    - name: "code_to_deepseek"
      description: "所有代码相关请求路由到 DeepSeek"
      priority: 100
      enabled: true
      expiresAt: ""                    # 空 = 永不过期；支持 ISO 8601，如 "2026-05-15T00:00:00Z"
      match:
        keywords: ["code", "debug", "refactor"]
        categories: ["computer science"]
        decisionPatterns: ["code_*"]
        matchOperator: "AND"
      action:
        forceModel: "deepseek-v31"
```

### 2.5 优先级

```
Plan Binding（运维意图，最高）
  → HTTP Header（客户端意图）
    → Prompt 注解（用户意图，最低）
```

同一来源内：`强制模型 > 强制决策 > 信号注入 > 软偏好 > 模型排除`

### 2.6 安全约束（不可绕过）

- **Authz 永不被跳过**：强制模型必须在用户授权集合内
- **信号注入白名单**：`authz`/`jailbreak`/`pii`/`kb` 等安全信号不可注入
- **Prompt 注解需 header 显式开启**：只有携带 `x-vsr-enable-route-hints: true` 的请求才扫描 prompt 中的 `/*+ VSR: ... */` 注解，否则视为普通文本原样转发。HTTP Header hint 不受此限制
- **Prompt 注解仅扫描前 4KB**，转发前保证清除，清除后二次扫描验证无残留
- **全局开关**：`hints.enabled: false` 可全局关闭所有 hint/binding 处理
- **请求级全局关闭**：`x-vsr-disable-route-hints: true` 可逐请求关闭所有 hint 处理（包括 header、prompt、binding）

---

## 3. Implementation Overview

### 3.1 涉及文件

| Layer | File | Change |
|-------|------|--------|
| Config | `pkg/config/selection_config.go` | 新增 `PlanBinding`, `PlanBindingMatch`, `PlanBindingAction`, `RoutingHintConfig` |
| Config | `pkg/config/canonical_config.go` | `CanonicalRouting` 新增 `PlanBindings` |
| Config | `pkg/config/canonical_global.go` | 新增 hints 全局配置 |
| Config | `pkg/config/canonical_routing_loader.go` | 解析 `planBindings` |
| Config | `pkg/config/canonical_export.go` | 导出 `planBindings` |
| Config | `pkg/config/validator.go` | 新增 `validatePlanBindings()` |
| Decision | `pkg/decision/hint_parser.go`（新） | 解析 header 和 prompt annotation |
| Decision | `pkg/decision/binding_engine.go`（新） | Binding 匹配引擎 |
| Decision | `pkg/decision/engine.go` | 增加 BindingEngine 集成（options 模式） |
| ExtProc | `pkg/extproc/req_filter_classification.go` | **主集成点**：`performDecisionEvaluation()` |
| ExtProc | `pkg/extproc/req_filter_cache.go` | hint 请求绕过缓存 |
| ExtProc | `pkg/extproc/req_filter_looper.go` | 排除 looper 内部请求 |
| Model | `pkg/modelselection/selector.go` | `SelectionContext` 增加 hint 字段 |
| Headers | `pkg/headers/headers.go` | 新增 request/response headers |
| Replay | `pkg/routerreplay/store/store.go` | Replay 记录包含 hint/binding 信息 |

### 3.2 关键设计决策

**1. 集成点在 extproc 层，不在 services 层**  
主流程走 `performDecisionEvaluation()`，而非 `ClassifyIntent()`，因为前者是 Envoy ext_proc 的实际调用路径，包含了缓存、looper、replay 的完整上下文。

**2. Binding 评估在信号评估之前**  
Binding 使用轻量的 `BindingMatchInput`（关键词+分类+authz role），而非 `SignalMatches`，避免鸡生蛋蛋生鸡问题。

**3. 不包含 embedding_rules 作为 binding 匹配条件**  
计算 embedding 来匹配 binding，再跑信号评估，会导致双倍 embedding 开销。

---

## 4. Competitive Landscape

### 4.1 调研范围

对当前 LLM routing 领域的知名项目进行了系统性调研，覆盖学术界开源项目和商业/闭源方案。

### 4.2 学术界 / 开源项目

| 项目 | 来源 | 核心机制 | 路由干预能力 |
|------|------|----------|-------------|
| **LLMRouter** | UIUC 2025 | 16+ 路由策略（最短队列、KV cache-aware、cost-based 等），可插拔 Route × Training 架构 | 无。策略全自动，无用户声明式干预 |
| **SmarterRouter** | 社区 | VRAM 感知的本地模型路由，面向 homelab 场景 | 无。关注显存效率而非路由可控性 |
| **Arch-Router** | Arch-Function 2025 | 1.2B 参数自回归 router，preference-aligned 路由 | 无。Router 本身是 LLM，引入额外延迟和成本 |
| **OPEA Router** | Intel / Linux Foundation | RouteLLM + Semantic Router 组合方案 | 无。组合已有方案，无 hint/binding 概念 |
| **TRACER** | 学术 | Surrogate model 路由 + parity gate 质量保障 | 无。聚焦路由保真度，无人工干预逃生舱 |

### 4.3 商业 / 闭源

| 项目 | 核心卖点 | 路由干预能力 |
|------|----------|-------------|
| **Morph Router** | Cost-aware difficulty classification，按难度将请求分发给不同模型 | 全自动，用户仅设预算上限，无逐请求控制 |
| **LiteLLM Proxy** | Semantic context router，通过 embeddings 匹配路由 | 规则式路由（"含 X 关键词走 Y"），最接近但无 hint 概念 |
| **NadirClaw** | 14 维评分 + sigmoid 校准的 scorer 路由 | 评分全自动，无干预接口 |
| **ClawRouter** | Agent-native，钱包认证路由 | 无路由干预机制 |

### 4.4 关键发现

**2025→2026 的共同趋势**：所有项目都在向多信号、可组合、决策驱动的架构演进，与我们现有的 signal→decision 管道方向一致。

**共同盲区**：无一提供"逃生舱"。当全自动路由决策出错时——无论是因为信号误判、模型选择偏差，还是紧急降级场景——用户没有任何声明式手段来纠正路由行为。

**我们的差异化**：Hint Injection（临时、请求级）+ Plan Binding（持久、注册级）+ 安全约束（Authz 不可绕过）三位一体。数据库领域用 25+ 年证明了优化器 + 人工 hint 的组合是必需品，而 LLM routing 行业目前仍处于"优化器刚诞生"的阶段——这正是本提案的独特价值和 timing。

---

## 5. Alternatives Considered

### 5.1 仅用 HTTP Header，不做 Prompt 注解

**放弃原因**：prompt 注解对非代理场景（SDK 直连）更友好，用户不需要记 header 名。OEM 场景下用户甚至不暴露 header 接口。

### 5.2 Binding 匹配也走 Decision Engine 的 RuleNode 树

**放弃原因**：Binding 的语义是"无条件覆盖"，而 RuleNode 树是用来做条件匹配的。混在一起会模糊优先级。且 Binding 需要在信号评估**之前**执行，而 RuleNode 依赖信号结果。

### 5.3 信号注入不做白名单，全放开

**放弃原因**：安全不可妥协。攻击者不能通过注入 `signal:authz=admin` 来提权。

---

## 6. Resolved Decisions

| # | 问题 | 决策 |
|---|------|------|
| 1 | Hint 和 Binding 是否一起做？ | **一起做**。Hint 覆盖临时需求，Binding 覆盖持久化规则，互补而非替代 |
| 2 | Binding 是否需要过期时间？ | **需要**，`expiresAt` 字段（ISO 8601 格式），空值 = 永不过期 |
| 3 | Prompt 注解是否需 header 前置开关？ | **需要**。设置 `x-vsr-enable-route-hints: true` 的请求才扫描 `/*+ VSR: */`，否则当普通文本转发 |

## 7. Open Questions

1. **是否需要独立的 Hint 模板系统？**（`template=low_cost` → 预定义的一组模型排除规则）留到 v2
2. **Binding 过期后是自动禁用还是自动删除？** **决定：自动禁用**（`enabled` → `false`），保留记录便于审计、复用和回滚
3. **是否需要 Binding 命中统计/监控 dashboard？** 对运维排查有帮助，建议作为 observability 的一部分

---

## 8. References

- Oracle Optimizer Hints: `/*+ FULL(e) INDEX(d idx) */`
- PostgreSQL pg_hint_plan: hint table + comment-based injection
- TiDB SQL Plan Management: `CREATE BINDING ... USING ...` with `sql_digest` matching
- NVIDIA LLM Router: `routing_strategy: "manual"` with `model` field override
- vLLM Semantic Router existing arch: `pkg/decision/engine.go`, `pkg/extproc/req_filter_classification.go`
- **LLMRouter** (UIUC 2025): 16+ pluggable routing strategies, Route × Training architecture — no user override mechanism
- **Arch-Router** (Arch-Function 2025): 1.2B autoregressive router, preference-aligned — router as LLM, extra latency/cost
- **OPEA Router** (Intel / LF): RouteLLM + Semantic Router combo — no hint/binding concept
- **TRACER**: Surrogate model routing + parity gate — fidelity-focused, no manual intervention
- **SmarterRouter**: VRAM-aware local model routing — homelab-oriented, no routing control
- **Morph Router**: Cost-aware difficulty classification (closed-source) — auto only, user sets budget ceiling
- **LiteLLM Proxy** (BerriAI): Semantic context router via embeddings — rule-based routing, closest analogue but no hint injection
- **Triton Inference Server** (NVIDIA): Model serving / inference engine, not a semantic router — complementary infrastructure layer
- **Ray Serve** (Anyscale): Distributed model orchestration — complementary, not competitive
- **Seldon Core**: K8s-native MLOps platform — complementary, not competitive
