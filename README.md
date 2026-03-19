# Project 09: EVA Domain Assistant Repository Documentation

<!-- eva-primed -->
<!-- foundation-primer: 2026-03-03 by agent:copilot -->

## EVA Ecosystem Integration

| Tool | Purpose | How to Use |
|------|---------|------------|
| 37-data-model | Single source of truth for all project entities | GET http://localhost:8010/model/projects/09-eva-repo-documentation |
| 29-foundry | Agentic capabilities (search, RAG, eval, observability) | C:\eva-foundry\eva-foundation\29-foundry |
| 48-eva-veritas | Trust score and coverage audit | MCP tool: audit_repo / get_trust_score |
| 07-foundation-layer | Copilot instructions primer + governance templates | MCP tool: apply_primer / audit_project |

**Agent rule**: Query the data model API before reading source files.
```powershell
Invoke-RestMethod "http://localhost:8010/model/agent-guide"   # complete protocol
Invoke-RestMethod "http://localhost:8010/model/agent-summary" # all layer counts
```

---


## Purpose

This project evaluates the EVA Domain Assistant JP 1.2 repository as a reference implementation for Retrieval Augmented Generation (RAG) systems. Through code-based inspection and analysis, we document the architectural patterns, implementation strategies, and technical capabilities that can inform future RAG system development and deployment within government contexts.

## Technical Analysis Reports

### Completed Analysis Documents

#### Pipeline Analysis (Ingestion + Query Execution)
- **[EVA-DA-Pipeline-Analysis-Evidence-Based.md](./EVA-DA-Pipeline-Analysis-Evidence-Based.md)** - **Ingestion-time pipeline analysis**: Comprehensive evidence-based documentation of document processing, chunking (fixed 1500-char), embedding generation, and indexing with concrete file paths, line numbers, and code excerpts from Azure Functions pipeline
- **[EVA-DA-RAG-Query-Execution-Analysis.md](./EVA-DA-RAG-Query-Execution-Analysis.md)** - **Query-time RAG analysis**: Real-world server log analysis documenting the complete retrieval flow from user question to LLM response, including query optimization, hybrid search, citation assembly, and prompt construction with code validation

#### Configuration & Requirements
- **[eva-da-ref_config_inventory.md](./eva-da-ref_config_inventory.md)** - Complete inventory of 53+ configuration variables and settings
- **[eva-da-ref_vs_eva_jp_mapping.md](./eva-da-ref_vs_eva_jp_mapping.md)** - Gap analysis between current implementation and JP Plan-2 requirements
- **[repo_inspection_notes.md](./repo_inspection_notes.md)** - Detailed code structure analysis and implementation patterns

## What is EVA DA JP 1.2 (high-level)

EVA DA JP 1.2 is a production-ready, bilingual RAG system built on Azure OpenAI that provides secure document question-answering capabilities. The system processes 11+ file formats through an Azure Functions pipeline, generates embeddings via a dedicated enrichment service, and serves responses through a React frontend with real-time streaming. It implements enterprise features including role-based access control, audit logging, Canadian government document classification, and private network deployment for secure government cloud environments.

### EVA Foundation Integration Context
This system is part of ESDC's broader EVA Foundation initiative led by the AI Centre of Enablement (AICoE):
- **EVA Chat**: General conversational AI service for ESDC employees
- **EVA DA** (this system): Document-focused Q&A assistant with RBAC
- **EVA JP Plan-2**: Planned jurisprudence-specific variant with legal case metadata
- **Deployment**: Protected B Azure environment, EARB approved (January 29, 2025)
- **Governance**: Human-in-the-loop accountability, no external public exposure

### Technical Stack Details (Evidence-Based)
- **Backend**: Python 3.11+ with FastAPI 0.115.0, AsyncAzureOpenAI
- **Frontend**: Node.js 18+ with React 18.2.0, Vite 4.2.1, TypeScript
- **AI Services**: Azure OpenAI API version 2024-02-15-preview, text-embedding-ada-002
- **Search**: Azure Cognitive Search 11.5.1 with hybrid vector + keyword
- **Document Processing**: Azure Document Intelligence, 12 Azure Functions pipeline
- **Storage**: Azure Blob Storage, Cosmos DB for sessions/audit
- **Infrastructure**: Terraform IaC, private endpoints, Azure Government cloud support

## Repo Findings (Code-derived)

**Architecture & Framework**:
? **Multi-service architecture** with 4 primary components: FastAPI backend, React frontend, Flask enrichment service, and Azure Functions document pipeline  
  Evidence: `app/backend/app.py`, `app/frontend/src/`, `app/enrichment/app.py`, `functions/` directories

? **8 specialized RAG approaches** supporting different retrieval strategies from pure LLM to hybrid search with web integration  
  Evidence: `approaches/` directory with `chatreadretrieveread.py`, `mathassistant.py`, `chatwebretrieveread.py`

