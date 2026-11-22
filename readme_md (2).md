# ⚡ SolVault — Servidor de Almacenamiento Rápido Descentralizado en Solana

Proyecto Final para el **Solana Engineering Track: Advanced**. **SolVault** es un protocolo on-chain construido con **Anchor** que conecta nodos de almacenamiento rápido off-chain (NVMe / Edge Cache / S3) con certificación criptográfica y micropagos nativos en **Solana**.

## 📌 Descripción General

El sistema permite a proveedores de infraestructura registrar sus nodos de almacenamiento rápido en Solana y establecer una tarifa por Megabyte (`price_per_mb_lamports`). Los usuarios suben sus archivos al servidor rápido y registran de forma inmutable en la blockchain su **hash SHA-256**, tamaño y sello de tiempo, ejecutando el pago automático mediante una **Cross-Program Invocation (CPI)** al System Program.

### Flujo de Trabajo (Off-Chain + On-Chain)

1. **Registro del Servidor:** El proveedor inicializa su cuenta PDA `StorageNode` con su `endpoint_url` y tarifa.
2. **Carga Rápida y Hashing:** El cliente calcula el hash `SHA-256` del archivo y lo envía al servidor rápido.
3. **Certificación y Pago On-Chain:** El usuario firma la instrucción `store_file`, transfiriendo los lamports correspondientes al proveedor y creando la PDA `FileRecord`.
4. **Auditoría e Integridad:** Cualquier cliente puede descargar el archivo del nodo rápido y verificar su autenticidad comparando su `SHA-256` con el almacenado en Solana.
5. **Gestión y Recuperación de Renta:** El propietario puede actualizar versiones del archivo (`update_file_record`) o eliminar el registro (`delete_file_record`) recuperando el depósito de renta de la cuenta.

## 🏗️ Arquitectura y Modelo de Cuentas (PDAs)

El programa utiliza **Program Derived Addresses (PDAs)** deterministas para garantizar aislamiento de estado y seguridad en los permisos:

| Cuenta (PDA) | Seeds (Semillas) | Campos Principales | Propósito |
| :--- | :--- | :--- | :--- |
| **`StorageNode`** | `["node", provider.key()]` | `provider`, `endpoint_url`, `price_per_mb_lamports`, `total_files`, `is_online`, `bump` | Almacena la configuración, URL del servidor rápido y tarifa del proveedor. |
| **`FileRecord`** | `["file", node.key(), owner.key(), file_id]` | `node`, `owner`, `file_id`, `content_hash` (`[u8; 32]`), `size_mb`, `uploaded_at`, `is_pinned`, `bump` | Certifica la propiedad, integridad (`SHA-256`) y metadatos del archivo. |

## 🛠️ Instrucciones del Programa (CRUD + CPI)

* **`register_node(endpoint_url, price_per_mb_lamports)`**: Inicializa un servidor de almacenamiento rápido en la red.
* **`store_file(file_id, content_hash, size_mb)`**: Calcula el costo total con aritmética segura (`checked_mul`), ejecuta la transferencia nativa de SOL al proveedor vía CPI y registra los metadatos del archivo.
* **`update_file_record(new_content_hash, is_pinned)`**: Permite al dueño legítimo actualizar el hash criptográfico (nueva versión) o cambiar el estado de anclaje (*pinning*).
* **`delete_file_record()`**: Cierra la cuenta `FileRecord` usando la restricción `close = owner`, devolviendo los lamports de renta a la wallet del usuario y decrementando el contador del nodo.

## 📂 Estructura del Repositorio

```text
sol-vault/
├── programs/
│   └── sol-vault/
│       └── src/
│           └── lib.rs          # Contrato inteligente en Rust (Anchor)
├── tests/
│   └── sol-vault.ts            # Suite de pruebas unitarias y de seguridad
├── Anchor.toml                 # Configuración de clusters y Program ID
└── README.md                   # Documentación técnica y evidencias
```

## 💻 Código Fuente Completo

### 1. Contrato Inteligente (`programs/sol-vault/src/lib.rs`)

