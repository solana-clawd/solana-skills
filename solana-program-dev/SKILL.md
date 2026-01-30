# Solana Program Development

## Description
Guides AI assistants in writing, testing, and deploying Solana programs (smart contracts) using both native Rust and the Anchor framework.

## When to Use
- Writing a new Solana program (on-chain code)
- Implementing program instructions, accounts, state
- Working with PDAs, CPIs, or system instructions
- Testing programs with LiteSVM, bankrun, or Mollusk
- Deploying or upgrading programs
- Debugging on-chain program errors

## Instructions

### For New Programs
1. Ask if the user prefers **Anchor** (recommended) or **native Rust**
2. Determine the program's purpose and required accounts
3. Scaffold the project structure
4. Implement instruction logic with proper validation
5. Write tests
6. Guide deployment

### Key Principles
- **Always validate all accounts** — check ownership, signers, writable
- **Use PDAs** for program-derived state (avoid storing keypairs)
- **Handle errors explicitly** with custom error enums
- **Minimize compute units** — Solana has per-instruction limits (200k default, 1.4M max per tx)
- **Consider rent exemption** for all account creation
- **Use `declare_program!`** (Anchor 0.30+) for CPI to avoid dependency hell

## Reference Files
- [anchor-programs.md](./anchor-programs.md) — Anchor framework patterns
- [native-programs.md](./native-programs.md) — Native Rust program patterns
- [testing.md](./testing.md) — Testing strategies and frameworks

## Common Patterns

### Anchor Program Skeleton
```rust
use anchor_lang::prelude::*;

declare_id!("YOUR_PROGRAM_ID");

#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.my_account.data = data;
        ctx.accounts.my_account.authority = ctx.accounts.authority.key();
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + MyAccount::INIT_SPACE,
        seeds = [b"my_account", authority.key().as_ref()],
        bump,
    )]
    pub my_account: Account<'info, MyAccount>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
#[derive(InitSpace)]
pub struct MyAccount {
    pub authority: Pubkey,
    pub data: u64,
}
```

### PDA Derivation
```rust
// On-chain
let (pda, bump) = Pubkey::find_program_address(
    &[b"seed", user.key().as_ref()],
    &program_id,
);

// In Anchor constraints
#[account(
    seeds = [b"seed", authority.key().as_ref()],
    bump,
)]
pub my_pda: Account<'info, MyData>,
```

### CPI Pattern (Anchor 0.30+ with declare_program!)
```rust
// In idls/other_program.json (place the IDL there)
declare_program!(other_program);

// Use in your program:
use other_program::cpi;
cpi::some_instruction(ctx, args)?;
```
