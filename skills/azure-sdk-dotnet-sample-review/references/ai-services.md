# AI Services & Vector Search

Azure OpenAI, Document Intelligence, Speech, embeddings, and vector similarity search patterns.

## AI-1: Azure.AI.OpenAI Client Configuration (HIGH)

Use `Azure.AI.OpenAI` v2.x with `DefaultAzureCredential`. The v2.x SDK uses `System.ClientModel` (not `Azure.Core.Pipeline`) for retry configuration.

DO:
```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using OpenAI.Chat;
using OpenAI.Embeddings;
using System.ClientModel.Primitives;

var credential = new DefaultAzureCredential();

var client = new AzureOpenAIClient(
    new Uri(config["Azure:OpenAIEndpoint"]!),
    credential,
    new AzureOpenAIClientOptions
    {
        RetryPolicy = new ClientRetryPolicy(maxRetries: 3),
    });

// Chat completion (v2.x pattern)
ChatClient chatClient = client.GetChatClient("gpt-4o");
ChatCompletion completion = await chatClient.CompleteChatAsync(
    [
        new SystemChatMessage("You are a helpful assistant."),
        new UserChatMessage("Hello!")
    ]);
Console.WriteLine(completion.Content[0].Text);

// Embeddings
EmbeddingClient embeddingClient = client.GetEmbeddingClient("text-embedding-3-small");
OpenAIEmbedding embedding = await embeddingClient.GenerateEmbeddingAsync("Sample text to embed");
ReadOnlyMemory<float> vector = embedding.ToFloats();
```

DON'T:
```csharp
// Don't use API keys in samples (prefer AAD)
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    new Azure.AzureKeyCredential(apiKey));

// Don't use Azure.Core.Pipeline.RetryPolicy (v1.x pattern, won't compile in v2.x)
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    credential,
    new AzureOpenAIClientOptions
    {
        RetryPolicy = new Azure.Core.Pipeline.RetryPolicy(maxRetries: 3),  // Wrong type for v2.x
    });
```

> **Note:** `Azure.AI.OpenAI` v2.x is built on `System.ClientModel`, not `Azure.Core`. Retry is configured via `System.ClientModel.Primitives.ClientRetryPolicy`.

---

## AI-2: API Version Documentation (LOW)

Hardcoded API versions should include a comment linking to version docs.

DO:
```csharp
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    credential,
    new AzureOpenAIClientOptions(AzureOpenAIClientOptions.ServiceVersion.V2024_10_21));
    // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
```

---

## AI-3: Document Intelligence and Speech (MEDIUM)

Use `Azure.AI.DocumentIntelligence` and `Microsoft.CognitiveServices.Speech` with `DefaultAzureCredential` where supported.

DO:
```csharp
using Azure.AI.DocumentIntelligence;
using Azure.Identity;

var credential = new DefaultAzureCredential();
var docClient = new DocumentIntelligenceClient(
    new Uri(config["Azure:DocumentIntelligenceEndpoint"]!),
    credential);

var operation = await docClient.AnalyzeDocumentAsync(
    WaitUntil.Completed,
    "prebuilt-invoice",
    BinaryData.FromStream(documentStream));

AnalyzeResult result = operation.Value;

// Speech SDK
using Microsoft.CognitiveServices.Speech;

var speechConfig = SpeechConfig.FromSubscription(
    config["Azure:SpeechKey"]!,
    config["Azure:SpeechRegion"]!);
using var recognizer = new SpeechRecognizer(speechConfig);
var speechResult = await recognizer.RecognizeOnceAsync();
Console.WriteLine($"Recognized: {speechResult.Text}");
```

---

## AI-4: Vector Dimension Validation (MEDIUM)

Embeddings must match the declared vector column dimension.

DO:
```csharp
const string EmbeddingModel = "text-embedding-3-small";  // 1536 dimensions
const int VectorDimension = 1536;

ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
if (embedding.Length != VectorDimension)
{
    throw new InvalidOperationException(
        $"Embedding dimension mismatch: expected {VectorDimension}, got {embedding.Length}. " +
        $"Ensure model '{EmbeddingModel}' matches table schema.");
}
```

DON'T:
```csharp
// Don't assume dimension without validation
ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
await InsertEmbeddingAsync(embedding);  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

## VEC-1: Vector Type Handling (MEDIUM)

Serialize vectors for storage and use `VECTOR_DISTANCE` for search.

DO (Azure SQL):
```csharp
ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
string vectorJson = JsonSerializer.Serialize(embedding.ToArray());

await using var cmd = new SqlCommand(
    "INSERT INTO [Hotels] ([Embedding]) VALUES (CAST(@Embedding AS VECTOR(1536)))",
    connection);
cmd.Parameters.Add(new SqlParameter("@Embedding", SqlDbType.NVarChar) { Value = vectorJson });
await cmd.ExecuteNonQueryAsync();

var searchCmd = new SqlCommand(@"
    SELECT TOP (@K) [Id], [Name],
        VECTOR_DISTANCE('cosine', [Embedding], CAST(@SearchEmbedding AS VECTOR(1536))) AS Distance
    FROM [Hotels]
    ORDER BY Distance ASC", connection);
searchCmd.Parameters.Add(new SqlParameter("@K", SqlDbType.Int) { Value = 5 });
searchCmd.Parameters.Add(new SqlParameter("@SearchEmbedding", SqlDbType.NVarChar)
    { Value = JsonSerializer.Serialize(searchEmbedding.ToArray()) });
```

DO (Azure AI Search):
```csharp
using Azure.Search.Documents;
using Azure.Search.Documents.Models;

var searchClient = new SearchClient(
    new Uri(config["Azure:SearchEndpoint"]!),
    "hotels-index",
    credential);

var options = new SearchOptions
{
    VectorSearch = new()
    {
        Queries =
        {
            new VectorizedQuery(searchEmbedding)
            {
                KNearestNeighborsCount = 5,
                Fields = { "DescriptionVector" }
            }
        }
    }
};

SearchResults<Hotel> results = await searchClient.SearchAsync<Hotel>("luxury hotel", options);
await foreach (SearchResult<Hotel> result in results.GetResultsAsync())
{
    Console.WriteLine($"{result.Document.Name} (score: {result.Score})");
}
```

---

## VEC-2: DiskANN Index (HIGH)

DiskANN (Azure SQL) requires >= 1000 rows. Check row count before creating index.

DO:
```csharp
await using var countCmd = new SqlCommand(
    $"SELECT COUNT(*) FROM [{tableName}]", connection);
int rowCount = (int)(await countCmd.ExecuteScalarAsync())!;

if (rowCount >= 1000)
{
    Console.WriteLine($"{rowCount} rows available. Creating DiskANN index...");
    await using var indexCmd = new SqlCommand(
        $"CREATE INDEX [ix_{tableName}_embedding_diskann] ON [{tableName}] ([Embedding]) USING DiskANN",
        connection);
    await indexCmd.ExecuteNonQueryAsync();
}
else
{
    Console.WriteLine($"Only {rowCount} rows. DiskANN requires >= 1000. Using exact search.");
}
```

DON'T:
```csharp
// Create DiskANN index without checking row count
await new SqlCommand("CREATE INDEX ... USING DiskANN", connection).ExecuteNonQueryAsync();
// Fails with: "DiskANN index requires at least 1000 rows"
```
