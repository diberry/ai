# AI Services and Vector Search (AI-1 through AI-4, VEC-1 through VEC-2)

Rules for Azure OpenAI, Document Intelligence, Speech, and vector search patterns.

## AI-1: Azure OpenAI Client Configuration (HIGH)

**Pattern:** Use `openai` SDK's `AzureOpenAI` class (not retired `@azure/openai`). Configure `timeout`, `maxRetries`, and `azureADTokenProvider`.

DO:
```typescript
import { AzureOpenAI } from 'openai';
import { DefaultAzureCredential, getBearerTokenProvider } from '@azure/identity';

const credential = new DefaultAzureCredential();
const azureADTokenProvider = getBearerTokenProvider(
  credential,
  'https://cognitiveservices.azure.com/.default'
);

const client = new AzureOpenAI({
  azureADTokenProvider,
  endpoint: config.AZURE_OPENAI_ENDPOINT,
  apiVersion: '2024-10-21',
  timeout: 30_000,
  maxRetries: 3,
});

// Chat completion
const response = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello!' }],
});

// Embeddings
const embeddingResponse = await client.embeddings.create({
  model: 'text-embedding-3-small',
  input: 'Sample text to embed',
});
```

DON'T:
```typescript
import { OpenAIClient, AzureKeyCredential } from '@azure/openai'; // Retired
const client = new AzureOpenAI({ apiKey: process.env.AZURE_OPENAI_API_KEY }); // Use AAD
```

---

## AI-1b: Streaming Chat Completions (HIGH)

**Pattern:** Use `stream: true` and iterate with `for await...of`.

DO:
```typescript
const stream = await client.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Explain quantum computing' }],
  stream: true,
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) process.stdout.write(content);
}

// With abort support
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 30_000);
try {
  const stream = await client.chat.completions.create(
    { model: 'gpt-4o', messages: [...], stream: true },
    { signal: controller.signal }
  );
  for await (const chunk of stream) { /* ... */ }
} finally {
  clearTimeout(timeout);
}
```

DON'T:
```typescript
// Collecting entire stream defeats the purpose
const chunks: string[] = [];
for await (const chunk of stream) {
  chunks.push(chunk.choices[0]?.delta?.content ?? '');
}
const fullResponse = chunks.join('');
```

---

## AI-2: API Version Documentation (LOW)

**Pattern:** Hardcoded API versions should include a comment linking to version docs.

DO:
```typescript
const client = new AzureOpenAI({
  apiVersion: '2024-10-21',
  // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
});
```

---

## AI-3: Document Intelligence and Speech SDK (MEDIUM)

**Pattern:** Use `@azure/ai-document-intelligence` with `DefaultAzureCredential`. For Speech, use `microsoft-cognitiveservices-speech-sdk`.

DO:
```typescript
import { DocumentIntelligenceClient } from '@azure/ai-document-intelligence';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const docClient = new DocumentIntelligenceClient(
  config.DOCUMENT_INTELLIGENCE_ENDPOINT,
  credential
);

const poller = await docClient.beginAnalyzeDocument('prebuilt-invoice', documentStream);
const result = await poller.pollUntilDone();

// Speech SDK (uses subscription key, AAD support limited)
import * as sdk from 'microsoft-cognitiveservices-speech-sdk';
const speechConfig = sdk.SpeechConfig.fromSubscription(config.SPEECH_KEY, config.SPEECH_REGION);
```

---

## AI-4: Vector Dimension Validation (MEDIUM)

**Pattern:** Embeddings must match declared vector column dimension. Validate at runtime.

DO:
```typescript
const EMBEDDING_MODEL = 'text-embedding-3-small'; // 1536 dimensions
const VECTOR_DIMENSION = 1536;

const embedding = await getEmbedding(text);
if (embedding.length !== VECTOR_DIMENSION) {
  throw new Error(
    `Embedding dimension mismatch: expected ${VECTOR_DIMENSION}, got ${embedding.length}`
  );
}
```

Common dimensions: text-embedding-3-small=1536, text-embedding-3-large=3072, text-embedding-ada-002=1536

---

## VEC-1: Vector Type Handling (MEDIUM)

**Pattern:** Use `CAST(@param AS VECTOR(dimension))` for Azure SQL. Serialize vectors as JSON strings.

DO (Azure SQL):
```typescript
await executeQueryAsync(
  connection,
  'INSERT INTO [Hotels] ([embedding]) VALUES (CAST(@embedding AS VECTOR(1536)))',
  [{ name: 'embedding', type: TYPES.NVarChar, value: JSON.stringify(embedding) }]
);

// Vector distance query
const sql = `
  SELECT TOP (@k) [id], [name],
    VECTOR_DISTANCE('cosine', [embedding], CAST(@searchEmbedding AS VECTOR(1536))) AS distance
  FROM [Hotels] ORDER BY distance ASC
`;
```

DO (Cosmos DB):
```typescript
const { resources } = await container.items.query({
  query: `SELECT TOP @k c.id, c.name, VectorDistance(c.embedding, @searchEmbedding) AS similarity
    FROM c ORDER BY VectorDistance(c.embedding, @searchEmbedding)`,
  parameters: [{ name: '@k', value: 5 }, { name: '@searchEmbedding', value: searchEmbedding }],
}).fetchAll();
```

DO (Azure AI Search):
```typescript
import { SearchClient } from '@azure/search-documents';
const credential = new DefaultAzureCredential();
const searchClient = new SearchClient(config.SEARCH_ENDPOINT, 'hotels-index', credential);

const results = await searchClient.search('luxury hotel', {
  vectorSearchOptions: {
    queries: [{ kind: 'vector', vector: searchEmbedding, kNearestNeighborsCount: 5, fields: ['descriptionVector'] }],
  },
});
```

---

## VEC-2: DiskANN Index (HIGH)

**Pattern:** DiskANN (Azure SQL) requires >=1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

DO:
```typescript
const countResult = await executeQueryAsync(connection, `SELECT COUNT(*) AS [count] FROM [${tableName}]`, []);
const rowCount = countResult[0][0].value;

if (rowCount >= 1000) {
  await executeQueryAsync(connection,
    `CREATE INDEX [ix_${tableName}_embedding_diskann] ON [${tableName}] ([embedding]) USING DiskANN`, []);
  // Use VECTOR_SEARCH for fast approximate search
} else {
  console.log(`Only ${rowCount} rows. DiskANN requires >=1000. Using exact search.`);
  // Fall back to VECTOR_DISTANCE (exact search)
}
```

DON'T:
```typescript
// Create DiskANN without checking row count
await executeQueryAsync(connection, 'CREATE INDEX ... USING DiskANN', []);
// Fails: "DiskANN index requires at least 1000 rows"
```
