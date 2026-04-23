# Infrastructure, azd, Async Patterns, CI/CD & Documentation

Bicep/Terraform IaC patterns, Azure Developer CLI integration, async/await patterns, CI/CD, testing, and README documentation quality.

## DOC-1: Expected Output (CRITICAL)

README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

DO:
```markdown
## Expected Output

Run the sample:
python main.py

You should see output similar to:
Connected to Azure Blob Storage
Container 'samples' created
Uploaded blob 'sample.txt' (14 bytes)
Downloaded blob content: "Hello, Azure!"

> Note: Exact output may vary based on your Azure environment.
```

DON'T:
```markdown
## Expected Output
Blob uploaded successfully  # Not actual output, fabricated
```

---

## DOC-2: Folder Path Links (CRITICAL)

All internal README links must match actual filesystem paths.

DO:
```markdown
## Project Structure

- [`src/main.py`](./src/main.py) -- Main entry point
- [`src/config.py`](./src/config.py) -- Configuration loader
- [`infra/main.bicep`](./infra/main.bicep) -- Infrastructure template
```

DON'T:
```markdown
- [`src/main.py`](./Python/src/main.py)  # Wrong path
```

---

## DOC-3: Troubleshooting Section (MEDIUM)

Include troubleshooting for common Azure errors (auth failures, firewall, RBAC, prerequisites).

DO:
```markdown
## Troubleshooting

### Authentication Errors

If you see "Failed to acquire token":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments (see Prerequisites)

### Common Azure SDK Errors

- `ResourceNotFoundError`: Resource doesn't exist or you don't have access
- `ServiceRequestError`: Incorrect endpoint URL in .env
- `ClientAuthenticationError`: No authentication method available (run `az login`)

### Python-Specific Issues

- `ModuleNotFoundError: No module named 'azure'`: Run `pip install -r requirements.txt`
- `ImportError`: Check Python version meets minimum requirement (3.10+)
```

---

## DOC-4: Prerequisites Section (HIGH)

Document all prerequisites clearly (Azure subscription, CLI tools, role assignments, services).

DO:
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Python**: Version 3.10 or later ([Download](https://www.python.org/downloads/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional)

### Role Assignments

Your Azure identity needs these role assignments:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
- `Cognitive Services OpenAI User` on the Azure OpenAI resource
```

---

## DOC-5: Setup Instructions (MEDIUM)

Provide clear, tested setup instructions. Include virtual environment creation.

DO:
```markdown
## Setup

### 1. Clone the repository
git clone https://github.com/Azure-Samples/azure-storage-blob-samples.git
cd azure-storage-blob-samples/quickstart

### 2. Create virtual environment and install dependencies
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
pip install -r requirements.txt

### 3. Provision Azure resources
azd up

### 4. Run the sample
python main.py
```

---

## DOC-7: Placeholder Values (MEDIUM)

READMEs must provide clear instructions for placeholder values.

DO:
```markdown
## Configuration

Copy `.env.sample` to `.env` and fill in your values:
cp .env.sample .env

Edit `.env` and replace placeholders:
- `AZURE_STORAGE_ACCOUNT_NAME`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `AZURE_KEYVAULT_URL`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI
```

DON'T:
```markdown
## Configuration

Set environment variables:
- `AZURE_STORAGE_ACCOUNT_NAME=<your-storage-account>`  # How do I find this?
- `AZURE_KEYVAULT_URL=<your-keyvault-url>`            # What format?
```

---

## IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)

Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

DO:
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
  name: 'storage-deployment'
  params: {
    name: storageAccountName
    location: location
  }
}
```

DON'T:
```bicep
module cognitiveServices 'br/public:avm/res/cognitive-services/account:0.7.1' = {
  // Outdated version (current is 1.0.1+)
}
```

---

## IaC-2: Bicep Parameter Validation (CRITICAL)

Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

DO:
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

DON'T:
```bicep
@description('Azure AD admin object ID')
param aadAdminObjectId string  // No validation, accepts empty string
```

---

## IaC-3: API Versions (MEDIUM)

Use current API versions (2023+). Avoid versions older than 2 years.

DO:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  // Current stable version
}
```

DON'T:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // 5+ years old
}
```

---

## IaC-4: RBAC Role Assignments (HIGH)

Create role assignments in Bicep for managed identities to access Azure resources.

DO:
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
}

resource storageBlobRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

Common role IDs:
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cosmos DB Account Reader Role: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

## IaC-5: Network Security (HIGH)

For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

DO (Quickstart):
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: openaiAccountName
  properties: {
    publicNetworkAccess: 'Enabled'  // OK for quickstart
    networkAcls: {
      defaultAction: 'Allow'
    }
  }
}

