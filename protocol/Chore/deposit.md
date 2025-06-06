# StarkNet Deposit 指南

> 本文结合官方 [starknet-docs](https://docs.starknet.io) 以及社区资料撰写，帮助开发者与用户快速了解如何将资产从以太坊 L1 存入 StarkNet L2（以下简称 *Deposit*）。

## 1. 为什么需要 Deposit？

在 StarkNet 生态中，链上资产原生存在于以太坊 L1。要在 L2 上使用这些资产，首先需要通过 **StarkGate 桥** 将 ERC-20 代币从 L1 锁定并在 L2 铸造等值映射（wrapped）代币。该过程即 *Deposit*。

* 优势：
  * L2 交易费用显著低于 L1。
  * 执行速度更快（秒级）。
* 典型场景：在 StarkNet DApp 中支付 GAS、交互 DeFi、Mint NFT 等。

---

## 2. 流程总览

```mermaid
graph LR
A[用户在 L1 发送 approve+deposit] -->|以太坊确认| B(StarkGate L1 合约)
B -->|L1->L2 Message| C[消息队列]
C -->|证明+执行| D(StarkGate L2 合约)
D -->|铸造映射代币| E[用户 L2 地址]
```

1. 用户首先对 ERC-20 代币调用 `approve` 授权 StarkGate L1 合约。
2. 随后调用 `deposit`（或 `depositAndApprove`）将指定数量的代币锁定在 L1 合约。
3. 交易在以太坊被打包并最终确定（≈ 12 秒/区块，需等待 15–20 个确认以降低重组风险）。
4. StarkGate Relayer 监听事件并向 L2 发送 **L1→L2 Message**。
5. 该消息在 L2 被消耗后，L2 合约为用户铸造等值 wrapped 代币。至此，Deposit 完成。

> ⚠️ 如果 L1 交易已确认但 L2 迟迟未到账，可在 Voyager / Starkscan 搜索 L1 hash 查看消息状态，或手动触发 `consumeMessage`。

---

## 3. 费用与时间

| 阶段 | 费用 | 预计耗时 |
|------|------|---------|
| L1 approve + deposit | 需支付 L1 Gas（与 ERC-20 转账相当） | 1–5 分钟 |
| L1→L2 Message 处理  | L2 fee（由 StarkGate 支付，已包含在 deposit 调用内） | <1 分钟 |

实际花费受网络拥堵与 GasPrice 影响。

---

## 4. 实战示例

### 4.1 使用 Argent X 钱包（UI）

1. 打开扩展 → Bridge → 选择 **Ethereum → StarkNet**。
2. 选定代币与数量，点击 *Deposit*。
3. Metamask 弹窗确认 L1 交易（若首次需 2 步：approve + deposit）。
4. 交易确认后几分钟，刷新 StarkNet 余额即可看到对应代币。

### 4.2 使用 Starkli CLI

```bash
# 安装
cargo install starkli --locked

# 查询支持的 token
starkli bridge tokens --network mainnet

# Deposit 0.05 ETH 到 L2
starkli bridge deposit \
  --network mainnet \
  --token ETH \
  --amount 0.05
```

CLI 会打印 L1 交易 hash，可在 Etherscan 及 Starkscan 追踪进度。

### 4.3 智能合约内触发（Cairo 1 示例）

```rust
use starknet::core::starknet_env::send_message_to_l1;

#[external]
fn deposit_to_user(user_l1: felt252, amount: u256) {
    // 构造 StarkGate L1 合约地址与 payload
    let starkgate_l1: felt252 = 0x050...; // mainnet 地址
    send_message_to_l1(starkgate_l1, array![user_l1, amount.low, amount.high]);
}
```

开发者可在合约中直接发送 L1 消息实现批量 deposit。

---

## 5. 如何查询 Deposit 状态？

1. 复制 **L1 tx hash**，在 Voyager / Starkscan 搜索。
2. 页面会展示 “L1 7 20→ L2 Message” 状态（Pending / Accepted）。
3. 当状态变为 *Consumed*，表示已在 L2 铸造完成。
4. 亦可通过 Starkli：

```bash
starkli tx status <l1_tx_hash>
```

---

## 6. 常见问题 (FAQ)

1. **ETH 与 WETH 的区别？** 目前 StarkNet 上的 ETH 已包含 `approve` 逻辑，可直接用作 GAS，无需包裹。
2. **最低 Deposit 数量？** 由各代币合约 `min_deposit` 限制，一般为 0.001 单位。
3. **可否取消 Deposit？** L1 交易一旦确认不可取消，只能等待 L2 铸造完成后再将资产 withdraw 回 L1。
4. **支持的主网 Token？** ETH、USDC、USDT、DAI、wBTC 等，完整列表见 StarkGate UI 或 `bridge tokens` 命令。

