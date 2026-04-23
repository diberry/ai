---
name: azure-sdk-go-sample-review
description: Review Azure SDK Go code samples for publication readiness covering authentication, Track 2 SDK patterns, security, Bicep infrastructure, azd integration, and documentation across Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, and AI services.
allowed-tools: []
---

UTILITY SKILL -- Reviews Azure SDK Go samples for Azure Samples org publication.

USE FOR: Reviewing Go samples using github.com/Azure/azure-sdk-for-go/sdk/* packages, Azure OpenAI, Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, AI Search, or Bicep/azd.

DO NOT USE FOR: General Go review without Azure SDK usage. Non-Go Azure SDKs. Article quality.

## Triage

1. Track 2 packages -- `azure-sdk-for-go/sdk/*` not `azure-sdk-for-go/services/*`
2. Auth -- `azidentity.NewDefaultAzureCredential`, no connection strings or keys
3. Security -- `.gitignore`, `.env.sample`, MIT LICENSE, `govulncheck` clean
4. Go -- Go 1.22+, `go vet` passes, `go.sum` committed
5. Errors -- `if err != nil` checked, no discarded errors
6. Docs -- README output from real runs, prerequisites
7. Infra -- Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, CVEs, missing LICENSE, committed .env, legacy SDK packages.

## References

- [references/project-setup.md](references/project-setup.md) -- go.mod, deps, env vars
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) -- Auth, credentials, pagination
- [references/ai-services.md](references/ai-services.md) -- OpenAI, vector search
- [references/data-services.md](references/data-services.md) -- Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) -- Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) -- Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) -- Bicep, azd, CI/CD, Go idioms, docs
