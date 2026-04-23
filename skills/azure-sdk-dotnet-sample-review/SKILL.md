---
name: "azure-sdk-dotnet-sample-review"
description: "Comprehensive review checklist for Azure SDK .NET (9/10/Aspire) code samples covering project setup, Azure SDK client patterns, authentication, data services (Cosmos DB, SQL, Storage, Tables), messaging (Service Bus, Event Hubs), AI services (OpenAI, Document Intelligence, Speech), Key Vault, .NET Aspire orchestration, infrastructure, documentation, and sample hygiene. Adapted from TypeScript review skill patterns, generalized for .NET Azure SDK ecosystem."
domain: "code-review"
confidence: "high"
source: "earned -- adapted from TypeScript review skill patterns, generalized for .NET Azure SDK ecosystem"
---

## Context

Use this skill when reviewing **.NET code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general C# review—it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (Track 2 `Azure.*` NuGet packages, client construction, pipeline options)
- **Authentication patterns** (`DefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Service Bus, Key Vault, AI services)
- **.NET Aspire orchestration** (AppHost, ServiceDefaults, service discovery, health checks)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)

This skill captures patterns and anti-patterns for Azure SDK .NET samples, including .NET Aspire orchestration patterns unique to the .NET ecosystem.

**Total rules: 71** (9 CRITICAL, 25 HIGH, 33 MEDIUM, 4 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **.csproj**: Uses Track 2 Azure SDK packages (`Azure.*`, not `Microsoft.Azure.*` — exception: `Microsoft.Azure.Cosmos` is current)
- [ ] **Authentication**: Uses `DefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `bin/`, `obj/`, `.env`, `appsettings.Development.json`
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: `dotnet list package --vulnerable` passes with no critical/high vulnerabilities
- [ ] **Nullable**: `<Nullable>enable</Nullable>` in .csproj
- [ ] **Error handling**: `catch` blocks present with proper exception types
- [ ] **Resource cleanup**: Clients properly disposed (`await using`, `IAsyncDisposable`)
- [ ] **Lock file**: `packages.lock.json` committed when `RestorePackagesWithLockFile` is set
- [ ] **Target framework**: `net8.0`, `net9.0`, or `net10.0`
- [ ] **Build succeeds**: `dotnet build` completes without errors
- [ ] **Sample runs**: `dotnet run` executes without crashes

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets**—Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication**—No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling**—Unhandled exceptions, no try/catch blocks, silent failures
4. **Broken references**—Missing NuGet packages, incorrect project references, build errors
5. **Security vulnerabilities**—`dotnet list package --vulnerable` shows critical or high CVEs
6. **Missing LICENSE**—No LICENSE file at ANY level of repo hierarchy (MIT required for Azure Samples org). ⚠️ Check repo root before flagging.
7. **Secrets committed**—Live credentials in `appsettings.json` or `.env` in version control. ⚠️ Verify with `git ls-files` before flagging.
8. **Track 1 packages**—Uses legacy `Microsoft.Azure.*` packages instead of `Azure.*` (**Exception:** `Microsoft.Azure.Cosmos` v3.x is the current Cosmos DB SDK)

---

## 1. Project Setup & Configuration

**What this section covers:** .csproj structure, target framework, dependency management, nullable reference types, environment configuration, and SDK pinning. These foundational patterns ensure samples build correctly and run reliably across environments.

### PS-1: Target Framework (HIGH)
**Pattern:** Target current supported .NET versions. Both LTS and STS releases are acceptable while in support.

✅ **DO:**
```xml
<!-- .csproj — .NET 9 (STS, supported until May 2026) -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

```xml
<!-- .csproj — .NET 8 (LTS, supported until November 2026) -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

❌ **DON'T:**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>  <!-- ❌ Out of support (Nov 2024) -->
    <Nullable>disable</Nullable>                <!-- ❌ Nullable should be enabled -->
  </PropertyGroup>
</Project>
```

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>  <!-- ❌ Out of support (May 2024) -->
  </PropertyGroup>
</Project>
```

**Why:** .NET 8 is the current LTS (Long Term Support, until Nov 2026). .NET 9 is STS (Standard Term Support, until May 2026). .NET 10 is the next LTS. Samples must target supported frameworks—either `net8.0`, `net9.0`, or `net10.0`.

---

### PS-2: Project Metadata in .csproj (MEDIUM)
**Pattern:** Include descriptive metadata for discoverability and maintenance.

✅ **DO:**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <OutputType>Exe</OutputType>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <RootNamespace>Azure.Samples.StorageBlob</RootNamespace>
    <AssemblyName>Azure.Samples.StorageBlob</AssemblyName>
    <Description>Upload and download blobs using Azure Blob Storage SDK</Description>
    <Authors>Microsoft Corporation</Authors>
    <Company>Microsoft Corporation</Company>
  </PropertyGroup>
</Project>
```

❌ **DON'T:**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <!-- ❌ Missing OutputType, Nullable, Description, Authors -->
  </PropertyGroup>
</Project>
```

**Note:** `ImplicitUsings` and `Nullable` should always be enabled for new .NET 9/10 projects.

---

### PS-3: NuGet Dependency Audit (CRITICAL)
**Pattern:** Every NuGet package must be referenced somewhere in code. No phantom dependencies. Use current Azure SDK Track 2 packages (`Azure.*`).

✅ **DO:**
```xml
<ItemGroup>
  <!-- Use latest stable versions — check NuGet for current releases -->
  <PackageReference Include="Azure.Storage.Blobs" Version="12.*" />
  <PackageReference Include="Azure.Identity" Version="1.*" />
  <PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.*" />
  <PackageReference Include="Azure.AI.OpenAI" Version="2.*" />
</ItemGroup>
```

❌ **DON'T:**
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Azure.Storage.Blob" Version="11.2.3" /> <!-- ❌ Track 1 (legacy) -->
  <PackageReference Include="Azure.Messaging.ServiceBus" Version="7.18.2" />   <!-- ❌ Listed but never used -->
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />              <!-- ❌ Use System.Text.Json -->
</ItemGroup>
```

**Why:** Phantom dependencies bloat samples and confuse users. `System.Text.Json` is built-in since .NET Core 3.0—prefer it over Newtonsoft.Json in new samples. Don't hardcode specific package versions in review guidance—they go stale quickly. Always use the latest stable version from NuGet.

---

### PS-4: Azure SDK Package Naming (HIGH)
**Pattern:** Use Track 2 packages (`Azure.*`) not Track 1 legacy packages (`Microsoft.Azure.*`).

✅ **DO (Track 2):**
```csharp
// ✅ Current generation Azure SDK packages
using Azure.Storage.Blobs;
using Azure.Security.KeyVault.Secrets;
using Azure.Messaging.ServiceBus;
using Microsoft.Azure.Cosmos;  // ✅ Current Cosmos DB SDK (v3.x) — Track 2 patterns despite Microsoft.Azure.* namespace
using Azure.Data.Tables;
using Azure.Identity;
using Azure.AI.OpenAI;
```

❌ **DON'T (Track 1 Legacy):**
```csharp
// ❌ Track 1 packages (legacy, avoid in new samples)
using Microsoft.Azure.Storage;             // Use Azure.Storage.Blobs
using Microsoft.Azure.KeyVault;            // Use Azure.Security.KeyVault.Secrets
using Microsoft.Azure.ServiceBus;          // Use Azure.Messaging.ServiceBus

// ❌ Obsolete patterns
using Microsoft.WindowsAzure.Storage;      // Ancient, never use
using Microsoft.Azure.Documents.Client;    // Legacy Cosmos DB SDK — use Microsoft.Azure.Cosmos v3.x
```

**Why:** Track 2 SDKs (`Azure.*`) are current generation with consistent APIs, `TokenCredential` support, and active maintenance. Track 1 (`Microsoft.Azure.*`) is legacy. **Exception:** `Microsoft.Azure.Cosmos` (v3.x) is the **current, recommended** Cosmos DB SDK—it follows Track 2 patterns despite the `Microsoft.Azure.*` namespace. The actual legacy package was `Microsoft.Azure.Documents.Client`.

---

### PS-5: Environment/Configuration (MEDIUM)
**Pattern:** Use `appsettings.json` + User Secrets for local development. Validate all required configuration with descriptive errors. Prefer `dotnet user-secrets` over `.env` files in .NET projects.

✅ **DO:**
```csharp
// Program.cs
using Microsoft.Extensions.Configuration;

var configuration = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json", optional: false)
    .AddJsonFile("appsettings.Development.json", optional: true)
    .AddEnvironmentVariables()
    .AddUserSecrets<Program>(optional: true)
    .Build();

string storageAccountName = configuration["Azure:StorageAccountName"]
    ?? throw new InvalidOperationException(
        "Missing 'Azure:StorageAccountName' in configuration. " +
        "Set it in appsettings.json or user secrets. " +
        "See README.md for setup instructions.");

string keyVaultUrl = configuration["Azure:KeyVaultUrl"]
    ?? throw new InvalidOperationException(
        "Missing 'Azure:KeyVaultUrl' in configuration.");
```

```json
// appsettings.json (committed—placeholder values only)
{
  "Azure": {
    "StorageAccountName": "<your-storage-account-name>",
    "KeyVaultUrl": "https://<your-keyvault-name>.vault.azure.net/",
    "CosmosEndpoint": "https://<your-cosmos-account>.documents.azure.com:443/"
  }
}
```

❌ **DON'T:**
```csharp
// ❌ Don't hardcode configuration
string storageAccount = "contosoprod";

// ❌ Don't silently fall back to defaults
string storageAccount = configuration["Azure:StorageAccountName"] ?? "devstoreaccount1";

// ❌ Don't use .env files for .NET projects — prefer dotnet user-secrets
// .env files require third-party packages; user-secrets is built-in
```

**Tip:** For local development, prefer `dotnet user-secrets` over `.env` files:
```bash
# Initialize user secrets for the project
dotnet user-secrets init

# Set secret values
dotnet user-secrets set "Azure:StorageAccountName" "mystorageaccount"
dotnet user-secrets set "Azure:KeyVaultUrl" "https://mykeyvault.vault.azure.net/"
```
User secrets are stored outside the project directory and never committed to source control.

---

### PS-6: Nullable Reference Types (MEDIUM)
**Pattern:** Enable nullable reference types. Use null checks and `??` operators consistently.

✅ **DO:**
```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
// ✅ Null-safe patterns
public async Task<string?> GetSecretValueAsync(string secretName)
{
    KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
    return secret.Value;  // Compiler warns if used without null check
}

// ✅ Caller handles nullable return
string? secretValue = await GetSecretValueAsync("db-password");
string connectionString = secretValue
    ?? throw new InvalidOperationException("Secret 'db-password' not found in Key Vault.");
```

❌ **DON'T:**
```csharp
// ❌ Suppressing nullable warnings without explanation
string secretValue = secret.Value!;  // ❌ Null-forgiving operator hides bugs

// ❌ Disabling nullable in .csproj
// <Nullable>disable</Nullable>
```

**Why:** Nullable reference types catch NullReferenceExceptions at compile time. Samples should model safe null handling.

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

[*.cs]
indent_style = space
indent_size = 4

[*.{csproj,props,targets}]
indent_style = space
indent_size = 2

[*.json]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

---

### PS-8: Lock File Management (HIGH)
**Pattern:** Enable and commit `packages.lock.json` for reproducible builds.

✅ **DO:**
```xml
<!-- Directory.Build.props or .csproj -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

```gitignore
# .gitignore
bin/
obj/
.vs/
*.user
appsettings.Development.json

# ✅ packages.lock.json is COMMITTED (not in .gitignore)
```

❌ **DON'T:**
```gitignore
# ❌ Don't ignore lock files
packages.lock.json
```

**Why:** Lock files ensure reproducible builds. Without them, `dotnet restore` may resolve different package versions on different machines.

---

### PS-9: CVE Scanning (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities.

✅ **DO:**
```bash
# Before submitting sample
dotnet list package --vulnerable

# Check for deprecated packages too
dotnet list package --deprecated

# In CI workflow
dotnet restore
dotnet list package --vulnerable --include-transitive
```

```yaml
# .github/workflows/ci.yml
- run: |
    dotnet list package --vulnerable --include-transitive 2>&1 | tee vulnerability-report.txt
    if grep -q "has the following vulnerable packages" vulnerability-report.txt; then
      echo "::error::Vulnerable packages found"
      exit 1
    fi
```

❌ **DON'T:**
```bash
# ❌ Don't ignore vulnerability warnings
dotnet list package --vulnerable
# found 3 vulnerable packages
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Package Legitimacy (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `Azure.*` namespace. Watch for typosquatting.

✅ **DO:**
```xml
<ItemGroup>
  <!-- ✅ Official Azure SDK packages (use latest stable versions from NuGet) -->
  <PackageReference Include="Azure.Storage.Blobs" Version="12.*" />
  <PackageReference Include="Azure.Identity" Version="1.*" />
  <PackageReference Include="Azure.AI.OpenAI" Version="2.*" />
</ItemGroup>
```

❌ **DON'T:**
```xml
<ItemGroup>
  <PackageReference Include="AzureStorageBlobs" Version="1.0.0" />  <!-- ❌ Typosquatting -->
  <PackageReference Include="Azure-Identity" Version="1.0.0" />     <!-- ❌ Not official -->
  <PackageReference Include="Azurre.Storage" Version="1.0.0" />     <!-- ❌ Typosquatting -->
</ItemGroup>
```

**Check:** All Azure SDK packages should use the `Azure.*` prefix and be published by Microsoft. Verify on nuget.org.

---

### PS-11: Version Pinning via global.json (MEDIUM)
**Pattern:** Pin SDK version with `global.json` to ensure consistent builds across environments.

✅ **DO:**
```json
// global.json
{
  "sdk": {
    "version": "9.0.200",
    "rollForward": "latestFeature"
  }
}
```

❌ **DON'T:**
```json
// ❌ Don't pin to exact patch with no rollForward
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "disable"  // ❌ Breaks if exact version not installed
  }
}
```

**Why:** `global.json` ensures all contributors use the same SDK version. `rollForward: "latestFeature"` allows patch updates within the feature band.

---

## 2. Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

### AZ-1: Client Construction with DefaultAzureCredential (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with credential-first pattern. Cache credential instances.

✅ **DO:**
```csharp
using Azure.Identity;
using Azure.Storage.Blobs;
using Azure.Security.KeyVault.Secrets;
using Azure.Messaging.ServiceBus;

