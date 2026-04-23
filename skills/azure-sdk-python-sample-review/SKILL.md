---
name: azure-sdk-python-sample-review
description: Review Azure SDK Python code samples for publication readiness covering authentication, Track 2 SDK patterns, security, Bicep infrastructure, azd integration, and documentation across Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, and AI services.
allowed-tools: []
---

UTILITY SKILL -- Reviews Azure SDK Python samples for Azure Samples org publication.

USE FOR: Reviewing Python samples using azure-* PyPI packages, Azure OpenAI, Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, AI Search, or Bicep/azd.

DO NOT USE FOR: General Python review without Azure SDK usage. Non-Python Azure SDKs. Article quality.

## Triage

1. Track 2 packages -- `azure-*` Track 2 PyPI packages, not legacy versions
2. Auth -- `DefaultAzureCredential`, no connection strings or keys
3. Security -- `.gitignore`, `.env.sample`, MIT LICENSE, `pip-audit` clean
4. Python -- 3.10+, type hints, `with`/`async with` for clients
5. Errors -- `try/except` with specific Azure exceptions, no bare `except:`
6. Docs -- README output from real runs, prerequisites
7. Infra -- Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, critical CVEs, missing LICENSE, committed .env, Track 1 packages.

## References

- [references/project-setup.md](references/project-setup.md) -- pyproject.toml, deps, venv
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) -- Auth, credentials, pagination
- [references/ai-services.md](references/ai-services.md) -- OpenAI, vector search
- [references/data-services.md](references/data-services.md) -- Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) -- Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) -- Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) -- Bicep, azd, CI/CD, async patterns, docs