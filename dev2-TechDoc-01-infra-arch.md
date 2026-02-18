# Infrastructure Architecture
## infoasst-dev2 Environment

**Evidence Source**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 1-647)  
**Environment**: EsDAICoESub (d2d4e571-e0f2-4f6c-901a-f88f7669bcba)  
**Resource Group**: infoasst-dev2  
**Region**: Canada East

---

## Diagram 1: Network Topology & Private Endpoints

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Internet (Public Traffic)                             │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     │ HTTPS/TLS 1.2+
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Azure Front Door (Optional)                            │
└────────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     │ 443
                                     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                    infoasst-dev2 Resource Group                               │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │                  VNet: infoasst-dev2-vnet (10.0.0.0/16)             │     │
│  │                                                                       │     │
│  │  ┌──────────────────────────────────────────────────────────────┐   │     │
│  │  │  Subnet: default (10.0.0.0/24)                               │   │     │
│  │  │                                                                │   │     │
│  │  │  ┌──────────────────────────────────────────────────────┐   │   │     │
│  │  │  │  App Service Plan: infoasst-webapp-asp-dev2         │   │   │     │
│  │  │  │    - SKU: PremiumV2 P1v2 (auto-scale 1-3)           │   │   │     │
│  │  │  │                                                        │   │   │     │
│  │  │  │  ┌────────────────────────────────────────────┐     │   │   │     │
│  │  │  │  │ App Service: infoasst-webapp-dev2          │     │   │   │     │
│  │  │  │  │   - Backend API (Python/FastAPI)           │     │   │   │     │
│  │  │  │  │   - Frontend SPA (React/TypeScript)        │     │   │   │     │
│  │  │  │  │   - Port: 443 (HTTPS only)                 │     │   │   │     │
│  │  │  │  │   - Health: /health                        │     │   │   │     │
│  │  │  │  └────────────────────────────────────────────┘     │   │   │     │
│  │  │  └──────────────────────────────────────────────────────┘   │   │     │
│  │  │                                                                │   │     │
│  │  │  ┌──────────────────────────────────────────────────────┐   │   │     │
│  │  │  │  App Service Plan: infoasst-enrichmentweb-asp-dev2  │   │   │     │
│  │  │  │    - SKU: PremiumV2 P1v2                             │   │   │     │
│  │  │  │                                                        │   │   │     │
│  │  │  │  ┌────────────────────────────────────────────┐     │   │   │     │
│  │  │  │  │ App Service: infoasst-enrichmentweb-dev2   │     │   │   │     │
│  │  │  │  │   - Flask Embedding API                    │     │   │   │     │
│  │  │  │  │   - Endpoints: /models, /models/{m}/embed  │     │   │   │     │
│  │  │  │  │   - Queue Polling Thread (5s interval)     │     │   │   │     │
│  │  │  │  └────────────────────────────────────────────┘     │   │   │     │
│  │  │  └──────────────────────────────────────────────────────┘   │   │     │
│  │  │                                                                │   │     │
│  │  │  ┌──────────────────────────────────────────────────────┐   │   │     │
│  │  │  │  App Service Plan: infoasst-func-asp-dev2            │   │   │     │
│  │  │  │    - SKU: ElasticPremium EP1                         │   │   │     │
│  │  │  │                                                        │   │   │     │
│  │  │  │  ┌────────────────────────────────────────────┐     │   │   │     │
│  │  │  │  │ Function App: infoasst-func-dev2           │     │   │   │     │
│  │  │  │  │   - 12 Azure Functions (Python 3.11)       │     │   │   │     │
│  │  │  │  │   - Blob triggers + Queue triggers         │     │   │   │     │
│  │  │  │  │   - Document processing pipeline           │     │   │   │     │
│  │  │  │  └────────────────────────────────────────────┘     │   │   │     │
│  │  │  └──────────────────────────────────────────────────────┘   │   │     │
│  │  └──────────────────────────────────────────────────────────────┘   │     │
│  │                                                                       │     │
│  │  ┌──────────────────────────────────────────────────────────────┐   │     │
│  │  │  Subnet: PrivateEndpointSubnet (10.0.1.0/24)                 │   │     │
│  │  │                                                                │   │     │
│  │  │  [Private Endpoints - 14 total]                               │   │     │
│  │  │                                                                │   │     │
│  │  │  1. infoasst-aoai-dev2-pe           → Azure OpenAI            │   │     │
│  │  │  2. infoasst-aisvc-dev2-pe          → AI Services             │   │     │
│  │  │  3. infoasst-search-dev2-pe         → Cognitive Search        │   │     │
│  │  │  4. infoasst-cosmos-dev2-pe         → Cosmos DB               │   │     │
│  │  │  5-8. infoasst-store-dev2-blob-pe   → Blob Storage (4 PEs)    │   │     │
│  │  │  9. infoasst-store-dev2-queue-pe    → Queue Storage           │   │     │
│  │  │  10. infoasst-store-dev2-file-pe    → File Storage            │   │     │
│  │  │  11. infoasst-store-dev2-table-pe   → Table Storage           │   │     │
│  │  │  12. infoasst-docint-dev2-pe        → Document Intelligence   │   │     │
│  │  │  13. infoasst-kv-dev2-pe            → Key Vault               │   │     │
│  │  │  14. infoasst-funcstore-dev2-pe     → Function Storage        │   │     │
│  │  │                                                                │   │     │
│  │  │  [Network Security Group: infoasst-dev2-nsg]                  │   │     │
│  │  │    - Inbound: Deny all except VNet, Azure services           │   │     │
│  │  │    - Outbound: Allow VNet, Azure services, Internet (443)    │   │     │
│  │  └──────────────────────────────────────────────────────────────┘   │     │
│  │                                                                       │     │
│  └───────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  [Azure Services - All with publicNetworkAccess: Disabled]                    │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Azure OpenAI: infoasst-aoai-dev2                                   │     │
│  │    - Endpoint: https://infoasst-aoai-dev2.openai.azure.com          │     │
│  │    - Deployments: gpt-4o, text-embedding-3-small                    │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  AI Services: infoasst-aisvc-dev2                                   │     │
│  │    - Endpoint: https://infoasst-aisvc-dev2.cognitiveservices...    │     │
│  │    - Services: Query optimization, content safety                   │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Cognitive Search: infoasst-search-dev2                             │     │
│  │    - Endpoint: https://infoasst-search-dev2.search.windows.net      │     │
│  │    - Indexes: Multiple (RBAC-controlled)                            │     │
│  │    - Hybrid Search: Vector (1536-dim) + Keyword (BM25)              │     │
│  │    - Semantic Ranking: Enabled                                      │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Cosmos DB: infoasst-cosmos-dev2                                    │     │
│  │    - Endpoint: https://infoasst-cosmos-dev2.documents.azure.com     │     │
│  │    - Databases: chatlogs, user_profile                              │     │
│  │    - Containers: conversations, group_management, user_info         │     │
│  │    - Consistency: Session                                           │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Storage Account: infoasststoredev2                                 │     │
│  │    - Endpoint: https://infoasststoredev2.blob.core.windows.net      │     │
│  │    - Containers: 20+ (upload-*, content-*, config)                  │     │
│  │    - Queues: 8 specialized queues                                   │     │
│  │    - Private Only: ✓ (4 blob, 1 queue, 1 file, 1 table PE)         │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Document Intelligence: infoasst-docint-dev2                        │     │
│  │    - Endpoint: https://infoasst-docint-dev2.cognitiveservices...   │     │
│  │    - Purpose: OCR for PDF, images                                   │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  Key Vault: infoasst-kv-dev2                                        │     │
│  │    - Endpoint: https://infoasst-kv-dev2.vault.azure.net             │     │
│  │    - Secrets: API keys, connection strings                          │     │
│  │    - Private Only: ✓                                                │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                │
│  [Private DNS Zones - 4 total]                                                │
│    1. privatelink.openai.azure.com                                            │
│    2. privatelink.cognitiveservices.azure.com                                 │
│    3. privatelink.search.windows.net                                          │
│    4. privatelink.documents.azure.com                                         │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

