# StarkNet 治理概览

## 1. 治理核心目标

1. **去中心化**：确保 StarkNet 协议的演进不依赖单一主体，而是由持币者、开发者与用户共同决定。
2. **安全与可升级性**：在持续交付性能改进的同时，维持网络及资金安全。
3. **社区自治**：将协议收入与生态基金交由社区管理，支持可持续的开发与生态扩张。

## 2. 关键治理角色

| 角色 | 职责 | 当前实现方式 |
| --- | --- | --- |
| StarkNet 代币持有者（STRK） | 通过 Snapshot / 链上投票决定提案 | 持币即赋权，未来可质押获取投票权倍率 |
| StarkNet Foundation | 提供法律实体、托管启动阶段金库、多签执行 | 含 3 个董事会 + 2 个专门委员会 |
| 开发者社区 | 撰写 StarkNet Improvement Proposal（SNIP）与核心代码 | GitHub PR / Cairo 代码贡献 |
| Security Council | 多签执行紧急升级、漏洞修复 | 规定时间窗内可快速合约替换 |

## 3. 代币经济与投票权

STRK 是 StarkNet 原生代币，主要用途：

* 支付网络交易费用（逐步取代 ETH 手续费）。
* 同质化治理票数，初始 **1 STRK = 1 票**。
* 未来质押机制激励排序器（Sequencer）或数据可用性服务。

> **分配速览**：社区 50.1%，投资机构 17.0%，核心贡献者 32.9%。当下社区配额主要由空投与基金会赠款逐步释放。

## 4. 提案流程（SNIP 流水线）

1. **概念稿 (Draft)**
   * 作者在 Forum 发布 RFC 形式，收集 5~7 天反馈。
2. **正式草案 (SNIP-Draft)**
   * 在 GitHub `starknet-sips` 仓库提交 PR，包含技术规范、动机、向后兼容性分析。
3. **温度检查 (Snapshot)**
   * 以 STRK 余额快照进行非约束性投票，超过 30% 反对或不足 10% 参与率则退回修改。
4. **链上投票**
   * 提案由 Foundation 多签提交 StarkNet 主网治理合约。
   * 投票持续 5–7 天，需满足 **⅔ 赞成 + ≥10% 代币参与** 方可通过。
5. **执行 / 延时**
   * 通过后进入 **Timelock(48h–7d)**，供安全审计与社区复查。
   * 由执行合约自动调用升级或资金流向，多签仅作确认。

## 5. 协议升级与安全机制

* **Sequencer 更替**：引入多运营商竞价或去中心化排序器网络，需要链上启用相关功能标志位（Feature Flag）。
* **Cairo 版本迁移**：通过 `cairo-lang` CompilerID 在合约层面声明兼容，升级提案需附带迁移脚本。
* **紧急暂停 (L1 Escape Hatch)**：Security Council 可在极端事件中暂停桥接或特定系统合约，最长 14 天，并需在结束前提交常规 SNIP 以永久修复。

## 6. 资金库与激励

| 资金池 | 主要用途 | 管理主体 |
| --- | --- | --- |
| Community Treasury | 生态扶持、黑客松、公共物品 | 社区治理（STRK 持有者） |
| Core Dev Pool | 长期核心协议开发、审计 | Foundation 技术委员会 |
| Security Reserve | 应急漏洞赏金、保险 | Security Council & Foundation |

## 7. 年度路线图中的治理里程碑

* 2023-11：StarkNet Foundation 注册并公布章程。
* 2024-Q1：公开 Security Council 成员与多签地址，启动试运行。
* 2024-Q2：首批 SNIP（#1 Sequencer Fee Model）通过并执行。
* 2024-H2：开放排序器许可制度与质押测试网。

## 8. 如何参与治理？

1. **持有 STRK**：关注空投、交易所购买或生态激励计划。
2. **加入讨论**：论坛 `community.starknet.io`、Discord `#governance` 频道与每月社区 Call。
3. **贡献提案**：阅读 `SNIP-0 Meta`，使用模板撰写提案并提交 PR。
4. **技术贡献**：Fork `starknet` & `cairo-lang`，提交性能优化或安全修复。
5. **Grants / Hackathon**：申请基金会 Grants 或参加官方黑客松获取资金支持。

