# Starknet Core合约分析

## 1. 合约概述

Starknet Core是Starknet系统的核心合约,负责:
- 验证和更新Starknet状态
- 处理L1和L2之间的消息传递
- 管理系统配置和程序哈希
- 处理数据可用性

## 2. 证明和状态更新流程

### 2.1 整体流程
1. 接收STARK证明和状态差异(State Diff)
2. 验证证明的有效性
3. 确认状态差异已发送
4. 更新链上Starknet状态
5. 完成交易执行
6. 将交易添加到以太坊区块

### 2.2 详细步骤
```solidity
function updateStateInternal(
    uint256[] calldata programOutput, 
    bytes32 stateTransitionFact
) internal
```

**关键点解析（updateStateInternal）**

* 内部函数，由外部入口包装器调用，保证状态更新逻辑集中。  
* `programOutput` 为 Cairo OS 生成的区块执行输出，包含状态根、消息、日志等。  
* `stateTransitionFact` = `keccak256(proof_hash, state_diff_hash)`；上传 SHARP 后链上只需验证其有效性。

1. 验证程序输出
   - 检查输出长度
   - 验证配置哈希
   - 验证数据范围
   ```solidity
   require(programOutput.length > StarknetOutput.HEADER_SIZE, "STARKNET_OUTPUT_TOO_SHORT");
   require(programOutput[StarknetOutput.CONFIG_HASH_OFFSET] == configHash(), "INVALID_CONFIG_HASH");
   ```

**关键点解析（程序输出校验）**

* 双重 `require` 同时检查长度与配置版本，第一时间过滤格式错误或旧版本数据。  
* `CONFIG_HASH_OFFSET` 锁定 Starknet OS 版本，避免不同版本输出混淆导致的安全隐患。

2. 验证状态转换
   - 生成状态转换事实
   - 验证STARK证明
   - 确认状态差异
   ```solidity
   bytes32 sharpFact = keccak256(abi.encode(factProgramHash, stateTransitionFact));
   require(IFactRegistry(verifier()).isValid(sharpFact), "NO_STATE_TRANSITION_PROOF");
   emit LogStateTransitionFact(stateTransitionFact);
   ```

**关键点解析（状态转换验证）**

* `sharpFact` 将程序哈希与状态转移哈希绑定，确保证明与区块一一对应。  
* 通过 `FactRegistry` 查询 SHARP 是否已验证对应证明。  
* 事件日志便于外部索引服务追踪区块与证明匹配关系。

3. 更新链上状态
   - 更新全局状态根
   - 更新区块号
   - 更新区块哈希
   ```solidity
   state().update(programOutput);
   ```

4. 处理消息
   - 处理L2到L1消息
   - 处理L1到L2消息
   - 收取消息费用
   ```solidity
   outputOffset += StarknetOutput.processMessages(
       true,  // isL2ToL1
       programOutput[outputOffset:],
       l2ToL1Messages(),
       collectorAddress
   );
   ```

**关键点解析（消息处理）**

* 单次遍历 `programOutput` 子数组，高效解析全部跨层消息。  
* 返回新的 `outputOffset`，后续解析继续从正确位置开始。  
* `collectorAddress` 收取消息费用，避免额外存储读写。

5. 完成交易
   - 发出状态更新事件
   - 记录状态转换事实
   - 将交易添加到以太坊区块
   ```solidity
   emit LogStateUpdate(state_.globalRoot, state_.blockNumber, state_.blockHash);
   ```

### 2.3 重入保护
```solidity
// 在状态更新开始时检查区块号
state().checkPrevBlockNumber(programOutput);

// 执行状态更新操作
updateStateInternal(programOutput, stateTransitionFact);

// 在状态更新结束时再次检查区块号
state().checkNewBlockNumber(programOutput);
```

**关键点解析（重入保护）**

* 前后两次区块号校验确保整段状态更新不可重入。  
* 阻止同一区块被多次写入或被并发调用篡改。

