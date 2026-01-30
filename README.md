# Solana Claude Skills

A comprehensive set of Claude AI skills for Solana development. These skills help AI assistants resolve dependency issues, write programs, manage tokens, and build client applications on Solana.

## Skills

### 🔧 [solana-deps](./solana-deps/SKILL.md) — Dependency & Compatibility Resolution
The #1 developer pain point. Resolves version mismatches between Anchor, Solana CLI, Rust, Platform Tools, and system libraries (GLIBC). Includes a full compatibility matrix, common error→fix mappings, and install guides.

### 📦 [solana-program-dev](./solana-program-dev/SKILL.md) — Program Development
Patterns for writing Solana programs using both native Rust and Anchor framework. Covers account structures, CPIs, PDAs, error handling, and testing with LiteSVM/Bankrun/Mollusk.

### 🪙 [solana-token-dev](./solana-token-dev/SKILL.md) — Token Development
SPL Token operations, Token-2022 extensions, and Metaplex metadata. Covers mint creation, transfers, token accounts, and the full Token Extensions API.

### 🌐 [solana-client-dev](./solana-client-dev/SKILL.md) — Client-Side Development
Building frontends and scripts with `@solana/web3.js` v2, wallet adapters, and RPC optimization patterns.

## How to Use

These skills are designed to be loaded by Claude-based AI assistants. Each skill directory contains:

- **`SKILL.md`** — Main entry point with description, triggers, and instructions
- **Reference files** — Detailed documentation on specific topics

Point your AI assistant's skill/tool configuration at any `SKILL.md` file to activate that skill.

## Version Info

- Last updated: January 2026
- Covers: Anchor 0.29.x through 0.33.x (unreleased), Solana CLI v1.16 through v3.1.x, Platform Tools v1.43 through v1.52
