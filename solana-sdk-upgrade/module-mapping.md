# Solana SDK v2 → v3 Module Mapping

Complete mapping of removed modules to their replacement crates.

## solana-sdk Module Removals

| Removed Module (`solana_sdk::*`) | Replacement Crate | Replacement Path |
|---|---|---|
| `address_lookup_table` | `solana_address_lookup_table_interface` | `solana_address_lookup_table_interface` |
| `alt_bn128` | `solana_bn254` | `solana_bn254` |
| `bpf_loader_upgradeable` | `solana_loader_v3_interface` | `solana_loader_v3_interface` |
| `client` | `solana_client_traits` | `solana_client_traits` |
| `commitment_config` | `solana_commitment_config` | `solana_commitment_config` |
| `compute_budget` | `solana_compute_budget_interface` | `solana_compute_budget_interface` |
| `decode_error` | `solana_decode_error` | `solana_decode_error` |
| `derivation_path` | `solana_derivation_path` | `solana_derivation_path` |
| `ed25519_instruction` | `solana_ed25519_program` | `solana_ed25519_program` |
| `exit` | `solana_validator_exit` | `solana_validator_exit` |
| `feature_set` | `agave_feature_set` | `agave_feature_set` |
| `feature` | `solana_feature_gate_interface` | `solana_feature_gate_interface` |
| `genesis_config` | `solana_genesis_config` | `solana_genesis_config` |
| `hard_forks` | `solana_hard_forks` | `solana_hard_forks` |
| `loader_instruction` | `solana_loader_v2_interface` | `solana_loader_v2_interface` |
| `loader_upgradeable_instruction` | `solana_loader_v3_interface` | `solana_loader_v3_interface::instruction` |
| `loader_v4` | `solana_loader_v4_interface` | `solana_loader_v4_interface` |
| `loader_v4_instruction` | `solana_loader_v4_interface` | `solana_loader_v4_interface::instruction` |
| `nonce` | `solana_nonce` | `solana_nonce` |
| `nonce_account` | `solana_nonce_account` | `solana_nonce_account` |
| `packet` | `solana_packet` | `solana_packet` |
| `poh_config` | `solana_poh_config` | `solana_poh_config` |
| `precompiles` | `agave_precompiles` | `agave_precompiles` |
| `program_utils` | `solana_bincode` | `solana_bincode::limited_deserialize` |
| `quic` | `solana_quic_definitions` | `solana_quic_definitions` |
| `reserved_account_keys` | `agave_reserved_account_keys` | `agave_reserved_account_keys` |
| `reward_info` | `solana_reward_info` | `solana_reward_info` |
| `reward_type` | `solana_reward_info` | `solana_reward_info` |
| `sdk_ids` | `solana_sdk_ids` | `solana_sdk_ids` |
| `secp256k1_instruction` | `solana_secp256k1_program` | `solana_secp256k1_program` |
| `secp256k1_recover` | `solana_secp256k1_recover` | `solana_secp256k1_recover` |
| `stake` | `solana_stake_interface` | `solana_stake_interface` |
| `stake_history` | `solana_stake_interface` | `solana_stake_interface::stake_history` |
| `system_instruction` | `solana_system_interface` | `solana_system_interface::instruction` |
| `system_program` | `solana_system_interface` | `solana_system_interface::program` |
| `system_transaction` | `solana_system_transaction` | `solana_system_transaction` |
| `transaction_context` | `solana_transaction_context` | `solana_transaction_context` |
| `vote` | `solana_vote_interface` | `solana_vote_interface` |

## solana-program Module Removals

| Removed Module (`solana_program::*`) | Replacement Crate | Replacement Path |
|---|---|---|
| `address_lookup_table` | `solana_address_lookup_table_interface` | `solana_address_lookup_table_interface` |
| `bpf_loader_upgradeable` | `solana_loader_v3_interface` | `solana_loader_v3_interface` |
| `decode_error` | `solana_decode_error` | `solana_decode_error` |
| `feature` | `solana_feature_gate_interface` | `solana_feature_gate_interface` |
| `loader_instruction` | `solana_loader_v2_interface` | `solana_loader_v2_interface` |
| `loader_upgradeable_instruction` | `solana_loader_v3_interface` | `solana_loader_v3_interface::instruction` |
| `loader_v4` | `solana_loader_v4_interface` | `solana_loader_v4_interface` |
| `loader_v4_instruction` | `solana_loader_v4_interface` | `solana_loader_v4_interface::instruction` |
| `message` | `solana_message` | `solana_message` |
| `nonce` | `solana_nonce` | `solana_nonce` |
| `program_utils` | `solana_bincode` | `solana_bincode::limited_deserialize` |
| `sanitize` | `solana_sanitize` | `solana_sanitize` |
| `sdk_ids` | `solana_sdk_ids` | `solana_sdk_ids` |
| `stake` | `solana_stake_interface` | `solana_stake_interface` |
| `stake_history` | `solana_stake_interface` | `solana_stake_interface::stake_history` |
| `system_instruction` | `solana_system_interface` | `solana_system_interface::instruction` |
| `system_program` | `solana_system_interface` | `solana_system_interface::program` |
| `vote` | `solana_vote_interface` | `solana_vote_interface` |

## SPL Crate Mapping

| v2 Crate (Program) | v3 Crate (Interface) | Notes |
|---|---|---|
| `spl-token` v6-v8 | `spl-token-interface` v2 | State, instruction, error modules preserved |
| `spl-token-2022` v5+ | `spl-token-2022-interface` v2 | Lighter deps, LTO-compatible |
| `spl-associated-token-account` v4+ | `spl-associated-token-account-interface` v2 | |
| `spl-memo` | `spl-memo-interface` v2 | |

### Why Interface Crates?

Program crates (like `spl-token`) contain a `cdylib` target, which prevents the Rust compiler from running Link-Time Optimization (LTO). Interface crates only declare a `lib` target with fewer dependencies, enabling:

```bash
cargo build-sbf --lto
```

> **NOTE:** LTO results are mixed — always profile program size and CU usage with and without `--lto`.

## Quick sed Commands for Common Renames

```bash
# System instructions
sed -i 's/solana_program::system_instruction/solana_system_interface::instruction/g' src/**/*.rs
sed -i 's/solana_sdk::system_instruction/solana_system_interface::instruction/g' src/**/*.rs

# System program
sed -i 's/solana_program::system_program/solana_system_interface::program/g' src/**/*.rs
sed -i 's/solana_sdk::system_program/solana_system_interface::program/g' src/**/*.rs

# Message
sed -i 's/solana_program::message/solana_message/g' src/**/*.rs

# Stake
sed -i 's/solana_program::stake/solana_stake_interface/g' src/**/*.rs
sed -i 's/solana_sdk::stake/solana_stake_interface/g' src/**/*.rs

# Vote
sed -i 's/solana_program::vote/solana_vote_interface/g' src/**/*.rs
sed -i 's/solana_sdk::vote/solana_vote_interface/g' src/**/*.rs

# SPL Token
sed -i 's/use spl_token::/use spl_token_interface::/g' src/**/*.rs
sed -i 's/use spl_token_2022::/use spl_token_2022_interface::/g' src/**/*.rs
```
