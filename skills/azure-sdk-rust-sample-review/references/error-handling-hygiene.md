# Error Handling, Data Management & Sample Hygiene

Rules ERR-1 through ERR-4, DATA-1 through DATA-2, and HYG-1 through HYG-5.

## ERR-1: Result<T, E> and ? Operator (HIGH)

All functions that can fail must return `Result<T, E>`. Use `?` for error propagation. Never use `unwrap()` or `expect()` in main code paths.

DO:
```rust
use std::error::Error;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let config = load_config()?;
    let credential = Arc::new(DefaultAzureCredential::new()?);

    let blob_client = create_blob_client(&config, credential.clone())?;
    upload_file(&blob_client, "data.json").await?;

    println!("Upload complete");
    Ok(())
}

async fn upload_file(
    client: &BlobClient,
    filename: &str,
) -> Result<(), Box<dyn Error>> {
    let content = std::fs::read(filename)?;
    client.put_block_blob(content).await?;
    Ok(())
}
```

DON'T:
```rust
// NEVER unwrap in main code paths
#[tokio::main]
async fn main() {
    let config = load_config().unwrap();  // Panics on error
    let credential = DefaultAzureCredential::new().unwrap();  // Panics
    let data = std::fs::read_to_string("data.json").unwrap();  // Panics
}

// Don't swallow errors silently
async fn upload(client: &BlobClient, data: &[u8]) {
    let _ = client.put_block_blob(data.to_vec()).await;  // Error silently ignored
}
```

---

## ERR-2: Custom Error Types with thiserror (MEDIUM)

DO:
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

DON'T:
```rust
// Don't use String as error type (loses type information)
async fn run() -> Result<(), String> {
    let config = load_config().map_err(|e| e.to_string())?;
    Ok(())
}
```

---

## ERR-3: Azure Error Handling (HIGH)

Handle `azure_core::Error` with contextual messages and troubleshooting hints.

DO:
```rust
use azure_core::Error as AzureError;

async fn get_secret(
    client: &SecretClient,
    name: &str,
) -> Result<String, Box<dyn std::error::Error>> {
    match client.get(name).await {
        Ok(secret) => Ok(secret.value.clone()),
        Err(e) => {
            eprintln!("Failed to get secret '{}': {}", name, e);
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

DON'T:
```rust
// Don't provide generic error messages
match client.get("my-secret").await {
    Ok(secret) => println!("{}", secret.value),
    Err(e) => eprintln!("Error: {}", e),  // No context or troubleshooting
}
```

---

## ERR-4: Panic vs Result (CRITICAL)

NEVER use `panic!()`, `unwrap()`, or `expect()` in sample code paths. Reserve for test code only.

DO:
```rust
fn parse_config(value: &str) -> Result<u32, std::num::ParseIntError> {
    value.parse::<u32>()
}

// Use expect() ONLY in tests
#[cfg(test)]
mod tests {
    #[test]
    fn test_parse_config() {
        let result = parse_config("42").expect("should parse valid number");
        assert_eq!(result, 42);
    }
}

// If unwrap is truly safe, document why
let home_dir = dirs::home_dir()
    .ok_or("HOME directory not found -- required for config file location")?;
```

DON'T:
```rust
// CRITICAL: Don't panic in samples
fn main() {
    let port: u16 = env::var("PORT").unwrap().parse().unwrap();
    // Two potential panics -- user gets unhelpful message
}

// Don't use panic! for error handling
if items.is_empty() {
    panic!("No items found!");  // Use Result::Err instead
}
```

---

## DATA-1: Pre-Computed Data Files (HIGH)

Commit all required data files to repo. Use `include_str!` or `include_bytes!` for embedded data, or load from filesystem at runtime.

DO:
```
repo/
  data/
    products.json
    products-with-vectors.json
  src/
    main.rs
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

// Option 1: Embed data at compile time (small files)
const PRODUCTS_JSON: &str = include_str!("../data/products.json");

fn load_embedded_products() -> Result<Vec<Product>, serde_json::Error> {
    serde_json::from_str(PRODUCTS_JSON)
}

// Option 2: Load from filesystem at runtime (large files)
fn load_products_from_file(path: &str) -> Result<Vec<Product>, Box<dyn std::error::Error>> {
    let content = std::fs::read_to_string(path)?;
    let products: Vec<Product> = serde_json::from_str(&content)?;
    Ok(products)
}
```

> FALSE POSITIVE PREVENTION: Before flagging a data file as missing, check the FULL PR file list, trace file paths in code, and check for monorepo patterns.

---

## DATA-2: JSON Data Loading with serde (MEDIUM)

Use `serde` and `serde_json` for type-safe JSON handling. Always deserialize into typed structs.

DO:
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

DON'T:
```rust
// Don't use untyped serde_json::Value for known data structures
let data: serde_json::Value = serde_json::from_str(&content)?;
let name = data["name"].as_str().unwrap();  // Fragile, no compile-time checks
```

---

## HYG-1: .gitignore (CRITICAL)

DO:
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

> FALSE POSITIVE PREVENTION: Before flagging `.env` as committed, verify the file is actually tracked by git. Run `git ls-files .env` -- if empty, the file is NOT tracked.

---

## HYG-2: .env.sample (HIGH)

Provide `.env.sample` with placeholder values. Never commit actual `.env` files.

DO:
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
```

DON'T:
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321
```

---

## HYG-3: Dead Code (MEDIUM)

Remove unused files, functions, and imports. Commented-out code confuses users.

DO:
```rust
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
```

DON'T:
```rust
// Commented-out code confuses users
// use azure_cosmos::prelude::*;
//
// async fn old_implementation() -> Result<(), Box<dyn Error>> {
//     // This was the old way...
// }

use azure_storage_blob::prelude::*;
```

---

## HYG-4: LICENSE File (HIGH)

All Azure Samples repositories must include MIT LICENSE file.

> FALSE POSITIVE PREVENTION: Before flagging a missing LICENSE, check the REPO ROOT and parent directories. In monorepos, a single license at the repo root covers all subdirectories.

---

## HYG-5: Repository Governance Files (MEDIUM)

Samples in Azure Samples org should reference or include: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

DO:
```markdown
## Contributing
This project welcomes contributions and suggestions. See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## Security
Microsoft takes security seriously. See [SECURITY.md](SECURITY.md).
```
