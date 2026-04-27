# Evaluation: Routing Hint Injection & Plan Binding

**Evaluated by:** Claude Opus 4.7
**Date:** 2026-04-27
**Source proposal:** `docs/proposals/routing-hint-and-plan-binding.md`

---

## Executive Summary

The proposal is **architecturally sound and feasible**, with a well-chosen integration point (`performDecisionEvaluation()`). The database-inspired hint+binding paradigm is the right abstraction. However, the current five hint types have **three critical gaps** for the 1.3 scenarios, and several underspecified integration details need resolution before implementation.

**Overall verdict:** Pass with revisions. Address the gaps below, then proceed.

---

## 1. Hint Type × Scenario Coverage Matrix

### 1.1 Mapping Current Hint Types to 1.3 Scenarios

| Scenario | `model=X` | `decision=X` | `prefer_model=X` | `exclude_models` | `signal=type/name` |
|----------|:---------:|:------------:|:----------------:|:----------------:|:------------------:|
| **A. 渐进式迁移** (10%→new model) | Weak (100% only) | Weak | Partial (no %) | No | No |
| **B. 成本护栏** (no expensive for chat) | Yes | Yes | No | **Yes (best fit)** | No |
| **C. CI/CD 集成** (header→Gemini) | **Yes (best fit)** | Yes | Overkill | Overkill | Overkill |
| **D. 紧急降级** (math→backup) | **Yes (best fit)** | Yes | Weak | Partial | No |

### 1.2 Critical Gap: Scenario A (Progressive Migration)

This is the most common production scenario, and **none of the five hint types support it well**:

- `model=X` gives 100% to one model, cannot express "10%"
- `prefer_model=X` biases selection but is non-deterministic and cannot express a percentage
- There is no `traffic_percent` or weighted split primitive

**Recommendation:** Add a `route_weight` hint type or extend `prefer_model` with an optional weight:

```
/*+ VSR: route_weights=deepseek-v31:10,qwen3-32b:90 */
```

Or as a binding:

```yaml
action:
  routeWeights:
    deepseek-v31: 10
    qwen3-32b: 90
```

This requires a new "weighted random" selection method or extending the static selector.

### 1.3 Second Gap: `prefer_model` is Underspecified

The proposal says "保持正常流程，偏置 Model Selection" but never defines the biasing mechanism. This is not a documentation issue—it's a **design gap**. Each selection algorithm would need a different biasing approach:

| Algorithm | How to bias toward `prefer_model=X` |
|-----------|-------------------------------------|
| Elo | Boost X's rating by N points before comparison |
| RouterDC | Boost X's similarity score by factor |
| AutoMix | Lower X's verification threshold |
| Static | Boost X's static score |