## 3. 核心组件

### 3.1 合约继承关系
```solidity
contract Starknet is
    Identity,
    StarknetMessaging,
    StarknetGovernance,
    GovernedFinalizable,
    StarknetOperator,
    ContractInitializer,
    ProxySupport
```

**关键点解析（继承关系）**

* 多继承划分功能模块：治理、消息、升级、运营职责分离。  
* `ProxySupport` 提供可升级机制，便于未来迁移。

### 3.2 重要状态变量
- programHash: Starknet OS程序哈希
- aggregatorProgramHash: 聚合器程序哈希
- configHash: Starknet配置哈希
- verifier: 验证者地址
- state: Starknet状态结构

## 4. 主要功能

### 4.1 状态更新
```solidity
function updateState(
    uint256[] calldata programOutput,
    uint256 onchainDataHash,
    uint256 onchainDataSize
) external onlyOperator
```

**关键点解析（updateState）**

* Sequencer 对应的 L1 Operator 调用入口；`onlyOperator` 控制权限。  
* `onchainDataHash/Size` 对应 StateDiff 数据可用性片段，支持分片上传。

状态更新流程:
1. 验证程序输出
2. 检查区块号(防重入)
3. 验证数据可用性
4. 生成状态转换事实
5. 更新内部状态
6. 处理L1/L2消息

### 4.2 KZG数据可用性
```solidity
function updateStateKzgDA(
    uint256[] calldata programOutput, 
    bytes[] calldata kzgProofs
) external onlyOperator
```

**关键点解析（updateStateKzgDA）**

* 面向 Dencun 后 KZG blob DA 方案，引入 `kzgProofs` 批量验证可用性。  
* 与 `updateState` 共享内部逻辑，只替换 DA 路径。

KZG数据可用性验证流程:
1. 验证程序输出
2. 检查区块号
3. 验证KZG证明
4. 生成状态转换事实
5. 更新内部状态

### 4.3 消息处理
- 处理L2到L1的消息
- 处理L1到L2的消息
- 收取消息费用

## 5. 安全机制

### 5.1 重入保护
```solidity
state().checkPrevBlockNumber(programOutput);
// ... 状态更新操作 ...
state().checkNewBlockNumber(programOutput);
```

### 5.2 权限控制
- onlyOperator: 只有操作者可以更新状态
- onlyGovernance: 只有治理合约可以修改配置

### 5.3 数据验证
- 验证程序输出值范围
- 验证配置哈希
- 验证KZG证明

## 6. 重要事件

```solidity
event LogStateUpdate(
    uint256 globalRoot,
    int256 blockNumber,
    uint256 blockHash
);

event LogStateTransitionFact(
    bytes32 stateTransitionFact
);

event ConfigHashChanged(
    address indexed changedBy,
    uint256 oldConfigHash,
    uint256 newConfigHash
);
```

## 7. 配置管理

### 7.1 程序哈希管理
```solidity
function setProgramHash(uint256 newProgramHash) 
    external 
    notFinalized 
    onlyGovernance

function setAggregatorProgramHash(uint256 newAggregatorProgramHash)
    external
    notFinalized
    onlyGovernance
```

### 7.2 配置哈希管理
```solidity
function setConfigHash(uint256 newConfigHash) 
    external 
    notFinalized 
    onlyGovernance
```

### 7.3 费用收集地址管理
```solidity
function setFeeCollectionAddress(address collector) 
    external 
    onlyGovernance
```

## 8. 初始化流程

初始化需要以下参数:
1. programHash_: Starknet OS程序哈希
2. aggregatorProgramHash_: 聚合器程序哈希
3. verifier_: 验证者地址
4. configHash_: 配置哈希
5. initialState: 初始状态

## 9. 注意事项

1. 状态更新必须按顺序进行
2. 所有验证必须通过才能更新状态
3. 操作者权限控制
4. 重入保护机制
5. 数据可用性验证
6. 消息处理顺序
7. 费用收集机制
