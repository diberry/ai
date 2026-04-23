---
name: "azure-sdk-python-sample-review"
description: "Comprehensive review checklist for Azure SDK Python code samples covering project setup, Azure SDK client patterns, authentication, data services, messaging, AI services, Key Vault, infrastructure, documentation, and sample hygiene."
domain: "code-review"
confidence: "high"
source: "earned -- adapted from TypeScript review skill patterns, generalized for Python Azure SDK ecosystem"
---

## Context

Use this skill when reviewing **Python code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general Python review—it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (Track 2 `azure-*` PyPI packages, client construction, pipeline options)
- **Authentication patterns** (`DefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Service Bus, Key Vault, AI services)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)
- **Async patterns** (asyncio, async clients, context managers)

This skill captures patterns and anti-patterns discovered during comprehensive reviews of Azure SDK Python samples, adapted from proven TypeScript review patterns and generalized for the Python Azure SDK ecosystem.

**Total rules: 75** (11 CRITICAL, 23 HIGH, 32 MEDIUM, 9 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **pyproject.toml or requirements.txt**: Uses Track 2 Azure SDK packages (`azure-*`, not `azure.mgmt` old-style)
- [ ] **Authentication**: Uses `DefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `.env`, `__pycache__/`, `*.pyc`, `.venv/`
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: `pip-audit` passes with no critical/high vulnerabilities
- [ ] **Type hints**: Functions have type annotations (PEP 484/585/604)
- [ ] **Error handling**: `try/except` blocks present with specific exception types
- [ ] **Resource cleanup**: Clients properly closed (`with` statements or `async with`)
- [ ] **Lock file**: `requirements.txt` with pinned versions or `poetry.lock` committed
- [ ] **No mixed package managers**: Only pip OR poetry OR uv (not multiple)
- [ ] **Imports work**: No broken imports, all dependencies installed
- [ ] **Sample runs**: `python main.py` executes without crashes
- [ ] **Python version**: 3.10+ required, 3.12/3.13 target

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets**—Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication**—No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling**—Unhandled exceptions, bare `except:` blocks, silent failures
4. **Broken imports**—Missing dependencies, incorrect import paths, `ModuleNotFoundError`
5. **Security vulnerabilities**—`pip-audit` shows critical or high CVEs
6. **Missing LICENSE**—No LICENSE file at ANY level of repo hierarchy (MIT required for Azure Samples org). ⚠️ Check repo root before flagging.
7. **.env file committed**—Live credentials in version control. ⚠️ Verify with `git ls-files .env`—a .env on disk but in .gitignore is NOT committed.
8. **Track 1 packages**—Uses legacy `azure-*` packages with pre-Track-2 API patterns (e.g., `azure-storage==0.36`, `azure-servicebus==0.50`). Note: recent `azure-mgmt-*` management plane packages follow Track 2 patterns.

---

## 1. Project Setup & Configuration

**What this section covers:** Package structure, Python version, dependency management, environment variables, and type hint configuration. These foundational patterns ensure samples install correctly and run reliably across environments.

### PS-1: Python Version (HIGH)
**Pattern:** Target Python 3.12 or 3.13 for new samples. Minimum supported is 3.10. Declare in `pyproject.toml` and README.

> **Note:** Python 3.9 reached end-of-life in October 2025. Use 3.10+ as the minimum.

✅ **DO:**
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

❌ **DON'T:**
```toml
# pyproject.toml
[project]
requires-python = ">=3.9"  # ❌ EOL (October 2025), no security patches
```

```python
# ❌ Legacy typing imports (unnecessary in 3.10+)
from typing import Dict, List, Optional, Union

def process_items(items: Optional[List[str]] = None) -> Dict[str, int]:
    ...
```

**Why:** Python 3.9 and below are end-of-life. Python 3.10+ is minimum for current Azure SDK packages and enables native `X | Y` union syntax without `from __future__ import annotations`. Target 3.12/3.13 for best performance and modern syntax.

---

### PS-2: pyproject.toml Metadata (MEDIUM)
**Pattern:** All sample packages must include complete metadata in `pyproject.toml` for discoverability and maintenance.

✅ **DO:**
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

❌ **DON'T:**
```toml
# pyproject.toml
[project]
name = "my-sample"
version = "1.0.0"
# ❌ Missing authors, license, repository, requires-python
```

**Note:** `pyproject.toml` is the modern standard (PEP 621). Prefer it over `setup.py` or `setup.cfg`.

---

### PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`azure-*`).

✅ **DO:**
```toml
# pyproject.toml
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",       # ✅ Track 2, used in src/main.py
    "azure-identity>=1.17.0",            # ✅ Used for auth
    "azure-keyvault-secrets>=4.8.0",     # ✅ Used in src/secrets.py
    "openai>=1.40.0",                    # ✅ For AzureOpenAI class
    "python-dotenv>=1.0.0",             # ✅ Used in src/config.py
]
```

❌ **DON'T:**
```toml
# pyproject.toml
[project]
dependencies = [
    "azure-storage>=0.36.0",             # ❌ Track 1 (legacy)
    "azure-ai-openai>=1.0.0",           # ❌ No such package (use openai SDK)
    "requests>=2.31.0",                  # ❌ Listed but never imported
    "azure-cosmos>=4.7.0",              # ❌ Listed but never imported
]
```

**Why:** Phantom dependencies bloat install time and confuse users about what the sample actually uses.

---

### PS-4: Azure SDK Package Naming—Track 2 (HIGH)
**Pattern:** Use Track 2 packages (`azure-*` on PyPI) not Track 1 legacy packages.

✅ **DO (Track 2):**
```python
# ✅ Current generation Azure SDK packages
from azure.storage.blob import BlobServiceClient
from azure.keyvault.secrets import SecretClient
from azure.servicebus import ServiceBusClient
from azure.cosmos import CosmosClient
from azure.data.tables import TableClient
from azure.identity import DefaultAzureCredential

# ✅ Azure OpenAI via openai SDK (not azure-ai-openai)
from openai import AzureOpenAI
```

❌ **DON'T (Track 1 Legacy):**
```python
# ❌ Track 1 packages (legacy, avoid in new samples)
from azure.storage import BlobService          # Use azure-storage-blob
from azure.keyvault import KeyVaultClient      # Use azure-keyvault-secrets
from azure.servicebus import ServiceBusService # Use azure-servicebus (Track 2)

# ❌ Nonexistent or retired packages
from azure.ai.openai import OpenAIClient       # Use openai SDK's AzureOpenAI
```

**Why:** Track 2 SDKs (`azure-*`) are current generation with better async support, consistent APIs, type hints, and active maintenance. Track 1 is legacy. Use the official `openai` PyPI package with the `AzureOpenAI` class for Azure OpenAI.

---

### PS-5: Configuration (.env, python-dotenv) (MEDIUM)
**Pattern:** Use `python-dotenv` to load `.env` files. Validate all required variables with descriptive errors.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ Don't silently fall back to defaults for Azure resources
config = {
    "storage_account": os.environ.get("AZURE_STORAGE_ACCOUNT_NAME", "devstoreaccount1"),
}

# ❌ Don't skip validation entirely
endpoint = os.environ["AZURE_OPENAI_ENDPOINT"]  # KeyError with no guidance
```

---

### PS-6: Type Hints (MEDIUM)
**Pattern:** Use modern type hints (PEP 484/585/604). All public functions must have type annotations. With 3.10+ as the minimum, `from __future__ import annotations` is no longer needed for `X | Y` syntax.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ No type hints at all
def create_blob_client(account_name, credential=None):
    if credential is None:
        credential = DefaultAzureCredential()
    return BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net",
        credential=credential,
    )
```

**Why:** Type hints improve readability, enable IDE autocompletion, and catch bugs early. Azure SDK Python packages ship with type stubs.

---

### PS-7: .editorconfig / pyproject.toml Formatting (LOW)
**Pattern:** Include `.editorconfig` and configure formatting tools in `pyproject.toml`. Use `ruff format` OR `black`—not both (ruff includes a formatter).

✅ **DO:**
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
# pyproject.toml — Option A: ruff for both linting AND formatting (recommended)
[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

# Option B: black for formatting (don't use both ruff format AND black)
# [tool.black]
# line-length = 120
# target-version = ["py312"]
```

> **⚠️ Don't run both `ruff format` and `black`**—`ruff` includes a near-identical formatter. Choose one.

---

### PS-8: Lock File (requirements.txt with Pinned Versions, poetry.lock) (HIGH)
**Pattern:** Commit lock files or pinned requirements. Use only one package manager per project.

✅ **DO:**
```text
# requirements.txt (pinned versions for reproducibility)
azure-identity==1.17.1
azure-storage-blob==12.22.0
azure-keyvault-secrets==4.8.0
openai==1.40.6
python-dotenv==1.0.1
```

```gitignore
# .gitignore
__pycache__/
*.pyc
.venv/
.env
.env.*
!.env.sample

# ✅ Lock file / requirements.txt is COMMITTED (not in .gitignore)
```

❌ **DON'T:**
```gitignore
# ❌ Don't ignore lock files
requirements.txt
poetry.lock
```

```
repo/
├── requirements.txt    # ❌ Mixed package managers
├── Pipfile.lock        # ❌ Don't mix pip and pipenv
├── poetry.lock         # ❌ Don't mix pip and poetry
```

**Why:** Pinned dependencies ensure reproducible builds. Mixed package managers cause version conflicts.

---

### PS-9: CVE Scanning (pip-audit, safety) (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. `pip-audit` must pass with no critical/high issues.

✅ **DO:**
```bash
# Before submitting sample
pip-audit

# Or using safety
safety check

# In CI
pip-audit --strict --desc
```

```yaml
# .github/workflows/ci.yml
- name: CVE scan
  run: |
    pip install pip-audit
    pip-audit --strict
```

❌ **DON'T:**
```bash
# ❌ Don't ignore audit warnings
pip-audit
# found 3 vulnerabilities
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Package Legitimacy (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `azure-*` namespace on PyPI. Watch for typosquatting.

✅ **DO:**
```toml
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",   # ✅ Official package
    "azure-cosmos>=4.7.0",           # ✅ Official package
    "azure-identity>=1.17.0",        # ✅ Official package
]
```

❌ **DON'T:**
```toml
[project]
dependencies = [
    "azurestorage-blob>=12.0.0",     # ❌ Typosquatting
    "azzure-identity>=1.0.0",        # ❌ Typosquatting (azzure)
    "azur-storage>=1.0.0",           # ❌ Not official package
]
```

**Check:** All Azure SDK packages use `azure-*` naming on PyPI and are published by Microsoft. Verify on pypi.org.

---

### PS-11: Version Pinning Strategy (MEDIUM)
**Pattern:** Use minimum version constraints (`>=`) in `pyproject.toml`, exact pins in `requirements.txt`.

✅ **DO:**
```toml
# pyproject.toml (flexible ranges for library compatibility)
[project]
dependencies = [
    "azure-storage-blob>=12.20.0",  # ✅ Minimum version
    "azure-identity>=1.17.0",
]
```

```text
# requirements.txt (exact pins for reproducibility)
azure-storage-blob==12.22.0
azure-identity==1.17.1
```

**Guidance:**
- **pyproject.toml**: Use `>=` ranges—allows compatible updates
- **requirements.txt**: Use `==` pins—ensures exact reproducibility
- **poetry.lock / uv.lock**: Commit lock files for exact versions

---

### PS-12: uv Package Manager (MEDIUM)
**Pattern:** `uv` is a modern, fast Python package manager. It's an alternative to `pip`—not required, but recommended for faster installs. Document as an option alongside `pip`.

✅ **DO:**
```markdown
## Setup

### Install dependencies

**Option A: pip (standard)**
```bash
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
pip install -r requirements.txt
```

**Option B: uv (faster)**
```bash
uv venv .venv
source .venv/bin/activate  # Linux/macOS
uv pip install -r requirements.txt
```
```

```bash
# uv equivalents for common pip commands
uv pip install azure-identity azure-storage-blob   # Install packages
uv pip install -r requirements.txt                  # Install from requirements
uv pip compile pyproject.toml -o requirements.txt   # Generate locked requirements
uv run python main.py                                # Run without activating venv
```

> **Note:** `uv` is optional. Always provide `pip` instructions as the primary path. Mention `uv` as a faster alternative for developers who have it installed.

---

### PS-13: Logging Configuration (MEDIUM)
**Pattern:** Azure SDK for Python uses the standard `logging` module. Document how to enable SDK logging for debugging.

✅ **DO:**
```python
import logging

# ✅ Enable Azure SDK debug logging
logging.basicConfig(level=logging.WARNING)
logging.getLogger("azure").setLevel(logging.DEBUG)

# ✅ For specific packages only
logging.getLogger("azure.identity").setLevel(logging.DEBUG)
logging.getLogger("azure.storage.blob").setLevel(logging.DEBUG)

# ✅ Or enable per-client with logging_enable
from azure.storage.blob import BlobServiceClient
client = BlobServiceClient(url, credential=credential, logging_enable=True)
```

❌ **DON'T:**
```python
# ❌ Don't leave DEBUG logging on in committed samples
logging.basicConfig(level=logging.DEBUG)  # ❌ Too noisy for users

# ❌ Don't silence all logging
logging.disable(logging.CRITICAL)  # ❌ Hides important errors
```

**Why:** Azure SDK debug logging helps diagnose auth failures, network errors, and retry behavior. Show it commented-out or behind a flag in samples.

**What this section covers:** Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

### AZ-1: Client Construction with DefaultAzureCredential (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

✅ **DO:**
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
from azure.keyvault.secrets import SecretClient
from azure.servicebus import ServiceBusClient
from azure.cosmos import CosmosClient

# ✅ Cache credential instance
credential = DefaultAzureCredential()

# ✅ Storage Blob
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
)

# ✅ Key Vault
secret_client = SecretClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)

# ✅ Service Bus
service_bus_client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)

