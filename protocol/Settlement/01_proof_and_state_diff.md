# Madara: 将STARK证明和State Diff作为交易发送到以太坊的实现

## 概述

Madara实现了两种将STARK证明和状态差异（State Diff）发送到以太坊的方法：

1. **通过calldata发送**  
   使用`update_state_calldata`方法，将状态数据直接包含在交易的calldata中。  
   [lib.rs:185-210](https://github.com/madara-alliance/madara/blob/2c88da11/orchestrator/crates/settlement-clients/ethereum/src/lib.rs#L185-L210)

2. **通过EIP-4844 blob发送**  
   使用`update_state_with_blobs`方法，通过更高效的数据blob机制发送状态差异。  
   [lib.rs:213-316](https://github.com/madara-alliance/madara/blob/2c88da11/orchestrator/crates/settlement-clients/ethereum/src/lib.rs#L213-L316)

## 详细实现流程

### 1. 状态差异收集与准备

- 从Starknet收集状态更新，转换为适合传输的格式。  
- 状态差异包括存储变更、部署的合约、声明的类和非ce更新，以Felt值（域元素）形式表示。

### 2. 数据转换与传输

- 将blob数据转换为BigUint值。
- 应用FFT（快速傅里叶变换）准备KZG承诺。
- 将转换后的数据分割为适合EIP-4844 blob交易的大小。

### 3. KZG证明生成

- 为每个状态差异数据块生成KZG证明，确保数据可用性。  
- 使用`EthereumSettlementClient::build_proof`方法生成并验证这些证明。

### 4. 构建EIP-4844交易

- 构建包含程序输出（STARK证明验证结果）、blob数据（状态差异）和KZG证明的EIP-4844 blob交易。
- 设置适当的gas参数和安全边际，以应对网络波动。

### 5. 准备交易数据

- 根据合约预期结构格式化交易输入数据，包括方法ID、程序输出和KZG证明数据。
- 使用`prepare_sidecar`准备blob sidecar（包含实际blob数据、承诺和证明）。

### 6. 智能合约交互

- 调用以太坊上的Starknet Validity Contract的`updateStateKzgDA`函数。
- 合约接口定义在`validity_interface.rs:55`，客户端实现在`validity_interface.rs:132-157`。

### 7. 交易提交与验证

- 提交交易后，等待交易最终性，需要特定数量的区块确认。
- 验证交易包含，确保状态更新在以太坊上成功处理。

## 代码实现

### 通过calldata发送

[lib.rs#L185-L212](https://github.com/madara-alliance/madara/blob/b6c0651f98df32ddaa073c41ad95326bbd1ca846/orchestrator/crates/settlement-clients/ethereum/src/lib.rs#L185-L212)

```rust
async fn update_state_calldata(
    &self,
    program_output: Vec<[u8; 32]>,
    onchain_data_hash: [u8; 32],
    onchain_data_size: [u8; 32],
) -> Result<String> {
    tracing::info!(
        log_type = "starting",
        category = "update_state",
        function_type = "calldata",
        "Updating state with calldata."
    );
    let program_output: Vec<U256> = vec_u8_32_to_vec_u256(program_output.as_slice())?;
    let onchain_data_hash: U256 = slice_u8_to_u256(&onchain_data_hash)?;
    let onchain_data_size = U256::from_be_bytes(onchain_data_size);
    let tx_receipt =
        self.core_contract_client.update_state(program_output, onchain_data_hash, onchain_data_size).await?;
    tracing::info!(
        log_type = "completed",
        category = "update_state",
        function_type = "calldata",
        tx_hash = %tx_receipt.transaction_hash,
        "State updated with calldata."
    );
    Ok(format!("0x{:x}", tx_receipt.transaction_hash))
}
```

**关键点解析（update_state_calldata）**

* 将原始 `program_output` / `onchain_data_hash` / `onchain_data_size` 转为 `U256`，匹配 Solidity ABI。  
* 直接调用核心合约 `update_state`，无需 Blob，兼容所有以太坊节点。  
* 双阶段日志（starting/completed）+ 交易哈希，方便链上追踪与告警。  
* 返回十六进制格式的交易哈希字符串，供上层系统记录。

### 通过EIP-4844 blob发送

[lib.rs#L213-L316](https://github.com/madara-alliance/madara/blob/b6c0651f98df32ddaa073c41ad95326bbd1ca846/orchestrator/crates/settlement-clients/ethereum/src/lib.rs#L213-L316)


```rust
async fn update_state_with_blobs(
    &self,
    program_output: Vec<[u8; 32]>,
    state_diff: Vec<Vec<u8>>,
    _nonce: u64,
) -> Result<String> {
    tracing::info!(
        log_type = "starting",
        category = "update_state",
        function_type = "blobs",
        "Updating state with blobs."
    );
    let (sidecar_blobs, sidecar_commitments, sidecar_proofs) = prepare_sidecar(&state_diff, &KZG_SETTINGS).await?;
    let sidecar = BlobTransactionSidecar::new(sidecar_blobs, sidecar_commitments, sidecar_proofs);

    let eip1559_est = self.provider.estimate_eip1559_fees(None).await?;
    let chain_id: u64 = self.provider.get_chain_id().await?.to_string().parse()?;

    let max_fee_per_blob_gas: u128 = self.provider.get_blob_base_fee().await?.to_string().parse()?;

    // calculating y_0 point
    let y_0 = Bytes32::from(
        convert_stark_bigint_to_u256(
            bytes_be_to_u128(&program_output[Y_LOW_POINT_OFFSET]),
            bytes_be_to_u128(&program_output[Y_HIGH_POINT_OFFSET]),
        )
        .to_be_bytes(),
    );

    // x_0_value : program_output[10]
    // Updated with starknet 0.13.2 spec
    let x_0_point = Bytes32::from_bytes(program_output[X_0_POINT_OFFSET].as_slice())
        .wrap_err("Failed to get x_0 point params")?;

    let kzg_proof = Self::build_proof(state_diff, x_0_point, y_0).wrap_err("Failed to build KZG proof")?.to_owned();

    let input_bytes = get_input_data_for_eip_4844(program_output, kzg_proof)?;

    let nonce = self.provider.get_transaction_count(self.wallet_address).await?.to_string().parse()?;

    // add a safety margin to the gas price to handle fluctuations
    let add_safety_margin = |n: u128, div_factor: u128| n + n / div_factor;

    let max_fee_per_gas: u128 = eip1559_est.max_fee_per_gas.to_string().parse()?;
    let max_priority_fee_per_gas: u128 = eip1559_est.max_priority_fee_per_gas.to_string().parse()?;

    let tx: TxEip4844 = TxEip4844 {
        chain_id,
        nonce,
        // we noticed Starknet uses the same limit on mainnet
        // https://etherscan.io/tx/0x8a58b936faaefb63ee1371991337ae3b99d74cb3504d73868615bf21fa2f25a1
        gas_limit: 5_500_000,
        max_fee_per_gas: add_safety_margin(max_fee_per_gas, 5),
        max_priority_fee_per_gas: add_safety_margin(max_priority_fee_per_gas, 5),
        to: self.core_contract_client.contract_address(),
        value: U256::from(0),
        access_list: AccessList(vec![]),
        blob_versioned_hashes: sidecar.versioned_hashes().collect(),
        max_fee_per_blob_gas: add_safety_margin(max_fee_per_blob_gas, 5),
        input: Bytes::from(hex::decode(input_bytes)?),
    };

    let tx_sidecar = TxEip4844WithSidecar { tx: tx.clone(), sidecar: sidecar.clone() };

    let mut variant = TxEip4844Variant::from(tx_sidecar);
    let signature = self.wallet.default_signer().sign_transaction(&mut variant).await?;
    let tx_signed = variant.into_signed(signature);
    let tx_envelope: TxEnvelope = tx_signed.into();

    #[cfg(feature = "testing")]
    let pending_transaction = {
        let txn_request = {
            test_config::configure_transaction(self.provider.clone(), tx_envelope, self.impersonate_account).await
        };
        self.provider.send_transaction(txn_request).await?
    };

    #[cfg(not(feature = "testing"))]
    let pending_transaction = {
        let encoded = tx_envelope.encoded_2718();
        self.provider.send_raw_transaction(encoded.as_slice()).await?
    };

    tracing::info!(
        log_type = "completed",
        category = "update_state",
        function_type = "blobs",
        "State updated with blobs."
    );

    log::warn!("⏳ Waiting for txn finality.......");

    let res = self.wait_for_tx_finality(&pending_transaction.tx_hash().to_string()).await?;

    match res {
        Some(_) => {
            log::info!("Txn hash : {:?} Finalized ✅", pending_transaction.tx_hash().to_string());
        }
        None => {
            log::error!("Txn hash not finalised");
        }
    }
    Ok(pending_transaction.tx_hash().to_string())
}
```

**关键点解析（update_state_with_blobs）**

* `prepare_sidecar` 将 `state_diff` 打包为 **Blob + KZG 承诺/证明**，大幅降低 L1 calldata 成本。  
* 动态估算 EIP-1559 与 Blob Gas 基础费率，并加安全裕度，防止价格跳变导致失败。  
* 构造 `TxEip4844`，填充 `blob_versioned_hashes / max_fee_per_blob_gas`，完全兼容 Cancun 升级。  
* 通过默认签名者签名，分别支持 **测试模式**（impersonate 账户）与 **生产模式**（raw tx）。  
* 发送后调用 `wait_for_tx_finality` 等待最终性，确保状态根安全落链。

## 智能合约交互

### 合约接口

[validity_interface.rs:44-60](https://github.com/madara-alliance/madara/blob/2c88da11/orchestrator/crates/settlement-clients/ethereum/src/clients/interfaces/validity_interface.rs#L44-L60)

```solidity
function updateState(uint256[] calldata programOutput, uint256 onchainDataHash, uint256 onchainDataSize) external;
function updateStateKzgDA(uint256[] calldata programOutput, bytes[] calldata kzgProofs) external;
```

**关键点解析（合约接口）**

* `updateState`：Calldata 路径，额外接收 `onchainDataHash / Size` 用于链上重放验证。  
* `updateStateKzgDA`：Blob 路径，接收 `kzgProofs` 数组，保证数据可用性并绑定 Blob 承诺。  

### 客户端实现

[validity_interface.rs:70-140](https://github.com/madara-alliance/madara/blob/2c88da11/orchestrator/crates/settlement-clients/ethereum/src/clients/interfaces/validity_interface.rs#L70-L140)

```rust
async fn update_state(
    &self,
    program_output: Vec<U256>,
    onchain_data_hash: U256,
    onchain_data_size: U256,
) -> Result<TransactionReceipt, StarknetValidityContractError> {
    let base_fee = self.as_ref().provider().as_ref().get_gas_price().await?;
    let from_address = self.as_ref().provider().as_ref().get_accounts().await?[0];
    let gas = self
        .as_ref()
        .updateState(program_output.clone(), onchain_data_hash, onchain_data_size)
        .from(from_address)
        .estimate_gas()
        .await?;
    let builder = self.as_ref().updateState(program_output, onchain_data_hash, onchain_data_size);
    builder
        .from(from_address)
        .nonce(2)
        .gas(gas)
        .gas_price(base_fee)
        .send()
        .await?
        .get_receipt()
        .await
        .map_err(StarknetValidityContractError::PendingTransactionError)
}

async fn update_state_kzg(
    &self,
    program_output: Vec<U256>,
    kzg_proof: [u8; 48],
) -> Result<TransactionReceipt, StarknetValidityContractError> {
    let base_fee = self.as_ref().provider().as_ref().get_gas_price().await?;
    let from_address = self.as_ref().provider().as_ref().get_accounts().await?[0];
    let proof_vec = vec![Bytes::from(kzg_proof.to_vec())];
    let gas = self
        .as_ref()
        .updateStateKzgDA(program_output.clone(), proof_vec.clone())
        .from(from_address)
        .estimate_gas()
        .await?;
    let builder = self.as_ref().updateStateKzgDA(program_output, proof_vec);
    builder
        .from(from_address)
        .nonce(2)
        .gas(gas)
        .gas_price(base_fee)
        .send()
        .await?
        .get_receipt()
        .await
        .map_err(StarknetValidityContractError::PendingTransactionError)
}
```

**关键点解析（客户端实现）**

* 先执行 `estimate_gas` 获取精确 Gas 预算，再设置 `nonce / gas_price`，避免重放。  
* Calldata 与 Blob 两条路径共用一套 Provider & Signer，便于维护。  
* `update_state_kzg` 将 48 字节 KZG 证明封装为 `Bytes`，并打包进数组以符合合约 ABI。  
* 所有函数返回 `TransactionReceipt`，上层可据此做事件解析与状态确认。

## 总结

Madara通过两种方式将STARK证明和状态差异发送到以太坊：

1. **通过calldata**：将状态数据直接包含在交易的calldata中，调用`updateState`函数。
2. **通过EIP-4844 blob**：使用更高效的数据blob机制，调用`updateStateKzgDA`函数。

这两种方法都确保了安全、可验证的状态传输，同时优化了gas成本。 