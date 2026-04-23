# Messaging Services & Key Vault

Service Bus, Event Hubs, and Key Vault patterns.

## MSG-1: Service Bus Patterns (HIGH)

Use `Azure.Messaging.ServiceBus` with `DefaultAzureCredential`. Handle message completion/abandonment.

DO:
```csharp
using Azure.Messaging.ServiceBus;
using Azure.Identity;

var credential = new DefaultAzureCredential();
await using var client = new ServiceBusClient(
    $"{namespaceName}.servicebus.windows.net", credential);

// Send messages
await using var sender = client.CreateSender("myqueue");
var messages = new List<ServiceBusMessage>
{
    new(BinaryData.FromObjectAsJson(new { OrderId = 1, Amount = 100 })),
    new(BinaryData.FromObjectAsJson(new { OrderId = 2, Amount = 200 })),
};
await sender.SendMessagesAsync(messages);

// Receive messages (MUST complete or abandon)
await using var receiver = client.CreateReceiver("myqueue");
var receivedMessages = await receiver.ReceiveMessagesAsync(maxMessages: 10,
    maxWaitTime: TimeSpan.FromSeconds(5));

foreach (var message in receivedMessages)
{
    try
    {
        Console.WriteLine($"Received: {message.Body}");
        await receiver.CompleteMessageAsync(message);
    }
    catch (Exception)
    {
        await receiver.AbandonMessageAsync(message);
    }
}
```

DON'T:
```csharp
// Don't use connection strings in samples
var client = new ServiceBusClient(connectionString);

// Don't forget to complete/abandon messages
foreach (var message in receivedMessages)
{
    Console.WriteLine(message.Body);  // Message never completed (will reappear)
}
```

---

## MSG-2: Event Hubs Patterns (MEDIUM)

Use `Azure.Messaging.EventHubs` for ingestion, `EventProcessorClient` for processing with checkpoint management.

DO:
```csharp
using Azure.Messaging.EventHubs;
using Azure.Messaging.EventHubs.Producer;
using Azure.Messaging.EventHubs.Processor;
using Azure.Storage.Blobs;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// Send events
await using var producer = new EventHubProducerClient(
    $"{namespaceName}.servicebus.windows.net", "myeventhub", credential);

using EventDataBatch batch = await producer.CreateBatchAsync();
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(new { Temperature = 23.5 })));
batch.TryAdd(new EventData(BinaryData.FromObjectAsJson(new { Temperature = 24.1 })));
await producer.SendAsync(batch);

// Process events with checkpoint store
var storageClient = new BlobContainerClient(
    new Uri($"https://{storageAccount}.blob.core.windows.net/eventhub-checkpoints"),
    credential);

var processor = new EventProcessorClient(
    storageClient, "$Default",
    $"{namespaceName}.servicebus.windows.net", "myeventhub", credential);

processor.ProcessEventAsync += async args =>
{
    Console.WriteLine($"Received: {args.Data.EventBody}");
    await args.UpdateCheckpointAsync();
};

processor.ProcessErrorAsync += args =>
{
    Console.WriteLine($"Error: {args.Exception.Message}");
    return Task.CompletedTask;
};

await processor.StartProcessingAsync();
```

---

## KV-1: Key Vault Client Patterns (HIGH)

Use `Azure.Security.KeyVault.Secrets`, `Azure.Security.KeyVault.Keys`, `Azure.Security.KeyVault.Certificates` with `DefaultAzureCredential`.

DO:
```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Security.KeyVault.Keys;
using Azure.Security.KeyVault.Certificates;
using Azure.Identity;

var credential = new DefaultAzureCredential();

// Secrets
var secretClient = new SecretClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

await secretClient.SetSecretAsync("db-password", "P@ssw0rd123");
KeyVaultSecret secret = await secretClient.GetSecretAsync("db-password");
Console.WriteLine($"Secret value: {secret.Value}");

// Keys (for encryption)
var keyClient = new KeyClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

KeyVaultKey key = await keyClient.CreateKeyAsync("my-encryption-key", KeyType.Rsa);
Console.WriteLine($"Key ID: {key.Id}");

// Certificates
var certClient = new CertificateClient(
    new Uri(config["Azure:KeyVaultUrl"]!), credential);

KeyVaultCertificateWithPolicy cert = await certClient.GetCertificateAsync("my-cert");
Console.WriteLine($"Certificate thumbprint: {Convert.ToHexString(cert.Properties.X509Thumbprint)}");
```

DON'T:
```csharp
// Don't hardcode secrets in samples
string dbPassword = "P@ssw0rd123";  // Use Key Vault
```