**Evidence**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 50-250)  
**Key Features**:
- 14 private endpoints - all Azure services isolated
- VNet with 2 subnets (default 10.0.0.0/24, PrivateEndpointSubnet 10.0.1.0/24)
- Network Security Group enforcing deny-by-default
- 4 Private DNS zones for name resolution
- All services configured with `publicNetworkAccess: Disabled`

---

## Diagram 2: Azure Resource Dependency Graph

```
                        [User Traffic]
                              │
                              ▼
                ┌─────────────────────────┐
                │  Frontend (React SPA)   │
                │  Served by App Service  │
                └─────────────┬───────────┘
                              │
                              │ HTTPS API calls
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│                   Backend API (app.py - 2557 lines)                   │
│                   FastAPI with 20+ endpoints                          │
└──┬────────────┬────────────┬────────────┬────────────┬───────────────┘
   │            │            │            │            │
   │            │            │            │            │
   ▼            ▼            ▼            ▼            ▼
┌──────┐  ┌──────────┐ ┌──────────┐ ┌─────────┐ ┌──────────┐
│Azure │  │ Cognitive│ │ Cosmos   │ │  Blob   │ │    AI    │
│OpenAI│  │  Search  │ │    DB    │ │ Storage │ │ Services │
└──┬───┘  └────┬─────┘ └────┬─────┘ └────┬────┘ └────┬─────┘
   │           │            │            │           │
   │           │            │            │           │
   │           │            │            │           │
   └───────────┴────────────┴────────────┴───────────┘
                     │
                     │ All via Private Endpoints
                     ▼
         ┌───────────────────────────┐
         │  VNet: infoasst-dev2-vnet │
         │  PrivateEndpointSubnet    │
         └───────────────────────────┘
                     │
                     ▼
         ┌───────────────────────────┐
         │    Private DNS Zones      │
         │  Name resolution for PEs  │
         └───────────────────────────┘


[Document Processing Pipeline]

User Upload → Blob Storage (upload-container)
                     │
                     │ Blob trigger
                     ▼
         ┌───────────────────────────┐
         │  Function App (12 funcs)  │
         │  infoasst-func-dev2       │
         └─────┬─────────────────────┘
               │
               ├─→ Queue: non-pdf-submit-queue
               ├─→ Queue: pdf-submit-queue
               ├─→ Queue: pdf-polling-queue
               ├─→ Queue: media-submit-queue
               ├─→ Queue: image-enrichment-queue
               ├─→ Queue: text-enrichment-queue
               ├─→ Queue: embeddings-queue
               └─→ Queue: resubmit-processing-queue
                     │
                     ▼
         ┌───────────────────────────┐
         │ Enrichment Web App        │
         │ Flask + Queue Polling     │
         └─────┬─────────────────────┘
               │
               ├─→ Azure OpenAI (embeddings)
               │
               └─→ Cognitive Search (indexing)
                     │
                     ▼
         ┌───────────────────────────┐
         │  Search Index              │
         │  (1536-dim vectors)        │
         └───────────────────────────┘


[Monitoring & Logging]

All Services ─→ Application Insights ─→ Azure Monitor
                     │
                     └─→ Log Analytics Workspace
```

