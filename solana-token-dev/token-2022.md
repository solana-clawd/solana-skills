# Token-2022 (Token Extensions)

## Program IDs
```
Token-2022:          TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb
Associated Token:    ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL (same for both)
```

## Overview
Token-2022 is the next-generation token program with built-in extensions. It's backward-compatible with SPL Token for basic operations but adds powerful features.

## Available Extensions

| Extension | Description |
|---|---|
| **Transfer Fees** | Automatic fee collection on transfers |
| **Transfer Hook** | Custom logic on every transfer (CPI to your program) |
| **Confidential Transfers** | Zero-knowledge encrypted balances |
| **Metadata** | On-chain metadata (name, symbol, URI) without Metaplex |
| **Metadata Pointer** | Points to metadata account |
| **Group Pointer** | Group token collections |
| **Group Member Pointer** | Member of a group |
| **Permanent Delegate** | Authority that can always transfer/burn |
| **Non-Transferable** | Soulbound tokens |
| **Close Authority** | Authority to close the mint |
| **Interest-Bearing** | Display interest accrual |
| **Default Account State** | Accounts start frozen |
| **CPI Guard** | Prevent CPI-based exploits |
| **Immutable Owner** | Token account owner can't change |
| **Required Memo** | Must include memo with transfers |

## Anchor Token-2022 Support (0.30+)

### Dependencies
```toml
[dependencies]
anchor-lang = "0.31.1"
anchor-spl = { version = "0.31.1", features = ["idl-build"] }
```

### Using Interface Types (Token-2022 Compatible)
```rust
use anchor_spl::{
    token_interface::{Mint, TokenAccount, TokenInterface},
    associated_token::AssociatedToken,
};

#[derive(Accounts)]
pub struct MyInstruction<'info> {
    // Works with BOTH Token and Token-2022
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(
        associated_token::mint = mint,
        associated_token::authority = authority,
        associated_token::token_program = token_program,
    )]
    pub token_account: InterfaceAccount<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

### Creating a Token-2022 Mint with Extensions (Anchor)

#### Transfer Fee Extension
```rust
#[derive(Accounts)]
pub struct CreateFeeToken<'info> {
    #[account(
        init,
        payer = authority,
        mint::decimals = 9,
        mint::authority = authority.key(),
        mint::freeze_authority = authority.key(),
        mint::token_program = token_program,
        extensions::transfer_fee::transfer_fee_config_authority = authority.key(),
        extensions::transfer_fee::withdraw_withheld_authority = authority.key(),
        extensions::transfer_fee::transfer_fee_basis_points = 100, // 1%
        extensions::transfer_fee::maximum_fee = 1_000_000_000,
    )]
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

#### Metadata Extension
```rust
#[derive(Accounts)]
pub struct CreateMetadataToken<'info> {
    #[account(
        init,
        payer = authority,
        mint::decimals = 9,
        mint::authority = authority.key(),
        mint::token_program = token_program,
        extensions::metadata_pointer::authority = authority.key(),
        extensions::metadata_pointer::metadata_address = mint,
    )]
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

#### Transfer Hook Extension
```rust
#[derive(Accounts)]
pub struct CreateHookToken<'info> {
    #[account(
        init,
        payer = authority,
        mint::decimals = 9,
        mint::authority = authority.key(),
        mint::token_program = token_program,
        extensions::transfer_hook::authority = authority.key(),
        extensions::transfer_hook::program_id = my_hook_program::ID,
    )]
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub token_program: Interface<'info, TokenInterface>,
}
```

#### Close Authority Extension
```rust
extensions::close_authority::authority = authority.key(),
```

#### Permanent Delegate Extension
```rust
extensions::permanent_delegate::delegate = delegate.key(),
```

#### Group Pointer Extension
```rust
extensions::group_pointer::authority = authority.key(),
extensions::group_pointer::group_address = mint,
```

#### Group Member Pointer Extension
```rust
extensions::group_member_pointer::authority = authority.key(),
extensions::group_member_pointer::member_address = mint,
```

### Transfer Hook Implementation

Transfer hooks allow custom logic on every token transfer. Your program must implement the `spl_transfer_hook_interface::execute` interface.

```rust
use anchor_lang::prelude::*;

declare_id!("YOUR_HOOK_PROGRAM_ID");

#[program]
pub mod transfer_hook {
    use super::*;

    // Required: implement the Execute interface
    #[interface(spl_transfer_hook_interface::execute)]
    pub fn execute_transfer(ctx: Context<ExecuteTransfer>, amount: u64) -> Result<()> {
        // Custom transfer logic here
        msg!("Transfer hook executed for amount: {}", amount);
        Ok(())
    }

    // Required: initialize extra account metas
    #[interface(spl_transfer_hook_interface::initialize_extra_account_meta_list)]
    pub fn initialize_extra_account_meta_list(
        ctx: Context<InitializeExtraAccountMetaList>,
    ) -> Result<()> {
        // Define additional accounts needed during transfer
        Ok(())
    }
}
```

## TypeScript Token-2022 Operations

```typescript
import {
    createMint,
    createInitializeTransferFeeConfigInstruction,
    createInitializeMetadataPointerInstruction,
    ExtensionType,
    getMintLen,
    TOKEN_2022_PROGRAM_ID,
} from "@solana/spl-token";

// Calculate mint account size with extensions
const extensions = [ExtensionType.TransferFeeConfig];
const mintLen = getMintLen(extensions);

// Create mint with transfer fee
const mintKeypair = Keypair.generate();
const tx = new Transaction().add(
    SystemProgram.createAccount({
        fromPubkey: payer.publicKey,
        newAccountPubkey: mintKeypair.publicKey,
        space: mintLen,
        lamports: await connection.getMinimumBalanceForRentExemption(mintLen),
        programId: TOKEN_2022_PROGRAM_ID,
    }),
    createInitializeTransferFeeConfigInstruction(
        mintKeypair.publicKey,
        feeConfigAuthority,
        withdrawWithheldAuthority,
        100,              // 1% (basis points)
        BigInt(1000000),  // max fee
        TOKEN_2022_PROGRAM_ID,
    ),
    createInitializeMintInstruction(
        mintKeypair.publicKey,
        9,
        mintAuthority,
        freezeAuthority,
        TOKEN_2022_PROGRAM_ID,
    ),
);
```

## CLI Token-2022 Operations

```bash
# Create Token-2022 token with transfer fee
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --transfer-fee 100 1000000000

# Create token with metadata
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-metadata

# Initialize metadata
spl-token initialize-metadata <MINT> "My Token" "MTK" "https://example.com/metadata.json"

# Create non-transferable token
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-non-transferable

# Create token with permanent delegate
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-permanent-delegate

# Harvest withheld fees (transfer fee tokens)
spl-token withdraw-withheld-tokens <MINT> <DESTINATION>
```

## Token-2022 vs SPL Token Decision Matrix

| Feature | SPL Token | Token-2022 |
|---|---|---|
| Basic mint/transfer/burn | ✅ | ✅ |
| DeFi protocol support | ✅ Wide | ⚠️ Growing |
| Transfer fees | ❌ | ✅ |
| On-chain metadata | ❌ | ✅ |
| Transfer hooks | ❌ | ✅ |
| Confidential transfers | ❌ | ✅ |
| Non-transferable | ❌ | ✅ |
| Permanent delegate | ❌ | ✅ |
| Interest bearing | ❌ | ✅ |
| Default frozen state | ❌ | ✅ |
| CPI Guard | ❌ | ✅ |
| Wallet support | ✅ Universal | ✅ Most major wallets |