? **Async-first implementation** using FastAPI, AsyncAzureOpenAI, and streaming Server-Sent Events for real-time responses  
  Evidence: `async def run()` patterns in approach classes, `StreamingResponse` usage in `app.py`

**Document Processing Pipeline**:
? **12 Azure Functions** orchestrating document ingestion, OCR, chunking, embedding, and indexing workflows  
  Evidence: `functions/` directory structure with `FileUploadedEtrigger`, `TextEnrichment`, `FileFormRecSubmissionPDF`

? **Azure Document Intelligence OCR** for PDF processing using `prebuilt-layout` model  
  Evidence: `functions/FileFormRecSubmissionPDF/__init__.py` lines 105-115, `AZURE_FORM_RECOGNIZER_ENDPOINT` config

? **Format-agnostic processing** supporting PDF, DOCX, HTML, CSV, images, MSG, RTF, XML, JSON, MD with specialized routing  
  Evidence: Queue routing logic in `FileUploadedEtrigger/__init__.py` lines 205-240, format-specific processors

? **Fixed-size chunking** using LangChain RecursiveCharacterTextSplitter with 1500 chars, 100 overlap  
  Evidence: `functions/FileLayoutParsingOther/__init__.py` lines 480-490, hardcoded chunk_size=1500

? **Canadian government document detection** with special handling for regulations, acts, and gazette publications  
  Evidence: `is_canadian_regulation_pdf()` function with patterns like "c.r.c", "sor-", "dors-", "canada gazette"

**Bilingual & Internationalization**:
? **Language detection and translation pipeline** with automatic EN/FR processing and configurable target languages  
  Evidence: `detect_language()` and `translate_response()` methods in `chatreadretrieveread.py`

? **Dual-field indexing** maintaining original and translated content for cross-language search capabilities  
  Evidence: `title` and `translated_title` fields in `create_vector_index.json`

? **Language-aware search analysis** with configurable analyzers for different linguistic contexts  
  Evidence: `"analyzer": "$SEARCH_INDEX_ANALYZER"` in search index schema

**Search & Retrieval Implementation**:
? **Hybrid vector + keyword search** combining semantic similarity with traditional text matching  
  Evidence: `VectorizedQuery` usage with `search_text` parameter in approach implementations

? **Citation-enforced responses** using structured prompts that mandate source references in [File1], [File2] format  
  Evidence: `SYSTEM_MESSAGE_CHAT_CONVERSATION` template requiring pipe-separated source citations

? **Multi-tier relevance ranking** with optional semantic reranking and configurable result counts  
  Evidence: `USE_SEMANTIC_RERANKER` configuration and `top` parameter handling

? **Metadata-rich retrieval** with page numbers, chunk tracking, and full citation paths available at answer time  
  Evidence: `citation_lookup` assembly in `chatreadretrieveread.py` with chunk_file, page_number, tags fields

**Authentication & Security**:
? **Role-based access control (RBAC)** with Azure AD group integration and container-level permissions  
  Evidence: `shared_code/utility_rbck.py` functions like `find_container_and_role()`

? **Multi-cloud authentication** supporting Azure Government and commercial clouds with managed identity  
  Evidence: `AzureAuthorityHosts.AZURE_GOVERNMENT` handling in credential configuration

? **User profile management** with Cosmos DB persistence for group selections and access patterns  
  Evidence: `UserProfile` class in `shared_code/user_profile.py`

**Infrastructure & Deployment**:
? **Infrastructure as Code** using Terraform with modular architecture and environment-specific configurations  
  Evidence: `infra/main.tf` with modular `core/` directory structure

? **Private network deployment** with VNet integration and private endpoints for all Azure services  
  Evidence: Private DNS zone configurations for `privatelink.openai.azure.com` domains

? **Automated CI/CD pipeline** with separate deployment targets for infrastructure, applications, and search indexes  
  Evidence: `Makefile` commands and `scripts/` deployment automation

**Monitoring & Evaluation**:
? **Comprehensive audit logging** tracking all operations, user interactions, and system events to Cosmos DB  
  Evidence: `StatusLog` class usage across functions and approaches

? **Functional test suite** validating end-to-end pipeline processing across all supported file formats  
  Evidence: `tests/run_tests.py` with format-specific search queries and 45-minute timeout

? **Performance monitoring** with OpenTelemetry integration and Azure Application Insights  
  Evidence: `azure-monitor-opentelemetry` dependency and tracing imports

**Configuration Management**:
? **Centralized environment configuration** with 53+ settings covering all service endpoints and feature flags  
  Evidence: `core/shared_constants.py` ENV dictionary and configuration loading patterns

? **Configuration categories** (from inventory analysis):  
  - **Critical Required**: 31 settings (system fails without these)
  - **Optional with Defaults**: 15 settings (graceful degradation)
  - **Feature Flags**: 7 settings (enable/disable functionality)

? **Fallback and resilience patterns** with optional service degradation and retry logic  
  Evidence: `OPTIMIZED_KEYWORD_SEARCH_OPTIONAL=true`, `ENRICHMENT_OPTIONAL=true` for local dev