# ✅ Cosmos DB
cosmos_client = CosmosClient(
    url=config["COSMOS_ENDPOINT"],
    credential=credential,
)
```

❌ **DON'T:**
```python
# ❌ Don't use connection strings in samples (prefer AAD auth)
blob_service_client = BlobServiceClient.from_connection_string(connection_string)

# ❌ Don't use account keys
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=account_key,  # ❌ Use DefaultAzureCredential
)

# ❌ Don't recreate credential for each client
blob_client = BlobServiceClient(url, credential=DefaultAzureCredential())
secret_client = SecretClient(url, credential=DefaultAzureCredential())
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code, etc.) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

### AZ-2: Client Options—Retry Policies and Timeouts (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and logging for production-ready samples.

> **Important:** Retry configuration differs between Azure SDK packages. Most clients use `RetryPolicy` from `azure.core.pipeline.policies`. The `azure-storage-*` packages accept shorthand kwargs (`retry_total`, `retry_backoff_factor`, etc.) directly on the client constructor.

✅ **DO (Generic—most Azure SDK clients):**
```python
from azure.core.pipeline.policies import RetryPolicy
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Generic retry pattern (works with all azure-* clients)
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
    # For debugging: enable logging
    # logging_enable=True,
)
```

✅ **DO (Storage-specific shorthand):**
```python
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Storage packages accept retry kwargs directly (azure-storage-blob, azure-storage-file-share, etc.)
blob_service_client = BlobServiceClient(
    account_url=f"https://{account_name}.blob.core.windows.net",
    credential=credential,
    retry_total=3,                # ⚠️ Storage-specific kwarg
    retry_backoff_factor=1.0,     # ⚠️ Storage-specific kwarg
    retry_backoff_max=30,         # ⚠️ Storage-specific kwarg
    connection_timeout=10,
    read_timeout=30,
)
```

❌ **DON'T:**
```python
# ❌ Don't use storage-specific kwargs on non-storage clients
secret_client = SecretClient(
    vault_url=url,
    credential=credential,
    retry_total=3,  # ❌ Not recognized by Key Vault client—use RetryPolicy
)

# ❌ Don't omit client options for samples that do meaningful work
blob_service_client = BlobServiceClient(
    account_url=url, credential=credential
)
# No retry policy, no timeout configuration
```

---

### AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
import os

# ✅ For samples: DefaultAzureCredential (works locally + cloud)
credential = DefaultAzureCredential()

# ✅ For production: Explicitly use managed identity when deployed
# System-assigned (simpler, auto-managed lifecycle)
credential = ManagedIdentityCredential()

# ✅ User-assigned (when multiple identities needed)
credential = ManagedIdentityCredential(
    client_id=os.environ["AZURE_CLIENT_ID"]
)

# Document in README:
# > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
# > automatically use the system-assigned managed identity when deployed to Azure.
# > Ensure your App Service / Container App / Function App has a managed identity
# > assigned with appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```python
# ❌ Don't hardcode service principal credentials in samples
from azure.identity import ClientSecretCredential

credential = ClientSecretCredential(tenant_id, client_id, client_secret)
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

### AZ-4: Token Management for Non-SDK HTTP (CRITICAL)
**Pattern:** For services without official SDK (custom APIs), get tokens with `get_token()`. Tokens expire after ~1 hour—implement refresh logic for long-running samples.

✅ **DO:**
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


