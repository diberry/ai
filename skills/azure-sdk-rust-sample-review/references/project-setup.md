# Project Setup & Configuration

Rules PS-1 through PS-11. Cargo project structure, Rust edition, dependency management, environment variables, and tooling configuration.

## PS-1: Rust Edition (HIGH)

Use Rust 2021 edition (latest stable). Set explicitly in Cargo.toml.

DO:
```toml
[package]
name = "azure-storage-blob-quickstart"
version = "0.1.0"
edition = "2021"
rust-version = "1.75"
```

DON'T:
```toml
[package]
name = "azure-storage-blob-quickstart"
version = "0.1.0"
edition = "2018"  # Outdated edition
# Missing rust-version
```

> 2024 Edition Note: `std::env::set_var` and `std::env::remove_var` are `unsafe` in 2024 edition. Most samples should use 2021 for now.

---

## PS-2: Cargo.toml Metadata (MEDIUM)

DO:
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

DON'T:
```toml
[package]
name = "my-sample"
version = "0.1.0"
edition = "2021"
# Missing authors, license, repository, description, publish = false
```

---

## PS-3: Dependency Audit (CRITICAL)

Every dependency in `[dependencies]` must be used in code. No phantom dependencies. Use official `azure_*` crates.

DO:
```toml
[dependencies]
azure_identity = "0.x"
azure_storage_blob = "0.x"
azure_core = "0.x"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
dotenvy = "0.15"
```

DON'T:
```toml
[dependencies]
azure_identity = "0.x"
azure_storage_blob = "0.x"
azure_cosmos = "0.x"  # Listed but never used
reqwest = "0.12"      # Not imported anywhere
rand = "0.8"          # Phantom dependency
```

---

## PS-4: Azure SDK Crate Naming (HIGH)

Use official Azure SDK crates with `azure_*` prefix from crates.io. Verify publisher is `azure-sdk` or Microsoft.

DO:
```toml
[dependencies]
azure_identity = "0.x"
azure_core = "0.x"
azure_storage_blob = "0.x"
azure_security_keyvault_secrets = "0.x"
azure_cosmos = "0.x"
```

```rust
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
use azure_core::Error as AzureError;
```

DON'T:
```toml
[dependencies]
azur_storage = "0.1.0"           # Typosquatting
azure-storage-blob = "0.1.0"     # Wrong naming (uses hyphens)
unofficial_azure_sdk = "0.1.0"   # Not official
```

> Preview Note: Some Azure service crates may not yet exist. Check crates.io for availability.

---

## PS-5: Configuration (MEDIUM)

Use `dotenvy` crate for `.env` file loading. Validate all required environment variables with descriptive errors.

DO:
```rust
use std::env;

#[derive(Debug)]
struct Config {
    storage_account_name: String,
    keyvault_url: String,
}

fn load_config() -> Result<Config, Box<dyn std::error::Error>> {
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

DON'T:
```rust
// Don't silently fall back to defaults
let account = env::var("AZURE_STORAGE_ACCOUNT_NAME")
    .unwrap_or_else(|_| "devstoreaccount1".to_string());

// Don't panic on missing env vars
let account = env::var("AZURE_STORAGE_ACCOUNT_NAME").unwrap();
```

---

## PS-6: Clippy Lints (MEDIUM)

DO:
```rust
#![deny(clippy::unwrap_used)]
#![warn(clippy::pedantic)]
#![allow(clippy::module_name_repetitions)]
```

```bash
cargo clippy -- -D warnings
```

DON'T:
```rust
// Don't suppress all clippy warnings
#![allow(clippy::all)]
```

---

## PS-7: rustfmt Configuration (LOW)

DO:
```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_small_heuristics = "Max"
```

```bash
cargo fmt --check
```

---

## PS-8: Cargo.lock Committed (HIGH)

Commit `Cargo.lock` for binary crates (samples). Do NOT commit for library crates.

DO:
```gitignore
# .gitignore
target/
.env
.env.*
!.env.sample
# Cargo.lock is COMMITTED (not in .gitignore) for binary samples
```

DON'T:
```gitignore
# Don't ignore Cargo.lock for binary crates
Cargo.lock
```

---

## PS-9: CVE Scanning (CRITICAL)

DO:
```bash
cargo install cargo-audit
cargo audit
cargo audit --deny warnings
```

DON'T:
```bash
# Don't ignore audit warnings
cargo audit
# found 3 vulnerabilities -- submitting anyway
```

---

## PS-10: Crate Legitimacy (MEDIUM)

Verify Azure SDK crates are from the official `azure-sdk` publisher on crates.io. Watch for typosquatting.

DON'T:
```toml
[dependencies]
azur_identity = "0.1.0"       # Typosquatting (azur, not azure)
azure-identity = "0.1.0"      # Wrong format (hyphens instead of underscores)
az_storage = "0.1.0"           # Not official package
```

---

## PS-11: Feature Flag Management (MEDIUM)

DO:
```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
azure_identity = { version = "0.x", features = ["enable_reqwest"] }
azure_storage_blob = { version = "0.x", default-features = false, features = ["enable_reqwest"] }
serde = { version = "1", features = ["derive"] }
```

DON'T:
```toml
[dependencies]
# Don't enable conflicting TLS backends
azure_identity = { version = "0.x", features = ["enable_reqwest", "enable_reqwest_rustls"] }
```
