---
name: "azure-sdk-java-sample-review"
description: "Comprehensive review checklist for Azure SDK Java code samples covering project setup, Azure SDK client patterns, authentication, data services, messaging, AI services, Key Vault, infrastructure, documentation, and sample hygiene."
domain: "code-review"
confidence: "high"
source: "earned -- adapted from TypeScript review skill patterns, generalized for Java Azure SDK ecosystem"
---

## Context

Use this skill when reviewing **Java code samples** for Azure SDKs intended for publication as Microsoft Azure samples. This differs from general Java review—it focuses on Azure SDK-specific concerns:

- **Azure SDK client patterns** (Track 2 `com.azure.*` packages, client builders, pipeline policies)
- **Authentication patterns** (`DefaultAzureCredential`, managed identities, token management)
- **Service-specific best practices** (Cosmos DB, SQL, Storage, Service Bus, Key Vault, AI services)
- **Sample hygiene** (credentials, build artifacts, dependency audit, .gitignore)
- **Documentation accuracy** (README output, troubleshooting, setup instructions)
- **Infrastructure-as-code** (Bicep/Terraform with AVM modules, API versions, parameter validation)
- **azd integration** (azure.yaml structure, hooks, service definitions)
- **Spring Boot / Spring Cloud Azure** integration patterns

This skill captures patterns and anti-patterns discovered during comprehensive reviews of Azure SDK Java samples, plus generalized patterns across the Azure SDK ecosystem.

**Total rules: 71** (8 CRITICAL, 26 HIGH, 33 MEDIUM, 4 LOW)

---

## Severity Legend

- **CRITICAL**: Security vulnerability or sample will not run. Must fix before any publication.
- **HIGH**: Major quality issue that will confuse users or cause production failures. Fix before merge.
- **MEDIUM**: Best practice violation. Should fix before publication for maintainability.
- **LOW**: Polish item, nice-to-have improvement. Address during review cycles.

---

## Quick Pre-Review Checklist (5-Minute Scan)

Use this checklist for rapid initial triage before deep review:

- [ ] **pom.xml / build.gradle**: Uses Track 2 Azure SDK packages (`com.azure:*`, not `com.microsoft.azure:*`)
- [ ] **Authentication**: Uses `DefaultAzureCredential` (not connection strings or hardcoded keys)
- [ ] **.gitignore**: Exists and includes `.env`, `target/`, `build/`, `*.class`, `.idea/`
- [ ] **No secrets**: No hardcoded credentials, API keys, or tokens in code
- [ ] **README.md**: Exists with prerequisites, setup steps, and expected output
- [ ] **LICENSE**: MIT license file present (required for Azure Samples)
- [ ] **Security**: No known critical/high CVEs in dependency tree
- [ ] **Java version**: Uses Java 17 or 21 LTS
- [ ] **Error handling**: `catch` blocks present with proper exception hierarchy
- [ ] **Resource cleanup**: Clients properly closed (try-with-resources or finally blocks)
- [ ] **BOM**: Uses `azure-sdk-bom` for version management
- [ ] **No mixed build tools**: Only Maven OR Gradle (not both)
- [ ] **Imports work**: No broken imports, all dependencies declared
- [ ] **Build succeeds**: `mvn compile` or `gradle build` completes without errors
- [ ] **Sample runs**: `mvn exec:java` or `java -jar` executes without crashes

---

## Blocker Issues (Auto-Reject)

These issues always block publication. Samples with any of these must be rejected immediately:

1. **Hardcoded secrets**—Any production credentials, API keys, connection strings, or tokens in code
2. **Missing authentication**—No auth implementation or uses insecure methods (hardcoded passwords, public keys)
3. **No error handling**—Unchecked exceptions, no try/catch blocks, silent failures
4. **Broken imports**—Missing dependencies, incorrect package names, class not found errors
5. **Security vulnerabilities**—`mvn dependency-check:check` or Dependabot shows critical/high CVEs
6. **Missing LICENSE**—No LICENSE file at ANY level of repo hierarchy (MIT required for Azure Samples org). ⚠️ Check repo root before flagging.
7. **.env file committed**—Live credentials in version control. ⚠️ Verify with `git ls-files .env`—a .env on disk but in .gitignore is NOT committed.
8. **Track 1 packages**—Uses legacy `com.microsoft.azure:*` packages instead of `com.azure:*`

---

## 1. Project Setup & Configuration

**What this section covers:** Build system configuration, Java version, dependency management, environment variables, and compiler settings. These foundational patterns ensure samples build correctly and run reliably across environments.

### PS-1: Java Version (HIGH)
**Pattern:** Use Java 17 or 21 LTS. Configure source/target compatibility in build files.

✅ **DO:**
```xml
<!-- pom.xml -->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

```groovy
// build.gradle
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

❌ **DON'T:**
```xml
<!-- ❌ Java 8 or 11 (not current LTS for new samples) -->
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
</properties>
```

**Why:** Java 17 and 21 are current LTS releases with modern language features (records, sealed classes, pattern matching). New Azure SDK samples should target these versions.

---

### PS-2: Maven/Gradle Project Metadata (MEDIUM)
**Pattern:** All sample projects must include complete metadata for discoverability and maintenance.

✅ **DO:**
```xml
<!-- pom.xml -->
<project>
    <groupId>com.azure.samples</groupId>
    <artifactId>azure-storage-blob-quickstart</artifactId>
    <version>1.0.0</version>
    <name>Azure Storage Blob Quickstart</name>
    <description>Upload and download blobs using Azure Blob Storage SDK for Java</description>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>https://opensource.org/licenses/MIT</url>
        </license>
    </licenses>

    <developers>
        <developer>
            <organization>Microsoft Corporation</organization>
        </developer>
    </developers>
</project>
```

❌ **DON'T:**
```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-sample</artifactId>
    <version>1.0.0</version>
    <!-- ❌ Missing name, description, license, developer info -->
</project>
```

**Note:** The `name` and `description` fields make samples discoverable. Include license metadata.

---

### PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`com.azure:*`).

✅ **DO:**
```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>       <!-- ✅ Track 2, used in BlobService.java -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>            <!-- ✅ Used for auth -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-security-keyvault-secrets</artifactId> <!-- ✅ Used in SecretsManager.java -->
    </dependency>
</dependencies>
```

❌ **DON'T:**
```xml
<dependencies>
    <dependency>
        <groupId>com.microsoft.azure</groupId>
        <artifactId>azure-storage</artifactId>             <!-- ❌ Track 1 (legacy) -->
        <version>8.6.6</version>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-messaging-servicebus</artifactId> <!-- ❌ Listed but never imported -->
    </dependency>
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>                       <!-- ❌ Not imported anywhere -->
    </dependency>
</dependencies>
```

**Why:** Phantom dependencies bloat the project and confuse users trying to learn which packages are needed.

---

### PS-4: Azure SDK Package Naming (HIGH)
**Pattern:** Use Track 2 packages (`com.azure:*`) not Track 1 legacy packages (`com.microsoft.azure:*`).

✅ **DO (Track 2):**
```java
// ✅ Current generation Azure SDK packages
import com.azure.storage.blob.BlobServiceClient;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.messaging.servicebus.ServiceBusClientBuilder;
import com.azure.cosmos.CosmosClient;
import com.azure.data.tables.TableClient;
import com.azure.identity.DefaultAzureCredentialBuilder;

// ✅ Azure OpenAI via com.azure:azure-ai-openai
import com.azure.ai.openai.OpenAIClient;
```

❌ **DON'T (Track 1 Legacy):**
```java
// ❌ Track 1 packages (legacy, avoid in new samples)
import com.microsoft.azure.storage.CloudStorageAccount;         // Use com.azure:azure-storage-blob
import com.microsoft.azure.keyvault.KeyVaultClient;             // Use com.azure:azure-security-keyvault-secrets
import com.microsoft.azure.servicebus.QueueClient;              // Use com.azure:azure-messaging-servicebus
```

**Why:** Track 2 SDKs (`com.azure:*`) are current generation with consistent APIs, built-in HTTP pipeline, and active maintenance. Track 1 (`com.microsoft.azure:*`) is legacy.

---

### PS-5: Configuration (MEDIUM)
**Pattern:** Use `application.properties` or `application.yml` for Spring Boot, or environment variables with validation for standalone apps. Never hardcode configuration.

✅ **DO (Standalone):**
```java
// src/main/java/com/azure/samples/Config.java
public record Config(
    String storageAccountName,
    String keyVaultUrl,
    String cosmosEndpoint
) {
    // Required environment variable names
    private static final List<String> REQUIRED_VARS = List.of(
        "AZURE_STORAGE_ACCOUNT_NAME",
        "AZURE_KEYVAULT_URL",
        "AZURE_COSMOS_ENDPOINT"
    );

    public static Config fromEnvironment() {
        // ✅ Use HashMap—Map.of() throws NullPointerException on null values
        Map<String, String> envValues = new HashMap<>();
        for (String key : REQUIRED_VARS) {
            envValues.put(key, System.getenv(key));
        }

        List<String> missing = envValues.entrySet().stream()
            .filter(e -> e.getValue() == null || e.getValue().isBlank())
            .map(Map.Entry::getKey)
            .toList();

        if (!missing.isEmpty()) {
            throw new IllegalStateException(
                "Missing required environment variables: " + String.join(", ", missing) + "\n"
                + "Set them in your shell or use application.properties / application.yml.\n"
                + "See README.md for required configuration."
            );
        }

        return new Config(
            envValues.get("AZURE_STORAGE_ACCOUNT_NAME"),
            envValues.get("AZURE_KEYVAULT_URL"),
            envValues.get("AZURE_COSMOS_ENDPOINT")
        );
    }
}
```

> **⚠️ Java Map.of() and null values:** `Map.of("key", null)` throws `NullPointerException` at map creation time. Since `System.getenv()` returns `null` for unset variables, always use `HashMap` when values may be null.
>
> **⚠️ .env files are not native to Java.** Unlike Node.js, Java does not natively load `.env` files. Recommend `System.getenv()` for standalone apps or `application.properties` / `application.yml` for Spring Boot. If a sample uses `.env` files, note that the `io.github.cdimascio:java-dotenv` library is required.

✅ **DO (Spring Boot):**
```yaml
# application.yml
azure:
  storage:
    account-name: ${AZURE_STORAGE_ACCOUNT_NAME}
  keyvault:
    url: ${AZURE_KEYVAULT_URL}
  cosmos:
    endpoint: ${AZURE_COSMOS_ENDPOINT}
```

❌ **DON'T:**
```java
// ❌ Don't hardcode configuration
String storageAccountName = "contosoprod";

// ❌ Don't silently fall back to defaults
String storageAccount = Optional.ofNullable(System.getenv("AZURE_STORAGE_ACCOUNT_NAME"))
    .orElse("devstoreaccount1");
```

---

### PS-6: Compiler Warnings (MEDIUM)
**Pattern:** Enable comprehensive compiler warnings. Address all warnings before publication.

✅ **DO:**
```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <compilerArgs>
            <arg>-Xlint:all</arg>
            <arg>-Xlint:-processing</arg>
        </compilerArgs>
        <showWarnings>true</showWarnings>
    </configuration>
</plugin>
```

❌ **DON'T:**
```java
// ❌ Suppress warnings without justification
@SuppressWarnings("unchecked")
public void processData(Object data) {
    List<String> items = (List<String>) data; // ❌ No explanation for suppression
}
```

