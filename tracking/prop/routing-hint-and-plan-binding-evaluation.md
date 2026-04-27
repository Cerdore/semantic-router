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
| LatencyAware | Skip latency penalty for X |
| KNN/KMeans/SVM | Hard to bias without retraining |

This makes `prefer_model` surprisingly expensive to implement correctly. Two options:

**Option A (Recommend):** Defer `prefer_model` to v2. It's the only hint type without a clear, universal implementation path.

**Option B:** Implement as a post-selection override — run normal selection, then if X is in candidates and the confidence gap between the winner and X is below a threshold, swap to X. This is simple but changes semantics (it's more "tiebreak" than "prefer").

### 1.4 Third Gap: No `allow_models` (Whitelist)

`exclude_models` removes specific models, but there's no way to say "only these two models are acceptable." You could use `decision=X` for this, but that requires a pre-configured decision. For ad-hoc scenarios (SRE troubleshooting at 3am), creating a decision config is too heavyweight.

**Recommendation:** Add `allow_models=A,B` as the dual of `exclude_models`.

### 1.5 Beyond 1.3: Hint Types for Future Scenarios

| Future Scenario | Needed Hint Type | Priority |
|-----------------|-----------------|----------|
| "Use cheapest model that passes quality bar" | `max_cost=X` or `min_quality=X` | P2 |
| "Run with this selection strategy" | `strategy=elo` | P2 |
| "This is a test, don't record in replay" | `replay=skip` | P3 |
| "Use this LoRA adapter" | `lora=X` | P3 |

---

## 2. Integration Point Feasibility

### 2.1 Proposed Flow (Correct)

The proposal places binding/hint evaluation **before** signal evaluation in `performDecisionEvaluation()`:

```
performDecisionEvaluation()
  ├── [NEW] 1. Plan Binding match? → force model/decision, skip to finalize
  ├── [NEW] 2. HTTP Header hint? → force model/decision, skip to finalize
  ├── [NEW] 3. Prompt annotation hint? → force model/decision, skip to finalize
  ├── 4. prepareSignalEvaluationInput()
  ├── 5. evaluateSignalsForDecision()
  ├── 6. runDecisionEngine()
  └── 7. finalizeDecisionEvaluation() → selectDecisionRuntimeModel()
```

This is the **correct placement** because:
- It sits before the expensive signal evaluation (saves cost when hints are used)
- It has access to headers (`ctx.Headers`) and request body
- It's after cache check (hint requests bypass cache — see 2.3)
- It can still fall through to normal flow when no hints match

### 2.2 `decision=X` and the CategoryName Problem

When `decision=X` skips signal evaluation, `CategoryName` will be empty in `buildSelectionContext()`. This impacts:

1. **ML selectors** (KNN, KMeans, SVM, MLP): They use `CategoryName` for one-hot encoding in feature vectors. An empty category degrades their accuracy.

2. **Elo**: `DecisionName` is set (it's `X`), so per-decision Elo pools work. But category-weighted Elo (`CategoryWeighted: true`) won't function correctly.

3. **RouterDC**: Uses query embedding, not category, so largely unaffected.

**Mitigation:** When handling `decision=X`, extract the category from the decision's own rule tree. The decision `X` has `Rules RuleCombination` which references domain rules by name. Parse the rule tree to find the first `Type: "domain"` leaf and use its `Name` as the category. This is a static extraction (no runtime signal eval needed).

```go
func extractCategoryFromDecision(d *config.Decision) string {
    return findFirstDomainRule(d.Rules)
}
```

### 2.3 Cache Interaction

The proposal correctly states that hinted requests must bypass the cache. However, there are two distinct cases:

**Case 1: Hint changes the routing target** (`model=X`, `decision=X`, `exclude_models`)
→ Cache read AND write must be skipped. A cached response was computed under a different routing decision and is invalid.

**Case 2: Hint only biases selection** (`prefer_model=X`)
→ If we keep `prefer_model`, cache read could be skipped but write is debatable. The response is still valid for the same query without the preference hint.

**Recommendation:** In v1, skip cache entirely when any hint is active. Simpler, safer.

### 2.4 Looper Interaction

The proposal mentions excluding looper internal requests. This is essential — looper requests carry `x-vsr-looper-request: true` and must never be re-routed by hints. The `isLooperRequest()` check already exists in the codebase; hint evaluation must check this and return early.

### 2.5 Explicit Model Preservation

The current code has this logic in `finalizeDecisionEvaluation()`:

```go
if !r.Config.IsAutoModelName(originalModel) {
    return decisionName, evaluationConfidence, reasoningDecision, ""
}
```

When a user explicitly requests `model=gpt-4` in the OpenAI request, the router preserves it. This must interact with hints carefully:

- `x-vsr-route-hint: model=deepseek` should **override** the explicit `model=gpt-4` in the request body — that's the point of the hint
- But this creates a subtlety: what if both the request body says `model=gpt-4` AND the header says `model=deepseek`? The proposal's priority chain (binding > header > prompt) doesn't address the request body's model field.

**Recommendation:** Hint-selected models should bypass the `IsAutoModelName` check entirely. When a hint resolves to a model, treat it as authoritative.

---

## 3. Security Model Deep Dive

### 3.1 What the Proposal Gets Right

- Authz signal is never skippable (forced models must be in user's authorized set)
- Security signals (`authz`/`jailbreak`/`pii`/`kb`) are on the injection whitelist blocklist
- Prompt annotations require an explicit opt-in header (`x-vsr-enable-route-hints`)
- Prompt annotations are scanned only in the first 4KB
- Prompt annotations are stripped before forwarding
- Double-scan verification after stripping
- Global kill switch (`hints.enabled: false`)
- Per-request kill switch (`x-vsr-disable-route-hints: true`)

### 3.2 Gaps in the Security Model

**Gap 1: `exclude_models` can be used for DoS**

If an attacker sends `x-vsr-route-hint: exclude_models=model-a,model-b,model-c` excluding all available models, the router reaches model selection with zero candidates. The current `selectModelFromCandidates` returns `nil` for empty candidates, causing a panic or empty route.

**Fix:** Validate that after applying `exclude_models`, at least one candidate remains. If not, ignore the hint and log a security event.

**Gap 2: Signal injection whitelist is too narrow**

The proposal blocks `authz`/`jailbreak`/`pii`/`kb` from injection but allows all other signals. Consider:

- `signal:modality=DIFFUSION` — routes a text request to an image generation model, potentially causing errors or unexpected behavior
- `signal:complexity=easy` — routes a genuinely hard problem to a weak model, causing poor UX (not a security issue, but a reliability one)

**Recommendation:** Add `modality` to the blocked list (it controls which model type is used). Consider making the blocked list configurable rather than hardcoded.

**Gap 3: `decision=X` bypasses PII/jailbreak detection**

When `decision=X` skips signal evaluation, it also skips PII and jailbreak detection. The proposal's security constraints say "Authz 永不被跳过" but PII/jailbreak are also security-critical. If a request with PII uses `decision=code_decision`, the PII won't be detected.

**Recommendation:** When `decision=X` or `model=X` is used, still run the **security-only** signal subset (authz, jailbreak, PII) before routing. These are typically fast (regex-based or lightweight classifiers) and the security cost is worth it.

**Gap 4: Binding priority can be exploited via config**

If an attacker gains write access to the binding YAML config (e.g., via a compromised CI/CD pipeline), they can inject a high-priority binding that routes all traffic to a malicious model. This isn't a code vulnerability, but the proposal should mention that binding config changes should go through review/approval workflows.

### 3.3 Authz Integration Detail

The proposal says "强制模型必须在用户授权集合内." The authz system already has `CredentialResolver.KeyForProvider()`. The check should be:

```go
func (r *OpenAIRouter) isModelAuthorized(model string, ctx *RequestContext) bool {
    provider := r.getProviderForModel(model)
    key, err := r.CredentialResolver.KeyForProvider(provider, model, ctx.Headers)
    return err == nil && key != ""
}
```

This leverages the existing credential chain (header injection → static config) without duplicating authz logic.

---

## 4. Detailed Implementation Risks

### 4.1 Prompt Annotation Parsing Complexity

Scanning `/*+ VSR: ... */` from prompt text seems simple but has edge cases:

1. **Multi-line annotations**: `/*+ VSR:\n model=X\n*/` — should this be supported? Oracle hints are single-line.
2. **Multiple annotations**: `/*+ VSR: model=X */ ... /*+ VSR: exclude_models=Y */` — combine or last-wins?
3. **Nested in code blocks**: User sends code containing `/*+ VSR: model=X */` as an example — false positive.
4. **JSON escape**: The prompt is a JSON string, so `/*+ VSR: model=X */` might appear escaped.

**Recommendation:**
- Single-line only, match with regex: `/\*\+\s*VSR:\s*(.+?)\s*\*/`
- First match wins (ignore subsequent annotations)
- Document that code blocks containing VSR hints are false positives (acceptable, since the header `x-vsr-enable-route-hints` is the real gate)
- The hint parser must operate on the **parsed** JSON body content, not the raw HTTP body

### 4.2 Binding Matching is Inherently Limited

The proposal uses `BindingMatchInput` (keywords + categories + authz role) to avoid the chicken-and-egg problem. This means bindings **cannot** match on:

- Embedding similarity ("requests similar to these examples")
- Complexity ("hard problems only")
- Conversation shape ("long multi-turn conversations")
- Language ("non-English requests")

This is a fundamental design tradeoff, not a bug. The proposal acknowledges it (Section 3.2, Decision 3). For the 1.3 scenarios, this is acceptable:
- Scenario A (migration): Can use category matching (`categories: ["computer science"]`)
- Scenario B (cost): Can use category matching (`categories: ["chat"]`)
- Scenario D (emergency): Can use keywords (`keywords: ["math", "calculus"]`)

But it should be explicitly documented as a v1 limitation.

### 4.3 Replay Record Schema Expansion

The `Record` struct already has ~50 fields. Adding hint/binding info requires:

```go
// New fields in Record
RoutingHint       *RoutingHintInfo    `json:"routing_hint,omitempty"`
MatchedBinding    string              `json:"matched_binding,omitempty"`
HintSource        string              `json:"hint_source,omitempty"` // "header", "prompt", "binding"
```

This is straightforward but needs coordination with the replay store backends (memory/redis/postgres/milvus). The postgres backend would need a schema migration.

### 4.4 Config Validation Complexity

Adding `PlanBinding` to `CanonicalRouting` means new validation in `validateConfigStructure()`:

```go
validatePlanBindingContracts() {
    for each binding:
        - name is unique
        - priority is non-negative
        - expiresAt is valid ISO 8601 or empty
        - match.keywords, match.categories, or match.decisionPatterns is non-empty
        - action.forceModel references a declared model
        - action.forceDecision references a declared decision
}
```

This is well-understood but adds to an already large validation surface (~10 validators).

---

## 5. Architecture Diagram (Corrected)

```
Request Arrives
  │
  ├── hints.enabled == false? ──→ Skip all hint/binding processing
  │
  ├── x-vsr-disable-route-hints? ──→ Skip all hint/binding processing
  │
  ├── isLooperRequest()? ──→ Skip hint processing (internal request)
  │
  ▼
┌─────────────────────────────────────────────┐
│ 1. PLAN BINDING MATCH                        │
│    - Iterate bindings sorted by priority desc│
│    - Match: keywords AND categories AND      │
│      decisionPatterns (per matchOperator)   │
│    - Check expiresAt (skip if expired)       │
│    - First match wins → apply action         │
│    - action.forceModel → validate authz      │
│    - action.forceDecision → validate exists  │
└───────────┬─────────────────────────────────┘
            │ no match
            ▼
┌─────────────────────────────────────────────┐
│ 2. HTTP HEADER HINT (x-vsr-route-hint)      │
│    - Parse hint value                        │
│    - Validate syntax                         │
│    - model=X → validate authz                │
│    - decision=X → validate exists            │
│    - signal=X → validate not in blocked list │
└───────────┬─────────────────────────────────┘
            │ no header hint
            ▼
┌─────────────────────────────────────────────┐
│ 3. PROMPT ANNOTATION (if header enabled)    │
│    - Check x-vsr-enable-route-hints header  │
│    - Scan first 4KB for /*+ VSR: ... */     │
│    - Parse and validate                      │
│    - Strip annotation before forwarding     │
│    - Double-scan to verify no residual       │
└───────────┬─────────────────────────────────┘
            │ no hint found
            ▼
┌─────────────────────────────────────────────┐
│ 4. NORMAL FLOW                               │
│    handleCaching() → signalEval() →          │
│    decisionEngine() → modelSelection()       │
│    BUT: apply exclude_models / prefer_model  │
│    from hints if present                     │
└─────────────────────────────────────────────┘
```

---

## 6. Revised Hint Type Proposal

Based on this evaluation, here's the recommended hint type set for v1:

| Type | Syntax | Behavior | v1 Status |
|------|--------|----------|-----------|
| **强制模型** | `model=X` | Skip signals+decision, route to X (authz-gated) | Keep |
| **强制决策** | `decision=X` | Skip signals, use decision X's ModelRefs | Keep |
| **模型排除** | `exclude_models=A,B` | Remove from candidate set (min 1 remaining) | Keep |
| **模型允许** | `allow_models=A,B` | Restrict candidate set to these models | **Add** |
| **信号注入** | `signal=type/name` | Inject virtual signal (security+modality blocked) | Keep, expand blocklist |
| ~~软偏好~~ | ~~`prefer_model=X`~~ | Underspecified, defer to v2 | **Defer to v2** |

v2 candidates:
- `prefer_model=X` (with defined biasing mechanism per algorithm)
- `route_weights=A:N,B:M` (percentage-based traffic splitting)
- `strategy=X` (force selection algorithm)
- `lora=X` (force LoRA adapter)

---

## 7. Step-by-Step Implementation Sequence (Revised)

### Phase 1: Data Model & Config (3 files)

1. **`pkg/config/selection_config.go`** — Add:
   - `PlanBinding` struct (Name, Description, Priority, Enabled, ExpiresAt, Match, Action)
   - `PlanBindingMatch` struct (Keywords, Categories, DecisionPatterns, MatchOperator)
   - `PlanBindingAction` struct (ForceModel, ForceDecision, ExcludeModels, AllowModels, SignalInjections)
   - `RoutingHintConfig` struct (Enabled bool)

2. **`pkg/config/canonical_config.go`** — Add `PlanBindings []PlanBinding` to `CanonicalRouting`

3. **`pkg/config/canonical_global.go`** — Add `Hints RoutingHintConfig` to `CanonicalRouterGlobal`

### Phase 2: Validation (1 new file, 1 modified)

4. **`pkg/config/validator_binding.go`** (new) — `validatePlanBindingContracts()`:
   - Unique names, valid priorities, valid ISO 8601 dates
   - At least one match criterion
   - At least one action
   - forceModel references declared model
   - forceDecision references declared decision
   - excludeModels/allowModels are mutually exclusive

5. **`pkg/config/validator.go`** — Add `validatePlanBindingContracts` to `validateConfigStructure()`

### Phase 3: Core Logic (3 new files, 2 modified)

6. **`pkg/decision/hint_parser.go`** (new):
   - `ParseHeaderHint(value string) (*RoutingHint, error)`
   - `ParsePromptAnnotation(text string) (*RoutingHint, error)`
   - `StripAnnotations(text string) string`
   - `RoutingHint` struct (Type, Model, Decision, ExcludeModels, AllowModels, Signals)

7. **`pkg/decision/binding_engine.go`** (new):
   - `BindingEngine` struct with `Bindings []config.PlanBinding`
   - `Match(ctx *BindingMatchInput) *config.PlanBinding`
   - `BindingMatchInput` struct (Keywords, Categories, AuthzRoles, DecisionPatterns)

8. **`pkg/decision/engine.go`** — Minor: Add `BindingEngine` field (via options pattern) to `DecisionEngine`

9. **`pkg/extproc/req_filter_classification.go`** — Modify `performDecisionEvaluation()`:
   - Add hint/binding evaluation block before signal evaluation
   - When hint matches: build `DecisionResult` directly, skip to `finalizeDecisionEvaluation()`
   - When `exclude_models` or `allow_models` present: apply to candidate filtering after decision matching

10. **`pkg/extproc/req_filter_cache.go`** — Add hint detection → skip cache

### Phase 4: Headers & Replay (2 modified)

11. **`pkg/headers/headers.go`** — Add:
    - `VSRRouteHint = "x-vsr-route-hint"`
    - `VSREnableRouteHints = "x-vsr-enable-route-hints"`
    - `VSRDisableRouteHints = "x-vsr-disable-route-hints"`
    - `VSRMatchedBinding = "x-vsr-matched-binding"` (response header)

12. **`pkg/routerreplay/store/store.go`** — Add hint/binding fields to `Record`

### Phase 5: Model Selection Integration (1 modified)

13. **`pkg/modelselection/selector.go`** / **`pkg/selection/selector.go`** — Add `ExcludeModels` and `AllowModels` fields to `SelectionContext`, update `selectModelFromCandidates()` to filter before selection

### Phase 6: Tests

14. Unit tests for hint parsing (valid/invalid syntax, edge cases)
15. Unit tests for binding matching (keyword, category, combination, expiration)
16. Unit tests for security constraints (authz bypass blocked, signal injection blocked)
17. Integration test: binding overrides decision engine
18. Integration test: header hint overrides signal evaluation
19. Integration test: prompt annotation stripped before forwarding

---

## 8. Risk Matrix

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `decision=X` breaks ML selectors (empty category) | High | Medium | Extract domain from decision's rule tree statically |
| `exclude_models` empties candidate set | Medium | High | Validate min 1 candidate remains, otherwise ignore hint |
| Prompt annotation false positives in code blocks | Medium | Low | Acceptable — header gate is the real control |
| Binding matching too limited for complex scenarios | Medium | Medium | Document as v1 limitation, add embedding support in v2 |
| `decision=X`/`model=X` bypasses PII/jailbreak detection | Medium | High | Run security-only signal subset even when skipping full eval |
| Config validation surface growing too large | Low | Medium | Extract hint/binding validation into own file from day 1 |

---

## 9. Conclusion

The proposal's architecture is correct: the three-layer priority (binding > header > prompt), the integration point in `performDecisionEvaluation()`, and the security constraints are well-designed. The database-inspired approach fills a real gap in the LLM routing landscape.

**Required changes before implementation:**

1. Add `allow_models=A,B` hint type (dual of exclude)
2. Defer `prefer_model` to v2 (or specify the biasing mechanism per algorithm)
3. Add `modality` to the signal injection blocklist
4. Run security-only signal subset (authz, jailbreak, PII) even when `decision=X`/`model=X` skips full evaluation
5. Validate that `exclude_models`/`allow_models` leaves at least 1 candidate
6. Extract domain category from decision rule tree when `decision=X` is used
