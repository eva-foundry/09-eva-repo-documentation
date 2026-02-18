# Document Ingestion Pipeline
## 6-Stage Processing Flow

**Evidence Sources**:
- EVA-DA-Pipeline-Analysis-Evidence-Based.md (680 lines)
- functions/FileLayoutParsingOther/__init__.py (lines 470-490)
- azure_search/create_vector_index.json (298 lines)

---

## Diagram 9: 6-Stage Ingestion Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Stage 1: Upload & Routing                             │
│                        FileUploadedEtrigger                                  │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
        User Upload → Blob Storage (upload-container)
                                    │
                                    │ Blob Trigger
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │ FileUploadedEtrigger/__init__.py                  │
        │                                                    │
        │ file_extension = os.path.splitext(blob_name)[1]   │
        │                                                    │
        │ Route Logic:                                       │
        │   if extension == ".pdf":                          │
        │     → pdf-submit-queue                             │
        │   elif extension in [".mp3", ".wav", ".mp4"]:      │
        │     → media-submit-queue                           │
        │   else:                                            │
        │     → non-pdf-submit-queue                         │
        └───────────────────────────┬───────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────┐
        │                           │                       │
        ▼                           ▼                       ▼
┌───────────────┐         ┌───────────────┐       ┌───────────────┐
│ pdf-submit-   │         │ non-pdf-      │       │ media-submit- │
│ queue         │         │ submit-queue  │       │ queue         │
└───────┬───────┘         └───────┬───────┘       └───────┬───────┘
        │                         │                       │
        │                         │                       │
┌───────────────────────────────────────────────────────────────────────────────┐
│                        Stage 2: Format-Specific Processing                    │
└───────────────────────────────────┬───────────────────────────────────────────┘
        │                         │                       │
        ▼                         ▼                       ▼
┌───────────────┐         ┌───────────────┐       ┌───────────────┐
│ PDF Path      │         │ Non-PDF Path  │       │ Media Path    │
└───────┬───────┘         └───────┬───────┘       └───────┬───────┘
        │                         │                       │
        ▼                         ▼                       ▼

[PDF Processing Branch]
┌─────────────────────────────────────────────────────────────┐
│ FileFormRecSubmissionPDF or FileLayoutParsingPDF            │
│                                                              │
│ 1. Download PDF from upload-container                       │
│ 2. Call Document Intelligence API:                          │
│    - Endpoint: infoasst-docint-dev2.cognitiveservices...   │
│    - Model: prebuilt-read                                   │
│    - Request: POST /formrecognizer/v3.0/read:analyze       │
│ 3. Extract:                                                  │
│    - Full text content                                      │
│    - Page numbers (for each text block)                     │
│    - Tables (detected and extracted)                        │
│    - Layout information (bounding boxes)                    │
│ 4. Save intermediate results:                               │
│    - Blob: {file_name}_extracted.json                       │
│    - Contains: pages[], content, metadata                   │
│ 5. Update StatusLog:                                        │
│    - State: PROCESSING                                      │
│    - Status: "OCR completed"                                │
│ 6. Send to text-enrichment-queue                            │
└─────────────────────────────────────────────────────────────┘

[Non-PDF Processing Branch]
┌─────────────────────────────────────────────────────────────┐
│ FileLayoutParsingOther/__init__.py                          │
│                                                              │
│ Supported Formats:                                          │
│   - DOCX, DOC → python-docx library                         │
│   - TXT → direct read                                       │
│   - HTML, HTM → BeautifulSoup parsing                       │
│   - XML → lxml parsing                                      │
│   - MD → Markdown parser                                    │
│   - CSV → pandas.read_csv()                                 │
│                                                              │
│ Processing Steps:                                           │
│ 1. Download file from upload-container                      │
│ 2. Detect MIME type and select parser                       │
│ 3. Extract text content                                     │
│ 4. Chunking (lines 470-490):                                │
│    text_splitter = RecursiveCharacterTextSplitter(          │
│      separators=["\n\n", "\n", ".", ",", " ", ""],         │
│      chunk_size=1500,                                        │
│      chunk_overlap=100                                       │
│    )                                                         │
│    chunks = text_splitter.split_text(full_text)             │
│                                                              │
│ 5. Metadata per chunk:                                      │
│    - chunk_id: 0, 1, 2, ...                                 │
│    - page: Estimated from chunk position                    │
│    - content: Chunk text (up to 1500 chars)                 │
│ 6. Save intermediate results                                │
│ 7. Update StatusLog                                         │
│ 8. Send to text-enrichment-queue                            │
└─────────────────────────────────────────────────────────────┘

