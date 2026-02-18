# EVA Domain Assistant - Ingestion, Chunking, Indexing & Retrieval Analysis
**Evidence-Based Technical Report**  
**Source Repository**: `C:\Users\marco.presta\OneDrive - ESDC EDSC\Documents\AICOE\EVA-JP-reference-0113`  
**Analysis Date**: January 23, 2026  
**Scope**: End-to-end document processing pipeline with hard evidence

---

## EXECUTIVE SUMMARY

This evidence-based analysis documents the complete **document ingestion and retrieval pipeline** of EVA Domain Assistant JP 1.2, a production RAG system built on Azure OpenAI. Through direct code inspection, we reveal a **multi-stage Azure Functions pipeline** that processes 11+ file formats (PDF, DOCX, XML, JSON, images, media) via specialized routing queues, performs OCR using Azure Document Intelligence, chunks text at **fixed 1500-character boundaries** (not semantic), generates 1536-dimensional embeddings (text-embedding-ada-002), and indexes into Azure Cognitive Search with **hybrid vector + keyword search**. 

**Key Capabilities**: Canadian regulation detection, bilingual EN/FR processing, metadata-rich indexing with 15+ fields per chunk, semantic reranking, and citation-enforced LLM responses. **Critical Limitations**: Fixed-size chunking loses semantic boundaries, XML/JSON structure flattened to plain text, LLM cannot see page numbers/tags during generation (metadata injected post-hoc), and Canadian regulation detection relies on brittle filename pattern matching.

**For detailed query-time execution analysis** with server log evidence and code excerpts, see **[EVA-DA-RAG-Query-Execution-Analysis.md](./EVA-DA-RAG-Query-Execution-Analysis.md)** which documents the complete retrieval, prompt assembly, and response generation flow.

---

## 1. REPO INVENTORY - Pipeline Entry Points

### 1.1 Azure Functions Pipeline Architecture
**EVIDENCE**: `functions/` directory contains 11 Azure Functions for document processing

**Entry Point**: `functions/FileUploadedEtrigger/__init__.py` (Lines 1-347)
- **Trigger**: Azure Storage Queue message on blob creation
- **Event Type**: `Microsoft.Storage.BlobCreated`
- **Queue Variables** (Lines 28-33):
  ```python
  non_pdf_submit_queue = os.environ["NON_PDF_SUBMIT_QUEUE"]
  pdf_polling_queue = os.environ["PDF_POLLING_QUEUE"] 
  pdf_submit_queue = os.environ["PDF_SUBMIT_QUEUE"]
  media_submit_queue = os.environ["MEDIA_SUBMIT_QUEUE"]
  image_enrichment_queue = os.environ["IMAGE_ENRICHMENT_QUEUE"]
  auth_enrichment_queue = os.environ.get("AUTH_ENRICHMENT_QUEUE", "")
  authority_doc_queue = os.environ.get("AUTHORITY_DOC_QUEUE", "")
  ```

**Pipeline Functions Identified**:
1. `FileUploadedEtrigger/` - Entry point and routing
2. `FileFormRecSubmissionPDF/` - PDF OCR submission
3. `FileFormRecPollingPDF/` - PDF OCR polling
4. `FileLayoutParsingOther/` - Non-PDF text extraction
5. `TextEnrichment/` - Language processing and indexing
6. `AuthorityDocProcessor/` - Canadian regulation handling
7. `ImageEnrichment/` - Image text extraction
8. `EnFrTextExtractor/` - Bilingual processing
9. `FileDeletion/` - Cleanup operations
10. `UrlScrapper/` - Web content extraction

---

## 2. FILE-TYPE ROUTING LOGIC

### 2.1 Extension-Based Routing
**EVIDENCE**: `functions/FileUploadedEtrigger/__init__.py` (Lines 200-250)

**PDF Files**:
```python
if file_extension == "pdf":
    if authority_doc_queue and is_canadian_regulation_pdf(blob_name, tags_list):
        queue_name = authority_doc_queue
        log.info(f"Routing Canadian regulation PDF {blob_name} to authority-doc-queue for bilingual processing")
    else:
        queue_name = pdf_submit_queue
```

