# StarkEx 验证失败处理流程

## 1. 验证失败概述

当 STARK 证明包含无效数据时，验证器会通过多层验证机制快速检测并拒绝该证明。这种设计确保了系统的安全性和效率。

## 2. 失败检测机制

### 2.1 早期检测

[StarkVerifier.sol:90-120](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/StarkVerifier.sol#L90-L120)

```solidity
function initVerifierParams(uint256[] memory publicInput, uint256[] memory proofParams)
    internal
    view
    returns (uint256[] memory ctx)
{
    // 参数有效性检查
    require(proofParams.length > PROOF_PARAMS_FRI_STEPS_OFFSET, "Invalid proofParams.");
    require(
        proofParams.length ==
            (PROOF_PARAMS_FRI_STEPS_OFFSET + proofParams[PROOF_PARAMS_N_FRI_STEPS_OFFSET]),
        "Invalid proofParams."
    );
    
    // 安全参数检查
    require(proofOfWorkBits <= 50, "proofOfWorkBits must be at most 50");
    require(proofOfWorkBits >= minProofOfWorkBits, "minimum proofOfWorkBits not satisfied");
    require(proofOfWorkBits < numSecurityBits, "Proofs may not be purely based on PoW.");
}
```

**关键点解析（initVerifierParams）**

* 早期校验 `proofParams` 长度与 FRI 步数，防止畸形输入耗费后续 gas。
* 对 `proofOfWorkBits` 做上下限约束，兼顾安全性与验证成本。
* 返回的 `ctx` 作为贯穿整个验证流程的上下文，保存中间状态。

### 2.2 验证失败点

1. 参数验证失败

[StarkVerifier.sol:115-130](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/StarkVerifier.sol#L115-L130)

```solidity
// 参数范围检查
require(logBlowupFactor <= 16, "logBlowupFactor must be at most 16");
require(logBlowupFactor >= 1, "logBlowupFactor must be at least 1");
```

**关键点解析（参数范围检查）**

* 将 Trace 扩展倍数 `logBlowupFactor` 限定在 \[1,16]，避免 gas 爆炸或安全不足。
* 通过简单 `require` 提前失败，节省后续验证步骤。

2. 工作量证明验证失败

[VerifierChannel.sol:200-220](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/VerifierChannel.sol#L200-L220)

```solidity
function verifyProofOfWork(uint256 channelPtr, uint256 proofOfWorkBits) internal pure {
    uint256 proofOfWorkThreshold = uint256(1) << (256 - proofOfWorkBits);
    require(proofOfWorkDigest < proofOfWorkThreshold, "Proof of work check failed.");
}
```

**关键点解析（verifyProofOfWork）**

* 计算阈值 `2^{256-bits}`，bits 越大难度越高。
* 与 `proofOfWorkDigest` 比较，超过阈值即失败，保证提交者付出相应算力。

3. FRI 验证失败

[Fri.sol:30-60](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/Fri.sol#L30-L60)

```solidity
function friVerifyLayers(uint256[] memory ctx) internal view virtual {
    require(
        friStatementContract.isValid(keccak256(abi.encodePacked(dataToHash))),
        "INVALIDATED_FRI_STATEMENT"
    );
}
```

**关键点解析（friVerifyLayers）**

* 通过外部 `friStatementContract` 验证组合多项式承诺。
* 对数据做 `keccak256` 哈希，避免大型数组直接传递。
* 任何层验证失败都抛出特定错误码，便于问题定位。

4. Merkle 证明验证失败

[MerkleStatementVerifier.sol:10-30](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/MerkleStatementVerifier.sol#L10-L30)

```solidity
function verifyMerkle(
    uint256 channelPtr,
    uint256 queuePtr,
    bytes32 root,
    uint256 n
) internal view virtual override returns (bytes32) {
    require(merkleStatementContract.isValid(statement), "INVALIDATED_MERKLE_STATEMENT");
}
```

**关键点解析（verifyMerkle）**

* 针对公共输入或内存队列的 Merkle 证明，确保叶子 → 根路径正确。
* 通过外部合约 `merkleStatementContract` 独立复验，降低主验证器复杂度。

## 3. 失败处理流程

### 3.1 验证流程中断
当检测到无效数据时，验证器会立即停止验证过程：
1. 抛出异常
2. 回滚所有状态更改
3. 拒绝整个证明和状态差异

### 3.2 错误传播

[StarkVerifier.sol:495-525](https://github.com/starkware-libs/starkex-contracts/blob/aecf37f2/evm-verifier/solidity/contracts/StarkVerifier.sol#L495-L525)

```solidity
function verifyProof(
    uint256[] memory proofParams,
    uint256[] memory proof,
    uint256[] memory publicInput
) internal view override {
    // 初始化验证参数
    uint256[] memory ctx = initVerifierParams(publicInput, proofParams);
    
    // 如果任何验证步骤失败，整个验证过程将立即终止
    ctx[MM_TRACE_COMMITMENT] = uint256(readHash(channelPtr, true));
    oodsConsistencyCheck(ctx);
    computeFirstFriLayer(ctx);
    friVerifyLayers(ctx);
}
```

**关键点解析（verifyProof）**

* 统一调度各子验证步骤，顺序依赖保证前置条件已满足。
* 使用 `ctx` 共享跨步骤数据，避免重复计算。
* 任意子过程 `require` 失败即触发回滚，确保状态一致性。

### 3.3 状态回滚
1. 所有中间状态被丢弃
2. 合约状态恢复到验证开始前
3. 交易被标记为失败

## 4. 安全影响

### 4.1 数据污染防护
- 单个无效数据会导致整个证明被拒绝
- 防止部分有效数据被接受
- 确保数据完整性

### 4.2 攻击防护
- 防止无效证明被接受
- 防止状态被错误更新
- 保护系统安全

## 5. 错误处理最佳实践

### 5.1 错误信息
```solidity
// 清晰的错误信息
require(proofParams.length > PROOF_PARAMS_FRI_STEPS_OFFSET, "Invalid proofParams.");
require(proofOfWorkBits >= minProofOfWorkBits, "minimum proofOfWorkBits not satisfied");
require(merkleStatementContract.isValid(statement), "INVALIDATED_MERKLE_STATEMENT");
```

**关键点解析（错误信息）**

* 为每类错误提供明确字符串，方便前端与监控系统快速定位。
* 统一使用 `require` 抛错风格，保持合约可读性。

### 5.2 资源管理
- 早期失败检测
- 及时释放资源
- 避免不必要的计算

### 5.3 日志记录
```solidity
event LogBool(bool val);
event LogDebug(uint256 val);
```

**关键点解析（日志记录）**

* 利用事件将调试信息写入链上日志，供离线解析，不影响状态。
* 区分布尔与数值日志，降低解析歧义。

## 6. 总结

StarkEx 的验证失败处理机制确保了：
1. 快速失败：一旦检测到无效数据，立即停止验证
2. 完全回滚：所有状态更改被撤销
3. 安全保证：防止任何无效数据被接受
4. 资源效率：避免在无效证明上浪费计算资源