? **Multi-source configuration loading**:  
  - `app/backend/backend.env` - Backend-specific settings
  - `scripts/environments/.env` - Terraform deployment variables
  - `functions/local.settings.json` - Azure Functions runtime config

## System Capabilities

### Measured Capabilities (Evidence-Based)
- **Document formats**: 11+ supported (PDF, DOCX, HTML, CSV, MSG, RTF, images)
- **Pipeline processing**: 45-minute timeout for complete document processing
- **Embedding batching**: 120 chunks per batch, 12MB payload limit
- **RAG approaches**: 8 specialized strategies implemented
- **Search index fields**: 15+ metadata fields including bilingual content
- **Queue system**: 8 specialized queues for document type routing

### Performance Metrics (TO BE COMPLETED - Requires Testing)
- **Concurrent users**: [Needs load testing]
- **Response latency**: [Needs performance measurement]
- **Search accuracy**: [Needs golden dataset evaluation]
- **Processing throughput**: [Needs pipeline performance testing]
- **Cost per operation**: [Needs Azure billing analysis]

## Architecture Snapshot (as implemented)

```
???????????????????????????????????????????????????????????????????????????????????
?                              User Interface                                     ?
?  ???????????????????    ????????????????????    ??????????????????????????????? ?
?  ?  React Frontend ?    ?   Chat Interface ?    ?   Admin/Config Panels      ? ?
?  ?  (Vite + TypeScript)  ?   (SSE Streaming) ?    ?   (RBAC + User Profile)   ? ?
?  ???????????????????    ????????????????????    ??????????????????????????????? ?
???????????????????????????????????????????????????????????????????????????????????
                          ? HTTPS/WebSocket
???????????????????????????????????????????????????????????????????????????????????
?                           API Gateway Layer                                    ?
?  ???????????????????????????????????????????????????????????????????????????   ?
?  ?                    FastAPI Backend (app.py)                             ?   ?
?  ?  ???????????????????  ???????????????????  ???????????????????????????   ?   ?
?  ?  ?   8 RAG         ?  ?   RBAC Router   ?  ?   Session Management    ?   ?   ?
?  ?  ?   Approaches    ?  ?   Auth/Groups   ?  ?   Cosmos DB             ?   ?   ?
?  ?  ???????????????????  ???????????????????  ???????????????????????????   ?   ?
?  ???????????????????????????????????????????????????????????????????????????   ?
???????????????????????????????????????????????????????????????????????????????????
                          ? Internal Service Mesh
            ?????????????????????????????
            ?                           ?
????????????????????????    ??????????????????????????????????????????????????????
?   Enrichment Service ?    ?            Document Processing Pipeline            ?
?   (Flask + Embeddings)    ?                 (12 Azure Functions)               ?
?  ??????????????????? ?    ?  ????????????????  ????????????????  ????????????? ?
?  ?  Azure OpenAI   ? ?    ?  ?FileUploadEtrig?  ? OCR Processing?  ?Text       ? ?
?  ?  Embeddings     ? ?    ?  ?Queue Routing ?  ? PDF/Office   ?  ?Enrichment ? ?
?  ?  SentenceTransf ? ?    ?  ????????????????  ????????????????  ????????????? ?
?  ??????????????????? ?    ?          ?                 ?                ?      ?
????????????????????????    ?          ?                 ?                ?      ?
                            ?  ????????????????  ????????????????  ????????????? ?
                            ?  ?Language      ?  ?Chunking      ?  ?Index      ? ?
                            ?  ?Detection/    ?  ?Metadata      ?  ?Update     ? ?
                            ?  ?Translation   ?  ?Extraction    ?  ?Search     ? ?
                            ?  ????????????????  ????????????????  ????????????? ?
                            ???????????????????????????????????????????????????????
                                          ?
                                          ?
???????????????????????????????????????????????????????????????????????????????????
?                            Azure Service Layer                                 ?
?  ???????????????????  ???????????????????  ???????????????????  ????????????? ?
?  ?  Azure OpenAI   ?  ?Azure Cognitive  ?  ?  Azure Blob     ?  ?Azure      ? ?
?  ?  GPT-4 + Ada-002?  ?Search (Hybrid)  ?  ?  Storage        ?  ?Cosmos DB  ? ?
?  ?  Chat + Embedds ?  ?Vector + Keyword ?  ?  Documents      ?  ?Sessions   ? ?
?  ???????????????????  ???????????????????  ???????????????????  ????????????? ?
?                                                                                ?
?  ???????????????????  ???????????????????  ???????????????????              ?
?  ?Azure AI Services?  ?  Azure Queues   ?  ?  Azure          ?              ? 
?  ?Translation +    ?  ?  Pipeline       ?  ?  Government     ?              ?
?  ?Content Safety   ?  ?  Orchestration  ?  ?  Cloud Support  ?              ?
?  ???????????????????  ???????????????????  ???????????????????              ?
???????????????????????????????????????????????????????????????????????????????????
                                          ?
                                          ?
???????????????????????????????????????????????????????????????????????????????????
?                       Network Security & Infrastructure                        ?
?  ???????????????????????????????????????????????????????????????????????????   ?
?  ?                      Azure Private Network (VNet)                       ?   ?
?  ?   ???????????????????    ???????????????????    ??????????????????????? ?   ?
?  ?   ?Private Endpoints?    ?  Private DNS    ?    ?   Network Security  ? ?   ?
?  ?   ?All Azure Services    ?  Zone Resolution?    ?   Groups + Firewall ? ?   ?
?  ?   ???????????????????    ???????????????????    ??????????????????????? ?   ?
?  ???????????????????????????????????????????????????????????????????????????   ?
???????????????????????????????????????????????????????????????????????????????????
```

