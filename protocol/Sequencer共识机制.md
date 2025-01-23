# Starknet Sequencer 共识机制分析

## 将区块发送给其他Sequencer

Starknet使用基于Tendermint的共识算法。当一个Sequencer构建了一个区块后，它会通过流式传输将区块发送给其他Sequencer：

[sequencer_consensus_context.rs:391-394](https://github.com/starkware-libs/sequencer/blob/0bb8b5d1/crates/apollo_consensus_orchestrator/src/sequencer_consensus_context.rs#L391-L394)
```rust
self.outbound_proposal_sender  
    .send((stream_id, proposal_receiver))  
    .await  
    .expect("Failed to send proposal receiver");
```

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

## 重要说明

- cende_ambassador组件负责将区块数据准备为适合Prover处理的格式
- 代码实现了一个完整的Tendermint共识流程，包括提议、投票（prevote和precommit）以及最终决策
- 代码中有一个用于处理延迟传播和重传的机制，确保网络问题不会影响共识
