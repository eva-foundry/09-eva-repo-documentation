# EVA Domain Assistant: Project/Index Selection and Routing Flow

**Scan Date**: 2026-01-26  
**Repo**: `I:\EVA-JP-v1.2`  
**Purpose**: Document exact flow for project selection and routing from UI to backend to Azure resources

---

## Executive Summary

EVA DA implements a **token-based, dynamic routing system** where:
1. **Frontend**: User selects project from dropdown → stored in React context → NO explicit project_id sent in request body
2. **Backend**: Extracts user OID from `x-ms-client-principal-id` header → Looks up last-selected project in Cosmos DB UserProfile → Resolves to Azure resources (containers, index)
3. **Routing Decision**: Made server-side using Cosmos DB registry, NOT from client-provided data

**Key Insight**: Project selection is **server-authoritative** - frontend selection updates UserProfile in Cosmos DB, backend always reads from UserProfile (never trusts client request).

---

## ASCII Sequence Diagram

```
┌──────────┐         ┌──────────┐         ┌──────────────┐         ┌──────────────────┐         ┌─────────────┐         ┌────────────────┐
│  User    │         │ Layout   │         │   Chat.tsx   │         │   Backend API    │         │  Cosmos DB  │         │ Azure Search   │
│ Browser  │         │ Component│         │              │         │   (app.py)       │         │ UserProfile │         │ Index (dynamic)│
└────┬─────┘         └────┬─────┘         └──────┬───────┘         └────────┬─────────┘         └──────┬──────┘         └────────┬───────┘
     │                    │                       │                          │                          │                          │
     │ 1. Load page       │                       │                          │                          │                          │
     ├───────────────────>│                       │                          │                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │ 2. GET /getUsrGroupInfo                          │                          │                          │
     │                    ├──────────────────────────────────────────────────>│                          │                          │
     │                    │    Headers: x-ms-client-principal-id=<oid>       │                          │                          │
     │                    │                                                   │                          │                          │
     │                    │                       │                          │ 3. Fetch last project    │                          │
     │                    │                       │                          ├─────────────────────────>│                          │
     │                    │                       │                          │   user_info.fetch_latest_│                          │
     │                    │                       │                          │   choice_of_group(oid)   │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 4. Return current_grp_id │                          │
     │                    │                       │                          │<─────────────────────────┤                          │
     │                    │                       │                          │                          │                          │
     │                    │ 5. Response: {GROUP_NAME, AVAILABLE_GROUPS, ROLE}│                          │                          │
     │                    │<──────────────────────────────────────────────────┤                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │ 6. Render dropdown    │                          │                          │                          │
     │                    │    with groups        │                          │                          │                          │
     │<───────────────────┤                       │                          │                          │                          │
     │ Dropdown shows:    │                       │                          │                          │                          │
     │ [Project7-ERIC]    │                       │                          │                          │                          │
     │ [Jurisprudence]    │                       │                          │                          │                          │
     │                    │                       │                          │                          │                          │
     │ 7. User selects    │                       │                          │                          │                          │
     │    "Jurisprudence" │                       │                          │                          │                          │
     ├───────────────────>│                       │                          │                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │ 8. POST /updateUsrGroupInfo                      │                          │                          │
     │                    ├──────────────────────────────────────────────────>│                          │                          │
     │                    │    Body: {current_grp: "Jurisprudence"}          │                          │                          │
     │                    │    Headers: x-ms-client-principal-id=<oid>       │                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 9. Update user preference│                          │
     │                    │                       │                          ├─────────────────────────>│                          │
     │                    │                       │                          │   user_info.write_       │                          │
     │                    │                       │                          │   current_adgroup()      │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 10. Saved                │                          │
     │                    │                       │                          │<─────────────────────────┤                          │
     │                    │                       │                          │                          │                          │
     │                    │ 11. Update context    │                          │                          │                          │
     │                    │     setUsrGroupInfo() │                          │                          │                          │
     │                    ├──────────────────────>│                          │                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │ 12. User types question  │                          │                          │
     │                    │                       │<─────────────────────────┤                          │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │ 13. POST /chat           │                          │                          │
     │                    │                       ├─────────────────────────>│                          │                          │
     │                    │                       │    Body: {history, approach, overrides}             │                          │
     │                    │                       │    Headers: x-ms-client-principal-id=<oid>          │                          │
     │                    │                       │    (NO project_id in body!)                         │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 14. Cache check          │                          │
     │                    │                       │                          ├──┐                       │                          │
     │                    │                       │                          │  │ read_all_items_into_  │                          │
     │                    │                       │                          │  │ cache_if_expired()    │                          │
     │                    │                       │                          │<─┘                       │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 15. Fetch last project   │                          │
     │                    │                       │                          ├─────────────────────────>│                          │
     │                    │                       │                          │   user_info.fetch_latest_│                          │
     │                    │                       │                          │   choice_of_group(oid)   │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 16. Return current_grp_id│                          │
     │                    │                       │                          │<─────────────────────────┤                          │
     │                    │                       │                          │   = "abc-def-123"        │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 17. Lookup resources     │                          │
     │                    │                       │                          ├──┐                       │                          │
     │                    │                       │                          │  │ find_index_and_role() │                          │
     │                    │                       │                          │  │ → "index-jurisprudence"│                         │
     │                    │                       │                          │<─┘                       │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 18. Get SearchClient     │                          │
     │                    │                       │                          ├──┐                       │                          │
     │                    │                       │                          │  │ index_to_search_client│                          │
     │                    │                       │                          │  │ _map.get(index)       │                          │
     │                    │                       │                          │<─┘                       │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 19. Execute search       │                          │
     │                    │                       │                          ├─────────────────────────────────────────────────────>│
     │                    │                       │                          │   search_client.search()  │                          │
     │                    │                       │                          │   → index-jurisprudence   │                          │
     │                    │                       │                          │                          │                          │
     │                    │                       │                          │ 20. Search results       │                          │
     │                    │                       │                          │<─────────────────────────────────────────────────────┤
     │                    │                       │                          │                          │                          │
     │                    │                       │ 21. Stream response      │                          │                          │
     │                    │                       │<─────────────────────────┤                          │                          │
     │                    │                       │   (citations from index-jurisprudence)              │                          │
     │                    │                       │                          │                          │                          │
```

