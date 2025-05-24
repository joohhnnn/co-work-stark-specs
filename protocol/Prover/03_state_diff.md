# StateDiff 说明

## 1. 什么是 StateDiff

**StateDiff**（状态差异）指的是区块链系统（如 Starknet）在处理一批交易或一个区块之后，系统状态（账户余额、合约存储、类哈希、nonce 等）发生的**所有增量变更集合**。它本质上描述了区块前后状态的差异，是交易执行正确性与节点同步效率的核心。

## 2. 代码实现位置

| 语言 / 模块 | 文件路径 | 关键结构体 |
|------------|---------------------------------------|----------------|
| Python（业务逻辑） | `starkex-contracts/cairo-lang/src/starkware/starknet/business_logic/fact_state/state.py` | `StateDiff` |
| Rust（Sequencer / Papyrus） | `sequencer/crates/starknet_api/src/state.rs` | `StateDiff` / `ThinStateDiff` |
| C++（Stone Prover） | 无显式结构体；通过 `WriteTrace` + 内存快照自动派生 | — |

> 说明：Stone Prover 侧仅需验证 *old_root → new_root* 的状态转换正确性，因此不必显式持有 `StateDiff` 对象，而是直接依赖 Execution Trace 与最终 Merkle Root。

## 3. 核心结构体（Python 实现示例）

[state.py:430-455](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/starknet/business_logic/fact_state/state.py#L430-L455)

```python
@marshmallow_dataclass.dataclass(frozen=True)
class StateDiff(EverestStateDiff, DBObject):
    address_to_class_hash: Mapping[int, int]
    nonces: Mapping[DataAvailabilityMode, Mapping[int, int]]
    storage_updates: Mapping[DataAvailabilityMode, Mapping[int, Mapping[int, int]]]
    declared_classes: Mapping[int, int]
    block_info: BlockInfo
```

**关键点解析（StateDiff）**

* **增量存储**：只保存变更字段，显著降低 L1 数据可用性成本。
* **数据可用性模式**：`DataAvailabilityMode` 区分 L1 / L2 放置策略，灵活适配不同安全假设。
* **可合并**：`squash` 方法支持链式合并，便于批量提交或跨块滚动窗口。
* **可提交**：`commit` 把 diff 应用到 `SharedState`，产生新的 `global_state_root`。
* **序列化友好**：借助 `marshmallow_dataclass` 自动生成 (JSON/Binary) 序列化逻辑，方便网络传输与落盘。

## 4. StateDiff 生成流程

1. **执行追踪**：调用 `CpuComponent::WriteTrace` 等函数获取指令级 Trace。
2. **状态对比**：比较交易前后的 `CachedState` 快照，生成写集。
3. **归纳压缩**：映射到 `StateDiff / ThinStateDiff`，过滤零变更字段并按地址排序。
4. **证明输入**：与 Execution Trace 一同喂给 STARK Prover，验证 `old_root → new_root` 转移。
5. **节点同步**：Sequencer 将 `ThinStateDiff` 打包广播，轻客户端可快速重建状态。

**关键点解析（StateDiff 生成流程）**

* **最小化写集**：只记录真实发生的写操作，`len()` ≪ 全量状态，降低 L1 *data availability* 费用。
* **快照差分**：利用 `CachedState` 前后差分，无需遍历整棵 Merkle Trie，生成速度与写集规模线性相关。
* **排序写集**：对地址与存储键做稳定排序，保证跨实现一致性并减少树重排。
* **可组合性**：生成的 Diff 可通过 `squash` 叠加，实现批量块合并或重组。
* **长度可估**：`get_os_encoded_length()` 在生成时即可预估 calldata 大小，便于 sequencer 做 *fee estimation*。
## 5. 相关文件索引

| 文件 | 作用 |
|------|------|
| `src/starkware/air/cpu/component/cpu_component.inl` | 记录寄存器 / 内存写操作，生成 Execution Trace |
| `business_logic/fact_state/state.py` | `StateDiff` 实现：squash、commit、序列化 |
| `starknet_api/src/state.rs` | `ThinStateDiff`：面向节点同步的瘦版本 |
| `core/aggregator/output_parser.py` | 将 Cairo OS 输出解析为 `OsStateDiff`，再映射到 `StateDiff` |

## 6. 总结

`StateDiff` 作为跨语言的**状态增量抽象层**，连接了交易执行、STARK 证明与节点同步三大流程；其高效、可合并、易序列化的设计是 Starknet 可验证性与数据可用性的重要基石。