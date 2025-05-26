# 执行追踪（Execution Trace）核心组件

## 概述

执行追踪是 Stone Prover 中的核心概念，它记录了交易执行的每个步骤和状态变化。这些记录为后续生成 STARK 证明提供了数学基础。执行追踪系统由以下几个核心组件组成：

## 1. TraceEntry 结构体

`TraceEntry` 结构体是执行追踪的基本单位，它记录了每个指令执行时的 CPU 状态。

[trace_utils.h:60-80](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/cairo/lang/vm/cpp/trace_utils.h#L60-L80)

```cpp
template <typename FieldElementT>
struct TraceEntry {
    FieldElementT ap;  // 分配指针（Allocation Pointer）
    FieldElementT fp;  // 帧指针（Frame Pointer）
    uint64_t pc;       // 程序计数器（Program Counter）
};
```

**关键点解析（TraceEntry）**

* 记录单步执行时 3 个关键 CPU 寄存器：`ap` / `fp` / `pc`，便于重建完整执行状态。  
* 使用 `FieldElementT` 泛型参数，使追踪可同时适配 **金田域** / **蒙哥马利域** 等不同字段实现。  
* 结构体极简，避免冗余字段，提高 Trace 序列化效率。

### 主要功能：
- 记录 CPU 寄存器的状态
- 追踪程序执行位置
- 为每个指令提供执行上下文

## 2. WriteTrace 方法

`WriteTrace` 方法是执行追踪的核心实现，负责执行指令并记录详细的执行过程。

[cpu_component.h:70-100](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/air/cpu/component/cpu_component.h#L70-L100)

```cpp
template <typename FieldElementT>
void CpuComponent<FieldElementT>::WriteTrace(
    const uint64_t instruction_index,
    const TraceEntry<FieldElementT>& values,
    const CpuMemory<FieldElementT>& memory,
    MemoryCell<FieldElementT>* memory_cell,
    RangeCheckCell<FieldElementT>* range_check_cell,
    const gsl::span<const gsl::span<FieldElementT>> trace
) const {
    // 指令解码
    const uint64_t encoded_instruction = ToUint64(memory.At(values.pc));
    const auto decoded_inst = DecodedInstruction::DecodeInstruction(encoded_instruction);
    
    // 记录操作数
    const uint64_t dst_addr = values.ComputeDstAddr(inst);
    const FieldElementT dst_value = memory.At(dst_addr);
    // ...
}
```

**关键点解析（CpuComponent::WriteTrace）**

* 以 **指令索引** 为主键写入追踪，不依赖绝对时间，方便乱序/并行执行记录。  
* 通过 `DecodedInstruction` 对 64bit 指令字进行解析，支持 Cairo 指令集多样操作。  
* `ComputeDstAddr` 结合当前寄存器状态计算目标地址，实现 **寄存器间接寻址**。  
* Trace 写入分三步：解码 → 读取操作数 → 更新寄存器 & 记录，保证单步原子性。  
* 结合 `RangeCheckCell` 在运行时校验操作数范围，为后续 AIR 约束提供输入。

### 主要功能：
- 指令解码和执行
- 内存地址计算
- 操作数获取和处理
- 运算结果计算
- 寄存器状态更新
- 执行追踪记录

## 3. Trace 类

`Trace` 类存储完整的执行追踪数据，以列的形式组织字段元素向量。

[trace.h:20-60](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/air/trace.h#L20-L60)

```cpp
class Trace {
public:
    // 获取追踪长度
    size_t Length() const;
    
    // 获取追踪宽度
    size_t Width() const;
    
    // 获取特定列的数据
    const std::vector<FieldElementT>& GetColumn(size_t index) const;
};
```

**关键点解析（Trace 类）**

* 使用 **列优先(column-oriented)** 存储，每列对应一个寄存器/内存字段，利于向量化与压缩。  
* `Length` = 执行步数，`Width` = 追踪列数，为 AIR 约束生成提供尺寸参数。  
* `GetColumn` 返回 const 引用，避免额外拷贝，保障大规模追踪访问效率。  
* 追踪数据可进一步序列化为 **Fri证明** 所需的多维矩阵输入。

### 主要功能：
- 存储所有执行步骤的数据
- 以列形式组织追踪数据
- 提供数据访问接口
- 支持追踪数据的序列化和反序列化

## 4. TraceContext 类

`TraceContext` 类管理整个追踪生成过程，提供追踪生成的上下文环境。

[trace_context.h:20-55](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/air/trace_context.h#L20-L55)

```cpp
class TraceContext {
public:
    // 获取追踪数据
    virtual Trace GetTrace() = 0;
    
    // 设置交互元素
    virtual void SetInteractionElements(const FieldElementVector& interaction_elms) = 0;
    
    // 获取交互追踪
    virtual Trace GetInteractionTrace() = 0;
};
```

**关键点解析（TraceContext 接口）**

* 抽象追踪生成流程，使不同组件（CPU、Pedersen、RangeCheck）可实现各自的 `TraceContext`。  
* `SetInteractionElements` 注入 **Fiat-Shamir 随机系数**，驱动多轮交互式证明非交互化。  
* `GetInteractionTrace` 返回交互列，用于验证方重建对数展开的约束多项式。  
* 通过纯虚接口 + 工厂方法，可根据证明规模动态选择 **单线程 / 并行** 实现。

### 主要功能：
- 初始化追踪生成参数
- 管理追踪生成过程
- 处理追踪数据的交互
- 提供追踪上下文环境

## 执行追踪工作流程

1. **初始化阶段**
   - 创建 TraceContext 实例
   - 设置追踪生成参数
   - 准备执行环境

2. **执行阶段**
   - 对每个指令创建 TraceEntry
   - 通过 WriteTrace 执行指令
   - 记录执行状态和结果

3. **数据组织阶段**
   - 将 TraceEntry 数据组织到 Trace 类中
   - 按列存储追踪数据
   - 准备数据用于证明生成

4. **证明生成阶段**
   - 使用追踪数据生成 STARK 证明
   - 验证执行正确性
   - 确保状态转换的有效性