# Error Handling, Data Management & Sample Hygiene

Python exception patterns, AzureError hierarchy, contextual error messages, sample data handling, repository hygiene, and governance.

## ERR-1: Azure SDK Exception Hierarchy (MEDIUM)

Use specific Azure SDK exceptions from `azure.core.exceptions`. Avoid bare `except:` blocks.

DO:
```python
from azure.core.exceptions import (
    AzureError,
    HttpResponseError,
    ResourceNotFoundError,
    ClientAuthenticationError,
    ServiceRequestError,
)

try:
    secret = secret_client.get_secret("db-password")
except ResourceNotFoundError:
    print("Secret 'db-password' not found in Key Vault")
    print("  Create it with: az keyvault secret set --name db-password --value <value>")
except ClientAuthenticationError as e:
    print(f"Authentication failed: {e.message}")
    print("  1. Run 'az login' to authenticate with Azure CLI")
    print("  2. Verify you have 'Key Vault Secrets User' role")
except HttpResponseError as e:
    print(f"HTTP error {e.status_code}: {e.message}")
except AzureError as e:
    print(f"Azure SDK error: {e}")
    raise
```

DON'T:
```python
# Bare except catches everything including KeyboardInterrupt
try:
    secret = secret_client.get_secret("db-password")
except:  # Never use bare except
    print("Something went wrong")

# Catching Exception too broadly
try:
    secret = secret_client.get_secret("db-password")
except Exception as e:  # Loses Azure-specific error context
    print(str(e))
```

---

## ERR-2: Contextual Error Messages (MEDIUM)

Provide actionable error messages with troubleshooting hints for common Azure errors.

DO:
```python
from azure.core.exceptions import ClientAuthenticationError

try:
    token = credential.get_token("https://storage.azure.com/.default")
except ClientAuthenticationError as e:
    print(f"Failed to acquire Azure Storage token: {e.message}")
    print("\nTroubleshooting:")
    print("  1. Run 'az login' to authenticate with Azure CLI")
    print("  2. Verify you have the 'Storage Blob Data Contributor' role")
    print("  3. Check your Azure subscription is active")
    print("  4. Ensure firewall rules allow access from your IP")
    raise
```

---

## ERR-3: Async Exception Handling (MEDIUM)

Handle exceptions in async code with proper cleanup in `finally` blocks or `async with`.

DO:
```python
import asyncio
from azure.servicebus.aio import ServiceBusClient
from azure.identity.aio import DefaultAzureCredential

async def process_messages() -> None:
    credential = DefaultAzureCredential()
    client = ServiceBusClient(
        fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
        credential=credential,
    )
    try:
        async with client:
            receiver = client.get_queue_receiver(queue_name="myqueue")
            async with receiver:
                messages = await receiver.receive_messages(max_message_count=10)
                for message in messages:
                    try:
                        print(f"Processing: {str(message)}")
                        await receiver.complete_message(message)
                    except Exception as e:
                        print(f"Failed to process message: {e}")
                        await receiver.abandon_message(message)
    finally:
        await credential.close()
```

DON'T:
```python
# No cleanup of async resources
async def process_messages():
    credential = DefaultAzureCredential()
    client = ServiceBusClient(namespace, credential=credential)
    receiver = client.get_queue_receiver(queue_name="myqueue")
    messages = await receiver.receive_messages(max_message_count=10)
    # receiver, client, credential never closed
```

---

## DATA-1: Pre-Computed Data Files (HIGH)

Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

DO:
```
repo/
├── data/
│   ├── products.json              # Sample data
│   ├── products-with-vectors.json # Pre-computed embeddings
├── src/
│   ├── main.py                    # Loads products-with-vectors.json
│   ├── generate_embeddings.py     # Generates embeddings (optional)
```

DON'T:
```
repo/
├── data/
│   ├── products.json              # Raw data
│   ├── .gitignore                 # products-with-vectors.json gitignored
├── src/
│   ├── main.py                    # Fails: FileNotFoundError
```