// ✅ Cache credential instance
var credential = new DefaultAzureCredential();

// ✅ Storage Blob
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential);

// ✅ Key Vault
var secretClient = new SecretClient(
    new Uri(config["Azure:KeyVaultUrl"]!),
    credential);

// ✅ Service Bus
var serviceBusClient = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net",
    credential);

// ✅ Cosmos DB (CosmosClientOptions is optional but recommended)
var cosmosClient = new CosmosClient(
    config["Azure:CosmosEndpoint"],
    credential,
    new CosmosClientOptions { SerializerOptions = new() { PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase } });
```

❌ **DON'T:**
```csharp
// ❌ Don't use connection strings in samples (prefer AAD auth)
var blobServiceClient = new BlobServiceClient(connectionString);

// ❌ Don't use account keys
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    new StorageSharedKeyCredential(accountName, accountKey));

// ❌ Don't recreate credential for each client
var blobClient = new BlobServiceClient(uri, new DefaultAzureCredential());
var secretClient = new SecretClient(uri, new DefaultAzureCredential());
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, VS Code, etc.) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

### AZ-2: Client Options and Retry Policies (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and diagnostics for production-ready samples.

✅ **DO:**
```csharp
using Azure.Storage.Blobs;
using Azure.Identity;
using Azure.Core;

var credential = new DefaultAzureCredential();

var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential,
    new BlobClientOptions
    {
        Retry =
        {
            MaxRetries = 3,
            Delay = TimeSpan.FromSeconds(1),
            MaxDelay = TimeSpan.FromSeconds(30),
            Mode = RetryMode.Exponential
        },
        Diagnostics =
        {
            IsLoggingContentEnabled = false,
            IsDistributedTracingEnabled = true
        }
    });
```

❌ **DON'T:**
```csharp
// ❌ Don't omit client options for samples that do meaningful work
var blobServiceClient = new BlobServiceClient(uri, credential);
// No retry policy, no timeout configuration
```

---

### AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```csharp
using Azure.Identity;

// ✅ For samples: DefaultAzureCredential (works locally + cloud)
var credential = new DefaultAzureCredential();

// ✅ For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
var credential = new ManagedIdentityCredential();

// ✅ User-assigned (when multiple identities needed)
var credential = new ManagedIdentityCredential(
    clientId: Environment.GetEnvironmentVariable("AZURE_CLIENT_ID"));

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
// > automatically use the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App has a managed identity
// > assigned with appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```csharp
// ❌ Don't hardcode service principal credentials in samples
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources.

---

### AZ-4: Token Management for Non-SDK HTTP (CRITICAL)
**Pattern:** For services requiring raw HTTP calls, get tokens with `GetTokenAsync()`. Tokens expire after ~1 hour—implement refresh logic for long-running samples.

✅ **DO:**
```csharp
using Azure.Core;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// ✅ Get token with expiration tracking
async Task<AccessToken> GetSqlTokenAsync(TokenCredential credential)
{
    return await credential.GetTokenAsync(
        new TokenRequestContext(new[] { "https://database.windows.net/.default" }),
        CancellationToken.None);
}

// ✅ Implement token refresh for long-running operations
AccessToken token = await GetSqlTokenAsync(credential);

bool IsTokenExpiringSoon(AccessToken token)
{
    return token.ExpiresOn < DateTimeOffset.UtcNow.AddMinutes(5);
}

// Before long operation
if (IsTokenExpiringSoon(token))
{
    Console.WriteLine("Token expiring soon, refreshing...");
    token = await GetSqlTokenAsync(credential);
}
```

❌ **DON'T:**
```csharp
// ❌ CRITICAL: Don't acquire token once and use for hours
var token = await credential.GetTokenAsync(
    new TokenRequestContext(new[] { "https://database.windows.net/.default" }),
    CancellationToken.None);
// ... hours of processing with same token (WILL EXPIRE after ~1 hour)
```

**Why:** Azure tokens expire after approximately 1 hour. Samples processing large datasets or running long operations MUST refresh tokens before expiration.

---

### AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI.

✅ **DO:**
```csharp
using Azure.Identity;

// ✅ For CI/CD environments (no interactive prompts)
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeInteractiveBrowserCredential = true,
    ExcludeWorkloadIdentityCredential = false,
    ExcludeManagedIdentityCredential = false,
});

// ✅ For local development (include Visual Studio / VS Code)
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ExcludeVisualStudioCredential = false,
    ExcludeVisualStudioCodeCredential = false,
    ExcludeAzureCliCredential = false,
});

// ✅ Document the credential chain in README
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
// > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > 2. Workload identity (Azure Kubernetes Service)
// > 3. Managed identity (App Service, Functions, Container Apps)
// > 4. Visual Studio
// > 5. Visual Studio Code
// > 6. Azure CLI (`az login`)
// > 7. Azure PowerShell
// > 8. Interactive browser (local development only)
```

---

### AZ-6: Resource Cleanup—IAsyncDisposable (MEDIUM)
**Pattern:** Samples must properly dispose clients. Use `await using` declarations or `IAsyncDisposable`.

✅ **DO:**
```csharp
using Azure.Messaging.ServiceBus;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// ✅ Pattern 1: await using declaration (preferred)
await using var client = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net", credential);

await using var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));
// ✅ sender and client disposed automatically at end of scope

// ✅ Pattern 2: try/finally for complex scenarios
ServiceBusClient? busClient = null;
try
{
    busClient = new ServiceBusClient(
        $"{namespaceName}.servicebus.windows.net", credential);
    var receiver = busClient.CreateReceiver("myqueue");
    var messages = await receiver.ReceiveMessagesAsync(maxMessages: 10);
    foreach (var message in messages)
    {
        Console.WriteLine($"Received: {message.Body}");
        await receiver.CompleteMessageAsync(message);
    }
}
finally
{
    if (busClient is not null)
        await busClient.DisposeAsync();  // ✅ Always cleanup
}
```

❌ **DON'T:**
```csharp
// ❌ Don't forget to dispose clients
var client = new ServiceBusClient(namespace, credential);
var sender = client.CreateSender("myqueue");
await sender.SendMessageAsync(new ServiceBusMessage("Hello"));
// ❌ Client never disposed (resource leak)
```

---

### AZ-7: Pagination with AsyncPageable<T> (HIGH)
**Pattern:** Use `await foreach` with `AsyncPageable<T>` for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;

// ✅ Blob Storage—iterate all pages
var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
await foreach (BlobItem blob in containerClient.GetBlobsAsync())
{
    Console.WriteLine($"Blob: {blob.Name}");
}

