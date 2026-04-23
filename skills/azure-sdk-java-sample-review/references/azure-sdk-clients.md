# Azure SDK Client Patterns

**What this section covers:** Authentication, credential management, client construction with builders, retry policies, managed identity patterns, pagination, async clients, logging, and HTTP client selection. These are foundational patterns that apply across ALL Azure SDK packages.

## AZ-1: Client Builder with DefaultAzureCredential (HIGH)
**Pattern:** Use `DefaultAzureCredential` for samples. Construct clients with the builder pattern. Cache credential instances.

DO:
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

// Cache credential instance
DefaultAzureCredential credential = new DefaultAzureCredentialBuilder().build();

// Storage Blob
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();

// Key Vault
SecretClient secretClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

// Service Bus
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();

// Cosmos DB
CosmosClient cosmosClient = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildClient();
```

DON'T:
```java
// Don't use connection strings in samples (prefer AAD auth)
BlobServiceClient client = new BlobServiceClientBuilder()
    .connectionString(connectionString)
    .buildClient();

// Don't use account keys
BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(new StorageSharedKeyCredential(accountName, accountKey))
    .buildClient();

// Don't recreate credential for each client
BlobServiceClient blobClient = new BlobServiceClientBuilder()
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
SecretClient secretClient = new SecretClientBuilder()
    .credential(new DefaultAzureCredentialBuilder().build())
    .buildClient();
```

**Why:** `DefaultAzureCredential` works locally (Azure CLI, IntelliJ, VS Code) and in cloud (managed identity). Connection strings and keys are less secure and harder to rotate.

---

## AZ-2: Client Options -- HttpPipelinePolicy, RetryOptions (MEDIUM)
**Pattern:** Configure retry policies, timeouts, and logging for production-ready samples. The Azure SDK for Java has built-in retry with sensible defaults -- quickstarts MAY rely on defaults; production samples SHOULD configure explicitly.

DO:
```java
import com.azure.core.http.policy.RetryOptions;
import com.azure.core.http.policy.ExponentialBackoffOptions;
import com.azure.storage.blob.BlobServiceClient;
import com.azure.storage.blob.BlobServiceClientBuilder;

// Production samples: configure retry explicitly
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

// Quickstarts: SDK defaults are acceptable (3 retries, exponential backoff)
BlobServiceClient quickstartClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildClient();
// Built-in retry policy handles transient failures automatically.
```

DON'T:
```java
// Don't disable retry for production samples
BlobServiceClient blobServiceClient = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .retryOptions(new RetryOptions(new ExponentialBackoffOptions().setMaxRetries(0)))
    .buildClient();
// Disabling retry causes failures on transient network errors
```

> **Note:** Azure SDK Java clients include built-in retry with exponential backoff by default. Quickstarts and simple samples do not need explicit retry configuration. Production-grade samples should document retry settings.

---

## AZ-3: Managed Identity Patterns (HIGH)
**Pattern:** For samples running in Azure, document when to use system-assigned vs user-assigned managed identity.

DO:
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.identity.ManagedIdentityCredentialBuilder;

// For samples: DefaultAzureCredential (works locally + cloud)
var credential = new DefaultAzureCredentialBuilder().build();

// For production: Explicitly use managed identity when deployed
// System-assigned (simpler, auto-managed lifecycle)
var credential = new ManagedIdentityCredentialBuilder().build();

// User-assigned (when multiple identities needed)
var credential = new ManagedIdentityCredentialBuilder()
    .clientId(System.getenv("AZURE_CLIENT_ID"))
    .build();

// Document in README:
// > **Production Deployment:** This sample uses `DefaultAzureCredential`, which will
// > automatically use the system-assigned managed identity when deployed to Azure.
// > Ensure your App Service / Container App / Spring App has a managed identity
// > assigned with appropriate role assignments (e.g., "Storage Blob Data Contributor").
```

DON'T:
```java
// Don't hardcode service principal credentials in samples
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

## AZ-4: Token Management for Non-SDK HTTP (CRITICAL)
**Pattern:** For services without dedicated SDK client support (custom APIs, direct REST calls), get tokens with `getToken()`. The Azure SDK's `TokenCredential` implementations handle token caching and refresh internally -- use `TokenRequestContext` and let the SDK manage expiration.

DO:
```java
import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.core.credential.TokenCredential;
import com.azure.core.credential.TokenRequestContext;
import com.azure.core.credential.AccessToken;

TokenCredential credential = new DefaultAzureCredentialBuilder().build();

// TokenRequestContext for scoped token acquisition
TokenRequestContext tokenContext = new TokenRequestContext()
    .addScopes("https://database.windows.net/.default");

// The SDK caches tokens and refreshes automatically before expiry
AccessToken tokenResponse = credential.getToken(tokenContext).block();

// For long-running operations, re-call getToken() -- the SDK returns
// a cached token if still valid, or refreshes transparently
String token = credential.getToken(tokenContext).block().getToken();

