# Data Services (Cosmos DB, SQL, Storage, Tables)

Database and storage client patterns, connection management, transactions, batching, and query parameterization.

## DB-1: Cosmos DB--azcosmos Patterns (HIGH)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos` with AAD credentials. Handle partitioned containers properly.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azcosmos.NewClient(config.CosmosEndpoint, cred, nil)
if err != nil {
    return fmt.Errorf("creating cosmos client: %w", err)
}

containerClient, err := client.NewContainer("mydb", "mycontainer")
if err != nil {
    return fmt.Errorf("getting container: %w", err)
}

// Query with partition key
pk := azcosmos.NewPartitionKeyString("electronics")
queryPager := containerClient.NewQueryItemsPager(
    "SELECT * FROM c WHERE c.category = @category",
    pk,
    &azcosmos.QueryOptions{
        QueryParameters: []azcosmos.QueryParameter{
            {Name: "@category", Value: "electronics"},
        },
    },
)

// Point read (most efficient)
pk = azcosmos.NewPartitionKeyString("partition-key-value")
resp, err := containerClient.ReadItem(ctx, pk, "item-id", nil)
if err != nil {
    return fmt.Errorf("reading item: %w", err)
}

// Create item
itemJSON := []byte(`{"id":"item-1","category":"electronics","name":"Laptop"}`)
_, err = containerClient.CreateItem(ctx, pk, itemJSON, nil)
if err != nil {
    return fmt.Errorf("creating item: %w", err)
}
```

**DON'T:**
```go
// Don't use primary key in samples
client, err := azcosmos.NewClientWithKey(endpoint, accountKey, nil)

// Don't omit partition key in queries (cross-partition queries are expensive)
queryPager := containerClient.NewQueryItemsPager(
    "SELECT * FROM c",
    azcosmos.PartitionKey{},  // Empty partition key
    nil,
)
```

---

## DB-2: Azure SQL with go-mssqldb (HIGH)

**Pattern:** Use `github.com/microsoft/go-mssqldb` with AAD token authentication. Use `database/sql` standard library interface.

**DO:**
```go
import (
    "database/sql"
    "fmt"

    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    mssql "github.com/microsoft/go-mssqldb"
    "github.com/microsoft/go-mssqldb/azuread"
)

// Connect with AAD authentication
func connectSQL(ctx context.Context) (*sql.DB, error) {
    // Option 1: Use azuread driver (recommended)
    connStr := fmt.Sprintf(
        "sqlserver://%s?database=%s&fedauth=ActiveDirectoryDefault",
        config.SQLServer, config.SQLDatabase,
    )
    db, err := sql.Open(azuread.DriverName, connStr)
    if err != nil {
        return nil, fmt.Errorf("opening SQL connection: %w", err)
    }

    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("pinging SQL server: %w", err)
    }

    return db, nil
}

// Use the connection
func queryProducts(ctx context.Context, db *sql.DB, category string) ([]Product, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT [id], [name], [category] FROM [Products] WHERE [category] = @p1",
        sql.Named("p1", category),
    )
    if err != nil {
        return nil, fmt.Errorf("querying products: %w", err)
    }
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Category); err != nil {
            return nil, fmt.Errorf("scanning row: %w", err)
        }
        products = append(products, p)
    }
    return products, rows.Err()
}
```

**DON'T:**
```go
// Don't use SQL authentication with password in samples
connStr := fmt.Sprintf("sqlserver://user:password@%s?database=%s", server, database)

// Don't ignore rows.Err()
for rows.Next() {
    rows.Scan(&p.ID, &p.Name)
}
// Missing rows.Err() check--may have encountered error during iteration
```

---

## DB-3: SQL Parameter Safety (HIGH)

**Pattern:** ALWAYS use parameterized queries. Go's `database/sql` supports named parameters via `sql.Named()`. Never build SQL by string concatenation.

**DO:**
```go
// Parameterized query--safe from SQL injection
rows, err := db.QueryContext(ctx,
    "SELECT [id], [name] FROM [Products] WHERE [category] = @category AND [price] < @maxPrice",
    sql.Named("category", category),
    sql.Named("maxPrice", maxPrice),
)

// Dynamic identifiers--use bracket quoting (validated against allowlist)
validTables := map[string]bool{"Products": true, "Orders": true}
if !validTables[tableName] {
    return fmt.Errorf("invalid table name: %s", tableName)
}
query := fmt.Sprintf("SELECT [id], [name] FROM [%s] WHERE [id] = @id", tableName)
rows, err := db.QueryContext(ctx, query, sql.Named("id", itemID))
```

**DON'T:**
```go
// CRITICAL: SQL injection vulnerability
query := fmt.Sprintf("SELECT * FROM Products WHERE category = '%s'", category)
rows, err := db.QueryContext(ctx, query)

