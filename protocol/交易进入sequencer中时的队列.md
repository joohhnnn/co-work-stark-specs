// ... existing code ...

## 四、Starknet Sequencer 的内存池交易队列

根据你的问题，我找到了 Starknet Sequencer 中关于交易在内存池中等待的相关代码实现。交易进入 Sequencer 后，确实会加入一个快速移动的队列，这主要在内存池(Mempool)的实现中。

### 交易进入 Sequencer 的流程

交易首先通过 Gateway 组件进入系统：

[gateway.rs:82-125](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_gateway/src/gateway.rs#L82-L125)

```rust
pub async fn add_tx(  
    &self,  
    tx: RpcTransaction,  
    p2p_message_metadata: Option<BroadcastedMessageMetadata>,  
) -> GatewayResult<GatewayOutput> {  
    debug!("Processing tx: {:?}", tx);  
      
    // ... 验证逻辑 ...  
      
    // 准备发送到内存池的交易  
    let add_tx_args = AddTransactionArgsWrapper { args: add_tx_args, p2p_message_metadata };  
    mempool_client_result_to_deprecated_gw_result(  
        self.mempool_client.add_tx(add_tx_args).await,  
    )?;  
      
    metric_counters.transaction_sent_to_mempool();  
      
    Ok(gateway_output)  
}
```

### 交易加入内存池队列

当交易被验证后，它们会被发送到内存池，并加入交易队列：

[mempool.rs:308-349](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_mempool/src/mempool.rs#L308-L349)

```rust
pub fn add_tx(&mut self, args: AddTransactionArgs) -> MempoolResult<()> {  
    let mut metric_handle = MempoolMetricHandle::new(&args.tx.tx);  
    metric_handle.count_transaction_received();  
  
    // 首先移除过期的交易  
    self.remove_expired_txs();  
    self.add_ready_declares();  
  
    // 检查内存池容量  
    if self.exceeds_capacity(&args.tx) {  
        // TODO(Dafna): we should be evicting transactions based on some policy here, instead of  
        // just returning an error.  
        return Err(MempoolError::MempoolFull);  
    }  
  
    let tx_reference = TransactionReference::new(&args.tx);  
    self.validate_incoming_tx(tx_reference, args.account_state.nonce)?;  
    self.handle_fee_escalation(&args.tx)?;  
  
    metric_handle.transaction_inserted();  
  
    // Declare交易会被延迟加入，其他交易直接加入  
    if let InternalRpcTransactionWithoutTxHash::Declare(_) = &args.tx.tx {  
        self.delayed_declares.push_back(self.clock.now(), args);  
    } else {  
        self.add_tx_inner(args);  
    }  
  
    self.update_state_metrics();  
    Ok(())  
}
```

交易添加到内存池后，会被插入到交易队列中：

[mempool.rs:355-375](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_mempool/src/mempool.rs#L355-L375)

```rust
fn add_tx_inner(&mut self, args: AddTransactionArgs) {  
    let AddTransactionArgs { tx, account_state } = args;  
    info!("Adding transaction to mempool.");  
    trace!("{tx:#?}");  
  
    let tx_reference = TransactionReference::new(&tx);  
  
    self.tx_pool  
        .insert(tx)  
        .expect("Duplicate transactions should cause an error during the validation stage.");  
  
    let AccountState { address, nonce: incoming_account_nonce } = account_state;  
    let account_nonce = self.state.resolve_nonce(address, incoming_account_nonce);  
    if tx_reference.nonce == account_nonce {  
        // 移除该账户可能已有的队列交易  
        self.tx_queue.remove(address);  
        // 将交易插入到交易队列  
        self.insert_to_tx_queue(tx_reference);  
    }  
}
```

### 交易队列实现

交易队列由两部分组成 - 优先队列和待处理队列：

[transaction_queue.rs:19-28](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_mempool/src/transaction_queue.rs#L19-L28)

```rust
#[derive(Debug, Default)]  
pub struct TransactionQueue {  
    gas_price_threshold: GasPrice,  
    // 气价高于阈值的交易（按小费排序）  
    priority_queue: BTreeSet<PriorityTransaction>,  
    // 气价低于阈值的交易（按价格排序）  
    pending_queue: BTreeSet<PendingTransaction>,  
    // 账户地址映射，用于高效的存在性检查  
    address_to_tx: HashMap<ContractAddress, TransactionReference>,  
}
```

交易被添加到队列的过程：

[transaction_queue.rs:35-58](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_mempool/src/transaction_queue.rs#L35-L58)

```rust
pub fn insert(  
    &mut self,  
    tx_reference: TransactionReference,  
    override_gas_price_threshold_check: bool,  
) {  
    assert_eq!(  
        self.address_to_tx.insert(tx_reference.address, tx_reference),  
        None,  
        "Only a single transaction from the same contract class can be in the mempool at a \  
         time."  
    );  
  
    // 根据gas价格和设置决定加入优先队列还是待处理队列  
    let to_pending_queue = !override_gas_price_threshold_check  
        && tx_reference.max_l2_gas_price < self.gas_price_threshold;  
    let new_tx_successfully_inserted = if to_pending_queue {  
        self.pending_queue.insert(tx_reference.into())  
    } else {  
        self.priority_queue.insert(tx_reference.into())  
    };  
    assert!(  
        new_tx_successfully_inserted,  
        "Keys should be unique; duplicates are checked prior."  
    );  
}
```

### 从队列中获取交易进行区块打包

当 Sequencer 需要创建区块时，它会从内存池中获取交易：

[mempool.rs:267-307](https://github.com/starkware-libs/sequencer/blob/master/crates/apollo_mempool/src/mempool.rs#L267-L307)

```rust
pub fn get_txs(&mut self, n_txs: usize) -> MempoolResult<Vec<InternalRpcTransaction>> {  
    self.add_ready_declares();  
    let mut eligible_tx_references: Vec<TransactionReference> = Vec::with_capacity(n_txs);  
    let mut n_remaining_txs = n_txs;  
  
    while n_remaining_txs > 0 && self.tx_queue.has_ready_txs() {  
        // 从优先队列中弹出一批交易  
        let chunk = self.tx_queue.pop_ready_chunk(n_remaining_txs);  
        let valid_txs = self.prune_expired_nonqueued_txs(chunk);  
  
        self.enqueue_next_eligible_txs(&valid_txs)?;  
        n_remaining_txs -= valid_txs.len();  
        eligible_tx_references.extend(valid_txs);  
    }  
  
    // 更新内存池状态  
    for tx_reference in &eligible_tx_references {  
        self.state.stage(tx_reference)?;  
    }  
  
    info!(  
        "Returned {} out of {n_txs} transactions, ready for sequencing.",  
        eligible_tx_references.len()  
    );  
      
    // ... 返回交易 ...  
}
```

### 注意

这些代码展示了交易如何在 Starknet Sequencer 的内存池中排队等待的实现。系统使用两级队列系统（优先队列和待处理队列）来确保高 gas 价格的交易优先被处理，同时确保所有交易都有机会被包含在区块中。这正是你描述的"当交易进入 Sequencer 时，它们加入一个快速移动的队列"的具体实现。