[Media Processing Branch]
┌─────────────────────────────────────────────────────────────┐
│ MediaSubmission/__init__.py                                 │
│                                                              │
│ 1. Download media file from upload-container                │
│ 2. Extract audio:                                           │
│    - If video: Extract audio track (ffmpeg)                 │
│    - If audio: Use directly                                 │
│ 3. Transcription:                                           │
│    - Azure Speech Services (preferred)                      │
│    - Or OpenAI Whisper API                                  │
│    - Request: POST /speech/transcribe                       │
│ 4. Format transcript:                                       │
│    - Timestamps for each segment                            │
│    - Speaker identification (if available)                  │
│    - Confidence scores                                      │
│ 5. Save transcript to intermediate blob                     │
│ 6. Update StatusLog                                         │
│ 7. Send to text-enrichment-queue                            │
└─────────────────────────────────────────────────────────────┘

        │                         │                       │
        └─────────────────────────┼───────────────────────┘
                                  │ All converge
                                  ▼
                        text-enrichment-queue

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Stage 3: Text Enrichment                              │
│                        TextEnrichment/__init__.py                            │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│ TextEnrichment Function                                     │
│ (EVA-DA-Pipeline-Analysis-Evidence-Based.md lines 200-400) │
│                                                              │
│ 1. Dequeue message from text-enrichment-queue               │
│    Message contains:                                        │
│      - file_path: "{upload_container}/{file_name}"          │
│      - intermediate_blob: "{file_name}_extracted.json"      │
│      - upload_container: "upload-groupA"                    │
│                                                              │
│ 2. Download intermediate results blob                       │
│    Contains: pages[], chunks[], raw_content                 │
│                                                              │
│ 3. Metadata Extraction:                                     │
│    For each chunk:                                          │
│      - id: "{file_name}_chunk_{chunk_id}"                   │
│      - file_name: Original filename                         │
│      - chunk_file: Chunk-specific ID                        │
│      - folder: Extract from file_path (before filename)     │
│      - pages: [page_numbers] (from Document Intelligence)   │
│      - content: Chunk text (max 1500 chars)                 │
│                                                              │
│ 4. Language Detection:                                      │
│    - Detect primary language (en, fr, etc.)                 │
│    - Set metadata field: "language": "en"                   │
│                                                              │
│ 5. Canadian Regulation Detection:                           │
│    Patterns checked (lines 250-300):                        │
│      - "CAN/" (Canadian standards prefix)                   │
│      - "CSA Z" (CSA standards)                              │
│      - "BNQ" (Bureau de normalisation du Québec)            │
│      - "NRC" (National Research Council)                    │
│      - Other authority patterns                             │
│    If detected:                                             │
│      - file_class = "authority"                             │
│    Else:                                                     │
│      - file_class = "general"                               │
│                                                              │
│ 6. Tag Extraction:                                          │
│    - Read blob metadata for "tags" field                    │
│    - Parse comma-separated tags                             │
│    - URL-decode tags                                        │
│    - Add to chunk metadata: tags: ["tag1", "tag2"]          │
│                                                              │
│ 7. Bilingual Content Handling:                              │
│    If file_class == "authority":                            │
│      - Detect EN/FR sections                                │
│      - Create separate fields:                              │
│        - title_en, content_en                               │
│        - title_fr, content_fr                               │
│                                                              │
│ 8. Prepare chunk for embedding:                             │
│    chunk_message = {                                        │
│      "id": "{file_name}_chunk_{chunk_id}",                  │
│      "file_name": "document.pdf",                           │
│      "chunk_file": "document_chunk_0",                      │
│      "folder": "legal/contracts",                           │
│      "file_class": "general",                               │
│      "pages": [1, 2],                                       │
│      "content": "Chunk text content...",                    │
│      "title": "Document Title",                             │
│      "tags": ["contract", "2024"],                          │
│      "upload_container": "upload-groupA"                    │
│    }                                                         │
│                                                              │
│ 9. Send chunk to embeddings-queue                           │
│    - One message per chunk                                  │
│    - Message encoding: JSON                                 │
│                                                              │
│ 10. Update StatusLog:                                       │
│     - State: ENRICHING                                      │
│     - Status: "Metadata extracted, {N} chunks queued"       │
└─────────────────────────────────────────────────────────────┘
                                  │
                                  │ One message per chunk
                                  ▼
                        embeddings-queue
                        (contains all chunks)

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Stage 4: Embedding Generation                         │
│                        Enrichment Service (Queue Polling)                    │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────┐
│ Enrichment Service poll_queue() - 5-second interval         │
│ (app/enrichment/app.py lines 350-500)                       │
│                                                              │
│ 1. Dequeue batch of messages (up to 120 chunks):            │
│    - DEQUEUE_MESSAGE_BATCH_SIZE = 1 (sequential)            │
│    - But processes up to 120 chunks before flushing         │
│                                                              │
│ 2. Extract chunk data from each message:                    │
│    - content: Text to embed                                 │
│    - metadata: All fields from TextEnrichment               │
│    - upload_container: For RBAC lookup                      │
│                                                              │
│ 3. RBAC-aware Index Routing:                                │
│    content_container = find_content_container_and_role_...( │
│      upload_container, group_items)                         │
│    search_index = find_index_and_role_based_on_upcontainer( │
│      upload_container, group_items)                         │
│    # Example: upload-groupA → content-groupA → index-groupA │
│                                                              │
│ 4. Generate embeddings:                                     │
│    Call Azure OpenAI (via /models/{model}/embed endpoint):  │
│      model = "azure-openai_text-embedding-3-small"          │
│      texts = [chunk1.content, chunk2.content, ...]          │
│      response = client.embeddings.create(                   │
│        model="text-embedding-3-small",                      │
│        input=texts                                          │
│      )                                                       │
│      embeddings = response.data[0].embedding  # 1536-dim    │
│                                                              │
│ 5. Assemble chunk documents:                                │
│    For each chunk:                                          │
│      chunk_doc = {                                          │
│        "id": chunk.id,                                      │
│        "file_name": chunk.file_name,                        │
│        "chunk_file": chunk.chunk_file,                      │
│        "folder": chunk.folder,                              │
│        "file_class": chunk.file_class,                      │
│        "pages": chunk.pages,                                │
│        "content": chunk.content,                            │
│        "contentVector": embeddings,  # 1536-dim float[]    │
│        "title": chunk.title,                                │
│        "tags": chunk.tags,                                  │
│        "@search.action": "mergeOrUpload"                    │
│      }                                                       │
│                                                              │
│ 6. Batch accumulation with size limits:                     │
│    batch = []                                               │
│    for chunk_doc in chunk_docs:                             │
│      if len(batch) >= 120:  # MAX_CHUNKS_PER_BATCH         │
│        _flush_batch(batch, search_index)                    │
│        batch.clear()                                        │
│      if _payload_bytes(batch + [chunk_doc]) > 12MB:        │
│        _flush_batch(batch, search_index)                    │
│        batch.clear()                                        │
│      batch.append(chunk_doc)                                │
│    _flush_batch(batch, search_index)  # Final flush         │
│                                                              │
│ 7. Update StatusLog:                                        │
│    - State: INDEXING                                        │
│    - Status: "Embeddings generated, uploading to index"     │
└─────────────────────────────────────────────────────────────┘
                                        │
                                        │ Batched upload
                                        ▼
                        Cognitive Search Index

