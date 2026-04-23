---
name: azure-sdk-rust-sample-review
description: Review Azure SDK Rust code samples for publication readiness covering authentication, azure_* crate patterns, security, Bicep infrastructure, azd integration, and documentation across Storage, Key Vault, Cosmos DB, and AI services. SDK is evolving so some crate APIs may change.
allowed-tools: []
---

UTILITY SKILL -- Reviews Azure SDK Rust samples for Azure Samples org publication.

USE FOR: Reviewing Rust samples using azure_* crates, Azure OpenAI, Cosmos DB, Storage, Key Vault, AI Search, or Bicep/azd.

DO NOT USE FOR: General Rust review without Azure SDK usage. Non-Rust Azure SDKs. Article quality.

## Triage

1. SDK crates -- `azure_*` crates, not unofficial wrappers
2. Auth -- `DefaultAzureCredential`, no connection strings or keys
3. Security -- `.gitignore`, `.env.sample`, MIT LICENSE, `cargo audit` clean
4. Rust -- edition 2021, `cargo clippy` passes, `Cargo.lock` committed
5. Errors -- `Result<T, E>` and `?` operator, no `unwrap()` in main paths
6. Docs -- README output from real runs, prerequisites
7. Infra -- Bicep param validation, current AVM versions

Auto-reject: hardcoded secrets, missing auth, broken imports, CVEs, missing LICENSE, committed .env, panics in sample code.

## References

- [references/project-setup.md](references/project-setup.md) -- Cargo.toml, deps, env
- [references/azure-sdk-clients.md](references/azure-sdk-clients.md) -- Auth, credentials, pagination
- [references/ai-services.md](references/ai-services.md) -- OpenAI, vector search
- [references/data-services.md](references/data-services.md) -- Cosmos, SQL, Storage
- [references/messaging-keyvault.md](references/messaging-keyvault.md) -- Service Bus, Event Hubs, KV
- [references/error-handling-hygiene.md](references/error-handling-hygiene.md) -- Errors, data, hygiene
- [references/infrastructure-cicd.md](references/infrastructure-cicd.md) -- Bicep, azd, CI/CD, Rust idioms, docs
