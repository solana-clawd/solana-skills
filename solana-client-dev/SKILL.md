# Solana Client-Side Development

## Description
Guides building frontends, scripts, and client applications that interact with Solana programs. Covers @solana/web3.js v2, wallet adapters, and RPC optimization.

## When to Use
- Building a frontend that connects to Solana
- Writing scripts to interact with on-chain programs
- Integrating wallet connections (Phantom, Solflare, etc.)
- Optimizing RPC usage and reducing costs
- Working with transactions, signatures, and confirmations
- Migrating from @solana/web3.js v1 to v2

## Instructions

### Key Decisions
1. **@solana/web3.js version:** v2 is the modern rewrite (tree-shakeable, functional). v1 is legacy but still widely used.
2. **Framework:** React (Next.js), Vue, Svelte, or plain Node.js scripts
3. **Wallet integration:** Use `@solana/wallet-adapter` for browser wallets

### General Approach
1. Set up RPC connection
2. Integrate wallet (if browser app)
3. Build transactions with program instructions
4. Sign and send transactions
5. Confirm and handle results

## Reference Files
- [web3js-v2.md](./web3js-v2.md) — @solana/web3.js v2 patterns
- [wallet-adapter.md](./wallet-adapter.md) — Wallet integration
- [rpc-optimization.md](./rpc-optimization.md) — RPC best practices

## Common Patterns

### Quick Transaction (v1 style, still common)
```typescript
import { Connection, PublicKey, Transaction } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";

const connection = new Connection("https://api.devnet.solana.com");
const provider = new anchor.AnchorProvider(connection, wallet, {});
const program = new anchor.Program(idl, provider);

const tx = await program.methods
    .myInstruction(new anchor.BN(42))
    .accounts({ myAccount })
    .rpc();
```
