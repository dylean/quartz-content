---
title: "The Ultimate Web3 Interview Prep Guide: Blockchain, Solidity, and Beyond"
slug: web3-interview-guide
published: 2026-01-10
description: A comprehensive guide to acing your Web3 technical interview, covering blockchain fundamentals, Solidity development, DApp best practices, backend infrastructure, and security.
tags:
  - Web3
  - Interview
  - Solidity
  - Blockchain
category: Web3
draft: false
---

Breaking into Web3 as a developer is an exciting challenge. The interview process often involves questions that span from low-level cryptography to high-level system design. Having been through this process myself, I put together this guide to help you prepare. The goal isn't just to memorize answers, but to understand the *why* behind each concept—that's what separates a good candidate from a great one.

---

## 1. Foundational Blockchain Concepts

### What's the difference between Layer 1 and Layer 2?

This is a classic opener. Here's how I think about it:

* **Layer 1 (L1)** is the base blockchain—Ethereum, Bitcoin, Solana. It's responsible for **security, consensus, and final settlement**. The trade-off is limited throughput (TPS) and higher gas fees.
* **Layer 2 (L2)** is a scaling solution built *on top* of L1—like Arbitrum or Optimism. L2s execute transactions off-chain and then "roll up" many transactions into a single, compressed proof that gets submitted back to L1.

The key insight: L2s *inherit* L1's security. They don't create their own; they borrow it. This is what makes them different from sidechains.

### Explain Proof of Work (PoW) vs. Proof of Stake (PoS)

* **PoW**: Miners compete using raw computational power (hashing) to solve a puzzle for the right to add a block. It's incredibly secure (requires 51% of network hashrate to attack) but energy-intensive and slow.
* **PoS**: Validators are chosen to propose blocks based on the amount of cryptocurrency they've "staked" (locked up). It's far more energy-efficient and offers faster finality. The potential downside is a "rich get richer" centralization risk. Ethereum moved to PoS with "The Merge."

### What is a Merkle Tree and why does it matter?

A Merkle Tree is a binary hash tree. Leaf nodes are hashes of individual data blocks (like transactions), and parent nodes are hashes of their children, all the way up to a single root hash.

**Why it matters:**

1. **Data Integrity**: If the Merkle Root is unchanged, you can be confident the entire dataset underneath it hasn't been tampered with.
2. **Light Client Verification (SPV)**: A "light node" doesn't need to download an entire block. It can verify a specific transaction's inclusion by just checking the Merkle Proof (a short path of hashes from the transaction to the root).

### How are Private Key, Public Key, and Address related?

This is a one-way street: `Private Key` → `Public Key` → `Address`.

1. **Private Key**: A randomly generated 256-bit integer. This is your secret. *Never share it.*
2. **Public Key**: Derived from the private key using Elliptic Curve Cryptography (ECDSA, specifically the secp256k1 curve).
3. **Address**: The Keccak-256 hash of the public key, truncated to the last 20 bytes, prefixed with `0x`.

Crucially, you cannot reverse this process. You can't derive a private key from an address or public key.

### Hard Fork vs. Soft Fork?

