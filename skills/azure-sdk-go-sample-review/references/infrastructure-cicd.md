# Infrastructure, azd, Go Idioms, CI/CD & Documentation

README/docs quality, Bicep/Terraform infrastructure, azd integration, Go-specific idioms, CI/CD patterns, and the comprehensive pre-review checklist.

---

## README & Documentation

### DOC-1: Expected Output (CRITICAL)

**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

**DO:**
```markdown
## Expected Output

Run the sample:

    go run .

You should see output similar to:

    Connected to Azure Blob Storage
    Container 'samples' created
    Uploaded blob 'sample.txt' (14 bytes)
    Downloaded blob content: "Hello, Azure!"

> Note: Exact output may vary based on your Azure environment.
```

### DOC-2: Folder Path Links (CRITICAL)

**Pattern:** All internal README links must match actual filesystem paths.

**DO:**
```markdown
## Project Structure

- [`main.go`](./main.go)--Main entry point
- [`config.go`](./config.go)--Configuration loader
- [`infra/main.bicep`](./infra/main.bicep)--Infrastructure template
```

### DOC-3: Troubleshooting Section (MEDIUM)

**Pattern:** Include troubleshooting for common Azure errors.

**DO:**
```markdown
## Troubleshooting

### Authentication Errors

If you see "failed to create credential":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments (see Prerequisites)

### Common Azure SDK Errors

- `ResponseError: ContainerNotFound`: Container must be created first
- `ResponseError: AuthorizationFailure`: Check RBAC role assignments
- `CredentialUnavailableError`: No authentication method available (run `az login`)
```

### DOC-4: Prerequisites Section (HIGH)

**Pattern:** Document all prerequisites clearly.

**DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Go**: Version 1.22 or later ([Download](https://go.dev/dl/))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional)

### Azure Resources

This sample requires:
- **Azure Storage Account** with a blob container
- **Azure Key Vault** (if using secrets)

### Role Assignments

Your Azure identity needs these role assignments:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
```

### DOC-5: Setup Instructions (MEDIUM)

**Pattern:** Provide clear, tested setup instructions.

**DO:**
```markdown
## Setup

### 1. Clone the repository

    git clone https://github.com/Azure-Samples/azure-storage-blob-go-quickstart.git
    cd azure-storage-blob-go-quickstart

### 2. Install dependencies

    go mod download

### 3. Provision Azure resources

    azd up

### 4. Set environment variables

    cp .env.sample .env
    # Edit .env with your values (or source azd outputs)

### 5. Run the sample

    source .env && go run .
```

### DOC-6: Go Version Strategy (LOW)

**Pattern:** Document minimum Go version in both README and go.mod.

### DOC-7: Placeholder Values (MEDIUM)

**Pattern:** READMEs must provide clear instructions for placeholder values with where to find them in Azure Portal.

---

## Infrastructure (Bicep/Terraform)

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)

**Pattern:** Use current stable versions of Azure Verified Modules.

**DO:**
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
    name: 'storage-deployment'
    params: { name: storageAccountName, location: location }
}
// Check latest: https://azure.github.io/Azure-Verified-Modules/
```

### IaC-2: Bicep Parameter Validation (CRITICAL)

**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators.

**DO:**
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

**DON'T:**
```bicep
param aadAdminObjectId string  // No validation--accepts empty string
```

### IaC-3: API Versions (MEDIUM)

**Pattern:** Use current API versions (2023+).

**DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
```

### IaC-4: RBAC Role Assignments (HIGH)

**Pattern:** Create role assignments in Bicep for managed identities.

**DO:**
```bicep
resource storageBlobDataContributorRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
    scope: storageAccount
    name: guid(storageAccount.id, appService.id, 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
    properties: {
        roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'ba92f5b4-2d11-453d-a403-e96b0029c9fe')
        principalId: appService.identity.principalId
        principalType: 'ServicePrincipal'
    }
}
```

**Common role IDs:**
- Storage Blob Data Contributor: `ba92f5b4-2d11-453d-a403-e96b0029c9fe`
- Key Vault Secrets User: `4633458b-17de-408a-b874-0445c86b69e6`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

### IaC-5: Network Security (HIGH)

**Pattern:** Document public vs private endpoint choices.

**DO (Quickstart):**
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
    properties: {
        publicNetworkAccess: 'Enabled'  // OK for quickstart
    }
}
// NOTE: For production, use private endpoints with defaultAction: 'Deny'.
```

### IaC-6: Output Values (MEDIUM)

**Pattern:** Output all values needed by the application using azd naming conventions.

**DO:**
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
```

### IaC-7: Resource Naming Conventions (HIGH)

**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

**DO:**
```bicep
var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

---

## Azure Developer CLI (azd)

### AZD-1: azure.yaml Structure (MEDIUM)

> **FALSE POSITIVE PREVENTION:**
> 1. `services`, `hooks`, and `host` in `azure.yaml` are OPTIONAL. Infrastructure-only samples need only `name` and `metadata`.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.

**DO:**
```yaml
name: azure-storage-blob-go-sample
metadata:
  template: azure-storage-blob-go-sample@0.0.1

services:
  app:
    project: ./
    language: go
    host: containerapp

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)
      go version > /dev/null || (echo "Go not installed" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "Provisioning complete"
      echo "Run 'go run .' to test the sample."
```

### AZD-2: Service Host Types for Go (MEDIUM)

**Pattern:** Choose correct `host` type for Go applications.

**DO:**
```yaml
# Container Apps (most common for Go)
services:
  api:
    project: ./
    language: go
    host: containerapp
    docker:
      path: ./Dockerfile

# Azure Functions (Go via custom handler)
services:
  func:
    project: ./
    language: go
    host: function

# App Service
services:
  web:
    project: ./
    language: go
    host: appservice
```

---

## Go Idioms

### GO-1: Context Propagation (HIGH)

**Pattern:** Always pass `context.Context` as the first parameter. Create contexts with timeouts for Azure operations.

**DO:**
```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
    defer cancel()

    if err := run(ctx); err != nil {
        log.Fatalf("Error: %v", err)
    }
}

func run(ctx context.Context) error {
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return fmt.Errorf("creating credential: %w", err)
    }

    // Pass ctx down--every Azure SDK call accepts context
    return uploadBlob(ctx, cred)
}

func uploadBlob(ctx context.Context, cred *azidentity.DefaultAzureCredential) error {
    client, err := azblob.NewClient(url, cred, nil)
    if err != nil {
        return fmt.Errorf("creating client: %w", err)
    }
    _, err = client.UploadBuffer(ctx, "container", "blob.txt", data, nil)  // ctx propagated
    return err
}
```

**DON'T:**
```go
func uploadBlob(cred *azidentity.DefaultAzureCredential) error {
    // Creating new context deep in call chain--loses timeout/cancellation
    ctx := context.Background()
    client, err := azblob.NewClient(url, cred, nil)
    if err != nil {
        return err
    }
    _, err = client.UploadBuffer(ctx, "container", "blob.txt", data, nil)
    return err
}
```

### GO-2: Interface-Based Design (MEDIUM)

**Pattern:** Accept interfaces, return structs. Use small interfaces for testability.

**DO:**
```go
// Define small interfaces for what you need
type BlobUploader interface {
    UploadBuffer(ctx context.Context, container, blob string, data []byte, opts *azblob.UploadBufferOptions) (azblob.UploadBufferResponse, error)
}

type Service struct {
    uploader BlobUploader
}

func NewService(uploader BlobUploader) *Service {
    return &Service{uploader: uploader}
}

func (s *Service) ProcessData(ctx context.Context, data []byte) error {
    _, err := s.uploader.UploadBuffer(ctx, "results", "output.json", data, nil)
    if err != nil {
        return fmt.Errorf("uploading results: %w", err)
    }
    return nil
}
```

### GO-3: Goroutine Safety (MEDIUM)

**Pattern:** When using goroutines with Azure SDK clients, document thread safety. Most Azure SDK clients are safe for concurrent use.

**DO:**
```go
// Azure SDK clients are safe for concurrent use
func processItemsConcurrently(ctx context.Context, client *azblob.Client, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10)  // Limit concurrency

    for _, item := range items {
        g.Go(func() error {
            data, err := json.Marshal(item)
            if err != nil {
                return fmt.Errorf("marshaling item %s: %w", item.ID, err)
            }
            _, err = client.UploadBuffer(ctx, "container", item.ID+".json", data, nil)
            if err != nil {
                return fmt.Errorf("uploading item %s: %w", item.ID, err)
            }
            return nil
        })
    }

    return g.Wait()
}
```

**DON'T:**
```go
// Don't launch unbounded goroutines
for _, item := range items {
    go func() {
        client.UploadBuffer(ctx, "container", item.ID+".json", data, nil)
        // Unbounded concurrency, no error handling, variable capture bug
    }()
}
```

### GO-4: Structured Logging with slog (LOW)

**Pattern:** Use `log/slog` (Go 1.22+) for structured logging in production-oriented samples. For quickstarts, `fmt.Println` is acceptable.

**DO:**
```go
import "log/slog"

func uploadBlob(ctx context.Context, container, blob string) error {
    slog.Info("uploading blob",
        "container", container,
        "blob", blob,
    )

    _, err := client.UploadBuffer(ctx, container, blob, data, nil)
    if err != nil {
        slog.Error("blob upload failed",
            "container", container,
            "blob", blob,
            "error", err,
        )
        return fmt.Errorf("uploading %s/%s: %w", container, blob, err)
    }

    slog.Info("blob uploaded successfully",
        "container", container,
        "blob", blob,
    )
    return nil
}
```

### GO-5: to.Ptr() Helper for Pointer Values (MEDIUM)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/azcore/to` for creating pointer values.

**DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore/to"

// Use to.Ptr() for pointer fields in Azure SDK structs
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: to.Ptr("gpt-4o"),
    MaxTokens:      to.Ptr[int32](1000),
    Temperature:    to.Ptr[float32](0.7),
}, nil)

