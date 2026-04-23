# AI Services (OpenAI, Document Intelligence, Speech) & Vector Search

**What this section covers:** AI service client patterns, API versioning, embeddings, chat completions, document analysis, vector similarity search, and approximate nearest neighbor indexing.

## AI-1: Azure OpenAI Client (HIGH)
**Pattern:** Use `com.azure:azure-ai-openai` with `DefaultAzureCredential`. Configure timeouts and retry options.

DO:
```java
import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.ChatCompletions;
import com.azure.ai.openai.models.ChatCompletionsOptions;
import com.azure.ai.openai.models.ChatRequestUserMessage;
import com.azure.ai.openai.models.EmbeddingsOptions;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

// Build client with AAD credential
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(config.azureOpenAiEndpoint())
    .credential(credential)
    .buildClient();

// Chat completion
ChatCompletions completions = client.getChatCompletions(
    "gpt-4o",
    new ChatCompletionsOptions(List.of(
        new ChatRequestUserMessage("Hello!")
    ))
);
System.out.println(completions.getChoices().get(0).getMessage().getContent());

// Embeddings
var embeddingsResult = client.getEmbeddings(
    "text-embedding-3-small",
    new EmbeddingsOptions(List.of("Sample text to embed"))
);
List<Float> embedding = embeddingsResult.getData().get(0).getEmbedding();
```

DON'T:
```java
// Don't use API keys in samples (prefer AAD)
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(endpoint)
    .credential(new AzureKeyCredential(apiKey))  // Use DefaultAzureCredential
    .buildClient();

// Don't use deprecated constructor patterns
// Check Azure SDK for Java release notes for current API
```

---

## AI-2: API Version Documentation (LOW)
**Pattern:** Hardcoded API versions should include a comment linking to version docs.

DO:
```java
OpenAIClient client = new OpenAIClientBuilder()
    .endpoint(endpoint)
    .credential(credential)
    .serviceVersion(OpenAIServiceVersion.V2024_10_21)
    // API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
    .buildClient();
```

---

## AI-3: Document Intelligence (MEDIUM)
**Pattern:** Use `com.azure:azure-ai-documentintelligence` with `DefaultAzureCredential`.

DO:
```java
import com.azure.ai.documentintelligence.DocumentIntelligenceClient;
import com.azure.ai.documentintelligence.DocumentIntelligenceClientBuilder;
import com.azure.ai.documentintelligence.models.AnalyzeResult;
import com.azure.identity.DefaultAzureCredentialBuilder;

var credential = new DefaultAzureCredentialBuilder().build();

DocumentIntelligenceClient client = new DocumentIntelligenceClientBuilder()
    .endpoint(config.documentIntelligenceEndpoint())
    .credential(credential)
    .buildClient();

// Analyze document
SyncPoller<AnalyzeResultOperation, AnalyzeResult> poller = client.beginAnalyzeDocument(
    "prebuilt-invoice",
    analyzeDocumentRequest
);
AnalyzeResult result = poller.getFinalResult();
```

---

## AI-4: Vector Dimension Validation (MEDIUM)
**Pattern:** Embeddings must match the declared vector column dimension. Dimension mismatches cause silent failures or runtime errors.

DO:
```java
// Document expected dimensions
private static final String EMBEDDING_MODEL = "text-embedding-3-small";  // 1536 dimensions
private static final int VECTOR_DIMENSION = 1536;

// Validate embedding size
List<Float> embedding = getEmbedding(text);
if (embedding.size() != VECTOR_DIMENSION) {
    throw new IllegalStateException(
        String.format("Embedding dimension mismatch: expected %d, got %d%n"
            + "Ensure model '%s' matches table schema.",
            VECTOR_DIMENSION, embedding.size(), EMBEDDING_MODEL)
    );
}
```

DON'T:
```java
// Don't assume dimension without validation
List<Float> embedding = getEmbedding(text);
insertEmbedding(embedding);  // May fail silently if dimension wrong
```

**Common dimensions:**
- `text-embedding-3-small`: 1536
- `text-embedding-3-large`: 3072
- `text-embedding-ada-002`: 1536

---

## AI-5: Speech SDK (HIGH)
**Pattern:** Use `com.microsoft.cognitiveservices.speech:client-sdk` for speech-to-text and text-to-speech. This is a separate SDK from the `com.azure:*` packages.

DO:
```xml
<!-- pom.xml - Speech SDK (separate from azure-sdk-bom) -->
<dependency>
    <groupId>com.microsoft.cognitiveservices.speech</groupId>
    <artifactId>client-sdk</artifactId>
    <version>1.42.0</version>  <!-- Check https://learn.microsoft.com/azure/ai-services/speech-service/releasenotes for latest -->
</dependency>
```