┌─────────────────────────────────────────────────────────────────────────────┐
│                        Stage 5: Search Index Upload                          │
│                        index_sections() function                             │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────┐
│ Upload to Cognitive Search                                  │
│ (app/enrichment/app.py lines 450-470)                       │
│                                                              │
│ search_client = SearchClient(                               │
│   endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"],            │
│   index_name=search_index,  # e.g., "index-groupA"          │
│   credential=azure_credential                               │
│ )                                                            │
│                                                              │
│ results = search_client.upload_documents(documents=batch)   │
│                                                              │
│ Response handling:                                          │
│   succeeded = sum([1 for r in results if r.succeeded])      │
│   failed = sum([1 for r in results if not r.succeeded])     │
│   log.debug(f"Indexed {len(results)} chunks, {succeeded} succeeded")│
│                                                              │
│ Error handling:                                             │
│   - If any failed: Log errors                               │
│   - Requeue failed chunks (max 3 attempts)                  │
│   - Update StatusLog with failure details                   │
└─────────────────────────────────────────────────────────────┘
                                        │
                                        │ Index updated
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Stage 6: Completion & Audit                           │
│                        StatusLog Update                                      │
└───────────────────────────────────────┬─────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────┐
│ Final StatusLog Update (Cosmos DB)                          │
│                                                              │
│ statusLog.upsert_document(                                  │
│   document_path="{upload_container}/{file_name}",           │
│   status="Indexing complete",                               │
│   status_classification=StatusClassification.INFO,          │
│   state=State.COMPLETE                                      │
│ )                                                            │
│ statusLog.save_document(document_path)                      │
│                                                              │
│ Document now searchable via:                                │
│   - Backend /chat endpoint (RAG queries)                    │
│   - Cognitive Search API (direct queries)                   │
│                                                              │
│ Total Time: ~30-120 seconds per document                    │
│   - Stage 1: <1 second (routing)                            │
│   - Stage 2: 5-30 seconds (OCR/parsing)                     │
│   - Stage 3: 2-5 seconds (enrichment)                       │
│   - Stage 4: 10-60 seconds (embedding generation)           │
│   - Stage 5: 5-10 seconds (index upload)                    │
│   - Stage 6: <1 second (status update)                      │
└─────────────────────────────────────────────────────────────┘

[Success Criteria]
✅ Document uploaded to upload-container
✅ Routed to appropriate queue
✅ Processed by format-specific function
✅ Text extracted and chunked (1500 chars)
✅ Metadata enriched (file_class, tags, etc.)
✅ Embeddings generated (1536-dim vectors)
✅ Uploaded to Cognitive Search index
✅ StatusLog shows State.COMPLETE
✅ Document searchable in chat interface
```

**Evidence**: 
- EVA-DA-Pipeline-Analysis-Evidence-Based.md (lines 1-680)
- functions/FileLayoutParsingOther/__init__.py (lines 470-490)
- app/enrichment/app.py (lines 350-500)

---

## Diagram 10: Queue Routing Tree

```
                        Blob Upload Event
                              │
                              │ Azure Event Grid
                              ▼
              ┌───────────────────────────────────┐
              │  FileUploadedEtrigger             │
              │  Blob Trigger                     │
              └───────────────┬───────────────────┘
                              │
                              │ file_extension = os.path.splitext(blob_name)[1].lower()
                              │
              ┌───────────────┼───────────────────────────────┐
              │               │                               │
              │ .pdf          │ .docx, .txt, .html, .xml     │ .mp3, .wav, .mp4, .avi
              ▼               ▼                               ▼
    ┌─────────────────┐ ┌─────────────────┐       ┌─────────────────┐
    │ pdf-submit-     │ │ non-pdf-submit- │       │ media-submit-   │
    │ queue           │ │ queue           │       │ queue           │
    └────────┬────────┘ └────────┬────────┘       └────────┬────────┘
             │                   │                         │
             │                   │                         │
    ┌────────┴────────┐          │                ┌────────┴────────┐
    │                 │          │                │                 │
    ▼                 ▼          ▼                ▼                 ▼
┌─────────┐     ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│FileForm-│     │FileLayout│ │FileLayout│  │Media-    │  │Image-    │
│RecSubmis│     │ParsingPDF│ │Parsing-  │  │Submission│  │Enrichment│
│sionPDF  │     │          │ │Other     │  │          │  │          │
└────┬────┘     └────┬─────┘ └────┬─────┘  └────┬─────┘  └────┬─────┘
     │               │            │             │             │
     │ OCR result    │ Parsed     │ Chunked     │ Transcript  │ OCR text
     │               │ text       │ text        │             │
     └───────────────┴────────────┴─────────────┴─────────────┘
                              │
                              │ All converge
                              ▼
                ┌───────────────────────────────────┐
                │  text-enrichment-queue            │
                │  (intermediate processing results)│
                └───────────────┬───────────────────┘
                                │
                                │ Queue Trigger
                                ▼
                ┌───────────────────────────────────┐
                │  TextEnrichment Function          │
                │  - Extract metadata               │
                │  - Detect language                │
                │  - Classify file_class            │
                │  - Parse tags                     │
                └───────────────┬───────────────────┘
                                │
                                │ One message per chunk
                                ▼
                ┌───────────────────────────────────┐
                │  embeddings-queue                 │
                │  (chunks ready for embedding)     │
                └───────────────┬───────────────────┘
                                │
                                │ Polled every 5 seconds
                                ▼
                ┌───────────────────────────────────┐
                │  Enrichment Service               │
                │  (Flask app with background worker)│
                └───────────────┬───────────────────┘
                                │
                                ├─→ Generate embeddings (Azure OpenAI)
                                ├─→ Batch chunks (max 120, 12MB)
                                └─→ Upload to Cognitive Search
                                      │
                                      ▼
                ┌───────────────────────────────────┐
                │  Cognitive Search Index           │
                │  (searchable documents)           │
                └───────────────────────────────────┘

[Error & Reprocessing Queues]

                Any Stage Failure
                      │
                      │ Error detected
                      ▼
        ┌───────────────────────────────────┐
        │  resubmit-processing-queue        │
        │  (retry queue)                    │
        └───────────────┬───────────────────┘
                        │
                        │ Queue Trigger
                        ▼
        ┌───────────────────────────────────┐
        │  ProcessingQueue Function         │
        │  - Check retry count              │
        │  - Reset StatusLog                │
        │  - Re-route to appropriate queue  │
        └───────────────┬───────────────────┘
                        │
                        │ If retry_count < MAX_REQUEUE_COUNT (3)
                        ▼
        Route back to original queue (pdf-submit-queue, etc.)
                        │
                        │ If retry_count >= MAX_REQUEUE_COUNT
                        ▼
        ┌───────────────────────────────────┐
        │  Dead-Letter Queue                │
        │  (permanently failed items)       │
        │  Manual intervention required     │
        └───────────────────────────────────┘

[Queue Configuration]

