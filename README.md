# co-work-stark-specs

> 本仓库旨在对 Starknet 协议实现中的核心组件进行系统性拆解与说明。

> 主要内容聚焦于 **Sequencer、Prover、Settlement** 三大模块，涉及的代码横跨
> `starkware-libs/sequencer`、`starkware-libs/stone-prover`、`starkware-libs/starkex-contracts`、
> `madara-alliance/madara`、`starkware-libs/cairo-lang` 等多个仓库。

## Architecture Overview (English Outline)

## 快速示例流程：一笔转账如何走完全流程

> 下面用一个简单的例子把 Sequencer、Prover、Settlement 这三大核心组件串在一起，帮助你在正式阅读文档前先建立整体印象。

假设用户 **Alice** 想在 Starknet 上给 **Bob** 转 1 ETH，整个过程会经历以下 3 个阶段：

### 1. Sequencer —— 排队、执行、打包

1. 交易首先进入 Sequencer 的内存池（mempool）。
2. Sequencer 从队列中挑选一批交易，本地执行，验证余额充足等规则。
3. 执行成功的交易被写入区块；失败的交易被丢弃或回滚。
4. 区块被封装（seal）后广播给其他 Sequencer，大家快速达成共识。

### 2. Prover —— 数学证明

1. 刚确认的区块被送入 Prover 队列，可与其他区块并行处理。
2. Prover 记录完整的 **Execution Trace**（执行轨迹）以及 **State Diff**（状态差异）。
3. 接着进行 "吹胀 + 混合" 运算，将潜在错误数据放大、搅匀。
4. 算法随机抽样生成 **STARK 证明**，一旦数据有误就无法通过。

### 3. Settlement —— 以太坊最终结算

1. Prover 将 "STARK 证明 + State Diff" 打包成一笔 L1 交易发送到以太坊。
2. 以太坊合约 **Verifier** 抽检证明，若发现任何不一致立即拒绝。
3. 通过检查后交给 **Starknet Core** 合约写入状态。
4. 更新后的状态随以太坊区块被持久存储，交易至此获得不可篡改的最终性。

> 以上示例展示了 **Sequencer → Prover → Settlement** 的闭环，它们是本仓库的阅读重点；其余章节如 Governance、Chore 等则提供辅助信息，帮助你更深入理解 Starknet 生态。

下列文档按照推荐的阅读顺序排列：

## First of First
- [First of First](firstOfFirst.md)

## Sequencer
1. [Sequencer Overview](protocol/Sequencer/01_sequencer_overview.md)
2. [Mempool](protocol/Sequencer/02_mempool.md)
3. [Transaction Flow](protocol/Sequencer/03_transaction_flow.md)
4. [Block Sealing](protocol/Sequencer/04_block_sealing.md)
5. [Consensus](protocol/Sequencer/05_consensus.md)

## Prover
1. [Prover Line-up](protocol/Prover/01_lineup.md)
2. [Execution Trace](protocol/Prover/02_execution_trace.md)
3. [State Diff](protocol/Prover/03_state_diff.md)
4. [Data Blow-up](protocol/Prover/04_data_blowup.md)
5. [Valid Data](protocol/Prover/05_valid_data.md)

## Settlement
1. [Proof & State Diff](protocol/Settlement/01_proof_and_state_diff.md)
2. [Verifier Flow](protocol/Settlement/02_verifier_flow.md)
3. [Failure Handling](protocol/Settlement/03_failure_handling.md)
4. [Starknet Core](protocol/Settlement/04_starknet_core.md)
5. [Finalized](protocol/Settlement/05_finalized.md)

## Governance
- [Governance](governance/governance.md)
- [Governance Demo](governance/governanceDEMO.md)

## Chore (补充资料)
- [Engine API](protocol/Chore/Engine-API.md)
- [Cairo VM](protocol/Chore/cairo-vm.md)
- [Derivation](protocol/Chore/derivation.md)
- [Deposit](protocol/Chore/deposit.md)
- [Withdraw](protocol/Chore/withdraw.md)
- [Madara Data Submission Job](protocol/Chore/madara-Data%20Submission%20Job.md)
- [Mastering the Stone Prover for Developers](protocol/Chore/Mastering%20the%20Stone%20Prover%20for%20Developers.md)
- [Starknet Version History](protocol/Chore/starknetVersionHistory.md)

## Glossary
- [术语表](glossary.md)

---

如有任何问题或建议，欢迎 Issue 或 PR。