// ✅ Key Vault—iterate all secrets
await foreach (SecretProperties secret in secretClient.GetPropertiesOfSecretsAsync())
{
    Console.WriteLine($"Secret: {secret.Name}");
}

// ✅ Cosmos DB—iterate query results with FeedIterator
using FeedIterator<Product> feed = container.GetItemQueryIterator<Product>(
    "SELECT * FROM c WHERE c.category = 'electronics'");

while (feed.HasMoreResults)
{
    FeedResponse<Product> response = await feed.ReadNextAsync();
    foreach (Product item in response)
    {
        Console.WriteLine($"Product: {item.Name}");
    }
}
```

❌ **DON'T:**
```csharp
// ❌ Only gets first page
var blobs = new List<BlobItem>();
await foreach (BlobItem blob in containerClient.GetBlobsAsync())
{
    blobs.Add(blob);
    if (blobs.Count >= 10) break;  // ❌ Stops after 10, may miss thousands
}
```

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns, API versioning, embeddings, chat completions, and document analysis. Focus on the official `Azure.AI.OpenAI` SDK for Azure OpenAI.

### AI-1: Azure.AI.OpenAI Client Configuration (HIGH)
**Pattern:** Use `Azure.AI.OpenAI` v2.x with `DefaultAzureCredential`. The v2.x SDK uses `System.ClientModel` (not `Azure.Core.Pipeline`) for retry configuration.

✅ **DO:**
```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using OpenAI.Chat;
using OpenAI.Embeddings;
using System.ClientModel.Primitives;

var credential = new DefaultAzureCredential();

// ✅ Azure.AI.OpenAI v2.x uses System.ClientModel for retry
var client = new AzureOpenAIClient(
    new Uri(config["Azure:OpenAIEndpoint"]!),
    credential,
    new AzureOpenAIClientOptions
    {
        RetryPolicy = new ClientRetryPolicy(maxRetries: 3),
    });

// ✅ Chat completion (v2.x pattern)
ChatClient chatClient = client.GetChatClient("gpt-4o");
ChatCompletion completion = await chatClient.CompleteChatAsync(
    [
        new SystemChatMessage("You are a helpful assistant."),
        new UserChatMessage("Hello!")
    ]);
Console.WriteLine(completion.Content[0].Text);

// ✅ Embeddings
EmbeddingClient embeddingClient = client.GetEmbeddingClient("text-embedding-3-small");
OpenAIEmbedding embedding = await embeddingClient.GenerateEmbeddingAsync("Sample text to embed");
ReadOnlyMemory<float> vector = embedding.ToFloats();
```

❌ **DON'T:**
```csharp
// ❌ Don't use API keys in samples (prefer AAD)
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    new Azure.AzureKeyCredential(apiKey));  // ❌ Use DefaultAzureCredential

// ❌ Don't use Azure.Core.Pipeline.RetryPolicy (v1.x pattern, won't compile in v2.x)
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    credential,
    new AzureOpenAIClientOptions
    {
        RetryPolicy = new Azure.Core.Pipeline.RetryPolicy(maxRetries: 3),  // ❌ Wrong type for v2.x
    });

// ❌ Don't use deprecated OpenAI client patterns
// The Azure.AI.OpenAI v2.x API replaced the v1.x patterns
```

> **Note:** `Azure.AI.OpenAI` v2.x is built on `System.ClientModel`, not `Azure.Core`. Retry is configured via `System.ClientModel.Primitives.ClientRetryPolicy`, not `Azure.Core.Pipeline.RetryPolicy`.

---

### AI-2: API Version Documentation (LOW)
**Pattern:** Hardcoded API versions should include a comment linking to version docs.

✅ **DO:**
```csharp
var client = new AzureOpenAIClient(
    new Uri(endpoint),
    credential,
    new AzureOpenAIClientOptions(AzureOpenAIClientOptions.ServiceVersion.V2024_10_21));
    // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
```

---

### AI-3: Document Intelligence and Speech (MEDIUM)
**Pattern:** Use `Azure.AI.DocumentIntelligence` and `Microsoft.CognitiveServices.Speech` with `DefaultAzureCredential` where supported.

✅ **DO:**
```csharp
using Azure.AI.DocumentIntelligence;
using Azure.Identity;

// ✅ Document Intelligence with AAD
var credential = new DefaultAzureCredential();
var docClient = new DocumentIntelligenceClient(
    new Uri(config["Azure:DocumentIntelligenceEndpoint"]!),
    credential);

var operation = await docClient.AnalyzeDocumentAsync(
    WaitUntil.Completed,
    "prebuilt-invoice",
    BinaryData.FromStream(documentStream));

AnalyzeResult result = operation.Value;

// ✅ Speech SDK
using Microsoft.CognitiveServices.Speech;

var speechConfig = SpeechConfig.FromSubscription(
    config["Azure:SpeechKey"]!,
    config["Azure:SpeechRegion"]!);
using var recognizer = new SpeechRecognizer(speechConfig);
var speechResult = await recognizer.RecognizeOnceAsync();
Console.WriteLine($"Recognized: {speechResult.Text}");
```

---

### AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension.

✅ **DO:**
```csharp
const string EmbeddingModel = "text-embedding-3-small";  // 1536 dimensions
const int VectorDimension = 1536;

// Validate embedding size
ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
if (embedding.Length != VectorDimension)
{
    throw new InvalidOperationException(
        $"Embedding dimension mismatch: expected {VectorDimension}, got {embedding.Length}. " +
        $"Ensure model '{EmbeddingModel}' matches table schema.");
}
```

❌ **DON'T:**
```csharp
// ❌ Don't assume dimension without validation
ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
await InsertEmbeddingAsync(embedding);  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization.

### DB-1: Cosmos DB SDK Patterns (HIGH)
**Pattern:** Use `Microsoft.Azure.Cosmos` with AAD credentials. Handle partitioned containers properly.

✅ **DO:**
```csharp
using Microsoft.Azure.Cosmos;
using Azure.Identity;

var credential = new DefaultAzureCredential();
var cosmosClient = new CosmosClient(
    config["Azure:CosmosEndpoint"],
    credential,
    new CosmosClientOptions
    {
        SerializerOptions = new CosmosSerializationOptions
        {
            PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
        }
    });

var database = cosmosClient.GetDatabase("mydb");
var container = database.GetContainer("mycontainer");

// ✅ Query with partition key
var query = new QueryDefinition("SELECT * FROM c WHERE c.category = @category")
    .WithParameter("@category", "electronics");

using FeedIterator<Product> feed = container.GetItemQueryIterator<Product>(query);
var results = new List<Product>();
while (feed.HasMoreResults)
{
    FeedResponse<Product> response = await feed.ReadNextAsync();
    results.AddRange(response);
}

// ✅ Point read (most efficient)
var response = await container.ReadItemAsync<Product>(
    "item-id", new PartitionKey("electronics"));
Product item = response.Resource;

// ✅ Upsert with partition key
await container.UpsertItemAsync(new Product
{
    Id = "item-id",
    Category = "electronics",
    Name = "Laptop"
}, new PartitionKey("electronics"));
```

❌ **DON'T:**
```csharp
// ❌ Don't use primary key in samples
var cosmosClient = new CosmosClient(endpoint, primaryKey);  // ❌ Use credential

// ❌ Don't omit partition key in queries (cross-partition queries are expensive)
var query = new QueryDefinition("SELECT * FROM c");
```

---

### DB-2: Azure SQL with Microsoft.Data.SqlClient (HIGH)
**Pattern:** Use `Microsoft.Data.SqlClient` with AAD authentication. Prefer `Authentication=Active Directory Default` connection string over manual token acquisition. Use parameterized queries.

✅ **DO:**
```csharp
using Microsoft.Data.SqlClient;

// ✅ Preferred: Connection string with Active Directory Default (simplest)
await using var connection = new SqlConnection(
    $"Server={config["Azure:SqlServer"]};Database={config["Azure:SqlDatabase"]};" +
    "Authentication=Active Directory Default;Encrypt=True;");
await connection.OpenAsync();

// ✅ Alternative: Manual token acquisition (when you need token for other purposes)
// using Azure.Identity;
// var credential = new DefaultAzureCredential();
// var token = await credential.GetTokenAsync(
//     new Azure.Core.TokenRequestContext(
//         new[] { "https://database.windows.net/.default" }));
// connection.AccessToken = token.Token;

// ✅ Parameterized query
await using var command = new SqlCommand(
    "SELECT [Id], [Name] FROM [Products] WHERE [Category] = @Category", connection);
command.Parameters.AddWithValue("@Category", "Electronics");

await using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    Console.WriteLine($"Product: {reader.GetString(1)}");
}
```

❌ **DON'T:**
```csharp
// ❌ Don't use SQL authentication in samples
var connection = new SqlConnection(
    "Server=myserver;Database=mydb;User=sa;Password=P@ss;");  // ❌

// ❌ Don't use string concatenation for queries
var query = $"SELECT * FROM Products WHERE Category = '{userInput}'";  // ❌ SQL injection!
```

**Why:** `Authentication=Active Directory Default` in the connection string delegates credential resolution to `Microsoft.Data.SqlClient`, which supports managed identity, Azure CLI, and Visual Studio credentials automatically—no manual token management needed.

---

### DB-3: SQL Parameter Safety (MEDIUM)
**Pattern:** ALL dynamic SQL identifiers (table names, column names) must use `[brackets]`. Values must use parameters.

✅ **DO:**
```csharp
string tableName = config["TableName"]!;

// ✅ Always bracket-quote identifiers, parameterize values
var command = new SqlCommand(
    $"SELECT [Id], [Name] FROM [{tableName}] WHERE [Id] = @Id", connection);
command.Parameters.Add(new SqlParameter("@Id", SqlDbType.Int) { Value = productId });
```

❌ **DON'T:**
```csharp
// ❌ Missing brackets on dynamic identifier + string concat for value
var query = $"SELECT Id, Name FROM {tableName} WHERE Id = {productId}";
```

---

### DB-4: Batch/Bulk Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows.

✅ **DO (SQL—Bulk Copy):**
```csharp
// ✅ SqlBulkCopy for large datasets
await using var bulkCopy = new SqlBulkCopy(connection);
bulkCopy.DestinationTableName = "[Products]";
bulkCopy.ColumnMappings.Add("Id", "Id");
bulkCopy.ColumnMappings.Add("Name", "Name");
bulkCopy.ColumnMappings.Add("Category", "Category");
bulkCopy.BatchSize = 1000;

var dataTable = new DataTable();
dataTable.Columns.Add("Id", typeof(int));
dataTable.Columns.Add("Name", typeof(string));
dataTable.Columns.Add("Category", typeof(string));

foreach (var product in products)
{
    dataTable.Rows.Add(product.Id, product.Name, product.Category);
}

await bulkCopy.WriteToServerAsync(dataTable);
```

