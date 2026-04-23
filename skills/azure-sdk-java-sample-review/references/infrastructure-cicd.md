# Infrastructure, Documentation, Spring Boot, CI/CD & Advanced SDK

**What this section covers:** README & documentation quality, Bicep/Terraform IaC patterns, Azure Verified Modules, azd integration, Spring Boot / Spring Cloud Azure patterns, CI/CD workflows, distributed tracing, GraalVM, and the comprehensive review checklist.

## DOC-1: Expected Output (CRITICAL)
**Pattern:** README "Expected output" sections must be copy-pasted from actual program runs. Never fabricate output.

DO:
```markdown
## Expected Output

Run the sample:
```bash
mvn compile exec:java -Dexec.mainClass="com.azure.samples.App"
```

You should see output similar to:

```
Connected to Azure Blob Storage
Container 'samples' created
Uploaded blob 'sample.txt' (14 bytes)
Downloaded blob content: "Hello, Azure!"
```

> Note: Exact output may vary based on your Azure environment.
```

DON'T:
```markdown
## Expected Output

```
Blob uploaded successfully  # Not actual output, fabricated
```
```

---

## DOC-2: Folder Path Links (CRITICAL)
**Pattern:** All internal README links must match actual filesystem paths.

DO:
```markdown
## Project Structure

- [`src/main/java/com/azure/samples/App.java`](./src/main/java/com/azure/samples/App.java) -- Main entry point
- [`src/main/java/com/azure/samples/Config.java`](./src/main/java/com/azure/samples/Config.java) -- Configuration loader
- [`infra/main.bicep`](./infra/main.bicep) -- Infrastructure template
```

DON'T:
```markdown
- [`src/App.java`](./Java/src/App.java)  # Wrong path
```

---

## DOC-3: Troubleshooting Section (MEDIUM)
**Pattern:** Include troubleshooting for common Azure + Java errors (auth failures, firewall, RBAC, prerequisites).

DO:
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

## DOC-4: Prerequisites Section (HIGH)
**Pattern:** Document all prerequisites clearly (Azure subscription, JDK, CLI tools, role assignments).

DO:
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

## DOC-5: Setup Instructions (MEDIUM)
**Pattern:** Provide clear, tested setup instructions. Include Azure resource provisioning.

DO:
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

## DOC-6: Java Version Strategy (LOW)
**Pattern:** Document minimum Java version in both README and build configuration.

DO:
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

## DOC-7: Placeholder Values (MEDIUM)
**Pattern:** READMEs must provide clear instructions for placeholder values.

DO:
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

## IaC-1: Azure Verified Module (AVM) Versions (CRITICAL)
**Pattern:** Use current stable versions of Azure Verified Modules. Check azure.github.io/Azure-Verified-Modules for latest.

DO:
```bicep
// Current AVM modules (check https://azure.github.io/Azure-Verified-Modules/)
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

DON'T:
```bicep
module cognitiveServices 'br/public:avm/res/cognitive-services/account:0.7.1' = {
  // Outdated version (current is 1.0.1+)
}
```

---

## IaC-2: Bicep Parameter Validation (CRITICAL)
**Pattern:** Use `@minLength`, `@maxLength`, `@allowed` decorators to validate required parameters.

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
@description('Azure AD admin object ID')
param aadAdminObjectId string  // No validation, accepts empty string
```

---

## IaC-3: API Versions (MEDIUM)
**Pattern:** Use current API versions (2023+). Avoid versions older than 2 years.

