---
title: "Solana Bootcamp Lesson 4: Solana's Vision & Building a Custom Token from Scratch"
slug: solana-bootcamp-lesson-4
published: 2026-01-14
description: A comprehensive look at Solana's role in the Internet Capital Market vision, followed by a hands-on guide to building a custom token program (Thai Zhu Coin) from scratch in Rust.
tags:
  - Solana
  - Rust
  - DeFi
  - RWA
category: Web3
draft: false
---

This final session of the bootcamp brought together everything we've learned. We started with a sweeping vision of how Solana is positioning itself as the backbone of a new global Internet Capital Market. Then, we put theory into practice by writing a complete, custom token program from scratch—no SPL Token library, just raw Rust and the `solana-program` crate.

---

## Part 1: Solana and the Internet Capital Market

### The Problem with Traditional Finance

The current financial system, with its paper-based origins and siloed databases, is fundamentally incompatible with the speed and global nature of the modern internet. Cross-border payments take days, settlement involves layers of intermediaries, and fees are opaque. The core issue? "Not your keys, not your money."

### The Vision: A Unified Global Ledger

The **Internet Capital Market** is the idea of a single, permissionless, global ledger for all financial assets. It's more efficient, more transparent, and more accessible. And it's powered by blockchain technology.

A compelling angle discussed was the convergence of **AI and Crypto**. As AI agents become more sophisticated and begin managing assets on our behalf, they will need a protocol they can "understand" to transact. Blockchain provides exactly that: a machine-readable, trustless protocol for value transfer. Imagine AI agents settling invoices or managing supply chain payments directly on Solana via protocols like X 402.

### Why Solana? The L1 Advantage

A key argument for Solana over Layer 2 solutions is **unified liquidity**.

| Feature | Layer 1 (Solana) | Layer 2 (Ethereum Ecosystem) |
| :--- | :--- | :--- |
| **Liquidity** | **Unified and global** on a single network. | **Fragmented** across dozens of L2s, requiring bridges. |
| **Composability** | Seamless, atomic transactions across all protocols. | Bridging is complex, slow, and a security risk. |
| **Performance** | High throughput, sub-second finality. | Limited by L1 congestion and sequencer centralization. |

Solana has also proven itself under pressure. Even during extreme events like high-profile token launches, the network has remained stable. This battle-tested reliability, combined with its focus on real protocol revenue (not just fundraising narratives), makes it a compelling infrastructure choice.

### The Three Pillars of an Internet Capital Market

1. **Payments**: Stablecoins like PYUSD (PayPal) enable instant, low-cost global transactions. Major players like Visa and Stripe are already integrating Solana for settlement.
2. **Trading & RWA**: Real-world assets are being tokenized on Solana. Platforms like Xstock allow users to trade tokenized US equities (NVDA, TSLA) 24/7. These tokens can then be used as collateral in DeFi protocols, earning "dividend + DeFi yield." Over $1 billion in RWA is now on Solana, with institutions like BlackRock and Franklin Templeton participating.
3. **Fundraising**: The line between traditional IPOs and token launches is blurring. Projects can raise capital directly from a global pool of investors without needing investment banks.

---

## Part 2: Building a Custom Token Program from Scratch

Forget the SPL Token library. In this section, we build a simple token called **Thai Zhu Coin** from the ground up to understand exactly how tokenization works at the lowest level.

### Fixing a Critical Security Bug

First, a callback to our previous lesson. Our "Simple Chain Storage" program had a severe vulnerability: it calculated the PDA but **never verified** that the account passed by the user actually matched that PDA.

**The Vulnerability**: An attacker could pass in *any* account as the `pda_account`, potentially overwriting other users' data.

**The Fix**: Always assert that the derived PDA matches the account provided.

```rust
// 1. Calculate the expected PDA
let (expected_pda, _bump_seed) = Pubkey::find_program_address(
    &[b"my_seed", user_account.key.as_ref()],
    program_id
);

// 2. Verify the account passed in matches the expected PDA
assert_eq!(
    pda_account.key,
    &expected_pda,
    "PDA account mismatch!"
);
```

**Design Philosophy**: In Solana, you don't need to explicitly store an `owner` field. If you verify that a PDA is derived from a user's public key, *and* that user has signed the transaction, you have cryptographically proven ownership.

### Blockchain as a State Machine

Before coding, let's understand the theory.

* **UTXO Model (Bitcoin)**: The chain validates *state transitions*. "Can I go from State A (crouching) to State B (2 meters in the air)?" It doesn't compute the path.
* **Account Model (Solana/Ethereum)**: The chain executes an *action* on a state. "I am crouching. I perform a 'jump' action. The VM computes my new state: 2 meters in the air."

