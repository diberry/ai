---
name: azure-sdk-java-sample-review
description: Review Azure SDK Java code samples for publication readiness covering authentication, Track 2 SDK patterns, security, Bicep infrastructure, azd integration, and documentation across Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, and AI services.
allowed-tools: []
---

UTILITY SKILL — Reviews Azure SDK Java samples for Azure Samples org publication.

USE FOR: Reviewing Java samples using com.azure.* packages, Azure OpenAI, Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, AI Search, Spring Boot, or Bicep/azd.

DO NOT USE FOR: General Java review without Azure SDK usage. Non-Java Azure SDKs. Article quality.

## Triage

1. Track 2 packages — `com.azure:*` not `com.microsoft.azure:*`; use `azure-sdk-bom`
2. Auth — `DefaultAzureCredential`, no connection strings or keys
3. Security — `.gitignore`, `.env.sample`, MIT LICENSE, no critical CVEs
4. Java — Java 17/21 LTS, proper exception handling
5. Errors — Specific exception types, try-with-resources
6. Docs — README output from real runs, prerequisites
7. Infra — Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, critical CVEs, missing LICENSE, committed .env, Track 1 packages.

## References

- [references/project-setup.md](references/project-setup.md) — Maven/Gradle, deps, BOM
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) — Auth, builders, pagination
- [references/ai-services.md](references/ai-services.md) — OpenAI, vector search
- [references/data-services.md](references/data-services.md) — Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) — Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) — Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) — Bicep, azd, Spring Boot, CI/CD, docs
