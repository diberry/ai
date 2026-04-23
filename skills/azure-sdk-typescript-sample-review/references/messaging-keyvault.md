# Messaging and Key Vault (MSG-1 through MSG-2, KV-1)

Rules for Service Bus, Event Hubs, and Key Vault patterns.

## MSG-1: Service Bus Patterns (HIGH)

**Pattern:** Use `@azure/service-bus` with `DefaultAzureCredential`. Complete or abandon every received message.

DO:
```typescript
import { ServiceBusClient, ServiceBusMessage } from '@azure/service-bus';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const serviceBusClient = new ServiceBusClient(
  `${namespace}.servicebus.windows.net`, credential
);

// Send
const sender = serviceBusClient.createSender('myqueue');
await sender.sendMessages([{ body: { orderId: 1 } }]);
await sender.close();

// Receive (MUST complete or abandon)
const receiver = serviceBusClient.createReceiver('myqueue');
const messages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });

for (const message of messages) {
  try {
    console.log(`Received: ${JSON.stringify(message.body)}`);
    await receiver.completeMessage(message); // Mark processed
  } catch (err) {
    await receiver.abandonMessage(message); // Return to queue
  }
}

await receiver.close();
await serviceBusClient.close();
```

DON'T:
```typescript
const client = ServiceBusClient.fromConnectionString(connStr); // Use AAD
for (const msg of messages) {
  console.log(msg.body); // Never completed, message reappears
}
```

---

## MSG-2: Event Hubs Patterns (MEDIUM)

**Pattern:** Use `@azure/event-hubs` for ingestion. Use `@azure/event-hubs-checkpointstore-blob` for consumer checkpoint management.

DO:
```typescript
import { EventHubProducerClient, EventHubConsumerClient } from '@azure/event-hubs';
import { BlobCheckpointStore } from '@azure/event-hubs-checkpointstore-blob';
import { ContainerClient } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// Send
const producer = new EventHubProducerClient(
  `${namespace}.servicebus.windows.net`, 'myeventhub', credential
);
const batch = await producer.createBatch();
batch.tryAdd({ body: { temperature: 23.5 } });
await producer.sendBatch(batch);
await producer.close();

// Receive with checkpoint store
const containerClient = new ContainerClient(
  `https://${storageAccount}.blob.core.windows.net/eventhub-checkpoints`, credential
);
const checkpointStore = new BlobCheckpointStore(containerClient);
const consumer = new EventHubConsumerClient(
  '$Default', `${namespace}.servicebus.windows.net`, 'myeventhub', credential, checkpointStore
);

consumer.subscribe({
  processEvents: async (events, context) => {
    for (const event of events) { console.log(event.body); }
    await context.updateCheckpoint(events[events.length - 1]);
  },
  processError: async (err) => { console.error(err.message); },
});
```

---

## KV-1: Key Vault Client Patterns (HIGH)

**Pattern:** Use `@azure/keyvault-secrets`, `@azure/keyvault-keys`, `@azure/keyvault-certificates` with `DefaultAzureCredential`.

DO:
```typescript
import { SecretClient } from '@azure/keyvault-secrets';
import { KeyClient } from '@azure/keyvault-keys';
import { CertificateClient } from '@azure/keyvault-certificates';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

const secretClient = new SecretClient(config.AZURE_KEYVAULT_URL, credential);
await secretClient.setSecret('db-password', 'P@ssw0rd123');
const secret = await secretClient.getSecret('db-password');

const keyClient = new KeyClient(config.AZURE_KEYVAULT_URL, credential);
const key = await keyClient.createKey('my-encryption-key', 'RSA');

const certClient = new CertificateClient(config.AZURE_KEYVAULT_URL, credential);
const cert = await certClient.getCertificate('my-cert');
```

DON'T:
```typescript
const dbPassword = 'P@ssw0rd123'; // Hardcoded secret, use Key Vault
```
