# AI Services & Vector Search

AI service client patterns, API versioning, embeddings, chat completions, document analysis, and vector similarity search across Azure data services.

## AI-1: Azure OpenAI Client Configuration (HIGH)

Use `openai` SDK's `AzureOpenAI` class. Configure `timeout`, `max_retries`, and `azure_ad_token_provider`.

DO:
```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential,
    "https://cognitiveservices.azure.com/.default",
)

client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    azure_endpoint=config["AZURE_OPENAI_ENDPOINT"],
    api_version="2024-10-21",  # See: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
    timeout=30.0,
    max_retries=3,
)

# Chat completion
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}],
)

# Embeddings
embedding_response = client.embeddings.create(
    model="text-embedding-3-small",
    input="Sample text to embed",
)

# Image generation
image_response = client.images.generate(
    model="dall-e-3",
    prompt="A photo of a cat",
    n=1,
    size="1024x1024",
)
```

DON'T:
```python
# Don't use API keys in samples (prefer AAD)
client = AzureOpenAI(
    api_key=os.environ["AZURE_OPENAI_API_KEY"],  # Use azure_ad_token_provider
    azure_endpoint=endpoint,
)

# Don't omit timeout/retries
client = AzureOpenAI(
    azure_ad_token_provider=token_provider,
    azure_endpoint=endpoint,
    # Missing timeout, max_retries
)
```

---

## AI-2: API Version Documentation (LOW)

Hardcoded API versions should include a comment linking to version docs.

DO:
```python
client = AzureOpenAI(
    api_version="2024-10-21",
    # API version reference: https://learn.microsoft.com/azure/ai-services/openai/api-version-deprecation
)
```

---

## AI-3: Document Intelligence (azure-ai-documentintelligence) (MEDIUM)

Use `azure-ai-documentintelligence` with `DefaultAzureCredential` where supported.

DO:
```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
doc_client = DocumentIntelligenceClient(
    endpoint=config["DOCUMENT_INTELLIGENCE_ENDPOINT"],
    credential=credential,
)

with open("invoice.pdf", "rb") as f:
    poller = doc_client.begin_analyze_document(
        model_id="prebuilt-invoice",
        analyze_request=f,
        content_type="application/octet-stream",
    )
result = poller.result()

for document in result.documents:
    print(f"Document type: {document.doc_type}")
    for field_name, field in document.fields.items():
        print(f"  {field_name}: {field.content}")
```

---

## AI-4: Vector Dimension Validation (MEDIUM)

Embeddings must match the declared vector column dimension. Dimension mismatches cause silent failures or runtime errors.

DO:
```python
EMBEDDING_MODEL = "text-embedding-3-small"  # 1536 dimensions
VECTOR_DIMENSION = 1536

embedding = await get_embedding(text)
if len(embedding) != VECTOR_DIMENSION:
    raise ValueError(
        f"Embedding dimension mismatch: expected {VECTOR_DIMENSION}, "
        f"got {len(embedding)}. "
        f"Ensure model '{EMBEDDING_MODEL}' matches table schema."
    )
```

DON'T:
```python
# Don't assume dimension without validation
embedding = await get_embedding(text)
await insert_embedding(embedding)  # May fail silently if dimension wrong
```

Common dimensions:
- `text-embedding-3-small`: 1536 (default), configurable via `dimensions` parameter
- `text-embedding-3-large`: 3072 (default), configurable via `dimensions` parameter
- `text-embedding-ada-002`: 1536 (fixed, not configurable)

> `text-embedding-3-small` and `text-embedding-3-large` support a `dimensions` parameter to reduce output size:
> ```python
> embedding_response = client.embeddings.create(
>     model="text-embedding-3-small",
>     input="Sample text",
>     dimensions=256,  # Reduced from default 1536
> )
> ```

---

## AI-5: Speech SDK (azure-cognitiveservices-speech) (MEDIUM)

Use `azure-cognitiveservices-speech` for speech-to-text, text-to-speech, and real-time transcription. The Speech SDK has its own auth pattern (not `DefaultAzureCredential` — it uses `SpeechConfig` with subscription key or AAD token).

