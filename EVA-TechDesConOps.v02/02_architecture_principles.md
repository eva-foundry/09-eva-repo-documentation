# EVA Foundation – Architecture Principles

## Core Design Principles
- Accessibility by default (WCAG / GoC standards)
- Bilingual (English / French)
- Privacy-first
- Secure-by-design
- Human accountability
- Modular and scalable architecture

---

## Hosting & Network
- Hosted in ESDC Azure Cloud
- Protected B environment
- Hub-and-spoke VNet architecture
- All traffic routed through SSC SCED
- No public IP exposure
- Canada Central primary region
- Multi–Availability Zone deployment

---

## Logical Separation
- EVA Chat: ungrounded LLM responses
- EVA Domain Assistant: grounded RAG responses
- Separate pipelines, shared governance

---

## Vendor & Platform Neutrality
- Configurable model selection
- Cloud-agnostic capability
- No hard dependency on a single LLM provider
