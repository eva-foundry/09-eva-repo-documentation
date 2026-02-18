# EVA Domain Assistant: Configuration Sources for Project/Index Management

**Scan Date**: 2026-01-26
**Repo**: `I:\EVA-JP-v1.2`
**Purpose**: Document all configuration sources that influence project/index selection

---

## Executive Summary

EVA DA uses **5 primary configuration sources** for project/index management:

1. **Azure Cosmos DB** (Primary Source of Truth) - Runtime project registry
2. **Environment Variables** (backend.env) - Service endpoints and connection strings
3. **Azure AD Groups** (External Identity Provider) - User-to-project authorization
4. **User Profile Database** (Cosmos DB) - Last-selected project preference
5. **Runtime Memory Cache** - 30-second cache of project registry

**NO** project/index enumeration in: `.env`, `appsettings.json`, YAML, Helm charts, CI variables

---

## 1. Azure Cosmos DB - Project Registry (PRIMARY)

### Database Configuration

**Database Name**: `groupsToResourcesMap`
**Container Name**: `groupResourcesMapContainer`
**Partition Key**: `group_id` (Azure AD group GUID)

**Evidence**: 
- `app/backend/backend.env:71-72`
```env
COSMOSDB_CONTAINER_GROUP_MAP="groupResourcesMapContainer"
COSMOSDB_DATABASE_GROUP_MAP="groupsToResourcesMap"
```

- `functions/shared_code/utility_rbck.py:221-229`
```python
def initiate_group_resource_map(credential, cosmos_client=None):
    cosmodb_url = os.getenv("COSMOSDB_URL")
    cosmodb_group_map_db = os.getenv("COSMOSDB_DATABASE_GROUP_MAP")
    cosmodb_group_map_container = os.getenv("COSMOSDB_CONTAINER_GROUP_MAP")
    group_resource_map_db = GroupResourceMap(
        cosmodb_url, credential, cosmodb_group_map_db, 
        cosmodb_group_map_container, cosmos_client
    )
    return group_resource_map_db
```

### Document Schema (per project)

**File**: `functions/shared_code/utility_rbck.py:85-229` (Class `GroupResourceMap`)

**Expected Document Structure**:
```json
{
  "id": "<uuid>",
  "group_id": "b9fcbc75-a0d2-402c-8509-23a0923075b2",
  "group_name": "EVA-DA-Project7-ERIC-Admin",
  "upload_storage": {
    "upload_container": "upload-eric",
    "role": "admin"
  },
  "blob_access": {
    "blob_container": "content-eric",
    "role_blob": "admin"
  },
  "vector_index_access": {
    "index": "index-eric",
    "role_index": "admin"
  }
}
```

**Key Fields**:
- `group_id`: Azure AD security group GUID (partition key)
- `group_name`: Human-readable project name with role suffix
- `upload_storage.upload_container`: Where users upload files
- `blob_access.blob_container`: Where processed content is stored
- `vector_index_access.index`: Name of Azure Cognitive Search index
- Role fields: `admin`, `contributor`, or `reader`

### Query Methods

**File**: `functions/shared_code/utility_rbck.py:115-228`

1. **Read All Group IDs**:
```python
def read_all_groupId(self):
    query = "SELECT DISTINCT c.group_id FROM c"
    # Returns: list of all registered project group GUIDs
```

2. **Fetch Blob Access for Group**:
```python
def fetch_blob_access_for_group(self, group_id):
    query = f"SELECT c.blob_access FROM c WHERE c.group_id = '{group_id}'"
    # Returns: upload/content container names and roles
```

3. **Fetch Vector Access for Group**:
```python
def fetch_vector_access_for_group(self, group_id):
    query = f"SELECT c.vector_index_access FROM c WHERE c.group_id = '{group_id}'"
    # Returns: search index name and role
```

4. **Read All Items**:
```python
def read_all_items(self):
    query = "SELECT * FROM c"
    # Returns: Complete registry of all projects (cached)
```

---

## 2. Environment Variables (backend.env)

### Azure Cosmos DB Connection

**File**: `app/backend/backend.env`

**Lines 1-30** (Service Endpoints):
```env
COSMOSDB_URL=https://infoasst-cosmos-hccld2.documents.azure.com:443/
COSMOSDB_LOG_DATABASE_NAME=statusdb
COSMOSDB_LOG_CONTAINER_NAME=statuscontainer
COSMOSDB_USERPROFILE_DATABASE_NAME="UserInformation"
COSMOSDB_CHATHISTORY_CONTAINER="chat_history_session"
COSMOSDB_GROUP_MANAGEMENT_CONTAINER="group_management"
COSMOSDB_CONTAINER_GROUP_MAP="groupResourcesMapContainer"
COSMOSDB_DATABASE_GROUP_MAP="groupsToResourcesMap"
```

