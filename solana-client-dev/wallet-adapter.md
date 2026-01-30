# Wallet Integration

## @solana/wallet-adapter

The standard library for connecting browser wallets (Phantom, Solflare, Backpack, etc.) to Solana dApps.

### Installation
```bash
# React
yarn add @solana/wallet-adapter-react @solana/wallet-adapter-react-ui \
  @solana/wallet-adapter-wallets @solana/wallet-adapter-base \
  @solana/web3.js

# The UI package includes a modal for wallet selection
```

### React Setup (Next.js / React)
```tsx
// providers/WalletProvider.tsx
"use client"; // Next.js App Router

import { FC, ReactNode, useMemo } from "react";
import {
    ConnectionProvider,
    WalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";
import { clusterApiUrl } from "@solana/web3.js";

// Import wallet adapter CSS
import "@solana/wallet-adapter-react-ui/styles.css";

export const SolanaProvider: FC<{ children: ReactNode }> = ({ children }) => {
    const endpoint = useMemo(() => clusterApiUrl("devnet"), []);

    // Wallets auto-detected via Wallet Standard
    // No need to manually list wallets in most cases
    const wallets = useMemo(() => [], []);

    return (
        <ConnectionProvider endpoint={endpoint}>
            <WalletProvider wallets={wallets} autoConnect>
                <WalletModalProvider>
                    {children}
                </WalletModalProvider>
            </WalletProvider>
        </ConnectionProvider>
    );
};
```

### Using the Wallet
```tsx
import { useWallet, useConnection } from "@solana/wallet-adapter-react";
import { WalletMultiButton } from "@solana/wallet-adapter-react-ui";

export function MyComponent() {
    const { connection } = useConnection();
    const { publicKey, sendTransaction, signTransaction } = useWallet();

    const handleClick = async () => {
        if (!publicKey) return;

        const tx = new Transaction().add(/* instruction */);
        tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;
        tx.feePayer = publicKey;

        const signature = await sendTransaction(tx, connection);
        await connection.confirmTransaction(signature, "confirmed");
    };

    return (
        <div>
            <WalletMultiButton />
            {publicKey && <p>Connected: {publicKey.toString()}</p>}
            <button onClick={handleClick} disabled={!publicKey}>
                Send Transaction
            </button>
        </div>
    );
}
```

### Using with Anchor
```tsx
import { useAnchorWallet, useConnection } from "@solana/wallet-adapter-react";
import { AnchorProvider, Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";
import idl from "../target/idl/my_program.json";

function useProgram() {
    const { connection } = useConnection();
    const wallet = useAnchorWallet();

    if (!wallet) return null;

    const provider = new AnchorProvider(connection, wallet, {
        commitment: "confirmed",
    });

    return new Program<MyProgram>(idl as any, provider);
}

// In component:
const program = useProgram();
if (program) {
    const tx = await program.methods
        .myInstruction(new BN(42))
        .accounts({ myAccount })
        .rpc();
}
```

## Wallet Standard

Modern wallets implement the [Wallet Standard](https://github.com/wallet-standard/wallet-standard). When using `@solana/wallet-adapter-react` with an empty wallets array, it auto-detects all Wallet Standard-compatible wallets.

### Supported Wallets (Auto-detected)
- Phantom
- Solflare
- Backpack
- Coinbase Wallet
- Brave Wallet
- And many more...

### Manual Wallet Selection (if needed)
```typescript
import { PhantomWalletAdapter } from "@solana/wallet-adapter-phantom";
import { SolflareWalletAdapter } from "@solana/wallet-adapter-solflare";

const wallets = useMemo(() => [
    new PhantomWalletAdapter(),
    new SolflareWalletAdapter(),
], []);
```

## Message Signing (Authentication)

```typescript
const { publicKey, signMessage } = useWallet();

async function signIn() {
    if (!publicKey || !signMessage) return;

    const message = new TextEncoder().encode(
        `Sign in to MyApp\nNonce: ${Date.now()}`
    );
    const signature = await signMessage(message);

    // Send publicKey + signature to backend for verification
    await fetch("/api/auth", {
        method: "POST",
        body: JSON.stringify({
            publicKey: publicKey.toString(),
            signature: Buffer.from(signature).toString("base64"),
            message: Buffer.from(message).toString("base64"),
        }),
    });
}
```

### Backend Verification (Node.js)
```typescript
import { PublicKey } from "@solana/web3.js";
import nacl from "tweetnacl";

function verifySignature(
    publicKeyStr: string,
    signatureBase64: string,
    messageBase64: string,
): boolean {
    const publicKey = new PublicKey(publicKeyStr);
    const signature = Buffer.from(signatureBase64, "base64");
    const message = Buffer.from(messageBase64, "base64");

    return nacl.sign.detached.verify(
        message,
        signature,
        publicKey.toBytes(),
    );
}
```

## Mobile Wallet Integration

### Solana Mobile Wallet Adapter
```bash
yarn add @solana-mobile/wallet-adapter-mobile
```

```typescript
import { SolanaMobileWalletAdapter } from "@solana-mobile/wallet-adapter-mobile";

const wallets = useMemo(() => [
    new SolanaMobileWalletAdapter({
        appIdentity: { name: "My App" },
        cluster: "devnet",
    }),
], []);
```
