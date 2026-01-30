# Solana Development Environment Install Guide

## Quick Install (One Command)

For new setups on supported systems, Solana provides a single install command:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://solana.com/install.sh | sh
```
This installs Rust, Solana CLI (stable), and Anchor CLI. After install, verify:
```bash
rustc --version && solana --version && anchor --version
```

If this works, skip to [Post-Install Configuration](#post-install-configuration). If not, follow the manual steps below.

---

## Manual Install: Modern Stack (Anchor 0.31.x)

### Prerequisites
- **Linux:** Ubuntu 24.04+ (for GLIBC ≥2.38) or Fedora 39+, Arch
- **macOS:** 14+ (Sonoma) recommended, 12+ works
- **Windows:** WSL2 with Ubuntu 24.04

#### Linux Dependencies
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y \
  build-essential pkg-config libudev-dev llvm libclang-dev \
  protobuf-compiler libssl-dev curl git

# Fedora
sudo dnf install -y gcc make pkgconfig systemd-devel llvm clang \
  protobuf-compiler openssl-devel curl git
```

### Step 1: Install Rust
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustc --version
# Should show: rustc 1.83.x or similar
```

### Step 2: Install Solana CLI
```bash
# Install stable (recommended)
sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"

# Or install a specific version
sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.7/install)"

# Add to PATH (add to your .bashrc/.zshrc)
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"

solana --version
# Should show: solana-cli 2.1.x
```

### Step 3: Install Anchor CLI via AVM
```bash
# Install AVM (Anchor Version Manager)
cargo install --git https://github.com/coral-xyz/anchor avm --force

# Install latest Anchor
avm install latest
avm use latest

# Verify
anchor --version
# Should show: anchor-cli 0.31.1
```

### Step 4: Install Node.js (for TypeScript tests)
```bash
# Using nvm (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
source ~/.bashrc  # or ~/.zshrc
nvm install 20
nvm use 20

# Install yarn
npm install -g yarn

node --version   # v20.x.x
yarn --version   # 1.22.x
```

---

## Manual Install: Legacy Stack (Anchor 0.30.x)

Use this if you need to support Ubuntu 20.04/22.04 or older systems.

### Step 1: Install Rust 1.79.0 (pinned)
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain 1.79.0
source "$HOME/.cargo/env"
rustc --version
# Must show: rustc 1.79.0
```

### Step 2: Install Solana CLI 1.18.x
```bash
sh -c "$(curl -sSfL https://release.anza.xyz/v1.18.26/install)"
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
solana --version
```

### Step 3: Install Anchor 0.30.1
```bash
cargo install --git https://github.com/coral-xyz/anchor --tag v0.30.1 anchor-cli --locked
anchor --version
```

---

## Docker-Based Development

For systems where native install is problematic, use Docker.

### Using the Official Anchor Docker Image
```bash
# Pull the image
docker pull solanafoundation/anchor:0.31.1

# Build inside Docker
docker run -v $(pwd):/workspace -w /workspace \
  solanafoundation/anchor:0.31.1 \
  anchor build

# Run tests (needs more setup for test-validator)
docker run -v $(pwd):/workspace -w /workspace \
  -p 8899:8899 -p 8900:8900 \
  solanafoundation/anchor:0.31.1 \
  bash -c "solana-test-validator & sleep 5 && anchor test --skip-local-validator"
```

### Custom Dockerfile
```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    build-essential pkg-config libudev-dev llvm libclang-dev \
    protobuf-compiler libssl-dev curl git nodejs npm && \
    npm install -g yarn

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Install Solana CLI
RUN sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.7/install)"
ENV PATH="/root/.local/share/solana/install/active_release/bin:${PATH}"

# Install Anchor CLI
RUN cargo install --git https://github.com/coral-xyz/anchor avm --force && \
    avm install 0.31.1 && avm use 0.31.1

# Generate keypair
RUN solana-keygen new --no-bip39-passphrase

WORKDIR /workspace
```

---

## Upgrading Between Versions

### Upgrading Solana CLI
```bash
# Upgrade to latest stable
agave-install init stable

# Or specific version
agave-install init 2.1.7

# Verify
solana --version
```

### Upgrading Anchor via AVM
```bash
# List available versions
avm list

# Install and switch
avm install 0.31.1
avm use 0.31.1

# Verify
anchor --version
```

### Upgrading Rust
```bash
rustup update stable
rustc --version
```

⚠️ **Warning:** Upgrading Rust past 1.79.0 will break Anchor 0.30.x builds. Use AVM which handles this automatically, or stay on Anchor 0.31+.

---

## Migration: Anchor 0.29 → 0.30

1. **Update dependencies:**
   ```toml
   # Cargo.toml
   [dependencies]
   anchor-lang = "0.30.1"
   anchor-spl = "0.30.1"
   
   [features]
   idl-build = ["anchor-lang/idl-build", "anchor-spl/idl-build"]
   ```

