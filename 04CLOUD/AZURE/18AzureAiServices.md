# Azure OpenAI & AI Services
## (analogous to AWS Bedrock & Rekognition/Comprehend/Textract)

Azure offers two tiers of AI services: **Azure OpenAI Service** (hosted GPT-4, embeddings, DALL-E with enterprise data privacy) and **Azure AI Services** (pre-built APIs for vision, speech, language, and document intelligence — no ML expertise needed).

---

## Part 1: Azure OpenAI Service
## (analogous to AWS Bedrock with OpenAI models)

Azure OpenAI gives you access to OpenAI models (GPT-4o, GPT-4, GPT-3.5-turbo, text-embedding-ada-002, DALL-E 3) hosted on Azure infrastructure — with data privacy guarantees, VNet support, RBAC, and Azure Monitor integration.

> Your data is **not** used to train OpenAI models. The API is compatible with the OpenAI SDK (just change the base URL).

---

### Creating a Resource and Deploying a Model

```bash
# Create an Azure OpenAI resource
az cognitiveservices account create \
  --resource-group myRG \
  --name my-openai \
  --kind OpenAI \
  --sku S0 \
  --location eastus

# Deploy a model (model name → your deployment name)
az cognitiveservices account deployment create \
  --resource-group myRG \
  --name my-openai \
  --deployment-name gpt-4o \
  --model-name gpt-4o \
  --model-version "2024-11-20" \
  --model-format OpenAI \
  --sku-capacity 100 \
  --sku-name Standard

# Get the endpoint
az cognitiveservices account show \
  --resource-group myRG \
  --name my-openai \
  --query properties.endpoint \
  --output tsv
```

---

### Node.js: Chat Completions

```bash
npm install openai @azure/identity
```

```javascript
const { AzureOpenAI } = require("openai");
const { DefaultAzureCredential, getBearerTokenProvider } = require("@azure/identity");

// Option 1: Managed Identity (recommended in production)
const credential = new DefaultAzureCredential();
const tokenProvider = getBearerTokenProvider(credential, "https://cognitiveservices.azure.com/.default");

const client = new AzureOpenAI({
  endpoint: process.env.AZURE_OPENAI_ENDPOINT,
  apiVersion: "2024-10-21",
  azureADTokenProvider: tokenProvider,
});

// Option 2: API Key (simpler for local dev)
const client = new AzureOpenAI({
  endpoint: process.env.AZURE_OPENAI_ENDPOINT,
  apiKey: process.env.AZURE_OPENAI_KEY,
  apiVersion: "2024-10-21",
});

// Chat completion
const response = await client.chat.completions.create({
  model: "gpt-4o",                  // your deployment name
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Summarize the key points of the Kyoto Protocol." },
  ],
  max_tokens: 500,
  temperature: 0.7,
});

console.log(response.choices[0].message.content);

// Streaming response
const stream = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Write a haiku about cloud computing." }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? "");
}
```

---

### Embeddings (for semantic search and RAG)

```javascript
// Generate embeddings for a text
const embeddingResponse = await client.embeddings.create({
  model: "text-embedding-3-large",    // your deployment name
  input: "How do I reset my password?",
});

const vector = embeddingResponse.data[0].embedding;  // float[] of 3072 dimensions
console.log("Vector length:", vector.length);

// Batch embeddings (documents for RAG)
const documents = [
  "Password reset: go to Settings → Security → Reset Password.",
  "Two-factor authentication can be enabled in the Security tab.",
  "To cancel your subscription, visit Account → Billing → Cancel.",
];

const batchResponse = await client.embeddings.create({
  model: "text-embedding-3-large",
  input: documents,
});

const documentVectors = batchResponse.data.map(d => d.embedding);
```

---

### Function Calling (structured outputs / tool use)

```javascript
const tools = [
  {
    type: "function",
    function: {
      name: "get_order_status",
      description: "Get the current status of a customer order",
      parameters: {
        type: "object",
        properties: {
          order_id: { type: "string", description: "The order ID" },
        },
        required: ["order_id"],
      },
    },
  },
];

const response = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "What's the status of my order ORD-12345?" }],
  tools,
  tool_choice: "auto",
});

const toolCall = response.choices[0].message.tool_calls?.[0];
if (toolCall) {
  const args = JSON.parse(toolCall.function.arguments);
  const status = await getOrderStatus(args.order_id);   // call your actual function

  // Send result back for a final response
  const finalResponse = await client.chat.completions.create({
    model: "gpt-4o",
    messages: [
      { role: "user", content: "What's the status of my order ORD-12345?" },
      response.choices[0].message,
      { role: "tool", tool_call_id: toolCall.id, content: JSON.stringify(status) },
    ],
    tools,
  });
  console.log(finalResponse.choices[0].message.content);
}
```

