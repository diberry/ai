# Error Handling, Data Management & Sample Hygiene

Go-specific error handling patterns, sample data handling, and repository hygiene for Azure SDK code.

## ERR-1: Error Wrapping with fmt.Errorf %w (MEDIUM)

**Pattern:** Wrap errors with context using `fmt.Errorf` and the `%w` verb. This preserves the error chain for `errors.Is` and `errors.As`.

**DO:**
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

// Callers can unwrap
err := uploadBlob(ctx, client, data)
if err != nil {
    var respErr *azcore.ResponseError
    if errors.As(err, &respErr) {
        fmt.Printf("Azure error code: %s\n", respErr.ErrorCode)
    }
    return fmt.Errorf("blob operation failed: %w", err)
}
```

**DON'T:**
```go
// Don't use %v--loses error chain
return fmt.Errorf("uploading blob: %v", err)

// Don't discard errors
client.UploadBuffer(ctx, "container", "blob.txt", data, nil)  // Error ignored
```

> **Note:** A simple `return err` after a single operation is idiomatic Go and acceptable when the function name already provides context.

---

## ERR-2: ResponseError Type Assertion (MEDIUM)

**Pattern:** Use `errors.As` to extract `azcore.ResponseError` for Azure-specific error details (status code, error code).

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

**DON'T:**
```go
// Don't use string matching on error messages
if strings.Contains(err.Error(), "not found") {
    // Fragile--error message may change across SDK versions
}

// Don't ignore the error type
if err != nil {
    log.Fatal(err)  // No context, no recovery options
}
```

---

## ERR-3: Context Cancellation Handling (HIGH)

**Pattern:** Check for context cancellation separately from other errors. Provide appropriate messages.

**DO:**
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

// Use timeouts for Azure operations
func withTimeout(parent context.Context) (context.Context, context.CancelFunc) {
    return context.WithTimeout(parent, 30*time.Second)
}
```

**DON'T:**
```go
// Don't ignore context cancellation
func processItems(ctx context.Context, items []Item) error {
    for _, item := range items {
        processItem(ctx, item)  // Ignores ctx.Done(), keeps running after cancel
    }
    return nil
}
```

---

## DATA-1: Pre-Computed Data Files (HIGH)

**Pattern:** Commit all required data files to repo. Use Go `embed` directive for bundling data.

**DO:**
```
repo/
+-- data/
|   +-- products.json              # Sample data
|   +-- products-with-vectors.json # Pre-computed embeddings
+-- main.go                        # Loads embedded data
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

**DON'T:**
```go
// Don't load from filesystem with hardcoded paths
data, err := os.ReadFile("../data/products.json")  // Fragile relative path

// Don't assume data files exist without checking
```

> **FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. Check the FULL PR file list--not just the immediate project directory.
> 2. Trace the file path in code relative to the working directory.
> 3. Check for monorepo patterns--data may be shared across samples.

---

## DATA-2: JSON Data Loading (MEDIUM)

**Pattern:** Use `embed` directive for static data. Use `encoding/json` for dynamic data with proper error handling.

**DO:**
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
        return nil, fmt.Errorf("products.json is empty--expected at least one product")
    }
    return products, nil
}
```

**DON'T:**
```go
// Don't ignore JSON parse errors
var products []Product
json.Unmarshal(data, &products)  // Error ignored--products may be nil
```

---

## HYG-1: .gitignore (CRITICAL)

**Pattern:** Always protect sensitive files, build artifacts, and binaries with comprehensive `.gitignore`.

**DO:**
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

# Vendor (optional--prefer modules)
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

> **FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed:
> 1. Check .gitignore--look in the project directory AND all parent directories.
> 2. Run `git ls-files .env`--if it returns empty, the file is NOT tracked.
> 3. A `.env` file on disk but gitignored is working as designed.

---

## HYG-2: .env.sample (HIGH)

**Pattern:** Provide `.env.sample` with placeholder values.

**DO:**
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
```

**DON'T:**
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # Real subscription ID
```

---

## HYG-3: Dead Code (HIGH)

**Pattern:** Remove unused files, functions, and imports. Go enforces no unused imports at compile time, but unused functions and files slip through.

**DO:**
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

**DON'T:**
```go
// Commented-out code confuses users
// import "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
//
// func oldImplementation() {
//     // This was the old way...
// }
```

---

## HYG-4: LICENSE File (HIGH)

**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

> **FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. Check the REPO ROOT--look for `LICENSE`, `LICENSE.md`, `LICENSE.txt` at the repository root.
> 2. Check parent directories--in monorepos, a single license at the repo root covers all subdirectories.

---

## HYG-5: Repository Governance Files (MEDIUM)

**Pattern:** Samples in Azure Samples org should reference governance files.

**DO:**
```markdown
## Contributing

This project welcomes contributions and suggestions. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Security

Microsoft takes security seriously. See [SECURITY.md](SECURITY.md).

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
```