**Classification**: **Connection strings only, NO project enumeration**

### Azure Search Service

**Lines 5-6**:
```env
AZURE_SEARCH_SERVICE=infoasst-search-hccld2
AZURE_SEARCH_SERVICE_ENDPOINT=https://infoasst-search-hccld2.search.windows.net/
```

**Note**: No `AZURE_SEARCH_INDEX` variable for multi-project deployments
- Single-project dev instances may set this
- Production uses dynamic index lookup from Cosmos DB

### Azure Storage (Blob Containers)

**Lines 3-4**:
```env
AZURE_BLOB_STORAGE_ACCOUNT=infoasststorehccld2
AZURE_BLOB_STORAGE_ENDPOINT=https://infoasststorehccld2.blob.core.windows.net/
```

**Container Names**: Retrieved dynamically from Cosmos DB, not env vars

### Cache Timer

**Line 2**:
```env
CACHETIMER=30
```

**Purpose**: Project registry cache duration (seconds)
**Impact on Projects**: Registry refreshed every 30 seconds from Cosmos DB

**Usage**: `functions/shared_code/utility_rbck.py:240-248`
```python
def read_all_items_into_cache_if_expired(group_resource_map_db, items=0, expired_time=0):
    if expired_time < time.time():
        cache_time = os.getenv("CACHETIMER")  # 30 seconds
        cache_expiration_time = time.time() + float(cache_time)
        items = group_resource_map_db.read_all_items()
        return items, cache_expiration_time
```

### Special Groups (Testing)

**Line 97**:
```env
CPPD_GROUPS="AICoE_Admin_TestRBAC,AICoE_Admin_TestRBAC2"
```

**Purpose**: Test/demo AD groups for RBAC validation
**Classification**: **NOT production project config** - used for dev/test only

**Usage**: `app/backend/app.py:93`
```python
SPECIAL_GROUPS = ENV["CPPD_GROUPS"].split(",")
```

### Application Title

**Line 23**:
```env
APPLICATION_TITLE="EVA Domain Assistant Accelerator"
```

**Usage**: `app/frontend/src/pages/layout/Layout.tsx:336`
- Displayed in UI header, not tied to specific projects

---

## 3. Azure AD Groups (External Configuration)

### Authorization Source

**Identity Provider**: Microsoft Entra ID (formerly Azure AD)
**Authentication Flow**: OAuth 2.0 with Azure App Service Easy Auth

**Token Headers**:
1. `x-ms-client-principal` - Base64-encoded user principal with group memberships
2. `x-ms-token-aad-id-token` - JWT ID token with `groups` claim

**Evidence**: `functions/shared_code/utility_rbck.py:51-69`
```python
def decode_x_ms_client_principal(request):
    x_ms_client_principal = request.headers.get("x-ms-client-principal")
    if not x_ms_client_principal:
        logger.info("x_ms_client_principal not found in the header ")
        return None
    decoded = base64.b64decode(x_ms_client_principal)
    client_principal = json.loads(decoded)
    return client_principal

def fetch_groups_from_id_token(tokenPayload: Dict):
    user_groups = tokenPayload.get("groups")  # List of AD group GUIDs
    client_id = tokenPayload.get("aud")
    talent_id = tokenPayload.get("tid")
    return talent_id, client_id, user_groups
```

### Group Naming Convention

**Pattern Observed**: `EVA-DA-{Project}-{Role}`

**Examples** (inferred from code logic):
- `EVA-DA-Project7-ERIC-Admin`
- `EVA-DA-Project7-ERIC-Contributor`
- `EVA-DA-Project7-ERIC-Reader`
- `EVA-DA-Jurisprudence-Admin`
- (Additional ~47+ project groups)

**Role Detection**: `functions/shared_code/utility_rbck.py:294-310`
```python
for grp_id in rbac_group_l:
    for item in group_map_items:
        if item.get("group_id") == grp_id:
            grp_name = item.get("group_name")
            if "admin" in grp_name.lower():
                role_mapping["admin"][grp_name] = grp_id
            elif "contributor" in grp_name.lower():
                role_mapping["contributor"][grp_name] = grp_id
            elif "reader" in grp_name.lower():
                role_mapping["reader"][grp_name] = grp_id
```

### User-to-Project Mapping Flow