// Don't use Sprintf for values
query := fmt.Sprintf("INSERT INTO [Products] VALUES ('%s', '%s')", name, category)
```

---

## DB-4: Batch Operations (HIGH)

**Pattern:** Avoid row-by-row operations. Use batch operations or transactions for multiple rows.

**DO:**
```go
// SQL batch insert with transaction
func batchInsertProducts(ctx context.Context, db *sql.DB, products []Product) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }
    defer tx.Rollback()

    stmt, err := tx.PrepareContext(ctx,
        "INSERT INTO [Products] ([id], [name], [category]) VALUES (@p1, @p2, @p3)",
    )
    if err != nil {
        return fmt.Errorf("preparing statement: %w", err)
    }
    defer stmt.Close()

    for _, p := range products {
        if _, err := stmt.ExecContext(ctx, sql.Named("p1", p.ID), sql.Named("p2", p.Name), sql.Named("p3", p.Category)); err != nil {
            return fmt.Errorf("inserting product %s: %w", p.ID, err)
        }
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }
    return nil
}
```

**DO (Cosmos DB--transactional batch):**
```go
// Cosmos transactional batch (same partition key, max 100 ops)
pk := azcosmos.NewPartitionKeyString("electronics")

batch := containerClient.NewTransactionalBatch(pk)
batch.CreateItem([]byte(`{"id":"1","category":"electronics","name":"Laptop"}`), nil)
batch.CreateItem([]byte(`{"id":"2","category":"electronics","name":"Mouse"}`), nil)
batch.UpsertItem([]byte(`{"id":"3","category":"electronics","name":"Keyboard"}`), nil)

batchResp, err := containerClient.ExecuteTransactionalBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("executing batch: %w", err)
}
// Check batchResp.Success for batch-level success
```

**DON'T:**
```go
// Row-by-row INSERT (50 round trips for 50 products)
for _, p := range products {
    _, err := db.ExecContext(ctx,
        "INSERT INTO [Products] VALUES (@p1, @p2, @p3)",
        sql.Named("p1", p.ID), sql.Named("p2", p.Name), sql.Named("p3", p.Category),
    )
    if err != nil {
        return err
    }
}
```

---

## DB-5: Azure Storage--azblob Patterns (MEDIUM)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/storage/azblob` with `DefaultAzureCredential`.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// Create blob service client
serviceClient, err := azblob.NewClient(
    fmt.Sprintf("https://%s.blob.core.windows.net/", accountName),
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating blob client: %w", err)
}

// Upload blob
_, err = serviceClient.UploadBuffer(ctx, "mycontainer", "myblob.txt",
    []byte("Hello, Azure!"), nil)
if err != nil {
    return fmt.Errorf("uploading blob: %w", err)
}

// Download blob
downloadResp, err := serviceClient.DownloadStream(ctx, "mycontainer", "myblob.txt", nil)
if err != nil {
    return fmt.Errorf("downloading blob: %w", err)
}
defer downloadResp.Body.Close()

data, err := io.ReadAll(downloadResp.Body)
if err != nil {
    return fmt.Errorf("reading blob: %w", err)
}
fmt.Printf("Downloaded: %s\n", string(data))
```

---

## DB-6: SAS Token Fallback (MEDIUM)

**Pattern:** For local development or CI environments where `DefaultAzureCredential` isn't available, provide SAS token fallback with documentation.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

func createBlobClient(accountName string) (*azblob.Client, error) {
    serviceURL := fmt.Sprintf("https://%s.blob.core.windows.net/", accountName)

    if sasToken := os.Getenv("AZURE_STORAGE_SAS_TOKEN"); sasToken != "" {
        // Local dev: SAS token
        fmt.Println("Using SAS token authentication (local dev)")
        return azblob.NewClientWithNoCredential(serviceURL+sasToken, nil)
    }

    // Production: AAD
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return nil, fmt.Errorf("creating credential: %w", err)
    }
    fmt.Println("Using DefaultAzureCredential (AAD)")
    return azblob.NewClient(serviceURL, cred, nil)
}
```

---

## DB-7: UploadStream for Large Files (HIGH)

**Pattern:** Use `UploadStream()` for streaming large files to blob storage. `UploadBuffer()` loads the entire file into memory; `UploadStream()` streams in chunks.

**DO:**
```go
import (
    "os"

    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob/blockblob"
)

// Stream large files without loading entirely into memory
func uploadLargeFile(ctx context.Context, client *azblob.Client, container, blobName, filePath string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("opening file: %w", err)
    }
    defer file.Close()

    _, err = client.UploadStream(ctx, container, blobName, file,
        &azblob.UploadStreamOptions{
            BlockSize:   4 * 1024 * 1024, // 4 MiB blocks
            Concurrency: 4,               // parallel uploads
        },
    )
    if err != nil {
        return fmt.Errorf("uploading stream: %w", err)
    }
    return nil
}
```

**DON'T:**
```go
// Don't load large files entirely into memory
data, _ := os.ReadFile("large-file.dat")  // May OOM for multi-GB files
client.UploadBuffer(ctx, "container", "blob", data, nil)
```
