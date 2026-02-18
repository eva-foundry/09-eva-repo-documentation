# Application Architecture
## Component-Level Technical Documentation

**Evidence Sources**:
- app/backend/app.py (2557 lines, newly read lines 1-1000)
- app/enrichment/app.py (870 lines, newly read lines 1-400)
- EVA-DA-Pipeline-Analysis-Evidence-Based.md (680 lines)

---

## Diagram 5: Backend API Routing (FastAPI/Quart)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Backend API: app.py (2557 lines)                          │
│                     Framework: FastAPI 0.115.0 (async)                        │
│                     Python: 3.11+, Port: 5000 (dev), 443 (prod)              │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     │ HTTP Request
                                     ▼
                    ┌─────────────────────────────────┐
                    │   Route Handler Dispatcher      │
                    │   (FastAPI automatic routing)   │
                    └─────────────────┬───────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│  Chat Routes  │           │ Upload Routes │           │ Session Routes│
│  (lines 600-  │           │ (lines 800-   │           │ (routers/     │
│   700)        │           │  900)         │           │  sessions.py) │
└───────┬───────┘           └───────┬───────┘           └───────┬───────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  POST /chat (lines 600-680)                                           │
│    Purpose: Multi-turn chat with RAG                                  │
│    Request: {history, approach, overrides, session_id}                │
│    Response: StreamingResponse (SSE)                                  │
│                                                                        │
│    Flow:                                                               │
│      1. Extract x-ms-client-principal-id from header                  │
│      2. Fetch user's current group ID (RBAC)                          │
│      3. Determine upload_container, content_container, index          │
│      4. Load session history (if session_id provided)                 │
│      5. Select approach: ReadRetrieveRead, CompareWorkWithWeb, etc.   │
│      6. Stream response chunks via SSE                                │
│                                                                        │
│    Key Code (lines 600-650):                                          │
│      global group_items, expired_time                                 │
│      group_items, expired_time = read_all_items_into_cache_if_expired(│
│        groupmapcontainer, group_items, expired_time)                  │
│      oid = header.get("x-ms-client-principal-id")                     │
│      _, current_grp_id = user_info.fetch_lastest_choice_of_group(oid) │
│      upload_container, role = find_upload_container_and_role(...)     │
│      content_container, role = find_container_and_role(...)           │
│      index, role = find_index_and_role(...)                           │
│                                                                        │
│    Approaches Initialized (lines 650-680):                            │
│      read_retrieve = ChatReadRetrieveReadApproach(                    │
│        search_client, AZURE_OPENAI_ENDPOINT, ...                      │
│      )                                                                 │
│      chat_approaches[Approaches.ReadRetrieveRead] = read_retrieve     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  POST /getalluploadstatus (lines 700-750)                             │
│    Purpose: Get status of all file uploads in timeframe               │
│    Request: {timeframe, state, folder, tag}                           │
│    Response: [{file_path, status, download_url, tags, ...}]           │
│                                                                        │
│    Flow:                                                               │
│      1. Get upload_container for user's group                         │
│      2. Query StatusLog (Cosmos DB) for files in timeframe            │
│      3. Generate SAS tokens for download URLs                         │
│      4. Return results with metadata                                  │
│                                                                        │
│    Key Code (lines 720-730):                                          │
│      results = statusLog.read_files_status_by_timeframe(              │
│        timeframe, State[state], folder, tag, upload_container)        │
│      user_delegation_key = blob_service_client.get_user_delegation... │
│      sas_token = generate_blob_sas(...)                               │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  POST /getcitation (lines 950-1000)                                   │
│    Purpose: Retrieve document citation for display                    │
│    Request: {citation}                                                 │
│    Response: Blob content (PDF, DOCX, etc.)                           │
│                                                                        │
│    Flow:                                                               │
│      1. Get content_container for user's group                        │
│      2. URL-decode citation path                                      │
│      3. Download blob from content_container                          │
│      4. Return blob with appropriate MIME type                        │
│                                                                        │
│    Key Code (lines 960-980):                                          │
│      container, role = find_container_and_role(request, group_items, │
│        current_grp_id)                                                 │
│      blob_container = content_container_to_content_blob_container_... │
│      replaced_citation = origin_citation.replace("%2F", "/")          │
│      blob = blob_container.get_blob_client(replaced_citation).downl...│
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│  GET /health (lines 600-620)                                           │
│    Purpose: Health check endpoint                                     │
│    Response: {status: "ready", uptime_seconds: 123.45, version: "..."} │
│                                                                        │
│  GET /getInfoData (lines 900-920)                                     │
│    Purpose: System information for frontend                           │
│    Response: {AZURE_OPENAI_CHATGPT_DEPLOYMENT, MODEL_NAME, ...}      │
│                                                                        │
│  GET /getWarningBanner (lines 930-940)                                │
│    Purpose: Warning banner text for UI                                │
│    Response: {WARNING_BANNER_TEXT: "..."}                             │
│                                                                        │
│  POST /deleteItems (lines 800-830)                                    │
│    Purpose: Delete uploaded file (owner role only)                    │
│    Request: {path}                                                     │
│    Response: true/false                                                │
│    RBAC Check: if "owner" not in role.lower(): raise HTTPException   │
│                                                                        │
│  POST /resubmitItems (lines 840-880)                                  │
│    Purpose: Resubmit failed upload to pipeline                        │
│    Request: {path, tag}                                                │
│    Response: true/false                                                │
│    Action: Re-upload blob with same metadata → triggers pipeline      │
│                                                                        │
│  POST /getfolders (lines 750-780)                                     │
│    Purpose: List all folders in upload container                      │
│    Response: ["folder1", "folder2", ...]                              │
│                                                                        │
│  POST /gettags (lines 780-800)                                        │
│    Purpose: List all unique tags from uploaded files                  │
│    Response: ["tag1", "tag2", ...]                                    │
│    Query: SELECT DISTINCT VALUE t FROM c JOIN t IN c.tags WHERE ...  │
│                                                                        │
│  POST /logstatus (lines 880-900)                                      │
│    Purpose: Log upload status to Cosmos DB                            │
│    Request: {path, status, status_classification, state}              │
│    Response: {status: 200}                                             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                     Initialization & Dependencies                      │
│                        (lines 1-600)                                   │
│                                                                        │
│  Core Imports (lines 1-50):                                           │
│    - FastAPI, StreamingResponse, HTTPException                        │
│    - Azure SDK: SearchClient, BlobServiceClient, CosmosClient        │
│    - OpenAI: get_bearer_token_provider                                │
│    - Shared code: shared_constants, utility_rbck, StatusLog           │
│                                                                        │
│  Environment Loading (lines 40-50):                                   │
│    load_dotenv(os.path.join(__file__, "backend.env"))                 │
│    load_dotenv(os.path.join(__file__, "../../scripts/environments/.env"))│
│                                                                        │
│  Azure Client Initialization (lines 100-150):                         │
│    from core.shared_constants import app_clients, AZURE_CREDENTIAL    │
│    - Centralized Azure clients (singleton pattern)                    │
│    - DefaultAzureCredential for authentication                        │
│    - Token provider for Azure OpenAI                                  │
│                                                                        │
│  Application Insights (lines 350-360):                                │
│    configure_azure_monitor(                                            │
│      connection_string=ENV["APPLICATIONINSIGHTS_CONNECTION_STRING"],  │
│      logger_name="DA_APP",                                            │
│      enable_live_metrics=True                                         │
│    )                                                                   │
│                                                                        │
│  Lifespan Management (lines 400-500):                                 │
│    @asynccontextmanager                                                │
│    async def lifespan(app: FastAPI):                                  │
│      # Startup: Initialize Cosmos DB, Blob Storage, Search clients   │
│      cosmosdb_client = CosmosClient(ENV["COSMOSDB_URL"], ...)         │
│      blob_client = BlobServiceClient(ENV["AZURE_BLOB_STORAGE_ENDPOINT"], ...)│
│      statusLog = StatusLog(...)                                       │
│      user_info = UserProfile(...)                                     │
│      groupmapcontainer = initiate_group_resource_map(...)             │
│      yield                                                             │
│      # Shutdown: Clean up clients                                     │
│      blob_client.close()                                              │
│                                                                        │
│  RAG Approaches Registration (lines 350-400):                         │
│    chat_approaches = {                                                 │
│      Approaches.ChatWebRetrieveRead: ChatWebRetrieveRead(...),        │
│      Approaches.CompareWorkWithWeb: CompareWorkWithWeb(...),          │
│      Approaches.GPTDirect: GPTDirectApproach(...),                    │
│    }                                                                   │
│    # ReadRetrieveRead added dynamically per request (line 670)       │
│                                                                        │
│  Custom Examples Management (lines 120-280):                          │
│    - persist_examples_to_blob() → config/examplelist.json            │
│    - refresh_examples_from_blob() → Load custom examples             │
│    - Group-based example configuration                                │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

