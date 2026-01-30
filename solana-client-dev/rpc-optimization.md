# RPC Optimization & Best Practices

## RPC Providers

### Free (Rate-Limited)
- `https://api.mainnet-beta.solana.com` — Solana Foundation (heavy rate limits)
- `https://api.devnet.solana.com` — Devnet
- `https://api.testnet.solana.com` — Testnet

### Paid (Recommended for Production)
- **Helius** — https://helius.dev (DAS API, webhooks, enhanced API)
- **Triton (RPC Pool)** — https://triton.one
- **QuickNode** — https://quicknode.com
- **Alchemy** — https://alchemy.com
- **Shyft** — https://shyft.to

### Specialized
- **Helius DAS API** — Digital Asset Standard for NFTs/tokens (faster than getProgramAccounts)
- **Geyser gRPC** — Real-time account/transaction streaming (Yellowstone)

## Connection Best Practices

### Commitment Levels
```typescript
// Fastest but least reliable:
{ commitment: "processed" }  // Node has processed but not confirmed

// Default, good balance:
{ commitment: "confirmed" }  // Supermajority of stake has voted

// Safest (for financial operations):
{ commitment: "finalized" }  // 31+ confirmed blocks (400ms slots × 31 ≈ 12s)
```

### Connection Configuration
```typescript
const connection = new Connection(RPC_URL, {
    commitment: "confirmed",
    confirmTransactionInitialTimeout: 60000,
    wsEndpoint: WS_URL, // Optional separate WebSocket endpoint
});
```

## Reducing RPC Calls

### 1. Use `getMultipleAccountsInfo` instead of multiple `getAccountInfo`
```typescript
// ❌ Bad: N RPC calls
for (const addr of addresses) {
    const account = await connection.getAccountInfo(addr);
}

// ✅ Good: 1 RPC call (max 100 accounts per call)
const accounts = await connection.getMultipleAccountsInfo(addresses);
```

### 2. Use `getProgramAccounts` with filters
```typescript
// ❌ Bad: Fetch all accounts then filter client-side
const all = await connection.getProgramAccounts(programId);

// ✅ Good: Filter on the RPC side
const filtered = await connection.getProgramAccounts(programId, {
    filters: [
        { dataSize: 165 },  // Token account size
        {
            memcmp: {
                offset: 32,  // Owner field offset
                bytes: ownerPubkey.toBase58(),
            },
        },
    ],
    encoding: "base64",
});
```

### 3. Use Anchor's account fetching
```typescript
// Fetch single account
const account = await program.account.myAccount.fetch(address);

// Fetch all with filters
const accounts = await program.account.myAccount.all([
    { memcmp: { offset: 8, bytes: authority.toBase58() } },
]);

// Fetch multiple known accounts
const accounts = await program.account.myAccount.fetchMultiple(addresses);
```

### 4. Cache aggressively
```typescript
// Cache blockhash (valid for ~60 seconds / 150 slots)
let cachedBlockhash = null;
let cachedAt = 0;

async function getBlockhash() {
    if (Date.now() - cachedAt < 30000) return cachedBlockhash;
    cachedBlockhash = (await connection.getLatestBlockhash()).blockhash;
    cachedAt = Date.now();
    return cachedBlockhash;
}
```

## Transaction Optimization

### Priority Fees
```typescript
import { ComputeBudgetProgram } from "@solana/web3.js";

// Get recommended priority fee
const recentFees = await connection.getRecentPrioritizationFees({
    lockedWritableAccounts: [accountAddress],
});
const medianFee = recentFees
    .map(f => f.prioritizationFee)
    .sort((a, b) => a - b)[Math.floor(recentFees.length / 2)];

const tx = new Transaction().add(
    ComputeBudgetProgram.setComputeUnitLimit({ units: 200_000 }),
    ComputeBudgetProgram.setComputeUnitPrice({
        microLamports: Math.max(medianFee, 1000),
    }),
    // ... your instructions
);
```

