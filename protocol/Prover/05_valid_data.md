# STARK证明系统中的数据验证流程

## 1. 概述

在STARK证明系统中，当所有数据都有效时，系统会通过一系列严格的数学验证步骤来确保交易的有效性。这个过程包括执行追踪生成、数据扩展和混合、随机采样以及证明生成等关键步骤。

## 2. 核心组件实现

### 2.1 STARK证明生成和执行追踪

- **证明生成核心实现**：`ProveStark` 方法（`stark.cc:384-456`）
  - 负责建立区块交易的数学有效性
  - 生成完整的STARK证明
  - 验证追踪数据的完整性

[stark.cc:400-430](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/stark/stark.cc#L400-L430)

```cpp
void StarkProver::ProveStark(std::unique_ptr<TraceContext> trace_context) {
  // 生成执行追踪
  ProfilingBlock profiling_block("Trace generation");
  Trace trace = trace_context->GetTrace();
  profiling_block.CloseBlock();
  
  // 验证追踪大小
  const size_t trace_length = trace.Length();
  const size_t first_trace_width = trace.Width();
  ValidateFirstTraceSize(trace_length, first_trace_width);
}
```

**关键点解析（ProveStark）**

* `TraceContext` 封装多种 AIR 布局，统一生成 Trace。
* `ProfilingBlock` 打点统计 Trace 生成耗时，便于性能分析。
* `ValidateFirstTraceSize` 立刻检查长度与宽度，防止异常数据耗尽内存。
* `profiling_block.CloseBlock()` 明确结束计时区间，确保统计准确。

### 2.2 数据扩展和混合处理

- **低度扩展(LDE)实现**：`EvalComposition` 方法（`composition_oracle.cc:113-177`）
  - 将执行追踪数据"blow up and mix"
  - 通过低度扩展确保数据完整性
  - 实现组合多项式评估

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

**关键点解析（DilutedCheckComponent::WriteTrace）**

* 对输入进行 **排序 + 稀疏复制**，将潜在错误放大至多个位置。
* `perm_interaction_elm / interaction_z / interaction_alpha` 提供随机性，将列混合进组合多项式。
* 转换为 `FieldElementT` 后按列式格式写入 Trace，便于后续 FRI 扩展。
* 算法整体复杂度为 *O(n log n)*（排序）+ *O(n)*（编码），对大规模数据友好。

### 2.3 随机采样机制

- **采样实现**：`ChooseQueryIndices` 函数（`fri_details.cc:71-85`）
  - 从扩展数据中选择随机样本
  - 确保采样的随机性和代表性
  - 为证明生成提供基础数据

[committed_trace.cc:210-235](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/stark/committed_trace.cc#L210-L235)

```cpp
std::map<RowCol, FieldElement> data_responses =
    table_verifier_->Query(data_queries, {} /* no integrity queries */);

// 验证追踪元素是否在基础域中
if (should_verify_base_field_) {
  InvokeFieldTemplateVersion(
      [&](auto field_tag) {
        using FieldElementT = typename decltype(field_tag)::type;
        for (const auto& [key, elm] : data_responses) {
          ASSERT_RELEASE(
              elm.template As<FieldElementT>().InBaseField(),
              "There is an element in the trace which is not in the base field.");
        }
      },
      field);
}
```

**关键点解析（ChooseQueryIndices & BaseFieldCheck）**

* `table_verifier_->Query` 一次性拉取随机行列组合，避免多次往返。
* `InvokeFieldTemplateVersion` 使用模板分发机制，对不同域类型复用验证逻辑。
* `InBaseField()` 确认元素位于基础域，杜绝扩域元素混入导致的约束失效。
* 随机采样覆盖率由安全参数控制，保证高概率发现不一致数据。

### 2.4 组合多项式和约束验证

- **多项式创建**：`CreateCompositionPolynomial` 函数（`stark.cc:46-58`）
  - 构建用于约束验证的组合多项式
  - 确保数学约束的正确性
  - 支持验证过程的完整性

### 2.5 低度测试和验证

- **FRI协议实现**：`PerformLowDegreeTest` 方法（`stark.cc:270-306`）
  - 执行低度测试确保数据有效性
  - 提供数学保证
  - 验证证明的正确性

## 3. 数据验证流程

当所有数据都有效时，系统会执行以下步骤：

1. **执行追踪生成**
   - 记录所有交易执行步骤
   - 生成完整的执行追踪数据
   - 确保追踪数据的完整性

2. **数据扩展和混合**
   - 执行低度扩展(LDE)
   - 进行数据混合处理
   - 确保任何错误都能被检测到

3. **随机采样和证明生成**
   - 从扩展数据中选择随机样本
   - 生成STARK证明
   - 确保证明的数学有效性

4. **验证过程**
   - 执行低度测试
   - 验证组合多项式
   - 确认数据有效性

## 5. 总结

STARK证明系统通过这一系列严格的验证步骤，确保了当所有数据都有效时，系统能够生成可靠的数学证明，为区块链交易提供可验证的安全性保证。整个过程结合了密码学、数学和计算机科学的多个领域，形成了一个完整的验证体系。