[Route Summary Table]

Route                    │ Method │ Purpose                      │ RBAC  │ Lines
─────────────────────────┼────────┼──────────────────────────────┼───────┼───────
/health                  │ GET    │ Health check                 │ No    │ 600-620
/chat                    │ POST   │ Multi-turn RAG chat          │ Yes   │ 600-680
/getalluploadstatus      │ POST   │ List uploaded files status   │ Yes   │ 700-750
/getfolders              │ POST   │ List folders                 │ Yes   │ 750-780
/deleteItems             │ POST   │ Delete file (owner only)     │ Yes   │ 800-830
/resubmitItems           │ POST   │ Resubmit failed upload       │ Yes   │ 840-880
/gettags                 │ POST   │ List unique tags             │ Yes   │ 780-800
/logstatus               │ POST   │ Log upload status            │ Yes   │ 880-900
/getcitation             │ POST   │ Get document citation        │ Yes   │ 950-1000
/getInfoData             │ GET    │ System information           │ Yes   │ 900-920
/getWarningBanner        │ GET    │ Warning banner text          │ No    │ 930-940
/getMaxCSVFileSize       │ GET    │ Max CSV upload size          │ No    │ 940-950
/sessions/*              │ Various│ Session management           │ Yes   │ routers/sessions.py
```

**Evidence**: app/backend/app.py (lines 1-1000)  
**Key Features**:
- 20+ FastAPI routes with async handlers
- RBAC enforcement at every endpoint (x-ms-client-principal-id header)
- Streaming responses via SSE for /chat
- Centralized Azure client management (core/shared_constants.py)
- Group-based resource isolation (Cosmos DB group_management container)
- Custom examples configuration per group

---

## Diagram 6: Enrichment Service Architecture (Flask)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│               Enrichment Service: app.py (870 lines)                          │
│               Framework: Flask                                                │
│               Port: 8080 (containerized)                                      │
│               Purpose: Embedding generation + Queue polling                   │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     │ HTTP API + Background Worker
                                     ▼
        ┌─────────────────────────────────────────────────────────┐
        │                      Main Components                     │
        └─────────────────┬──────────────────┬────────────────────┘
                          │                  │
                          ▼                  ▼
        ┌─────────────────────────┐  ┌──────────────────────────┐
        │    Flask API Server     │  │  Background Worker       │
        │    (REST endpoints)     │  │  (Queue Polling Thread)  │
        └────────────┬────────────┘  └────────────┬─────────────┘
                     │                            │
                     ▼                            ▼

┌────────────────────────────────────────────────────────────────────────┐
│                        Flask API Endpoints                             │
│                        (lines 250-350)                                 │
│                                                                        │
│  GET / (redirect to /docs)                                            │
│    Purpose: Redirect root to Swagger UI                               │
│                                                                        │
│  GET /health (lines 260-280)                                          │
│    Purpose: Health check                                              │
│    Response: {status: "ready", uptime_seconds: 123.45, version: "..."}│
│    Implementation:                                                     │
│      IS_READY = True (set after models loaded)                        │
│      uptime = datetime.now() - start_time                             │
│                                                                        │
│  GET /models (lines 280-290)                                          │
│    Purpose: List available embedding models                           │
│    Response: {models: [{model: "...", vector_size: 1536}, ...]}      │
│    Models:                                                             │
│      - azure-openai_text-embedding-3-small (1536-dim)                 │
│      - sentence-transformers models (various dimensions)              │
│                                                                        │
│  GET /models/{model} (lines 290-300)                                  │
│    Purpose: Get specific model information                            │
│    Request: model name in path                                        │
│    Response: {model: "...", vector_size: 1536, ...}                   │
│                                                                        │
│  POST /models/{model}/embed (lines 300-350)                           │
│    Purpose: Generate embeddings for text list                         │
│    Request: {texts: ["text1", "text2", ...]}                          │
│    Response: {model: "...", data: [0.123, -0.456, ...], model_info: {...}}│
│                                                                        │
│    Implementation (lines 310-340):                                    │
│      if model not in models:                                          │
│        return {"message": f"Model {model} not found"}                 │
│      model_obj = models[model]                                        │
│      if model.startswith("azure-openai_"):                            │
│        embeddings = model_obj.encode(texts)                           │
│        embeddings = embeddings.data[0].embedding                      │
│      else:                                                             │
│        embeddings = model_obj.encode(texts)                           │
│        embeddings = embeddings.tolist()[0]                            │
│      return {"model": model, "data": embeddings, "model_info": ...}  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                     Background Queue Polling Worker                    │
│                     (lines 350-400+)                                   │
│                                                                        │
│  Startup Event (lines 350-360):                                       │
│    @app.on_event("startup")                                            │
│    def startup_event():                                                │
│      poll_thread = threading.Thread(target=poll_queue_thread)         │
│      poll_thread.daemon = True                                        │
│      poll_thread.start()                                              │
│                                                                        │
│  Poll Loop (lines 360-380):                                           │
│    def poll_queue_thread():                                            │
│      while True:                                                       │
│        poll_queue()                                                    │
│        time.sleep(5)  # 5-second polling interval                     │
│                                                                        │
│  Queue Processing (lines 380-500):                                    │
│    def poll_queue():                                                   │
│      # 1. Dequeue messages from embeddings-queue                      │
│      # 2. Process each message:                                       │
│      #    - Extract chunk data (content, metadata)                    │
│      #    - Generate embeddings via Azure OpenAI                      │
│      #    - Batch chunks (max 120 per batch, 12MB limit)             │
│      #    - Upload batch to Cognitive Search                          │
│      # 3. Handle errors:                                              │
│      #    - Requeue with backoff (MAX_EMBEDDING_REQUEUE_COUNT=3)     │
│      #    - Update StatusLog in Cosmos DB                             │
│                                                                        │
│    Batching Logic (lines 400-450):                                    │
│      MAX_CHUNKS_PER_BATCH = 120                                       │
│      MAX_PAYLOAD_BYTES = 12 * 1024**2  # 12 MB                        │
│      batch = []                                                        │
│      for chunk in chunks:                                             │
│        if len(batch) >= MAX_CHUNKS_PER_BATCH:                         │
│          _flush_batch(batch, search_index)                            │
│        if _payload_bytes(batch + [chunk]) > MAX_PAYLOAD_BYTES:       │
│          _flush_batch(batch, search_index)                            │
│        batch.append(chunk)                                            │
│      _flush_batch(batch, search_index)                                │
│                                                                        │
│    Upload to Search (lines 450-470):                                  │
│      def index_sections(chunks, search_index):                        │
│        search_client = SearchClient(                                  │
│          endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],               │
│          index_name=search_index,                                     │
│          credential=azure_credential                                  │
│        )                                                               │
│        results = search_client.upload_documents(documents=chunks)     │
│        succeeded = sum([1 for r in results if r.succeeded])           │
│        log.debug(f"Indexed {len(results)} chunks, {succeeded} succeeded")│
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│                     Model Loading & Initialization                     │
│                     (lines 1-250)                                      │
│                                                                        │
│  Environment Setup (lines 1-100):                                     │
│    ENV = {                                                             │
│      "EMBEDDINGS_QUEUE": "embeddings-queue",                          │
│      "AZURE_OPENAI_ENDPOINT": "https://...",                          │
│      "AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME": "text-embedding-3-small",│
│      "AZURE_SEARCH_SERVICE_ENDPOINT": "https://...",                  │
│      "TARGET_EMBEDDINGS_MODEL": "text-embedding-3-small",             │
│      "EMBEDDING_VECTOR_SIZE": 1536,                                   │
│      "MAX_CHUNKS_PER_BATCH": 120,                                     │
│      "MAX_PAYLOAD_BYTES": 12 * 1024**2,                               │
│    }                                                                   │
│                                                                        │
│  Azure Client Setup (lines 100-150):                                  │
│    openai.api_base = ENV["AZURE_OPENAI_ENDPOINT"]                     │
│    openai.api_type = "azure_ad"                                       │
│    azure_credential = ManagedIdentityCredential()                     │
│    token_provider = get_bearer_token_provider(azure_credential, ...) │
│    client = AzureOpenAI(azure_endpoint=..., azure_ad_token_provider=...)│
│                                                                        │
│  Model Wrappers (lines 150-200):                                      │
│    class AzOAIEmbedding(object):                                      │
│      def __init__(self, deployment_name):                             │
│        self.deployment_name = deployment_name                         │
│      @retry(wait=wait_random_exponential(...), stop=stop_after_attempt(5))│
│      def encode(self, texts):                                         │
│        response = client.embeddings.create(                           │
│          model=self.deployment_name, input=texts)                     │
│        return response                                                │
│                                                                        │
│    class STModel(object):  # Sentence Transformers                    │
│      def __init__(self, deployment_name):                             │
│        self.deployment_name = deployment_name                         │
│      @retry(wait=wait_random_exponential(...), stop=stop_after_attempt(5))│
│      def encode(self, texts):                                         │
│        model = SentenceTransformer(self.deployment_name)              │
│        response = model.encode(texts)                                 │
│        return response                                                │
│                                                                        │
│  Model Registration (lines 200-250):                                  │
│    models, model_info = load_models()  # Load sentence-transformers  │
│    # Add Azure OpenAI Embedding                                       │
│    models["azure-openai_" + ENV["AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME"]]│
│      = AzOAIEmbedding(ENV["AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME"]) │
│    model_info["azure-openai_" + ...] = {                              │
│      "model": "azure-openai_text-embedding-3-small",                  │
│      "vector_size": 1536,                                             │
│    }                                                                   │
│    IS_READY = True  # Signal models loaded                            │
│                                                                        │
│  StatusLog & RBAC (lines 40-80):                                      │
│    from shared_code.status_log import StatusLog                       │
│    from shared_code.utility_rbck import (                             │
│      initiate_group_resource_map,                                     │
│      read_all_items_into_cache_if_expired,                            │
│      find_content_container_and_role_based_on_upcontainer,            │
│      find_index_and_role_based_on_upcontainer,                        │
│    )                                                                   │
│    statusLog = StatusLog(ENV["COSMOSDB_URL"], azure_credential, ...) │
│    groupmapcontainer = initiate_group_resource_map(azure_credential)  │
│    group_items, expired_time = read_all_items_into_cache_if_expired(...)│
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

[Data Flow Diagram]

┌─────────────────────┐
│ Embeddings Queue    │ ← Messages from TextEnrichment function
│ (Storage Queue)     │
└──────────┬──────────┘
           │ 5-second polling (background thread)
           ▼
┌─────────────────────┐
│ Enrichment Service  │
│  poll_queue()       │
└──────────┬──────────┘
           │
           ├─→ 1. Dequeue batch of messages (up to 120 chunks)
           │
           ├─→ 2. Extract chunk data:
           │      - content (text to embed)
           │      - metadata (file_name, folder, chunk_id, etc.)
           │      - upload_container (for RBAC lookup)
           │
           ├─→ 3. Determine target index:
           │      search_index = find_index_and_role_based_on_upcontainer(
           │        upload_container, group_items)
           │
           ├─→ 4. Generate embeddings:
           │      POST /models/azure-openai_text-embedding-3-small/embed
           │      Request: {texts: [chunk1.content, chunk2.content, ...]}
           │      Response: {data: [[emb1], [emb2], ...]}
           │
           ├─→ 5. Batch chunks with size limits:
           │      - Max 120 chunks per batch
           │      - Max 12 MB payload size
           │      - Flush batch when limits reached
           │
           ├─→ 6. Upload to Cognitive Search:
           │      search_client.upload_documents(chunks)
           │      Each chunk: {id, content, contentVector, metadata...}
           │
           ├─→ 7. Update StatusLog:
           │      statusLog.upsert_document(status="Indexed", state=State.COMPLETE)
           │
           └─→ 8. Error handling:
                 - Requeue with backoff (max 3 attempts)
                 - Log error to Cosmos DB
                 - Continue processing next messages

[Performance Characteristics]

- Polling Interval: 5 seconds (configurable)
- Batch Size: Up to 120 chunks (MAX_CHUNKS_PER_BATCH)
- Payload Limit: 12 MB (MAX_PAYLOAD_BYTES)
- Retry Logic: Exponential backoff, max 5 attempts
- Concurrency: Single-threaded queue processing (daemon thread)
- Embedding Model: text-embedding-3-small (1536 dimensions)
- Throughput: ~100-200 chunks/minute (depending on queue depth)
```

**Evidence**: app/enrichment/app.py (lines 1-400)  
**Key Features**:
- Flask REST API for on-demand embedding generation
- Background queue polling thread (5-second interval)
- Batching with size limits (120 chunks, 12 MB)
- Retry logic with exponential backoff
- RBAC-aware index routing
- StatusLog integration for audit trail

---

## Diagram 8: Azure Functions Orchestration (12 Functions)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                  Azure Function App: infoasst-func-dev2                       │
│                  Runtime: Python 3.11, Azure Functions v4                     │
│                  Hosting: ElasticPremium EP1 (auto-scale)                     │
│                  Location: functions/ directory                               │
└────────────────────────────────────────────────────────────────────────────────┘

[Document Processing Pipeline - 6 Stages]

Stage 1: Upload & Routing
───────────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: FileUploadedEtrigger                                     │
│  Location: functions/FileUploadedEtrigger/__init__.py               │
│  Trigger: Blob trigger on upload-* containers                       │
│  Purpose: Route uploaded files to appropriate processing queue      │
│                                                                      │
│  Logic (EVA-DA-Pipeline-Analysis-Evidence-Based.md lines 50-90):   │
│    file_extension = os.path.splitext(blob_name)[1].lower()         │
│    if file_extension == ".pdf":                                     │
│      queue_name = "pdf-submit-queue"                                │
│    elif file_extension in [".mp3", ".wav", ".mp4", ".avi"]:        │
│      queue_name = "media-submit-queue"                              │
│    else:                                                             │
│      queue_name = "non-pdf-submit-queue"                            │
│    queue_client.send_message(json.dumps({                           │
│      "blob_name": blob_name,                                        │
│      "file_extension": file_extension,                              │
│      "upload_container": container_name                             │
│    }))                                                               │
│                                                                      │
│  Output: Message to queue (pdf-submit-queue, non-pdf-submit-queue, │
│          media-submit-queue)                                         │
└─────────────────────────────────────────────────────────────────────┘

Stage 2a: PDF Processing
─────────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: FileFormRecSubmissionPDF                                 │
│  Location: functions/FileFormRecSubmissionPDF/__init__.py           │
│  Trigger: Queue trigger on pdf-submit-queue                         │
│  Purpose: OCR PDF files using Document Intelligence                 │
│                                                                      │
│  Logic:                                                              │
│    1. Download PDF from upload-container                            │
│    2. Call Document Intelligence API (prebuilt-read model)          │
│    3. Extract:                                                       │
│       - Text content (full document)                                │
│       - Page numbers                                                │
│       - Tables (if present)                                         │
│       - Layout information                                          │
│    4. Save extracted text to intermediate blob                      │
│    5. Send message to text-enrichment-queue                         │
│                                                                      │
│  Output: Message to text-enrichment-queue                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Function: FileLayoutParsingPDF                                     │
│  Location: functions/FileLayoutParsingPDF/__init__.py               │
│  Trigger: Queue trigger on pdf-submit-queue (parallel processing)   │
│  Purpose: Advanced PDF layout parsing (alternative to FormRec)      │
│                                                                      │
│  Logic:                                                              │
│    - Similar to FileFormRecSubmissionPDF                            │
│    - Uses different PDF parsing library (pypdf, pdfminer)           │
│    - Fallback option if Document Intelligence unavailable           │
│                                                                      │
│  Output: Message to text-enrichment-queue                           │
└─────────────────────────────────────────────────────────────────────┘

Stage 2b: Non-PDF Processing
─────────────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: FileLayoutParsingOther                                   │
│  Location: functions/FileLayoutParsingOther/__init__.py             │
│  Trigger: Queue trigger on non-pdf-submit-queue                     │
│  Purpose: Parse non-PDF documents (DOCX, TXT, XML, HTML, etc.)     │
│                                                                      │
│  Supported Formats (lines 1-50):                                    │
│    - .docx, .doc (Microsoft Word)                                   │
│    - .txt (plain text)                                              │
│    - .html, .htm (HTML)                                             │
│    - .xml (XML)                                                      │
│    - .md (Markdown)                                                  │
│    - .csv (CSV)                                                      │
│                                                                      │
│  Chunking Algorithm (lines 470-490):                                │
│    text_splitter = RecursiveCharacterTextSplitter(                  │
│      separators=["\n\n", "\n", ".", ",", " ", ""],                 │
│      chunk_size=1500,                                                │
│      chunk_overlap=100                                               │
│    )                                                                 │
│    chunks = text_splitter.split_text(full_text)                     │
│                                                                      │
│  Logic:                                                              │
│    1. Download file from upload-container                           │
│    2. Detect file type and parse accordingly                        │
│    3. Extract text content                                          │
│    4. Split into chunks (1500 chars, 100 overlap)                   │
│    5. Save chunks to intermediate blob                              │
│    6. Send message to text-enrichment-queue                         │
│                                                                      │
│  Output: Message to text-enrichment-queue                           │
└─────────────────────────────────────────────────────────────────────┘

Stage 2c: Media Processing
───────────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: MediaSubmission                                          │
│  Location: functions/MediaSubmission/__init__.py                    │
│  Trigger: Queue trigger on media-submit-queue                       │
│  Purpose: Process audio/video files                                 │
│                                                                      │
│  Logic:                                                              │
│    1. Download media file from upload-container                     │
│    2. Extract audio track (if video)                                │
│    3. Transcribe using Azure Speech Services or OpenAI Whisper      │
│    4. Save transcript to intermediate blob                          │
│    5. Send message to text-enrichment-queue                         │
│                                                                      │
│  Output: Message to text-enrichment-queue                           │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Function: ImageEnrichment                                          │
│  Location: functions/ImageEnrichment/__init__.py                    │
│  Trigger: Queue trigger on image-enrichment-queue                   │
│  Purpose: Extract text from images (OCR)                            │
│                                                                      │
│  Logic:                                                              │
│    1. Download image from upload-container                          │
│    2. Call Document Intelligence API (OCR)                          │
│    3. Extract text content                                          │
│    4. Save extracted text to intermediate blob                      │
│    5. Send message to text-enrichment-queue                         │
│                                                                      │
│  Output: Message to text-enrichment-queue                           │
└─────────────────────────────────────────────────────────────────────┘

Stage 3: Text Enrichment
─────────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: TextEnrichment                                           │
│  Location: functions/TextEnrichment/__init__.py                     │
│  Trigger: Queue trigger on text-enrichment-queue                    │
│  Purpose: Extract metadata and prepare for embedding                │
│                                                                      │
│  Logic (EVA-DA-Pipeline-Analysis-Evidence-Based.md lines 200-400): │
│    1. Download extracted text from intermediate blob                │
│    2. Extract metadata:                                              │
│       - file_name, folder, file_class                               │
│       - chunk_id, chunk_file (per chunk)                            │
│       - page numbers                                                 │
│       - language detection                                          │
│    3. Canadian regulation detection:                                │
│       - Check for patterns: "CAN/", "CSA Z", "BNQ", etc.            │
│       - Set file_class = "authority" if detected                    │
│    4. Prepare chunks for embedding:                                 │
│       - Each chunk: {id, content, metadata}                         │
│    5. Send chunks to embeddings-queue                               │
│                                                                      │
│  Metadata Extracted:                                                 │
│    - id: "{file_name}_chunk_{chunk_number}"                         │
│    - file_name: Original filename                                   │
│    - chunk_file: Chunk-specific identifier                          │
│    - folder: Upload folder path                                     │
│    - file_class: "general" or "authority"                           │
│    - pages: [page_numbers]                                          │
│    - content: Chunk text content                                    │
│    - title: Document title (if available)                           │
│    - translated_title: (if bilingual)                               │
│                                                                      │
│  Output: Messages to embeddings-queue (one per chunk)               │
└─────────────────────────────────────────────────────────────────────┘

Stage 4: Embedding Generation
──────────────────────────────
[Handled by Enrichment Service - see Diagram 6]
- Poll embeddings-queue (5-second interval)
- Generate embeddings via Azure OpenAI
- Upload to Cognitive Search with metadata

Stage 5: Reprocessing
──────────────────────
┌─────────────────────────────────────────────────────────────────────┐
│  Function: ProcessingQueue                                          │
│  Location: functions/ProcessingQueue/__init__.py                    │
│  Trigger: Queue trigger on resubmit-processing-queue                │
│  Purpose: Reprocess failed or resubmitted files                     │
│                                                                      │
│  Logic:                                                              │
│    1. Receive resubmit request from backend (/resubmitItems)        │
│    2. Reset StatusLog entry to State.QUEUED                         │
│    3. Route to appropriate queue based on file extension            │
│    4. Continue with normal pipeline flow                            │
│                                                                      │
│  Output: Message to appropriate queue (pdf-submit-queue, etc.)      │
└─────────────────────────────────────────────────────────────────────┘

[Queue Routing Tree]

                    Blob Upload (upload-* container)
                              │
                              │ Blob Trigger
                              ▼
                    FileUploadedEtrigger
                              │
                              │ Route by file extension
                ┌─────────────┼─────────────┐
                │             │             │
                ▼             ▼             ▼
        pdf-submit-queue  non-pdf-submit  media-submit-queue
                │         -queue               │
        ┌───────┴────────┐    │        ┌───────┴───────┐
        ▼                ▼    ▼        ▼               ▼
  FileFormRec-   FileLayout-  FileLayout-  MediaSubmission  ImageEnrichment
  SubmissionPDF  ParsingPDF   ParsingOther     │              │
        │                │    │                │              │
        └────────────────┴────┴────────────────┴──────────────┘
                              │
                              │ All converge to
                              ▼
                    text-enrichment-queue
                              │
                              │ Queue Trigger
                              ▼
                        TextEnrichment
                              │
                              │ Send chunks
                              ▼
                      embeddings-queue
                              │
                              │ Polled by Enrichment Service
                              ▼
                    Cognitive Search (indexed)

[Error Handling & Monitoring]

Each function implements:
- StatusLog updates (Cosmos DB) at each stage
- Retry logic with exponential backoff
- Error messages to resubmit-processing-queue
- Application Insights telemetry
- Dead-letter queue for permanently failed items
```

**Evidence**: 
- EVA-DA-Pipeline-Analysis-Evidence-Based.md (lines 1-680)
- functions/FileLayoutParsingOther/__init__.py (lines 470-490)

**Key Features**:
- 12 Azure Functions forming 6-stage pipeline
- Blob trigger + 7 queue triggers
- Fixed chunking: 1500 characters, 100 overlap
- Canadian regulation detection (authority classification)
- RBAC-aware container routing
- StatusLog audit trail at every stage

---

**Next Section**: [Document Ingestion Pipeline](dev2-TechDoc-03-doc-ingestion.md) - Detailed flowcharts for all 6 stages
