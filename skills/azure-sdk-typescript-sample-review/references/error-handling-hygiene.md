# Error Handling, Data Management, and Sample Hygiene (ERR-1 through ERR-2, DATA-1 through DATA-2, HYG-1 through HYG-5)

Rules for error patterns, sample data, and repository hygiene.

## ERR-1: Type-Safe Error Catching (MEDIUM)

**Pattern:** Use `catch(err: unknown)` with type narrowing. Avoid `catch(err: any)`.

DO:
```typescript
try {
  await someAsyncOperation();
} catch (err: unknown) {
  if (err instanceof Error) {
    console.error(`Operation failed: ${err.message}`);
  } else {
    console.error(`Unknown error: ${String(err)}`);
  }
  throw err;
}
```

DON'T:
```typescript
try { await op(); } catch (err: any) { console.error(err.message); } // Loses type safety
```

---

## ERR-2: Contextual Error Messages (MEDIUM)

**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

DO:
```typescript
try {
  await credential.getToken('https://storage.azure.com/.default');
} catch (err: unknown) {
  if (err instanceof Error) {
    console.error(`Failed to acquire Azure Storage token: ${err.message}`);
    console.error(`Troubleshooting:`);
    console.error(`  1. Run 'az login' to authenticate`);
    console.error(`  2. Verify "Storage Blob Data Contributor" role`);
    console.error(`  3. Check your Azure subscription is active`);
  }
  throw err;
}
```

---

## DATA-1: Pre-Computed Data Files (HIGH)

**Pattern:** Commit all required data files to repo. Pre-computed embeddings avoid requiring API calls on first run.

DO:
```
repo/
  data/
    products.json
    products-with-vectors.json   # Pre-computed embeddings committed
  src/
    index.ts                     # Loads products-with-vectors.json
```

DON'T: Gitignore the pre-computed data file — causes runtime error on first run.

---

## DATA-2: JSON Data Loading (MEDIUM)

**Pattern:** Use ESM-compatible JSON imports with type assertions.

DO:
```typescript
import { fileURLToPath } from 'url';
import { dirname, join } from 'path';
import { readFileSync } from 'fs';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

function loadProducts(): Product[] {
  const dataPath = join(__dirname, '..', 'data', 'products.json');
  return JSON.parse(readFileSync(dataPath, 'utf-8')) as Product[];
}
```

---

## HYG-1: .gitignore (CRITICAL)

**Pattern:** Protect sensitive files, build artifacts, and dependencies.

DO:
```gitignore
.env
.env.local
.env.*.local
!.env.sample
!.env.example
node_modules/
dist/
build/
*.tsbuildinfo
.azure/
.DS_Store
Thumbs.db
coverage/
```

---

## HYG-2: .env.sample (HIGH)

**Pattern:** Provide `.env.sample` with placeholder values. Never commit actual `.env`.

DO:
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
```

DON'T: Commit `.env` with real subscription IDs, tenant IDs, or endpoints.

---

## HYG-3: Dead Code (HIGH)

**Pattern:** Remove unused files, functions, imports, and commented-out code.

DON'T:
```typescript
// import { OldClient } from '@azure/old-package';
// async function oldImplementation() { ... }
```

Also remove unreferenced infrastructure files (e.g., `abbreviations.json` not used by any Bicep).

---

## HYG-4: LICENSE File (HIGH)

**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

```
MIT License
Copyright (c) Microsoft Corporation.
```

---

## HYG-5: Repository Governance Files (MEDIUM)

**Pattern:** Include or reference CONTRIBUTING.md, SECURITY.md, CODEOWNERS.

DO:
```markdown
## Contributing
This project welcomes contributions. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Security
See [SECURITY.md](SECURITY.md).
```