**Why:** `-Xlint:all` catches deprecation warnings, unchecked casts, and other common bugs at compile time.

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

[*.java]
indent_style = space
indent_size = 4

[*.{xml,yml,yaml,json}]
indent_style = space
indent_size = 2

[*.md]
trim_trailing_whitespace = false
```

---

### PS-8: Lock File / Dependency Tree (HIGH)
**Pattern:** Use Maven Wrapper or Gradle Wrapper for reproducible builds. Commit wrapper files.

✅ **DO:**
```
repo/
├── mvnw                    # ✅ Maven Wrapper script (Unix)
├── mvnw.cmd                # ✅ Maven Wrapper script (Windows)
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties  # ✅ Wrapper config
├── pom.xml
```

```bash
# Generate Maven Wrapper
mvn wrapper:wrapper -Dmaven=3.9.9

# Verify dependency tree
mvn dependency:tree
```

❌ **DON'T:**
```
repo/
├── pom.xml                 # ❌ No wrapper, depends on user's Maven installation
```

```
repo/
├── pom.xml                 # ❌ Mixed build tools
├── build.gradle            # ❌ Don't mix Maven and Gradle
```

**Why:** Wrapper scripts ensure every developer and CI system uses the same build tool version. Mixed build tools cause confusion and version conflicts.

---

### PS-9: CVE Scanning (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. Dependency checks must pass with no critical/high issues.

✅ **DO:**
```xml
<!-- pom.xml - OWASP dependency-check plugin -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.4</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
    </configuration>
</plugin>
```

```bash
# Before submitting sample
mvn dependency-check:check

# Or use Dependabot / GitHub Security Advisories
```

❌ **DON'T:**
```bash
# ❌ Don't ignore vulnerability warnings
mvn dependency-check:check
# Found 3 high severity vulnerabilities
# ❌ Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

### PS-10: Package Legitimacy Check (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `com.azure` group. Watch for typosquatting.

✅ **DO:**
```xml
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>                       <!-- ✅ Official group -->
        <artifactId>azure-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>                       <!-- ✅ Official group -->
        <artifactId>azure-identity</artifactId>
    </dependency>
</dependencies>
```

❌ **DON'T:**
```xml
<dependencies>
    <dependency>
        <groupId>com.azurre</groupId>                      <!-- ❌ Typosquatting (azurre) -->
        <artifactId>azure-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>io.azure</groupId>                        <!-- ❌ Not official group -->
        <artifactId>storage-blob</artifactId>
    </dependency>
</dependencies>
```

**Check:** All Azure SDK packages should use groupId `com.azure` and be published by Microsoft. Verify on [Maven Central](https://central.sonatype.com/namespace/com.azure).

---

### PS-11: BOM Version Management (MEDIUM)
**Pattern:** Use `azure-sdk-bom` to manage Azure SDK dependency versions centrally. Avoid specifying versions on individual Azure SDK dependencies.

✅ **DO:**
```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-sdk-bom</artifactId>
            <version>1.2.29</version>                      <!-- ✅ Single version to manage -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>        <!-- ✅ No version needed -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>            <!-- ✅ No version needed -->
    </dependency>
</dependencies>
```

```groovy
// build.gradle
dependencies {
    implementation platform('com.azure:azure-sdk-bom:1.2.29')
    implementation 'com.azure:azure-storage-blob'          // ✅ No version needed
    implementation 'com.azure:azure-identity'              // ✅ No version needed
}
```

❌ **DON'T:**
```xml
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>
        <version>12.28.1</version>                         <!-- ❌ Manual version, may conflict -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>
        <version>1.14.2</version>                          <!-- ❌ Manual version, may conflict -->
    </dependency>
</dependencies>
```

**Why:** The BOM ensures all Azure SDK packages use compatible versions. Manual version pinning risks incompatible transitive dependencies.

---

## 2. Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction with builders, retry policies, and managed identity patterns. These are foundational patterns that apply across ALL Azure SDK packages.

### AZ-1: Client Builder with DefaultAzureCredential (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with the builder pattern. Cache credential instances.

✅ **DO:**
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.identity.DefaultAzureCredential;
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.messaging.servicebus.ServiceBusClientBuilder;
import com.azure.messaging.servicebus.ServiceBusSenderClient;
import com.azure.cosmos.CosmosClient;
import com.azure.cosmos.CosmosClientBuilder;

// ✅ Cache credential instance
DefaultAzureCredential credential = new DefaultAzureCredentialBuilder().build();

// ✅ Storage Blob
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();

// ✅ Key Vault
SecretClient secretClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

// ✅ Service Bus
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();

// ✅ Cosmos DB
CosmosClient cosmosClient = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildClient();
```

❌ **DON'T:**
```java
// ❌ Don't use connection strings in samples (prefer AAD auth)
BlobServiceClient client = new BlobServiceClientBuilder()
    .connectionString(connectionString)
    .buildClient();

// ❌ Don't use account keys
BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(new StorageSharedKeyCredential(accountName, accountKey))
    .buildClient();

// ❌ Don't recreate credential for each client
BlobServiceClient blobClient = new BlobServiceClientBuilder()
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
SecretClient secretClient = new SecretClientBuilder()
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, IntelliJ, VS Code) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

### AZ-2: Client Options—HttpPipelinePolicy, RetryOptions (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and logging for production-ready samples. The Azure SDK for Java has built-in retry (via `HttpPipelinePolicy`) with sensible defaults—quickstarts MAY rely on defaults; production samples SHOULD configure explicitly.

✅ **DO:**
```java
import com.azure.core.http.policy.RetryOptions;
import com.azure.core.http.policy.ExponentialBackoffOptions;
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;

// ✅ Production samples: configure retry explicitly
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .retryOptions(new RetryOptions(
        new ExponentialBackoffOptions()
            .setMaxRetries(3)
            .setBaseDelay(java.time.Duration.ofSeconds(1))
            .setMaxDelay(java.time.Duration.ofSeconds(30))
    ))
    .buildClient();

// ✅ Quickstarts: SDK defaults are acceptable (3 retries, exponential backoff)
BlobServiceClient quickstartClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();
// Built-in retry policy handles transient failures automatically.
```

❌ **DON'T:**
```java
// ❌ Don't disable retry for production samples
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .retryOptions(new RetryOptions(new ExponentialBackoffOptions().setMaxRetries(0)))
    .buildClient();
// Disabling retry causes failures on transient network errors
```

> **Note:** Azure SDK Java clients include built-in retry with exponential backoff by default. Quickstarts and simple samples do not need explicit retry configuration. Production-grade samples should document retry settings.

---

### AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

✅ **DO:**
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.identity.ManagedIdentityCredentialBuilder;

// ✅ For samples: DefaultAzureCredential (works locally + cloud)
var credential = new DefaultAzureCredentialBuilder().build();

// ✅ For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
var credential = new ManagedIdentityCredentialBuilder().build();

// ✅ User-assigned (when multiple identities needed)
var credential = new ManagedIdentityCredentialBuilder()
    .clientId(System.getenv("AZURE_CLIENT_ID"))
    .build();

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
// > automatically use the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App / Spring App has a managed identity
// > assigned with appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

❌ **DON'T:**
```java
// ❌ Don't hardcode service principal credentials in samples
var credential = new ClientSecretCredentialBuilder()
    .tenantId(tenantId)
    .clientId(clientId)
    .clientSecret(clientSecret)
    .build();
```

**When to use:**
- **System-assigned**: Default choice for single-identity scenarios. Identity lifecycle tied to resource.
- **User-assigned**: Multiple identities per resource, or identity shared across resources, or identity needs to outlive the resource.

---

### AZ-4: Token Management for Non-SDK HTTP (CRITICAL)
**Pattern:** For services without dedicated SDK client support (custom APIs, direct REST calls), get tokens with `getToken()`. The Azure SDK's `TokenCredential` implementations handle token caching and refresh internally—use `TokenRequestContext` and let the SDK manage expiration.

✅ **DO:**
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenCredential;
import com.azure.core.credential.TokenRequestContext;
import com.azure.core.credential.AccessToken;

TokenCredential credential = new DefaultAzureCredentialBuilder().build();

// ✅ TokenRequestContext for scoped token acquisition
TokenRequestContext tokenContext = new TokenRequestContext()
    .addScopes("https://database.windows.net/.default");

// ✅ The SDK caches tokens and refreshes automatically before expiry
AccessToken tokenResponse = credential.getToken(tokenContext).block();

// ✅ For long-running operations, re-call getToken()—the SDK returns
// a cached token if still valid, or refreshes transparently
String token = credential.getToken(tokenContext).block().getToken();

// Use token in JDBC connection...
SQLServerDataSource dataSource = new SQLServerDataSource();
dataSource.setAccessToken(token);
```

✅ **DO (When SDK client is NOT available—direct REST calls):**
```java
// ✅ For direct HTTP calls to Azure APIs, wrap token acquisition in a helper
// that re-calls getToken() before each request (SDK handles caching internally)
private String getAccessToken(TokenCredential credential, String scope) {
    return credential.getToken(
        new TokenRequestContext().addScopes(scope)
    ).block().getToken();
}

// Before each HTTP request in a long-running loop:
for (DataBatch batch : batches) {
    String token = getAccessToken(credential, "https://database.windows.net/.default");
    // SDK returns cached token if valid, refreshes if expiring soon
    httpClient.sendRequest(buildRequest(batch, token));
}
```

❌ **DON'T:**
```java
// ❌ CRITICAL: Don't acquire token once and use for hours
AccessToken token = credential.getToken(
    new TokenRequestContext().addScopes("https://database.windows.net/.default")
).block();
String tokenValue = token.getToken();
// ... hours of processing with same tokenValue string (WILL EXPIRE after ~1 hour)

// ❌ Don't manually track expiration with custom methods
// The SDK's TokenCredential handles caching and refresh internally
boolean isTokenExpiringSoon(OffsetDateTime expiresAt) {  // ❌ Unnecessary
    return OffsetDateTime.now().plusMinutes(5).isAfter(expiresAt);
}
```

**Why:** Azure SDK's `TokenCredential.getToken()` handles caching and proactive refresh internally. Re-call `getToken()` before each use in long-running operations—the SDK returns the cached token if still valid. Don't extract the token string once and reuse it for hours.

---

### AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI.

✅ **DO:**
```java
import com.azure.identity.DefaultAzureCredentialBuilder;

// ✅ For CI/CD environments (no interactive prompts)
var credential = new DefaultAzureCredentialBuilder()
    .excludeInteractiveBrowserCredential()
    .build();

// ✅ For local development (include IntelliJ/VS Code auth)
var credential = new DefaultAzureCredentialBuilder()
    .excludeSharedTokenCacheCredential()
    .build();

// ✅ Document the credential chain in README
// > **Authentication:** This sample uses `DefaultAzureCredential`, which tries:
// > 1. Environment variables (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET)
// > 2. Workload identity (Azure Kubernetes Service)
// > 3. Managed identity (App Service, Functions, Container Apps, Spring Apps)
// > 4. Azure CLI (`az login`)
// > 5. IntelliJ IDEA credential
// > 6. Azure PowerShell
// > 7. Interactive browser (local development only)
```

---

### AZ-6: Resource Cleanup—try-with-resources, AutoCloseable (MEDIUM)
**Pattern:** Samples must properly close clients. Use try-with-resources for `Closeable`/`AutoCloseable` clients.

✅ **DO:**
```java
import com.azure.messaging.servicebus.ServiceBusClientBuilder;
import com.azure.messaging.servicebus.ServiceBusSenderClient;

