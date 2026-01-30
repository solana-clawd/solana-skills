# Token Metadata (Metaplex)

## Overview
Metaplex Token Metadata is the dominant metadata standard on Solana. It adds rich metadata (name, symbol, image, attributes) to any SPL Token or Token-2022 mint.

## Program IDs
```
Token Metadata:      metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s
```

## Metadata Account Structure

Each mint can have an associated **Metadata PDA** derived as:
```
seeds = ["metadata", metadata_program_id, mint_pubkey]
```

### Metadata Fields
```json
{
    "name": "My Token",
    "symbol": "MTK",
    "uri": "https://arweave.net/xxx",
    "seller_fee_basis_points": 500,
    "creators": [
        {
            "address": "...",
            "verified": true,
            "share": 100
        }
    ],
    "collection": {
        "verified": true,
        "key": "COLLECTION_MINT_ADDRESS"
    }
}
```

### Off-Chain Metadata (JSON at URI)
```json
{
    "name": "My NFT #1",
    "symbol": "MNFT",
    "description": "A description of the NFT",
    "image": "https://arweave.net/image.png",
    "animation_url": "https://arweave.net/video.mp4",
    "external_url": "https://myproject.com",
    "attributes": [
        { "trait_type": "Color", "value": "Blue" },
        { "trait_type": "Level", "value": 5 }
    ],
    "properties": {
        "files": [
            { "uri": "https://arweave.net/image.png", "type": "image/png" }
        ],
        "category": "image"
    }
}
```

## Using Metaplex in Anchor Programs

### Dependencies
```toml
[dependencies]
anchor-lang = "0.31.1"
anchor-spl = { version = "0.31.1", features = ["metadata", "idl-build"] }
mpl-token-metadata = "5.0"  # Check latest version
```

### Creating Metadata
```rust
use anchor_spl::metadata::{
    create_metadata_accounts_v3,
    CreateMetadataAccountsV3,
    Metadata,
    mpl_token_metadata::types::DataV2,
};

#[derive(Accounts)]
pub struct CreateTokenMetadata<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,
    pub mint: Account<'info, Mint>,
    /// CHECK: Metadata account PDA
    #[account(
        mut,
        seeds = [
            b"metadata",
            metadata_program.key().as_ref(),
            mint.key().as_ref(),
        ],
        bump,
        seeds::program = metadata_program.key(),
    )]
    pub metadata_account: UncheckedAccount<'info>,
    pub mint_authority: Signer<'info>,
    pub system_program: Program<'info, System>,
    pub metadata_program: Program<'info, Metadata>,
    pub rent: Sysvar<'info, Rent>,
}

pub fn create_metadata(
    ctx: Context<CreateTokenMetadata>,
    name: String,
    symbol: String,
    uri: String,
) -> Result<()> {
    let data = DataV2 {
        name,
        symbol,
        uri,
        seller_fee_basis_points: 0,
        creators: None,
        collection: None,
        uses: None,
    };

    create_metadata_accounts_v3(
        CpiContext::new(
            ctx.accounts.metadata_program.to_account_info(),
            CreateMetadataAccountsV3 {
                metadata: ctx.accounts.metadata_account.to_account_info(),
                mint: ctx.accounts.mint.to_account_info(),
                mint_authority: ctx.accounts.mint_authority.to_account_info(),
                payer: ctx.accounts.payer.to_account_info(),
                update_authority: ctx.accounts.mint_authority.to_account_info(),
                system_program: ctx.accounts.system_program.to_account_info(),
                rent: ctx.accounts.rent.to_account_info(),
            },
        ),
        data,
        true,  // is_mutable
        true,  // update_authority_is_signer
        None,  // collection details
    )?;
    Ok(())
}
```

## TypeScript Metaplex Operations

### Using @metaplex-foundation/mpl-token-metadata (Umi)
```bash
yarn add @metaplex-foundation/mpl-token-metadata @metaplex-foundation/umi @metaplex-foundation/umi-bundle-defaults
```

```typescript
import { createUmi } from "@metaplex-foundation/umi-bundle-defaults";
import {
    createMetadataAccountV3,
    mplTokenMetadata,
} from "@metaplex-foundation/mpl-token-metadata";
import { publicKey } from "@metaplex-foundation/umi";

const umi = createUmi("https://api.devnet.solana.com")
    .use(mplTokenMetadata());

// Create metadata for existing mint
await createMetadataAccountV3(umi, {
    mint: publicKey(mintAddress),
    mintAuthority: umi.identity,
    payer: umi.payer,
    data: {
        name: "My Token",
        symbol: "MTK",
        uri: "https://arweave.net/metadata.json",
        sellerFeeBasisPoints: 0,
        creators: null,
        collection: null,
        uses: null,
    },
    isMutable: true,
    collectionDetails: null,
}).sendAndConfirm(umi);
```

### Creating NFTs with Metaplex
```typescript
import { createNft } from "@metaplex-foundation/mpl-token-metadata";
import { generateSigner, percentAmount } from "@metaplex-foundation/umi";

const mint = generateSigner(umi);

await createNft(umi, {
    mint,
    name: "My NFT",
    symbol: "MNFT",
    uri: "https://arweave.net/nft-metadata.json",
    sellerFeeBasisPoints: percentAmount(5), // 5% royalty
    creators: [
        { address: umi.identity.publicKey, verified: true, share: 100 },
    ],
}).sendAndConfirm(umi);
```

## Token-2022 Embedded Metadata (Alternative to Metaplex)

For simpler use cases, Token-2022 has built-in metadata without needing Metaplex:

```bash
# Create token with metadata extension
spl-token create-token --program-id TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb \
  --enable-metadata

# Set metadata
spl-token initialize-metadata <MINT> "Token Name" "SYM" "https://example.com/meta.json"

# Update metadata field
spl-token update-metadata <MINT> name "New Name"
spl-token update-metadata <MINT> uri "https://new-uri.com/meta.json"
```

### When to Use Token-2022 Metadata vs Metaplex
| | Token-2022 Metadata | Metaplex |
|---|---|---|
| Simplicity | ✅ Built-in, no extra program | ❌ Extra CPI, more complexity |
| Cost | ✅ Cheaper (no extra account) | ❌ Extra account rent |
| Ecosystem support | ⚠️ Growing | ✅ Widely supported |
| Rich features | ❌ Basic fields only | ✅ Collections, royalties, uses |
| NFT standard | ❌ Not standard | ✅ The NFT standard |
| Fungible tokens | ✅ Good enough | ✅ Full support |
