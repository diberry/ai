# Infrastructure, CI/CD, Rust Idioms & Documentation

Rules DOC-1 through DOC-7, IaC-1 through IaC-7, AZD-1 through AZD-2, RS-1 through RS-7, CI-1 through CI-3, and the comprehensive pre-review checklist.

## DOC-1: Expected Output (CRITICAL)

README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

DO:
```markdown
## Expected Output

Run the sample:
```bash
cargo run
```

You should see output similar to:

```
Connected to Azure Blob Storage
Container 'samples' created
Uploaded blob 'sample.txt' (14 bytes)
Downloaded blob content: "Hello, Azure!"
```

> Note: Exact output may vary based on your Azure environment.
```

---

## DOC-2: Folder Path Links (CRITICAL)

All internal README links must match actual filesystem paths.

DO:
```markdown
- [`src/main.rs`](./src/main.rs) -- Main entry point
- [`infra/main.bicep`](./infra/main.bicep) -- Infrastructure template
```

---

## DOC-3: Troubleshooting Section (MEDIUM)

DO:
```markdown
## Troubleshooting

### Authentication Errors
If you see "Failed to acquire token":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments

### Build Errors
If `cargo build` fails:
1. Ensure Rust 1.75+ is installed: `rustup update stable`
2. Check required system dependencies (OpenSSL dev headers on Linux)

### Common Azure SDK Errors
- `azure_core::Error` with "Unauthorized": Run `az login` or check role assignments
- `azure_core::Error` with "NotFound": Verify resource exists and endpoint URL is correct
- `tiberius::Error` with connection refused: Check firewall rules for Azure SQL
```

---

## DOC-4: Prerequisites Section (HIGH)

DO:
```markdown
## Prerequisites
- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Rust**: Version 1.75 or later ([Install via rustup](https://rustup.rs/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional)

### Role Assignments
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
```

---

## DOC-5: Setup Instructions (MEDIUM)

Provide clear, tested setup instructions including Azure resource provisioning.

---

## DOC-6: Rust Version Strategy (LOW)

Document minimum Rust version in both README and Cargo.toml `rust-version` field.

---

## DOC-7: Placeholder Values (MEDIUM)

READMEs must provide clear instructions for replacing placeholder values, including where to find them in Azure Portal.

---

## IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)

Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

DO:
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
  name: 'storage-deployment'
  params: {
    name: storageAccountName
    location: location
  }
}
```

DON'T:
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.7.0' = {
  // Outdated version
}
```

---

## IaC-2: Bicep Parameter Validation (CRITICAL)

Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

DO:
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

DON'T:
```bicep
param aadAdminObjectId string  // No validation, accepts empty string
```

---

## IaC-3: API Versions (MEDIUM)

Use current API versions (2023+). Avoid versions older than 2 years.

DO:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
```

---

## IaC-4: RBAC Role Assignments (HIGH)

DO:
```bicep
resource storageBlobDataContributorRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      'ba92f5b4-2d11-453d-a403-e96b0029c9fe'  // Storage Blob Data Contributor
    )
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

Common role IDs:
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cosmos DB Account Reader Role: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

## IaC-5: Network Security (HIGH)

For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

DO (Quickstart):
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: openaiAccountName
  properties: {
    publicNetworkAccess: 'Enabled'  // OK for quickstart
    networkAcls: { defaultAction: 'Allow' }
  }
}
// NOTE: This quickstart uses public endpoints for simplicity.
// For production, use private endpoints and set defaultAction: 'Deny'.
```

---

## IaC-6: Output Values (MEDIUM)

Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

DO:
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_STORAGE_BLOB_ENDPOINT string = storageAccount.properties.primaryEndpoints.blob
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
```

---

## IaC-7: Resource Naming Conventions (HIGH)

Follow Cloud Adoption Framework (CAF) naming conventions.

DO:
```bicep
var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

---

## AZD-1: azure.yaml Structure (MEDIUM)

> FALSE POSITIVE PREVENTION: `services`, `hooks`, and `host` fields are OPTIONAL. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct. Check parent directories for monorepo layouts.

DO:
```yaml
name: azure-storage-blob-rust-sample
metadata:
  template: azure-storage-blob-rust-sample@0.0.1

services:
  app:
    project: ./
    language: rust
    host: containerapp
    docker:
      path: ./Dockerfile

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)
      rustc --version || (echo "Rust not installed. Visit https://rustup.rs/" && exit 1)
