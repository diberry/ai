---
name: "azure-sdk-rust-sample-review"
description: "Comprehensive review checklist for Azure SDK Rust code samples covering project setup, Azure SDK client patterns, authentication, data services, infrastructure, documentation, and sample hygiene. Note: Azure SDK for Rust is evolving — some crate APIs may change."
domain: "code-review"
confidence: "medium"
source: "earned -- adapted from TypeScript review skill patterns, generalized for Rust Azure SDK ecosystem (preview-aware)"
---

## Context

Use this skill when reviewing **Rust code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general Rust review—it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (`azure_*` crates from crates.io, client construction, pipeline options)
- **Authentication patterns** (`DefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Key Vault, AI services)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)
- **Rust idioms** (ownership, borrowing, error propagation, async runtimes, RAII cleanup)

> **⚠️ SDK Maturity Note:** The Azure SDK for Rust is still evolving. Many crates (e.g., `azure_storage_blob`, `azure_identity`, `azure_core`) are available on crates.io but some are in **preview** or **alpha**. Crate APIs may change between versions. This skill focuses on patterns that will remain stable (auth, error handling, async, project structure) and notes where crate APIs may change. Always check [crates.io](https://crates.io/search?q=azure_) and the [Azure SDK for Rust GitHub repo](https://github.com/Azure/azure-sdk-for-rust) for the latest status.

**Total rules: 70** (9 CRITICAL, 24 HIGH, 32 MEDIUM, 5 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **Cargo.toml**: Uses `azure_*` crates (not unofficial wrappers)
- [ ] **Authentication**: Uses `DefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `.env`, `target/`
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: `cargo audit` passes with no critical/high vulnerabilities
- [ ] **Rust Edition**: `edition = "2021"` in Cargo.toml
- [ ] **Error handling**: Uses `Result<T, E>` and `?` operator (no `unwrap()` in main paths)
- [ ] **Resource cleanup**: Clients properly dropped (RAII pattern, no leaks)
- [ ] **Lock file**: Cargo.lock committed (for binary crates)
- [ ] **Clippy**: `cargo clippy` passes with no warnings
- [ ] **Imports work**: `cargo check` succeeds
- [ ] **Build succeeds**: `cargo build` completes without errors
- [ ] **Sample runs**: `cargo run` executes without panics

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets**—Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication**—No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling**—Uses `unwrap()` or `expect()` in main code paths, no `Result` returns
4. **Broken imports**—Missing dependencies, incorrect crate names, `cargo check` fails
5. **Security vulnerabilities**—`cargo audit` shows critical or high CVEs
6. **Missing LICENSE**—No LICENSE file at ANY level of repo hierarchy (MIT required for Azure Samples org). ⚠️ Check repo root before flagging.
7. **.env file committed**—Live credentials in version control. ⚠️ Verify with `git ls-files .env`—a .env on disk but in .gitignore is NOT committed.
8. **Panics in sample code**—Uses `unwrap()`, `expect()`, or `panic!()` in non-test code paths without justification

---

## 1. Project Setup & Configuration

**What this section covers:** Cargo project structure, Rust edition, dependency management, environment variables, and tooling configuration. These foundational patterns ensure samples compile correctly and run reliably across environments.

### PS-1: Rust Edition (HIGH)
**Pattern:** Use Rust 2021 edition (latest stable). Set explicitly in Cargo.toml. Note: Rust 2024 edition is emerging — see notes below.

✅ **DO:**
```toml
# Cargo.toml
[package]
name = "azure-storage-blob-quickstart"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"  # Minimum supported Rust version
```

❌ **DON'T:**
```toml
[package]
name = "azure-storage-blob-quickstart"
version = "0.1.0"
edition = "2018"  # ❌ Outdated edition
# ❌ Missing rust-version
```

**Why:** Rust 2021 edition includes important language improvements (disjoint capture in closures, new prelude items). `rust-version` documents the minimum toolchain required.

> **⚠️ 2024 Edition Note:** Rust 2024 edition changes some safety rules — notably `std::env::set_var` and `std::env::remove_var` are `unsafe` in 2024. If targeting 2024 edition, wrap env var mutations in `unsafe {}` blocks. Most samples should use 2021 for now, with 2024 readiness notes where relevant.

---

### PS-2: Cargo.toml Metadata (MEDIUM)
**Pattern:** All sample packages must include complete metadata for discoverability and maintenance.

✅ **DO:**
```toml
[package]
name = "azure-storage-blob-quickstart"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"
description = "Upload and download blobs using Azure Blob Storage SDK for Rust"
authors = ["Microsoft Corporation"]
license = "MIT"
repository = "https://github.com/Azure-Samples/azure-storage-blob-samples"
publish = false  # Samples should not be published to crates.io
```

❌ **DON'T:**
```toml
[package]
name = "my-sample"
version = "0.1.0"
edition = "2021"
# ❌ Missing authors, license, repository, description, publish = false
```

**Why:** `publish = false` prevents accidental publication to crates.io. Metadata helps with discoverability.

---

### PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency in `[dependencies]` must be used in code. No phantom dependencies. Use official `azure_*` crates.

✅ **DO:**
```toml
[dependencies]
azure_identity = "0.x"  # ✅ Used for DefaultAzureCredential
azure_storage_blob = "0.x"  # ✅ Used in src/main.rs
azure_core = "0.x"  # ✅ Used for error types, pipeline
tokio = { version = "1", features = ["full"] }  # ✅ Async runtime
serde = { version = "1", features = ["derive"] }  # ✅ Used for deserialization
serde_json = "1"                  # ✅ Used for JSON parsing
dotenvy = "0.15"                  # ✅ Used for .env loading
```

❌ **DON'T:**
```toml
[dependencies]
azure_identity = "0.x"  # Check crates.io for latest
azure_storage_blob = "0.x"  # Check crates.io for latest
azure_cosmos = "0.x"  # ❌ Listed but never used
reqwest = "0.12"                  # ❌ Not imported anywhere
rand = "0.8"                      # ❌ Phantom dependency
```

**Why:** Phantom dependencies increase compile times, audit surface area, and confuse users reading the sample.

---

### PS-4: Azure SDK Crate Naming (HIGH)
**Pattern:** Use official Azure SDK crates with `azure_*` prefix from crates.io. Verify publisher is `azure-sdk` or Microsoft.

✅ **DO:**
```toml
# ✅ Official Azure SDK for Rust crates (verify names on crates.io)
[dependencies]
azure_identity = "0.x"                    # Check crates.io for latest version
azure_core = "0.x"                        # Check crates.io for latest version
azure_storage_blob = "0.x"               # Check crates.io for latest version
azure_security_keyvault_secrets = "0.x"  # Check crates.io for latest version
azure_cosmos = "0.x"  # azure_messaging_servicebus — check crates.io for availability and current name
# azure_messaging_event_hubs — check crates.io for availability and current name
```

```rust
// ✅ Official crate imports
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
use azure_core::Error as AzureError;
```

❌ **DON'T:**
```toml
[dependencies]
azur_storage = "0.1.0"           # ❌ Typosquatting
azure-storage-blob = "0.1.0"     # ❌ Wrong naming (uses hyphens)
unofficial_azure_sdk = "0.1.0"   # ❌ Not official
```

**Why:** Official Azure SDK crates use the `azure_*` prefix and are published by the Azure SDK team. Check crates.io for publisher verification.

> **⚠️ Preview Note:** Some Azure service crates may not yet exist in the Rust SDK. Check [crates.io](https://crates.io/search?q=azure_) for availability. If a crate doesn't exist, note it in the README and consider using the REST API via `azure_core` or `reqwest`.

---

### PS-5: Configuration (MEDIUM)
**Pattern:** Use `dotenvy` crate for `.env` file loading. Validate all required environment variables with descriptive errors.

✅ **DO:**
```rust
use std::env;

#[derive(Debug)]
struct Config {
    storage_account_name: String,
    keyvault_url: String,
}

fn load_config() -> Result<Config, Box<dyn std::error::Error>> {
    // Load .env file if present (ignore errors if not found)
    let _ = dotenvy::dotenv();

    let required_vars = [
        "AZURE_STORAGE_ACCOUNT_NAME",
        "AZURE_KEYVAULT_URL",
    ];

    let missing: Vec<&str> = required_vars
        .iter()
        .filter(|v| env::var(v).is_err())
        .copied()
        .collect();

    if !missing.is_empty() {
        return Err(format!(
            "Missing required environment variables: {}\n\
             Create a .env file with these values or set them in your environment.\n\
             See .env.sample for required variables.",
            missing.join(", ")
        ).into());
    }

    Ok(Config {
        storage_account_name: env::var("AZURE_STORAGE_ACCOUNT_NAME")?,
        keyvault_url: env::var("AZURE_KEYVAULT_URL")?,
    })
}
```

❌ **DON'T:**
```rust
// ❌ Don't silently fall back to defaults
let account = env::var("AZURE_STORAGE_ACCOUNT_NAME")
    .unwrap_or_else(|_| "devstoreaccount1".to_string());

// ❌ Don't panic on missing env vars
let account = env::var("AZURE_STORAGE_ACCOUNT_NAME").unwrap();
```

---

### PS-6: Clippy Lints (MEDIUM)
**Pattern:** Ensure `cargo clippy` passes with no warnings. Configure recommended lints for samples.

✅ **DO:**
```rust
// src/main.rs — top-level lint configuration
#![deny(clippy::unwrap_used)]
#![warn(clippy::pedantic)]
#![allow(clippy::module_name_repetitions)]
```

```bash
# Run clippy before submitting
cargo clippy -- -D warnings
```

❌ **DON'T:**
```rust
// ❌ Don't suppress all clippy warnings
#![allow(clippy::all)]

// ❌ Don't ignore clippy output
// cargo clippy
// warning: unused variable: `x`
// Submitting sample anyway...
```

**Why:** Clippy catches common mistakes and enforces idiomatic Rust. `deny(clippy::unwrap_used)` prevents accidental `unwrap()` in samples.

---

### PS-7: rustfmt Configuration (LOW)
**Pattern:** Include `rustfmt.toml` for consistent formatting. Run `cargo fmt` before submission.

✅ **DO:**
```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_small_heuristics = "Max"
```

```bash
# Format before submitting
cargo fmt --check
```

---

### PS-8: Cargo.lock Committed (HIGH)
**Pattern:** Commit `Cargo.lock` for binary crates (samples). Do NOT commit for library crates.

✅ **DO:**
```gitignore
# .gitignore
target/
.env
.env.*
!.env.sample

# ✅ Cargo.lock is COMMITTED (not in .gitignore) for binary samples
```

❌ **DON'T:**
```gitignore
# ❌ Don't ignore Cargo.lock for binary crates
Cargo.lock
```

**Why:** `Cargo.lock` ensures reproducible builds for binary crates. Users get the exact versions tested by the sample author.

---

### PS-9: CVE Scanning (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. `cargo audit` must pass with no critical/high issues.

✅ **DO:**
```bash
# Install cargo-audit
cargo install cargo-audit

# Before submitting sample
cargo audit

# Check results — must have no critical/high vulnerabilities
cargo audit --deny warnings
```

❌ **DON'T:**
```bash
# ❌ Don't ignore audit warnings
cargo audit
# found 3 vulnerabilities
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Crate Legitimacy (MEDIUM)
**Pattern:** Verify Azure SDK crates are from the official `azure-sdk` publisher on crates.io. Watch for typosquatting.

✅ **DO:**
```toml
[dependencies]
azure_identity = "0.x"  # ✅ Official — check crates.io/crates/azure_identity
azure_storage_blob = "0.x"  # ✅ Official
azure_core = "0.x"  # ✅ Official
```

❌ **DON'T:**
```toml
[dependencies]
azur_identity = "0.1.0"       # ❌ Typosquatting (azur, not azure)
azure-identity = "0.1.0"      # ❌ Wrong format (hyphens instead of underscores)
az_storage = "0.1.0"           # ❌ Not official package
```

**Check:** All Azure SDK crates should use `azure_*` naming and be published by the Azure SDK team. Verify on [crates.io](https://crates.io).

---

### PS-11: Feature Flag Management (MEDIUM)
**Pattern:** Use Cargo feature flags to manage optional dependencies. Document required features in README.

✅ **DO:**
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
azure_identity = { version = "0.x", features = ["enable_reqwest"] }  # Check crates.io for latest; feature name may change
azure_storage_blob = { version = "0.x", default-features = false, features = ["enable_reqwest"] }  # Check crates.io for latest
serde = { version = "1", features = ["derive"] }
```

```markdown
<!-- README.md -->
## Dependencies

This sample uses the `reqwest` HTTP backend for Azure SDK crates.
Check the crate documentation for current feature flag names — they may evolve.
```

❌ **DON'T:**
```toml
[dependencies]
# ❌ Enabling all features when only a subset is needed
tokio = { version = "1", features = ["full"] }  # OK for samples
azure_identity = { version = "0.x", features = ["enable_reqwest", "enable_reqwest_rustls"] }
# ❌ Don't enable conflicting TLS backends
```

**Why:** Feature flags control compile-time dependencies. Conflicting features (e.g., two TLS backends) cause build errors.

---

## 2. Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK crates.

### AZ-1: Client Construction with DefaultAzureCredential (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with credential-first pattern. Share credential via `Arc`.

✅ **DO:**
```rust
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ✅ Create credential once, wrap in Arc for sharing
    let credential = Arc::new(DefaultAzureCredential::new()?);

    // ✅ Storage Blob — pass None for default ClientOptions
    let blob_service_client = BlobServiceClient::new(
        &format!("https://{}.blob.core.windows.net", account_name),
        credential.clone(),
        None,  // Use default ClientOptions
    );

    // ✅ Reuse same credential for another client
    let keyvault_client = azure_security_keyvault_secrets::SecretClient::new(
        &config.keyvault_url,
        credential.clone(),
        None,  // Use default ClientOptions
    )?;

    Ok(())
}
```

❌ **DON'T:**
```rust
// ❌ Don't use connection strings in samples (prefer AAD auth)
let blob_service_client = BlobServiceClient::from_connection_string(&connection_string)?;

// ❌ Don't recreate credential for each client
let client1 = BlobServiceClient::new(url, Arc::new(DefaultAzureCredential::new()?));
let client2 = SecretClient::new(url, Arc::new(DefaultAzureCredential::new()?));
// ❌ Two separate credential instances — wasteful
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code, etc.) and in cloud (managed identity). Wrapping in `Arc` allows zero-cost sharing across clients without cloning the credential itself.

---

### AZ-2: Client Options (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and transport options for production-ready samples. The Rust SDK uses struct initialization, not builder pattern.

✅ **DO:**
```rust
use azure_core::ClientOptions;
use std::time::Duration;

// ✅ Struct initialization with defaults (Rust SDK pattern)
let options = ClientOptions {
    // Configure retry, transport, etc. as needed
    ..Default::default()
};

// ✅ Pass options to client constructor
let blob_service_client = BlobServiceClient::new(
    &format!("https://{}.blob.core.windows.net", account_name),
    credential.clone(),
    Some(options),
);
```

❌ **DON'T:**
```rust
// ❌ Don't omit client options for samples that do meaningful work
let client = BlobServiceClient::new(url, credential.clone(), None);
// No retry policy, no timeout configuration
```

> **⚠️ Preview Note:** Client options API may vary across Azure SDK Rust crate versions. Check the specific crate's documentation for the current `ClientOptions` struct fields and initialization patterns.

---

### AZ-3: Managed Identity (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```rust
use azure_identity::{DefaultAzureCredential, ManagedIdentityCredential};
use std::sync::Arc;

// ✅ For samples: DefaultAzureCredential (works locally + cloud)
let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
let credential = Arc::new(ManagedIdentityCredential::default());

// ✅ User-assigned (when multiple identities needed)
// Check azure_identity docs for current constructor — API may vary
let client_id = std::env::var("AZURE_CLIENT_ID")?;
let credential = Arc::new(
    ManagedIdentityCredential::new(Some(&client_id))?
);

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
// > automatically use the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App has a managed identity assigned with
// > appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode service principal credentials in samples
let credential = ClientSecretCredential::new(tenant_id, client_id, client_secret);
```

---

### AZ-4: Token Management—TokenCredential Trait (CRITICAL)
**Pattern:** For services without official Rust SDK crate, get tokens via `TokenCredential::get_token()`. Tokens expire after ~1 hour—implement refresh logic for long-running samples.

✅ **DO:**
```rust
use azure_core::auth::TokenCredential;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;
use std::time::{Duration, Instant};

let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Get token with scope
let token_response = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;

let token = &token_response.token;

// ✅ Implement token refresh for long-running operations
async fn get_fresh_token(
    credential: &dyn TokenCredential,
    scope: &str,
) -> Result<String, azure_core::Error> {
    let response = credential.get_token(&[scope]).await?;
    Ok(response.token.clone())
}

// ✅ Refresh before long operations
let token = get_fresh_token(
    credential.as_ref(),
    "https://database.windows.net/.default"
).await?;
```

❌ **DON'T:**
```rust
// ❌ CRITICAL: Don't acquire token once and use for hours
let token = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

**Why:** Azure tokens expire after approximately 1 hour. Samples processing large datasets or running long operations MUST refresh tokens before expiration.

---

### AZ-5: Credential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Document the credential chain in README.

✅ **DO:**
```rust
use azure_identity::DefaultAzureCredential;

// ✅ Default credential chain (tries multiple auth methods)
let credential = DefaultAzureCredential::new()?;

// Document in README:
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries
// > multiple credential sources in order. The exact chain order in the Rust SDK
// > may differ from other language SDKs — check the current `azure_identity` docs
// > for the authoritative order. Typical sources include:
// > - Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > - Managed identity (App Service, Functions, Container Apps)
// > - Azure CLI (`az login`)
```

---

### AZ-6: Resource Cleanup—Drop Trait, RAII (MEDIUM)
**Pattern:** Rust's RAII (Resource Acquisition Is Initialization) handles cleanup via the `Drop` trait. Ensure clients go out of scope properly. For explicit cleanup, use scoped blocks or `drop()`.

✅ **DO:**
```rust
use azure_storage_blob::prelude::*;

async fn upload_blob(
    client: &BlobServiceClient,
    container: &str,
    blob: &str,
    data: &[u8],
) -> Result<(), Box<dyn std::error::Error>> {
    let container_client = client.container_client(container);
    let blob_client = container_client.blob_client(blob);

    // ✅ RAII: blob_client dropped at end of function
    blob_client.put_block_blob(data).await?;

    Ok(())
}

// ✅ Explicit scope for cleanup
async fn process_messages() -> Result<(), Box<dyn std::error::Error>> {
    {
        let client = create_service_bus_client()?;
        // ... process messages ...
        // ✅ client dropped here, connection closed
    }
    // Connection is guaranteed closed at this point
    println!("All resources cleaned up");
    Ok(())
}
```

❌ **DON'T:**
```rust
// ❌ Don't leak resources by storing in a global/static without cleanup
static mut CLIENT: Option<BlobServiceClient> = None;

// ❌ Don't use std::mem::forget to skip Drop
let client = create_client()?;
std::mem::forget(client);  // ❌ Resource leak
```

**Why:** Rust's ownership system and `Drop` trait provide deterministic cleanup without a garbage collector. Samples should demonstrate proper resource lifecycle.

---

### AZ-7: Pagination with Pageable/Stream (HIGH)
**Pattern:** Use async streams (`Pageable` / `Stream`) for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```rust
use azure_storage_blob::prelude::*;
use futures::StreamExt;

// ✅ Iterate all pages of blobs
let container_client = blob_service_client.container_client("mycontainer");
let mut blob_stream = container_client.list_blobs().into_stream();

while let Some(page) = blob_stream.next().await {
    let page = page?;
    for blob in page.blobs.blobs() {
        println!("Blob: {}", blob.name);
    }
}

// ✅ Collect all items (use with caution for large datasets)
let all_blobs: Vec<_> = container_client
    .list_blobs()
    .into_stream()
    .try_collect()
    .await?;
```

❌ **DON'T:**
```rust
// ❌ Only gets first page
let response = container_client.list_blobs().into_future().await?;
for blob in response.blobs.blobs() {
    println!("Blob: {}", blob.name);
}
// ❌ Remaining pages silently lost
```

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns for Azure OpenAI and other cognitive services in Rust. Note that not all Azure AI services have official Rust SDK crates yet.

### AI-1: Azure OpenAI Patterns (HIGH)
**Pattern:** Use the `azure_openai` crate if available, or the `async-openai` crate with Azure configuration. Always use AAD authentication where possible.

✅ **DO:**
```rust
// ✅ Using async-openai with Azure configuration (community crate)
use async_openai::{
    config::AzureConfig,
    Client,
    types::CreateChatCompletionRequestArgs,
};

let config = AzureConfig::new()
    .with_api_base(&format!(
        "https://{}.openai.azure.com",
        env::var("AZURE_OPENAI_RESOURCE_NAME")?
    ))
    .with_api_version("2024-10-21")
    .with_deployment_id("gpt-4o");

let client = Client::with_config(config);

// ✅ Chat completion
let request = CreateChatCompletionRequestArgs::default()
    .model("gpt-4o")
    .messages(vec![
        ChatCompletionRequestMessageArgs::default()
            .role(Role::User)
            .content("Hello!")
            .build()?
    ])
    .build()?;

let response = client.chat().create(request).await?;
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode API keys
let config = AzureConfig::new()
    .with_api_key("sk-abc123...");  // ❌ Use environment variables or AAD

// ❌ Don't omit API version
let config = AzureConfig::new()
    .with_api_base(endpoint);
    // ❌ Missing api_version — may use incompatible default
```

> **⚠️ Preview Note:** The official `azure_openai` Rust crate may not be available yet. The `async-openai` crate provides Azure support but is community-maintained. Check crates.io for the latest Azure OpenAI Rust options.

---

### AI-2: Embedding Dimension Validation (MEDIUM)
**Pattern:** Always validate that embedding vector dimensions match the target index/column configuration before insertion. Dimension mismatches cause silent data corruption or runtime errors.

✅ **DO:**
```rust
const EMBEDDING_MODEL: &str = "text-embedding-3-small";
const VECTOR_DIMENSION: usize = 1536;

/// Validate embedding dimensions against index configuration.
fn validate_embedding(
    embedding: &[f32],
    expected_dim: usize,
    model_name: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    if embedding.len() != expected_dim {
        return Err(format!(
            "Vector dimension mismatch: model '{}' produced {} dimensions, \
             but index expects {}. Update model or index configuration.",
            model_name, embedding.len(), expected_dim
        ).into());
    }
    Ok(())
}

// ✅ Validate before every insertion
let embedding = get_embedding(&text, EMBEDDING_MODEL).await?;
validate_embedding(&embedding, VECTOR_DIMENSION, EMBEDDING_MODEL)?;
insert_embedding(&embedding).await?;
```

❌ **DON'T:**
```rust
// ❌ Don't assume dimension without validation
let embedding = get_embedding(&text).await?;
insert_embedding(&embedding).await?;  // May fail silently if dimension wrong
```

---

### AI-3: Document Intelligence Patterns (MEDIUM)
**Pattern:** Use Azure Document Intelligence (Form Recognizer) for document analysis. No official Rust crate exists yet — use REST API via `reqwest` with AAD tokens from `azure_identity`.

✅ **DO:**
```rust
// ✅ REST-based Document Intelligence using reqwest + azure_identity
use azure_identity::DefaultAzureCredential;
use azure_core::auth::TokenCredential;
use reqwest::Client;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);
let token = credential
    .get_token(&["https://cognitiveservices.azure.com/.default"])
    .await?;

let endpoint = env::var("AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT")?;
let http_client = Client::new();

// ✅ Analyze document with prebuilt layout model
let response = http_client
    .post(format!(
        "{}/documentintelligence/documentModels/prebuilt-layout:analyze?api-version=2024-11-30",
        endpoint
    ))
    .bearer_auth(&token.token)
    .header("Content-Type", "application/pdf")
    .body(pdf_bytes)
    .send()
    .await?;

// ✅ Poll for result using Operation-Location header
let operation_url = response
    .headers()
    .get("Operation-Location")
    .ok_or("Missing Operation-Location header")?
    .to_str()?;
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode API keys in REST calls
let response = http_client
    .post(&url)
    .header("Ocp-Apim-Subscription-Key", "abc123...")  // ❌ Use AAD tokens
    .send()
    .await?;
```

> **⚠️ SDK Maturity Note:** No official `azure_ai_document_intelligence` Rust crate exists yet. Use the REST API with `reqwest` and AAD authentication via `azure_identity`. Check [crates.io](https://crates.io/search?q=azure_ai) for updates.

---

### AI-4: API Version Documentation (MEDIUM)
**Pattern:** When using Azure AI services via REST API (no official SDK crate), always document the API version used and note where to find the latest version.

✅ **DO:**
```rust
// ✅ Document API version in code and README
const API_VERSION: &str = "2024-10-21";

let url = format!(
    "{}/openai/deployments/{}/chat/completions?api-version={}",
    endpoint, deployment, API_VERSION
);

// In README:
// > This sample uses Azure OpenAI API version 2024-10-21.
// > Check https://learn.microsoft.com/azure/ai-services/openai/reference
// > for the latest stable API version.
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode API version without documentation
let url = format!("{}/openai/deployments/{}/chat/completions?api-version=2023-05-15", endpoint, deployment);
// ❌ No note about which version, why, or where to find updates
```

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization for Rust Azure SDK crates.

### DB-1: Cosmos DB Patterns (HIGH)
**Pattern:** Use `azure_cosmos` crate with AAD credentials. Handle partitioned containers properly.

✅ **DO:**
```rust
use azure_cosmos::prelude::*;
use azure_identity::DefaultAzureCredential;
use serde::{Deserialize, Serialize};
use std::sync::Arc;

#[derive(Debug, Serialize, Deserialize)]
struct Product {
    id: String,
    category: String,  // Partition key
    name: String,
    price: f64,
}

let credential = Arc::new(DefaultAzureCredential::new()?);

let client = CosmosClient::new(
    &config.cosmos_endpoint,
    credential.clone(),
    None,  // Use default ClientOptions
);

let database = client.database_client("mydb");
let container = database.container_client("products");

// ✅ Query with partition key
let query = "SELECT * FROM c WHERE c.category = @category";
let mut results = container
    .query_documents::<Product>(query)
    .partition_key(&PartitionKey::from("electronics"))
    .into_stream();

while let Some(page) = results.next().await {
    let page = page?;
    for doc in page.documents() {
        println!("Product: {} — ${:.2}", doc.name, doc.price);
    }
}

// ✅ Point read (most efficient)
let response = container
    .document_client("item-id", &PartitionKey::from("electronics"))?
    .get_document::<Product>()
    .await?;
```

❌ **DON'T:**
```rust
// ❌ Don't use primary key in samples
let client = CosmosClient::with_key(&endpoint, &primary_key);

// ❌ Don't omit partition key (cross-partition queries are expensive)
let results = container
    .query_documents::<Product>("SELECT * FROM c")
    .into_stream();
```

> **⚠️ Preview Note:** The `azure_cosmos` crate API may change. Check the latest version on crates.io.

---

### DB-2: Azure SQL with tiberius (HIGH)
**Pattern:** Use the `tiberius` crate for Azure SQL connections with AAD token authentication.

✅ **DO:**
```rust
use tiberius::{Client, Config, AuthMethod, ColumnData};
use tokio::net::TcpStream;
use tokio_util::compat::TokioAsyncWriteCompatExt;
use azure_identity::DefaultAzureCredential;
use azure_core::auth::TokenCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Get AAD token for Azure SQL
let token = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;

// ✅ Connect with AAD token
let mut config = Config::new();
config.host(&config_values.sql_server);
config.port(1433);
config.database(&config_values.sql_database);
config.authentication(AuthMethod::aad_token(token.token.clone()));
config.encryption(tiberius::EncryptionLevel::Required);
config.trust_cert();

let tcp = TcpStream::connect(config.get_addr()).await?;
tcp.set_nodelay(true)?;
let mut client = Client::connect(config, tcp.compat_write()).await?;

// ✅ Parameterized query
let results = client
    .query(
        "SELECT [id], [name] FROM [Products] WHERE [category] = @P1",
        &[&"Electronics"],
    )
    .await?
    .into_results()
    .await?;

for row in &results[0] {
    let id: i32 = row.get(0).unwrap_or_default();
    let name: &str = row.get(1).unwrap_or_default();
    println!("Product {}: {}", id, name);
}
```

❌ **DON'T:**
```rust
// ❌ Don't use SQL Server authentication in samples
config.authentication(AuthMethod::sql_server("sa", "Password123!"));

// ❌ Don't disable encryption
config.encryption(tiberius::EncryptionLevel::Off);
```

---

### DB-3: SQL Parameter Safety (MEDIUM)
**Pattern:** ALL dynamic SQL identifiers (table names, column names) must use `[brackets]`. Values MUST use parameterized queries.

✅ **DO:**
```rust
let table_name = &config.table_name;

// ✅ Always bracket-quote identifiers
let create_table = format!(
    "CREATE TABLE [{}] (
        [id] INT PRIMARY KEY,
        [name] NVARCHAR(100),
        [category] NVARCHAR(50)
    )",
    table_name
);

// ✅ Parameterized values
let query = format!(
    "SELECT [id], [name] FROM [{}] WHERE [id] = @P1",
    table_name
);
let results = client.query(&query, &[&item_id]).await?;
```

❌ **DON'T:**
```rust
// ❌ Missing brackets on dynamic identifier
let query = format!("SELECT id, name FROM {} WHERE id = @P1", table_name);

// ❌ NEVER concatenate user values into SQL
let query = format!("SELECT * FROM [Products] WHERE [name] = '{}'", user_input);
```

---

### DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

✅ **DO:**
```rust
// ✅ Batch insert with tiberius — build VALUES clause
const BATCH_SIZE: usize = 10;

for chunk in items.chunks(BATCH_SIZE) {
    let values_clause: String = chunk
        .iter()
        .enumerate()
        .map(|(i, _)| format!("(@P{}, @P{}, @P{})", i * 3 + 1, i * 3 + 2, i * 3 + 3))
        .collect::<Vec<_>>()
        .join(", ");

    let sql = format!(
        "INSERT INTO [Products] ([id], [name], [category]) VALUES {}",
        values_clause
    );

    let mut params: Vec<&dyn tiberius::ToSql> = Vec::new();
    for item in chunk {
        params.push(&item.id);
        params.push(&item.name as &dyn tiberius::ToSql);
        params.push(&item.category as &dyn tiberius::ToSql);
    }

    client.execute(&sql, &params).await?;
}

// Why batch size 10: SQL Server has ~2100 parameter limit.
// 10 rows * 3 params/row = 30 params, well under limit.
```

❌ **DON'T:**
```rust
// ❌ Row-by-row INSERT (50 round trips for 50 rows)
for item in &items {
    client.execute(
        "INSERT INTO [Products] VALUES (@P1, @P2, @P3)",
        &[&item.id, &item.name as &dyn tiberius::ToSql, &item.category as &dyn tiberius::ToSql],
    ).await?;
}
```

---

### DB-5: Azure Storage (MEDIUM)
**Pattern:** Use `azure_storage_blob` crate with `DefaultAzureCredential`.

✅ **DO:**
```rust
use azure_storage_blob::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;
use futures::StreamExt;

let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Blob Storage
let blob_service_client = BlobServiceClient::new(
    &format!("https://{}.blob.core.windows.net", account_name),
    credential.clone(),
);

let container_client = blob_service_client.container_client("mycontainer");
container_client.create().await?;

let blob_client = container_client.blob_client("myblob.txt");

// ✅ Upload
blob_client
    .put_block_blob(b"Hello, Azure!".to_vec())
    .await?;

// ✅ Download
let response = blob_client.get_content().await?;
let content = String::from_utf8(response)?;
println!("Downloaded: {}", content);

// ✅ List blobs with pagination
let mut stream = container_client.list_blobs().into_stream();
while let Some(page) = stream.next().await {
    let page = page?;
    for blob in page.blobs.blobs() {
        println!("Blob: {}", blob.name);
    }
}
```

> **⚠️ Preview Note:** The `azure_storage_blob` API may change between versions. Check the latest documentation.

---

### DB-6: Storage SAS Token Fallback (MEDIUM)
**Pattern:** When `DefaultAzureCredential` is not available (e.g., client-side scenarios), use SAS tokens as a fallback. Never hardcode SAS tokens — generate them server-side or from environment variables.

✅ **DO:**
```rust
use azure_storage_blob::prelude::*;

// ✅ Primary: Use DefaultAzureCredential (preferred)
// ✅ Fallback: SAS token from environment variable
let blob_service_client = if let Ok(sas_token) = std::env::var("AZURE_STORAGE_SAS_TOKEN") {
    // SAS fallback for environments without managed identity
    BlobServiceClient::with_sas_token(
        &format!("https://{}.blob.core.windows.net", account_name),
        &sas_token,
    )?
} else {
    // Preferred: AAD credential
    let credential = Arc::new(DefaultAzureCredential::new()?);
    BlobServiceClient::new(
        &format!("https://{}.blob.core.windows.net", account_name),
        credential,
        None,
    )
};
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode SAS tokens
let client = BlobServiceClient::with_sas_token(
    &url,
    "sv=2022-11-02&ss=b&srt=sco&sp=rwdlacup&se=2025-12-31..."  // ❌ Hardcoded SAS
)?;
```

> **⚠️ Note:** SAS token API may differ in the current `azure_storage_blob` crate — check documentation. The pattern of preferring AAD credentials over SAS tokens is the important takeaway.

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for Azure Service Bus and Event Hubs in Rust. Note that official Rust crates may be in preview or not yet available for all messaging services.

### MSG-1: Service Bus Patterns (LOW)
**Pattern:** Use the official Service Bus crate if available on crates.io (check for `azure_messaging_servicebus` or similar), or REST API via `azure_core`.

✅ **DO:**
```rust
// ✅ Check crates.io for current Service Bus crate name and availability
// The example below uses a hypothetical API — verify against actual crate docs
use azure_messaging_servicebus::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

let client = ServiceBusClient::new(
    &format!("{}.servicebus.windows.net", namespace),
    credential.clone(),
);

// ✅ Send message
let sender = client.create_sender("myqueue")?;
sender.send_message("Hello from Rust!").await?;

// ✅ Receive and complete messages
let receiver = client.create_receiver("myqueue")?;
let messages = receiver.receive_messages(10).await?;

for message in &messages {
    println!("Received: {:?}", message.body());
    receiver.complete_message(message).await?;  // ✅ Mark as processed
}
```

❌ **DON'T:**
```rust
// ❌ Don't forget to complete/abandon messages
for message in &messages {
    println!("{:?}", message.body());
    // ❌ Message never completed — will reappear in queue
}
```

> **⚠️ Preview Note:** Check crates.io for `azure_messaging_servicebus` availability. If not available, use the REST API with `azure_core` HTTP client and AAD tokens.

---

### MSG-2: Event Hubs Patterns (LOW)
**Pattern:** Check crates.io for the Event Hubs crate (e.g., `azure_messaging_event_hubs` or similar). Always use AAD authentication and handle partitioned event streams correctly.

✅ **DO:**
```rust
// ✅ Check crates.io for current Event Hubs crate name and availability
// The example below uses a hypothetical API — verify against actual crate docs
use azure_messaging_event_hubs::producer::ProducerClient;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);
let namespace = env::var("EVENTHUB_NAMESPACE")?;
let hub_name = env::var("EVENTHUB_NAME")?;

let producer = ProducerClient::new(
    format!("{}.servicebus.windows.net", namespace),
    hub_name,
    credential.clone(),
    None,
)?;

// ✅ Create batch and send events
let mut batch = producer.create_batch(None).await?;
batch.try_add_event_data("event-payload-1")?;
batch.try_add_event_data("event-payload-2")?;
producer.send_batch(batch).await?;
```

❌ **DON'T:**
```rust
// ❌ Don't use connection strings in code
let producer = ProducerClient::from_connection_string(
    "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=...",  // ❌ Use AAD
    hub_name,
)?;

// ❌ Don't send events one at a time — use batching
for payload in &payloads {
    producer.send_event(payload).await?;  // ❌ Inefficient, no batching
}
```

> **⚠️ Preview Note:** Check crates.io for `azure_messaging_eventhubs` availability. If not available, use the REST API via `reqwest` with AAD tokens from `azure_identity`, sending to the Event Hubs REST endpoint.

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault with the `azure_security_keyvault_secrets` crate.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `azure_security_keyvault_secrets` crate with `DefaultAzureCredential`.

✅ **DO:**
```rust
use azure_security_keyvault_secrets::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Secrets
let secret_client = SecretClient::new(
    &config.keyvault_url,
    credential.clone(),
    None,  // Use default ClientOptions
)?;

// ✅ Set a secret
secret_client.set("db-password", "P@ssw0rd123").await?;

// ✅ Get a secret
let secret = secret_client.get("db-password").await?;
println!("Secret value: {}", secret.value);
```

❌ **DON'T:**
```rust
// ❌ Don't hardcode secrets in samples
let db_password = "P@ssw0rd123";  // ❌ Use Key Vault

// ❌ Don't log secret values in production code
println!("Secret: {}", secret.value);  // OK in samples, not production
```

> **⚠️ Preview Note:** Check crates.io for `azure_security_keyvault_secrets` availability and current API.

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations for Azure SQL and Cosmos DB in Rust.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Serialize vectors as JSON strings for Azure SQL VECTOR type. Validate dimensions before insertion.

✅ **DO:**
```rust
use serde_json;

let embedding: Vec<f32> = get_embedding(&text).await?;
let embedding_json = serde_json::to_string(&embedding)?;

// ✅ Azure SQL — CAST as VECTOR
let sql = format!(
    "INSERT INTO [{}] ([id], [embedding]) VALUES (@P1, CAST(@P2 AS VECTOR(1536)))",
    table_name
);
client.execute(&sql, &[&item_id, &embedding_json]).await?;

// ✅ Vector distance query
let search_json = serde_json::to_string(&search_embedding)?;
let sql = format!(
    "SELECT TOP (@P1) [id], [name], \
     VECTOR_DISTANCE('cosine', [embedding], CAST(@P2 AS VECTOR(1536))) AS distance \
     FROM [{}] ORDER BY distance ASC",
    table_name
);
let results = client
    .query(&sql, &[&top_k, &search_json])
    .await?
    .into_results()
    .await?;
```

---

### VEC-2: DiskANN Index Configuration (MEDIUM)
**Pattern:** Use DiskANN indexes for large-scale vector search in Azure SQL. Configure appropriate distance metric and index parameters in SQL DDL. DiskANN provides high-recall approximate nearest neighbor search.

✅ **DO:**
```rust
// ✅ Create DiskANN index in Azure SQL via tiberius
let create_index_sql = "
    CREATE COLUMNSTORE INDEX ix_embedding_diskann
    ON [dbo].[documents] ([embedding])
    WITH (
        VECTOR_INDEX_TYPE = DISKANN,
        DISTANCE_METRIC = 'cosine',
        MAX_NEIGHBORS = 64,
        L_VALUE = 100
    );
";
client.execute(create_index_sql, &[]).await?;

// ✅ Query using DiskANN index — use VECTOR_DISTANCE with matching metric
let search_sql = "
    SELECT TOP (@P1) [id], [title],
        VECTOR_DISTANCE('cosine', [embedding], CAST(@P2 AS VECTOR(1536))) AS distance
    FROM [dbo].[documents]
    ORDER BY distance ASC
";
let results = client
    .query(search_sql, &[&top_k, &search_embedding_json])
    .await?
    .into_results()
    .await?;
```

❌ **DON'T:**
```rust
// ❌ Don't use mismatched distance metrics between index and query
// Index uses 'cosine' but query uses 'dot'
let sql = "
    SELECT TOP 10 [id],
        VECTOR_DISTANCE('dot', [embedding], CAST(@P1 AS VECTOR(1536))) AS distance
    FROM [dbo].[documents]  -- Index built with DISTANCE_METRIC = 'cosine'
    ORDER BY distance ASC
";

// ❌ Don't skip index creation for large datasets — full scan is too slow
// Without DiskANN index, queries scan all rows
```

---

## 8. Error Handling

**What this section covers:** Rust-specific error handling patterns using `Result<T, E>`, the `?` operator, custom error types, and Azure SDK error integration. Proper error handling is a MAJOR differentiator in Rust—the type system enforces it at compile time.

### ERR-1: Result<T, E> and ? Operator (HIGH)
**Pattern:** All functions that can fail must return `Result<T, E>`. Use `?` for error propagation. Never use `unwrap()` or `expect()` in main code paths.

✅ **DO:**
```rust
use std::error::Error;

// ✅ Return Result from main
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let config = load_config()?;  // ✅ Propagate with ?
    let credential = Arc::new(DefaultAzureCredential::new()?);

    let blob_client = create_blob_client(&config, credential.clone())?;
    upload_file(&blob_client, "data.json").await?;

    println!("✅ Upload complete");
    Ok(())
}

// ✅ Functions return Result
async fn upload_file(
    client: &BlobClient,
    filename: &str,
) -> Result<(), Box<dyn Error>> {
    let content = std::fs::read(filename)?;
    client.put_block_blob(content).await?;
    Ok(())
}
```

❌ **DON'T:**
```rust
// ❌ NEVER unwrap in main code paths
#[tokio::main]
async fn main() {
    let config = load_config().unwrap();  // ❌ Panics on error
    let credential = DefaultAzureCredential::new().unwrap();  // ❌ Panics
    let data = std::fs::read_to_string("data.json").unwrap();  // ❌ Panics
}

// ❌ Don't swallow errors silently
async fn upload(client: &BlobClient, data: &[u8]) {
    let _ = client.put_block_blob(data.to_vec()).await;  // ❌ Error silently ignored
}
```

**Why:** `unwrap()` causes panics which crash the program with no useful error message. `Result` + `?` propagates errors with context, allowing callers to handle failures gracefully.

---

### ERR-2: Custom Error Types with thiserror (MEDIUM)
**Pattern:** For samples with multiple error sources, use `thiserror` to create typed error enums.

✅ **DO:**
```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum SampleError {
    #[error("Configuration error: {0}")]
    Config(String),

    #[error("Azure authentication failed: {0}")]
    Auth(#[from] azure_core::Error),

    #[error("Database error: {0}")]
    Database(#[from] tiberius::error::Error),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("JSON serialization error: {0}")]
    Json(#[from] serde_json::Error),
}

async fn run() -> Result<(), SampleError> {
    let config = load_config().map_err(|e| SampleError::Config(e.to_string()))?;
    let credential = DefaultAzureCredential::new()?;  // Auto-converts via #[from]
    let data = std::fs::read("data.json")?;  // Auto-converts via #[from]
    Ok(())
}
```

❌ **DON'T:**
```rust
// ❌ Don't use String as error type (loses type information)
async fn run() -> Result<(), String> {
    let config = load_config().map_err(|e| e.to_string())?;
    Ok(())
}

// ❌ Don't use Box<dyn Error> everywhere in complex samples
// (OK for simple samples, but typed errors are better for complex ones)
```

**Why:** `thiserror` provides ergonomic error types with automatic `From` implementations, making error propagation with `?` seamless across different error types.

---

### ERR-3: Azure Error Handling (HIGH)
**Pattern:** Handle `azure_core::Error` with contextual messages and troubleshooting hints for common Azure errors.

✅ **DO:**
```rust
use azure_core::Error as AzureError;

async fn get_secret(
    client: &SecretClient,
    name: &str,
) -> Result<String, Box<dyn std::error::Error>> {
    match client.get(name).await {
        Ok(secret) => Ok(secret.value.clone()),
        Err(e) => {
            eprintln!("❌ Failed to get secret '{}': {}", name, e);
            eprintln!("\nTroubleshooting:");
            eprintln!("  1. Run 'az login' to authenticate with Azure CLI");
            eprintln!("  2. Verify you have the 'Key Vault Secrets User' role");
            eprintln!("  3. Check your Key Vault URL is correct: {}", client.vault_url());
            eprintln!("  4. Ensure firewall rules allow access from your IP");
            Err(e.into())
        }
    }
}
```

❌ **DON'T:**
```rust
// ❌ Don't provide generic error messages
match client.get("my-secret").await {
    Ok(secret) => println!("{}", secret.value),
    Err(e) => eprintln!("Error: {}", e),  // ❌ No context or troubleshooting
}
```

---

### ERR-4: Panic vs Result (CRITICAL)
**Pattern:** NEVER use `panic!()`, `unwrap()`, or `expect()` in sample code paths. Reserve these for truly unrecoverable situations or test code only.

✅ **DO:**
```rust
// ✅ Use Result for all fallible operations
fn parse_config(value: &str) -> Result<u32, std::num::ParseIntError> {
    value.parse::<u32>()
}

// ✅ Use expect() ONLY in tests
#[cfg(test)]
mod tests {
    #[test]
    fn test_parse_config() {
        let result = parse_config("42").expect("should parse valid number");
        assert_eq!(result, 42);
    }
}

// ✅ If unwrap is truly safe, document why
let home_dir = dirs::home_dir()
    .ok_or("HOME directory not found — required for config file location")?;
```

❌ **DON'T:**
```rust
// ❌ CRITICAL: Don't panic in samples
fn main() {
    let port: u16 = env::var("PORT").unwrap().parse().unwrap();
    // Two potential panics — user gets unhelpful "called unwrap() on Err" message
}

// ❌ Don't use panic! for error handling
if items.is_empty() {
    panic!("No items found!");  // ❌ Use Result::Err instead
}
```

**Why:** Panics crash the program with a stack trace—unhelpful for users running samples. `Result` provides actionable error messages. Rust's type system makes this easy to enforce.

---

## 9. Data Management

**What this section covers:** Sample data handling, embedded files, JSON loading, and data validation.

### DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Use `include_str!` or `include_bytes!` for embedded data, or load from filesystem at runtime.

✅ **DO:**
```
repo/
├── data/
│   ├── products.json              # ✅ Sample data
│   ├── products-with-vectors.json # ✅ Pre-computed embeddings
├── src/
│   ├── main.rs                    # Loads data files
```

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Product {
    id: String,
    name: String,
    category: String,
    embedding: Option<Vec<f32>>,
}

// ✅ Option 1: Embed data at compile time (small files)
const PRODUCTS_JSON: &str = include_str!("../data/products.json");

fn load_embedded_products() -> Result<Vec<Product>, serde_json::Error> {
    serde_json::from_str(PRODUCTS_JSON)
}

// ✅ Option 2: Load from filesystem at runtime (large files)
fn load_products_from_file(path: &str) -> Result<Vec<Product>, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let products: Vec<Product> = serde_json::from_str(&content)?;
    Ok(products)
}
```

❌ **DON'T:**
```
repo/
├── data/
│   ├── products.json              # ✅ Raw data
│   ├── .gitignore                 # ❌ products-with-vectors.json gitignored
├── src/
│   ├── main.rs                    # ❌ Fails: file not found at runtime
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list**—not just the immediate project directory. Data files often live in sibling directories, parent directories, or shared `data/` folders.
> 2. **Trace the file path in code** relative to the working directory the binary runs from.
> 3. **Check for monorepo patterns**—in sample collections, data may be shared across multiple samples via a common parent directory.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR or repo at the path the code resolves to at runtime.

---

### DATA-2: JSON Data Loading with serde (MEDIUM)
**Pattern:** Use `serde` and `serde_json` for type-safe JSON handling. Always deserialize into typed structs.

✅ **DO:**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Hotel {
    id: String,
    name: String,
    description: String,
    #[serde(default)]
    embedding: Vec<f32>,
}

fn load_hotels(path: &str) -> Result<Vec<Hotel>, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let hotels: Vec<Hotel> = serde_json::from_str(&content)?;

    println!("Loaded {} hotels from {}", hotels.len(), path);
    Ok(hotels)
}
```

❌ **DON'T:**
```rust
// ❌ Don't use untyped serde_json::Value for known data structures
let data: serde_json::Value = serde_json::from_str(&content)?;
let name = data["name"].as_str().unwrap();  // ❌ Fragile, no compile-time checks
```

---

## 10. Sample Hygiene

**What this section covers:** Repository hygiene, security, and governance. Covers .gitignore patterns, environment file protection, license files, and repository governance.

### HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and target directory with comprehensive `.gitignore`.

✅ **DO:**
```gitignore
# Build artifacts
target/

# Environment variables (may contain credentials)
.env
.env.local
.env.*.local
.env.development
.env.production
!.env.sample
!.env.example

# Azure
.azure/

# IDE
.vscode/
.idea/
*.swp
*.sublime-*

# OS
.DS_Store
Thumbs.db
*.log

# Rust-specific
**/*.rs.bk
*.pdb
```

❌ **DON'T:**
```
repo/
├── .env                    # ❌ Live credentials committed!
├── target/                 # ❌ Build artifacts committed (hundreds of MB)
├── Cargo.lock              # ✅ Should be committed for binaries
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed, you MUST verify the file is actually tracked by git:
> 1. **Check .gitignore**—look in the project directory AND all parent directories for `.gitignore` entries covering `.env`.
> 2. **Run `git ls-files .env`**—if it returns empty, the file is NOT tracked and is NOT a security issue.
> 3. A `.env` file that exists on disk but is gitignored is working as designed—developers create it locally from `.env.sample`.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked, or if no `.gitignore` exists at all.

---

### HYG-2: .env.sample (HIGH)
**Pattern:** Provide `.env.sample` with placeholder values. Never commit actual `.env` or any `.env.*` files (except `.env.sample` / `.env.example`).

✅ **DO:**
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
```

❌ **DON'T:**
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # ❌ Real subscription ID
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321       # ❌ Real tenant ID
```

---

### HYG-3: Dead Code (MEDIUM)
**Pattern:** Remove unused files, functions, and imports. Commented-out code confuses users.

✅ **DO:**
```rust
// Only import what you use
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
```

❌ **DON'T:**
```rust
// ❌ Commented-out code confuses users
// use azure_cosmos::prelude::*;
//
// async fn old_implementation() -> Result<(), Box<dyn Error>> {
//     // This was the old way...
// }

use azure_storage_blob::prelude::*;
```

**Why:** Dead code significantly confuses users trying to learn from samples. Rust's compiler warns about unused imports—heed those warnings.

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── Cargo.toml
├── src/
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── Cargo.toml
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT**—look for `LICENSE`, `LICENSE.md`, `LICENSE.txt`, or similar at the repository root.
> 2. **Check parent directories**—in monorepos and sample collections, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.
> 4. Only flag if NO license file exists at ANY level of the repo hierarchy above the sample.

---

### HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference or include governance files: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

✅ **DO:**
```markdown
# README.md

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA). See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## Security

Microsoft takes security seriously. If you believe you have found a security vulnerability,
please report it as described in [SECURITY.md](SECURITY.md).

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
```

---

## 11. README & Documentation

**What this section covers:** Documentation quality, accuracy, and completeness. Covers expected output, troubleshooting, prerequisites, and setup instructions.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:
```bash
cargo run
```

You should see output similar to:

```
✅ Connected to Azure Blob Storage
✅ Container 'samples' created
✅ Uploaded blob 'sample.txt' (14 bytes)
✅ Downloaded blob content: "Hello, Azure!"
```

> Note: Exact output may vary based on your Azure environment.
```

❌ **DON'T:**
```markdown
## Expected Output

```
✅ Blob uploaded successfully  # ❌ Not actual output, fabricated
```
```

---

### DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

✅ **DO:**
```markdown
## Project Structure

- [`src/main.rs`](./src/main.rs) — Main entry point
- [`src/config.rs`](./src/config.rs) — Configuration loader
- [`infra/main.bicep`](./infra/main.bicep) — Infrastructure template
- [`Cargo.toml`](./Cargo.toml) — Dependencies and metadata
```

❌ **DON'T:**
```markdown
- [`src/main.rs`](./rust/src/main.rs)  # ❌ Wrong path
```

---

### DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure and Rust errors.

✅ **DO:**
```markdown
## Troubleshooting

### Authentication Errors

If you see "Failed to acquire token":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments (see Prerequisites)

### Build Errors

If `cargo build` fails:
1. Ensure Rust 1.75+ is installed: `rustup update stable`
2. Check required system dependencies (OpenSSL dev headers on Linux)
3. Verify all environment variables are set (some crates need them at build time)

### Common Azure SDK Errors

- `azure_core::Error` with "Unauthorized": Run `az login` or check role assignments
- `azure_core::Error` with "NotFound": Verify resource exists and endpoint URL is correct
- `tiberius::Error` with connection refused: Check firewall rules for Azure SQL
```

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly (Azure subscription, Rust toolchain, role assignments, services).

✅ **DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Rust**: Version 1.75 or later ([Install via rustup](https://rustup.rs/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional, for infrastructure deployment)

### Azure Resources

This sample requires:
- **Azure Storage Account** with a blob container
- **Azure Key Vault** (if using secrets)

### Role Assignments

Your Azure identity needs these role assignments:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
```

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions. Include Azure resource provisioning.

✅ **DO:**
```markdown
## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Azure-Samples/azure-storage-blob-rust.git
cd azure-storage-blob-rust
```

### 2. Build the project

```bash
cargo build
```

### 3. Provision Azure resources

```bash
azd up
```

### 4. Run the sample

```bash
cargo run
```
```

---

### DOC-6: Rust Version Strategy (LOW)
**Pattern:** Document minimum Rust version in both README and Cargo.toml `rust-version` field.

✅ **DO:**
```toml
# Cargo.toml
[package]
rust-version = "1.75"
```

```markdown
## Prerequisites

- **Rust**: Version 1.75 or later required. Install via [rustup](https://rustup.rs/).
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

✅ **DO:**
```markdown
## Configuration

Copy `.env.sample` to `.env` and fill in your values:

```bash
cp .env.sample .env
```

Edit `.env` and replace placeholders:
- `AZURE_STORAGE_ACCOUNT_NAME`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `AZURE_KEYVAULT_URL`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI
```

❌ **DON'T:**
```markdown
Set environment variables:
- `AZURE_STORAGE_ACCOUNT_NAME=<your-storage-account>`  # ❌ How do I find this?
```

---

## 12. Infrastructure (Bicep/Terraform)

**What this section covers:** Infrastructure-as-code patterns for Azure resources. These patterns are language-agnostic and apply equally to Rust samples.

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

✅ **DO:**
```bicep
// ✅ Current AVM modules
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
  name: 'storage-deployment'
  params: {
    name: storageAccountName
    location: location
  }
}
```

❌ **DON'T:**
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.7.0' = {
  // ❌ Outdated version
}
```

---

### IaC-2: Bicep Parameter Validation (CRITICAL)
**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

✅ **DO:**
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

❌ **DON'T:**
```bicep
param aadAdminObjectId string  // ❌ No validation, accepts empty string
```

---

### IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2023+). Avoid versions older than 2 years.

✅ **DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
```

❌ **DON'T:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // ❌ 5+ years old
}
```

---

### IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities to access Azure resources.

✅ **DO:**
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

**Common role IDs:**
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cosmos DB Account Reader Role: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

### IaC-5: Network Security (HIGH)
**Pattern:** For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

✅ **DO (Quickstart):**
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

### IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

✅ **DO:**
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_STORAGE_BLOB_ENDPOINT string = storageAccount.properties.primaryEndpoints.blob
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
```

---

### IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

✅ **DO:**
```bicep
var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

❌ **DON'T:**
```bicep
var storageAccountName = 'mystorageaccount123'  // ❌ Inconsistent naming
```

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns for Rust samples.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata for Rust projects.

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `azure.yaml` as missing or incomplete:
> 1. The `services`, `hooks`, and `host` fields are **OPTIONAL**. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct.
> 2. **Check parent directories**—in monorepo layouts, `azure.yaml` often lives one or more levels ABOVE the language-specific project folder.

✅ **DO:**
```yaml
name: azure-storage-blob-rust-sample
metadata:
  template: azure-storage-blob-rust-sample@0.0.1

services:
  app:
    project: ./
    language: rust
    host: containerapp  # Rust samples typically use containerapp or appservice
    docker:
      path: ./Dockerfile

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "❌ Not logged in. Run 'az login'" && exit 1)
      rustc --version || (echo "❌ Rust not installed. Visit https://rustup.rs/" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "✅ Provisioning complete"
      echo "Run 'cargo run' to test the sample."
```

---

### AZD-2: Dockerfile for Rust (MEDIUM)
**Pattern:** Use multi-stage Docker builds for Rust samples deployed via azd.

✅ **DO:**
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

❌ **DON'T:**
```dockerfile
# ❌ Don't ship debug builds
FROM rust:1.75
COPY . .
RUN cargo build
CMD ["./target/debug/azure-sample"]  # ❌ Debug build, large image
```

---

## 14. Rust Idioms

**What this section covers:** Rust-specific patterns that distinguish idiomatic Azure SDK Rust samples from direct ports of other languages.

### RS-1: Ownership and Borrowing in Client APIs (HIGH)
**Pattern:** Use `Arc<T>` for sharing credential across clients. Pass references where possible instead of cloning.

✅ **DO:**
```rust
use std::sync::Arc;
use azure_identity::DefaultAzureCredential;

// ✅ Share credential via Arc
let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Clone Arc (cheap reference count increment)
let blob_client = create_blob_client(credential.clone());
let kv_client = create_keyvault_client(credential.clone());

// ✅ Pass data by reference when possible
async fn process_items(items: &[Product]) -> Result<(), Box<dyn std::error::Error>> {
    for item in items {
        println!("Processing: {}", item.name);
    }
    Ok(())
}

// ✅ Use &str instead of String for function parameters
fn format_endpoint(account_name: &str) -> String {
    format!("https://{}.blob.core.windows.net", account_name)
}
```

❌ **DON'T:**
```rust
// ❌ Don't clone expensive types unnecessarily
let credential = DefaultAzureCredential::new()?;
let cred_clone1 = credential.clone();  // ❌ Full clone (if not Arc-wrapped)
let cred_clone2 = credential.clone();

// ❌ Don't take ownership when a reference suffices
fn process_items(items: Vec<Product>) {  // ❌ Takes ownership, forces caller to clone
    for item in &items {
        println!("{}", item.name);
    }
}
```

**Why:** Rust's ownership model prevents data races at compile time. Using `Arc` for shared resources and `&T` for borrowed data is idiomatic and zero-cost.

---

### RS-2: Async Runtime (tokio) Configuration (HIGH)
**Pattern:** Use `tokio` as the async runtime. Configure appropriately for sample complexity. Consider runtime flavor for resource-constrained environments.

✅ **DO:**
```rust
// ✅ Simple sample — use tokio::main macro with full features (multi-thread default)
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = load_config()?;
    run_sample(&config).await?;
    Ok(())
}

// ✅ For Azure Functions or resource-constrained environments:
#[tokio::main(flavor = "current_thread")]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Single-threaded runtime — lower overhead, suitable for simple samples
    run_sample().await
}

// ✅ Cargo.toml — enable full tokio features for samples
// [dependencies]
// tokio = { version = "1", features = ["full"] }
```

```rust
// ✅ For concurrent operations — use tokio::spawn or join!
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

// ✅ For CPU-heavy work, use spawn_blocking to avoid starving the async runtime
let result = tokio::task::spawn_blocking(move || {
    // CPU-intensive computation (e.g., hashing, compression, parsing large files)
    compute_hash(&large_data)
}).await??;
```

❌ **DON'T:**
```rust
// ❌ Don't block the async runtime
#[tokio::main]
async fn main() {
    let data = std::fs::read_to_string("large_file.json").unwrap();  // ❌ Blocking IO in async
    // Use tokio::fs::read_to_string for large files, or spawn_blocking for CPU work
}

// ❌ Don't mix async runtimes
// Using both tokio and async-std causes conflicts
```

---

### RS-3: Error Propagation Patterns (MEDIUM)
**Pattern:** Use `?` with `map_err()` for adding context. Consider `anyhow` for sample-level error handling.

✅ **DO:**
```rust
use std::error::Error;

// ✅ Add context with map_err
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

// ✅ Use anyhow for convenience in samples (optional)
// [dependencies]
// anyhow = "1"
use anyhow::{Context, Result};

async fn run() -> Result<()> {
    let config = load_config()
        .context("Failed to load configuration — check .env file")?;
    let credential = DefaultAzureCredential::new()
        .context("Failed to create Azure credential — run 'az login'")?;
    Ok(())
}
```

❌ **DON'T:**
```rust
// ❌ Don't lose error context
let credential = DefaultAzureCredential::new()?;  // OK but no context
// If this fails, user sees raw Azure error without guidance
```

---

### RS-4: Logging with tracing Crate (LOW)
**Pattern:** Use `tracing` crate for structured logging. Configure with `tracing-subscriber`.

✅ **DO:**
```rust
use tracing::{info, warn, error};
use tracing_subscriber;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ✅ Initialize logging
    tracing_subscriber::fmt()
        .with_env_filter("azure=info,sample=debug")
        .init();

    info!("Starting Azure Storage sample");

    match upload_blob(&client, "test.txt", b"Hello").await {
        Ok(_) => info!("Upload successful"),
        Err(e) => error!("Upload failed: {}", e),
    }

    Ok(())
}
```

```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

❌ **DON'T:**
```rust
// ❌ Don't use println! for all logging in production-style samples
println!("Starting upload...");
println!("Error: {}", e);
// OK for simple quickstarts, but tracing is better for complex samples
```

---

### RS-5: Feature Flags for Optional Dependencies (MEDIUM)
**Pattern:** Use Cargo feature flags when a sample has optional components (e.g., vector search, AI features).

✅ **DO:**
```toml
# Cargo.toml
[features]
default = ["storage"]
storage = ["azure_storage_blob"]
ai = ["async-openai"]
vector-search = ["ai"]

[dependencies]
azure_identity = "0.x"       # Check crates.io for latest
azure_core = "0.x"           # Check crates.io for latest
azure_storage_blob = { version = "0.x", optional = true }   # Check crates.io for latest
async-openai = { version = "0.25", optional = true }
```

```rust
// ✅ Conditional compilation
#[cfg(feature = "ai")]
async fn generate_embedding(text: &str) -> Result<Vec<f32>, Box<dyn Error>> {
    // AI-specific code
    todo!()
}

#[cfg(not(feature = "ai"))]
async fn generate_embedding(_text: &str) -> Result<Vec<f32>, Box<dyn Error>> {
    Err("AI feature not enabled. Build with: cargo build --features ai".into())
}
```

---

### RS-6: Send + Sync Bounds for Async Use (HIGH)
**Pattern:** Azure SDK types shared across async tasks must be `Send + Sync`. Ensure your types that wrap SDK clients satisfy these bounds.

✅ **DO:**
```rust
use std::sync::Arc;
use azure_identity::DefaultAzureCredential;

// ✅ Arc<DefaultAzureCredential> is Send + Sync — safe to share across tasks
let credential = Arc::new(DefaultAzureCredential::new()?);

// ✅ Spawn tasks that share the credential
let cred = credential.clone();
let handle = tokio::spawn(async move {
    // cred is Send + Sync, so this compiles
    let token = cred.get_token(&["https://storage.azure.com/.default"]).await?;
    Ok::<_, Box<dyn std::error::Error + Send + Sync>>(token)
});
```

❌ **DON'T:**
```rust
// ❌ Don't use Rc<T> (not Send) in async code shared across tasks
use std::rc::Rc;
let credential = Rc::new(DefaultAzureCredential::new()?);  // ❌ Rc is !Send
// tokio::spawn(async move { credential.get_token(...) })  // Won't compile
```

**Why:** Tokio's multi-threaded runtime requires futures to be `Send`. All Azure SDK client types are designed to be `Send + Sync` when wrapped in `Arc`.

---

### RS-7: TokenCredential Trait Bounds (MEDIUM)
**Pattern:** When writing generic functions that accept any Azure credential, use `TokenCredential` trait bounds with appropriate lifetime and Send/Sync constraints.

✅ **DO:**
```rust
use azure_core::auth::TokenCredential;
use std::sync::Arc;

// ✅ Accept any credential type via trait object
async fn get_storage_token(
    credential: &dyn TokenCredential,
) -> Result<String, azure_core::Error> {
    let response = credential
        .get_token(&["https://storage.azure.com/.default"])
        .await?;
    Ok(response.token.clone())
}

// ✅ Or accept via Arc for shared ownership
async fn create_clients(
    credential: Arc<dyn TokenCredential>,
) -> Result<(), Box<dyn std::error::Error>> {
    let blob_client = BlobServiceClient::new(url, credential.clone(), None);
    let kv_client = SecretClient::new(kv_url, credential.clone(), None)?;
    Ok(())
}
```

---

## 15. CI/CD & Testing

**What this section covers:** Continuous integration patterns, testing, and build validation for Rust Azure SDK samples.

### CI-1: Cargo Test and Clippy in CI (HIGH)
**Pattern:** Run `cargo clippy`, `cargo test`, and `cargo audit` in CI.

✅ **DO:**
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

```toml
# Cargo.toml — add integration test support
[[test]]
name = "integration"
path = "tests/integration.rs"
```

---

### CI-2: Test Patterns for Azure SDK Samples (MEDIUM)
**Pattern:** Provide unit tests with mocks for Azure services. Integration tests should be optional and documented.

✅ **DO:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_config_loading() {
        // ✅ Test configuration validation
        // ⚠️ Note: std::env::set_var is `unsafe` in Rust 2024 edition.
        // For 2021 edition this is safe, but for 2024 wrap in unsafe {}:
        //   unsafe { std::env::set_var("KEY", "value"); }
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

### CI-3: cargo-deny for License and Vulnerability Auditing (MEDIUM)
**Pattern:** Use `cargo deny check` in CI to audit licenses and detect known vulnerabilities in the dependency tree.

✅ **DO:**
```yaml
# .github/workflows/ci.yml (add step)
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

**Why:** `cargo deny` goes beyond `cargo audit` — it also checks license compatibility, duplicate crate versions, and banned crates. Essential for Azure samples that must meet Microsoft Open Source compliance.

---

## Pre-Review Checklist (Comprehensive)

Use this comprehensive checklist before submitting an Azure SDK Rust sample for review:

### 🔧 Project Setup
- [ ] Rust edition 2021 in Cargo.toml
- [ ] `rust-version` field set to minimum supported version
- [ ] Cargo.toml has `authors`, `license`, `description`, `publish = false`
- [ ] Every dependency is used in code (no phantom deps)
- [ ] Using official `azure_*` crates (not unofficial wrappers)
- [ ] Environment variables validated with descriptive errors
- [ ] `dotenvy` for .env loading (not raw file parsing)
- [ ] `cargo clippy` passes with no warnings
- [ ] Cargo.lock committed (for binary crates)
- [ ] `cargo audit` passes (no critical/high CVEs)
- [ ] `rustfmt.toml` present and `cargo fmt --check` passes

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `.env`, `.env.*`, `target/`, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed
- [ ] No build artifacts committed (`target/`)
- [ ] Dead code removed (unused imports, functions, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included

### ☁️ Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential wrapped in `Arc` and reused across clients
- [ ] Client options configured (retry, timeout where applicable)
- [ ] Token refresh implemented for long-running operations
- [ ] Managed identity pattern documented in README
- [ ] Pagination handled with async streams (`into_stream()`, `StreamExt`)
- [ ] Resource cleanup via RAII (Drop trait, scoped blocks)

### 🦀 Rust Idioms
- [ ] `Result<T, E>` and `?` operator for all fallible operations
- [ ] No `unwrap()` or `expect()` in main code paths
- [ ] Custom error types with `thiserror` for complex samples
- [ ] `Arc<T>` for sharing resources across async tasks (must be `Send + Sync`)
- [ ] References (`&T`) preferred over cloning where possible
- [ ] `tokio` runtime configured correctly (consider `flavor = "current_thread"` for constrained envs)
- [ ] `tracing` for structured logging (complex samples)
- [ ] Feature flags for optional dependencies
- [ ] `spawn_blocking` used for CPU-heavy work in async context

### 🗄️ Data Services (if applicable)
- [ ] SQL: tiberius with AAD token authentication
- [ ] SQL: All dynamic identifiers use `[brackets]`
- [ ] SQL: Values parameterized (no string concatenation)
- [ ] Cosmos: Queries include partition key
- [ ] Storage: Blob client patterns with AAD auth
- [ ] Batch operations for multiple rows (not row-by-row)
- [ ] Pre-computed data files committed to repo

### ❌ Error Handling
- [ ] All functions return `Result<T, E>` (no panics in main paths)
- [ ] Error messages are contextual with troubleshooting hints
- [ ] `map_err()` or `anyhow::Context` for error context
- [ ] Azure errors include service-specific troubleshooting

### 📄 Documentation
- [ ] README "Expected output" copy-pasted from real run
- [ ] All internal folder/file links match actual paths
- [ ] Prerequisites section complete (Rust version, Azure CLI, role assignments)
- [ ] Troubleshooting section covers common errors
- [ ] Setup instructions clear and tested
- [ ] Placeholder values have clear replacement instructions
- [ ] Rust version documented in README and Cargo.toml

### 🏗️ Infrastructure (if applicable)
- [ ] AVM module versions current
- [ ] Bicep parameters validated with decorators
- [ ] API versions current (2023+)
- [ ] RBAC role assignments in Bicep
- [ ] Resource naming follows CAF conventions
- [ ] `azure.yaml` complete (or minimal for infra-only)
- [ ] Multi-stage Dockerfile for containerized deployment

### 🧪 CI/CD
- [ ] `cargo clippy -- -D warnings` in CI
- [ ] `cargo fmt --check` in CI
- [ ] `cargo test` in CI
- [ ] `cargo audit` in CI
- [ ] `cargo deny check` in CI (license + vulnerability audit)
- [ ] Build succeeds in CI

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`azure-sdk-typescript-sample-review`](../azure-sdk-typescript-sample-review/SKILL.md)**: TypeScript-specific Azure SDK patterns (the template this skill was adapted from)
- **[`azure-sdk-dotnet-sample-review`](../azure-sdk-dotnet-sample-review/SKILL.md)**: .NET 9/10 + Aspire Azure SDK patterns
- **[`azure-sdk-java-sample-review`](../azure-sdk-java-sample-review/SKILL.md)**: Java 17/21 + Spring Boot Azure SDK patterns
- **[`azure-sdk-python-sample-review`](../azure-sdk-python-sample-review/SKILL.md)**: Python 3.9+ + async Azure SDK patterns
- **[`azure-sdk-go-sample-review`](../azure-sdk-go-sample-review/SKILL.md)**: Go 1.21+ Azure SDK patterns
- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency

---

## Scope Note: Azure Services with Rust SDK Support

The Azure SDK for Rust is still maturing. The following shows approximate crate availability:

### Available on crates.io (check for latest versions):
- `azure_identity` — Authentication (DefaultAzureCredential, managed identity)
- `azure_core` — Core types, error handling, HTTP pipeline
- `azure_storage_blob` — Blob Storage
- `azure_storage` — Storage common types
- `azure_security_keyvault_secrets` — Key Vault secrets, keys, certificates
- `azure_cosmos` — Cosmos DB

### Limited or Preview Availability:
- Azure Service Bus — Check crates.io for `azure_messaging_servicebus` or similar
- Azure Event Hubs — Check crates.io for `azure_messaging_event_hubs` or similar; may require REST API
- Azure OpenAI — Community crate `async-openai` supports Azure; check crates.io for official `azure_openai`
- Azure SQL — Use `tiberius` crate with AAD tokens
- Azure AI Search — May require REST API approach
- Azure Communication Services — REST API approach
- Azure Monitor — REST API approach

### Not Yet Available (use REST API via `azure_core` or `reqwest`):
- Azure Event Grid
- Azure App Configuration
- Azure Cache for Redis
- Azure SignalR

For services without official Rust crates, apply the core patterns from Sections 1–2 (Project Setup, Azure SDK Client Patterns) and use `azure_core` HTTP client or `reqwest` with AAD tokens from `azure_identity`.

> **Always check [crates.io](https://crates.io/search?q=azure_) and the [Azure SDK for Rust GitHub repo](https://github.com/Azure/azure-sdk-for-rust) for the latest availability.**

---

## Reference Links

### Azure SDK for Rust
- [Azure SDK for Rust — GitHub](https://github.com/Azure/azure-sdk-for-rust)
- [azure_identity on crates.io](https://crates.io/crates/azure_identity)
- [azure_core on crates.io](https://crates.io/crates/azure_core)
- [azure_storage_blob on crates.io](https://crates.io/crates/azure_storage_blob)
- [azure_cosmos on crates.io](https://crates.io/crates/azure_cosmos)
- [azure_security_keyvault_secrets on crates.io](https://crates.io/crates/azure_security_keyvault_secrets)

### Rust Ecosystem
- [Rust Installation — rustup.rs](https://rustup.rs/)
- [Rust 2021 Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2021/)
- [Rust 2024 Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2024/)
- [tokio — Async Runtime](https://tokio.rs/)
- [serde — Serialization Framework](https://serde.rs/)
- [thiserror — Error Derive Macro](https://crates.io/crates/thiserror)
- [anyhow — Error Handling for Applications](https://crates.io/crates/anyhow)
- [tracing — Structured Logging](https://crates.io/crates/tracing)
- [tiberius — SQL Server/Azure SQL Driver](https://crates.io/crates/tiberius)
- [dotenvy — .env File Loader](https://crates.io/crates/dotenvy)
- [cargo-audit — Security Auditing](https://crates.io/crates/cargo-audit)
- [cargo-deny — License and Vulnerability Auditing](https://crates.io/crates/cargo-deny)

### Azure Authentication & Identity
- [DefaultAzureCredential](https://learn.microsoft.com/azure/developer/rust/azure-sdk-overview)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework — Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK Rust sample patterns** adapted from the comprehensive TypeScript review skill and generalized for the Rust Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (9 rules): Hardcoded secrets, phantom deps, CVE scanning, token refresh, AVM versions, parameter validation, .gitignore, fabricated output, panic in samples
- **HIGH** (24 rules): Client construction, managed identity, pagination, Rust edition, lock file, OpenAI config, Cosmos patterns, SQL patterns, batch operations, RBAC, network security, resource naming, ownership patterns, async runtime, pre-computed data, .env.sample, LICENSE, prerequisites, CI/CD, Send+Sync bounds, crate naming, Key Vault, Azure errors, error propagation
- **MEDIUM** (32 rules): Client options, config loading, clippy, feature flags, crate legitimacy, credential config, resource cleanup, embedding validation, API version docs, SQL safety, Storage, Storage SAS fallback, messaging, error types, JSON loading, troubleshooting, setup, placeholders, API versions, governance, output values, azd structure, Dockerfile, error propagation patterns, feature flag patterns, test patterns, dead code, cargo-deny, TokenCredential bounds, vector types, DiskANN config, Document Intelligence
- **LOW** (5 rules): rustfmt config, tracing logging, Rust version docs, Service Bus patterns, Event Hubs patterns

### Service Coverage
- **Core SDK**: Authentication, credentials, managed identities, client patterns, token management, pagination, RAII cleanup
- **Data**: Cosmos DB, Azure SQL (tiberius), Storage (Blob), batch operations
- **AI**: Azure OpenAI (via async-openai), embeddings, vector dimensions
- **Messaging**: Service Bus (if crate available), REST API fallback
- **Security**: Key Vault (secrets, keys, certificates)
- **Vector Search**: Azure SQL, Cosmos DB
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming
- **Rust Idioms**: Ownership/borrowing, async runtime (tokio flavors, spawn_blocking), error propagation, tracing, feature flags, Send+Sync bounds, TokenCredential trait bounds

### Key Differentiators from TypeScript Skill
- **Error handling**: Rust's type system enforces `Result<T, E>` at compile time—no silent failures possible
- **Resource cleanup**: RAII via `Drop` trait replaces `try/finally` and `Symbol.asyncDispose`
- **Ownership**: `Arc<T>` sharing pattern replaces JavaScript's reference-by-default
- **SDK maturity**: Confidence is "medium" because Azure SDK for Rust is still evolving
- **Async runtime**: `tokio` configuration is a Rust-specific concern

Apply these patterns to ensure Azure SDK Rust samples are **secure, idiomatic, well-documented, and ready for publication**.
