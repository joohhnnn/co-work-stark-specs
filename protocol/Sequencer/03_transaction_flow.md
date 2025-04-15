# Starknet Sequencer 交易处理流程代码解析

## 摘要

本文对应整体流程的第 3-4 步：

* **交易排队 (Step 3 & 4)**：交易先进入 `TransactionQueue`，按 gas 价格及账户顺序排序，分优先与待处理两条队列。
* **批量处理 (Step 4)**：区块构建器每次从队列弹出一批交易，批量执行；成功交易写入 `execution_infos`，失败交易写入 `rejected_tx_hashes`。

下文通过源码展示这一过程的实现细节。

starkware-libs/sequencer 仓库中与交易排队、处理和区块填充相关的代码实现。

## 交易排队（Transactions line up in groups）

在 Starknet Sequencer 中，交易排队是通过 TransactionQueue 结构实现的：

[transaction_queue.rs:19-28](https://github.com/starkware-libs/sequencer/blob/ff7b6c78232c8e78a35f0bd6f5ee4bb43ef5b0bc/crates/apollo_mempool/src/transaction_queue.rs#L19-L28)

```rust
#[derive(Debug, Default)]  
pub struct TransactionQueue {  
    gas_price_threshold: GasPrice,  
    // Transactions with gas price above gas price threshold (sorted by tip).  
    priority_queue: BTreeSet<PriorityTransaction>,  
    // Transactions with gas price below gas price threshold (sorted by price).  
    pending_queue: BTreeSet<PendingTransaction>,  
    // Set of account addresses for efficient existence checks.  
    address_to_tx: HashMap<ContractAddress, TransactionReference>,  
}
```

交易被分为两组（两个队列）：

- priority_queue：包含 gas 价格高于阈值的优先交易
- pending_queue：包含 gas 价格低于阈值的待处理交易

当 mempool 模块需要获取交易时，会先从优先队列中获取：

[transaction_queue.rs:69-77](https://github.com/starkware-libs/sequencer/blob/ff7b6c78232c8e78a35f0bd6f5ee4bb43ef5b0bc/crates/apollo_mempool/src/transaction_queue.rs#L69-L77)

```rust
pub fn pop_ready_chunk(&mut self, n_txs: usize) -> Vec<TransactionReference> {  
    let txs: Vec<TransactionReference> =  
        (0..n_txs).filter_map(|_| self.priority_queue.pop_last().map(|tx| tx.0)).collect();  
    for tx in &txs {  
        self.address_to_tx.remove(&tx.address);  
    }  
  
    txs  
}
```

**关键点解析（TransactionQueue::pop_ready_chunk）**

* 仅从 `priority_queue` 中弹出，确保高费用交易优先。
* `pop_last()` 每次取出集合尾部（即最高 tip）的交易，天然按费用降序。
* 弹出后立刻从 `address_to_tx` 中删除，保证同一账户只能有一笔待执行交易。

## 交易处理和失败交易标记（Rejected or failed transactions are flagged）

在区块构建过程中，collect_execution_results_and_stream_txs 函数负责处理交易执行结果并标记失败的交易：

[block_builder.rs:326-354](https://github.com/starkware-libs/sequencer/blob/ff7b6c78232c8e78a35f0bd6f5ee4bb43ef5b0bc/crates/apollo_batcher/src/block_builder.rs#L326-L354)

```rust
match result {  
    Ok(tx_execution_info) => {  
        *l2_gas_used = l2_gas_used  
            .checked_add(tx_execution_info.receipt.gas.l2_gas)  
            .expect("Total L2 gas overflow.");  
  
        execution_data.execution_infos.insert(tx_hash, tx_execution_info);  
  
        if let Some(output_content_sender) = output_content_sender {  
            output_content_sender.send(input_tx)?;  
        }  
    }  
    Err(err) => {  
        debug!(  
            "Transaction {} failed with error: {}.",  
            tx_hash,  
            err.log_compatible_to_string()  
        );  
        if fail_on_err {  
            return Err(BlockBuilderError::FailOnError(  
                FailOnErrorCause::TransactionFailed(err),  
            ));  
        }  
        execution_data.rejected_tx_hashes.insert(tx_hash);  
    }  
}
```

**关键点解析（collect_execution_results_and_stream_txs）**

1. 成功路径：
   * 汇总 `l2_gas_used`，为 fee market & gas price 调整做数据基础。
   * `execution_infos` 保存成功交易的详细执行信息，后续写入区块与生成证明都要用到。
2. 失败路径：
   * 失败交易哈希写入 `rejected_tx_hashes`，方便 mempool 重新调度或客户端查询失败原因。
   * 如果 `fail_on_err` 打开，则区块构建立即终止并回滚，常用于单元测试或 dev 网络。

## 区块填充（They fill a block along with hundreds of other transactions）

交易如何填充到区块中，在 build_block 方法中实现：

[block_builder.rs:193-278](https://github.com/starkware-libs/sequencer/blob/ff7b6c78232c8e78a35f0bd6f5ee4bb43ef5b0bc/crates/apollo_batcher/src/block_builder.rs#L193-L278)

```rust
async fn build_block(&mut self) -> BlockBuilderResult<BlockExecutionArtifacts> {  
    let mut block_is_full = false;  
    let mut l2_gas_used = GasAmount::ZERO;  
    let mut execution_data = BlockTransactionExecutionData::default();  
    // TODO(yael 6/10/2024): delete the timeout condition once the executor has a timeout  
    while !block_is_full {  
        if tokio::time::Instant::now() >= self.execution_params.deadline {  
            info!("Block builder deadline reached.");  
            if self.execution_params.fail_on_err {  
                return Err(BlockBuilderError::FailOnError(FailOnErrorCause::DeadlineReached));  
            }  
            break;  
        }  
        if self.abort_signal_receiver.try_recv().is_ok() {  
            info!("Received abort signal. Aborting block builder.");  
            return Err(BlockBuilderError::Aborted);  
        }  
        let next_txs = match self.tx_provider.get_txs(self.tx_chunk_size).await {  
            // ... 省略一些错误处理代码  
            Ok(result) => result,  
        };  
        let next_tx_chunk = match next_txs {  
            NextTxs::Txs(txs) => txs,  
            NextTxs::End => break,  
        };  
        debug!("Got {} transactions from the transaction provider.", next_tx_chunk.len());  
        // ... 处理交易并构建区块  
        block_is_full = collect_execution_results_and_stream_txs(  
            next_tx_chunk,  
            results,  
            &mut l2_gas_used,  
            &mut execution_data,  
            &self.output_content_sender,  
            self.execution_params.fail_on_err,  
        )  
        .await?;  
    }  
    // ... 完成区块构建  
    Ok(BlockExecutionArtifacts {  
        execution_data,  
        commitment_state_diff: state_diff,  
        compressed_state_diff,  
        bouncer_weights,  
        l2_gas_used,  
        casm_hash_computation_data,  
    })  
}
```

**关键点解析（BlockBuilder::build_block）**

* "外层 while + 内层批处理" 结构，兼顾 **吞吐量** 与 **延迟**：
  * 超时 (`deadline`) 或 `abort_signal` 触发时立即停止，避免长时间锁占用。
  * 每批交易大小 `tx_chunk_size` 可动态配置，精细控制并发度。
* 通过 `collect_execution_results_and_stream_txs` 将执行结果转为区块内部中间状态；该函数返回布尔值 `block_is_full` 以指示是否已达到区块 gas 上限。
* 最终产出的 `BlockExecutionArtifacts` 既包含 state diff，又包含 gas 统计、Cairo CASM 哈希等，为 Prover 打包数据提供一站式输入。

## 区块的封装和批准

这个功能主要是在 sequencer 代码的批处理器(Batcher)组件中实现的。具体来说，当区块通过共识被批准后，会调用 `commit_proposal_and_block` 方法来完成区块的封装和批准。这个方法在 `crates/apollo_batcher/src/batcher.rs` 文件中：

[batcher.rs:552-626](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_batcher/src/batcher.rs#L552-L626)

```rust
async fn commit_proposal_and_block(  
    &mut self,  
    height: BlockNumber,  
    state_diff: ThinStateDiff,  
    address_to_nonce: HashMap<ContractAddress, Nonce>,  
    consumed_l1_handler_tx_hashes: IndexSet<TransactionHash>,  
    rejected_tx_hashes: HashSet<TransactionHash>,  
) -> BatcherResult<()> {  
    info!(  
        "Committing block at height {} and notifying mempool & L1 event provider of the block.",  
        height  
    );  
    trace!("Rejected transactions: {:#?}, State diff: {:#?}.", rejected_tx_hashes, state_diff);  
  
    // Commit the proposal to the storage.  
    self.storage_writer.commit_proposal(height, state_diff).map_err(|err| {  
        error!("Failed to commit proposal to storage: {}", err);  
        BatcherError::InternalError  
    })?;  
  
    // Notify the L1 provider of the new block.  
    let rejected_l1_handler_tx_hashes = rejected_tx_hashes  
        .iter()  
        .copied()  
        .filter(|tx_hash| consumed_l1_handler_tx_hashes.contains(tx_hash))  
        .collect();  
  
    let l1_provider_result = self  
        .l1_provider_client  
        .commit_block(  
            consumed_l1_handler_tx_hashes.iter().copied().collect(),  
            rejected_l1_handler_tx_hashes,  
            height,  
        )  
        .await;  
  
    // Return error if the commit to the L1 provider failed.  
    if let Err(err) = l1_provider_result {  
        match err {  
            L1ProviderClientError::L1ProviderError(L1ProviderError::UnexpectedHeight {  
                expected_height,  
                got,  
            }) => {  
                error!(  
                    "Unexpected height while committing block in L1 provider: expected={:?}, \  
                     got={:?}",  
                    expected_height, got  
                );  
            }  
            other_err => {  
                error!(  
                    "Unexpected error while committing block in L1 provider: {:?}",  
                    other_err  
                );  
            }  
        }  
        // Rollback the state diff in the storage.  
        self.storage_writer.revert_block(height);  
        return Err(BatcherError::InternalError);  
    }  
  
    // Notify the mempool of the new block.  
    let mempool_result = self  
        .mempool_client  
        .commit_block(CommitBlockArgs { address_to_nonce, rejected_tx_hashes })  
        .await;  
  
    if let Err(mempool_err) = mempool_result {  
        error!("Failed to commit block to mempool: {}", mempool_err);  
        // TODO(AlonH): Should we rollback the state diff and return an error?  
    };  
  
    STORAGE_HEIGHT.increment(1);  
    Ok(())  
}
```

**关键点解析（Batcher::commit_proposal_and_block）**

* 事务式处理流程：先写本地存储，再通知外部组件（L1 Provider、Mempool），任何一步失败都会回滚或记录错误，确保 **原子性**。  
* `rejected_l1_handler_tx_hashes` 通过交集过滤，只向 L1 报告真正被拒绝且已消费的系统交易，避免冗余数据。  
* 对 L1 Provider 的错误分类处理：`UnexpectedHeight` 单独日志，方便定位高度不一致问题；其他错误统一回滚并返回 `InternalError`。  
* `STORAGE_HEIGHT.increment(1)` 作为 Prometheus 计数器，记录链上高度推进速率。  
* TODO 注释提示：若通知 mempool 失败，目前仅打印日志，未来可能补充更严格的一致性保障。

`decision_reached` 方法是触发这整个过程的入口点：

[batcher.rs:501-528](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_batcher/src/batcher.rs#L501-L528)

```rust
pub async fn decision_reached(  
    &mut self,  
    input: DecisionReachedInput,  
) -> BatcherResult<DecisionReachedResponse> {  
    let height = self.active_height.ok_or(BatcherError::NoActiveHeight)?;  
  
    let proposal_id = input.proposal_id;  
    let proposal_result = self.executed_proposals.lock().await.remove(&proposal_id);  
    let block_execution_artifacts = proposal_result  
        .ok_or(BatcherError::ExecutedProposalNotFound { proposal_id })?  
        .map_err(|err| {  
            error!("Failed to get block execution artifacts: {}", err);  
            BatcherError::InternalError  
        })?;  
    let state_diff = block_execution_artifacts.thin_state_diff();  
    let n_txs = u64::try_from(block_execution_artifacts.tx_hashes().len())  
        .expect("Number of transactions should fit in u64");  
    let n_rejected_txs =  
        u64::try_from(block_execution_artifacts.execution_data.rejected_tx_hashes.len())  
            .expect("Number of rejected transactions should fit in u64");  
    self.commit_proposal_and_block(  
        height,  
        state_diff.clone(),  
        block_execution_artifacts.address_to_nonce(),  
        block_execution_artifacts.execution_data.consumed_l1_handler_tx_hashes,  
        block_execution_artifacts.execution_data.rejected_tx_hashes,  
    )  
    .await?;  
    // ...其余代码  
}
```

**关键点解析（Batcher::decision_reached）**

* 入口负责在 **共识完成** 后处理执行产物：解析提案 ID → 获取执行结果 → 提交区块。  
* 通过 `executed_proposals.lock().await.remove(&proposal_id)` 从缓存中取出异步执行结果，防止重复消费。  
* 将 `block_execution_artifacts` 拆分为 `state_diff`、`address_to_nonce` 等多种结构，供后续 commit 方法用。  
* 使用 `u64::try_from` 将交易数、拒绝交易数安全转换为指标，避免潜在溢出。  
* 调用 `commit_proposal_and_block` 之后，函数还会继续广播共识结果、更新度量等（此处省略），实现 **职责分离**。

### Notes

这些代码展示了 Starknet Sequencer 如何封装和批准区块的过程：

- 区块的封装和批准是在共识达成后通过 `commit_proposal_and_block` 方法完成的
- 整个过程包括将区块提交到存储、通知 L1 提供者和内存池
- 如果任何步骤失败，会回滚变更以保持系统一致性

以上代码实现了 Starknet Sequencer 对交易的处理流程，包括交易排队、处理和区块填充。其中值得注意的是:

- 交易根据 gas 价格被分成优先队列和待处理队列
- 失败的交易会被明确标记并从区块中排除
- 区块构建是一个批量处理过程，即"一组一组"地处理交易
- 区块构建会继续直到区块满了、达到时间限制或收到中止信号

完整的代码逻辑比这里展示的更复杂，包括详细的错误处理、验证和内存池管理。
