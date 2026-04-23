# Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization. Includes service-specific best practices for Cosmos DB, Azure SQL (JDBC), and Storage.

## DB-1: Cosmos DB SDK (HIGH)
**Pattern:** Use `com.azure:azure-cosmos` with AAD credentials. Handle partitioned containers properly.

DO:
```java
import com.azure.cosmos.CosmosClient;
import com.azure.cosmos.CosmosClientBuilder;
import com.azure.cosmos.CosmosContainer;
import com.azure.cosmos.models.CosmosItemRequestOptions;
import com.azure.cosmos.models.CosmosQueryRequestOptions;
import com.azure.cosmos.models.PartitionKey;
import com.azure.cosmos.models.SqlParameter;
import com.azure.cosmos.models.SqlQuerySpec;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

CosmosClient client = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildClient();

CosmosContainer container = client
    .getDatabase("mydb")
    .getContainer("mycontainer");

// Query with partition key and parameterized query
SqlQuerySpec query = new SqlQuerySpec(
    "SELECT * FROM c WHERE c.category = @category",
    List.of(new SqlParameter("@category", "electronics"))
);
CosmosPagedIterable<JsonNode> items = container.queryItems(
    query,
    new CosmosQueryRequestOptions().setPartitionKey(new PartitionKey("electronics")),
    JsonNode.class
);

// Point read (most efficient)
container.readItem("item-id", new PartitionKey("electronics"), JsonNode.class);

// Create with partition key
container.createItem(Map.of(
    "id", "item-id",
    "category", "electronics",
    "name", "Laptop"
));
```

DON'T:
```java
// Don't use primary key in samples
CosmosClient client = new CosmosClientBuilder()
    .endpoint(endpoint)
    .key(primaryKey)  // Use credential(credential) with AAD
    .buildClient();

// Don't omit partition key (cross-partition queries are expensive)
container.queryItems("SELECT * FROM c", new CosmosQueryRequestOptions(), JsonNode.class);
```

---

## DB-2: Azure SQL with JDBC (HIGH)
**Pattern:** Use `mssql-jdbc` with AAD token authentication. Use connection pooling (HikariCP). For simplified passwordless auth, use `azure-identity-extensions` JDBC plugin.

DO:
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenRequestContext;
import com.microsoft.sqlserver.jdbc.SQLServerDataSource;

var credential = new DefaultAzureCredentialBuilder().build();

// Get AAD token for Azure SQL
String token = credential.getToken(
    new TokenRequestContext().addScopes("https://database.windows.net/.default")
).block().getToken();

// Connect with AAD token
SQLServerDataSource dataSource = new SQLServerDataSource();
dataSource.setServerName(config.sqlServer());
dataSource.setDatabaseName(config.sqlDatabase());
dataSource.setAccessToken(token);
dataSource.setEncrypt("true");

try (Connection conn = dataSource.getConnection()) {
    // Parameterized query
    try (PreparedStatement stmt = conn.prepareStatement(
            "SELECT * FROM [Products] WHERE [Category] = ?")) {
        stmt.setString(1, "Electronics");
        try (ResultSet rs = stmt.executeQuery()) {
            while (rs.next()) {
                System.out.println(rs.getString("name"));
            }
        }
    }
}
```

DO (Simplified passwordless auth with azure-identity-extensions):
```xml
<!-- pom.xml - JDBC plugin for passwordless Azure SQL auth -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity-extensions</artifactId>
    <version>1.2.0</version>
</dependency>
```

```java
// Connection string with ActiveDirectoryDefault authentication
// No manual token acquisition needed -- the JDBC plugin handles it
String url = "jdbc:sqlserver://myserver.database.windows.net:1433;"
    + "database=mydb;"
    + "encrypt=true;"
    + "authentication=ActiveDirectoryDefault;";

try (Connection conn = DriverManager.getConnection(url)) {
    // Token acquisition, caching, and refresh handled automatically
    try (PreparedStatement stmt = conn.prepareStatement("SELECT * FROM [Products]")) {
        // ...
    }
}
```

DON'T:
```java
// Don't use SQL authentication with passwords in samples
String url = "jdbc:sqlserver://server.database.windows.net;"
    + "database=mydb;user=admin;password=P@ssw0rd";  // Hardcoded credentials