* **Hard Fork**: A protocol change that *loosens* the rules. Old nodes will reject new blocks. If the network doesn't unanimously upgrade, it can split into two separate chains (e.g., Ethereum and Ethereum Classic after the DAO hack).
* **Soft Fork**: A protocol change that *tightens* the rules. New blocks are still valid under the old rules (they're just *more* restrictive). It's backward-compatible.

---

## 2. Smart Contract Development (Solidity)

### What's the difference between `view` and `pure`?

* **`view`**: The function **reads** state variables but does not modify them.
* **`pure`**: The function **neither reads nor modifies** state. It's a pure calculation based only on its inputs.

**Fee note**: Calling `view` or `pure` functions externally (from off-chain) is free. But if another *on-chain* function calls them internally, gas is still consumed.

### Explain `delegatecall` vs. `call`. (This is a favorite!)

This one trips up a lot of people.

* **`call`**: A standard call. Contract A calls Contract B. The code runs in **B's context**: `msg.sender` becomes A, and any state changes happen to **B's storage**.
* **`delegatecall`**: A delegate call. Contract A `delegatecall`s Contract B. The code is from B, but it runs in **A's context**: `msg.sender` remains the original caller, and state changes happen to **A's storage**.

**Why it matters**: `delegatecall` is the magic behind upgradeable contracts. You have a Proxy contract (which holds all the data) and a Logic contract (which holds the code). When a user interacts with the Proxy, it `delegatecall`s the Logic contract. To upgrade, you just point the Proxy to a new Logic contract—the data stays put.

### How do you optimize for gas?

1. **Minimize Storage Writes**: `SSTORE` is the most expensive opcode. Use `memory` or `calldata` for temporary variables.
2. **Variable Packing**: Multiple smaller variables (e.g., `uint64`, `uint128`, `bool`) can fit into a single 256-bit storage slot if declared consecutively. The compiler will pack them.
3. **Other Tips**: Cache storage reads in a local variable if you'll use them multiple times in a loop. Use `unchecked { ... }` for arithmetic where overflow is impossible.

### What is a Reentrancy Attack and how do you prevent it?

**The Attack**: A malicious contract calls your `withdraw()` function. Your contract sends it ETH. Before your contract updates the user's balance to zero, the attacker's `receive()` or `fallback()` function is triggered, which immediately calls `withdraw()` *again*. Since the balance wasn't updated yet, the attacker drains funds repeatedly.

**Prevention**:

1. **Checks-Effects-Interactions Pattern**: Do all your state updates (Effects) *before* making any external calls (Interactions).

    ```solidity
    // Good: Update balance first, then send
    balances[msg.sender] = 0; // Effect
    (bool success, ) = msg.sender.call{value: amount}(""); // Interaction
    ```

2. **Reentrancy Guard**: Use OpenZeppelin's `ReentrancyGuard` and the `nonReentrant` modifier. It uses a mutex lock to prevent a function from being called while it's already executing.

### How do upgradeable contracts work?

Using the **Proxy Pattern**:

1. **Proxy Contract**: This is what users interact with. It holds all the storage (data).
2. **Logic Contract**: This contains the business logic. It's stateless.

When the Proxy receives a call, it uses `delegatecall` to execute the Logic contract's code in its own context. To upgrade, you deploy a new Logic contract and update the Proxy to point to it. The data in the Proxy is preserved.

---

## 3. Frontend and DApp Interaction

### How do you detect wallet connection/disconnection?

Using the `window.ethereum` provider API:

* **`accountsChanged`**: Fired when the user switches accounts or disconnects.
* **`chainChanged`**: Fired when the user switches networks. Best practice is to reload the page when this happens to avoid state inconsistencies.

### What's the difference between Signing and a Transaction?

* **Signing**: An off-chain operation. You use your private key to create a cryptographic signature for a piece of data. It's **free** (no gas). Common uses: "Sign-in with Ethereum" (SIWE), EIP-712 Permit signatures.
* **Transaction**: An on-chain operation. Data is broadcast to the network, included in a block, and changes the blockchain's state. It **costs gas**.

### What if your RPC node goes down?

Build for resilience.

* Configure a **list of RPC providers** (e.g., Infura, Alchemy, QuickNode).
* Implement a **fallback/failover mechanism**: If the primary provider times out or returns an error, automatically switch to a backup.

### How do you handle transaction "finality"?

Never show a "Success!" message just because a transaction was sent. Here's a better UI flow:

1. **Pending**: Transaction submitted, but not yet in a block. (Show a spinner)
2. **Included**: Transaction is in a block. (Acknowledge, but warn it could still revert due to a reorg)
3. **Finalized**: Enough block confirmations have passed (e.g., 12 for Ethereum mainnet), or the L2 batch has been submitted to L1. (Now you can confidently confirm success)

---

## 4. Backend and Infrastructure

### How do you index blockchain data? The Graph vs. Self-Hosted?

* **The Problem**: On-chain data is optimized for writing, not querying. You can't just do `SELECT * FROM transfers WHERE user = X`.
* **The Graph**: A decentralized indexing protocol. You write a `subgraph.yaml` to define what events to listen for. It's convenient but has some latency and relies on a third-party network.
* **Self-Hosted Indexer (Go/Rust)**: You run your own service that scans blocks via RPC (`eth_getBlock`), parses logs, and stores them in a traditional database (PostgreSQL, etc.). This gives you full control over real-time updates and complex queries—essential for enterprise-grade applications.

### How do Ethereum Events/Logs work?

When a contract `emit`s an event, the data is stored in the block's `ReceiptsRoot` via a Bloom Filter.

* **Topics**: Up to 3 parameters can be marked as `indexed`. These go into the `topics` array and can be efficiently searched.
* **Data**: Non-indexed parameters are ABI-encoded and stored in the `data` field. You can't search on this directly; you have to decode it after fetching.

### What is MEV and what is Flashbots?

* **MEV (Maximal Extractable Value)**: The profit that block producers (miners/validators) can extract by reordering, inserting, or censoring transactions within a block. A classic example is "front-running" a large DEX trade.
* **Flashbots**: A private transaction submission layer. Instead of broadcasting your transaction to the public mempool (where everyone can see it), you send it directly to a block builder with a "tip." This prevents other searchers from seeing and front-running your trade.

---

## 5. Security and Advanced Topics

### How do Flash Loans work?

A flash loan allows you to borrow a huge amount of capital with no collateral, as long as you repay it within the *same transaction*.

**Atomic Guarantee**: `Borrow` → `Use Funds (e.g., arbitrage)` → `Repay + Fee`. If the repayment fails at the end of the transaction, the EVM reverts everything, as if the loan never happened.

### ERC-20, ERC-721, ERC-1155: What's the difference?

* **ERC-20**: Fungible tokens. Every token is identical (like USDC or USDT).
* **ERC-721**: Non-Fungible Tokens (NFTs). Each token has a unique `tokenId`.
* **ERC-1155**: The multi-token standard. A single contract can manage both fungible and non-fungible tokens, and it supports **batch transfers**, significantly reducing gas costs.

### Why is on-chain randomness so hard?

Blockchains are deterministic by design. If you use something like a block hash or timestamp as a random seed, a miner/validator can predict or even manipulate it.

**The Solution**: **Chainlink VRF (Verifiable Random Function)**. An off-chain oracle generates a random number along with a cryptographic proof. The on-chain contract verifies the proof, guaranteeing that the randomness hasn't been tampered with.

### What are Zero-Knowledge Proofs (ZKPs)?

**The Concept**: A Prover can convince a Verifier that they know a secret (e.g., a private key or a solution) *without revealing the secret itself*.

**Web3 Applications**:

1. **Privacy**: Tornado Cash used ZKPs to prove you had a valid deposit without revealing *which* deposit was yours.
2. **Scaling (ZK-Rollups)**: L2s like zkSync and StarkNet run computations off-chain and generate a tiny "validity proof." L1 only needs to verify this proof, not re-execute all the transactions.

---

Good luck with your interviews! Feel free to reach out if you want to dive deeper into any of these topics.