// Key Vault key creation
createResp, err := keyClient.CreateKey(ctx, "my-key", azkeys.CreateKeyParameters{
    Kty: to.Ptr(azkeys.KeyTypeRSA),
}, nil)
```

**DON'T:**
```go
// Don't create local variables just for pointers
name := "gpt-4o"
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: &name,  // Verbose--use to.Ptr()
}, nil)
```

### GO-6: LRO/Poller Patterns--runtime.Poller (HIGH)

**Pattern:** Long-running operations (LROs) in the Azure SDK return `*runtime.Poller[T]`. Use `BeginXxx()` methods and `PollUntilDone()`.

**DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore/runtime"

// Start a long-running operation
poller, err := client.BeginCreateOrUpdate(ctx, resourceGroupName, resourceName, parameters, nil)
if err != nil {
    return fmt.Errorf("starting create operation: %w", err)
}

// Wait for completion
result, err := poller.PollUntilDone(ctx, &runtime.PollUntilDoneOptions{
    Frequency: 10 * time.Second,
})
if err != nil {
    return fmt.Errorf("waiting for create operation: %w", err)
}
fmt.Printf("Resource created: %s\n", *result.ID)
```

**DON'T:**
```go
// Don't ignore the poller--operation may not be complete
poller, _ := client.BeginCreateOrUpdate(ctx, rg, name, params, nil)
// Resource may not exist yet!
```

### GO-7: ARM Client Patterns (HIGH)

**Pattern:** Azure Resource Manager (ARM) clients use a `NewClient(subscriptionID, cred, nil)` constructor pattern.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/resources/armresources"
    "github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/compute/armcompute"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// ARM clients take subscriptionID as first parameter
rgClient, err := armresources.NewResourceGroupsClient(subscriptionID, cred, nil)
if err != nil {
    return fmt.Errorf("creating resource groups client: %w", err)
}

// List resource groups
pager := rgClient.NewListPager(nil)
for pager.More() {
    page, err := pager.NextPage(ctx)
    if err != nil {
        return fmt.Errorf("listing resource groups: %w", err)
    }
    for _, rg := range page.Value {
        fmt.Printf("Resource Group: %s (%s)\n", *rg.Name, *rg.Location)
    }
}
```

### GO-8: Sovereign Cloud Configuration (HIGH)

**Pattern:** For Azure Government, Azure China, or other sovereign clouds, configure `cloud.Configuration` in `ClientOptions`.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azcore"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/cloud"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
)

// Azure Government
cred, err := azidentity.NewDefaultAzureCredential(&azidentity.DefaultAzureCredentialOptions{
    ClientOptions: azcore.ClientOptions{
        Cloud: cloud.AzureGovernment,
    },
})
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

blobClient, err := azblob.NewClient(serviceURL, cred, &azblob.ClientOptions{
    ClientOptions: azcore.ClientOptions{
        Cloud: cloud.AzureGovernment,
    },
})
if err != nil {
    return fmt.Errorf("creating blob client: %w", err)
}
```

**DON'T:**
```go
// Don't hardcode government endpoints without setting cloud config
client, err := azblob.NewClient("https://myaccount.blob.core.usgovcloudapi.net/", cred, nil)
// Missing cloud configuration--token audience will be wrong
```

### GO-9: AuthenticationFailedError Handling (MEDIUM)

**Pattern:** Use `errors.As` with `*azcore.ResponseError` to detect authentication failures and provide actionable guidance.

**DO:**
```go
import (
    "errors"
    "net/http"

    "github.com/Azure/azure-sdk-for-go/sdk/azcore"
)

_, err := client.UploadBuffer(ctx, "container", "blob.txt", data, nil)
if err != nil {
    var respErr *azcore.ResponseError
    if errors.As(err, &respErr) {
        if respErr.StatusCode == http.StatusUnauthorized || respErr.StatusCode == http.StatusForbidden {
            fmt.Println("Authentication/authorization failed. Check:")
            fmt.Println("  1. Run 'az login' to refresh credentials")
            fmt.Println("  2. Verify role assignments (e.g., 'Storage Blob Data Contributor')")
            fmt.Println("  3. Check managed identity configuration if deployed to Azure")
        }
    }
    return fmt.Errorf("uploading blob: %w", err)
}
```