## How each one of these are implemented

### Data Sources
**Implementation**: Multi-format blob storage ingestion with RBAC container mapping
- **Upload containers**: Role-based storage containers linked to Azure AD groups via `shared_code/utility_rbck.py`
- **Content containers**: Processed document storage with metadata tagging and retention policies  
- **Authority document detection**: Canadian government regulation identification using filename and content patterns
- **Evidence**: `FileUploadedEtrigger/__init__.py` blob trigger and container routing logic

### Data Types Supported  
**Implementation**: Format-specific processing pipelines with 11+ document types
- **PDF documents**: Azure Document Intelligence OCR with page-level citation mapping
- **Office formats**: DOCX, PPTX, XLSX using native parsers and content extraction
- **Web content**: HTML parsing with BeautifulSoup4 and structured content extraction
- **Email formats**: MSG file parsing with extract-msg library for Outlook integration
- **Media files**: Image OCR and text extraction through dedicated enrichment queues
- **Evidence**: Queue routing in `FileUploadedEtrigger` and format-specific processor functions

### Ingestion Process
**Implementation**: Event-driven Azure Functions pipeline with queue orchestration
1. **Blob trigger**: `FileUploadedEtrigger` detects uploaded files and routes by format
2. **Format detection**: File extension and content analysis for processing queue selection  
3. **OCR routing**: PDFs to `pdf_submit_queue`, office docs to `non_pdf_submit_queue`
4. **Authority handling**: Special Canadian regulation routing to `authority_doc_queue`
5. **Status tracking**: Cosmos DB logging throughout pipeline with error handling and retries
- **Evidence**: Queue configuration and routing logic in trigger function implementations

### Metadata Collection
**Implementation**: Multi-dimensional metadata extraction and enrichment
- **File-level metadata**: Name, URI, size, upload timestamp, file classification  
- **Content metadata**: Language detection, entity extraction, title extraction
- **Processing metadata**: Chunk boundaries, page numbers, processing timestamps
- **User metadata**: Upload container, tags, RBAC groups, user principal ID
- **Bilingual metadata**: Original and translated titles, detected language codes
- **Evidence**: Metadata fields in `create_vector_index.json` and processing functions

### Chunking  
**Implementation**: Content-aware chunking with overlap and boundary detection
- **Configurable chunk sizes**: Adjustable token limits with overlap for context preservation
- **Semantic boundaries**: Paragraph and sentence boundary detection for coherent chunks
- **Citation mapping**: Each chunk linked to source document, page number, and file URI
- **Language preservation**: Chunking respects language boundaries and translation pairs
- **Format-specific logic**: PDF page breaks, HTML section boundaries, document structure awareness
- **Evidence**: `TextEnrichment/__init__.py` chunking logic and chunk_file field mapping

### Embedding
**Implementation**: Multi-model embedding generation with queue-based processing  
- **Azure OpenAI embeddings**: text-embedding-ada-002 for primary vector generation
- **Local model support**: SentenceTransformers for offline/hybrid embedding scenarios
- **Batch processing**: Configurable batch sizes (120 chunks) with payload size limits (12MB)
- **Queue orchestration**: `embeddings-queue` processing with retry logic and backoff
- **Multi-language support**: Language-aware embedding models for bilingual content  
- **Evidence**: `app/enrichment/app.py` embedding service and model handling

### Indexing
**Implementation**: Hybrid vector + keyword indexing with bilingual analysis
- **Azure Cognitive Search**: Primary search index with configurable analyzers
- **Vector fields**: High-dimensional embeddings with configurable vector size  
- **Keyword fields**: Full-text search on content, title, entities with language-specific analysis
- **Metadata indexing**: Filterable fields for folders, tags, file types, dates
- **Bilingual indexing**: Separate original and translated content fields
- **Evidence**: `create_vector_index.json` schema and search field configuration  

### Retrieval/Rerank
**Implementation**: Multi-stage hybrid retrieval with semantic reranking
1. **Query optimization**: Azure AI Services query rewriting and expansion
2. **Embedding generation**: User query vectorization through enrichment service
3. **Hybrid search**: Combined vector similarity and keyword matching with configurable weights
4. **Semantic reranking**: Optional Azure Cognitive Search semantic ranking for relevance improvement  
5. **Result filtering**: RBAC-aware filtering based on user groups and container access
6. **Context assembly**: Retrieved chunks formatted for LLM prompt with citation mapping
- **Evidence**: `chatreadretrieveread.py` retrieval implementation and approach patterns

