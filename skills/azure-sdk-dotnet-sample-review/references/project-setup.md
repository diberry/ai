# Project Setup & Configuration

Rules for .csproj structure, target framework, dependency management, nullable reference types, environment configuration, and SDK pinning.

## PS-1: Target Framework (HIGH)

Target current supported .NET versions. Both LTS and STS releases are acceptable while in support.

DO:
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

DON'T:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>  <!-- Out of support (Nov 2024) -->
    <Nullable>disable</Nullable>                <!-- Nullable should be enabled -->
  </PropertyGroup>
</Project>
```

**Why:** .NET 8 is the current LTS (until Nov 2026). .NET 9 is STS (until May 2026). .NET 10 is the next LTS. Samples must target supported frameworks.

---

## PS-2: Project Metadata in .csproj (MEDIUM)

Include descriptive metadata for discoverability and maintenance.

DO:
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

DON'T:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <!-- Missing OutputType, Nullable, Description, Authors -->
  </PropertyGroup>
</Project>
```

---

## PS-3: NuGet Dependency Audit (CRITICAL)

Every NuGet package must be referenced somewhere in code. No phantom dependencies. Use current Azure SDK Track 2 packages (`Azure.*`).

DO:
```xml
<ItemGroup>
  <PackageReference Include="Azure.Storage.Blobs" Version="12.*" />
  <PackageReference Include="Azure.Identity" Version="1.*" />
  <PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.*" />
  <PackageReference Include="Azure.AI.OpenAI" Version="2.*" />
</ItemGroup>
```

DON'T:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Azure.Storage.Blob" Version="11.2.3" /> <!-- Track 1 (legacy) -->
  <PackageReference Include="Azure.Messaging.ServiceBus" Version="7.18.2" />   <!-- Listed but never used -->
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />              <!-- Use System.Text.Json -->
</ItemGroup>
```

**Why:** Phantom dependencies bloat samples and confuse users. `System.Text.Json` is built-in since .NET Core 3.0—prefer it over Newtonsoft.Json in new samples.

---

## PS-4: Azure SDK Package Naming (HIGH)

Use Track 2 packages (`Azure.*`) not Track 1 legacy packages (`Microsoft.Azure.*`).

DO (Track 2):
```csharp
using Azure.Storage.Blobs;
using Azure.Security.KeyVault.Secrets;
using Azure.Messaging.ServiceBus;
using Microsoft.Azure.Cosmos;  // Current Cosmos DB SDK (v3.x) — Track 2 patterns despite Microsoft.Azure.* namespace
using Azure.Data.Tables;
using Azure.Identity;
using Azure.AI.OpenAI;
```

DON'T (Track 1 Legacy):
```csharp
using Microsoft.Azure.Storage;             // Use Azure.Storage.Blobs
using Microsoft.Azure.KeyVault;            // Use Azure.Security.KeyVault.Secrets
using Microsoft.Azure.ServiceBus;          // Use Azure.Messaging.ServiceBus
using Microsoft.WindowsAzure.Storage;      // Ancient, never use
using Microsoft.Azure.Documents.Client;    // Legacy Cosmos DB SDK — use Microsoft.Azure.Cosmos v3.x
```

**Why:** Track 2 SDKs (`Azure.*`) are current generation with consistent APIs, `TokenCredential` support, and active maintenance. **Exception:** `Microsoft.Azure.Cosmos` (v3.x) is the current, recommended Cosmos DB SDK.

---

## PS-5: Environment/Configuration (MEDIUM)

Use `appsettings.json` + User Secrets for local development. Validate all required configuration with descriptive errors.

DO:
```csharp
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

DON'T:
```csharp
string storageAccount = "contosoprod";  // Don't hardcode
string storageAccount = configuration["Azure:StorageAccountName"] ?? "devstoreaccount1";  // Don't silently fall back
```

**Tip:** For local development, prefer `dotnet user-secrets` over `.env` files:
```bash
dotnet user-secrets init
dotnet user-secrets set "Azure:StorageAccountName" "mystorageaccount"
```

---

## PS-6: Nullable Reference Types (MEDIUM)

Enable nullable reference types. Use null checks and `??` operators consistently.

DO:
```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
public async Task<string?> GetSecretValueAsync(string secretName)
{
    KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
    return secret.Value;
}

string? secretValue = await GetSecretValueAsync("db-password");
string connectionString = secretValue
    ?? throw new InvalidOperationException("Secret 'db-password' not found in Key Vault.");
```

DON'T:
```csharp
string secretValue = secret.Value!;  // Null-forgiving operator hides bugs
// <Nullable>disable</Nullable>
```

---

## PS-7: .editorconfig (LOW)

Include `.editorconfig` for consistent formatting across editors.

DO:
```ini
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

## PS-8: Lock File Management (HIGH)

Enable and commit `packages.lock.json` for reproducible builds.

DO:
```xml
<!-- Directory.Build.props or .csproj -->
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

DON'T:
```gitignore
# Don't ignore lock files
packages.lock.json
```

**Why:** Lock files ensure reproducible builds. Without them, `dotnet restore` may resolve different package versions on different machines.

---

## PS-9: CVE Scanning (CRITICAL)

Samples must not ship with known security vulnerabilities.

DO:
```bash
dotnet list package --vulnerable
dotnet list package --deprecated
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

---

## PS-10: Package Legitimacy (MEDIUM)

Verify Azure SDK packages are from official `Azure.*` namespace. Watch for typosquatting.

DO:
```xml
<ItemGroup>
  <PackageReference Include="Azure.Storage.Blobs" Version="12.*" />
  <PackageReference Include="Azure.Identity" Version="1.*" />
  <PackageReference Include="Azure.AI.OpenAI" Version="2.*" />
</ItemGroup>
```

DON'T:
```xml
<ItemGroup>
  <PackageReference Include="AzureStorageBlobs" Version="1.0.0" />  <!-- Typosquatting -->
  <PackageReference Include="Azure-Identity" Version="1.0.0" />     <!-- Not official -->
</ItemGroup>
```

---

## PS-11: Version Pinning via global.json (MEDIUM)

Pin SDK version with `global.json` to ensure consistent builds across environments.

DO:
```json
{
  "sdk": {
    "version": "9.0.200",
    "rollForward": "latestFeature"
  }
}
```

DON'T:
```json
{
  "sdk": {
    "version": "9.0.100",
    "rollForward": "disable"
  }
}
```

**Why:** `global.json` ensures all contributors use the same SDK version. `rollForward: "latestFeature"` allows patch updates within the feature band.
