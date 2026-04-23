---
name: "azure-sdk-typescript-sample-review"
description: "Comprehensive review checklist for Azure SDK TypeScript code samples covering project setup, Azure SDK client patterns, authentication, data services (Cosmos DB, SQL, Storage, Tables), messaging (Service Bus, Event Hubs), AI services (OpenAI, Document Intelligence, Speech), Key Vault, infrastructure, documentation, and sample hygiene. Earned from production sample reviews."
domain: "code-review"
confidence: "high"
source: "earned -- distilled from multi-agent reviews of Azure SDK TypeScript samples including Azure SQL vector search (29 findings), plus generalized patterns across Azure SDK ecosystem"
---

## Context

Use this skill when reviewing **TypeScript code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general TypeScript review (`typescript-review` skill) — it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (Track 2 `@azure/*` packages, client construction, pipeline options)
- **Authentication patterns** (`DefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Service Bus, Key Vault, AI services)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)

This skill captures patterns and anti-patterns discovered during comprehensive reviews of Azure SDK TypeScript samples, particularly the Azure SQL vector search sample (29 findings) plus generalized patterns across the Azure SDK ecosystem.

**Total rules: 68** (10 CRITICAL, 22 HIGH, 29 MEDIUM, 7 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **Package.json**: Uses Track 2 Azure SDK packages (`@azure/*`, not `azure-*`)
- [ ] **Authentication**: Uses `DefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `.env`, `.env.*`, `node_modules/`, `dist/`
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: `npm audit` passes with no critical/high vulnerabilities
- [ ] **TypeScript**: `strict: true` in tsconfig.json
- [ ] **Error handling**: `catch` blocks present with type-safe error narrowing
- [ ] **Resource cleanup**: Clients properly closed/disposed (try/finally blocks)
- [ ] **Lock file**: package-lock.json committed (not .gitignored)
- [ ] **No mixed package managers**: Only npm OR yarn OR pnpm (not multiple)
- [ ] **Imports work**: No broken imports, all dependencies installed
- [ ] **Build succeeds**: `npm run build` completes without errors
- [ ] **Sample runs**: `npm start` executes without crashes

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets** — Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication** — No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling** — Uncaught promises, no try/catch blocks, silent failures
4. **Broken imports** — Missing dependencies, incorrect import paths, package not found errors
5. **Security vulnerabilities** — `npm audit` shows critical or high CVEs
6. **Missing LICENSE** — No LICENSE file (MIT required for Azure Samples org)
7. **.env file committed** — Live credentials in version control
8. **Track 1 packages** — Uses legacy `azure-*` packages instead of `@azure/*`

---

## 1. Project Setup & Configuration

**What this section covers:** Package structure, TypeScript configuration, dependency management, environment variables, and Node.js runtime settings. These foundational patterns ensure samples build correctly and run reliably across environments.

### PS-1: ESM Configuration (HIGH)
**Pattern:** For modern Node.js projects (20.12+), use native ESM with proper TypeScript configuration.

✅ **DO:**
```json
// package.json
{
  "type": "module",
  "scripts": {
    "start": "npx tsx src/index.ts",
    "start:env": "node --env-file=.env --import tsx src/index.ts"
  },
  "engines": {
    "node": ">=20.12.0"
  }
}

// tsconfig.json
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "target": "ES2022"
  }
}

// src/index.ts
import { getConfig } from './config.js';  // ✅ .js extension for ESM
import { BlobServiceClient } from '@azure/storage-blob';
```

❌ **DON'T:**
```json
// tsconfig.json
{
  "compilerOptions": {
    "module": "commonjs",          // ❌ Wrong for ESM
    "moduleResolution": "node"     // ❌ Wrong for ESM
  }
}
```

**Why:** ESM is the future of Node.js. `moduleResolution: "node"` doesn't understand `"type": "module"`. Note: `--loader tsx/esm` is deprecated — use `npx tsx` directly or `--import tsx` to register the loader.

---

### PS-2: Package Metadata (MEDIUM)
**Pattern:** All sample packages must include complete metadata for discoverability and maintenance.

✅ **DO:**
```json
{
  "name": "azure-storage-blob-quickstart",
  "version": "1.0.0",
  "description": "Upload and download blobs using Azure Blob Storage SDK",
  "author": "Microsoft Corporation",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/Azure-Samples/azure-storage-blob-samples"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

❌ **DON'T:**
```json
{
  "name": "my-sample",
  "version": "1.0.0"
  // ❌ Missing author, license, repository, engines
}
```

**Note:** The `engines` field documents the minimum Node.js version. Update README to match.

---

### PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`@azure/*`).

✅ **DO:**
```json
// package.json
{
  "dependencies": {
    "@azure/storage-blob": "^12.25.0",    // ✅ Track 2, used in src/index.ts
    "@azure/identity": "^4.5.0",          // ✅ Used for auth
    "@azure/keyvault-secrets": "^4.9.0",  // ✅ Used in src/secrets.ts
    "openai": "^4.73.0"                   // ✅ For AzureOpenAI class
  },
  "devDependencies": {
    "tsx": "^4.19.2",                     // ✅ Used in npm scripts
    "typescript": "^5.7.2"                // ✅ Used for type checking
  }
}
```

❌ **DON'T:**
```json
{
  "dependencies": {
    "azure-storage": "^2.10.7",            // ❌ Track 1 (legacy)
    "@azure/openai": "^1.0.0",             // ❌ Retired (use openai SDK's AzureOpenAI)
    "dotenv": "^16.0.0",                   // ❌ Not imported (using --env-file)
    "@azure/service-bus": "^7.9.0",        // ❌ Listed but never imported
    "@types/jsonwebtoken": "^9.0.0"        // ❌ jsonwebtoken not used
  }
}
```

**Production Issue:** Vector search sample had `@azure/openai` (retired), `dotenv` (unused), `@types/jsonwebtoken`, `@types/uuid` (never imported).

---

### PS-4: Azure SDK Package Naming (HIGH)
**Pattern:** Use Track 2 packages (`@azure/*` scoped) not Track 1 legacy packages (`azure-*`).

✅ **DO (Track 2):**
```typescript
// ✅ Current generation Azure SDK packages
import { BlobServiceClient } from '@azure/storage-blob';
import { SecretClient } from '@azure/keyvault-secrets';
import { ServiceBusClient } from '@azure/service-bus';
import { CosmosClient } from '@azure/cosmos';
import { TableClient } from '@azure/data-tables';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Azure OpenAI via openai SDK (not @azure/openai)
import { AzureOpenAI } from 'openai';
```

❌ **DON'T (Track 1 Legacy):**
```typescript
// ❌ Track 1 packages (legacy, avoid in new samples)
import * as azure from 'azure-storage';           // Use @azure/storage-blob
import * as KeyVault from 'azure-keyvault';      // Use @azure/keyvault-secrets
import * as serviceBus from 'azure-sb';          // Use @azure/service-bus

// ❌ Retired package
import { OpenAIClient } from '@azure/openai';    // Use openai SDK's AzureOpenAI
```

**Why:** Track 2 SDKs (`@azure/*`) are current generation with better TypeScript support, consistent APIs, and active maintenance. Track 1 (`azure-*`) is legacy. The `@azure/openai` package was retired — use the official `openai` SDK with the `AzureOpenAI` class.

---

### PS-5: Environment Variables (MEDIUM)
**Pattern:** Use Node.js 20.12+ native `--env-file` instead of `dotenv`. Validate all required variables with descriptive errors. Note: `--env-file` requires Node.js 20.12 or later (not available in earlier 20.x releases).

✅ **DO:**
```typescript
// src/config.ts
export function getConfig() {
  const required = {
    AZURE_STORAGE_ACCOUNT_NAME: process.env.AZURE_STORAGE_ACCOUNT_NAME,
    AZURE_KEYVAULT_URL: process.env.AZURE_KEYVAULT_URL,
    AZURE_SERVICE_BUS_NAMESPACE: process.env.AZURE_SERVICE_BUS_NAMESPACE,
  };

  const missing = Object.entries(required)
    .filter(([_, value]) => !value)
    .map(([key]) => key);

  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(', ')}\n` +
      `Create a .env file with these values or set them in your environment.\n` +
      `See .env.sample for required variables.`
    );
  }

  return required as Record<keyof typeof required, string>;
}

// package.json
{
  "scripts": {
    "start": "node --env-file=.env --import tsx src/index.ts"
  }
}
```

❌ **DON'T:**
```typescript
// ❌ Don't use dotenv in Node.js 20.12+
import 'dotenv/config';

// ❌ Don't silently fail on missing vars
const config = {
  storageAccount: process.env.AZURE_STORAGE_ACCOUNT_NAME || 'devstoreaccount1',
};
```

---

### PS-6: TypeScript Strict Flags (MEDIUM)
**Pattern:** Enable strict mode. Document workarounds with tracking links to upstream issues.

✅ **DO:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "strictPropertyInitialization": false  // Workaround for @azure/msal-node bug
    // See: https://github.com/AzureAD/microsoft-authentication-library-for-js/issues/1234
  }
}
```

❌ **DON'T:**
```json
{
  "compilerOptions": {
    "strict": true,
    "strictPropertyInitialization": false  // ❌ No explanation or tracking link
  }
}
```

**Note:** `strict: true` catches many common bugs at compile time. Upgraded from LOW to MEDIUM severity.

---

### PS-7: .editorconfig (LOW)
**Pattern:** Include `.editorconfig` for consistent formatting across editors.

✅ **DO:**
```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.{ts,tsx,js,jsx,json}]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

---

### PS-8: Lock File Management (HIGH)
**Pattern:** Commit lock files. Use only one package manager per project.

✅ **DO:**
```gitignore
# .gitignore
node_modules/
dist/
.env
.env.*
!.env.sample

# ✅ Lock file is COMMITTED (not in .gitignore)
```

```json
// package.json - pick ONE package manager
{
  "packageManager": "npm@10.5.0"
}
```

❌ **DON'T:**
```gitignore
# ❌ Don't ignore lock files
package-lock.json
yarn.lock
pnpm-lock.yaml
```

```
repo/
├── package-lock.json  # ❌ Mixed package managers
├── yarn.lock          # ❌ Don't mix npm and yarn
```

**Why:** Lock files ensure reproducible builds. Mixed package managers cause version conflicts.

---

### PS-9: CVE Scanning (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. `npm audit` must pass with no critical/high issues.

✅ **DO:**
```bash
# Before submitting sample
npm audit

# If vulnerabilities found
npm audit fix

# Check results
npm audit --audit-level=high
```

```json
// package.json - add audit check to CI
{
  "scripts": {
    "test": "npm audit --audit-level=high && npm run typecheck"
  }
}
```

❌ **DON'T:**
```bash
# ❌ Don't ignore audit warnings
npm audit
# found 3 high severity vulnerabilities
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Package Legitimacy Check (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `@azure/*` scope. Watch for typosquatting.

✅ **DO:**
```json
{
  "dependencies": {
    "@azure/storage-blob": "^12.25.0",  // ✅ Official package
    "@azure/cosmos": "^4.2.0",          // ✅ Official package
    "@azure/identity": "^4.5.0"         // ✅ Official package
  }
}
```

❌ **DON'T:**
```json
{
  "dependencies": {
    "azure-storage-blob": "^12.0.0",  // ❌ Typosquatting (missing @)
    "@azzure/storage": "^1.0.0",      // ❌ Typosquatting (azzure)
    "azur-storage": "^1.0.0"          // ❌ Not official package
  }
}
```

**Check:** All Azure SDK packages should be scoped `@azure/*` and published by Microsoft. Verify on npmjs.com.

---

### PS-11: Version Pinning Strategy (MEDIUM)
**Pattern:** Use caret ranges (`^`) for flexibility, exact versions for critical reproducibility.

✅ **DO:**
```json
{
  "dependencies": {
    "@azure/storage-blob": "^12.25.0",  // ✅ Caret range (recommended)
    "@azure/identity": "^4.5.0"
  },
  "devDependencies": {
    "typescript": "5.7.2"  // ✅ Exact version for tooling OK
  }
}
```

**Guidance:**
- **Samples for learning**: Use caret ranges (`^1.2.3`) — allows minor/patch updates
- **Samples for production reference**: Consider exact versions (`1.2.3`) for reproducibility
- **Lock file committed**: Ensures exact versions installed regardless

---

## 2. Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

### AZ-1: Client Construction with Credentials (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

✅ **DO:**
```typescript
import { DefaultAzureCredential } from '@azure/identity';
import { BlobServiceClient } from '@azure/storage-blob';
import { SecretClient } from '@azure/keyvault-secrets';
import { ServiceBusClient } from '@azure/service-bus';
import { CosmosClient } from '@azure/cosmos';

// ✅ Cache credential instance
const credential = new DefaultAzureCredential();

// ✅ Storage Blob
const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  credential
);

// ✅ Key Vault
const secretClient = new SecretClient(
  config.AZURE_KEYVAULT_URL,
  credential
);

// ✅ Service Bus
const serviceBusClient = new ServiceBusClient(
  `${namespace}.servicebus.windows.net`,
  credential
);

// ✅ Cosmos DB
const cosmosClient = new CosmosClient({
  endpoint: config.COSMOS_ENDPOINT,
  aadCredentials: credential
});
```

❌ **DON'T:**
```typescript
// ❌ Don't use connection strings in samples (prefer AAD auth)
const blobServiceClient = BlobServiceClient.fromConnectionString(connectionString);

// ❌ Don't use account keys
const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  new StorageSharedKeyCredential(accountName, accountKey)
);

// ❌ Don't recreate credential for each client
const blobClient = new BlobServiceClient(url, new DefaultAzureCredential());
const secretClient = new SecretClient(url, new DefaultAzureCredential());
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code, etc.) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

### AZ-2: Client Options and Retry Policies (MEDIUM)
**Pattern:** Production samples SHOULD configure retry policies, timeouts, and logging. Quickstarts MAY rely on Azure SDK built-in defaults (3 retries with exponential backoff).

✅ **DO (production samples):**
```typescript
import { BlobServiceClient } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  credential,
  {
    retryOptions: {
      maxRetries: 3,
      retryDelayInMs: 1000,
      maxRetryDelayInMs: 30000,
    },
    // For debugging: enable logging
    // loggingOptions: { 
    //   logger: console.log,
    //   allowedHeaderNames: ['x-ms-request-id'],
    // },
  }
);
```

✅ **OK (quickstarts — SDK defaults are fine):**
```typescript
// Azure SDK clients have built-in retry (3 retries, exponential backoff).
// Quickstarts may omit explicit retry configuration for simplicity.
const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  credential
);
```

❌ **DON'T:**
```typescript
// ❌ Don't disable retries for production samples
const blobServiceClient = new BlobServiceClient(url, credential, {
  retryOptions: { maxRetries: 0 },  // ❌ Disables built-in retry
});
```

**Note:** Azure SDK clients include built-in retry with exponential backoff by default. Quickstarts don't need explicit retry configuration — only flag this for production-grade samples or when retry is explicitly disabled.

---

### AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```typescript
import { DefaultAzureCredential, ManagedIdentityCredential } from '@azure/identity';

// ✅ For samples: DefaultAzureCredential (works locally + cloud)
const credential = new DefaultAzureCredential();

// ✅ For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
const credential = new ManagedIdentityCredential();

// ✅ User-assigned (when multiple identities needed, or cross-resource-group access)
const credential = new ManagedIdentityCredential({
  clientId: process.env.AZURE_CLIENT_ID  // From azd outputs or env vars
});

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
// > automatically use the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App / Function App has a managed identity
// > assigned with appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```typescript
// ❌ Don't hardcode service principal credentials in samples
const credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources, or identity needs to outlive the resource.

---

### AZ-4: Token Management for Non-SDK Clients (CRITICAL)
**Pattern:** For services without official SDK (SQL, custom APIs), get tokens with `getToken()`. Tokens expire after ~1 hour — implement refresh logic for long-running samples.

✅ **DO:**
```typescript
import { DefaultAzureCredential, TokenCredential, AccessToken } from '@azure/identity';

const credential = new DefaultAzureCredential();

// ✅ Azure SQL - Get token with expiration tracking
async function getAzureSqlToken(credential: TokenCredential): Promise<AccessToken> {
  return await credential.getToken('https://database.windows.net/.default');
}

// ✅ Implement token refresh for long-running operations
let tokenResponse = await getAzureSqlToken(credential);
const tokenExpiresAt = tokenResponse.expiresOnTimestamp;

// Check expiration before use (refresh if < 5 minutes remaining)
function isTokenExpiringSoon(expiresOnTimestamp: number): boolean {
  const now = Date.now();
  const fiveMinutesInMs = 5 * 60 * 1000;
  return (expiresOnTimestamp - now) < fiveMinutesInMs;
}

// Before long operation
if (isTokenExpiringSoon(tokenExpiresAt)) {
  console.log('Token expiring soon, refreshing...');
  tokenResponse = await getAzureSqlToken(credential);
}

// Use token in connection config...
```

❌ **DON'T:**
```typescript
// ❌ CRITICAL: Don't acquire token once and use for hours
const token = await credential.getToken('https://database.windows.net/.default');
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

**Why:** Azure tokens expire after approximately 1 hour. Samples processing large datasets or running long operations MUST refresh tokens before expiration. Failure to do so causes authentication failures in production.

**Production Issue:** Vector search sample acquired Azure SQL token once without refresh logic, causing failures in long-running demos.

---

### AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI, include managed identity for cloud.

✅ **DO:**
```typescript
import { DefaultAzureCredential, DefaultAzureCredentialOptions } from '@azure/identity';

// ✅ For CI/CD environments (no interactive prompts)
const credential = new DefaultAzureCredential({
  excludeInteractiveBrowserCredential: true,
  excludeWorkloadIdentityCredential: false,  // Keep for K8s
  excludeManagedIdentityCredential: false,   // Keep for Azure
});

// ✅ For local development (include browser auth)
const credential = new DefaultAzureCredential({
  excludeInteractiveBrowserCredential: false,
});

// ✅ Document the credential chain in README
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
// > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > 2. Workload identity (Azure Kubernetes Service)
// > 3. Managed identity (App Service, Functions, Container Apps)
// > 4. Azure Developer CLI (`azd auth login`)
// > 5. Azure CLI (`az login`)
// > 6. Azure PowerShell
// > 7. Interactive browser (local development only)
```

---

### AZ-6: Resource Cleanup Patterns (MEDIUM)
**Pattern:** Samples must properly close/dispose clients that hold connections (e.g., Service Bus, Event Hubs). Use `try/finally` blocks. Note: Azure SDK JS/TS clients do NOT support `Symbol.asyncDispose` / `await using` — always use manual cleanup. Some clients (e.g., `BlobServiceClient`, `SecretClient`) are stateless HTTP clients and don't require explicit cleanup.

✅ **DO:**
```typescript
import { ServiceBusClient } from '@azure/service-bus';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Use try/finally for clients that hold connections
const credential = new DefaultAzureCredential();
const client = new ServiceBusClient(namespace, credential);

try {
  const sender = client.createSender('myqueue');
  await sender.sendMessages({ body: 'Hello' });
  await sender.close();
} finally {
  await client.close();  // ✅ Always cleanup
}

// ✅ Stateless HTTP clients (Storage, Key Vault) don't need explicit cleanup
const blobServiceClient = new BlobServiceClient(url, credential);
// No .close() needed — stateless HTTP client
```

❌ **DON'T:**
```typescript
// ❌ Don't forget to close clients
const client = new ServiceBusClient(namespace, credential);
const sender = client.createSender('myqueue');
await sender.sendMessages({ body: 'Hello' });
// ❌ Client never closed (resource leak)
```

---

### AZ-8: AbortController and Cancellation (MEDIUM)
**Pattern:** Long-running SDK operations should support cancellation via `AbortSignal`. Use `AbortController` to implement timeouts and user-initiated cancellation.

✅ **DO:**
```typescript
import { BlobServiceClient } from '@azure/storage-blob';

// ✅ Timeout-based cancellation
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);  // 30s timeout

try {
  const containerClient = blobServiceClient.getContainerClient('mycontainer');
  const blockBlobClient = containerClient.getBlockBlobClient('largefile.zip');
  
  await blockBlobClient.upload(data, data.length, {
    abortSignal: controller.signal,
  });
} finally {
  clearTimeout(timeout);
}

// ✅ User-initiated cancellation (e.g., in a CLI tool)
const controller = new AbortController();
process.on('SIGINT', () => {
  console.log('Cancelling operation...');
  controller.abort();
});

await longRunningOperation({ abortSignal: controller.signal });
```

**Note:** Most Azure SDK operations accept `{ abortSignal }` in their options parameter. Pass `AbortController.signal` to enable cancellation.

---

### AZ-9: Diagnostic Logging with @azure/logger (MEDIUM)
**Pattern:** Use `@azure/logger` for SDK diagnostic output. Enable during development and troubleshooting.

✅ **DO:**
```typescript
import { setLogLevel } from '@azure/logger';

// ✅ Enable SDK logging for debugging
setLogLevel('info');     // Options: 'verbose', 'info', 'warning', 'error'

// ✅ Selectively enable per-client logging
import { setLogLevel, AzureLogger } from '@azure/logger';
AzureLogger.log = (...args) => {
  console.log('[Azure SDK]', ...args);
};
setLogLevel('warning');

// ✅ In production samples, show how to enable but leave commented out
// import { setLogLevel } from '@azure/logger';
// setLogLevel('info');  // Uncomment for SDK diagnostic logging
```

❌ **DON'T:**
```typescript
// ❌ Don't use console.log to manually trace SDK calls
console.log('About to call blob upload...');
await blockBlobClient.upload(data, data.length);
console.log('Upload done');
```

**Note:** `@azure/logger` integrates with all `@azure/*` SDK packages. Add `"@azure/logger"` to devDependencies when including diagnostic patterns.
**Pattern:** Use `for await...of` loops for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```typescript
import { BlobServiceClient } from '@azure/storage-blob';

// ✅ Blob Storage - iterate all pages
const containerClient = blobServiceClient.getContainerClient('mycontainer');
for await (const blob of containerClient.listBlobsFlat()) {
  console.log(`Blob: ${blob.name}`);
}

// ✅ Cosmos DB - iterate all pages
const { resources: items } = await container.items
  .query('SELECT * FROM c')
  .fetchAll();  // Or use .fetchNext() for manual paging

// ✅ Service Bus - receive multiple batches
const receiver = client.createReceiver('myqueue');
let hasMoreMessages = true;

while (hasMoreMessages) {
  const messages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });
  if (messages.length === 0) {
    hasMoreMessages = false;
    break;
  }
  
  for (const message of messages) {
    console.log(`Received: ${message.body}`);
    await receiver.completeMessage(message);
  }
}
```

❌ **DON'T:**
```typescript
// ❌ CRITICAL BUG: Only gets first page
const containerClient = blobServiceClient.getContainerClient('mycontainer');
const blobs = [];
for await (const blob of containerClient.listBlobsFlat()) {
  blobs.push(blob);
  if (blobs.length >= 10) break;  // ❌ Stops after 10, may miss thousands
}
```

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns, API versioning, embeddings, chat completions, and document analysis. Focus on the official `openai` SDK for Azure OpenAI (not the retired `@azure/openai`).

### AI-1: Azure OpenAI Client Configuration (HIGH)
**Pattern:** Use `openai` SDK's `AzureOpenAI` class (not retired `@azure/openai`). Configure `timeout`, `maxRetries`, and `azureADTokenProvider`.

✅ **DO:**
```typescript
import { AzureOpenAI } from 'openai';
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';

const credential = new DefaultAzureCredential();
const azureADTokenProvider = getBearerTokenProvider(
  credential,
  'https://cognitiveservices.azure.com/.default'
);

const client = new AzureOpenAI({
  azureADTokenProvider,
  endpoint: config.AZURE_OPENAI_ENDPOINT,
  apiVersion: '2024-10-21',  // See: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
  timeout: 30_000,            // 30 second timeout
  maxRetries: 3,              // Retry up to 3 times
});

// ✅ Chat completion
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello!' }],
});

// ✅ Embeddings
const embeddingResponse = await client.embeddings.create({
  model: 'text-embedding-3-small',
  input: 'Sample text to embed',
});

// ✅ Image generation
const imageResponse = await client.images.generate({
  model: 'dall-e-3',
  prompt: 'A photo of a cat',
  n: 1,
  size: '1024x1024',
});
```

❌ **DON'T:**
```typescript
// ❌ Don't use retired @azure/openai package
import { OpenAIClient, AzureKeyCredential } from '@azure/openai';

// ❌ Don't omit timeout/retries
const client = new AzureOpenAI({
  azureADTokenProvider,
  // Missing timeout, maxRetries
});

// ❌ Don't use API keys in samples (prefer AAD)
const client = new AzureOpenAI({
  apiKey: process.env.AZURE_OPENAI_API_KEY,  // ❌ Use azureADTokenProvider
});
```

**Production Issue:** Sample had no timeout or retry configuration for OpenAI client.

---

### AI-1b: Streaming Chat Completions (HIGH)
**Pattern:** For streaming responses, use `client.chat.completions.create()` with `stream: true` and iterate with `for await...of`. This is the standard pattern for real-time AI chat applications.

✅ **DO:**
```typescript
import { AzureOpenAI } from 'openai';

// ✅ Streaming chat completions
const stream = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Explain quantum computing' }],
  stream: true,
});

// ✅ Async iteration pattern
for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) {
    process.stdout.write(content);  // Stream to terminal
  }
}
console.log();  // Newline after streaming completes

// ✅ With abort support for cancellation
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);

try {
  const stream = await client.chat.completions.create(
    {
      model: 'gpt-4o',
      messages: [{ role: 'user', content: 'Hello!' }],
      stream: true,
    },
    { signal: controller.signal }
  );

  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content;
    if (content) process.stdout.write(content);
  }
} finally {
  clearTimeout(timeout);
}
```

❌ **DON'T:**
```typescript
// ❌ Don't collect entire stream into memory then process
const chunks: string[] = [];
for await (const chunk of stream) {
  chunks.push(chunk.choices[0]?.delta?.content ?? '');
}
const fullResponse = chunks.join('');  // ❌ Defeats purpose of streaming
```

**Why:** Streaming is critical for AI chat applications — it provides immediate user feedback. The `for await...of` pattern is the idiomatic way to consume Azure OpenAI streams in TypeScript.

---

### AI-2: API Version Documentation (LOW)
**Pattern:** Hardcoded API versions should include a comment linking to version docs.

✅ **DO:**
```typescript
const client = new AzureOpenAI({
  apiVersion: '2024-10-21',
  // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
});
```

---

### AI-3: Document Intelligence and Speech SDK (MEDIUM)
**Pattern:** Use `@azure/ai-document-intelligence` with `DefaultAzureCredential`. For Speech, use `microsoft-cognitiveservices-speech-sdk` (there is no `@azure/ai-speech` package).

✅ **DO:**
```typescript
import { DocumentIntelligenceClient } from '@azure/ai-document-intelligence';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Document Intelligence with AAD
const credential = new DefaultAzureCredential();
const docClient = new DocumentIntelligenceClient(
  config.DOCUMENT_INTELLIGENCE_ENDPOINT,
  credential
);

const poller = await docClient.beginAnalyzeDocument(
  'prebuilt-invoice',
  documentStream
);
const result = await poller.pollUntilDone();

// ✅ Speech SDK (uses subscription key, AAD support limited)
import * as sdk from 'microsoft-cognitiveservices-speech-sdk';

const speechConfig = sdk.SpeechConfig.fromSubscription(
  config.SPEECH_KEY,
  config.SPEECH_REGION
);
const audioConfig = sdk.AudioConfig.fromDefaultMicrophoneInput();
const recognizer = new sdk.SpeechRecognizer(speechConfig, audioConfig);
```

---

### AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause silent failures or runtime errors.

✅ **DO:**
```typescript
// ✅ Document expected dimensions
const EMBEDDING_MODEL = 'text-embedding-3-small';  // 1536 dimensions
const VECTOR_DIMENSION = 1536;

// SQL table creation
const createTable = `
  CREATE TABLE [Documents] (
    [id] INT PRIMARY KEY,
    [content] NVARCHAR(MAX),
    [embedding] VECTOR(${VECTOR_DIMENSION})  -- Must match model output
  )
`;

// Validate embedding size
const embedding = await getEmbedding(text);
if (embedding.length !== VECTOR_DIMENSION) {
  throw new Error(
    `Embedding dimension mismatch: expected ${VECTOR_DIMENSION}, got ${embedding.length}\n` +
    `Ensure model '${EMBEDDING_MODEL}' matches table schema.`
  );
}
```

❌ **DON'T:**
```typescript
// ❌ Don't assume dimension without validation
const embedding = await getEmbedding(text);
await insertEmbedding(embedding);  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization. Includes service-specific best practices for Cosmos DB, Azure SQL (Tedious), Storage, and Tables.

### DB-1: Cosmos DB Client Patterns (HIGH)
**Pattern:** Use `@azure/cosmos` with AAD credentials. Handle partitioned containers properly. Use specific token scope.

✅ **DO:**
```typescript
import { CosmosClient } from '@azure/cosmos';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// ✅ Cosmos DB uses specific token scope
// The SDK handles token acquisition internally when using aadCredentials.
// If you need a token directly (rare), the scope is:
// 'https://{account-name}.documents.azure.com/.default'

const client = new CosmosClient({
  endpoint: config.COSMOS_ENDPOINT,
  aadCredentials: credential,
});

const database = client.database('mydb');
const container = database.container('mycontainer');

// ✅ Query with partition key
const { resources: items } = await container.items
  .query({
    query: 'SELECT * FROM c WHERE c.category = @category',
    parameters: [{ name: '@category', value: 'electronics' }],
  })
  .fetchAll();

// ✅ Point read (most efficient)
const { resource: item } = await container.item('item-id', 'partition-key-value').read();

// ✅ Create with partition key
await container.items.create({
  id: 'item-id',
  category: 'electronics',  // Partition key
  name: 'Laptop',
});

// ✅ Bulk operations (multiple partition keys supported, max 100 ops per batch)
const operations = [
  { operationType: 'Create' as const, resourceBody: { id: '1', category: 'electronics', name: 'Laptop' } },
  { operationType: 'Create' as const, resourceBody: { id: '2', category: 'electronics', name: 'Mouse' } },
  { operationType: 'Upsert' as const, resourceBody: { id: '3', category: 'electronics', name: 'Keyboard' } },
];
const bulkResult = await container.items.bulk(operations);
```

❌ **DON'T:**
```typescript
// ❌ Don't use primary key in samples
const client = new CosmosClient({
  endpoint: config.COSMOS_ENDPOINT,
  key: config.COSMOS_PRIMARY_KEY,  // ❌ Use aadCredentials
});

// ❌ Don't omit partition key in queries (cross-partition queries are expensive)
const { resources } = await container.items
  .query('SELECT * FROM c')
  .fetchAll();

// ❌ Don't use wrong token scope
const token = await credential.getToken('https://management.azure.com/.default');  // Wrong!
```

**Note:** Cosmos DB requires account-specific token scope, not generic Azure scopes.

---

### DB-2: SQL Database Patterns (Tedious Driver) (HIGH)
**Pattern:** Use Promise wrappers for tedious callbacks. Enable `rowCollectionOnDone`. Use native transaction methods.

✅ **DO:**
```typescript
import { Connection, Request, TYPES } from 'tedious';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Get AAD token
const credential = new DefaultAzureCredential();
const tokenResponse = await credential.getToken(
  'https://database.windows.net/.default'
);

// ✅ Connect with AAD token
const connection = new Connection({
  server: config.AZURE_SQL_SERVER,
  authentication: {
    type: 'azure-active-directory-access-token',
    options: {
      token: tokenResponse.token,
    },
  },
  options: {
    database: config.AZURE_SQL_DATABASE,
    encrypt: true,
    rowCollectionOnDone: true,  // ✅ Simplifies row handling
  },
});

await connectAsync(connection);

// ✅ Parameterized query
await executeQueryAsync(
  connection,
  'SELECT * FROM [Products] WHERE [Category] = @category',
  [{ name: 'category', type: TYPES.NVarChar, value: 'Electronics' }]
);

// ✅ Transaction with native methods (CRITICAL pattern)
try {
  await beginTransactionAsync(connection);
  await executeQueryAsync(connection, 'INSERT INTO [Orders] ...');
  await executeQueryAsync(connection, 'UPDATE [Inventory] ...');
  await commitTransactionAsync(connection);
} catch (err) {
  await rollbackTransactionAsync(connection);
  throw err;
}

// Helper functions
function connectAsync(connection: Connection): Promise<void> {
  return new Promise((resolve, reject) => {
    connection.on('connect', (err) => {
      if (err) reject(err);
      else resolve();
    });
    connection.connect();
  });
}

function beginTransactionAsync(connection: Connection): Promise<void> {
  return new Promise((resolve, reject) => {
    connection.beginTransaction((err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}

function commitTransactionAsync(connection: Connection): Promise<void> {
  return new Promise((resolve, reject) => {
    connection.commitTransaction((err) => {
      if (err) reject(err);
      else resolve();
    });
  });
}
```

❌ **DON'T:**
```typescript
// ❌ NEVER use raw SQL transaction commands with tedious
await executeQueryAsync(connection, 'BEGIN TRANSACTION');
await executeQueryAsync(connection, 'INSERT INTO ...');
await executeQueryAsync(connection, 'COMMIT TRANSACTION');

// Why: Tedious wraps execSql calls in sp_executesql, creating nested
// transaction scope. Use native beginTransaction/commitTransaction instead.
```

**Production Issue:** Vector search sample originally used raw `BEGIN TRANSACTION` / `COMMIT TRANSACTION` SQL, which fails because tedious wraps calls in `sp_executesql`, creating a nested transaction scope. Runtime testing revealed the bug — transaction was never committed.

---

### DB-3: SQL Identifier Quoting (MEDIUM)
**Pattern:** ALL dynamic SQL identifiers (table names, column names, index names) must use `[brackets]`.

✅ **DO:**
```typescript
const tableName = config.TABLE_NAME;  // User-provided

// ✅ Always bracket-quote identifiers
const createTable = `
  CREATE TABLE [${tableName}] (
    [id] INT PRIMARY KEY,
    [name] NVARCHAR(100),
    [category] NVARCHAR(50)
  )
`;

const query = `SELECT [id], [name] FROM [${tableName}] WHERE [id] = @id`;
```

❌ **DON'T:**
```typescript
// ❌ Missing brackets on dynamic identifier
const query = `SELECT id, name FROM ${tableName} WHERE id = @id`;
```

---

### DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

✅ **DO (SQL - Batch Insert):**
```typescript
// ✅ Batch size of 100 rows (SQL Server max ~2100 params, 100 rows * 3 params = 300)
const BATCH_SIZE = 100;
const items = [...]; // Array of items

for (let i = 0; i < items.length; i += BATCH_SIZE) {
  const batch = items.slice(i, i + BATCH_SIZE);
  
  // Build VALUES clause: (@id0, @name0, @cat0), (@id1, @name1, @cat1), ...
  const valuesClause = batch
    .map((_, idx) => `(@id${idx}, @name${idx}, @category${idx})`)
    .join(', ');
  
  const sql = `INSERT INTO [Products] ([id], [name], [category]) VALUES ${valuesClause}`;
  
  const params = batch.flatMap((item, idx) => [
    { name: `id${idx}`, type: TYPES.Int, value: item.id },
    { name: `name${idx}`, type: TYPES.NVarChar, value: item.name },
    { name: `category${idx}`, type: TYPES.NVarChar, value: item.category },
  ]);
  
  await executeQueryAsync(connection, sql, params);
}

// Why batch size 100: SQL Server has ~2100 parameter limit. 
// 100 rows * 3 params/row = 300 params, well under limit.
// Adjust batch size based on your column count: max_rows ≈ floor(2100 / params_per_row).
```

✅ **DO (Cosmos DB - Bulk):**
```typescript
// ✅ Cosmos bulk operations (use container.items.bulk(), NOT .batch())
const BULK_BATCH_SIZE = 100;  // Cosmos DB supports up to 100 operations per bulk call
const items = [...]; // Array of items to insert

for (let i = 0; i < items.length; i += BULK_BATCH_SIZE) {
  const batch = items.slice(i, i + BULK_BATCH_SIZE);
  const operations = batch.map(item => ({
    operationType: 'Create' as const,
    resourceBody: item,
  }));
  const result = await container.items.bulk(operations);
  console.log(`Bulk inserted ${result.length} items`);
}

// Why 100 max: Cosmos DB bulk() processes up to 100 operations per call.
// Note: container.items.batch() does NOT exist — always use .bulk().
```

✅ **DO (Storage Blob - Parallel):**
```typescript
// ✅ Parallel uploads (not batch, but more efficient than sequential)
const uploadPromises = files.map(file => 
  blobClient.uploadFile(file.path)
);
await Promise.all(uploadPromises);
```

❌ **DON'T:**
```typescript
// ❌ Row-by-row INSERT (50 round trips for 50 rows)
for (const item of items) {
  await executeQueryAsync(
    connection,
    'INSERT INTO [Products] VALUES (@id, @name, @category)',
    [
      { name: 'id', type: TYPES.Int, value: item.id },
      { name: 'name', type: TYPES.NVarChar, value: item.name },
      { name: 'category', type: TYPES.NVarChar, value: item.category },
    ]
  );
}
```

**Production Issue:** Vector search sample inserted 50 hotels one at a time. Changed to batch inserts (fewer round trips).

---

### DB-5: Azure Storage Patterns (MEDIUM)
**Pattern:** Use `@azure/storage-blob`, `@azure/storage-file-share`, `@azure/data-tables` with `DefaultAzureCredential`.

✅ **DO:**
```typescript
import { BlobServiceClient } from '@azure/storage-blob';
import { ShareServiceClient } from '@azure/storage-file-share';
import { TableClient } from '@azure/data-tables';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// ✅ Blob Storage
const blobServiceClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  credential
);

const containerClient = blobServiceClient.getContainerClient('mycontainer');
await containerClient.createIfNotExists();

const blockBlobClient = containerClient.getBlockBlobClient('myblob.txt');
await blockBlobClient.uploadData(Buffer.from('Hello, Azure!'));

const downloadResponse = await blockBlobClient.download();
const content = await streamToString(downloadResponse.readableStreamBody!);

// ✅ File Share
const shareServiceClient = new ShareServiceClient(
  `https://${accountName}.file.core.windows.net`,
  credential
);

const shareClient = shareServiceClient.getShareClient('myshare');
await shareClient.createIfNotExists();

// ✅ Table Storage
const tableClient = new TableClient(
  `https://${accountName}.table.core.windows.net`,
  'mytable',
  credential
);

await tableClient.createTable();
await tableClient.createEntity({
  partitionKey: 'partition1',
  rowKey: 'row1',
  name: 'Sample',
});

const entities = tableClient.listEntities({
  queryOptions: { filter: `PartitionKey eq 'partition1'` }
});
```

---

### DB-6: Storage SAS Fallback (MEDIUM)
**Pattern:** For local development or CI environments where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

✅ **DO:**
```typescript
import { BlobServiceClient, StorageSharedKeyCredential } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Try AAD first, fall back to SAS for local dev
let blobServiceClient: BlobServiceClient;

if (process.env.AZURE_STORAGE_SAS_TOKEN) {
  // Local dev: SAS token
  blobServiceClient = new BlobServiceClient(
    `https://${accountName}.blob.core.windows.net${process.env.AZURE_STORAGE_SAS_TOKEN}`
  );
  console.log('Using SAS token authentication (local dev)');
} else {
  // Production: AAD
  const credential = new DefaultAzureCredential();
  blobServiceClient = new BlobServiceClient(
    `https://${accountName}.blob.core.windows.net`,
    credential
  );
  console.log('Using DefaultAzureCredential (AAD)');
}

// Document in README:
// > **Local Development:** If `az login` doesn't work, generate a SAS token:
// > ```bash
// > az storage container generate-sas --account-name <name> --name <container> \
// >   --permissions acdlrw --expiry 2025-12-31T23:59:00Z
// > ```
// > Add to .env: `AZURE_STORAGE_SAS_TOKEN=?sv=2021-06-08&...`
```

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for queues, topics, event ingestion, and event-driven architectures. Focus on reliable message handling, checkpoint management, and proper resource cleanup.

### MSG-1: Service Bus Patterns (HIGH)
**Pattern:** Use `@azure/service-bus` with `DefaultAzureCredential`. Handle message sessions, dead-letter queues.

✅ **DO:**
```typescript
import { ServiceBusClient, ServiceBusMessage } from '@azure/service-bus';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const serviceBusClient = new ServiceBusClient(
  `${namespace}.servicebus.windows.net`,
  credential
);

// ✅ Send messages
const sender = serviceBusClient.createSender('myqueue');
const messages: ServiceBusMessage[] = [
  { body: { orderId: 1, amount: 100 } },
  { body: { orderId: 2, amount: 200 } },
];
await sender.sendMessages(messages);
await sender.close();

// ✅ Receive messages (MUST complete or abandon)
const receiver = serviceBusClient.createReceiver('myqueue');
const receivedMessages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });

for (const message of receivedMessages) {
  try {
    console.log(`Received: ${JSON.stringify(message.body)}`);
    // Process message
    await receiver.completeMessage(message);  // ✅ Mark as processed
  } catch (err) {
    await receiver.abandonMessage(message);  // ✅ Return to queue
  }
}

await receiver.close();
await serviceBusClient.close();
```

❌ **DON'T:**
```typescript
// ❌ Don't use connection strings in samples
const serviceBusClient = ServiceBusClient.fromConnectionString(connectionString);

// ❌ Don't forget to complete/abandon messages
for (const message of receivedMessages) {
  console.log(message.body);  // ❌ Message never completed (will reappear)
}
```

---

### MSG-2: Event Hubs Patterns (MEDIUM)
**Pattern:** Use `@azure/event-hubs` for ingestion, `@azure/event-hubs-checkpointstore-blob` for processing with checkpoint management.

✅ **DO:**
```typescript
import { EventHubProducerClient, EventHubConsumerClient } from '@azure/event-hubs';
import { BlobCheckpointStore } from '@azure/event-hubs-checkpointstore-blob';
import { ContainerClient } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// ✅ Send events
const producer = new EventHubProducerClient(
  `${namespace}.servicebus.windows.net`,
  'myeventhub',
  credential
);

const batch = await producer.createBatch();
batch.tryAdd({ body: { temperature: 23.5 } });
batch.tryAdd({ body: { temperature: 24.1 } });
await producer.sendBatch(batch);
await producer.close();

// ✅ Receive events with checkpoint store
const containerClient = new ContainerClient(
  `https://${storageAccount}.blob.core.windows.net/eventhub-checkpoints`,
  credential
);

const checkpointStore = new BlobCheckpointStore(containerClient);

const consumer = new EventHubConsumerClient(
  '$Default',
  `${namespace}.servicebus.windows.net`,
  'myeventhub',
  credential,
  checkpointStore
);

consumer.subscribe({
  processEvents: async (events, context) => {
    for (const event of events) {
      console.log(`Received: ${JSON.stringify(event.body)}`);
    }
    await context.updateCheckpoint(events[events.length - 1]);  // ✅ Checkpoint
  },
  processError: async (err, context) => {
    console.error(`Error: ${err.message}`);
  },
});
```

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault. Covers secrets, keys, and certificates with AAD authentication.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `@azure/keyvault-secrets`, `@azure/keyvault-keys`, `@azure/keyvault-certificates` with `DefaultAzureCredential`.

✅ **DO:**
```typescript
import { SecretClient } from '@azure/keyvault-secrets';
import { KeyClient } from '@azure/keyvault-keys';
import { CertificateClient } from '@azure/keyvault-certificates';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// ✅ Secrets
const secretClient = new SecretClient(config.AZURE_KEYVAULT_URL, credential);

await secretClient.setSecret('db-password', 'P@ssw0rd123');
const secret = await secretClient.getSecret('db-password');
console.log(`Secret value: ${secret.value}`);

// ✅ Keys (for encryption)
const keyClient = new KeyClient(config.AZURE_KEYVAULT_URL, credential);

const key = await keyClient.createKey('my-encryption-key', 'RSA');
console.log(`Key ID: ${key.id}`);

// ✅ Certificates
const certClient = new CertificateClient(config.AZURE_KEYVAULT_URL, credential);

const cert = await certClient.getCertificate('my-cert');
console.log(`Certificate thumbprint: ${cert.properties.x509Thumbprint}`);
```

❌ **DON'T:**
```typescript
// ❌ Don't hardcode secrets in samples
const dbPassword = 'P@ssw0rd123';  // ❌ Use Key Vault

// ❌ Don't use Key Vault for every sample (adds complexity)
// Use Key Vault when demonstrating secret management or when the sample
// explicitly covers secure configuration patterns.
```

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations across Azure data services. Includes embedding storage, distance calculations, and approximate nearest neighbor search with DiskANN.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Use `CAST(@param AS VECTOR(dimension))` for Azure SQL vector parameters. Serialize vectors as JSON strings.

✅ **DO (Azure SQL):**
```typescript
const embedding = [0.1, 0.2, 0.3, ...];  // 1536 floats from OpenAI

await executeQueryAsync(
  connection,
  'INSERT INTO [Hotels] ([embedding]) VALUES (CAST(@embedding AS VECTOR(1536)))',
  [
    { name: 'embedding', type: TYPES.NVarChar, value: JSON.stringify(embedding) },
  ]
);

// ✅ Vector distance query
const sql = `
  SELECT TOP (@k)
    [id],
    [name],
    VECTOR_DISTANCE('cosine', [embedding], CAST(@searchEmbedding AS VECTOR(1536))) AS distance
  FROM [Hotels]
  ORDER BY distance ASC
`;
```

✅ **DO (Cosmos DB Vector Search):**
```typescript
// ✅ Cosmos DB vector search (preview)
const { resources } = await container.items
  .query({
    query: `
      SELECT TOP @k c.id, c.name, VectorDistance(c.embedding, @searchEmbedding) AS similarity
      FROM c
      ORDER BY VectorDistance(c.embedding, @searchEmbedding)
    `,
    parameters: [
      { name: '@k', value: 5 },
      { name: '@searchEmbedding', value: searchEmbedding },
    ],
  })
  .fetchAll();
```

✅ **DO (Azure AI Search):**
```typescript
import { SearchClient } from '@azure/search-documents';
import { DefaultAzureCredential } from '@azure/identity';

// ✅ Prefer DefaultAzureCredential (identity-based auth)
const credential = new DefaultAzureCredential();
const searchClient = new SearchClient(
  config.SEARCH_ENDPOINT,
  'hotels-index',
  credential
);

// ✅ Fallback: API key auth (for environments without AAD support)
// import { AzureKeyCredential } from '@azure/search-documents';
// const searchClient = new SearchClient(
//   config.SEARCH_ENDPOINT,
//   'hotels-index',
//   new AzureKeyCredential(config.SEARCH_API_KEY)
// );

const searchResults = await searchClient.search('luxury hotel', {
  vectorSearchOptions: {
    queries: [{
      kind: 'vector',
      vector: searchEmbedding,
      kNearestNeighborsCount: 5,
      fields: ['descriptionVector'],
    }],
  },
});

for await (const result of searchResults.results) {
  console.log(`${result.document.name} (score: ${result.score})`);
}
```

---

### VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires ≥1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

✅ **DO:**
```typescript
// Check row count before creating DiskANN index
const countResult = await executeQueryAsync(
  connection,
  `SELECT COUNT(*) AS [count] FROM [${tableName}]`,
  []
);
const rowCount = countResult[0][0].value;

if (rowCount >= 1000) {
  console.log(`✅ ${rowCount} rows available. Creating DiskANN index...`);
  
  await executeQueryAsync(
    connection,
    `CREATE INDEX [ix_${tableName}_embedding_diskann]
     ON [${tableName}] ([embedding])
     USING DiskANN`,
    []
  );
  
  // ✅ Use VECTOR_SEARCH for fast approximate search
  const sql = `
    SELECT [id], [name]
    FROM VECTOR_SEARCH(
      CAST(@searchEmbedding AS VECTOR(1536)),
      [${tableName}],
      [embedding],
      @k
    ) VS
    JOIN [${tableName}] H ON VS.[id] = H.[id]
  `;
  
  // Note on response shape: VECTOR_SEARCH returns rows with VS.[id] and distance.
  // Join with original table to get full row data. Response is array of rows,
  // each row is array of column values (tedious format with rowCollectionOnDone: true).
} else {
  console.log(`⚠️ Only ${rowCount} rows. DiskANN requires ≥1000. Using exact search.`);
  
  // Fall back to VECTOR_DISTANCE (exact search)
  const sql = `
    SELECT TOP (@k) [id], [name],
      VECTOR_DISTANCE('cosine', [embedding], CAST(@searchEmbedding AS VECTOR(1536))) AS distance
    FROM [${tableName}]
    ORDER BY distance ASC
  `;
}
```

❌ **DON'T:**
```typescript
// ❌ Create DiskANN index without checking row count
await executeQueryAsync(connection, 'CREATE INDEX ... USING DiskANN', []);
// Fails with: "DiskANN index requires at least 1000 rows"
```

**Production Issue:** Sample didn't guard DiskANN index creation, causing runtime errors with small datasets.

**Technical Note:** Tedious returns query results as arrays of rows when `rowCollectionOnDone: true` is set. Each row is an array of column values. The `VECTOR_SEARCH` function returns a virtual table with `[id]` and `distance` columns, which must be joined with the original table to retrieve full row data.

---

## 8. Error Handling

**What this section covers:** Type-safe error catching, contextual error messages, and troubleshooting guidance. Proper error handling prevents silent failures and helps users diagnose issues.

### ERR-1: Type-Safe Error Catching (MEDIUM)
**Pattern:** Use `catch(err: unknown)` with type narrowing. Avoid `catch(err: any)`.

✅ **DO:**
```typescript
try {
  await someAsyncOperation();
} catch (err: unknown) {
  if (err instanceof Error) {
    console.error(`Operation failed: ${err.message}`);
    if (err.stack) {
      console.error(err.stack);
    }
  } else {
    console.error(`Unknown error: ${String(err)}`);
  }
  throw err;
}
```

❌ **DON'T:**
```typescript
try {
  await someAsyncOperation();
} catch (err: any) {  // ❌ Loses type safety
  console.error(err.message);  // ❌ No guarantee err has .message
}
```

---

### ERR-2: Contextual Error Messages (MEDIUM)
**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

✅ **DO:**
```typescript
try {
  await credential.getToken('https://storage.azure.com/.default');
} catch (err: unknown) {
  if (err instanceof Error) {
    console.error(`❌ Failed to acquire Azure Storage token: ${err.message}`);
    console.error(`\nTroubleshooting:`);
    console.error(`  1. Run 'az login' to authenticate with Azure CLI`);
    console.error(`  2. Verify you have the "Storage Blob Data Contributor" role`);
    console.error(`  3. Check your Azure subscription is active`);
    console.error(`  4. Ensure firewall rules allow access from your IP`);
  }
  throw err;
}
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
│   ├── index.ts                   # Loads products-with-vectors.json
│   ├── generate-embeddings.ts     # Generates embeddings (optional)
```

❌ **DON'T:**
```
repo/
├── data/
│   ├── products.json              # ✅ Raw data
│   ├── .gitignore                 # ❌ products-with-vectors.json gitignored
├── src/
│   ├── index.ts                   # ❌ Fails: File not found
```

**Production Issue:** Vector search sample's `HotelsData_Vector.json` missing from repo, causing runtime error.

---

### DATA-2: JSON Data Loading (MEDIUM)
**Pattern:** Use ESM-compatible JSON imports with type assertions.

✅ **DO:**
```typescript
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';
import { readFileSync } from 'fs';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

interface Product {
  id: string;
  name: string;
  category: string;
}

function loadProducts(): Product[] {
  const dataPath = join(__dirname, '..', 'data', 'products.json');
  const data = readFileSync(dataPath, 'utf-8');
  return JSON.parse(data) as Product[];
}
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

# Dependencies
node_modules/

# Build output
dist/
build/
*.tsbuildinfo

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
coverage/
.nyc_output/
```

❌ **DON'T:**
```
repo/
├── .env                    # ❌ Live credentials committed!
├── .env.local              # ❌ Local overrides committed!
├── node_modules/           # ❌ 200MB of dependencies committed
├── dist/                   # ❌ Build artifacts committed
```

**Production Issue:** Vector search sample had `.env` with live Azure subscription ID, tenant ID, and endpoints committed. No `.gitignore` file. Also had `dist/` and `node_modules/` committed.

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

**Production Issue:** Sample committed `.env` with real subscription ID, tenant ID, and endpoints. Critical security issue.

---

### HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, functions, and imports. Commented-out code confuses users.

✅ **DO:**
```typescript
// Only import what you use
import { BlobServiceClient } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';
```

❌ **DON'T:**
```typescript
// ❌ Commented-out code confuses users
// import { OldClient } from '@azure/old-package';
// 
// async function oldImplementation() {
//   // This was the old way...
// }

import { BlobServiceClient } from '@azure/storage-blob';
```

```
infra/
├── abbreviations.json      # ❌ Never referenced by any Bicep file
```

**Production Issue:** Sample had `abbreviations.json` in `infra/` that was never referenced by Bicep templates.

**Note:** Dead code severity upgraded from MEDIUM to HIGH — it significantly confuses users trying to learn from samples.

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── package.json
├── src/
```

```
LICENSE file content:
MIT License

Copyright (c) Microsoft Corporation.

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction...
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── package.json
```

**Why:** All repositories in Azure-Samples organization require MIT license.

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

```
repo/
├── LICENSE
├── README.md
├── CONTRIBUTING.md      # ✅ Contribution guidelines
├── SECURITY.md          # ✅ Security reporting
├── CODEOWNERS           # ✅ Code ownership (optional)
```

---

## 11. README & Documentation

**What this section covers:** Documentation quality, accuracy, and completeness. Covers expected output, troubleshooting, prerequisites, and setup instructions. Documentation must be accurate enough for users to succeed on first run.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:
```bash
npm start
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

**Production Issue:** Vector search README showed hotel names that didn't match actual program output. Expected output was completely fabricated.

---

### DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

✅ **DO:**
```markdown
## Project Structure

- [`src/index.ts`](./src/index.ts) — Main entry point
- [`src/config.ts`](./src/config.ts) — Configuration loader
- [`infra/main.bicep`](./infra/main.bicep) — Infrastructure template
```

❌ **DON'T:**
```markdown
- [`src/index.ts`](./TypeScript/src/index.ts)  # ❌ Wrong path
```

**Production Issue:** README linked to `./TypeScript/` but actual folder was `vector-search-query-typescript/`.

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
- For services behind VNets, ensure you're connected to the VNet

### RBAC Permission Errors

If you see "Authorization failed" or "Forbidden":
- Verify you have the required role assignments:
  - Storage: "Storage Blob Data Contributor"
  - Key Vault: "Key Vault Secrets User"
  - Cosmos DB: "Cosmos DB Account Reader" + "Cosmos DB Data Contributor"
- Role assignments may take 5-10 minutes to propagate

### Common Azure SDK Errors

- `RestError: The specified container does not exist`: Container must be created first
- `ServiceRequestError: ENOTFOUND`: Incorrect endpoint URL in .env
- `CredentialUnavailableError`: No authentication method available (run `az login`)
```

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly (Azure subscription, CLI tools, role assignments, services).

✅ **DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Node.js**: Version 20 or later ([Download](https://nodejs.org/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional, for infrastructure deployment)

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

To assign roles:
```bash
az role assignment create --role "Storage Blob Data Contributor" \
  --assignee <your-email@domain.com> \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>
```
```

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions. Include Azure resource provisioning.

✅ **DO:**
```markdown
## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Azure-Samples/azure-storage-blob-samples.git
cd azure-storage-blob-samples/quickstart
```

### 2. Install dependencies

```bash
npm install
```

### 3. Provision Azure resources

```bash
azd up
```

This will:
- Provision Storage Account
- Assign role permissions to your identity
- Output environment variables to `.env`

### 4. Run the sample

```bash
npm start
```
```

---

### DOC-6: Node.js Version Strategy (LOW)
**Pattern:** Document minimum Node.js version in both README and package.json engines field.

✅ **DO:**
```json
// package.json
{
  "engines": {
    "node": ">=20.0.0"
  }
}
```

```markdown
# README.md

## Prerequisites

- **Node.js**: Version 20.12 or later required for `--env-file` support
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values. No ambiguous `<your-resource-name>` without explanation.

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

**What this section covers:** Infrastructure-as-code patterns, Azure Verified Modules, parameter validation, API versioning, resource naming, and role assignments. Ensures infrastructure code follows Azure best practices.

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

module keyVault 'br/public:avm/res/key-vault/vault:0.11.0' = {
  name: 'keyvault-deployment'
  params: {
    name: keyVaultName
    location: location
  }
}

module cognitiveServices 'br/public:avm/res/cognitive-services/account:1.0.1' = {
  name: 'openai-deployment'
  params: {
    name: openaiAccountName
    kind: 'OpenAI'
    location: location
  }
}

// Check latest versions at: https://azure.github.io/Azure-Verified-Modules/
```

❌ **DON'T:**
```bicep
module cognitiveServices 'br/public:avm/res/cognitive-services/account:0.7.1' = {
  // ❌ Outdated version (current is 1.0.1+)
}
```

**Production Issue:** Vector search sample used Cognitive Services AVM module version 0.7.1 vs current 1.0.1+.

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

@description('Storage account SKU')
@allowed(['Standard_LRS', 'Standard_GRS', 'Premium_LRS'])
param storageAccountSku string = 'Standard_LRS'
```

❌ **DON'T:**
```bicep
@description('Azure AD admin object ID')
param aadAdminObjectId string  // ❌ No validation, accepts empty string
```

**Production Issue:** Vector search sample's `aadAdminObjectId` parameter accepted empty string, creating SQL server with no administrator.

---

### IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2024+). Avoid versions older than 2 years.

✅ **DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  // Current stable version
}

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  // Current stable version
}

resource appService 'Microsoft.Web/sites@2023-12-01' = {
  // Current version
}

resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  // Preview OK for new features (e.g., vector search)
}
```

❌ **DON'T:**
```bicep
resource resourceGroup 'Microsoft.Resources/resourceGroups@2021-04-01' = {
  // ❌ 3+ years old, use 2024-03-01 or later
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // ❌ 5+ years old
}
```

**Note:** Examples updated to use 2023-2024 API versions (per reviewer feedback).

---

### IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities to access Azure resources.

✅ **DO:**
```bicep
// App Service with system-assigned managed identity
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    // ...
  }
}

// Role assignment: App Service -> Storage Blob Data Contributor
resource storageBlobDataContributorRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: storageAccount
  name: guid(storageAccount.id, appService.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')  // Storage Blob Data Contributor
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Role assignment: App Service -> Key Vault Secrets User
resource keyVaultSecretsUserRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  scope: keyVault
  name: guid(keyVault.id, appService.id, '4633458b-17de-408a-b874-0445c86b69e6')
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')  // Key Vault Secrets User
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Common role IDs:**
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Storage Blob Data Reader: `2a2b9908-6ea1-4ae2-8e65-a410df84e7d1`
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
// For production deployments, use private endpoints and set defaultAction: 'Deny'.
// See: https://learn.microsoft.com/azure/ai-services/cognitive-services-virtual-networks
```

✅ **DO (Production):**
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: openaiAccountName
  properties: {
    publicNetworkAccess: 'Disabled'
    networkAcls: {
      defaultAction: 'Deny'
    }
  }
}

module privateEndpoint 'br/public:avm/res/network/private-endpoint:0.4.0' = {
  name: 'openai-private-endpoint'
  params: {
    name: '${openaiAccountName}-pe'
    privateLinkServiceConnections: [
      {
        name: '${openaiAccountName}-connection'
        properties: {
          privateLinkServiceId: openai.id
          groupIds: ['account']
        }
      }
    ]
  }
}
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
output AZURE_COSMOS_ENDPOINT string = cosmos.properties.documentEndpoint
output AZURE_SERVICE_BUS_NAMESPACE string = serviceBusNamespace.name
```

---

### IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions. Use consistent naming patterns.

✅ **DO:**
```bicep
@description('Prefix for all resources')
param resourcePrefix string = 'contoso'

@description('Environment (dev, test, prod)')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

// ✅ Consistent naming pattern: {prefix}-{service}-{environment}
var storageAccountName = '${resourcePrefix}st${environment}'  // Max 24 chars, alphanumeric only
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
var cosmosAccountName = '${resourcePrefix}-cosmos-${environment}'

// Reference: https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming
```

❌ **DON'T:**
```bicep
// ❌ Inconsistent naming
var storageAccountName = 'mystorageaccount123'
var keyVaultName = 'kv-${uniqueString(resourceGroup().id)}'
var appServiceName = 'webapp'
```

**CAF naming patterns:**
- Storage Account: `{prefix}st{env}` (max 24 chars, lowercase alphanumeric)
- Key Vault: `{prefix}-kv-{env}` (3-24 chars, alphanumeric and hyphens)
- App Service: `{prefix}-app-{env}`
- Cosmos DB: `{prefix}-cosmos-{env}`
- Function App: `{prefix}-func-{env}`

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns, azure.yaml structure, service definitions, hooks, and host types. Ensures samples can be deployed end-to-end with `azd up`.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

✅ **DO:**
```yaml
name: azure-storage-blob-sample
metadata:
  template: azure-storage-blob-sample@0.0.1

services:
  app:
    project: ./
    language: ts
    host: appservice  # or: containerapp, function, staticwebapp, aks

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
      echo "Key Vault: ${AZURE_KEYVAULT_URL}"
      echo ""
      echo "Run 'npm install && npm start' to test the sample."
  
  predeploy:
    shell: sh
    run: |
      echo "Building application..."
      npm run build

  postdeploy:
    shell: sh
    run: |
      echo "✅ Deployment complete"
      echo "Application URL: ${SERVICE_APP_ENDPOINT_URL}"

# Infrastructure outputs (from infra/main.bicep)
# - AZURE_STORAGE_ACCOUNT_NAME
# - AZURE_STORAGE_BLOB_ENDPOINT
# - AZURE_KEYVAULT_URL
```

---

### AZD-2: Service Host Types (MEDIUM)
**Pattern:** Choose correct `host` type for different application types.

✅ **DO:**
```yaml
# App Service (Node.js, Python, Java web apps)
services:
  web:
    project: ./
    language: ts
    host: appservice

# Azure Functions
services:
  api:
    project: ./
    language: ts
    host: function

# Container Apps
services:
  backend:
    project: ./
    language: ts
    host: containerapp
    docker:
      path: ./Dockerfile

# Static Web Apps (frontend only)
services:
  frontend:
    project: ./
    language: ts
    host: staticwebapp

# Azure Kubernetes Service (AKS)
services:
  microservice:
    project: ./
    language: ts
    host: aks
    k8s:
      manifests:
        - ./manifests/deployment.yaml
        - ./manifests/service.yaml
```

**Supported hosts:** `appservice`, `function`, `containerapp`, `staticwebapp`, `aks`

---

## 14. CI/CD & Testing

**What this section covers:** Continuous integration patterns, type checking, linting, and build validation. Ensures samples build correctly in CI before publication.

### CI-1: Type Checking (HIGH)
**Pattern:** Run TypeScript type checking in CI before build.

✅ **DO:**
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high  # CVE check
      - run: npm run typecheck                # tsc --noEmit
      - run: npm run build
      - run: npm test
```

```json
// package.json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "test": "echo \"No tests yet\" && exit 0"
  }
}
```

---

### CI-2: Dependency Audit in CI (MEDIUM)
**Pattern:** Run `npm audit` in CI to catch known vulnerabilities before publication. Fail the build on critical/high CVEs.

✅ **DO:**
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high
```

```json
// package.json
{
  "scripts": {
    "audit": "npm audit --audit-level=high",
    "pretest": "npm run audit"
  }
}
```

**Why:** Automated CVE scanning prevents publishing samples with known security vulnerabilities. Run alongside type checking (CI-1) for comprehensive CI coverage.

---

## Pre-Review Checklist (Comprehensive)

Use this comprehensive checklist before submitting an Azure SDK TypeScript sample for review:

### 🔧 Project Setup
- [ ] ESM configuration correct (`"type": "module"`, `moduleResolution: "nodenext"`)
- [ ] Package.json has `author`, `license`, `repository`, `engines`
- [ ] Every dependency is imported somewhere (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`@azure/*`, not `azure-*`)
- [ ] Using `openai` SDK with `AzureOpenAI` class (not retired `@azure/openai`)
- [ ] Environment variables validated with descriptive errors
- [ ] Node.js 20+ `--env-file` used (not `dotenv`)
- [ ] TypeScript `strict: true` enabled
- [ ] package-lock.json committed
- [ ] Only one package manager (npm OR yarn OR pnpm, not mixed)
- [ ] `npm audit` passes (no critical/high CVEs)
- [ ] .editorconfig present (optional but recommended)

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `.env`, `.env.*`, `node_modules/`, `dist/`, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed (subscription IDs, tenant IDs, connection strings, API keys)
- [ ] No build artifacts committed (`dist/`, `*.tsbuildinfo`)
- [ ] Dead code removed (unused files, imports, functions, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included
- [ ] Package legitimacy verified (all `@azure/*` from Microsoft)

### ☁️ Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Client options configured (retry policies, timeouts where applicable)
- [ ] Token refresh implemented for long-running operations (CRITICAL)
- [ ] Managed identity pattern documented in README (system vs user-assigned)
- [ ] Service-specific client patterns followed (Storage, Cosmos, Service Bus, etc.)
- [ ] Pagination handled with `for await...of` or `fetchAll()`
- [ ] Resource cleanup in try/finally blocks for clients that hold connections

### 🗄️ Data Services (if applicable)
- [ ] SQL: Tedious uses native `beginTransaction`/`commitTransaction` (NOT raw SQL)
- [ ] SQL: All dynamic identifiers use `[brackets]`
- [ ] SQL: Values parameterized (no string concatenation)
- [ ] SQL/Cosmos: Batch operations for multiple rows (not row-by-row)
- [ ] Cosmos: Queries include partition key (avoid cross-partition)
- [ ] Cosmos: Uses correct token scope (`https://{account}.documents.azure.com/.default`)
- [ ] Storage: Blob/Table/File client patterns followed
- [ ] Storage: SAS fallback documented for local dev (optional)
- [ ] Batch size rationale documented (SQL param limits, Cosmos 100 limit)

### 🤖 AI Services (if applicable)
- [ ] Using `openai` SDK with `AzureOpenAI` class (not `@azure/openai`)
- [ ] OpenAI client has `azureADTokenProvider`, `timeout`, `maxRetries`
- [ ] API versions documented with links to version docs
- [ ] Vector search: DiskANN index creation guarded by row count check (≥1000 for SQL)
- [ ] Vector search: Fallback to exact search for small datasets
- [ ] Vector dimensions validated (match between model and storage)
- [ ] Pre-computed embedding data file committed to repo

### 💬 Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses checkpoint store for processing
- [ ] Connection strings avoided (prefer AAD auth)

### ❌ Error Handling
- [ ] `catch(err: unknown)` with type narrowing (not `any`)
- [ ] Error messages are contextual and actionable
- [ ] Auth/network/RBAC errors include troubleshooting hints

### 📄 Documentation
- [ ] README "Expected output" copy-pasted from real run (not fabricated)
- [ ] All internal folder/file links match actual filesystem paths
- [ ] Prerequisites section complete (subscription, CLI, role assignments)
- [ ] Troubleshooting section covers common Azure errors
- [ ] Setup instructions clear and tested
- [ ] Placeholder values have clear replacement instructions
- [ ] Node.js version documented in README and package.json

### 🏗️ Infrastructure (if applicable)
- [ ] Azure Verified Module versions current (check azure.github.io/Azure-Verified-Modules)
- [ ] Bicep parameters validated (`@minLength`, `@maxLength`, `@allowed`)
- [ ] API versions current (2023-2024+, not 3+ years old)
- [ ] Network security documented (public vs. private endpoints)
- [ ] RBAC role assignments created for managed identities
- [ ] Resource naming follows CAF conventions (`{prefix}-{service}-{env}`)
- [ ] Output values follow azd naming conventions (`AZURE_*`)
- [ ] `azure.yaml` includes services, hooks, metadata
- [ ] azd hooks documented (preprovision, postprovision, predeploy, postdeploy)

### 🧪 CI/CD
- [ ] Type checking runs in CI (`tsc --noEmit`)
- [ ] CVE scanning runs in CI (`npm audit --audit-level=high`)
- [ ] Build succeeds in CI
- [ ] Tests pass (or `exit 0` if no tests yet)

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency
- **[`typescript-review`](../typescript-review/SKILL.md)**: General TypeScript patterns, type safety, async patterns, and Node.js specifics (not Azure SDK specific)

---

## Scope Note: Services Not Yet Covered

This skill focuses on the most commonly used Azure services in TypeScript samples. The following services are not yet covered in detail, but the general patterns (authentication, client construction, error handling) apply:

- Azure Communication Services
- Azure Cache for Redis
- Azure Monitor
- Azure Container Registry
- Azure App Configuration
- Azure SignalR Service
- Azure Static Web Apps (authoring)
- Azure API Management

### Azure Functions (v4 Programming Model)

For Azure Functions Node.js samples, use the **v4 programming model** (`@azure/functions` v4):

```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

// ✅ v4 model: function registration pattern
app.http('httpTrigger', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    context.log('HTTP trigger processed a request.');
    const name = request.query.get('name') || (await request.text()) || 'World';
    return { body: `Hello, ${name}!` };
  },
});
```

The v4 model replaces the legacy v3 `function.json` + `index.ts` pattern with a single-file registration approach. See [Azure Functions Node.js v4 docs](https://learn.microsoft.com/azure/azure-functions/functions-reference-node?tabs=typescript%2Cwindows%2Cazure-cli&pivots=nodejs-model-v4).

For samples using these services, apply the core patterns from Sections 1-2 (Project Setup, Azure SDK Client Patterns) and reference service-specific documentation.

---

## Reference Links

Consolidation of all documentation links referenced throughout this skill:

### Azure SDK & Authentication
- [Azure SDK for JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/)
- [DefaultAzureCredential](https://learn.microsoft.com/javascript/api/@azure/identity/defaultazurecredential)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)

### API Versioning
- [Azure OpenAI API Versions](https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation)
- [Azure REST API Specifications](https://github.com/Azure/azure-rest-api-specs)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework - Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Security
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Azure Private Endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)
- [Cognitive Services Virtual Networks](https://learn.microsoft.com/azure/ai-services/cognitive-services-virtual-networks)

### Node.js
- [Node.js Downloads](https://nodejs.org/)
- [Node.js --env-file documentation](https://nodejs.org/docs/latest-v20.x/api/cli.html#--env-fileconfig)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK TypeScript sample patterns** distilled from comprehensive reviews including the Azure SQL vector search sample (29 findings) plus generalized patterns across the Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (10 rules): Credentials, phantom deps, CVE scanning, token refresh, AVM versions, parameter validation, .gitignore, fabricated output, broken auth, missing error handling
- **HIGH** (22 rules): Client construction, token management, managed identity, pagination, OpenAI config, streaming completions, database transactions, DiskANN guards, batch operations, RBAC, lock files, role assignments, pre-computed data, .env.sample, prerequisites, dead code, LICENSE file, resource naming
- **MEDIUM** (29 rules): Client options, retry policies, identifier quoting, embeddings, error handling, JSON loading, troubleshooting, azd structure, type checking, dependency audit CI, strict flags, version pinning, SAS fallback, dimensions, placeholder docs, resource cleanup, API versions, governance files, abort/cancellation, diagnostic logging
- **LOW** (7 rules): API version docs, .editorconfig, CONTRIBUTING.md, scope notes, Node.js version, region availability, Azure Functions v4

### Service Coverage
- **Core SDK**: Authentication, credentials, managed identities, client patterns, token management, pagination, resource cleanup
- **Data**: Cosmos DB, Azure SQL (Tedious), Storage (Blob/Table/File), batch operations
- **AI**: Azure OpenAI (embeddings, chat, streaming, images), Document Intelligence, Speech, vector dimensions
- **Messaging**: Service Bus, Event Hubs, Event Grid, checkpoint management
- **Security**: Key Vault (secrets, keys, certificates)
- **Vector Search**: Azure SQL DiskANN, Cosmos DB, AI Search
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming

Apply these patterns to ensure Azure SDK TypeScript samples are **secure, accurate, maintainable, and ready for publication**.