```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("DGKNgbjk9ki9Fjkdr8oeB3kBAF7YSyrUNTVmqfHoQQWV");

#[program]
pub mod sol_vault {
    use super::*;

    /// 1. Registra un servidor de almacenamiento rápido y su tarifa por MB
    pub fn register_node(
        ctx: Context<RegisterNode>,
        endpoint_url: String,
        price_per_mb_lamports: u64,
    ) -> Result<()> {
        require!(endpoint_url.len() <= 64, VaultError::UrlTooLong);
        require!(price_per_mb_lamports > 0, VaultError::InvalidPrice);

        let node = &mut ctx.accounts.storage_node;
        node.provider = ctx.accounts.provider.key();
        node.endpoint_url = endpoint_url;
        node.price_per_mb_lamports = price_per_mb_lamports;
        node.total_files = 0;
        node.is_online = true;
        node.bump = ctx.bumps.storage_node;

        Ok(())
    }

    /// 2. Registra un archivo rápido on-chain y paga al proveedor mediante CPI
    pub fn store_file(
        ctx: Context<StoreFile>,
        file_id: String,
        content_hash: [u8; 32],
        size_mb: u32,
    ) -> Result<()> {
        require!(file_id.len() > 0 && file_id.len() <= 32, VaultError::InvalidFileId);
        require!(size_mb > 0, VaultError::InvalidFileSize);

        let node = &mut ctx.accounts.storage_node;
        require!(node.is_online, VaultError::NodeOffline);

        let total_cost = node
            .price_per_mb_lamports
            .checked_mul(size_mb as u64)
            .ok_or(VaultError::MathOverflow)?;

        let cpi_ctx = CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            Transfer {
                from: ctx.accounts.owner.to_account_info(),
                to: ctx.accounts.provider.to_account_info(),
            },
        );
        transfer(cpi_ctx, total_cost)?;

        let record = &mut ctx.accounts.file_record;
        let clock = Clock::get()?;

        record.node = node.key();
        record.owner = ctx.accounts.owner.key();
        record.file_id = file_id;
        record.content_hash = content_hash;
        record.size_mb = size_mb;
        record.uploaded_at = clock.unix_timestamp;
        record.is_pinned = true;
        record.bump = ctx.bumps.file_record;

        node.total_files = node.total_files.saturating_add(1);

        Ok(())
    }

    /// 3. Actualiza el hash (nueva versión) o estado de anclaje (pin) del archivo
    pub fn update_file_record(
        ctx: Context<UpdateFileRecord>,
        new_content_hash: Option<[u8; 32]>,
        is_pinned: Option<bool>,
    ) -> Result<()> {
        let record = &mut ctx.accounts.file_record;
        let clock = Clock::get()?;

        if let Some(hash) = new_content_hash {
            record.content_hash = hash;
            record.uploaded_at = clock.unix_timestamp;
        }
        if let Some(pinned) = is_pinned {
            record.is_pinned = pinned;
        }

        Ok(())
    }

    /// 4. Elimina el registro del archivo y devuelve la renta (lamports) al dueño
    pub fn delete_file_record(ctx: Context<DeleteFileRecord>) -> Result<()> {
        let node = &mut ctx.accounts.storage_node;
        node.total_files = node.total_files.saturating_sub(1);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct RegisterNode<'info> {
    #[account(
        init,
        payer = provider,
        space = 8 + StorageNode::INIT_SPACE,
        seeds = [b"node", provider.key().as_ref()],
        bump
    )]
    pub storage_node: Account<'info, StorageNode>,
    #[account(mut)]
    pub provider: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
#[instruction(file_id: String)]
pub struct StoreFile<'info> {
    #[account(
        mut,
        seeds = [b"node", storage_node.provider.as_ref()],
        bump = storage_node.bump,
        has_one = provider
    )]
    pub storage_node: Account<'info, StorageNode>,
    #[account(
        init,
        payer = owner,
        space = 8 + FileRecord::INIT_SPACE,
        seeds = [
            b"file",
            storage_node.key().as_ref(),
            owner.key().as_ref(),
            file_id.as_bytes()
        ],
        bump
    )]
    pub file_record: Account<'info, FileRecord>,
    /// CHECK: Validado mediante constraint has_one = provider en storage_node
    #[account(mut)]
    pub provider: UncheckedAccount<'info>,
    #[account(mut)]
    pub owner: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateFileRecord<'info> {
    #[account(
        mut,
        seeds = [
            b"file",
            file_record.node.as_ref(),
            owner.key().as_ref(),
            file_record.file_id.as_bytes()
        ],
        bump = file_record.bump,
        has_one = owner
    )]
    pub file_record: Account<'info, FileRecord>,
    pub owner: Signer<'info>,
}

#[derive(Accounts)]
pub struct DeleteFileRecord<'info> {
    #[account(
        mut,
        seeds = [b"node", storage_node.provider.as_ref()],
        bump = storage_node.bump
    )]
    pub storage_node: Account<'info, StorageNode>,
    #[account(
        mut,
        close = owner,
        seeds = [
            b"file",
            storage_node.key().as_ref(),
            owner.key().as_ref(),
            file_record.file_id.as_bytes()
        ],
        bump = file_record.bump,
        has_one = owner
    )]
    pub file_record: Account<'info, FileRecord>,
    #[account(mut)]
    pub owner: Signer<'info>,
}

#[account]
#[derive(InitSpace)]
pub struct StorageNode {
    pub provider: Pubkey,
    #[max_len(64)]
    pub endpoint_url: String,
    pub price_per_mb_lamports: u64,
    pub total_files: u32,
    pub is_online: bool,
    pub bump: u8,
}

#[account]
#[derive(InitSpace)]
pub struct FileRecord {
    pub node: Pubkey,
    pub owner: Pubkey,
    #[max_len(32)]
    pub file_id: String,
    pub content_hash: [u8; 32],
    pub size_mb: u32,
    pub uploaded_at: i64,
    pub is_pinned: bool,
    pub bump: u8,
}

#[error_code]
pub enum VaultError {
    #[msg("La URL del servidor no puede exceder los 64 caracteres.")]
    UrlTooLong,
    #[msg("El precio por MB debe ser mayor a 0 lamports.")]
    InvalidPrice,
    #[msg("El identificador del archivo debe tener entre 1 y 32 caracteres.")]
    InvalidFileId,
    #[msg("El tamaño del archivo debe ser al menos 1 MB.")]
    InvalidFileSize,
    #[msg("El servidor de almacenamiento se encuentra fuera de línea.")]
    NodeOffline,
    #[msg("Error de desbordamiento aritmético al calcular el costo.")]
    MathOverflow,
}
```

