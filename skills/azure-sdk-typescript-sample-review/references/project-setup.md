# Project Setup Rules (SETUP-1 through SETUP-8)

Rules for TypeScript project configuration, dependencies, and environment.

## SETUP-1: ESM Configuration (CRITICAL)

**Pattern:** Use ES Modules with correct tsconfig settings.

DO:
```json
// package.json
{
  "type": "module",
  "engines": { "node": ">=20.0.0" }
}
```
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*.ts"]
}
```

DON'T:
```json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node"
  }
}
```

---

## SETUP-2: Track 2 Azure SDK Packages (CRITICAL)

**Pattern:** Use `@azure/*` packages (Track 2), not legacy `azure-*` (Track 1).

DO:
```json
{
  "dependencies": {
    "@azure/identity": "^4.5.0",
    "@azure/storage-blob": "^12.25.0",
    "@azure/cosmos": "^4.2.0",
    "@azure/service-bus": "^7.9.0"
  }
}
```

DON'T:
```json
{
  "dependencies": {
    "azure-storage": "^2.10.7",
    "@azure/openai": "^1.0.0"
  }
}
```

The `@azure/openai` package is retired. Use the `openai` SDK with `AzureOpenAI` class instead.

---

## SETUP-3: Phantom Dependencies (CRITICAL)

**Pattern:** Every package in dependencies must be imported somewhere. Every import must have a corresponding dependency.

Audit command:
```bash
npx depcheck
```

---

## SETUP-4: Environment Variable Validation (HIGH)

**Pattern:** Validate all required env vars at startup with descriptive error messages. Use Node.js 20+ `--env-file` flag (not dotenv).

DO:
```typescript
function getRequiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) {
    throw new Error(
      `Missing required environment variable: ${name}\n` +
      `Copy .env.sample to .env and fill in your values.`
    );
  }
  return value;
}

const config = {
  AZURE_STORAGE_ACCOUNT_NAME: getRequiredEnv('AZURE_STORAGE_ACCOUNT_NAME'),
  AZURE_KEYVAULT_URL: getRequiredEnv('AZURE_KEYVAULT_URL'),
};
```

```json
// package.json
{
  "scripts": {
    "start": "node --env-file=.env dist/index.js"
  }
}
```

DON'T:
```typescript
import dotenv from 'dotenv';
dotenv.config();
const accountName = process.env.AZURE_STORAGE_ACCOUNT_NAME;
```

---

## SETUP-5: Lock File (HIGH)

**Pattern:** Commit `package-lock.json`. Use only one package manager.

---

## SETUP-6: TypeScript Strict Mode (MEDIUM)

**Pattern:** Enable `strict: true` in tsconfig.json.

---

## SETUP-7: Package.json Metadata (MEDIUM)

**Pattern:** Include `author`, `license`, `repository`, `engines` fields.

DO:
```json
{
  "name": "azure-storage-blob-sample",
  "version": "1.0.0",
  "description": "Azure Blob Storage quickstart with DefaultAzureCredential",
  "author": "Microsoft",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/Azure-Samples/azure-storage-blob-samples"
  },
  "engines": { "node": ">=20.0.0" }
}
```

---

## SETUP-8: Package Legitimacy (MEDIUM)

**Pattern:** Verify all `@azure/*` packages are published by Microsoft. Check npm registry for unexpected publishers.
