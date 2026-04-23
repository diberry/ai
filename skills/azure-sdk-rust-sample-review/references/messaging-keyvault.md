# Messaging Services & Key Vault

Rules MSG-1, MSG-2, and KV-1. Service Bus, Event Hubs, and Key Vault patterns for Rust.

## MSG-1: Service Bus Patterns (LOW)

Use the official Service Bus crate if available on crates.io (check for `azure_messaging_servicebus` or similar), or REST API via `azure_core`.

DO:
```rust
// Check crates.io for current Service Bus crate name and availability
// The example below uses a hypothetical API -- verify against actual crate docs
use azure_messaging_servicebus::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

let client = ServiceBusClient::new(
    &format!("{}.servicebus.windows.net", namespace),
    credential.clone(),
);

// Send message
let sender = client.create_sender("myqueue")?;
sender.send_message("Hello from Rust!").await?;

// Receive and complete messages
let receiver = client.create_receiver("myqueue")?;
let messages = receiver.receive_messages(10).await?;

for message in &messages {
    println!("Received: {:?}", message.body());
    receiver.complete_message(message).await?;  // Mark as processed
}
```

DON'T:
```rust
// Don't forget to complete/abandon messages
for message in &messages {
    println!("{:?}", message.body());
    // Message never completed -- will reappear in queue
}
```

> Preview Note: Check crates.io for `azure_messaging_servicebus` availability. If not available, use the REST API with `azure_core` HTTP client and AAD tokens.

---

## MSG-2: Event Hubs Patterns (LOW)

Check crates.io for the Event Hubs crate. Always use AAD authentication and handle partitioned event streams correctly.

DO:
```rust
// Check crates.io for current Event Hubs crate name and availability
use azure_messaging_event_hubs::producer::ProducerClient;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);
let namespace = env::var("EVENTHUB_NAMESPACE")?;
let hub_name = env::var("EVENTHUB_NAME")?;

let producer = ProducerClient::new(
    format!("{}.servicebus.windows.net", namespace),
    hub_name,
    credential.clone(),
    None,
)?;

// Create batch and send events
let mut batch = producer.create_batch(None).await?;
batch.try_add_event_data("event-payload-1")?;
batch.try_add_event_data("event-payload-2")?;
producer.send_batch(batch).await?;
```

DON'T:
```rust
// Don't use connection strings in code
let producer = ProducerClient::from_connection_string(
    "Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=...",
    hub_name,
)?;

// Don't send events one at a time -- use batching
for payload in &payloads {
    producer.send_event(payload).await?;  // Inefficient, no batching
}
```

---

## KV-1: Key Vault Client Patterns (HIGH)

Use `azure_security_keyvault_secrets` crate with `DefaultAzureCredential`.

DO:
```rust
use azure_security_keyvault_secrets::prelude::*;
use azure_identity::DefaultAzureCredential;
use std::sync::Arc;

let credential = Arc::new(DefaultAzureCredential::new()?);

let secret_client = SecretClient::new(
    &config.keyvault_url,
    credential.clone(),
    None,
)?;

// Set a secret
secret_client.set("db-password", "P@ssw0rd123").await?;

// Get a secret
let secret = secret_client.get("db-password").await?;
println!("Secret value: {}", secret.value);
```

DON'T:
```rust
// Don't hardcode secrets in samples
let db_password = "P@ssw0rd123";  // Use Key Vault

// Don't log secret values in production code
println!("Secret: {}", secret.value);  // OK in samples, not production
```

> Preview Note: Check crates.io for `azure_security_keyvault_secrets` availability and current API.
