# Common Solana Development Errors & Solutions

## GLIBC Errors

### `GLIBC_2.39 not found` / `GLIBC_2.38 not found`
```
anchor: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.39' not found (required by anchor)
```

**Cause:** Anchor 0.31+ binaries are built on newer Linux and require GLIBC ≥2.38. Anchor 0.32+ requires ≥2.39.

**Solutions (pick one):**
1. **Upgrade OS** (best): Ubuntu 24.04+ has GLIBC 2.39
2. **Build from source:**
   ```bash
   # For Anchor 0.31.x:
   cargo install --git https://github.com/solana-foundation/anchor --tag v0.31.1 anchor-cli
   
   # For Anchor 0.32.x:
   cargo install --git https://github.com/solana-foundation/anchor --tag v0.32.1 anchor-cli
   ```
3. **Use Docker:**
   ```bash
   docker run -v $(pwd):/workspace -w /workspace solanafoundation/anchor:0.31.1 anchor build
   ```
4. **Use AVM with source build:**
   ```bash
   avm install 0.31.1 --from-source
   ```

---

## Rust / Cargo Errors

### `anchor-cli` fails to install with Rust 1.80 (`time` crate issue)
```
error[E0635]: unknown feature `proc_macro_span_shrink`
 --> .cargo/registry/src/.../time-macros-0.2.16/src/lib.rs
```