### Designing the Thai Zhu Coin

**Goal**: A minimal, custom token where we implement `mint` and `transfer` instructions ourselves.

**Account Data**:
Each user has a PDA storing a single `u64` balance (8 bytes, Big Endian).

```rust
pub struct ThaiZhuAccount {
    pub balance: u64,
}
```

**Instruction Data** (9 bytes total):

| Offset | Field | Type | Description |
| :--- | :--- | :--- | :--- |
| `0` | Tag | `u8` | `0` = Mint, `1` = Transfer |
| `1..9` | Amount | `u64` | Big Endian amount |

### Defining Account Inputs

Solana requires you to explicitly list all accounts an instruction will interact with.

**Mint Instruction (Create new tokens)**

| # | Account | Signer? | Writable? | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| 0 | Mint Authority | ✅ | ❌ | Must be a hardcoded admin (e.g., "Ada"). |
| 1 | Token Account | ❌ | ✅ | The PDA that receives the new tokens. |
| 2 | System Program | ❌ | ❌ | Used to create the account if needed. |
| 3 | Rent Sysvar | ❌ | ❌ | To check rent exemption. |

**Transfer Instruction**

| # | Account | Signer? | Writable? | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| 0 | From Signer | ✅ | ❌ | The sender's wallet; authorizes the transfer. |
| 1 | From PDA | ❌ | ✅ | The sender's token balance account. |
| 2 | To PDA | ❌ | ✅ | The recipient's token balance account. |
| 3 | To Wallet | ❌ | ❌ | The recipient's main wallet (used to derive To PDA). |
| 4 | System Program | ❌ | ❌ | Used to create the recipient's PDA if needed. |
| 5 | Rent Sysvar | ❌ | ❌ | |

### Key Implementation Details

**1. Auto-Initialization (Better UX)**

When transferring tokens, if the recipient doesn't have a token account yet, our program will **automatically create one**. The sender pays the rent. This removes a friction point where users would otherwise need to "register" before they could receive funds.

**2. Safe Math (Prevent Exploits)**

Arithmetic in crypto is dangerous. An integer underflow (e.g., `50 - 51`) on an unsigned integer can wrap around to a massive number, effectively printing money for an attacker. Rust's `checked_sub` and `checked_add` return `None` on overflow/underflow, letting us handle it gracefully.

```rust
// ❌ DANGEROUS: Wraparound vulnerability
// from.balance -= amount;

// ✅ SAFE: Returns an error if underflow would occur
from.balance = from.balance
    .checked_sub(amount)
    .ok_or(ProgramError::InsufficientFunds)?;

to.balance = to.balance
    .checked_add(amount)
    .ok_or(ProgramError::Overflow)?;
```

**Atomicity**: Solana instructions are atomic. If `checked_add` fails after `checked_sub` has already happened, the entire transaction reverts. The state is never corrupted.

---

## Q&A Highlights

> **Q: Why are `solana-program` builds so fragile?**
>
> The dependency tree is complex. Minor version mismatches cause cascading build failures. **Pinocchio** is a new, leaner library being developed to replace `solana-program`. It compiles smaller binaries (saving on deployment SOL costs) and has fewer dependencies.

> **Q: What if my transaction is too big?**
>
> Solana transactions are limited to **~1232 bytes** due to MTU constraints. Complex DeFi transactions with dozens of accounts can easily exceed this. The solution is **Address Lookup Tables (ALT)**. You store a list of frequently used addresses on-chain once, and then in your transaction, you reference them by a 1-byte index instead of a 32-byte public key.

> **Q: How does Solana handle concurrent transactions?**
>
> Solana's **Sealevel runtime** is massively parallel. It looks at the accounts each transaction declares:
>
> * If two transactions touch the same **writable** account, they are serialized (queued).
> * If their accounts don't overlap, they run in **parallel**.
>
> This means `A -> C` and `B -> C` transfers will be serialized because both write to C. There are **no race conditions**.

---

## Key Takeaways

* **Solana's ambition is to be the unified ledger for a global Internet Capital Market**, encompassing payments, trading, and fundraising.
* **L1 unified liquidity is a core advantage** over fragmented L2 ecosystems.
* **Always verify PDAs.** Calculating a PDA and verifying the passed account are two different things.
* **Use `checked_math`.** Underflow/overflow exploits are common and devastating.
* **Understand account inputs.** Explicitly defining signers and writable accounts is fundamental to Solana security.
* **Explore Pinocchio and ALTs.** These are practical tools for building efficient, real-world applications.

Congratulations on completing this bootcamp series! You now have a solid foundation in both the vision and the technical reality of building on Solana.