> FALSE POSITIVE PREVENTION: Before flagging a data file as missing:
> 1. Check the FULL PR file list -- data files often live in sibling or parent directories.
> 2. Trace the file path in code relative to the working directory.
> 3. Check for monorepo patterns -- data may be shared across samples.

---

## DATA-2: JSON Data Loading (MEDIUM)

Use `pathlib.Path` for file paths. Use `importlib.resources` for package data.

DO:
```python
import json
from pathlib import Path
from dataclasses import dataclass


@dataclass
class Product:
    id: str
    name: str
    category: str


def load_products() -> list[Product]:
    """Load products from JSON data file."""
    data_path = Path(__file__).parent.parent / "data" / "products.json"

    if not data_path.exists():
        raise FileNotFoundError(
            f"Data file not found: {data_path}\n"
            f"Ensure 'data/products.json' exists in the project root."
        )

    with open(data_path, encoding="utf-8") as f:
        raw_data = json.load(f)

    return [Product(**item) for item in raw_data]
```

DON'T:
```python
# Hard-coded relative paths that break based on cwd
with open("data/products.json") as f:
    data = json.load(f)

# No error message when file missing
data = json.load(open("products.json"))  # Also: unclosed file handle
```

---

## HYG-1: .gitignore (CRITICAL)

Always protect sensitive files, build artifacts, and dependencies with comprehensive `.gitignore`.

DO:
```gitignore
# Environment variables (may contain credentials)
.env
.env.local
.env.*.local
.env.development
.env.production
.env.test
!.env.sample
!.env.example

# Python
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
dist/
build/
*.egg

# Virtual environments
.venv/
venv/
ENV/

# Azure
.azure/
*.zip

# IDE
.vscode/
.idea/
*.swp
*.sublime-*

# OS
.DS_Store
Thumbs.db
*.log

# Test coverage
htmlcov/
.coverage
.pytest_cache/
.mypy_cache/
.ruff_cache/
```

DON'T:
```
repo/
├── .env                    # Live credentials committed!
├── __pycache__/            # Bytecode committed
├── .venv/                  # Virtual environment committed
├── dist/                   # Build artifacts committed
```

> FALSE POSITIVE PREVENTION: Before flagging `.env` as committed:
> 1. Check .gitignore in project and parent directories.
> 2. Run `git ls-files .env` -- if empty, the file is NOT tracked.
> 3. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked.

---

## HYG-2: .env.sample (HIGH)

Provide `.env.sample` with placeholder values. Never commit actual `.env` files.

DO:
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/

.gitignore:
  .env
  .env.*
  !.env.sample
  !.env.example
```

DON'T:
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # Real subscription ID
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321       # Real tenant ID
```

---

## HYG-3: Dead Code (HIGH)

Remove unused files, functions, and imports. Commented-out code confuses users.

DO:
```python
# Only import what you use
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential
```

DON'T:
```python
# Commented-out code confuses users
# from azure.old_package import OldClient
#
# def old_implementation():
#     """This was the old way..."""
#     pass

from azure.storage.blob import BlobServiceClient
```

---

## HYG-4: LICENSE File (HIGH)

All Azure Samples repositories must include MIT LICENSE file.

DO:
```
repo/
├── LICENSE              # MIT license (required for Azure Samples org)
├── README.md
├── pyproject.toml
├── src/
```

> FALSE POSITIVE PREVENTION: Before flagging a missing LICENSE:
> 1. Check the REPO ROOT for `LICENSE`, `LICENSE.md`, `LICENSE.txt`.
> 2. Check parent directories -- in monorepos, a single license at root covers all subdirectories.
> 3. Only flag if NO license file exists at ANY level of the repo hierarchy.

---

## HYG-5: Repository Governance Files (MEDIUM)

Samples in Azure Samples org should reference or include governance files: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

DO:
```markdown
# README.md

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA). See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## Security

Microsoft takes security seriously. If you believe you have found a security vulnerability,
please report it as described in [SECURITY.md](SECURITY.md).

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
```
