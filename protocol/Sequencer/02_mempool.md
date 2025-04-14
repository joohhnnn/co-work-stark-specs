## 摘要

当交易通过网关进入 Starknet Sequencer 时，它们首先会被放入内存池 (Mempool)。内存池负责：

1. 验证交易的基本合法性（签名、格式、费用等）
2. 根据 gas 价格和账户 nonce 对交易进行优先级排序
3. 将满足条件的交易按顺序推送给区块构建器 (Batcher)

下文通过源码片段详细说明这一流程。

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

**关键点解析（gateway.rs::add_tx）**

* `add_tx` 处于 Sequencer 的最外围 —— Gateway 组件；它只做轻量校验，然后将交易异步推送给后端 Mempool。
* `AddTransactionArgsWrapper` 将交易本体与 P2P 元数据打包，方便后续追踪消息来源（本地 RPC / P2P）。
* `mempool_client.add_tx` 是真正改变系统状态的调用；返回错误即代表交易被拒绝。
* `metric_counters.*` 系列指标为 Prometheus 统计埋点，便于运营侧监控 TPS 与拒绝率。

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

**关键点解析（mempool.rs::add_tx）**

* `metric_handle` 负责埋点计数：`transaction_received` 与 `transaction_inserted` 两阶段可拆分定位瓶颈。  
* 写入前先执行 `remove_expired_txs` 与 `add_ready_declares`，实现 **垃圾回收 + 延迟 Declare 升格**，保持池内健康。  
* `exceeds_capacity` 拒绝超额写入；当前返回 `MempoolFull`，后续 TODO 将改为 **基于费率的逐出策略**。  
* `validate_incoming_tx` 做签名、nonce、fee cap 等完整校验；`handle_fee_escalation` 则可根据链上负载动态抬高费用，防御 spam。  
* 对 Declare 交易采用 `delayed_declares` 链表分流，待合约类哈希生成后再批量入池，避免影响普通调用交易时效。  
* 最后 `update_state_metrics` 刷新全局指标（容量、账户分布），供 Prometheus 仪表盘实时监控。

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

**关键点解析（mempool.rs::add_tx_inner）**

* `tx_pool.insert` 保存原始交易体，确保可通过 txHash 直接检索；若重复会在上层校验阶段被挡下。  
* 通过 `resolve_nonce` 读取链上最新 nonce，并与交易 nonce 比较：  
  * 若相等说明 **立刻可执行**，将其插入 `tx_queue`。  
  * 若更大则说明该交易"过期"，后续会在 `remove_expired_txs` 被清理。  
* 插入前 `tx_queue.remove(address)` 先移除同账户旧交易，实现 **一账户单队列条目** 的状态约束，防止 nonce 冲突。  
* `insert_to_tx_queue` 仅存放轻量引用 `TransactionReference`，与 `tx_pool` 存储分离，可降低内存占用。

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

**关键点解析（TransactionQueue）**

* `priority_queue` / `pending_queue` 使用 `BTreeSet` 保证 O(log n) 插入删除并天然排序。
* `gas_price_threshold` 在后台动态调整，可实现简易的 fee market：高于阈值的交易更快被打包。
* `address_to_tx` 充当 "一账户一条" 的去重索引，避免同一账户并行多笔非递增 nonce 交易导致的状态冲突。

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

**关键点解析（TransactionQueue::insert）**

* 通过 `address_to_tx.insert` 的 `assert_eq!` 保证单账户同时只保留一笔待处理交易；违反约束将直接 panic，有助于早期发现逻辑错误。  
* 根据 `max_l2_gas_price` 与动态 `gas_price_threshold` 将交易路由至 `priority_queue` 或 `pending_queue`，实现两级 **fee market**。  
* `override_gas_price_threshold_check` 允许后台逻辑强制加入高优先级，常用于升级或系统交易。  
* 利用 `BTreeSet` 按 `(gas_price, tx_hash)` 排序，可在 O(log n) 时间完成插入/删除且保持稳定顺序。  
* `assert!(new_tx_successfully_inserted)` 双保险确保键唯一，即便上游逻辑失误也能快速定位。

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

**关键点解析（mempool.rs::get_txs）**

* 循环弹出 `priority_queue` 批量交易，通过 `pop_ready_chunk` + `prune_expired_nonqueued_txs` 过滤过期或冲突交易。  
* `enqueue_next_eligible_txs` 在每次弹出后递补下一笔同账户交易，维持 **nonce 连续性**，避免区块构建时再次回滚。  
* 将已选交易 `stage` 到状态机，以便并发检查余额 / nonce，有效降低后续执行阶段的失败率。  
* 日志 `Returned {x} out of {n_txs}` 提供吞吐统计，为观察 mempool "出块效率" 的关键指标。  
* 返回 `InternalRpcTransaction` 数组直接供区块构建器使用，后续不再访问 mempool 状态，降低锁竞争。

### 注意

这些代码展示了交易如何在 Starknet Sequencer 的内存池中排队等待的实现。系统使用两级队列系统（优先队列和待处理队列）来确保高 gas 价格的交易优先被处理，同时确保所有交易都有机会被包含在区块中。这正是你描述的"当交易进入 Sequencer 时，它们加入一个快速移动的队列"的具体实现。