// ✅ Pattern 1: try-with-resources (preferred)
try (ServiceBusSenderClient sender = new ServiceBusClientBuilder()
        .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
        .credential(credential)
        .sender()
        .queueName("myqueue")
        .buildClient()) {

    sender.sendMessage(new ServiceBusMessage("Hello"));
}  // ✅ sender.close() called automatically

// ✅ Pattern 2: try-with-resources for CosmosClient (implements Closeable)
try (CosmosClient cosmosClient = new CosmosClientBuilder()
        .endpoint(endpoint)
        .credential(credential)
        .buildClient()) {

    CosmosDatabase database = cosmosClient.getDatabase("mydb");
    // ... operations
}  // ✅ cosmosClient.close() called automatically
```

❌ **DON'T:**
```java
// ❌ Don't forget to close clients
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();

sender.sendMessage(new ServiceBusMessage("Hello"));
// ❌ Client never closed (resource leak, connection pool exhaustion)
```

---

### AZ-7: Pagination with PagedIterable/PagedFlux (HIGH)
**Pattern:** Use `PagedIterable` (sync) or `PagedFlux` (async) for paginated Azure SDK responses. Samples that only process the first page silently lose data.

✅ **DO:**
```java
import com.azure.storage.blob.BlobContainerClient;
import com.azure.storage.blob.models.BlobItem;
import com.azure.core.http.rest.PagedIterable;

// ✅ Blob Storage—iterate all pages (sync)
BlobContainerClient containerClient = blobServiceClient.getBlobContainerClient("mycontainer");
PagedIterable<BlobItem> blobs = containerClient.listBlobs();
for (BlobItem blob : blobs) {
    System.out.println("Blob: " + blob.getName());
}

// ✅ Blob Storage—iterate all pages (async with Reactor)
blobServiceAsyncClient.getBlobContainerAsyncClient("mycontainer")
    .listBlobs()
    .doOnNext(blob -> System.out.println("Blob: " + blob.getName()))
    .blockLast();  // Only in samples; production uses subscribe()

// ✅ Cosmos DB—iterate query results
CosmosPagedIterable<JsonNode> items = container.queryItems(
    "SELECT * FROM c WHERE c.category = @category",
    new CosmosQueryRequestOptions(),
    JsonNode.class
);
for (JsonNode item : items) {
    System.out.println("Item: " + item.get("name").asText());
}
```

❌ **DON'T:**
```java
// ❌ CRITICAL BUG: Only gets first page
PagedIterable<BlobItem> blobs = containerClient.listBlobs();
List<BlobItem> firstPage = blobs.streamByPage().findFirst()
    .map(page -> page.getElements().stream().toList())
    .orElse(List.of());
// ❌ Silently ignores all subsequent pages
```

> **Note:** `PagedResponse<T>` provides both `getElements()` (returns `IterableStream<T>`) and `getValue()` (returns `List<T>`, inherited from `Response<List<T>>`). Prefer `getElements()` for consistency. For most use cases, iterate `PagedIterable` directly with `forEach()` or `stream()`—it handles pagination automatically.

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

### AZ-8: Async Client Patterns—Reactor (Mono/Flux) (HIGH)
**Pattern:** Azure SDK for Java provides `*AsyncClient` variants for all services using Project Reactor (`Mono<T>` and `Flux<T>`). Samples should demonstrate async patterns when the use case benefits from non-blocking I/O.

✅ **DO:**
```java
import com.azure.cosmos.CosmosAsyncClient;
import com.azure.cosmos.CosmosAsyncContainer;
import com.azure.cosmos.CosmosClientBuilder;
import com.azure.cosmos.models.PartitionKey;
import com.azure.storage.blob.BlobAsyncClient;
import com.azure.storage.blob.BlobServiceAsyncClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.security.keyvault.secrets.SecretAsyncClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Cosmos DB async client
CosmosAsyncClient cosmosAsyncClient = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildAsyncClient();

CosmosAsyncContainer container = cosmosAsyncClient
    .getDatabase("mydb")
    .getContainer("mycontainer");

// ✅ Async point read with Mono
container.readItem("item-id", new PartitionKey("electronics"), JsonNode.class)
    .subscribe(
        response -> System.out.println("Item: " + response.getItem()),
        error -> System.err.println("Error: " + error.getMessage())
    );

// ✅ Async query with Flux
container.queryItems("SELECT * FROM c WHERE c.category = 'electronics'",
        new CosmosQueryRequestOptions(), JsonNode.class)
    .byPage()
    .flatMap(page -> Flux.fromIterable(page.getElements()))
    .doOnNext(item -> System.out.println("Item: " + item.get("name").asText()))
    .blockLast();  // Only in samples; production uses subscribe()

// ✅ Blob Storage async
BlobServiceAsyncClient blobAsyncClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildAsyncClient();

blobAsyncClient.getBlobContainerAsyncClient("mycontainer")
    .getBlobAsyncClient("sample.txt")
    .downloadContent()
    .subscribe(content -> System.out.println("Content: " + content.toString()));

// ✅ Key Vault async
SecretAsyncClient secretAsyncClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildAsyncClient();

secretAsyncClient.getSecret("my-secret")
    .subscribe(secret -> System.out.println("Secret: " + secret.getValue()));
```

❌ **DON'T:**
```java
// ❌ Don't block on every async call (defeats the purpose)
String value = secretAsyncClient.getSecret("my-secret")
    .block()  // ❌ Blocking—use sync client instead
    .getValue();

// ❌ Don't ignore errors in subscribe
container.readItem("id", partitionKey, JsonNode.class)
    .subscribe(response -> process(response));  // ❌ No error handler
```

> **When to use async:** Use `*AsyncClient` for high-throughput scenarios (batch processing, event-driven), reactive web frameworks (Spring WebFlux), or when composing multiple service calls. Use sync clients for simple samples, CLI tools, and Spring MVC applications.

---

### AZ-9: Logging Configuration—SLF4J (HIGH)
**Pattern:** Azure SDK for Java uses SLF4J for logging. Configure logging to help users troubleshoot issues. Document log configuration in README.

✅ **DO:**
```xml
<!-- pom.xml - Add SLF4J implementation -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>2.0.16</version>
</dependency>

<!-- Or for Spring Boot samples (already included via spring-boot-starter) -->
<!-- Logback is the default Spring Boot logging implementation -->
```

```properties
# src/main/resources/simplelogger.properties (for slf4j-simple)
org.slf4j.simpleLogger.defaultLogLevel=info
# ✅ Enable Azure SDK verbose logging for troubleshooting
org.slf4j.simpleLogger.log.com.azure=info
# ✅ Enable HTTP request/response logging
org.slf4j.simpleLogger.log.com.azure.core.http=debug
```

```xml
<!-- logback.xml (for Logback / Spring Boot) -->
<configuration>
    <logger name="com.azure" level="INFO" />
    <!-- ✅ Verbose Azure SDK logging for debugging -->
    <logger name="com.azure.core.http" level="DEBUG" />
    <!-- ✅ Identity/credential troubleshooting -->
    <logger name="com.azure.identity" level="DEBUG" />
</configuration>
```

```java
// ✅ Enable HTTP logging on client builder
import com.azure.core.http.policy.HttpLogDetailLevel;
import com.azure.core.http.policy.HttpLogOptions;

BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .httpLogOptions(new HttpLogOptions().setLogLevel(HttpLogDetailLevel.BODY_AND_HEADERS))
    .buildClient();
```

❌ **DON'T:**
```java
// ❌ Don't use System.out for logging in production samples
System.out.println("Connecting to storage...");

// ❌ Don't ignore SLF4J "no binding" warnings
// SLF4J: No SLF4J providers were found.
// This means no logging implementation is on the classpath
```

> **README guidance:** Document how to enable verbose logging: *"To troubleshoot Azure SDK errors, set the `com.azure` logger to `DEBUG` in your logging configuration."*

---

### AZ-10: HttpClient Selection (MEDIUM)
**Pattern:** Azure SDK for Java supports multiple HTTP client implementations. Document which is in use and when to switch.

✅ **DO:**
```xml
<!-- pom.xml -->
<!-- Default: Netty (included transitively by most Azure SDK packages) -->
<!-- No additional dependency needed for Netty -->

<!-- Alternative: OkHttp (smaller footprint, good for Android/CLI) -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-core-http-okhttp</artifactId>
</dependency>

<!-- Alternative: JDK HttpClient (Java 11+, no extra dependencies) -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-core-http-jdk-httpclient</artifactId>
</dependency>
```

```java
// ✅ Explicitly set HTTP client (optional—defaults to auto-detected)
import com.azure.core.http.HttpClient;

BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .httpClient(HttpClient.createDefault())  // Uses auto-detected implementation
    .buildClient();
```

> **When to switch:** Use **Netty** (default) for high-throughput server apps. Use **OkHttp** for smaller footprint or Android. Use **JDK HttpClient** to minimize dependencies in Java 11+ projects.

---

## 3. Azure AI Services (OpenAI, Document Intelligence, Speech)

**What this section covers:** AI service client patterns, API versioning, embeddings, chat completions, and document analysis using the official `com.azure:azure-ai-openai` SDK.

### AI-1: Azure OpenAI Client (HIGH)
**Pattern:** Use `com.azure:azure-ai-openai` with `DefaultAzureCredential`. Configure timeouts and retry options.

✅ **DO:**
```java
import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.ChatCompletions;
import com.azure.ai.openai.models.ChatCompletionsOptions;
import com.azure.ai.openai.models.ChatRequestUserMessage;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Build client with AAD credential
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(config.azureOpenAiEndpoint())
    .credential(credential)
    .buildClient();

// ✅ Chat completion
ChatCompletions completions = client.getChatCompletions(
    "gpt-4o",
    new ChatCompletionsOptions(List.of(
        new ChatRequestUserMessage("Hello!")
    ))
);
System.out.println(completions.getChoices().get(0).getMessage().getContent());

// ✅ Embeddings
var embeddingsResult = client.getEmbeddings(
    "text-embedding-3-small",
    new EmbeddingsOptions(List.of("Sample text to embed"))
);
List<Float> embedding = embeddingsResult.getData().get(0).getEmbedding();
```

❌ **DON'T:**
```java
// ❌ Don't use API keys in samples (prefer AAD)
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(apiKey))  // ❌ Use DefaultAzureCredential
    .buildClient();

// ❌ Don't use deprecated constructor patterns
// Check Azure SDK for Java release notes for current API
```

---

### AI-2: API Version Documentation (LOW)
**Pattern:** Hardcoded API versions should include a comment linking to version docs.

✅ **DO:**
```java
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .serviceVersion(OpenAIServiceVersion.V2024_10_21)
    // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
    .buildClient();
```

---

### AI-3: Document Intelligence (MEDIUM)
**Pattern:** Use `com.azure:azure-ai-documentintelligence` with `DefaultAzureCredential`.

✅ **DO:**
```java
import com.azure.ai.documentintelligence.DocumentIntelligenceClient;
import com.azure.ai.documentintelligence.DocumentIntelligenceClientBuilder;
import com.azure.ai.documentintelligence.models.AnalyzeResult;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

