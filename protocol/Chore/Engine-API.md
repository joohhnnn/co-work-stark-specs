在op中，op-node和op-geth是通过 Engine API 进行通信的

如果stark 也是分共识层和执行层，那么starknet 的共识层和执行层的通信方式需要在这里展开。

在 StarkNet 当前的开源实现中（sequencer 仓库），共识层与执行层的边界由 `Apollo Consensus` 与 `Batcher`（区块构建器 + Cairo 执行）两个独立进程 / 模块划分。两者之间不是像以太坊 MEV-Boost 那样通过 HTTP+JSON-RPC，而是使用 gRPC + proto 消息进行点到点流式通信，其协议由 `apollo_batcher_types`、`apollo_consensus` 等 crate 中的类型共同定义。

### 1. 角色划分
1. **Batcher（执行层）**
   • 负责收集 mempool 交易、依次执行 Cairo 代码、生成状态差分（state diff）与区块提案。
2. **Apollo Consensus（共识层）**
   • 对等节点之间采用 BFT 算法（HotStuff 变体）就 *哪一个区块提案* 达成一致。
   • 得票通过后，把最终决议告知本地 Batcher，由后者落盘并准备下一高度。
3. **State-sync / Papyrus / Juno（全节点）**
   • 通过订阅 `state_diff` 或重新执行交易来同步链状态，不直接参与上面接口。

### 2. 核心 gRPC 接口（摘自源码）
```12:43:sequencer/crates/apollo_batcher_types/src/batcher_types.rs
pub struct ProposeBlockInput { … }
…
pub enum SendProposalContent { Txs(Vec<InternalConsensusTransaction>), Finish, Abort }
…
pub struct ValidateBlockInput { … }
…
pub struct DecisionReachedInput { proposal_id: ProposalId }
```
这些消息在运行时形成 4 条主要调用链：

| 时序 | 调用方 → 被调方 | 方法 | 作用 |
| ---- | -------------- | ---- | ---- |
| ① | 共识层 | `ProposeBlockInput` | 向执行层请求在 *当前高度 / 轮次* 构建区块提案 |
| ② | 执行层 → 共识层 | `SendProposalContent`(Txs … / Finish) **流式返回** | 分批回传交易与最终 `state_diff_commitment` |
| ③ | 共识层（其他验证者） | `ValidateBlockInput` | 在本地复验提案的正确性（重放执行） |
| ④ | 共识层 | `DecisionReachedInput` | 共识达成后通知执行层将区块写入 canonical 链 |

### 3. 典型流程示意
1. **提案阶段**：轮到 *本节点* 成为 proposer 时，Apollo Consensus 调用 `ProposeBlockInput` 并启动 10-15 s deadline。Batcher 随即执行 mempool 并通过流式接口把交易批次（`Txs`）送回，最后发送 `Finish` 标识结束。
2. **验证阶段**：其余验证者收到提案后，对同一内容调用 `ValidateBlockInput`。Batcher 使用相同的执行逻辑确认 `(state_diff_commitment, gas_price, 时间戳…)` 与提案匹配，若失败则返回 `InvalidProposal`，共识层据此投反对票。
3. **投票 / 决议**：Apollo Consensus 使用 P2P 广播 `Vote`、`ProposalPart`（位于 `apollo_protobuf::consensus`）达成 BFT 保证的 *pre-commit → commit*。
4. **落盘阶段**：`DecisionReachedInput` 将 commit 结果交回 Batcher，Batcher 产生 `DecisionReachedResponse`，其中包含：
   • 精简状态差分 `ThinStateDiff`（供 state-sync）
   • 交易执行信息 / 中央对象（供 L1 证明）
   • 区块 Gas 使用量
   执行层随后：
   a) 更新本地数据库；
   b) 触发 **Prover** 生成 Cairo 证明；
   c) 通过 `StateSync` 服务向网络广播区块摘要与状态差分。

### 4. 与以太坊 Engine API 的异同
| 维度 | OP Engine API | StarkNet Batcher gRPC |
| -- | -- | -- |
| 连接数 | 共识↔︎执行一对一 | 可部署为同机或跨机，支持多实例（CLI 指 `--batcher-endpoint`） |
| 传输 | HTTP JSON-RPC | gRPC + protobuf（双向流） |
| 数据流 | 单请求单响应（`newPayload`, `forkchoiceUpdated`） | 长连接，连续流传递交易批次与心跳 |
| 校验方式 | 执行层本地 EVM Re-execute | 执行层 Cairo VM 重放，额外验证 `GasPrice`、`L1 Data Gas` 上下界 |

### 5. 未来演进
* `staking` 阶段去中心化后，多个 Sequencer / Validator 将通过上述通道各自驱动本地执行层；共识算法计划与 STRK 质押合并，接口层保持不变。
* 社区还有 [madara](https://github.com/keep-starknet-strange/madara) 等独立实现；它们通常复用相同的 gRPC schema，以便快速接入新的共识客户端。