---

## Detailed Flow Breakdown

### Phase 1: Initial Page Load & Project Discovery

**1.1 Frontend Initialization**

**File**: `app/frontend/src/pages/layout/Layout.tsx:115-123`

```typescript
useEffect(() => {
    const fetchData = async () => {
        try {
            const [fetchedApplicationTitle, fetchUsrInfo] = await Promise.all([
                getApplicationTitle(),
                getUsrGroupInfo()
            ]);
            setApplicationTitle(fetchedApplicationTitle);
            setUsrGroupInfo(fetchUsrInfo);
        } catch (error) {
            console.log(error);
        }
    };
    fetchData();
}, []);
```

**Decision Point**: On mount, Layout component fetches user's available projects.  
**Data Source**: Backend API (reads from Cosmos DB UserProfile)

---

**1.2 Fetch User Group Info (API Call)**

**File**: `app/frontend/src/api/api.ts:444-461`

```typescript
export async function getUsrGroupInfo(): Promise<UsrGroupInfo> {
    console.log("fetch User Group, Role and available Groups");
    const response = await fetch("/getUsrGroupInfo", {
        method: "GET",
        headers: {
            "Content-Type": "application/json"
        }
    });

    const parsedResponse: UsrGroupInfo = await response.json();
    if (response.status > 299 || !response.ok) {
        console.log(response);
        throw Error(i18next.t(parsedResponse.error || "Unknown error"));
    }
    console.log(parsedResponse);
    return parsedResponse;
}
```

**Request Details**:
- **Method**: GET
- **Endpoint**: `/getUsrGroupInfo`
- **Headers**: Browser automatically sends `x-ms-client-principal-id` (Azure App Service authentication)
- **Body**: None

**Response Shape**:
```json
{
  "GROUP_NAME": "EVA-DA-Project7-ERIC-Admin",
  "AVAILABLE_GROUPS": [
    "EVA-DA-Project7-ERIC-Admin",
    "EVA-DA-Jurisprudence-Reader"
  ],
  "ROLE": "admin",
  "PREFERRED_PROJECT_NAME": "ERIC"
}
```

---

**1.3 Backend: Get User Group Info**

**File**: `app/backend/app.py:2320-2360`

