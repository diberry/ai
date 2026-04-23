# Azure SDK Client Patterns

Rules AZ-1 through AZ-7. Authentication, credential management, client construction, retry policies, managed identity, and pagination.

## AZ-1: Client Construction with DefaultAzureCredential (HIGH)

Use `DefaultAzureCredential` for samples. Share credential via `Arc`.

DO:
```rust
use azure_identity::DefaultAzureCredential;
use azure_storage_blob::prelude::*;
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let credential = Arc::new(DefaultAzureCredential::new()?);

    let blob_service_client = BlobServiceClient::new(
        &format!("https://{}.blob.core.windows.net", account_name),
        credential.clone(),
        None,
    );

    let keyvault_client = azure_security_keyvault_secrets::SecretClient::new(
        &config.keyvault_url,
        credential.clone(),
        None,
    )?;

    Ok(())
}
```

DON'T:
```rust
// Don't use connection strings in samples (prefer AAD auth)
let blob_service_client = BlobServiceClient::from_connection_string(&connection_string)?;

// Don't recreate credential for each client
let client1 = BlobServiceClient::new(url, Arc::new(DefaultAzureCredential::new()?));
let client2 = SecretClient::new(url, Arc::new(DefaultAzureCredential::new()?));
// Two separate credential instances -- wasteful
```

---

## AZ-2: Client Options (MEDIUM)

Configure retry policies, timeouts, and transport options. The Rust SDK uses struct initialization, not builder pattern.

DO:
```rust
use azure_core::ClientOptions;

let options = ClientOptions {
    ..Default::default()
};

let blob_service_client = BlobServiceClient::new(
    &format!("https://{}.blob.core.windows.net", account_name),
    credential.clone(),
    Some(options),
);
```

> Preview Note: Client options API may vary across Azure SDK Rust crate versions.

---

## AZ-3: Managed Identity (HIGH)

DO:
```rust
use azure_identity::{DefaultAzureCredential, ManagedIdentityCredential};
use std::sync::Arc;

// For samples: DefaultAzureCredential (works locally + cloud)
let credential = Arc::new(DefaultAzureCredential::new()?);

// For production: Explicitly use managed identity when deployed
let credential = Arc::new(ManagedIdentityCredential::default());

// User-assigned (when multiple identities needed)
let client_id = std::env::var("AZURE_CLIENT_ID")?;
let credential = Arc::new(
    ManagedIdentityCredential::new(Some(&client_id))?
);
```

DON'T:
```rust
// Don't hardcode service principal credentials in samples
let credential = ClientSecretCredential::new(tenant_id, client_id, client_secret);
```

---

## AZ-4: Token Management -- TokenCredential Trait (CRITICAL)

For services without official Rust SDK crate, get tokens via `TokenCredential::get_token()`. Tokens expire after ~1 hour -- implement refresh logic for long-running samples.

DO:
```rust
use azure_core::auth::TokenCredential;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

let token_response = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;

let token = &token_response.token;

async fn get_fresh_token(
    credential: &dyn TokenCredential,
    scope: &str,
) -> Result<String, azure_core::Error> {
    let response = credential.get_token(&[scope]).await?;
    Ok(response.token.clone())
}

let token = get_fresh_token(
    credential.as_ref(),
    "https://database.windows.net/.default"
).await?;
```

DON'T:
```rust
// CRITICAL: Don't acquire token once and use for hours
let token = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

---

## AZ-5: Credential Configuration (MEDIUM)

DO:
```rust
use azure_identity::DefaultAzureCredential;

let credential = DefaultAzureCredential::new()?;

// Document in README:
// > Authentication: This sample uses DefaultAzureCredential, which tries
// > multiple credential sources in order. Check current azure_identity docs.
```

---

## AZ-6: Resource Cleanup -- Drop Trait, RAII (MEDIUM)

Rust's RAII handles cleanup via the `Drop` trait. Ensure clients go out of scope properly.

DO:
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
    blob_client.put_block_blob(data).await?;
    Ok(())
}

// Explicit scope for cleanup
async fn process_messages() -> Result<(), Box<dyn std::error::Error>> {
    {
        let client = create_service_bus_client()?;
        // ... process messages ...
        // client dropped here, connection closed
    }
    println!("All resources cleaned up");
    Ok(())
}
```

DON'T:
```rust
// Don't leak resources by storing in a global/static without cleanup
static mut CLIENT: Option<BlobServiceClient> = None;

// Don't use std::mem::forget to skip Drop
let client = create_client()?;
std::mem::forget(client);  // Resource leak
```

---

## AZ-7: Pagination with Pageable/Stream (HIGH)

Use async streams (`Pageable` / `Stream`) for paginated Azure SDK responses. Samples that only process the first page silently lose data.

DO:
```rust
use azure_storage_blob::prelude::*;
use futures::StreamExt;

let container_client = blob_service_client.container_client("mycontainer");
let mut blob_stream = container_client.list_blobs().into_stream();

while let Some(page) = blob_stream.next().await {
    let page = page?;
    for blob in page.blobs.blobs() {
        println!("Blob: {}", blob.name);
    }
}

// Collect all items (use with caution for large datasets)
let all_blobs: Vec<_> = container_client
    .list_blobs()
    .into_stream()
    .try_collect()
    .await?;
```

DON'T:
```rust
// Only gets first page
let response = container_client.list_blobs().into_future().await?;
for blob in response.blobs.blobs() {
    println!("Blob: {}", blob.name);
}
// Remaining pages silently lost
```