DO:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = { }
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = { }
resource appService 'Microsoft.Web/sites@2023-12-01' = { }
```

DON'T:
```bicep
resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  // 5+ years old
}
```

---

## IaC-4: RBAC Role Assignments (HIGH)
**Pattern:** Create role assignments in Bicep for managed identities to access Azure resources.

DO:
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

## IaC-5: Network Security (HIGH)
**Pattern:** For quickstart samples, public endpoints acceptable with security comment. For production samples, use private endpoints.

DO (Quickstart):
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

## IaC-6: Output Values (MEDIUM)
**Pattern:** Output all values needed by the application. Follow azd naming conventions (`AZURE_*`).

DO:
```bicep
output AZURE_STORAGE_ACCOUNT_NAME string = storageAccount.name
output AZURE_KEYVAULT_URL string = keyVault.properties.vaultUri
output AZURE_OPENAI_ENDPOINT string = openai.properties.endpoint
output AZURE_COSMOS_ENDPOINT string = cosmos.properties.documentEndpoint
```

---

## IaC-7: Resource Naming Conventions (HIGH)
**Pattern:** Follow Cloud Adoption Framework (CAF) naming conventions.

DO:
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

DON'T:
```bicep
var storageAccountName = 'mystorageaccount123'
var keyVaultName = 'kv-${uniqueString(resourceGroup().id)}'
```

---

## AZD-1: azure.yaml Structure (MEDIUM)
**Pattern:** Complete `azure.yaml` with services, hooks, and metadata.

> **FALSE POSITIVE PREVENTION:** Before flagging `azure.yaml` as missing or incomplete:
> 1. The `services`, `hooks`, and `host` fields in `azure.yaml` are **OPTIONAL**. For infrastructure-only samples, a minimal `azure.yaml` with just `name` and `metadata` is correct.
> 2. Do NOT flag missing optional fields if `azd up` and `azd down` work correctly.
> 3. **Check parent directories** -- in monorepo/multi-sample layouts, `azure.yaml` often lives one or more levels ABOVE the language-specific project folder.

DO:
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
      az account show > /dev/null || (echo "Not logged in. Run 'az login'" && exit 1)
      java -version || (echo "Java not found. Install JDK 17+" && exit 1)

  postprovision:
    shell: sh
    run: |
      echo "Provisioning complete"
      echo "Run './mvnw compile exec:java' to test the sample."

  predeploy:
    shell: sh
    run: |
      echo "Building application..."
      ./mvnw clean package -DskipTests
```

---

## AZD-2: Service Host Types (MEDIUM)
**Pattern:** Choose correct `host` type for Java application types.

DO:
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

## SB-1: Spring Cloud Azure Configuration (HIGH)
**Pattern:** Use `spring-cloud-azure-starter` for auto-configured Azure SDK clients. Let Spring manage client lifecycle.

DO:
```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.azure.spring</groupId>
            <artifactId>spring-cloud-azure-dependencies</artifactId>
            <version>${spring-cloud-azure.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Define version in properties -- use latest stable compatible with your Spring Boot version -->
<!-- Check https://learn.microsoft.com/azure/developer/java/spring-framework/developer-guide-overview -->
<properties>
    <spring-cloud-azure.version>5.19.0</spring-cloud-azure.version>
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
        managed-identity-enabled: true    # Uses managed identity in cloud
      storage:
        blob:
          account-name: ${AZURE_STORAGE_ACCOUNT_NAME}
```

```java
// Spring auto-configures BlobServiceClient
@Service
public class StorageService {

    private final BlobServiceClient blobServiceClient;

    public StorageService(BlobServiceClient blobServiceClient) {
        this.blobServiceClient = blobServiceClient;  // Injected by Spring
    }

    public void uploadBlob(String containerName, String blobName, String content) {
        blobServiceClient.getBlobContainerClient(containerName)
            .getBlobClient(blobName)
            .upload(BinaryData.fromString(content), true);
    }
}
```

DON'T:
```java
// Don't manually create clients in Spring Boot apps
@Service
public class StorageService {

    public void uploadBlob() {
        // Bypasses Spring auto-configuration, ignores retries, identity settings
        BlobServiceClient client = new BlobServiceClientBuilder()
            .connectionString(System.getenv("AZURE_STORAGE_CONNECTION_STRING"))
            .buildClient();
    }
}
```

**Why:** Spring Cloud Azure auto-configures clients with proper credential chains, retry policies, and connection pooling. Manual creation bypasses these features.

---

## SB-2: Spring Boot Starter Dependencies (MEDIUM)
**Pattern:** Use purpose-specific Spring Cloud Azure starters instead of manual Azure SDK dependencies.

DO:
```xml
<dependencies>
    <!-- Purpose-specific starters -->
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

DON'T:
```xml
<dependencies>
    <!-- Don't mix raw Azure SDK with Spring starters (version conflicts) -->
    <dependency>
        <groupId>com.azure.spring</groupId>
        <artifactId>spring-cloud-azure-starter-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>
        <version>12.28.1</version>  <!-- Conflicts with Spring starter version -->
    </dependency>
</dependencies>
```

---

## SB-3: Property Source Configuration (MEDIUM)
**Pattern:** Use Azure App Configuration or Key Vault as Spring property sources for externalized configuration.

DO:
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
// Key Vault secrets available as Spring properties
@Value("${db-password}")
private String dbPassword;  // Resolved from Key Vault secret named "db-password"
```

DON'T:
```java
// Don't manually fetch secrets in Spring Boot
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

## SB-4: Health Indicators (MEDIUM)
**Pattern:** Enable Spring Cloud Azure health indicators for Azure service connectivity checks.

DO:
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
// Custom health check (when auto-configured indicator isn't available)
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

DON'T:
```yaml
# Don't disable health checks without explanation
management:
  health:
    azure-cosmos:
      enabled: false  # Why disabled?
