# Starknet Sequencer 交易处理流程代码解析

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

这段代码明确显示：

- 成功的交易被添加到 execution_infos 中，准备被填入区块
- 失败的交易被标记并添加到 rejected_tx_hashes 集合中
- 如果配置为失败即终止(fail_on_err)，则会抛出错误；否则只是标记交易被拒绝并继续处理其他交易

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

这个方法展示了整个区块构建过程：

- 循环获取交易批次并处理，直到区块满了或达到截止时间
- 每一批交易通过 collect_execution_results_and_stream_txs 函数处理
- 成功的交易添加到执行信息集合中
- 最终生成包含所有成功执行的交易的区块构件

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