// Use token in JDBC connection...
SQLServerDataSource dataSource = new SQLServerDataSource();
dataSource.setAccessToken(token);
```

DO (When SDK client is NOT available -- direct REST calls):
```java
// For direct HTTP calls to Azure APIs, wrap token acquisition in a helper
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

DON'T:
```java
// CRITICAL: Don't acquire token once and use for hours
AccessToken token = credential.getToken(
    new TokenRequestContext().addScopes("https://database.windows.net/.default")
).block();
String tokenValue = token.getToken();
// ... hours of processing with same tokenValue string (WILL EXPIRE after ~1 hour)

// Don't manually track expiration with custom methods
// The SDK's TokenCredential handles caching and refresh internally
boolean isTokenExpiringSoon(OffsetDateTime expiresAt) {  // Unnecessary
    return OffsetDateTime.now().plusMinutes(5).isAfter(expiresAt);
}
```

**Why:** Azure SDK's `TokenCredential.getToken()` handles caching and proactive refresh internally. Re-call `getToken()` before each use in long-running operations -- the SDK returns the cached token if still valid. Don't extract the token string once and reuse it for hours.

---

## AZ-5: DefaultAzureCredential Configuration (MEDIUM)
**Pattern:** Configure which credential types `DefaultAzureCredential` tries. Exclude interactive browser for CI.

DO:
```java
import com.azure.identity.DefaultAzureCredentialBuilder;

// For CI/CD environments (no interactive prompts)
var credential = new DefaultAzureCredentialBuilder()
    .excludeInteractiveBrowserCredential()
    .build();

// For local development (include IntelliJ/VS Code auth)
var credential = new DefaultAzureCredentialBuilder()
    .excludeSharedTokenCacheCredential()
    .build();

// Document the credential chain in README
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

## AZ-6: Resource Cleanup -- try-with-resources, AutoCloseable (MEDIUM)
**Pattern:** Samples must properly close clients. Use try-with-resources for `Closeable`/`AutoCloseable` clients.

DO:
```java
import com.azure.messaging.servicebus.ServiceBusClientBuilder;
import com.azure.messaging.servicebus.ServiceBusSenderClient;

// Pattern 1: try-with-resources (preferred)
try (ServiceBusSenderClient sender = new ServiceBusClientBuilder()
        .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
        .credential(credential)
        .sender()
        .queueName("myqueue")
        .buildClient()) {

    sender.sendMessage(new ServiceBusMessage("Hello"));
}  // sender.close() called automatically

// Pattern 2: try-with-resources for CosmosClient (implements Closeable)
try (CosmosClient cosmosClient = new CosmosClientBuilder()
        .endpoint(endpoint)
        .credential(credential)
        .buildClient()) {

    CosmosDatabase database = cosmosClient.getDatabase("mydb");
    // ... operations
}  // cosmosClient.close() called automatically
```

DON'T:
```java
// Don't forget to close clients
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();

sender.sendMessage(new ServiceBusMessage("Hello"));
// Client never closed (resource leak, connection pool exhaustion)
```

---

## AZ-7: Pagination with PagedIterable/PagedFlux (HIGH)
**Pattern:** Use `PagedIterable` (sync) or `PagedFlux` (async) for paginated Azure SDK responses. Samples that only process the first page silently lose data.

DO:
```java
import com.azure.storage.blob.BlobContainerClient;
import com.azure.storage.blob.models.BlobItem;
import com.azure.core.http.rest.PagedIterable;

// Blob Storage -- iterate all pages (sync)
BlobContainerClient containerClient = blobServiceClient.getBlobContainerClient("mycontainer");
PagedIterable<BlobItem> blobs = containerClient.listBlobs();
for (BlobItem blob : blobs) {
    System.out.println("Blob: " + blob.getName());
}

// Blob Storage -- iterate all pages (async with Reactor)
blobServiceAsyncClient.getBlobContainerAsyncClient("mycontainer")
    .listBlobs()
    .doOnNext(blob -> System.out.println("Blob: " + blob.getName()))
    .blockLast();  // Only in samples; production uses subscribe()

// Cosmos DB -- iterate query results
CosmosPagedIterable<JsonNode> items = container.queryItems(
    "SELECT * FROM c WHERE c.category = @category",
    new CosmosQueryRequestOptions(),
    JsonNode.class
);
for (JsonNode item : items) {
    System.out.println("Item: " + item.get("name").asText());
}
```

DON'T:
```java
// CRITICAL BUG: Only gets first page
PagedIterable<BlobItem> blobs = containerClient.listBlobs();
List<BlobItem> firstPage = blobs.streamByPage().findFirst()
    .map(page -> page.getElements().stream().toList())
    .orElse(List.of());