**Cause:** Anchor 0.30.x uses a `time` crate version incompatible with Rust ≥1.80 ([anchor#3143](https://github.com/coral-xyz/anchor/pull/3143)).

**Solutions:**
1. **Use AVM** — it auto-selects `rustc 1.79.0` for Anchor < 0.31 ([anchor#3315](https://github.com/coral-xyz/anchor/pull/3315))
2. **Pin Rust version:**
   ```bash
   rustup install 1.79.0
   rustup default 1.79.0
   cargo install --git https://github.com/coral-xyz/anchor --tag v0.30.1 anchor-cli --locked
   ```
3. **Upgrade to Anchor 0.31+** which fixes this issue

### `unexpected_cfgs` warnings flooding build output
```
warning: unexpected `cfg` condition name: `feature`
```

**Cause:** Newer Rust versions (1.80+) are stricter about `cfg` conditions.

**Solution:** Add to your program's `Cargo.toml`:
```toml
[lints.rust]
unexpected_cfgs = { level = "allow" }
```
Or upgrade to Anchor 0.31+ which handles this.

### `error[E0603]: module inner is private`
**Cause:** Version mismatch between `anchor-lang` crate and Anchor CLI.

**Solution:** Ensure `anchor-lang` in Cargo.toml matches your `anchor --version`.

---

## Build Errors

### `cargo build-sbf` not found
```
error: no such command: `build-sbf`
```

**Cause:** Solana CLI not installed, or PATH not set.

**Solutions:**
1. Install Solana CLI: `sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"`
2. Add to PATH: `export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"`
3. Verify: `solana --version`

### `cargo build-bpf` is deprecated
```
Warning: cargo-build-bpf is deprecated. Use cargo-build-sbf instead.
```

**Cause:** As of Anchor 0.30.0, `cargo build-sbf` is the default. BPF target is deprecated in favor of SBF.

**Solution:** This is just a warning if you're using older tooling. Anchor 0.30+ handles this automatically. If calling manually, use `cargo build-sbf`.

### Platform tools download failure
```
Error: Failed to download platform-tools
```
or
```
error: could not compile `solana-program`
```

**Solutions:**
1. **Clear cache and retry:**
   ```bash
   rm -rf ~/.cache/solana/
   cargo build-sbf
   ```
2. **Manual platform tools install:**
   ```bash
   # Check which version you need
   solana --version
   # Download manually from:
   # https://github.com/anza-xyz/platform-tools/releases
   ```
3. **Check disk space** (see "No space left" error below)

### `anchor build` IDL generation fails
```
Error: IDL build failed
```
or
```
BPF SDK: /home/user/.local/share/solana/install/releases/2.1.7/solana-release/bin/sdk/sbf
Error: Function _ZN5anchor...
```

**Solutions:**
1. **Ensure `idl-build` feature is enabled (required since 0.30.0):**
   ```toml
   [features]
   default = []
   idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
   ```
2. **Set ANCHOR_LOG for debugging:**
   ```bash
   ANCHOR_LOG=1 anchor build
   ```
3. **Skip IDL generation:**
   ```bash
   anchor build --no-idl
   ```
4. **Check for nightly Rust interference:**
   ```bash
   # IDL generation uses proc-macro2 which may need nightly features
   # Override with stable:
   RUSTUP_TOOLCHAIN=stable anchor build
   ```

### `anchor build` error with `proc_macro2` / `local_file` method not found
```
error[E0599]: no method named `local_file` found for struct `proc_macro2::Span`
```

**Cause:** proc-macro2 API change in newer nightly Rust.

**Solutions:**
1. Upgrade to Anchor 0.31.1+ (fixed in [#3663](https://github.com/solana-foundation/anchor/pull/3663))
2. Use stable Rust: `RUSTUP_TOOLCHAIN=stable anchor build`
3. Pin proc-macro2: `cargo update -p proc-macro2 --precise 1.0.86`

---

## Installation Errors

### `No space left on device` during Solana install
```
error: No space left on device (os error 28)
```

**Cause:** Solana CLI + platform tools can use 2-5 GB. Multiple versions compound this.

**Solutions:**
1. **Clean old versions:**
   ```bash
   # List installed versions
   ls ~/.local/share/solana/install/releases/
   
   # Remove old ones (keep only what you need)
   rm -rf ~/.local/share/solana/install/releases/1.16.*
   rm -rf ~/.local/share/solana/install/releases/1.17.*
   
   # Also clean cache
   rm -rf ~/.cache/solana/
   ```
2. **Clean Cargo/Rust caches:**
   ```bash
   cargo cache --autoclean  # if cargo-cache is installed
   # or manually:
   rm -rf ~/.cargo/registry/cache/
   rm -rf target/
   ```
3. **Clean AVM:**
   ```bash
   ls ~/.avm/bin/
   # Remove unused anchor versions
   ```

### `agave-install` not found
```
error: agave-install: command not found
```

**Cause:** Anchor CLI 0.31+ migrates to `agave-install` for Solana versions ≥1.18.19.

**Solution:** Install via the Solana install script (which installs both):
```bash
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"
```

---

## Testing Errors

### `solana-test-validator` crashes or hangs
```
Error: failed to start validator
```

**Solutions:**
1. **Kill existing validators:**
   ```bash
   pkill -f solana-test-validator
   # or
   solana-test-validator --kill
   ```
2. **Clean ledger:**
   ```bash
   rm -rf test-ledger/
   ```
3. **Check port availability:**
   ```bash
   lsof -i :8899  # RPC port
   lsof -i :8900  # Websocket port
   ```
4. **Consider Surfpool** as a modern alternative to `solana-test-validator`:
   ```bash
   curl --proto '=https' --tlsv1.2 -LsSf https://github.com/txtx/surfpool/releases/latest/download/surfpool-installer.sh | sh
   ```

### Anchor test fails with `Connection refused` / IPv6 issue
```
Error: connect ECONNREFUSED ::1:8899
```

**Cause:** Node.js 17+ resolves `localhost` to IPv6 `::1` by default, but `solana-test-validator` binds to `127.0.0.1`.

**Solutions:**
1. **Use Anchor 0.30+** which defaults to `127.0.0.1` instead of `localhost`
2. **Set NODE_OPTIONS:**
   ```bash
   NODE_OPTIONS="--dns-result-order=ipv4first" anchor test
   ```
3. **Edit Anchor.toml:**
   ```toml
   [provider]
   cluster = "http://127.0.0.1:8899"
   ```

---

## Anchor Version Migration Issues

### Anchor 0.29 → 0.30 Migration Errors

**`accounts` method type errors in TypeScript:**
```
Argument of type '{ ... }' is not assignable to parameter of type 'ResolvedAccounts<...>'
```

**Solution:** Change `.accounts({...})` to `.accountsPartial({...})` or remove auto-resolved accounts from the call.

**Missing `idl-build` feature:**
```
Error: `idl-build` feature is missing
```

**Solution:** Add to each program's Cargo.toml:
```toml
[features]
idl-build = ["anchor-lang/idl-build"]
```

**`overflow-checks` not specified:**
```
Error: overflow-checks must be specified in workspace Cargo.toml
```

**Solution:** Add to workspace `Cargo.toml`:
```toml
[profile.release]
overflow-checks = true
```

### Anchor 0.30 → 0.31 Migration Errors

**Solana v1 → v2 crate conflicts:**
```
error[E0308]: mismatched types
expected `solana_program::pubkey::Pubkey`
found `solana_sdk::pubkey::Pubkey`
```

**Solution:** Remove direct `solana-program` and `solana-sdk` dependencies. Use them through `anchor-lang`:
```rust
use anchor_lang::prelude::*;
// NOT: use solana_program::pubkey::Pubkey;
```

**`Discriminator` trait changes:**
```
error[E0277]: the trait bound `MyAccount: Discriminator` is not satisfied
```

**Solution:** Ensure you derive `#[account]` on your structs. The discriminator is now dynamically sized.

### Anchor 0.31 → 0.32 Migration Errors

**`solana-program` dependency warning becomes error:**
Anchor 0.32 fully removes `solana-program` as a dependency. If your code imports from `solana_program::*`, change to the smaller crates:
```rust
// Before (0.31):
use solana_program::pubkey::Pubkey;

// After (0.32):
use solana_pubkey::Pubkey;
// Or use anchor's re-export:
use anchor_lang::prelude::*;
```

**Duplicate mutable accounts error:**
```
Error: Duplicate mutable account
```
Anchor 0.32+ disallows duplicate mutable accounts by default. Use the `dup` constraint:
```rust
#[derive(Accounts)]
pub struct MyInstruction<'info> {
    #[account(mut)]
    pub account_a: Account<'info, MyAccount>,
    #[account(mut, dup = account_a)]
    pub account_b: Account<'info, MyAccount>,
}
```

---

## Miscellaneous Errors

### `solana airdrop` fails
```
Error: airdrop request failed
```

**Cause:** Rate limiting on devnet/testnet.

**Solutions:**
1. Wait and retry
2. Use the web faucet: https://faucet.solana.com
3. For testing, use localnet where airdrops are unlimited

### Anchor IDL account authority mismatch
```
Error: Authority did not sign
```

**Solution:** The IDL authority is the program's upgrade authority. Check with:
```bash
solana program show <PROGRAM_ID>
```

### `declare_program!` not finding IDL file
```
Error: file not found: idls/my_program.json
```

**Solution:** Place the IDL JSON in the `idls/` directory at the workspace root. The filename must match the program name (snake_case):
```
workspace/
├── idls/
│   └── my_program.json
├── programs/
│   └── my_program/
└── Anchor.toml
```
