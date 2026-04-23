# Project Setup & Configuration

Package structure, Python version, dependency management, environment variables, and type hint configuration. These foundational patterns ensure samples install correctly and run reliably across environments.

## PS-1: Python Version (HIGH)

Target Python 3.12 or 3.13 for new samples. Minimum supported is 3.10. Declare in `pyproject.toml` and README.

> **Note:** Python 3.9 reached end-of-life in October 2025. Use 3.10+ as the minimum.

DO:
```toml
# pyproject.toml
[project]
requires-python = ">=3.10"

[tool.ruff]
target-version = "py312"
```

```python
# src/main.py — use modern syntax available in 3.10+
def process_items(items: list[str] | None = None) -> dict[str, int]:
    """Process items using modern union syntax (PEP 604, 3.10+)."""
    results: dict[str, int] = {}
    for item in items or []:
        results[item] = len(item)
    return results
```

DON'T:
```toml
# pyproject.toml
[project]
requires-python = ">=3.9"  # EOL (October 2025), no security patches
```

```python
# Legacy typing imports (unnecessary in 3.10+)
from typing import Dict, List, Optional, Union

def process_items(items: Optional[List[str]] = None) -> Dict[str, int]:
    ...
```

---

## PS-2: pyproject.toml Metadata (MEDIUM)

All sample packages must include complete metadata in `pyproject.toml` for discoverability and maintenance.

DO:
```toml
# pyproject.toml
[project]
name = "azure-storage-blob-quickstart"
version = "1.0.0"
description = "Upload and download blobs using Azure Blob Storage SDK"
authors = [{ name = "Microsoft Corporation" }]
license = { text = "MIT" }
readme = "README.md"
requires-python = ">=3.10"

[project.urls]
Repository = "https://github.com/Azure-Samples/azure-storage-blob-samples"
```

DON'T:
```toml
# pyproject.toml
[project]
name = "my-sample"
version = "1.0.0"
# Missing authors, license, repository, requires-python
```

---

## PS-3: Dependency Audit (CRITICAL)

Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`azure-*`).

DO:
```toml
# pyproject.toml
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",       # Track 2, used in src/main.py
    "azure-identity>=1.17.0",            # Used for auth
    "azure-keyvault-secrets>=4.8.0",     # Used in src/secrets.py
    "openai>=1.40.0",                    # For AzureOpenAI class
    "python-dotenv>=1.0.0",             # Used in src/config.py
]
```

DON'T:
```toml
# pyproject.toml
[project]
dependencies = [
    "azure-storage>=0.36.0",             # Track 1 (legacy)
    "azure-ai-openai>=1.0.0",           # No such package (use openai SDK)
    "requests>=2.31.0",                  # Listed but never imported
    "azure-cosmos>=4.7.0",              # Listed but never imported
]
```

---

## PS-4: Azure SDK Package Naming — Track 2 (HIGH)

Use Track 2 packages (`azure-*` on PyPI) not Track 1 legacy packages.

DO (Track 2):
```python
from azure.storage.blob import BlobServiceClient
from azure.keyvault.secrets import SecretClient
from azure.servicebus import ServiceBusClient
from azure.cosmos import CosmosClient
from azure.data.tables import TableClient
from azure.identity import DefaultAzureCredential

# Azure OpenAI via openai SDK (not azure-ai-openai)
from openai import AzureOpenAI
```

DON'T (Track 1 Legacy):
```python
from azure.storage import BlobService          # Use azure-storage-blob
from azure.keyvault import KeyVaultClient      # Use azure-keyvault-secrets
from azure.servicebus import ServiceBusService # Use azure-servicebus (Track 2)

# Nonexistent or retired packages
from azure.ai.openai import OpenAIClient       # Use openai SDK's AzureOpenAI
```

---

## PS-5: Configuration (.env, python-dotenv) (MEDIUM)

Use `python-dotenv` to load `.env` files. Validate all required variables with descriptive errors.

DO:
```python
# src/config.py
import os
from dotenv import load_dotenv

load_dotenv()

def get_config() -> dict[str, str]:
    """Load and validate required environment variables."""
    required = {
        "AZURE_STORAGE_ACCOUNT_NAME": os.environ.get("AZURE_STORAGE_ACCOUNT_NAME"),
        "AZURE_KEYVAULT_URL": os.environ.get("AZURE_KEYVAULT_URL"),
        "AZURE_SERVICE_BUS_NAMESPACE": os.environ.get("AZURE_SERVICE_BUS_NAMESPACE"),
    }

    missing = [key for key, value in required.items() if not value]

    if missing:
        raise ValueError(
            f"Missing required environment variables: {', '.join(missing)}\n"
            f"Create a .env file with these values or set them in your environment.\n"
            f"See .env.sample for required variables."
        )

    return {k: v for k, v in required.items() if v is not None}
```

