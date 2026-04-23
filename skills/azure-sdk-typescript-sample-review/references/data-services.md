# Data Services (DB-1 through DB-6)

Rules for Cosmos DB, Azure SQL (tedious), Storage, and Tables.

## DB-1: Cosmos DB Client Patterns (HIGH)

**Pattern:** Use `@azure/cosmos` with `aadCredentials`. Handle partitioned containers. Use specific token scope.

DO:
```typescript
import { CosmosClient } from '@azure/cosmos';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const client = new CosmosClient({
  endpoint: config.COSMOS_ENDPOINT,
  aadCredentials: credential,
});

const container = client.database('mydb').container('mycontainer');

// Query with partition key
const { resources: items } = await container.items.query({
  query: 'SELECT * FROM c WHERE c.category = @category',
  parameters: [{ name: '@category', value: 'electronics' }],
}).fetchAll();

// Point read (most efficient)
const { resource: item } = await container.item('item-id', 'partition-key-value').read();

// Bulk operations (max 100 per call, use .bulk() not .batch())
const operations = [
  { operationType: 'Create' as const, resourceBody: { id: '1', category: 'electronics', name: 'Laptop' } },
];
const bulkResult = await container.items.bulk(operations);
```

DON'T:
```typescript
const client = new CosmosClient({ endpoint, key: config.COSMOS_PRIMARY_KEY }); // Use aadCredentials
const { resources } = await container.items.query('SELECT * FROM c').fetchAll(); // Missing partition key
```

Note: Cosmos DB requires account-specific token scope: `https://{account}.documents.azure.com/.default`

---

## DB-2: SQL Database Patterns - Tedious Driver (HIGH)

**Pattern:** Use Promise wrappers for tedious callbacks. Enable `rowCollectionOnDone`. Use native transaction methods.

DO:
```typescript
import { Connection, Request, TYPES } from 'tedious';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const tokenResponse = await credential.getToken('https://database.windows.net/.default');

const connection = new Connection({
  server: config.AZURE_SQL_SERVER,
  authentication: {
    type: 'azure-active-directory-access-token',
    options: { token: tokenResponse.token },
  },
  options: {
    database: config.AZURE_SQL_DATABASE,
    encrypt: true,
    rowCollectionOnDone: true,
  },
});

// Parameterized query
await executeQueryAsync(connection,
  'SELECT * FROM [Products] WHERE [Category] = @category',
  [{ name: 'category', type: TYPES.NVarChar, value: 'Electronics' }]
);

// Transaction with native methods (CRITICAL)
try {
  await beginTransactionAsync(connection);
  await executeQueryAsync(connection, 'INSERT INTO [Orders] ...');
  await commitTransactionAsync(connection);
} catch (err) {
  await rollbackTransactionAsync(connection);
  throw err;
}
```

DON'T:
```typescript
// NEVER use raw SQL transaction commands with tedious
await executeQueryAsync(connection, 'BEGIN TRANSACTION');
await executeQueryAsync(connection, 'COMMIT TRANSACTION');
// Tedious wraps in sp_executesql, creating nested transaction scope
```

---

## DB-3: SQL Identifier Quoting (MEDIUM)

**Pattern:** ALL dynamic SQL identifiers must use `[brackets]`.

DO: `SELECT [id], [name] FROM [${tableName}] WHERE [id] = @id`

DON'T: `SELECT id, name FROM ${tableName} WHERE id = @id`

---

## DB-4: Batch Operations (HIGH)

**Pattern:** Avoid row-by-row operations. Use batch operations.

DO (SQL):
```typescript
const BATCH_SIZE = 100; // SQL Server ~2100 param limit: 100 rows * 3 params = 300
for (let i = 0; i < items.length; i += BATCH_SIZE) {
  const batch = items.slice(i, i + BATCH_SIZE);
  const valuesClause = batch.map((_, idx) => `(@id${idx}, @name${idx}, @cat${idx})`).join(', ');
  const sql = `INSERT INTO [Products] ([id], [name], [category]) VALUES ${valuesClause}`;
  const params = batch.flatMap((item, idx) => [
    { name: `id${idx}`, type: TYPES.Int, value: item.id },
    { name: `name${idx}`, type: TYPES.NVarChar, value: item.name },
    { name: `cat${idx}`, type: TYPES.NVarChar, value: item.category },
  ]);
  await executeQueryAsync(connection, sql, params);
}
```

DO (Cosmos DB): Use `container.items.bulk(operations)` with max 100 ops per call.

DO (Storage Blob): `await Promise.all(files.map(f => blobClient.uploadFile(f.path)));`

DON'T:
```typescript
for (const item of items) {
  await executeQueryAsync(connection, 'INSERT INTO [Products] VALUES (@id, @name)', [...]); // N round trips
}
```

---

## DB-5: Azure Storage Patterns (MEDIUM)

**Pattern:** Use `@azure/storage-blob`, `@azure/storage-file-share`, `@azure/data-tables` with `DefaultAzureCredential`.

DO:
```typescript
import { BlobServiceClient } from '@azure/storage-blob';
import { TableClient } from '@azure/data-tables';
const credential = new DefaultAzureCredential();

const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`, credential
);
const containerClient = blobServiceClient.getContainerClient('mycontainer');
await containerClient.createIfNotExists();

const tableClient = new TableClient(
  `https://${accountName}.table.core.windows.net`, 'mytable', credential
);
```

---

## DB-6: Storage SAS Fallback (MEDIUM)

**Pattern:** For local dev where `DefaultAzureCredential` isn't available, provide SAS token fallback.

DO:
```typescript
let blobServiceClient: BlobServiceClient;
if (process.env.AZURE_STORAGE_SAS_TOKEN) {
  blobServiceClient = new BlobServiceClient(
    `https://${accountName}.blob.core.windows.net${process.env.AZURE_STORAGE_SAS_TOKEN}`
  );
} else {
  blobServiceClient = new BlobServiceClient(
    `https://${accountName}.blob.core.windows.net`, new DefaultAzureCredential()
  );
}
```