Queue Name                  │ Message TTL │ Max Delivery Count │ Visibility Timeout
────────────────────────────┼─────────────┼────────────────────┼───────────────────
pdf-submit-queue            │ 7 days      │ 10                 │ 300 seconds (5 min)
non-pdf-submit-queue        │ 7 days      │ 10                 │ 300 seconds
media-submit-queue          │ 7 days      │ 10                 │ 600 seconds (10 min)
pdf-polling-queue           │ 7 days      │ 10                 │ 300 seconds
image-enrichment-queue      │ 7 days      │ 10                 │ 300 seconds
text-enrichment-queue       │ 7 days      │ 10                 │ 300 seconds
embeddings-queue            │ 7 days      │ 10                 │ 600 seconds
resubmit-processing-queue   │ 30 days     │ 3                  │ 900 seconds (15 min)

[Message Format Examples]

pdf-submit-queue message:
{
  "blob_name": "legal/contract.pdf",
  "file_extension": ".pdf",
  "upload_container": "upload-groupA",
  "retry_count": 0
}

text-enrichment-queue message:
{
  "file_path": "upload-groupA/legal/contract.pdf",
  "intermediate_blob": "contract_extracted.json",
  "upload_container": "upload-groupA",
  "pages_count": 10
}

embeddings-queue message:
{
  "id": "contract_chunk_5",
  "file_name": "contract.pdf",
  "chunk_file": "contract_chunk_5",
  "folder": "legal",
  "file_class": "general",
  "pages": [5, 6],
  "content": "This agreement is entered into...",
  "title": "Service Agreement",
  "tags": ["contract", "legal", "2024"],
  "upload_container": "upload-groupA"
}
```

**Evidence**: EVA-DA-Pipeline-Analysis-Evidence-Based.md (lines 50-200)

---

## Diagram 11: Chunking Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Chunking Algorithm: RecursiveCharacterTextSplitter              │
│              Location: functions/FileLayoutParsingOther/__init__.py          │
│              Lines: 470-490                                                  │
└─────────────────────────────────────────────────────────────────────────────┘

Input: Full document text (extracted from PDF, DOCX, etc.)
  │
  │ Example: "This is a sample document.\n\nIt has multiple paragraphs.\nAnd some
  │           sentences. Here's another one, with commas. And more text..."
  │           (Length: 5000 characters)
  ▼

┌─────────────────────────────────────────────────────────────────┐
│ Configuration (lines 477-482):                                  │
│                                                                  │
│   text_splitter = RecursiveCharacterTextSplitter(               │
│     separators=["\n\n", "\n", ".", ",", " ", ""],              │
│     chunk_size=1500,                                             │
│     chunk_overlap=100                                            │
│   )                                                              │
│                                                                  │
│ Parameters:                                                      │
│   - separators: Hierarchical list of split points               │
│     1. "\n\n" → Paragraph boundaries (preferred)                 │
│     2. "\n"   → Line boundaries                                  │
│     3. "."    → Sentence boundaries                              │
│     4. ","    → Clause boundaries                                │
│     5. " "    → Word boundaries                                  │
│     6. ""     → Character boundaries (last resort)               │
│   - chunk_size: 1500 characters (target size)                   │
│   - chunk_overlap: 100 characters (context preservation)        │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Attempt Split with First Separator ("\n\n")             │
│                                                                  │
│   text = "Para1 text...\n\nPara2 text...\n\nPara3 text..."     │
│   splits = text.split("\n\n")                                   │
│   → ["Para1 text...", "Para2 text...", "Para3 text..."]         │
│                                                                  │
│   For each split:                                               │
│     if len(split) <= 1500:                                      │
│       Add to current chunk                                      │
│     else:                                                        │
│       Recursively split with next separator ("\n")              │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Recursive Splitting with Fallback Separators            │
│                                                                  │
│   If a segment > 1500 chars:                                    │
│     Try next separator ("\n")                                   │
│     If still > 1500:                                            │
│       Try next separator (".")                                  │
│     If still > 1500:                                            │
│       Try next separator (",")                                  │
│     If still > 1500:                                            │
│       Try next separator (" ")                                  │
│     If still > 1500:                                            │
│       Force split at character boundary ("")                    │
│                                                                  │
│   Example: Long paragraph (2000 chars)                          │
│     1. Split by "\n" → Multiple lines                           │
│     2. Combine lines until ~1500 chars                          │
│     3. Create Chunk 1: 1450 chars (lines 1-15)                  │
│     4. Create Chunk 2: 550 chars (lines 14-20, with 100 overlap)│
└─────────────────────────────────────────────────────────────────┘
  │
  ▼

┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Apply Overlap (100 characters)                          │
│                                                                  │
│   Chunk 1: [chars 0-1450]                                       │
│   Overlap: [chars 1350-1450] (last 100 chars of Chunk 1)        │
│   Chunk 2: [chars 1350-2850] (starts with overlap)              │
│   Overlap: [chars 2750-2850]                                    │
│   Chunk 3: [chars 2750-4250]                                    │
│   ...                                                            │
│                                                                  │
│   Why overlap?                                                  │
│     - Preserve context across chunk boundaries                  │
│     - Ensure sentences/concepts not cut mid-thought             │
│     - Improve RAG retrieval accuracy                            │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼

┌─────────────────────────────────────────────────────────────────┐
│ Output: List of Chunks                                          │
│                                                                  │
│   chunks = text_splitter.split_text(full_text)                  │
│   → [                                                            │
│       "Chunk 0: This is a sample document...",  (1450 chars)    │
│       "Chunk 1: ...ment.\n\nIt has multiple...", (1480 chars)   │
│       "Chunk 2: ...aragraphs.\nAnd some...",     (1420 chars)   │
│       "Chunk 3: ...sentences. Here's...",       (1070 chars)    │
│     ]                                                            │
│                                                                  │
│   Total chunks: 4                                               │
│   Total characters: 5000                                        │
│   Average chunk size: 1355 chars                                │
│   Overlap per chunk: 100 chars                                  │
└─────────────────────────────────────────────────────────────────┘

[Visualization: Chunk Boundaries with Overlap]

Document: [================================================] 5000 chars

Chunk 0:  [======================]                          (0-1450)
Overlap:                     [==]                           (1350-1450)
Chunk 1:                     [======================]       (1350-2850)
Overlap:                                        [==]        (2750-2850)
Chunk 2:                                        [==========](2750-4250)
Overlap:                                                [==](4150-4250)
Chunk 3:                                                [==](4150-5000)

Key:
  [====] Chunk content
  [==]   Overlapping region (shared between chunks)

[Edge Cases Handled]

1. Very Short Documents (<1500 chars):
   - Single chunk, no splitting
   - Overlap not applied

2. Very Long Paragraphs (>1500 chars):
   - Fallback to sentence boundaries (".")
   - Then clause boundaries (",")
   - Finally word boundaries (" ")

3. No Natural Boundaries:
   - Force split at character boundary ("")
   - Ensures chunk_size limit respected

4. Empty Documents:
   - Return empty list []
   - Skip processing

5. Unicode/Special Characters:
   - Handled correctly (UTF-8 encoding)
   - No truncation mid-character

[Performance Characteristics]

- Average Time: 10-50ms per 1000 characters
- Memory: O(n) where n = document size
- Chunk Count: ⌈(doc_length - chunk_overlap) / (chunk_size - chunk_overlap)⌉
- Example: 10,000 char doc → ~7-8 chunks

[Quality Metrics]

✅ Preserves sentence boundaries (preferred)
✅ Maintains context with overlap
✅ Respects chunk_size limit (max 1500 chars)
✅ Consistent chunk sizes (avg ~1400 chars)
✅ Minimal information loss
```

