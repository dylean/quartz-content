---
title: "Solana Bootcamp Lesson 3: The Internet Capital Market & Building On-Chain Storage"
slug: solana-bootcamp-lesson-3
published: 2026-01-10
description: Exploring the vision of an Internet Capital Market (Stablecoins, RWA, Payments) and a hands-on Rust tutorial for building a simple on-chain data storage program on Solana.
tags:
  - Solana
  - Rust
  - SBF
  - Chain_Storage
  - RWA
category: Web3
draft: false
---

This session was a tale of two worlds: a high-level vision for the future of global finance and a deep dive into the low-level mechanics of Solana smart contract development. We explored how blockchain technology is poised to revolutionize capital markets and then got our hands dirty writing Rust code for a simple on-chain storage program.

---

## Part 1: The Vision of an Internet Capital Market

### The Evolution of Finance on the Blockchain

The crypto industry has gone through distinct phases:

1. **2009 - Bitcoin**: Introduced peer-to-peer payments and the idea of disintermediation.
2. **2020 - DeFi Summer**: The explosion of decentralized applications like Uniswap established Ethereum as the "world computer."
3. **Now - The Internet Capital Market**: The focus is shifting from pure decentralization to **compliance** and **enterprise adoption**. The goal is to bring the entire global capital market onto a unified, transparent, and efficient blockchain network.

This new paradigm rests on **three pillars**: **Stablecoins**, **Real World Assets (RWA)**, and **Payments**.

### Stablecoins: The Foundation

Stablecoins are the bridge between traditional finance and the on-chain world. There are two primary models:

| Feature | Issued Stablecoin | Tokenized Deposits |
| :--- | :--- | :--- |
| **Examples** | USDT, USDC, PYUSD | Digital HKD, JPM Coin |
| **Backing** | Fiat/Treasuries held by a custodian | 1:1 anchored to bank deposits |
| **Focus** | Broad circulation, DeFi composability | Interbank settlement, compliance |

Solana's **Token-2022 (Token Extensions)** standard provides native features crucial for compliant stablecoin issuance:

* **Permanent Delegate**: Allows the issuer to retain control (freeze/burn tokens), a legal requirement in many jurisdictions.
* **Transfer Hook**: Executes a custom program on every transfer, enabling real-time KYC/AML checks.
* **Confidential Transfer**: Hides transfer amounts using cryptography, expected to see widespread adoption starting in 2026.

### Real World Assets (RWA): Tokenizing Everything

RWA represents the next frontier: bringing off-chain assets onto the blockchain.

* **From T+2 to T+0**: Traditional settlement takes days. With tokenized assets, settlement is instant and global.
* **The Asset Ladder**:
  * **Level 1 (Stablecoins)**: The earliest form of RWA.
  * **Level 2 (Treasuries/Money Market Funds)**: Major players like **BlackRock** and **Franklin Templeton** are issuing tokenized fund shares on Solana.
  * **Level 3 (Equities)**: Companies like **Xstock** are tokenizing US stocks, enabling 24/7 trading, fractional ownership, and DeFi composability (e.g., using stock tokens as collateral for loans).

---

## Part 2: Building On-Chain Storage with Rust

Now, let's get practical. We're going to build a simple program that allows any user to store arbitrary data on the Solana blockchain, in their own personal account.

### The Solana Account Model Revisited

Remember, in Solana, **everything is an account**. Here's a quick refresher on the structure:

| Field | Type | Description |
| :--- | :--- | :--- |
| `lamports` | `u64` | SOL balance ($10^9$ lamports = 1 SOL). |
| `data` | `Vec<u8>` | The binary data stored in the account. |
| `owner` | `Pubkey` | The program that can modify this account. |
| `executable` | `bool` | `true` if this account contains a runnable program. |
| `rent_epoch` | `u64` | Rent exemption status. |

### SBF and the Program Entrypoint

Solana programs are compiled to **SBF (Solana Bytecode Format)**, a sandboxed, high-performance virtual machine format derived from BPF.

A Solana program doesn't have a `main` function. Instead, it uses the `entrypoint!` macro to define its entry point:

```rust
entrypoint!(process_instruction);

fn process_instruction(
    program_id: &Pubkey,      // This program's ID
    accounts: &[AccountInfo], // All accounts passed to the instruction
    instruction_data: &[u8],  // The instruction's parameters (raw bytes)
) -> ProgramResult {
    // Your business logic here...
    Ok(())
}
```

### Key Concepts for the Storage Program

**1. Program Derived Addresses (PDAs)**