# ✅ Implement token refresh for long-running operations
token = get_azure_sql_token(credential)

# Before long operation, check expiration
if is_token_expiring_soon(token):
    print("Token expiring soon, refreshing...")
    token = get_azure_sql_token(credential)

# Use token.token in connection config...
```

❌ **DON'T:**
```python
# ❌ CRITICAL: Don't acquire token once and use for hours
token = credential.get_token("https://database.windows.net/.default")
# ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

**Why:** Azure tokens expire after approximately 1 hour. Samples processing large datasets or running long operations MUST refresh tokens before expiration.

---

### AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI, include managed identity for cloud.

✅ **DO:**
```python
from azure.identity import DefaultAzureCredential

# ✅ For CI/CD environments (no interactive prompts)
credential = DefaultAzureCredential(
    exclude_interactive_browser_credential=True,
    exclude_workload_identity_credential=False,  # Keep for K8s
    exclude_managed_identity_credential=False,   # Keep for Azure
)

# ✅ For local development (include browser auth)
credential = DefaultAzureCredential(
    exclude_interactive_browser_credential=False,
)

# ✅ Document the credential chain in README
# > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
# > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
# > 2. Workload identity (Azure Kubernetes Service)
# > 3. Managed identity (App Service, Functions, Container Apps)
# > 4. Azure CLI (`az login`)
# > 5. Azure PowerShell
# > 6. Interactive browser (local development only)
```

---

### AZ-6: Resource Cleanup—Context Managers, async with (MEDIUM)
**Pattern:** Samples must properly close/dispose clients. Use `with` statements or `async with` for automatic cleanup.

✅ **DO:**
```python
from azure.servicebus import ServiceBusClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Pattern 1: Context manager (sync)
with ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
) as client:
    sender = client.get_queue_sender(queue_name="myqueue")
    with sender:
        sender.send_messages(ServiceBusMessage("Hello"))
    # ✅ client.close() called automatically at block exit

# ✅ Pattern 2: try/finally
client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)
try:
    sender = client.get_queue_sender(queue_name="myqueue")
    sender.send_messages(ServiceBusMessage("Hello"))
    sender.close()
finally:
    client.close()  # ✅ Always cleanup
```

❌ **DON'T:**
```python
# ❌ Don't forget to close clients
client = ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
)
sender = client.get_queue_sender(queue_name="myqueue")
sender.send_messages(ServiceBusMessage("Hello"))
# ❌ Client and sender never closed (resource leak)
```

---

### AZ-7: Pagination with ItemPaged/AsyncItemPaged (HIGH)
**Pattern:** Use `for` loops for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```python
from azure.storage.blob import BlobServiceClient

# ✅ Blob Storage—iterate all pages automatically
container_client = blob_service_client.get_container_client("mycontainer")
for blob in container_client.list_blobs():
    print(f"Blob: {blob.name}")

# ✅ Cosmos DB—iterate all pages
container = database.get_container_client("mycontainer")
items = container.query_items(
    query="SELECT * FROM c WHERE c.category = @category",
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics",
)
for item in items:
    print(f"Item: {item['name']}")

# ✅ Key Vault—iterate all secrets
secret_client = SecretClient(vault_url=vault_url, credential=credential)
for secret_properties in secret_client.list_properties_of_secrets():
    print(f"Secret: {secret_properties.name}")

# ✅ Async pagination
async for blob in container_client.list_blobs():
    print(f"Blob: {blob.name}")
```

❌ **DON'T:**
```python
# ❌ CRITICAL BUG: Only gets first page manually
blobs = []
for blob in container_client.list_blobs():
    blobs.append(blob)
    if len(blobs) >= 10:
        break  # ❌ Stops after 10, may miss thousands
```

> **Note:** A `break` after a small count (e.g., 10) is acceptable in quickstart/demo samples to keep output short. For production-style samples, always iterate all pages. Soften this flag in quickstart reviews.

**Why:** Azure APIs return paginated results. The Python SDK's `ItemPaged` handles pagination automatically in `for` loops. Samples must demonstrate proper iteration or users will silently lose data in production.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns, API versioning, embeddings, chat completions, and document analysis. Focus on the official `openai` SDK for Azure OpenAI.

### AI-1: Azure OpenAI Client Configuration (HIGH)
**Pattern:** Use `openai` SDK's `AzureOpenAI` class. Configure `timeout`, `max_retries`, and `azure_ad_token_provider`.

✅ **DO:**
```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default",
)

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    azure_endpoint=config["AZURE_OPENAI_ENDPOINT"],
    api_version="2024-10-21",  # See: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
    timeout=30.0,              # 30 second timeout
    max_retries=3,             # Retry up to 3 times
)

# ✅ Chat completion
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}],
)

# ✅ Embeddings
embedding_response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Sample text to embed",
)

# ✅ Image generation
image_response = client.images.generate(
    model="dall-e-3",
    prompt="A photo of a cat",
    n=1,
    size="1024x1024",
)
```

❌ **DON'T:**
```python
# ❌ Don't use API keys in samples (prefer AAD)
client = AzureOpenAI(
    api_key=os.environ["AZURE_OPENAI_API_KEY"],  # ❌ Use azure_ad_token_provider
    azure_endpoint=endpoint,
)

# ❌ Don't omit timeout/retries
client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    azure_endpoint=endpoint,
    # Missing timeout, max_retries
)
```

---

### AI-2: API Version Documentation (LOW)
**Pattern:** Hardcoded API versions should include a comment linking to version docs.

✅ **DO:**
```python
client = AzureOpenAI(
    api_version="2024-10-21",
    # API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
)
```

---

### AI-3: Document Intelligence (azure-ai-documentintelligence) (MEDIUM)
**Pattern:** Use `azure-ai-documentintelligence` with `DefaultAzureCredential` where supported.

✅ **DO:**
```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
doc_client = DocumentIntelligenceClient(
    endpoint=config["DOCUMENT_INTELLIGENCE_ENDPOINT"],
    credential=credential,
)

# ✅ Analyze document with prebuilt model
with open("invoice.pdf", "rb") as f:
    poller = doc_client.begin_analyze_document(
        model_id="prebuilt-invoice",
        analyze_request=f,
        content_type="application/octet-stream",
    )
result = poller.result()

for document in result.documents:
    print(f"Document type: {document.doc_type}")
    for field_name, field in document.fields.items():
        print(f"  {field_name}: {field.content}")
```

---

### AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause silent failures or runtime errors.

✅ **DO:**
```python
# ✅ Document expected dimensions
EMBEDDING_MODEL = "text-embedding-3-small"  # 1536 dimensions
VECTOR_DIMENSION = 1536

# Validate embedding size
embedding = await get_embedding(text)
if len(embedding) != VECTOR_DIMENSION:
    raise ValueError(
        f"Embedding dimension mismatch: expected {VECTOR_DIMENSION}, "
        f"got {len(embedding)}. "
        f"Ensure model '{EMBEDDING_MODEL}' matches table schema."
    )
```

❌ **DON'T:**
```python
# ❌ Don't assume dimension without validation
embedding = await get_embedding(text)
await insert_embedding(embedding)  # May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536 (default), configurable via `dimensions` parameter
- `text-embedding-3-large`: 3072 (default), configurable via `dimensions` parameter
- `text-embedding-ada-002`: 1536 (fixed, not configurable)

> **Note:** `text-embedding-3-small` and `text-embedding-3-large` support a `dimensions` parameter to reduce output size (e.g., `dimensions=256`). When using reduced dimensions, update `VECTOR_DIMENSION` accordingly:
> ```python
> embedding_response = client.embeddings.create(
>     model="text-embedding-3-small",
>     input="Sample text",
>     dimensions=256,  # Reduced from default 1536
> )
> ```

---

### AI-5: Speech SDK (azure-cognitiveservices-speech) (MEDIUM)
**Pattern:** Use `azure-cognitiveservices-speech` for speech-to-text, text-to-speech, and real-time transcription. The Speech SDK has its own auth pattern (not `DefaultAzureCredential`—it uses `SpeechConfig` with subscription key or AAD token).

✅ **DO:**
```python
import azure.cognitiveservices.speech as speechsdk
from azure.identity import DefaultAzureCredential

# ✅ Option A: Speech key (simpler for quickstarts)
speech_config = speechsdk.SpeechConfig(
    subscription=config["AZURE_SPEECH_KEY"],
    region=config["AZURE_SPEECH_REGION"],
)

# ✅ Option B: AAD token (production)
credential = DefaultAzureCredential()
token = credential.get_token("https://cognitiveservices.azure.com/.default")
auth_token = f"aad#{config['AZURE_SPEECH_RESOURCE_ID']}#{token.token}"
speech_config = speechsdk.SpeechConfig(auth_token=auth_token, region=config["AZURE_SPEECH_REGION"])

