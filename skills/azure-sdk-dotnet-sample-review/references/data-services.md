# Data Services (Cosmos DB, SQL, Storage, Tables)

Database and storage client patterns, connection management, transactions, batching, and query parameterization.

## DB-1: Cosmos DB SDK Patterns (HIGH)

Use `Microsoft.Azure.Cosmos` with AAD credentials. Handle partitioned containers properly.

DO:
```csharp
using Microsoft.Azure.Cosmos;
using Azure.Identity;

var credential = new DefaultAzureCredential();
var cosmosClient = new CosmosClient(
    config["Azure:CosmosEndpoint"],
    credential,
    new CosmosClientOptions
    {
        SerializerOptions = new CosmosSerializationOptions
        {
            PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
        }
    });

var database = cosmosClient.GetDatabase("mydb");
var container = database.GetContainer("mycontainer");

// Query with partition key
var query = new QueryDefinition("SELECT * FROM c WHERE c.category = @category")
    .WithParameter("@category", "electronics");

using FeedIterator<Product> feed = container.GetItemQueryIterator<Product>(query);
var results = new List<Product>();
while (feed.HasMoreResults)
{
    FeedResponse<Product> response = await feed.ReadNextAsync();
    results.AddRange(response);
}

// Point read (most efficient)
var response = await container.ReadItemAsync<Product>(
    "item-id", new PartitionKey("electronics"));
Product item = response.Resource;

// Upsert with partition key
await container.UpsertItemAsync(new Product
{
    Id = "item-id",
    Category = "electronics",
    Name = "Laptop"
}, new PartitionKey("electronics"));
```

DON'T:
```csharp
// Don't use primary key in samples
var cosmosClient = new CosmosClient(endpoint, primaryKey);

// Don't omit partition key in queries (cross-partition queries are expensive)
var query = new QueryDefinition("SELECT * FROM c");
```

---

## DB-2: Azure SQL with Microsoft.Data.SqlClient (HIGH)

Use `Microsoft.Data.SqlClient` with AAD authentication. Prefer `Authentication=Active Directory Default` connection string. Use parameterized queries.

DO:
```csharp
using Microsoft.Data.SqlClient;

// Preferred: Connection string with Active Directory Default (simplest)
await using var connection = new SqlConnection(
    $"Server={config["Azure:SqlServer"]};Database={config["Azure:SqlDatabase"]};" +
    "Authentication=Active Directory Default;Encrypt=True;");
await connection.OpenAsync();

// Parameterized query
await using var command = new SqlCommand(
    "SELECT [Id], [Name] FROM [Products] WHERE [Category] = @Category", connection);
command.Parameters.AddWithValue("@Category", "Electronics");

await using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine($"Product: {reader.GetString(1)}");
}
```

DON'T:
```csharp
// Don't use SQL authentication in samples
var connection = new SqlConnection(
    "Server=myserver;Database=mydb;User=sa;Password=P@ss;");

// Don't use string concatenation for queries
var query = $"SELECT * FROM Products WHERE Category = '{userInput}'";  // SQL injection!
```

**Why:** `Authentication=Active Directory Default` delegates credential resolution to `Microsoft.Data.SqlClient`, which supports managed identity, Azure CLI, and Visual Studio credentials automatically.

---

## DB-3: SQL Parameter Safety (MEDIUM)

ALL dynamic SQL identifiers (table names, column names) must use `[brackets]`. Values must use parameters.

DO:
```csharp
string tableName = config["TableName"]!;

var command = new SqlCommand(
    $"SELECT [Id], [Name] FROM [{tableName}] WHERE [Id] = @Id", connection);
command.Parameters.Add(new SqlParameter("@Id", SqlDbType.Int) { Value = productId });
```

DON'T:
```csharp
// Missing brackets on dynamic identifier + string concat for value
var query = $"SELECT Id, Name FROM {tableName} WHERE Id = {productId}";
```

---

## DB-4: Batch/Bulk Operations (HIGH)

Avoid row-by-row operations. Use batch operations for multiple rows.

DO (SQL -- Bulk Copy):
```csharp
await using var bulkCopy = new SqlBulkCopy(connection);
bulkCopy.DestinationTableName = "[Products]";
bulkCopy.ColumnMappings.Add("Id", "Id");
bulkCopy.ColumnMappings.Add("Name", "Name");
bulkCopy.ColumnMappings.Add("Category", "Category");
bulkCopy.BatchSize = 1000;

var dataTable = new DataTable();
dataTable.Columns.Add("Id", typeof(int));
dataTable.Columns.Add("Name", typeof(string));
dataTable.Columns.Add("Category", typeof(string));

foreach (var product in products)
{
    dataTable.Rows.Add(product.Id, product.Name, product.Category);
}

await bulkCopy.WriteToServerAsync(dataTable);
```

DO (Cosmos -- Transactional Batch):
```csharp
TransactionalBatch batch = container.CreateTransactionalBatch(
    new PartitionKey("electronics"));

batch.CreateItem(new Product { Id = "1", Category = "electronics", Name = "Laptop" });
batch.CreateItem(new Product { Id = "2", Category = "electronics", Name = "Mouse" });
batch.UpsertItem(new Product { Id = "3", Category = "electronics", Name = "Keyboard" });

using TransactionalBatchResponse response = await batch.ExecuteAsync();
if (!response.IsSuccessStatusCode)
{
    Console.WriteLine($"Batch failed: {response.StatusCode}");
}
```

DON'T:
```csharp
// Row-by-row INSERT (50 round trips for 50 rows)
foreach (var product in products)
{
    await using var cmd = new SqlCommand(
        "INSERT INTO [Products] VALUES (@Id, @Name, @Category)", connection);
    cmd.Parameters.AddWithValue("@Id", product.Id);
    cmd.Parameters.AddWithValue("@Name", product.Name);
    cmd.Parameters.AddWithValue("@Category", product.Category);
    await cmd.ExecuteNonQueryAsync();
}
```

---

## DB-5: Azure Storage (Azure.Storage.Blobs) Patterns (MEDIUM)

Use `Azure.Storage.Blobs`, `Azure.Storage.Files.Shares`, `Azure.Data.Tables` with `DefaultAzureCredential`.

DO:
```csharp
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// Blob Storage
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential);

var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
await containerClient.CreateIfNotExistsAsync();

var blobClient = containerClient.GetBlobClient("myblob.txt");
await blobClient.UploadAsync(BinaryData.FromString("Hello, Azure!"), overwrite: true);

BlobDownloadResult download = await blobClient.DownloadContentAsync();
string content = download.Content.ToString();

// Table Storage
var tableClient = new TableClient(
    new Uri($"https://{accountName}.table.core.windows.net"),
    "mytable",
    credential);

await tableClient.CreateIfNotExistsAsync();
await tableClient.AddEntityAsync(new TableEntity("partition1", "row1")
{
    { "Name", "Sample" }
});
```

---

## DB-6: SAS Token Fallback (MEDIUM)

For local development where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

DO:
```csharp
BlobServiceClient blobServiceClient;
string? sasToken = config["Azure:StorageSasToken"];

if (!string.IsNullOrEmpty(sasToken))
{
    blobServiceClient = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net{sasToken}"));
    Console.WriteLine("Using SAS token authentication (local dev)");
}
else
{
    var credential = new DefaultAzureCredential();
    blobServiceClient = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        credential);
    Console.WriteLine("Using DefaultAzureCredential (AAD)");
}
```
