# Testing Solana Programs

## Testing Framework Comparison

| Framework | Language | Speed | Fidelity | Best For |
|---|---|---|---|---|
| **LiteSVM** | Rust / TypeScript | ⚡ Very Fast | High | Unit & integration tests (recommended) |
| **Mollusk** | Rust | ⚡ Very Fast | Medium | Single-instruction unit tests |
| **Bankrun** | TypeScript | ⚡ Fast | High | JS/TS integration tests (legacy, use litesvm) |
| **solana-program-test** | Rust | 🐌 Slow | High | Full BanksServer tests (legacy) |
| **solana-test-validator** | Any | 🐌 Very Slow | Highest | E2E with real validator |
| **Surfpool** | Any | 🏎️ Fast | Highest | Modern replacement for test-validator |

## LiteSVM (Recommended)

### Rust Setup
```toml
# Cargo.toml
[dev-dependencies]
litesvm = "0.6"
solana-keypair = "2.1"
solana-message = "2.1"
solana-signer = "2.1"
solana-transaction = "2.1"
solana-system-interface = "0.2"
```

### Basic LiteSVM Test (Rust)
```rust
#[cfg(test)]
mod tests {
    use litesvm::LiteSVM;
    use solana_keypair::Keypair;
    use solana_signer::Signer;
    use solana_message::Message;
    use solana_transaction::Transaction;

    #[test]
    fn test_initialize() {
        let mut svm = LiteSVM::new();

        // Load program
        let program_id = Pubkey::new_unique();
        svm.add_program_from_file(program_id, "target/deploy/my_program.so")
            .unwrap();

        // Setup accounts
        let authority = Keypair::new();
        svm.airdrop(&authority.pubkey(), 10_000_000_000).unwrap();

        // Build instruction
        let ix = my_program::instruction::initialize(
            &program_id,
            &authority.pubkey(),
            42,
        );

        // Build and send transaction
        let tx = Transaction::new(
            &[&authority],
            Message::new(&[ix], Some(&authority.pubkey())),
            svm.latest_blockhash(),
        );

        let result = svm.send_transaction(tx);
        assert!(result.is_ok());

        // Verify account state
        let (pda, _) = Pubkey::find_program_address(
            &[b"my_account", authority.pubkey().as_ref()],
            &program_id,
        );
        let account = svm.get_account(&pda).unwrap();
        assert_eq!(account.data.len(), ACCOUNT_SIZE);
    }
}
```

### LiteSVM with Anchor (Rust)
```rust
#[cfg(test)]
mod tests {
    use anchor_lang::prelude::*;
    use anchor_lang::InstructionData;
    use anchor_lang::ToAccountMetas;
    use litesvm::LiteSVM;

    #[test]
    fn test_anchor_program() {
        let mut svm = LiteSVM::new();
        svm.add_program_from_file(
            my_program::ID,
            "target/deploy/my_program.so",
        ).unwrap();

        let authority = Keypair::new();
        svm.airdrop(&authority.pubkey(), 10_000_000_000).unwrap();

        let (my_account, _bump) = Pubkey::find_program_address(
            &[b"my_account", authority.pubkey().as_ref()],
            &my_program::ID,
        );

        let ix = Instruction {
            program_id: my_program::ID,
            accounts: my_program::accounts::Initialize {
                my_account,
                authority: authority.pubkey(),
                system_program: system_program::ID,
            }.to_account_metas(None),
            data: my_program::instruction::Initialize {
                data: 42,
            }.data(),
        };

        let tx = Transaction::new(
            &[&authority],
            Message::new(&[ix], Some(&authority.pubkey())),
            svm.latest_blockhash(),
        );

        svm.send_transaction(tx).unwrap();
    }
}
```

### LiteSVM (TypeScript/JavaScript)
```bash
yarn add --dev litesvm
```

