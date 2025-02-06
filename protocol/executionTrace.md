# 执行追踪（Execution Trace）核心组件

## 概述

执行追踪是 Stone Prover 中的核心概念，它记录了交易执行的每个步骤和状态变化。这些记录为后续生成 STARK 证明提供了数学基础。执行追踪系统由以下几个核心组件组成：

## 1. TraceEntry 结构体

`TraceEntry` 结构体是执行追踪的基本单位，它记录了每个指令执行时的 CPU 状态。

```cpp
template <typename FieldElementT>
struct TraceEntry {
    FieldElementT ap;  // 分配指针（Allocation Pointer）
    FieldElementT fp;  // 帧指针（Frame Pointer）
    uint64_t pc;       // 程序计数器（Program Counter）
};
```

### 主要功能：
- 记录 CPU 寄存器的状态
- 追踪程序执行位置
- 为每个指令提供执行上下文

## 2. WriteTrace 方法

`WriteTrace` 方法是执行追踪的核心实现，负责执行指令并记录详细的执行过程。

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

### 主要功能：
- 指令解码和执行
- 内存地址计算
- 操作数获取和处理
- 运算结果计算
- 寄存器状态更新
- 执行追踪记录

## 3. Trace 类

`Trace` 类存储完整的执行追踪数据，以列的形式组织字段元素向量。

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

### 主要功能：
- 存储所有执行步骤的数据
- 以列形式组织追踪数据
- 提供数据访问接口
- 支持追踪数据的序列化和反序列化

## 4. TraceContext 类

`TraceContext` 类管理整个追踪生成过程，提供追踪生成的上下文环境。

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

## 技术特点

1. **完整性**
   - 记录所有执行步骤
   - 追踪所有状态变化
   - 确保执行过程可验证

2. **效率**
   - 优化的数据结构
   - 高效的执行记录
   - 支持并行处理

3. **可验证性**
   - 提供数学证明基础
   - 支持状态验证
   - 确保执行正确性

## 应用场景

1. **交易验证**
   - 验证交易执行正确性
   - 确保状态转换有效
   - 提供执行证明

2. **状态追踪**
   - 记录系统状态变化
   - 追踪内存访问
   - 监控执行流程

3. **证明生成**
   - 为 STARK 证明提供数据
   - 支持零知识证明
   - 确保系统安全性
