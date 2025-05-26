# 区块处理与并行处理机制

## 概述

石头证明器（Stone Prover）实现了一个高效的区块处理系统，该系统具有两个主要特性：
1. 区块加入快速移动的队列等待处理
2. 区块被组织成组并并行处理

## 实现架构

### 1. 任务队列系统

任务队列系统由 `TaskManager` 类实现，它维护一个任务队列和线程池。

#### TaskManager 类定义
[task_manager.h:40-80](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/utils/task_manager.h#L40-L80)

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

**关键点解析（TaskManager::ParallelFor）**

* `start_idx` / `end_idx`：要并行遍历的 **闭区间索引范围**；内部会按任务粒度切分。
* `func`：用户提供的回调，签名为 `void(const TaskInfo&)`，其中 `TaskInfo` 封装当前子任务的范围与线程本地上下文。
* `max_chunk_size_for_lambda`：控制一次回调可处理的 **最大元素个数**，过大浪费 cache，过小增加调度开销。
* `min_work_chunk`：当剩余工作量低于该阈值时直接串行处理，避免递归切分导致的线程争用。
* 内部使用 **工作窃取(work-stealing) 线程池**：空闲线程可从其他线程队列中"偷"任务，实现负载均衡。
* 通过互斥锁与条件变量实现队列同步，保证任务入队/出队的 **线程安全**。

#### 任务队列实现
- 当 `ParallelFor` 被调用时，新任务被添加到队列中
- 任务在队列中等待，由线程池中的工作线程处理
- 使用条件变量和互斥锁确保线程安全

### 2. 并行处理机制

并行处理由 `ParallelTableProver` 类实现，它将大的段分解为多个子段进行并行处理。

#### ParallelTableProver 类定义
[parallel_table_prover.h:20-60](https://github.com/starkware-libs/stone-prover/blob/1414a545/src/starkware/commitment_scheme/parallel_table_prover.h#L20-L60)

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

**关键点解析（ParallelTableProver::AddSegmentForCommitment）**

* 将一个大型 `segment` 切分为 `n_tasks_per_segment_` 份子段，每段大小≈`segment.size()/n_tasks_per_segment_`。
* `masks` 用于按列筛选或置零，被 **按相同策略** 切分以保持对齐。
* 通过 `TaskManager::ParallelFor` 并行计算每个子段的 **多项式承诺(commitment)**，提升 CPU 利用率。
* 支持 **流水线化**：可在前一 segment 计算时异步提交下一段任务，最大化线程池吞吐。
* 处理完所有子段后在主线程合并结果，生成整个 segment 的承诺值，以保证 **确定性** 与 **顺序一致性**。

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