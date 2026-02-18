# EVA Domain Assistant 1.2 Reference vs EVA/JP Plan-2 Mapping

**Analysis Date**: January 23, 2026  
**Reference Implementation**: C:\Users\marco.presta\OneDrive - ESDC EDSC\Documents\AICOE\EVA-JP-reference-0113  
**Target**: EVA/Jurisprudence Plan-2 with case-aware locked metadata fields

---

## Architecture Mapping Table

| Capability / Concern | EVA DA (as implemented) | EVA DA / JP Plan-2 Target | Delta / Work Needed | Evidence (paths, symbols) |
|---------------------|------------------------|---------------------------|-------------------|-------------------------|
| **Runtime Stack** | FastAPI backend + React SPA + Flask enrichment service + 12 Azure Functions | Same multi-service pattern but optimize for JP case processing workflows | **Medium** - Add JP-specific case processing functions, legal document classification | `app/backend/app.py`, `app/frontend/src/`, `app/enrichment/app.py`, `functions/` directory |
| **LLM Provider Abstraction** | Direct Azure OpenAI integration with AsyncAzureOpenAI client | Maintain Azure OpenAI with potential for other providers | **Small** - Abstract LLM calls behind interface for provider flexibility | `chatreadretrieveread.py` line 40: `self.client = AsyncAzureOpenAI()`, `core/shared_constants.py` |
| **Prompt Orchestration + Citation Enforcement** | Template-based prompts with mandatory [File1], [File2] citation format | Enhanced legal citation format supporting case law, statute, and regulation citations | **Medium** - Legal citation parser, case number formatting, court hierarchy recognition | `SYSTEM_MESSAGE_CHAT_CONVERSATION` template, `FOLLOW_UP_QUESTIONS_PROMPT_CONTENT` |
| **Retrieval: Vector Store/Search Integration** | Azure Cognitive Search with hybrid vector + keyword search | Same hybrid approach optimized for legal terminology and case precedent matching | **Small** - Legal-specific search configurations, precedent ranking | `approaches/chatreadretrieveread.py` lines 200+: `VectorizedQuery`, `azure_search/create_vector_index.json` |
| **Chunking Strategy** | Content-aware chunking with configurable boundaries, page number preservation | JP Plan-2 requires case-aware chunking preserving legal structure (holdings, rationale, decision sections) | **Large** - Legal document structure detection, section-aware chunking, holding sentence extraction | `functions/TextEnrichment/__init__.py`, chunking logic needs legal document awareness |
| **Embeddings Model Config + Batching** | Azure OpenAI text-embedding-ada-002 + local SentenceTransformers, batch size 120, 12MB payload limit | Same embedding models with legal domain fine-tuning consideration | **Small** - Legal domain embedding evaluation, potentially custom legal embeddings | `app/enrichment/app.py` lines 50+: `MAX_CHUNKS_PER_BATCH=120`, `TARGET_EMBEDDINGS_MODEL` |
| **Reranking** | Optional Azure Cognitive Search semantic reranking | Enhanced legal relevance reranking considering case importance, precedent weight | **Medium** - Legal authority ranking, citation count weighting, court hierarchy scoring | `USE_SEMANTIC_RERANKER` flag, semantic ranking configuration |
| **Ingestion Pipeline** | Event-driven Azure Functions with format detection, OCR, language processing | JP-specific ingestion supporting case metadata extraction, court document parsing | **Large** - JP case metadata extraction, court system integration, automated case classification | `functions/FileUploadedEtrigger/__init__.py`, needs JP case processing |
| **Metadata Schema Handling** | Generic document fields (file_name, content, tags, folder, processed_datetime) | **JP Plan-2 case-aware locked fields**: parent_id, id, pdf_link, web_link, case_id, docket_number, court, court_level, decision_date, decision_makers, style_of_cause, language, page_count, legal_domain, ei_topics[], intent[], outcome[], negative_flags[], statutes[], sections[], holding_sentences[] | **Large** - Complete schema redesign, case metadata parsers, legal field validation, structured legal taxonomy | `azure_search/create_vector_index.json` - current schema vs JP Plan-2 requirements |
| **Document Storage** | Azure Blob Storage with container-based RBAC | Same blob storage with JP case file organization and legal document retention policies | **Medium** - JP case folder structure, legal retention policies, evidence chain preservation | `AZURE_BLOB_STORAGE_ACCOUNT` config, blob storage containers |
| **Index Schema / Filters / Faceting** | Basic filtering by folder, tags, file_class, processed_datetime | JP Plan-2 legal faceting: court_level, legal_domain, decision_date ranges, ei_topics[], statutes[], outcome[] | **Large** - Legal taxonomy facets, court hierarchy filters, case law precedent indexing | `create_vector_index.json` fields need JP legal taxonomy |
| **Language Handling** | EN/FR detection, translation, dual-field indexing (title/translated_title) | Enhanced legal bilingual processing with legal term preservation, French case law handling | **Medium** - Legal terminology dictionaries, French Canadian legal language models | `detect_language()`, `translate_response()` methods, `TARGET_TRANSLATION_LANGUAGE` |
| **Evaluation Harness** | Functional test suite with format-specific queries, 45-minute pipeline validation | JP golden questions dataset, legal accuracy evaluation, precedent retrieval testing | **Large** - JP case law golden dataset, legal accuracy metrics, expert evaluation workflow | `tests/run_tests.py`, `search_queries` dictionary - needs JP legal test cases |
| **Admin UI** | Basic React frontend with settings panels | JP Plan-2 admin interface for case metadata management, legal taxonomy administration | **Large** - JP case management UI, metadata editor, legal taxonomy browser, case relationship visualization | `app/frontend/src/pages/chat/Chat.tsx` - needs JP admin interface |
| **AuthN/AuthZ** | Azure AD integration with group-based RBAC, container-level permissions | Enhanced legal access controls, case sensitivity levels, attorney-client privilege handling | **Medium** - Legal access classifications, case sensitivity controls, audit trail enhancements | `shared_code/utility_rbck.py`, `UserProfile` class - needs legal access controls |
| **Logging/Telemetry/Audit Trail** | Cosmos DB audit logging, OpenTelemetry tracing, status tracking | Enhanced legal audit requirements, case access logging, privilege protection | **Medium** - Legal compliance logging, access audit trails, privilege boundary enforcement | `shared_code/status_log.py`, `StatusLog` class - needs legal audit enhancements |
| **Deployment/IaC** | Terraform with Azure Government cloud support, private network deployment | Same Terraform pattern with JP-specific configurations, legal compliance requirements | **Small** - JP environment-specific configurations, legal compliance validation | `infra/main.tf`, `scripts/` deployment - needs JP config customization |
| **Security Posture** | Content filtering, rate limiting, private endpoints, RBAC | Enhanced legal content protection, case sensitivity handling, privilege preservation | **Medium** - Legal content classification, attorney-client privilege detection, case sensitivity controls | `azure_ai_endpoint` content safety, rate limiting configs - needs legal protections |
| **Observability** | Azure Application Insights, rich console logging, performance tracking | Legal performance monitoring, case processing metrics, precedent matching analytics | **Small** - JP-specific dashboards, legal workflow monitoring, case processing KPIs | OpenTelemetry integration, monitoring configs - needs JP legal metrics |