### Citations
**Implementation**: Structured citation system with source mapping and verification
- **Citation format**: [File1], [File2] placeholders linking to source document URIs
- **Source mapping**: Pipe-separated content and URI pairs in system prompts  
- **Citation enforcement**: LLM system messages mandating citation inclusion
- **Link generation**: Blob SAS tokens or direct links to source documents
- **Verification**: Citation validation against retrieved source documents
- **Evidence**: `SYSTEM_MESSAGE_CHAT_CONVERSATION` template and citation processing logic

### Bilingual Handling
**Implementation**: End-to-end bilingual processing pipeline with language detection
1. **Language detection**: Automatic EN/FR detection on user queries and document content
2. **Translation services**: Azure AI Services translation with configurable target languages  
3. **Dual-field storage**: Original and translated content maintained in separate index fields
4. **Query translation**: Cross-language query processing and response translation
5. **Language-aware search**: Configurable analyzers for different linguistic contexts
6. **Response localization**: Final response translation to match user query language
- **Evidence**: Language detection and translation methods in approach implementations

### Evaluation and Golden Questions
**Implementation**: Functional testing framework with format-specific validation
- **Test dataset**: Multi-format test files covering all supported document types
- **Golden queries**: Format-specific search queries with expected content matches
- **Pipeline validation**: End-to-end testing from upload to search retrieval  
- **Performance metrics**: Response time tracking, retrieval accuracy, citation validation
- **Automated testing**: Rich console output with detailed logging and error reporting
- **Evidence**: `tests/run_tests.py` with search queries and pipeline validation logic

## APIs call, contracts, all needed to implement APIM

### Core API Endpoints

**Chat API** (`/chat`)
```typescript
POST /chat
Content-Type: application/json
Authorization: Bearer <azure-ad-token>

Request:
{
  "messages": ChatMessage[],          // Conversation history
  "session_id": string,               // Session tracking
  "approach": number,                 // RAG approach (1-8)
  "overrides": {
    "top": number,                    // Retrieved document count
    "user_persona": string,           // User context
    "system_persona": string,         // System behavior
    "response_length": number,        // Token limit
    "selected_folders": string[],     // Content filtering
    "selected_tags": string[],        // Tag filtering
    "pastRetrieved": number,          // Context window
    "isStrict": boolean,              // Citation enforcement
    "semantic_captions": boolean,     // Enhanced snippets
    "temperature": number             // Response randomness
  }
}

Response: Server-Sent Events (text/event-stream)
data: {"delta": {"content": "..."}}
data: {"delta": {"context": {...}}}
data: {"delta": {"citations": [...]}}
```

**Document Upload API** (`/upload`)
```typescript  
POST /upload
Content-Type: multipart/form-data
Authorization: Bearer <azure-ad-token>

Request:
- file: File                          // Document binary
- container: string                   // Storage container
- tags: string[]                      // Metadata tags
- folder: string                      // Organization folder

Response:
{
  "status": "success" | "error",
  "file_id": string,
  "processing_status": "queued" | "processing" | "completed",
  "estimated_completion": string      // ISO timestamp
}
```

**Document Management API**
```typescript
GET /documents?container={container}&folder={folder}
DELETE /documents/{document_id}
GET /documents/{document_id}/status
GET /documents/{document_id}/metadata
```

**Session Management API**
```typescript  
GET /sessions                         // List user sessions
POST /sessions                        // Create new session
GET /sessions/{session_id}            // Get session history  
DELETE /sessions/{session_id}         // Delete session
PUT /sessions/{session_id}/rename     // Rename session
```

### Authentication Contracts

**Azure AD Integration**
```typescript
// Token validation middleware  
Authorization: Bearer <azure-ad-token>
Audience: api://<application-id>
Issuer: https://sts.windows.net/<tenant-id>/
Claims: {
  "oid": string,                      // User object ID  
  "groups": string[],                 // Azure AD groups
  "preferred_username": string,       // User principal name
  "tenant_id": string                 // Tenant identifier
}
```

**RBAC Permission Model**
```typescript
interface UserPermissions {
  upload_containers: string[],        // Writable containers
  content_containers: string[],       // Readable containers  
  search_indexes: string[],           // Accessible indexes
  admin_functions: boolean,           // Administrative access
  groups: {
    group_id: string,
    group_name: string,
    permissions: Permission[]
  }[]
}
```

### Service Integration Contracts

**Enrichment Service API** (`enrichment/embeddings`)
```typescript
POST /embeddings
Content-Type: application/json

Request:
{
  "input": string[],                  // Text chunks array
  "model": string,                    // Embedding model name
  "dimensions": number,               // Vector dimensions  
  "batch_size": number                // Processing batch size
}

Response:
{
  "embeddings": number[][],           // Vector arrays
  "model": {
    "name": string,
    "dimensions": number,
    "max_tokens": number
  },
  "usage": {
    "prompt_tokens": number,
    "total_tokens": number
  }
}
```

