# Anchor Framework Patterns

## Project Structure (Anchor 0.31+)

```
my_project/
├── Anchor.toml
├── Cargo.toml              # Workspace manifest
├── programs/
│   └── my_program/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs       # Program entry, declare_id!, #[program]
│           ├── instructions/ # Individual instruction handlers
│           │   ├── mod.rs
│           │   ├── initialize.rs
│           │   └── update.rs
│           ├── state/        # Account structs
│           │   ├── mod.rs
│           │   └── my_account.rs
│           ├── error.rs      # Custom errors
│           └── constants.rs  # Seeds, sizes, etc.
├── tests/
│   └── my_program.ts
├── idls/                     # For declare_program! (CPI)
├── migrations/
│   └── deploy.ts
└── target/
```

## Anchor.toml Configuration

```toml
[toolchain]
anchor_version = "0.31.1"
solana_version = "2.1.7"

[features]
resolution = true

[programs.localnet]
my_program = "11111111111111111111111111111111"

[programs.devnet]
my_program = "YOUR_DEVNET_PROGRAM_ID"

[provider]
cluster = "localnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"

[test.validator]
url = "https://api.mainnet-beta.solana.com"

[[test.validator.clone]]
address = "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"

[workspace]
members = ["programs/*"]

[test]
startup_wait = 5000
```

## Account Constraints Reference

### Common Constraints
```rust
#[derive(Accounts)]
pub struct MyInstruction<'info> {
    // Init a new account (PDA)
    #[account(
        init,
        payer = authority,
        space = 8 + MyAccount::INIT_SPACE,
        seeds = [b"my_seed", authority.key().as_ref()],
        bump,
    )]
    pub my_account: Account<'info, MyAccount>,

    // Mutable existing account with validation
    #[account(
        mut,
        seeds = [b"my_seed", authority.key().as_ref()],
        bump,
        has_one = authority,  // my_account.authority == authority.key()
        constraint = my_account.data > 0 @ MyError::InvalidData,
    )]
    pub my_account_mut: Account<'info, MyAccount>,

    // Close an account (reclaim rent)
    #[account(
        mut,
        close = authority,
        has_one = authority,
    )]
    pub closing_account: Account<'info, MyAccount>,

    // Realloc (resize account)
    #[account(
        mut,
        realloc = 8 + MyAccount::INIT_SPACE + new_size,
        realloc::payer = authority,
        realloc::zero = false,
    )]
    pub resizable_account: Account<'info, MyAccount>,

    // Token account constraints
    #[account(
        init,
        payer = authority,
        associated_token::mint = mint,
        associated_token::authority = authority,
    )]
    pub token_account: Account<'info, TokenAccount>,

    #[account(mut)]
    pub authority: Signer<'info>,
    pub mint: Account<'info, Mint>,
    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```

### Account Types
| Type | Description |
|---|---|
| `Account<'info, T>` | Deserialized account with ownership check |
| `Signer<'info>` | Must be a transaction signer |
| `Program<'info, T>` | Validated program account |
| `SystemAccount<'info>` | System-owned account |
| `UncheckedAccount<'info>` | No validation (use `/// CHECK:` doc comment) |
| `AccountInfo<'info>` | Raw account info (no checks) |
| `Box<Account<'info, T>>` | Heap-allocated (reduces stack usage) |
| `Option<Account<'info, T>>` | Optional account |
| `InterfaceAccount<'info, T>` | For Token-2022 compatibility |
| `Interface<'info, T>` | Interface program (Token or Token-2022) |
| `LazyAccount<'info, T>` | Deferred deserialization (Anchor 0.31+) |

## Custom Errors
```rust
#[error_code]
pub enum MyError {
    #[msg("The provided data is invalid")]
    InvalidData,
    #[msg("Unauthorized access")]
    Unauthorized,
    #[msg("Account already initialized")]
    AlreadyInitialized,
    #[msg("Insufficient funds")]
    InsufficientFunds,
    #[msg("Arithmetic overflow")]
    Overflow,
}
```