**Evidence**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 250-400)  
**Key Dependencies**:
- Backend depends on: Azure OpenAI, Cognitive Search, Cosmos DB, Blob Storage, AI Services
- Function App depends on: Blob Storage (triggers), Queue Storage, Document Intelligence
- Enrichment App depends on: Azure OpenAI (embeddings), Queue Storage, Cognitive Search
- All services require: VNet integration, Private Endpoints, Private DNS

---

## Diagram 3: Three-Tier Application Stack

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PRESENTATION TIER                                 │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Frontend SPA (React 18.2.0 + TypeScript)                      │  │
│  │    Location: app/frontend/                                      │  │
│  │    Build: Vite 4.2.1                                            │  │
│  │    Components: Chat, Upload, Sessions, Admin                    │  │
│  │    State: React Context (AppContext.tsx)                        │  │
│  │    API Client: src/api/api.ts                                   │  │
│  │    Served by: App Service (infoasst-webapp-dev2)                │  │
│  │    Port: 443 (HTTPS)                                            │  │
│  └────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────┬─────────────────────────────────┘
                                     │
                                     │ HTTPS/CORS
                                     │ JSON/SSE
                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     APPLICATION TIER                                  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Backend API (FastAPI - app.py 2557 lines)                     │  │
│  │    Location: app/backend/                                       │  │
│  │    Framework: FastAPI 0.115.0 (async)                          │  │
│  │    Python: 3.11+                                                │  │
│  │    Port: 5000 (dev), 443 (prod)                                │  │
│  │                                                                  │  │
│  │  Key Routes (lines 600-1000):                                   │  │
│  │    - POST /chat                 → Chat with RAG                │  │
│  │    - POST /upload               → Document upload              │  │
│  │    - GET  /sessions             → Session history              │  │
│  │    - POST /getalluploadstatus   → Upload status                │  │
│  │    - POST /getcitation          → Get document citation        │  │
│  │    - GET  /health               → Health check                 │  │
│  │    - GET  /getInfoData          → System info                  │  │
│  │                                                                  │  │
│  │  RAG Approaches (lines 350-400):                                │  │
│  │    - ChatReadRetrieveReadApproach (primary)                    │  │
│  │    - ChatWebRetrieveRead (web search integration)              │  │
│  │    - CompareWebWithWork, CompareWorkWithWeb                    │  │
│  │    - GPTDirectApproach                                          │  │
│  │                                                                  │  │
│  │  Dependencies:                                                   │  │
│  │    - core/shared_constants.py   → Azure client initialization  │  │
│  │    - approaches/                → RAG implementations           │  │
│  │    - routers/sessions.py        → Session management           │  │
│  │    - auth/                      → Entra ID OAuth 2.0           │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Enrichment Service (Flask - app.py 870 lines)                 │  │
│  │    Location: app/enrichment/                                    │  │
│  │    Framework: Flask                                             │  │
│  │    Purpose: Embedding generation + Queue polling               │  │
│  │                                                                  │  │
│  │  Key Endpoints (lines 250-300):                                 │  │
│  │    - GET  /models               → List embedding models        │  │
│  │    - GET  /models/{model}       → Model info                   │  │
│  │    - POST /models/{model}/embed → Generate embeddings          │  │
│  │                                                                  │  │
│  │  Background Worker (lines 350-400):                             │  │
│  │    - poll_queue_thread()        → 5-second polling interval    │  │
│  │    - Process embeddings-queue messages                         │  │
│  │    - Batch upload to Cognitive Search                          │  │
│  │                                                                  │  │
│  │  Models Supported:                                               │  │
│  │    - azure-openai_text-embedding-3-small (1536-dim)            │  │
│  │    - Sentence Transformers models                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Azure Functions (12 Functions - Python 3.11)                  │  │
│  │    Location: functions/                                         │  │
│  │    Runtime: Python 3.11, Azure Functions v4                    │  │
│  │                                                                  │  │
│  │  Functions:                                                      │  │
│  │    1. FileUploadedEtrigger      → Router (blob trigger)        │  │
│  │    2. FileFormRecSubmissionPDF  → PDF OCR                      │  │
│  │    3. FileLayoutParsingPDF      → PDF layout parsing           │  │
│  │    4. FileLayoutParsingOther    → Non-PDF parsing (1500 chars)│  │
│  │    5. MediaSubmission           → Image/audio processing       │  │
│  │    6. ImageEnrichment           → Image analysis               │  │
│  │    7. TextEnrichment            → Metadata extraction          │  │
│  │    8. ProcessingQueue           → Reprocess failed items       │  │
│  │    9-12. [Additional pipeline functions]                       │  │
│  │                                                                  │  │
│  │  Chunking (FileLayoutParsingOther/__init__.py lines 470-490):  │  │
│  │    text_splitter = RecursiveCharacterTextSplitter(              │  │
│  │      separators=["\n\n", "\n", ".", ",", " ", ""],            │  │
│  │      chunk_size=1500,                                           │  │
│  │      chunk_overlap=100                                          │  │
│  │    )                                                             │  │
│  └────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────┬─────────────────────────────────┘
                                     │
                                     │ Azure SDK
                                     │ Private Endpoints
                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        DATA TIER                                      │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Azure OpenAI (infoasst-aoai-dev2)                             │  │