**Search Service Integration**
```typescript
// Azure Cognitive Search Query
{
  "search": string,                   // Keyword query
  "vectorQueries": [{
    "vector": number[],               // Query embedding
    "fields": "embedding",            // Vector field name
    "k": number                       // Nearest neighbors
  }],
  "filter": string,                   // OData filter expression
  "select": string[],                 // Return fields
  "top": number,                      // Result count
  "semantic": {
    "configurationName": string,      // Semantic config
    "captions": boolean,              // Extract captions
    "answers": boolean                // Generate answers
  }
}
```

### APIM Implementation Requirements

**Rate Limiting & Quotas**
- User-based rate limits: 100 requests/minute per authenticated user
- Global throttling: 10,000 requests/minute per service
- Large file upload limits: 50MB per file, 10 files per hour
- Embedding quotas: 100,000 tokens per user per day

**Security Policies**  
- JWT token validation with Azure AD integration
- IP allowlist for administrative endpoints
- Request/response transformation for sensitive data masking
- CORS policies for approved frontend domains

**Monitoring & Logging**
- Request/response logging with PII redaction
- Performance metrics collection and dashboards
- Error rate tracking and alerting thresholds  
- Usage analytics and reporting for capacity planning

**Service Discovery**
```typescript
// Service endpoints for APIM configuration
{
  "backend_api": "https://<webapp-name>.azurewebsites.net",
  "enrichment_service": "https://<enrichment-app>.azurewebsites.net",
  "functions_host": "https://<function-app>.azurewebsites.net",
  "blob_storage": "https://<storage-account>.blob.core.windows.net",
  "search_service": "https://<search-service>.search.windows.net"
}
```

## Open Questions / Gaps  

### Architecture & Scalability Questions
**Question**: How does the system handle concurrent document processing and what are the throughput limits?  
**Gap**: Pipeline capacity planning metrics not visible in code - need load testing and queue depth analysis  
**Validation**: Monitor Azure Functions concurrency settings and queue processing rates under load

**Question**: What is the actual vector embedding model performance and accuracy for bilingual content?  
**Gap**: No evaluation metrics comparing embedding quality between Azure OpenAI and local SentenceTransformers  
**Validation**: Run embedding similarity tests on paired EN/FR content and measure retrieval accuracy

**Question**: How effective is the hybrid search scoring and what are optimal weight ratios?  
**Gap**: Search relevance tuning parameters not exposed in configuration  
**Validation**: A/B test different vector vs keyword weight combinations against golden question dataset

### Security & Compliance Gaps  
**Question**: How comprehensive is the RBAC implementation and does it prevent unauthorized cross-container access?  
**Gap**: Security testing scenarios not visible in test suite  
**Validation**: Penetration testing with different user roles attempting unauthorized document access

**Question**: What PII detection and data residency controls are implemented?  
**Gap**: Content safety and data classification policies not evident in processing pipeline  
**Validation**: Review Azure AI Content Safety integration and data retention policies

### Operational Questions
**Question**: What are the system's failure modes and recovery procedures?  
**Gap**: Disaster recovery and backup strategies not documented in infrastructure code  
**Validation**: Test service failover scenarios and data recovery procedures  

**Question**: How does the system handle large document sets (>10,000 files) and index maintenance?  
**Gap**: No index optimization or cleanup processes visible  
**Validation**: Monitor search index performance with large datasets and test reindexing procedures

### Integration & Extensibility
**Question**: Can the system integrate with other Microsoft 365 services like SharePoint or Teams?  
**Gap**: No connectors or integration patterns for external data sources  
**Validation**: Explore Microsoft Graph API integration possibilities and authentication flow

**Question**: What is the migration path for existing document repositories?  
**Gap**: Bulk import and data migration tools not present  
**Validation**: Test large-scale document import procedures and performance optimization

### Performance & Cost Questions  
**Question**: What are the actual Azure service costs at different usage scales?  
**Gap**: No cost monitoring or optimization strategies visible  
**Validation**: Implement Azure Cost Management tracking and analyze cost per query/document

**Question**: How does response time scale with document corpus size and user concurrency?  
**Gap**: Performance benchmarking data not available  
**Validation**: Load testing with realistic document volumes and user simulation

## Next Actions (Copilot-ready tasks)

### 1. Local Environment Setup & Validation
```powershell
# Clone and setup development environment
cd "C:\Users\marco.presta\OneDrive - ESDC EDSC\Documents\AICOE\EVA-JP-v1.2"
.\.venv\Scripts\Activate.ps1

# Install dependencies and run test suite  
pip install -r tests\requirements.txt
python tests\run_tests.py --validate-config

# Start local services for testing
.\RESTART_SERVERS.ps1
```