# ✅ Speech-to-text (from microphone)
speech_config.speech_recognition_language = "en-US"
audio_config = speechsdk.audio.AudioConfig(use_default_microphone=True)
recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)

result = recognizer.recognize_once()
if result.reason == speechsdk.ResultReason.RecognizedSpeech:
    print(f"Recognized: {result.text}")
elif result.reason == speechsdk.ResultReason.NoMatch:
    print("No speech could be recognized")

# ✅ Text-to-speech
speech_config.speech_synthesis_voice_name = "en-US-AriaNeural"
synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config)
result = synthesizer.speak_text_async("Hello from Azure Speech Service").get()

if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
    print("Speech synthesized successfully")

# ✅ Continuous recognition (for long audio)
def recognized_handler(evt):
    print(f"Recognized: {evt.result.text}")

recognizer.recognized.connect(recognized_handler)
recognizer.start_continuous_recognition()
```

❌ **DON'T:**
```python
# ❌ Don't hardcode speech key in source
speech_config = speechsdk.SpeechConfig(
    subscription="abc123def456",  # ❌ Hardcoded key
    region="eastus",
)

# ❌ Don't ignore result.reason (may silently fail)
result = recognizer.recognize_once()
print(result.text)  # ❌ Crashes if recognition failed
```

**Package:** `pip install azure-cognitiveservices-speech`

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization. Includes service-specific best practices for Cosmos DB, Azure SQL, and Storage.

### DB-1: Cosmos DB (azure-cosmos) Patterns (HIGH)
**Pattern:** Use `azure-cosmos` with AAD credentials. Handle partitioned containers properly.

✅ **DO:**
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

# ✅ Query with partition key
items = container.query_items(
    query="SELECT * FROM c WHERE c.category = @category",
    parameters=[{"name": "@category", "value": "electronics"}],
    partition_key="electronics",
)
for item in items:
    print(f"Item: {item['name']}")

# ✅ Point read (most efficient)
item = container.read_item(item="item-id", partition_key="partition-key-value")

# ✅ Create with partition key
container.create_item(body={
    "id": "item-id",
    "category": "electronics",  # Partition key
    "name": "Laptop",
})

# ✅ Upsert
container.upsert_item(body={
    "id": "item-id",
    "category": "electronics",
    "name": "Laptop Pro",
})
```

❌ **DON'T:**
```python
# ❌ Don't use primary key in samples
client = CosmosClient(
    url=config["COSMOS_ENDPOINT"],
    credential=config["COSMOS_PRIMARY_KEY"],  # ❌ Use DefaultAzureCredential
)

# ❌ Don't omit partition key in queries (cross-partition queries are expensive)
items = container.query_items(
    query="SELECT * FROM c",
    enable_cross_partition_query=True,  # ❌ Expensive, avoid in samples
)
```

---

### DB-2: Azure SQL with pyodbc/pymssql (HIGH)
**Pattern:** Use `pyodbc` with AAD token authentication. Use parameterized queries.

> **Tip:** For simpler setups, use `Authentication=Active Directory Default` in the connection string instead of manual token acquisition. This delegates authentication to the ODBC driver and works with managed identity, Azure CLI, etc.

✅ **DO (Recommended—connection string with Active Directory Default):**
```python
import pyodbc

# ✅ Simplest approach: let the ODBC driver handle AAD auth
connection = pyodbc.connect(
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={config['AZURE_SQL_SERVER']};"
    f"DATABASE={config['AZURE_SQL_DATABASE']};"
    f"Authentication=Active Directory Default;"
    f"Encrypt=yes;TrustServerCertificate=no;"
)

# ✅ Parameterized query
cursor = connection.cursor()
cursor.execute(
    "SELECT * FROM [Products] WHERE [Category] = ?",
    ("Electronics",),
)
rows = cursor.fetchall()

connection.close()
```

✅ **DO (Manual token—when you need token for other purposes):**
```python
import pyodbc
import struct
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Get AAD token
token = credential.get_token("https://database.windows.net/.default")

# ✅ Connect with AAD token (pyodbc struct format)
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

# ✅ Parameterized query
cursor = connection.cursor()
cursor.execute(
    "SELECT * FROM [Products] WHERE [Category] = ?",
    ("Electronics",),
)
rows = cursor.fetchall()

# ✅ Always close connection
connection.close()
```

❌ **DON'T:**
```python
# ❌ Don't use SQL Server authentication in samples
connection = pyodbc.connect(
    f"SERVER={server};DATABASE={database};UID={username};PWD={password}"
)

# ❌ Don't use string formatting for query values
cursor.execute(f"SELECT * FROM Products WHERE Category = '{category}'")  # SQL injection!
```

---

### DB-3: SQL Parameter Safety (Parameterized Queries) (MEDIUM)
**Pattern:** ALL query values must use parameterized queries. ALL dynamic SQL identifiers (table names, column names) must use bracket quoting.

✅ **DO:**
```python
# ✅ Parameterized values
cursor.execute(
    "SELECT [id], [name] FROM [Products] WHERE [category] = ? AND [price] > ?",
    (category, min_price),
)

# ✅ Bracket-quoted dynamic identifiers
table_name = config["TABLE_NAME"]
cursor.execute(
    f"SELECT [id], [name] FROM [{table_name}] WHERE [id] = ?",
    (item_id,),
)
```

❌ **DON'T:**
```python
# ❌ String interpolation for values (SQL injection!)
cursor.execute(f"SELECT * FROM Products WHERE category = '{category}'")

# ❌ Missing brackets on dynamic identifiers
cursor.execute(f"SELECT id, name FROM {table_name} WHERE id = ?", (item_id,))
```

---

### DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

✅ **DO (SQL—Batch Insert):**
```python
BATCH_SIZE = 10  # SQL Server max ~2100 params; 10 rows * 3 params = 30

items = [...]  # List of items

for i in range(0, len(items), BATCH_SIZE):
    batch = items[i : i + BATCH_SIZE]

    # Build VALUES clause
    placeholders = ", ".join(["(?, ?, ?)"] * len(batch))
    sql = f"INSERT INTO [Products] ([id], [name], [category]) VALUES {placeholders}"

    params = []
    for item in batch:
        params.extend([item["id"], item["name"], item["category"]])

    cursor.execute(sql, params)

connection.commit()

# Why batch size 10: SQL Server has ~2100 parameter limit.
# 10 rows * 3 params/row = 30 params, well under limit.
```

✅ **DO (Cosmos DB—Transactional Batch):**
```python
# ✅ Cosmos transactional batch (same partition key)
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

❌ **DON'T:**
```python
# ❌ Row-by-row INSERT (50 round trips for 50 items)
for item in items:
    cursor.execute(
        "INSERT INTO [Products] VALUES (?, ?, ?)",
        (item["id"], item["name"], item["category"]),
    )
    connection.commit()  # ❌ Commit per row is very slow
```

---

### DB-5: Azure Storage (azure-storage-blob) (MEDIUM)
**Pattern:** Use `azure-storage-blob`, `azure-storage-file-share`, `azure-data-tables` with `DefaultAzureCredential`.

✅ **DO:**
```python
from azure.storage.blob import BlobServiceClient
from azure.data.tables import TableClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Blob Storage
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

# ✅ Table Storage
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

### DB-6: SAS Token Fallback (MEDIUM)
**Pattern:** For local development or CI environments where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

✅ **DO:**
```python
import os
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential

# ✅ Try AAD first, fall back to SAS for local dev
sas_token = os.environ.get("AZURE_STORAGE_SAS_TOKEN")

if sas_token:
    # Local dev: SAS token
    blob_service_client = BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net{sas_token}"
    )
    print("Using SAS token authentication (local dev)")
else:
    # Production: AAD
    credential = DefaultAzureCredential()
    blob_service_client = BlobServiceClient(
        account_url=f"https://{account_name}.blob.core.windows.net",
        credential=credential,
    )
    print("Using DefaultAzureCredential (AAD)")
```

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for queues, topics, event ingestion, and event-driven architectures. Focus on reliable message handling, checkpoint management, and proper resource cleanup.

### MSG-1: Service Bus Patterns (HIGH)
**Pattern:** Use `azure-servicebus` with `DefaultAzureCredential`. Handle message completion, dead-letter queues.

