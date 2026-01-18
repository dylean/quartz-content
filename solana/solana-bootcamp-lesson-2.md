---
title: "Solana Bootcamp Lesson 2: Building a Sniper Bot & Deep Dive into SPL Tokens"
slug: solana-bootcamp-lesson-2
published: 2026-01-08
description: A comprehensive guide to Solana sniper bot architecture, covering Jito bundles, Websocket event listening, honeypot detection, and a hands-on walkthrough of SPL Token creation.
tags:
  - Solana
  - Sniper_Bot
  - Jito
  - SPL_Token
  - Web3_js
category: Web3
draft: false
---

In this session, we explored two fascinating topics at opposite ends of the Solana development spectrum: the high-stakes world of sniper bots and the fundamental architecture of the SPL Token standard. Whether you're here to understand how the fastest traders operate or to learn how to mint your own token, this post has you covered.

---

## Part 1: The Anatomy of a Solana Sniper Bot

### Why Sniper Bots Dominate

When a new token is listed on a decentralized exchange (DEX) like Raydium, its price can change dramatically within seconds. Manual trading—opening Twitter, clicking a link, connecting a wallet, and signing a transaction—takes 5-15 seconds. On Solana, where blocks are produced every ~400 milliseconds, that's an eternity. A sniper bot can:

* **React in Milliseconds**: It listens directly to the blockchain for new token events.
* **Execute on Block 0**: It can submit a buy order in the very first block after a liquidity pool is created.
* **Protect Against MEV**: By using private transaction channels like Jito, it avoids being front-run or "sandwiched" by other bots.

### Sniping Strategies: A Risk-Reward Spectrum

Different strategies target different moments in a token's lifecycle, each with its own risk profile.

| Strategy | Trigger Signal | How It Works | Pros | Cons |
| :--- | :--- | :--- | :--- | :--- |
| **Liquidity Sniping** | `InitializePool` log | Listens for a DEX pool creation event and buys instantly. | Lowest possible entry price. | **Extremely High Risk**: Prone to rug pulls and honeypots. Requires robust safety checks. |
| **Blind Sniping** | `Create` instruction | Targets "internal" launch platforms (like Pump.fun), buying at the moment of creation, even before DEX listing. | Fastest entry. | **99.9% of tokens go to zero**. Essentially a lottery. Best paired with "smart money" tracking. |
| **Authority Sniping** | `SetAuthority(None)` or `Burn` | Waits for the project team to **renounce mint authority** or **burn LP tokens**, confirming it's not a scam, before buying. | **Much Safer**: Reduces rug pull risk significantly. | Entry price is higher, reducing potential profit. Best for larger, more conservative capital. |

### Solana's Network: No Mempool?

A key difference between Solana and Ethereum is the absence of a public mempool. On Ethereum, pending transactions are visible to everyone, making it easy for searchers to identify and front-run profitable trades.

Solana uses **Gulf Stream**: transactions are forwarded directly to the currently scheduled **Leader** validator. This makes traditional front-running harder, but it also means transactions can be "lost" (due to UDP packet loss) and the network is susceptible to spam attacks.

**Jito: Solana's MEV Solution**

Jito provides a private, reliable channel for transaction submission. Think of it as a "VIP lane."

* **Bundles**: Multiple transactions are grouped into a bundle that executes atomically. Either all succeed, or all fail. This is critical for strategies that involve both a buy and a sell.
* **No Land, No Pay**: You only pay the Jito "tip" if your transaction is actually included in a block.
* **Privacy**: Your transactions are not visible to other searchers in the public network, preventing sandwich attacks.

### The Speed Tech Stack

Building a competitive bot requires the right tools:

1. **Websockets, Not Polling**: Forget `while(true)` loops hitting an HTTP endpoint. Use `connection.onLogs()` to subscribe to real-time events. Set commitment to `processed` for minimum latency.
2. **Avoid the Double Hop**: A common problem is that the initial Websocket log doesn't contain all the data (like the token's Mint address). You then have to make a second HTTP call (`getTransaction`), adding latency. The solution is to use a **Geyser-enabled RPC node** that streams full transaction data via gRPC.
3. **Dynamic Priority Fees**: To ensure your transaction lands, you must calculate a competitive priority fee. A common approach is to query the median fee of recent blocks and multiply it (e.g., 1.5x - 2x).

### The Security Checklist (Before You Pull the Trigger)

A few milliseconds spent on safety checks can save you from losing everything.