// NOTE: This quickstart uses public endpoints for simplicity.
// For production, use private endpoints and set defaultAction: 'Deny'.
```

---

## IaC-6: Output Values (MEDIUM)

Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

DO:
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_STORAGE_BLOB_ENDPOINT string = storageAccount.properties.primaryEndpoints.blob
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
```

---

## IaC-7: Resource Naming Conventions (HIGH)

Follow Cloud Adoption Framework (CAF) naming conventions.

DO:
```bicep
@description('Prefix for all resources')
param resourcePrefix string = 'contoso'

@description('Environment (dev, test, prod)')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

DON'T:
```bicep
// Inconsistent naming
var storageAccountName = 'mystorageaccount123'
var keyVaultName = 'kv-${uniqueString(resourceGroup().id)}'
```

---

## AZD-1: azure.yaml Structure (MEDIUM)

Complete `azure.yaml` with services, hooks, and metadata.

> FALSE POSITIVE PREVENTION: The `services`, `hooks`, and `host` fields are OPTIONAL. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct. Check parent directories in monorepo layouts.

DO:
```yaml
name: azure-storage-blob-sample
metadata:
  template: azure-storage-blob-sample@0.0.1

services:
  app:
    project: ./
    language: py
    host: appservice

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "Provisioning complete"
      echo "Storage Account: ${AZURE_STORAGE_ACCOUNT_NAME}"
```

---

## AZD-2: Service Host Types (MEDIUM)

Choose correct `host` type for Python applications.

DO:
```yaml
# App Service (Flask, Django, FastAPI)
services:
  web:
    project: ./
    language: py
    host: appservice

# Azure Functions
services:
  api:
    project: ./
    language: py
    host: function

# Container Apps
services:
  backend:
    project: ./
    language: py
    host: containerapp
    docker:
      path: ./Dockerfile
```

---

## AZD-3: Azure Functions Python v2 Model (MEDIUM)

Azure Functions Python v2 uses a decorator-based programming model.

DO (v2 programming model -- recommended):
```python
# function_app.py
import azure.functions as func
import logging

app = func.FunctionApp()


@app.function_name("HttpTrigger")
@app.route(route="hello", auth_level=func.AuthLevel.ANONYMOUS)
def hello(req: func.HttpRequest) -> func.HttpResponse:
    """HTTP trigger function using v2 decorator model."""
    logging.info("Python HTTP trigger function processed a request.")
    name = req.params.get("name", "World")
    return func.HttpResponse(f"Hello, {name}!")


@app.function_name("BlobTrigger")
@app.blob_trigger(arg_name="myblob", path="samples-workitems/{name}", connection="AzureWebJobsStorage")
def blob_trigger(myblob: func.InputStream) -> None:
    """Blob trigger using v2 decorator model."""
    logging.info(f"Blob trigger: {myblob.name}, Size: {myblob.length} bytes")


@app.function_name("TimerTrigger")
@app.timer_trigger(schedule="0 */5 * * * *", arg_name="timer")
def timer_trigger(timer: func.TimerRequest) -> None:
    """Timer trigger that runs every 5 minutes."""
    logging.info("Timer trigger executed")
```

DON'T:
```python
# v1 model (function.json + __init__.py) -- avoid for new samples
```

---

## ASYNC-1: Async Client Usage (HIGH)

Import async clients from `.aio` submodules. Use `async with` for automatic cleanup.

DO (Preferred -- async context manager for credential):
```python
from azure.storage.blob.aio import BlobServiceClient
from azure.identity.aio import DefaultAzureCredential

async def upload_blob(account_name: str, data: bytes) -> None:
    """Upload a blob using async client with proper credential cleanup."""
    async with DefaultAzureCredential() as credential:
        async with BlobServiceClient(
            account_url=f"https://{account_name}.blob.core.windows.net",
            credential=credential,
        ) as blob_service_client:
            container_client = blob_service_client.get_container_client("mycontainer")
            blob_client = container_client.get_blob_client("myblob.txt")
            await blob_client.upload_blob(data, overwrite=True)
```

DO (Alternative -- try/finally for credential cleanup):
```python
from azure.storage.blob.aio import BlobServiceClient
from azure.identity.aio import DefaultAzureCredential

async def upload_blob(account_name: str, data: bytes) -> None:
    """Upload a blob using async client."""
    credential = DefaultAzureCredential()
    try:
        async with BlobServiceClient(
            account_url=f"https://{account_name}.blob.core.windows.net",
            credential=credential,
        ) as blob_service_client:
            container_client = blob_service_client.get_container_client("mycontainer")
            blob_client = container_client.get_blob_client("myblob.txt")
            await blob_client.upload_blob(data, overwrite=True)
    finally:
        await credential.close()
