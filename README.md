# co-work-stark-specs

> 本仓库旨在对 Starknet 协议实现中的核心组件进行系统性拆解与说明。

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