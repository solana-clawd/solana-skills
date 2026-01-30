# @solana/web3.js v2

## Overview
`@solana/web3.js` v2 is a complete rewrite of the Solana JavaScript SDK. It's tree-shakeable, functional (not class-based), and significantly smaller when bundled.

## Installation
```bash
# v2 (new)
yarn add @solana/web3.js@2
# or specific packages:
yarn add @solana/addresses @solana/keys @solana/transactions @solana/rpc

# v1 (legacy, still widely used)
yarn add @solana/web3.js@1
```

## Key Differences: v1 vs v2

| Feature | v1 | v2 |
|---|---|---|
| API style | Class-based | Functional |
| Tree-shaking | ❌ Poor | ✅ Excellent |
| Bundle size | ~300KB | ~30KB (used only) |
| `PublicKey` | `new PublicKey("...")` | `address("...")` |
| `Keypair` | `Keypair.generate()` | `await generateKeyPair()` |
| `Connection` | `new Connection(url)` | `createSolanaRpc(url)` |
| Transactions | `Transaction` / `VersionedTransaction` | `pipe(createTransaction, ...)` |

## v2 Core Concepts

### Addresses (replaces PublicKey)
```typescript
import { address, getAddressFromPublicKey } from "@solana/addresses";

// From string
const addr = address("11111111111111111111111111111111");

// From public key bytes
const addr2 = await getAddressFromPublicKey(publicKey);
```

### RPC Connection
```typescript
import { createSolanaRpc, createSolanaRpcSubscriptions } from "@solana/rpc";

const rpc = createSolanaRpc("https://api.devnet.solana.com");
const rpcSubscriptions = createSolanaRpcSubscriptions("wss://api.devnet.solana.com");

// Get balance
const balance = await rpc.getBalance(address("...")).send();

// Get account info
const accountInfo = await rpc.getAccountInfo(address("..."), {
    encoding: "base64",
}).send();
```

### Key Generation
```typescript
import { generateKeyPair, createKeyPairFromBytes } from "@solana/keys";

// Generate new keypair
const keypair = await generateKeyPair();

// From secret key bytes
const keypair2 = await createKeyPairFromBytes(secretKeyBytes);
```

### Building Transactions
```typescript
import {
    pipe,
    createTransactionMessage,
    setTransactionMessageFeePayer,
    setTransactionMessageLifetimeUsingBlockhash,
    appendTransactionMessageInstruction,
    compileTransaction,
    signTransaction,
    sendAndConfirmTransaction,
} from "@solana/web3.js";

const transaction = pipe(
    createTransactionMessage({ version: 0 }),
    tx => setTransactionMessageFeePayer(feePayer, tx),
    tx => setTransactionMessageLifetimeUsingBlockhash(blockhash, tx),
    tx => appendTransactionMessageInstruction(instruction, tx),
);

const compiledTx = compileTransaction(transaction);
const signedTx = await signTransaction([keypair], compiledTx);
await sendAndConfirmTransaction(rpc, signedTx, { commitment: "confirmed" });
```

### Instructions
```typescript
import { type IInstruction } from "@solana/instructions";

const instruction: IInstruction = {
    programAddress: address("YOUR_PROGRAM_ID"),
    accounts: [
        { address: address("..."), role: AccountRole.WRITABLE_SIGNER },
        { address: address("..."), role: AccountRole.WRITABLE },
        { address: address("..."), role: AccountRole.READONLY },
    ],
    data: new Uint8Array([...]),
};
```

## v1 Patterns (Still Common)

### Connection & Provider Setup
```typescript
import { Connection, clusterApiUrl, Keypair } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";

const connection = new Connection(clusterApiUrl("devnet"), "confirmed");

// For scripts (with keypair):
const wallet = new anchor.Wallet(Keypair.fromSecretKey(secretKey));
const provider = new anchor.AnchorProvider(connection, wallet, {
    commitment: "confirmed",
    preflightCommitment: "confirmed",
});
anchor.setProvider(provider);

// For browser (with wallet adapter):
// const provider = new anchor.AnchorProvider(connection, walletAdapter, {});
```

### Interacting with Anchor Programs
```typescript
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

const program = anchor.workspace.MyProgram as Program<MyProgram>;

// Call instruction
const tx = await program.methods
    .initialize(new anchor.BN(42))
    .accounts({
        myAccount: myAccountPubkey,
        // authority and system_program auto-resolved (0.30+)
    })
    .rpc();

// Fetch account
const account = await program.account.myAccount.fetch(myAccountPubkey);

// Fetch all accounts of a type
const allAccounts = await program.account.myAccount.all();

// With filters
const filtered = await program.account.myAccount.all([
    { memcmp: { offset: 8, bytes: authorityPubkey.toBase58() } },
]);

// Simulate
const simResult = await program.methods
    .myInstruction()
    .accounts({...})
    .simulate();
```

### PDA Derivation
```typescript
// v1
const [pda, bump] = PublicKey.findProgramAddressSync(
    [Buffer.from("seed"), authority.toBuffer()],
    programId,
);

// v2
import { getProgramDerivedAddress } from "@solana/addresses";
const [pda, bump] = await getProgramDerivedAddress({
    programAddress: address("YOUR_PROGRAM_ID"),
    seeds: [
        new TextEncoder().encode("seed"),
        new Uint8Array(authorityAddress),
    ],
});
```

### Transaction with Priority Fees
```typescript
import { ComputeBudgetProgram } from "@solana/web3.js";

const tx = new Transaction()
    .add(
        ComputeBudgetProgram.setComputeUnitLimit({ units: 200_000 }),
        ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 50_000 }),
        // ... your instruction
    );
```

### Versioned Transactions (Lookup Tables)
```typescript
import {
    TransactionMessage,
    VersionedTransaction,
    AddressLookupTableAccount,
} from "@solana/web3.js";

// Fetch lookup table
const lookupTableAccount = await connection
    .getAddressLookupTable(lookupTableAddress)
    .then(res => res.value);

const messageV0 = new TransactionMessage({
    payerKey: payer.publicKey,
    recentBlockhash: blockhash,
    instructions: [/* ... */],
}).compileToV0Message([lookupTableAccount]);

const tx = new VersionedTransaction(messageV0);
tx.sign([payer]);
const sig = await connection.sendTransaction(tx);
```