✅ **DO:**
```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

with ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
) as client:
    # ✅ Send messages
    with client.get_queue_sender(queue_name="myqueue") as sender:
        messages = [
            ServiceBusMessage(body='{"orderId": 1, "amount": 100}'),
            ServiceBusMessage(body='{"orderId": 2, "amount": 200}'),
        ]
        sender.send_messages(messages)

    # ✅ Receive messages (MUST complete or abandon)
    with client.get_queue_receiver(queue_name="myqueue") as receiver:
        received_messages = receiver.receive_messages(
            max_message_count=10,
            max_wait_time=5,
        )
        for message in received_messages:
            try:
                print(f"Received: {str(message)}")
                receiver.complete_message(message)  # ✅ Mark as processed
            except Exception:
                receiver.abandon_message(message)  # ✅ Return to queue
```

❌ **DON'T:**
```python
# ❌ Don't use connection strings in samples
client = ServiceBusClient.from_connection_string(connection_string)

# ❌ Don't forget to complete/abandon messages
for message in received_messages:
    print(str(message))  # ❌ Message never completed (will reappear)
```

---

### MSG-2: Event Hubs Patterns (MEDIUM)
**Pattern:** Use `azure-eventhub` for ingestion, `azure-eventhub-checkpointstoreblob-aio` for async processing with checkpoint management.

✅ **DO:**
```python
from azure.eventhub import EventHubProducerClient, EventData
from azure.eventhub.aio import EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Send events (sync)
producer = EventHubProducerClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    eventhub_name="myeventhub",
    credential=credential,
)

with producer:
    batch = producer.create_batch()
    batch.add(EventData('{"temperature": 23.5}'))
    batch.add(EventData('{"temperature": 24.1}'))
    producer.send_batch(batch)

# ✅ Receive events with checkpoint store (async)
checkpoint_store = BlobCheckpointStore(
    blob_account_url=f"https://{storage_account}.blob.core.windows.net",
    container_name="eventhub-checkpoints",
    credential=credential,
)

consumer = EventHubConsumerClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    consumer_group="$Default",
    eventhub_name="myeventhub",
    credential=credential,
    checkpoint_store=checkpoint_store,
)


async def on_event(partition_context, event):
    print(f"Received: {event.body_as_str()}")
    await partition_context.update_checkpoint(event)  # ✅ Checkpoint


async with consumer:
    await consumer.receive(on_event=on_event)
```

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault. Covers secrets, keys, and certificates with AAD authentication.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `azure-keyvault-secrets`, `azure-keyvault-keys`, `azure-keyvault-certificates` with `DefaultAzureCredential`.

✅ **DO:**
```python
from azure.keyvault.secrets import SecretClient
from azure.keyvault.keys import KeyClient
from azure.keyvault.certificates import CertificateClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# ✅ Secrets
secret_client = SecretClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)

secret_client.set_secret("db-password", "P@ssw0rd123")
secret = secret_client.get_secret("db-password")
print(f"Secret value: {secret.value}")

# ✅ Keys (for encryption)
key_client = KeyClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)
key = key_client.create_rsa_key("my-encryption-key")
print(f"Key ID: {key.id}")

# ✅ Certificates
cert_client = CertificateClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)
cert = cert_client.get_certificate("my-cert")
print(f"Certificate thumbprint: {cert.properties.x509_thumbprint}")
```

❌ **DON'T:**
```python
# ❌ Don't hardcode secrets in samples
db_password = "P@ssw0rd123"  # ❌ Use Key Vault

# ❌ Don't use Key Vault for every sample (adds complexity)
# Use Key Vault when demonstrating secret management or when the sample
# explicitly covers secure configuration patterns.
```

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations across Azure data services. Includes embedding storage, distance calculations, and approximate nearest neighbor search.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Use `CAST(@param AS VECTOR(dimension))` for Azure SQL vector parameters. Serialize vectors as JSON strings.

✅ **DO (Azure SQL):**
```python
import json

embedding = [0.1, 0.2, 0.3, ...]  # 1536 floats from OpenAI

# ✅ Insert vector—cast to VECTOR type
cursor.execute(
    "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(? AS VECTOR(1536)))",
    (json.dumps(embedding),),
)

# ✅ Vector distance query
sql = """
    SELECT TOP (?)
        [id],
        [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC
"""
cursor.execute(sql, (k, json.dumps(search_embedding)))
```

✅ **DO (Cosmos DB Vector Search):**
```python
# ✅ Cosmos DB vector search
items = container.query_items(
    query="""
        SELECT TOP @k c.id, c.name,
            VectorDistance(c.embedding, @searchEmbedding) AS similarity
        FROM c
        ORDER BY VectorDistance(c.embedding, @searchEmbedding)
    """,
    parameters=[
        {"name": "@k", "value": 5},
        {"name": "@searchEmbedding", "value": search_embedding},
    ],
)
```

✅ **DO (Azure AI Search):**
```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
search_client = SearchClient(
    endpoint=config["SEARCH_ENDPOINT"],
    index_name="hotels-index",
    credential=credential,
)

# ✅ Use VectorizedQuery class (not raw dicts)
vector_query = VectorizedQuery(
    vector=search_embedding,
    k_nearest_neighbors=5,
    fields="descriptionVector",
)

results = search_client.search(
    search_text="luxury hotel",
    vector_queries=[vector_query],
)

for result in results:
    print(f"{result['name']} (score: {result['@search.score']})")
```

❌ **DON'T (Azure AI Search):**
```python
# ❌ Don't use raw dicts for vector queries—use VectorizedQuery class
results = search_client.search(
    search_text="luxury hotel",
    vector_queries=[{
        "kind": "vector",
        "vector": search_embedding,
        "k_nearest_neighbors": 5,
        "fields": "descriptionVector",
    }],
)
```

---

### VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires ≥1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

✅ **DO:**
```python
# Check row count before creating DiskANN index
cursor.execute(f"SELECT COUNT(*) FROM [{table_name}]")
row_count = cursor.fetchone()[0]

if row_count >= 1000:
    print(f"✅ {row_count} rows available. Creating DiskANN index...")

    cursor.execute(
        f"CREATE INDEX [ix_{table_name}_embedding_diskann] "
        f"ON [{table_name}] ([embedding]) "
        f"USING DiskANN"
    )
    connection.commit()
else:
    print(f"⚠️ Only {row_count} rows. DiskANN requires ≥1000. Using exact search.")

    # Fall back to VECTOR_DISTANCE (exact search)
    sql = f"""
        SELECT TOP (?) [id], [name],
            VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
        FROM [{table_name}]
        ORDER BY distance ASC
    """
```

❌ **DON'T:**
```python
# ❌ Create DiskANN index without checking row count
cursor.execute(f"CREATE INDEX ... ON [{table_name}] ([embedding]) USING DiskANN")
# Fails with: "DiskANN index requires at least 1000 rows"
```

---

## 8. Error Handling

**What this section covers:** Python exception patterns, AzureError hierarchy, contextual error messages, and troubleshooting guidance. Proper error handling prevents silent failures and helps users diagnose issues.

### ERR-1: Azure SDK Exception Hierarchy (MEDIUM)
**Pattern:** Use specific Azure SDK exceptions from `azure.core.exceptions`. Avoid bare `except:` blocks.

✅ **DO:**
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
    print("❌ Secret 'db-password' not found in Key Vault")
    print("  Create it with: az keyvault secret set --name db-password --value <value>")
except ClientAuthenticationError as e:
    print(f"❌ Authentication failed: {e.message}")
    print("  1. Run 'az login' to authenticate with Azure CLI")
    print("  2. Verify you have 'Key Vault Secrets User' role")
except HttpResponseError as e:
    print(f"❌ HTTP error {e.status_code}: {e.message}")
except AzureError as e:
    print(f"❌ Azure SDK error: {e}")
    raise
```

❌ **DON'T:**
```python
# ❌ Bare except catches everything including KeyboardInterrupt
try:
    secret = secret_client.get_secret("db-password")
except:  # ❌ Never use bare except
    print("Something went wrong")

# ❌ Catching Exception too broadly
try:
    secret = secret_client.get_secret("db-password")
except Exception as e:  # ❌ Loses Azure-specific error context
    print(str(e))
```

---

### ERR-2: Contextual Error Messages (MEDIUM)
**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

✅ **DO:**
```python
from azure.core.exceptions import ClientAuthenticationError

try:
    token = credential.get_token("https://storage.azure.com/.default")
except ClientAuthenticationError as e:
    print(f"❌ Failed to acquire Azure Storage token: {e.message}")
    print("\nTroubleshooting:")
    print("  1. Run 'az login' to authenticate with Azure CLI")
    print("  2. Verify you have the 'Storage Blob Data Contributor' role")
    print("  3. Check your Azure subscription is active")
    print("  4. Ensure firewall rules allow access from your IP")
    raise