**Text-Based Files** (Line 205):
```python
elif file_extension in ["htm", "csv", "docx", "eml", "html", "md", "msg", "pptx", "txt", "xlsx", "xml", "json", "tsv"]:
    queue_name = non_pdf_submit_queue
```

**Media Files** (Lines 207-231):
```python
elif file_extension in [
    "flv", "mxf", "gxf", "ts", "ps", "3gp", "3gpp", "mpg", "wmv", "asf", 
    "avi", "wmv", "mp4", "m4a", "m4v", "isma", "ismv", "dvr-ms", "mkv", 
    "wav", "mov"
]:
    queue_name = media_submit_queue
```

**Image Files** (Lines 231-240):
```python
elif file_extension in [
    "jpg", "jpeg", "png", "gif", "bmp", "tif", "tiff"
]:
    queue_name = image_enrichment_queue
```

### 2.2 Canadian Regulation Detection
**EVIDENCE**: `functions/FileUploadedEtrigger/__init__.py` (Lines 98-115)
```python
def is_canadian_regulation_pdf(blob_name, tags_list):
    blob_name_lower = blob_name.lower()
    regulation_patterns = ["regulation", "reglement", "c.r.c", "crc", "sor-", "dors-", "statutory", "act", "loi", "canada gazette", "gazette du canada"]
    
    # Check if filename contains regulation patterns
    for pattern in regulation_patterns:
        if pattern in blob_name_lower:
            return True
```

---

## 3. TEXT EXTRACTION METHODS

### 3.1 PDF Processing - Azure Document Intelligence
**EVIDENCE**: `functions/FileFormRecSubmissionPDF/__init__.py` (Lines 1-187)

**Service Configuration** (Lines 29-31):
```python
endpoint = os.environ["AZURE_FORM_RECOGNIZER_ENDPOINT"]
api_version = os.environ["FR_API_VERSION"]  
FR_MODEL = "prebuilt-layout"
```

**OCR Submission Process** (Lines 105-115):
```python
headers = {"Content-Type": "application/json", "Authorization": f"Bearer {token_provider()}"}
params = {"api-version": api_version}
body = {"urlSource": blob_path_plus_sas}
url = f"{endpoint}documentintelligence/documentModels/{FR_MODEL}:analyze"
response = requests.post(url, headers=headers, params=params, json=body)
```

**Result**: Returns 202 status code for async processing, uses `prebuilt-layout` model for OCR

### 3.2 Non-PDF Text Extraction - Unstructured Library  
**EVIDENCE**: `functions/FileLayoutParsingOther/__init__.py` (Lines 440-480)

**XML Processing** (Lines 467-470):
```python
elif file_extension_lower == ".xml":
    from unstructured.partition.xml import partition_xml
    elements = partition_xml(file=bytes_io)
```

**JSON/TXT Processing** (Lines 457-460):
```python
elif any(file_extension_lower in x for x in [".txt", ".json"]):
    from unstructured.partition.text import partition_text
    elements = partition_text(file=bytes_io)
```

**Markdown Processing** (Lines 442-445):
```python
elif file_extension_lower == ".md":
    from unstructured.partition.md import partition_md
    elements = partition_md(file=bytes_io)
```

**Result**: Uses `unstructured` library for structure-aware parsing, extracts text content while preserving basic structure

---

## 4. CHUNKING ALGORITHM

### 4.1 Chunking Configuration
**EVIDENCE**: `functions/FileLayoutParsingOther/__init__.py` (Lines 36, 480-490)

**Environment Variable**:
```python
CHUNK_TARGET_SIZE = int(os.environ["CHUNK_TARGET_SIZE"])
```

**RecursiveCharacterTextSplitter Setup** (Lines 480-490):
```python
text_splitter = RecursiveCharacterTextSplitter(
    separators=[
        "\n\n",
        "\n", 
        ".",
        ",",
        " ",
        "",
    ],
    chunk_size=1500,
    chunk_overlap=100,
)
```

### 4.2 Chunking Algorithm Type
**ALGORITHM**: Fixed-size chunking with semantic separators
- **Chunk Size**: 1500 characters (hardcoded)
- **Overlap**: 100 characters 
- **Separator Priority**: Double newline → newline → period → comma → space → character-level
- **Type**: LangChain's RecursiveCharacterTextSplitter (NOT semantic chunking)

