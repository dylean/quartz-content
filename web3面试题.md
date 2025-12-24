Web3 的技术栈非常广泛，面试题通常取决于你申请的岗位（合约开发、前端交互、后端/基础设施、安全审计等）。

为了涵盖全面，我将面试题分为 **五个核心板块**，从基础概念到高级实战。

---

### 1. 基础概念 (General Concepts)

*适用于所有 Web3 岗位，考察对去中心化核心逻辑的理解。*

* **L1 与 L2 的区别是什么？**
* *考察点：* 主网（Layer 1）的安全性与共识，扩展层（Layer 2）的 Rollups (Optimistic vs ZK) 机制，以及它们如何解决“不可能三角”（Scalability, Security, Decentralization）。


* **请解释工作量证明 (PoW) 与权益证明 (PoS)。**
* *考察点：* 共识机制的原理、能源消耗、抗攻击能力（如 51% 攻击）以及以太坊 Merge 的意义。


* **什么是 Merkle Tree (默克尔树)？它在区块链中有什么作用？**
* *考察点：* 数据结构知识。主要用于轻节点验证（SPV）和快速验证数据完整性。


* **私钥、公钥和地址是如何生成的？**
* *考察点：* 椭圆曲线加密 (ECDSA/EdDSA)、哈希函数 (Keccak-256) 的转换过程。


* **硬分叉 (Hard Fork) 和软分叉 (Soft Fork) 有什么区别？**
* *考察点：* 协议升级的兼容性问题。



---

### 2. 智能合约开发 (Smart Contract / Solidity)

*核心岗位，主要考察 Solidity 语法、EVM 原理及设计模式。*

* **`view` 和 `pure` 关键字的区别是什么？**
* *考察点：* 状态变量的读取权限，以及它们对 Gas 费用的影响（链下调用免费，链上调用依然消耗 Gas）。


* **请解释 `delegatecall` 和 `call` 的区别。**
* *考察点：* 上下文（Context）的保存。`delegatecall` 在调用者的上下文中执行代码（用于代理合约模式），而 `call` 切换到被调用者的上下文。


* **以太坊中的 Gas 机制是如何工作的？如何优化合约的 Gas 消耗？**
* *考察点：* Gas Limit vs Gas Price，EIP-1559。优化技巧：存储优化（Packing slots）、使用 `calldata` 代替 `memory`、避免循环中的昂贵操作。


* **什么是重入攻击 (Reentrancy Attack)？如何防止？**
* *考察点：* 安全意识。DAO 事件回顾。防御：Checks-Effects-Interactions 模式，或使用 `ReentrancyGuard` 修饰符。


* **如何实现智能合约的可升级性 (Upgradability)？**
* *考察点：* 代理模式 (Proxy Pattern)，如 Transparent Proxy 或 UUPS。数据存储布局（Storage Layout）的冲突问题。



---

### 3. 前端与 DApp 交互 (Web3.js / Ethers.js / Viem)

*适用于 Web3 全栈或前端开发，考察与链交互的能力。*

* **如何检测和处理用户的钱包连接与断开？**
* *考察点：* `window.ethereum` 对象，EIP-1193 标准，处理 `accountsChanged` 和 `chainChanged` 事件。


* **解释签名 (Sign) 和 交易 (Transaction) 的区别。**
* *考察点：* 签名是链下行为（免费，用于身份验证或 Permit），交易是链上行为（消耗 Gas，改变状态）。


* **RPC 节点是什么？如果公共 RPC 挂了怎么处理？**
* *考察点：* 对基础设施的理解。解决方案：使用私有节点（Infura/Alchemy/QuickNode）或实现 RPC 轮询/回退机制。


* **在前端如何处理区块链数据的“最终性” (Finality) 问题？**
* *考察点：* 区块确认数，如何处理链重组 (Reorg) 导致的前端数据回滚。



---

### 4. 后端与基础设施 (Backend / DevOps / Go / Rust)

*适用于构建链节点、索引器或高性能后端的岗位。*

* **如何索引区块链数据？(The Graph vs 自建索引)**
* *考察点：* 链上数据查询困难（只能按 Key 查），需要通过监听 Logs/Events 构建关系型数据库。


* **以太坊的事件 (Events/Logs) 机制是如何工作的？Topics 是什么？**
* *考察点：* Bloom Filter（布隆过滤器）用于快速检索日志，Topics 的索引限制（最多 3 个 indexed 参数）。


* **解释一下 MEV (最大可提取价值) 以及 Flashbots。**
* *考察点：* 矿工/验证者如何通过排序交易获利（三明治攻击、抢跑）。私有交易池的作用。


* **如果让你设计一个去中心化交易所 (DEX) 的后端撮合引擎，你会考虑哪些因素？**
* *考察点：* 链上撮合 vs 链下撮合（Orderbook vs AMM），状态同步延迟，并发处理。



---

### 5. Web3 安全与进阶 (Security & Advanced)

*高阶或安全岗位必问。*

* **什么是闪电贷 (Flash Loan)？它常被用于什么场景？**
* *考察点：* 在同一个区块内借出并归还资金的原子性。用途：套利、清算、更换抵押品。


* **解释 ERC-20, ERC-721, ERC-1155 的核心区别。**
* *考察点：* 同质化代币 vs 非同质化代币 vs 混合标准（及批量转账优化）。


* **如何生成随机数？为什么在链上生成真随机数很难？**
* *考察点：* 区块数据（时间戳、难度）是可预测的。解决方案：Chainlink VRF (Verifiable Random Function)。


* **Zero-Knowledge Proofs (零知识证明) 的基本原理及其在 Web3 的应用？**
* *考察点：* 证明者向验证者证明“我知道”而不泄露具体内容。应用：隐私交易 (Zcash, Tornado Cash)、扩容 (ZK-Rollups)。



---

### 建议的准备策略

1. **针对性准备：** 如果你面的是合约岗，**Reentrancy** 和 **Gas Optimization** 是必问的；如果面的是后端，**事件监听** 和 **节点交互** 是重点。
2. **实战演示：** Web3 面试非常看重 GitHub 或者是链上的实战记录。
3. **关注新趋势：** 稍微了解一下 Account Abstraction (ERC-4337) 或 Modular Blockchain (模块化区块链) 会是加分项。

**你想针对其中某个特定的技术栈（比如 Golang 后端 或 Solidity 合约）进行更深度的模拟面试吗？**