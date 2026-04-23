# Azure SDK Client Patterns (SDK-1 through SDK-8)

Rules for authentication, credential management, client construction, and resource lifecycle.

## SDK-1: DefaultAzureCredential (CRITICAL)

**Pattern:** Always use `DefaultAzureCredential` from `@azure/identity`. Never use connection strings or API keys in samples.

DO:
```typescript
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();

// Reuse this credential for ALL clients
const blobClient = new BlobServiceClient(
  `https://${accountName}.blob.core.windows.net`,
  credential
);
const secretClient = new SecretClient(vaultUrl, credential);
```

DON'T:
```typescript
const client = BlobServiceClient.fromConnectionString(connStr);
const client = new SecretClient(url, new AzureKeyCredential(apiKey));
```

---

## SDK-2: Credential Caching (HIGH)

**Pattern:** Create ONE `DefaultAzureCredential` instance and pass it to all clients.

DON'T:
```typescript
const blobClient = new BlobServiceClient(url, new DefaultAzureCredential());
const cosmosClient = new CosmosClient({ endpoint, aadCredentials: new DefaultAzureCredential() });
// Two credential instances is wasteful
```

---

## SDK-3: Token Refresh for Long-Running Operations (CRITICAL)

**Pattern:** For operations over 1 hour, use `getBearerTokenProvider` or implement token refresh. `DefaultAzureCredential.getToken()` returns tokens valid ~1 hour.

DO:
```typescript
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';

const credential = new DefaultAzureCredential();
const tokenProvider = getBearerTokenProvider(
  credential,
  'https://cognitiveservices.azure.com/.default'
);

// tokenProvider automatically refreshes on each call
const client = new AzureOpenAI({ azureADTokenProvider: tokenProvider });
```

DON'T:
```typescript
// Token cached once, expires after ~1 hour
const token = await credential.getToken('https://cognitiveservices.azure.com/.default');
const client = new SomeClient({ token: token.token }); // Stale after 1h
```

---

## SDK-4: Client Options (MEDIUM)

**Pattern:** Configure retry policies, timeouts, and user agent where available.

DO:
```typescript
const client = new BlobServiceClient(url, credential, {
  retryOptions: {
    maxRetries: 3,
    retryDelayInMs: 1000,
    maxRetryDelayInMs: 30000,
  },
  userAgentOptions: {
    userAgentPrefix: 'azure-samples/storage-quickstart',
  },
});
```

---

## SDK-5: Managed Identity Documentation (HIGH)

**Pattern:** README should document both system-assigned and user-assigned managed identity patterns for deployed scenarios.

---

## SDK-6: Pagination (HIGH)

**Pattern:** Handle paginated responses completely. Use `for await...of` for async iterators or `fetchAll()`/`fetchNext()` for query results.

DO:
```typescript
// Storage - async iterator
const containerClient = blobServiceClient.getContainerClient('mycontainer');
const blobs = [];
for await (const blob of containerClient.listBlobsFlat()) {
  blobs.push(blob.name);
}

// Cosmos DB - fetchAll
const { resources: items } = await container.items
  .query('SELECT * FROM c')
  .fetchAll();

// Service Bus - receive batches
const receiver = client.createReceiver('myqueue');
let hasMore = true;
while (hasMore) {
  const messages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });
  if (messages.length === 0) { hasMore = false; break; }
  for (const msg of messages) {
    await receiver.completeMessage(msg);
  }
}
```

DON'T:
```typescript
// Only gets first page
for await (const blob of containerClient.listBlobsFlat()) {
  blobs.push(blob);
  if (blobs.length >= 10) break; // May miss thousands
}
```

---

## SDK-7: Resource Cleanup (MEDIUM)

**Pattern:** Close clients that hold connections in `try/finally` blocks.

DO:
```typescript
const serviceBusClient = new ServiceBusClient(ns, credential);
try {
  // ... use client
} finally {
  await serviceBusClient.close();
}
```

---

## SDK-8: Service-Specific Token Scopes (HIGH)

**Pattern:** Each Azure service has its own token scope. Use the correct one.

Common scopes:
- Storage: `https://storage.azure.com/.default`
- Key Vault: `https://vault.azure.net/.default`
- Cosmos DB: `https://{account}.documents.azure.com/.default`
- Azure SQL: `https://database.windows.net/.default`
- Cognitive Services: `https://cognitiveservices.azure.com/.default`
- Service Bus: `https://servicebus.azure.net/.default`
