# Starknet Sequencer 共识机制分析

## 摘要

本文聚焦整体流程的 **Step 6–7**：

1. 区块封装后如何通过网络广播给其他 Sequencer
2. 采用 Tendermint 风格的两阶段投票（Prevote / Precommit）达成 2/3 共识
3. 共识达成后如何将最终区块发送给 Prover 进行零知识证明生成

相关核心源码节选如下。

## 将区块发送给其他Sequencer

Starknet使用基于Tendermint的共识算法。当一个Sequencer构建了一个区块后，它会通过流式传输将区块发送给其他Sequencer：

[sequencer_consensus_context.rs:391-394](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L391-L394)
```rust
self.outbound_proposal_sender  
    .send((stream_id, proposal_receiver))  
    .await  
    .expect("Failed to send proposal receiver");
```

**关键点解析（outbound_proposal_sender::send）**

* 每个高度/轮次使用 `stream_id` 建立独立的 mpsc channel，避免不同区块提议数据串流互相干扰。  
* 通过 `(stream_id, proposal_receiver)` 传输 **接收端句柄**，实现零拷贝流水式发送，减少内存占用。  
* 若 `send` 失败立即 `expect` panic，因这属于致命的内部逻辑错误（本地队列应永不满）。

提议的区块内容分多个部分发送，包括ProposalInit（初始化信息）、BlockInfo（区块信息）、Transactions（交易批次）和ProposalFin（最终确认）：

[sequencer_consensus_context.rs:881-886](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L881-L886)
```rust
proposal_sender  
    .send(ProposalPart::Init(init))  
    .await  
    .expect("Failed to send proposal init");  
proposal_sender  
    .send(ProposalPart::BlockInfo(block_info.clone()))  
    .await  
    .expect("Failed to send block info");
```

**关键点解析（ProposalPart 分片发送）**

* 提案数据拆分为 `Init` / `BlockInfo` / `Transactions` / `Fin` 四段，支持 **流式 & 并行** 传输，降低单条消息尺寸。  
* `proposal_sender` 是 unbounded channel，允许后台任务持续 push 分片而不阻塞共识主循环。  
* 先发送 `Init` 保证对端初始化上下文，再发送 `BlockInfo` 和大量 `Transactions` 批次，最后 `Fin` 携带 commitment 进行完整性校验。  
* 分片顺序严格约定，若网络乱序到达由接收端 state machine 负责缓存并重组。

当重新提议区块时（例如在共识过程中更高的轮次），也会发送类似的消息：

[sequencer_consensus_context.rs:532-535](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L532-L535)
```rust
proposal_sender  
    .send(ProposalPart::Fin(ProposalFin { proposal_commitment: id }))  
    .await  
    .expect("Failed to broadcast proposal fin");
```

## Sequencer投票机制

Sequencer收到区块提议后进行验证，验证成功后发送投票。投票分为两种类型：Prevote（预投票）和Precommit（预提交）：

