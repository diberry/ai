# AI Services & Vector Search

Rules AI-1 through AI-4 and VEC-1 through VEC-2. Azure OpenAI, Document Intelligence, embeddings, vector search in Azure SQL and Cosmos DB.

## AI-1: Azure OpenAI Patterns (HIGH)

Use `azure_openai` crate if available, or `async-openai` crate with Azure configuration. Always use AAD authentication where possible.

DO:
```rust
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

DON'T:
```rust
// Don't hardcode API keys
let config = AzureConfig::new()
    .with_api_key("sk-abc123...");

// Don't omit API version
let config = AzureConfig::new()
    .with_api_base(endpoint);
    // Missing api_version -- may use incompatible default
```

> Preview Note: The official `azure_openai` Rust crate may not be available yet. The `async-openai` crate is community-maintained.

---

## AI-2: Embedding Dimension Validation (MEDIUM)

Always validate embedding vector dimensions match the target index/column configuration before insertion.

DO:
```rust
const EMBEDDING_MODEL: &str = "text-embedding-3-small";
const VECTOR_DIMENSION: usize = 1536;

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

let embedding = get_embedding(&text, EMBEDDING_MODEL).await?;
validate_embedding(&embedding, VECTOR_DIMENSION, EMBEDDING_MODEL)?;
insert_embedding(&embedding).await?;
```

DON'T:
```rust
// Don't assume dimension without validation
let embedding = get_embedding(&text).await?;
insert_embedding(&embedding).await?;  // May fail silently if dimension wrong
```

---

## AI-3: Document Intelligence Patterns (MEDIUM)

No official Rust crate exists yet -- use REST API via `reqwest` with AAD tokens from `azure_identity`.

DO:
```rust
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

// Poll for result using Operation-Location header
let operation_url = response
    .headers()
    .get("Operation-Location")
    .ok_or("Missing Operation-Location header")?
    .to_str()?;
```

DON'T:
```rust
// Don't hardcode API keys in REST calls
let response = http_client
    .post(&url)
    .header("Ocp-Apim-Subscription-Key", "abc123...")  // Use AAD tokens
    .send()
    .await?;
```

---

## AI-4: API Version Documentation (MEDIUM)

When using Azure AI services via REST API, always document the API version used.

DO:
```rust
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

DON'T:
```rust
// Don't hardcode API version without documentation
let url = format!("{}/openai/deployments/{}/chat/completions?api-version=2023-05-15", endpoint, deployment);
// No note about which version, why, or where to find updates
```

---

## VEC-1: Vector Type Handling (MEDIUM)

Serialize vectors as JSON strings for Azure SQL VECTOR type. Validate dimensions before insertion.

DO:
```rust
use serde_json;

let embedding: Vec<f32> = get_embedding(&text).await?;
let embedding_json = serde_json::to_string(&embedding)?;

// Azure SQL -- CAST as VECTOR
let sql = format!(
    "INSERT INTO [{}] ([id], [embedding]) VALUES (@P1, CAST(@P2 AS VECTOR(1536)))",
    table_name
);
client.execute(&sql, &[&item_id, &embedding_json]).await?;

// Vector distance query
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

## VEC-2: DiskANN Index Configuration (MEDIUM)

Use DiskANN indexes for large-scale vector search in Azure SQL.

DO:
```rust
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

DON'T:
```rust
// Don't use mismatched distance metrics between index and query
// Index uses 'cosine' but query uses 'dot'
let sql = "
    SELECT TOP 10 [id],
        VECTOR_DISTANCE('dot', [embedding], CAST(@P1 AS VECTOR(1536))) AS distance
    FROM [dbo].[documents]  -- Index built with DISTANCE_METRIC = 'cosine'
    ORDER BY distance ASC
";
```
