# Project Setup & Configuration

**What this section covers:** Build system configuration, Java version, dependency management, environment variables, and compiler settings. These foundational patterns ensure samples build correctly and run reliably across environments.

## PS-1: Java Version (HIGH)
**Pattern:** Use Java 17 or 21 LTS. Configure source/target compatibility in build files.

DO:
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

DON'T:
```xml
<!-- Java 8 or 11 (not current LTS for new samples) -->
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
</properties>
```

**Why:** Java 17 and 21 are current LTS releases with modern language features (records, sealed classes, pattern matching). New Azure SDK samples should target these versions.

---

## PS-2: Maven/Gradle Project Metadata (MEDIUM)
**Pattern:** All sample projects must include complete metadata for discoverability and maintenance.

DO:
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

DON'T:
```xml
<project>
    <groupId>com.example</groupId>
    <artifactId>my-sample</artifactId>
    <version>1.0.0</version>
    <!-- Missing name, description, license, developer info -->
</project>
```

**Note:** The `name` and `description` fields make samples discoverable. Include license metadata.

---

## PS-3: Dependency Audit (CRITICAL)
**Pattern:** Every dependency must be imported somewhere. No phantom dependencies. Use current Azure SDK Track 2 packages (`com.azure:*`).

DO:
```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>       <!-- Track 2, used in BlobService.java -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>            <!-- Used for auth -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-security-keyvault-secrets</artifactId> <!-- Used in SecretsManager.java -->
    </dependency>
</dependencies>
```

DON'T:
```xml
<dependencies>
    <dependency>
        <groupId>com.microsoft.azure</groupId>
        <artifactId>azure-storage</artifactId>             <!-- Track 1 (legacy) -->
        <version>8.6.6</version>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-messaging-servicebus</artifactId> <!-- Listed but never imported -->
    </dependency>
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>                       <!-- Not imported anywhere -->
    </dependency>
</dependencies>
```

**Why:** Phantom dependencies bloat the project and confuse users trying to learn which packages are needed.

---

## PS-4: Azure SDK Package Naming (HIGH)
**Pattern:** Use Track 2 packages (`com.azure:*`) not Track 1 legacy packages (`com.microsoft.azure:*`).

DO (Track 2):
```java
// Current generation Azure SDK packages
import com.azure.storage.blob.BlobServiceClient;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.messaging.servicebus.ServiceBusClientBuilder;
import com.azure.cosmos.CosmosClient;
import com.azure.data.tables.TableClient;
import com.azure.identity.DefaultAzureCredentialBuilder;

// Azure OpenAI via com.azure:azure-ai-openai
import com.azure.ai.openai.OpenAIClient;
```

DON'T (Track 1 Legacy):
```java
// Track 1 packages (legacy, avoid in new samples)
import com.microsoft.azure.storage.CloudStorageAccount;         // Use com.azure:azure-storage-blob
import com.microsoft.azure.keyvault.KeyVaultClient;             // Use com.azure:azure-security-keyvault-secrets
import com.microsoft.azure.servicebus.QueueClient;              // Use com.azure:azure-messaging-servicebus
```

**Why:** Track 2 SDKs (`com.azure:*`) are current generation with consistent APIs, built-in HTTP pipeline, and active maintenance. Track 1 (`com.microsoft.azure:*`) is legacy.

---

## PS-5: Configuration (MEDIUM)
**Pattern:** Use `application.properties` or `application.yml` for Spring Boot, or environment variables with validation for standalone apps. Never hardcode configuration.

DO (Standalone):
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
        // Use HashMap -- Map.of() throws NullPointerException on null values
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

> **Java Map.of() and null values:** `Map.of("key", null)` throws `NullPointerException` at map creation time. Since `System.getenv()` returns `null` for unset variables, always use `HashMap` when values may be null.
>
> **.env files are not native to Java.** Unlike Node.js, Java does not natively load `.env` files. Recommend `System.getenv()` for standalone apps or `application.properties` / `application.yml` for Spring Boot. If a sample uses `.env` files, note that the `io.github.cdimascio:java-dotenv` library is required.

DO (Spring Boot):
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

DON'T:
```java
// Don't hardcode configuration
String storageAccountName = "contosoprod";

// Don't silently fall back to defaults
String storageAccount = Optional.ofNullable(System.getenv("AZURE_STORAGE_ACCOUNT_NAME"))
    .orElse("devstoreaccount1");
```