---

### Image Generation (DALL-E 3)

```javascript
const imageResponse = await client.images.generate({
  model: "dall-e-3",               // your deployment name
  prompt: "A futuristic data center surrounded by mountains at sunset, digital art",
  n: 1,
  size: "1024x1024",
  quality: "hd",
  style: "vivid",
});

console.log("Image URL:", imageResponse.data[0].url);
```

---

## Part 2: Azure AI Services
## (analogous to AWS Rekognition / Comprehend / Textract / Polly / Transcribe)

Azure AI Services (formerly Cognitive Services) are pre-built AI APIs — no model training needed.

---

### Creating an AI Services Resource

```bash
# Multi-service resource (covers Vision, Language, Speech, etc.)
az cognitiveservices account create \
  --resource-group myRG \
  --name my-ai-services \
  --kind CognitiveServices \
  --sku S0 \
  --location eastus
```

---

### Computer Vision (image analysis — analogous to AWS Rekognition)

```bash
npm install @azure-rest/ai-vision-image-analysis
```

```javascript
const ImageAnalysisClient = require("@azure-rest/ai-vision-image-analysis").default;
const { DefaultAzureCredential } = require("@azure/identity");

const client = ImageAnalysisClient(
  process.env.VISION_ENDPOINT,
  new DefaultAzureCredential()
);

const result = await client.path("/imageanalysis:analyze").post({
  body: { url: "https://example.com/my-image.jpg" },
  queryParameters: {
    features: ["Caption", "Tags", "Objects", "People", "Read"],
    language: "en",
  },
  contentType: "application/json",
});

const { captionResult, tagsResult, readResult } = result.body;

console.log("Caption:", captionResult.text);
console.log("Tags:", tagsResult.values.map(t => t.name).join(", "));
console.log("Extracted text:", readResult.blocks.map(b => b.lines.map(l => l.text).join(" ")).join("\n"));
```

---

### Language Service (text analytics — analogous to AWS Comprehend)

```bash
npm install @azure/ai-language-text
```

```javascript
const { TextAnalysisClient } = require("@azure/ai-language-text");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new TextAnalysisClient(
  process.env.LANGUAGE_ENDPOINT,
  new DefaultAzureCredential()
);

const documents = [
  "Azure is a fantastic cloud platform. I love working with it!",
  "The deployment failed again and I'm extremely frustrated.",
  "Call me at +1-800-555-1234 or email john.doe@company.com",
];

// Sentiment analysis
const sentimentResults = await client.analyze("SentimentAnalysis", documents);
for (const result of sentimentResults) {
  if (!result.error) {
    console.log(`Sentiment: ${result.sentiment}, Scores:`, result.confidenceScores);
  }
}

// Named entity recognition (PII detection)
const piiResults = await client.analyze("PiiEntityRecognition", documents);
for (const result of piiResults) {
  if (!result.error) {
    console.log("PII entities:", result.entities.map(e => `${e.text} (${e.category})`));
    console.log("Redacted:", result.redactedText);
  }
}

// Key phrase extraction
const kpResults = await client.analyze("KeyPhraseExtraction", documents);
for (const result of kpResults) {
  if (!result.error) {
    console.log("Key phrases:", result.keyPhrases);
  }
}

// Language detection
const langResults = await client.analyze("LanguageDetection", documents);
for (const result of langResults) {
  if (!result.error) {
    console.log("Language:", result.primaryLanguage.name, result.primaryLanguage.confidenceScore);
  }
}
```

---

### Document Intelligence (analogous to AWS Textract)

Extract structured data from invoices, receipts, forms, and ID documents.

```bash
npm install @azure/ai-form-recognizer
```

```javascript
const { DocumentAnalysisClient } = require("@azure/ai-form-recognizer");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new DocumentAnalysisClient(
  process.env.DOCUMENT_INTELLIGENCE_ENDPOINT,
  new DefaultAzureCredential()
);

// Analyze an invoice (prebuilt model)
const poller = await client.beginAnalyzeDocumentFromUrl(
  "prebuilt-invoice",
  "https://example.com/invoice.pdf"
);

const result = await poller.pollUntilDone();

for (const invoice of result.documents) {
  const fields = invoice.fields;
  console.log("Vendor:", fields["VendorName"]?.value);
  console.log("Invoice Date:", fields["InvoiceDate"]?.value);
  console.log("Total:", fields["InvoiceTotal"]?.value);
  console.log("Line items:", fields["Items"]?.values?.map(item => ({
    description: item.properties["Description"]?.value,
    amount: item.properties["Amount"]?.value,
  })));
}

// Extract text from any document (layout model)
const layoutPoller = await client.beginAnalyzeDocumentFromUrl(
  "prebuilt-layout",
  "https://example.com/report.pdf"
);
const layoutResult = await layoutPoller.pollUntilDone();
for (const page of layoutResult.pages) {
  for (const line of page.lines) {
    console.log(line.content);
  }
}
```