✅ **DO (Cosmos—Transactional Batch):**
```csharp
// ✅ Cosmos transactional batch (same partition key, max 100 ops)
TransactionalBatch batch = container.CreateTransactionalBatch(
    new PartitionKey("electronics"));

batch.CreateItem(new Product { Id = "1", Category = "electronics", Name = "Laptop" });
batch.CreateItem(new Product { Id = "2", Category = "electronics", Name = "Mouse" });
batch.UpsertItem(new Product { Id = "3", Category = "electronics", Name = "Keyboard" });

using TransactionalBatchResponse response = await batch.ExecuteAsync();
if (!response.IsSuccessStatusCode)
{
    Console.WriteLine($"Batch failed: {response.StatusCode}");
}
```

❌ **DON'T:**
```csharp
// ❌ Row-by-row INSERT (50 round trips for 50 rows)
foreach (var product in products)
{
    await using var cmd = new SqlCommand(
        "INSERT INTO [Products] VALUES (@Id, @Name, @Category)", connection);
    cmd.Parameters.AddWithValue("@Id", product.Id);
    cmd.Parameters.AddWithValue("@Name", product.Name);
    cmd.Parameters.AddWithValue("@Category", product.Category);
    await cmd.ExecuteNonQueryAsync();
}
```

---

### DB-5: Azure Storage (Azure.Storage.Blobs) Patterns (MEDIUM)
**Pattern:** Use `Azure.Storage.Blobs`, `Azure.Storage.Files.Shares`, `Azure.Data.Tables` with `DefaultAzureCredential`.

✅ **DO:**
```csharp
using Azure.Storage.Blobs;
using Azure.Data.Tables;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// ✅ Blob Storage
var blobServiceClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"),
    credential);

var containerClient = blobServiceClient.GetBlobContainerClient("mycontainer");
await containerClient.CreateIfNotExistsAsync();

var blobClient = containerClient.GetBlobClient("myblob.txt");
await blobClient.UploadAsync(BinaryData.FromString("Hello, Azure!"), overwrite: true);

BlobDownloadResult download = await blobClient.DownloadContentAsync();
string content = download.Content.ToString();

// ✅ Table Storage
var tableClient = new TableClient(
    new Uri($"https://{accountName}.table.core.windows.net"),
    "mytable",
    credential);

await tableClient.CreateIfNotExistsAsync();
await tableClient.AddEntityAsync(new TableEntity("partition1", "row1")
{
    { "Name", "Sample" }
});
```

---

### DB-6: SAS Token Fallback (MEDIUM)
**Pattern:** For local development where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

✅ **DO:**
```csharp
using Azure.Storage.Blobs;
using Azure.Identity;

BlobServiceClient blobServiceClient;
string? sasToken = config["Azure:StorageSasToken"];

if (!string.IsNullOrEmpty(sasToken))
{
    // Local dev: SAS token
    blobServiceClient = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net{sasToken}"));
    Console.WriteLine("Using SAS token authentication (local dev)");
}
else
{
    // Production: AAD
    var credential = new DefaultAzureCredential();
    blobServiceClient = new BlobServiceClient(
        new Uri($"https://{accountName}.blob.core.windows.net"),
        credential);
    Console.WriteLine("Using DefaultAzureCredential (AAD)");
}
```

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for queues, topics, event ingestion, and event-driven architectures.

### MSG-1: Service Bus Patterns (HIGH)
**Pattern:** Use `Azure.Messaging.ServiceBus` with `DefaultAzureCredential`. Handle message completion/abandonment.

✅ **DO:**
```csharp
using Azure.Messaging.ServiceBus;
using Azure.Identity;

var credential = new DefaultAzureCredential();
await using var client = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net", credential);

// ✅ Send messages
await using var sender = client.CreateSender("myqueue");
var messages = new List<ServiceBusMessage>
{
    new(BinaryData.FromObjectAsJson(new { OrderId = 1, Amount = 100 })),
    new(BinaryData.FromObjectAsJson(new { OrderId = 2, Amount = 200 })),
};
await sender.SendMessagesAsync(messages);

// ✅ Receive messages (MUST complete or abandon)
await using var receiver = client.CreateReceiver("myqueue");
var receivedMessages = await receiver.ReceiveMessagesAsync(maxMessages: 10,
    maxWaitTime: TimeSpan.FromSeconds(5));

foreach (var message in receivedMessages)
{
    try
    {
        Console.WriteLine($"Received: {message.Body}");
        await receiver.CompleteMessageAsync(message);  // ✅ Mark as processed
    }
    catch (Exception)
    {
        await receiver.AbandonMessageAsync(message);  // ✅ Return to queue
    }
}
```

❌ **DON'T:**
```csharp
// ❌ Don't use connection strings in samples
var client = new ServiceBusClient(connectionString);

// ❌ Don't forget to complete/abandon messages
foreach (var message in receivedMessages)
{
    Console.WriteLine(message.Body);  // ❌ Message never completed (will reappear)
}
```

---

### MSG-2: Event Hubs Patterns (MEDIUM)
**Pattern:** Use `Azure.Messaging.EventHubs` for ingestion, `EventProcessorClient` for processing with checkpoint management.

✅ **DO:**
```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// ✅ Send events
await using var producer = new EventHubProducerClient(
    $"{namespaceName}.servicebus.windows.net", "myeventhub", credential);

using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(new { Temperature = 23.5 })));
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(new { Temperature = 24.1 })));
await producer.SendAsync(batch);

// ✅ Process events with checkpoint store
var storageClient = new BlobContainerClient(
    new Uri($"https://{storageAccount}.blob.core.windows.net/eventhub-checkpoints"),
    credential);

var processor = new EventProcessorClient(
    storageClient, "$Default",
    $"{namespaceName}.servicebus.windows.net", "myeventhub", credential);

processor.ProcessEventAsync += async args =>
{
    Console.WriteLine($"Received: {args.Data.EventBody}");
    await args.UpdateCheckpointAsync();  // ✅ Checkpoint
};

processor.ProcessErrorAsync += args =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
    return Task.CompletedTask;
};

await processor.StartProcessingAsync();
```

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `Azure.Security.KeyVault.Secrets`, `Azure.Security.KeyVault.Keys`, `Azure.Security.KeyVault.Certificates` with `DefaultAzureCredential`.

✅ **DO:**
```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Security.KeyVault.Keys;
using Azure.Security.KeyVault.Certificates;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// ✅ Secrets
var secretClient = new SecretClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

await secretClient.SetSecretAsync("db-password", "P@ssw0rd123");
KeyVaultSecret secret = await secretClient.GetSecretAsync("db-password");
Console.WriteLine($"Secret value: {secret.Value}");

// ✅ Keys (for encryption)
var keyClient = new KeyClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

KeyVaultKey key = await keyClient.CreateKeyAsync("my-encryption-key", KeyType.Rsa);
Console.WriteLine($"Key ID: {key.Id}");

// ✅ Certificates
var certClient = new CertificateClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

KeyVaultCertificateWithPolicy cert = await certClient.GetCertificateAsync("my-cert");
Console.WriteLine($"Certificate thumbprint: {Convert.ToHexString(cert.Properties.X509Thumbprint)}");
```

❌ **DON'T:**
```csharp
// ❌ Don't hardcode secrets in samples
string dbPassword = "P@ssw0rd123";  // ❌ Use Key Vault
```

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations across Azure data services.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Serialize vectors for storage and use `VECTOR_DISTANCE` for search.

✅ **DO (Azure SQL):**
```csharp
// ✅ Insert vector
ReadOnlyMemory<float> embedding = await GetEmbeddingAsync(text);
string vectorJson = JsonSerializer.Serialize(embedding.ToArray());

await using var cmd = new SqlCommand(
    "INSERT INTO [Hotels] ([Embedding]) VALUES (CAST(@Embedding AS VECTOR(1536)))",
    connection);
cmd.Parameters.Add(new SqlParameter("@Embedding", SqlDbType.NVarChar) { Value = vectorJson });
await cmd.ExecuteNonQueryAsync();

// ✅ Vector distance query
var searchCmd = new SqlCommand(@"
    SELECT TOP (@K) [Id], [Name],
        VECTOR_DISTANCE('cosine', [Embedding], CAST(@SearchEmbedding AS VECTOR(1536))) AS Distance
    FROM [Hotels]
    ORDER BY Distance ASC", connection);
searchCmd.Parameters.Add(new SqlParameter("@K", SqlDbType.Int) { Value = 5 });
searchCmd.Parameters.Add(new SqlParameter("@SearchEmbedding", SqlDbType.NVarChar)
    { Value = JsonSerializer.Serialize(searchEmbedding.ToArray()) });
```

✅ **DO (Azure AI Search):**
```csharp
using Azure.Search.Documents;
using Azure.Search.Documents.Models;

var searchClient = new SearchClient(
    new Uri(config["Azure:SearchEndpoint"]!),
    "hotels-index",
    credential);

var options = new SearchOptions
{
    VectorSearch = new()
    {
        Queries =
        {
            new VectorizedQuery(searchEmbedding)
            {
                KNearestNeighborsCount = 5,
                Fields = { "DescriptionVector" }
            }
        }
    }
};

SearchResults<Hotel> results = await searchClient.SearchAsync<Hotel>("luxury hotel", options);
await foreach (SearchResult<Hotel> result in results.GetResultsAsync())
{
    Console.WriteLine($"{result.Document.Name} (score: {result.Score})");
}
```

---

### VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires ≥1000 rows. Check row count before creating index.

✅ **DO:**
```csharp
// Check row count before creating DiskANN index
await using var countCmd = new SqlCommand(
    $"SELECT COUNT(*) FROM [{tableName}]", connection);
int rowCount = (int)(await countCmd.ExecuteScalarAsync())!;

if (rowCount >= 1000)
{
    Console.WriteLine($"✅ {rowCount} rows available. Creating DiskANN index...");
    await using var indexCmd = new SqlCommand(
        $"CREATE INDEX [ix_{tableName}_embedding_diskann] ON [{tableName}] ([Embedding]) USING DiskANN",
        connection);
    await indexCmd.ExecuteNonQueryAsync();
}
else
{
    Console.WriteLine($"⚠️ Only {rowCount} rows. DiskANN requires ≥1000. Using exact search.");
    // Fall back to VECTOR_DISTANCE (exact search)
}
```

