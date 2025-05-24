# STARK 证明系统中的执行追踪扩展与验证机制

## 1. 概述

STARK 证明系统通过执行追踪(Execution Trace)来记录和验证交易执行的每个步骤。系统会对执行追踪进行扩展和混合处理,以确保任何错误都能被及时发现。这个过程类似于化学实验中的混合器,将数据"blow up"(扩展)并混合,使得任何问题都变得显而易见。

## 2. 核心组件

### 2.1 执行追踪生成

[stark.cc:400-420](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/stark/stark.cc#L400-L420)

```cpp
void StarkProver::ProveStark(std::unique_ptr<TraceContext> trace_context) {
  // 生成执行追踪
  ProfilingBlock profiling_block("Trace generation");
  Trace trace = trace_context->GetTrace();
  
  // 验证追踪大小
  const size_t trace_length = trace.Length();
  const size_t first_trace_width = trace.Width();
  ValidateFirstTraceSize(trace_length, first_trace_width);
}
```

**关键点解析（StarkProver::ProveStark）**

* 以 `TraceContext` 为抽象层，支持不同 AIR Layout 的 Trace 生成。
* 在生成后立刻做 *Size Check*，避免恶意或错误输入导致内存爆炸。
* `ProfilingBlock` 方便对 Trace 阶段进行性能分析与热点定位。

---

### 2.2 数据扩展与混合

[diluted_check.inl:50-80](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/air/components/diluted_check/diluted_check.inl#L50-L80)

```cpp
void DilutedCheckComponentProverContext1<FieldElementT>::WriteTrace(
    FieldElementT perm_interaction_elm,
    FieldElementT interaction_z,
    FieldElementT interaction_alpha,
    gsl::span<const gsl::span<FieldElementT>> interaction_trace) const {
  
  // 数据排序和扩展
  std::vector<uint64_t> sorted_values(data_.begin(), data_.end());
  std::sort(sorted_values.begin(), sorted_values.end());

  // 转换为域元素
  std::vector<FieldElementT> elements;
  elements.reserve(data_.size());
  for (const uint64_t value : data_) {
    elements.push_back(FieldElementT::FromUint(value));
  }
}
```

**关键点解析（WriteTrace）**

* 数据排序和扩展：将数据排序并扩展，提高错误覆盖率。
* 转换为域元素：将排序后的数据转换为域元素，便于后续处理。
* 随机元素 `perm_interaction_elm / interaction_z / interaction_alpha` 混入组合多项式，提升安全性。
* 列式 Trace 写入完成后参与 FRI 扩展，与主 Trace 共同验证。

---

### 2.3 Out-of-Domain Sampling (OODS)

OODS 是验证过程的关键部分,在 `src/starkware/stark/oods.cc` 中实现:

[oods.cc:180-210](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/stark/oods.cc#L180-L210)

```cpp
std::vector<std::tuple<size_t, FieldElement, FieldElement>> VerifyOods(
    const ListOfCosets& evaluation_domain, 
    VerifierChannel* channel,
    const CompositionOracleVerifier& original_oracle,
    const FftBases& composition_eval_bases,
    const bool use_extension_field,
    const bool verifier_friendly_channel_updates) {
    
  // 验证追踪的有效性
  ASSERT_RELEASE(
      trace_side_value == broken_side_value, 
      "Out of domain sampling verification failed");
}
```

**关键点解析（VerifyOods）**

* OODS 通过随机挑战点确保多项式在"域外"也满足约束，防伪造。
* 若 `trace_side_value ≠ broken_side_value`，说明存在不一致 → 证明失败。
* `verifier_friendly_channel_updates` 决定是否在通道里回写额外信息，加速客户端验证。

---

## 3. 安全机制

### 3.1 数据污染检测

1. **数据扩展**：将长度 *n* 的 Trace 扩展成 *ρ·n*（ρ ≫ 1），提高错误覆盖率。
2. **随机混合**：通过随机域元素将列数据混合进组合多项式，避免结构化攻击。
3. **OODS 验证**：随机挑战点使错误无法只在"域内"隐藏。

### 3.2 错误放大效应

任何单个字节的错误都会在扩展域被复制多次，并在 OODS 时被捕捉 → 整个区块被标记为无效。

---

## 4. 实现细节

### 4.1 多项式分解器

在 `src/starkware/composition_polynomial/breaker.h` 中定义:

[breaker.h:20-45](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/composition_polynomial/breaker.h#L20-L45)

```cpp
class PolynomialBreaker {
  // 将大多项式分解为多个小多项式
  virtual std::vector<FieldElement> BreakCompositionPolynomial(
      const FieldElementSpan& composition_poly) = 0;
};
```

**关键点解析（PolynomialBreaker）**

* 将大多项式分解为多个小多项式，便于后续处理。

### 4.2 验证流程

* 通过分治降低 FFT 复杂度，从 *O(n log n)* 降至多核可分摊的子问题。
* 支持按需拆分或直接走递归路径，适配不同硬件。

### 4.2 整体验证流程

> 生成 Trace → 数据扩展与混合 → OODS 抽样 → 组合多项式验证 → 结果汇总

---

## 5. 安全保证

* 任意伪造都会在扩展域被**指数级放大**。
* 数学约束 + 随机挑战双重保护，保证零知识与完备性。
* 验证逻辑链路清晰，可在轻客户端快速复验。

---

## 6. 总结

Execution Trace 的扩展（Data Blow-Up）以及随机混合，使 STARK 证明系统能够在不暴露隐私的前提下放大并捕捉任何篡改痕迹；配合 OODS 与组合多项式验证，形成了完整而高效的安全闭环。
