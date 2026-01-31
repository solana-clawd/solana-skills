# Solana SDK v3 Breaking Changes

## Address / Pubkey

SDK v3 introduces the `Address` type as a better-named, more flexible version of `Pubkey`. `Pubkey` becomes a type alias for `Address`.

```rust
// v3: Pubkey is now an alias for Address
// If you see type errors like "expected Address, found Pubkey":
// It means one of your dependencies hasn't upgraded to v3 yet.

// Both work in v3:
use solana_pubkey::Pubkey;    // alias
use solana_address::Address;  // new canonical name
```

**Action:** If you get `Address` vs `Pubkey` type mismatches, it means a dependency is still on v2. Upgrade that dependency or use `.into()` conversions temporarily.

## AccountInfo

The `rent_epoch` field has been removed. It's now called `_unused`.

```rust
// v2: AccountInfo had rent_epoch as the last parameter to ::new()
let info = AccountInfo::new(
    key, is_signer, is_writable, lamports,
    data, owner, executable, rent_epoch  // ← removed in v3
);

// v3: The final parameter is gone
let info = AccountInfo::new(
    key, is_signer, is_writable, lamports,
    data, owner, executable
);
```

**Action:** Remove the last argument from `AccountInfo::new()` calls. If you were reading `rent_epoch`, stop — it's unused on-chain.

## Hash

The inner bytes of `Hash` are now private.

```rust
// v2:
let bytes = hash.0;  // direct field access

// v3:
let bytes = hash.as_bytes();  // use accessor
// or
let bytes: &[u8; 32] = hash.as_ref();
```

**Action:** Replace `hash.0` with `hash.as_bytes()`.

## Keypair

`Keypair::from_bytes` is removed. Use `try_from` instead.

```rust
// v2:
let keypair = Keypair::from_bytes(&bytes)?;

// v3:
let keypair = Keypair::try_from(&bytes[..])?;
// or
let keypair = Keypair::try_from(bytes.as_slice())?;
```

**Action:** Replace `Keypair::from_bytes()` with `Keypair::try_from()`.

## InstructionError / ProgramError — BorshIoError

The `BorshIoError` variant no longer carries a `String`.

```rust
// v2:
InstructionError::BorshIoError("something went wrong".to_string())
ProgramError::BorshIoError("something went wrong".to_string())

// v3:
InstructionError::BorshIoError
ProgramError::BorshIoError
```

**Action:** Remove the string parameter from `BorshIoError` variants. If you were matching on the string, match on the variant alone.

## Program Memory — Unsafe Blocks

All on-chain memory operations (`sol_memcpy`, `sol_memset`, `sol_memmove`, `sol_memcmp`) are now marked `unsafe`.

```rust
// v2:
solana_program::program_memory::sol_memcpy(dst, src, len);

// v3:
unsafe {
    solana_program::program_memory::sol_memcpy(dst, src, len);
}
```

**Action:** Wrap all `sol_mem*` calls in `unsafe` blocks. This is a safety improvement — these operations can cause UB if misused.

## Sysvar — SysvarSerialize Import

If using `Sysvar::from_account_info`, you now need to import `SysvarSerialize`:

```rust
// v2:
use solana_program::sysvar::Sysvar;
let clock = Clock::from_account_info(&clock_info)?;

// v3:
use solana_sysvar::{Sysvar, SysvarSerialize};  // need both
let clock = Clock::from_account_info(&clock_info)?;
```

**Action:** Add `SysvarSerialize` to your sysvar imports.

## StakeHistory Location

`StakeHistory` moved from `solana_sysvar` to `solana_stake_interface`.

```rust
// v2:
use solana_program::sysvar::stake_history::StakeHistory;
// or
use solana_sysvar::stake_history::StakeHistory;

// v3:
use solana_stake_interface::stake_history::StakeHistory;
```

## VoteState Renames

```rust
// v2:
use solana_vote_interface::VoteState;
VoteState::new_current(...);
convert_to_current(...);

// v3:
use solana_vote_interface::VoteStateV3;
VoteStateV3::new_v3(...);
convert_to_v3(...);
```

## Genesis — ClusterType

`ClusterType` moved to its own crate:

```rust
// v2:
use solana_sdk::genesis_config::ClusterType;

// v3:
use solana_cluster_type::ClusterType;
```

## Summary Checklist

- [ ] Remove `rent_epoch` from `AccountInfo::new()` calls
- [ ] Replace `hash.0` with `hash.as_bytes()`
- [ ] Replace `Keypair::from_bytes()` with `Keypair::try_from()`
- [ ] Remove string from `BorshIoError` variants
- [ ] Wrap `sol_mem*` calls in `unsafe` blocks
- [ ] Add `SysvarSerialize` import alongside `Sysvar`
- [ ] Update `StakeHistory` import path
- [ ] Rename `VoteState` → `VoteStateV3`, `convert_to_current` → `convert_to_v3`
- [ ] Update `ClusterType` import
- [ ] Replace all removed module imports (see [module-mapping.md](./module-mapping.md))
