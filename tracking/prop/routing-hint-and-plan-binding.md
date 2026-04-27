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