### 2. Pruebas Automatizadas (`tests/sol-vault.ts`)

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { SolVault } from "../target/types/sol_vault";
import { assert } from "chai";
import * as crypto from "crypto";

describe("sol-vault: servidor de almacenamiento rápido", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.SolVault as Program<SolVault>;
  const serverProvider = provider.wallet;
  const user = anchor.web3.Keypair.generate();
  const attacker = anchor.web3.Keypair.generate();

  const fileId = "backup-db-2026.tar.zst";

  const [nodePda] = anchor.web3.PublicKey.findProgramAddressSync(
    [Buffer.from("node"), serverProvider.publicKey.toBuffer()],
    program.programId
  );

  const [filePda] = anchor.web3.PublicKey.findProgramAddressSync(
    [
      Buffer.from("file"),
      nodePda.toBuffer(),
      user.publicKey.toBuffer(),
      Buffer.from(fileId),
    ],
    program.programId
  );

  before(async () => {
    for (const kp of [user, attacker]) {
      const sig = await provider.connection.requestAirdrop(
        kp.publicKey,
        2 * anchor.web3.LAMPORTS_PER_SOL
      );
      await provider.connection.confirmTransaction(sig);
    }
  });

  it("1. Registra el servidor de almacenamiento rápido", async () => {
    const pricePerMb = new anchor.BN(1_000_000);

    await program.methods
      .registerNode("https://fast-edge-01.solvault.io", pricePerMb)
      .accounts({
        storageNode: nodePda,
        provider: serverProvider.publicKey,
      })
      .rpc();

    const nodeAccount = await program.account.storageNode.fetch(nodePda);
    assert.equal(nodeAccount.endpointUrl, "https://fast-edge-01.solvault.io");
    assert.isTrue(nodeAccount.isOnline);
    assert.equal(nodeAccount.totalFiles, 0);
  });

  it("2. Almacena metadatos de archivo, verifica SHA-256 y paga al servidor", async () => {
    const fileContent = Buffer.from("Contenido binario ultra rápido en caché");
    const sha256Hash = Array.from(
      crypto.createHash("sha256").update(fileContent).digest()
    );
    const sizeMb = 50;

    const initialProviderBalance = await provider.connection.getBalance(
      serverProvider.publicKey
    );

    await program.methods
      .storeFile(fileId, sha256Hash, sizeMb)
      .accounts({
        storageNode: nodePda,
        fileRecord: filePda,
        provider: serverProvider.publicKey,
        owner: user.publicKey,
      })
      .signers([user])
      .rpc();

    const finalProviderBalance = await provider.connection.getBalance(
      serverProvider.publicKey
    );
    const fileAccount = await program.account.fileRecord.fetch(filePda);
    const nodeAccount = await program.account.storageNode.fetch(nodePda);

    assert.equal(
      finalProviderBalance - initialProviderBalance,
      50 * 1_000_000
    );
    assert.deepEqual(fileAccount.contentHash, sha256Hash);
    assert.equal(nodeAccount.totalFiles, 1);
  });

  it("3. Actualiza el hash del archivo (v2) por su dueño legítimo", async () => {
    const newHash = Array.from(
      crypto.createHash("sha256").update("Contenido actualizado v2").digest()
    );

    await program.methods
      .updateFileRecord(newHash, false)
      .accounts({
        fileRecord: filePda,
        owner: user.publicKey,
      })
      .signers([user])
      .rpc();

    const updatedRecord = await program.account.fileRecord.fetch(filePda);
    assert.deepEqual(updatedRecord.contentHash, newHash);
    assert.isFalse(updatedRecord.isPinned);
  });

  it("4. Rechaza modificación si firma un atacante (Prueba de Seguridad)", async () => {
    const fakeHash = new Array(32).fill(9);

    try {
      await program.methods
        .updateFileRecord(fakeHash, true)
        .accounts({
          fileRecord: filePda,
          owner: attacker.publicKey,
        })
        .signers([attacker])
        .rpc();
      assert.fail("La transacción debió fallar por semillas/has_one inválidas");
    } catch (err) {
      assert.include(err.message, "AnchorError");
    }
  });

  it("5. Elimina el archivo on-chain y recupera la renta", async () => {
    await program.methods
      .deleteFileRecord()
      .accounts({
        storageNode: nodePda,
        fileRecord: filePda,
        owner: user.publicKey,
      })
      .signers([user])
      .rpc();

    const closedAccount = await provider.connection.getAccountInfo(filePda);
    const nodeAccount = await program.account.storageNode.fetch(nodePda);

    assert.isNull(closedAccount);
    assert.equal(nodeAccount.totalFiles, 0);
  });
});
```

### 3. Configuración (`Anchor.toml`)

```toml
[toolchain]

