# EVA Chat – Solution Requirements (Extracted)

## 1. Infrastructure Architecture

### INF01 – Access
- EVA Chat is accessible wherever GCNet is available
- Remote users must connect via VPN
- Access is provided through a web interface

### INF02 – Hosting
- Hosted in ESDC Azure Cloud
- Exclusively connected to a Virtual Network (VNet)

### INF03 – User Authentication
- Authentication handled via Microsoft Entra ID (Azure AD)
- Centralized identity and access management

### INF04 – Traffic Management
- All inbound and outbound traffic routed through SSC SCED
- Ensures compliance with GoC security policies

### INF05 – SCED Failover
- Failover between Canada Central and Canada East
- Improves platform resilience

---

## 2. Cloud Connectivity

### CC01 – Private Cloud Connectivity
- Peered with ESDC Azure Hub
- Dedicated private Azure subscription for AI services

### CC02 – Cloud-to-Ground Restriction
- No direct Azure PaaS to SSC on-prem connectivity

### CC03 – Internal Use Only
- No public IP addresses assigned

### CC04 – Secure Traffic Routing
- SSC Trusted Interconnection Points (TIP)

### CC08 – Data Residency
- Hosted in Azure Canada Central

### CC09 – Availability Zones
- Multi-zone deployment
- Independent power, network, and cooling per zone

---

## 3. Interoperability

### IOP01
- No integration with internal applications
- Interaction limited to CSP and LLMs

### IOP02
- Configurable model selection based on product needs

---

## 4. Accessibility

### ACC01 – Compliance
- Follows GoC Standard on Web Accessibility

### ACC02 – Bilingualism
- English and French support

### ACC03 – Accessibility Features
- Keyboard navigation
- Screen reader compatibility
- Text alternatives
- High-contrast modes
- Font resizing
- Speech-to-text and text-to-speech
- Clear error messaging

### ACC04 – Browser Compatibility
- Edge, Chrome, Firefox

### ACC06 – Accessibility Testing
- Periodic testing as part of validation strategy

---

## 5. User Interface

### UI01 – User Settings
- Upload documents
- Chat with LLM
- Speech-to-text and text-to-speech

### UI02 – Workspace
- Saved prompts
- Stored documents
- Model interaction

### UI03 – Permissions
- Language selection
- System prompt configuration
- Interface options
- Memory enable/disable (default OFF)
- Audio configuration
- Import/export/archive chat history
- Profile management

### UI04 – Archived Chats
- Retrieval of archived conversations

---

## 6. IT Security

### ITS01 – RBAC
- Role-Based Access Control enforced

### ITS02 – User Roles
- Admin
- User
- Reader

### ITS03 – Authentication
- Microsoft Entra ID SSO

### ITS04 – Deployment
- Internal-only secured web platform

### ITS05 – Cloud Flexibility
- Vendor-agnostic, multi-cloud capable

### ITS06 – Data Encryption
- TLS 1.3 in transit
- Encrypted at rest with Microsoft-managed keys

### ITS07 – API Exposure
- No external API exposure