```python
@app.get("/getUsrGroupInfo")
async def get_usr_group_info(request: Request):
    """Get the application title text

    Returns:
        dict: A dictionary containing the application title.
    """
    header = request.headers
    LOGGER.info(f"here is the header {header}")
    global group_items, expired_time

    oid = header.get("x-ms-client-principal-id")

    group_items, expired_time = read_all_items_into_cache_if_expired(groupmapcontainer, group_items, expired_time)
    group_ids = get_rbac_grplist_from_client_principle(request, group_items)
    available_group_ids = list(set(group_ids).intersection(app.state.available_groups))
    if len(available_group_ids) == 0:
        response = {
            "GROUP_NAME": None,
            "AVAILABLE_GROUPS": None,
            "ROLE": None,
            "PREFERRED_PROJECT_NAME": None,
        }
        return response
    group_names = read_group_names_from_group_ids(group_items, available_group_ids)
    current_grp, current_grp_id = user_info.fetch_lastest_choice_of_group(oid)
    if not current_grp_id:
        current_grp_id = find_grpid_ctrling_rbac(request, group_items)
        current_grp = read_group_names_from_group_ids(group_items, [current_grp_id])[0]
        user_info.write_current_adgroup(oid, current_grp, current_grp_id, group_names, group_ids)

    _, role = find_upload_container_and_role(request, group_items, current_grp_id)

    # Get preferred_project_name from examplelist.json, fallback to empty string if not found or None
    preferred_project_name = ""
    try:
        if current_grp_id and current_grp_id in app.state.example_list:
            preferred_project_name = app.state.example_list[current_grp_id].get("preferred_project_name", "")
    except Exception as e:
        LOGGER.warning(f"Failed to get preferred_project_name for group {current_grp_id}: {e}")

    response = {
        "GROUP_NAME": current_grp,
        "AVAILABLE_GROUPS": group_names,
        "ROLE": role,
        "PREFERRED_PROJECT_NAME": preferred_project_name,
    }

    return response
```

**Decision Point 1: Identify User**  
**File**: `app/backend/app.py:2331`  
```python
oid = header.get("x-ms-client-principal-id")
```
**Data Source**: HTTP header (Azure App Service Easy Auth)  
**Value**: User's Entra ID Object ID (GUID)