❌ **DON'T:**
```csharp
// ❌ Create DiskANN index without checking row count
await new SqlCommand("CREATE INDEX ... USING DiskANN", connection).ExecuteNonQueryAsync();
// Fails with: "DiskANN index requires at least 1000 rows"
```

---

## 8. Error Handling

**What this section covers:** Exception handling, contextual error messages, and Azure SDK-specific exception types.

### ERR-1: Azure SDK Exception Handling (MEDIUM)
**Pattern:** Catch `Azure.RequestFailedException` for Azure service errors. Use pattern matching for specific status codes.

✅ **DO:**
```csharp
using Azure;

try
{
    await blobClient.DownloadContentAsync();
}
catch (RequestFailedException ex) when (ex.Status == 404)
{
    Console.WriteLine($"Blob not found: {ex.Message}");
    Console.WriteLine("Ensure the container and blob exist before downloading.");
}
catch (RequestFailedException ex) when (ex.Status == 403)
{
    Console.WriteLine($"Access denied: {ex.Message}");
    Console.WriteLine("Troubleshooting:");
    Console.WriteLine("  1. Run 'az login' to authenticate with Azure CLI");
    Console.WriteLine("  2. Verify you have 'Storage Blob Data Contributor' role");
    Console.WriteLine("  3. Check your Azure subscription is active");
}
catch (RequestFailedException ex)
{
    Console.WriteLine($"Azure service error ({ex.Status}): {ex.Message}");
    throw;
}
catch (Exception ex)
{
    Console.WriteLine($"Unexpected error: {ex.Message}");
    throw;
}
```

❌ **DON'T:**
```csharp
// ❌ Catching generic Exception loses Azure-specific context
try
{
    await blobClient.DownloadContentAsync();
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);  // ❌ No status code, no troubleshooting
}
```

---

### ERR-2: Contextual Error Messages (MEDIUM)
**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

✅ **DO:**
```csharp
try
{
    await credential.GetTokenAsync(
        new Azure.Core.TokenRequestContext(
            new[] { "https://storage.azure.com/.default" }));
}
catch (Azure.Identity.AuthenticationFailedException ex)
{
    Console.Error.WriteLine($"❌ Failed to acquire Azure Storage token: {ex.Message}");
    Console.Error.WriteLine();
    Console.Error.WriteLine("Troubleshooting:");
    Console.Error.WriteLine("  1. Run 'az login' to authenticate with Azure CLI");
    Console.Error.WriteLine("  2. Or sign in to Visual Studio/VS Code with your Azure account");
    Console.Error.WriteLine("  3. Verify you have the 'Storage Blob Data Contributor' role");
    Console.Error.WriteLine("  4. Check your Azure subscription is active");
    throw;
}
```

---

## 9. Data Management

**What this section covers:** Sample data handling, embedded resources, JSON data files, and data validation.

### DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

✅ **DO:**
```
repo/
├── data/
│   ├── products.json              # ✅ Sample data
│   ├── products-with-vectors.json # ✅ Pre-computed embeddings
├── src/
│   ├── Program.cs                 # Loads products-with-vectors.json
│   ├── GenerateEmbeddings.cs      # Generates embeddings (optional)
```

❌ **DON'T:**
```
repo/
├── data/
│   ├── products.json              # ✅ Raw data
│   ├── .gitignore                 # ❌ products-with-vectors.json gitignored
├── src/
│   ├── Program.cs                 # ❌ Fails: File not found
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list**—not just the immediate project directory.
> 2. **Trace the file path in code** relative to the working directory.
> 3. **Check for monorepo patterns**—data may be shared across multiple samples.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR.

---

### DATA-2: JSON Data Loading (MEDIUM)
**Pattern:** Use `System.Text.Json` with strongly-typed models. Embed data files or use path resolution.

✅ **DO:**
```csharp
using System.Text.Json;

public record Product(string Id, string Name, string Category);

// ✅ Load from file with proper path resolution
string dataPath = Path.Combine(AppContext.BaseDirectory, "..", "..", "..", "data", "products.json");
string json = await File.ReadAllTextAsync(dataPath);
List<Product> products = JsonSerializer.Deserialize<List<Product>>(json,
    new JsonSerializerOptions { PropertyNameCaseInsensitive = true })
    ?? throw new InvalidOperationException("Failed to deserialize products.json");

// ✅ Or use embedded resources
// Add to .csproj: <EmbeddedResource Include="data\products.json" />
using Stream? stream = typeof(Program).Assembly.GetManifestResourceStream(
    "Azure.Samples.StorageBlob.data.products.json")
    ?? throw new InvalidOperationException("Embedded resource not found");
List<Product> products = await JsonSerializer.DeserializeAsync<List<Product>>(stream)
    ?? throw new InvalidOperationException("Failed to deserialize embedded products.json");
```

❌ **DON'T:**
```csharp
// ❌ Don't use Newtonsoft.Json in new samples
using Newtonsoft.Json;
var products = JsonConvert.DeserializeObject<List<Product>>(json);

// ❌ Don't use hardcoded relative paths
string json = File.ReadAllText("../data/products.json");  // ❌ Breaks depending on CWD
```

---

## 10. Sample Hygiene

**What this section covers:** Repository hygiene, security, and governance.

### HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and dependencies.

✅ **DO:**
```gitignore
# Build output
bin/
obj/
*.user

# Environment / Secrets
.env
.env.*
!.env.sample
appsettings.Development.json
appsettings.*.local.json

# IDE
.vs/
.vscode/
*.suo
*.sln.docstates

# Azure
.azure/

# OS
.DS_Store
Thumbs.db
*.log

# NuGet
packages/
*.nupkg

# Test results
TestResults/
coverage/
```

❌ **DON'T:**
```
repo/
├── appsettings.Development.json  # ❌ Live credentials committed!
├── .env                          # ❌ Secrets committed!
├── bin/                          # ❌ Build artifacts committed
├── obj/                          # ❌ Build intermediates committed
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed, you MUST verify:
> 1. **Check .gitignore**—look in the project directory AND all parent directories.
> 2. **Run `git ls-files .env`**—if empty, the file is NOT tracked.
> 3. A `.env` on disk but gitignored is NOT a security issue.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked.

---

### HYG-2: appsettings.json / .env.sample (HIGH)
**Pattern:** Provide `appsettings.json` with placeholder values OR `.env.sample`. Never commit actual secrets.

✅ **DO:**
```json
// appsettings.json (committed—placeholders only)
{
  "Azure": {
    "StorageAccountName": "<your-storage-account>",
    "KeyVaultUrl": "https://<your-keyvault>.vault.azure.net/",
    "CosmosEndpoint": "https://<your-cosmos>.documents.azure.com:443/",
    "OpenAIEndpoint": "https://<your-openai>.openai.azure.com/"
  }
}
```

❌ **DON'T:**
```json
// ❌ Real values committed
{
  "Azure": {
    "StorageAccountName": "contosoprod",
    "SubscriptionId": "12345678-1234-1234-1234-123456789abc"
  }
}
```

---

### HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, methods, and using directives.

✅ **DO:**
```csharp
// Only import what you use
using Azure.Storage.Blobs;
using Azure.Identity;
```

❌ **DON'T:**
```csharp
// ❌ Commented-out code confuses users
// using Azure.Messaging.ServiceBus;  // Was used in old version
// 
// async Task OldImplementation()
// {
//     // This was the old way...
// }

using Azure.Storage.Blobs;
```

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── src/
│   └── MySample.csproj
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── src/
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT**—look for `LICENSE`, `LICENSE.md`, `LICENSE.txt` at the repository root.
> 2. **Check parent directories**—in monorepos, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.

---

### HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference governance files.

✅ **DO:**
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

**What this section covers:** Documentation quality, accuracy, and completeness.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:

```bash
dotnet run
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

---

### DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

✅ **DO:**
```markdown
## Project Structure

- [`src/Program.cs`](./src/Program.cs)—Main entry point
- [`src/Services/BlobService.cs`](./src/Services/BlobService.cs)—Blob operations
- [`infra/main.bicep`](./infra/main.bicep)—Infrastructure template
```

❌ **DON'T:**
```markdown
## Project Structure

- [`src/Program.cs`](./app/Program.cs)—❌ Link path doesn't match description
- [`BlobService.cs`](./src/BlobService.cs)—❌ File is actually in src/Services/
- [`infrastructure`](./infrastructure/)—❌ Folder is actually named "infra/"
```

**Why:** Broken links in README frustrate users and signal a poorly maintained sample. Verify every link with `git ls-files` or the PR file list.

---

### DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure errors.

✅ **DO:**
```markdown
## Troubleshooting

### Authentication Errors

If you see "AuthenticationFailedException":
1. Run `az login` to authenticate with Azure CLI
2. Or sign in to Visual Studio with your Azure account
3. Verify your Azure subscription is active: `az account show`

### RBAC Permission Errors

If you see "RequestFailedException (403)":
- Verify role assignments:
  - Storage: "Storage Blob Data Contributor"
  - Key Vault: "Key Vault Secrets User"
  - Cosmos DB: "Cosmos DB Built-in Data Contributor"
- Role assignments may take 5-10 minutes to propagate
```

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly.

✅ **DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **.NET SDK**: Version 9.0 or later ([Download](https://dot.net/download))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional)

### Azure Resources

This sample requires:
- **Azure Storage Account** with a blob container
- **Azure Key Vault** (if using secrets)
- **Azure OpenAI** deployment (if using AI features)

### Role Assignments

Your Azure identity needs:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
- `Cognitive Services OpenAI User` on the Azure OpenAI resource
```

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions.

✅ **DO:**
```markdown
## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Azure-Samples/azure-storage-blob-dotnet.git
cd azure-storage-blob-dotnet
```

### 2. Provision Azure resources

```bash
azd up
```

### 3. Run the sample

```bash
dotnet run --project src/MySample.csproj
```
```

---

### DOC-6: .NET Version Strategy (LOW)
**Pattern:** Document minimum .NET version in README.

✅ **DO:**
```markdown
## Prerequisites

