# Infrastructure, azd, .NET Aspire, DI, Azure Functions, CI/CD & Documentation

Bicep/Terraform patterns, azd integration, .NET Aspire orchestration, DI patterns, Azure Functions, CI/CD, and README/documentation rules.

---

## README & Documentation

### DOC-1: Expected Output (CRITICAL)

README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

DO:
```markdown
## Expected Output

Run the sample:

    dotnet run

You should see output similar to:

    Connected to Azure Blob Storage
    Container 'samples' created
    Uploaded blob 'sample.txt' (14 bytes)
    Downloaded blob content: "Hello, Azure!"

> Note: Exact output may vary based on your Azure environment.
```

### DOC-2: Folder Path Links (CRITICAL)

All internal README links must match actual filesystem paths. Verify every link with `git ls-files` or the PR file list.

### DOC-3: Troubleshooting Section (MEDIUM)

Include troubleshooting for common Azure errors (auth, RBAC, network).

DO:
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

### DOC-4: Prerequisites Section (HIGH)

Document all prerequisites clearly including Azure resources and role assignments.

DO:
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **.NET SDK**: Version 9.0 or later ([Download](https://dot.net/download))
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)

### Role Assignments

Your Azure identity needs:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
- `Cognitive Services OpenAI User` on the Azure OpenAI resource
```

### DOC-5: Setup Instructions (MEDIUM)

Provide clear, tested setup instructions with clone, provision, and run steps.

### DOC-6: .NET Version Strategy (LOW)

Document minimum .NET version in README.

### DOC-7: Placeholder Values (MEDIUM)

READMEs must provide clear instructions for placeholder values including where to find values in Azure Portal.

---

## Infrastructure (Bicep/Terraform)

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)

Use current stable versions of Azure Verified Modules.

DO:
```bicep
module storage 'br/public:avm/res/storage/storage-account:0.14.0' = {
  name: 'storage-deployment'
  params: {
    name: storageAccountName
    location: location
  }
}
```

### IaC-2: Bicep Parameter Validation (CRITICAL)

Use `@minLength`, `@maxLength`, `@allowed` decorators.

DO:
```bicep
@description('Azure AD admin object ID')
@minLength(36)
@maxLength(36)
param aadAdminObjectId string

@description('Azure region for deployment')
@allowed(['eastus', 'eastus2', 'westus2', 'westus3', 'centralus'])
param location string = 'eastus'
```

DON'T:
```bicep
param aadAdminObjectId string  // No validation, accepts empty string
```

### IaC-3: API Versions (MEDIUM)

Use current API versions (2023+). Avoid versions older than 2 years.

### IaC-4: RBAC Role Assignments (HIGH)

Create role assignments in Bicep for managed identities.

DO:
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

### IaC-5: Network Security (HIGH)

For quickstart samples, public endpoints acceptable with security comment.

### IaC-6: Output Values (MEDIUM)

Output all values needed by the application.

DO:
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
output AZURE_COSMOS_ENDPOINT string = cosmos.properties.documentEndpoint
```

### IaC-7: Resource Naming Conventions (HIGH)

Follow Cloud Adoption Framework (CAF) naming conventions.

---

## Azure Developer CLI (azd)

### AZD-1: azure.yaml Structure (MEDIUM)

> **FALSE POSITIVE PREVENTION:** `services`, `hooks`, and `host` fields are OPTIONAL for infrastructure-only samples.

DO:
```yaml
name: azure-storage-blob-dotnet-sample
metadata:
  template: azure-storage-blob-dotnet-sample@0.0.1

services:
  app:
    project: ./src
    language: csharp
    host: appservice

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)
```

### AZD-2: Service Host Types (MEDIUM)

Choose correct `host` type for .NET applications: `appservice`, `function`, or `containerapp`.

---

## .NET Aspire Integration

### ASP-1: AppHost Configuration (HIGH)

Configure the Aspire AppHost to orchestrate Azure resources and application services.

DO:
```csharp
// AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

var storage = builder.AddAzureStorage("storage")
    .RunAsEmulator();
var blobs = storage.AddBlobs("blobs");

var cosmos = builder.AddAzureCosmosDB("cosmos")
    .RunAsEmulator();
var database = cosmos.AddDatabase("mydb");

var openai = builder.AddAzureOpenAI("openai")
    .AddDeployment(new AzureOpenAIDeployment("gpt-4o", "gpt-4o", "2024-08-06"));

var api = builder.AddProject<Projects.MyApi>("api")
    .WithReference(blobs)
    .WithReference(database)
    .WithReference(openai)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

DON'T:
```csharp
// Don't hardcode connection strings in AppHost
var api = builder.AddProject<Projects.MyApi>("api")
    .WithEnvironment("ConnectionStrings__Storage",
        "DefaultEndpointsProtocol=https;AccountName=...");  // Use .WithReference()

// Don't skip RunAsEmulator for local development
var storage = builder.AddAzureStorage("storage");
// Without RunAsEmulator, requires live Azure resources for local dev
```

### ASP-2: ServiceDefaults Setup (MEDIUM)

Use the ServiceDefaults project for shared configuration -- OpenTelemetry, health checks, resilience.

DO:
```csharp
// ServiceDefaults/Extensions.cs
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

// MyApi/Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
var app = builder.Build();
app.MapDefaultEndpoints();  // /health and /alive
```

### ASP-3: Service Discovery Patterns (MEDIUM)

Use Aspire service discovery for inter-service communication instead of hardcoded URLs.

DO:
```csharp
builder.Services.AddHttpClient<ApiClient>(client =>
{
    client.BaseAddress = new Uri("https+http://api");  // Resolved by Aspire
});
```

DON'T:
```csharp
builder.Services.AddHttpClient<ApiClient>(client =>
{
    client.BaseAddress = new Uri("https://localhost:5001");  // Hardcoded
});
```

### ASP-4: Health Checks (MEDIUM)

Register health checks for Azure dependencies. Map health endpoints.

### ASP-5: Aspire Dashboard / OpenTelemetry (LOW)

Leverage the Aspire dashboard for local observability. Configure OTLP exporters for production.

### ASP-6: Aspire Client Integration Packages (HIGH)

In Aspire service projects, use `Aspire.Azure.*` client integration packages instead of manual client construction.

DO:
```csharp
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();

builder.AddAzureBlobClient("blobs");
builder.AddAzureCosmosClient("cosmos");
builder.AddAzureKeyVaultClient("keyvault");
builder.AddAzureOpenAIClient("openai");
```

DON'T:
```csharp
// Don't manually construct clients in Aspire projects
var blobClient = new BlobServiceClient(
    new Uri("https://mystorageaccount.blob.core.windows.net"),
    new DefaultAzureCredential());  // Bypasses Aspire DI, health checks, telemetry
```

**Key packages:**
| Aspire Package | DI Method |
|---|---|
| `Aspire.Azure.Storage.Blobs` | `builder.AddAzureBlobClient()` |
| `Aspire.Azure.Data.Cosmos` | `builder.AddAzureCosmosClient()` |
| `Aspire.Azure.Security.KeyVault` | `builder.AddAzureKeyVaultClient()` |
| `Aspire.Azure.AI.OpenAI` | `builder.AddAzureOpenAIClient()` |
| `Aspire.Azure.Messaging.ServiceBus` | `builder.AddAzureServiceBusClient()` |

### ASP-7: Aspire DI vs Manual Client Construction (HIGH)

Distinguish between Aspire DI-based client registration and manual construction. Use the correct pattern for the project type.

DO (Aspire project):
```csharp
builder.AddAzureBlobClient("blobs");  // Name matches AppHost resource name
```

DO (Non-Aspire project):
```csharp
using Microsoft.Extensions.Azure;

builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddBlobServiceClient(
        new Uri($"https://{config["Azure:StorageAccountName"]}.blob.core.windows.net"));
    clientBuilder.UseCredential(new DefaultAzureCredential());
});
```

DO (Console app / simple sample):
```csharp
var credential = new DefaultAzureCredential();
var blobClient = new BlobServiceClient(
    new Uri($"https://{accountName}.blob.core.windows.net"), credential);
