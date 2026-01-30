# Solana Version Compatibility Matrix

## Master Compatibility Table

| Anchor Version | Release Date | Solana CLI | Rust Version | Platform Tools | GLIBC Req | Node.js | Key Notes |
|---|---|---|---|---|---|---|---|
| **0.32.x** | Oct 2025 | 2.1.x+ | 1.79–1.85+ (stable) | v1.50+ | ≥2.39 | ≥17 | Replaces `solana-program` with smaller crates; IDL builds on stable Rust; removes Solang |
| **0.31.1** | Apr 2025 | 2.0.x–2.1.x | 1.79–1.83 | v1.47+ | ≥2.38 | ≥17 | New Docker image `solanafoundation/anchor`; published under solana-foundation org |
| **0.31.0** | Mar 2025 | 2.0.x–2.1.x | 1.79–1.83 | v1.47+ | ≥2.38 | ≥17 | Solana v2 upgrade; dynamic discriminators; `LazyAccount`; `declare_program!` improvements |
| **0.30.1** | Jun 2024 | 1.18.x (rec: 1.18.8+) | 1.75–1.79 | v1.43 | ≥2.31 | ≥16 | `declare_program!` macro; legacy IDL conversion; `RUSTUP_TOOLCHAIN` override |
| **0.30.0** | Apr 2024 | 1.18.x (rec: 1.18.8) | 1.75–1.79 | v1.43 | ≥2.31 | ≥16 | New IDL spec; token extensions; `cargo build-sbf` default; `idl-build` feature required |
| **0.29.0** | Oct 2023 | 1.16.x–1.17.x | 1.68–1.75 | v1.37–v1.41 | ≥2.28 | ≥16 | Account reference changes; `idl build` compilation method; `.anchorversion` file |

## Solana CLI Version Mapping

| Solana CLI | Agave Version | Era | solana-program Crate | Platform Tools | Status |
|---|---|---|---|---|---|
| **3.1.x** | v3.1.x | Jan 2026 | N/A (validator only) | v1.52 | Edge/Beta |
| **3.0.x** | v3.0.x | Late 2025 | N/A (validator only) | v1.52 | Stable (mainnet) |
| **2.1.x** | v2.1.x | Mid 2025 | 2.x | v1.47–v1.51 | Stable |
| **2.0.x** | v2.0.x | Early 2025 | 2.x | v1.44–v1.47 | Legacy |
| **1.18.x** | N/A (pre-Anza) | 2024 | 1.18.x | v1.43 | Legacy |
| **1.17.x** | N/A | 2023 | 1.17.x | v1.37–v1.41 | Deprecated |
| **1.16.x** | N/A | 2023 | 1.16.x | v1.35–v1.37 | Deprecated |

### Important: Solana CLI v3.x
As of Agave v3.0.0, Anza **no longer publishes the `agave-validator` binary**. Operators must build from source. The CLI tools (for program development) remain available via `agave-install` or the install script.

## Platform Tools → Rust Toolchain Mapping

| Platform Tools | Bundled Rust | LLVM/Clang | Notes |
|---|---|---|---|
| **v1.52** | ~1.85 (solana fork) | Clang 20 | Latest; used by Solana CLI 3.x |
| **v1.51** | ~1.84 (solana fork) | Clang 19 | |
| **v1.50** | ~1.83 (solana fork) | Clang 19 | |
| **v1.49** | ~1.82 (solana fork) | Clang 18 | |
| **v1.48** | ~1.81 (solana fork) | Clang 18 | |
| **v1.47** | ~1.80 (solana fork) | Clang 17 | Used by Anchor 0.31.x |
| **v1.46** | ~1.79 (solana fork) | Clang 17 | |
| **v1.45** | ~1.79 (solana fork) | Clang 17 | |
| **v1.44** | ~1.78 (solana fork) | Clang 16 | |
| **v1.43** | ~1.75 (solana fork) | Clang 16 | Used by Anchor 0.30.x/Solana 1.18.x |

