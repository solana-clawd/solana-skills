# Solana SDK v2 → v3 Upgrade Skill

Guide for upgrading Solana Rust programs from SDK v2.x to v3.x.

## References
- [migration-guide.md](./migration-guide.md) — Step-by-step upgrade process
- [module-mapping.md](./module-mapping.md) — Complete v2 → v3 module/crate mapping
- [breaking-changes.md](./breaking-changes.md) — Breaking API changes and fixes
- [spl-migration.md](./spl-migration.md) — SPL crate upgrades (spl-token → spl-token-interface)

## Quick Start

The recommended upgrade path:
1. Upgrade to latest v2 crates
2. Fix all deprecation warnings (they tell you exactly what to change)
3. Switch to SPL interface crates v1 (optional but recommended)
4. Upgrade to v3 crates
5. Upgrade SPL interface crates to v2 (optional)

## Source
Based on https://github.com/anza-xyz/solana-sdk/blob/master/README.md