**Evidence**: functions/FileLayoutParsingOther/__init__.py (lines 470-490)

---

## Diagram 12: Embedding Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Embedding Generation Sequence Diagram                      │
│                   Enrichment Service → Azure OpenAI                          │
└─────────────────────────────────────────────────────────────────────────────┘

Enrichment Service          Embeddings Queue          Azure OpenAI           Search Index
     (Flask)              (Storage Queue)          (text-embedding-3-small)  (Cognitive Search)
        │                      │                          │                        │
        │                      │                          │                        │
        │ ┌──────────────────┐ │                          │                        │
        │ │ Background       │ │                          │                        │
        │ │ Thread:          │ │                          │                        │
        │ │ poll_queue()     │ │                          │                        │
        │ │ Every 5 seconds  │ │                          │                        │
        │ └────────┬─────────┘ │                          │                        │
        │          │            │                          │                        │
        │          ▼            │                          │                        │
        ├─────────────────────►│                          │                        │
        │  1. Dequeue messages │                          │                        │
        │  (batch up to 120)   │                          │                        │
        │◄─────────────────────┤                          │                        │
        │  Messages:           │                          │                        │
        │  [{chunk1}, {chunk2},│                          │                        │
        │   ..., {chunk120}]   │                          │                        │
        │                      │                          │                        │
        │ 2. Extract data      │                          │                        │
        │    from messages:    │                          │                        │
        │    - content (text)  │                          │                        │
        │    - metadata        │                          │                        │
        │    - upload_container│                          │                        │
        │                      │                          │                        │
        │ 3. RBAC lookup:      │                          │                        │
        │    upload_container  │                          │                        │
        │    → content_container                          │                        │
        │    → search_index    │                          │                        │
        │    (via group_items) │                          │                        │
        │                      │                          │                        │
        │ 4. Prepare embedding request                    │                        │
        │    texts = [chunk1.content, chunk2.content, ...]│                        │
        │                      │                          │                        │
        ├──────────────────────────────────────────────►│                        │
        │  POST /deployments/text-embedding-3-small/    │                        │
        │       embeddings/create                        │                        │
        │  Headers:                                      │                        │
        │    Authorization: Bearer {token}              │                        │
        │    Content-Type: application/json             │                        │
        │  Body: {                                       │                        │
        │    "input": [                                  │                        │
        │      "This is chunk 1 content...",             │                        │
        │      "This is chunk 2 content...",             │                        │
        │      ...                                       │                        │
        │    ],                                          │                        │
        │    "model": "text-embedding-3-small"           │                        │
        │  }                                             │                        │
        │                      │                         │                        │
        │                      │                     5. Process request           │
        │                      │                     (Azure OpenAI)               │
        │                      │                     - Tokenize text              │
        │                      │                     - Generate embeddings        │
        │                      │                     - 1536-dimensional vectors   │
        │                      │                         │                        │
        │◄─────────────────────────────────────────────┤                        │
        │  Response: {                                  │                        │
        │    "data": [                                  │                        │
        │      {                                        │                        │
        │        "embedding": [0.123, -0.456, ...],    │  (1536 floats)         │
        │        "index": 0                             │                        │
        │      },                                       │                        │
        │      {                                        │                        │
        │        "embedding": [-0.789, 0.234, ...],    │  (1536 floats)         │
        │        "index": 1                             │                        │
        │      },                                       │                        │
        │      ...                                      │                        │
        │    ],                                         │                        │
        │    "model": "text-embedding-3-small",         │                        │
        │    "usage": {                                 │                        │
        │      "prompt_tokens": 450,                    │                        │
        │      "total_tokens": 450                      │                        │
        │    }                                          │                        │
        │  }                                            │                        │
        │                      │                         │                        │
        │ 6. Assemble chunk documents                   │                        │
        │    for each chunk:   │                         │                        │
        │      chunk_doc = {   │                         │                        │
        │        "id": chunk.id,                         │                        │
        │        "content": chunk.content,               │                        │
        │        "contentVector": embedding[i],          │  (1536 floats)         │
        │        ...metadata   │                         │                        │
        │      }               │                         │                        │
        │                      │                         │                        │
        │ 7. Batch with size limits                     │                        │
        │    - Max 120 chunks per batch                 │                        │
        │    - Max 12 MB payload                        │                        │
        │                      │                         │                        │
        │ 8. Upload to Search Index                     │                        │
        ├────────────────────────────────────────────────────────────────────►│
        │  POST /indexes/{index-name}/docs/index       │                        │
        │  Headers:                                      │                        │
        │    api-key: {search-key}                       │                        │
        │    Content-Type: application/json             │                        │
        │  Body: {                                       │                        │
        │    "value": [                                  │                        │
        │      {                                         │                        │
        │        "@search.action": "mergeOrUpload",      │                        │
        │        "id": "doc_chunk_1",                    │                        │
        │        "content": "Chunk text...",             │                        │
        │        "contentVector": [0.123, -0.456, ...], │                        │
        │        "file_name": "document.pdf",            │                        │
        │        "folder": "legal",                      │                        │
        │        "pages": [1, 2],                        │                        │
        │        ...                                     │                        │
        │      },                                        │                        │
        │      ...                                       │                        │
        │    ]                                           │                        │
        │  }                                             │                        │
        │                      │                         │                        │
        │                      │                         │                    9. Index documents
        │                      │                         │                    - Store content
        │                      │                         │                    - Store vectors
        │                      │                         │                    - Update metadata
        │                      │                         │                    - Build HNSW graph
        │                      │                         │                        │
        │◄───────────────────────────────────────────────────────────────────────┤
        │  Response: {                                   │                        │
        │    "value": [                                  │                        │
        │      {                                         │                        │
        │        "key": "doc_chunk_1",                   │                        │
        │        "status": true,                         │                        │
        │        "statusCode": 200                       │                        │
        │      },                                        │                        │
        │      ...                                       │                        │
        │    ]                                           │                        │
        │  }                                             │                        │
        │                      │                         │                        │
        │ 10. Update StatusLog │                         │                        │
        │     (Cosmos DB)      │                         │                        │
        │     State: COMPLETE  │                         │                        │
        │                      │                         │                        │
        │ 11. Delete messages from queue                │                        │
        ├─────────────────────►│                         │                        │
        │  (acknowledge)       │                         │                        │
        │                      │                         │                        │
        │ 12. Loop back to Step 1                        │                        │
        │     (5-second delay) │                         │                        │
        │                      │                         │                        │

