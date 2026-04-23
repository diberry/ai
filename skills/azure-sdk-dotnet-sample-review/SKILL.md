---
name: azure-sdk-dotnet-sample-review
description: Review Azure SDK .NET samples for publication readiness covering authentication, Track 2 patterns, security, Bicep, azd, and docs across Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, and AI services.
allowed-tools: []
---

UTILITY SKILL — Reviews Azure SDK .NET samples for Azure Samples org publication.

USE FOR: Reviewing .NET samples using Azure.* NuGet packages, Azure OpenAI, Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, AI Search, or Bicep/azd.

DO NOT USE FOR: General C# review without Azure SDK usage. Non-.NET Azure SDKs. Article quality.

## Triage

1. Track 2 packages — `Azure.*` not `Microsoft.Azure.*` (exception: `Microsoft.Azure.Cosmos`)
2. Auth — `DefaultAzureCredential`, no connection strings or API keys
3. Security — `.gitignore`, `.env.sample`, MIT LICENSE, `dotnet list package --vulnerable` clean
4. .NET — nullable enabled, net8.0/net9.0
5. Errors — `RequestFailedException` with status checks
6. Docs — README output from real runs, prerequisites
7. Infra — Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, critical CVEs, missing LICENSE, committed .env, Track 1 packages.

## References

- [references/project-setup.md](references/project-setup.md) — .csproj, deps, config
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) — Auth, credentials, pagination
- [references/ai-services.md](references/ai-services.md) — OpenAI, vector search
- [references/data-services.md](references/data-services.md) — Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) — Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) — Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) — Bicep, azd, Aspire, CI/CD, docs