---

## JP Plan-2 Locked Metadata Fields Analysis

### Core Document Identity
- `parent_id`, `id` - **Missing** - Need hierarchical case relationships
- `pdf_link`, `web_link` - **Partial** - Current `file_uri` field, needs dual-link support
- `metadata_storage_path`, `metadata_storage_content_type`, `metadata_storage_content_md5`, `metadata_storage_last_modified`, `metadata_storage_size` - **Present** - Available in blob properties

### JP Case-Specific Fields (All Missing - Require Implementation)
- `case_id`, `docket_number` - Case identification system
- `court`, `court_level` - Court hierarchy and jurisdiction
- `decision_date`, `decision_makers` - Temporal and authority metadata  
- `style_of_cause` - Case title formatting
- `page_count` - Document pagination (partially exists as `pages` array)
- `legal_domain` - Legal area classification
- `ei_topics[]`, `intent[]`, `outcome[]` - Structured legal taxonomy arrays
- `negative_flags[]` - Legal warning/limitation indicators
- `statutes[]`, `sections[]` - Legislative reference arrays
- `holding_sentences[]` - Key legal precedent extraction

### Implementation Gap Assessment
- **Current Schema Coverage**: ~15% of JP Plan-2 requirements
- **Legal Taxonomy Integration**: Not implemented
- **Case Relationship Modeling**: Not present
- **Court System Integration**: Not configured
- **Legal Document Structure Recognition**: Basic content chunking only

