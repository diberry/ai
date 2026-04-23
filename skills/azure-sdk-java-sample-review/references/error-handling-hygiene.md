# Error Handling, Data Management & Sample Hygiene

**What this section covers:** Java exception hierarchy, contextual error messages, sample data handling, pre-computed files, .gitignore patterns, environment file protection, license files, and repository governance.

## ERR-1: Azure SDK Exception Hierarchy (MEDIUM)
**Pattern:** Catch specific Azure SDK exceptions (`HttpResponseException`, `ResourceNotFoundException`) before general `Exception`. Log status codes and error messages.

DO:
```java
import com.azure.core.exception.HttpResponseException;
import com.azure.core.exception.ResourceNotFoundException;

try {
    secretClient.getSecret("my-secret");
} catch (ResourceNotFoundException e) {
    System.err.printf("Secret not found: %s%n", e.getMessage());
    System.err.println("Ensure the secret exists in Key Vault and your identity has 'Key Vault Secrets User' role.");
} catch (HttpResponseException e) {
    System.err.printf("Azure request failed (status %d): %s%n", e.getResponse().getStatusCode(), e.getMessage());
    if (e.getResponse().getStatusCode() == 403) {
        System.err.println("Check RBAC role assignments for your identity.");
    }
} catch (Exception e) {
    System.err.printf("Unexpected error: %s%n", e.getMessage());
    throw e;
}
```

DON'T:
```java
try {
    secretClient.getSecret("my-secret");
} catch (Exception e) {  // Catches everything, loses specific error info
    e.printStackTrace();  // Stack trace dump is not user-friendly
}
```

---

## ERR-2: Contextual Error Messages (MEDIUM)
**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

DO:
```java
try {
    var credential = new DefaultAzureCredentialBuilder().build();
    credential.getToken(new TokenRequestContext()
        .addScopes("https://storage.azure.com/.default")).block();
} catch (Exception e) {
    System.err.println("Failed to acquire Azure Storage token: " + e.getMessage());
    System.err.println();
    System.err.println("Troubleshooting:");
    System.err.println("  1. Run 'az login' to authenticate with Azure CLI");
    System.err.println("  2. Verify you have the 'Storage Blob Data Contributor' role");
    System.err.println("  3. Check your Azure subscription is active");
    System.err.println("  4. Ensure firewall rules allow access from your IP");
    throw e;
}
```

---

## DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

DO:
```
repo/
├── src/main/resources/
│   ├── data/
│   │   ├── products.json              # Sample data
│   │   ├── products-with-vectors.json # Pre-computed embeddings
├── src/main/java/
│   ├── App.java                       # Loads products-with-vectors.json from resources
│   ├── GenerateEmbeddings.java        # Generates embeddings (optional)
```

DON'T:
```
repo/
├── src/main/resources/data/
│   ├── products.json              # Raw data
│   ├── .gitignore                 # products-with-vectors.json gitignored
├── src/main/java/
│   ├── App.java                   # Fails: Resource not found
```

> **FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list** -- not just the immediate project directory. Data files often live in sibling directories, parent directories, or shared `data/` folders.
> 2. **Trace the file path in code** relative to the classpath. Check for `getResourceAsStream()`, `ClassLoader.getResource()`, or `Path.of()` calls.
> 3. **Check for monorepo patterns** -- in sample collections, data may be shared across multiple samples via a common parent directory.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR or repo at the path the code resolves to at runtime.

---

## DATA-2: Resource File Loading (MEDIUM)
**Pattern:** Use classpath resource loading for bundled data files.

DO:
```java
import java.io.InputStream;
import com.fasterxml.jackson.databind.ObjectMapper;

// Load from classpath (works in JAR and IDE)
try (InputStream is = App.class.getResourceAsStream("/data/products.json")) {
    if (is == null) {
        throw new IllegalStateException("Resource not found: /data/products.json");
    }
    List<Product> products = objectMapper.readValue(is, new TypeReference<>() {});
}
```

DON'T:
```java
// Hardcoded file path (breaks when running from different directories)
Path path = Path.of("src/main/resources/data/products.json");
List<Product> products = objectMapper.readValue(path.toFile(), new TypeReference<>() {});
```

---

## HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and dependencies with comprehensive `.gitignore`.

DO:
```gitignore
# Environment variables (may contain credentials)
.env
.env.local
.env.*
!.env.sample
!.env.example

# Build output
target/
build/
*.class
*.jar
*.war

# IDE
.idea/
*.iml
.vscode/
.project
.classpath
.settings/

# Gradle
.gradle/

# OS
.DS_Store
Thumbs.db
*.log

# Azure
.azure/

# Test coverage
jacoco/
```

DON'T:
```
repo/
├── .env                    # Live credentials committed!
├── target/                 # Build artifacts committed
├── .idea/                  # IDE config committed
```

> **FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed, you MUST verify the file is actually tracked by git:
> 1. **Check .gitignore** -- look in the project directory AND all parent directories for `.gitignore` entries covering `.env`.
> 2. **Run `git ls-files .env`** -- if it returns empty, the file is NOT tracked and is NOT a security issue.
> 3. A `.env` file that exists on disk but is gitignored is working as designed -- developers create it locally from `.env.sample`.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked, or if no `.gitignore` exists at all.

---

## HYG-2: .env.sample (HIGH)
**Pattern:** Provide `.env.sample` with placeholder values. Never commit actual `.env` or any `.env.*` files (except `.env.sample` / `.env.example`).

DO:
```
.env.sample:
  AZURE_STORAGE_ACCOUNT_NAME=your-storage-account
  AZURE_KEYVAULT_URL=https://your-keyvault.vault.azure.net/
  AZURE_COSMOS_ENDPOINT=https://your-cosmos.documents.azure.com:443/
  AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
```

DON'T:
```
.env (committed):
  AZURE_STORAGE_ACCOUNT_NAME=contosoprod
  AZURE_SUBSCRIPTION_ID=12345678-1234-1234-1234-123456789abc  # Real subscription ID
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321       # Real tenant ID
```

---

## HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, functions, and imports. Commented-out code confuses users.

DO:
```java
// Only import what you use
import com.azure.storage.blob.BlobServiceClient;
import com.azure.identity.DefaultAzureCredentialBuilder;
```

DON'T:
```java
// Commented-out code confuses users
// import com.microsoft.azure.storage.CloudStorageAccount;
//
// public class OldImplementation {
//     // This was the old way...
// }

import com.azure.storage.blob.BlobServiceClient;
```

---

## HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

DO:
```
repo/
├── LICENSE              # MIT license (required for Azure Samples org)
├── README.md
├── pom.xml
├── src/
```

DON'T:
```
repo/
├── README.md            # Missing LICENSE file
├── pom.xml
```

> **FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT** -- look for `LICENSE`, `LICENSE.md`, `LICENSE.txt`, or similar at the repository root.
> 2. **Check parent directories** -- in monorepos and sample collections, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.
> 4. Only flag if NO license file exists at ANY level of the repo hierarchy above the sample.

---

## HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference or include governance files: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

DO:
```markdown
# README.md

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA). See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## Security

Microsoft takes security seriously. If you believe you have found a security vulnerability,
please report it as described in [SECURITY.md](SECURITY.md).

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
```