DocumentIntelligenceClient client = new DocumentIntelligenceClientBuilder()
    .endpoint(config.documentIntelligenceEndpoint())
    .credential(credential)
    .buildClient();

// ✅ Analyze document
SyncPoller<AnalyzeResultOperation, AnalyzeResult> poller = client.beginAnalyzeDocument(
    "prebuilt-invoice",
    analyzeDocumentRequest
);
AnalyzeResult result = poller.getFinalResult();
```

---

### AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause silent failures or runtime errors.

✅ **DO:**
```java
// ✅ Document expected dimensions
private static final String EMBEDDING_MODEL = "text-embedding-3-small";  // 1536 dimensions
private static final int VECTOR_DIMENSION = 1536;

// Validate embedding size
List<Float> embedding = getEmbedding(text);
if (embedding.size() != VECTOR_DIMENSION) {
    throw new IllegalStateException(
        String.format("Embedding dimension mismatch: expected %d, got %d%n"
            + "Ensure model '%s' matches table schema.",
            VECTOR_DIMENSION, embedding.size(), EMBEDDING_MODEL)
    );
}
```

❌ **DON'T:**
```java
// ❌ Don't assume dimension without validation
List<Float> embedding = getEmbedding(text);
insertEmbedding(embedding);  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

### AI-5: Speech SDK (HIGH)
**Pattern:** Use `com.microsoft.cognitiveservices.speech:client-sdk` for speech-to-text and text-to-speech. This is a separate SDK from the `com.azure:*` packages.

✅ **DO:**
```xml
<!-- pom.xml - Speech SDK (separate from azure-sdk-bom) -->
<dependency>
    <groupId>com.microsoft.cognitiveservices.speech</groupId>
    <artifactId>client-sdk</artifactId>
    <version>1.42.0</version>  <!-- Check https://learn.microsoft.com/azure/ai-services/speech-service/releasenotes for latest -->
</dependency>
```

```java
import com.microsoft.cognitiveservices.speech.*;
import com.microsoft.cognitiveservices.speech.audio.AudioConfig;

// ✅ Speech-to-text with AAD auth (preferred)
// Note: SpeechConfig supports both key and AAD token auth
String token = credential.getToken(
    new TokenRequestContext().addScopes("https://cognitiveservices.azure.com/.default")
).block().getToken();
String region = config.speechRegion();  // e.g., "eastus"

SpeechConfig speechConfig = SpeechConfig.fromAuthorizationToken(token, region);
speechConfig.setSpeechRecognitionLanguage("en-US");

// ✅ Recognize from microphone
try (SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig)) {
    SpeechRecognitionResult result = recognizer.recognizeOnceAsync().get();
    if (result.getReason() == ResultReason.RecognizedSpeech) {
        System.out.println("Recognized: " + result.getText());
    } else if (result.getReason() == ResultReason.NoMatch) {
        System.out.println("No speech recognized.");
    }
}  // ✅ Recognizer closed automatically

// ✅ Text-to-speech
try (SpeechSynthesizer synthesizer = new SpeechSynthesizer(speechConfig)) {
    SpeechSynthesisResult result = synthesizer.SpeakTextAsync("Hello from Azure!").get();
    if (result.getReason() == ResultReason.SynthesizingAudioCompleted) {
        System.out.println("Speech synthesized successfully.");
    }
}

// ✅ Recognize from audio file
AudioConfig audioConfig = AudioConfig.fromWavFileInput("sample.wav");
try (SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig, audioConfig)) {
    SpeechRecognitionResult result = recognizer.recognizeOnceAsync().get();
    System.out.println("Recognized: " + result.getText());
}
```

❌ **DON'T:**
```java
// ❌ Don't use subscription key directly in samples (prefer AAD token)
SpeechConfig config = SpeechConfig.fromSubscription(subscriptionKey, region);

// ❌ Don't forget to close SpeechRecognizer/SpeechSynthesizer (native resources)
SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig);
recognizer.recognizeOnceAsync().get();
// ❌ Native resources leaked
```

> **Note:** The Speech SDK (`com.microsoft.cognitiveservices.speech:client-sdk`) is NOT managed by `azure-sdk-bom`. Pin the version explicitly. The SDK includes native libraries—ensure the correct platform classifier is available at runtime.

---

## 4. Data Services (Cosmos DB, SQL, Storage, Tables)

**What this section covers:** Database and storage client patterns, connection management, transactions, batching, and query parameterization. Includes service-specific best practices for Cosmos DB, Azure SQL (JDBC), and Storage.

### DB-1: Cosmos DB SDK (HIGH)
**Pattern:** Use `com.azure:azure-cosmos` with AAD credentials. Handle partitioned containers properly.

✅ **DO:**
```java
import com.azure.cosmos.CosmosClient;
import com.azure.cosmos.CosmosClientBuilder;
import com.azure.cosmos.CosmosContainer;
import com.azure.cosmos.models.CosmosItemRequestOptions;
import com.azure.cosmos.models.CosmosQueryRequestOptions;
import com.azure.cosmos.models.PartitionKey;
import com.azure.cosmos.models.SqlParameter;
import com.azure.cosmos.models.SqlQuerySpec;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

CosmosClient client = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildClient();

CosmosContainer container = client
    .getDatabase("mydb")
    .getContainer("mycontainer");

// ✅ Query with partition key and parameterized query
SqlQuerySpec query = new SqlQuerySpec(
    "SELECT * FROM c WHERE c.category = @category",
    List.of(new SqlParameter("@category", "electronics"))
);
CosmosPagedIterable<JsonNode> items = container.queryItems(
    query,
    new CosmosQueryRequestOptions().setPartitionKey(new PartitionKey("electronics")),
    JsonNode.class
);

// ✅ Point read (most efficient)
container.readItem("item-id", new PartitionKey("electronics"), JsonNode.class);

// ✅ Create with partition key
container.createItem(Map.of(
    "id", "item-id",
    "category", "electronics",
    "name", "Laptop"
));
```

❌ **DON'T:**
```java
// ❌ Don't use primary key in samples
CosmosClient client = new CosmosClientBuilder()
    .endpoint(endpoint)
    .key(primaryKey)  // ❌ Use credential(credential) with AAD
    .buildClient();

// ❌ Don't omit partition key (cross-partition queries are expensive)
container.queryItems("SELECT * FROM c", new CosmosQueryRequestOptions(), JsonNode.class);
```

---

### DB-2: Azure SQL with JDBC (HIGH)
**Pattern:** Use `mssql-jdbc` with AAD token authentication. Use connection pooling (HikariCP). For simplified passwordless auth, use `azure-identity-extensions` JDBC plugin.

✅ **DO:**
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenRequestContext;
import com.microsoft.sqlserver.jdbc.SQLServerDataSource;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Get AAD token for Azure SQL
String token = credential.getToken(
    new TokenRequestContext().addScopes("https://database.windows.net/.default")
).block().getToken();

// ✅ Connect with AAD token
SQLServerDataSource dataSource = new SQLServerDataSource();
dataSource.setServerName(config.sqlServer());
dataSource.setDatabaseName(config.sqlDatabase());
dataSource.setAccessToken(token);
dataSource.setEncrypt("true");

try (Connection conn = dataSource.getConnection()) {
    // ✅ Parameterized query
    try (PreparedStatement stmt = conn.prepareStatement(
            "SELECT * FROM [Products] WHERE [Category] = ?")) {
        stmt.setString(1, "Electronics");
        try (ResultSet rs = stmt.executeQuery()) {
            while (rs.next()) {
                System.out.println(rs.getString("name"));
            }
        }
    }
}
```

✅ **DO (Simplified passwordless auth with azure-identity-extensions):**
```xml
<!-- pom.xml - JDBC plugin for passwordless Azure SQL auth -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-identity-extensions</artifactId>
    <version>1.2.0</version>
</dependency>
```

```java
// ✅ Connection string with ActiveDirectoryDefault authentication
// No manual token acquisition needed—the JDBC plugin handles it
String url = "jdbc:sqlserver://myserver.database.windows.net:1433;"
    + "database=mydb;"
    + "encrypt=true;"
    + "authentication=ActiveDirectoryDefault;";

try (Connection conn = DriverManager.getConnection(url)) {
    // Token acquisition, caching, and refresh handled automatically
    try (PreparedStatement stmt = conn.prepareStatement("SELECT * FROM [Products]")) {
        // ...
    }
}
```
```

❌ **DON'T:**
```java
// ❌ Don't use SQL authentication with passwords in samples
String url = "jdbc:sqlserver://server.database.windows.net;"
    + "database=mydb;user=admin;password=P@ssw0rd";  // ❌ Hardcoded credentials
Connection conn = DriverManager.getConnection(url);
```

---

### DB-3: SQL Parameter Safety—PreparedStatement (MEDIUM)
**Pattern:** ALL SQL values must use `PreparedStatement` with parameter placeholders. Never concatenate user input into SQL strings.

✅ **DO:**
```java
// ✅ Parameterized query with PreparedStatement
try (PreparedStatement stmt = conn.prepareStatement(
        "SELECT [id], [name] FROM [Products] WHERE [category] = ? AND [price] < ?")) {
    stmt.setString(1, category);
    stmt.setDouble(2, maxPrice);
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            System.out.printf("Product: %s ($%.2f)%n", rs.getString("name"), rs.getDouble("price"));
        }
    }
}
```

❌ **DON'T:**
```java
// ❌ CRITICAL: SQL injection vulnerability
String query = "SELECT * FROM Products WHERE category = '" + userInput + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);  // ❌ SQL injection risk
```

**Why:** String concatenation in SQL enables injection attacks. `PreparedStatement` sanitizes all inputs automatically.

---

### DB-4: Batch Operations (HIGH)
**Pattern:** Avoid row-by-row operations. Use batch operations for multiple rows. Document batch size rationale.

✅ **DO (SQL—Batch Insert):**
```java
// ✅ Batch insert with PreparedStatement
private static final int BATCH_SIZE = 100;

try (PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO [Products] ([id], [name], [category]) VALUES (?, ?, ?)")) {

    for (int i = 0; i < items.size(); i++) {
        Product item = items.get(i);
        stmt.setInt(1, item.id());
        stmt.setString(2, item.name());
        stmt.setString(3, item.category());
        stmt.addBatch();

        if ((i + 1) % BATCH_SIZE == 0) {
            stmt.executeBatch();  // ✅ Execute every BATCH_SIZE rows
        }
    }
    stmt.executeBatch();  // ✅ Execute remaining rows
}
// Why batch size 100: Balances network round trips vs memory usage.
```

✅ **DO (Cosmos DB—Bulk):**
```java
// ✅ Cosmos DB bulk operations
CosmosContainer container = cosmosClient.getDatabase("mydb").getContainer("mycontainer");

List<CosmosItemOperation> operations = items.stream()
    .map(item -> CosmosBulkOperations.getCreateItemOperation(
        item, new PartitionKey(item.getCategory())))
    .toList();

container.executeBulkOperations(operations);
// Why: Bulk executor optimizes throughput automatically (batching + parallelism).
```

