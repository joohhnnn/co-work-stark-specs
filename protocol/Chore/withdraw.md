# Starknet 提现（Withdraw）指南

> 适用范围：使用 StarkGate 或自建桥接合约，将资产从 Starknet L2 转回以太坊 L1。

---

## 1. 基本概念

在 Starknet 生态中，**提现 (withdraw)** 指的是将资产从二层 (L2) 网络转移回以太坊主网 (L1) 的过程。本质上，它包含两笔交易与一次跨链消息：

1. **L2 侧的 initiate_token_withdraw**
   * 在桥接合约 (StarkGate) 上调用 `initiate_token_withdraw`。
   * L2 合约会 **燃烧**(burn) 等额代币，并向 L1 桥发送一条带有金额与接收地址的消息。
2. **等待状态更新**
   * 该跨链消息会随下一个批次的有效性证明提交到 L1；常见等待时间为 **~4–8 小时**，高峰期可能更久。
   * Starknet 探索器 (Voyager / StarkScan) 中交易状态需从 *Accepted on L2* 变为 *Accepted on L1* 才可继续。
3. **L1 侧的 withdraw**
   * 在以太坊桥合约上调用 `withdraw`（或 UI 中的 *Complete transfer* / *Claim*）。
   * 合约验证跨链消息并释放等额 ERC-20 资产至指定的 L1 地址。

---

## 2. 前置条件

| 需求               | 说明 |
|--------------------|------|
| 钱包               | L2：Argent X / Braavos；L1：MetaMask 等|
| 资产 & 手续费       | 需在 L2 拥有待提现资产及足够 ETH 支付 L2 Gas；L1 需预留少量 ETH 以执行 `withdraw`|
| 桥接合约地址        | 请在 [StarkGate 文档](https://docs.starknet.io/starkgate/overview/) 中确认对应 Token 的 L1/L2 地址|
| 区块浏览器          | Voyager 或 StarkScan（L2）；Etherscan（L1）|

---

## 3. 手动提现步骤

### 3.1 在 L2 发起提现

```bash
# CLI 示例：使用 starknet-js (TypeScript)
import { RpcProvider, Account, stark } from "starknet";

const provider = new RpcProvider({ nodeUrl: "https://free-rpc.net" });
const account = new Account(provider, YOUR_L2_ADDRESS, YOUR_L2_PRIVATE_KEY);

const starkGateL2 = "0x0123...";           // L2 Bridge 合约
const l1Token     = "0xA0b8...";           // ERC-20 L1 地址
const l1Recipient = "0xBEEF...";           // 收款人 L1 地址
const amount      = stark.bnToUint256("1000000");  // 6 位 USDC 示例

await account.execute({
  contractAddress: starkGateL2,
  entrypoint: "initiate_token_withdraw",
  calldata: [l1Token, l1Recipient, amount.low, amount.high]
});
```

> 也可在 Voyager / StarkScan 的 **Write Contract** 页直接填写 `l1_token`、`l1_recipient`、`amount` 并发送交易。

### 3.2 等待跨链消息被消费

1. 在交易详情页查看 `L1 status` 字段。
2. 若显示 *Accepted on L2*，耐心等待；更新频率≈每个 Starknet 证明。
3. 当状态变为 *Accepted on L1* 时，即可前往下一步。

### 3.3 在 L1 领取资产

两种方式：

* **官方 UI**：重新打开 StarkGate，连接同一钱包 → *Transfer log* → *Complete transfer* → 签名 MetaMask 交易。
* **手动调用**：在 Etherscan "Write as Proxy" 中调用 `withdraw`：
  * `recipient`：你的 L1 地址
  * `token`：ERC-20 合约地址
  * `amount`：uint256 数值，与发起时一致

交易确认后，资产会自动转入 `recipient`。

---

## 4. 一键提现 (1-Click Withdraw)

SpaceShard 提供的 **一键提现** 服务可自动完成第 3.3 步，流程如下：

1. 在 StarkGate 提现界面勾选 "Use the automatic withdrawal service by SpaceShard"。
2. 额外支付一笔小额 L2 ETH 作为服务费。
3. 服务端检测到 L1 状态后自动为你提交 `withdraw`，无需手动领取。
---