**Note:** Platform Tools ship a **forked** Rust compiler from [anza-xyz/rust](https://github.com/anza-xyz/rust). The version numbers approximate the upstream Rust equivalent. The forked compiler includes SBF/SBPF target support.

## GLIBC Requirements by OS

| OS / Distro | GLIBC Version | Compatible Anchor |
|---|---|---|
| **Ubuntu 24.04 (Noble)** | 2.39 | All (0.29–0.32+) |
| **Ubuntu 22.04 (Jammy)** | 2.35 | 0.29–0.30.x only |
| **Ubuntu 20.04 (Focal)** | 2.31 | 0.29–0.30.x only |
| **Debian 12 (Bookworm)** | 2.36 | 0.29–0.30.x only |
| **Debian 13 (Trixie)** | 2.40 | All |
| **Fedora 39+** | ≥2.38 | All |
| **Arch Linux (rolling)** | Latest | All |
| **macOS 14+ (Sonoma)** | N/A (no GLIBC) | All |
| **macOS 12-13** | N/A | All |
| **Windows WSL2 (Ubuntu)** | Depends on distro | See Ubuntu version |

### Why GLIBC matters
Anchor 0.31+ and 0.32+ binaries are compiled against newer GLIBC. If your system's GLIBC is too old, you'll get:
```
anchor: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.38' not found
```

**Solutions:**
1. Upgrade your OS (recommended)
2. Build Anchor from source: `cargo install --git https://github.com/coral-xyz/anchor --tag v0.31.1 anchor-cli`
3. Use Docker (see install-guide.md)

## Anchor ↔ solana-program Crate Versions

| Anchor | anchor-lang Crate | Uses solana-program | Notes |
|---|---|---|---|
| **0.32.x** | 0.32.x | Replaced by individual `solana-*` crates | `solana-program` no longer a direct dep |
| **0.31.x** | 0.31.x | 2.x | Upgraded to Solana v2 crate ecosystem |
| **0.30.x** | 0.30.x | 1.18.x | Last version using Solana v1 crates |
| **0.29.x** | 0.29.x | 1.16.x–1.17.x | |

### Solana v2 Crate Ecosystem (Anchor 0.31+)
Anchor 0.31+ uses the Solana v2 crate structure. The monolithic `solana-program` crate is being split into smaller crates:
- `solana-pubkey` / `solana-address`
- `solana-instruction`
- `solana-account-info`
- `solana-msg`
- `solana-invoke`
- `solana-entrypoint`
- etc.

Anchor 0.32+ fully replaces `solana-program` with these smaller crates. When using `Anchor 0.31.x`, the `anchor build` command warns if you have `solana-program` as a direct dependency — it should come through `anchor-lang`.

## Anchor CLI ↔ anchor-lang Crate Compatibility

The Anchor CLI checks version compatibility with the `anchor-lang` crate used in your project. **Mismatched versions will produce a warning.** Always keep these in sync:

```toml
# Cargo.toml
[dependencies]
anchor-lang = "0.31.1"

# Must match CLI:
# anchor --version → anchor-cli 0.31.1
```

## SPL Token Crate Versions

| Anchor | anchor-spl | spl-token | spl-token-2022 | spl-associated-token-account |
|---|---|---|---|---|
| **0.32.x** | 0.32.x | Latest compatible | Latest compatible | Latest compatible |
| **0.31.x** | 0.31.x | 6.x | 5.x | 4.x |
| **0.30.x** | 0.30.x | 4.x–6.x | 3.x–4.x | 3.x |
| **0.29.x** | 0.29.x | 4.x | 1.x–3.x | 2.x–3.x |

## Node.js / TypeScript Requirements

| Anchor | @coral-xyz/anchor | Node.js | TypeScript |
|---|---|---|---|
| **0.32.x** | 0.32.x | ≥17 | 5.x |
| **0.31.x** | 0.31.x | ≥17 | 5.x |
| **0.30.x** | 0.30.x | ≥16 | 4.x–5.x |
| **0.29.x** | 0.29.x | ≥16 | 4.x |

## Known Working Combinations (Tested)

### 🟢 Modern (Recommended for new projects — Jan 2026)
```
Anchor CLI: 0.31.1
Solana CLI: 2.1.7 (stable)
Rust: 1.83.0
Platform Tools: v1.50
Node.js: 20.x LTS
OS: Ubuntu 24.04 or macOS 14+
```

### 🟢 Latest Anchor (Cutting edge)
```
Anchor CLI: 0.32.1
Solana CLI: 2.1.7+
Rust: 1.84.0+
Platform Tools: v1.52
Node.js: 20.x LTS
OS: Ubuntu 24.04+ (GLIBC ≥2.39) or macOS 14+
```

### 🟡 Legacy Compatible (For older systems)
```
Anchor CLI: 0.30.1
Solana CLI: 1.18.26
Rust: 1.79.0
Platform Tools: v1.43
Node.js: 18.x LTS
OS: Ubuntu 20.04+ or macOS 12+
```

### 🟡 Transitional (Upgrading from 0.30 → 0.31)
```
Anchor CLI: 0.31.0
Solana CLI: 2.0.x
Rust: 1.79.0
Platform Tools: v1.47
Node.js: 20.x LTS
OS: Ubuntu 24.04 or macOS 14+
```
