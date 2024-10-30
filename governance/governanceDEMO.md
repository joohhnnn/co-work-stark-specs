# StarkNet 治理代码演示（Governance DEMO）

> 该文档在先前治理综述基础上，提供更完整的 **Cairo 1.0** 与 **Python SDK** 示例，涵盖：
>
> 1. 基础提案 / 投票流程
> 2. 委托投票（Delegate Voting）
> 3. Sequencer 质押（Staking）
>
> 代码仅供学习参考，未做详尽优化与安全审计，切勿直接用于生产环境。

---

## 目录

1. Cairo 合约
   * 1.1 `Governance.cairo` – 提案、投票、委托
   * 1.2 `Staking.cairo` – Sequencer 质押
2. Python 交互示例
   * 2.1 创建提案
   * 2.2 委托投票 & 投票
   * 2.3 质押 & 取回

---

## 1. Cairo 合约

### 1.1 `Governance.cairo`
```cairo
%lang starknet
%builtins pedersen range_check

from starkware.cairo.common.uint256 import Uint256, add_uint256
from starkware.cairo.common.math import assert_nn

// ————————————————————————————
// 数据结构
// ————————————————————————————
struct Proposal {
    felt for_votes;
    felt against_votes;
    felt deadline;
    felt executed;
}

@storage_var // 提案存储
func proposals(id: felt) -> Proposal:
end

@storage_var // 账户 → 被委托人
func delegate_of(account: felt) -> felt:
end

@storage_var // 账户可用票数（通常与 ERC20 余额或质押挂钩）
func voting_power(account: felt) -> felt:
end

const VOTING_PERIOD = 60 * 60 * 24 * 3;  // 3 天

// ————————————————————————————
// 内部工具
// ————————————————————————————
// 返回投票权实际持有人（若委托则返回被委托地址）
func _resolve_delegate(addr: felt) -> felt {
    alloc_locals
    let (d) = delegate_of.read(addr)
    if d == 0:
        return (addr)
    else:
        return (d)
    end
}

// ————————————————————————————
// 外部接口
// ————————————————————————————
@external
func set_voting_power{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(who: felt, amount: felt):
    // 演示方便，仅合约 OWNER 可调用；省略权限控制
    voting_power.write(who, amount)
    return ()
end

@external
func delegate{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(to: felt):
    delegate_of.write(contract_caller_address(), to)
    return ()
end

@external
func create_proposal{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(id: felt, ts: felt):
    let (old) = proposals.read(id)
    assert old.deadline = 0  // 未占用
    let p = Proposal(0, 0, ts + VOTING_PERIOD, 0)
    proposals.write(id, p)
    return ()
end

@external
func cast_vote{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(id: felt, support: felt):
    alloc_locals
    let caller = contract_caller_address()
    let (prop) = proposals.read(id)
    assert prop.deadline != 0

    // 获取实际投票地址
    let (delegate_addr) = _resolve_delegate(caller)

    // 读取投票权并置 0（单次投票）
    let (power) = voting_power.read(caller)
    assert power > 0
    voting_power.write(caller, 0)

    if support == 1:
        let new_for = prop.for_votes + power
        proposals.write(id, Proposal(new_for, prop.against_votes, prop.deadline, prop.executed))
    else:
        let new_against = prop.against_votes + power
        proposals.write(id, Proposal(prop.for_votes, new_against, prop.deadline, prop.executed))
    end
    return ()
end

@external
func execute_proposal{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(id: felt, ts: felt):
    alloc_locals
    let (prop) = proposals.read(id)
    assert prop.executed == 0
    assert ts >= prop.deadline
    assert_nn(prop.for_votes - prop.against_votes)  // 多数
    proposals.write(id, Proposal(prop.for_votes, prop.against_votes, prop.deadline, 1))
    // 此处可调用升级逻辑或资金流
    return ()
end
```

要点说明：
1. `delegate_of` 将账户映射到被委托人，实现简单委托；深度委托可递归解析。
2. `voting_power` 直接写入以简化示例，生产中应与 ERC20 Snapshot 或质押绑定。
3. 投票后重置 `voting_power`，避免重复计票。

---

### 1.2 `Staking.cairo`（Sequencer 质押）
```cairo
%lang starknet
%builtins pedersen range_check

@storage_var
func stakes(sequencer: felt) -> felt:
end

@storage_var
func total_staked() -> felt:
end

// 质押函数（附带质押金额）
@external
func stake{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(amount: felt):
    alloc_locals
    let sender = contract_caller_address()
    let (old) = stakes.read(sender)
    stakes.write(sender, old + amount)
    let (tot) = total_staked.read()
    total_staked.write(tot + amount)
    return ()
end

// 提取质押
@external
func unstake{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(amount: felt):
    alloc_locals
    let sender = contract_caller_address()
    let (old) = stakes.read(sender)
    assert old >= amount
    stakes.write(sender, old - amount)
    let (tot) = total_staked.read()
    total_staked.write(tot - amount)
    return ()
end

// 返回 sequencer 权重（可按质押占比计算）
@view
func weight_of{syscall_ptr: felt*, pedersen_ptr: HashBuiltin*, range_check_ptr}(seq: felt) -> felt:
    let (amt) = stakes.read(seq)
    return (amt)
end
```

实际网络中，Sequencer 选举会结合权重、排名与罚没（slashing）。此处仅给出最小可行质押逻辑。

---

## 2. Python 交互示例（starknet_py）
> 假设已部署上述两个合约，并获得 ABI。

### 2.1 创建提案
```python
from starknet_py.net.account.account_client import AccountClient
from starknet_py.contract import Contract
import time, asyncio

async def create_proposal(account: AccountClient, gov_addr: str, pid: int):
    gov = await Contract.from_address(gov_addr, account)
    tx = await gov.functions["create_proposal"].invoke(pid, int(time.time()), max_fee=1_000_000_000_000_000)
    print("Creating proposal...")
    await tx.wait_for_acceptance()
```

### 2.2 委托 & 投票
```python
aSYNC def delegate_and_vote(account, gov_addr, delegatee: str, pid: int):
    gov = await Contract.from_address(gov_addr, account)

    # 1) 委托
    tx1 = await gov.functions["delegate"].invoke(delegatee, max_fee=5e14)
    await tx1.wait_for_acceptance()

    # 2) 投票（假设您是 delegatee 身份）
    tx2 = await gov.functions["cast_vote"].invoke(pid, 1, max_fee=5e14)
    await tx2.wait_for_acceptance()

    print("vote casted")
```

### 2.3 质押 & 取回
```python
async def stake_seq(account, staking_addr: str, amount: int):
    staking = await Contract.from_address(staking_addr, account)
    tx = await staking.functions["stake"].invoke(amount, max_fee=1e15)
    await tx.wait_for_acceptance()
    print("staked", amount)

async def unstake_seq(account, staking_addr: str, amount: int):
    staking = await Contract.from_address(staking_addr, account)
    tx = await staking.functions["unstake"].invoke(amount, max_fee=1e15)
    await tx.wait_for_acceptance()
    print("unstaked", amount)
```

---

## 结语
本示例演示了 StarkNet 治理的关键要素：
* **提案 / 投票 / 委托**：演示简单委托模型，可扩展为 Snapshot 余额同步等高级特性。
* **Sequencer 质押**：为未来去中心化排序器奠定基础，可衍生权重计算、惩罚机制。

配合上文治理流程与角色介绍，您可以快速搭建本地测试网或测试网合约，体验完整的 StarkNet 去中心化治理流程。 