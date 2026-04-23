# AI Services (OpenAI, Document Intelligence, Speech) & Vector Search

AI service client patterns, API versioning, embeddings, chat completions, and vector similarity search using the Azure SDK for Go.

## AI-1: Azure OpenAI Client--azopenai (HIGH)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/ai/azopenai` with `DefaultAzureCredential`.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/ai/azopenai"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azopenai.NewClient(
    config.AzureOpenAIEndpoint,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating openai client: %w", err)
}

// Chat completion
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: to.Ptr("gpt-4o"),
    Messages: []azopenai.ChatRequestMessageClassification{
        &azopenai.ChatRequestUserMessage{
            Content: azopenai.NewChatRequestUserMessageContent("Hello!"),
        },
    },
}, nil)
if err != nil {
    return fmt.Errorf("chat completion: %w", err)
}

fmt.Println(*resp.Choices[0].Message.Content)

// Embeddings
embResp, err := client.GetEmbeddings(ctx, azopenai.EmbeddingsOptions{
    DeploymentName: to.Ptr("text-embedding-3-small"),
    Input:          []string{"Sample text to embed"},
}, nil)
if err != nil {
    return fmt.Errorf("getting embeddings: %w", err)
}
```

**DON'T:**
```go
// Don't use API keys in samples (prefer AAD)
client, err := azopenai.NewClientWithKeyCredential(
    endpoint,
    azcore.NewKeyCredential(apiKey),  // Use DefaultAzureCredential
    nil,
)
```

---

## AI-2: API Version Documentation (LOW)

**Pattern:** When API versions are configurable, document the version and link to deprecation schedule.

**DO:**
```go
// The Azure OpenAI Go SDK manages API versions internally.
// For REST-based calls, specify the version explicitly:
// API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
const apiVersion = "2024-10-21"
```

---

## AI-3: Document Intelligence (MEDIUM)

**Pattern:** Use the Azure SDK for Go Document Intelligence package with `DefaultAzureCredential`.

> **Package Path:** The Go SDK package for Document Intelligence may be at
> `github.com/Azure/azure-sdk-for-go/sdk/ai/documentintelligence/azdocumentintelligence`.
> Verify the exact import path on pkg.go.dev before using--the package may be in preview.

**DO:**
```go
import (
    // Verify package path on pkg.go.dev -- may be in preview
    "github.com/Azure/azure-sdk-for-go/sdk/ai/documentintelligence/azdocumentintelligence"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azdocumentintelligence.NewClient(
    config.DocumentIntelligenceEndpoint,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating doc intelligence client: %w", err)
}

poller, err := client.BeginAnalyzeDocument(ctx, "prebuilt-invoice", documentContent, nil)
if err != nil {
    return fmt.Errorf("starting analysis: %w", err)
}

result, err := poller.PollUntilDone(ctx, nil)
if err != nil {
    return fmt.Errorf("analyzing document: %w", err)
}
```

---

## AI-4: Vector Dimension Validation (MEDIUM)

**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause runtime errors.

**DO:**
```go
const (
    embeddingModel  = "text-embedding-3-small"  // 1536 dimensions
    vectorDimension = 1536
)

// Validate embedding size after retrieval
embedding := embResp.Data[0].Embedding
if len(embedding) != vectorDimension {
    return fmt.Errorf(
        "embedding dimension mismatch: expected %d, got %d (model: %s)",
        vectorDimension, len(embedding), embeddingModel,
    )
}
```

**DON'T:**
```go
// Don't assume dimension without validation
embedding := embResp.Data[0].Embedding
insertEmbedding(embedding)  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

## AI-5: Azure Speech SDK (MEDIUM)

**Pattern:** The Azure Speech SDK for Go uses the Cognitive Services Speech SDK (CGo bindings) or REST APIs. For Go samples, the REST-based approach is more portable.

> **Note:** The Go Speech SDK requires CGo and platform-specific native libraries.
> For quickstarts, consider using the REST API with `azidentity` for authentication.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

// Get token for Speech Service via REST
cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://cognitiveservices.azure.com/.default"},
})
if err != nil {
    return fmt.Errorf("acquiring speech token: %w", err)
}

// Use token.Token with Speech REST API
```

---

## VEC-1: Vector Type Handling (MEDIUM)

**Pattern:** Serialize vectors as JSON strings for Azure SQL vector parameters.

**DO (Azure SQL):**
```go
import (
    "database/sql"
    "encoding/json"
)

embedding := []float32{0.1, 0.2, 0.3} // 1536 floats from OpenAI
embJSON, err := json.Marshal(embedding)
if err != nil {
    return fmt.Errorf("marshaling embedding: %w", err)
}

// Insert with CAST
_, err = db.ExecContext(ctx,
    "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(@p1 AS VECTOR(1536)))",
    sql.Named("p1", string(embJSON)),
)

// Vector distance query
rows, err := db.QueryContext(ctx, `
    SELECT TOP (@k)
        [id], [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(@searchEmbedding AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC`,
    sql.Named("k", 5),
    sql.Named("searchEmbedding", string(searchEmbJSON)),
)
```

---

## VEC-2: DiskANN Index (HIGH)

**Pattern:** DiskANN (Azure SQL) requires >=1000 rows. Check row count before creating index.

**DO:**
```go
var rowCount int
err := db.QueryRowContext(ctx,
    fmt.Sprintf("SELECT COUNT(*) FROM [%s]", tableName),
).Scan(&rowCount)
if err != nil {
    return fmt.Errorf("counting rows: %w", err)
}

if rowCount >= 1000 {
    fmt.Printf("%d rows available. Creating DiskANN index...\n", rowCount)
    _, err = db.ExecContext(ctx, fmt.Sprintf(
        "CREATE INDEX [ix_%s_embedding_diskann] ON [%s] ([embedding]) USING DiskANN",
        tableName, tableName,
    ))
    if err != nil {
        return fmt.Errorf("creating DiskANN index: %w", err)
    }
} else {
    fmt.Printf("Only %d rows. DiskANN requires >=1000. Using exact search.\n", rowCount)
}
```

**DON'T:**
```go
// Create DiskANN index without checking row count
_, err = db.ExecContext(ctx, "CREATE INDEX ... USING DiskANN")
// Fails with: "DiskANN index requires at least 1000 rows"
```