```

DON'T:
```python
# Using sync client inside async function
async def upload_blob(account_name: str, data: bytes) -> None:
    from azure.storage.blob import BlobServiceClient  # Sync client!
    client = BlobServiceClient(url, credential=credential)
    # This blocks the event loop!
```

---

## ASYNC-2: Async Context Managers (MEDIUM)

All async Azure SDK clients support `async with`. Use it for automatic resource cleanup.

DO:
```python
from azure.servicebus.aio import ServiceBusClient
from azure.identity.aio import DefaultAzureCredential

async def send_message(namespace: str, queue_name: str) -> None:
    credential = DefaultAzureCredential()
    try:
        async with ServiceBusClient(
            fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
            credential=credential,
        ) as client:
            async with client.get_queue_sender(queue_name=queue_name) as sender:
                await sender.send_messages(ServiceBusMessage("Hello"))
    finally:
        await credential.close()
```

DON'T:
```python
# Manual close without try/finally
async def send_message(namespace: str, queue_name: str) -> None:
    credential = DefaultAzureCredential()
    client = ServiceBusClient(namespace, credential=credential)
    sender = client.get_queue_sender(queue_name=queue_name)
    await sender.send_messages(ServiceBusMessage("Hello"))
    await sender.close()
    await client.close()
    await credential.close()
    # If send_messages raises, close() never called
```

---

## ASYNC-3: asyncio.run() vs Event Loop (MEDIUM)

Use `asyncio.run()` as the entry point. Don't manually manage event loops.

DO:
```python
import asyncio


async def main() -> None:
    """Main async entry point."""
    credential = DefaultAzureCredential()
    # ... async operations
    await credential.close()


if __name__ == "__main__":
    asyncio.run(main())
```

DON'T:
```python
# Don't manually manage event loops
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()

# Don't use nest_asyncio hacks
import nest_asyncio
nest_asyncio.apply()
```

---

## ASYNC-4: Concurrent Operations with asyncio.gather (LOW)

Use `asyncio.gather` for concurrent independent operations. Use `asyncio.Semaphore` to limit concurrency.

DO:
```python
import asyncio
from azure.storage.blob.aio import BlobServiceClient


async def upload_files(
    blob_service_client: BlobServiceClient,
    files: list[tuple[str, bytes]],
    max_concurrent: int = 10,
) -> None:
    """Upload multiple files concurrently with limited concurrency."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def upload_one(name: str, data: bytes) -> None:
        async with semaphore:
            container_client = blob_service_client.get_container_client("uploads")
            blob_client = container_client.get_blob_client(name)
            await blob_client.upload_blob(data, overwrite=True)
            print(f"Uploaded: {name}")

    await asyncio.gather(
        *(upload_one(name, data) for name, data in files)
    )
```

DON'T:
```python
# Sequential uploads (slow for many files)
for name, data in files:
    blob_client = container_client.get_blob_client(name)
    await blob_client.upload_blob(data, overwrite=True)

# Unbounded concurrency (can overwhelm service)
await asyncio.gather(
    *(upload_one(name, data) for name, data in thousands_of_files)
)
```

---

## ASYNC-5: httpx Async Transport (LOW)

Azure SDK can use `httpx` as an alternative async HTTP transport.

DO:
```python
from azure.core.pipeline.transport import HttpXTransport
from azure.storage.blob.aio import BlobServiceClient
from azure.identity.aio import DefaultAzureCredential

async def upload_with_httpx(account_name: str, data: bytes) -> None:
    """Use httpx transport for async operations."""
    transport = HttpXTransport()
    async with DefaultAzureCredential() as credential:
        async with BlobServiceClient(
            account_url=f"https://{account_name}.blob.core.windows.net",
            credential=credential,
            transport=transport,
        ) as client:
            blob_client = client.get_container_client("mycontainer").get_blob_client("myblob.txt")
            await blob_client.upload_blob(data, overwrite=True)
```

> `httpx` transport is optional. The default `aiohttp` transport works well for most scenarios.

---

## CI-1: pytest and Testing (HIGH)

Include tests with pytest. Run type checking and linting in CI.

DO:
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pip install pip-audit ruff pytest
      - run: pip-audit --strict           # CVE check
      - run: ruff check .                 # Linting
      - run: ruff format --check .        # Formatting
      - run: python -m pytest tests/      # Tests
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

---

## CI-2: Type Checking with mypy or pyright (MEDIUM)

Run static type checking in CI.

DO:
```yaml
# In CI workflow
- run: pip install mypy
- run: mypy src/ --ignore-missing-imports
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

DON'T:
```python
# Skip type checking entirely
# No mypy or pyright configuration
# No type annotations
```