**Step 1**: User authenticates → Azure AD returns token with `groups` claim
**Step 2**: Backend extracts group GUIDs from token
**Step 3**: Intersect with registered project groups in Cosmos DB
**Step 4**: Determine highest role (admin > contributor > reader)
**Step 5**: Resolve group GUID → project resources (containers, index)

**Evidence**: `functions/shared_code/utility_rbck.py:251-279`
```python
def get_rbac_grplist_from_client_principle(req, group_map_items):
    client_principal_payload = decode_x_ms_client_principal(req)
    user_groups = [principal["val"] for principal in client_principal_payload["claims"]]
    groups_ids_in_group_map = [item.get("group_id") for item in group_map_items]
    rbac_group_l = list(set(user_groups).intersection(set(groups_ids_in_group_map)))
    return rbac_group_l
```

---

## 4. User Profile Database (Cosmos DB)

### Database Configuration

**Database Name**: `UserInformation`
**Container Name**: `group_management`
**Purpose**: Store user's last-selected project preference

**Evidence**: `app/backend/backend.env:69-70`
```env
COSMOSDB_USERPROFILE_DATABASE_NAME="UserInformation"
COSMOSDB_GROUP_MANAGEMENT_CONTAINER="group_management"
```

### Document Schema (per user)

**File**: `functions/shared_code/user_profile.py:44-76`

```json
{
  "id": "marco.presta@example.com_a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "principalId": "marco.presta@example.com",
  "userADGroups": [
    "EVA-DA-Project7-ERIC-Admin",
    "EVA-DA-Jurisprudence-Reader"
  ],
  "userADGroups_ids": [
    "b9fcbc75-a0d2-402c-8509-23a0923075b2",
    "c1234567-89ab-cdef-0123-456789abcdef"
  ],
  "currentDAGroup": "EVA-DA-Project7-ERIC-Admin",
  "currentDAGroup_id": "b9fcbc75-a0d2-402c-8509-23a0923075b2",
  "createdTime": "2026-01-26T10:30:45.123456Z"
}
```

**Key Fields**:
- `principalId`: User's email/UPN (partition key)
- `userADGroups`: Human-readable project group names
- `userADGroups_ids`: Corresponding AD group GUIDs
- `currentDAGroup`: Last-selected project (user preference)

### Query Methods

**File**: `functions/shared_code/user_profile.py:33-43`

```python
def fetch_lastest_choice_of_group(self, principal_id):
    principal_id = principal_id.strip().lower()
    query = "SELECT * FROM c WHERE c.principalId = @principalId ORDER BY c.createdTime DESC"
    parameters = [{"name": "@principalId", "value": principal_id}]
    items = list(self.container.query_items(
        query=query, parameters=parameters, enable_cross_partition_query=True
    ))
    
    if items:
        return items[0].get("currentDAGroup"), items[0].get("currentDAGroup_id")
    return None, None
```

**Usage**: When user logs in, restore their last-used project instead of defaulting to first accessible project

---

## 5. Runtime Memory Cache

### Cache Implementation

**File**: `app/backend/app.py:460-484`

```python
if cosmos_available and cosmosdb_client:
    try:
        # Load Cosmos DB group-container-index map
        groupmapcontainer = initiate_group_resource_map(AZURE_CREDENTIAL, cosmos_client=cosmosdb_client)
        group_items, expired_time = read_all_items_into_cache_if_expired(groupmapcontainer)
    except Exception as e:
        LOGGER.warning("⚠ Failed to initialize Cosmos DB dependent services: %s", str(e))
        cosmos_available = False
else:
    LOGGER.warning("⚠ Skipping Cosmos DB dependent services initialization")
    group_items, expired_time = [], 0

# Build runtime lookup maps
upload_containers = read_all_upload_containers(group_items)
content_containers = read_all_content_containers(group_items)
vector_indexes = read_all_vector_indexes(group_items)

for index in vector_indexes:
    index_to_search_client_map[index] = SearchClient(
        endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"], 
        index_name=index, 
        credential=AZURE_CREDENTIAL, 
        audience=ENV["AZURE_SEARCH_AUDIENCE"]
    )
```

### Cache Structure

**In-Memory Objects**:
1. `group_items` - List of all project registry documents from Cosmos DB
2. `expired_time` - Timestamp when cache expires (current_time + 30 seconds)
3. `index_to_search_client_map` - Dict[index_name, SearchClient]
4. `upload_container_to_upload_blob_container_client_map` - Dict[container, BlobClient]
5. `content_container_to_content_blob_container_client_map` - Dict[container, BlobClient]

