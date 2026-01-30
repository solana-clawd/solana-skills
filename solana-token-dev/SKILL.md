# Solana Token Development

## Description
Guides creation and management of tokens on Solana using SPL Token, Token-2022 (Token Extensions), and Metaplex metadata standards.

## When to Use
- Creating fungible or non-fungible tokens
- Working with SPL Token or Token-2022 programs
- Implementing token extensions (transfer hooks, transfer fees, metadata, etc.)
- Managing token accounts, minting, burning, transferring
- Adding metadata to tokens (Metaplex)
- Building token-gated applications

## Instructions

### Deciding Token Standard
1. **SPL Token (Original)** — Simple fungible/NFT tokens. Widely supported.
2. **Token-2022 (Token Extensions)** — Modern standard with built-in extensions:
   - Transfer Fees, Transfer Hooks, Confidential Transfers
   - Metadata (on-chain, no Metaplex needed), Metadata Pointer
   - Group/Member Pointers, Permanent Delegate
   - Non-Transferable (Soulbound), Close Authority
3. **Metaplex Metadata** — Rich metadata for NFTs and fungible tokens (works with both token programs)

### Key Decision:
- New tokens in 2025+: **Prefer Token-2022** unless you need maximum ecosystem compatibility
- NFTs with rich metadata: **Token-2022 + embedded metadata** or **SPL Token + Metaplex**
- DeFi tokens: **SPL Token** still has broadest DeFi protocol support

## Reference Files
- [spl-token.md](./spl-token.md) — SPL Token operations
- [token-2022.md](./token-2022.md) — Token Extensions
- [token-metadata.md](./token-metadata.md) — Metaplex metadata

## Common Patterns

### Create Mint (Anchor)
```rust
#[account(
    init,
    payer = authority,
    mint::decimals = 9,
    mint::authority = authority,
    mint::freeze_authority = authority,
)]
pub mint: Account<'info, Mint>,
```

### Mint Tokens (Anchor)
```rust
use anchor_spl::token::{mint_to, MintTo};

mint_to(
    CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        MintTo {
            mint: ctx.accounts.mint.to_account_info(),
            to: ctx.accounts.token_account.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    ),
    amount,
)?;
```

### Transfer Tokens (Anchor)
```rust
use anchor_spl::token::{transfer, Transfer};

transfer(
    CpiContext::new(
        ctx.accounts.token_program.to_account_info(),
        Transfer {
            from: ctx.accounts.from_ata.to_account_info(),
            to: ctx.accounts.to_ata.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        },
    ),
    amount,
)?;
```
