# Seeker Staking Program - Integration Guide

> **Complete documentation for integrating with the Seeker Staking Program on Solana**

Made this doc for anyone who wants to interact with the seeker-staking-vault on their own app since there are no docs out there for it. (thank the AI)

---

## Table of Contents

1. [Overview](#overview)
2. [Program Constants](#program-constants)
3. [Program Derived Addresses (PDAs)](#program-derived-addresses-pdas)
4. [Account Data Structures](#account-data-structures)
5. [Instructions](#instructions)
   - [Stake](#1-stake)
   - [Unstake](#2-unstake)
   - [Cancel Unstake](#3-cancel-unstake)
   - [Withdraw](#4-withdraw)
6. [Working with Shares](#working-with-shares)
7. [Reading On-Chain Data](#reading-on-chain-data)
8. [Complete Transaction Examples](#complete-transaction-examples)
9. [Error Handling](#error-handling)
10. [Best Practices](#best-practices)
11. [Troubleshooting](#troubleshooting)

---

## Overview

The Seeker Staking Program allows users to stake SKR tokens to earn rewards. The program uses a **share-based system** where staked tokens are converted to shares based on the current share price. As rewards accumulate, the share price increases, meaning each share represents more tokens over time.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Shares** | Represent your portion of the staking pool. Calculated as `tokens * 1e9 / sharePrice` |
| **Share Price** | Increases over time as rewards are distributed. Scaled by 1e9 |
| **Cooldown Period** | 48 hours between initiating unstake and being able to withdraw |
| **Guardian Pool** | Delegation pool for guardian rewards. Most users use the default pool |

### Staking Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SEEKER STAKING LIFECYCLE                          │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  WALLET  │  stake  │  STAKED  │ unstake │ COOLDOWN │withdraw │  WALLET  │
  │  (SKR)   │ ──────► │ (shares) │ ──────► │(48 hours)│ ──────► │  (SKR)   │
  └──────────┘         └──────────┘         └──────────┘         └──────────┘
                              │                   │
                              │   cancel_unstake  │
                              │ ◄─────────────────┘
                              │
```

---

## Program Constants

```typescript
import { PublicKey } from '@solana/web3.js';

// Program ID
export const SEEKER_PROGRAM_ID = new PublicKey('SKRskrmtL83pcL4YqLWt6iPefDqwXQWHSw9S9vz94BZ');

// SKR Token
export const SKR_TOKEN_MINT = new PublicKey('SKRbvo6Gf7GondiT3BbTfuRDPqLWei4j2Qy2NPGZhW3');
export const SKR_DECIMALS = 6;

// Pre-computed PDAs (for convenience)
export const STAKE_CONFIG_PDA = new PublicKey('4HQy82s9CHTv1GsYKnANHMiHfhcqesYkK6sB3RDSYyqw');
export const STAKE_VAULT_PDA = new PublicKey('8isViKbwhuhFhsv2t8vaFL74pKCqaFPQXo1KkeQwZbB8');
export const EVENT_AUTHORITY_PDA = new PublicKey('8rUTGg1XoyuvK9G64S7d37m3HtLZH24oPeMmXkpJH8ir');

// Default Guardian Pool (Seeker's main guardian)
export const DEFAULT_GUARDIAN_POOL = new PublicKey('DPJ58trLsF9yPrBa2pk6UaRkvqW8hWUYjawe788WBuqr');

// Timing Constants
export const UNSTAKE_COOLDOWN_SECONDS = 172800; // 48 hours
export const UNSTAKE_COOLDOWN_MS = 48 * 60 * 60 * 1000;
```

---

## Program Derived Addresses (PDAs)

All PDAs are derived using Anchor's standard approach with specific seeds.

### StakeConfig PDA

Global configuration account containing program settings and share price.

```typescript
const [STAKE_CONFIG_PDA] = PublicKey.findProgramAddressSync(
  [Buffer.from('stake_config')],
  SEEKER_PROGRAM_ID
);
// Result: 4HQy82s9CHTv1GsYKnANHMiHfhcqesYkK6sB3RDSYyqw
```

### StakeVault PDA

The token account holding all staked SKR tokens.

```typescript
const [STAKE_VAULT_PDA] = PublicKey.findProgramAddressSync(
  [Buffer.from('stake_vault')],
  SEEKER_PROGRAM_ID
);
// Result: 8isViKbwhuhFhsv2t8vaFL74pKCqaFPQXo1KkeQwZbB8
```

### Event Authority PDA

Used for Anchor event emission via CPI.

```typescript
const [EVENT_AUTHORITY_PDA] = PublicKey.findProgramAddressSync(
  [Buffer.from('__event_authority')],
  SEEKER_PROGRAM_ID
);
// Result: 8rUTGg1XoyuvK9G64S7d37m3HtLZH24oPeMmXkpJH8ir
```

### UserStake PDA

Per-user staking account. Each user has one account per guardian pool.

```typescript
function getUserStakePDA(
  user: PublicKey,
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from('user_stake'),
      STAKE_CONFIG_PDA.toBuffer(),   // 32 bytes
      user.toBuffer(),                // 32 bytes  
      guardianPool.toBuffer(),        // 32 bytes
    ],
    SEEKER_PROGRAM_ID
  );
}
```

---

## Account Data Structures

### UserStake Account (169 bytes)

| Offset | Size | Field | Type | Description |
|--------|------|-------|------|-------------|
| 0 | 8 | discriminator | `[u8; 8]` | `[102, 53, 163, 107, 9, 138, 87, 153]` |
| 8 | 1 | bump | `u8` | PDA bump seed |
| 9 | 32 | stake_config | `Pubkey` | StakeConfig account reference |
| 41 | 32 | user | `Pubkey` | User's wallet address |
| 73 | 32 | guardian_pool | `Pubkey` | Guardian delegation pool |
| 105 | 16 | shares | `u128` | User's share balance |
| 121 | 16 | cost_basis | `u128` | Original token amount staked (raw units) |
| 137 | 16 | cumulative_commission | `u128` | Commission tracking field |
| 153 | 8 | unstaking_amount | `u64` | Tokens pending unstake (raw units) |
| 161 | 8 | unstake_timestamp | `i64` | Unix timestamp when unstake was initiated |

### StakeConfig Account

| Offset | Size | Field | Type | Description |
|--------|------|-------|------|-------------|
| 0 | 8 | discriminator | `[u8; 8]` | `[238, 151, 43, 3, 11, 151, 63, 176]` |
| 8 | 1 | bump | `u8` | PDA bump seed |
| 9 | 32 | authority | `Pubkey` | Admin authority |
| 41 | 32 | mint | `Pubkey` | SKR token mint |
| 73 | 32 | stake_vault | `Pubkey` | Vault PDA |
| 105 | 8 | min_stake_amount | `u64` | Minimum stake requirement (raw units) |
| 113 | 8 | cooldown_seconds | `u64` | Unstake cooldown period |
| 121 | 16 | total_shares | `u128` | Total shares across all users |
| 137 | 16 | share_price | `u128` | Current share price (scaled by 1e9) |

---

## Instructions

### Instruction Discriminators

All instructions use 8-byte Anchor-style discriminators derived from `sha256("global:<instruction_name>")[0..8]`.

| Instruction | Discriminator (hex) | Discriminator (bytes) |
|-------------|--------------------|-----------------------|
| `stake` | `ceb0ca12c8d1b36c` | `[206, 176, 202, 18, 200, 209, 179, 108]` |
| `unstake` | `5a5f6b2acd7c32e1` | `[90, 95, 107, 42, 205, 124, 50, 225]` |
| `cancel_unstake` | `404135e37d9903a7` | `[64, 65, 53, 227, 125, 153, 3, 167]` |
| `withdraw` | `b712469c946da122` | `[183, 18, 70, 156, 148, 109, 161, 34]` |

---

### 1. Stake

Stakes SKR tokens into the program, receiving shares based on current share price.

#### Instruction Data Format

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | discriminator | `[206, 176, 202, 18, 200, 209, 179, 108]` |
| 8 | 8 | amount | `u64` - Amount to stake in raw units (multiply display amount by 10^6) |

**Total: 16 bytes**

#### Required Accounts (12)

| # | Account | Signer | Writable | Description |
|---|---------|--------|----------|-------------|
| 0 | user_stake | ❌ | ✅ | UserStake PDA (derived from user + guardian_pool) |
| 1 | stake_config | ❌ | ✅ | StakeConfig PDA |
| 2 | guardian_pool | ❌ | ✅ | Guardian delegation pool |
| 3 | payer | ✅ | ✅ | Transaction fee payer (signer) |
| 4 | user | ❌ | ❌ | User's wallet (usually same as payer) |
| 5 | user_token_account | ❌ | ✅ | User's SKR Associated Token Account |
| 6 | stake_vault | ❌ | ✅ | Vault token account (STAKE_VAULT_PDA) |
| 7 | mint | ❌ | ❌ | SKR token mint |
| 8 | token_program | ❌ | ❌ | SPL Token Program |
| 9 | system_program | ❌ | ❌ | System Program |
| 10 | event_authority | ❌ | ❌ | Event Authority PDA |
| 11 | program | ❌ | ❌ | Seeker Program ID |

#### TypeScript Implementation

```typescript
import { PublicKey, TransactionInstruction, SystemProgram } from '@solana/web3.js';
import { TOKEN_PROGRAM_ID, getAssociatedTokenAddress } from '@solana/spl-token';

async function buildStakeInstruction(
  payer: PublicKey,
  user: PublicKey,
  amount: bigint, // Raw amount with decimals (e.g., 100 SKR = 100_000_000n)
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): Promise<TransactionInstruction> {
  const [userStakePDA] = getUserStakePDA(user, guardianPool);
  const userTokenAccount = await getAssociatedTokenAddress(SKR_TOKEN_MINT, user);

  // Build instruction data
  const data = Buffer.alloc(16);
  
  // Write discriminator (bytes 0-7)
  const discriminator = Buffer.from([206, 176, 202, 18, 200, 209, 179, 108]);
  discriminator.copy(data, 0);
  
  // Write amount as little-endian u64 (bytes 8-15)
  let val = amount;
  for (let i = 0; i < 8; i++) {
    data[8 + i] = Number(val & BigInt(0xff));
    val = val >> BigInt(8);
  }

  return new TransactionInstruction({
    programId: SEEKER_PROGRAM_ID,
    keys: [
      { pubkey: userStakePDA, isSigner: false, isWritable: true },
      { pubkey: STAKE_CONFIG_PDA, isSigner: false, isWritable: true },
      { pubkey: guardianPool, isSigner: false, isWritable: true },
      { pubkey: payer, isSigner: true, isWritable: true },
      { pubkey: user, isSigner: false, isWritable: false },
      { pubkey: userTokenAccount, isSigner: false, isWritable: true },
      { pubkey: STAKE_VAULT_PDA, isSigner: false, isWritable: true },
      { pubkey: SKR_TOKEN_MINT, isSigner: false, isWritable: false },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: SystemProgram.programId, isSigner: false, isWritable: false },
      { pubkey: EVENT_AUTHORITY_PDA, isSigner: false, isWritable: false },
      { pubkey: SEEKER_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    data,
  });
}
```

---

### 2. Unstake

Initiates unstaking of shares. The corresponding tokens enter a **48-hour cooldown period** before they can be withdrawn.

> ⚠️ **Important**: Unstake takes **shares** as input, not token amount. You must convert tokens to shares using the current share price.

#### Instruction Data Format

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | discriminator | `[90, 95, 107, 42, 205, 124, 50, 225]` |
| 8 | 16 | shares | `u128` - Number of shares to unstake |

**Total: 24 bytes**

#### Required Accounts (8)

| # | Account | Signer | Writable | Description |
|---|---------|--------|----------|-------------|
| 0 | user_stake | ❌ | ✅ | UserStake PDA |
| 1 | stake_config | ❌ | ✅ | StakeConfig PDA |
| 2 | guardian_pool | ❌ | ✅ | Guardian delegation pool |
| 3 | user | ✅ | ❌ | User's wallet (signer, must match UserStake.user) |
| 4 | stake_vault | ❌ | ❌ | Vault token account |
| 5 | mint | ❌ | ❌ | SKR token mint |
| 6 | event_authority | ❌ | ❌ | Event Authority PDA |
| 7 | program | ❌ | ❌ | Seeker Program ID |

#### TypeScript Implementation

```typescript
async function buildUnstakeInstruction(
  user: PublicKey,
  shares: bigint, // Number of shares to unstake
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): Promise<TransactionInstruction> {
  const [userStakePDA] = getUserStakePDA(user, guardianPool);

  // Build instruction data
  const data = Buffer.alloc(24);
  
  // Write discriminator (bytes 0-7)
  const discriminator = Buffer.from([90, 95, 107, 42, 205, 124, 50, 225]);
  discriminator.copy(data, 0);
  
  // Write shares as little-endian u128 (bytes 8-23)
  let val = shares;
  for (let i = 0; i < 16; i++) {
    data[8 + i] = Number(val & BigInt(0xff));
    val = val >> BigInt(8);
  }

  return new TransactionInstruction({
    programId: SEEKER_PROGRAM_ID,
    keys: [
      { pubkey: userStakePDA, isSigner: false, isWritable: true },
      { pubkey: STAKE_CONFIG_PDA, isSigner: false, isWritable: true },
      { pubkey: guardianPool, isSigner: false, isWritable: true },
      { pubkey: user, isSigner: true, isWritable: false },
      { pubkey: STAKE_VAULT_PDA, isSigner: false, isWritable: false },
      { pubkey: SKR_TOKEN_MINT, isSigner: false, isWritable: false },
      { pubkey: EVENT_AUTHORITY_PDA, isSigner: false, isWritable: false },
      { pubkey: SEEKER_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    data,
  });
}
```

---

### 3. Cancel Unstake

Cancels a pending unstake request, returning shares to the user's staked balance.

> 💡 **Note**: Can only be called while in cooldown period, before withdrawal.

#### Instruction Data Format

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | discriminator | `[64, 65, 53, 227, 125, 153, 3, 167]` |

**Total: 8 bytes** (no additional arguments)

#### Required Accounts (7)

| # | Account | Signer | Writable | Description |
|---|---------|--------|----------|-------------|
| 0 | user_stake | ❌ | ✅ | UserStake PDA |
| 1 | stake_config | ❌ | ✅ | StakeConfig PDA |
| 2 | guardian_pool | ❌ | ✅ | Guardian delegation pool |
| 3 | user | ✅ | ❌ | User's wallet (signer) |
| 4 | stake_vault | ❌ | ❌ | Vault token account |
| 5 | event_authority | ❌ | ❌ | Event Authority PDA |
| 6 | program | ❌ | ❌ | Seeker Program ID |

#### TypeScript Implementation

```typescript
async function buildCancelUnstakeInstruction(
  user: PublicKey,
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): Promise<TransactionInstruction> {
  const [userStakePDA] = getUserStakePDA(user, guardianPool);

  // Only discriminator, no arguments
  const data = Buffer.from([64, 65, 53, 227, 125, 153, 3, 167]);

  return new TransactionInstruction({
    programId: SEEKER_PROGRAM_ID,
    keys: [
      { pubkey: userStakePDA, isSigner: false, isWritable: true },
      { pubkey: STAKE_CONFIG_PDA, isSigner: false, isWritable: true },
      { pubkey: guardianPool, isSigner: false, isWritable: true },
      { pubkey: user, isSigner: true, isWritable: false },
      { pubkey: STAKE_VAULT_PDA, isSigner: false, isWritable: false },
      { pubkey: EVENT_AUTHORITY_PDA, isSigner: false, isWritable: false },
      { pubkey: SEEKER_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    data,
  });
}
```

---

### 4. Withdraw

Withdraws tokens after the 48-hour cooldown period has elapsed.

> ⚠️ **Requirement**: `current_time >= unstake_timestamp + cooldown_seconds`

#### Instruction Data Format

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | discriminator | `[183, 18, 70, 156, 148, 109, 161, 34]` |

**Total: 8 bytes** (no additional arguments)

#### Required Accounts (8)

| # | Account | Signer | Writable | Description |
|---|---------|--------|----------|-------------|
| 0 | user_stake | ❌ | ✅ | UserStake PDA |
| 1 | stake_config | ❌ | ✅ | StakeConfig PDA |
| 2 | user | ✅ | ✅ | User's wallet (signer) |
| 3 | stake_vault | ❌ | ✅ | Vault token account |
| 4 | user_token_account | ❌ | ✅ | User's SKR ATA |
| 5 | token_program | ❌ | ❌ | SPL Token Program |
| 6 | event_authority | ❌ | ❌ | Event Authority PDA |
| 7 | program | ❌ | ❌ | Seeker Program ID |

#### TypeScript Implementation

```typescript
async function buildWithdrawInstruction(
  user: PublicKey,
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): Promise<TransactionInstruction> {
  const [userStakePDA] = getUserStakePDA(user, guardianPool);
  const userTokenAccount = await getAssociatedTokenAddress(SKR_TOKEN_MINT, user);

  // Only discriminator, no arguments
  const data = Buffer.from([183, 18, 70, 156, 148, 109, 161, 34]);

  return new TransactionInstruction({
    programId: SEEKER_PROGRAM_ID,
    keys: [
      { pubkey: userStakePDA, isSigner: false, isWritable: true },
      { pubkey: STAKE_CONFIG_PDA, isSigner: false, isWritable: true },
      { pubkey: user, isSigner: true, isWritable: true },
      { pubkey: STAKE_VAULT_PDA, isSigner: false, isWritable: true },
      { pubkey: userTokenAccount, isSigner: false, isWritable: true },
      { pubkey: TOKEN_PROGRAM_ID, isSigner: false, isWritable: false },
      { pubkey: EVENT_AUTHORITY_PDA, isSigner: false, isWritable: false },
      { pubkey: SEEKER_PROGRAM_ID, isSigner: false, isWritable: false },
    ],
    data,
  });
}
```

---

## Working with Shares

The program uses a share-based accounting system. Understanding share calculations is essential for correct integration.

### Share Price Scaling

The share price is stored as a `u128` scaled by **1e9** (1 billion). A share price of `1_000_000_000` means 1 token = 1 share.

### Conversion Functions

```typescript
/**
 * Convert token amount to shares
 * @param tokens - Raw token amount (with decimals)
 * @param sharePrice - Current share price from StakeConfig (scaled by 1e9)
 * @returns Number of shares
 */
function tokensToShares(tokens: bigint, sharePrice: bigint): bigint {
  return (tokens * BigInt(1_000_000_000)) / sharePrice;
}

/**
 * Convert shares to token amount
 * @param shares - Number of shares
 * @param sharePrice - Current share price from StakeConfig (scaled by 1e9)
 * @returns Raw token amount (with decimals)
 */
function sharesToTokens(shares: bigint, sharePrice: bigint): bigint {
  return (shares * sharePrice) / BigInt(1_000_000_000);
}

/**
 * Convert display amount to raw (add decimals)
 * @param amount - Human-readable amount (e.g., 100 for 100 SKR)
 * @returns Raw amount with 6 decimals
 */
function toRawAmount(amount: number): bigint {
  return BigInt(Math.floor(amount * 1_000_000));
}

/**
 * Convert raw amount to display (remove decimals)
 * @param raw - Raw amount with 6 decimals
 * @returns Human-readable amount
 */
function fromRawAmount(raw: bigint): number {
  return Number(raw) / 1_000_000;
}
```

### Example: Unstake 100 SKR

```typescript
async function unstakeByAmount(
  connection: Connection,
  user: PublicKey,
  displayAmount: number // e.g., 100 for 100 SKR
): Promise<TransactionInstruction> {
  // 1. Get current share price
  const sharePrice = await fetchSharePrice(connection);
  
  // 2. Convert display amount to raw tokens
  const rawTokens = toRawAmount(displayAmount); // 100 SKR -> 100_000_000n
  
  // 3. Convert tokens to shares
  const shares = tokensToShares(rawTokens, sharePrice);
  
  // 4. Build unstake instruction with shares
  return buildUnstakeInstruction(user, shares);
}
```

---

## Reading On-Chain Data

### Fetch User's Stake Data

```typescript
import { Connection, PublicKey } from '@solana/web3.js';

interface UserStakeData {
  exists: boolean;
  shares: bigint;
  costBasis: bigint;
  unstakingAmount: bigint;
  unstakeTimestamp: number;
}

async function fetchUserStakeAccount(
  connection: Connection,
  user: PublicKey,
  guardianPool: PublicKey = DEFAULT_GUARDIAN_POOL
): Promise<UserStakeData | null> {
  const [userStakePDA] = getUserStakePDA(user, guardianPool);
  const accountInfo = await connection.getAccountInfo(userStakePDA);
  
  if (!accountInfo || !accountInfo.data) {
    return null;
  }

  const data = accountInfo.data;
  
  // Verify discriminator
  const expectedDisc = Buffer.from([102, 53, 163, 107, 9, 138, 87, 153]);
  if (!expectedDisc.equals(data.slice(0, 8))) {
    return null;
  }

  return {
    exists: true,
    shares: decodeU128(data, 105),
    costBasis: decodeU128(data, 121),
    unstakingAmount: decodeU64(data, 153),
    unstakeTimestamp: Number(decodeI64(data, 161)),
  };
}
```

### Fetch Current Share Price

```typescript
async function fetchSharePrice(connection: Connection): Promise<bigint> {
  const accountInfo = await connection.getAccountInfo(STAKE_CONFIG_PDA);
  
  if (!accountInfo || !accountInfo.data) {
    return BigInt(1_000_000_000); // Default: 1:1 ratio
  }

  // share_price is at offset 137 (u128)
  return decodeU128(accountInfo.data, 137);
}
```

### Helper: Decode Functions (Browser-Compatible)

```typescript
function decodeU64(data: Buffer, offset: number): bigint {
  let result = BigInt(0);
  for (let i = 7; i >= 0; i--) {
    result = (result << BigInt(8)) + BigInt(data[offset + i]);
  }
  return result;
}

function decodeI64(data: Buffer, offset: number): bigint {
  const unsigned = decodeU64(data, offset);
  if (unsigned >= BigInt('0x8000000000000000')) {
    return unsigned - BigInt('0x10000000000000000');
  }
  return unsigned;
}

function decodeU128(data: Buffer, offset: number): bigint {
  const low = decodeU64(data, offset);
  const high = decodeU64(data, offset + 8);
  return low + (high << BigInt(64));
}
```

---

## Complete Transaction Examples

### Full Stake Flow

```typescript
import { Connection, Transaction, PublicKey } from '@solana/web3.js';
import { 
  createAssociatedTokenAccountInstruction, 
  getAssociatedTokenAddress,
  getAccount 
} from '@solana/spl-token';

async function createStakeTransaction(
  connection: Connection,
  owner: PublicKey,
  amountToStake: number // Human-readable (e.g., 100 for 100 SKR)
): Promise<Transaction> {
  const transaction = new Transaction();
  
  // Ensure user has ATA for SKR (create if needed)
  const userTokenAccount = await getAssociatedTokenAddress(SKR_TOKEN_MINT, owner);
  try {
    await getAccount(connection, userTokenAccount);
  } catch {
    transaction.add(
      createAssociatedTokenAccountInstruction(
        owner,           // payer
        userTokenAccount, // ata
        owner,           // owner
        SKR_TOKEN_MINT   // mint
      )
    );
  }

  // Add stake instruction
  const rawAmount = toRawAmount(amountToStake);
  const stakeIx = await buildStakeInstruction(owner, owner, rawAmount);
  transaction.add(stakeIx);

  // Finalize transaction
  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = owner;

  return transaction;
}

// Usage with wallet adapter
async function executeStake(wallet: WalletAdapter, connection: Connection, amount: number) {
  const tx = await createStakeTransaction(connection, wallet.publicKey, amount);
  const signedTx = await wallet.signTransaction(tx);
  const signature = await connection.sendRawTransaction(signedTx.serialize());
  await connection.confirmTransaction(signature, 'confirmed');
  return signature;
}
```

### Full Unstake + Withdraw Flow

```typescript
async function createUnstakeTransaction(
  connection: Connection,
  owner: PublicKey,
  amountToUnstake: number
): Promise<Transaction> {
  // Convert amount to shares
  const sharePrice = await fetchSharePrice(connection);
  const rawAmount = toRawAmount(amountToUnstake);
  const shares = tokensToShares(rawAmount, sharePrice);

  const transaction = new Transaction();
  const unstakeIx = await buildUnstakeInstruction(owner, shares);
  transaction.add(unstakeIx);

  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = owner;

  return transaction;
}

async function createWithdrawTransaction(
  connection: Connection,
  owner: PublicKey
): Promise<Transaction> {
  const transaction = new Transaction();

  // Ensure user has ATA for receiving tokens
  const userTokenAccount = await getAssociatedTokenAddress(SKR_TOKEN_MINT, owner);
  try {
    await getAccount(connection, userTokenAccount);
  } catch {
    transaction.add(
      createAssociatedTokenAccountInstruction(
        owner,
        userTokenAccount,
        owner,
        SKR_TOKEN_MINT
      )
    );
  }

  const withdrawIx = await buildWithdrawInstruction(owner);
  transaction.add(withdrawIx);

  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = owner;

  return transaction;
}
```

### Check Withdrawal Eligibility

```typescript
async function canWithdraw(
  connection: Connection,
  user: PublicKey
): Promise<{ eligible: boolean; reason?: string; timeRemaining?: number }> {
  const stakeData = await fetchUserStakeAccount(connection, user);
  
  if (!stakeData) {
    return { eligible: false, reason: 'No stake account found' };
  }
  
  if (stakeData.unstakingAmount === BigInt(0)) {
    return { eligible: false, reason: 'No pending unstake' };
  }
  
  const currentTime = Math.floor(Date.now() / 1000);
  const unlockTime = stakeData.unstakeTimestamp + UNSTAKE_COOLDOWN_SECONDS;
  
  if (currentTime < unlockTime) {
    return { 
      eligible: false, 
      reason: 'Cooldown period not complete',
      timeRemaining: unlockTime - currentTime
    };
  }
  
  return { eligible: true };
}
```

---

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `AccountNotFound` | UserStake PDA doesn't exist | First stake creates the account automatically |
| `InsufficientFunds` | Not enough SKR tokens | Check user's token balance |
| `InvalidAccountData` | Wrong account passed | Verify PDA derivation |
| `CooldownNotComplete` | Trying to withdraw too early | Wait for 48-hour cooldown |
| `AlreadyUnstaking` | Pending unstake exists | Withdraw or cancel first |
| `NoUnstakeInProgress` | No pending unstake to cancel/withdraw | Check unstakingAmount first |

### Error Handling Pattern

```typescript
try {
  const signature = await connection.sendRawTransaction(signedTx.serialize());
  await connection.confirmTransaction(signature, 'confirmed');
} catch (error) {
  if (error.message.includes('insufficient funds')) {
    throw new Error('Insufficient SKR balance');
  }
  if (error.message.includes('0x1')) {
    throw new Error('Invalid instruction format');
  }
  // Check transaction logs for specific program errors
  console.error('Transaction failed:', error);
  throw error;
}
```

---

## Best Practices

### 1. Always Verify PDAs

```typescript
// Pre-compute and verify PDAs match expected values
const [derivedConfig] = PublicKey.findProgramAddressSync(
  [Buffer.from('stake_config')],
  SEEKER_PROGRAM_ID
);
console.assert(derivedConfig.equals(STAKE_CONFIG_PDA), 'PDA mismatch!');
```

### 2. Handle Account Creation

The UserStake account is created automatically on first stake, but ensure the user has a token ATA:

```typescript
// Always check/create ATA before stake or withdraw
const userATA = await getAssociatedTokenAddress(SKR_TOKEN_MINT, user);
try {
  await getAccount(connection, userATA);
} catch {
  // Add ATA creation instruction to transaction
}
```

### 3. Use Fresh Share Price

Share price changes over time. Always fetch current price before unstaking:

```typescript
// DON'T cache share price for long periods
const sharePrice = await fetchSharePrice(connection); // Fresh fetch
const shares = tokensToShares(rawAmount, sharePrice);
```

### 4. Simulate Before Sending

```typescript
const simulation = await connection.simulateTransaction(transaction);
if (simulation.value.err) {
  console.error('Simulation failed:', simulation.value.err);
  console.error('Logs:', simulation.value.logs);
  throw new Error('Transaction would fail');
}
```

### 5. Browser Compatibility

Use manual byte encoding for browser environments:

```typescript
// Instead of Buffer.writeBigUInt64LE (not available in all browsers)
function encodeU64(value: bigint): Buffer {
  const buf = Buffer.alloc(8);
  let val = value;
  for (let i = 0; i < 8; i++) {
    buf[i] = Number(val & BigInt(0xff));
    val = val >> BigInt(8);
  }
  return buf;
}
```

---

## Troubleshooting

### Transaction Fails with "Invalid instruction"

1. Check discriminator bytes are correct
2. Verify all accounts are in correct order
3. Ensure instruction data size matches expected

### UserStake PDA Not Found

```typescript
// Verify PDA derivation
const [pda, bump] = getUserStakePDA(user);
console.log('Expected PDA:', pda.toBase58());
console.log('Bump:', bump);

// Check if account exists on-chain
const accountInfo = await connection.getAccountInfo(pda);
console.log('Exists:', accountInfo !== null);
```

### Share Calculation Mismatch

```typescript
// Debug share calculations
const sharePrice = await fetchSharePrice(connection);
console.log('Share Price:', sharePrice.toString());
console.log('Share Price (human):', Number(sharePrice) / 1e9);

const rawTokens = toRawAmount(100); // 100 SKR
const shares = tokensToShares(rawTokens, sharePrice);
console.log('100 SKR = shares:', shares.toString());

const backToTokens = sharesToTokens(shares, sharePrice);
console.log('Shares back to tokens:', backToTokens.toString());
```

### Decode Transaction for Debugging

```typescript
// Decode an existing transaction to verify format
const tx = await connection.getTransaction(signature, {
  maxSupportedTransactionVersion: 0
});

tx.transaction.message.instructions.forEach((ix, idx) => {
  const data = Buffer.from(ix.data, 'base64');
  console.log(`Instruction ${idx}:`);
  console.log('  Discriminator:', data.slice(0, 8).toString('hex'));
  console.log('  Data length:', data.length);
});
```

---

## Reference Implementation

For a complete working implementation, see:
- [`/src/lib/staking-program.ts`](../src/lib/staking-program.ts) - Full TypeScript implementation
- [`/scripts/inspect-userstake-account.js`](../scripts/inspect-userstake-account.js) - Account inspection utility
- [`/scripts/decode-tx-instructions.js`](../scripts/decode-tx-instructions.js) - Transaction decoder

---

## Support

- **Discord**: [Ninja Protocol Discord](https://discord.gg/ninjaprotocol)
- **GitHub Issues**: Report bugs or request features
- **Documentation**: This file is the authoritative reference

---

*Last updated: January 2026*
