# EVA Domain Assistant 1.2 Configuration Inventory

**Source Analysis**: C:\Users\marco.presta\OneDrive - ESDC EDSC\Documents\AICOE\EVA-JP-reference-0113  
**Analysis Date**: January 23, 2026  
**Total Configuration Items**: 53+ environment variables and settings

---

## LLM & AI Services Configuration

### Azure OpenAI Settings
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_OPENAI_ENDPOINT` | GPT model service endpoint | `app/backend/core/shared_constants.py`, approach classes | **Required** |
| `AZURE_OPENAI_CHATGPT_DEPLOYMENT` | Chat completion model deployment name | `approaches/chatreadretrieveread.py`, completion calls | **Required** |
| `AZURE_OPENAI_CHATGPT_MODEL_NAME` | Model identifier for token limits and capabilities | `core/modelhelper.py`, `get_token_limit()` | **Required** |
| `AZURE_OPENAI_CHATGPT_MODEL_VERSION` | Specific model version for consistency | Approach initialization, model tracking | Optional |
| `AZURE_OPENAI_EMBEDDING_DEPLOYMENT_NAME` | Embedding model deployment | `app/enrichment/app.py`, vector generation | Optional (if local embeddings used) |
| `AZURE_OPENAI_EMBEDDINGS_MODEL_NAME` | Embedding model identifier | Enrichment service model selection | Optional |
| `AZURE_OPENAI_API_VERSION` | API version for consistency | `openai.api_version` setting | **Required** |
| `AZURE_OPENAI_AUTHORITY_HOST` | Government/commercial cloud selection | Credential authority configuration | **Required** |

### Azure AI Services  
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_AI_ENDPOINT` | Cognitive Services endpoint for translation, content safety | `approaches/chatreadretrieveread.py`, query optimization | **Required** |
| `AZURE_AI_KEY` | Service authentication key | AI services authentication | Optional (if using managed identity) |
| `AZURE_AI_LOCATION` | Service region for compliance and latency | AI services configuration | **Required** |
| `AZURE_AI_CREDENTIAL_DOMAIN` | Token provider domain | Credential token scoping | **Required** |

---

## Retrieval & Vector Store Configuration

### Azure Cognitive Search
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_SEARCH_SERVICE` | Search service name | Search client initialization | **Required** |
| `AZURE_SEARCH_SERVICE_ENDPOINT` | Full search service URL | `SearchClient` initialization across approaches | **Required** |
| `AZURE_SEARCH_INDEX` | Primary search index name | Search queries, document indexing | **Required** |
| `AZURE_SEARCH_AUDIENCE` | Cloud audience (AzurePublicCloud/AzureUSGovernment) | Search service targeting | **Required** |
| `USE_SEMANTIC_RERANKER` | Enable enhanced relevance ranking | Search configuration, query enhancement | Optional (defaults to "true") |

### Embedding Configuration
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `TARGET_EMBEDDINGS_MODEL` | Embedding model identifier for filename escaping | `approaches/chatreadretrieveread.py`, model naming | **Required** |
| `EMBEDDING_VECTOR_SIZE` | Vector dimensions for index configuration | Enrichment service, index schema | **Required** |
| `USE_AZURE_OPENAI_EMBEDDINGS` | Toggle between Azure OpenAI vs local embeddings | Enrichment service model selection | Optional (defaults to "false") |
| `ENRICHMENT_ENDPOINT` | Enrichment service URL | Approach classes, embedding generation calls | **Required** |

### Search Behavior Settings
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `KB_FIELDS_CONTENT` | Content field name in search index | Search queries, content retrieval | **Required** |
| `KB_FIELDS_SOURCEFILE` | Source file field name | Citation generation, source linking | **Required** |
| `KB_FIELDS_PAGENUMBER` | Page number field name | Citation page references | **Required** |
| `KB_FIELDS_CHUNKFILE` | Chunk file field name | Document chunk tracking | **Required** |

---

## Storage Configuration

### Azure Blob Storage
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_BLOB_STORAGE_ACCOUNT` | Storage account name | Blob client initialization | **Required** |
| `AZURE_BLOB_STORAGE_ENDPOINT` | Full blob storage URL | `BlobServiceClient` creation | **Required** |
| `BLOB_STORAGE_ACCOUNT_ENDPOINT` | Alternative endpoint format | Functions blob client setup | **Required** |
| `AZURE_BLOB_CONTENT_CONTAINER` | Processed documents container | Document retrieval, content access | **Required** |
| `AZURE_BLOB_UPLOAD_CONTAINER` | Upload staging container | File upload processing | **Required** |