A PDA is an account whose address is deterministically derived from a program ID and a set of "seeds." Think of it like this: your company (the Program) opens a subsidiary bank account (the PDA). The account is technically owned by the company, but you, as an employee, can manage it through official company procedures.

* **Key Property**: A PDA has no private key. It cannot sign transactions. The program must use `invoke_signed` to authorize actions on its behalf.

**2. Rent Exemption**

To prevent state bloat, Solana requires accounts to hold a minimum SOL balance (rent). If an account holds enough SOL to cover two years of rent, it becomes **rent-exempt** and the balance is never debited.

* **Calculation**: `Rent::get()?.minimum_balance(data_length)`

### The Code: Initialize and Write Data

Let's write the logic to create a user's storage account and write initial data to it.

```rust
// 1. Get accounts from the instruction
let account_info_iter = &mut accounts.iter();
let user = next_account_info(account_info_iter)?; // The user (signer, pays fees)
let pda_account = next_account_info(account_info_iter)?; // The storage PDA
let system_program = next_account_info(account_info_iter)?;

// 2. Derive the PDA and verify it matches the one passed in
let (expected_pda, bump_seed) = Pubkey::find_program_address(
    &[b"storage", user.key.as_ref()],
    program_id
);
assert_eq!(pda_account.key, &expected_pda);

// 3. If the PDA doesn't exist yet (has 0 lamports), create it
if pda_account.lamports() == 0 {
    let space = instruction_data.len();
    let rent = Rent::get()?.minimum_balance(space);

    // CPI (Cross-Program Invocation) to the System Program
    invoke_signed(
        &system_instruction::create_account(
            user.key,
            pda_account.key,
            rent,
            space as u64,
            program_id // Set the new account's owner to this program
        ),
        &[user.clone(), pda_account.clone(), system_program.clone()],
        // The seeds used to sign for the PDA
        &[&[b"storage", user.key.as_ref(), &[bump_seed]]],
    )?;
}

// 4. Write the data
let mut data = pda_account.try_borrow_mut_data()?;
data[..].copy_from_slice(instruction_data);
```

### The Code: Update Data with Realloc

What if the user wants to update their data to something longer or shorter?

```rust
let new_len = new_data.len();

// 1. Resize the account's data buffer
pda_account.realloc(new_len, false)?;

// 2. Adjust the rent balance
let required_lamports = Rent::get()?.minimum_balance(new_len);
let current_lamports = pda_account.lamports();

if current_lamports < required_lamports {
    // Data got BIGGER: User needs to pay more rent
    let diff = required_lamports - current_lamports;
    invoke(
        &system_instruction::transfer(user.key, pda_account.key, diff),
        &[user.clone(), pda_account.clone(), system_program.clone()],
    )?;
} else {
    // Data got SMALLER: Refund the excess rent to the user
    let diff = current_lamports - required_lamports;
    // Since our program owns the PDA, we can directly modify its lamports
    **pda_account.try_borrow_mut_lamports()? -= diff;
    **user.try_borrow_mut_lamports()? += diff;
}

// 3. Write the new data
pda_account.try_borrow_mut_data()?.copy_from_slice(new_data);
```

---

## Q&A Highlights

> **Q: Why do I get CORS errors when calling the official RPC from my browser?**
>
> This is a browser security feature. For local development, use a proxy in your bundler (e.g., Vite's `server.proxy`). For production, use a CORS-enabled RPC provider like Helius or QuickNode.

> **Q: Native `solana-program` vs. Anchor Framework?**
>
> * **Native**: Lower level. You manually handle serialization and account validation. Great for learning the fundamentals. Recommended for beginners.
> * **Anchor**: A higher-level framework (think React for Solana). It handles most of the boilerplate (account validation, Borsh serialization, CPIs). **Strongly recommended for production applications.**

> **Q: What is Pinocchio?**
>
> Pinocchio is a new, lightweight library being developed as an alternative to the official `solana-program` crate. It has fewer dependencies, resulting in smaller compiled program sizes (which means lower deployment costs) and faster performance.

---

## Key Takeaways

* **The Internet Capital Market is coming.** Solana is positioned as a key infrastructure layer for compliant stablecoins, tokenized RWA, and global payments.
* **Token-2022 is the compliance layer.** Features like Permanent Delegate and Transfer Hook are essential for regulated assets.
* **PDAs are powerful.** They allow programs to "own" accounts without needing private keys, enabling complex on-chain logic.
* **Rent is real.** Always calculate and handle rent exemption when creating or resizing accounts.
* **Start native, then graduate to Anchor.** Understanding the fundamentals makes you a much better Anchor developer.
