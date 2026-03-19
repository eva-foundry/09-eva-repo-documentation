<!-- eva-primed-plan -->

## EVA Ecosystem Tools

- Data model: GET http://localhost:8010/model/projects/09-eva-repo-documentation
- 29-foundry agents: C:\eva-foundry\eva-foundation\29-foundry\agents\
- 48-eva-veritas audit: run audit_repo MCP tool

---

# EVA Domain Assistant (dev2) - Comprehensive Architecture Documentation Proposal
2026-01-29

## EXECUTIVE SUMMARY

This proposal outlines comprehensive visual documentation of the **EVA Jurisprudence Domain Assistant** deployment in **infoasst-dev2**, integrating:

1. **Azure Infrastructure Topology** (63 resources discovered)
2. **Document Ingestion Pipeline** (12 Azure Functions, 8 processing queues)
3. **RAG Query Execution Flow** (5-stage hybrid search with GPT-4)
4. **Service Interconnections** (Frontend ? Backend ? Enrichment ? Azure services)
5. **Data Flow Diagrams** (Blob storage ? Vector index ? LLM response)

**Deliverable:** 
Comprehensive set of documents with **15+ ASCII diagrams** and **10+ SVG diagrams** documenting every architectural layer from network topology to token-level prompt construction.

---

## PROPOSED DIAGRAM CATALOG

### Part 1: Infrastructure Architecture (4 diagrams)
1. **Network Topology** - VNet, subnets, NSG, DNS resolver (ASCII + SVG)
2. **Private Endpoint Architecture** - 14 endpoints with DNS resolution (SVG)
3. **Resource Dependency Graph** - 63 resources deployment order (ASCII)
4. **Security Boundaries** - Zone-based security model (SVG)

### Part 2: Application Architecture (4 diagrams)
5. **Three-Tier Stack** - Frontend/Backend/Functions/Data layers (ASCII)
6. **Backend API Routing** - Route dispatcher with approach mapping (ASCII)
7. **Frontend Component Tree** - React component hierarchy (ASCII)
8. **Enrichment Service** - Flask embedding service + queue processor (ASCII + SVG)

### Part 3: Document Ingestion (6 diagrams)
9. **Complete Ingestion Flow** - Multi-page swimlane (SVG - 6 stages)
10. **Azure Functions Orchestration** - Function chain with queues (ASCII)
11. **Queue Routing Decision Tree** - File-type routing logic (ASCII)
12. **Chunking Algorithm** - Text splitting visualization (ASCII)
13. **Embedding Generation** - Sequence diagram (SVG)
14. **Search Index Schema** - Field hierarchy with annotations (ASCII)

### Part 4: RAG Query Execution (5 diagrams)
15. **Five-Stage RAG Pipeline** - Full horizontal swimlane (SVG)
16. **Hybrid Search Mechanism** - Algorithm + scoring (ASCII + SVG)
17. **Citation Assembly** - Metadata transformation (ASCII)
18. **LLM Prompt Construction** - Token-level breakdown (ASCII)
19. **Streaming Response** - SSE sequence with timeline (SVG)

### Part 5: Data Flows (4 diagrams)
20. **Blob Storage Architecture** - Container structure tree (ASCII)
21. **Cosmos DB Data Model** - Database/container/document schema (ASCII)
22. **Configuration Management** - Source hierarchy (ASCII)
23. **Monitoring & Telemetry** - Application Insights flow (SVG)

### Part 6: Security Architecture (2 diagrams)
24. **Authentication/Authorization Flow** - Azure AD sequence (SVG)
25. **RBAC Permission Model** - Permission matrix (ASCII)

---

## KEY FEATURES

**Technical Depth:**
- Every diagram references actual source code with file paths and line numbers
- Real configuration values from dev2 environment
- Actual queue names, container paths, field names from production code

**Visual Clarity:**
- ASCII diagrams for quick text-file viewing
- SVG diagrams for detailed architecture visualization
- Multi-page swimlanes for complex workflows
- Color-coded components by resource type

**Comprehensive Coverage:**
- Network layer (VNet, private endpoints, DNS)
- Application layer (React, FastAPI, Flask, Functions)
- Data layer (OpenAI, Search, Cosmos, Blob)
- Security layer (Azure AD, RBAC, private endpoints)
- Processing workflows (ingestion pipeline, query execution)

---

## IMPLEMENTATION ESTIMATE

- **Total Time:** 13-17 hours
- **Deliverable Size:** ~30,000 words + 25 diagrams
- **Document Filenames
dev2-TechDec-00-INDEX.md
dev2-TechDec-01-infra-arch.md
dev2-TechDec-02-application-arch.md
dev2-TechDec-03-doc-ingestion.md
dev2-TechDec-04-rag-query.md
dev2-TechDec-05-data-flows.md
dev2-TechDec-06-security-arch.md

- **Assets:** diagrams/ folder with 10 SVG files + source files

---

This will create the **most comprehensive visual reference of the EVA system** ever produced, combining real infrastructure topology with application architecture and validated against actual source code.

