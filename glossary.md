存放一切讲解的时候遇到的名次解释，放入这里，并在文章的相关位置引入链接 参考 https://github.com/ethereum-optimism/specs/blob/main/specs/glossary.md#glossary

## 目录
- [General Terms](#general-terms)
- [Sequencing](#sequencing)
- [其他](#其他（随着protocol目录下的内容更新）)

## general-terms

### L1 (Layer 1)
面向最终结算与安全保证的底层区块链，本项目默认指以太坊主网。

### L2 (Layer 2)
运行在 L1 之上的扩容网络，通过 Rollup 等技术获得更高吞吐与更低成本，同时继承 L1 的安全属性。

### Validity Rollup （有效性 Rollup）
一种 Rollup 形态，使用零知识证明确保 L2 状态正确性；ZK Rollup 属于此类别。

### ZK Rollup
运用零知识证明（SNARK/STARK）的 Validity Rollup 具体实现，Starknet 即为 ZK Rollup。

### EOA（Externally Owned Account）
外部账户，由私钥直接控制；与合约账户（Contract Account）相对。

### Contract Account（合约账户）
其逻辑由智能合约代码定义的账户，可自定义签名规则与多重控制逻辑。

### Rollup
将大量 L2 交易压缩后提交摘要到 L1，以换取扩容性同时保持安全的通用技术总称。

### STARK
Scalable Transparent ARgument of Knowledge，无需可信设置的零知识证明系统，是 Starknet 的核心加密基元。

### SNARK
Succinct Non-interactive ARgument of Knowledge，需要可信设置但证明更短的零知识证明系统。

### Cairo
Starkware 提出的基于埃及象形文字理念的通用型编程语言，用于编写智能合约与生成 STARK 证明的程序。

### Cairo VM
执行 Cairo 字节码的虚拟机，Sequencer 与 Prover 均在其上运行。

### MPT（Merkle Patricia Trie）
以太坊状态存储结构，兼具前缀树与 Merkle 树特性，方便生成状态根哈希与成员证明。

### State Root（状态根）
整棵状态树（例如 MPT）的根哈希，区块头中的关键字段，用于证明整条链的状态正确性。

### Gas
衡量链上运算或存储资源消耗的单位；用户为每笔交易预付相应费用。

### Nonce
账户的递增计数器，用于防止交易重放并确定同一账户内交易顺序。

### Calldata
交易调用参数或跨链消息数据；在 L1 ↔ L2 桥接中亦常用此术语。

### Data Availability（DA）
确保 Rollup 交易数据长期可获得性的机制；缺失 DA 则无法从摘要重建完整状态。

## sequencing

### Sequencer（排序器）
Layer 2 网络中的"打包矿工"，负责接收交易、确定执行顺序、生成区块及状态差分，并喂给 Prover。

### Mempool
排序器的内存池，用于缓存待执行交易并按 gas 费/nonce 等指标排序。

### Batcher
负责将多笔交易或多个 L2 区块打包，提交给 Prover 或 L1 合约的后台进程/组件。

### Versioned State
Sequencer 内部的多版本状态快照结构，允许并发执行不同交易并在最后安全合并 diff。

### Worker Executor（Worker）
Sequencer 中实际并发执行交易的工作线程，按任务切片（chunk）获取交易并返回 Execution Output。

### State Diff（状态差分）
单笔或一批交易对全局状态产生的增量修改集合，最后被合并生成新的 State Root。

### L1 Handler
由 L1 发起的系统交易（跨链消息），在 L2 上自动执行以保持两层状态一致。

### DeployAccount
Starknet 特有的交易类型，用于创建账户合约并初始化公钥、签名算法等元数据。

### Invoke
最常见的交易类型，调用已部署合约中的函数，可携带 calldata 并触发事件。

### Priority Fee（Max L2 Gas Price）
用户可设置的额外费用，用以提高交易在 Sequencer 内部队列中的优先级。

## 其他（随着protocol目录下的内容更新）

