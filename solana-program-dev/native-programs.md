# Native Rust Program Development (Without Anchor)

## When to Use Native
- Maximum control over serialization/deserialization
- Minimal binary size (Anchor adds ~30-50KB overhead)
- Learning how Solana works at a lower level
- Programs that don't need Anchor's account validation macros

## Program Structure

```
my_native_program/
├── Cargo.toml
├── src/
│   ├── lib.rs           # Entrypoint
│   ├── processor.rs     # Instruction processing
│   ├── instruction.rs   # Instruction enum & deserialization
│   ├── state.rs         # Account data structures
│   └── error.rs         # Custom errors
```

### Cargo.toml
```toml
[package]
name = "my-program"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "lib"]

[dependencies]
solana-program = "2.1"  # or use smaller crates for Solana v2
borsh = "1.5"
thiserror = "2.0"

# For Solana v2 smaller crate approach:
# solana-account-info = "2.1"
# solana-entrypoint = "2.1"
# solana-msg = "2.1"
# solana-pubkey = "2.1"
# solana-program-error = "2.1"
# solana-instruction = "2.1"
```

## Entrypoint (lib.rs)
```rust
use solana_program::{
    account_info::AccountInfo,
    entrypoint,
    entrypoint::ProgramResult,
    pubkey::Pubkey,
};

mod processor;
mod instruction;
mod state;
mod error;

entrypoint!(process_instruction);

fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    processor::process(program_id, accounts, instruction_data)
}
```

## Instruction Enum (instruction.rs)
```rust
use borsh::{BorshDeserialize, BorshSerialize};

#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub enum MyInstruction {
    /// Initialize a new account
    /// Accounts:
    /// 0. `[writable, signer]` New account
    /// 1. `[signer]` Authority
    /// 2. `[]` System program
    Initialize { data: u64 },

    /// Update account data
    /// Accounts:
    /// 0. `[writable]` Account to update
    /// 1. `[signer]` Authority
    Update { new_data: u64 },
}
```

## State (state.rs)
```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::pubkey::Pubkey;

pub const ACCOUNT_DISCRIMINATOR: [u8; 8] = [0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08];
pub const ACCOUNT_SIZE: usize = 8 + 32 + 8 + 1; // disc + pubkey + u64 + bool

#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct MyAccount {
    pub discriminator: [u8; 8],
    pub authority: Pubkey,
    pub data: u64,
    pub is_initialized: bool,
}
```

## Processor (processor.rs)
```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint::ProgramResult,
    msg,
    program::invoke_signed,
    program_error::ProgramError,
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::Sysvar,
};

use crate::instruction::MyInstruction;
use crate::state::{MyAccount, ACCOUNT_DISCRIMINATOR, ACCOUNT_SIZE};
use crate::error::MyError;

pub fn process(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = MyInstruction::try_from_slice(instruction_data)
        .map_err(|_| ProgramError::InvalidInstructionData)?;

    match instruction {
        MyInstruction::Initialize { data } => {
            process_initialize(program_id, accounts, data)
        }
        MyInstruction::Update { new_data } => {
            process_update(program_id, accounts, new_data)
        }
    }
}

fn process_initialize(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    data: u64,
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
    let authority = next_account_info(accounts_iter)?;
    let system_program = next_account_info(accounts_iter)?;

    // Validate signer
    if !authority.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // Derive PDA
    let (pda, bump) = Pubkey::find_program_address(
        &[b"my_account", authority.key.as_ref()],
        program_id,
    );

    if pda != *account.key {
        return Err(ProgramError::InvalidSeeds);
    }

    // Create account via CPI
    let rent = Rent::get()?;
    let lamports = rent.minimum_balance(ACCOUNT_SIZE);

    invoke_signed(
        &system_instruction::create_account(
            authority.key,
            account.key,
            lamports,
            ACCOUNT_SIZE as u64,
            program_id,
        ),
        &[authority.clone(), account.clone(), system_program.clone()],
        &[&[b"my_account", authority.key.as_ref(), &[bump]]],
    )?;

    // Serialize state
    let state = MyAccount {
        discriminator: ACCOUNT_DISCRIMINATOR,
        authority: *authority.key,
        data,
        is_initialized: true,
    };
    state.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Account initialized with data: {}", data);
    Ok(())
}

fn process_update(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    new_data: u64,
) -> ProgramResult {
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
    let authority = next_account_info(accounts_iter)?;

    // Validate
    if !authority.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }
    if account.owner != program_id {
        return Err(ProgramError::IncorrectProgramId);
    }

    // Deserialize
    let mut state = MyAccount::try_from_slice(&account.data.borrow())?;

    if !state.is_initialized {
        return Err(MyError::NotInitialized.into());
    }
    if state.authority != *authority.key {
        return Err(MyError::Unauthorized.into());
    }

    // Update
    state.data = new_data;
    state.serialize(&mut &mut account.data.borrow_mut()[..])?;

    msg!("Data updated to: {}", new_data);
    Ok(())
}
```

## Custom Errors (error.rs)
```rust
use solana_program::program_error::ProgramError;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("Account not initialized")]
    NotInitialized,
    #[error("Unauthorized")]
    Unauthorized,
    #[error("Already initialized")]
    AlreadyInitialized,
}

impl From<MyError> for ProgramError {
    fn from(e: MyError) -> Self {
        ProgramError::Custom(e as u32)
    }
}
```

## Building
```bash
cargo build-sbf

# Output: target/deploy/my_program.so
```

## Key Differences from Anchor

| Feature | Anchor | Native |
|---|---|---|
| Account validation | Automatic via macros | Manual checks required |
| Serialization | Automatic (Borsh) | Manual (choose format) |
| Error handling | `#[error_code]` | Custom `ProgramError` impl |
| IDL generation | Automatic | Manual or none |
| Binary size | ~100-200 KB | ~30-80 KB |
| CPI safety | Type-checked | Manual account passing |
| PDA validation | `seeds` constraint | Manual `find_program_address` |
| Account discriminator | Automatic 8 bytes | Manual (optional) |

## Security Checklist (Native Programs)
- [ ] All signers verified with `is_signer`
- [ ] Account ownership checked (`account.owner == program_id`)
- [ ] PDAs verified against expected derivation
- [ ] Writable accounts checked with `is_writable`
- [ ] Account data size validated before deserialization
- [ ] Arithmetic overflow checked (use `checked_*` methods)
- [ ] Rent exemption verified for new accounts
- [ ] Duplicate accounts handled (no double-spend within instruction)
- [ ] Close account properly: zero data, transfer lamports, set owner to system program
