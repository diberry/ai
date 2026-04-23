# Infrastructure, azd, CI/CD, and Documentation (IaC-1 through IaC-7, AZD-1 through AZD-2, CI-1 through CI-2, DOC-1 through DOC-7)

Rules for Bicep/Terraform, Azure Developer CLI, CI/CD pipelines, and README documentation.

## IaC-1: Azure Verified Module Versions (CRITICAL)

**Pattern:** Use current stable AVM versions. Check azure.github.io/Azure-Verified-Modules.

DO:
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = { ... }
module keyVault 'br/public:avm/res/key-vault/vault:0.11.0' = { ... }
module cognitiveServices 'br/public:avm/res/cognitive-services/account:1.0.1' = { ... }
```

---

## IaC-2: Bicep Parameter Validation (CRITICAL)

**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators.

DO:
```bicep
@description('Azure AD admin object ID')
@minLength(36) @maxLength(36)
param aadAdminObjectId string

@allowed(['Standard_LRS', 'Standard_GRS', 'Premium_LRS'])
param storageAccountSku string = 'Standard_LRS'
```

DON'T: Accept empty string for required parameters like admin object IDs.

---

## IaC-3: API Versions (MEDIUM)

**Pattern:** Use current API versions (2023+). Avoid versions older than 2 years.

DO:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { ... }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { ... }
```

---

## IaC-4: RBAC Role Assignments (HIGH)

**Pattern:** Create role assignments in Bicep for managed identities.

DO:
```bicep
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

Common role IDs:
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`
- Cosmos DB Account Reader: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`

---

## IaC-5: Network Security (HIGH)

**Pattern:** Quickstart samples may use public endpoints with a comment. Production samples should use private endpoints.

DO (Quickstart):
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  properties: {
    publicNetworkAccess: 'Enabled'
    networkAcls: { defaultAction: 'Allow' }
  }
}
// NOTE: For production, use private endpoints and set defaultAction: 'Deny'.
```

---

## IaC-6: Output Values (MEDIUM)

**Pattern:** Output all values needed by the application. Follow azd naming (`AZURE_*`).

DO:
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
```

---

## IaC-7: Resource Naming Conventions (HIGH)

**Pattern:** Follow CAF naming: `{prefix}-{service}-{env}`.

DO:
```bicep
var storageAccountName = '${resourcePrefix}st${environment}' // Max 24 chars
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

---

## AZD-1: azure.yaml Structure (MEDIUM)

**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

DO:
```yaml
name: azure-storage-blob-sample
metadata:
  template: azure-storage-blob-sample@0.0.1
services:
  app:
    project: ./
    language: ts
    host: appservice
hooks:
  preprovision:
    shell: sh
    run: az account show > /dev/null || (echo "Not logged in" && exit 1)
  postprovision:
    shell: sh
    run: echo "Provisioning complete"
```

---

## AZD-2: Service Host Types (MEDIUM)

Supported hosts: `appservice`, `function`, `containerapp`, `staticwebapp`, `aks`

---

## CI-1: Type Checking in CI (HIGH)

DO:
```yaml
steps:
  - run: npm ci
  - run: npm audit --audit-level=high
  - run: npm run typecheck  # tsc --noEmit
  - run: npm run build
  - run: npm test
```

---

## CI-2: Dependency Audit (MEDIUM)

**Pattern:** Run `npm audit --audit-level=high` in CI to catch known CVEs.

---

## DOC-1: Expected Output (CRITICAL)

**Pattern:** README "Expected output" must be copy-pasted from actual program runs. Never fabricate output.

---

## DOC-2: Folder Path Links (CRITICAL)

**Pattern:** All internal README links must match actual filesystem paths.

---

## DOC-3: Troubleshooting Section (MEDIUM)

**Pattern:** Include troubleshooting for common Azure errors (auth, firewall, RBAC).

DO:
```markdown
## Troubleshooting
### Authentication Errors
- Run `az login`
- Verify required role assignments
### RBAC Permission Errors
- Storage: "Storage Blob Data Contributor"
- Key Vault: "Key Vault Secrets User"
- Role assignments may take 5-10 minutes to propagate
```

---

## DOC-4: Prerequisites Section (HIGH)

**Pattern:** Document all prerequisites: Azure subscription, CLI tools, role assignments, services.

---

## DOC-5: Setup Instructions (MEDIUM)

**Pattern:** Provide clear, tested setup steps: clone, install, provision, run.

---

## DOC-6: Node.js Version Strategy (LOW)

**Pattern:** Document minimum Node.js version in both README and package.json engines.

---

## DOC-7: Placeholder Values (MEDIUM)

**Pattern:** Placeholder values in README must have clear replacement instructions with examples of where to find values in Azure Portal.

---

## Pre-Review Checklist

### Project Setup
- ESM configuration correct
- Package.json has author, license, repository, engines
- No phantom dependencies
- Track 2 Azure SDK packages
- Environment variables validated
- TypeScript strict: true
- package-lock.json committed
- npm audit passes

### Security and Hygiene
- .gitignore protects .env, node_modules, dist
- .env.sample provided
- No live credentials committed
- Dead code removed
- LICENSE file present (MIT)
- CONTRIBUTING.md, SECURITY.md referenced

### Azure SDK Patterns
- DefaultAzureCredential used
- Credential cached and reused
- Token refresh for long-running ops
- Pagination handled completely
- Resource cleanup in try/finally

### Documentation
- Expected output from real runs
- All links match filesystem paths
- Prerequisites complete
- Troubleshooting section present
- Setup instructions tested

### Infrastructure (if applicable)
- AVM versions current
- Bicep parameters validated
- API versions current (2023+)
- RBAC role assignments created
- Resource naming follows CAF
- azure.yaml complete with hooks