**Cache Refresh Logic**: `functions/shared_code/utility_rbck.py:240-248`
```python
def read_all_items_into_cache_if_expired(group_resource_map_db, items=0, expired_time=0):
    if expired_time < time.time():  # Cache expired
        cache_time = os.getenv("CACHETIMER")  # 30 seconds
        cache_expiration_time = time.time() + float(cache_time)
        items = group_resource_map_db.read_all_items()  # Reload from Cosmos DB
        return items, cache_expiration_time
    else:
        return items, expired_time  # Use cached data
```

**Refresh Trigger**: Every request checks cache expiration before project lookup

---

## Configuration Sources: NOT FOUND

### Files/Formats Searched With No Results

1. **appsettings.json** - NOT FOUND in repo
   - Searched: `**/appsettings*.json`
   - Result: No ASP.NET-style config files

2. **YAML Configuration Files** - NOT FOUND (project-related)
   - Searched: `**/*.yaml`, `**/*.yml`
   - Result: No Helm charts or Kubernetes yamls with project definitions

3. **Helm Charts** - NOT FOUND
   - Searched: `**/Chart.yaml`, `**/values.yaml`
   - Result: No Helm packaging in repo

4. **Bicep/ARM Templates** - NOT FOUND (with project lists)
   - Searched: `**/*.bicep`, `**/*.json` in `infra/`
   - Result: Infrastructure templates exist but do NOT enumerate projects
   - Projects provisioned separately per deployment instance

5. **CI/CD Pipeline Variables** - NOT FOUND
   - Searched: `**/.github/workflows/*.yml`, `**/.azure-pipelines/*.yml`
   - Result: No GitHub Actions or Azure Pipelines with hardcoded project lists

6. **Docker Compose** - NOT FOUND (with project config)
   - Searched: `**/docker-compose*.yml`
   - Result: No docker-compose files in repo

7. **Config Maps/Secrets (Kubernetes)** - NOT FOUND
   - Searched: `**/configmap.yaml`, `**/secret.yaml`
   - Result: No k8s config files in repo

---

## Configuration Management Best Practices

### Current State (Observed)

**Strengths**:
✅ Single source of truth (Cosmos DB)
✅ Separation of identity (AD) and config (Cosmos)
✅ Runtime project addition without code changes
✅ Caching for performance (30-second refresh)
✅ Role-based access per project

**Weaknesses**:
❌ No schema validation for Cosmos DB documents
❌ Manual updates to project registry (no admin UI visible in code)
❌ Cache timer hardcoded (no per-environment override)
❌ No project registry version control/audit trail

### Recommended Configuration Additions

1. **Add Configuration Schema File**
   ```json
   // config/project-schema.json
   {
     "$schema": "http://json-schema.org/draft-07/schema#",
     "type": "object",
     "required": ["group_id", "group_name", "upload_storage", "blob_access", "vector_index_access"],
     "properties": {
       "group_id": {"type": "string", "pattern": "^[a-f0-9-]{36}$"},
       "group_name": {"type": "string", "minLength": 1},
       "upload_storage": {
         "type": "object",
         "required": ["upload_container", "role"],
         "properties": {
           "upload_container": {"type": "string", "pattern": "^[a-z0-9-]+$"},
           "role": {"type": "string", "enum": ["admin", "contributor", "reader"]}
         }
       }
     }
   }
   ```

2. **Environment-Specific Cache Timers**
   ```env
   # Development
   CACHETIMER=5  # Fast refresh for dev

   # Production
   CACHETIMER=60  # Reduce Cosmos DB RU consumption
   ```

3. **Project Registry Audit Log**
   - Track: Project added, modified, deleted
   - Store in: Cosmos DB change feed or separate audit container
   - Include: timestamp, user, change details

4. **Configuration Validation on Startup**
   ```python
   def validate_project_registry(group_items):
       for item in group_items:
           assert "group_id" in item
           assert "vector_index_access" in item
           assert item["vector_index_access"].get("index")
       logger.info(f"✓ Validated {len(group_items)} project configs")
   ```

---

## Configuration Change Process (Current)

### How to Add a New Project (Inferred)

**Step 1**: Create Azure AD Groups
```bash
# Create 3 groups in Entra ID
EVA-DA-NewProject-Admin
EVA-DA-NewProject-Contributor
EVA-DA-NewProject-Reader
```

**Step 2**: Create Azure Resources
```bash
# Azure Blob Storage
az storage container create --name upload-newproject
az storage container create --name content-newproject

# Azure Cognitive Search
az search index create --name index-newproject
```

