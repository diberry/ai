# Messaging Services & Key Vault

Messaging patterns for queues, topics, event ingestion, and event-driven architectures. Secure secrets storage and retrieval using Azure Key Vault.

## MSG-1: Service Bus Patterns (HIGH)

Use `azure-servicebus` with `DefaultAzureCredential`. Handle message completion, dead-letter queues.

DO:
```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

with ServiceBusClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    credential=credential,
) as client:
    # Send messages
    with client.get_queue_sender(queue_name="myqueue") as sender:
        messages = [
            ServiceBusMessage(body='{"orderId": 1, "amount": 100}'),
            ServiceBusMessage(body='{"orderId": 2, "amount": 200}'),
        ]
        sender.send_messages(messages)

    # Receive messages (MUST complete or abandon)
    with client.get_queue_receiver(queue_name="myqueue") as receiver:
        received_messages = receiver.receive_messages(
            max_message_count=10,
            max_wait_time=5,
        )
        for message in received_messages:
            try:
                print(f"Received: {str(message)}")
                receiver.complete_message(message)  # Mark as processed
            except Exception:
                receiver.abandon_message(message)  # Return to queue
```

DON'T:
```python
# Don't use connection strings in samples
client = ServiceBusClient.from_connection_string(connection_string)

# Don't forget to complete/abandon messages
for message in received_messages:
    print(str(message))  # Message never completed (will reappear)
```

---

## MSG-2: Event Hubs Patterns (MEDIUM)

Use `azure-eventhub` for ingestion, `azure-eventhub-checkpointstoreblob-aio` for async processing with checkpoint management.

DO:
```python
from azure.eventhub import EventHubProducerClient, EventData
from azure.eventhub.aio import EventHubConsumerClient
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Send events (sync)
producer = EventHubProducerClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    eventhub_name="myeventhub",
    credential=credential,
)

with producer:
    batch = producer.create_batch()
    batch.add(EventData('{"temperature": 23.5}'))
    batch.add(EventData('{"temperature": 24.1}'))
    producer.send_batch(batch)

# Receive events with checkpoint store (async)
checkpoint_store = BlobCheckpointStore(
    blob_account_url=f"https://{storage_account}.blob.core.windows.net",
    container_name="eventhub-checkpoints",
    credential=credential,
)

consumer = EventHubConsumerClient(
    fully_qualified_namespace=f"{namespace}.servicebus.windows.net",
    consumer_group="$Default",
    eventhub_name="myeventhub",
    credential=credential,
    checkpoint_store=checkpoint_store,
)


async def on_event(partition_context, event):
    print(f"Received: {event.body_as_str()}")
    await partition_context.update_checkpoint(event)


async with consumer:
    await consumer.receive(on_event=on_event)
```

---

## KV-1: Key Vault Client Patterns (HIGH)

Use `azure-keyvault-secrets`, `azure-keyvault-keys`, `azure-keyvault-certificates` with `DefaultAzureCredential`.

DO:
```python
from azure.keyvault.secrets import SecretClient
from azure.keyvault.keys import KeyClient
from azure.keyvault.certificates import CertificateClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()

# Secrets
secret_client = SecretClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)

secret_client.set_secret("db-password", "P@ssw0rd123")
secret = secret_client.get_secret("db-password")
print(f"Secret value: {secret.value}")

# Keys (for encryption)
key_client = KeyClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)
key = key_client.create_rsa_key("my-encryption-key")
print(f"Key ID: {key.id}")

# Certificates
cert_client = CertificateClient(
    vault_url=config["AZURE_KEYVAULT_URL"],
    credential=credential,
)
cert = cert_client.get_certificate("my-cert")
print(f"Certificate thumbprint: {cert.properties.x509_thumbprint}")
```

DON'T:
```python
# Don't hardcode secrets in samples
db_password = "P@ssw0rd123"  # Use Key Vault

# Don't use Key Vault for every sample (adds complexity)
# Use Key Vault when demonstrating secret management or when the sample
# explicitly covers secure configuration patterns.
```