❌ **DON'T:**
```java
// ❌ Row-by-row INSERT (50 round trips for 50 items)
for (Product item : items) {
    try (PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO [Products] VALUES (?, ?, ?)")) {
        stmt.setInt(1, item.id());
        stmt.setString(2, item.name());
        stmt.setString(3, item.category());
        stmt.executeUpdate();  // ❌ One round trip per row
    }
}
```

---

### DB-5: Azure Storage—Blob (MEDIUM)
**Pattern:** Use `com.azure:azure-storage-blob` with `DefaultAzureCredential`.

✅ **DO:**
```java
import com.azure.storage.blob.BlobClient;
import com.azure.storage.blob.BlobContainerClient;
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();

BlobContainerClient containerClient = blobServiceClient.getBlobContainerClient("mycontainer");
containerClient.createIfNotExists();

// ✅ Upload
BlobClient blobClient = containerClient.getBlobClient("sample.txt");
blobClient.upload(BinaryData.fromString("Hello, Azure!"), true);

// ✅ Download
BinaryData content = blobClient.downloadContent();
System.out.println("Content: " + content.toString());

// ✅ List blobs
containerClient.listBlobs().forEach(blob ->
    System.out.println("Blob: " + blob.getName())
);
```

---

### DB-6: SAS Token Fallback (MEDIUM)
**Pattern:** For local development where `DefaultAzureCredential` isn't available, provide SAS token fallback with clear documentation.

✅ **DO:**
```java
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

// ✅ Try AAD first, fall back to SAS for local dev
BlobServiceClient blobServiceClient;

String sasToken = System.getenv("AZURE_STORAGE_SAS_TOKEN");
if (sasToken != null && !sasToken.isBlank()) {
    blobServiceClient = new BlobServiceClientBuilder()
        .endpoint("https://" + accountName + ".blob.core.windows.net")
        .sasToken(sasToken)
        .buildClient();
    System.out.println("Using SAS token authentication (local dev)");
} else {
    var credential = new DefaultAzureCredentialBuilder().build();
    blobServiceClient = new BlobServiceClientBuilder()
        .endpoint("https://" + accountName + ".blob.core.windows.net")
        .credential(credential)
        .buildClient();
    System.out.println("Using DefaultAzureCredential (AAD)");
}
```

---

## 5. Messaging Services (Service Bus, Event Hubs, Event Grid)

**What this section covers:** Messaging patterns for queues, topics, event ingestion, event-driven architectures, and event routing. Focus on reliable message handling, checkpoint management, and proper resource cleanup.

### MSG-1: Service Bus Patterns (HIGH)
**Pattern:** Use `com.azure:azure-messaging-servicebus` with `DefaultAzureCredential`. Complete or abandon messages.

✅ **DO:**
```java
import com.azure.messaging.servicebus.*;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Send messages
try (ServiceBusSenderClient sender = new ServiceBusClientBuilder()
        .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
        .credential(credential)
        .sender()
        .queueName("myqueue")
        .buildClient()) {

    sender.sendMessage(new ServiceBusMessage("{\"orderId\": 1, \"amount\": 100}"));
}

// ✅ Receive messages with processor (recommended for production)
ServiceBusProcessorClient processor = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .processor()
    .queueName("myqueue")
    .processMessage(context -> {
        System.out.println("Received: " + context.getMessage().getBody().toString());
        // ✅ Auto-completed by processor in PEEK_LOCK mode
    })
    .processError(context -> {
        System.err.println("Error: " + context.getException().getMessage());
    })
    .buildProcessorClient();

processor.start();
// ... wait for processing
processor.stop();
processor.close();
```

❌ **DON'T:**
```java
// ❌ Don't use connection strings in samples
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .connectionString(connectionString)  // ❌ Use credential-based auth
    .sender()
    .queueName("myqueue")
    .buildClient();

// ❌ Don't forget to close clients
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace)
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();
sender.sendMessage(new ServiceBusMessage("Hello"));
// ❌ Client never closed
```

---

### MSG-2: Event Hubs Patterns (MEDIUM)
**Pattern:** Use `com.azure:azure-messaging-eventhubs` for ingestion, `com.azure:azure-messaging-eventhubs-checkpointstore-blob` for processing with checkpoint management.

✅ **DO:**
```java
import com.azure.messaging.eventhubs.*;
import com.azure.messaging.eventhubs.checkpointstore.blob.BlobCheckpointStore;
import com.azure.storage.blob.BlobContainerAsyncClient;
import com.azure.storage.blob.BlobContainerClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Send events
try (EventHubProducerClient producer = new EventHubClientBuilder()
        .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
        .eventHubName("myeventhub")
        .credential(credential)
        .buildProducerClient()) {

    EventDataBatch batch = producer.createBatch();
    batch.tryAdd(new EventData("{\"temperature\": 23.5}"));
    batch.tryAdd(new EventData("{\"temperature\": 24.1}"));
    producer.send(batch);
}

// ✅ Receive events with checkpoint store
BlobContainerAsyncClient blobClient = new BlobContainerClientBuilder()
    .endpoint("https://" + storageAccount + ".blob.core.windows.net")
    .containerName("eventhub-checkpoints")
    .credential(credential)
    .buildAsyncClient();

EventProcessorClient processor = new EventProcessorClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .eventHubName("myeventhub")
    .consumerGroup("$Default")
    .credential(credential)
    .checkpointStore(new BlobCheckpointStore(blobClient))
    .processEvent(context -> {
        System.out.println("Received: " + context.getEventData().getBodyAsString());
        context.updateCheckpoint();  // ✅ Checkpoint after processing
    })
    .processError(context -> {
        System.err.println("Error: " + context.getThrowable().getMessage());
    })
    .buildEventProcessorClient();

processor.start();
```

---

## 6. Key Vault and Secrets Management

**What this section covers:** Secure secrets storage and retrieval using Azure Key Vault. Covers secrets, keys, and certificates with AAD authentication.

### KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `com.azure:azure-security-keyvault-secrets`, `com.azure:azure-security-keyvault-keys`, `com.azure:azure-security-keyvault-certificates` with `DefaultAzureCredential`.

✅ **DO:**
```java
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.security.keyvault.keys.KeyClient;
import com.azure.security.keyvault.keys.KeyClientBuilder;
import com.azure.security.keyvault.certificates.CertificateClient;
import com.azure.security.keyvault.certificates.CertificateClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// ✅ Secrets
SecretClient secretClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

secretClient.setSecret("db-password", "P@ssw0rd123");
String secretValue = secretClient.getSecret("db-password").getValue();
System.out.println("Secret value: " + secretValue);

// ✅ Keys (for encryption)
KeyClient keyClient = new KeyClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

var key = keyClient.createRsaKey(new CreateRsaKeyOptions("my-encryption-key"));
System.out.println("Key ID: " + key.getId());

// ✅ Certificates
CertificateClient certClient = new CertificateClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

var cert = certClient.getCertificate("my-cert");
System.out.println("Certificate name: " + cert.getName());
```

❌ **DON'T:**
```java
// ❌ Don't hardcode secrets in samples
String dbPassword = "P@ssw0rd123";  // ❌ Use Key Vault

// ❌ Don't use Key Vault for every sample (adds complexity)
// Use Key Vault when demonstrating secret management or when the sample
// explicitly covers secure configuration patterns.
```

---

## 7. Vector Search Patterns (Azure SQL, Cosmos DB, AI Search)

**What this section covers:** Vector similarity search implementations across Azure data services. Includes embedding storage, distance calculations, and approximate nearest neighbor search.

### VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Serialize vectors as JSON strings for Azure SQL VECTOR type. Use appropriate parameter handling for each service.

✅ **DO (Azure SQL):**
```java
// ✅ Insert vector as JSON string with CAST
String insertSql = "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(? AS VECTOR(1536)))";
try (PreparedStatement stmt = conn.prepareStatement(insertSql)) {
    // Serialize float array as JSON string
    String embeddingJson = objectMapper.writeValueAsString(embedding);
    stmt.setString(1, embeddingJson);
    stmt.executeUpdate();
}

// ✅ Vector distance query
String searchSql = """
    SELECT TOP (?)
        [id], [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC
    """;
try (PreparedStatement stmt = conn.prepareStatement(searchSql)) {
    stmt.setInt(1, k);
    stmt.setString(2, objectMapper.writeValueAsString(searchEmbedding));
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            System.out.printf("%s (distance: %.4f)%n", rs.getString("name"), rs.getFloat("distance"));
        }
    }
}
```

✅ **DO (Cosmos DB Vector Search):**
```java
// ✅ Cosmos DB vector search
SqlQuerySpec query = new SqlQuerySpec(
    "SELECT TOP @k c.id, c.name, VectorDistance(c.embedding, @searchEmbedding) AS similarity "
    + "FROM c ORDER BY VectorDistance(c.embedding, @searchEmbedding)",
    List.of(
        new SqlParameter("@k", 5),
        new SqlParameter("@searchEmbedding", searchEmbedding)
    )
);

container.queryItems(query, new CosmosQueryRequestOptions(), JsonNode.class)
    .forEach(item -> System.out.printf("%s (similarity: %.4f)%n",
        item.get("name").asText(), item.get("similarity").asDouble()));
```

---

### VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires ≥1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

✅ **DO:**
```java
// Check row count before creating DiskANN index
int rowCount;
try (Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM [" + tableName + "]")) {
    rs.next();
    rowCount = rs.getInt(1);
}

if (rowCount >= 1000) {
    System.out.printf("✅ %d rows available. Creating DiskANN index...%n", rowCount);
    try (Statement stmt = conn.createStatement()) {
        stmt.execute("CREATE INDEX [ix_" + tableName + "_embedding_diskann] "
            + "ON [" + tableName + "] ([embedding]) USING DiskANN");
    }
} else {
    System.out.printf("⚠️ Only %d rows. DiskANN requires ≥1000. Using exact search.%n", rowCount);
    // Fall back to VECTOR_DISTANCE (exact search)
}
```

❌ **DON'T:**
```java
// ❌ Create DiskANN index without checking row count
stmt.execute("CREATE INDEX ... USING DiskANN");
// Fails with: "DiskANN index requires at least 1000 rows"
```

---

## 8. Error Handling

**What this section covers:** Java exception hierarchy, contextual error messages, and troubleshooting guidance. Proper error handling prevents silent failures and helps users diagnose issues.

### ERR-1: Azure SDK Exception Hierarchy (MEDIUM)
**Pattern:** Catch specific Azure SDK exceptions (`HttpResponseException`, `ResourceNotFoundException`) before general `Exception`. Log status codes and error messages.

✅ **DO:**
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

❌ **DON'T:**
```java
try {
    secretClient.getSecret("my-secret");
} catch (Exception e) {  // ❌ Catches everything, loses specific error info
    e.printStackTrace();  // ❌ Stack trace dump is not user-friendly
}
```

---

### ERR-2: Contextual Error Messages (MEDIUM)
**Pattern:** Provide actionable error messages with troubleshooting hints for common Azure errors.

