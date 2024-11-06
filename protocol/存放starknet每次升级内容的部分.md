# Starknet 版本升级历史

## v0.13.2 (2024年8月28日)
- 序列器中的乐观并行化
- 应用递归（"blockpacking"）
- 新的区块哈希定义
- 计算成本降低 50%

## v0.13.0 (2024年1月10日)
- v3 交易支持
- 支持 STRK 支付费用
- Sierra v1.4.0
- 改进 `secp256k1_mul` 和 `secp256r1_mul` 系统调用性能
- 计算成本降低约 50%

## v0.12.3 (2023年11月19日)
- 部分移除 Starknet feeder gateway 支持
- 网关性能优化
- 支持 `secp256r1` 系统调用
- 限制 `__validate__` 和 `DeployAccount` 构造函数的访问

## v0.12.2 (2023年9月4日)
- 启用 P2P 认证
- 解决查询不匹配问题
- 增加每笔交易的最大 Cairo 步骤数

## v0.12.1 (2023年8月21日)
- Mempool 验证
- 包含失败交易
- Keccak 内置函数

## v0.12.0 (2023年7月12日)
- 使用 Rust blockifier 和 LambdaClass 的 Cairo VM
- 支持 Cairo 编译器 2.0.0
- 交易状态变更
- 添加实验性 `get_block_hash` 系统调用

## v0.11.2 (2023年5月31日)
- 升级到 Cairo 1.0 v1.0.0-rc0

## v0.11.0 (2023年3月29日)
- 支持 Cairo 1.0 智能合约
- 引入 Sierra 中间层
- 新的 `declare` 交易版本
- 新的 `replace_class` 系统调用

## v0.10.3 (2022年12月12日)
- 性能优化
- 添加 `starknet-class-hash` 命令
- 不再支持 `deploy` 交易

## v0.10.2 (2022年11月29日)
- 引入序列器并行化
- 添加 `estimate_fee_bulk` 端点
- 改进排序性能
- 更改内置函数比率

## v0.10.1 (2022年10月25日)
- 添加 `DeployAccount` 交易
- 改进 L1 费用计算
- 添加 `uint256_mul_div_mod` 到 `uint256.cairo`

## v0.10.0 (2022年9月5日)
- 引入账户抽象设计的下一步
- 验证/执行分离
- 新的交易版本（版本 1）
- 支持 L1 消息费用

## v0.9.1 (2022年7月20日)
- 添加 `get_block_traces` API
- 在 `get_state_update` 中添加已声明合约列表
- 添加 `starknet_version` 字段
- 支持 `and` 在 if 语句中

## v0.9.0 (2022年6月6日)
- 引入合约类/实例范式
- 添加 `declare` 交易类型
- 重命名 `delegate_call` 为 `library_call`
- 添加 `deploy` 系统调用

## 版本升级总结
这些版本升级展示了 Starknet 的持续发展，从最初的合约部署到现在的复杂功能，包括：
1. 性能优化
2. 费用模型改进
3. 新功能添加
4. 安全性增强
5. 开发者体验提升

每个版本都带来了重要的改进，使 Starknet 成为一个更强大、更高效的 L2 解决方案。