Connection conn = DriverManager.getConnection(url);
```

---

## DB-3: SQL Parameter Safety -- PreparedStatement (MEDIUM)
**Pattern:** ALL SQL values must use `PreparedStatement` with parameter placeholders. Never concatenate user input into SQL strings.

DO:
```java
// Parameterized query with PreparedStatement
try (PreparedStatement stmt = conn.prepareStatement(
        "SELECT [id], [name] FROM [Products] WHERE [category] = ? AND [price] < ?")) {
    stmt.setString(1, category);
    stmt.setDouble(2, maxPrice);
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            System.out.printf("Product: %s ($%.2f)%n", rs.getString("name"), rs.getDouble("price"));
        }
    }
}
```

DON'T:
```java
// CRITICAL: SQL injection vulnerability
String query = "SELECT * FROM Products WHERE category = '" + userInput + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);  // SQL injection risk
```

**Why:** String concatenation in SQL enables injection attacks. `PreparedStatement` sanitizes all inputs automatically.

---

## DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

DO (SQL -- Batch Insert):
```java
// Batch insert with PreparedStatement
private static final int BATCH_SIZE = 100;

try (PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO [Products] ([id], [name], [category]) VALUES (?, ?, ?)")) {

    for (int i = 0; i < items.size(); i++) {
        Product item = items.get(i);
        stmt.setInt(1, item.id());
        stmt.setString(2, item.name());
        stmt.setString(3, item.category());
        stmt.addBatch();

        if ((i + 1) % BATCH_SIZE == 0) {
            stmt.executeBatch();  // Execute every BATCH_SIZE rows
        }
    }
    stmt.executeBatch();  // Execute remaining rows
}
// Why batch size 100: Balances network round trips vs memory usage.
```

DO (Cosmos DB -- Bulk):
```java
// Cosmos DB bulk operations
CosmosContainer container = cosmosClient.getDatabase("mydb").getContainer("mycontainer");

List<CosmosItemOperation> operations = items.stream()
    .map(item -> CosmosBulkOperations.getCreateItemOperation(
        item, new PartitionKey(item.getCategory())))
    .toList();

container.executeBulkOperations(operations);
// Why: Bulk executor optimizes throughput automatically (batching + parallelism).
```

DON'T:
```java
// Row-by-row INSERT (50 round trips for 50 items)
for (Product item : items) {
    try (PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO [Products] VALUES (?, ?, ?)")) {
        stmt.setInt(1, item.id());
        stmt.setString(2, item.name());
        stmt.setString(3, item.category());
        stmt.executeUpdate();  // One round trip per row
    }
}
```

---

## DB-5: Azure Storage -- Blob (MEDIUM)
**Pattern:** Use `com.azure:azure-storage-blob` with `DefaultAzureCredential`.

DO:
```java
import com.azure.storage.blob.BlobClient;
import com.azure.storage.blob.BlobContainerClient;
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();

BlobContainerClient containerClient = blobServiceClient.getBlobContainerClient("mycontainer");
containerClient.createIfNotExists();

// Upload
BlobClient blobClient = containerClient.getBlobClient("sample.txt");
blobClient.upload(BinaryData.fromString("Hello, Azure!"), true);

// Download
BinaryData content = blobClient.downloadContent();
System.out.println("Content: " + content.toString());

// List blobs
containerClient.listBlobs().forEach(blob ->
    System.out.println("Blob: " + blob.getName())
);
```

---

## DB-6: SAS Token Fallback (MEDIUM)
**Pattern:** For local development where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

DO:
```java
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

// Try AAD first, fall back to SAS for local dev
BlobServiceClient blobServiceClient;

String sasToken = System.getenv("AZURE_STORAGE_SAS_TOKEN");
if (sasToken != null && !sasToken.isBlank()) {
    blobServiceClient = new BlobServiceClientBuilder()
        .endpoint("https://" + accountName + ".blob.core.windows.net")
        .sasToken(sasToken)
        .buildClient();
    System.out.println("Using SAS token authentication (local dev)");
} else {
    var credential = new DefaultAzureCredentialBuilder().build();
    blobServiceClient = new BlobServiceClientBuilder()
        .endpoint("https://" + accountName + ".blob.core.windows.net")
        .credential(credential)
        .buildClient();
    System.out.println("Using DefaultAzureCredential (AAD)");
}
```