[consensus.rs:25-39](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_protobuf/src/consensus.rs#L25-L39)
```rust
#[derive(Debug, Default, Hash, Clone, Eq, PartialEq, Serialize, Deserialize)]  
pub enum VoteType {  
    Prevote,  
    #[default]  
    Precommit,  
}  
  
#[derive(Debug, Default, Hash, Clone, Eq, PartialEq, Serialize, Deserialize)]  
pub struct Vote {  
    pub vote_type: VoteType,  
    pub height: u64,  
    pub round: u32,  
    pub block_hash: Option<BlockHash>,  
    pub voter: ContractAddress,  
}
```

**关键点解析（Vote & VoteType）**

* Tendermint 共识的两阶段投票：`Prevote` → `Precommit`，两轮投票均通过才能进入下一高度。
* `Serialize/Deserialize` 便于通过 gRPC / P2P 网络高效传输。
* `Hash + Eq` 允许在 `HashMap<(round, voter), Vote>` 中做 O(1) 去重查询。

投票通过广播方式发送给其他Sequencer：

[single_height_consensus.rs:580-581](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus/src/single_height_consensus.rs#L580-L581)
```rust
info!("Broadcasting {vote:?}");  
context.broadcast(vote).await?;
```

而broadcast实现在sequencer_consensus_context.rs中：

[sequencer_consensus_context.rs:554-558](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L554-L558)
```rust
async fn broadcast(&mut self, message: Vote) -> Result<(), ConsensusError> {  
    trace!("Broadcasting message: {message:?}");  
    self.vote_broadcast_client.broadcast_message(message).await?;  
    Ok(())  
}
```

**关键点解析（SequencerConsensusContext::broadcast）**

* 将投票发送给网络层的唯一出口，抽象出底层传输细节。
* `trace!` 级日志仅在调试环境启用，主网默认关闭，避免性能损耗。
* 采用 `?` 早返回错误，让上层共识状态机能及时进入异常处理分支。

## 达成共识后发送区块给Provers

当收到足够多的投票（超过2/3的验证者）时，共识达成。在single_height_consensus.rs中检查是否有足够的支持票：

[single_height_consensus.rs:609-628](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus/src/single_height_consensus.rs#L609-L628)
```rust
let supporting_precommits: Vec<Vote> = self  
    .validators  
    .iter()  
    .filter_map(|v| {  
        let vote = self.precommits.get(&(round, *v))?;  
        if vote.block_hash == Some(proposal_id) { Some(vote.clone()) } else { None }  
    })  
    .collect();  
let quorum_size =  
    usize::try_from(self.state_machine.quorum_size()).expect("u32 should fit in usize");  
// TODO(matan): Check actual weights.  
if quorum_size > supporting_precommits.len() {  
    let msg = format!(  
        "Not enough supporting votes. quorum_size: {quorum_size}, num_supporting_votes: \  
         {}. supporting_votes: {supporting_precommits:?}",  
        supporting_precommits.len(),  
    );  
    return Err(invalid_decision(msg));  
}
```

**关键点解析（Quorum Check）**

* `quorum_size` 通过 `state_machine.quorum_size()` 计算（通常为总权重的 2/3 + 1）。
* 若 `supporting_precommits` 数量不足则提前返回错误，驱动状态机进入下一轮 `round += 1` 并重新提议区块。
* TODO 注释提醒未来需按验证者权重而非简单计数衡量票数，兼容 PoS 权重差异。

一旦达成共识，区块被认为是最终确定的，并在decision_reached方法中进行处理：

[sequencer_consensus_context.rs:560-678](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L560-L678)
```rust
async fn decision_reached(  
    &mut self,  
    block: ProposalCommitment,  
    precommits: Vec<Vote>,  
) -> Result<(), ConsensusError> {  
    let height = precommits[0].height;  
    info!("Finished consensus for height: {height}. Agreed on block: {:#064x}", block.0);  
  
    // ... [处理区块数据]  
  
    // 发送区块到state_sync_client  
    state_sync_client.add_new_block(sync_block).await.expect("Failed to add new block.");  
  
    // 准备数据给Prover  
    let _ = self  
        .cende_ambassador  
        .prepare_blob_for_next_height(BlobParameters {  
            block_info: cende_block_info,  
            state_diff,  
            compressed_state_diff: central_objects.compressed_state_diff,  
            transactions,  
            execution_infos: central_objects.execution_infos,  
            bouncer_weights: central_objects.bouncer_weights,  
            casm_hash_computation_data: central_objects.casm_hash_computation_data,  
            fee_market_info: FeeMarketInfo {  
                l2_gas_consumed: l2_gas_used,  
                next_l2_gas_price: self.l2_gas_price,  
            },  
        })  
        .await  
        .inspect_err(|e| {  
            error!("Failed to prepare blob for next height: {e:?}");  
        });  
    // ...  
}
```

**关键点解析（SequencerConsensusContext::decision_reached）**

* 达成 2/3\+ 共识后触发：封装 `sync_block` 发送至状态同步组件，同时准备向 Prover 输出的数据 blob。  
* `prepare_blob_for_next_height` 异步运行，若失败仅记录日志不回滚共识，充分解耦 **证明生成** 与 **共识安全**。  
* 通过 `FeeMarketInfo` 预计算下一个高度的 L2 gas 价格，实现 **动态费率调节**。  
* 函数内部还将 `central_objects`（包含 state diff、执行信息等）序列化为可压缩 blob，减少网络与存储开销。

## 重要说明

- cende_ambassador组件负责将区块数据准备为适合Prover处理的格式
- 代码实现了一个完整的Tendermint共识流程，包括提议、投票（prevote和precommit）以及最终决策
- 代码中有一个用于处理延迟传播和重传的机制，确保网络问题不会影响共识
