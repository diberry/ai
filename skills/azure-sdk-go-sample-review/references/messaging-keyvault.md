# Messaging Services & Key Vault

Messaging patterns for queues, topics, and event ingestion. Secure secrets storage and retrieval using Azure Key Vault with AAD authentication.

## MSG-1: Service Bus--azservicebus Patterns (HIGH)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus` with `DefaultAzureCredential`.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azservicebus"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azservicebus.NewClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace), cred, nil,
)
if err != nil {
    return fmt.Errorf("creating service bus client: %w", err)
}
defer client.Close(ctx)

// Send messages
sender, err := client.NewSender("myqueue", nil)
if err != nil {
    return fmt.Errorf("creating sender: %w", err)
}
defer sender.Close(ctx)

batch, err := sender.NewMessageBatch(ctx, nil)
if err != nil {
    return fmt.Errorf("creating message batch: %w", err)
}

err = batch.AddMessage(&azservicebus.Message{Body: []byte(`{"orderId":1}`)}, nil)
if err != nil {
    return fmt.Errorf("adding message: %w", err)
}

err = sender.SendMessageBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("sending batch: %w", err)
}

// Receive messages (MUST complete or abandon)
receiver, err := client.NewReceiverForQueue("myqueue", nil)
if err != nil {
    return fmt.Errorf("creating receiver: %w", err)
}
defer receiver.Close(ctx)

messages, err := receiver.ReceiveMessages(ctx, 10, nil)
if err != nil {
    return fmt.Errorf("receiving messages: %w", err)
}

for _, msg := range messages {
    fmt.Printf("Received: %s\n", string(msg.Body))
    if err := receiver.CompleteMessage(ctx, msg, nil); err != nil {
        // Abandon on failure--message returns to queue
        _ = receiver.AbandonMessage(ctx, msg, nil)
        return fmt.Errorf("completing message: %w", err)
    }
}
```

**DON'T:**
```go
// Don't use connection strings in samples
client, err := azservicebus.NewClientFromConnectionString(connStr, nil)

// Don't forget to complete/abandon messages
for _, msg := range messages {
    fmt.Println(string(msg.Body))  // Message never completed (will reappear)
}
```

---

## MSG-2: Event Hubs--azeventhubs Patterns (MEDIUM)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/azeventhubs` for event ingestion.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/azeventhubs"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// Producer--send events
producerClient, err := azeventhubs.NewProducerClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace),
    "myeventhub",
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating producer: %w", err)
}
defer producerClient.Close(ctx)

batch, err := producerClient.NewEventDataBatch(ctx, nil)
if err != nil {
    return fmt.Errorf("creating batch: %w", err)
}

err = batch.AddEventData(&azeventhubs.EventData{
    Body: []byte(`{"temperature": 23.5}`),
}, nil)
if err != nil {
    return fmt.Errorf("adding event: %w", err)
}

err = producerClient.SendEventDataBatch(ctx, batch, nil)
if err != nil {
    return fmt.Errorf("sending batch: %w", err)
}

// Consumer--receive events with processor
consumerClient, err := azeventhubs.NewConsumerClient(
    fmt.Sprintf("%s.servicebus.windows.net", namespace),
    "myeventhub",
    azeventhubs.DefaultConsumerGroup,
    cred,
    nil,
)
if err != nil {
    return fmt.Errorf("creating consumer: %w", err)
}
defer consumerClient.Close(ctx)
```

---

## MSG-3: Event Grid--azeventgrid Patterns (MEDIUM)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/messaging/eventgrid/azeventgrid` for publishing events to Event Grid topics.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/messaging/eventgrid/azeventgrid"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

client, err := azeventgrid.NewClient(config.EventGridEndpoint, cred, nil)
if err != nil {
    return fmt.Errorf("creating event grid client: %w", err)
}
```

> **Note:** Verify exact package path on pkg.go.dev--the Event Grid Go SDK may be in preview.

---

## KV-1: Key Vault Client Patterns (HIGH)

**Pattern:** Use `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets`, `azkeys`, `azcertificates` with `DefaultAzureCredential`.

**DO:**
```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azkeys"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azcertificates"
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
)

cred, err := azidentity.NewDefaultAzureCredential(nil)
if err != nil {
    return fmt.Errorf("creating credential: %w", err)
}

// Secrets
secretClient, err := azsecrets.NewClient(config.KeyVaultURL, cred, nil)
if err != nil {
    return fmt.Errorf("creating secret client: %w", err)
}

// Set a secret
_, err = secretClient.SetSecret(ctx, "db-password", azsecrets.SetSecretParameters{
    Value: to.Ptr("P@ssw0rd123"),
}, nil)
if err != nil {
    return fmt.Errorf("setting secret: %w", err)
}

// Get a secret
resp, err := secretClient.GetSecret(ctx, "db-password", "", nil)
if err != nil {
    return fmt.Errorf("getting secret: %w", err)
}
fmt.Printf("Secret value: %s\n", *resp.Value)

// Keys (for encryption)
keyClient, err := azkeys.NewClient(config.KeyVaultURL, cred, nil)
if err != nil {
    return fmt.Errorf("creating key client: %w", err)
}

createResp, err := keyClient.CreateKey(ctx, "my-encryption-key", azkeys.CreateKeyParameters{
    Kty: to.Ptr(azkeys.KeyTypeRSA),
}, nil)
if err != nil {
    return fmt.Errorf("creating key: %w", err)
}
fmt.Printf("Key ID: %s\n", *createResp.Key.KID)
```

**DON'T:**
```go
// Don't hardcode secrets in samples
dbPassword := "P@ssw0rd123"  // Use Key Vault
```