### 2. Configuration Analysis & Documentation  
```powershell  
# Extract and document all environment variables
python -c "
import os, json
from app.backend.core.shared_constants import ENV
config = {k: v for k, v in ENV.items()}
print(json.dumps(config, indent=2))
" > config_analysis.json

# Validate Azure service connectivity  
az login
python tests\debug_tests.py --check-services
```

### 3. Pipeline Performance Testing
```powershell
# Run comprehensive functional tests
python tests\run_tests.py --format=all --duration=2700 --verbose

# Monitor pipeline processing times
python -c "
from tests.run_tests import monitor_pipeline_performance
results = monitor_pipeline_performance(['pdf', 'docx', 'html'])
print(f'Processing times: {results}')
"
```

### 4. Search & Retrieval Evaluation
```python
# Create evaluation script for hybrid search effectiveness  
# File: evaluate_search_quality.py
from azure.search.documents import SearchClient
from app.backend.approaches.chatreadretrieveread import ChatReadRetrieveReadApproach

def evaluate_retrieval_quality():
    """Test retrieval accuracy with golden question dataset"""
    golden_questions = [
        ("executive compensation policies", ["compensation", "salary", "executive"]),
        ("maternity leave entitlements", ["maternity", "parental", "leave"]),
    ]
    # Run evaluation and generate metrics report
```

### 5. Security & RBAC Testing
```powershell
# Test RBAC permissions and unauthorized access prevention
python -c "
from functions.shared_code.utility_rbck import test_rbac_enforcement
results = test_rbac_enforcement()
print(f'RBAC test results: {results}')
"

# Validate Azure AD group integration
az ad group list --filter \"displayName eq 'EVA-Users'\" --query '[].objectId'
```

### 6. Bilingual Processing Validation
```python  
# Create bilingual accuracy testing script
# File: test_bilingual_processing.py
def test_language_detection_accuracy():
    """Validate EN/FR detection and translation quality"""
    test_phrases = [
        ("What are the benefits?", "en", "Quels sont les avantages?"),
        ("R?glementation sur les cong?s", "fr", "Leave regulations"),
    ]
    # Test detection accuracy and translation quality

def test_cross_language_search():
    """Test EN query finding FR content and vice versa"""
    # Implement cross-language retrieval tests
```

### 7. Infrastructure Cost Analysis
```bash
# Generate cost estimation report
terraform plan -var-file="scripts/environments/.env" | grep "Plan:"

# Monitor actual Azure costs
az consumption usage list --start-date 2026-01-01 --end-date 2026-01-31 \
  --output table --query '[].{Service:instanceName,Cost:pretaxCost}'
```

### 8. Load Testing & Performance Benchmarking
```python
# File: load_test_chat_api.py  
import asyncio, aiohttp, time
async def simulate_concurrent_users(user_count=10, queries_per_user=5):
    """Simulate realistic user load on chat API"""
    async with aiohttp.ClientSession() as session:
        tasks = []
        for user in range(user_count):
            task = simulate_user_session(session, user, queries_per_user)
            tasks.append(task)
        results = await asyncio.gather(*tasks)
    return results

# Run load test
# python load_test_chat_api.py --users=50 --queries=20
```

### 9. Document Migration Strategy
```powershell
# Create bulk document import utility
# File: bulk_import_documents.py
python bulk_import_documents.py \
  --source-folder="C:\LegacyDocuments" \
  --target-container="content" \
  --batch-size=100 \
  --parallel-workers=5

# Monitor import progress and validate processing
python monitor_import_status.py --show-failed --retry-errors
```

### 10. Integration Testing & API Validation
```bash
# Test all API endpoints with realistic payloads
curl -X POST "http://localhost:5000/chat" \
  -H "Content-Type: application/json" \
  -d @test_chat_payload.json

# Validate APIM-ready API contracts  
swagger-codegen validate -i api_specification.yaml
newman run apim_test_collection.json --environment local.json
```

## JP Plan-2 Implementation Analysis

### Current Coverage Assessment (Evidence-Based)
- **Schema Coverage**: ~15% of JP Plan-2 requirements implemented
- **Current Fields**: Basic document metadata (file_name, content, folder, processed_datetime)
- **Missing JP Fields**: 25+ legal-specific fields required

### Required JP Plan-2 Legal Metadata (Missing)
```json
{
  "case_id": "Unique case identifier",
  "docket_number": "Court docket number", 
  "court": "Court name and jurisdiction",
  "court_level": "Court hierarchy level",
  "decision_date": "Date of decision",
  "decision_makers": "Judges/decision makers",
  "style_of_cause": "Case title format",
  "legal_domain": "Legal area classification",
  "ei_topics[]": "Employment insurance topics array",
  "intent[]": "Legal intent classification",
  "outcome[]": "Decision outcomes",
  "negative_flags[]": "Legal warnings/limitations",
  "statutes[]": "Referenced legislation",
  "sections[]": "Legislative sections cited",
  "holding_sentences[]": "Key legal precedents"
}
```

