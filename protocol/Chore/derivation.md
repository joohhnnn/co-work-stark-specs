# Starknet Sequencer 如何将 L2 区块标记为安全

在 Starknet 里，一旦 L1（以太坊）上的 Starknet Core 合约完成 settlement，Sequencer／全节点会依照下列流程把相应 L2 区块标记为「安全 (safe)」：

## 摘要

本文聚焦于 Starknet 在 **L1 证明确认 (settlement) 完成后**，Sequencer / 全节点如何将对应的 L2 区块状态从 *AcceptedOnL2* 提升为 *AcceptedOnL1* 并进而认定为 **安全 (safe)**。主要涵盖：

1. L1 证明事件的监听与解析
2. 本地 block status 的更新与 safe head 计算
3. Storage marker、RPC 以及交易 finality 的联动关系

---

## 流程详解

## 1. 生成区块（PENDING / ACCEPTED_ON_L2）
- Sequencer 把用户交易打包成 L2 block，立刻广播给网络。
- 这时区块只是 accepted_on_l2，尚未经过任何 L1 验证。

## 2. 生成并提交 STARK 证明
- Prover（SHARP 聚合器）对若干连续区块生成一个 STARK 证明。
- Operator 调用 StarknetCore.update_state(new_root, proof, …) 把证明、state diff 等写入以太坊。
- Core 合约内部调用 GPS Verifier 合约验证证明，验证成功后把 global_root 更新为 new_root，并发出 StateUpdate 事件。

## 3. L1 证明确认（ACCEPTED_ON_L1）
- 当 update_state 交易被包含进某个以太坊区块并成功执行后，对应的 L2 block(s) 状态就变为 accepted_on_l1。
- Core 合约在事件里会带上 block_number, old_root, new_root，供 L2 节点索引。

## 4. 节点监听 L1 事件并更新本地状态
- Sequencer / 全节点都跑一个监听任务（在 Pathfinder 中叫 EthSyncWorker；在 Juno 中叫 l1syncer）。
- 它订阅以太坊节点的日志，当检测到 StateUpdate 或 LogStateUpdate 事件并且与本地区块号匹配时，就把该区块的 status 从 PENDING/ACCEPTED_ON_L2 提升为 ACCEPTED_ON_L1。

## 5. 计算 safe head
- 节点维护三条「链头」指针
  - latest: 本地收到的最高区块（可能尚未上链）。
  - unsafe: 最后一个与 Sequencer 同步的头。
  - safe: 最后一个「被 L1 验证且其所在以太坊区块已达到 finality 深度」的头。
- 当状态改为 ACCEPTED_ON_L1 且其以太坊区块深度 ≥ N （客户端一般设 12–64 个确认），该区块就被视为 safe。
- RPC starknet_syncing 就会把 safe_head 和 unsafe_head 返回给用户，表示哪一部分是不可逆转的。

## 6. 后果
- safe 之前的区块状态已被 L1 的有效性证明锁定，Sequencer 不能重组或回滚它们。
- 用户或合约在读取链状态时可根据 safe/unsafe 头决定风险容忍度。