- **.NET SDK**: Version 9.0 or later ([Download](https://dot.net/download))
  - Verify: `dotnet --version`
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

✅ **DO:**
```markdown
## Configuration

Update `appsettings.json` with your values:

- `Azure:StorageAccountName`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `Azure:KeyVaultUrl`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI

Or use User Secrets for local development:
```bash
dotnet user-secrets set "Azure:StorageAccountName" "mystorageaccount"
```
```

---

## 12. Infrastructure (Bicep/Terraform)

**What this section covers:** Infrastructure-as-code patterns shared across languages.

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules.

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

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

❌ **DON'T:**
```bicep
param aadAdminObjectId string  // ❌ No validation, accepts empty string
```

---

### IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2023+). Avoid versions older than 2 years.

✅ **DO:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
resource appService 'Microsoft.Web/sites@2023-12-01' = { }
```

❌ **DON'T:**
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // ❌ 5+ years old
}
```

---

### IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities.

✅ **DO:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
}

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
- Cosmos DB Built-in Data Contributor: `00000000-0000-0000-0000-000000000002`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

### IaC-5: Network Security (HIGH)
**Pattern:** For quickstart samples, public endpoints acceptable with security comment.

✅ **DO (Quickstart):**
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: openaiAccountName
  properties: {
    publicNetworkAccess: 'Enabled'  // OK for quickstart
  }
}
// NOTE: For production, use private endpoints and set publicNetworkAccess: 'Disabled'.
```

---

### IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application.

✅ **DO:**
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
output AZURE_COSMOS_ENDPOINT string = cosmos.properties.documentEndpoint
```

---

### IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

✅ **DO:**
```bicep
@description('Prefix for all resources')
param resourcePrefix string = 'contoso'

@description('Environment (dev, test, prod)')
@allowed(['dev', 'test', 'prod'])
param environment string = 'dev'

var storageAccountName = '${resourcePrefix}st${environment}'
var keyVaultName = '${resourcePrefix}-kv-${environment}'
var appServiceName = '${resourcePrefix}-app-${environment}'
```

❌ **DON'T:**
```bicep
var storageAccountName = 'mystorageaccount123'  // ❌ Inconsistent naming
```

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns, azure.yaml structure, service definitions, hooks.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `azure.yaml` as missing or incomplete:
> 1. `services`, `hooks`, and `host` fields are **OPTIONAL** for infrastructure-only samples.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.
> 3. **Check parent directories**—in monorepo layouts, `azure.yaml` often lives above the project folder.

✅ **DO:**
```yaml
name: azure-storage-blob-dotnet-sample
metadata:
  template: azure-storage-blob-dotnet-sample@0.0.1

services:
  app:
    project: ./src
    language: csharp
    host: appservice  # or: containerapp, function

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "Provisioning complete"
      echo "Storage: ${AZURE_STORAGE_ACCOUNT_NAME}"
```

---

### AZD-2: Service Host Types (MEDIUM)
**Pattern:** Choose correct `host` type for .NET applications.

✅ **DO:**
```yaml
# App Service (.NET web apps)
services:
  web:
    project: ./src/WebApp
    language: csharp
    host: appservice

# Azure Functions
services:
  api:
    project: ./src/Functions
    language: csharp
    host: function

# Container Apps (.NET Aspire or containerized)
services:
  backend:
    project: ./src/Api
    language: csharp
    host: containerapp
    docker:
      path: ./src/Api/Dockerfile
```

---

## 14. .NET Aspire Integration

**What this section covers:** .NET Aspire orchestration patterns—AppHost, ServiceDefaults, service discovery, health checks, and OpenTelemetry. This section is unique to .NET and has no TypeScript equivalent.

### ASP-1: AppHost Configuration (HIGH)
**Pattern:** Configure the Aspire AppHost to orchestrate Azure resources and application services.

✅ **DO:**
```csharp
// AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// ✅ Add Azure resources
var storage = builder.AddAzureStorage("storage")
    .RunAsEmulator();  // Use Azurite for local dev
var blobs = storage.AddBlobs("blobs");

var cosmos = builder.AddAzureCosmosDB("cosmos")
    .RunAsEmulator();  // Use Cosmos emulator for local dev
var database = cosmos.AddDatabase("mydb");

var openai = builder.AddAzureOpenAI("openai")
    .AddDeployment(new AzureOpenAIDeployment("gpt-4o", "gpt-4o", "2024-08-06"));

var serviceBus = builder.AddAzureServiceBus("messaging")
    .AddQueue("orders");

// ✅ Add application projects with resource references
var api = builder.AddProject<Projects.MyApi>("api")
    .WithReference(blobs)
    .WithReference(database)
    .WithReference(openai)
    .WithReference(serviceBus)
    .WithExternalHttpEndpoints();

var web = builder.AddProject<Projects.MyWeb>("web")
    .WithReference(api)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

```xml
<!-- AppHost/AppHost.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <!-- Use the latest stable Aspire SDK version — check NuGet for current release -->
  <Sdk Name="Aspire.AppHost.Sdk" Version="9.*" />
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <IsAspireHost>true</IsAspireHost>
  </PropertyGroup>
  <ItemGroup>
    <!-- Use latest stable versions — do not hardcode specific patch versions -->
    <PackageReference Include="Aspire.Hosting.Azure.Storage" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Azure.CosmosDB" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Azure.CognitiveServices" Version="9.*" />
    <PackageReference Include="Aspire.Hosting.Azure.ServiceBus" Version="9.*" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\MyApi\MyApi.csproj" />
    <ProjectReference Include="..\MyWeb\MyWeb.csproj" />
  </ItemGroup>
</Project>
```

❌ **DON'T:**
```csharp
// ❌ Don't hardcode connection strings in AppHost
var api = builder.AddProject<Projects.MyApi>("api")
    .WithEnvironment("ConnectionStrings__Storage",
        "DefaultEndpointsProtocol=https;AccountName=...");  // ❌ Use .WithReference()

// ❌ Don't skip RunAsEmulator for local development
var storage = builder.AddAzureStorage("storage");
// ❌ Without RunAsEmulator, requires live Azure resources for local dev
```

**Why:** Aspire's `.WithReference()` pattern automatically wires connection strings and enables service discovery. `RunAsEmulator()` enables fully local development without Azure resources.

---

### ASP-2: ServiceDefaults Setup (MEDIUM)
**Pattern:** Use the ServiceDefaults project for shared configuration—OpenTelemetry, health checks, resilience.

✅ **DO:**
```csharp
// ServiceDefaults/Extensions.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;

public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(
        this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });
        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(
        this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation();
            });

        builder.AddOpenTelemetryExporters();
        return builder;
    }

    public static IHostApplicationBuilder AddDefaultHealthChecks(
        this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);
        return builder;
    }
}
```

```csharp
// MyApi/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();  // ✅ Adds telemetry, health checks, resilience
// ... configure services
var app = builder.Build();
app.MapDefaultEndpoints();  // ✅ /health (readiness — checks Azure deps) and /alive (liveness — basic self-check)
```

❌ **DON'T:**
```csharp
// ❌ Don't manually configure OpenTelemetry in every project
// ❌ Don't skip AddServiceDefaults
var builder = WebApplication.CreateBuilder(args);
// Missing builder.AddServiceDefaults();
```

---

### ASP-3: Service Discovery Patterns (MEDIUM)
**Pattern:** Use Aspire service discovery for inter-service communication instead of hardcoded URLs.

✅ **DO:**
```csharp
// MyWeb/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();

// ✅ Register HTTP client using service discovery name
builder.Services.AddHttpClient<ApiClient>(client =>
{
    client.BaseAddress = new Uri("https+http://api");  // ✅ Resolved by Aspire
});

// ApiClient.cs
public class ApiClient(HttpClient httpClient)
{
    public async Task<List<Product>> GetProductsAsync()
    {
        return await httpClient.GetFromJsonAsync<List<Product>>("/api/products")
            ?? [];
    }
}
```

❌ **DON'T:**
```csharp
// ❌ Don't hardcode service URLs
builder.Services.AddHttpClient<ApiClient>(client =>
{
    client.BaseAddress = new Uri("https://localhost:5001");  // ❌ Hardcoded
});
```

**Why:** Aspire's service discovery resolves service names (`"https+http://api"`) to actual endpoints at runtime, working seamlessly across local development and cloud deployment.

---

### ASP-4: Health Checks (MEDIUM)
**Pattern:** Register health checks for Azure dependencies. Map health endpoints.

✅ **DO:**
```csharp
// MyApi/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();

// ✅ Add Azure resource health checks
builder.Services.AddHealthChecks()
    .AddAzureBlobStorage(
        sp => sp.GetRequiredService<BlobServiceClient>(),
        name: "azure-blob-storage",
        tags: new[] { "ready" })
    .AddAzureCosmosDB(
        sp => sp.GetRequiredService<CosmosClient>(),
        name: "azure-cosmos-db",
        tags: new[] { "ready" });

var app = builder.Build();
app.MapDefaultEndpoints();  // ✅ /health (readiness, checks dependencies) and /alive (liveness, basic self-check)
```

❌ **DON'T:**
```csharp
// ❌ Don't skip health checks for Azure dependencies
var app = builder.Build();
// ❌ No health endpoints mapped
```

---

### ASP-5: Aspire Dashboard / OpenTelemetry (LOW)
**Pattern:** Leverage the Aspire dashboard for local observability. Configure OTLP exporters for production.

✅ **DO:**
```csharp
// The Aspire dashboard starts automatically with the AppHost.
// No additional configuration needed for local development.
// For production, configure OTLP exporter:

// ServiceDefaults/Extensions.cs
private static IHostApplicationBuilder AddOpenTelemetryExporters(
    this IHostApplicationBuilder builder)
{
    var useOtlp = !string.IsNullOrWhiteSpace(
        builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

    if (useOtlp)
    {
        builder.Services.AddOpenTelemetry()
            .UseOtlpExporter();
    }
    return builder;
}
```

**Why:** The Aspire dashboard provides traces, logs, and metrics visualization out-of-the-box during development. For production, configure OTLP export to Azure Monitor or another backend.

---

### ASP-6: Aspire Client Integration Packages (HIGH)
**Pattern:** In Aspire service projects, use `Aspire.Azure.*` client integration packages instead of manual client construction. These packages provide DI-friendly registration, health checks, and telemetry out-of-the-box.

✅ **DO:**
```xml
<!-- MyApi/MyApi.csproj — Aspire client integration packages -->
<ItemGroup>
  <!-- Use latest stable versions — check NuGet for current releases -->
  <PackageReference Include="Aspire.Azure.Storage.Blobs" Version="9.*" />
  <PackageReference Include="Aspire.Azure.Data.Cosmos" Version="9.*" />
  <PackageReference Include="Aspire.Azure.Security.KeyVault" Version="9.*" />
  <PackageReference Include="Aspire.Azure.AI.OpenAI" Version="9.*" />
</ItemGroup>
```

```csharp
// MyApi/Program.cs — Aspire DI registration
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();

// ✅ Aspire client integrations — automatically wired via .WithReference() in AppHost
builder.AddAzureBlobClient("blobs");           // Registers BlobServiceClient in DI
builder.AddAzureCosmosClient("cosmos");         // Registers CosmosClient in DI
builder.AddAzureKeyVaultClient("keyvault");     // Registers SecretClient in DI
builder.AddAzureOpenAIClient("openai");         // Registers AzureOpenAIClient in DI

// ✅ Inject via constructor
public class ProductService(BlobServiceClient blobClient, CosmosClient cosmosClient)
{
    public async Task UploadAsync(string name, Stream content)
    {
        var container = blobClient.GetBlobContainerClient("products");
        await container.GetBlobClient(name).UploadAsync(content, overwrite: true);
    }
}
```

❌ **DON'T:**
```csharp
// ❌ Don't manually construct clients in Aspire projects
var blobClient = new BlobServiceClient(
    new Uri("https://mystorageaccount.blob.core.windows.net"),
    new DefaultAzureCredential());  // ❌ Bypasses Aspire DI, health checks, telemetry
```

**Why:** Aspire client integration packages (`Aspire.Azure.*`) automatically configure health checks, OpenTelemetry tracing, retry policies, and connection management. Manual client construction bypasses all of these.

**Key packages:**
| Aspire Package | Registers | DI Method |
|---|---|---|
| `Aspire.Azure.Storage.Blobs` | `BlobServiceClient` | `builder.AddAzureBlobClient()` |
| `Aspire.Azure.Data.Cosmos` | `CosmosClient` | `builder.AddAzureCosmosClient()` |
| `Aspire.Azure.Security.KeyVault` | `SecretClient` | `builder.AddAzureKeyVaultClient()` |
| `Aspire.Azure.AI.OpenAI` | `AzureOpenAIClient` | `builder.AddAzureOpenAIClient()` |
| `Aspire.Azure.Messaging.ServiceBus` | `ServiceBusClient` | `builder.AddAzureServiceBusClient()` |

---

### ASP-7: Aspire DI vs Manual Client Construction (HIGH)
**Pattern:** Distinguish between Aspire DI-based client registration and manual construction. Use the correct pattern for the project type.

✅ **DO (Aspire project):**
```csharp
// In Aspire projects, use builder.AddAzure*() — clients are wired via .WithReference()
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
builder.AddAzureBlobClient("blobs");  // ✅ Name matches AppHost resource name

// Inject via DI
app.MapGet("/upload", async (BlobServiceClient client) =>
{
    // Client is pre-configured with connection, retry, telemetry
    var container = client.GetBlobContainerClient("uploads");
    // ...
});
```

✅ **DO (Non-Aspire project):**
```csharp
// Without Aspire, use Microsoft.Extensions.Azure for DI registration
using Microsoft.Extensions.Azure;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddBlobServiceClient(
        new Uri($"https://{config["Azure:StorageAccountName"]}.blob.core.windows.net"));
    clientBuilder.AddSecretClient(
        new Uri(config["Azure:KeyVaultUrl"]!));
    clientBuilder.UseCredential(new DefaultAzureCredential());
});
```

✅ **DO (Console app / simple sample):**
```csharp
// For console apps without DI, construct clients directly
var credential = new DefaultAzureCredential();
var blobClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"), credential);
```

❌ **DON'T:**
```csharp
// ❌ Don't mix patterns — don't manually construct in an Aspire project
// ❌ Don't use new BlobServiceClient(...) when Aspire.Azure.Storage.Blobs is available
```

---

## 14a. Azure SDK Dependency Injection Patterns

**What this section covers:** DI registration for Azure SDK clients outside of Aspire, `Microsoft.Extensions.Azure`, and `CancellationToken` propagation.

### DI-1: Microsoft.Extensions.Azure (MEDIUM)
**Pattern:** For non-Aspire web apps and hosted services, use `Microsoft.Extensions.Azure` (`AddAzureClients`) for DI-friendly Azure SDK client registration.

✅ **DO:**
```csharp
using Microsoft.Extensions.Azure;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAzureClients(clientBuilder =>
{
    // ✅ Register clients with URIs from configuration
    clientBuilder.AddBlobServiceClient(
        new Uri($"https://{builder.Configuration["Azure:StorageAccountName"]}.blob.core.windows.net"));
    clientBuilder.AddSecretClient(
        new Uri(builder.Configuration["Azure:KeyVaultUrl"]!));
    clientBuilder.AddServiceBusClient(
        $"{builder.Configuration["Azure:ServiceBusNamespace"]}.servicebus.windows.net");

    // ✅ Shared credential for all clients
    clientBuilder.UseCredential(new DefaultAzureCredential());
});

// ✅ Inject via constructor
public class OrderService(ServiceBusClient serviceBusClient)
{
    public async Task SendOrderAsync(Order order, CancellationToken cancellationToken)
    {
        await using var sender = serviceBusClient.CreateSender("orders");
        await sender.SendMessageAsync(
            new ServiceBusMessage(BinaryData.FromObjectAsJson(order)),
            cancellationToken);
    }
}
```

❌ **DON'T:**
```csharp
// ❌ Don't register Azure clients as singletons manually
builder.Services.AddSingleton(_ => new BlobServiceClient(uri, new DefaultAzureCredential()));
// ❌ Misses retry policy configuration, credential sharing, and proper lifecycle management
```

**Why:** `Microsoft.Extensions.Azure` provides a consistent pattern for registering Azure SDK clients with shared credentials, retry policies, and proper lifecycle management.

---

### DI-2: CancellationToken Propagation (MEDIUM)
**Pattern:** Always pass `CancellationToken` through async call chains. All Azure SDK async methods accept `CancellationToken` as the last parameter.

✅ **DO:**
```csharp
// ✅ API endpoint propagates CancellationToken
app.MapGet("/products", async (
    CosmosClient cosmosClient,
    CancellationToken cancellationToken) =>
{
    var container = cosmosClient.GetDatabase("mydb").GetContainer("products");
    var query = new QueryDefinition("SELECT * FROM c");

    using FeedIterator<Product> feed = container.GetItemQueryIterator<Product>(query);
    var results = new List<Product>();
    while (feed.HasMoreResults)
    {
        FeedResponse<Product> response = await feed.ReadNextAsync(cancellationToken);
        results.AddRange(response);
    }
    return results;
});

// ✅ Service method accepts and passes CancellationToken
public async Task<string> GetSecretAsync(string name, CancellationToken cancellationToken = default)
{
    KeyVaultSecret secret = await _secretClient.GetSecretAsync(name, cancellationToken: cancellationToken);
    return secret.Value;
}
```

❌ **DON'T:**
```csharp
// ❌ Don't drop CancellationToken — request can't be cancelled
public async Task<string> GetSecretAsync(string name)
{
    KeyVaultSecret secret = await _secretClient.GetSecretAsync(name);  // ❌ No cancellation
    return secret.Value;
}
```

**Why:** Without `CancellationToken`, HTTP requests can't be cancelled when clients disconnect, and graceful shutdown is impaired. ASP.NET Core provides `CancellationToken` automatically via endpoint binding.

---

### DI-3: IAsyncEnumerable Streaming Patterns (MEDIUM)
**Pattern:** Azure SDK uses `AsyncPageable<T>` (which implements `IAsyncEnumerable<T>`) for paginated results. Use `await foreach` for efficient streaming without materializing all results in memory.

✅ **DO:**
```csharp
// ✅ Stream results with await foreach
await foreach (BlobItem blob in containerClient.GetBlobsAsync(cancellationToken: cancellationToken))
{
    Console.WriteLine($"Blob: {blob.Name}");
}

// ✅ Stream Key Vault secrets
await foreach (SecretProperties secret in secretClient.GetPropertiesOfSecretsAsync(cancellationToken))
{
    Console.WriteLine($"Secret: {secret.Name}");
}

// ✅ Convert to list only when needed
var allBlobs = new List<BlobItem>();
await foreach (BlobItem blob in containerClient.GetBlobsAsync(cancellationToken: cancellationToken))
{
    allBlobs.Add(blob);
}

// ✅ Use LINQ with System.Linq.Async for filtering
// Install: dotnet add package System.Linq.Async
var largeBlobs = await containerClient.GetBlobsAsync()
    .Where(b => b.Properties.ContentLength > 1_000_000)
    .ToListAsync(cancellationToken);
```

❌ **DON'T:**
```csharp
// ❌ Don't call .ToList() on the first page only
var blobs = containerClient.GetBlobsAsync().AsPages().First();  // ❌ Only first page
```

**Why:** `AsyncPageable<T>` transparently handles pagination across all Azure SDK list operations. `await foreach` is the idiomatic C# pattern and avoids loading all results into memory at once.

---

## 14b. Azure Functions

**What this section covers:** Azure Functions patterns for .NET, specifically the isolated worker model.

### FUNC-1: Isolated Worker Model (MEDIUM)
**Pattern:** New Azure Functions projects should use the **isolated worker model** (out-of-process). The in-process model is being deprecated.

✅ **DO:**
```csharp
// Program.cs — Isolated worker model
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services =>
    {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();
    })
    .Build();

host.Run();
```

```xml
<!-- .csproj — Isolated worker model -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <AzureFunctionsVersion>v4</AzureFunctionsVersion>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="2.*" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.*" />
    <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="2.*" />
  </ItemGroup>
</Project>
```

❌ **DON'T:**
```csharp
// ❌ Don't use in-process model for new projects (being deprecated)
// In-process uses Microsoft.NET.Sdk.Functions and class library pattern
// [FunctionName("MyFunction")] — this is the old in-process attribute
```

**Why:** The isolated worker model runs functions in a separate process, giving you full control over dependencies, middleware, and .NET versions. The in-process model is scheduled for deprecation.

---

## 15. CI/CD & Testing

**What this section covers:** Continuous integration, build validation, and testing patterns for .NET samples.

### CI-1: Build and Test Validation (HIGH)
**Pattern:** Run build, test, and vulnerability scanning in CI.

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
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - run: dotnet restore
      - run: dotnet build --no-restore --warnaserror
      - run: dotnet test --no-build --verbosity normal
      - name: Check for vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln.txt
          if grep -q "has the following vulnerable packages" vuln.txt; then
            echo "::error::Vulnerable packages found"
            exit 1
          fi
```