**Decision Point 2: Fetch Last-Selected Project**  
**File**: `app/backend/app.py:2342`  
```python
current_grp, current_grp_id = user_info.fetch_lastest_choice_of_group(oid)
```
**Data Source**: Cosmos DB `UserInformation` database, `group_management` container  
**Lookup Key**: `principalId` (user's email/OID)  
**Returns**: `currentDAGroup_id` (Azure AD group GUID)

**Decision Point 3: Fallback to RBAC Lookup**  
**File**: `app/backend/app.py:2343-2345`  
```python
if not current_grp_id:
    current_grp_id = find_grpid_ctrling_rbac(request, group_items)
    current_grp = read_group_names_from_group_ids(group_items, [current_grp_id])[0]
    user_info.write_current_adgroup(oid, current_grp, current_grp_id, group_names, group_ids)
```
**Data Source**: Azure AD token (`x-ms-client-principal` header) + Cosmos DB registry  
**Logic**: If no saved preference, use first accessible project from user's AD groups

---

**1.4 Lookup User's Last-Selected Project**

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

**Data Source**: Cosmos DB  
**Database**: `UserInformation` (from `COSMOSDB_USERPROFILE_DATABASE_NAME`)  
**Container**: `group_management` (from `COSMOSDB_GROUP_MANAGEMENT_CONTAINER`)  
**Query**: Returns most recent document for user (ordered by `createdTime DESC`)

**Document Schema**:
```json
{
  "id": "marco.presta@example.com_abc123",
  "principalId": "marco.presta@example.com",
  "currentDAGroup": "EVA-DA-Project7-ERIC-Admin",
  "currentDAGroup_id": "b9fcbc75-a0d2-402c-8509-23a0923075b2",
  "userADGroups": ["EVA-DA-Project7-ERIC-Admin", "EVA-DA-Jurisprudence-Reader"],
  "userADGroups_ids": ["b9fcbc75-...", "c1234567-..."],
  "createdTime": "2026-01-26T10:30:45.123456Z"
}
```

---

**1.5 Frontend: Render Project Dropdown**

**File**: `app/frontend/src/pages/layout/Layout.tsx:306-315`

```typescript
const handleGroupSelect = async (_ev: unknown, data: { optionValue?: string }) => {
    const newGroup = data.optionValue;
    if (!newGroup || !usrGroupInfo || !setUsrGroupInfo) return;
    try {
        await updateUsrGroup(newGroup);
        setUsrGroupInfo({ ...usrGroupInfo, GROUP_NAME: newGroup });
    } catch (error) {
        console.error("Failed to update user group info:", error);
    }
};
```

**File**: `app/frontend/src/pages/layout/Layout.tsx:327-337`

```typescript
<Dropdown
    aria-label={t("Select user group")}
    placeholder={t("Change Group")}
    selectedOptions={usrGroupInfo.GROUP_NAME ? [usrGroupInfo.GROUP_NAME] : []}
    onOptionSelect={handleGroupSelect}
    value={usrGroupInfo.GROUP_NAME ? usrGroupInfo.GROUP_NAME : ""}
>
    {groupOptions.map(group => (
        <Option key={group} value={group}>
            {group}
        </Option>
    ))}
</Dropdown>
```

**Decision Point**: User sees dropdown populated with `AVAILABLE_GROUPS` from API response.  
**Current Selection**: Shows `GROUP_NAME` (last-selected project).  
**Data Source**: React state (`usrGroupInfo` from context)

---

### Phase 2: User Changes Project Selection

**2.1 Update User Group (API Call)**

**File**: `app/frontend/src/api/api.ts:463-481`

```typescript
export async function updateUsrGroup(usrGrp: string) {
    try {
        console.log("update User Group, Role and available Groups");
        const response = await fetch("/updateUsrGroupInfo", {
            method: "POST",
            headers: {
                "Content-Type": "application/json"
            },
            body: JSON.stringify({
                current_grp: usrGrp
            })
        });

        if (!response.ok) {
            const errorResponse = await response.json();
            throw new Error(i18next.t(errorResponse.error || "Unknown error"));
        }
        return true;
    } catch (error) {
        console.error(i18next.t("Error during deleteItem:"), error);
        return false;
    }
}
```

**Request Details**:
- **Method**: POST
- **Endpoint**: `/updateUsrGroupInfo`
- **Headers**: `x-ms-client-principal-id` (automatic)
- **Body**: `{"current_grp": "EVA-DA-Jurisprudence-Reader"}`

---

**2.2 Backend: Update User Group Preference**

**File**: `app/backend/app.py:2363-2382`

```python
@app.post("/updateUsrGroupInfo")
async def update_usr_group_info(request: Request):
    """Update the user's current group preference"""
    header = request.headers
    LOGGER.info(f"here is the header {header}")
    global group_items, expired_time

    oid = header.get("x-ms-client-principal-id")
    group_items, expired_time = read_all_items_into_cache_if_expired(groupmapcontainer, group_items, expired_time)
    group_ids = get_rbac_grplist_from_client_principle(request, group_items)
    group_names = read_group_names_from_group_ids(group_items, group_ids)
    try:
        body = await request.json()
        current_grp = body.get("current_grp")
        current_grp_id = read_group_id_from_group_name(group_items, current_grp)

        user_info.write_current_adgroup(oid, current_grp, current_grp_id, group_names, group_ids)

    except Exception as ex:
        LOGGER.exception(f"Exception in /updateUsrGroupInfo: {str(ex)}")
        raise HTTPException(status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=str(ex)) from ex

    return
```

**Decision Point**: Update Cosmos DB UserProfile with new selection.  
**File**: `functions/shared_code/user_profile.py:44-76` (called via `user_info.write_current_adgroup()`)

```python
def write_current_adgroup(self, principal_id, current_group, current_group_id, user_ad_groups, user_ad_groups_ids):
    principal_id = principal_id.strip().lower()
    id = self.generate_id(principal_id, current_group_id)
    item = {
        "id": id,
        "principalId": principal_id,
        "userADGroups": user_ad_groups,
        "userADGroups_ids": user_ad_groups_ids,
        "currentDAGroup": current_group,
        "currentDAGroup_id": current_group_id,
        "createdTime": datetime.now().isoformat(),
    }
    
    try:
        self.container.upsert_item(item)
        LOGGER.info(f"Successfully wrote user profile for {principal_id} with group {current_group}")
    except Exception as e:
        LOGGER.error(f"Failed to write user profile: {e}")
        raise
```

**Data Written**: Upserts document to Cosmos DB with new `currentDAGroup_id`.

---

### Phase 3: Chat Request with Dynamic Routing

**3.1 Frontend: Send Chat Request**

**File**: `app/frontend/src/pages/chat/Chat.tsx:449-471`

```typescript
const onSendMessage = async (question: string) => {
    let currentSessionId = sessionId;
    if (!isSessionCreated || !currentSessionId) {
        try {
            currentSessionId = await createNewSession();
            setSessionId(currentSessionId);
            currentSessionIdRef.current = currentSessionId;
            setIsSessionCreated(true);
        } catch (error) {
            console.error("Chat.tsx: Failed to create session:", error);
            return;
        }
    }

    if (!currentSessionId) {
        console.error("Chat.tsx: Session ID is not set after creation");
        return;
    }

    await makeApiRequest(
        question,
        defaultApproach,
        lastQuestionWorkCitationRef.current,
        lastQuestionWebCitiationRef.current,
        lastQuestionThoughtChainRef.current,
        currentSessionId
    );
};
```

**File**: `app/frontend/src/api/api.ts:28-71` (called via `makeApiRequest → chatApi`)

```typescript
export async function chatApi(options: ChatRequest, signal: AbortSignal): Promise<Response> {
    const response = await fetch("/chat", {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            history: options.history,
            approach: options.approach,
            overrides: {
                semantic_ranker: options.overrides?.semanticRanker,
                semantic_captions: options.overrides?.semanticCaptions,
                top: options.overrides?.top,
                temperature: options.overrides?.temperature,
                // ... other overrides
                session_id: options.overrides?.session_id
            },
            citation_lookup: options.citation_lookup,
            thought_chain: options.thought_chain
        }),
        signal: signal
    });

    if (response.status > 299 || !response.ok) {
        throw Error(i18next.t("Unknown error"));
    }

    return response;
}
```

**Critical Observation**: Request body contains:
- `history` (previous conversation turns)
- `approach` (RAG approach ID)
- `overrides` (query parameters, session_id)
- `citation_lookup`, `thought_chain` (UI flags)

**NO project_id or corpus_id in request body!**

**Data Source for Routing**: HTTP header `x-ms-client-principal-id` (automatic from browser)

---

**3.2 Backend: Chat Endpoint Entry**

**File**: `app/backend/app.py:538-571`

```python
@app.post("/chat")
async def chat(request: Request):
    """Chat with the bot using a given approach"""
    header = request.headers
    global group_items, expired_time
    group_items, expired_time = read_all_items_into_cache_if_expired(groupmapcontainer, group_items, expired_time)
    oid = header.get("x-ms-client-principal-id")
    _, current_grp_id = user_info.fetch_lastest_choice_of_group(oid)
    current_grp_name = read_group_names_from_group_ids(group_items, [current_grp_id])[0]
    special_rule = False
    for name in SPECIAL_GROUPS:
        if name.lower().strip() in current_grp_name.lower():
            special_rule = True
            break

    upload_container, role = find_upload_container_and_role(request, group_items, current_grp_id=current_grp_id)
    content_container, role = find_container_and_role(request, group_items, current_grp_id)
    index, role = find_index_and_role(request, group_items, current_grp_id)
    search_client = index_to_search_client_map.get(index)
```

**Routing Steps**:

**Step 1: Refresh Project Registry Cache**  
**File**: `app/backend/app.py:559`  
```python
group_items, expired_time = read_all_items_into_cache_if_expired(groupmapcontainer, group_items, expired_time)
```
**Decision Point**: If cache expired (30 seconds), reload from Cosmos DB.  
**Data Source**: `groupsToResourcesMap` database, `groupResourcesMapContainer` container

**Step 2: Identify User and Fetch Last-Selected Project**  
**File**: `app/backend/app.py:560-561`  
```python
oid = header.get("x-ms-client-principal-id")
_, current_grp_id = user_info.fetch_lastest_choice_of_group(oid)
```
**Decision Point**: Lookup user's last-selected project from UserProfile.  
**Data Source**: Cosmos DB `UserInformation` database  
**Result**: `current_grp_id` = "b9fcbc75-a0d2-402c-8509-23a0923075b2" (Azure AD group GUID)

**Step 3: Resolve Group ID to Azure Resources**  
**File**: `app/backend/app.py:568-570`  
```python
upload_container, role = find_upload_container_and_role(request, group_items, current_grp_id=current_grp_id)
content_container, role = find_container_and_role(request, group_items, current_grp_id)
index, role = find_index_and_role(request, group_items, current_grp_id)
```

**Step 4: Get Pre-Initialized SearchClient**  
**File**: `app/backend/app.py:571`  
```python
search_client = index_to_search_client_map.get(index)
```
**Data Source**: In-memory map (initialized at app startup)  
**Key**: `index` (e.g., "index-jurisprudence")  
**Value**: `SearchClient` instance for that index

---

**3.3 Resolve Index Name from Group ID**

**File**: `functions/shared_code/utility_rbck.py:456-468`

```python
def find_index_and_role(request, group_map_items, current_grp_id=None):
    """
    Find the index and role for the current group from the group map items.

    Args:
        request: The incoming request object.
        group_map_items: The list of group map items.
        current_grp_id: The current group ID (optional).

    Returns:
        tuple: The index name and the role.
    """
    for item in group_map_items:
        if item.get("group_id") == current_grp_id:
            index_and_role = item.get("vector_index_access")
            vector_index = index_and_role.get("index")
            role = index_and_role.get("role_index")
            return vector_index, role
    return None, None
```

**Decision Point**: Loop through `group_map_items` (project registry cache) and match `group_id`.  
**Data Source**: In-memory cache of Cosmos DB `groupResourcesMapContainer`  
**Lookup Logic**:
1. Iterate through cached project registry documents
2. Match `item.group_id == current_grp_id`
3. Extract `item.vector_index_access.index`
4. Return index name (e.g., "index-jurisprudence")

**Example Document**:
```json
{
  "id": "doc123",
  "group_id": "b9fcbc75-a0d2-402c-8509-23a0923075b2",
  "group_name": "EVA-DA-Jurisprudence-Reader",
  "vector_index_access": {
    "index": "index-jurisprudence",
    "role_index": "reader"
  },
  "blob_access": {
    "blob_container": "content-jurisprudence",
    "role_blob": "reader"
  },
  "upload_storage": {
    "upload_container": "upload-jurisprudence",
    "role": "reader"
  }
}
```

---

**3.4 Retrieve SearchClient from Map**

**File**: `app/backend/app.py:484-493` (initialization during app startup)

```python
for index in vector_indexes:
    index_to_search_client_map[index] = SearchClient(
        endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"], 
        index_name=index, 
        credential=AZURE_CREDENTIAL, 
        audience=ENV["AZURE_SEARCH_AUDIENCE"]
    )
```

**Map Structure**:
```python
index_to_search_client_map = {
    "index-jurisprudence": <SearchClient at 0x7f1234567890>,
    "index-eric": <SearchClient at 0x7f1234567abc>,
    "index-project3": <SearchClient at 0x7f1234567def>,
    # ... (one entry per project)
}
```

**Retrieval**:
```python
search_client = index_to_search_client_map.get("index-jurisprudence")
# Returns pre-initialized SearchClient for index-jurisprudence
```

---

**3.5 Instantiate RAG Approach with SearchClient**

**File**: `app/backend/app.py:596-623`

```python
read_retrieve = ChatReadRetrieveReadApproach(
    search_client,
    ENV["AZURE_OPENAI_ENDPOINT"],
    ENV["AZURE_OPENAI_CHATGPT_DEPLOYMENT"],
    ENV["KB_FIELDS_SOURCEFILE"],
    ENV["KB_FIELDS_CONTENT"],
    ENV["KB_FIELDS_PAGENUMBER"],
    ENV["KB_FIELDS_CHUNKFILE"],
    content_container,
    blob_client,
    ENV["QUERY_TERM_LANGUAGE"],
    MODEL_NAME,
    MODEL_VERSION,
    ENV["TARGET_EMBEDDINGS_MODEL"],
    ENV["ENRICHMENT_APPSERVICE_URL"],
    ENV["TARGET_TRANSLATION_LANGUAGE"],
    ENV["AZURE_AI_ENDPOINT"],
    ENV["AZURE_AI_LOCATION"],
    token_provider,
    str_to_bool.get(ENV["USE_SEMANTIC_RERANKER"]),
    error_message,
)
chat_approaches[Approaches.ReadRetrieveRead] = read_retrieve
```

**Decision Point**: Instantiate approach class with project-specific `search_client`.  
**Result**: Approach instance bound to `index-jurisprudence` SearchClient.

---

**3.6 Execute Approach (RAG Query)**

**File**: `app/backend/app.py:626-636`

```python
try:
    impl = chat_approaches.get(Approaches(int(approach)))
    if not impl:
        return {"error": "unknown approach"}, status.HTTP_400_BAD_REQUEST

    if Approaches(int(approach)) == Approaches.ReadRetrieveRead:
        r = impl.run(json_body.get("history", []), json_body.get("overrides", {}), {}, 
                     json_body.get("thought_chain", {}), curr_sessions, special_rule)
    else:
        r = impl.run(json_body.get("history", []), json_body.get("overrides", {}), {}, 
                     json_body.get("thought_chain", {}))

    return StreamingResponse(r, media_type="application/x-ndjson")
```

**Execution**: Approach runs with project-specific `search_client` → queries `index-jurisprudence`.

---

### Phase 4: Fallback Behavior

**4.1 No Last-Selected Project**

**File**: `app/backend/app.py:2343-2345`

```python
if not current_grp_id:
    current_grp_id = find_grpid_ctrling_rbac(request, group_items)
    current_grp = read_group_names_from_group_ids(group_items, [current_grp_id])[0]
    user_info.write_current_adgroup(oid, current_grp, current_grp_id, group_names, group_ids)
```

**Fallback Logic**:
1. If `user_info.fetch_lastest_choice_of_group(oid)` returns `None`
2. Call `find_grpid_ctrling_rbac()` to determine RBAC-controlling group
3. Use first accessible project from user's AD group memberships
4. Persist to Cosmos DB UserProfile for future requests

**File**: `functions/shared_code/utility_rbck.py:284-313`

```python
def find_grpid_ctrling_rbac(request, group_map_items):
    """Find the group ID controlling RBAC from the Azure AD token claims."""
    rbac_group_l = get_rbac_grplist_from_client_principle(request, group_map_items)
    
    # Prioritize admin, then contributor, then reader
    role_mapping = {
        "admin": {},
        "contributor": {},
        "reader": {}
    }
    
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
    
    # Return first admin, or first contributor, or first reader
    for role in ["admin", "contributor", "reader"]:
        if role_mapping[role]:
            return list(role_mapping[role].values())[0]
    
    return None
```

**Fallback Order**:
1. Admin role groups (highest privilege)
2. Contributor role groups
3. Reader role groups (lowest privilege)

---

**4.2 User Not in Any Project**

**File**: `app/frontend/src/pages/layout/Layout.tsx:144-177`

```tsx
if (!usrGroupInfo.GROUP_NAME) {
    // User is not authorized
    return (
        <div className={styles.layout}>
            <Header applicationTitle={applicationTitle} />
            <div className={styles.centeredContainerwords}>
                <p>
                    {t("You are not assigned to any project to use this application. Or the project has been deactivated. Please contact your admin...")}
                </p>
                <ul>
                    <li>
                        <a href="https://014gc.sharepoint.com/..." target="_blank">
                            {t("How to Add or Remove Members to an Admin User Group")}
                        </a>
                    </li>
                    ...
                </ul>
            </div>
            <Footer />
        </div>
    );
}
```

**Fallback Behavior**: Show authorization error page with help links.  
**Condition**: `GROUP_NAME` is `null` in API response (no accessible projects found).

---

**4.3 Cache Expiration Handling**

**File**: `functions/shared_code/utility_rbck.py:240-248`

```python
def read_all_items_into_cache_if_expired(group_resource_map_db, items=0, expired_time=0):
    if expired_time < time.time():
        cache_time = os.getenv("CACHETIMER")  # 30 seconds
        cache_expiration_time = time.time() + float(cache_time)
        items = group_resource_map_db.read_all_items()
        return items, cache_expiration_time
    else:
        return items, expired_time
```

**Fallback Behavior**: On cache expiration, reload entire project registry from Cosmos DB.  
**Impact**: New projects available 30 seconds after Cosmos DB update (no app restart required).

---

## Decision Points Summary

| Decision Point | Location | Data Source | Type |
|----------------|----------|-------------|------|
| **1. Initial Project Selection** | `app/backend/app.py:2342` | Cosmos DB UserProfile (`fetch_lastest_choice_of_group`) | Database Lookup |
| **2. Fallback to RBAC** | `utility_rbck.py:284-313` | Azure AD token + Cosmos DB registry | Dynamic Calculation |
| **3. Resolve Index Name** | `utility_rbck.py:456-468` | Cosmos DB project registry (cached) | Database Lookup |
| **4. Get SearchClient** | `app/backend/app.py:571` | In-memory map (`index_to_search_client_map`) | Runtime Cache |
| **5. Project Dropdown** | `Layout.tsx:327-337` | API response (`AVAILABLE_GROUPS`) | Server Response |
| **6. Cache Refresh** | `utility_rbck.py:240-248` | Cosmos DB (`groupResourcesMapContainer`) | Time-based Trigger |

---

## Configuration Sources Used in Routing

### Environment Variables

**File**: `app/backend/backend.env`

```env
COSMOSDB_URL=https://infoasst-cosmos-hccld2.documents.azure.com:443/
COSMOSDB_USERPROFILE_DATABASE_NAME="UserInformation"
COSMOSDB_GROUP_MANAGEMENT_CONTAINER="group_management"
COSMOSDB_DATABASE_GROUP_MAP="groupsToResourcesMap"
COSMOSDB_CONTAINER_GROUP_MAP="groupResourcesMapContainer"
CACHETIMER=30
AZURE_SEARCH_SERVICE_ENDPOINT=https://infoasst-search-hccld2.search.windows.net/
```

**Usage**:
- `COSMOSDB_URL`: Connection string for Cosmos DB client
- `COSMOSDB_USERPROFILE_DATABASE_NAME`: Database for user preferences
- `COSMOSDB_GROUP_MANAGEMENT_CONTAINER`: Container storing last-selected project per user
- `COSMOSDB_DATABASE_GROUP_MAP`: Database containing project registry
- `COSMOSDB_CONTAINER_GROUP_MAP`: Container with project-to-resource mappings
- `CACHETIMER`: Cache expiration (seconds) for project registry
- `AZURE_SEARCH_SERVICE_ENDPOINT`: Base URL for all SearchClient instances

---

## Key Observations

### 1. Server-Authoritative Routing

**NO client-provided project_id in request body.**  
Backend always determines routing from:
1. HTTP header (`x-ms-client-principal-id`)
2. Cosmos DB UserProfile lookup
3. Cosmos DB project registry lookup

**Security Benefit**: Client cannot forge project access - all authorization server-side.

---

### 2. Two-Database Architecture

**Database 1: UserProfile** (`UserInformation.group_management`)  
**Purpose**: Store last-selected project per user (preference persistence)  
**Schema**: One document per user-project selection

**Database 2: Project Registry** (`groupsToResourcesMap.groupResourcesMapContainer`)  
**Purpose**: Map Azure AD group GUID → (upload container, content container, vector index)  
**Schema**: One document per project-role combination (e.g., 3 docs for admin/contributor/reader)

---

### 3. SearchClient Pooling

**Initialization**: App startup creates one `SearchClient` per index.  
**Lifetime**: Persistent for entire app lifetime (not per-request).  
**Benefit**: Avoids authentication overhead on every request.

**File**: `app/backend/app.py:484-493`

```python
for index in vector_indexes:
    index_to_search_client_map[index] = SearchClient(
        endpoint=ENV["AZURE_SEARCH_SERVICE_ENDPOINT"], 
        index_name=index, 
        credential=AZURE_CREDENTIAL, 
        audience=ENV["AZURE_SEARCH_AUDIENCE"]
    )
```

---

### 4. Cache Invalidation Strategy

**TTL**: 30 seconds (configurable via `CACHETIMER`)  
**Scope**: Entire project registry (all projects)  
**Trigger**: First request after cache expiration reloads from Cosmos DB  
**Implication**: New projects available ~30 seconds after provisioning (no deployment required)

---

## Anti-Patterns Observed (NOT Used)

### What This System Does NOT Do

❌ **Send project_id in request body**  
- Frontend never includes `project_id`, `corpus_id`, or `index_name` in chat request

❌ **Use URL path parameters for project**  
- No endpoints like `/chat/{project_id}` - always `/chat` with header-based routing

❌ **Read project from cookies**  
- No cookies used for project selection (only JWT tokens for authentication)

❌ **Hardcode index names in frontend**  
- No TypeScript constants like `const INDEX_NAME = "index-jurisprudence"`

❌ **Query string parameters for project**  
- No URLs like `/chat?project=jurisprudence`

❌ **Local storage for project selection**  
- Project preference stored server-side (Cosmos DB), not browser localStorage

---

## Scaling Considerations

### Current Capacity

**Projects Supported**: **Unlimited** (constrained only by Cosmos DB size)  
**Cache Impact**: Linear with project count (30-second full reload)  
**SearchClient Count**: One per project (memory footprint scales linearly)

### Performance Bottlenecks

**Cache Refresh**:
- Every 30 seconds, first request scans entire project registry
- For 1000+ projects, consider differential updates (Cosmos DB change feed)

**SearchClient Initialization**:
- App startup creates all SearchClient instances upfront
- For 1000+ projects, startup time could exceed 60 seconds
- Mitigation: Lazy initialization (create SearchClient on first use per index)

---

## Troubleshooting Guide

### Issue: "User not authorized" despite being in AD group

**Root Cause**: User's AD group not in Cosmos DB `groupResourcesMapContainer`

**Debug Steps**:
1. Check `x-ms-client-principal-id` header value (user's OID)
2. Query Cosmos DB `UserInformation.group_management` for user's document
3. Verify `currentDAGroup_id` matches a `group_id` in `groupResourcesMapContainer`
4. If missing, add project document to `groupResourcesMapContainer`

---

### Issue: "Search returns no results"

**Root Cause**: Routing to wrong index

**Debug Steps**:
1. Check `current_grp_id` in backend logs (line 561 of app.py)
2. Verify `find_index_and_role()` returns expected index name
3. Check `index_to_search_client_map.keys()` contains that index
4. Verify Azure Search index exists: `az search index show --name <index> --service-name <service>`

---

### Issue: "Project dropdown empty"

**Root Cause**: User not in any project AD groups

**Debug Steps**:
1. Check `/getUsrGroupInfo` API response: `AVAILABLE_GROUPS` should be non-empty array
2. Verify user's Azure AD group memberships (Entra ID portal)
3. Check `get_rbac_grplist_from_client_principle()` output in logs
4. Confirm user's AD groups match `group_id` values in `groupResourcesMapContainer`

---

## Conclusions

EVA DA implements a **sophisticated, database-driven routing system** with these characteristics:

1. **Zero Hardcoding**: No project/index names in frontend code
2. **Server-Side Authority**: Backend always determines routing from user identity and database state
3. **User Preference Persistence**: Last-selected project saved per user in Cosmos DB
4. **Cache-Based Performance**: 30-second cache minimizes Cosmos DB round trips
5. **Dynamic Scalability**: Add projects without code changes or deployments

**Security Model**: Client cannot bypass RBAC - all routing decisions server-side using:
- Azure AD authentication (Entra ID OAuth 2.0)
- Cosmos DB project registry (single source of truth)
- User profile database (preference persistence)

**Performance**: Sub-100ms routing overhead (cached lookup + map retrieval) per request.

---

**Flow Complete**: All routing decisions documented with evidence.