```typescript
import { LiteSVM } from "litesvm";
import { Keypair, PublicKey, Transaction, SystemProgram } from "@solana/web3.js";
import * as anchor from "@coral-xyz/anchor";

describe("my_program", () => {
    let svm: LiteSVM;
    let authority: Keypair;

    beforeAll(() => {
        svm = LiteSVM.default();
        svm.addProgramFromFile(
            new PublicKey("YOUR_PROGRAM_ID"),
            "target/deploy/my_program.so"
        );

        authority = Keypair.generate();
        svm.airdrop(authority.publicKey, 10_000_000_000n);
    });

    it("initializes", () => {
        // Build and send transaction
        const tx = new Transaction().add(
            // ... your instruction
        );
        tx.recentBlockhash = svm.latestBlockhash();
        tx.sign(authority);

        const result = svm.sendTransaction(tx);
        expect(result.err).toBeNull();
    });
});
```

## Mollusk (Single-Instruction Testing)

Best for unit testing individual instructions in isolation.

```toml
[dev-dependencies]
mollusk-svm = "0.1"
```

```rust
#[cfg(test)]
mod tests {
    use mollusk_svm::Mollusk;
    use solana_instruction::Instruction;

    #[test]
    fn test_instruction() {
        let program_id = Pubkey::new_unique();
        let mollusk = Mollusk::new(&program_id, "target/deploy/my_program.so");

        let instruction = Instruction {
            program_id,
            accounts: vec![/* account metas */],
            data: vec![/* instruction data */],
        };

        let accounts = vec![/* (pubkey, account) tuples */];

        let result = mollusk.process_instruction(&instruction, &accounts);
        assert!(!result.program_result.is_err());
    }
}
```

## Anchor TypeScript Tests (Default)

The standard Anchor test setup using `anchor test`:

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";
import { assert } from "chai";

describe("my_program", () => {
    const provider = anchor.AnchorProvider.env();
    anchor.setProvider(provider);

    const program = anchor.workspace.MyProgram as Program<MyProgram>;

    it("Initializes", async () => {
        const authority = provider.wallet;

        const [myAccount] = anchor.web3.PublicKey.findProgramAddressSync(
            [Buffer.from("my_account"), authority.publicKey.toBuffer()],
            program.programId
        );

        const tx = await program.methods
            .initialize(new anchor.BN(42))
            .accounts({
                myAccount,
                // authority and system_program auto-resolved in 0.30+
            })
            .rpc();

        console.log("Transaction signature:", tx);

        const account = await program.account.myAccount.fetch(myAccount);
        assert.equal(account.data.toNumber(), 42);
        assert.equal(
            account.authority.toString(),
            authority.publicKey.toString()
        );
    });
});
```

## Surfpool (Modern Test Validator Alternative)

```bash
# Install
curl --proto '=https' --tlsv1.2 -LsSf \
  https://github.com/txtx/surfpool/releases/latest/download/surfpool-installer.sh | sh

# Start (replaces solana-test-validator)
surfpool

# Use with Anchor
anchor test --skip-local-validator
# (Point Anchor.toml cluster to surfpool's URL)
```

## Testing Best Practices

1. **Use LiteSVM for Rust tests** — much faster than `solana-program-test`
2. **Use Mollusk for instruction-level unit tests** — instant, no validator overhead
3. **Test error cases explicitly** — verify that invalid inputs produce expected errors
4. **Test PDA derivation** — ensure seeds produce expected addresses
5. **Test account constraints** — verify unauthorized access is rejected
6. **Use `anchor test` for integration tests** — TypeScript tests with real accounts
7. **Consider Surfpool** for tests needing full validator fidelity with better DX
8. **Clone mainnet state** for tests that depend on existing programs (SPL Token, etc.)

### Testing Error Handling
```rust
// LiteSVM
let result = svm.send_transaction(bad_tx);
assert!(result.is_err());
// Check specific error if needed:
// let err = result.unwrap_err();
```

```typescript
// Anchor TS
try {
    await program.methods.initialize(new anchor.BN(0))
        .accounts({...})
        .rpc();
    assert.fail("Should have failed");
} catch (err) {
    assert.include(err.message, "InvalidData");
}
```