│  │    - gpt-4o deployment (chat completions)                      │  │
│  │    - text-embedding-3-small (1536-dim vectors)                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Cognitive Search (infoasst-search-dev2)                       │  │
│  │    - Multiple indexes (RBAC-controlled)                        │  │
│  │    - Hybrid search: Vector (HNSW) + Keyword (BM25)             │  │
│  │    - Semantic ranking enabled                                   │  │
│  │    - Schema: 15+ metadata fields, contentVector (1536-dim)     │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Cosmos DB (infoasst-cosmos-dev2)                              │  │
│  │    - Database: chatlogs                                         │  │
│  │      - Container: conversations (session history)              │  │
│  │    - Database: user_profile                                     │  │
│  │      - Container: group_management (RBAC mappings)             │  │
│  │      - Container: user_info (user profiles)                    │  │
│  │    - Consistency: Session                                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Blob Storage (infoasststoredev2)                              │  │
│  │    - Containers: 20+ (upload-*, content-*, config)             │  │
│  │    - Queues: 8 specialized queues                              │  │
│  │      - non-pdf-submit-queue                                     │  │
│  │      - pdf-submit-queue, pdf-polling-queue                     │  │
│  │      - media-submit-queue, image-enrichment-queue              │  │
│  │      - text-enrichment-queue, embeddings-queue                 │  │
│  │      - resubmit-processing-queue                               │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Document Intelligence (infoasst-docint-dev2)                  │  │
│  │    - OCR for PDF, images                                        │  │
│  │    - Layout analysis                                            │  │
│  │    - Table extraction                                           │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

