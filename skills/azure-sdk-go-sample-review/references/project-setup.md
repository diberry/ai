# Project Setup & Configuration

Module structure, Go version, dependency management, environment variables, and tooling. These foundational patterns ensure samples build correctly and run reliably across environments.

## PS-1: Go Version (HIGH)

**Pattern:** Target Go 1.22+ in go.mod. Use current stable Go features.

**DO:**
```go
// go.mod
module github.com/Azure-Samples/azure-storage-blob-go-quickstart

go 1.22

require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0
)
```

**DON'T:**
```go
// go.mod
module github.com/Azure-Samples/azure-storage-blob-go-quickstart

go 1.18  // Too old--missing slog, slices, and other modern features
```

**Why:** Go 1.22+ provides `log/slog` for structured logging, improved generics, better error handling, and fixes the loop variable capture bug.

---

## PS-2: go.mod Module Path (MEDIUM)

**Pattern:** Use a descriptive module path that matches the repository location.

**DO:**
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

**DON'T:**
```go
// go.mod
module my-sample  // Not a valid Go module path, no repository location

go 1.22
```

---

## PS-3: Dependency Audit (CRITICAL)

**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`github.com/Azure/azure-sdk-for-go/sdk/*`).

**DO:**
```go
// go.mod
require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0   // Used in main.go
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0 // Used in storage.go
    github.com/Azure/azure-sdk-for-go/sdk/azcore v1.14.0       // Used for policy.ClientOptions
)
```

**DON'T:**
```go
require (
    github.com/Azure/azure-sdk-for-go/services/storage/mgmt/2021-09-01/storage v0.0.0  // Track 1 (legacy)
    github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus v1.7.0                 // Listed but never imported
    github.com/joho/godotenv v1.5.1                                                      // Not needed--use os.Getenv
)
```

---

## PS-4: Azure SDK Package Naming (HIGH)

**Pattern:** Use Track 2 packages (`github.com/Azure/azure-sdk-for-go/sdk/*`) not Track 1 legacy packages (`github.com/Azure/azure-sdk-for-go/services/*`).

**DO (Track 2):**
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

**DON'T (Track 1 Legacy):**
```go
import (
    "github.com/Azure/azure-sdk-for-go/services/storage/mgmt/2021-09-01/storage"  // Track 1
    "github.com/Azure/azure-storage-blob-go/azblob"                                // Deprecated standalone
    "github.com/Azure/go-autorest/autorest"                                         // Legacy auth
)
```

---

## PS-5: Environment Variables (MEDIUM)

**Pattern:** Use `os.Getenv` with validation for required variables. Provide `.env.sample` with placeholder values.

**DO:**
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

**DON'T:**
```go
// Don't silently use defaults for Azure resources
storageAccount := os.Getenv("AZURE_STORAGE_ACCOUNT_NAME")
if storageAccount == "" {
    storageAccount = "devstoreaccount1"  // Silent fallback hides misconfiguration
}

// Don't use godotenv when os.Getenv works fine
import "github.com/joho/godotenv"
godotenv.Load()
```

---

## PS-6: Go Vet / staticcheck (MEDIUM)

**Pattern:** Ensure `go vet ./...` and ideally `staticcheck ./...` pass with zero warnings.

**DO:**
```bash
# Before submitting sample
go vet ./...
staticcheck ./...
```

```go
// Clean code--no vet warnings
func processItem(ctx context.Context, item Item) error {
    if item.ID == "" {
        return fmt.Errorf("item ID is required")
    }
    // Process...
    return nil
}
```

**DON'T:**
```go
// go vet catches this: Printf format %d has arg of wrong type string
fmt.Printf("Processing item %d\n", item.Name)

// go vet catches this: unreachable code
return nil
fmt.Println("Done")
```

---

## PS-7: .editorconfig / gofmt (LOW)

**Pattern:** Include `.editorconfig` for consistent formatting. Always run `gofmt` or `goimports`.

**DO:**
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

---

## PS-8: go.sum Committed (HIGH)

**Pattern:** Commit `go.sum` for reproducible builds. Never gitignore it.

**DO:**
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

# go.sum is COMMITTED (not in .gitignore)
```

**DON'T:**
```gitignore
# Don't ignore go.sum
go.sum
```

---

## PS-9: CVE Scanning--govulncheck (CRITICAL)

**Pattern:** Samples must not ship with known security vulnerabilities. `govulncheck` must pass.

**DO:**
```bash
# Install govulncheck
go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan for vulnerabilities
govulncheck ./...

# Expected: No vulnerabilities found.
```

**DON'T:**
```bash
# Don't ignore vulnerability warnings
govulncheck ./...
# Vulnerability #1: GO-2024-1234
# Submitting sample anyway
```

---

## PS-10: Package Legitimacy Check (MEDIUM)

**Pattern:** Verify Azure SDK packages are from official `github.com/Azure/azure-sdk-for-go/sdk/*` paths. Watch for typosquatting.

**DO:**
```go
require (
    github.com/Azure/azure-sdk-for-go/sdk/azidentity v1.7.0    // Official
    github.com/Azure/azure-sdk-for-go/sdk/storage/azblob v1.4.0 // Official
    github.com/Azure/azure-sdk-for-go/sdk/data/azcosmos v1.1.0  // Official
)
```

**DON'T:**
```go
require (
    github.com/azure/azure-sdk-for-go/sdk/azidentity v1.7.0  // Wrong case (azure vs Azure)
    github.com/Azure/azure-sdk-go/sdk/storage v1.0.0          // Wrong module path
    github.com/azuresdk/storage-blob v1.0.0                   // Not official
)
```

**Check:** All Azure SDK packages must be under `github.com/Azure/azure-sdk-for-go/sdk/`. Verify on pkg.go.dev.

---

## PS-11: Dependency Tidying--go mod tidy (MEDIUM)

**Pattern:** Run `go mod tidy` before submitting. Ensures go.mod and go.sum are clean.

**DO:**
```bash
# Before submitting sample
go mod tidy
go mod verify

# Verify no changes after tidy
git diff go.mod go.sum
# Should show no changes if already tidy
```
