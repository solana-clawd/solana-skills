# Solana Dependency & Compatibility Resolution

## Description
Resolves the notoriously painful Solana toolchain dependency issues — version mismatches between Anchor, Solana CLI, Rust, Platform Tools, GLIBC, and related crates. This is the #1 developer pain point in the Solana ecosystem.

## When to Use
Activate this skill when you encounter ANY of these situations:
- Setting up a new Solana development environment
- Upgrading Anchor, Solana CLI, or Rust versions
- Build errors mentioning `cargo build-sbf`, `cargo build-bpf`, platform-tools, or rustc versions
- GLIBC errors (`GLIBC_2.38 not found`, `GLIBC_2.39 not found`)
- `solana-program` crate version conflicts
- Anchor IDL generation failures
- "No space left on device" during Solana toolchain install
- Any error during `anchor build`, `anchor test`, or program deployment
- Migration between Anchor versions (0.29→0.30→0.31→0.32)
- Migration from `solana-program-test`/`bankrun` to `litesvm`

## Instructions

### Step 1: Diagnose Current Environment
Ask the user to run these commands and share the output:
```bash
# Check all versions
rustc --version
solana --version
anchor --version
avm --version
cargo build-sbf --version 2>/dev/null || echo "cargo build-sbf not found"
node --version
yarn --version || npm --version

# Check OS/GLIBC
uname -a
ldd --version 2>&1 | head -1  # Linux only
cat /etc/os-release 2>/dev/null | head -5  # Linux only

# Check Anchor.toml toolchain settings
cat Anchor.toml 2>/dev/null | grep -A5 '\[toolchain\]'
```

### Step 2: Consult the Compatibility Matrix
Refer to [compatibility-matrix.md](./compatibility-matrix.md) to find the correct version combination for the user's needs.

### Step 3: Check Common Errors
If the user has a specific error, check [common-errors.md](./common-errors.md) for the exact error message and its fix.

### Step 4: Guide Installation
If a fresh install or upgrade is needed, refer to [install-guide.md](./install-guide.md) for step-by-step instructions.

## Reference Files
- [compatibility-matrix.md](./compatibility-matrix.md) — Full version compatibility tables
- [common-errors.md](./common-errors.md) — Error message → solution mappings
- [install-guide.md](./install-guide.md) — Clean install & upgrade procedures

## Common Patterns

### Quick Version Check
```bash
rustc --version && solana --version && anchor --version
```

### Safe Modern Stack (as of Jan 2026)
```
Anchor 0.31.1+ with Solana CLI 2.1.x, Rust 1.79-1.83, Platform Tools v1.47+
Requires: Ubuntu 24.04+ or macOS 14+ (for GLIBC 2.38+)
```

### Legacy-Compatible Stack
```
Anchor 0.30.1 with Solana CLI 1.18.x, Rust 1.79.0, Platform Tools v1.43
Works on: Ubuntu 20.04+, macOS 12+
```

### Override Rust for Anchor Build
When AVM installs Anchor < 0.31, it auto-uses `rustc 1.79.0` to avoid the Rust 1.80 `time` crate issue (#3143).

### Anchor.toml Toolchain Override
```toml
[toolchain]
anchor_version = "0.31.1"
solana_version = "2.1.7"
```