[Timing Details]

Step 1: Dequeue messages          → 50-100ms (Azure Queue Storage)
Step 2-3: Extract & RBAC lookup   → 10-20ms (in-memory cache)
Step 4: Prepare request            → <1ms
Step 5: Azure OpenAI embedding    → 500-2000ms (depends on batch size)
Step 6-7: Assemble & batch        → 10-50ms
Step 8-9: Upload to Search        → 200-1000ms (depends on batch size)
Step 10: Update StatusLog         → 50-100ms (Cosmos DB)
Step 11: Delete messages          → 50-100ms
──────────────────────────────────────────────────────────────────
Total per batch:                   ~1-4 seconds (for 120 chunks)

[Error Handling]

Azure OpenAI Errors:
- 429 (Rate Limit): Exponential backoff (5 retries max)
- 401/403: Token refresh, retry
- 500: Retry with backoff
- Timeout: Retry with backoff

Search Index Errors:
- 413 (Payload Too Large): Split batch, retry
- 503 (Service Unavailable): Retry with backoff
- Failed chunks: Requeue to embeddings-queue

Requeue Logic:
- If retry_count < MAX_EMBEDDING_REQUEUE_COUNT (3):
  - Add 60s * retry_count to visibility timeout
  - Requeue message
- Else:
  - Move to dead-letter queue
  - Update StatusLog with permanent failure
```

**Evidence**: app/enrichment/app.py (lines 1-500)

---

## Diagram 13: Blob Storage Containers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Storage Account: infoasststoredev2                              │
│              Endpoint: https://infoasststoredev2.blob.core.windows.net      │
│              Private Endpoints: 4 (blob, queue, file, table)                │
└─────────────────────────────────────────────────────────────────────────────┘

[Upload Containers - User File Uploads (20+ containers)]

┌────────────────────────────────────────────────────────────┐
│  upload-{group-name}                                       │
│  Purpose: Initial file uploads, RBAC-isolated             │
│  Access: Write (authenticated users), Read (functions)    │
│  Lifecycle: Retain 90 days, then move to archive tier     │
│                                                             │
│  Structure:                                                │
│    /{folder}/                                              │
│      {filename}.pdf                                        │
│      {filename}.docx                                       │
│      {filename}.txt                                        │
│                                                             │
│  Metadata (per blob):                                      │
│    - tags: "legal,contract,2024" (comma-separated)        │
│    - upload_user_id: "{oid}" (Azure AD object ID)         │
│    - upload_timestamp: "{ISO 8601}"                        │
│    - file_class: "general" or "authority"                 │
│                                                             │
│  Blob Trigger:                                             │
│    - Function: FileUploadedEtrigger                        │
│    - Pattern: upload-*/                                   │
│    - Event: Microsoft.Storage.BlobCreated                  │
│                                                             │
│  Examples:                                                 │
│    - upload-groupA/legal/contract.pdf                      │
│    - upload-groupA/hr/policy.docx                          │
│    - upload-groupB/finance/report.xlsx                     │
│    - upload-admins/system/config.json                      │
└────────────────────────────────────────────────────────────┘

[Content Containers - Processed Documents (20+ containers)]

┌────────────────────────────────────────────────────────────┐
│  content-{group-name}                                      │
│  Purpose: Processed documents for retrieval               │
│  Access: Read-only (backend), Write (functions)           │
│  Lifecycle: Retain indefinitely (or per policy)           │
│                                                             │
│  Structure:                                                │
│    /{folder}/                                              │
│      {filename}.pdf (original file)                        │
│      {filename}_extracted.json (intermediate results)      │
│      {filename}_chunks/ (chunk directory)                  │
│        chunk_0.json                                        │
│        chunk_1.json                                        │
│        ...                                                 │
│                                                             │
│  Chunk JSON Format:                                        │
│    {                                                        │
│      "id": "contract_chunk_5",                             │
│      "content": "Chunk text content...",                   │
│      "metadata": {                                         │
│        "file_name": "contract.pdf",                        │
│        "chunk_id": 5,                                      │
│        "pages": [5, 6],                                    │
│        "file_class": "general",                            │
│        "tags": ["legal", "contract"]                       │
│      }                                                      │
│    }                                                        │
│                                                             │
│  RBAC Mapping:                                             │
│    upload-groupA → content-groupA                          │
│    upload-groupB → content-groupB                          │
│    (defined in Cosmos DB group_management container)       │
│                                                             │
│  Examples:                                                 │
│    - content-groupA/legal/contract.pdf                     │
│    - content-groupA/legal/contract_extracted.json          │
│    - content-groupA/legal/contract_chunks/chunk_0.json     │
└────────────────────────────────────────────────────────────┘

[Configuration Container]

┌────────────────────────────────────────────────────────────┐
│  config                                                    │
│  Purpose: System configuration, custom examples           │
│  Access: Read (backend, frontend), Write (admin)          │
│                                                             │
│  Files:                                                     │
│    - examplelist.json (custom example questions per group)│
│    - lexicon.xlsx (EN↔FR translation glossary)            │
│    - system_config.json (global settings)                 │
│                                                             │
│  examplelist.json Structure:                               │
│    {                                                        │
│      "{group_id}": {                                       │
│        "title": "Sample Questions",                        │
│        "examples": [                                       │
│          {                                                 │
│            "text": "How do I apply for EI?",              │
│            "value": "How do I apply for EI?"              │
│          },                                                │
│          ...                                               │
│        ]                                                    │
│      }                                                      │
│    }                                                        │
│                                                             │
│  lexicon.xlsx:                                             │
│    - EN_FR sheet: English → French term mappings          │
│    - FR_EN sheet: French → English term mappings          │
│    - Used for translation consistency                      │
└────────────────────────────────────────────────────────────┘

[Function App Internal Storage]

┌────────────────────────────────────────────────────────────┐
│  azure-webjobs-hosts                                       │
│  Purpose: Azure Functions runtime state                   │
│  Access: Functions runtime only                           │
│  Content: Function keys, host configuration, logs         │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  azure-webjobs-secrets                                     │
│  Purpose: Function app secrets                            │
│  Access: Functions runtime only                           │
│  Content: Connection strings, API keys                    │
└────────────────────────────────────────────────────────────┘

[Temporary/Scratch Containers]

┌────────────────────────────────────────────────────────────┐
│  temp-processing                                           │
│  Purpose: Intermediate processing files                   │
│  Lifecycle: Auto-delete after 7 days (TTL)                │
│  Content: Temporary OCR results, partial extractions      │
└────────────────────────────────────────────────────────────┘

[Access Patterns]

User Upload:
  User (authenticated) → Backend /upload → upload-{group} container
                                        ↓
                            FileUploadedEtrigger (Blob Trigger)
                                        ↓
                            Processing Pipeline (Functions)
                                        ↓
                            content-{group} container ← Backend /getcitation

Search Query:
  User → Backend /chat → Cognitive Search (queries index)
                       ↓
                       Citation references: content-{group}/{file}
                       ↓
                       Backend /getcitation → Blob download

Configuration:
  Backend startup → config/examplelist.json (cached with ETag)
                 → config/lexicon.xlsx (loaded into memory)

[Storage Costs (Estimated)]

Container Type          │ Storage (GB) │ Operations/month │ Cost/month
────────────────────────┼──────────────┼──────────────────┼───────────
upload-* (20 containers)│ 50-200       │ 10K writes       │ $5-20
content-* (20 containers)│ 100-500      │ 50K reads        │ $10-50
config                  │ <1           │ 1K reads         │ <$1
Function internal       │ 5-10         │ 100K ops         │ $5-10
temp-processing         │ 10-50        │ 20K writes/reads │ $5-15
─────────────────────────────────────────────────────────────────────
Total                   │ 165-761 GB   │ 181K ops         │ $25-96/month

[Security]

- All containers: Private endpoints only (publicNetworkAccess: Disabled)
- RBAC: Azure AD authentication required
- Encryption: At-rest (256-bit AES), In-transit (TLS 1.2+)
- Access Tiers: Hot (upload/content), Cool (temp), Archive (old files)
- Soft Delete: Enabled (7-day retention)
- Versioning: Enabled on config container only
- Lifecycle Management: Auto-archive after 90 days (upload containers)
```

