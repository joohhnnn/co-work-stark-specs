# Starknet Sequencer 复杂交易处理能力分析

## 概述

本文对应整体流程的 **Step 1–2**：Sequencer 负责执行交易、提议区块，并依托 Cairo VM 与并发执行架构处理比以太坊更复杂的交易。

Starknet 是一个基于以太坊的 Layer 2 有效性证明（Validity Rollup）网络，也被称为零知识证明（ZK rollup）。它通过 STARK 密码学证明系统来降低交易数据的大小，从而实现安全且低成本的交易处理。与以太坊相比，Starknet 能够处理更复杂的交易，同时保持以太坊的可组合性和安全性。

Starknet Sequencer 通过其独特的设计架构，能够处理比以太坊更复杂的交易。本文将从代码层面分析其核心优势。

## 核心特性

### 1. 并发交易执行系统

Starknet Sequencer 实现了并发交易执行机制，允许同时处理多个交易，显著提升了吞吐量：

[worker_logic.rs#L68-L76](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/concurrency/worker_logic.rs#L68-L76)
```rust
pub struct WorkerExecutor<'a, S: StateReader> {  
    pub scheduler: Scheduler,  
    pub state: ThreadSafeVersionedState<S>,  
    pub chunk: &'a [Transaction],  
    pub execution_outputs: Box<[Mutex<Option<ExecutionTaskOutput>>]>,  
    pub block_context: &'a BlockContext,  
    pub bouncer: Mutex<&'a mut Bouncer>,  
    pub metrics: ConcurrencyMetrics,  
}
```

**关键点解析（WorkerExecutor 结构体）**

* `scheduler`：决定任务调度顺序，支持动态负载均衡；可根据 `chunk_size` 把交易切分成多个任务。
* `state`：线程安全的版本化状态（`ThreadSafeVersionedState`），允许多个执行线程在同一全局状态快照上读写并通过增量 diff 合并。
* `chunk`：分配给当前 Worker 的交易片段切片；避免复制，提高内存效率。
* `execution_outputs`：与 `chunk` 等长的 `Mutex<Option<ExecutionTaskOutput>>` 数组，用于跨线程回传执行结果。
* `block_context`：包含链 ID、block number、L1 gas price 等区块级常量，各 Worker 只读共享。
* `bouncer`：运行时资源守卫；在交易过度消耗资源时可提前终止，防止 DOS。
* `metrics`：执行期间实时记录指标（执行时间、失败率），用于后续性能分析。

与以太坊的单线程顺序执行模型相比，并发执行系统可以大幅提升复杂交易的处理能力：

[transaction_executor.rs#L204-L215](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/blockifier/transaction_executor.rs#L204-L215)
```rust
pub fn execute_txs(  
    &mut self,  
    txs: &[Transaction],  
) -> Vec<TransactionExecutorResult<TransactionExecutionOutput>> {  
    if !self.config.concurrency_config.enabled {  
        log::debug!("Executing transactions sequentially.");  
        self.execute_txs_sequentially(txs)  
    } else {  
        log::debug!("Executing transactions concurrently.");  
        let chunk_size = self.config.concurrency_config.chunk_size;  
        let n_workers = self.config.concurrency_config.n_workers;  
        // ...并发执行代码...  
    }  
}
```

**关键点解析（TransactionExecutor::execute_txs）**

* 通过 `concurrency_config.enabled` 开关实现 **按需并发**，方便链运营人员在出现兼容性问题时快速回退到串行模式。
* `chunk_size` 与 `n_workers` 双参数决定并行度：
  * `chunk_size` 越小，调度更灵活但线程切换开销更高。
  * `n_workers` 上限受 CPU 核数及 IO 限制；内部会基于 Rayon 或 tokio-task 分配线程池。
* 并发路径下：
  1. 先将交易列表切片并映射到 `WorkerExecutor` 任务
  2. 各任务独立执行，期间通过 **版本化状态** 隔离互相写集
  3. Scheduler 在收集所有 diff 后合并为全局 state diff，解决写冲突
* 结合 Cairo VM 的确定性特性，在保证可重入的前提下实现无锁并行执行。

### 2. 多样化交易类型支持

Starknet 支持更丰富的交易类型，可以实现更复杂的操作：

[transaction_execution.rs#L42-L46](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/transaction/transaction_execution.rs#L42-L46)
```rust
#[derive(Clone, Debug, derive_more::From)]  
pub enum Transaction {  
    Account(AccountTransaction),  
    L1Handler(L1HandlerTransaction),  
}
```

**关键点解析（Transaction 枚举）**

* `Transaction` 枚举将 Starknet 可执行交易抽象为 **两大类**：  
  * `Account`：由账户合约发起的常规交易（如 `Invoke`、`DeployAccount`），代表用户主动行为。  
  * `L1Handler`：由 L1→L2 消息触发的系统交易，保证跨链通信的原子性与顺序性。  
* `#[derive(Clone, Debug, derive_more::From)]` 宏派生：  
  * `From` 允许将 `AccountTransaction` / `L1HandlerTransaction` **隐式转换**为 `Transaction`，减少样板代码。  
  * `Clone` / `Debug` 便于测试、日志与并发场景下的任务复制。  
* 该枚举是 Sequencer 交易执行流水线的 **统一入口类型**，确保不同交易路径共用调度和计费逻辑。

与以太坊相比，Starknet 提供了更先进的交易执行接口：

[transactions.rs#L47-L84](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/transaction/transactions.rs#L47-L84)
```rust
pub trait ExecutableTransaction<U: UpdatableState>: Sized {  
    fn execute(  
        &self,  
        state: &mut U,  
        block_context: &BlockContext,  
    ) -> TransactionExecutionResult<TransactionExecutionInfo> {  
        // ...  
    }  
  
    fn execute_raw(  
        &self,  
        state: &mut TransactionalState<'_, U>,  
        block_context: &BlockContext,  
        concurrency_mode: bool,  
    ) -> TransactionExecutionResult<TransactionExecutionInfo>;  
}
```

**关键点解析（ExecutableTransaction trait）**

* **通用接口**：通过泛型 `U: UpdatableState` 解耦执行逻辑与具体的状态后端（内存、数据库或模拟环境）。  
* `execute`：高阶包装方法，内部构造 `TransactionalState` 并处理错误回滚，调用方无需关心细节。  
* `execute_raw`：核心实现路径，额外的 `concurrency_mode` 标志允许在并发模式下 **跳过部分昂贵的一致性检查** 以提升性能。  
* 返回 `TransactionExecutionInfo`，包含写集、事件、L1 消息等元数据，为后续 **fee 结算** 与 **state diff** 提供基础。  
* trait object 化让不同交易类型（`Account` / `L1Handler`）在调度层以统一接口处理，减少代码分支。

### 3. 可恢复的交易执行机制

Starknet 实现了可恢复的交易执行流程，允许比以太坊更灵活地处理复杂逻辑：

```mermaid
flowchart TD  
    subgraph "Sequential Execution"  
        S_Tx1["Transaction 1"] --> S_Tx2["Transaction 2"] --> S_Tx3["Transaction 3"]  
    end  
      
    subgraph "Concurrent Execution"  
        C_Tx1["Transaction 1"] --> VersionedState["Versioned State"]  
        C_Tx2["Transaction 2"] --> VersionedState  
        C_Tx3["Transaction 3"] --> VersionedState  
          
        VersionedState --> Scheduler["Scheduler"]  
        Scheduler --> Worker1["Worker 1"]  
        Scheduler --> Worker2["Worker 2"]  
          
        Worker1 --> Results["Results Collector"]  
        Worker2 --> Results  
    end
```

### 4. Cairo VM 执行环境

Starknet 使用 Cairo VM 执行环境，能够执行更复杂的计算：

[entry_point.rs#L159-L210](https://github.com/starkware-libs/sequencer/blob/8cd9dd20/crates/blockifier/src/execution/entry_point.rs#L159-L210)
```rust
pub fn execute(  
    mut self,  
    state: &mut dyn State,  
    context: &mut EntryPointExecutionContext,  
    remaining_gas: &mut u64,  
) -> EntryPointExecutionResult<CallInfo> {  
    // ...实现代码...  
      
    // 使用 Cairo VM 执行合约代码  
    execute_entry_point_call_wrapper(  
        self.into_executable(class_hash),  
        compiled_class,  
        state,  
        context,  
        remaining_gas,  
    )  
}
```

**关键点解析（EntryPoint::execute 函数）**

* 封装单次 **合约入口点调用** 的完整生命周期。  
* 关键参数：  
  * `state`：可变的全局状态抽象，负责读写 storage、class 等数据。  
  * `context`：执行上下文（caller、calldata、fee instrument 等），用于计费与嵌套调用管理。  
  * `remaining_gas`：对剩余 gas 的可变引用，在多级调用链中 **共享预算**。  
* 执行流程：  
  1. 将 `self` 转换为可执行对象（`into_executable`），绑定目标合约字节码。  
  2. 调用 `execute_entry_point_call_wrapper` 在 **Cairo VM** 中运行合约代码。  
  3. 捕获 VM 结果并包装为 `CallInfo`，包含返回值、事件、资源用量等。  
* 该设计保证在 Cairo VM 的 **确定性沙盒** 中运行，为 STARK 证明生成提供可重现的 trace。  
* 返回 `EntryPointExecutionResult`，供 Sequencer 后续 **state diff 合并** 与 **fee 结算**。

### 5. 资源跟踪与优化

Starknet 对资源进行更精细的跟踪和管理：

```mermaid
flowchart TD
subgraph "Gas Resources"
L1Gas["L1 Gas"]
L2Gas["L2 Gas"]
L1DataGas["L1 Data Gas"]
end
subgraph "Resource Tracking"
VMSteps["VM Steps"]
ComputationResources["Computation Resources"]
StateResources["State Resources"]
ExecutionSummary["Execution Summary"]
end
```

### 6. 账户抽象原生支持

Starknet 的账户结构受到以太坊 EIP-4337 的启发，但与以太坊不同，Starknet 原生支持账户抽象：

- 使用智能合约账户替代外部拥有账户（EOA）
- 支持自定义验证逻辑
- 所有合约（包括账户合约）都具有 nonce
- 非账户合约的 nonce 必须为零，这与以太坊不同

### 7. 数据可用性优化

Starknet 在数据可用性方面进行了多项优化：

- 使用 EIP-4844 实现更便宜的数据可用性
- 状态差异（state diffs）现在作为 blobs 而不是 calldata 发送
- 在 Starknet 0.13.1 版本中，支持使用 blobs 或 calldata 发送状态差异
- 在 0.13.3 版本中，引入了压缩状态差异机制

## 总结

Starknet 相比以太坊的主要优势包括：

1. 并发执行架构
2. Cairo VM 执行环境
3. 丰富的交易类型支持
4. 可恢复的交易执行机制
5. 精细的资源管理系统
6. 原生账户抽象支持
7. 优化的数据可用性机制
8. 高效的 L1-L2 消息传递

这些特性使 Starknet 能够：
- 处理更复杂的计算和状态管理
- 提供更高的交易处理效率
- 降低交易成本
- 保持与以太坊的兼容性和安全性
- 支持更丰富的应用场景

## 参考说明

- 以上代码引用主要来自 starkware-libs/sequencer 仓库
- Transaction Flow wiki 页面提供了很好的高级概述
- 所引用的代码展示了与以太坊相比的核心优势