**Evidence**: 
- app/backend/app.py (lines 1-1000, 600-800 for routes)
- app/enrichment/app.py (lines 1-400)
- functions/FileLayoutParsingOther/__init__.py (lines 470-490)

**Key Characteristics**:
- **Presentation**: React SPA with TypeScript, served by App Service
- **Application**: FastAPI backend (async), Flask enrichment, 12 Azure Functions
- **Data**: Azure OpenAI, Cognitive Search, Cosmos DB, Blob Storage, Document Intelligence
- **Communication**: All via private endpoints, VNet-isolated

---

## Diagram 4: Security Boundaries & Isolation

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         INTERNET (Untrusted Zone)                             │
└────────────────────────────────────────┬─────────────────────────────────────┘
                                         │
                                         │ HTTPS/TLS 1.2+
                                         │ Certificate Validation
                                         ▼
                    ┌─────────────────────────────────────┐
                    │     Security Boundary #1:           │
                    │     Network Isolation                │
                    │  - Public endpoint disabled          │
                    │  - VNet integration required         │
                    │  - NSG deny-by-default               │
                    └─────────────────┬───────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                  AZURE VNET: infoasst-dev2-vnet (10.0.0.0/16)                │
│                        (Trusted Internal Zone)                                │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                   Security Boundary #2:                                 │  │
│  │                   Identity & Access Control                             │  │
│  │   - Azure AD authentication (Entra ID)                                  │  │
│  │   - Managed Identity for services                                       │  │
│  │   - RBAC at document/index level                                        │  │
│  │   - No API keys in code                                                 │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Subnet: default (10.0.0.0/24)                                          │  │
│  │                                                                          │  │
│  │  [Compute Resources - Managed Identity Enabled]                         │  │
│  │    - App Service: infoasst-webapp-dev2                                  │  │
│  │    - App Service: infoasst-enrichmentweb-dev2                           │  │
│  │    - Function App: infoasst-func-dev2                                   │  │
│  │                                                                          │  │
│  │  [Security Controls]                                                     │  │
│  │    - HTTPS enforced (TLS 1.2 minimum)                                   │  │
│  │    - Authentication: OAuth 2.0 with PKCE                                │  │
│  │    - Session management: JWT tokens                                     │  │
│  │    - CORS: Whitelisted origins only                                     │  │
│  │    - Input validation: Pydantic models                                  │  │
│  │    - Rate limiting: Per-user, per-endpoint                              │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Subnet: PrivateEndpointSubnet (10.0.1.0/24)                            │  │
│  │                                                                          │  │
│  │                   Security Boundary #3:                                  │  │
│  │                   Data-at-Rest Isolation                                 │  │
│  │   - All data services behind private endpoints                          │  │
│  │   - No public internet access                                           │  │
│  │   - Private DNS resolution only                                         │  │
│  │                                                                          │  │
│  │  [Private Endpoints - 14 total]                                         │  │
│  │    1. Azure OpenAI              → publicNetworkAccess: Disabled         │  │
│  │    2. AI Services                → publicNetworkAccess: Disabled         │  │
│  │    3. Cognitive Search           → publicNetworkAccess: Disabled         │  │
│  │    4. Cosmos DB                  → publicNetworkAccess: Disabled         │  │
│  │    5-8. Blob Storage (4 PEs)     → publicNetworkAccess: Disabled         │  │
│  │    9. Queue Storage              → publicNetworkAccess: Disabled         │  │
│  │    10. File Storage              → publicNetworkAccess: Disabled         │  │
│  │    11. Table Storage             → publicNetworkAccess: Disabled         │  │
│  │    12. Document Intelligence     → publicNetworkAccess: Disabled         │  │
│  │    13. Key Vault                 → publicNetworkAccess: Disabled         │  │
│  │    14. Function Storage          → publicNetworkAccess: Disabled         │  │
│  │                                                                          │  │
│  │  [Network Security Group: infoasst-dev2-nsg]                            │  │
│  │    Inbound Rules:                                                        │  │
│  │      - Priority 100: Deny All                                           │  │
│  │      - Priority 200: Allow VNet                                         │  │
│  │      - Priority 300: Allow AzureLoadBalancer                            │  │
│  │    Outbound Rules:                                                       │  │
│  │      - Priority 100: Allow VNet                                         │  │
│  │      - Priority 200: Allow Azure Services (ServiceTag)                  │  │
│  │      - Priority 300: Allow Internet (443) for Azure SDK                 │  │
│  │      - Priority 400: Deny All                                           │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                   Security Boundary #4:                                  │  │
│  │                   Data-in-Transit Encryption                             │  │
│  │   - All connections via HTTPS (TLS 1.2+)                                │  │
│  │   - Azure SDK uses secure channels                                      │  │
│  │   - Private endpoint traffic stays on Microsoft backbone                │  │
│  │   - Certificate validation enforced                                     │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                   Security Boundary #5:                                  │  │
│  │                   Application-Level Security                             │  │
│  │                                                                          │  │
│  │  [Backend Security (app.py)]                                             │  │
│  │    - Input validation: Pydantic models, type hints                      │  │
│  │    - Output sanitization: XSS prevention                                │  │
│  │    - SQL injection protection: Parameterized queries                    │  │
│  │    - Error handling: Structured logging, no stack traces to user       │  │
│  │    - Rate limiting: 100 req/min global, 20 req/min /chat               │  │
│  │    - Session management: Cosmos DB with TTL (30 min)                   │  │
│  │                                                                          │  │
│  │  [RBAC Enforcement (shared_code/utility_rbck.py)]                       │  │
│  │    - User → Group mapping (Azure AD)                                    │  │
│  │    - Group → Container/Index mapping (Cosmos DB)                        │  │
│  │    - Document-level access control                                      │  │
│  │    - Role validation: owner, contributor, reader                        │  │
│  │                                                                          │  │
│  │  [Audit Logging]                                                         │  │
│  │    - All operations logged to Cosmos DB (StatusLog)                     │  │
│  │    - Application Insights for monitoring                                │  │
│  │    - Retention: 7 years (compliance)                                    │  │
│  │    - Immutable logs (append-only)                                       │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘

[Private DNS Zones - Name Resolution Security]

┌──────────────────────────────────────────────────────────────────┐
│  privatelink.openai.azure.com                                     │
│    Resolves: infoasst-aoai-dev2.openai.azure.com                 │
│    To: 10.0.1.x (Private IP in PrivateEndpointSubnet)            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  privatelink.cognitiveservices.azure.com                          │
│    Resolves: infoasst-aisvc-dev2.cognitiveservices.azure.com     │
│    To: 10.0.1.y (Private IP in PrivateEndpointSubnet)            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  privatelink.search.windows.net                                   │
│    Resolves: infoasst-search-dev2.search.windows.net             │
│    To: 10.0.1.z (Private IP in PrivateEndpointSubnet)            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  privatelink.documents.azure.com                                  │
│    Resolves: infoasst-cosmos-dev2.documents.azure.com            │
│    To: 10.0.1.w (Private IP in PrivateEndpointSubnet)            │
└──────────────────────────────────────────────────────────────────┘

[Key Security Principles]

1. Defense in Depth: 5 layers of security
2. Zero Trust: Verify every access, assume breach
3. Least Privilege: Minimum permissions required
4. Encryption Everywhere: At-rest and in-transit
5. Immutable Logging: Audit trail for compliance
```

**Evidence**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 400-550)  
**Security Model**:
- **5 Defense Layers**: Network, Identity, Data-at-Rest, Data-in-Transit, Application
- **Zero Trust Architecture**: All services require authentication, no implicit trust
- **Private Endpoints**: 14 endpoints ensure no public internet exposure
- **RBAC**: Document/index-level access control via Azure AD + Cosmos DB
- **Audit Trail**: All operations logged, 7-year retention

---

## Resource Summary

| Category | Count | Purpose |
|----------|-------|---------|
| **Compute** | 6 | 3 App Services + 3 App Service Plans |
| **AI Services** | 4 | Azure OpenAI, AI Services, Cognitive Search, Document Intelligence |
| **Data** | 2 | Cosmos DB, Blob Storage |
| **Network** | 17 | 1 VNet, 1 NSG, 14 Private Endpoints, 1 DNS Resolver |
| **Security** | 5 | 4 Private DNS Zones, 1 Key Vault |
| **Total** | 63 | Complete EVA RAG deployment |

**Estimated Monthly Cost**: $540-920 (based on SKUs and usage patterns)  
**Evidence**: EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md (lines 550-647)

---

**Next Section**: [Application Architecture](dev2-TechDoc-02-application-arch.md) - Detailed component diagrams for Backend API, Enrichment Service, and Azure Functions