### Compute Unit Optimization
```typescript
// Simulate first to get actual CU usage
const simulation = await connection.simulateTransaction(tx);
const unitsUsed = simulation.value.unitsConsumed;

// Set limit to actual + buffer (10-20%)
ComputeBudgetProgram.setComputeUnitLimit({
    units: Math.ceil(unitsUsed * 1.2),
});
```

### Transaction Confirmation Strategies
```typescript
// Strategy 1: Simple (good enough for most)
const sig = await sendTransaction(tx, connection);
await connection.confirmTransaction(sig, "confirmed");

// Strategy 2: With timeout and retry
async function sendWithRetry(tx: Transaction, connection: Connection) {
    const { blockhash, lastValidBlockHeight } =
        await connection.getLatestBlockhash("confirmed");
    tx.recentBlockhash = blockhash;

    const sig = await connection.sendRawTransaction(tx.serialize(), {
        skipPreflight: false,
        maxRetries: 3,
    });

    const confirmation = await connection.confirmTransaction({
        signature: sig,
        blockhash,
        lastValidBlockHeight,
    }, "confirmed");

    if (confirmation.value.err) {
        throw new Error(`Transaction failed: ${JSON.stringify(confirmation.value.err)}`);
    }
    return sig;
}

// Strategy 3: Aggressive retry (for congested periods)
async function sendWithAggressiveRetry(tx: VersionedTransaction, connection: Connection) {
    const rawTx = tx.serialize();
    const startTime = Date.now();
    const timeout = 60000;

    const sig = await connection.sendRawTransaction(rawTx, {
        skipPreflight: true,  // Skip to reduce latency
        maxRetries: 0,        // We handle retries
    });

    // Retry sending every 2 seconds
    const retryInterval = setInterval(async () => {
        if (Date.now() - startTime > timeout) {
            clearInterval(retryInterval);
            return;
        }
        try {
            await connection.sendRawTransaction(rawTx, {
                skipPreflight: true,
                maxRetries: 0,
            });
        } catch {}
    }, 2000);

    try {
        const result = await connection.confirmTransaction(sig, "confirmed");
        clearInterval(retryInterval);
        return sig;
    } catch {
        clearInterval(retryInterval);
        throw new Error("Transaction failed to confirm");
    }
}
```

## WebSocket Subscriptions

### Account Changes
```typescript
const subscriptionId = connection.onAccountChange(
    accountAddress,
    (accountInfo) => {
        console.log("Account changed:", accountInfo);
    },
    "confirmed",
);

// Cleanup
connection.removeAccountChangeListener(subscriptionId);
```

### Program Account Changes
```typescript
connection.onProgramAccountChange(
    programId,
    (keyedAccountInfo) => {
        const { accountId, accountInfo } = keyedAccountInfo;
        console.log("Program account changed:", accountId.toString());
    },
    "confirmed",
    [{ dataSize: 165 }], // Optional filters
);
```

### Log Subscriptions (for events)
```typescript
connection.onLogs(
    programId,
    (logs) => {
        console.log("Transaction:", logs.signature);
        for (const log of logs.logs) {
            // Parse Anchor events from logs
            if (log.startsWith("Program data:")) {
                const eventData = log.slice("Program data: ".length);
                // Decode base64 event data
            }
        }
    },
    "confirmed",
);
```

## Common Pitfalls

1. **Don't use `processed` commitment for financial operations** — transactions can be dropped
2. **Don't hardcode `mainnet-beta` RPC URL** — use environment variables
3. **Don't ignore `getLatestBlockhash` expiry** — blockhashes expire after ~60s
4. **Don't skip priority fees on mainnet** — transactions may never land
5. **Don't poll in tight loops** — use WebSocket subscriptions instead
6. **Don't send preflight with `skipPreflight: true` during development** — you'll miss errors
7. **Do handle `TransactionExpiredBlockheightExceededError`** — retry with new blockhash
8. **Do use versioned transactions** for more than 35 accounts (Address Lookup Tables)