**Evidence**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 100-200)

---

## Diagram 14: Search Index Schema (1536-dim vectors)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 Azure Cognitive Search Index Schema                          │
│                 Source: azure_search/create_vector_index.json                │
│                 Lines: 1-298 (complete schema definition)                    │
└─────────────────────────────────────────────────────────────────────────────┘

[Field Definitions - Complete Schema]

┌────────────────────────────────────────────────────────────────┐
│  Metadata Fields (Standard Search)                            │
├────────────────┬──────────┬──────────┬────────────────────────┤
│ Field Name     │ Type     │ Features │ Purpose                │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ id             │ Edm.String│ Key     │ Unique chunk ID        │
│                │          │ Searchable│ "file_chunk_0"        │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ file_name      │ Edm.String│ Filterable│ Source document      │
│                │          │ Facetable│ "contract.pdf"         │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ chunk_file     │ Edm.String│ Filterable│ Chunk identifier     │
│                │          │          │ "contract_chunk_5"     │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ folder         │ Edm.String│ Filterable│ Upload folder path   │
│                │          │ Facetable│ "legal/contracts"      │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ file_class     │ Edm.String│ Filterable│ "general" or         │
│                │          │ Facetable│ "authority"            │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ pages          │Collection │ Filterable│ Page numbers         │
│                │(Edm.Int32)│          │ [1, 2, 3]             │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ tags           │Collection │ Filterable│ User-assigned tags   │
│                │(Edm.String)│ Facetable│ ["legal", "2024"]    │
└────────────────┴──────────┴──────────┴────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  Content Fields (Full-Text Search)                            │
├────────────────┬──────────┬──────────┬────────────────────────┤
│ Field Name     │ Type     │ Features │ Purpose                │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ content        │ Edm.String│Searchable│ Main chunk text       │
│                │          │ Retrievable│ Up to 1500 chars    │
│                │          │ Analyzer: │ Used for keyword      │
│                │          │ en.microsoft│ search (BM25)       │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ title          │ Edm.String│Searchable│ Document title        │
│                │          │ Retrievable│ Boosted in search    │
│                │          │ Analyzer: │ Weight: 2.0           │
│                │          │ en.microsoft│                      │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ translated_title│Edm.String│Searchable│ French title         │
│                │          │ Retrievable│ (if bilingual)       │
│                │          │ Analyzer: │                        │
│                │          │ fr.microsoft│                      │
└────────────────┴──────────┴──────────┴────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  Vector Field (Embedding Search)                              │
├────────────────┬──────────┬──────────┬────────────────────────┤
│ Field Name     │ Type     │ Features │ Purpose                │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ contentVector  │Collection │Searchable│ Embedding vector      │
│                │(Edm.Single)│ Dimensions:│ Generated by Azure │
│                │          │ 1536     │ OpenAI text-embedding-│
│                │          │ Algorithm:│ 3-small               │
│                │          │ HNSW     │                        │
│                │          │ Metric:  │                        │
│                │          │ cosine   │                        │
└────────────────┴──────────┴──────────┴────────────────────────┘

[Bilingual Authority Fields]