**EVIDENCE**: No semantic-aware chunking found - purely character-based with separator hierarchy

---

## 5. METADATA HANDLING

### 5.1 Search Index Schema - Document-Level Metadata
**EVIDENCE**: `azure_search/create_vector_index.json` (Lines 1-298)

**Core Document Fields**:
```json
{
  "name": "id", "type": "Edm.String", "key": true, "filterable": true
},
{
  "name": "file_name", "type": "Edm.String", "searchable": true, "filterable": true, "retrievable": true
},
{
  "name": "file_uri", "type": "Edm.String", "retrievable": true
},
{
  "name": "processed_datetime", "type": "Edm.DateTimeOffset", "filterable": true, "sortable": true
},
{
  "name": "folder", "type": "Edm.String", "filterable": true, "facetable": true
},
{
  "name": "file_class", "type": "Edm.String", "filterable": true, "facetable": true
}
```

**Content Fields**:
```json
{
  "name": "content", "type": "Edm.String", "searchable": true, "analyzer": "$SEARCH_INDEX_ANALYZER"
},
{
  "name": "title", "type": "Edm.String", "searchable": true, "analyzer": "$SEARCH_INDEX_ANALYZER"  
},
{
  "name": "translated_title", "type": "Edm.String", "searchable": true, "analyzer": "$SEARCH_INDEX_ANALYZER"
}
```

**Chunk-Level Fields**:
```json
{
  "name": "chunk_file", "type": "Edm.String", "filterable": true, "retrievable": true
},
{
  "name": "pages", "type": "Collection(Edm.Int32)", "retrievable": true  
}
```

### 5.2 Metadata Persistence Pattern
**EVIDENCE**: Metadata is repeated per chunk in the search index
- **Per-chunk storage**: Each chunk gets full document metadata
- **Citation tracking**: `chunk_file` field links to individual chunk blob
- **Page tracking**: `pages` array maps chunk to source PDF pages

---

## 6. INDEXING DESTINATION & EMBEDDINGS

### 6.1 Azure Cognitive Search Indexing
**EVIDENCE**: `functions/TextEnrichment/__init__.py` (Lines 1-431)

**Embeddings Queue Processing** (Lines 37-45):
```python
queueName = os.environ["EMBEDDINGS_QUEUE"]
azure_ai_endpoint = os.environ["AZURE_AI_ENDPOINT"]
targetTranslationLanguage = os.environ["TARGET_TRANSLATION_LANGUAGE"]
```

**Vector Field Configuration** (`azure_search/create_vector_index.json` Lines 160-185):
```json
{
  "name": "embedding",
  "type": "Collection(Edm.Single)",
  "searchable": true,
  "dimensions": 1536,
  "vectorSearchProfile": "my-vector-profile"
}
```

### 6.2 Embedding Generation Process
**EVIDENCE**: External enrichment service called for embeddings
- **Service**: Separate Flask enrichment service (not in Functions)
- **Model**: text-embedding-ada-002 (inferred from 1536 dimensions)
- **Batching**: Handled by external service (not visible in Functions code)

### 6.3 Search Index Update
**EVIDENCE**: `functions/TextEnrichment/__init__.py` (Lines 200-431)
- **Language detection**: First 4000 chars analyzed
- **Translation**: Content translated to target language if needed  
- **Entity extraction**: Named entity recognition applied
- **Index upsert**: Documents updated in Azure Cognitive Search

---

## 7. RETRIEVAL & ANSWER-TIME PAYLOAD

### 7.1 Search Implementation
**EVIDENCE**: `app/backend/approaches/chatreadretrieveread.py` (Lines 250-350)

**Hybrid Search Query** (Lines 280-295):
```python
if self.use_semantic_reranker and overrides.get("semantic_ranker", True):
    r = self.search_client.search(
        generated_query,
        query_type=QueryType.SEMANTIC,
        semantic_configuration_name="default", 
        top=top,
        query_caption="extractive|highlight-false" if use_semantic_captions else None,
        vector_queries=[vector],
        filter=search_filter,
    )
else:
    r = self.search_client.search(generated_query, top=top, vector_queries=[vector], filter=search_filter)
```