```

---

### ERR-3: Async Exception Handling (MEDIUM)
**Pattern:** Handle exceptions in async code with proper cleanup in `finally` blocks or `async with`.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ No cleanup of async resources
async def process_messages():
    credential = DefaultAzureCredential()
    client = ServiceBusClient(namespace, credential=credential)
    receiver = client.get_queue_receiver(queue_name="myqueue")
    messages = await receiver.receive_messages(max_message_count=10)
    # ❌ receiver, client, credential never closed
```

---

## 9. Data Management

**What this section covers:** Sample data handling, pre-computed files, JSON loading, and data validation. Ensures samples run reliably on first execution without requiring data generation.

### DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

✅ **DO:**
```
repo/
├── data/
│   ├── products.json              # ✅ Sample data
│   ├── products-with-vectors.json # ✅ Pre-computed embeddings
├── src/
│   ├── main.py                    # Loads products-with-vectors.json
│   ├── generate_embeddings.py     # Generates embeddings (optional)
```

❌ **DON'T:**
```
repo/
├── data/
│   ├── products.json              # ✅ Raw data
│   ├── .gitignore                 # ❌ products-with-vectors.json gitignored
├── src/
│   ├── main.py                    # ❌ Fails: FileNotFoundError
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list**—not just the immediate project directory. Data files often live in sibling directories, parent directories, or shared `data/` folders.
> 2. **Trace the file path in code** relative to the working directory the code actually runs from (check `pyproject.toml` scripts, any `os.path.join` or `pathlib.Path` in source).
> 3. **Check for monorepo patterns**—in sample collections, data may be shared across multiple samples via a common parent directory.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR or repo at the path the code resolves to at runtime.

---

### DATA-2: JSON Data Loading (MEDIUM)
**Pattern:** Use `pathlib.Path` for file paths. Use `importlib.resources` for package data.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ Hard-coded relative paths that break based on cwd
with open("data/products.json") as f:
    data = json.load(f)

# ❌ No error message when file missing
data = json.load(open("products.json"))  # ❌ Also: unclosed file handle
```

---

## 10. Sample Hygiene

**What this section covers:** Repository hygiene, security, and governance. Covers .gitignore patterns, environment file protection, license files, and repository governance.

### HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and dependencies with comprehensive `.gitignore`.

✅ **DO:**
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

❌ **DON'T:**
```
repo/
├── .env                    # ❌ Live credentials committed!
├── __pycache__/            # ❌ Bytecode committed
├── .venv/                  # ❌ Virtual environment committed
├── dist/                   # ❌ Build artifacts committed
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed, you MUST verify the file is actually tracked by git:
> 1. **Check .gitignore**—look in the project directory AND all parent directories for `.gitignore` entries covering `.env`.
> 2. **Run `git ls-files .env`**—if it returns empty, the file is NOT tracked and is NOT a security issue.
> 3. A `.env` file that exists on disk but is gitignored is working as designed—developers create it locally from `.env.sample`.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked, or if no `.gitignore` exists at all.

---

### HYG-2: .env.sample (HIGH)
**Pattern:** Provide `.env.sample` with placeholder values. Never commit actual `.env` or any `.env.*` files (except `.env.sample` / `.env.example`).

✅ **DO:**
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

❌ **DON'T:**
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # ❌ Real subscription ID
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321       # ❌ Real tenant ID
```

---

### HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, functions, and imports. Commented-out code confuses users.

✅ **DO:**
```python
# Only import what you use
from azure.storage.blob import BlobServiceClient
from azure.identity import DefaultAzureCredential
```

❌ **DON'T:**
```python
# ❌ Commented-out code confuses users
# from azure.old_package import OldClient
#
# def old_implementation():
#     """This was the old way..."""
#     pass

from azure.storage.blob import BlobServiceClient
```

**Note:** Dead code severity upgraded from MEDIUM to HIGH—it significantly confuses users trying to learn from samples.

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── pyproject.toml
├── src/
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── pyproject.toml
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT**—look for `LICENSE`, `LICENSE.md`, `LICENSE.txt`, `license.txt`, or similar at the repository root.
> 2. **Check parent directories**—in monorepos and sample collections, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.
> 4. Only flag if NO license file exists at ANY level of the repo hierarchy above the sample.

---

### HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference or include governance files: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

✅ **DO:**
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

---

## 11. README & Documentation

**What this section covers:** Documentation quality, accuracy, and completeness. Covers expected output, troubleshooting, prerequisites, and setup instructions.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:
```bash
python main.py
```

You should see output similar to:

```
✅ Connected to Azure Blob Storage
✅ Container 'samples' created
✅ Uploaded blob 'sample.txt' (14 bytes)
✅ Downloaded blob content: "Hello, Azure!"
```

> Note: Exact output may vary based on your Azure environment.
```

❌ **DON'T:**
```markdown
## Expected Output

```
✅ Blob uploaded successfully  # ❌ Not actual output, fabricated
```
```

---

### DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

✅ **DO:**
```markdown
## Project Structure

- [`src/main.py`](./src/main.py)—Main entry point
- [`src/config.py`](./src/config.py)—Configuration loader
- [`infra/main.bicep`](./infra/main.bicep)—Infrastructure template
```

❌ **DON'T:**
```markdown
- [`src/main.py`](./Python/src/main.py)  # ❌ Wrong path
```

---

### DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure errors (auth failures, firewall, RBAC, prerequisites).

✅ **DO:**
```markdown
## Troubleshooting

### Authentication Errors

If you see "Failed to acquire token":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments (see Prerequisites)

### Firewall/Network Errors

If you see "Cannot connect to [service]":
- Check the service's firewall rules allow your IP
- Verify network security groups (NSGs) aren't blocking traffic

### Common Azure SDK Errors

- `ResourceNotFoundError`: Resource doesn't exist or you don't have access
- `ServiceRequestError`: Incorrect endpoint URL in .env
- `ClientAuthenticationError`: No authentication method available (run `az login`)

### Python-Specific Issues

- `ModuleNotFoundError: No module named 'azure'`: Run `pip install -r requirements.txt`
- `ImportError`: Check Python version meets minimum requirement (3.10+)
- `ssl.SSLCertVerificationError`: Update certificates: `pip install certifi`
```

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly (Azure subscription, CLI tools, role assignments, services).

✅ **DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Python**: Version 3.10 or later ([Download](https://www.python.org/downloads/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional)

### Azure Resources

This sample requires:
- **Azure Storage Account** with a blob container
- **Azure Key Vault** (if using secrets)
- **Azure OpenAI** deployment (if using AI features)

### Role Assignments

Your Azure identity needs these role assignments:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
- `Cognitive Services OpenAI User` on the Azure OpenAI resource
```

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions. Include virtual environment creation.

✅ **DO:**
```markdown
## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Azure-Samples/azure-storage-blob-samples.git
cd azure-storage-blob-samples/quickstart
```

### 2. Create virtual environment and install dependencies

```bash
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

pip install -r requirements.txt
```

### 3. Provision Azure resources

```bash
azd up
```

### 4. Run the sample

```bash
python main.py
```
```

---

### DOC-6: Python Version Strategy (LOW)
**Pattern:** Document minimum Python version in README, `pyproject.toml`, and any CI configuration.

✅ **DO:**
```toml
# pyproject.toml
[project]
requires-python = ">=3.10"
```

```markdown
# README.md

## Prerequisites

- **Python**: Version 3.10 or later (3.12+ recommended for best performance)
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

✅ **DO:**
```markdown
## Configuration

Copy `.env.sample` to `.env` and fill in your values:

```bash
cp .env.sample .env
```

Edit `.env` and replace placeholders:
- `AZURE_STORAGE_ACCOUNT_NAME`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `AZURE_KEYVAULT_URL`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI
```

❌ **DON'T:**
```markdown
## Configuration

Set environment variables:
- `AZURE_STORAGE_ACCOUNT_NAME=<your-storage-account>`  # ❌ How do I find this?
- `AZURE_KEYVAULT_URL=<your-keyvault-url>`            # ❌ What format?
```

---

## 12. Infrastructure (Bicep/Terraform)

**What this section covers:** Infrastructure-as-code patterns, Azure Verified Modules, parameter validation, API versioning, resource naming, and role assignments. Same IaC rules apply regardless of application language.

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

✅ **DO:**
```bicep
// ✅ Current AVM modules (check https://azure.github.io/Azure-Verified-Modules/)
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
  name: 'storage-deployment'
  params: {
    name: storageAccountName
    location: location
  }
}
```

❌ **DON'T:**
```bicep
module cognitiveServices 'br/public:avm/res/cognitive-services/account:0.7.1' = {
  // ❌ Outdated version (current is 1.0.1+)
}
```

---