### Azure Cosmos DB
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_COSMOSDB_ENDPOINT` | Cosmos DB service URL | Database client initialization | **Required** |
| `COSMOSDB_URL` | Alternative Cosmos DB URL format | Functions database access | **Required** |
| `COSMOSDB_LOG_DATABASE_NAME` | Audit/status logging database | Status logging, audit trails | **Required** |
| `COSMOSDB_LOG_CONTAINER_NAME` | Log container within database | Log entry storage | **Required** |
| `COSMOSDB_CHATHISTORY_CONTAINER` | Chat session storage container | Session persistence | **Required** |

### Queue Storage
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_QUEUE_STORAGE_ENDPOINT` | Queue service URL | Queue client initialization | **Required** |
| `EMBEDDINGS_QUEUE` | Embedding processing queue name | Enrichment service, batch processing | **Required** |
| `TEXT_ENRICHMENT_QUEUE` | Text processing queue name | Document pipeline, enrichment trigger | **Required** |
| `PDF_SUBMIT_QUEUE` | PDF processing queue | PDF document routing | **Required** |
| `NON_PDF_SUBMIT_QUEUE` | Non-PDF document processing | Office document routing | **Required** |
| `PDF_POLLING_QUEUE` | PDF status polling queue | OCR status monitoring | **Required** |
| `MEDIA_SUBMIT_QUEUE` | Media file processing queue | Image/media document routing | **Required** |
| `IMAGE_ENRICHMENT_QUEUE` | Image processing queue | Image text extraction | **Required** |
| `AUTH_ENRICHMENT_QUEUE` | Authority document processing | Canadian regulation handling | Optional |
| `AUTHORITY_DOC_QUEUE` | Authority document queue | Government document classification | Optional |

---

## Authentication & Authorization

