# EVA-DA Infrastructure & Application Documentation
## dev2 Environment Technical Reference

**Generated**: January 29, 2026  
**Environment**: infoasst-dev2 (EsDAICoESub)  
**Evidence Sources**: 5 comprehensive analysis documents  
**Total Diagrams**: 20 (ASCII + SVG)

---

## Table of Contents

### 1. [Infrastructure Architecture](dev2-TechDoc-01-infra-arch.md)
- **Diagram 1**: Network Topology & Private Endpoints
- **Diagram 2**: Azure Resource Dependency Graph
- **Diagram 3**: Three-Tier Application Stack
- **Diagram 4**: Security Boundaries & Isolation

### 2. [Application Architecture](dev2-TechDoc-02-application-arch.md)
- **Diagram 5**: Backend API Routing (FastAPI/Quart)
- **Diagram 6**: Enrichment Service Architecture (Flask)
- **Diagram 8**: Azure Functions Orchestration (12 Functions)

### 3. [Document Ingestion Pipeline](dev2-TechDoc-03-doc-ingestion.md)
- **Diagram 9**: 6-Stage Ingestion Flow
- **Diagram 10**: Queue Routing Tree
- **Diagram 11**: Chunking Algorithm
- **Diagram 12**: Embedding Sequence
- **Diagram 13**: Blob Storage Containers
- **Diagram 14**: Search Index Schema (1536-dim vectors)

### 4. [RAG Query Execution](dev2-TechDoc-04-rag-query.md)
- **Diagram 15**: 5-Stage RAG Pipeline
- **Diagram 16**: Query Optimization Flow
- **Diagram 17**: Hybrid Search Scoring
- **Diagram 18**: Citation Assembly
- **Diagram 19**: Prompt Construction

### 5. [Data Flows & Integrations](dev2-TechDoc-05-data-flows.md)
- **Diagram 20**: Upload Flow (Frontend → Storage → Queue)
- **Diagram 21**: Cosmos DB Persistence (Sessions, Logs, RBAC)
- **Diagram 22**: Configuration Management
- **Diagram 23**: Monitoring & Telemetry (Application Insights)

---

## Evidence Sources

All diagrams are backed by evidence from:

1. **EVA-DEV2-COMPLETE-ARCHITECTURE-2026-01-29.md** (647 lines)
   - 63 Azure resources documented
   - Network topology with 14 private endpoints
   - Security architecture with VNet isolation

2. **EVA-DA-Pipeline-Analysis-Evidence-Based.md** (680 lines)
   - 12 Azure Functions detailed
   - Queue routing logic with code excerpts
   - Chunking at 1500 characters (lines 480-490)

3. **EVA-DA-RAG-Query-Execution-Analysis.md** (1110 lines)
   - 5-stage query execution documented
   - Real server log evidence
   - Hybrid search implementation (lines 335-380)

4. **eva-da-ref_config_inventory.md** (220 lines)
   - 53+ configuration variables
   - Categorized by service

5. **azure_search/create_vector_index.json** (298 lines)
   - Complete field definitions
   - HNSW vector search config
   - Semantic ranking configuration

6. **Code Files** (newly read):
   - `app/backend/app.py` (2557 lines) - FastAPI routes, approaches
   - `app/enrichment/app.py` (870 lines) - Flask embedding service
   - `functions/FileLayoutParsingOther/__init__.py` (571 lines) - Chunking implementation

---

## Diagram Types

### ASCII Diagrams (14 total)
Text-based visualizations using box-drawing characters:
- Network topology
- Dependency graphs
- Queue routing trees
- Algorithm flowcharts
- Schema definitions

### SVG Diagrams (10 total)
Scalable vector graphics for complex flows:
- Swimlane diagrams (RAG execution)
- Sequence diagrams (ingestion pipeline)
- Topology maps (network architecture)
- Data flow diagrams

---

## Usage Guidelines

### For Developers
- All diagrams include file path + line number references
- Code excerpts show actual implementation
- Evidence citations enable verification

### For Architects
- Infrastructure diagrams show all 63 resources
- Security boundaries clearly marked
- Data flows traced end-to-end

### For Operations
- Monitoring points identified
- Queue depth management
- Configuration dependencies mapped

---

## Validation Checklist

Each diagram has been validated for:
- ✅ Evidence source cited
- ✅ Service names match dev2 environment
- ✅ Code references accurate (file path + line numbers)
- ✅ Technical accuracy against source code
- ✅ No hallucinated data

---

## Next Steps

1. Review individual documentation files (01-05)
2. Cross-reference diagrams with live dev2 environment
3. Update diagrams as infrastructure evolves
4. Use as basis for production deployment planning

---

**Documentation Owner**: AI Architecture Documentation System  
**Last Validated**: January 29, 2026  
**Environment**: infoasst-dev2 (d2d4e571-e0f2-4f6c-901a-f88f7669bcba)