### IaC-2: Bicep Parameter Validation (CRITICAL)
**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

✅ **DO:**
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

❌ **DON'T:**
```bicep
@description('Azure AD admin object ID')
param aadAdminObjectId string  // ❌ No validation, accepts empty string
```

---

### IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2023+). Avoid versions older than 2 years.

✅ **DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  // Current stable version
}
```

❌ **DON'T:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // ❌ 5+ years old
}
```

---

### IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities to access Azure resources.

✅ **DO:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
}

// Role assignment: App Service -> Storage Blob Data Contributor
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

**Common role IDs:**
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cosmos DB Account Reader Role: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

### IaC-5: Network Security (HIGH)
**Pattern:** For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

✅ **DO (Quickstart):**
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

### IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

✅ **DO:**
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_STORAGE_BLOB_ENDPOINT string = storageAccount.properties.primaryEndpoints.blob
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
```

---

### IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

✅ **DO:**
```bicep
@description('Prefix for all resources')
param resourcePrefix string = 'contoso'

@description('Environment (dev, test, prod)')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

// ✅ Consistent naming pattern
var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

❌ **DON'T:**
```bicep
// ❌ Inconsistent naming
var storageAccountName = 'mystorageaccount123'
var keyVaultName = 'kv-${uniqueString(resourceGroup().id)}'
```

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns, azure.yaml structure, service definitions, and hooks.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `azure.yaml` as missing or incomplete:
> 1. The `services`, `hooks`, and `host` fields in `azure.yaml` are **OPTIONAL**. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.
> 3. Only flag if required fields (`name`) are missing or if the file causes `azd` commands to fail.
> 4. **Check parent directories**—in monorepo/multi-sample layouts, `azure.yaml` often lives one or more levels ABOVE the language-specific project folder.

✅ **DO:**
```yaml
name: azure-storage-blob-sample
metadata:
  template: azure-storage-blob-sample@0.0.1

services:
  app:
    project: ./
    language: py
    host: appservice  # or: containerapp, function

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "❌ Not logged in. Run 'az login'" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "✅ Provisioning complete"
      echo "Storage Account: ${AZURE_STORAGE_ACCOUNT_NAME}"
      echo ""
      echo "Run 'pip install -r requirements.txt && python main.py' to test."

  predeploy:
    shell: sh
    run: |
      echo "Installing dependencies..."
      pip install -r requirements.txt
```

---

### AZD-2: Service Host Types (MEDIUM)
**Pattern:** Choose correct `host` type for Python applications.

✅ **DO:**
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

### AZD-3: Azure Functions Python v2 Model (MEDIUM)
**Pattern:** Azure Functions Python v2 uses a decorator-based programming model. Use the `@app` decorators instead of `function.json` files.

✅ **DO (v2 programming model—recommended):**
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

❌ **DON'T:**
```python
# ❌ v1 model (function.json + __init__.py)—avoid for new samples
# function.json — ❌ Don't use this pattern for new projects
# {
#   "bindings": [{"type": "httpTrigger", "direction": "in", "name": "req"}]
# }
```

> **Note:** The v2 programming model (`azure-functions>=1.18.0`) is the recommended approach for new Azure Functions Python projects. It replaces the v1 `function.json` + `__init__.py` pattern with decorators.

---

## 14. Async Patterns

**What this section covers:** Python-specific async/await patterns for Azure SDK clients. The Azure SDK for Python provides async versions of most clients under `.aio` submodules.

### ASYNC-1: Async Client Usage (HIGH)
**Pattern:** Import async clients from `.aio` submodules. Use `async with` for automatic cleanup. Always close `DefaultAzureCredential` in a `try/finally` block or use it as an async context manager.

✅ **DO (Preferred—async context manager for credential):**
```python
from azure.storage.blob.aio import BlobServiceClient
from azure.identity.aio import DefaultAzureCredential

async def upload_blob(account_name: str, data: bytes) -> None:
    """Upload a blob using async client with proper credential cleanup."""
    # ✅ DefaultAzureCredential supports async with (recommended)
    async with DefaultAzureCredential() as credential:
        async with BlobServiceClient(
            account_url=f"https://{account_name}.blob.core.windows.net",
            credential=credential,
        ) as blob_service_client:
            container_client = blob_service_client.get_container_client("mycontainer")
            blob_client = container_client.get_blob_client("myblob.txt")
            await blob_client.upload_blob(data, overwrite=True)
    # ✅ Both credential and client closed automatically
```

✅ **DO (Alternative—try/finally for credential cleanup):**
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
        await credential.close()  # ✅ Always close credential
```

❌ **DON'T:**
```python
# ❌ Using sync client inside async function
async def upload_blob(account_name: str, data: bytes) -> None:
    from azure.storage.blob import BlobServiceClient  # ❌ Sync client!
    client = BlobServiceClient(url, credential=credential)
    # This blocks the event loop!
```

**Why:** Mixing sync clients in async code blocks the event loop, defeating the purpose of async. Always use `.aio` clients in async contexts.

---

### ASYNC-2: Async Context Managers (MEDIUM)
**Pattern:** All async Azure SDK clients support `async with`. Use it for automatic resource cleanup.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ Manual close without try/finally
async def send_message(namespace: str, queue_name: str) -> None:
    credential = DefaultAzureCredential()
    client = ServiceBusClient(namespace, credential=credential)
    sender = client.get_queue_sender(queue_name=queue_name)
    await sender.send_messages(ServiceBusMessage("Hello"))
    await sender.close()
    await client.close()
    await credential.close()
    # ❌ If send_messages raises, close() never called
```

---

### ASYNC-3: asyncio.run() vs Event Loop (MEDIUM)
**Pattern:** Use `asyncio.run()` as the entry point. Don't manually manage event loops.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ Don't manually manage event loops
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()

# ❌ Don't use nest_asyncio hacks
import nest_asyncio
nest_asyncio.apply()
```

**Why:** `asyncio.run()` is the recommended entry point since Python 3.7. It creates a new event loop, runs the coroutine, and cleans up properly.

---

### ASYNC-4: Concurrent Operations with asyncio.gather (LOW)
**Pattern:** Use `asyncio.gather` for concurrent independent operations. Use `asyncio.Semaphore` to limit concurrency.

✅ **DO:**
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
            print(f"✅ Uploaded: {name}")

    await asyncio.gather(
        *(upload_one(name, data) for name, data in files)
    )
```

❌ **DON'T:**
```python
# ❌ Sequential uploads (slow for many files)
for name, data in files:
    blob_client = container_client.get_blob_client(name)
    await blob_client.upload_blob(data, overwrite=True)

# ❌ Unbounded concurrency (can overwhelm service)
await asyncio.gather(
    *(upload_one(name, data) for name, data in thousands_of_files)
)
```

---

### ASYNC-5: httpx Async Transport (LOW)
**Pattern:** Azure SDK can use `httpx` as an alternative async HTTP transport for environments that benefit from it (e.g., HTTP/2 support).

