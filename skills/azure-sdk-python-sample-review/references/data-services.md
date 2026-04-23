# Data Services (Cosmos DB, SQL, Storage, Tables)

Database and storage client patterns, connection management, transactions, batching, and query parameterization. Includes service-specific best practices for Cosmos DB, Azure SQL, and Storage.

## DB-1: Cosmos DB (azure-cosmos) Patterns (HIGH)

Use `azure-cosmos` with AAD credentials. Handle partitioned containers properly.

DO:
```python
from azure.cosmos import CosmosClient, PartitionKey
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

client = CosmosClient(
    url=config["COSMOS_ENDPOINT"],
    credential=credential,
)

database = client.get_database_client("mydb")
container = database.get_container_client("mycontainer")

# Query with partition key
items = container.query_items(
    query="SELECT * FROM c WHERE c.category = @category",
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics",
)
for item in items:
    print(f"Item: {item['name']}")

# Point read (most efficient)
item = container.read_item(item="item-id", partition_key="partition-key-value")

# Create with partition key
container.create_item(body={
    "id": "item-id",
    "category": "electronics",  # Partition key
    "name": "Laptop",
})

# Upsert
container.upsert_item(body={
    "id": "item-id",
    "category": "electronics",
    "name": "Laptop Pro",
})
```

DON'T:
```python
# Don't use primary key in samples
client = CosmosClient(
    url=config["COSMOS_ENDPOINT"],
    credential=config["COSMOS_PRIMARY_KEY"],  # Use DefaultAzureCredential
)

# Don't omit partition key in queries (cross-partition queries are expensive)
items = container.query_items(
    query="SELECT * FROM c",
    enable_cross_partition_query=True,  # Expensive, avoid in samples
)
```

---

## DB-2: Azure SQL with pyodbc/pymssql (HIGH)

Use `pyodbc` with AAD token authentication. Use parameterized queries.

> For simpler setups, use `Authentication=Active Directory Default` in the connection string instead of manual token acquisition.

DO (Recommended -- connection string with Active Directory Default):
```python
import pyodbc

# Simplest approach: let the ODBC driver handle AAD auth
connection = pyodbc.connect(
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={config['AZURE_SQL_SERVER']};"
    f"DATABASE={config['AZURE_SQL_DATABASE']};"
    f"Authentication=Active Directory Default;"
    f"Encrypt=yes;TrustServerCertificate=no;"
)

# Parameterized query
cursor = connection.cursor()
cursor.execute(
    "SELECT * FROM [Products] WHERE [Category] = ?",
    ("Electronics",),
)
rows = cursor.fetchall()

connection.close()
```

DO (Manual token -- when you need token for other purposes):
```python
import pyodbc
import struct
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

token = credential.get_token("https://database.windows.net/.default")

token_bytes = token.token.encode("utf-16-le")
token_struct = struct.pack(f"<I{len(token_bytes)}s", len(token_bytes), token_bytes)
SQL_COPT_SS_ACCESS_TOKEN = 1256

connection = pyodbc.connect(
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={config['AZURE_SQL_SERVER']};"
    f"DATABASE={config['AZURE_SQL_DATABASE']};"
    f"Encrypt=yes;TrustServerCertificate=no;",
    attrs_before={SQL_COPT_SS_ACCESS_TOKEN: token_struct},
)

cursor = connection.cursor()
cursor.execute(
    "SELECT * FROM [Products] WHERE [Category] = ?",
    ("Electronics",),
)
rows = cursor.fetchall()

connection.close()
```

DON'T:
```python
# Don't use SQL Server authentication in samples
connection = pyodbc.connect(
    f"SERVER={server};DATABASE={database};UID={username};PWD={password}"
)

# Don't use string formatting for query values
cursor.execute(f"SELECT * FROM Products WHERE Category = '{category}'")  # SQL injection!
```

---

## DB-3: SQL Parameter Safety (Parameterized Queries) (MEDIUM)

ALL query values must use parameterized queries. ALL dynamic SQL identifiers must use bracket quoting.

DO:
```python
# Parameterized values
cursor.execute(
    "SELECT [id], [name] FROM [Products] WHERE [category] = ? AND [price] > ?",
    (category, min_price),
)

# Bracket-quoted dynamic identifiers
table_name = config["TABLE_NAME"]
cursor.execute(
    f"SELECT [id], [name] FROM [{table_name}] WHERE [id] = ?",
    (item_id,),
)
```

DON'T:
```python
# String interpolation for values (SQL injection!)
cursor.execute(f"SELECT * FROM Products WHERE category = '{category}'")

# Missing brackets on dynamic identifiers
cursor.execute(f"SELECT id, name FROM {table_name} WHERE id = ?", (item_id,))
```

---

## DB-4: Batch Operations (HIGH)

Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

DO (SQL -- Batch Insert):
```python
BATCH_SIZE = 10  # SQL Server max ~2100 params; 10 rows * 3 params = 30

items = [...]  # List of items

for i in range(0, len(items), BATCH_SIZE):
    batch = items[i : i + BATCH_SIZE]

    placeholders = ", ".join(["(?, ?, ?)"] * len(batch))
    sql = f"INSERT INTO [Products] ([id], [name], [category]) VALUES {placeholders}"

    params = []
    for item in batch:
        params.extend([item["id"], item["name"], item["category"]])

    cursor.execute(sql, params)

connection.commit()
```

DO (Cosmos DB -- Transactional Batch):
```python
operations = [
    ("create", ({"id": "1", "category": "electronics", "name": "Laptop"},)),
    ("create", ({"id": "2", "category": "electronics", "name": "Mouse"},)),
    ("upsert", ({"id": "3", "category": "electronics", "name": "Keyboard"},)),
]

container.execute_item_batch(
    batch_operations=operations,
    partition_key="electronics",
)
```

DON'T:
```python
# Row-by-row INSERT (50 round trips for 50 items)
for item in items:
    cursor.execute(
        "INSERT INTO [Products] VALUES (?, ?, ?)",
        (item["id"], item["name"], item["category"]),
    )
    connection.commit()  # Commit per row is very slow
```

---

## DB-5: Azure Storage (azure-storage-blob) (MEDIUM)

Use `azure-storage-blob`, `azure-storage-file-share`, `azure-data-tables` with `DefaultAzureCredential`.

DO:
```python
from azure.storage.blob import BlobServiceClient
from azure.data.tables import TableClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Blob Storage
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
)

container_client = blob_service_client.get_container_client("mycontainer")
container_client.create_container()

blob_client = container_client.get_blob_client("myblob.txt")
blob_client.upload_blob(b"Hello, Azure!", overwrite=True)

download_stream = blob_client.download_blob()
content = download_stream.readall().decode("utf-8")

# Table Storage
table_client = TableClient(
    endpoint=f"https://{account_name}.table.core.windows.net",
    table_name="mytable",
    credential=credential,
)

table_client.create_table()
table_client.create_entity(entity={
    "PartitionKey": "partition1",
    "RowKey": "row1",
    "Name": "Sample",
})
```

---

## DB-6: SAS Token Fallback (MEDIUM)

For local development or CI environments where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

DO:
```python
import os
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential

# Try AAD first, fall back to SAS for local dev
sas_token = os.environ.get("AZURE_STORAGE_SAS_TOKEN")

if sas_token:
    blob_service_client = BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net{sas_token}"
    )
    print("Using SAS token authentication (local dev)")
else:
    credential = DefaultAzureCredential()
    blob_service_client = BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net",
        credential=credential,
    )
    print("Using DefaultAzureCredential (AAD)")
```
