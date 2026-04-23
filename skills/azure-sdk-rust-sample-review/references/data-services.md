# Data Services (Cosmos DB, SQL, Storage)

Rules DB-1 through DB-6. Database and storage client patterns, connection management, transactions, batching, and query parameterization.

## DB-1: Cosmos DB Patterns (HIGH)

Use `azure_cosmos` crate with AAD credentials. Handle partitioned containers properly.

DO:
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
    None,
);

let database = client.database_client("mydb");
let container = database.container_client("products");

// Query with partition key
let query = "SELECT * FROM c WHERE c.category = @category";
let mut results = container
    .query_documents::<Product>(query)
    .partition_key(&PartitionKey::from("electronics"))
    .into_stream();

while let Some(page) = results.next().await {
    let page = page?;
    for doc in page.documents() {
        println!("Product: {} -- ${:.2}", doc.name, doc.price);
    }
}

// Point read (most efficient)
let response = container
    .document_client("item-id", &PartitionKey::from("electronics"))?
    .get_document::<Product>()
    .await?;
```

DON'T:
```rust
// Don't use primary key in samples
let client = CosmosClient::with_key(&endpoint, &primary_key);

// Don't omit partition key (cross-partition queries are expensive)
let results = container
    .query_documents::<Product>("SELECT * FROM c")
    .into_stream();
```

> Preview Note: The `azure_cosmos` crate API may change. Check latest version on crates.io.

---

## DB-2: Azure SQL with tiberius (HIGH)

Use the `tiberius` crate for Azure SQL connections with AAD token authentication.

DO:
```rust
use tiberius::{Client, Config, AuthMethod, ColumnData};
use tokio::net::TcpStream;
use tokio_util::compat::TokioAsyncWriteCompatExt;
use azure_identity::DefaultAzureCredential;
use azure_core::auth::TokenCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

let token = credential
    .get_token(&["https://database.windows.net/.default"])
    .await?;

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

// Parameterized query
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

DON'T:
```rust
// Don't use SQL Server authentication in samples
config.authentication(AuthMethod::sql_server("sa", "Password123!"));

// Don't disable encryption
config.encryption(tiberius::EncryptionLevel::Off);
```

---

## DB-3: SQL Parameter Safety (MEDIUM)

ALL dynamic SQL identifiers (table names, column names) must use `[brackets]`. Values MUST use parameterized queries.

DO:
```rust
let table_name = &config.table_name;

// Always bracket-quote identifiers
let create_table = format!(
    "CREATE TABLE [{}] (
        [id] INT PRIMARY KEY,
        [name] NVARCHAR(100),
        [category] NVARCHAR(50)
    )",
    table_name
);

// Parameterized values
let query = format!(
    "SELECT [id], [name] FROM [{}] WHERE [id] = @P1",
    table_name
);
let results = client.query(&query, &[&item_id]).await?;
```

DON'T:
```rust
// Missing brackets on dynamic identifier
let query = format!("SELECT id, name FROM {} WHERE id = @P1", table_name);

// NEVER concatenate user values into SQL
let query = format!("SELECT * FROM [Products] WHERE [name] = '{}'", user_input);
```

---

## DB-4: Batch Operations (HIGH)

Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

DO:
```rust
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

DON'T:
```rust
// Row-by-row INSERT (50 round trips for 50 rows)
for item in &items {
    client.execute(
        "INSERT INTO [Products] VALUES (@P1, @P2, @P3)",
        &[&item.id, &item.name as &dyn tiberius::ToSql, &item.category as &dyn tiberius::ToSql],
    ).await?;
}
```

---

## DB-5: Azure Storage (MEDIUM)

Use `azure_storage_blob` crate with `DefaultAzureCredential`.

DO:
```rust
use azure_storage_blob::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;
use futures::StreamExt;

let credential = Arc::new(DefaultAzureCredential::new()?);

let blob_service_client = BlobServiceClient::new(
    &format!("https://{}.blob.core.windows.net", account_name),
    credential.clone(),
);

let container_client = blob_service_client.container_client("mycontainer");
container_client.create().await?;

let blob_client = container_client.blob_client("myblob.txt");

// Upload
blob_client
    .put_block_blob(b"Hello, Azure!".to_vec())
    .await?;

// Download
let response = blob_client.get_content().await?;
let content = String::from_utf8(response)?;
println!("Downloaded: {}", content);

// List blobs with pagination
let mut stream = container_client.list_blobs().into_stream();
while let Some(page) = stream.next().await {
    let page = page?;
    for blob in page.blobs.blobs() {
        println!("Blob: {}", blob.name);
    }
}
```

---

## DB-6: Storage SAS Token Fallback (MEDIUM)

When `DefaultAzureCredential` is not available, use SAS tokens as a fallback. Never hardcode SAS tokens.

DO:
```rust
use azure_storage_blob::prelude::*;

let blob_service_client = if let Ok(sas_token) = std::env::var("AZURE_STORAGE_SAS_TOKEN") {
    BlobServiceClient::with_sas_token(
        &format!("https://{}.blob.core.windows.net", account_name),
        &sas_token,
    )?
} else {
    let credential = Arc::new(DefaultAzureCredential::new()?);
    BlobServiceClient::new(
        &format!("https://{}.blob.core.windows.net", account_name),
        credential,
        None,
    )
};
```

DON'T:
```rust
// Don't hardcode SAS tokens
let client = BlobServiceClient::with_sas_token(
    &url,
    "sv=2022-11-02&ss=b&srt=sco&sp=rwdlacup&se=2025-12-31..."  // Hardcoded SAS
)?;
```