✅ **DO:**
```python
# ✅ Optional: Use httpx transport for HTTP/2 or custom transport needs
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

> **Note:** `httpx` transport is optional. The default `aiohttp` transport works well for most scenarios. Install: `pip install httpx`.

---

## 15. CI/CD & Testing

**What this section covers:** Continuous integration patterns, testing with pytest, linting, and build validation.

### CI-1: pytest and Testing (HIGH)
**Pattern:** Include tests with pytest. Run type checking and linting in CI.

✅ **DO:**
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

### CI-2: Type Checking with mypy or pyright (MEDIUM)
**Pattern:** Run static type checking in CI.

✅ **DO:**
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

❌ **DON'T:**
```python
# ❌ Skip type checking entirely
# No mypy or pyright configuration
# No type annotations
```

---

## Pre-Review Checklist (Comprehensive)

Use this comprehensive checklist before submitting an Azure SDK Python sample for review:

### 🔧 Project Setup
- [ ] Python version 3.10+ specified in `pyproject.toml` (`requires-python = ">=3.10"`)
- [ ] `pyproject.toml` has `authors`, `license`, `urls`, `requires-python`
- [ ] Every dependency is imported somewhere (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`azure-*`, not legacy)
- [ ] Using `openai` SDK with `AzureOpenAI` class
- [ ] Environment variables validated with descriptive errors
- [ ] `python-dotenv` used for `.env` loading
- [ ] Type hints on all public functions
- [ ] `requirements.txt` with pinned versions committed
- [ ] Only one package manager (pip OR poetry OR uv, not mixed)
- [ ] `pip-audit` passes (no critical/high CVEs)
- [ ] `.editorconfig` present (optional but recommended)

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `.env`, `__pycache__/`, `.venv/`, `dist/`, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed (subscription IDs, tenant IDs, connection strings, API keys)
- [ ] No build artifacts committed (`dist/`, `*.egg-info/`, `__pycache__/`)
- [ ] Dead code removed (unused files, imports, functions, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included
- [ ] Package legitimacy verified (all `azure-*` from Microsoft on PyPI)

### ☁️ Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Client options configured (retry policies, timeouts where applicable)
- [ ] Token refresh implemented for long-running operations (CRITICAL)
- [ ] Managed identity pattern documented in README
- [ ] Service-specific client patterns followed
- [ ] Pagination handled with `for` loops over `ItemPaged`/`AsyncItemPaged`
- [ ] Resource cleanup with `with`/`async with` or `try/finally`

### 🗄️ Data Services (if applicable)
- [ ] SQL: Uses `pyodbc` with AAD token authentication
- [ ] SQL: All dynamic identifiers use `[brackets]`
- [ ] SQL: Values parameterized (no string formatting)
- [ ] SQL/Cosmos: Batch operations for multiple rows (not row-by-row)
- [ ] Cosmos: Queries include partition key (avoid cross-partition)
- [ ] Cosmos: Uses `DefaultAzureCredential` (not primary key)
- [ ] Storage: Blob/Table client patterns followed
- [ ] Batch size rationale documented

### 🤖 AI Services (if applicable)
- [ ] Using `openai` SDK with `AzureOpenAI` class
- [ ] OpenAI client has `azure_ad_token_provider`, `timeout`, `max_retries`
- [ ] API versions documented with links
- [ ] Vector dimensions validated (match between model and storage)
- [ ] Pre-computed embedding data file committed to repo

### 💬 Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses checkpoint store for processing
- [ ] Connection strings avoided (prefer AAD auth)

### ❌ Error Handling
- [ ] Specific Azure exceptions caught (`AzureError` hierarchy, not bare `except`)
- [ ] Error messages are contextual and actionable
- [ ] Auth/network/RBAC errors include troubleshooting hints
- [ ] Async exceptions handled with cleanup in `finally`

### 🔄 Async Patterns (if applicable)
- [ ] Async clients from `.aio` submodules (not sync in async context)
- [ ] `async with` used for client lifecycle
- [ ] `asyncio.run()` as entry point (not manual event loop)
- [ ] `asyncio.Semaphore` for bounded concurrency

### 📄 Documentation
- [ ] README "Expected output" copy-pasted from real run (not fabricated)
- [ ] All internal folder/file links match actual filesystem paths
- [ ] Prerequisites section complete (subscription, CLI, role assignments)
- [ ] Troubleshooting section covers common Azure and Python errors
- [ ] Setup instructions include virtual environment creation
- [ ] Placeholder values have clear replacement instructions
- [ ] Python version documented in README and `pyproject.toml`

### 🏗️ Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated (`@minLength`, `@maxLength`, `@allowed`)
- [ ] API versions current (2023-2024+)
- [ ] Network security documented
- [ ] RBAC role assignments created for managed identities
- [ ] Resource naming follows CAF conventions
- [ ] Output values follow azd naming conventions (`AZURE_*`)
- [ ] `azure.yaml` present with correct `language: py`

### 🧪 CI/CD
- [ ] `pip-audit` scanning runs in CI
- [ ] Linting runs in CI (ruff or flake8)
- [ ] Type checking runs in CI (mypy or pyright)
- [ ] Tests pass with pytest
- [ ] Multiple Python versions tested (3.10, 3.12, 3.13)

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency
- **[`azure-sdk-typescript-sample-review`](../azure-sdk-typescript-sample-review/SKILL.md)**: TypeScript equivalent of this skill—shared IaC and azd patterns
- **[`azure-sdk-dotnet-sample-review`](../azure-sdk-dotnet-sample-review/SKILL.md)**: .NET 9/10 + Aspire
- **[`azure-sdk-java-sample-review`](../azure-sdk-java-sample-review/SKILL.md)**: Java 17/21 + Spring Boot
- **[`azure-sdk-go-sample-review`](../azure-sdk-go-sample-review/SKILL.md)**: Go 1.21+
- **[`azure-sdk-rust-sample-review`](../azure-sdk-rust-sample-review/SKILL.md)**: Rust 2021 edition

---

## Scope Note: Services Not Yet Covered

This skill focuses on the most commonly used Azure services in Python samples. The following services are not yet covered in detail, but the general patterns (authentication, client construction, error handling) apply:

- Azure Communication Services
- Azure Cache for Redis
- Azure Monitor (azure-monitor-query)
- Azure Container Registry
- Azure App Configuration (azure-appconfiguration)
- Azure SignalR Service
- Azure API Management
- Azure Machine Learning (azure-ai-ml)

For samples using these services, apply the core patterns from Sections 1–2 (Project Setup, Azure SDK Client Patterns) and reference service-specific documentation.

---

## Reference Links

Consolidation of all documentation links referenced throughout this skill:

### Azure SDK & Authentication
- [Azure SDK for Python](https://learn.microsoft.com/python/api/overview/azure/)
- [Azure SDK for Python—PyPI packages](https://learn.microsoft.com/azure/developer/python/sdk/azure-sdk-library-package-index)
- [DefaultAzureCredential](https://learn.microsoft.com/python/api/azure-identity/azure.identity.defaultazurecredential)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure Identity client library for Python](https://pypi.org/project/azure-identity/)

### API Versioning
- [Azure OpenAI API Versions](https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation)
- [Azure REST API Specifications](https://github.com/Azure/azure-rest-api-specs)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework—Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Security
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Azure Private Endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)
- [pip-audit](https://pypi.org/project/pip-audit/)

### Speech & AI Services
- [Azure Speech SDK for Python](https://learn.microsoft.com/azure/ai-services/speech-service/quickstarts/setup-platform?pivots=programming-language-python)
- [Azure AI Search Python SDK](https://learn.microsoft.com/python/api/azure-search-documents/)

### Python
- [Python Downloads](https://www.python.org/downloads/)
- [PEP 621—pyproject.toml metadata](https://peps.python.org/pep-0621/)
- [PEP 604—Union types X | Y](https://peps.python.org/pep-0604/)
- [PEP 585—Builtin generics](https://peps.python.org/pep-0585/)
- [Ruff Linter](https://docs.astral.sh/ruff/)
- [uv Package Manager](https://docs.astral.sh/uv/)
- [pytest](https://docs.pytest.org/)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK Python sample patterns** adapted from comprehensive TypeScript review patterns and generalized for the Python Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (11 rules): Credentials, phantom deps, CVE scanning, token refresh, AVM versions, parameter validation, .gitignore, fabricated output, broken links, broken auth, missing error handling
- **HIGH** (23 rules): Client construction, token management, managed identity, pagination, OpenAI config, database patterns, DiskANN guards, batch operations, RBAC, lock files, role assignments, pre-computed data, .env.sample, prerequisites, dead code, LICENSE file, resource naming, network security, async clients, CI testing, Service Bus, Key Vault, Azure Functions v2
- **MEDIUM** (32 rules): Client options, retry policies, identifier quoting, embeddings, error handling, JSON loading, troubleshooting, azd structure, type hints, version pinning, SAS fallback, dimensions, placeholder docs, resource cleanup, API versions, governance files, pyproject.toml metadata, env config, package legitimacy, async context managers, event loop entry, type checking, exception hierarchy, contextual errors, async errors, setup docs, host types, Speech SDK, uv package manager, logging configuration, AI Search vector queries, Active Directory Default
- **LOW** (9 rules): API version docs, .editorconfig/formatting, CONTRIBUTING.md, scope notes, Python version docs, concurrent gather, region availability, embedding dimensions config, httpx async transport

### Service Coverage
- **Core SDK**: Authentication, credentials, managed identities, client patterns, token management, pagination, resource cleanup, retry policies (generic + storage-specific), logging
- **Data**: Cosmos DB, Azure SQL (pyodbc + Active Directory Default), Storage (Blob/Table), batch operations
- **AI**: Azure OpenAI (embeddings, chat, images, configurable dimensions), Document Intelligence, vector dimensions, Speech SDK (STT/TTS)
- **Messaging**: Service Bus, Event Hubs, checkpoint management
- **Security**: Key Vault (secrets, keys, certificates)
- **Vector Search**: Azure SQL DiskANN, Cosmos DB, AI Search (VectorizedQuery)
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming, Azure Functions v2
- **Async**: Async clients (.aio), context managers (including credential), asyncio.run(), bounded concurrency, httpx transport
- **Tooling**: uv package manager, ruff format, pip-audit, logging configuration

Apply these patterns to ensure Azure SDK Python samples are **secure, accurate, maintainable, and ready for publication**.