DO:
```python
import azure.cognitiveservices.speech as speechsdk
from azure.identity import DefaultAzureCredential

# Option A: Speech key (simpler for quickstarts)
speech_config = speechsdk.SpeechConfig(
    subscription=config["AZURE_SPEECH_KEY"],
    region=config["AZURE_SPEECH_REGION"],
)

# Option B: AAD token (production)
credential = DefaultAzureCredential()
token = credential.get_token("https://cognitiveservices.azure.com/.default")
auth_token = f"aad#{config['AZURE_SPEECH_RESOURCE_ID']}#{token.token}"
speech_config = speechsdk.SpeechConfig(auth_token=auth_token, region=config["AZURE_SPEECH_REGION"])

# Speech-to-text (from microphone)
speech_config.speech_recognition_language = "en-US"
audio_config = speechsdk.audio.AudioConfig(use_default_microphone=True)
recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)

result = recognizer.recognize_once()
if result.reason == speechsdk.ResultReason.RecognizedSpeech:
    print(f"Recognized: {result.text}")
elif result.reason == speechsdk.ResultReason.NoMatch:
    print("No speech could be recognized")

# Text-to-speech
speech_config.speech_synthesis_voice_name = "en-US-AriaNeural"
synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config)
result = synthesizer.speak_text_async("Hello from Azure Speech Service").get()

if result.reason == speechsdk.ResultReason.SynthesizingAudioCompleted:
    print("Speech synthesized successfully")

# Continuous recognition (for long audio)
def recognized_handler(evt):
    print(f"Recognized: {evt.result.text}")

recognizer.recognized.connect(recognized_handler)
recognizer.start_continuous_recognition()
```

DON'T:
```python
# Don't hardcode speech key in source
speech_config = speechsdk.SpeechConfig(
    subscription="abc123def456",  # Hardcoded key
    region="eastus",
)

# Don't ignore result.reason (may silently fail)
result = recognizer.recognize_once()
print(result.text)  # Crashes if recognition failed
```

Package: `pip install azure-cognitiveservices-speech`

---

## VEC-1: Vector Type Handling (MEDIUM)

Use `CAST(@param AS VECTOR(dimension))` for Azure SQL vector parameters. Serialize vectors as JSON strings.

DO (Azure SQL):
```python
import json

embedding = [0.1, 0.2, 0.3, ...]  # 1536 floats from OpenAI

# Insert vector — cast to VECTOR type
cursor.execute(
    "INSERT INTO [Hotels] ([embedding]) VALUES (CAST(? AS VECTOR(1536)))",
    (json.dumps(embedding),),
)

# Vector distance query
sql = """
    SELECT TOP (?)
        [id],
        [name],
        VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
    FROM [Hotels]
    ORDER BY distance ASC
"""
cursor.execute(sql, (k, json.dumps(search_embedding)))
```

DO (Cosmos DB Vector Search):
```python
items = container.query_items(
    query="""
        SELECT TOP @k c.id, c.name,
            VectorDistance(c.embedding, @searchEmbedding) AS similarity
        FROM c
        ORDER BY VectorDistance(c.embedding, @searchEmbedding)
    """,
    parameters=[
        {"name": "@k", "value": 5},
        {"name": "@searchEmbedding", "value": search_embedding},
    ],
)
```

DO (Azure AI Search):
```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
search_client = SearchClient(
    endpoint=config["SEARCH_ENDPOINT"],
    index_name="hotels-index",
    credential=credential,
)

# Use VectorizedQuery class (not raw dicts)
vector_query = VectorizedQuery(
    vector=search_embedding,
    k_nearest_neighbors=5,
    fields="descriptionVector",
)

results = search_client.search(
    search_text="luxury hotel",
    vector_queries=[vector_query],
)

for result in results:
    print(f"{result['name']} (score: {result['@search.score']})")
```

DON'T (Azure AI Search):
```python
# Don't use raw dicts for vector queries — use VectorizedQuery class
results = search_client.search(
    search_text="luxury hotel",
    vector_queries=[{
        "kind": "vector",
        "vector": search_embedding,
        "k_nearest_neighbors": 5,
        "fields": "descriptionVector",
    }],
)
```

---

## VEC-2: DiskANN Index (HIGH)

DiskANN (Azure SQL) requires >= 1000 rows. Check row count before creating index. Fall back to exact search if insufficient data.

DO:
```python
cursor.execute(f"SELECT COUNT(*) FROM [{table_name}]")
row_count = cursor.fetchone()[0]

if row_count >= 1000:
    print(f"{row_count} rows available. Creating DiskANN index...")

    cursor.execute(
        f"CREATE INDEX [ix_{table_name}_embedding_diskann] "
        f"ON [{table_name}] ([embedding]) "
        f"USING DiskANN"
    )
    connection.commit()
else:
    print(f"Only {row_count} rows. DiskANN requires >= 1000. Using exact search.")

    sql = f"""
        SELECT TOP (?) [id], [name],
            VECTOR_DISTANCE('cosine', [embedding], CAST(? AS VECTOR(1536))) AS distance
        FROM [{table_name}]
        ORDER BY distance ASC
    """
```

DON'T:
```python
# Create DiskANN index without checking row count
cursor.execute(f"CREATE INDEX ... ON [{table_name}] ([embedding]) USING DiskANN")
# Fails with: "DiskANN index requires at least 1000 rows"
```
