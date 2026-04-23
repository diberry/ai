# Messaging Services & Key Vault

**What this section covers:** Messaging patterns for queues, topics, event ingestion, checkpoint management, and proper resource cleanup. Secure secrets storage and retrieval using Azure Key Vault.

## MSG-1: Service Bus Patterns (HIGH)
**Pattern:** Use `com.azure:azure-messaging-servicebus` with `DefaultAzureCredential`. Complete or abandon messages.

DO:
```java
import com.azure.messaging.servicebus.*;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// Send messages
try (ServiceBusSenderClient sender = new ServiceBusClientBuilder()
        .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
        .credential(credential)
        .sender()
        .queueName("myqueue")
        .buildClient()) {

    sender.sendMessage(new ServiceBusMessage("{\"orderId\": 1, \"amount\": 100}"));
}

// Receive messages with processor (recommended for production)
ServiceBusProcessorClient processor = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace + ".servicebus.windows.net")
    .credential(credential)
    .processor()
    .queueName("myqueue")
    .processMessage(context -> {
        System.out.println("Received: " + context.getMessage().getBody().toString());
        // Auto-completed by processor in PEEK_LOCK mode
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

DON'T:
```java
// Don't use connection strings in samples
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .connectionString(connectionString)  // Use credential-based auth
    .sender()
    .queueName("myqueue")
    .buildClient();

// Don't forget to close clients
ServiceBusSenderClient sender = new ServiceBusClientBuilder()
    .fullyQualifiedNamespace(namespace)
    .credential(credential)
    .sender()
    .queueName("myqueue")
    .buildClient();
sender.sendMessage(new ServiceBusMessage("Hello"));
// Client never closed
```

---

## MSG-2: Event Hubs Patterns (MEDIUM)
**Pattern:** Use `com.azure:azure-messaging-eventhubs` for ingestion, `com.azure:azure-messaging-eventhubs-checkpointstore-blob` for processing with checkpoint management.

DO:
```java
import com.azure.messaging.eventhubs.*;
import com.azure.messaging.eventhubs.checkpointstore.blob.BlobCheckpointStore;
import com.azure.storage.blob.BlobContainerAsyncClient;
import com.azure.storage.blob.BlobContainerClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// Send events
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

// Receive events with checkpoint store
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
        context.updateCheckpoint();  // Checkpoint after processing
    })
    .processError(context -> {
        System.err.println("Error: " + context.getThrowable().getMessage());
    })
    .buildEventProcessorClient();

processor.start();
```

---

## KV-1: Key Vault Client Patterns (HIGH)
**Pattern:** Use `com.azure:azure-security-keyvault-secrets`, `com.azure:azure-security-keyvault-keys`, `com.azure:azure-security-keyvault-certificates` with `DefaultAzureCredential`.

DO:
```java
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.azure.security.keyvault.keys.KeyClient;
import com.azure.security.keyvault.keys.KeyClientBuilder;
import com.azure.security.keyvault.certificates.CertificateClient;
import com.azure.security.keyvault.certificates.CertificateClientBuilder;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// Secrets
SecretClient secretClient = new SecretClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

secretClient.setSecret("db-password", "P@ssw0rd123");
String secretValue = secretClient.getSecret("db-password").getValue();
System.out.println("Secret value: " + secretValue);

// Keys (for encryption)
KeyClient keyClient = new KeyClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

var key = keyClient.createRsaKey(new CreateRsaKeyOptions("my-encryption-key"));
System.out.println("Key ID: " + key.getId());

// Certificates
CertificateClient certClient = new CertificateClientBuilder()
    .vaultUrl(config.keyVaultUrl())
    .credential(credential)
    .buildClient();

var cert = certClient.getCertificate("my-cert");
System.out.println("Certificate name: " + cert.getName());
```

DON'T:
```java
// Don't hardcode secrets in samples
String dbPassword = "P@ssw0rd123";  // Use Key Vault

// Don't use Key Vault for every sample (adds complexity)
// Use Key Vault when demonstrating secret management or when the sample
// explicitly covers secure configuration patterns.
```
