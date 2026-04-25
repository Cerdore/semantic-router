# CA-3 — ensureLoRAInitialized Broken Double-checked Locking Allows Double C Init

- **来源**: 代码分析 (unreported)
- **优先级**: P1
- **状态**: open
- **发现日期**: 2026-04-25
- **涉及模块**: classification/unified_classifier.go:479-500

## 问题描述

在检查与初始化之间释放并重新获取锁，创建窗口允许两个 goroutine 同时调用 `C.init_lora_unified_classifier()`:

```go
uc.mu.Lock()
if uc.loraInitialized { uc.mu.Unlock(); return nil }
uc.mu.Unlock()           // ← 窗口: G2 可在此获取锁

uc.mu.Lock()             // ← G2 也已检查 loraInitialized，发现为 false
defer uc.mu.Unlock()
if uc.loraInitialized { return nil }  // 双重检查（不充分）
```

## 影响

可能导致 C 侧双重初始化、ML 模型权重在内存中重复，或崩溃。

## 修复

正确的双重检查加锁不应在检查后解锁:

```go
func (uc *UnifiedClassifier) ensureLoRAInitialized() error {
    uc.mu.Lock()
    defer uc.mu.Unlock()

    if uc.loraInitialized {
        return nil
    }
    if err := uc.initializeLoRABindings(); err != nil {
        return err
    }
    uc.loraInitialized = true
    return nil
}
```

## 备注

-