* [ ] **Mint Authority Revoked**: Is `mint_authority` set to `None`? If not, the team can print unlimited tokens.
* [ ] **Freeze Authority Revoked**: Is `freeze_authority` set to `None`? If not, they can freeze your tokens, making them unsellable (a "honeypot").
* [ ] **LP Burned or Locked**: Are the Liquidity Provider tokens burned or in a time-lock contract? If they're in the team's wallet, they can pull all liquidity.
* [ ] **Holder Distribution**: Are the top 10 wallets holding >30% of supply? Are they funded from a common source? This indicates "insider" or "sniper" wallets.
* [ ] **Simulate the Sell**: Use `simulateTransaction` to test a sell. If the simulation fails, it's a honeypot. **Never buy.**

---

## Part 2: The Solana Token Architecture

### The Account Model: Everything is an Account

Solana's design philosophy is that "everything is an account." All data on the chain resides in accounts. A program (smart contract) is an account. A wallet is an account. A token balance is... you guessed it, an account.

| Field | Type | Description |
| :--- | :--- | :--- |
| `Pubkey` | Address | The unique 32-byte identifier. |
| `Lamports` | `u64` | SOL balance in lamports ($10^9$ lamports = 1 SOL). |
| `Data` | `Vec<u8>` | Arbitrary binary data (e.g., token balance, NFT metadata). |
| `Owner` | `Pubkey` | The program that can modify this account's data. |
| `Executable` | `bool` | If `true`, this account contains a runnable program. |

### The Three Pillars of an SPL Token

Holding a token on Solana involves three distinct account types:

1. **Mint Account**: This is the token's identity card. It defines the token's `decimals`, `supply`, `mint_authority` (who can create more), and `freeze_authority` (who can freeze balances).
2. **Associated Token Account (ATA)**: This is where your balance actually lives. It's a unique account derived from your wallet address and the Mint address. Formula: `ATA = PDA(User_Wallet, Mint_Address, Token_Program_ID)`.
3. **Metadata Account**: Provided by the Metaplex standard, this account stores human-readable information like the token's `name`, `symbol`, and `uri` (a link to its image/JSON metadata).

### Hands-On: Creating a Token

**Method A: Using the CLI**

The `spl-token` CLI is perfect for quick experiments.

```bash
# 1. Create a new token (Mint Account)
spl-token create-token
# Output: Creating token <YOUR_MINT_ADDRESS>

# 2. Create an Associated Token Account to hold your new token
spl-token create-account <YOUR_MINT_ADDRESS>

# 3. Mint 1000 tokens to your ATA
spl-token mint <YOUR_MINT_ADDRESS> 1000
```

**Method B: Using Web3.js / spl-token Library**

For DApp integration, you'll use the `@solana/spl-token` library.

```typescript
import { 
    createMint, 
    getOrCreateAssociatedTokenAccount, 
    mintTo 
} from '@solana/spl-token';

// 1. Create a new Mint (the "printing press")
const mint = await createMint(
    connection,
    payer,           // Wallet paying for account creation fees
    mintAuthority,   // Pubkey allowed to mint new tokens
    freezeAuthority, // Pubkey allowed to freeze accounts (or null)
    9                // Decimals (9 is standard for most tokens)
);

// 2. Get or create the recipient's ATA
// This will automatically create the ATA if it doesn't exist,
// with the `payer` covering the rent cost.
const tokenAccount = await getOrCreateAssociatedTokenAccount(
    connection,
    payer,
    mint,
    recipientWallet.publicKey
);

// 3. Mint tokens to the ATA
await mintTo(
    connection,
    payer,
    mint,
    tokenAccount.address,
    mintAuthority,              // The authority that's allowed to mint
    1_000_000_000               // Amount (1000 tokens * 10^9 decimals)
);
```

### Token-2022: The Next Generation

The original SPL Token Program had limitations. Token-2022 (also called Token Extensions) addresses these with native support for:

* **Transfer Tax**: Automatically deduct a fee on every transfer.
* **Interest Bearing Tokens**: Tokens that accrue value over time.
* **Non-Transferable Tokens (Soulbound)**: Perfect for credentials or achievements.
* **Confidential Transfers**: Hide the transfer amount using zero-knowledge proofs.
* **Embedded Metadata**: Store name, symbol, and URI directly in the token account, removing the need for a separate Metaplex Metadata Account.

---

## Key Takeaways

* **Sniper bots thrive on speed and information asymmetry.** They use Websockets, Jito bundles, and Geyser streams to react in milliseconds.
* **Security is non-negotiable.** Check authorities, simulate sells, and analyze holder distribution before aping in.
* **Solana's Account Model is unique.** State is stored in accounts, not in contracts. Programs are stateless logic.
* **Token creation is easy.** Use the native SPL Token Program; no contract deployment required.
* **Token-2022 is the future.** It offers powerful new features like transfer taxes and confidential transfers out of the box.