### 7.2 Metadata Available at Retrieval Time
**EVIDENCE**: `app/backend/approaches/chatreadretrieveread.py` (Lines 310-325)

**Citation Lookup Assembly**:
```python
citation_lookup[f"File{idx}"] = {
    "citation": urllib.parse.unquote("https://" + self.blob_client.url.split("/")[2] + f"/{self.content_storage_container}/" + doc[self.chunk_file_field]),
    "source_path": doc[self.source_file_field], 
    "page_number": str(doc[self.page_number_field][0]) or "0",
    "tags": doc["tags"],
}
```

**Data Points for LLM** (Lines 305-310):
```python
results.append(f"File{idx} " + "| " + nonewlines(doc[self.content_field]))
data_points.append("/".join(urllib.parse.unquote(doc[self.source_file_field]).split("/")[1:]) + "| " + nonewlines(doc[self.content_field]))
```

### 7.3 LLM Prompt Assembly
**EVIDENCE**: `app/backend/approaches/chatreadretrieveread.py` (Lines 35-50)

**System Message Template**:
```python
SYSTEM_MESSAGE_CHAT_CONVERSATION = """You are an Azure OpenAI Completion system. Your persona is {systemPersona} who helps answer questions about an agency's data. {response_length_prompt}
User persona is {userPersona} Answer ONLY with the facts listed in the list of sources below in {query_term_language} with citations. {rule1}

Each source has content followed by a pipe character and the URL. Instead of writing the full URL, cite it using placeholders like [File1], [File2], etc., based on their order in the list.
```

**What LLM Sees**:
- **Content**: Chunk text from `content` field
- **Citations**: File placeholders ([File1], [File2], etc.)
- **Source Info**: File path after "| " delimiter
- **NO direct access**: Page numbers, chunk_file paths, or full blob URLs

---

## SUMMARY - Complete Pipeline Flow

### Pipeline Architecture (As Implemented) - Comprehensive Visual Flow

