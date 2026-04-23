# Azure SDK Client Patterns

Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

## AZ-1: Client Construction with azidentity (HIGH)

**Pattern:** Use `azidentity.NewDefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

**DO:**
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
    // Cache credential instance--reuse across clients
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        log.Fatalf("failed to create credential: %v", err)
    }

    ctx := context.Background()

    // Storage Blob
    blobClient, err := azblob.NewClient(
        fmt.Sprintf("https://%s.blob.core.windows.net/", accountName),
        cred,
        nil,
    )
    if err != nil {
        log.Fatalf("failed to create blob client: %v", err)
    }

    // Key Vault Secrets
    secretClient, err := azsecrets.NewClient(config.KeyVaultURL, cred, nil)
    if err != nil {
        log.Fatalf("failed to create secret client: %v", err)
    }

    // Service Bus
    sbClient, err := azservicebus.NewClient(
        fmt.Sprintf("%s.servicebus.windows.net", namespace),
        cred,
        nil,
    )
    if err != nil {
        log.Fatalf("failed to create service bus client: %v", err)
    }
    defer sbClient.Close(ctx)

    // Cosmos DB
    cosmosClient, err := azcosmos.NewClient(config.CosmosEndpoint, cred, nil)
    if err != nil {
        log.Fatalf("failed to create cosmos client: %v", err)
    }
}
```

**DON'T:**
```go
// Don't use connection strings in samples (prefer AAD auth)
client, err := azblob.NewClientFromConnectionString(connectionString, nil)

// Don't use shared keys
cred, _ := azblob.NewSharedKeyCredential(accountName, accountKey)

// Don't recreate credential for each client
cred1, _ := azidentity.NewDefaultAzureCredential(nil)
blobClient, _ := azblob.NewClient(url, cred1, nil)
cred2, _ := azidentity.NewDefaultAzureCredential(nil)  // Wasteful
secretClient, _ := azsecrets.NewClient(url, cred2, nil)
```

---

## AZ-2: Client Options--azcore.ClientOptions, Retry (MEDIUM)

**Pattern:** Configure retry policies, timeouts, and logging for production-ready samples.

**DO:**
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
    },
}

blobClient, err := azblob.NewClient(serviceURL, cred, clientOptions)
if err != nil {
    return fmt.Errorf("creating blob client: %w", err)
}
```

**DON'T:**
```go
// Don't use extremely short timeouts or misconfigure retry policies
clientOptions := &azblob.ClientOptions{
    ClientOptions: policy.ClientOptions{
        Retry: policy.RetryOptions{
            MaxRetries: 0,  // No retries--Azure calls are inherently transient
        },
    },
}
```

> **Note:** Passing `nil` for `ClientOptions` is idiomatic Go and acceptable for quickstarts. The SDK provides sensible defaults (3 retries, exponential backoff).

---

## AZ-3: Managed Identity Patterns (HIGH)

**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

**DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

// For samples: DefaultAzureCredential (works locally + cloud)
cred, err := azidentity.NewDefaultAzureCredential(nil)

// For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
cred, err := azidentity.NewManagedIdentityCredential(nil)

// User-assigned (when multiple identities needed)
cred, err := azidentity.NewManagedIdentityCredential(&azidentity.ManagedIdentityCredentialOptions{
    ID: azidentity.ClientID(os.Getenv("AZURE_CLIENT_ID")),
})
```

**DON'T:**
```go
// Don't hardcode service principal credentials in samples
cred, err := azidentity.NewClientSecretCredential(tenantID, clientID, clientSecret, nil)
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

## AZ-4: Token Management--azcore.TokenCredential (CRITICAL)

**Pattern:** For services without official SDK (SQL, custom APIs), get tokens with `GetToken()`. Tokens expire after ~1 hour--implement refresh logic for long-running samples.

**DO:**
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

// Token refresh for long-running operations
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

**DON'T:**
```go
// CRITICAL: Don't acquire token once and use for hours
token, _ := cred.GetToken(ctx, policy.TokenRequestOptions{
    Scopes: []string{"https://database.windows.net/.default"},
})
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

---

## AZ-5: DefaultAzureCredential Configuration (MEDIUM)

**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive credentials for CI.

**DO:**
```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

// For CI/CD environments (no interactive prompts)
cred, err := azidentity.NewDefaultAzureCredential(&azidentity.DefaultAzureCredentialOptions{
    DisableInstanceDiscovery: false,
    TenantID:                 os.Getenv("AZURE_TENANT_ID"),
})

// Document the credential chain in README:
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
// > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > 2. Workload identity (Azure Kubernetes Service)
// > 3. Managed identity (App Service, Functions, Container Apps)
// > 4. Azure CLI (`az login`)
// > 5. Azure Developer CLI (`azd auth login`)
```

**DON'T:**
```go
// Don't ignore credential errors
cred, _ := azidentity.NewDefaultAzureCredential(nil)
```

---

## AZ-6: Resource Cleanup--defer client.Close() (MEDIUM)

**Pattern:** Samples must properly close clients using `defer`. Always pass a context to `Close()`.

**DO:**
```go
func run(ctx context.Context) error {
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return fmt.Errorf("creating credential: %w", err)
    }

    // Service Bus--close with defer
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

**DON'T:**
```go
// Don't forget to close clients
sbClient, err := azservicebus.NewClient(namespace, cred, nil)
sender, _ := sbClient.NewSender("myqueue", nil)
sender.SendMessage(ctx, &azservicebus.Message{Body: []byte("hello")}, nil)
// Client and sender never closed (resource leak)
```

---

## AZ-7: Pagination with runtime.Pager (HIGH)

**Pattern:** Use `runtime.Pager` for paginated Azure SDK responses. Samples that only process the first page silently lose data.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/storage/azblob"
    "github.com/Azure/azure-sdk-for-go/sdk/azcore/runtime"
)

// Blob Storage--iterate all pages
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

// Cosmos DB--iterate query results
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

// Key Vault--list all secrets
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

**DON'T:**
```go
// Only gets first page
pager := containerClient.NewListBlobsFlatPager(nil)
page, _ := pager.NextPage(ctx)
for _, blob := range page.Segment.BlobItems {
    fmt.Println(*blob.Name)
}
// Stops after first page--may miss thousands of blobs
```