```

---

## AZD-2: Dockerfile for Rust (MEDIUM)

Use multi-stage Docker builds for Rust samples deployed via azd.

DO:
```dockerfile
# Build stage
FROM rust:1.75 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ ./src/
RUN cargo build --release

# Runtime stage
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/azure-sample /usr/local/bin/
ENTRYPOINT ["azure-sample"]
```

DON'T:
```dockerfile
# Don't ship debug builds
FROM rust:1.75
COPY . .
RUN cargo build
CMD ["./target/debug/azure-sample"]  # Debug build, large image
```

---

## RS-1: Ownership and Borrowing in Client APIs (HIGH)

Use `Arc<T>` for sharing credential across clients. Pass references where possible instead of cloning.

DO:
```rust
use std::sync::Arc;
use azure_identity::DefaultAzureCredential;

let credential = Arc::new(DefaultAzureCredential::new()?);
let blob_client = create_blob_client(credential.clone());
let kv_client = create_keyvault_client(credential.clone());

// Pass data by reference when possible
async fn process_items(items: &[Product]) -> Result<(), Box<dyn std::error::Error>> {
    for item in items {
        println!("Processing: {}", item.name);
    }
    Ok(())
}

// Use &str instead of String for function parameters
fn format_endpoint(account_name: &str) -> String {
    format!("https://{}.blob.core.windows.net", account_name)
}
```

DON'T:
```rust
// Don't clone expensive types unnecessarily
let credential = DefaultAzureCredential::new()?;
let cred_clone1 = credential.clone();  // Full clone (if not Arc-wrapped)

// Don't take ownership when a reference suffices
fn process_items(items: Vec<Product>) {  // Takes ownership, forces caller to clone
    for item in &items {
        println!("{}", item.name);
    }
}
```

---

## RS-2: Async Runtime (tokio) Configuration (HIGH)

DO:
```rust
// Simple sample -- use tokio::main macro
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = load_config()?;
    run_sample(&config).await?;
    Ok(())
}

// For resource-constrained environments:
#[tokio::main(flavor = "current_thread")]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    run_sample().await
}
```

```rust
// For concurrent operations
use tokio::try_join;

async fn run_parallel_uploads(
    client: &BlobServiceClient,
    files: &[(&str, &[u8])],
) -> Result<(), Box<dyn std::error::Error>> {
    let futures: Vec<_> = files
        .iter()
        .map(|(name, data)| upload_blob(client, name, data))
        .collect();

    futures::future::try_join_all(futures).await?;
    Ok(())
}

// For CPU-heavy work, use spawn_blocking
let result = tokio::task::spawn_blocking(move || {
    compute_hash(&large_data)
}).await??;
```

DON'T:
```rust
// Don't block the async runtime
#[tokio::main]
async fn main() {
    let data = std::fs::read_to_string("large_file.json").unwrap();  // Blocking IO in async
}

// Don't mix async runtimes (tokio + async-std = conflicts)
```

---

## RS-3: Error Propagation Patterns (MEDIUM)

DO:
```rust
use std::error::Error;

async fn connect_to_storage(
    account: &str,
) -> Result<BlobServiceClient, Box<dyn Error>> {
    let credential = DefaultAzureCredential::new()
        .map_err(|e| format!("Failed to create Azure credential: {e}"))?;

    let client = BlobServiceClient::new(
        &format!("https://{account}.blob.core.windows.net"),
        Arc::new(credential),
    );
    Ok(client)
}

// Use anyhow for convenience in samples (optional)
use anyhow::{Context, Result};

async fn run() -> Result<()> {
    let config = load_config()
        .context("Failed to load configuration -- check .env file")?;
    let credential = DefaultAzureCredential::new()
        .context("Failed to create Azure credential -- run 'az login'")?;
    Ok(())
}
```

---

## RS-4: Logging with tracing Crate (LOW)

DO:
```rust
use tracing::{info, warn, error};
use tracing_subscriber;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt()
        .with_env_filter("azure=info,sample=debug")
        .init();

    info!("Starting Azure Storage sample");
    Ok(())
}
```

---

## RS-5: Feature Flags for Optional Dependencies (MEDIUM)

DO:
```toml
[features]
default = ["storage"]
storage = ["azure_storage_blob"]
ai = ["async-openai"]
vector-search = ["ai"]

[dependencies]
azure_storage_blob = { version = "0.x", optional = true }
async-openai = { version = "0.25", optional = true }
```

```rust
#[cfg(feature = "ai")]
async fn generate_embedding(text: &str) -> Result<Vec<f32>, Box<dyn Error>> {
    todo!()
}