2. **Update Anchor.toml:**
   ```toml
   [toolchain]
   anchor_version = "0.30.1"
   ```

3. **Add overflow-checks to workspace Cargo.toml:**
   ```toml
   [profile.release]
   overflow-checks = true
   ```

4. **Update TypeScript:**
   ```bash
   yarn add @coral-xyz/anchor@0.30.1
   ```

5. **Fix `.accounts()` calls:**
   - Change to `.accountsPartial()` if you were relying on the old behavior
   - Remove system programs/sysvars from account lists (now auto-resolved)

6. **Fix IDL imports:**
   - The IDL format has changed. Regenerate with `anchor build`.
   - If using `declare_program!`, place IDL in `idls/` directory.

## Migration: Anchor 0.30 → 0.31

1. **System requirements:** Need GLIBC ≥2.38 (Ubuntu 24.04+ or macOS)

2. **Update dependencies:**
   ```toml
   [dependencies]
   anchor-lang = "0.31.1"
   anchor-spl = "0.31.1"
   ```

3. **Update Solana CLI to v2:**
   ```bash
   sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.7/install)"
   ```

4. **Remove direct `solana-program` dependency.** Use through `anchor-lang`:
   ```rust
   // Use anchor's re-exports
   use anchor_lang::prelude::*;
   ```

5. **Update TypeScript:**
   ```bash
   yarn add @coral-xyz/anchor@0.31.1
   ```

6. **Update Docker images (if using `anchor verify`):**
   - Change from `backpackapp/build` to `solanafoundation/anchor`

## Migration: bankrun → LiteSVM

[LiteSVM](https://github.com/LiteSVM/litesvm) is the modern replacement for `solana-program-test` and the `bankrun` JS wrapper.

### Rust Tests
```toml
# Cargo.toml
[dev-dependencies]
litesvm = "0.6"  # Check latest on crates.io
```

```rust
// Before (solana-program-test):
use solana_program_test::*;
let (mut banks_client, payer, recent_blockhash) = ProgramTest::new(
    "my_program",
    my_program::ID,
    processor!(my_program::entry),
).start().await;

// After (litesvm):
use litesvm::LiteSVM;
let mut svm = LiteSVM::new();
svm.add_program_from_file(my_program::ID, "target/deploy/my_program.so");
svm.airdrop(&payer.pubkey(), 10_000_000_000).unwrap();
let tx_result = svm.send_transaction(tx).unwrap();
```

### TypeScript Tests (bankrun → litesvm)
```bash
# Remove old
yarn remove solana-bankrun

# Install new
yarn add litesvm
```

```typescript
// Before (bankrun):
import { start } from "solana-bankrun";
const context = await start([{ name: "my_program", programId }], []);

// After (litesvm):
import { LiteSVM } from "litesvm";
const svm = LiteSVM.default();
svm.addProgramFromFile(programId, "target/deploy/my_program.so");
```

---

## Post-Install Configuration

### Set Solana CLI to localhost for development
```bash
solana config set --url localhost
# or for devnet:
solana config set --url devnet
```

### Generate a development keypair
```bash
solana-keygen new --no-bip39-passphrase
# Creates ~/.config/solana/id.json
```

### Verify everything works
```bash
# Create a test project
anchor init my_test_project
cd my_test_project

# Build
anchor build

# Test
anchor test
```

### Configure Anchor.toml toolchain
```toml
[toolchain]
anchor_version = "0.31.1"
solana_version = "2.1.7"

[features]
resolution = true

[programs.localnet]
my_program = "YOUR_PROGRAM_ID_HERE"

[provider]
cluster = "localnet"
wallet = "~/.config/solana/id.json"
```

---

## Troubleshooting

### Version Mismatch Detection
```bash
# Quick check script - save as check-versions.sh
#!/bin/bash
echo "=== Solana Dev Environment ==="
echo "Rust:    $(rustc --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Solana:  $(solana --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Anchor:  $(anchor --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "AVM:     $(avm --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Node:    $(node --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Yarn:    $(yarn --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "OS:      $(uname -a)"
if command -v ldd &>/dev/null; then
    echo "GLIBC:   $(ldd --version 2>&1 | head -1)"
fi
echo ""
echo "=== Anchor.toml Toolchain ==="
cat Anchor.toml 2>/dev/null | grep -A10 '\[toolchain\]' || echo "No Anchor.toml found"
```

### Nuclear Option: Complete Clean Reinstall
```bash
# WARNING: This removes everything
rustup self uninstall
rm -rf ~/.local/share/solana
rm -rf ~/.cache/solana
rm -rf ~/.avm
rm -rf ~/.cargo

# Then follow the install steps above from scratch
```