## Events
```rust
#[event]
pub struct MyEvent {
    pub authority: Pubkey,
    pub data: u64,
    pub timestamp: i64,
}

// Emit in instruction:
emit!(MyEvent {
    authority: ctx.accounts.authority.key(),
    data: 42,
    timestamp: Clock::get()?.unix_timestamp,
});
```

## CPI (Cross-Program Invocation)

### Using declare_program! (Recommended, Anchor 0.30+)
```rust
// Place the target program's IDL at: idls/target_program.json
declare_program!(target_program);

use target_program::cpi::accounts::SomeInstruction;
use target_program::cpi::some_instruction;

pub fn my_cpi_handler(ctx: Context<MyCpi>) -> Result<()> {
    let cpi_ctx = CpiContext::new(
        ctx.accounts.target_program.to_account_info(),
        SomeInstruction {
            account: ctx.accounts.some_account.to_account_info(),
            signer: ctx.accounts.authority.to_account_info(),
        },
    );
    some_instruction(cpi_ctx, args)?;
    Ok(())
}
```

### CPI with PDA Signer
```rust
let seeds = &[
    b"my_pda",
    authority.key().as_ref(),
    &[bump],
];
let signer_seeds = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    ctx.accounts.target_program.to_account_info(),
    SomeInstruction { ... },
    signer_seeds,
);
```

### System Program CPI (Transfer SOL)
```rust
use anchor_lang::system_program;

system_program::transfer(
    CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        system_program::Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
        },
    ),
    amount,
)?;
```

## Space Calculation

### Using InitSpace derive (Recommended)
```rust
#[account]
#[derive(InitSpace)]
pub struct MyAccount {
    pub authority: Pubkey,     // 32
    pub data: u64,             // 8
    pub is_active: bool,       // 1
    #[max_len(32)]
    pub name: String,          // 4 + 32 = 36
    #[max_len(10)]
    pub items: Vec<u64>,       // 4 + (10 * 8) = 84
    pub optional: Option<u64>, // 1 + 8 = 9
}
// Total space = 8 (discriminator) + InitSpace
```

### Manual calculation
```
Account discriminator: 8 bytes
Pubkey: 32 bytes
u64/i64: 8 bytes
u32/i32: 4 bytes
u16/i16: 2 bytes
u8/i8/bool: 1 byte
String: 4 (len prefix) + content bytes
Vec<T>: 4 (len prefix) + (count * size_of::<T>())
Option<T>: 1 + size_of::<T>()
Enum: 1 (variant) + largest variant size
```

## Deployment

### Localnet
```bash
anchor test  # Auto starts validator, deploys, tests
```

### Devnet
```bash
solana config set --url devnet
solana airdrop 5

anchor build
anchor deploy --provider.cluster devnet

# Or with priority fees:
anchor deploy --provider.cluster devnet -- --with-compute-unit-price 10000
```

### Mainnet
```bash
solana config set --url mainnet-beta
anchor build --verifiable  # For verifiable builds
anchor deploy --provider.cluster mainnet -- --with-compute-unit-price 50000

# Upload IDL (automatic in 0.32+, use --no-idl to skip)
anchor idl init --filepath target/idl/my_program.json <PROGRAM_ID>
# Or upgrade existing:
anchor idl upgrade --filepath target/idl/my_program.json <PROGRAM_ID>
```

### Program Upgrade
```bash
# Deploy upgrade
anchor upgrade target/deploy/my_program.so --program-id <PROGRAM_ID>

# Set new upgrade authority
solana program set-upgrade-authority <PROGRAM_ID> --new-upgrade-authority <NEW_AUTHORITY>

# Make immutable (WARNING: irreversible!)
solana program set-upgrade-authority <PROGRAM_ID> --final
```