---

## PS-6: Compiler Warnings (MEDIUM)
**Pattern:** Enable comprehensive compiler warnings. Address all warnings before publication.

DO:
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

DON'T:
```java
// Suppress warnings without justification
@SuppressWarnings("unchecked")
public void processData(Object data) {
    List<String> items = (List<String>) data; // No explanation for suppression
}
```

**Why:** `-Xlint:all` catches deprecation warnings, unchecked casts, and other common bugs at compile time.

---

## PS-7: .editorconfig (LOW)
**Pattern:** Include `.editorconfig` for consistent formatting across editors.

DO:
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

## PS-8: Lock File / Dependency Tree (HIGH)
**Pattern:** Use Maven Wrapper or Gradle Wrapper for reproducible builds. Commit wrapper files.

DO:
```
repo/
├── mvnw                    # Maven Wrapper script (Unix)
├── mvnw.cmd                # Maven Wrapper script (Windows)
├── .mvn/
│   └── wrapper/
│       └── maven-wrapper.properties  # Wrapper config
├── pom.xml
```

```bash
# Generate Maven Wrapper
mvn wrapper:wrapper -Dmaven=3.9.9

# Verify dependency tree
mvn dependency:tree
```

DON'T:
```
repo/
├── pom.xml                 # No wrapper, depends on user's Maven installation
```

```
repo/
├── pom.xml                 # Mixed build tools
├── build.gradle            # Don't mix Maven and Gradle
```

**Why:** Wrapper scripts ensure every developer and CI system uses the same build tool version. Mixed build tools cause confusion and version conflicts.

---

## PS-9: CVE Scanning (CRITICAL)
**Pattern:** Samples must not ship with known security vulnerabilities. Dependency checks must pass with no critical/high issues.

DO:
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

DON'T:
```bash
# Don't ignore vulnerability warnings
mvn dependency-check:check
# Found 3 high severity vulnerabilities
# Submitting sample anyway
```

**Why:** Known CVEs expose users to security risks. All Azure samples must pass security scans.

---

## PS-10: Package Legitimacy Check (MEDIUM)
**Pattern:** Verify Azure SDK packages are from official `com.azure` group. Watch for typosquatting.

DO:
```xml
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>                       <!-- Official group -->
        <artifactId>azure-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>                       <!-- Official group -->
        <artifactId>azure-identity</artifactId>
    </dependency>
</dependencies>
```

DON'T:
```xml
<dependencies>
    <dependency>
        <groupId>com.azurre</groupId>                      <!-- Typosquatting (azurre) -->
        <artifactId>azure-storage-blob</artifactId>
    </dependency>
    <dependency>
        <groupId>io.azure</groupId>                        <!-- Not official group -->
        <artifactId>storage-blob</artifactId>
    </dependency>
</dependencies>
```

**Check:** All Azure SDK packages should use groupId `com.azure` and be published by Microsoft. Verify on [Maven Central](https://central.sonatype.com/namespace/com.azure).

---

## PS-11: BOM Version Management (MEDIUM)
**Pattern:** Use `azure-sdk-bom` to manage Azure SDK dependency versions centrally. Avoid specifying versions on individual Azure SDK dependencies.

DO:
```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.azure</groupId>
            <artifactId>azure-sdk-bom</artifactId>
            <version>1.2.29</version>                      <!-- Single version to manage -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>        <!-- No version needed -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>            <!-- No version needed -->
    </dependency>
</dependencies>
```

```groovy
// build.gradle
dependencies {
    implementation platform('com.azure:azure-sdk-bom:1.2.29')
    implementation 'com.azure:azure-storage-blob'          // No version needed
    implementation 'com.azure:azure-identity'              // No version needed
}
```

DON'T:
```xml
<dependencies>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-storage-blob</artifactId>
        <version>12.28.1</version>                         <!-- Manual version, may conflict -->
    </dependency>
    <dependency>
        <groupId>com.azure</groupId>
        <artifactId>azure-identity</artifactId>
        <version>1.14.2</version>                          <!-- Manual version, may conflict -->
    </dependency>
</dependencies>
```

**Why:** The BOM ensures all Azure SDK packages use compatible versions. Manual version pinning risks incompatible transitive dependencies.