---

### Speech Service (analogous to AWS Transcribe + Polly)

```bash
npm install microsoft-cognitiveservices-speech-sdk
```

```javascript
const sdk = require("microsoft-cognitiveservices-speech-sdk");

const speechConfig = sdk.SpeechConfig.fromSubscription(
  process.env.SPEECH_KEY,
  process.env.SPEECH_REGION
);

// Speech to Text (microphone)
speechConfig.speechRecognitionLanguage = "en-US";
const audioConfig = sdk.AudioConfig.fromDefaultMicrophoneInput();
const recognizer = new sdk.SpeechRecognizer(speechConfig, audioConfig);

recognizer.recognizeOnceAsync(result => {
  console.log("Recognized:", result.text);
});

// Text to Speech
speechConfig.speechSynthesisVoiceName = "en-US-JennyNeural";
const synthesizer = new sdk.SpeechSynthesizer(speechConfig);

synthesizer.speakTextAsync(
  "Hello! I am an Azure neural voice.",
  result => {
    // result.audioData contains the WAV bytes
    require("fs").writeFileSync("output.wav", Buffer.from(result.audioData));
    synthesizer.close();
  }
);
```

---

### Azure AI Search (vector + semantic search — analogous to AWS OpenSearch)

Azure AI Search is commonly paired with Azure OpenAI for **RAG (Retrieval-Augmented Generation)** — store document chunks with their embeddings, then retrieve the most relevant ones at query time.

```bash
npm install @azure/search-documents
```

```javascript
const { SearchClient, SearchIndexClient, AzureKeyCredential } = require("@azure/search-documents");
const { DefaultAzureCredential } = require("@azure/identity");

const indexClient = new SearchIndexClient(
  process.env.SEARCH_ENDPOINT,
  new DefaultAzureCredential()
);

// Create an index with a vector field
await indexClient.createOrUpdateIndex({
  name: "documents",
  fields: [
    { name: "id",       type: "Edm.String", key: true },
    { name: "content",  type: "Edm.String", searchable: true },
    { name: "embedding", type: "Collection(Edm.Single)", searchable: true,
      vectorSearchDimensions: 3072, vectorSearchProfileName: "myProfile" },
  ],
  vectorSearch: {
    profiles: [{ name: "myProfile", algorithmConfigurationName: "myHnsw" }],
    algorithms: [{ name: "myHnsw", kind: "hnsw" }],
  },
});

const searchClient = new SearchClient(
  process.env.SEARCH_ENDPOINT,
  "documents",
  new DefaultAzureCredential()
);

// Upload documents with embeddings
await searchClient.uploadDocuments([
  { id: "1", content: "Reset password in Settings → Security.", embedding: vectorFor("Reset password...") },
  { id: "2", content: "Cancel subscription in Account → Billing.", embedding: vectorFor("Cancel subscription...") },
]);

// Vector search (semantic similarity)
const queryVector = await generateEmbedding("How do I change my password?");

const results = await searchClient.search("*", {
  vectorSearchOptions: {
    queries: [{ kind: "vector", vector: queryVector, fields: ["embedding"], kNearestNeighborsCount: 5 }],
  },
  select: ["id", "content"],
});

for await (const result of results.results) {
  console.log(`Score: ${result.score}  Content: ${result.document.content}`);
}
```

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Hosted LLMs | Bedrock (multiple providers) | Azure OpenAI (OpenAI models) |
| Image analysis | Rekognition | Azure AI Vision |
| Text analytics | Comprehend | Azure AI Language |
| Document extraction | Textract | Azure Document Intelligence |
| Speech to text | Transcribe | Azure Speech Service |
| Text to speech | Polly | Azure Neural TTS |
| Vector search / RAG | OpenSearch Serverless | Azure AI Search |
| OpenAI API compatibility | No | Yes (drop-in with base URL change) |
| Data privacy | Depends on model/provider | Azure OpenAI — data not used for training |