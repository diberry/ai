# Error Handling, Data Management & Sample Hygiene

Exception handling, data files, and repository hygiene rules.

## ERR-1: Azure SDK Exception Handling (MEDIUM)

Catch `Azure.RequestFailedException` for Azure service errors. Use pattern matching for specific status codes.

DO:
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

DON'T:
```csharp
// Catching generic Exception loses Azure-specific context
try
{
    await blobClient.DownloadContentAsync();
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);  // No status code, no troubleshooting
}
```

---

## ERR-2: Contextual Error Messages (MEDIUM)

Provide actionable error messages with troubleshooting hints for common Azure errors.

DO:
```csharp
try
{
    await credential.GetTokenAsync(
        new Azure.Core.TokenRequestContext(
            new[] { "https://storage.azure.com/.default" }));
}
catch (Azure.Identity.AuthenticationFailedException ex)
{
    Console.Error.WriteLine($"Failed to acquire Azure Storage token: {ex.Message}");
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

## DATA-1: Pre-Computed Data Files (HIGH)

Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

DO:
```
repo/
  data/
    products.json              # Sample data
    products-with-vectors.json # Pre-computed embeddings
  src/
    Program.cs                 # Loads products-with-vectors.json
    GenerateEmbeddings.cs      # Generates embeddings (optional)
```

DON'T:
```
repo/
  data/
    products.json              # Raw data
    .gitignore                 # products-with-vectors.json gitignored
  src/
    Program.cs                 # Fails: File not found
```

> **FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. Check the FULL PR file list -- not just the immediate project directory.
> 2. Trace the file path in code relative to the working directory.
> 3. Check for monorepo patterns -- data may be shared across multiple samples.

---

## DATA-2: JSON Data Loading (MEDIUM)

Use `System.Text.Json` with strongly-typed models. Embed data files or use path resolution.

DO:
```csharp
using System.Text.Json;

public record Product(string Id, string Name, string Category);

string dataPath = Path.Combine(AppContext.BaseDirectory, "..", "..", "..", "data", "products.json");
string json = await File.ReadAllTextAsync(dataPath);
List<Product> products = JsonSerializer.Deserialize<List<Product>>(json,
    new JsonSerializerOptions { PropertyNameCaseInsensitive = true })
    ?? throw new InvalidOperationException("Failed to deserialize products.json");
```

DON'T:
```csharp
// Don't use Newtonsoft.Json in new samples
using Newtonsoft.Json;
var products = JsonConvert.DeserializeObject<List<Product>>(json);

// Don't use hardcoded relative paths
string json = File.ReadAllText("../data/products.json");  // Breaks depending on CWD
```

---

## HYG-1: .gitignore (CRITICAL)

Always protect sensitive files, build artifacts, and dependencies.

DO:
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

> **FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed:
> 1. Check .gitignore in the project directory AND all parent directories.
> 2. Run `git ls-files .env` -- if empty, the file is NOT tracked.
> 3. A `.env` on disk but gitignored is NOT a security issue.

---

## HYG-2: appsettings.json / .env.sample (HIGH)

Provide `appsettings.json` with placeholder values OR `.env.sample`. Never commit actual secrets.

DO:
```json
{
  "Azure": {
    "StorageAccountName": "<your-storage-account>",
    "KeyVaultUrl": "https://<your-keyvault>.vault.azure.net/",
    "CosmosEndpoint": "https://<your-cosmos>.documents.azure.com:443/",
    "OpenAIEndpoint": "https://<your-openai>.openai.azure.com/"
  }
}
```

DON'T:
```json
{
  "Azure": {
    "StorageAccountName": "contosoprod",
    "SubscriptionId": "12345678-1234-1234-1234-123456789abc"
  }
}
```

---

## HYG-3: Dead Code (HIGH)

Remove unused files, methods, and using directives.

DO:
```csharp
using Azure.Storage.Blobs;
using Azure.Identity;
```

DON'T:
```csharp
// using Azure.Messaging.ServiceBus;  // Was used in old version
//
// async Task OldImplementation()
// {
//     // This was the old way...
// }

using Azure.Storage.Blobs;
```

---

## HYG-4: LICENSE File (HIGH)

All Azure Samples repositories must include MIT LICENSE file.

> **FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. Check the REPO ROOT for `LICENSE`, `LICENSE.md`, `LICENSE.txt`.
> 2. Check parent directories -- in monorepos, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.

---

## HYG-5: Repository Governance Files (MEDIUM)

Samples in Azure Samples org should reference governance files.

DO:
```
repo/
  LICENSE
  README.md
  CONTRIBUTING.md      # Contribution guidelines
  SECURITY.md          # Security reporting
  CODEOWNERS           # Code ownership (optional)
```
