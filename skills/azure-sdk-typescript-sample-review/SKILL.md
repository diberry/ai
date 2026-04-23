---
name: azure-sdk-typescript-sample-review
description: Review Azure SDK TypeScript code samples for publication readiness covering authentication, Track 2 SDK patterns, security, Bicep infrastructure, azd integration, and documentation across Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, and AI services.
allowed-tools: []
---

UTILITY SKILL — Reviews Azure SDK TypeScript samples for Azure Samples org publication.

USE FOR: Reviewing TypeScript samples using @azure/* packages, Azure OpenAI, Cosmos DB, SQL, Storage, Service Bus, Event Hubs, Key Vault, AI Search, or Bicep/azd.

DO NOT USE FOR: General TypeScript review without Azure SDKs. Non-TypeScript Azure SDKs. Article quality.

## Triage

1. Track 2 packages — `@azure/*` not `azure-*`; `openai` not `@azure/openai`
2. Auth — `DefaultAzureCredential`, no connection strings or keys
3. Security — `.gitignore`, `.env.sample`, MIT LICENSE, `npm audit` clean
4. TypeScript — `strict: true`, ESM, `moduleResolution: "nodenext"`
5. Errors — `catch(err: unknown)` with type narrowing
6. Docs — README output from real runs, prerequisites
7. Infra — Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, critical CVEs, missing LICENSE, committed .env, Track 1 packages.

## References

- [references/project-setup.md](references/project-setup.md) — ESM, deps, env vars
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) — Auth, credentials, pagination
- [references/ai-services.md](references/ai-services.md) — OpenAI, vector search
- [references/data-services.md](references/data-services.md) — Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) — Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) — Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) — Bicep, azd, CI/CD, docs