✅ **DO:**
```java
try {
    var credential = new DefaultAzureCredentialBuilder().build();
    credential.getToken(new TokenRequestContext()
        .addScopes("https://storage.azure.com/.default")).block();
} catch (Exception e) {
    System.err.println("❌ Failed to acquire Azure Storage token: " + e.getMessage());
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

## 9. Data Management

**What this section covers:** Sample data handling, pre-computed files, JSON loading, and data validation. Ensures samples run reliably on first execution without requiring data generation.

### DATA-1: Pre-Computed Data Files (HIGH)
**Pattern:** Commit all required data files to repo. Pre-computed embeddings avoid requiring Azure OpenAI API calls on first run.

✅ **DO:**
```
repo/
├── src/main/resources/
│   ├── data/
│   │   ├── products.json              # ✅ Sample data
│   │   ├── products-with-vectors.json # ✅ Pre-computed embeddings
├── src/main/java/
│   ├── App.java                       # Loads products-with-vectors.json from resources
│   ├── GenerateEmbeddings.java        # Generates embeddings (optional)
```

❌ **DON'T:**
```
repo/
├── src/main/resources/data/
│   ├── products.json              # ✅ Raw data
│   ├── .gitignore                 # ❌ products-with-vectors.json gitignored
├── src/main/java/
│   ├── App.java                   # ❌ Fails: Resource not found
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a data file as missing:
> 1. **Check the FULL PR file list**—not just the immediate project directory. Data files often live in sibling directories, parent directories, or shared `data/` folders.
> 2. **Trace the file path in code** relative to the classpath. Check for `getResourceAsStream()`, `ClassLoader.getResource()`, or `Path.of()` calls.
> 3. **Check for monorepo patterns**—in sample collections, data may be shared across multiple samples via a common parent directory.
> 4. Only flag as missing if the file truly does not exist anywhere in the PR or repo at the path the code resolves to at runtime.

---

### DATA-2: Resource File Loading (MEDIUM)
**Pattern:** Use classpath resource loading for bundled data files.

✅ **DO:**
```java
import java.io.InputStream;
import com.fasterxml.jackson.databind.ObjectMapper;

// ✅ Load from classpath (works in JAR and IDE)
try (InputStream is = App.class.getResourceAsStream("/data/products.json")) {
    if (is == null) {
        throw new IllegalStateException("Resource not found: /data/products.json");
    }
    List<Product> products = objectMapper.readValue(is, new TypeReference<>() {});
}
```

❌ **DON'T:**
```java
// ❌ Hardcoded file path (breaks when running from different directories)
Path path = Path.of("src/main/resources/data/products.json");
List<Product> products = objectMapper.readValue(path.toFile(), new TypeReference<>() {});
```

---

## 10. Sample Hygiene

**What this section covers:** Repository hygiene, security, and governance. Covers .gitignore patterns, environment file protection, license files, and repository governance.

### HYG-1: .gitignore (CRITICAL)
**Pattern:** Always protect sensitive files, build artifacts, and dependencies with comprehensive `.gitignore`.

✅ **DO:**
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

❌ **DON'T:**
```
repo/
├── .env                    # ❌ Live credentials committed!
├── target/                 # ❌ Build artifacts committed
├── .idea/                  # ❌ IDE config committed
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `.env` or credential files as committed, you MUST verify the file is actually tracked by git:
> 1. **Check .gitignore**—look in the project directory AND all parent directories for `.gitignore` entries covering `.env`.
> 2. **Run `git ls-files .env`**—if it returns empty, the file is NOT tracked and is NOT a security issue.
> 3. A `.env` file that exists on disk but is gitignored is working as designed—developers create it locally from `.env.sample`.
> 4. Only flag as CRITICAL if `git ls-files` confirms the file IS tracked, or if no `.gitignore` exists at all.

---

### HYG-2: .env.sample (HIGH)
**Pattern:** Provide `.env.sample` with placeholder values. Never commit actual `.env` or any `.env.*` files (except `.env.sample` / `.env.example`).

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
  AZURE_TENANT_ID=87654321-4321-4321-4321-cba987654321       # ❌ Real tenant ID
```

---

### HYG-3: Dead Code (HIGH)
**Pattern:** Remove unused files, functions, and imports. Commented-out code confuses users.

✅ **DO:**
```java
// Only import what you use
import com.azure.storage.blob.BlobServiceClient;
import com.azure.identity.DefaultAzureCredentialBuilder;
```

❌ **DON'T:**
```java
// ❌ Commented-out code confuses users
// import com.microsoft.azure.storage.CloudStorageAccount;
//
// public class OldImplementation {
//     // This was the old way...
// }

import com.azure.storage.blob.BlobServiceClient;
```

---

### HYG-4: LICENSE File (HIGH)
**Pattern:** All Azure Samples repositories must include MIT LICENSE file.

✅ **DO:**
```
repo/
├── LICENSE              # ✅ MIT license (required for Azure Samples org)
├── README.md
├── pom.xml
├── src/
```

❌ **DON'T:**
```
repo/
├── README.md            # ❌ Missing LICENSE file
├── pom.xml
```

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging a missing LICENSE:
> 1. **Check the REPO ROOT**—look for `LICENSE`, `LICENSE.md`, `LICENSE.txt`, or similar at the repository root.
> 2. **Check parent directories**—in monorepos and sample collections, a single license at the repo root covers all subdirectories.
> 3. Per-sample LICENSE files are NOT required when the repo root already has one.
> 4. Only flag if NO license file exists at ANY level of the repo hierarchy above the sample.

---

### HYG-5: Repository Governance Files (MEDIUM)
**Pattern:** Samples in Azure Samples org should reference or include governance files: CONTRIBUTING.md, CODEOWNERS, SECURITY.md.

✅ **DO:**
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

---

## 11. README & Documentation

**What this section covers:** Documentation quality, accuracy, and completeness. Covers expected output, troubleshooting, prerequisites, and setup instructions.

### DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

✅ **DO:**
```markdown
## Expected Output

Run the sample:
```bash
mvn compile exec:java -Dexec.mainClass="com.azure.samples.App"
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

- [`src/main/java/com/azure/samples/App.java`](./src/main/java/com/azure/samples/App.java)—Main entry point
- [`src/main/java/com/azure/samples/Config.java`](./src/main/java/com/azure/samples/Config.java)—Configuration loader
- [`infra/main.bicep`](./infra/main.bicep)—Infrastructure template
```

❌ **DON'T:**
```markdown
- [`src/App.java`](./Java/src/App.java)  # ❌ Wrong path
```

---

### DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure + Java errors (auth failures, firewall, RBAC, prerequisites).

✅ **DO:**
```markdown
## Troubleshooting

### Authentication Errors

If you see "Failed to acquire token":
1. Run `az login` to authenticate with Azure CLI
2. Verify your Azure subscription is active: `az account show`
3. Check you have the required role assignments (see Prerequisites)

### JDBC Connection Errors

If you see "Login failed for user":
- For AAD auth, ensure your IP is allowed in the SQL Server firewall
- Verify the access token is not expired (tokens last ~1 hour)
- Check the database name is correct in your configuration

### Common Azure SDK Errors

- `ResourceNotFoundException`: The resource does not exist at the specified URI
- `HttpResponseException (403)`: Missing RBAC role assignment for your identity
- `CredentialUnavailableException`: No authentication method available (run `az login`)
```

---

### DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly (Azure subscription, JDK, CLI tools, role assignments).

✅ **DO:**
```markdown
## Prerequisites