DON'T:
```python
# Don't silently fall back to defaults for Azure resources
config = {
    "storage_account": os.environ.get("AZURE_STORAGE_ACCOUNT_NAME", "devstoreaccount1"),
}

# Don't skip validation entirely
endpoint = os.environ["AZURE_OPENAI_ENDPOINT"]  # KeyError with no guidance
```

---

## PS-6: Type Hints (MEDIUM)

Use modern type hints (PEP 484/585/604). All public functions must have type annotations.

DO:
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient


def create_blob_client(
    account_name: str,
    credential: DefaultAzureCredential | None = None,
) -> BlobServiceClient:
    """Create a BlobServiceClient with DefaultAzureCredential."""
    if credential is None:
        credential = DefaultAzureCredential()
    return BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net",
        credential=credential,
    )


def process_blobs(blob_names: list[str]) -> dict[str, int]:
    """Process blob names and return size mapping."""
    return {name: len(name) for name in blob_names}
```

DON'T:
```python
# No type hints at all
def create_blob_client(account_name, credential=None):
    if credential is None:
        credential = DefaultAzureCredential()
    return BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net",
        credential=credential,
    )
```

---

## PS-7: .editorconfig / pyproject.toml Formatting (LOW)

Include `.editorconfig` and configure formatting tools in `pyproject.toml`. Use `ruff format` OR `black` — not both.

DO:
```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.py]
indent_style = space
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

```toml
# pyproject.toml — ruff for both linting AND formatting (recommended)
[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]
```

> Don't run both `ruff format` and `black` — `ruff` includes a near-identical formatter. Choose one.

---

## PS-8: Lock File (HIGH)

Commit lock files or pinned requirements. Use only one package manager per project.

DO:
```text
# requirements.txt (pinned versions for reproducibility)
azure-identity==1.17.1
azure-storage-blob==12.22.0
azure-keyvault-secrets==4.8.0
openai==1.40.6
python-dotenv==1.0.1
```

DON'T:
```gitignore
# Don't ignore lock files
requirements.txt
poetry.lock
```

```
repo/
├── requirements.txt    # Mixed package managers
├── Pipfile.lock        # Don't mix pip and pipenv
├── poetry.lock         # Don't mix pip and poetry
```

---

## PS-9: CVE Scanning (pip-audit, safety) (CRITICAL)

Samples must not ship with known security vulnerabilities. `pip-audit` must pass with no critical/high issues.

DO:
```bash
pip-audit
safety check
pip-audit --strict --desc
```

```yaml
# .github/workflows/ci.yml
- name: CVE scan
  run: |
    pip install pip-audit
    pip-audit --strict
```

DON'T:
```bash
# Don't ignore audit warnings
pip-audit
# found 3 vulnerabilities
# Submitting sample anyway
```

---

## PS-10: Package Legitimacy (MEDIUM)

Verify Azure SDK packages are from official `azure-*` namespace on PyPI. Watch for typosquatting.

DO:
```toml
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",   # Official package
    "azure-cosmos>=4.7.0",           # Official package
    "azure-identity>=1.17.0",        # Official package
]
```

DON'T:
```toml
[project]
dependencies = [
    "azurestorage-blob>=12.0.0",     # Typosquatting
    "azzure-identity>=1.0.0",        # Typosquatting (azzure)
    "azur-storage>=1.0.0",           # Not official package
]
```

---

## PS-11: Version Pinning Strategy (MEDIUM)

Use minimum version constraints (`>=`) in `pyproject.toml`, exact pins in `requirements.txt`.

DO:
```toml
# pyproject.toml (flexible ranges for library compatibility)
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",  # Minimum version
    "azure-identity>=1.17.0",
]
```

```text
# requirements.txt (exact pins for reproducibility)
azure-storage-blob==12.22.0
azure-identity==1.17.1
```

---

## PS-12: uv Package Manager (MEDIUM)

`uv` is a modern, fast Python package manager. Document as an option alongside `pip`.

DO:
```markdown
## Setup

### Install dependencies

**Option A: pip (standard)**
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

**Option B: uv (faster)**
uv venv .venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

> `uv` is optional. Always provide `pip` instructions as the primary path.

---

## PS-13: Logging Configuration (MEDIUM)

Azure SDK for Python uses the standard `logging` module. Document how to enable SDK logging for debugging.

DO:
```python
import logging

# Enable Azure SDK debug logging
logging.basicConfig(level=logging.WARNING)
logging.getLogger("azure").setLevel(logging.DEBUG)

# For specific packages only
logging.getLogger("azure.identity").setLevel(logging.DEBUG)
logging.getLogger("azure.storage.blob").setLevel(logging.DEBUG)

# Or enable per-client with logging_enable
from azure.storage.blob import BlobServiceClient
client = BlobServiceClient(url, credential=credential, logging_enable=True)
```

DON'T:
```python
# Don't leave DEBUG logging on in committed samples
logging.basicConfig(level=logging.DEBUG)  # Too noisy for users

# Don't silence all logging
logging.disable(logging.CRITICAL)  # Hides important errors
```