```

---

## SB-5: Spring Profiles for Azure (LOW)
**Pattern:** Use Spring profiles to separate local development and Azure deployment configurations.

DO:
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

## CI-1: Maven/Gradle CI Workflow (HIGH)
**Pattern:** Run build, test, and security checks in CI before publication.

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

## CI-2: JUnit 5 Test Patterns (MEDIUM)
**Pattern:** Include at least basic tests that validate configuration loading and client construction.

DO:
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfEnvironmentVariable;
import static org.junit.jupiter.api.Assertions.*;

class ConfigTest {

    @Test
    void configThrowsOnMissingVars() {
        // Validate error handling for missing env vars
        assertThrows(IllegalStateException.class, () -> Config.fromEnvironment());
    }
}

// Integration test (only runs with Azure credentials)
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

## ADV-1: Distributed Tracing with OpenTelemetry (MEDIUM)
**Pattern:** Use `com.azure:azure-core-tracing-opentelemetry` for distributed tracing across Azure SDK operations.

DO:
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
// Azure SDK automatically picks up the OpenTelemetry tracer
// when azure-core-tracing-opentelemetry is on the classpath.
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

// Azure SDK calls are now automatically traced
BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .buildClient();
// All operations on this client produce OpenTelemetry spans
```

> **Note:** For Azure Monitor integration, use `com.azure:azure-monitor-opentelemetry-exporter` to send traces to Application Insights.

---

## ADV-2: GraalVM Native Image (MEDIUM)
**Pattern:** For Azure SDK Java applications compiled with GraalVM native-image, add the `azure-core-native` compatibility package.

DO:
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

> **Note:** Not all Azure SDK features are fully compatible with native-image. Test thoroughly. The `azure-core-native` package provides GraalVM reflection configuration for core Azure SDK types. Check the [Azure SDK for Java native image documentation](https://learn.microsoft.com/azure/developer/java/sdk/overview) for service-specific support status.

---

## Pre-Review Checklist (Comprehensive)

### Project Setup
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

### Security & Hygiene
- [ ] `.gitignore` protects `.env`, `target/`, `build/`, `.idea/`, `.azure/`
- [ ] `.env.sample` provided with placeholders (no real credentials)
- [ ] No live credentials committed
- [ ] No build artifacts committed
- [ ] Dead code removed
- [ ] LICENSE file present (MIT required for Azure Samples)
- [ ] Package legitimacy verified (all `com.azure:*` from Microsoft)

### Azure SDK Patterns
- [ ] `DefaultAzureCredential` used for authentication
- [ ] Credential instance cached and reused across clients
- [ ] Token refresh handled via SDK's `getToken()` re-calls
- [ ] Pagination handled with `PagedIterable` / `PagedFlux`
- [ ] Resource cleanup with try-with-resources or finally blocks
- [ ] SLF4J logging configured

### Data Services (if applicable)
- [ ] SQL: JDBC with AAD token authentication
- [ ] SQL: All queries use `PreparedStatement`
- [ ] SQL/Cosmos: Batch operations for multiple rows
- [ ] Cosmos: Queries include partition key
- [ ] Cosmos: Uses `credential()` with AAD

### AI Services (if applicable)
- [ ] Using `com.azure:azure-ai-openai` with AAD credential
- [ ] Vector dimensions validated
- [ ] Pre-computed embedding data file committed
- [ ] Speech SDK version pinned explicitly (not in BOM)

### Messaging (if applicable)
- [ ] Service Bus messages completed/abandoned properly
- [ ] Event Hubs uses checkpoint store for processing

### Spring Boot (if applicable)
- [ ] Spring Cloud Azure starters used (not raw SDK dependencies)
- [ ] Auto-configured clients injected (not manually created)
- [ ] Health indicators enabled

### Documentation
- [ ] README "Expected output" from real run
- [ ] All internal links match actual filesystem paths
- [ ] Prerequisites section complete
- [ ] Troubleshooting section covers common errors

### Infrastructure (if applicable)
- [ ] Azure Verified Module versions current
- [ ] Bicep parameters validated
- [ ] API versions current (2023+)
- [ ] RBAC role assignments created for managed identities
- [ ] Resource naming follows CAF conventions

### CI/CD
- [ ] Build runs in CI (`./mvnw verify`)
- [ ] CVE scanning runs in CI
- [ ] Tests pass (JUnit 5)