- **Azure Subscription**: [Create a free account](https://azure.com/free)
- **Java**: JDK 17 or later ([Microsoft OpenJDK](https://learn.microsoft.com/java/openjdk/download) or [Adoptium](https://adoptium.net/))
- **Maven**: 3.9+ ([Download](https://maven.apache.org/download.cgi)) or use included Maven Wrapper (`./mvnw`)
- **Azure CLI**: [Install instructions](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Azure Developer CLI (azd)**: [Install instructions](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd) (optional, for infrastructure deployment)

### Role Assignments

Your Azure identity needs these role assignments:
- `Storage Blob Data Contributor` on the Storage Account
- `Key Vault Secrets User` on the Key Vault
- `Cognitive Services OpenAI User` on the Azure OpenAI resource
```

---

### DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions. Include Azure resource provisioning.

✅ **DO:**
```markdown
## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Azure-Samples/azure-storage-blob-java-samples.git
cd azure-storage-blob-java-samples/quickstart
```

### 2. Build the project

```bash
./mvnw compile
```

### 3. Provision Azure resources

```bash
azd up
```

### 4. Run the sample

```bash
./mvnw exec:java -Dexec.mainClass="com.azure.samples.App"
```
```

---

### DOC-6: Java Version Strategy (LOW)
**Pattern:** Document minimum Java version in both README and build configuration.

✅ **DO:**
```xml
<!-- pom.xml -->
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
</properties>
```

```markdown
# README.md

## Prerequisites

- **Java**: JDK 17 or later required for records and sealed classes support
```

---

### DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

✅ **DO:**
```markdown
## Configuration

Copy `.env.sample` to `.env` and fill in your values:

```bash
cp .env.sample .env
```

Edit `.env` and replace placeholders:
- `AZURE_STORAGE_ACCOUNT_NAME`: Your storage account name (e.g., `mystorageaccount`)
  - Find in Azure Portal: Storage Account > Overview > Name
- `AZURE_KEYVAULT_URL`: Your Key Vault URL (e.g., `https://mykeyvault.vault.azure.net/`)
  - Find in Azure Portal: Key Vault > Overview > Vault URI
```

---

## 12. Infrastructure (Bicep/Terraform)

**What this section covers:** Infrastructure-as-code patterns, Azure Verified Modules, parameter validation, API versioning, resource naming, and role assignments. Shared rules—identical to TypeScript skill since IaC is language-agnostic.

### IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

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

module keyVault 'br/public:avm/res/key-vault/vault:0.11.0' = {
  name: 'keyvault-deployment'
  params: {
    name: keyVaultName
    location: location
  }
}
```

❌ **DON'T:**
```bicep
module cognitiveServices 'br/public:avm/res/cognitive-services/account:0.7.1' = {
  // ❌ Outdated version (current is 1.0.1+)
}
```

---

### IaC-2: Bicep Parameter Validation (CRITICAL)
**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

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
@description('Azure AD admin object ID')
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
**Pattern:** Create role assignments in Bicep for managed identities to access Azure resources.

✅ **DO:**
```bicep
resource appService 'Microsoft.Web/sites@2023-12-01' = {
  name: appServiceName
  location: location
  identity: { type: 'SystemAssigned' }
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
- Cosmos DB Account Reader Role: `fbdf93bf-df7d-467e-a4d2-9458aa1360c8`
- Cognitive Services OpenAI User: `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`

---

### IaC-5: Network Security (HIGH)
**Pattern:** For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

✅ **DO (Quickstart):**
```bicep
resource openai 'Microsoft.CognitiveServices/accounts@2023-05-01' = {
  name: openaiAccountName
  properties: {
    publicNetworkAccess: 'Enabled'
    networkAcls: { defaultAction: 'Allow' }
  }
}
// NOTE: This quickstart uses public endpoints for simplicity.
// For production, use private endpoints and set defaultAction: 'Deny'.
```

---

### IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

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
var storageAccountName = 'mystorageaccount123'
var keyVaultName = 'kv-${uniqueString(resourceGroup().id)}'
```

---

## 13. Azure Developer CLI (azd)

**What this section covers:** azd integration patterns, azure.yaml structure, service definitions, hooks, and host types.

### AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

> **⚠️ FALSE POSITIVE PREVENTION:** Before flagging `azure.yaml` as missing or incomplete:
> 1. The `services`, `hooks`, and `host` fields in `azure.yaml` are **OPTIONAL**. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.
> 3. **Check parent directories**—in monorepo/multi-sample layouts, `azure.yaml` often lives one or more levels ABOVE the language-specific project folder.

✅ **DO:**
```yaml
name: azure-storage-blob-java-sample
metadata:
  template: azure-storage-blob-java-sample@0.0.1

services:
  app:
    project: ./
    language: java
    host: appservice

hooks:
  preprovision:
    shell: sh
    run: |
      echo "Validating prerequisites..."
      az account show > /dev/null || (echo "❌ Not logged in. Run 'az login'" && exit 1)
      java -version || (echo "❌ Java not found. Install JDK 17+" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "✅ Provisioning complete"
      echo "Run './mvnw compile exec:java' to test the sample."

  predeploy:
    shell: sh
    run: |
      echo "Building application..."
      ./mvnw clean package -DskipTests
```

---

### AZD-2: Service Host Types (MEDIUM)
**Pattern:** Choose correct `host` type for Java application types.

✅ **DO:**
```yaml
# App Service (Spring Boot, standalone Java web apps)
services:
  web:
    project: ./
    language: java
    host: appservice

# Azure Functions (Java)
services:
  api:
    project: ./
    language: java
    host: function

# Container Apps (any Java app with Dockerfile)
services:
  backend:
    project: ./
    language: java
    host: containerapp
    docker:
      path: ./Dockerfile

# Azure Spring Apps
services:
  springapp:
    project: ./
    language: java
    host: springapp
```

**Supported hosts:** `appservice`, `function`, `containerapp`, `staticwebapp`, `aks`, `springapp`

---

## 14. Spring Boot Integration

**What this section covers:** Spring Cloud Azure integration patterns, auto-configuration, property sources, and health indicators. These patterns apply when samples use Spring Boot with Azure services.

### SB-1: Spring Cloud Azure Configuration (HIGH)
**Pattern:** Use `spring-cloud-azure-starter` for auto-configured Azure SDK clients. Let Spring manage client lifecycle.

✅ **DO:**
```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.azure.spring</groupId>
            <artifactId>spring-cloud-azure-dependencies</artifactId>
            <version>${spring-cloud-azure.version}</version>  <!-- ✅ Use property, set to latest stable -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Define version in properties—use latest stable compatible with your Spring Boot version -->
<!-- Check https://learn.microsoft.com/azure/developer/java/spring-framework/developer-guide-overview -->
<properties>
    <spring-cloud-azure.version>5.19.0</spring-cloud-azure.version>  <!-- Update to latest stable -->
</properties>

<dependencies>
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-storage-blob</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  cloud:
    azure:
      credential:
        managed-identity-enabled: true    # ✅ Uses managed identity in cloud
      storage:
        blob:
          account-name: ${AZURE_STORAGE_ACCOUNT_NAME}
```

```java
// ✅ Spring auto-configures BlobServiceClient
@Service
public class StorageService {

    private final BlobServiceClient blobServiceClient;

    public StorageService(BlobServiceClient blobServiceClient) {
        this.blobServiceClient = blobServiceClient;  // ✅ Injected by Spring
    }

    public void uploadBlob(String containerName, String blobName, String content) {
        blobServiceClient.getBlobContainerClient(containerName)
            .getBlobClient(blobName)
            .upload(BinaryData.fromString(content), true);
    }
}
```

❌ **DON'T:**
```java
// ❌ Don't manually create clients in Spring Boot apps
@Service
public class StorageService {

    public void uploadBlob() {
        // ❌ Bypasses Spring auto-configuration, ignores retries, identity settings
        BlobServiceClient client = new BlobServiceClientBuilder()
            .connectionString(System.getenv("AZURE_STORAGE_CONNECTION_STRING"))
            .buildClient();
    }
}
```

**Why:** Spring Cloud Azure auto-configures clients with proper credential chains, retry policies, and connection pooling. Manual creation bypasses these features.

---

### SB-2: Spring Boot Starter Dependencies (MEDIUM)
**Pattern:** Use purpose-specific Spring Cloud Azure starters instead of manual Azure SDK dependencies.

✅ **DO:**
```xml
<dependencies>
    <!-- ✅ Purpose-specific starters -->
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-keyvault-secrets</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-servicebus</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-cosmos</artifactId>
    </dependency>
</dependencies>
```

❌ **DON'T:**
```xml
<dependencies>
    <!-- ❌ Don't mix raw Azure SDK with Spring starters (version conflicts) -->
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>
        <version>12.28.1</version>  <!-- ❌ Conflicts with Spring starter version -->
    </dependency>
</dependencies>
```

---

### SB-3: Property Source Configuration (MEDIUM)
**Pattern:** Use Azure App Configuration or Key Vault as Spring property sources for externalized configuration.

✅ **DO:**
```xml
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-starter-keyvault-secrets</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    azure:
      keyvault:
        secret:
          property-sources:
            - endpoint: ${AZURE_KEYVAULT_URL}
              credential:
                managed-identity-enabled: true
```

```java
// ✅ Key Vault secrets available as Spring properties
@Value("${db-password}")
private String dbPassword;  // ✅ Resolved from Key Vault secret named "db-password"
```

❌ **DON'T:**
```java
// ❌ Don't manually fetch secrets in Spring Boot
@PostConstruct
public void init() {
    SecretClient client = new SecretClientBuilder()
        .vaultUrl(vaultUrl)
        .credential(new DefaultAzureCredentialBuilder().build())
        .buildClient();
    this.dbPassword = client.getSecret("db-password").getValue();
}
```

---

### SB-4: Health Indicators (MEDIUM)
**Pattern:** Enable Spring Cloud Azure health indicators for Azure service connectivity checks.

✅ **DO:**
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health
  health:
    azure-cosmos:
      enabled: true
    azure-storage:
      enabled: true
```

```java
// ✅ Custom health check (when auto-configured indicator isn't available)
@Component
public class AzureOpenAiHealthIndicator extends AbstractHealthIndicator {

    private final OpenAIClient openAIClient;

    public AzureOpenAiHealthIndicator(OpenAIClient openAIClient) {
        this.openAIClient = openAIClient;
    }

    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            openAIClient.getChatCompletions("gpt-4o",
                new ChatCompletionsOptions(List.of(new ChatRequestUserMessage("ping"))));
            builder.up().withDetail("service", "Azure OpenAI");
        } catch (Exception e) {
            builder.down(e);
        }
    }
}
```

❌ **DON'T:**
```yaml
# ❌ Don't disable health checks without explanation
management:
  health:
    azure-cosmos:
      enabled: false  # ❌ Why disabled?
```

---

### SB-5: Spring Profiles for Azure (LOW)
**Pattern:** Use Spring profiles to separate local development and Azure deployment configurations.

✅ **DO:**
```yaml
# application.yml (shared)
spring:
  cloud:
    azure:
      credential:
        managed-identity-enabled: false  # Default: use CLI auth

---
# application-azure.yml (production profile)
spring:
  cloud:
    azure:
      credential:
        managed-identity-enabled: true
```

```bash
# Run locally (uses Azure CLI auth)
./mvnw spring-boot:run

# Run in Azure (uses managed identity)
SPRING_PROFILES_ACTIVE=azure ./mvnw spring-boot:run
```

---

## 15. CI/CD & Testing

**What this section covers:** Continuous integration patterns, Maven/Gradle CI workflows, JUnit 5, and build validation.

### CI-1: Maven/Gradle CI Workflow (HIGH)
**Pattern:** Run build, test, and security checks in CI before publication.

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
      - uses: actions/setup-java@v4
        with:
          distribution: 'microsoft'
          java-version: '17'
          cache: 'maven'
      - run: ./mvnw verify                        # Compile + test + package
      - run: ./mvnw dependency-check:check         # CVE scan
```

```groovy
// build.gradle - equivalent for Gradle
plugins {
    id 'java'
    id 'org.owasp.dependencycheck' version '10.0.4'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

---

### CI-2: JUnit 5 Test Patterns (MEDIUM)
**Pattern:** Include at least basic tests that validate configuration loading and client construction.

✅ **DO:**
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;
import static org.junit.jupiter.api.Assertions.*;

class ConfigTest {

    @Test
    void configThrowsOnMissingVars() {
        // ✅ Validate error handling for missing env vars
        assertThrows(IllegalStateException.class, () -> Config.fromEnvironment());
    }
}

// ✅ Integration test (only runs with Azure credentials)
@EnabledIfEnvironmentVariable(named = "AZURE_STORAGE_ACCOUNT_NAME", matches = ".+")
class StorageIntegrationTest {

    @Test
    void canConnectToStorage() {
        var credential = new DefaultAzureCredentialBuilder().build();
        var client = new BlobServiceClientBuilder()
            .endpoint("https://" + System.getenv("AZURE_STORAGE_ACCOUNT_NAME") + ".blob.core.windows.net")
            .credential(credential)
            .buildClient();
        assertNotNull(client.getAccountName());
    }
}
```

---

## 16. Advanced SDK Configuration

**What this section covers:** Distributed tracing, GraalVM native-image compatibility, and other cross-cutting concerns for production-ready Azure SDK Java applications.

### ADV-1: Distributed Tracing with OpenTelemetry (MEDIUM)
**Pattern:** Use `com.azure:azure-core-tracing-opentelemetry` for distributed tracing across Azure SDK operations.

✅ **DO:**
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-core-tracing-opentelemetry</artifactId>
</dependency>
<!-- OpenTelemetry SDK -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```java
// ✅ Azure SDK automatically picks up the OpenTelemetry tracer
// when azure-core-tracing-opentelemetry is on the classpath.
// Configure the OpenTelemetry SDK as usual:
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(
        OtlpGrpcSpanExporter.builder().build()
    ).build())
    .build();

OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    .buildAndRegisterGlobal();

// ✅ Azure SDK calls are now automatically traced
// Spans include HTTP requests, retries, and service-specific operations
BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .buildClient();
// All operations on this client produce OpenTelemetry spans
```

> **Note:** For Azure Monitor integration, use `com.azure:azure-monitor-opentelemetry-exporter` to send traces to Application Insights.

---

### ADV-2: GraalVM Native Image (MEDIUM)
**Pattern:** For Azure SDK Java applications compiled with GraalVM native-image, add the `azure-core-native` compatibility package.

✅ **DO:**
```xml
<!-- pom.xml - GraalVM native-image support -->
<dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-core-native</artifactId>
</dependency>
```

```bash
# Build native image with Maven
mvn -Pnative native:compile
```

> **Note:** Not all Azure SDK features are fully compatible with native-image. Test thoroughly. The `azure-core-native` package provides GraalVM reflection configuration for core Azure SDK types. Check the [Azure SDK for Java native image documentation](https://learn.microsoft.com/azure/developer/java/sdk/native-image) for service-specific support status.

---

## Pre-Review Checklist (Comprehensive)

Use this comprehensive checklist before submitting an Azure SDK Java sample for review:

### 🔧 Project Setup
- [ ] Java 17 or 21 LTS configured in build file
- [ ] Build file has complete metadata (name, description, license, developer)
- [ ] Every dependency is imported somewhere (no phantom deps)
- [ ] Using Track 2 Azure SDK packages (`com.azure:*`, not `com.microsoft.azure:*`)
- [ ] Environment variables validated with descriptive errors
- [ ] `azure-sdk-bom` used for version management
- [ ] Maven/Gradle Wrapper committed
- [ ] Only one build tool (Maven OR Gradle, not both)
- [ ] No critical/high CVEs in dependency tree
- [ ] Compiler warnings enabled (`-Xlint:all`)
- [ ] .editorconfig present (optional but recommended)

### 🔐 Security & Hygiene
- [ ] `.gitignore` protects `.env`, `target/`, `build/`, `.idea/`, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed (subscription IDs, tenant IDs, connection strings, API keys)
- [ ] No build artifacts committed (`target/`, `build/`, `*.class`)
- [ ] Dead code removed (unused files, imports, methods, commented-out code)
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] CONTRIBUTING.md, SECURITY.md referenced or included
- [ ] Package legitimacy verified (all `com.azure:*` from Microsoft)

### ☁️ Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Client options configured (retry policies, timeouts where applicable; quickstarts may use defaults)
- [ ] Token refresh handled via SDK's `getToken()` re-calls (not manual expiration tracking)
- [ ] Managed identity pattern documented in README (system vs user-assigned)
- [ ] Service-specific client builder patterns followed
- [ ] Pagination handled with `PagedIterable` / `PagedFlux`
- [ ] Resource cleanup with try-with-resources or finally blocks
- [ ] Async patterns use `*AsyncClient` with Reactor (`Mono`/`Flux`) where appropriate
- [ ] SLF4J logging configured (logging implementation on classpath)
- [ ] HttpClient selection documented if non-default

### 🗄️ Data Services (if applicable)
- [ ] SQL: JDBC with AAD token authentication (not SQL auth with passwords)
- [ ] SQL: All queries use `PreparedStatement` (no string concatenation)
- [ ] SQL/Cosmos: Batch operations for multiple rows (not row-by-row)
- [ ] Cosmos: Queries include partition key (avoid cross-partition)
- [ ] Cosmos: Uses `credential()` with AAD (not `.key()`)
- [ ] Storage: Blob/Table client patterns followed
- [ ] Storage: SAS fallback documented for local dev (optional)
- [ ] Batch size rationale documented

### 🤖 AI Services (if applicable)
- [ ] Using `com.azure:azure-ai-openai` with AAD credential
- [ ] OpenAI client has timeout and retry configuration
- [ ] API versions documented with links to version docs
- [ ] Vector search: DiskANN index creation guarded by row count check (≥1000 for SQL)
- [ ] Vector dimensions validated (match between model and storage)
- [ ] Pre-computed embedding data file committed to repo
- [ ] Speech SDK: `com.microsoft.cognitiveservices.speech:client-sdk` version pinned explicitly (not in BOM)
- [ ] Speech SDK: `SpeechRecognizer`/`SpeechSynthesizer` closed via try-with-resources

### 💬 Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses checkpoint store for processing
- [ ] Connection strings avoided (prefer AAD auth)

### 🌱 Spring Boot (if applicable)
- [ ] Spring Cloud Azure starters used (not raw SDK dependencies)
- [ ] `spring-cloud-azure-dependencies` BOM imported
- [ ] Auto-configured clients injected (not manually created)
- [ ] Health indicators enabled for Azure services
- [ ] Spring profiles separate local vs Azure config

### ❌ Error Handling
- [ ] Catches specific Azure exceptions (`HttpResponseException`, `ResourceNotFoundException`)
- [ ] Error messages are contextual and actionable
- [ ] Auth/network/RBAC errors include troubleshooting hints

### 📄 Documentation
- [ ] README "Expected output" copy-pasted from real run (not fabricated)
- [ ] All internal folder/file links match actual filesystem paths
- [ ] Prerequisites section complete (subscription, JDK, CLI, role assignments)
- [ ] Troubleshooting section covers common Azure + Java errors
- [ ] Setup instructions clear and tested
- [ ] Placeholder values have clear replacement instructions
- [ ] Java version documented in README and build file

### 🏗️ Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated (`@minLength`, `@maxLength`, `@allowed`)
- [ ] API versions current (2023-2024+)
- [ ] Network security documented (public vs. private endpoints)
- [ ] RBAC role assignments created for managed identities
- [ ] Resource naming follows CAF conventions
- [ ] Output values follow azd naming conventions (`AZURE_*`)
- [ ] `azure.yaml` includes services, hooks, metadata

### 🔬 Advanced (if applicable)
- [ ] Distributed tracing: `azure-core-tracing-opentelemetry` configured if needed
- [ ] GraalVM native-image: `azure-core-native` included if targeting native compilation
- [ ] SQL passwordless auth: `azure-identity-extensions` JDBC plugin considered

### 🧪 CI/CD
- [ ] Build runs in CI (`./mvnw verify`)
- [ ] CVE scanning runs in CI
- [ ] Tests pass (JUnit 5)
- [ ] Java setup uses Microsoft OpenJDK or Adoptium distribution

---

## Companion Skills

For additional review concerns, reference these complementary skills:

- **[`azure-sdk-typescript-sample-review`](../azure-sdk-typescript-sample-review/SKILL.md)**: TypeScript Azure SDK sample review (same structure, different language)
- **[`azure-sdk-dotnet-sample-review`](../azure-sdk-dotnet-sample-review/SKILL.md)**: .NET 9/10 + Aspire Azure SDK sample review
- **[`azure-sdk-python-sample-review`](../azure-sdk-python-sample-review/SKILL.md)**: Python 3.9+ + async Azure SDK sample review
- **[`azure-sdk-go-sample-review`](../azure-sdk-go-sample-review/SKILL.md)**: Go 1.21+ Azure SDK sample review
- **[`azure-sdk-rust-sample-review`](../azure-sdk-rust-sample-review/SKILL.md)**: Rust 2021 edition Azure SDK sample review
- **[`acrolinx-score-improvement`](../acrolinx-score-improvement/SKILL.md)**: Article quality, readability, style, and terminology consistency

---

## Scope Note: Services Not Yet Covered

This skill focuses on the most commonly used Azure services in Java samples. The following services are not yet covered in detail, but the general patterns (authentication, client construction, error handling) apply:

- Azure Communication Services
- Azure Cache for Redis
- Azure Monitor
- Azure Container Registry
- Azure App Configuration (standalone—Spring integration covered in SB-3)
- Azure SignalR Service
- Azure API Management
- Azure Event Grid (basic patterns similar to Event Hubs; uses `com.azure:azure-messaging-eventgrid`)

For samples using these services, apply the core patterns from Sections 1–2 (Project Setup, Azure SDK Client Patterns) and reference service-specific documentation.

---

## Reference Links

Consolidation of all documentation links referenced throughout this skill:

### Azure SDK & Authentication
- [Azure SDK for Java](https://learn.microsoft.com/java/api/overview/azure/)
- [Azure SDK for Java—GitHub](https://github.com/Azure/azure-sdk-for-java)
- [DefaultAzureCredential](https://learn.microsoft.com/java/api/com.azure.identity.defaultazurecredential)
- [Managed Identities](https://learn.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)
- [azure-sdk-bom](https://central.sonatype.com/artifact/com.azure/azure-sdk-bom)
- [azure-identity-extensions (JDBC plugin)](https://central.sonatype.com/artifact/com.azure/azure-identity-extensions)
- [azure-core-tracing-opentelemetry](https://central.sonatype.com/artifact/com.azure/azure-core-tracing-opentelemetry)
- [azure-core-native (GraalVM)](https://central.sonatype.com/artifact/com.azure/azure-core-native)
- [Azure SDK Java Logging](https://learn.microsoft.com/azure/developer/java/sdk/logging-overview)
- [Azure SDK Java HTTP Clients](https://learn.microsoft.com/azure/developer/java/sdk/http-client-pipeline)
- [Azure SDK Java Native Image](https://learn.microsoft.com/azure/developer/java/sdk/native-image)

### Spring Cloud Azure
- [Spring Cloud Azure Documentation](https://learn.microsoft.com/azure/developer/java/spring-framework/)
- [Spring Cloud Azure Starters](https://learn.microsoft.com/azure/developer/java/spring-framework/spring-cloud-azure-overview)
- [Spring Cloud Azure Samples](https://github.com/Azure-Samples/azure-spring-boot-samples)

### API Versioning
- [Azure OpenAI API Versions](https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation)
- [Azure REST API Specifications](https://github.com/Azure/azure-rest-api-specs)

### AI & Speech
- [Azure Speech SDK for Java](https://learn.microsoft.com/azure/ai-services/speech-service/quickstarts/setup-platform?pivots=programming-language-java)
- [Speech SDK Release Notes](https://learn.microsoft.com/azure/ai-services/speech-service/releasenotes)
- [Azure AI OpenAI SDK for Java](https://central.sonatype.com/artifact/com.azure/azure-ai-openai)

### Infrastructure
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Cloud Adoption Framework—Naming Conventions](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure Developer CLI (azd)](https://learn.microsoft.com/azure/developer/azure-developer-cli/)

### Security
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Azure Private Endpoints](https://learn.microsoft.com/azure/private-link/private-endpoint-overview)

### Java
- [Microsoft OpenJDK](https://learn.microsoft.com/java/openjdk/download)
- [Maven Central—com.azure](https://central.sonatype.com/namespace/com.azure)
- [JUnit 5](https://junit.org/junit5/)

### Microsoft Open Source
- [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/)
- [Azure Samples GitHub](https://github.com/Azure-Samples)

---

## Summary

This skill captures **Azure SDK Java sample patterns** adapted from the TypeScript review skill and generalized for the Java Azure SDK ecosystem:

### Severity Breakdown
- **CRITICAL** (11 rules): Credentials, phantom deps, CVE scanning, token refresh, AVM versions, parameter validation, .gitignore, fabricated output, broken auth, missing error handling, SQL injection
- **HIGH** (22 rules): Client builder patterns, token management, managed identity, pagination, OpenAI client, database patterns, DiskANN guards, batch operations, RBAC, lock files, role assignments, pre-computed data, .env.sample, prerequisites, dead code, LICENSE, resource naming, Spring Cloud Azure config, CI workflow, network security, Service Bus patterns, Java version
- **MEDIUM** (27 rules): Client options, retry policies, SQL parameters, embeddings, error handling, JSON loading, troubleshooting, azd structure, compiler warnings, BOM management, SAS fallback, dimensions, placeholder docs, resource cleanup, API versions, governance files, Spring starters, property sources, health indicators, host types, setup instructions, project metadata, configuration, package legitimacy, test patterns, data loading, Document Intelligence
- **LOW** (8 rules): API version docs, .editorconfig, CONTRIBUTING.md, scope notes, Java version strategy, Spring profiles, region availability, placeholder guidance

### Service Coverage
- **Core SDK**: Authentication, credentials, managed identities, client builder patterns, token management, pagination, resource cleanup
- **Data**: Cosmos DB, Azure SQL (JDBC), Storage (Blob), batch operations
- **AI**: Azure OpenAI (embeddings, chat), Document Intelligence, vector dimensions
- **Messaging**: Service Bus, Event Hubs, checkpoint management
- **Security**: Key Vault (secrets, keys, certificates)
- **Vector Search**: Azure SQL DiskANN, Cosmos DB
- **Spring Boot**: Spring Cloud Azure starters, auto-configuration, property sources, health indicators, profiles
- **Infrastructure**: Bicep/Terraform, AVM modules, azd integration, RBAC, CAF naming

Apply these patterns to ensure Azure SDK Java samples are **secure, accurate, maintainable, and ready for publication**.