### GO-10: ETag / Optimistic Concurrency (MEDIUM)

**Pattern:** Use ETags for optimistic concurrency control with Cosmos DB and Storage.

**DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore"

// Cosmos DB--use ETag for conditional updates
readResp, err := containerClient.ReadItem(ctx, pk, itemID, nil)
if err != nil {
    return fmt.Errorf("reading item: %w", err)
}

// Update only if the item hasn't changed
_, err = containerClient.ReplaceItem(ctx, pk, itemID, updatedData, &azcosmos.ItemOptions{
    IfMatchEtag: &readResp.ETag,
})
if err != nil {
    var respErr *azcore.ResponseError
    if errors.As(err, &respErr) && respErr.StatusCode == http.StatusPreconditionFailed {
        return fmt.Errorf("item was modified by another process--retry: %w", err)
    }
    return fmt.Errorf("replacing item: %w", err)
}

// Blob Storage--conditional upload
_, err = client.UploadBuffer(ctx, "container", "blob.txt", data,
    &azblob.UploadBufferOptions{
        AccessConditions: &azblob.AccessConditions{
            ModifiedAccessConditions: &azblob.ModifiedAccessConditions{
                IfMatch: &currentETag,
            },
        },
    },
)
```

---

## CI/CD & Testing

### CI-1: Build and Vet in CI (HIGH)

**Pattern:** Run `go build`, `go vet`, and `govulncheck` in CI.

**DO:**
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go mod download
      - run: go vet ./...
      - run: go build ./...
      - run: go test ./...
      - name: Run govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
```

### CI-2: Test Patterns (MEDIUM)

**Pattern:** Write table-driven tests. Use `testing.Short()` to skip integration tests.

**DO:**
```go
func TestLoadConfig(t *testing.T) {
    tests := []struct {
        name    string
        envVars map[string]string
        wantErr bool
    }{
        {
            name: "all vars set",
            envVars: map[string]string{
                "AZURE_STORAGE_ACCOUNT_NAME": "testaccount",
                "AZURE_KEYVAULT_URL":         "https://test.vault.azure.net/",
            },
            wantErr: false,
        },
        {
            name:    "missing vars",
            envVars: map[string]string{},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            for k, v := range tt.envVars {
                t.Setenv(k, v)
            }
            _, err := loadConfig()
            if (err != nil) != tt.wantErr {
                t.Errorf("loadConfig() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}

func TestBlobUpload(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }
    // Integration test that requires Azure resources...
}
```

---

## Pre-Review Checklist (Comprehensive)

### Project Setup
- [ ] go.mod specifies Go 1.22+ (`go 1.22` directive)
- [ ] Module path matches repository URL
- [ ] Every dependency is imported somewhere (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`sdk/*`, not `services/*`)
- [ ] `go mod tidy` run (no stale deps)
- [ ] `go vet ./...` passes
- [ ] `go build ./...` compiles without errors
- [ ] go.sum committed
- [ ] `govulncheck ./...` passes (no known CVEs)

### Security & Hygiene
- [ ] `.gitignore` protects `.env`, `.env.*`, binaries, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed
- [ ] Dead code removed (unused files, functions, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)

### Azure SDK Patterns
- [ ] `azidentity.NewDefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Token refresh implemented for long-running operations
- [ ] Pagination handled with `pager.More()` / `pager.NextPage()`
- [ ] Resource cleanup with `defer client.Close(ctx)`

### Error Handling
- [ ] Every error return checked (`if err != nil`)
- [ ] Errors wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] `azcore.ResponseError` type assertions for Azure errors
- [ ] Context cancellation handled properly

### Go Idioms
- [ ] `context.Context` passed as first parameter
- [ ] Interfaces used for testability
- [ ] Goroutine concurrency bounded with `errgroup`

### Documentation
- [ ] README "Expected output" from real run (not fabricated)
- [ ] All internal links match actual filesystem paths
- [ ] Prerequisites complete (Go version, subscription, CLI, roles)
- [ ] Setup instructions clear and tested

### Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated
- [ ] API versions current (2023+)
- [ ] RBAC role assignments created for managed identities
- [ ] Output values follow azd naming conventions (`AZURE_*`)

### CI/CD
- [ ] `go vet`, `go build`, `go test` in CI workflow
- [ ] `govulncheck` in CI
