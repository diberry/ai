# Azure SDK Client Patterns

Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

## AZ-1: Client Construction with DefaultAzureCredential (HIGH)

Use `DefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

DO:
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
from azure.keyvault.secrets import SecretClient
from azure.servicebus import ServiceBusClient
from azure.cosmos import CosmosClient

# Cache credential instance
credential = DefaultAzureCredential()

# Storage Blob
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
)

# Key Vault
secret_client = SecretClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)

# Service Bus
service_bus_client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)

# Cosmos DB
cosmos_client = CosmosClient(
    url=config["COSMOS_ENDPOINT"],
    credential=credential,
)
```

DON'T:
```python
# Don't use connection strings in samples (prefer AAD auth)
blob_service_client = BlobServiceClient.from_connection_string(connection_string)

# Don't use account keys
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=account_key,  # Use DefaultAzureCredential
)

# Don't recreate credential for each client
blob_client = BlobServiceClient(url, credential=DefaultAzureCredential())
secret_client = SecretClient(url, credential=DefaultAzureCredential())
```

---

## AZ-2: Client Options — Retry Policies and Timeouts (MEDIUM)

Configure retry policies, timeouts, and logging for production-ready samples.

> Retry configuration differs between Azure SDK packages. Most clients use `RetryPolicy` from `azure.core.pipeline.policies`. The `azure-storage-*` packages accept shorthand kwargs directly on the client constructor.

DO (Generic — most Azure SDK clients):
```python
from azure.core.pipeline.policies import RetryPolicy
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

retry_policy = RetryPolicy(
    retry_total=3,
    retry_backoff_factor=0.8,
    retry_backoff_max=30,
)

secret_client = SecretClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
    retry_policy=retry_policy,
    connection_timeout=10,
    read_timeout=30,
)
```

DO (Storage-specific shorthand):
```python
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Storage packages accept retry kwargs directly
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
    retry_total=3,
    retry_backoff_factor=1.0,
    retry_backoff_max=30,
    connection_timeout=10,
    read_timeout=30,
)
```

DON'T:
```python
# Don't use storage-specific kwargs on non-storage clients
secret_client = SecretClient(
    vault_url=url,
    credential=credential,
    retry_total=3,  # Not recognized by Key Vault client — use RetryPolicy
)
```

---

## AZ-3: Managed Identity Patterns (HIGH)

For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

DO:
```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
import os

# For samples: DefaultAzureCredential (works locally + cloud)
credential = DefaultAzureCredential()

# For production: Explicitly use managed identity when deployed
# System-assigned (simpler, auto-managed lifecycle)
credential = ManagedIdentityCredential()

# User-assigned (when multiple identities needed)
credential = ManagedIdentityCredential(
    client_id=os.environ["AZURE_CLIENT_ID"]
)
```

DON'T:
```python
# Don't hardcode service principal credentials in samples
from azure.identity import ClientSecretCredential

credential = ClientSecretCredential(tenant_id, client_id, client_secret)
```

When to use:
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

## AZ-4: Token Management for Non-SDK HTTP (CRITICAL)

For services without official SDK, get tokens with `get_token()`. Tokens expire after ~1 hour — implement refresh logic for long-running samples.

DO:
```python
import time
from azure.identity import DefaultAzureCredential
from azure.core.credentials import AccessToken

credential = DefaultAzureCredential()


def get_azure_sql_token(credential: DefaultAzureCredential) -> AccessToken:
    """Get an access token for Azure SQL Database."""
    return credential.get_token("https://database.windows.net/.default")


def is_token_expiring_soon(token: AccessToken, buffer_seconds: int = 300) -> bool:
    """Check if token will expire within buffer_seconds (default 5 minutes)."""
    return (token.expires_on - time.time()) < buffer_seconds


token = get_azure_sql_token(credential)

# Before long operation, check expiration
if is_token_expiring_soon(token):
    print("Token expiring soon, refreshing...")
    token = get_azure_sql_token(credential)
```

DON'T:
```python
# CRITICAL: Don't acquire token once and use for hours
token = credential.get_token("https://database.windows.net/.default")
# ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

---

## AZ-5: DefaultAzureCredential Configuration (MEDIUM)

Configure which credential types `DefaultAzureCredential` tries.

DO:
```python
from azure.identity import DefaultAzureCredential

# For CI/CD environments (no interactive prompts)
credential = DefaultAzureCredential(
    exclude_interactive_browser_credential=True,
    exclude_workload_identity_credential=False,  # Keep for K8s
    exclude_managed_identity_credential=False,   # Keep for Azure
)

# For local development (include browser auth)
credential = DefaultAzureCredential(
    exclude_interactive_browser_credential=False,
)
```

---

## AZ-6: Resource Cleanup — Context Managers, async with (MEDIUM)

Samples must properly close/dispose clients. Use `with` statements or `async with` for automatic cleanup.

DO:
```python
from azure.servicebus import ServiceBusClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Pattern 1: Context manager (sync)
with ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
) as client:
    sender = client.get_queue_sender(queue_name="myqueue")
    with sender:
        sender.send_messages(ServiceBusMessage("Hello"))

# Pattern 2: try/finally
client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)
try:
    sender = client.get_queue_sender(queue_name="myqueue")
    sender.send_messages(ServiceBusMessage("Hello"))
    sender.close()
finally:
    client.close()
```

DON'T:
```python
# Don't forget to close clients
client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)
sender = client.get_queue_sender(queue_name="myqueue")
sender.send_messages(ServiceBusMessage("Hello"))
# Client and sender never closed (resource leak)
```

---

## AZ-7: Pagination with ItemPaged/AsyncItemPaged (HIGH)

Use `for` loops for paginated Azure SDK responses. Samples that only process the first page silently lose data.

DO:
```python
from azure.storage.blob import BlobServiceClient

# Blob Storage — iterate all pages automatically
container_client = blob_service_client.get_container_client("mycontainer")
for blob in container_client.list_blobs():
    print(f"Blob: {blob.name}")

# Cosmos DB — iterate all pages
container = database.get_container_client("mycontainer")
items = container.query_items(
    query="SELECT * FROM c WHERE c.category = @category",
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics",
)
for item in items:
    print(f"Item: {item['name']}")

# Key Vault — iterate all secrets
secret_client = SecretClient(vault_url=vault_url, credential=credential)
for secret_properties in secret_client.list_properties_of_secrets():
    print(f"Secret: {secret_properties.name}")

# Async pagination
async for blob in container_client.list_blobs():
    print(f"Blob: {blob.name}")
```

DON'T:
```python
# CRITICAL BUG: Only gets first page manually
blobs = []
for blob in container_client.list_blobs():
    blobs.append(blob)
    if len(blobs) >= 10:
        break  # Stops after 10, may miss thousands
```

> A `break` after a small count is acceptable in quickstart/demo samples to keep output short. For production-style samples, always iterate all pages.
