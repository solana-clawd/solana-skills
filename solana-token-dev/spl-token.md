# SPL Token Operations

## Program IDs
```
SPL Token:           TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA
Associated Token:    ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL
```

## Anchor SPL Token Dependencies
```toml
[dependencies]
anchor-lang = "0.31.1"
anchor-spl = { version = "0.31.1", features = ["idl-build"] }

[features]
idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
```

```rust
use anchor_spl::{
    token::{Token, TokenAccount, Mint, mint_to, MintTo, transfer, Transfer, burn, Burn},
    associated_token::AssociatedToken,
};
```

## Create a Mint

### Anchor Constraints
```rust
#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(
        init,
        payer = authority,
        mint::decimals = 9,
        mint::authority = authority.key(),
        mint::freeze_authority = authority.key(),
    )]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}
```

### PDA Mint Authority
```rust
#[derive(Accounts)]
pub struct CreatePdaMint<'info> {
    #[account(
        init,
        payer = payer,
        mint::decimals = 6,
        mint::authority = mint_authority,
        seeds = [b"my_mint"],
        bump,
    )]
    pub mint: Account<'info, Mint>,
    /// CHECK: PDA used as mint authority
    #[account(seeds = [b"mint_auth"], bump)]
    pub mint_authority: UncheckedAccount<'info>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub rent: Sysvar<'info, Rent>,
}
```

## Create Associated Token Account

```rust
#[derive(Accounts)]
pub struct CreateAta<'info> {
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = owner,
    )]
    pub token_account: Account<'info, TokenAccount>,
    pub mint: Account<'info, Mint>,
    /// CHECK: Token account owner
    pub owner: UncheckedAccount<'info>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```

## Mint Tokens

```rust
pub fn mint_tokens(ctx: Context<MintTokens>, amount: u64) -> Result<()> {
    mint_to(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
        ),
        amount,
    )?;
    Ok(())
}
```

### Mint with PDA Authority
```rust
pub fn mint_with_pda(ctx: Context<MintWithPda>, amount: u64) -> Result<()> {
    let seeds = &[b"mint_auth", &[ctx.bumps.mint_authority]];
    let signer_seeds = &[&seeds[..]];

    mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            signer_seeds,
        ),
        amount,
    )?;
    Ok(())
}
```

## Transfer Tokens

```rust
pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    transfer(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.from.to_account_info(),
                to: ctx.accounts.to.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        amount,
    )?;
    Ok(())
}
```

## Burn Tokens

```rust
pub fn burn_tokens(ctx: Context<BurnTokens>, amount: u64) -> Result<()> {
    burn(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Burn {
                mint: ctx.accounts.mint.to_account_info(),
                from: ctx.accounts.token_account.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
        amount,
    )?;
    Ok(())
}
```

## Close Token Account

```rust
use anchor_spl::token::{close_account, CloseAccount};

pub fn close_token_account(ctx: Context<CloseTokenAccount>) -> Result<()> {
    close_account(
        CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            CloseAccount {
                account: ctx.accounts.token_account.to_account_info(),
                destination: ctx.accounts.authority.to_account_info(),
                authority: ctx.accounts.authority.to_account_info(),
            },
        ),
    )?;
    Ok(())
}
```

## TypeScript / Client-Side Token Operations

### Using @solana/spl-token
```bash
yarn add @solana/spl-token
```

```typescript
import {
    createMint,
    getOrCreateAssociatedTokenAccount,
    mintTo,
    transfer,
    getAccount,
    getMint,
} from "@solana/spl-token";
import { Connection, Keypair, PublicKey } from "@solana/web3.js";

// Create mint
const mint = await createMint(
    connection,
    payer,         // fee payer
    authority,     // mint authority
    freezeAuth,    // freeze authority (null for none)
    9,             // decimals
);

// Create token account
const tokenAccount = await getOrCreateAssociatedTokenAccount(
    connection,
    payer,
    mint,
    owner,
);

// Mint tokens
await mintTo(
    connection,
    payer,
    mint,
    tokenAccount.address,
    authority,
    1_000_000_000, // 1 token with 9 decimals
);

// Transfer tokens
await transfer(
    connection,
    payer,
    sourceAccount.address,
    destinationAccount.address,
    owner,
    500_000_000,
);

// Check balance
const accountInfo = await getAccount(connection, tokenAccount.address);
console.log("Balance:", accountInfo.amount.toString());
```

## CLI Token Operations

```bash
# Create a new token
spl-token create-token

# Create token account
spl-token create-account <MINT_ADDRESS>

# Mint tokens
spl-token mint <MINT_ADDRESS> 100

# Transfer
spl-token transfer <MINT_ADDRESS> 50 <RECIPIENT_ADDRESS>

# Check balance
spl-token balance <MINT_ADDRESS>

# Get token account info
spl-token account-info --address <TOKEN_ACCOUNT>

# Burn tokens
spl-token burn <TOKEN_ACCOUNT> 10

# Close empty account
spl-token close --address <TOKEN_ACCOUNT>

# Set authorities
spl-token authorize <MINT_ADDRESS> mint <NEW_AUTHORITY>
spl-token authorize <MINT_ADDRESS> freeze <NEW_AUTHORITY>
```
