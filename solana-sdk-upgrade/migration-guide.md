# Solana SDK v2 → v3 Migration Guide

## Overview

SDK v3 continues the modularization of the Solana crate ecosystem. The monolithic `solana-sdk` and `solana-program` crates are further broken down into smaller, focused crates. SPL libraries are split into interface and program crates for lighter dependencies and LTO support.

## Step-by-Step Upgrade Process

### Step 1: Upgrade to Latest v2 Crates

Before jumping to v3, upgrade all Solana dependencies to the latest v2 versions:

```toml
# Cargo.toml — upgrade to latest v2
[dependencies]
solana-program = "2.3"    # or latest 2.x
solana-sdk = "2.3"        # if used (client-side)
```

```bash
cargo update
cargo build 2>&1 | grep "deprecated"
```

### Step 2: Fix All Deprecation Warnings

v2 deprecation warnings tell you exactly what v3 requires. Fix them all:

```bash
# Build and capture warnings
RUSTFLAGS="-D warnings" cargo build 2>&1
```

Common deprecations:
- `solana_sdk::address_lookup_table` → use `solana_address_lookup_table_interface`
- `solana_sdk::system_instruction` → use `solana_system_interface::instruction`
- `solana_program::message` → use `solana_message`
- etc. (see [module-mapping.md](./module-mapping.md) for complete list)

### Step 3: Switch to SPL Interface Crates v1 (Optional)

This step is optional but makes the v3 upgrade smoother:

```toml
# Replace program crates with interface crates (v1)
[dependencies]
# Before:
# spl-token = "6"
# spl-token-2022 = "5"

# After:
spl-token-interface = "1"
spl-token-2022-interface = "1"
spl-associated-token-account-interface = "1"
spl-memo-interface = "1"
```

The interface crates mirror the module structure of the program crates (`state`, `instruction`, `error`), so import changes are minimal.

### Step 4: Upgrade to v3 Crates

Now upgrade to v3:

```toml
[dependencies]
solana-program = "3"      # for on-chain programs
solana-sdk = "3"          # for client-side code

# Or use individual crates (preferred for on-chain):
solana-pubkey = "3"
solana-account-info = "3"
solana-instruction = "3"
solana-msg = "3"
solana-program-entrypoint = "3"
solana-program-error = "3"
solana-system-interface = "1"
```

Fix any remaining compilation errors (see [breaking-changes.md](./breaking-changes.md)).

### Step 5: Upgrade SPL Interface Crates to v2 (Optional)

If you switched to interface crates in step 3, upgrade to v2:

```toml
[dependencies]
spl-token-interface = "2"
spl-token-2022-interface = "2"
spl-associated-token-account-interface = "2"
spl-memo-interface = "2"
```

## Cargo.toml Example: Before and After

### Before (v2)
```toml
[dependencies]
solana-program = "2.3"
spl-token = "6"
spl-token-2022 = "5"
spl-associated-token-account = "4"

[dev-dependencies]
solana-sdk = "2.3"
```

### After (v3)
```toml
[dependencies]
solana-program = "3"
spl-token-interface = "2"
spl-token-2022-interface = "2"
spl-associated-token-account-interface = "2"

[dev-dependencies]
solana-sdk = "3"
```

### After (v3, modular — recommended for on-chain)
```toml
[dependencies]
solana-pubkey = "3"
solana-account-info = "3"
solana-instruction = "3"
solana-msg = "3"
solana-program-entrypoint = "3"
solana-program-error = "3"
solana-system-interface = "1"
spl-token-interface = "2"

[dev-dependencies]
solana-keypair = "3"
solana-transaction = "3"
solana-native-token = "3"
```

## Common Import Changes

```rust
// ─── Before (v2) ───
use solana_program::pubkey::Pubkey;
use solana_program::account_info::AccountInfo;
use solana_program::system_instruction;
use solana_program::msg;
use solana_sdk::signer::Signer;

// ─── After (v3, using solana-program) ───
use solana_program::pubkey::Pubkey;       // still works
use solana_program::account_info::AccountInfo;  // still works
use solana_system_interface::instruction as system_instruction;
use solana_program::msg;                  // still works

// ─── After (v3, using individual crates) ───
use solana_pubkey::Pubkey;
use solana_account_info::AccountInfo;
use solana_system_interface::instruction as system_instruction;
use solana_msg::msg;
```

## Anchor Programs

Anchor 0.32+ already uses the modular Solana crate structure internally. If you're using Anchor:

- **Anchor 0.30.x**: Uses `solana-program` v1.18.x — upgrade to Anchor 0.31+ first
- **Anchor 0.31.x**: Uses `solana-program` v2.x — can follow this guide for custom code
- **Anchor 0.32.x**: Already uses individual `solana-*` crates — mostly v3-ready

For Anchor programs, most Solana types come through `anchor_lang::prelude::*`, so direct SDK changes are minimal unless you import Solana crates directly.

## Testing Your Migration

```bash
# 1. Build
cargo build-sbf

# 2. Run tests
cargo test

# 3. Check for any remaining v2 imports
grep -rn "solana_sdk::\|solana_program::" src/ --include="*.rs" | \
  grep -E "(address_lookup_table|bpf_loader|system_instruction|system_program|stake::|vote::)" 

# 4. Try LTO build (optional, with interface crates)
cargo build-sbf --lto
```