```json
// global.json (ensure CI uses correct SDK)
{
  "sdk": {
    "version": "9.0.200",
    "rollForward": "latestFeature"
  }
}
```

---

## Pre-Review Checklist (Comprehensive)

Use this comprehensive checklist before submitting an Azure SDK .NET sample for review:

### 🔧 Project Setup
- [ ] Target framework `net8.0`, `net9.0`, or `net10.0`
- [ ] `<Nullable>enable</Nullable>` in .csproj
- [ ] `<ImplicitUsings>enable</ImplicitUsings>` in .csproj
- [ ] Project metadata (Description, Authors) present
- [ ] Every NuGet package is used in code (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`Azure.*`, exception: `Microsoft.Azure.Cosmos` is current)
- [ ] Configuration validated with descriptive errors
- [ ] `global.json` pins SDK version
- [ ] `packages.lock.json` committed (if lock file enabled)
- [ ] `dotnet list package --vulnerable` passes
- [ ] .editorconfig present (optional but recommended)

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `bin/`, `obj/`, `.env`, `appsettings.Development.json`, `.azure/`
- [ ] `appsettings.json` has placeholders only (no real credentials)
- [ ] No live credentials committed (subscription IDs, tenant IDs, connection strings, API keys)
- [ ] No build artifacts committed (`bin/`, `obj/`)
- [ ] Dead code removed (unused files, methods, using directives)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included
- [ ] Package legitimacy verified (all `Azure.*` from Microsoft)

### ☁️ Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Client options configured (retry policies, diagnostics)
- [ ] Token refresh implemented for long-running operations (CRITICAL)
- [ ] Managed identity documented in README
- [ ] Pagination handled with `await foreach` / `AsyncPageable<T>`
- [ ] Resource cleanup with `await using` / `IAsyncDisposable`
- [ ] `CancellationToken` propagated through async call chains
- [ ] `Microsoft.Extensions.Azure` / `AddAzureClients()` used for DI (web apps)
- [ ] `IAsyncEnumerable` streaming patterns used (not materializing all pages)

### 🗄️ Data Services (if applicable)
- [ ] SQL: Uses `Microsoft.Data.SqlClient` with AAD tokens (not SQL auth)
- [ ] SQL: All identifiers use `[brackets]`
- [ ] SQL: Values parameterized (`SqlParameter`, not string concat)
- [ ] SQL: Bulk operations (`SqlBulkCopy`) for large datasets
- [ ] Cosmos: Queries include partition key
- [ ] Cosmos: Transactional batch for multi-operation scenarios
- [ ] Storage: Blob/Table client patterns followed
- [ ] Storage: SAS fallback documented for local dev (optional)

### 🤖 AI Services (if applicable)
- [ ] Using `Azure.AI.OpenAI` with `DefaultAzureCredential`
- [ ] Client configured with retry policies
- [ ] API versions documented with links
- [ ] Vector dimensions validated
- [ ] Pre-computed embedding data committed to repo

### 💬 Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses `EventProcessorClient` with checkpoint store
- [ ] Connection strings avoided (prefer AAD auth)

### ❌ Error Handling
- [ ] `RequestFailedException` caught for Azure errors
- [ ] Status code pattern matching for specific error scenarios
- [ ] Error messages include troubleshooting guidance

### 📄 Documentation
- [ ] README "Expected output" from real run (not fabricated)
- [ ] All internal links match actual filesystem paths
- [ ] Prerequisites complete (.NET SDK, Azure CLI, role assignments)
- [ ] Troubleshooting section covers auth/RBAC/network errors
- [ ] Setup instructions clear and tested
- [ ] Placeholder values have clear instructions

### 🏗️ Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated (`@minLength`, `@maxLength`, `@allowed`)
- [ ] API versions current (2023+)
- [ ] RBAC role assignments for managed identities
- [ ] Resource naming follows CAF conventions
- [ ] `azure.yaml` services, hooks, metadata

### 🚀 .NET Aspire (if applicable)
- [ ] AppHost configures Azure resources with `.AddAzure*()`
- [ ] `RunAsEmulator()` for local development
- [ ] ServiceDefaults project with OpenTelemetry + health checks
- [ ] Service discovery (`https+http://servicename`) used—not hardcoded URLs
- [ ] Health checks registered for Azure dependencies
- [ ] `.WithReference()` wires dependencies—not manual env vars
- [ ] `Aspire.Azure.*` client integration packages used in service projects
- [ ] `builder.AddAzureBlobClient()` etc. used—not manual `new BlobServiceClient()`
- [ ] Aspire SDK versions not hardcoded to specific patch (use latest stable)

### 🧪 CI/CD
- [ ] `dotnet build --warnaserror` in CI
- [ ] `dotnet test` in CI
- [ ] CVE scanning (`dotnet list package --vulnerable`) in CI
- [ ] `global.json` ensures consistent SDK in CI

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`azure-sdk-typescript-sample-review`](../azure-sdk-typescript-sample-review/SKILL.md)**: TypeScript Azure SDK sample review (sister skill)
- **[`azure-sdk-java-sample-review`](../azure-sdk-java-sample-review/SKILL.md)**: Java 17/21 + Spring Boot Azure SDK sample review
- **[`azure-sdk-python-sample-review`](../azure-sdk-python-sample-review/SKILL.md)**: Python 3.9+ + async Azure SDK sample review
- **[`azure-sdk-go-sample-review`](../azure-sdk-go-sample-review/SKILL.md)**: Go 1.21+ Azure SDK sample review
- **[`azure-sdk-rust-sample-review`](../azure-sdk-rust-sample-review/SKILL.md)**: Rust 2021 edition Azure SDK sample review
- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency

---

## Scope Note: Services Not Yet Covered

This skill focuses on the most commonly used Azure services in .NET samples. The following services are not yet covered in detail, but the general patterns (authentication, client construction, error handling) apply:

- Azure Communication Services
- Azure Cache for Redis (`StackExchange.Redis` / `Microsoft.Extensions.Caching.StackExchangeRedis`)
- Azure Monitor / Application Insights
- Azure Container Registry
- Azure App Configuration (`Azure.Data.AppConfiguration`)
- Azure SignalR Service
- Azure API Management

For samples using these services, apply the core patterns from Sections 1–2 (Project Setup, Azure SDK Client Patterns) and reference service-specific documentation.

---

## Reference Links

Consolidation of all documentation links referenced throughout this skill:

### Azure SDK & Authentication
- [Azure SDK for .NET](https://learn.microsoft.com/dotnet/azure/sdk/azure-sdk-for-dotnet)
- [Azure SDK Package Index](https://azure.github.io/azure-sdk/releases/latest/dotnet.html)
- [DefaultAzureCredential](https://learn.microsoft.com/dotnet/api/azure.identity.defaultazurecredential)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
- [Azure.Identity](https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme)

### .NET Aspire
- [.NET Aspire Overview](https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview)
- [Aspire Service Discovery](https://learn.microsoft.com/dotnet/aspire/networking/service-discovery/overview)
- [Aspire Azure Integrations](https://learn.microsoft.com/dotnet/aspire/fundamentals/integrations-overview)
- [Aspire Health Checks](https://learn.microsoft.com/dotnet/aspire/fundamentals/health-checks)

### API Versioning
- [Azure OpenAI API Versions](https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation)
- [Azure REST API Specifications](https://github.com/Azure/azure-rest-api-specs)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework—Naming](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Security
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Azure Private Endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)

### .NET
- [.NET Downloads](https://dot.net/download)
- [.NET Support Policy](https://dotnet.microsoft.com/platform/support/policy)
- [C# Language Reference](https://learn.microsoft.com/dotnet/csharp/)
- [System.Text.Json](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview)
- [Microsoft.Data.SqlClient](https://learn.microsoft.com/sql/connect/ado-net/microsoft-ado-net-sql-server)
- [Microsoft.Extensions.Azure](https://learn.microsoft.com/dotnet/api/overview/azure/microsoft.extensions.azure-readme)
- [Azure Functions Isolated Worker](https://learn.microsoft.com/azure/azure-functions/dotnet-isolated-process-guide)
- [dotnet user-secrets](https://learn.microsoft.com/aspnet/core/security/app-secrets)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK .NET sample patterns** adapted from comprehensive TypeScript sample reviews plus generalized .NET and Aspire patterns across the Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (9 rules): Phantom deps, CVE scanning, token refresh, AVM versions, parameter validation, .gitignore, fabricated output, broken links, credentials
- **HIGH** (25 rules): Client construction, managed identity, pagination, OpenAI config, database patterns, DiskANN guards, batch operations, RBAC, lock files, role assignments, pre-computed data, appsettings, prerequisites, dead code, LICENSE, resource naming, target framework, SDK naming, Service Bus, Key Vault, Aspire AppHost, Aspire client integration packages, Aspire DI vs manual, CI build, network security
- **MEDIUM** (33 rules): Client options, retry policies, SQL parameters, embeddings, error handling, JSON loading, troubleshooting, azd structure, nullable types, version pinning, SAS fallback, dimensions, placeholder docs, resource cleanup, API versions, governance files, setup instructions, configuration, .NET version docs, Aspire ServiceDefaults, service discovery, health checks, Event Hubs, metadata, host types, Document Intelligence, Storage patterns, project metadata, package legitimacy, Microsoft.Extensions.Azure DI, CancellationToken propagation, IAsyncEnumerable streaming, Azure Functions isolated worker
- **LOW** (4 rules): API version docs, .editorconfig, .NET version strategy, Aspire dashboard/OTLP

### Service Coverage
- **Core SDK**: Authentication, credentials, managed identities, client patterns, token management, pagination, resource cleanup, CancellationToken propagation, IAsyncEnumerable streaming
- **DI Patterns**: Microsoft.Extensions.Azure (AddAzureClients), Aspire client integration packages (Aspire.Azure.*)
- **Data**: Cosmos DB, Azure SQL (Microsoft.Data.SqlClient), Storage (Blob/Table/File), batch operations
- **AI**: Azure OpenAI (embeddings, chat, System.ClientModel retry), Document Intelligence, Speech, vector dimensions
- **Messaging**: Service Bus, Event Hubs, checkpoint management
- **Security**: Key Vault (secrets, keys, certificates)
- **Vector Search**: Azure SQL DiskANN, Cosmos DB, AI Search
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming
- **.NET Aspire**: AppHost orchestration, ServiceDefaults, service discovery, health checks, OpenTelemetry, client integration packages
- **Azure Functions**: Isolated worker model

Apply these patterns to ensure Azure SDK .NET samples are **secure, accurate, maintainable, and ready for publication**.