```java
import com.microsoft.cognitiveservices.speech.*;
import com.microsoft.cognitiveservices.speech.audio.AudioConfig;

// Speech-to-text with AAD auth (preferred)
String token = credential.getToken(
    new TokenRequestContext().addScopes("https://cognitiveservices.azure.com/.default")
).block().getToken();
String region = config.speechRegion();  // e.g., "eastus"

SpeechConfig speechConfig = SpeechConfig.fromAuthorizationToken(token, region);
speechConfig.setSpeechRecognitionLanguage("en-US");

// Recognize from microphone
try (SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig)) {
    SpeechRecognitionResult result = recognizer.recognizeOnceAsync().get();
    if (result.getReason() == ResultReason.RecognizedSpeech) {
        System.out.println("Recognized: " + result.getText());
    } else if (result.getReason() == ResultReason.NoMatch) {
        System.out.println("No speech recognized.");
    }
}  // Recognizer closed automatically

// Text-to-speech
try (SpeechSynthesizer synthesizer = new SpeechSynthesizer(speechConfig)) {
    SpeechSynthesisResult result = synthesizer.SpeakTextAsync("Hello from Azure!").get();
    if (result.getReason() == ResultReason.SynthesizingAudioCompleted) {
        System.out.println("Speech synthesized successfully.");
    }
}

// Recognize from audio file
AudioConfig audioConfig = AudioConfig.fromWavFileInput("sample.wav");
try (SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig, audioConfig)) {
    SpeechRecognitionResult result = recognizer.recognizeOnceAsync().get();
    System.out.println("Recognized: " + result.getText());
}
```

DON'T:
```java
// Don't use subscription key directly in samples (prefer AAD token)
SpeechConfig config = SpeechConfig.fromSubscription(subscriptionKey, region);

// Don't forget to close SpeechRecognizer/SpeechSynthesizer (native resources)
SpeechRecognizer recognizer = new SpeechRecognizer(speechConfig);
recognizer.recognizeOnceAsync().get();
// Native resources leaked
```

> **Note:** The Speech SDK (`com.microsoft.cognitiveservices.speech:client-sdk`) is NOT managed by `azure-sdk-bom`. Pin the version explicitly. The SDK includes native libraries -- ensure the correct platform classifier is available at runtime.

---

## VEC-1: Vector Type Handling (MEDIUM)
**Pattern:** Serialize vectors as JSON strings for Azure SQL VECTOR type. Use appropriate parameter handling for each service.

DO (Azure SQL):
```java
// Insert vector as JSON string with CAST
String insertSql = "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(? AS VECTOR(1536)))";
try (PreparedStatement stmt = conn.prepareStatement(insertSql)) {
    // Serialize float array as JSON string
    String embeddingJson = objectMapper.writeValueAsString(embedding);
    stmt.setString(1, embeddingJson);
    stmt.executeUpdate();
}

// Vector distance query
String searchSql = """
    SELECT TOP (?)
        [id], [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC
    """;
try (PreparedStatement stmt = conn.prepareStatement(searchSql)) {
    stmt.setInt(1, k);
    stmt.setString(2, objectMapper.writeValueAsString(searchEmbedding));
    try (ResultSet rs = stmt.executeQuery()) {
        while (rs.next()) {
            System.out.printf("%s (distance: %.4f)%n", rs.getString("name"), rs.getFloat("distance"));
        }
    }
}
```

DO (Cosmos DB Vector Search):
```java
// Cosmos DB vector search
SqlQuerySpec query = new SqlQuerySpec(
    "SELECT TOP @k c.id, c.name, VectorDistance(c.embedding, @searchEmbedding) AS similarity "
    + "FROM c ORDER BY VectorDistance(c.embedding, @searchEmbedding)",
    List.of(
        new SqlParameter("@k", 5),
        new SqlParameter("@searchEmbedding", searchEmbedding)
    )
);

container.queryItems(query, new CosmosQueryRequestOptions(), JsonNode.class)
    .forEach(item -> System.out.printf("%s (similarity: %.4f)%n",
        item.get("name").asText(), item.get("similarity").asDouble()));
```

---

## VEC-2: DiskANN Index (HIGH)
**Pattern:** DiskANN (Azure SQL) requires >= 1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

DO:
```java
// Check row count before creating DiskANN index
int rowCount;
try (Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM [" + tableName + "]")) {
    rs.next();
    rowCount = rs.getInt(1);
}

if (rowCount >= 1000) {
    System.out.printf("%d rows available. Creating DiskANN index...%n", rowCount);
    try (Statement stmt = conn.createStatement()) {
        stmt.execute("CREATE INDEX [ix_" + tableName + "_embedding_diskann] "
            + "ON [" + tableName + "] ([embedding]) USING DiskANN");
    }
} else {
    System.out.printf("Only %d rows. DiskANN requires >= 1000. Using exact search.%n", rowCount);
    // Fall back to VECTOR_DISTANCE (exact search)
}
```

DON'T:
```java
// Create DiskANN index without checking row count
stmt.execute("CREATE INDEX ... USING DiskANN");
// Fails with: "DiskANN index requires at least 1000 rows"
```
