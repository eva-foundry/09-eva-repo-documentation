# EVA Domain Assistant – Solution Requirements (Extracted)

## 1. Infrastructure Architecture

### INF01 – Access & Authentication
- Web interface access for ESDC employees

### INF02 – Hosting
- Hosted in ESDC Azure
- VNet + private endpoints only

---

## 2. Cloud Connectivity

### CC01-DA – Code Management
- Azure DevOps repositories
- EVA Portal Project
- Docker-based execution via Azure Pipelines

### CC02-DA – Containerized Development
- Development inside containers
- Dev containers hosted via GitHub Codespaces

### CC03-DA – Baseline Configuration
- Admin configuration vs user baseline configuration

### CC04-DA – Configuration Reviews
- Quarterly baseline review meetings

---

## 3. Interoperability

### IOP01
- No integration with internal applications
- Interaction limited to OpenAI services and LLMs

---

## 4. Accessibility

### ACC01
- GoC Web Accessibility compliance

### ACC02
- Bilingualism compliance

---

## 5. User Interface

### UI01 – Baseline Configuration
- Number of retrieved documents
- User persona
- System persona
- Response length
- Conversation type
- Folder selection
- Tag-based filtering

### UI02 – User Settings
- Upload/manage documents
- Chat with LLM
- Speech-to-text / text-to-speech
- Document translation (EN/FR)

---

## 6. IT Security

### IT01-DA – Access Control
- Group-based access

### IT02-DA – Roles
- Admin (Blob + Vector Index)
- Contributor (upload/manage)
- Viewer (read/query only)

### IT03-DA – File Upload Processing
- Storage and vector index selected by user role

### IT04-DA – Role Verification
- Azure AD validation

---

## 7. Data Processing

### DP01 – Upload Trigger
- FileUploadedFunc initiates processing

### DP02 – File Type Detection
- PDF vs non-PDF text
- Images vs media

### DP03 – Queue Routing
- pdf_submit_queue
- non_pdf_submit_queue
- image_enrichment_queue
- media_enrichment_queue

### DP04 – Storage
- Processed output stored in Blob Storage
- Embeddings stored separately