### Identity & Access Management  
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_SUBSCRIPTION_ID` | Azure subscription identifier | Resource management, deployment | **Required** |
| `AZURE_ARM_MANAGEMENT_API` | Azure Resource Manager API endpoint | Resource management operations | Optional (defaults to public cloud) |
| `LOCAL_DEBUG` | Development vs production authentication mode | Credential selection (DefaultAzureCredential vs ManagedIdentity) | **Required** |

### RBAC Configuration
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `APPLICATION_TITLE` | Application display name | Frontend branding, logging context | Optional (defaults to "EVA Domain Assistant") |
| `CHAT_WARNING_BANNER_TEXT` | User notification message | Frontend warning display | Optional |

---

## Telemetry & Logging

### Application Insights & Monitoring
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `LOG_LEVEL` | Logging verbosity control | Logger configuration across all modules | Optional (defaults to "INFO") |
| `APP_LOGGER_NAME` | Application logger identifier | Logger naming, correlation | Optional |

### Performance & Processing Limits
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `DEQUEUE_MESSAGE_BATCH_SIZE` | Queue processing batch size | Enrichment service queue handling | Optional (defaults to 1) |
| `MAX_CHUNKS_PER_BATCH` | Embedding batch size limit | Enrichment service batch processing | Optional (defaults to 120) |
| `MAX_PAYLOAD_BYTES` | Request payload size limit | Enrichment service, API limits | Optional (defaults to 12MB) |
| `MAX_EMBEDDING_REQUEUE_COUNT` | Embedding retry attempts | Error handling, resilience | Optional (defaults to 3) |
| `EMBEDDING_REQUEUE_BACKOFF` | Embedding retry delay seconds | Backoff strategy | Optional (defaults to 60) |
| `MAX_ENRICHMENT_REQUEUE_COUNT` | Text enrichment retry attempts | Document processing resilience | Optional (defaults to 3) |
| `ENRICHMENT_BACKOFF` | Text enrichment retry delay | Processing backoff strategy | Optional (defaults to 60) |

---

## Feature Flags & Behavior Controls

### Language & Localization
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `TARGET_TRANSLATION_LANGUAGE` | Primary processing language | Translation service, language detection | **Required** |
| `QUERY_TERM_LANGUAGE` | Search query language context | Query processing, search configuration | **Required** |

### Content Processing
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `EN_FR_DOC_PROCESSING` | Bilingual document processing flag | Document pipeline language handling | Optional |
| `MAX_SECONDS_HIDE_ON_UPLOAD` | Upload processing timeout | File upload status management | Optional |

### Performance Tuning
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `UTF8` | Text encoding specification | Enrichment service text processing | Optional (defaults to "utf-8") |
| `MAX_CHARS_FOR_DETECTION` | Language detection sample size | Text analysis, language identification | Optional (defaults to 4000) |

---

## Rate Limiting & Safety Configuration

### Content Safety (Inferred from Code Patterns)
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `CONTENT_FILTER_ENABLED` | Azure OpenAI content filtering | Chat completion error handling | Optional |
| `SAFETY_CLASSIFICATION_THRESHOLD` | Content safety scoring limits | Response filtering | Optional |

### API Rate Limiting (Azure Service Limits)
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `OPENAI_REQUEST_TIMEOUT` | OpenAI API request timeout | Chat completion calls | Optional |
| `SEARCH_REQUEST_TIMEOUT` | Search service timeout | Search query limits | Optional |
| `BLOB_OPERATION_TIMEOUT` | Blob storage operation timeout | Document access limits | Optional |

---

## Environment-Specific Configuration

### Development vs Production
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `AZURE_OPENAI_AUTHORITY_HOST` | Cloud authority (AzureCloud vs AzureUSGovernment) | Credential configuration | **Required** |
| `LOCAL_DEBUG` | Local development mode toggle | Authentication method selection | **Required** |

### Network & Security
| Config Key | Purpose | Where Used | Required/Optional |
|------------|---------|------------|-------------------|
| `ALLOWED_ORIGINS` | CORS allowed origins for API access | FastAPI CORS configuration | **Required** |
| `PRIVATE_ENDPOINT_ENABLED` | Private network access mode | Service endpoint configuration | Optional |

---

## Configuration Management Analysis

### Configuration Loading Patterns
**Primary Loading**: `dotenv.load_dotenv()` from multiple sources:
- `app/backend/backend.env` - Backend-specific settings
- `scripts/environments/.env` - Terraform deployment variables  
- `functions/local.settings.json` - Azure Functions runtime config

### Required vs Optional Breakdown
- **Critical Required Settings**: 31 configurations (must be set or system fails)
- **Optional with Defaults**: 15 configurations (have fallback values)
- **Feature Flags**: 7 configurations (enable/disable functionality)

### Security Considerations  
- **Secrets Management**: No hardcoded secrets found; uses Azure Key Vault or environment variables
- **Credential Patterns**: Consistent use of Azure Managed Identity in production, DefaultAzureCredential in development
- **Government Cloud Support**: Proper authority host configuration for Azure Government compliance

### JP Plan-2 Configuration Gaps
**Missing JP-Specific Settings**:
- Court system API endpoints and credentials
- Legal taxonomy service configuration  
- Case metadata validation rules
- JP case ID format specifications
- Legal document classification thresholds
- Attorney-client privilege detection settings
- Legal audit trail requirements
- Case sensitivity classification levels

**Estimated Additional Config Items for JP**: ~25-30 additional settings required for full JP Plan-2 implementation.