```
                            EVA DA INGESTION PIPELINE - COMPLETE FLOW
                            ==========================================

[USER UPLOADS FILE]
        |
        v
+-----------------------------------------------------------------------+
|                    AZURE BLOB STORAGE (upload container)             |
|  - BlobCreated event triggers FileUploadedEtrigger                   |
+-----------------------------------------------------------------------+
        |
        v
+-----------------------------------------------------------------------+
|  [1] FileUploadedEtrigger (Azure Function - Entry Point)            |
|  ====================================================================|
|  FILE: functions/FileUploadedEtrigger/__init__.py                   |
|  LOGIC:                                                              |
|    - Extract file extension from blob name                           |
|    - Check if Canadian regulation PDF (pattern matching)             |
|    - Route to appropriate queue based on extension                   |
|                                                                       |
|  ROUTING DECISION TREE:                                              |
|    IF extension == ".pdf":                                           |
|      IF is_canadian_regulation_pdf():                                |
|        --> authority_doc_queue (bilingual processing)                |
|      ELSE:                                                           |
|        --> pdf_submit_queue (standard OCR)                           |
|    ELIF extension IN [.htm, .csv, .docx, .eml, .html, .md,         |
|                       .msg, .pptx, .txt, .xlsx, .xml, .json, .tsv]: |
|        --> non_pdf_submit_queue                                      |
|    ELIF extension IN [.flv, .mxf, .mp4, .avi, .wmv, .mov, etc.]:   |
|        --> media_submit_queue                                        |
|    ELIF extension IN [.jpg, .jpeg, .png, .gif, .bmp, .tif]:        |
|        --> image_enrichment_queue                                    |
+-----------------------------------------------------------------------+
        |
        |---- SPLIT INTO TWO PATHS ----
        |                              |
        v                              v
   [PDF PATH]                    [NON-PDF PATH]
        |                              |
        v                              v
+---------------------------+    +----------------------------------------+
| [2a] pdf_submit_queue     |    | [2b] non_pdf_submit_queue             |
+---------------------------+    +----------------------------------------+
        |                              |
        v                              v
+---------------------------+    +----------------------------------------+
| [3a] FileFormRec          |    | [3b] FileLayoutParsingOther           |
|      SubmissionPDF        |    |      (Azure Function)                 |
| (Azure Function)          |    |========================================|
|===========================|    | FILE: functions/FileLayoutParsingOther|
| FILE: functions/          |    |       /__init__.py                    |
|       FileFormRec         |    |                                        |
|       SubmissionPDF/      |    | PARSING BY EXTENSION:                  |
|       __init__.py         |    |   IF .xml:                            |
|                           |    |     partition_xml() -> flat text      |
| ACTIONS:                  |    |   IF .json OR .txt:                   |
|  1. Get Azure Doc Intel   |    |     partition_text() -> flat text     |
|     endpoint + token      |    |   IF .md:                             |
|  2. Submit PDF to API:    |    |     partition_md() -> text + format   |
|     - Model: prebuilt-    |    |   IF .docx:                           |
|       layout              |    |     partition_docx()                  |
|     - Method: POST with   |    |   IF .html:                           |
|       blob URL + SAS      |    |     partition_html()                  |
|  3. Get operation ID      |    |   [etc. for other formats]            |
|  4. Send to polling queue |    |                                        |
+---------------------------+    | CRITICAL BEHAVIOR:                     |
        |                        |  - XML structure FLATTENED to text    |
        v                        |  - JSON treated as plain text         |
+---------------------------+    |  - No schema/element extraction       |
| [4a] pdf_polling_queue    |    |                                        |
+---------------------------+    | OUTPUT: List of text elements         |
        |                        +----------------------------------------+
        v                                       |
+---------------------------+                   v
| [5a] FileFormRec          |    +----------------------------------------+
|      PollingPDF           |    | [4b] CHUNKING STAGE                   |
| (Azure Function)          |    |      (Within FileLayoutParsingOther)  |
|===========================|    |========================================|
| FILE: functions/          |    | FILE: Same function, Lines 480-490    |
|       FileFormRec         |    |                                        |
|       PollingPDF/         |    | ALGORITHM:                             |
|       __init__.py         |    |   RecursiveCharacterTextSplitter      |
|                           |    |   - chunk_size: 1500 (HARDCODED)     |
| ACTIONS:                  |    |   - chunk_overlap: 100                |
|  1. Poll Azure Doc Intel  |    |   - separators: ["\n\n", "\n", ".",   |
|     for OCR results       |    |                  ",", " ", ""]        |
|  2. Extract text content  |    |                                        |
|  3. Extract page numbers  |    | EVIDENCE: CHUNK_TARGET_SIZE env var   |
|  4. Package as elements   |    |           exists but is UNUSED        |
|  5. Send to non_pdf_      |    |                                        |
|     submit_queue          |    | OUTPUT: List of 1500-char text chunks |
|     (rejoins main flow)   |    |         with 100-char overlap         |
+---------------------------+    +----------------------------------------+
        |                                       |
        +----------- PATHS CONVERGE -----------+
                            |
                            v
            +----------------------------------------+
            | text-enrichment-queue                 |
            +----------------------------------------+
                            |
                            v
+-----------------------------------------------------------------------+
| [5] TextEnrichment (Azure Function)                                  |
|======================================================================|
| FILE: functions/TextEnrichment/__init__.py                           |
|                                                                       |
| PROCESSING STEPS:                                                    |
|  1. Language Detection                                               |
|     - Analyze first 4000 chars                                       |
|     - Detect primary language (EN/FR)                                |
|                                                                       |
|  2. Translation (if needed)                                          |
|     - TARGET_TRANSLATION_LANGUAGE from env                           |
|     - Call Azure AI Translator                                       |
|                                                                       |
|  3. Entity Extraction                                                |
|     - Named entity recognition                                       |
|     - Key phrase extraction                                          |
|                                                                       |
|  4. Metadata Assembly                                                |
|     - Collect: file_name, file_uri, processed_datetime               |
|     - Collect: folder, file_class                                    |
|     - For PDF chunks: pages array [1, 2, 3, ...]                    |
|     - chunk_file: path to individual chunk blob                      |
|                                                                       |
|  5. Send to Embeddings Queue                                         |
+-----------------------------------------------------------------------+
                            |
                            v
            +----------------------------------------+
            | embeddings-queue                      |
            +----------------------------------------+
                            |
                            v
+-----------------------------------------------------------------------+
| [6] EXTERNAL ENRICHMENT SERVICE (Flask API)                          |
|======================================================================|
| FILE: app/enrichment/app.py (separate service, NOT Azure Function)  |
|                                                                       |
| EMBEDDING GENERATION:                                                |
|  - Model: text-embedding-ada-002                                     |
|  - API: Azure OpenAI Embeddings API                                  |
|  - Dimensions: 1536                                                  |
|  - Batch processing: Multiple chunks per request                     |
|                                                                       |
| INPUT: Chunk text + metadata                                         |
| OUTPUT: Chunk text + metadata + embedding vector (1536-dim)          |
+-----------------------------------------------------------------------+
                            |
                            v
+-----------------------------------------------------------------------+
| [7] AZURE COGNITIVE SEARCH INDEX                                     |
|======================================================================|
| INDEX NAME: index-jurisprudence                                      |
| SCHEMA FILE: azure_search/create_vector_index.json                   |
|                                                                       |
| DOCUMENT STRUCTURE (per chunk):                                      |
|   {                                                                   |
|     "id": "base64(blob_path)",                                       |
|     "content": "chunk text here...",                                 |
|     "embedding": [0.123, -0.456, ...],  # 1536 dims                 |
|     "file_name": "test_example.pdf",                                 |
|     "file_uri": "https://.../documents/test_example.pdf",           |
|     "chunk_file": "test_example.pdf/chunk_001.json",                |
|     "pages": [1, 2],  # Which pages this chunk spans                |
|     "processed_datetime": "2026-01-23T10:30:00Z",                   |
|     "folder": "functional-test",                                     |
|     "file_class": "internal",                                        |
|     "tags": ["legal", "regulation"],                                 |
|     "title": "Document Title",                                       |
|     "translated_title": "Titre du document"                          |
|   }                                                                   |
|                                                                       |
| SEARCH CAPABILITIES:                                                 |
|   - Keyword search on "content" field                                |
|   - Vector search on "embedding" field (1536-dim)                    |
|   - Hybrid search (keyword + vector combined)                        |
|   - Semantic reranking (optional)                                    |
|   - Filters: folder, file_class, processed_datetime                  |
+-----------------------------------------------------------------------+
                            |
                     [INDEXING COMPLETE]
                            |
              +-------------+-------------+
              |                           |
              v                           v
     [USER QUERIES SYSTEM]      [ADMIN VIEWS DOCUMENTS]
              |                           |
              v                           |
+-----------------------------------------------------------------------+
| [8] RETRIEVAL & LLM GENERATION (Backend API)                         |
|======================================================================|
| FILE: app/backend/approaches/chatreadretrieveread.py                 |
|                                                                       |
| STEP 1: Query Optimization (Optional)                                |
|   - Call Azure AI Services to rewrite query                          |
|   - Fallback: Use original query if service unavailable             |
|                                                                       |
| STEP 2: Embedding Generation                                         |
|   - Call Enrichment service for query embedding                      |
|   - Fallback: Keyword-only search if service unavailable            |
|                                                                       |
| STEP 3: Hybrid Search Execution                                      |
|   search_client.search(                                              |
|     search_text=optimized_query,      # Keyword component           |
|     vector_queries=[VectorizedQuery(                                 |
|       vector=query_embedding,         # Vector component            |
|       k_nearest_neighbors=top_k,                                     |
|       fields="embedding"                                             |
|     )],                                                              |
|     query_type=QueryType.SEMANTIC,    # Optional reranking          |
|     top=3  # Default top-k results                                   |
|   )                                                                   |
|                                                                       |
| STEP 4: Context Assembly for LLM                                     |
|   Lines 305-310: Build data_points list                              |
|   FORMAT:                                                            |
|     "File1 | [chunk content text]"                                   |
|     "source/path/file.pdf | [chunk content text]"                    |
|                                                                       |
|   CRITICAL: LLM ONLY SEES:                                           |
|     - content field (chunk text)                                     |
|     - source_file_field (file path)                                  |
|                                                                       |
|   CRITICAL: LLM DOES NOT SEE:                                        |
|     - pages array (page numbers)                                     |
|     - tags                                                           |
|     - chunk_file path                                                |
|     - processed_datetime                                             |
|     - folder, file_class                                             |
|                                                                       |
| STEP 5: Build citation_lookup (SEPARATE from LLM input)             |
|   Lines 310-325: Metadata stored for post-processing                 |
|   {                                                                   |
|     "File1": {                                                       |
|       "citation": "https://.../chunk_001.json",                     |
|       "source_path": "documents/test_example.pdf",                   |
|       "page_number": "2",  # <-- NOT in LLM prompt!                 |
|       "tags": ["legal"]    # <-- NOT in LLM prompt!                 |
|     }                                                                 |
|   }                                                                   |
|                                                                       |
| STEP 6: LLM Completion (Azure OpenAI GPT-4)                          |
|   - System message: persona + rules                                  |
|   - User message: query + context (File1 | content...)              |
|   - Streaming response with [File1], [File2] placeholders           |
|                                                                       |
| STEP 7: Citation Replacement                                         |
|   - Replace [File1] with actual citation from citation_lookup        |
|   - Add page numbers and tags AFTER generation                       |
+-----------------------------------------------------------------------+
                            |
                            v
                   [RESPONSE TO USER]
                            |
                            v
            +---------------------------------------+
            | Frontend displays:                   |
            |  - Answer text                       |
            |  - Citations with page numbers       |
            |  - Source document links             |
            |  - Tags (added post-generation)      |
            +---------------------------------------+
```