┌────────────────────────────────────────────────────────────────┐
│  For file_class="authority" only                              │
├────────────────┬──────────┬──────────┬────────────────────────┤
│ Field Name     │ Type     │ Features │ Purpose                │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ title_en       │ Edm.String│Searchable│ English title         │
│                │          │ Analyzer:│                        │
│                │          │ en.microsoft│                      │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ content_en     │ Edm.String│Searchable│ English content       │
│                │          │ Analyzer:│                        │
│                │          │ en.microsoft│                      │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ title_fr       │ Edm.String│Searchable│ French title          │
│                │          │ Analyzer:│                        │
│                │          │ fr.microsoft│                      │
├────────────────┼──────────┼──────────┼────────────────────────┤
│ content_fr     │ Edm.String│Searchable│ French content        │
│                │          │ Analyzer:│                        │
│                │          │ fr.microsoft│                      │
└────────────────┴──────────┴──────────┴────────────────────────┘

[Vector Search Configuration (HNSW Algorithm)]

┌────────────────────────────────────────────────────────────────┐
│  HNSW (Hierarchical Navigable Small World) Parameters         │
│  Source: azure_search/create_vector_index.json (lines 200-250)│
├────────────────────────────┬───────────────────────────────────┤
│ Parameter                  │ Value & Purpose                   │
├────────────────────────────┼───────────────────────────────────┤
│ algorithm                  │ "hnsw"                            │
│                            │ Approximate nearest neighbor      │
├────────────────────────────┼───────────────────────────────────┤
│ m (connections per layer)  │ 4                                 │
│                            │ Trade-off: Recall vs. memory      │
│                            │ Higher m = better recall, more RAM│
├────────────────────────────┼───────────────────────────────────┤
│ efConstruction             │ 400                               │
│                            │ Build-time effort                 │
│                            │ Higher = better graph quality     │
├────────────────────────────┼───────────────────────────────────┤
│ efSearch                   │ 500                               │
│                            │ Query-time effort                 │
│                            │ Higher = better recall, slower    │
├────────────────────────────┼───────────────────────────────────┤
│ metric                     │ "cosine"                          │
│                            │ Similarity function               │
│                            │ Normalized dot product            │
├────────────────────────────┼───────────────────────────────────┤
│ dimensions                 │ 1536                              │
│                            │ text-embedding-3-small output     │
└────────────────────────────┴───────────────────────────────────┘

[Semantic Configuration]

┌────────────────────────────────────────────────────────────────┐
│  Semantic Ranking Configuration                               │
│  Source: azure_search/create_vector_index.json (lines 250-280)│
├────────────────────────────┬───────────────────────────────────┤
│ Configuration              │ Value                             │
├────────────────────────────┼───────────────────────────────────┤
│ name                       │ "default"                         │
├────────────────────────────┼───────────────────────────────────┤
│ titleField                 │ "title"                           │
│                            │ Primary title for semantic ranking│
├────────────────────────────┼───────────────────────────────────┤
│ prioritizedContentFields   │ ["content", "content_en",         │
│                            │  "content_fr"]                    │
│                            │ Fields used for semantic scoring  │
├────────────────────────────┼───────────────────────────────────┤
│ prioritizedKeywordsFields  │ ["tags", "folder"]                │
│                            │ Metadata for semantic context     │
└────────────────────────────┴───────────────────────────────────┘

[BM25 Similarity Configuration]

┌────────────────────────────────────────────────────────────────┐
│  BM25 (Best Match 25) for Keyword Search                      │
│  Source: azure_search/create_vector_index.json (lines 280-298)│
├────────────────────────────┬───────────────────────────────────┤
│ Parameter                  │ Value & Purpose                   │
├────────────────────────────┼───────────────────────────────────┤
│ k1                         │ 1.2                               │
│                            │ Term frequency saturation         │
│                            │ Controls diminishing returns      │
├────────────────────────────┼───────────────────────────────────┤
│ b                          │ 0.75                              │
│                            │ Length normalization              │
│                            │ Penalizes longer documents        │
└────────────────────────────┴───────────────────────────────────┘

[Index Statistics (Typical)]

Metric                     │ Value
───────────────────────────┼──────────────────
Total Documents            │ 50,000-500,000
Total Chunks               │ (depends on doc size)
Index Size (GB)            │ 5-50 GB
  - Content fields         │ 2-20 GB (text)
  - Vector fields          │ 3-30 GB (1536-dim × count)
Vector Memory (GB)         │ ~6 bytes per dimension per doc
                           │ = 1536 × 6 × count / 1024^3
Average Query Latency      │ 50-200ms (p50)
                           │ 100-500ms (p95)
Throughput                 │ 100-500 queries/second

[Example Document in Index]

{
  "id": "contract_chunk_5",
  "file_name": "service_agreement.pdf",
  "chunk_file": "service_agreement_chunk_5",
  "folder": "legal/contracts",
  "file_class": "general",
  "pages": [5, 6],
  "content": "This agreement is entered into between...",
  "contentVector": [0.123, -0.456, 0.789, ..., -0.234],  // 1536 floats
  "title": "Service Agreement",
  "translated_title": null,
  "tags": ["contract", "legal", "2024"],
  "title_en": null,
  "content_en": null,
  "title_fr": null,
  "content_fr": null
}

[Query Types Supported]

1. Keyword Search (BM25):
   - Query: "employment insurance benefits"
   - Searches: content, title fields
   - Analyzer: en.microsoft (stemming, stop words)

2. Vector Search (HNSW):
   - Query embedding: [0.234, -0.567, ...]
   - Metric: Cosine similarity
   - Returns: Top K nearest neighbors

3. Hybrid Search (Keyword + Vector):
   - Combines BM25 scores + vector scores
   - Weighted average: 40% keyword, 60% vector
   - Best of both worlds

4. Semantic Search:
   - Uses semantic ranking on top of keyword/hybrid
   - Microsoft semantic model
   - Improves relevance for natural language queries

5. Filtered Search:
   - Filter by: folder, file_class, tags, pages
   - Example: folder eq 'legal' and tags/any(t: t eq 'contract')
```

**Evidence**: azure_search/create_vector_index.json (lines 1-298)

---

**Summary**: Document Ingestion Pipeline Complete

✅ **6 Diagrams Created**:
- Diagram 9: 6-Stage Ingestion Flow (complete pipeline)
- Diagram 10: Queue Routing Tree (8 queues)
- Diagram 11: Chunking Algorithm (1500 chars, 100 overlap)
- Diagram 12: Embedding Sequence (Azure OpenAI integration)
- Diagram 13: Blob Storage Containers (20+ containers)
- Diagram 14: Search Index Schema (1536-dim vectors, HNSW config)

**Next Section**: [RAG Query Execution](dev2-TechDoc-04-rag-query.md) - 5-stage query pipeline with hybrid search
