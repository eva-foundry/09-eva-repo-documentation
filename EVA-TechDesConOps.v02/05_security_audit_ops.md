# Security, Audit, and Operations (Extracted)

## Audit & Compliance

### AUD01 – Log Monitoring
- Weekly monitoring
- Logs stored in Azure Logs

### AUD02 – Log Content
- Requests only
- No user input unless explicitly required

### AUD03 – Data Privacy
- Configurable retention policies
- Alignment with ESDC data directives

### AUD04 – Chat History
- Retained 7–35 days
- Configurable

### AUD05 – User Logs
- Anonymized by default

### AUD06 – Model Training
- User inputs NOT used to retrain models

### AUD07 – Content Filtering
- Microsoft content filtering
- Prevents misuse and prohibited language

---

## Incident & Contingency

### INC01 – Detection
- Failures, breaches, performance degradation
- Real-time alerts

### INC02 – Communication
- Automated admin alerts
- User notifications
- Teams channel updates
- Post-incident reports

### INC03 – Contingency
- Automated failover
- Redundancy
- Recovery plans
- Backup protocols

### INC04 – NSD Routing
- User ticket submission via NSD