---

## Top 10 Engineering Tasks

### 1. **JP Legal Metadata Schema Implementation** - **Size: L**
**Rationale**: Complete redesign of search index schema to support JP Plan-2 locked fields, including legal taxonomy, case relationships, and court hierarchy.
**Dependencies**: Legal taxonomy definition, court system integration requirements

### 2. **Case-Aware Document Processing Pipeline** - **Size: L** 
**Rationale**: Rebuild ingestion pipeline to extract case metadata, recognize legal document structure, parse court documents, and populate JP-specific fields.
**Dependencies**: Legal document parsing libraries, court system APIs, case ID validation

### 3. **Legal Document Chunking Strategy** - **Size: L**
**Rationale**: Replace generic chunking with legal structure-aware chunking that preserves holdings, rationale, and decision sections for accurate precedent matching.
**Dependencies**: Legal document structure analysis, holding sentence extraction algorithms

### 4. **JP Case Management Admin Interface** - **Size: L**
**Rationale**: Build comprehensive admin UI for case metadata management, legal taxonomy administration, and case relationship visualization.
**Dependencies**: JP metadata schema completion, legal workflow requirements

### 5. **Legal Citation Enhancement** - **Size: M**
**Rationale**: Upgrade citation system to handle case law citations, statute references, regulation citations with proper legal formatting and validation.
**Dependencies**: Legal citation standards, court citation formats, statutory reference validation

### 6. **JP Golden Questions Dataset & Evaluation** - **Size: L**
**Rationale**: Create comprehensive JP case law test dataset with golden questions covering precedent matching, legal reasoning, and case outcome prediction.
**Dependencies**: Legal expert input, case law corpus, evaluation criteria definition

### 7. **Legal Taxonomy Integration** - **Size: M**
**Rationale**: Implement ei_topics[], legal_domain, statutes[], sections[] taxonomy with proper classification and faceted search.
**Dependencies**: Legal taxonomy standards, classification algorithms, taxonomy maintenance workflow

### 8. **Enhanced Legal Access Controls** - **Size: M**
**Rationale**: Implement case sensitivity levels, attorney-client privilege protection, and enhanced audit trails for legal compliance.
**Dependencies**: Legal access requirements, privilege detection algorithms, compliance audit standards

### 9. **Court System Integration & Validation** - **Size: M**
**Rationale**: Build integrations for court system APIs, case ID validation, docket number verification, and automated case status updates.
**Dependencies**: Court system API access, case validation services, legal data source agreements

### 10. **Legal Performance Optimization** - **Size: S**
**Rationale**: Optimize search and retrieval for legal queries including precedent matching, authority ranking, and court hierarchy weighting.
**Dependencies**: Legal search requirements, precedent ranking algorithms, court authority weightings

---

## Implementation Priority Recommendations

### Phase 1: Foundation (Tasks 1, 2, 3)
Focus on core infrastructure changes - metadata schema, document processing, and legal chunking. These are foundational and block other features.

### Phase 2: Legal Features (Tasks 5, 7, 9)  
Implement legal-specific functionality - citations, taxonomy, and court system integration. Builds on foundation to provide JP-specific capabilities.

### Phase 3: User Experience (Tasks 4, 6, 8)
Add administrative interfaces, evaluation harnesses, and enhanced security. Requires stable foundation and legal features.

### Phase 4: Optimization (Task 10)
Performance tuning and optimization once core functionality is stable and validated.

**Total Estimated Effort**: ~8-12 months with dedicated team of 4-6 engineers plus legal domain experts.