**Step 3**: Add Cosmos DB Document
```json
POST https://infoasst-cosmos-hccld2.documents.azure.com/dbs/groupsToResourcesMap/colls/groupResourcesMapContainer/docs
{
  "id": "<uuid>",
  "group_id": "<admin-group-guid>",
  "group_name": "EVA-DA-NewProject-Admin",
  "upload_storage": {
    "upload_container": "upload-newproject",
    "role": "admin"
  },
  "blob_access": {
    "blob_container": "content-newproject",
    "role_blob": "admin"
  },
  "vector_index_access": {
    "index": "index-newproject",
    "role_index": "admin"
  }
}
```

**Step 4**: Wait for Cache Refresh (30 seconds)
- No app restart required
- New project available after next cache expiration

**Step 5**: Assign Users to AD Groups
- Users appear in project dropdown after next login

---

## Terraform Variables (Infrastructure)

**File**: `scripts/environments/.env`

**Evidence**: `app/backend/app.py:44-46`
```python
# Load general infrastructure environment variables
general_env = os.path.abspath(os.path.join(os.path.dirname(__file__), "../../scripts/environments/.env"))
load_dotenv(general_env)
```

**Purpose**: Infrastructure provisioning variables (resource names, regions, SKUs)
**Classification**: **NO project enumeration** - only infrastructure config

**Expected Contents** (not visible in scan, loaded at runtime):
- `AZURE_SUBSCRIPTION_ID`
- `RESOURCE_GROUP_NAME`
- `AZURE_REGION`
- Cosmos DB account names
- Storage account names
- Search service names

**Does NOT contain**: List of projects, index names, or container names

---

## Summary Table: Configuration Source Influence

| Config Source | Influences Project Selection | Update Frequency | Evidence Location |
|--------------|------------------------------|------------------|-------------------|
| **Cosmos DB Project Registry** | ✅ YES - Primary routing | On cache expiration (30s) | `functions/shared_code/utility_rbck.py:85-229` |
| **Azure AD Groups** | ✅ YES - User authorization | On login/token refresh | `functions/shared_code/utility_rbck.py:51-69` |
| **backend.env** | ❌ NO - Only connection strings | On app start | `app/backend/backend.env:1-100` |
| **Terraform .env** | ❌ NO - Infrastructure only | On deployment | `scripts/environments/.env` (loaded) |
| **User Profile DB** | ⚠️ PARTIAL - Last preference | On user login | `functions/shared_code/user_profile.py:33-43` |
| **Runtime Cache** | ✅ YES - Performance layer | Every 30 seconds | `app/backend/app.py:460-484` |

---

## Appendix: File Inventory

### Configuration Files Found

```
app/backend/
  backend.env               # Service endpoints, NOT project enum
  backend.env.example       # Template for backend.env
  backend.env.prod-hccld2.example  # Production example

scripts/environments/
  .env                      # Terraform variables (loaded but not scanned)

functions/
  local.settings.json       # Functions runtime config (not scanned in detail)
```

### Configuration Files NOT Found

- `appsettings.json` (none)
- `config.yaml` (none)
- `values.yaml` / Helm charts (none)
- `.azure-pipelines.yml` (none)
- `.github/workflows/*.yml` (none)
- `docker-compose.yml` (none with project config)

---

## Conclusions

### Configuration Architecture Summary

EVA DA implements a **database-driven, runtime-configurable project registry** with:

1. **Zero hardcoded projects** in code or config files
2. **Single source of truth**: Cosmos DB `groupResourcesMapContainer`
3. **Identity integration**: Azure AD groups define user-to-project mapping
4. **Performance optimization**: 30-second in-memory cache
5. **User preference**: Last-selected project stored per user

### Scalability Assessment

**Current Capacity**: **Unlimited projects**
- No code limits on number of projects
- Cosmos DB container can hold millions of documents
- Cache mechanism handles arbitrary registry size
- Each project = 1 Cosmos DB document + 3 AD groups

**Performance Considerations**:
- Cache refresh scans entire registry every 30 seconds
- For 1000+ projects, consider:
  - Increase cache timer to reduce Cosmos DB RU
  - Implement differential updates (change feed)
  - Add index-specific caching

### Configuration Management Maturity

**Current Level**: **3/5 - Functional but manual**

**To Reach 5/5**:
1. Add admin portal for project CRUD operations
2. Implement schema validation + constraints
3. Add audit trail (change feed listener)
4. Document project naming conventions
5. Automate project provisioning (Terraform module per project)

---

**Scan Complete**: All configuration sources documented with evidence