```

---

## Azure SDK Dependency Injection Patterns

### DI-1: Microsoft.Extensions.Azure (MEDIUM)

For non-Aspire web apps and hosted services, use `Microsoft.Extensions.Azure` (`AddAzureClients`) for DI-friendly Azure SDK client registration.

DO:
```csharp
using Microsoft.Extensions.Azure;
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAzureClients(clientBuilder =>
{
    clientBuilder.AddBlobServiceClient(
        new Uri($"https://{builder.Configuration["Azure:StorageAccountName"]}.blob.core.windows.net"));
    clientBuilder.AddSecretClient(
        new Uri(builder.Configuration["Azure:KeyVaultUrl"]!));
    clientBuilder.AddServiceBusClient(
        $"{builder.Configuration["Azure:ServiceBusNamespace"]}.servicebus.windows.net");
    clientBuilder.UseCredential(new DefaultAzureCredential());
});
```

DON'T:
```csharp
// Don't register Azure clients as singletons manually
builder.Services.AddSingleton(_ => new BlobServiceClient(uri, new DefaultAzureCredential()));
// Misses retry policy configuration, credential sharing, and proper lifecycle management
```

### DI-2: CancellationToken Propagation (MEDIUM)

Always pass `CancellationToken` through async call chains.

DO:
```csharp
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
```

DON'T:
```csharp
// Don't drop CancellationToken -- request can't be cancelled
public async Task<string> GetSecretAsync(string name)
{
    KeyVaultSecret secret = await _secretClient.GetSecretAsync(name);  // No cancellation
    return secret.Value;
}
```

### DI-3: IAsyncEnumerable Streaming Patterns (MEDIUM)

Use `await foreach` with `AsyncPageable<T>` for efficient streaming.

DO:
```csharp
await foreach (BlobItem blob in containerClient.GetBlobsAsync(cancellationToken: cancellationToken))
{
    Console.WriteLine($"Blob: {blob.Name}");
}

