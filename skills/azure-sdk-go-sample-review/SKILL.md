---
name: "azure-sdk-go-sample-review"
description: "Comprehensive review checklist for Azure SDK Go code samples covering project setup, Azure SDK client patterns, authentication, data services, messaging, AI services, Key Vault, infrastructure, documentation, and sample hygiene."
domain: "code-review"
confidence: "high"
source: "earned -- adapted from TypeScript review skill patterns, generalized for Go Azure SDK ecosystem"
---

## Context

Use this skill when reviewing **Go code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general Go review—it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (`github.com/Azure/azure-sdk-for-go/sdk/*` packages, client construction, pipeline options)
- **Authentication patterns** (`azidentity.NewDefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Service Bus, Key Vault, AI services)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)
- **Go idioms** (error handling with `if err != nil`, context propagation, interfaces, goroutine safety)

This skill captures patterns and anti-patterns for Azure SDK Go samples, adapted from comprehensive reviews across the Azure SDK ecosystem.

**Total rules: 75** (11 CRITICAL, 24 HIGH, 32 MEDIUM, 8 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **go.mod**: Uses `github.com/Azure/azure-sdk-for-go/sdk/*` packages (not legacy `github.com/Azure/azure-sdk-for-go/services/*`)
- [ ] **Authentication**: Uses `azidentity.NewDefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `.env`, `.env.*`, vendor/, binaries
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: `govulncheck ./...` passes with no known vulnerabilities
- [ ] **Go version**: go.mod specifies Go 1.22+ (`go 1.22` directive)
- [ ] **Error handling**: Every error return is checked (`if err != nil`)
- [ ] **Resource cleanup**: Clients properly closed with `defer` statements
- [ ] **go.sum**: Committed (not .gitignored)
- [ ] **Imports work**: `go build ./...` compiles without errors
- [ ] **go vet**: `go vet ./...` passes with no warnings
- [ ] **Sample runs**: `go run .` executes without crashes

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets**—Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication**—No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling**—Unchecked error returns, discarded errors with `_`, silent failures
4. **Broken imports**—Missing dependencies, incorrect import paths, module not found errors
5. **Security vulnerabilities**—`govulncheck` shows known CVEs
6. **Missing LICENSE**—No LICENSE file at ANY level of repo hierarchy (MIT required for Azure Samples org). ⚠️ Check repo root before flagging.
7. **.env file committed**—Live credentials in version control. ⚠️ Verify with `git ls-files .env`—a .env on disk but in .gitignore is NOT committed.
8. **Legacy SDK packages**—Uses `github.com/Azure/azure-sdk-for-go/services/*` instead of `github.com/Azure/azure-sdk-for-go/sdk/*`

---

## 1. Project Setup & Configuration

**What this section covers:** Module structure, Go version, dependency management, environment variables, and tooling. These foundational patterns ensure samples build correctly and run reliably across environments.

### PS-1: Go Version (HIGH)
**Pattern:** Target Go 1.22+ in go.mod. Use current stable Go features.

✅ **DO:**
```go
// go.mod
module github.com/Azure-Samples/azure-storage-blob-go-quickstart

go 1.22

require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0
)
```

❌ **DON'T:**
```go
// go.mod
module github.com/Azure-Samples/azure-storage-blob-go-quickstart

go 1.18  // ❌ Too old—missing slog, slices, and other modern features
```

**Why:** Go 1.22+ provides `log/slog` for structured logging, improved generics, better error handling, and fixes the loop variable capture bug. Older versions lack features expected in modern samples.

---

### PS-2: go.mod Module Path (MEDIUM)
**Pattern:** Use a descriptive module path that matches the repository location. Include Go version and complete metadata.

✅ **DO:**
```go
// go.mod
module github.com/Azure-Samples/azure-storage-blob-go-quickstart

go 1.22

require (
    github.com/Azure/azure-sdk-for-go/sdk/azcore v1.14.0
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0
)
```

❌ **DON'T:**
```go
// go.mod
module my-sample  // ❌ Not a valid Go module path, no repository location

go 1.22
```

**Why:** Module paths should match the repository URL for `go get` to work correctly and for discoverability.

---

### PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`github.com/Azure/azure-sdk-for-go/sdk/*`).

✅ **DO:**
```go
// go.mod
require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0   // ✅ Used in main.go
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0 // ✅ Used in storage.go
    github.com/Azure/azure-sdk-for-go/sdk/azcore v1.14.0       // ✅ Used for policy.ClientOptions
)
```

❌ **DON'T:**
```go
require (
    github.com/Azure/azure-sdk-for-go/services/storage/mgmt/2021-09-01/storage v0.0.0  // ❌ Track 1 (legacy)
    github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus v1.7.0                 // ❌ Listed but never imported
    github.com/joho/godotenv v1.5.1                                                      // ❌ Not needed—use os.Getenv
)
```

**Why:** Phantom dependencies inflate the module graph, slow builds, and confuse users about what the sample actually uses.

---

### PS-4: Azure SDK Package Naming (HIGH)
**Pattern:** Use Track 2 packages (`github.com/Azure/azure-sdk-for-go/sdk/*`) not Track 1 legacy packages (`github.com/Azure/azure-sdk-for-go/services/*`).

✅ **DO (Track 2):**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets"
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
    "github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos"
    "github.com/Azure/azure-sdk-for-go/sdk/data/aztables"
    "github.com/Azure/azure-sdk-for-go/sdk/ai/azopenai"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/to"
)
```

❌ **DON'T (Track 1 Legacy):**
```go
import (
    "github.com/Azure/azure-sdk-for-go/services/storage/mgmt/2021-09-01/storage"  // ❌ Track 1
    "github.com/Azure/azure-storage-blob-go/azblob"                                // ❌ Deprecated standalone
    "github.com/Azure/go-autorest/autorest"                                         // ❌ Legacy auth
)
```

**Why:** Track 2 SDKs (`sdk/*`) are current generation with consistent APIs, `azcore` pipeline, and active maintenance. Track 1 (`services/*`) and standalone modules (`azure-storage-blob-go`) are legacy.

---

### PS-5: Environment Variables (MEDIUM)
**Pattern:** Use `os.Getenv` with validation for required variables. Provide `.env.sample` with placeholder values.

✅ **DO:**
```go
package main

import (
    "fmt"
    "os"
    "strings"
)

type Config struct {
    StorageAccountName string
    KeyVaultURL        string
    CosmosEndpoint     string
}

func loadConfig() (*Config, error) {
    required := map[string]string{
        "AZURE_STORAGE_ACCOUNT_NAME": os.Getenv("AZURE_STORAGE_ACCOUNT_NAME"),
        "AZURE_KEYVAULT_URL":         os.Getenv("AZURE_KEYVAULT_URL"),
        "AZURE_COSMOS_ENDPOINT":      os.Getenv("AZURE_COSMOS_ENDPOINT"),
    }

    var missing []string
    for key, val := range required {
        if val == "" {
            missing = append(missing, key)
        }
    }

    if len(missing) > 0 {
        return nil, fmt.Errorf(
            "missing required environment variables: %s\n"+
                "Create a .env file and source it, or set them in your environment.\n"+
                "See .env.sample for required variables",
            strings.Join(missing, ", "),
        )
    }

    return &Config{
        StorageAccountName: required["AZURE_STORAGE_ACCOUNT_NAME"],
        KeyVaultURL:        required["AZURE_KEYVAULT_URL"],
        CosmosEndpoint:     required["AZURE_COSMOS_ENDPOINT"],
    }, nil
}
```

❌ **DON'T:**
```go
// ❌ Don't silently use defaults for Azure resources
storageAccount := os.Getenv("AZURE_STORAGE_ACCOUNT_NAME")
if storageAccount == "" {
    storageAccount = "devstoreaccount1"  // ❌ Silent fallback hides misconfiguration
}

// ❌ Don't use godotenv when os.Getenv works fine
import "github.com/joho/godotenv"
godotenv.Load()
```

**Why:** Explicit validation catches misconfiguration at startup rather than at first Azure API call, saving debugging time.

---

### PS-6: Go Vet / staticcheck (MEDIUM)
**Pattern:** Ensure `go vet ./...` and ideally `staticcheck ./...` pass with zero warnings.

✅ **DO:**
```bash
# Before submitting sample
go vet ./...
staticcheck ./...
```

```go
// ✅ Clean code—no vet warnings
func processItem(ctx context.Context, item Item) error {
    if item.ID == "" {
        return fmt.Errorf("item ID is required")
    }
    // Process...
    return nil
}
```

❌ **DON'T:**
```go
// ❌ go vet catches this: Printf format %d has arg of wrong type string
fmt.Printf("Processing item %d\n", item.Name)

// ❌ go vet catches this: unreachable code
return nil
fmt.Println("Done")
```

**Why:** `go vet` catches real bugs. `staticcheck` finds additional issues like unused code and deprecated API usage.

---

### PS-7: .editorconfig / gofmt (LOW)
**Pattern:** Include `.editorconfig` for consistent formatting. Always run `gofmt` or `goimports`.

✅ **DO:**
```ini
# .editorconfig
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.go]
indent_style = tab
indent_size = 4

[*.md]
trim_trailing_whitespace = false
```

**Why:** Go enforces tab indentation via `gofmt`. The `.editorconfig` ensures editors match before formatting runs.

---

### PS-8: go.sum Committed (HIGH)
**Pattern:** Commit `go.sum` for reproducible builds. Never gitignore it.

✅ **DO:**
```gitignore
# .gitignore
.env
.env.*
!.env.sample
!.env.example

vendor/
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out
.azure/

# ✅ go.sum is COMMITTED (not in .gitignore)
```

❌ **DON'T:**
```gitignore
# ❌ Don't ignore go.sum
go.sum
```

**Why:** `go.sum` contains cryptographic checksums for every dependency. Without it, builds are not reproducible and `go mod verify` fails.

---

### PS-9: CVE Scanning—govulncheck (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. `govulncheck` must pass.

✅ **DO:**
```bash
# Install govulncheck
go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan for vulnerabilities
govulncheck ./...

# Expected: No vulnerabilities found.
```

❌ **DON'T:**
```bash
# ❌ Don't ignore vulnerability warnings
govulncheck ./...
# Vulnerability #1: GO-2024-1234
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Package Legitimacy Check (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `github.com/Azure/azure-sdk-for-go/sdk/*` paths. Watch for typosquatting.

✅ **DO:**
```go
require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0    // ✅ Official
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0 // ✅ Official
    github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos v1.1.0  // ✅ Official
)
```

❌ **DON'T:**
```go
require (
    github.com/azure/azure-sdk-for-go/sdk/azidentity v1.7.0  // ❌ Wrong case (azure vs Azure)
    github.com/Azure/azure-sdk-go/sdk/storage v1.0.0          // ❌ Wrong module path
    github.com/azuresdk/storage-blob v1.0.0                   // ❌ Not official
)
```

**Check:** All Azure SDK packages must be under `github.com/Azure/azure-sdk-for-go/sdk/`. Verify on pkg.go.dev.

---

### PS-11: Dependency Tidying—go mod tidy (MEDIUM)
**Pattern:** Run `go mod tidy` before submitting. Ensures go.mod and go.sum are clean.

✅ **DO:**
```bash
# Before submitting sample
go mod tidy
go mod verify

# Verify no changes after tidy
git diff go.mod go.sum
# Should show no changes if already tidy
```

❌ **DON'T:**
```bash
# ❌ Don't submit without tidying
# go.mod has unused dependencies
# go.sum has stale checksums
```

**Why:** `go mod tidy` removes unused dependencies, adds missing ones, and updates `go.sum`. It ensures the module graph is minimal and correct.

---

## 2. Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

### AZ-1: Client Construction with azidentity (HIGH)
**Pattern:** Use `azidentity.NewDefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

✅ **DO:**
```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets"
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
    "github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos"
)

func main() {
    // ✅ Cache credential instance—reuse across clients
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        log.Fatalf("failed to create credential: %v", err)
    }

    ctx := context.Background()

    // ✅ Storage Blob
    blobClient, err := azblob.NewClient(
        fmt.Sprintf("https://%s.blob.core.windows.net/", accountName),
        cred,
        nil,
    )
    if err != nil {
        log.Fatalf("failed to create blob client: %v", err)
    }

    // ✅ Key Vault Secrets
    secretClient, err := azsecrets.NewClient(config.KeyVaultURL, cred, nil)
    if err != nil {
        log.Fatalf("failed to create secret client: %v", err)
    }

    // ✅ Service Bus
    sbClient, err := azservicebus.NewClient(
        fmt.Sprintf("%s.servicebus.windows.net", namespace),
        cred,
        nil,
    )
    if err != nil {
        log.Fatalf("failed to create service bus client: %v", err)
    }
    defer sbClient.Close(ctx)

    // ✅ Cosmos DB
    cosmosClient, err := azcosmos.NewClient(config.CosmosEndpoint, cred, nil)
    if err != nil {
        log.Fatalf("failed to create cosmos client: %v", err)
    }
}
```

❌ **DON'T:**
```go
// ❌ Don't use connection strings in samples (prefer AAD auth)
client, err := azblob.NewClientFromConnectionString(connectionString, nil)

// ❌ Don't use shared keys
cred, _ := azblob.NewSharedKeyCredential(accountName, accountKey)

// ❌ Don't recreate credential for each client
cred1, _ := azidentity.NewDefaultAzureCredential(nil)
blobClient, _ := azblob.NewClient(url, cred1, nil)
cred2, _ := azidentity.NewDefaultAzureCredential(nil)  // ❌ Wasteful
secretClient, _ := azsecrets.NewClient(url, cred2, nil)
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

### AZ-2: Client Options—azcore.ClientOptions, Retry (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and logging for production-ready samples.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "time"
)

clientOptions := &azblob.ClientOptions{
    ClientOptions: policy.ClientOptions{
        Retry: policy.RetryOptions{
            MaxRetries:    3,
            RetryDelay:    1 * time.Second,
            MaxRetryDelay: 30 * time.Second,
        },
        // For debugging:
        // Logging: policy.LogOptions{
        //     IncludeBody: true,
        // },
    },
}

blobClient, err := azblob.NewClient(serviceURL, cred, clientOptions)
if err != nil {
    return fmt.Errorf("creating blob client: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't use extremely short timeouts or misconfigure retry policies
clientOptions := &azblob.ClientOptions{
    ClientOptions: policy.ClientOptions{
        Retry: policy.RetryOptions{
            MaxRetries: 0,  // ❌ No retries—Azure calls are inherently transient
        },
    },
}
```

> **Note:** Passing `nil` for `ClientOptions` is idiomatic Go and acceptable for quickstarts. The SDK provides sensible defaults (3 retries, exponential backoff). Only flag missing options in production-oriented samples that should demonstrate explicit configuration.

**Why:** Default retry policies are reasonable but samples should demonstrate configuring them explicitly for production guidance.

---

### AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

// ✅ For samples: DefaultAzureCredential (works locally + cloud)
cred, err := azidentity.NewDefaultAzureCredential(nil)

// ✅ For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
cred, err := azidentity.NewManagedIdentityCredential(nil)

// ✅ User-assigned (when multiple identities needed)
cred, err := azidentity.NewManagedIdentityCredential(&azidentity.ManagedIdentityCredentialOptions{
    ID: azidentity.ClientID(os.Getenv("AZURE_CLIENT_ID")),
})

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which
// > automatically uses the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App has a managed identity assigned with
// > appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```go
// ❌ Don't hardcode service principal credentials in samples
cred, err := azidentity.NewClientSecretCredential(tenantID, clientID, clientSecret, nil)
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

### AZ-4: Token Management—azcore.TokenCredential (CRITICAL)
**Pattern:** For services without official SDK (SQL, custom APIs), get tokens with `GetToken()`. Tokens expire after ~1 hour—implement refresh logic for long-running samples.

✅ **DO:**
```go
import (
    "context"
    "time"

    "github.com/Azure/azure-sdk-for-go/sdk/azcore"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

func getAzureSQLToken(ctx context.Context, cred azcore.TokenCredential) (azcore.AccessToken, error) {
    return cred.GetToken(ctx, policy.TokenRequestOptions{
        Scopes: []string{"https://database.windows.net/.default"},
    })
}

// ✅ Token refresh for long-running operations
func refreshableToken(ctx context.Context, cred azcore.TokenCredential) (string, error) {
    token, err := getAzureSQLToken(ctx, cred)
    if err != nil {
        return "", fmt.Errorf("acquiring SQL token: %w", err)
    }

    // Check expiration before use (refresh if < 5 minutes remaining)
    if time.Until(token.ExpiresOn) < 5*time.Minute {
        token, err = getAzureSQLToken(ctx, cred)
        if err != nil {
            return "", fmt.Errorf("refreshing SQL token: %w", err)
        }
    }

    return token.Token, nil
}
```

❌ **DON'T:**
```go
// ❌ CRITICAL: Don't acquire token once and use for hours
token, _ := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://database.windows.net/.default"},
})
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

**Why:** Azure tokens expire after approximately 1 hour. Samples processing large datasets MUST refresh tokens before expiration.

---

### AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive credentials for CI.

✅ **DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

// ✅ For CI/CD environments (no interactive prompts)
cred, err := azidentity.NewDefaultAzureCredential(&azidentity.DefaultAzureCredentialOptions{
    DisableInstanceDiscovery: false,
    TenantID:                 os.Getenv("AZURE_TENANT_ID"),  // Optional: scope to specific tenant
})

// ✅ Document the credential chain in README
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
// > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > 2. Workload identity (Azure Kubernetes Service)
// > 3. Managed identity (App Service, Functions, Container Apps)
// > 4. Azure CLI (`az login`)
// > 5. Azure Developer CLI (`azd auth login`)
```

❌ **DON'T:**
```go
// ❌ Don't ignore credential errors
cred, _ := azidentity.NewDefaultAzureCredential(nil)
```

**Why:** Understanding the credential chain helps users debug authentication failures in different environments.

---

### AZ-6: Resource Cleanup—defer client.Close() (MEDIUM)
**Pattern:** Samples must properly close clients using `defer`. Always pass a context to `Close()`.

✅ **DO:**
```go
func run(ctx context.Context) error {
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return fmt.Errorf("creating credential: %w", err)
    }

    // ✅ Service Bus—close with defer
    sbClient, err := azservicebus.NewClient(namespace, cred, nil)
    if err != nil {
        return fmt.Errorf("creating service bus client: %w", err)
    }
    defer sbClient.Close(ctx)

    sender, err := sbClient.NewSender("myqueue", nil)
    if err != nil {
        return fmt.Errorf("creating sender: %w", err)
    }
    defer sender.Close(ctx)

    err = sender.SendMessage(ctx, &azservicebus.Message{
        Body: []byte(`{"orderId": 1}`),
    }, nil)
    if err != nil {
        return fmt.Errorf("sending message: %w", err)
    }

    return nil
}
```

❌ **DON'T:**
```go
// ❌ Don't forget to close clients
sbClient, err := azservicebus.NewClient(namespace, cred, nil)
sender, _ := sbClient.NewSender("myqueue", nil)
sender.SendMessage(ctx, &azservicebus.Message{Body: []byte("hello")}, nil)
// ❌ Client and sender never closed (resource leak)
```

**Why:** Unclosed clients leak connections and goroutines. `defer` ensures cleanup even when errors occur.

---

### AZ-7: Pagination with runtime.Pager (HIGH)
**Pattern:** Use `runtime.Pager` for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/runtime"
)

// ✅ Blob Storage—iterate all pages
pager := containerClient.NewListBlobsFlatPager(nil)
for pager.More() {
    page, err := pager.NextPage(ctx)
    if err != nil {
        return fmt.Errorf("listing blobs: %w", err)
    }
    for _, blob := range page.Segment.BlobItems {
        fmt.Printf("Blob: %s\n", *blob.Name)
    }
}

// ✅ Cosmos DB—iterate query results
queryPager := containerClient.NewQueryItemsPager(
    "SELECT * FROM c WHERE c.category = @category",
    azcosmos.NewPartitionKeyString("electronics"),
    &azcosmos.QueryOptions{
        QueryParameters: []azcosmos.QueryParameter{
            {Name: "@category", Value: "electronics"},
        },
    },
)
for queryPager.More() {
    response, err := queryPager.NextPage(ctx)
    if err != nil {
        return fmt.Errorf("querying cosmos: %w", err)
    }
    for _, item := range response.Items {
        fmt.Printf("Item: %s\n", string(item))
    }
}

// ✅ Key Vault—list all secrets
secretPager := secretClient.NewListSecretPropertiesPager(nil)
for secretPager.More() {
    page, err := secretPager.NextPage(ctx)
    if err != nil {
        return fmt.Errorf("listing secrets: %w", err)
    }
    for _, secret := range page.Value {
        fmt.Printf("Secret: %s\n", secret.ID.Name())
    }
}
```

❌ **DON'T:**
```go
// ❌ Only gets first page
pager := containerClient.NewListBlobsFlatPager(nil)
page, _ := pager.NextPage(ctx)
for _, blob := range page.Segment.BlobItems {
    fmt.Println(*blob.Name)
}
// ❌ Stops after first page—may miss thousands of blobs
```

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns, API versioning, embeddings, and chat completions using the Azure SDK for Go.

### AI-1: Azure OpenAI Client—azopenai (HIGH)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/ai/azopenai` with `DefaultAzureCredential`.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/ai/azopenai"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azopenai.NewClient(
    config.AzureOpenAIEndpoint,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating openai client: %w", err)
}

// ✅ Chat completion
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: to.Ptr("gpt-4o"),
    Messages: []azopenai.ChatRequestMessageClassification{
        &azopenai.ChatRequestUserMessage{
            Content: azopenai.NewChatRequestUserMessageContent("Hello!"),
        },
    },
}, nil)
if err != nil {
    return fmt.Errorf("chat completion: %w", err)
}

fmt.Println(*resp.Choices[0].Message.Content)

// ✅ Embeddings
embResp, err := client.GetEmbeddings(ctx, azopenai.EmbeddingsOptions{
    DeploymentName: to.Ptr("text-embedding-3-small"),
    Input:          []string{"Sample text to embed"},
}, nil)
if err != nil {
    return fmt.Errorf("getting embeddings: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't use API keys in samples (prefer AAD)
client, err := azopenai.NewClientWithKeyCredential(
    endpoint,
    azcore.NewKeyCredential(apiKey),  // ❌ Use DefaultAzureCredential
    nil,
)
```

**Why:** AAD authentication is more secure and consistent with other Azure SDK patterns. API keys should be avoided in samples.

---

### AI-2: API Version Documentation (LOW)
**Pattern:** When API versions are configurable, document the version and link to deprecation schedule.

✅ **DO:**
```go
// The Azure OpenAI Go SDK manages API versions internally.
// For REST-based calls, specify the version explicitly:
// API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
const apiVersion = "2024-10-21"
```

---

### AI-3: Document Intelligence (MEDIUM)
**Pattern:** Use the Azure SDK for Go Document Intelligence package with `DefaultAzureCredential`.

> **⚠️ Package Path:** The Go SDK package for Document Intelligence may be at
> `github.com/Azure/azure-sdk-for-go/sdk/ai/documentintelligence/azdocumentintelligence`.
> Verify the exact import path on [pkg.go.dev](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/ai/)
> before using—the package may be in preview or not yet published.

✅ **DO:**
```go
import (
    // Verify package path on pkg.go.dev — may be in preview
    "github.com/Azure/azure-sdk-for-go/sdk/ai/documentintelligence/azdocumentintelligence"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azdocumentintelligence.NewClient(
    config.DocumentIntelligenceEndpoint,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating doc intelligence client: %w", err)
}

poller, err := client.BeginAnalyzeDocument(ctx, "prebuilt-invoice", documentContent, nil)
if err != nil {
    return fmt.Errorf("starting analysis: %w", err)
}

result, err := poller.PollUntilDone(ctx, nil)
if err != nil {
    return fmt.Errorf("analyzing document: %w", err)
}
```

---

### AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause runtime errors.

✅ **DO:**
```go
const (
    embeddingModel  = "text-embedding-3-small"  // 1536 dimensions
    vectorDimension = 1536
)

// Validate embedding size after retrieval
embedding := embResp.Data[0].Embedding
if len(embedding) != vectorDimension {
    return fmt.Errorf(
        "embedding dimension mismatch: expected %d, got %d (model: %s)",
        vectorDimension, len(embedding), embeddingModel,
    )
}
```

❌ **DON'T:**
```go
// ❌ Don't assume dimension without validation
embedding := embResp.Data[0].Embedding
insertEmbedding(embedding)  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

### AI-5: Azure Speech SDK (MEDIUM)
**Pattern:** The Azure Speech SDK for Go uses the Cognitive Services Speech SDK (CGo bindings) or REST APIs. For Go samples, the REST-based approach is more portable.

> **⚠️ Note:** The Go Speech SDK requires CGo and platform-specific native libraries.
> For quickstarts, consider using the REST API with `azidentity` for authentication.
> Check [Azure Speech SDK documentation](https://learn.microsoft.com/azure/ai-services/speech-service/)
> for current Go support status.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

// ✅ Get token for Speech Service via REST
cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

token, err := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://cognitiveservices.azure.com/.default"},
})
if err != nil {
    return fmt.Errorf("acquiring speech token: %w", err)
}

// Use token.Token with Speech REST API
```

**Why:** Speech-to-text and text-to-speech are important AI scenarios. Documenting the Go authentication pattern helps users integrate with the REST API.

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization.

### DB-1: Cosmos DB—azcosmos Patterns (HIGH)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos` with AAD credentials. Handle partitioned containers properly.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azcosmos.NewClient(config.CosmosEndpoint, cred, nil)
if err != nil {
    return fmt.Errorf("creating cosmos client: %w", err)
}

containerClient, err := client.NewContainer("mydb", "mycontainer")
if err != nil {
    return fmt.Errorf("getting container: %w", err)
}

// ✅ Query with partition key
pk := azcosmos.NewPartitionKeyString("electronics")
queryPager := containerClient.NewQueryItemsPager(
    "SELECT * FROM c WHERE c.category = @category",
    pk,
    &azcosmos.QueryOptions{
        QueryParameters: []azcosmos.QueryParameter{
            {Name: "@category", Value: "electronics"},
        },
    },
)

// ✅ Point read (most efficient)
pk = azcosmos.NewPartitionKeyString("partition-key-value")
resp, err := containerClient.ReadItem(ctx, pk, "item-id", nil)
if err != nil {
    return fmt.Errorf("reading item: %w", err)
}

// ✅ Create item
itemJSON := []byte(`{"id":"item-1","category":"electronics","name":"Laptop"}`)
_, err = containerClient.CreateItem(ctx, pk, itemJSON, nil)
if err != nil {
    return fmt.Errorf("creating item: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't use primary key in samples
client, err := azcosmos.NewClientWithKey(endpoint, accountKey, nil)

// ❌ Don't omit partition key in queries (cross-partition queries are expensive)
queryPager := containerClient.NewQueryItemsPager(
    "SELECT * FROM c",
    azcosmos.PartitionKey{},  // ❌ Empty partition key
    nil,
)
```

---

### DB-2: Azure SQL with go-mssqldb (HIGH)
**Pattern:** Use `github.com/microsoft/go-mssqldb` with AAD token authentication. Use `database/sql` standard library interface.

✅ **DO:**
```go
import (
    "database/sql"
    "fmt"

    "github.com/Azure/azure-sdk-for-go/sdk/azcore/policy"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    mssql "github.com/microsoft/go-mssqldb"
    "github.com/microsoft/go-mssqldb/azuread"
)

// ✅ Connect with AAD authentication
func connectSQL(ctx context.Context) (*sql.DB, error) {
    // Option 1: Use azuread driver (recommended)
    connStr := fmt.Sprintf(
        "sqlserver://%s?database=%s&fedauth=ActiveDirectoryDefault",
        config.SQLServer, config.SQLDatabase,
    )
    db, err := sql.Open(azuread.DriverName, connStr)
    if err != nil {
        return nil, fmt.Errorf("opening SQL connection: %w", err)
    }

    if err := db.PingContext(ctx); err != nil {
        return nil, fmt.Errorf("pinging SQL server: %w", err)
    }

    return db, nil
}

// ✅ Use the connection
func queryProducts(ctx context.Context, db *sql.DB, category string) ([]Product, error) {
    rows, err := db.QueryContext(ctx,
        "SELECT [id], [name], [category] FROM [Products] WHERE [category] = @p1",
        sql.Named("p1", category),
    )
    if err != nil {
        return nil, fmt.Errorf("querying products: %w", err)
    }
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        if err := rows.Scan(&p.ID, &p.Name, &p.Category); err != nil {
            return nil, fmt.Errorf("scanning row: %w", err)
        }
        products = append(products, p)
    }
    return products, rows.Err()
}
```

❌ **DON'T:**
```go
// ❌ Don't use SQL authentication with password in samples
connStr := fmt.Sprintf("sqlserver://user:password@%s?database=%s", server, database)

// ❌ Don't ignore rows.Err()
for rows.Next() {
    rows.Scan(&p.ID, &p.Name)
}
// ❌ Missing rows.Err() check—may have encountered error during iteration
```

---

### DB-3: SQL Parameter Safety (HIGH)
**Pattern:** ALWAYS use parameterized queries. Go's `database/sql` supports named parameters via `sql.Named()`. Never build SQL by string concatenation.

✅ **DO:**
```go
// ✅ Parameterized query—safe from SQL injection
rows, err := db.QueryContext(ctx,
    "SELECT [id], [name] FROM [Products] WHERE [category] = @category AND [price] < @maxPrice",
    sql.Named("category", category),
    sql.Named("maxPrice", maxPrice),
)

// ✅ Dynamic identifiers—use bracket quoting (validated against allowlist)
validTables := map[string]bool{"Products": true, "Orders": true}
if !validTables[tableName] {
    return fmt.Errorf("invalid table name: %s", tableName)
}
query := fmt.Sprintf("SELECT [id], [name] FROM [%s] WHERE [id] = @id", tableName)
rows, err := db.QueryContext(ctx, query, sql.Named("id", itemID))
```

❌ **DON'T:**
```go
// ❌ CRITICAL: SQL injection vulnerability
query := fmt.Sprintf("SELECT * FROM Products WHERE category = '%s'", category)
rows, err := db.QueryContext(ctx, query)

// ❌ Don't use Sprintf for values
query := fmt.Sprintf("INSERT INTO [Products] VALUES ('%s', '%s')", name, category)
```

**Why:** Go's `database/sql` supports parameterized queries natively, but developers must explicitly use `sql.Named()` or positional `$1` params. String concatenation is the default instinct in Go and MUST be caught.

---

### DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations or transactions for multiple rows.

✅ **DO:**
```go
// ✅ SQL batch insert with transaction
func batchInsertProducts(ctx context.Context, db *sql.DB, products []Product) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("beginning transaction: %w", err)
    }
    defer tx.Rollback()

    stmt, err := tx.PrepareContext(ctx,
        "INSERT INTO [Products] ([id], [name], [category]) VALUES (@p1, @p2, @p3)",
    )
    if err != nil {
        return fmt.Errorf("preparing statement: %w", err)
    }
    defer stmt.Close()

    for _, p := range products {
        if _, err := stmt.ExecContext(ctx, sql.Named("p1", p.ID), sql.Named("p2", p.Name), sql.Named("p3", p.Category)); err != nil {
            return fmt.Errorf("inserting product %s: %w", p.ID, err)
        }
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("committing transaction: %w", err)
    }
    return nil
}
```

✅ **DO (Cosmos DB—transactional batch):**
```go
// ✅ Cosmos transactional batch (same partition key, max 100 ops)
pk := azcosmos.NewPartitionKeyString("electronics")

batch := containerClient.NewTransactionalBatch(pk)
batch.CreateItem([]byte(`{"id":"1","category":"electronics","name":"Laptop"}`), nil)
batch.CreateItem([]byte(`{"id":"2","category":"electronics","name":"Mouse"}`), nil)
batch.UpsertItem([]byte(`{"id":"3","category":"electronics","name":"Keyboard"}`), nil)

batchResp, err := containerClient.ExecuteTransactionalBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("executing batch: %w", err)
}
// Check batchResp.Success for batch-level success
```

❌ **DON'T:**
```go
// ❌ Row-by-row INSERT (50 round trips for 50 products)
for _, p := range products {
    _, err := db.ExecContext(ctx,
        "INSERT INTO [Products] VALUES (@p1, @p2, @p3)",
        sql.Named("p1", p.ID), sql.Named("p2", p.Name), sql.Named("p3", p.Category),
    )
    if err != nil {
        return err
    }
}
```

**Why:** Batch operations reduce round trips and improve performance. 50 individual inserts take 50 round trips; a prepared statement in a transaction is far more efficient.

---

### DB-5: Azure Storage—azblob Patterns (MEDIUM)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/storage/azblob` with `DefaultAzureCredential`.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// ✅ Create blob service client
serviceClient, err := azblob.NewClient(
    fmt.Sprintf("https://%s.blob.core.windows.net/", accountName),
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating blob client: %w", err)
}

// ✅ Upload blob
_, err = serviceClient.UploadBuffer(ctx, "mycontainer", "myblob.txt",
    []byte("Hello, Azure!"), nil)
if err != nil {
    return fmt.Errorf("uploading blob: %w", err)
}

// ✅ Download blob
downloadResp, err := serviceClient.DownloadStream(ctx, "mycontainer", "myblob.txt", nil)
if err != nil {
    return fmt.Errorf("downloading blob: %w", err)
}
defer downloadResp.Body.Close()

data, err := io.ReadAll(downloadResp.Body)
if err != nil {
    return fmt.Errorf("reading blob: %w", err)
}
fmt.Printf("Downloaded: %s\n", string(data))
```

---

### DB-6: SAS Token Fallback (MEDIUM)
**Pattern:** For local development or CI environments where `DefaultAzureCredential` isn't available, provide SAS token fallback with documentation.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

func createBlobClient(accountName string) (*azblob.Client, error) {
    serviceURL := fmt.Sprintf("https://%s.blob.core.windows.net/", accountName)

    if sasToken := os.Getenv("AZURE_STORAGE_SAS_TOKEN"); sasToken != "" {
        // Local dev: SAS token
        fmt.Println("Using SAS token authentication (local dev)")
        return azblob.NewClientWithNoCredential(serviceURL+sasToken, nil)
    }

    // Production: AAD
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return nil, fmt.Errorf("creating credential: %w", err)
    }
    fmt.Println("Using DefaultAzureCredential (AAD)")
    return azblob.NewClient(serviceURL, cred, nil)
}
```

---

### DB-7: UploadStream for Large Files (HIGH)
**Pattern:** Use `UploadStream()` for streaming large files to blob storage. `UploadBuffer()` loads the entire file into memory; `UploadStream()` streams in chunks.

✅ **DO:**
```go
import (
    "os"

    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob/blockblob"
)

// ✅ Stream large files without loading entirely into memory
func uploadLargeFile(ctx context.Context, client *azblob.Client, container, blobName, filePath string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("opening file: %w", err)
    }
    defer file.Close()

    _, err = client.UploadStream(ctx, container, blobName, file,
        &azblob.UploadStreamOptions{
            BlockSize:   4 * 1024 * 1024, // 4 MiB blocks
            Concurrency: 4,               // parallel uploads
        },
    )
    if err != nil {
        return fmt.Errorf("uploading stream: %w", err)
    }
    return nil
}
```

❌ **DON'T:**
```go
// ❌ Don't load large files entirely into memory
data, _ := os.ReadFile("large-file.dat")  // ❌ May OOM for multi-GB files
client.UploadBuffer(ctx, "container", "blob", data, nil)
```

**Why:** `UploadStream` streams data in configurable block sizes with parallel uploads, avoiding out-of-memory errors for large files.

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for queues, topics, and event ingestion. Focus on reliable message handling and proper resource cleanup.

### MSG-1: Service Bus—azservicebus Patterns (HIGH)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus` with `DefaultAzureCredential`.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azservicebus.NewClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace), cred, nil,
)
if err != nil {
    return fmt.Errorf("creating service bus client: %w", err)
}
defer client.Close(ctx)

// ✅ Send messages
sender, err := client.NewSender("myqueue", nil)
if err != nil {
    return fmt.Errorf("creating sender: %w", err)
}
defer sender.Close(ctx)

batch, err := sender.NewMessageBatch(ctx, nil)
if err != nil {
    return fmt.Errorf("creating message batch: %w", err)
}

err = batch.AddMessage(&azservicebus.Message{Body: []byte(`{"orderId":1}`)}, nil)
if err != nil {
    return fmt.Errorf("adding message: %w", err)
}

err = sender.SendMessageBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("sending batch: %w", err)
}

// ✅ Receive messages (MUST complete or abandon)
receiver, err := client.NewReceiverForQueue("myqueue", nil)
if err != nil {
    return fmt.Errorf("creating receiver: %w", err)
}
defer receiver.Close(ctx)

messages, err := receiver.ReceiveMessages(ctx, 10, nil)
if err != nil {
    return fmt.Errorf("receiving messages: %w", err)
}

for _, msg := range messages {
    fmt.Printf("Received: %s\n", string(msg.Body))
    if err := receiver.CompleteMessage(ctx, msg, nil); err != nil {
        // ✅ Abandon on failure—message returns to queue
        _ = receiver.AbandonMessage(ctx, msg, nil)
        return fmt.Errorf("completing message: %w", err)
    }
}
```

❌ **DON'T:**
```go
// ❌ Don't use connection strings in samples
client, err := azservicebus.NewClientFromConnectionString(connStr, nil)

// ❌ Don't forget to complete/abandon messages
for _, msg := range messages {
    fmt.Println(string(msg.Body))  // ❌ Message never completed (will reappear)
}
```

---

### MSG-2: Event Hubs—azeventhubs Patterns (MEDIUM)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/azeventhubs` for event ingestion.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azeventhubs"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// ✅ Producer—send events
producerClient, err := azeventhubs.NewProducerClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace),
    "myeventhub",
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating producer: %w", err)
}
defer producerClient.Close(ctx)

batch, err := producerClient.NewEventDataBatch(ctx, nil)
if err != nil {
    return fmt.Errorf("creating batch: %w", err)
}

err = batch.AddEventData(&azeventhubs.EventData{
    Body: []byte(`{"temperature": 23.5}`),
}, nil)
if err != nil {
    return fmt.Errorf("adding event: %w", err)
}

err = producerClient.SendEventDataBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("sending batch: %w", err)
}

// ✅ Consumer—receive events with processor
consumerClient, err := azeventhubs.NewConsumerClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace),
    "myeventhub",
    azeventhubs.DefaultConsumerGroup,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating consumer: %w", err)
}
defer consumerClient.Close(ctx)
```

---

### MSG-3: Event Grid—azeventgrid Patterns (MEDIUM)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/eventgrid/azeventgrid` for publishing events to Event Grid topics.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/eventgrid/azeventgrid"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azeventgrid.NewClient(config.EventGridEndpoint, cred, nil)
if err != nil {
    return fmt.Errorf("creating event grid client: %w", err)
}
```

> **Note:** Verify exact package path on [pkg.go.dev](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/messaging/eventgrid/)—
> the Event Grid Go SDK may be in preview.

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault with AAD authentication.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets`, `azkeys`, `azcertificates` with `DefaultAzureCredential`.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azkeys"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azcertificates"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// ✅ Secrets
secretClient, err := azsecrets.NewClient(config.KeyVaultURL, cred, nil)
if err != nil {
    return fmt.Errorf("creating secret client: %w", err)
}

// Set a secret
_, err = secretClient.SetSecret(ctx, "db-password", azsecrets.SetSecretParameters{
    Value: to.Ptr("P@ssw0rd123"),
}, nil)
if err != nil {
    return fmt.Errorf("setting secret: %w", err)
}

// Get a secret
resp, err := secretClient.GetSecret(ctx, "db-password", "", nil)
if err != nil {
    return fmt.Errorf("getting secret: %w", err)
}
fmt.Printf("Secret value: %s\n", *resp.Value)

// ✅ Keys (for encryption)
keyClient, err := azkeys.NewClient(config.KeyVaultURL, cred, nil)
if err != nil {
    return fmt.Errorf("creating key client: %w", err)
}

createResp, err := keyClient.CreateKey(ctx, "my-encryption-key", azkeys.CreateKeyParameters{
    Kty: to.Ptr(azkeys.KeyTypeRSA),
}, nil)
if err != nil {
    return fmt.Errorf("creating key: %w", err)
}
fmt.Printf("Key ID: %s\n", *createResp.Key.KID)
```

❌ **DON'T:**
```go
// ❌ Don't hardcode secrets in samples
dbPassword := "P@ssw0rd123"  // ❌ Use Key Vault
```

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations across Azure data services.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Serialize vectors as JSON strings for Azure SQL vector parameters.

✅ **DO (Azure SQL):**
```go
import (
    "database/sql"
    "encoding/json"
)

embedding := []float32{0.1, 0.2, 0.3} // 1536 floats from OpenAI
embJSON, err := json.Marshal(embedding)
if err != nil {
    return fmt.Errorf("marshaling embedding: %w", err)
}

// ✅ Insert with CAST
_, err = db.ExecContext(ctx,
    "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(@p1 AS VECTOR(1536)))",
    sql.Named("p1", string(embJSON)),
)

// ✅ Vector distance query
rows, err := db.QueryContext(ctx, `
    SELECT TOP (@k)
        [id], [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(@searchEmbedding AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC`,
    sql.Named("k", 5),
    sql.Named("searchEmbedding", string(searchEmbJSON)),
)
```

---

### VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires ≥1000 rows. Check row count before creating index.

✅ **DO:**
```go
var rowCount int
err := db.QueryRowContext(ctx,
    fmt.Sprintf("SELECT COUNT(*) FROM [%s]", tableName),
).Scan(&rowCount)
if err != nil {
    return fmt.Errorf("counting rows: %w", err)
}

if rowCount >= 1000 {
    fmt.Printf("✅ %d rows available. Creating DiskANN index...\n", rowCount)
    _, err = db.ExecContext(ctx, fmt.Sprintf(
        "CREATE INDEX [ix_%s_embedding_diskann] ON [%s] ([embedding]) USING DiskANN",
        tableName, tableName,
    ))
    if err != nil {
        return fmt.Errorf("creating DiskANN index: %w", err)
    }
} else {
    fmt.Printf("⚠️ Only %d rows. DiskANN requires ≥1000. Using exact search.\n", rowCount)
}
```

❌ **DON'T:**
```go
// ❌ Create DiskANN index without checking row count
_, err = db.ExecContext(ctx, "CREATE INDEX ... USING DiskANN")
// Fails with: "DiskANN index requires at least 1000 rows"
```

---

## 8. Error Handling

**What this section covers:** Go-specific error handling patterns for Azure SDK code. Go uses error returns instead of exceptions—this is a strength that makes error flows explicit and reviewable.

### ERR-1: Error Wrapping with fmt.Errorf %w (MEDIUM)
**Pattern:** Wrap errors with context using `fmt.Errorf` and the `%w` verb. This preserves the error chain for `errors.Is` and `errors.As`.

✅ **DO:**
```go
import (
    "errors"
    "fmt"
)

func uploadBlob(ctx context.Context, client *azblob.Client, data []byte) error {
    _, err := client.UploadBuffer(ctx, "container", "blob.txt", data, nil)
    if err != nil {
        return fmt.Errorf("uploading blob to container: %w", err)
    }
    return nil
}

// ✅ Callers can unwrap
err := uploadBlob(ctx, client, data)
if err != nil {
    var respErr *azcore.ResponseError
    if errors.As(err, &respErr) {
        fmt.Printf("Azure error code: %s\n", respErr.ErrorCode)
    }
    return fmt.Errorf("blob operation failed: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't use %v—loses error chain
return fmt.Errorf("uploading blob: %v", err)

// ❌ Don't discard errors
client.UploadBuffer(ctx, "container", "blob.txt", data, nil)  // ❌ Error ignored
```

> **Note:** A simple `return err` after a single operation is idiomatic Go and acceptable when the
> function name already provides context (e.g., `func uploadBlob(...) error`). Only flag bare
> `return err` when wrapping would add meaningful debugging context—such as in multi-step functions
> or when the caller can't easily determine which operation failed.

**Why:** `%w` wrapping preserves the full error chain so callers can use `errors.Is` and `errors.As` for programmatic error inspection. `%v` breaks the chain.

---

### ERR-2: ResponseError Type Assertion (MEDIUM)
**Pattern:** Use `errors.As` to extract `azcore.ResponseError` for Azure-specific error details (status code, error code).

✅ **DO:**
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
        switch respErr.StatusCode {
        case http.StatusNotFound:
            fmt.Println("Container not found. Create it first.")
        case http.StatusForbidden:
            fmt.Println("Access denied. Check role assignments:")
            fmt.Println("  - Storage Blob Data Contributor required")
        case http.StatusConflict:
            fmt.Println("Blob already exists.")
        default:
            fmt.Printf("Azure error: %s (HTTP %d)\n", respErr.ErrorCode, respErr.StatusCode)
        }
    }
    return fmt.Errorf("blob upload failed: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't use string matching on error messages
if strings.Contains(err.Error(), "not found") {
    // Fragile—error message may change across SDK versions
}

// ❌ Don't ignore the error type
if err != nil {
    log.Fatal(err)  // ❌ No context, no recovery options
}
```

---

### ERR-3: Context Cancellation Handling (HIGH)
**Pattern:** Check for context cancellation separately from other errors. Provide appropriate messages.

✅ **DO:**
```go
import (
    "context"
    "errors"
)

func processItems(ctx context.Context, items []Item) error {
    for _, item := range items {
        if err := ctx.Err(); err != nil {
            return fmt.Errorf("processing cancelled after %d items: %w", len(items), err)
        }

        if err := processItem(ctx, item); err != nil {
            if errors.Is(err, context.Canceled) {
                return fmt.Errorf("operation cancelled: %w", err)
            }
            if errors.Is(err, context.DeadlineExceeded) {
                return fmt.Errorf("operation timed out: %w", err)
            }
            return fmt.Errorf("processing item %s: %w", item.ID, err)
        }
    }
    return nil
}

// ✅ Use timeouts for Azure operations
func withTimeout(parent context.Context) (context.Context, context.CancelFunc) {
    return context.WithTimeout(parent, 30*time.Second)
}
```

❌ **DON'T:**
```go
// ❌ Don't ignore context cancellation
func processItems(ctx context.Context, items []Item) error {
    for _, item := range items {
        processItem(ctx, item)  // ❌ Ignores ctx.Done(), keeps running after cancel
    }
    return nil
}
```

**Why:** Azure operations can take significant time. Context cancellation enables graceful shutdown, timeout handling, and resource cleanup.

---

## 9. Data Management

**What this section covers:** Sample data handling, embedded files, JSON loading, and data validation.

### DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Use Go `embed` directive for bundling data.

✅ **DO:**
```
repo/
├── data/
│   ├── products.json              # ✅ Sample data
│   ├── products-with-vectors.json # ✅ Pre-computed embeddings
├── main.go                        # Loads embedded data
```

```go
import "embed"

//go:embed data/products-with-vectors.json
var productsJSON []byte

func loadProducts() ([]Product, error) {
    var products []Product
    if err := json.Unmarshal(productsJSON, &products); err != nil {
        return nil, fmt.Errorf("parsing products data: %w", err)
    }
    return products, nil
}
```

❌ **DON'T:**
```go
// ❌ Don't load from filesystem with hardcoded paths
data, err := os.ReadFile("../data/products.json")  // ❌ Fragile relative path

// ❌ Don't assume data files exist without checking
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list**—not just the immediate project directory.
> 2. **Trace the file path in code** relative to the working directory.
> 3. **Check for monorepo patterns**—data may be shared across samples.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR.

---

### DATA-2: JSON Data Loading (MEDIUM)
**Pattern:** Use `embed` directive for static data. Use `encoding/json` for dynamic data with proper error handling.

✅ **DO:**
```go
import (
    "embed"
    "encoding/json"
    "fmt"
)

//go:embed data/products.json
var productsData []byte

type Product struct {
    ID       string  `json:"id"`
    Name     string  `json:"name"`
    Category string  `json:"category"`
    Price    float64 `json:"price"`
}

func loadProducts() ([]Product, error) {
    var products []Product
    if err := json.Unmarshal(productsData, &products); err != nil {
        return nil, fmt.Errorf("parsing products.json: %w", err)
    }
    if len(products) == 0 {
        return nil, fmt.Errorf("products.json is empty—expected at least one product")
    }
    return products, nil
}
```

❌ **DON'T:**
```go
// ❌ Don't ignore JSON parse errors
var products []Product
json.Unmarshal(data, &products)  // ❌ Error ignored—products may be nil
```

---

## 10. Sample Hygiene

**What this section covers:** Repository hygiene, security, and governance.

### HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and binaries with comprehensive `.gitignore`.

✅ **DO:**
```gitignore
# Environment variables (may contain credentials)
.env
.env.local
.env.*.local
.env.development
.env.production
!.env.sample
!.env.example

# Go binaries
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out

# Vendor (optional—prefer modules)
vendor/

# Azure
.azure/

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Test coverage
coverage.out
coverage.html
```

❌ **DON'T:**
```
repo/
├── .env                    # ❌ Live credentials committed!
├── vendor/                 # ❌ 100MB+ of vendored deps committed
├── myapp.exe               # ❌ Binary committed
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed:
> 1. **Check .gitignore**—look in the project directory AND all parent directories.
> 2. **Run `git ls-files .env`**—if it returns empty, the file is NOT tracked.
> 3. A `.env` file on disk but gitignored is working as designed.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked.

---

### HYG-2: .env.sample (HIGH)
**Pattern:** Provide `.env.sample` with placeholder values.

✅ **DO:**
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
```

❌ **DON'T:**
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # ❌ Real subscription ID
```

---

### HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, functions, and imports. Go enforces no unused imports at compile time, but unused functions and files slip through.

✅ **DO:**
```go
package main

import (
    "context"
    "fmt"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
)
// All imports used, all functions called
```

❌ **DON'T:**
```go
// ❌ Commented-out code confuses users
// import "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
//
// func oldImplementation() {
//     // This was the old way...
// }
```

```
infra/
├── abbreviations.json      # ❌ Never referenced by any Bicep file
```

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── go.mod
├── main.go
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── go.mod
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT**—look for `LICENSE`, `LICENSE.md`, `LICENSE.txt` at the repository root.
> 2. **Check parent directories**—in monorepos, a single license at the repo root covers all subdirectories.
> 3. Only flag if NO license file exists at ANY level of the repo hierarchy above the sample.

---

### HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference governance files.

✅ **DO:**
```markdown
## Contributing

This project welcomes contributions and suggestions. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Security

Microsoft takes security seriously. See [SECURITY.md](SECURITY.md).

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
```

---

## 11. README & Documentation

**What this section covers:** Documentation quality, accuracy, and completeness.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:

    go run .

You should see output similar to:

    ✅ Connected to Azure Blob Storage
    ✅ Container 'samples' created
    ✅ Uploaded blob 'sample.txt' (14 bytes)
    ✅ Downloaded blob content: "Hello, Azure!"

> Note: Exact output may vary based on your Azure environment.
```

❌ **DON'T:**
```markdown
## Expected Output

    ✅ Blob uploaded successfully  # ❌ Not actual output, fabricated
```

---

### DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

✅ **DO:**
```markdown
## Project Structure

- [`main.go`](./main.go)—Main entry point
- [`config.go`](./config.go)—Configuration loader
- [`infra/main.bicep`](./infra/main.bicep)—Infrastructure template
```

❌ **DON'T:**
```markdown
- [`main.go`](./Go/main.go)  # ❌ Wrong path
```

---

### DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure errors.

✅ **DO:**
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

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly.

✅ **DO:**
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

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions.

✅ **DO:**
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

---

### DOC-6: Go Version Strategy (LOW)
**Pattern:** Document minimum Go version in both README and go.mod.

✅ **DO:**
```markdown
## Prerequisites

- **Go**: Version 1.22 or later required for `log/slog` structured logging and loop variable fix
```

```go
// go.mod
go 1.22
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

✅ **DO:**
```markdown
## Configuration

Copy `.env.sample` to `.env` and fill in your values:

    cp .env.sample .env

Edit `.env` and replace placeholders:
- `AZURE_STORAGE_ACCOUNT_NAME`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `AZURE_KEYVAULT_URL`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI
```

❌ **DON'T:**
```markdown
Set environment variables:
- `AZURE_STORAGE_ACCOUNT_NAME=<your-storage-account>`  # ❌ How do I find this?
```

---

## 12. Infrastructure (Bicep/Terraform)

**What this section covers:** Infrastructure-as-code patterns. These are language-agnostic and match the TypeScript skill.

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules.

✅ **DO:**
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
    name: 'storage-deployment'
    params: { name: storageAccountName, location: location }
}
// Check latest: https://azure.github.io/Azure-Verified-Modules/
```

❌ **DON'T:**
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.7.0' = {
    // ❌ Outdated version
}
```

---

### IaC-2: Bicep Parameter Validation (CRITICAL)
**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators.

✅ **DO:**
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

❌ **DON'T:**
```bicep
param aadAdminObjectId string  // ❌ No validation—accepts empty string
```

---

### IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2023+).

✅ **DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
```

❌ **DON'T:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = { }  // ❌ 5+ years old
```

---

### IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities.

✅ **DO:**
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

---

### IaC-5: Network Security (HIGH)
**Pattern:** Document public vs private endpoint choices.

✅ **DO (Quickstart):**
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
    properties: {
        publicNetworkAccess: 'Enabled'  // OK for quickstart
    }
}
// NOTE: For production, use private endpoints with defaultAction: 'Deny'.
```

---

### IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application using azd naming conventions.

✅ **DO:**
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
```

---

### IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

✅ **DO:**
```bicep
var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

❌ **DON'T:**
```bicep
var storageAccountName = 'mystorageaccount123'  // ❌ No naming pattern
```

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns for Go samples.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with Go-specific service definitions.

> **⚠️ FALSE POSITIVE PREVENTION:**
> 1. `services`, `hooks`, and `host` in `azure.yaml` are **OPTIONAL**. Infrastructure-only samples need only `name` and `metadata`.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.
> 3. **Check parent directories**—in monorepos, `azure.yaml` often lives above the language-specific folder.

✅ **DO:**
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
      az account show > /dev/null || (echo "❌ Not logged in. Run 'az login'" && exit 1)
      go version > /dev/null || (echo "❌ Go not installed" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "✅ Provisioning complete"
      echo "Run 'go run .' to test the sample."
```

---

### AZD-2: Service Host Types for Go (MEDIUM)
**Pattern:** Choose correct `host` type for Go applications.

✅ **DO:**
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

## 14. Go Idioms

**What this section covers:** Go-specific patterns that ensure idiomatic, readable, and maintainable Azure SDK samples.

### GO-1: Context Propagation (HIGH)
**Pattern:** Always pass `context.Context` as the first parameter. Create contexts with timeouts for Azure operations. Never use `context.Background()` deep in the call chain.

✅ **DO:**
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

    // ✅ Pass ctx down—every Azure SDK call accepts context
    return uploadBlob(ctx, cred)
}

func uploadBlob(ctx context.Context, cred *azidentity.DefaultAzureCredential) error {
    client, err := azblob.NewClient(url, cred, nil)
    if err != nil {
        return fmt.Errorf("creating client: %w", err)
    }
    _, err = client.UploadBuffer(ctx, "container", "blob.txt", data, nil)  // ✅ ctx propagated
    return err
}
```

❌ **DON'T:**
```go
func uploadBlob(cred *azidentity.DefaultAzureCredential) error {
    // ❌ Creating new context deep in call chain—loses timeout/cancellation
    ctx := context.Background()
    client, err := azblob.NewClient(url, cred, nil)
    if err != nil {
        return err
    }
    _, err = client.UploadBuffer(ctx, "container", "blob.txt", data, nil)
    return err
}
```

**Why:** Context enables timeout propagation, cancellation, and graceful shutdown. Creating `context.Background()` deep in the call stack defeats these mechanisms.

---

### GO-2: Interface-Based Design (MEDIUM)
**Pattern:** Accept interfaces, return structs. Use small interfaces for testability and flexibility.

✅ **DO:**
```go
// ✅ Define small interfaces for what you need
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

❌ **DON'T:**
```go
// ❌ Don't accept concrete types when an interface would do
func ProcessData(ctx context.Context, client *azblob.Client, data []byte) error {
    // Tightly coupled—hard to test without real Azure connection
}
```

**Why:** Interface-based design makes samples easier to test and understand. It also demonstrates Go best practices.

---

### GO-3: Goroutine Safety (MEDIUM)
**Pattern:** When using goroutines with Azure SDK clients, document thread safety. Most Azure SDK clients are safe for concurrent use.

✅ **DO:**
```go
// ✅ Azure SDK clients are safe for concurrent use
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

❌ **DON'T:**
```go
// ❌ Don't launch unbounded goroutines
for _, item := range items {
    go func() {
        client.UploadBuffer(ctx, "container", item.ID+".json", data, nil)
        // ❌ Unbounded concurrency, no error handling, variable capture bug
    }()
}
```

**Why:** Unbounded goroutines overwhelm Azure service rate limits. Use `errgroup` with `SetLimit` for controlled concurrency.

---

### GO-4: Structured Logging with slog (LOW)
**Pattern:** Use `log/slog` (Go 1.22+) for structured logging in production-oriented samples. For quickstarts, `fmt.Println` is acceptable.

✅ **DO:**
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

❌ **DON'T:**
```go
// ❌ Avoid fmt.Println for logging in production-quality samples (OK for quickstarts)
fmt.Println("Uploading blob...")
fmt.Printf("Error: %v\n", err)
```

**Why:** `log/slog` provides structured, leveled logging that's production-ready. `fmt.Println` is acceptable for quickstarts and tutorials but loses structured context for production guidance.

---

### GO-5: to.Ptr() Helper for Pointer Values (MEDIUM)
**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/azcore/to` for creating pointer values. Many Azure SDK structs use pointer fields—`to.Ptr()` is the idiomatic helper.

✅ **DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore/to"

// ✅ Use to.Ptr() for pointer fields in Azure SDK structs
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: to.Ptr("gpt-4o"),
    MaxTokens:      to.Ptr[int32](1000),
    Temperature:    to.Ptr[float32](0.7),
}, nil)

// ✅ Key Vault key creation
createResp, err := keyClient.CreateKey(ctx, "my-key", azkeys.CreateKeyParameters{
    Kty: to.Ptr(azkeys.KeyTypeRSA),
}, nil)
```

❌ **DON'T:**
```go
// ❌ Don't create local variables just for pointers
name := "gpt-4o"
resp, err := client.GetChatCompletions(ctx, azopenai.ChatCompletionsOptions{
    DeploymentName: &name,  // ❌ Verbose—use to.Ptr()
}, nil)
```

**Why:** `to.Ptr()` is the standard Azure SDK helper for pointer creation, making code concise and consistent.

---

### GO-6: LRO/Poller Patterns—runtime.Poller (HIGH)
**Pattern:** Long-running operations (LROs) in the Azure SDK return `*runtime.Poller[T]`. Use `BeginXxx()` methods and `PollUntilDone()` to wait for completion.

✅ **DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore/runtime"

// ✅ Start a long-running operation
poller, err := client.BeginCreateOrUpdate(ctx, resourceGroupName, resourceName, parameters, nil)
if err != nil {
    return fmt.Errorf("starting create operation: %w", err)
}

// ✅ Wait for completion
result, err := poller.PollUntilDone(ctx, &runtime.PollUntilDoneOptions{
    Frequency: 10 * time.Second, // Poll interval (default: 30s for ARM)
})
if err != nil {
    return fmt.Errorf("waiting for create operation: %w", err)
}
fmt.Printf("Resource created: %s\n", *result.ID)

// ✅ Document Intelligence example
poller, err := diClient.BeginAnalyzeDocument(ctx, "prebuilt-invoice", content, nil)
if err != nil {
    return fmt.Errorf("starting analysis: %w", err)
}
analyzeResult, err := poller.PollUntilDone(ctx, nil)
if err != nil {
    return fmt.Errorf("analyzing document: %w", err)
}
```

❌ **DON'T:**
```go
// ❌ Don't ignore the poller—operation may not be complete
poller, _ := client.BeginCreateOrUpdate(ctx, rg, name, params, nil)
// ❌ Resource may not exist yet!
```

**Why:** ARM resource provisioning, Cosmos DB operations, and AI service calls often use LROs. The `BeginXxx` + `PollUntilDone` pattern is critical for correctness.

---

### GO-7: ARM Client Patterns (HIGH)
**Pattern:** Azure Resource Manager (ARM) clients use a `NewClient(subscriptionID, cred, nil)` constructor pattern. Import `armXxx` packages for management-plane operations.

✅ **DO:**
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

// ✅ ARM clients take subscriptionID as first parameter
rgClient, err := armresources.NewResourceGroupsClient(subscriptionID, cred, nil)
if err != nil {
    return fmt.Errorf("creating resource groups client: %w", err)
}

// ✅ List resource groups
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

// ✅ VM client
vmClient, err := armcompute.NewVirtualMachinesClient(subscriptionID, cred, nil)
if err != nil {
    return fmt.Errorf("creating VM client: %w", err)
}
```

**Why:** ARM clients are the management plane for Azure resources. The `NewClient(subscriptionID, cred, opts)` pattern is consistent across all `arm*` packages.

---

### GO-8: Sovereign Cloud Configuration (HIGH)
**Pattern:** For Azure Government, Azure China, or other sovereign clouds, configure `cloud.Configuration` in `ClientOptions`. The default is Azure Public Cloud.

✅ **DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azcore"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/cloud"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
)

// ✅ Azure Government
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

// ✅ Azure China
// Cloud: cloud.AzureChina
```

❌ **DON'T:**
```go
// ❌ Don't hardcode government endpoints without setting cloud config
client, err := azblob.NewClient("https://myaccount.blob.core.usgovcloudapi.net/", cred, nil)
// ❌ Missing cloud configuration—token audience will be wrong
```

**Why:** Sovereign clouds have different authentication endpoints and token audiences. Both the credential AND the client must use the same `cloud.Configuration`.

---

### GO-9: AuthenticationFailedError Handling (MEDIUM)
**Pattern:** Use `errors.As` with `*azcore.ResponseError` to detect authentication failures and provide actionable guidance.

✅ **DO:**
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

**Why:** Authentication errors are the most common issue for new users. Providing actionable guidance saves debugging time.

---

### GO-10: ETag / Optimistic Concurrency (MEDIUM)
**Pattern:** Use ETags for optimistic concurrency control with Cosmos DB and Storage. This prevents lost updates in concurrent scenarios.

✅ **DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azcore"

// ✅ Cosmos DB—use ETag for conditional updates
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
        return fmt.Errorf("item was modified by another process—retry: %w", err)
    }
    return fmt.Errorf("replacing item: %w", err)
}

// ✅ Blob Storage—conditional upload
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

**Why:** Without ETags, concurrent updates can silently overwrite each other (last-write-wins). Optimistic concurrency prevents data loss.

---

## 15. CI/CD & Testing

**What this section covers:** Continuous integration patterns, build validation, and testing.

### CI-1: Build and Vet in CI (HIGH)
**Pattern:** Run `go build`, `go vet`, and `govulncheck` in CI.

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

---

### CI-2: Test Patterns (MEDIUM)
**Pattern:** Write table-driven tests. Use `testing.Short()` to skip integration tests.

✅ **DO:**
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

Use this comprehensive checklist before submitting an Azure SDK Go sample for review:

### 🔧 Project Setup
- [ ] go.mod specifies Go 1.22+ (`go 1.22` directive)
- [ ] Module path matches repository URL
- [ ] Every dependency is imported somewhere (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`sdk/*`, not `services/*`)
- [ ] `go mod tidy` run (no stale deps)
- [ ] `go vet ./...` passes
- [ ] `go build ./...` compiles without errors
- [ ] go.sum committed
- [ ] `govulncheck ./...` passes (no known CVEs)
- [ ] .editorconfig present (optional)

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `.env`, `.env.*`, binaries, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed
- [ ] No build artifacts or binaries committed
- [ ] Dead code removed (unused files, functions, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included

### ☁️ Azure SDK Patterns
- [ ] `azidentity.NewDefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Client options configured (retry policies where applicable)
- [ ] Token refresh implemented for long-running operations
- [ ] Managed identity pattern documented in README
- [ ] Pagination handled with `pager.More()` / `pager.NextPage()`
- [ ] Resource cleanup with `defer client.Close(ctx)`

### ❌ Error Handling
- [ ] Every error return checked (`if err != nil`)
- [ ] Errors wrapped with context (`fmt.Errorf("...: %w", err)`)
- [ ] `azcore.ResponseError` type assertions for Azure errors
- [ ] Context cancellation handled properly
- [ ] No discarded errors (`_` for error values)

### 🏗️ Go Idioms
- [ ] `context.Context` passed as first parameter
- [ ] Interfaces used for testability
- [ ] Goroutine concurrency bounded with `errgroup`
- [ ] Structured logging with `log/slog` (where appropriate)

### 🗄️ Data Services (if applicable)
- [ ] SQL: Uses `go-mssqldb` with AAD auth (`fedauth=ActiveDirectoryDefault`)
- [ ] SQL: Parameterized queries with `sql.Named()` (no string concatenation)
- [ ] SQL: `rows.Close()` deferred, `rows.Err()` checked
- [ ] Cosmos: Queries include partition key
- [ ] Storage: Blob/Table client patterns followed
- [ ] Batch operations for multiple rows (not row-by-row)

### 🤖 AI Services (if applicable)
- [ ] Using `azopenai` package with `DefaultAzureCredential`
- [ ] Vector dimensions validated
- [ ] DiskANN index creation guarded by row count check

### 💬 Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses proper consumer/producer patterns
- [ ] Connection strings avoided (prefer AAD auth)

### 📄 Documentation
- [ ] README "Expected output" from real run (not fabricated)
- [ ] All internal links match actual filesystem paths
- [ ] Prerequisites complete (Go version, subscription, CLI, roles)
- [ ] Troubleshooting section covers common Azure errors
- [ ] Setup instructions clear and tested
- [ ] Go version documented in README and go.mod

### 🏗️ Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated
- [ ] API versions current (2023+)
- [ ] RBAC role assignments created for managed identities
- [ ] Resource naming follows CAF conventions
- [ ] Output values follow azd naming conventions (`AZURE_*`)

### 🧪 CI/CD
- [ ] `go vet`, `go build`, `go test` in CI workflow
- [ ] `govulncheck` in CI
- [ ] Table-driven tests for configuration logic

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`azure-sdk-typescript-sample-review`](../azure-sdk-typescript-sample-review/SKILL.md)**: TypeScript Azure SDK sample review patterns
- **[`azure-sdk-dotnet-sample-review`](../azure-sdk-dotnet-sample-review/SKILL.md)**: .NET 9/10 + Aspire Azure SDK sample review patterns
- **[`azure-sdk-java-sample-review`](../azure-sdk-java-sample-review/SKILL.md)**: Java 17/21 + Spring Boot Azure SDK sample review patterns
- **[`azure-sdk-python-sample-review`](../azure-sdk-python-sample-review/SKILL.md)**: Python 3.9+ + async Azure SDK sample review patterns
- **[`azure-sdk-rust-sample-review`](../azure-sdk-rust-sample-review/SKILL.md)**: Rust 2021 edition Azure SDK sample review patterns
- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency

---

## Scope Note: Services Not Yet Covered

This skill focuses on the most commonly used Azure services in Go samples. The following services are not yet covered in detail, but the general patterns (authentication, client construction, error handling) apply:

- Azure Communication Services
- Azure Cache for Redis
- Azure Monitor
- Azure Container Registry
- Azure App Configuration
- Azure SignalR Service
- Azure API Management
- Azure Container Apps (management plane)

For samples using these services, apply the core patterns from Sections 1–2 (Project Setup, Azure SDK Client Patterns) and reference service-specific documentation.

---

## Reference Links

### Azure SDK for Go
- [Azure SDK for Go Documentation](https://learn.microsoft.com/azure/developer/go/)
- [Azure SDK for Go GitHub](https://github.com/Azure/azure-sdk-for-go)
- [Azure SDK for Go Packages (pkg.go.dev)](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk)
- [azidentity—DefaultAzureCredential](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azidentity#DefaultAzureCredential)
- [azcore—Pipeline & Retry](https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azcore/policy)

### Go Tooling
- [Go Downloads](https://go.dev/dl/)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [staticcheck](https://staticcheck.dev/)
- [Go Modules Reference](https://go.dev/ref/mod)

### API Versioning
- [Azure OpenAI API Versions](https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation)
- [Azure REST API Specifications](https://github.com/Azure/azure-rest-api-specs)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework—Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Security
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Azure Private Endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK Go sample patterns** adapted from comprehensive reviews across the Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (11 rules): Credentials, phantom deps, CVE scanning (govulncheck), token refresh, AVM versions, parameter validation, .gitignore, fabricated output, broken auth, missing error handling, broken links
- **HIGH** (22 rules): Client construction, token management, managed identity, pagination, OpenAI config, SQL patterns, parameter safety, DiskANN guards, batch operations, RBAC, go.sum, role assignments, pre-computed data, .env.sample, prerequisites, dead code, LICENSE, resource naming, network security, context propagation, context cancellation, CI build
- **MEDIUM** (28 rules): Client options, retry policies, module path, environment variables, error wrapping, ResponseError assertions, identifier quoting, embeddings, JSON loading, troubleshooting, azd structure, test patterns, go vet, version pinning, SAS fallback, dimensions, placeholder docs, resource cleanup, API versions, governance files, interface design, goroutine safety, data loading, host types, setup instructions, output values, dependency tidying, package legitimacy
- **LOW** (7 rules): API version docs, .editorconfig/gofmt, Go version docs, structured logging (slog), scope notes, region availability, AI-2 docs

### Service Coverage
- **Core SDK**: Authentication (azidentity), credentials, managed identities, client patterns, token management, pagination (runtime.Pager), resource cleanup (defer)
- **Data**: Cosmos DB (azcosmos), Azure SQL (go-mssqldb), Storage (azblob), batch operations, parameterized queries
- **AI**: Azure OpenAI (azopenai), Document Intelligence, vector dimensions
- **Messaging**: Service Bus (azservicebus), Event Hubs (azeventhubs)
- **Security**: Key Vault (azsecrets, azkeys, azcertificates)
- **Vector Search**: Azure SQL DiskANN, Cosmos DB
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming
- **Go Idioms**: Context propagation, interfaces, goroutine safety, structured logging, error wrapping

Apply these patterns to ensure Azure SDK Go samples are **secure, idiomatic, maintainable, and ready for publication**.