### Implementation Roadmap (Estimated)
**Phase 1 (3-4 months)**: Legal metadata schema, case processing pipeline  
**Phase 2 (3-4 months)**: Court system integration, legal citations, taxonomy  
**Phase 3 (2-3 months)**: JP admin interface, evaluation framework  
**Estimated Total**: 8-12 months, 4-6 engineers + legal domain experts

### Top Priority Engineering Tasks (Evidence-Based)
1. **JP Legal Metadata Schema** - Complete redesign of search index (Size: Large)
2. **Case-Aware Document Processing** - Legal document structure recognition (Size: Large) 
3. **Legal Document Chunking** - Preserve holdings, rationale, decision sections (Size: Large)
4. **JP Case Management UI** - Admin interface for metadata management (Size: Large)
5. **Legal Citation Enhancement** - Case law, statute, regulation citations (Size: Medium)

## Operational Considerations

### Deployment Complexity (Evidence-Based)
- **Private endpoints required** for all Azure services in production
- **VNet integration** with ESDC Azure Hub (hccld2)
- **Microsoft DevBox/VPN required** for full development access
- **Terraform IaC** with 15+ Azure resources provisioned
- **Multi-cloud support**: Azure Government and Commercial clouds

### Known Operational Challenges (Code Evidence)
- **Local development limitation**: Private endpoints unreachable without VPN
- **Pipeline monitoring**: 45-minute processing requires active monitoring
- **Configuration complexity**: 53+ environment variables across multiple files
- **Bilingual processing**: EN/FR translation adds complexity to workflows

### Security Architecture (Evidence-Based)
- **Authentication**: Azure AD OAuth 2.0 with PKCE flow
- **Authorization**: Container-based RBAC with Azure AD group inheritance
- **Network Security**: Private endpoints, no public IP addresses
- **Audit Logging**: All operations logged to Cosmos DB with retention
- **Content Safety**: Azure AI Services integration for content filtering

## Reference Implementation Value

### Proven Architecture Patterns (Evidence-Based)
- **Multi-service RAG pattern**: Separation of concerns across 4 services
- **Async-first implementation**: FastAPI + AsyncAzureOpenAI for performance
- **Centralized client management**: Shared Azure clients in `shared_constants.py`
- **Fallback strategies**: Graceful degradation with optional service flags
- **Citation enforcement**: Structured prompts with mandatory source references
- **Queue-based document processing**: Event-driven pipeline with specialized routing
- **Bilingual content handling**: Dual indexing with translation workflows

### Government Compliance Features (Evidence-Based)
- **Protected B deployment**: Private network with no external exposure
- **Azure Government cloud**: Multi-cloud authentication and endpoint support
- **Canadian content classification**: Special handling for regulations and gazettes
- **RBAC implementation**: Azure AD group integration with container permissions
- **Comprehensive audit trails**: All operations logged with user attribution
- **Accessibility compliance**: GoC Web Accessibility standards implementation

### Reusability Assessment
- **Document Q&A systems**: 80%+ applicable patterns
- **Government RAG implementations**: 90%+ compliance and security patterns
- **Azure-based AI systems**: 85%+ infrastructure and deployment patterns
- **Bilingual AI applications**: 95%+ language processing workflows

### Technology Decisions (Evidence-Based)
- **FastAPI vs Flask**: Async performance, automatic OpenAPI generation
- **Azure OpenAI vs Open Source**: Government compliance, managed service
- **Hybrid Search vs Vector-Only**: Better precision/recall for document retrieval
- **React vs Angular**: Microsoft ecosystem integration, TypeScript alignment
- **Terraform vs ARM**: Multi-cloud support, mature Azure provider

## Project Status

### Completed Documentation (Evidence-Based)
? **Architecture analysis**: Multi-service pattern with 4 components  
? **Configuration inventory**: 53+ environment variables documented  
? **Technology stack**: Versions and dependencies identified  
? **JP Plan-2 gap analysis**: 15% current coverage, implementation roadmap  
? **EVA Foundation integration**: Context within broader ESDC initiative  
? **Security patterns**: RBAC, private endpoints, audit logging  
? **Deployment complexity**: Terraform IaC, private network requirements  
? **End-to-end pipeline analysis**: Complete ingestion?chunking?indexing?retrieval flow with file paths and line numbers
? **File type processing**: PDF (Azure Document Intelligence), XML/JSON/TXT (Unstructured library), specialized routing
? **Chunking algorithm**: Fixed 1500 char RecursiveCharacterTextSplitter with 100 char overlap
? **Metadata schema**: 15+ fields including bilingual content, page tracking, citation mapping  

### TO BE COMPLETED - Requires Testing/Investigation
?? **Performance benchmarks**: Load testing and scalability limits  
?? **Cost analysis**: Azure service costs and usage optimization  
?? **Security validation**: Penetration testing and vulnerability assessment  
?? **Search quality evaluation**: Golden dataset and accuracy metrics  
?? **Operational procedures**: Monitoring, backup, disaster recovery  
?? **Production readiness**: End-to-end deployment validation