// Use LINQ with System.Linq.Async for filtering
var largeBlobs = await containerClient.GetBlobsAsync()
    .Where(b => b.Properties.ContentLength > 1_000_000)
    .ToListAsync(cancellationToken);
```

---

## Azure Functions

### FUNC-1: Isolated Worker Model (MEDIUM)

New Azure Functions projects should use the isolated worker model (out-of-process).

DO:
```csharp
// Program.cs
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

DON'T:
```csharp
// Don't use in-process model for new projects (being deprecated)
// [FunctionName("MyFunction")] — this is the old in-process attribute
```

---

## CI/CD & Testing

### CI-1: Build and Test Validation (HIGH)

Run build, test, and vulnerability scanning in CI.

DO:
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

---

## Pre-Review Checklist (Comprehensive)

### Project Setup
- [ ] Target framework `net8.0`, `net9.0`, or `net10.0`
- [ ] `<Nullable>enable</Nullable>` in .csproj
- [ ] Every NuGet package is used in code (no phantom deps)
- [ ] Track 2 Azure SDK packages (`Azure.*`, exception: `Microsoft.Azure.Cosmos`)
- [ ] Configuration validated with descriptive errors
- [ ] `global.json` pins SDK version
- [ ] `packages.lock.json` committed (if lock file enabled)
- [ ] `dotnet list package --vulnerable` passes

### Security & Hygiene
- [ ] `.gitignore` protects `bin/`, `obj/`, `.env`, `appsettings.Development.json`
- [ ] `appsettings.json` has placeholders only
- [ ] No live credentials committed
- [ ] LICENSE file present (MIT required for Azure Samples)

### Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused
- [ ] Pagination handled with `await foreach` / `AsyncPageable<T>`
- [ ] Resource cleanup with `await using` / `IAsyncDisposable`
- [ ] `CancellationToken` propagated through async call chains

### Documentation
- [ ] README "Expected output" from real run
- [ ] All internal links match actual filesystem paths
- [ ] Prerequisites complete (.NET SDK, Azure CLI, role assignments)
- [ ] Troubleshooting section covers auth/RBAC errors

### Infrastructure (if applicable)
- [ ] AVM versions current
- [ ] Bicep parameters validated
- [ ] RBAC role assignments for managed identities
- [ ] `azure.yaml` present and valid

### .NET Aspire (if applicable)
- [ ] AppHost configures Azure resources with `.AddAzure*()`
- [ ] `RunAsEmulator()` for local development
- [ ] ServiceDefaults with OpenTelemetry + health checks
- [ ] `Aspire.Azure.*` client integration packages used
- [ ] `.WithReference()` wires dependencies

### CI/CD
- [ ] `dotnet build --warnaserror` in CI
- [ ] `dotnet test` in CI
- [ ] CVE scanning in CI