#[cfg(not(feature = "ai"))]
async fn generate_embedding(_text: &str) -> Result<Vec<f32>, Box<dyn Error>> {
    Err("AI feature not enabled. Build with: cargo build --features ai".into())
}
```

---

## RS-6: Send + Sync Bounds for Async Use (HIGH)

`Arc<DefaultAzureCredential>` is `Send + Sync` -- safe to share across tasks.

DO:
```rust
let credential = Arc::new(DefaultAzureCredential::new()?);

let cred = credential.clone();
let handle = tokio::spawn(async move {
    let token = cred.get_token(&["https://storage.azure.com/.default"]).await?;
    Ok::<_, Box<dyn std::error::Error + Send + Sync>>(token)
});
```

DON'T:
```rust
// Don't use Rc<T> (not Send) in async code shared across tasks
use std::rc::Rc;
let credential = Rc::new(DefaultAzureCredential::new()?);  // Rc is !Send
```

---

## RS-7: TokenCredential Trait Bounds (MEDIUM)

When writing generic functions that accept any Azure credential, use `TokenCredential` trait bounds.

DO:
```rust
use azure_core::auth::TokenCredential;
use std::sync::Arc;

async fn get_storage_token(
    credential: &dyn TokenCredential,
) -> Result<String, azure_core::Error> {
    let response = credential
        .get_token(&["https://storage.azure.com/.default"])
        .await?;
    Ok(response.token.clone())
}

async fn create_clients(
    credential: Arc<dyn TokenCredential>,
) -> Result<(), Box<dyn std::error::Error>> {
    let blob_client = BlobServiceClient::new(url, credential.clone(), None);
    let kv_client = SecretClient::new(kv_url, credential.clone(), None)?;
    Ok(())
}
```

---

## CI-1: Cargo Test and Clippy in CI (HIGH)

DO:
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo build --release
      - run: cargo test
      - run: cargo install cargo-audit && cargo audit
```

---

## CI-2: Test Patterns for Azure SDK Samples (MEDIUM)

DO:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_config_loading() {
        std::env::set_var("AZURE_STORAGE_ACCOUNT_NAME", "testaccount");
        std::env::set_var("AZURE_KEYVAULT_URL", "https://test.vault.azure.net/");

        let config = load_config().expect("Config should load with valid env vars");
        assert_eq!(config.storage_account_name, "testaccount");
    }

    #[test]
    fn test_endpoint_formatting() {
        let endpoint = format_endpoint("myaccount");
        assert_eq!(endpoint, "https://myaccount.blob.core.windows.net");
    }

    #[test]
    fn test_embedding_validation() {
        let valid = vec![0.0_f32; 1536];
        assert!(validate_embedding(&valid).is_ok());

        let invalid = vec![0.0_f32; 100];
        assert!(validate_embedding(&invalid).is_err());
    }
}
```

---

## CI-3: cargo-deny for License and Vulnerability Auditing (MEDIUM)

DO:
```yaml
- run: cargo install cargo-deny && cargo deny check
```

```toml
# deny.toml
[advisories]
vulnerability = "deny"
unmaintained = "warn"

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause", "ISC"]
```

---

## Pre-Review Checklist (Comprehensive)

### Project Setup
- [ ] Rust edition 2021 in Cargo.toml
- [ ] `rust-version` field set
- [ ] Cargo.toml has `authors`, `license`, `description`, `publish = false`
- [ ] Every dependency used in code (no phantom deps)
- [ ] Official `azure_*` crates used
- [ ] Environment variables validated with descriptive errors
- [ ] `cargo clippy` passes, Cargo.lock committed, `cargo audit` passes

### Security & Hygiene
- [ ] `.gitignore` protects `.env`, `target/`, `.azure/`
- [ ] `.env.sample` provided, no live credentials, LICENSE present

### Azure SDK Patterns
- [ ] `DefaultAzureCredential` used, credential in `Arc`, pagination handled

### Rust Idioms
- [ ] `Result<T, E>` and `?` operator, no `unwrap()` in main paths
- [ ] `Arc<T>` for sharing, references preferred, `tokio` configured correctly

### Documentation
- [ ] README output from real runs, prerequisites complete, troubleshooting included

### Infrastructure (if applicable)
- [ ] AVM versions current, Bicep params validated, RBAC assignments present

### CI/CD
- [ ] `cargo clippy`, `cargo test`, `cargo audit`, `cargo deny check` in CI