// Silently ignores all subsequent pages
```

> **Note:** `PagedResponse<T>` provides both `getElements()` (returns `IterableStream<T>`) and `getValue()` (returns `List<T>`, inherited from `Response<List<T>>`). Prefer `getElements()` for consistency. For most use cases, iterate `PagedIterable` directly with `forEach()` or `stream()` -- it handles pagination automatically.

**Why:** Azure APIs return paginated results. Samples must demonstrate proper pagination or users will silently lose data in production.

---

## AZ-8: Async Client Patterns -- Reactor (Mono/Flux) (HIGH)
**Pattern:** Azure SDK for Java provides `*AsyncClient` variants for all services using Project Reactor (`Mono<T>` and `Flux<T>`). Samples should demonstrate async patterns when the use case benefits from non-blocking I/O.

DO:
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

// Cosmos DB async client
CosmosAsyncClient cosmosAsyncClient = new CosmosClientBuilder()
    .endpoint(config.cosmosEndpoint())
    .credential(credential)
    .buildAsyncClient();

CosmosAsyncContainer container = cosmosAsyncClient
    .getDatabase("mydb")
    .getContainer("mycontainer");

// Async point read with Mono
container.readItem("item-id", new PartitionKey("electronics"), JsonNode.class)
    .subscribe(
        response -> System.out.println("Item: " + response.getItem()),
        error -> System.err.println("Error: " + error.getMessage())
    );

// Async query with Flux
container.queryItems("SELECT * FROM c WHERE c.category = 'electronics'",
        new CosmosQueryRequestOptions(), JsonNode.class)
    .byPage()
    .flatMap(page -> Flux.fromIterable(page.getElements()))
    .doOnNext(item -> System.out.println("Item: " + item.get("name").asText()))
    .blockLast();  // Only in samples; production uses subscribe()

// Blob Storage async
BlobServiceAsyncClient blobAsyncClient = new BlobServiceClientBuilder()
    .endpoint("https://" + accountName + ".blob.core.windows.net")
    .credential(credential)
    .buildAsyncClient();

blobAsyncClient.getBlobContainerAsyncClient("mycontainer")
    .getBlobAsyncClient("sample.txt")
    .downloadContent()
    .subscribe(content -> System.out.println("Content: " + content.toString()));

// Key Vault async
SecretAsyncClient secretAsyncClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildAsyncClient();

secretAsyncClient.getSecret("my-secret")
    .subscribe(secret -> System.out.println("Secret: " + secret.getValue()));
```

DON'T:
```java
// Don't block on every async call (defeats the purpose)
String value = secretAsyncClient.getSecret("my-secret")
    .block()  // Blocking -- use sync client instead
    .getValue();

// Don't ignore errors in subscribe
container.readItem("id", partitionKey, JsonNode.class)
    .subscribe(response -> process(response));  // No error handler
```

> **When to use async:** Use `*AsyncClient` for high-throughput scenarios (batch processing, event-driven), reactive web frameworks (Spring WebFlux), or when composing multiple service calls. Use sync clients for simple samples, CLI tools, and Spring MVC applications.

---

## AZ-9: Logging Configuration -- SLF4J (HIGH)
**Pattern:** Azure SDK for Java uses SLF4J for logging. Configure logging to help users troubleshoot issues. Document log configuration in README.

DO:
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
# Enable Azure SDK verbose logging for troubleshooting
org.slf4j.simpleLogger.log.com.azure=info
# Enable HTTP request/response logging
org.slf4j.simpleLogger.log.com.azure.core.http=debug
```

```xml
<!-- logback.xml (for Logback / Spring Boot) -->
<configuration>
    <logger name="com.azure" level="INFO" />
    <!-- Verbose Azure SDK logging for debugging -->
    <logger name="com.azure.core.http" level="DEBUG" />
    <!-- Identity/credential troubleshooting -->
    <logger name="com.azure.identity" level="DEBUG" />
</configuration>
```

```java
// Enable HTTP logging on client builder
import com.azure.core.http.policy.HttpLogDetailLevel;
import com.azure.core.http.policy.HttpLogOptions;

BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .httpLogOptions(new HttpLogOptions().setLogLevel(HttpLogDetailLevel.BODY_AND_HEADERS))
    .buildClient();
```

DON'T:
```java
// Don't use System.out for logging in production samples
System.out.println("Connecting to storage...");

// Don't ignore SLF4J "no binding" warnings
// SLF4J: No SLF4J providers were found.
// This means no logging implementation is on the classpath
```

> **README guidance:** Document how to enable verbose logging: *"To troubleshoot Azure SDK errors, set the `com.azure` logger to `DEBUG` in your logging configuration."*

---

## AZ-10: HttpClient Selection (MEDIUM)
**Pattern:** Azure SDK for Java supports multiple HTTP client implementations. Document which is in use and when to switch.

DO:
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
// Explicitly set HTTP client (optional -- defaults to auto-detected)
import com.azure.core.http.HttpClient;

BlobServiceClient client = new BlobServiceClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .httpClient(HttpClient.createDefault())  // Uses auto-detected implementation
    .buildClient();
```

> **When to switch:** Use **Netty** (default) for high-throughput server apps. Use **OkHttp** for smaller footprint or Android. Use **JDK HttpClient** to minimize dependencies in Java 11+ projects.
