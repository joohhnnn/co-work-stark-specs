# 区块处理与并行处理机制

## 概述

石头证明器（Stone Prover）实现了一个高效的区块处理系统，该系统具有两个主要特性：
1. 区块加入快速移动的队列等待处理
2. 区块被组织成组并并行处理

## 实现架构

### 1. 任务队列系统

任务队列系统由 `TaskManager` 类实现，它维护一个任务队列和线程池。

#### TaskManager 类定义
```cpp
class TaskManager {
public:
    // 并行执行函数
    void ParallelFor(
        uint64_t start_idx, 
        uint64_t end_idx, 
        const std::function<void(const TaskInfo&)>& func,
        uint64_t max_chunk_size_for_lambda = 1, 
        uint64_t min_work_chunk = 1
    );
};
```

#### 任务队列实现
- 当 `ParallelFor` 被调用时，新任务被添加到队列中
- 任务在队列中等待，由线程池中的工作线程处理
- 使用条件变量和互斥锁确保线程安全

### 2. 并行处理机制

并行处理由 `ParallelTableProver` 类实现，它将大的段分解为多个子段进行并行处理。

#### ParallelTableProver 类定义
```cpp
class ParallelTableProver {
public:
    // 将段分解为子段并并行处理
    void AddSegmentForCommitment(
        const std::vector<FieldElementSpan>& segment,
        const std::vector<FieldElementSpan>& masks
    );
};
```

#### 并行处理实现
- 将段分解为 `n_tasks_per_segment_` 个子段
- 使用 `TaskManager` 并行处理这些子段
- 通过任务分组优化处理效率

## 工作流程

1. **区块入队**
   - 区块进入快速移动的队列
   - 由 `TaskManager` 管理队列和任务分配

2. **任务分组**
   - 区块被组织成组（segments）
   - 每个组被分解为多个子段

3. **并行处理**
   - 子段被分配给线程池中的工作线程
   - 多个子段同时处理，提高效率

4. **结果合并**
   - 处理结果被合并
   - 生成最终的 STARK 证明

## 技术特点

1. **分层并行化架构**
   - `TaskManager` 提供底层任务队列和线程池管理
   - `ParallelTableProver` 实现高层段处理并行化

2. **动态任务分配**
   - 根据系统资源动态调整任务大小
   - 确保处理效率最大化

3. **线程安全**
   - 使用互斥锁和条件变量
   - 确保多线程环境下的数据一致性

## 相关代码链接

- [TaskManager 队列系统](https://github.com/starkware-libs/stone-prover/blob/main/src/starkware/utils/task_manager.h)
- [TaskManager 实现](https://github.com/starkware-libs/stone-prover/blob/main/src/starkware/utils/task_manager.cc)
- [ParallelTableProver 定义](https://github.com/starkware-libs/stone-prover/blob/main/src/starkware/commitment_scheme/parallel_table_prover.h)
- [ParallelTableProver 实现](https://github.com/starkware-libs/stone-prover/blob/main/src/starkware/commitment_scheme/parallel_table_prover.cc)
