# Azure SDK Client Patterns

Authentication, credential management, client construction, retry policies, managed identity patterns, pagination, and resource cleanup.

## AZ-1: Client Construction with DefaultAzureCredential (HIGH)

Use `DefaultAzureCredential` for samples. Cache credential instances.

DO:
```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Security.KeyVault.Secrets;
using Azure.Messaging.ServiceBus;

var credential = new DefaultAzureCredential();

// Storage Blob
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential);

// Key Vault
var secretClient = new SecretClient(
    new Uri(config["Azure:KeyVaultUrl"]!),
    credential);

// Service Bus
var serviceBusClient = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net",
    credential);

// Cosmos DB
var cosmosClient = new CosmosClient(
    config["Azure:CosmosEndpoint"],
    credential,
    new CosmosClientOptions { SerializerOptions = new() { PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase } });
```

DON'T:
```csharp
// Don't use connection strings in samples
var blobServiceClient = new BlobServiceClient(connectionString);

// Don't use account keys
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    new StorageSharedKeyCredential(accountName, accountKey));

// Don't recreate credential for each client
var blobClient = new BlobServiceClient(uri, new DefaultAzureCredential());
var secretClient = new SecretClient(uri, new DefaultAzureCredential());
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code, etc.) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

## AZ-2: Client Options and Retry Policies (MEDIUM)

Configure retry policies, timeouts, and diagnostics for production-ready samples.

DO:
```csharp
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential,
    new BlobClientOptions
    {
        Retry =
        {
            MaxRetries = 3,
            Delay = TimeSpan.FromSeconds(1),
            MaxDelay = TimeSpan.FromSeconds(30),
            Mode = RetryMode.Exponential
        },
        Diagnostics =
        {
            IsLoggingContentEnabled = false,
            IsDistributedTracingEnabled = true
        }
    });
```

---

## AZ-3: Managed Identity Patterns (HIGH)

For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

DO:
```csharp
// For samples: DefaultAzureCredential (works locally + cloud)
var credential = new DefaultAzureCredential();

// For production: Explicitly use managed identity when deployed
var credential = new ManagedIdentityCredential();

// User-assigned (when multiple identities needed)
var credential = new ManagedIdentityCredential(
    clientId: Environment.GetEnvironmentVariable("AZURE_CLIENT_ID"));
```

DON'T:
```csharp
// Don't hardcode service principal credentials in samples
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

## AZ-4: Token Management for Non-SDK HTTP (CRITICAL)

For services requiring raw HTTP calls, get tokens with `GetTokenAsync()`. Tokens expire after ~1 hour—implement refresh logic for long-running samples.

DO:
```csharp
var credential = new DefaultAzureCredential();

async Task<AccessToken> GetSqlTokenAsync(TokenCredential credential)
{
    return await credential.GetTokenAsync(
        new TokenRequestContext(new[] { "https://database.windows.net/.default" }),
        CancellationToken.None);
}

AccessToken token = await GetSqlTokenAsync(credential);

bool IsTokenExpiringSoon(AccessToken token)
{
    return token.ExpiresOn < DateTimeOffset.UtcNow.AddMinutes(5);
}

if (IsTokenExpiringSoon(token))
{
    Console.WriteLine("Token expiring soon, refreshing...");
    token = await GetSqlTokenAsync(credential);
}
```

DON'T:
```csharp
// CRITICAL: Don't acquire token once and use for hours
var token = await credential.GetTokenAsync(
    new TokenRequestContext(new[] { "https://database.windows.net/.default" }),
    CancellationToken.None);
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

---

## AZ-5: DefaultAzureCredential Configuration (MEDIUM)

Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI.

DO:
```csharp
// For CI/CD environments (no interactive prompts)
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeInteractiveBrowserCredential = true,
    ExcludeWorkloadIdentityCredential = false,
    ExcludeManagedIdentityCredential = false,
});

// For local development (include Visual Studio / VS Code)
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeVisualStudioCredential = false,
    ExcludeVisualStudioCodeCredential = false,
    ExcludeAzureCliCredential = false,
});
```

---

## AZ-6: Resource Cleanup — IAsyncDisposable (MEDIUM)

Samples must properly dispose clients. Use `await using` declarations or `IAsyncDisposable`.

DO:
```csharp
// Pattern 1: await using declaration (preferred)
await using var client = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net", credential);
await using var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));

// Pattern 2: try/finally for complex scenarios
ServiceBusClient? busClient = null;
try
{
    busClient = new ServiceBusClient(
        $"{namespaceName}.servicebus.windows.net", credential);
    var receiver = busClient.CreateReceiver("myqueue");
    var messages = await receiver.ReceiveMessagesAsync(maxMessages: 10);
    foreach (var message in messages)
    {
        Console.WriteLine($"Received: {message.Body}");
        await receiver.CompleteMessageAsync(message);
    }
}
finally
{
    if (busClient is not null)
        await busClient.DisposeAsync();
}
```

DON'T:
```csharp
var client = new ServiceBusClient(namespace, credential);
var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));
// Client never disposed (resource leak)
```

---

## AZ-7: Pagination with AsyncPageable<T> (HIGH)

Use `await foreach` with `AsyncPageable<T>` for paginated Azure SDK responses. Samples that only process the first page silently lose data.

DO:
```csharp
// Blob Storage — iterate all pages
await foreach (BlobItem blob in containerClient.GetBlobsAsync())
{
    Console.WriteLine($"Blob: {blob.Name}");
}

// Key Vault — iterate all secrets
await foreach (SecretProperties secret in secretClient.GetPropertiesOfSecretsAsync())
{
    Console.WriteLine($"Secret: {secret.Name}");
}

// Cosmos DB — iterate query results with FeedIterator
using FeedIterator<Product> feed = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.category = 'electronics'");

while (feed.HasMoreResults)
{
    FeedResponse<Product> response = await feed.ReadNextAsync();
    foreach (Product item in response)
    {
        Console.WriteLine($"Product: {item.Name}");
    }
}
```

DON'T:
```csharp
// Only gets first page
var blobs = new List<BlobItem>();
await foreach (BlobItem blob in containerClient.GetBlobsAsync())
{
    blobs.Add(blob);
    if (blobs.Count >= 10) break;  // Stops after 10, may miss thousands
}
```

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.
