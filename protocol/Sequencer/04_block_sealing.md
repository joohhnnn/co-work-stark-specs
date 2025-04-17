# Starknet Sequencer 区块封装（Sealing）与批准流程解析

## 摘要

当区块构建器 (Batcher) 完成区块组装并通过内部验证后，Sequencer 会"封装(seal)"该区块并提交共识流程。本文聚焦于：

1. Batcher 如何调用 `commit_proposal_and_block` 将区块及状态差异写入本地存储
2. 如何将区块信息通知 L1 Provider 与 MemPool
3. 若任一环节失败如何回滚 (rollback) 以保证一致性

下列源码节选展示了这一过程的关键实现。

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

* **三段式提交策略**：先本地 `commit_proposal` 再外部 `commit_block`，最后通知 mempool，层层递进减少跨系统回滚概率。
* 对 L1 Provider 失败采取 **回滚+错误返回**，而 mempool 失败仅告警，体现 "外部一致性优先" 的容错权衡。
* `rejected_l1_handler_tx_hashes` 使用 set 交集过滤，避免向 L1 Provider 重报已消费或无关的 L1Handler 交易。
* 使用 `STORAGE_HEIGHT` Prometheus 指标追踪链高度，方便监控滞后或 fork。
* TODO 注释指出未来可能实现基于费率或队列的 mempool 回滚策略，暴露潜在技术债。



这个方法是在共识达成后由 `decision_reached` 方法调用的，整个过程实际上就是区块被"sealed and approved"的实现。方法执行以下关键步骤：

- 将区块提案（proposal）和状态差异（state diff）提交到存储
- 通知 L1 提供者（L1 provider）关于新区块
- 通知内存池（mempool）关于新区块
- 递增存储高度计数器

在这个过程中，如果提交到 L1 提供者失败，它会回滚状态差异并返回错误。

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

* 从内存 `executed_proposals` 中移除提案，确保同一高度只有一个已决提案，防止 double-commit。
* 将执行产物拆为多种子数据（state_diff、nonce map 等）传递给 commit 函数，实现 **数据解耦**。
* `FailOnError` 场景可通过 `commit_proposal_and_block` 的返回值向上冒泡，触发上层共识重试。
* 度量转换 `u64::try_from` 避免潜在溢出并保持类型安全。
* 函数末尾（省略处）还会负责广播高度完成事件、刷新 metrics 等 —— 体现 **单职责 + 事件驱动** 设计。

### Notes

这些代码展示了 Starknet Sequencer 如何封装和批准区块的过程：

- 区块的封装和批准是在共识达成后通过 `commit_proposal_and_block` 方法完成的
- 整个过程包括将区块提交到存储、通知 L1 提供者和内存池
- 如果任何步骤失败，会回滚变更以保持系统一致性