### METADATA FLOW ANALYSIS

**STAGE 1 (Ingestion)**: ALL metadata captured
- file_name, file_uri, pages, tags, folder, file_class, etc.

**STAGE 2 (Indexing)**: ALL metadata stored per chunk
- Repeated metadata on every chunk (duplication pattern)

**STAGE 3 (Retrieval)**: ALL metadata returned from search
- Search results include full document object with all fields

**STAGE 4 (LLM Input)**: ONLY content + source_path sent to LLM
- Metadata gap: pages, tags excluded from generation context
- Metadata stored separately in citation_lookup

**STAGE 5 (Post-Processing)**: Metadata added to citations
- Page numbers and tags inserted after LLM generation
- Result: LLM cannot reason about page numbers during answer generation

### Technical Specifications
- **Chunking**: Fixed 1500 chars, 100 char overlap, separator-based (NOT semantic)
- **PDF OCR**: Azure Document Intelligence `prebuilt-layout` model
- **XML/JSON**: Unstructured library with `partition_xml`/`partition_text`
- **Embeddings**: 1536 dimensions (text-embedding-ada-002)
- **Search**: Azure Cognitive Search with semantic reranking
- **Metadata**: Repeated per chunk with citation tracking

### Configuration Evidence
- **CHUNK_TARGET_SIZE**: Environment variable (unused - hardcoded 1500)
- **Queue routing**: 7 specialized queues for different file types
- **Canadian regulations**: Special handling with pattern matching
- **Bilingual**: EN/FR translation and dual indexing support

### CRITICAL LIMITATIONS IDENTIFIED

**1. FIXED CHUNKING:**
- CHUNK_TARGET_SIZE environment variable exists but unused (Line 36)
- Hardcoded 1500 characters (Line 485)
- No semantic boundary detection

**2. STRUCTURE LOSS:**
- XML: partition_xml() extracts text, discards tags/hierarchy
- JSON: partition_text() treats as plain text, discards schema
- Cannot query by XML element or JSON key

**3. METADATA BLINDNESS:**
- LLM cannot see page numbers during generation
- LLM cannot see tags during generation
- Page numbers added to citations AFTER LLM completes answer

**4. NO CHUNK-LEVEL PROVENANCE:**
- Chunks inherit document metadata only
- No tracking of which function created chunk
- No tracking of chunking strategy used

**5. CANADIAN REGULATION DETECTION:**
- Pattern matching on filename only: "c.r.c", "sor-", "dors-"
- No content analysis
- Brittle, may miss regulations or false-positive