[features]
seeds = false
skip-lint = false

[programs.devnet]
sol_vault = "DGKNgbjk9ki9Fjkdr8oeB3kBAF7YSyrUNTVmqfHoQQWV"

[programs.localnet]
sol_vault = "DGKNgbjk9ki9Fjkdr8oeB3kBAF7YSyrUNTVmqfHoQQWV"

[registry]
url = "https://api.apr.dev"

[provider]
cluster = "devnet"
wallet = "~/.config/solana/id.json"

[scripts]
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
```

## 🚀 Instalación y Despliegue en Devnet

1. **Clonar e instalar dependencias:**
   ```bash
   git clone https://github.com/TU_USUARIO/sol-vault.git
   cd sol-vault
   yarn install
   ```
2. **Compilar y probar:**
   ```bash
   anchor build
   anchor test
   ```
3. **Desplegar en Devnet:**
   ```bash
   solana config set --url devnet
   solana airdrop 2
   anchor deploy --provider.cluster devnet
   ```

## 📋 Evidencias de Despliegue (Solana Devnet)

* **Program ID (Devnet):** `DGKNgbjk9ki9Fjkdr8oeB3kBAF7YSyrUNTVmqfHoQQWV`
* **Programa en Solana Explorer:** [Ver contrato en Devnet](https://explorer.solana.com/address/DGKNgbjk9ki9Fjkdr8oeB3kBAF7YSyrUNTVmqfHoQQWV?cluster=devnet)
* **Transacción `register_node`:** [Ver en Solana Explorer](https://explorer.solana.com/tx/TU_HASH_AQUI?cluster=devnet)
* **Transacción `store_file`:** [Ver en Solana Explorer](https://explorer.solana.com/tx/TU_HASH_AQUI?cluster=devnet)
* **Transacción `update_file_record`:** [Ver en Solana Explorer](https://explorer.solana.com/tx/TU_HASH_AQUI?cluster=devnet)
* **Transacción `delete_file_record`:** [Ver en Solana Explorer](https://explorer.solana.com/tx/TU_HASH_AQUI?cluster=devnet)

## 📄 Licencia

Este proyecto está bajo la **Licencia MIT**.