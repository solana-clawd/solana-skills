# SPL Crate Migration for SDK v3

## Why Interface Crates?

SPL libraries have been split into **interface** and **program** crates:

- **Program crates** (`spl-token`, `spl-token-2022`): Contain the full program with a `cdylib` target. This prevents Link-Time Optimization (LTO).
- **Interface crates** (`spl-token-interface`, `spl-token-2022-interface`): Contain only the types, instructions, and errors. Declare only a `lib` target, enabling LTO.

For on-chain programs that _use_ SPL tokens (but aren't the token program themselves), interface crates are strictly better — lighter dependencies, faster builds, LTO support.

## Migration Table

| Old Crate | Old Version | New Crate | New Version (SDK v2) | New Version (SDK v3) |
|---|---|---|---|---|
| `spl-token` | v6–v8 | `spl-token-interface` | v1 | v2 |
| `spl-token-2022` | v5+ | `spl-token-2022-interface` | v1 | v2 |
| `spl-associated-token-account` | v4+ | `spl-associated-token-account-interface` | v1 | v2 |
| `spl-memo` | any | `spl-memo-interface` | v1 | v2 |

## Cargo.toml Changes

```toml
# Before (SDK v2, program crates)
[dependencies]
spl-token = "6"
spl-token-2022 = "5"
spl-associated-token-account = "4"
spl-memo = "5"

# After (SDK v3, interface crates)
[dependencies]
spl-token-interface = "2"
spl-token-2022-interface = "2"
spl-associated-token-account-interface = "2"
spl-memo-interface = "2"
```

## Import Changes

The interface crates preserve the same module structure as the program crates:

```rust
// ─── Before ───
use spl_token::state::{Account, Mint};
use spl_token::instruction::{initialize_mint, transfer};
use spl_token::error::TokenError;
use spl_token::id;

// ─── After ───
use spl_token_interface::state::{Account, Mint};
use spl_token_interface::instruction::{initialize_mint, transfer};
use spl_token_interface::error::TokenError;
use spl_token_interface::id;
```

```rust
// ─── Before ───
use spl_token_2022::state::Account;
use spl_token_2022::extension::{ExtensionType, StateWithExtensions};

// ─── After ───
use spl_token_2022_interface::state::Account;
use spl_token_2022_interface::extension::{ExtensionType, StateWithExtensions};
```

```rust
// ─── Before ───
use spl_associated_token_account::get_associated_token_address;
use spl_associated_token_account::instruction::create_associated_token_account;

// ─── After ───
use spl_associated_token_account_interface::get_associated_token_address;
use spl_associated_token_account_interface::instruction::create_associated_token_account;
```

## Two-Step Migration (Recommended)

If you want to minimize risk, do the SPL migration in two steps:

### Step 1: Switch to interface crates v1 (still on SDK v2)
```toml
[dependencies]
solana-program = "2.3"           # still v2
spl-token-interface = "1"        # v1 works with SDK v2
```

Verify everything builds and tests pass.

### Step 2: Upgrade to SDK v3 + interface crates v2
```toml
[dependencies]
solana-program = "3"             # upgrade to v3
spl-token-interface = "2"        # v2 works with SDK v3
```

## LTO Builds

With interface crates, you can enable LTO:

```bash
cargo build-sbf --lto
```

> **Note:** LTO results are mixed. Always compare:
> - Program binary size (`ls -la target/deploy/*.so`)
> - Compute unit usage in tests
> 
> Some programs get smaller, others don't benefit. Profile before committing to LTO.

## Anchor Programs

For Anchor programs, SPL types typically come through `anchor-spl`:

```toml
[dependencies]
anchor-lang = "0.32"
anchor-spl = "0.32"
```

`anchor-spl` v0.32 already handles the SPL crate wiring internally. You generally don't need to depend on SPL crates directly unless you're doing custom CPI or low-level token operations outside of Anchor's macros.

If you DO have direct SPL imports alongside Anchor:
```rust
// Before:
use spl_token::instruction::transfer;

// After (if Anchor doesn't cover your use case):
use spl_token_interface::instruction::transfer;
```
