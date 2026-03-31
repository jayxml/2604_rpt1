# Predictive JIT Supply Chain with SAP BTP

> **Session overview**
>
> | Part | Format | What you will do |
> |------|--------|------------------|
> | **Part 1 — Hands-On Exercise** | Self-paced | Build predictive delay scoring with SAP-RPT-1 |
> | **Part 2 — Hands-On Exercise** | Self-paced | Extend to agentic mitigation with ReAct pattern |
> | **Part 3 — Architecture Discussion** | Presenter-led | Walk through production architecture patterns for JIT risk management on SAP BTP |

---

# Part 3: Architecture Discussion (Presenter-Led)

> **Note for participants:** The sections below are **not hands-on steps**. Your instructor will walk through these topics as a guided discussion. You can follow along and refer back to this document after the session.

## Contents

| Scenario | Description | Audience |
|----------|-------------|----------|
| **A** | From Notebook to Production CAP Application | Developers + Architects |
| **B** | End-to-End Data Integration from SAP S/4HANA | Integration Specialists |
| **C** | HANA Cloud as the Prediction Data Store | Data Engineers |
| **D** | Go-Live Checklist and Operational Excellence | All |

---

## Scenario A: From Notebook to Production CAP Application

### Use Case A: Production-Grade JIT Risk Dashboard

**Business situation:** The supply chain team needs a governed, always-on system that continuously scores incoming POs for delay risk. They cannot rely on ad-hoc notebook execution or manual monitoring.

**Why this use case matters:**
- Production line efficiency depends on proactive risk detection
- Business users need a simple dashboard, not Python notebooks
- Audit trail and approval workflows are required for compliance

**What the end users need:**
- A Fiori-style application showing current PO risk status
- Automatic alerts when Red-tier risks are detected
- One-click access to mitigation proposals
- Approval workflow before sourcing changes are executed

### Notebook → Production: What Changes?

| Concern | Notebook (Today) | Production (Target) |
|---------|------------------|---------------------|
| **User interface** | Code cells | Fiori / SAP Build Apps |
| **Authentication** | `.env` file | XSUAA + service bindings |
| **Data freshness** | Static CSV files | Live S/4HANA integration |
| **Model inference** | Manual trigger | Event-driven or scheduled |
| **Observability** | Print statements | Structured logging + dashboards |
| **Approval workflow** | Console input | BTP Workflow / Build Process Automation |
| **Error handling** | Exceptions | Retry policies, dead-letter queues |
| **Scalability** | Single user | Multi-tenant CAP deployment |

> **Important:**
> The code snippets in this scenario are **illustrative only**. They demonstrate patterns, not runnable implementations.

### A.1 Target Solution Architecture (High Level)

```text
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                          │
│              SAP Fiori / SAP Build Apps                         │
├─────────────────────────────────────────────────────────────────┤
│                     Application Layer                           │
│               CAP Service (Node.js / Java)                      │
│    ┌────────────────┐  ┌─────────────────┐  ┌──────────────┐   │
│    │ /assess-risk   │  │ /get-mitigation │  │ /approve     │   │
│    └───────┬────────┘  └────────┬────────┘  └──────┬───────┘   │
│            │     Risk Scoring Engine     │         │            │
│            v                             v         v            │
│    ┌──────────────┐          ┌─────────────────────────┐       │
│    │  SAP-RPT-1   │          │   Agent Orchestrator    │       │
│    │  Inference   │          │   (LLM + Tools)         │       │
│    └──────┬───────┘          └──────────┬──────────────┘       │
├───────────┼──────────────────────────────┼──────────────────────┤
│           v        SAP AI Services       v                      │
│    ┌────────────────────────────────────────────────┐          │
│    │  AI Core · SAP-RPT-1 · Gen AI Hub · Claude     │          │
│    └────────────────────────────────────────────────┘          │
├─────────────────────────────────────────────────────────────────┤
│                  Data & Integration Layer                       │
│    S/4HANA APIs · Integration Suite · HANA Cloud (optional)     │
├─────────────────────────────────────────────────────────────────┤
│                  Platform & Security Layer                      │
│         XSUAA · Destinations · Service Bindings                 │
└─────────────────────────────────────────────────────────────────┘
```

**SAP BTP value mapping:**
- **CAP** gives standard service APIs, security integration, and extension-ready app model
- **AI Core** provides managed AI runtime with SAP-RPT-1 and LLM access
- **Gen AI Hub SDK** enables consistent orchestration patterns
- **Integration Suite** connects to live ERP data

### A.2 Recommended Build Steps (CAP Productization)

1. **Create CAP service APIs:**
   - `POST /api/assess-po-risk` - Score a single PO
   - `POST /api/batch-assess` - Score multiple POs
   - `GET /api/risk-dashboard` - Current risk status
   - `POST /api/generate-mitigation` - Create proposal
   - `POST /api/approve-mitigation` - Approval endpoint

2. **Implement a risk scoring service:**
   - Wrap SAP-RPT-1 inference calls
   - Cache historical context for efficiency
   - Return standardized risk tier + confidence

3. **Add governance controls:**
   - Persist all predictions with timestamps
   - Log agent reasoning traces
   - Require approval for sourcing changes

4. **Add enterprise security:**
   - XSUAA scopes for viewer vs approver personas
   - Secure secrets via service bindings
   - Audit log all decisions

5. **Add operational quality:**
   - Structured logs to SAP Cloud Logging
   - Alert on high-risk PO spikes
   - Dashboard for model accuracy metrics

### A.3 CAP Sample Code (Illustrative)

```javascript
// srv/risk-service.js (illustrative only)
const cds = require('@sap/cds');

module.exports = cds.service.impl(async function () {
  
  this.on('assessPORisk', async (req) => {
    const { poId } = req.data;
    
    // Fetch PO details from S/4HANA
    const poDetails = await fetchPOFromS4(poId);
    
    // Get historical context
    const historicalContext = await getHistoricalPOContext(poDetails.vendorId);
    
    // Call SAP-RPT-1 for prediction
    const prediction = await callRPT1Model(poDetails, historicalContext);
    
    // Apply business policy
    const riskTier = deriveRiskTier(prediction.delayDays);
    
    // Persist for audit
    await persistPrediction(poId, prediction, riskTier);
    
    return {
      poId,
      predictedDelay: prediction.delayDays,
      riskTier,
      recommendation: getRiskRecommendation(riskTier, poDetails.isCritical)
    };
  });

  this.on('generateMitigation', async (req) => {
    const { poId, riskAssessment } = req.data;
    
    // Only for high-risk scenarios
    if (riskAssessment.riskTier !== 'Red') {
      return { status: 'NOT_REQUIRED' };
    }
    
    // Run agent for mitigation proposal
    const agentResult = await runMitigationAgent(poId, riskAssessment);
    
    // Persist proposal
    const proposalId = await persistProposal(agentResult);
    
    return {
      proposalId,
      status: 'PENDING_APPROVAL',
      recommendation: agentResult.recommendation,
      alternativeVendor: agentResult.suggestedVendor
    };
  });
});
```

### A.4 AI Service Integration Layer

```javascript
// srv/lib/ai-service.js (illustrative only)
const { OrchestrationService } = require('gen_ai_hub.orchestration');

async function callRPT1Model(poDetails, historicalContext) {
  // Format data for SAP-RPT-1
  const payload = formatForRPT1(poDetails, historicalContext);
  
  // Call AI Core deployment
  const response = await aiCoreClient.predict(
    process.env.RPT1_DEPLOYMENT_URL,
    { input: payload }
  );
  
  return {
    delayDays: response.prediction,
    confidence: response.confidence
  };
}

async function runMitigationAgent(poId, riskAssessment) {
  const orchestration = new OrchestrationService({
    apiUrl: process.env.ORCH_DEPLOYMENT_URL
  });
  
  // Agent with tools for supplier lookup, capacity check, etc.
  const result = await orchestration.run({
    template: agentTemplate,
    llm: { name: 'anthropic--claude-4.5-sonnet' },
    context: { poId, riskAssessment }
  });
  
  return parseAgentResult(result);
}
```

### A.5 Deployment Steps (CAP + AI Services)

1. **Provision or bind services:**
   - SAP AI Core (for RPT-1 and LLM access)
   - XSUAA (authentication)
   - Destination Service (for S/4HANA connectivity)
   - SAP HANA Cloud (optional, for persistence)

2. **Configure environment:**
   - Bind service keys
   - Set deployment URLs for AI models
   - Configure destination to S/4HANA

3. **Build and deploy:**
   ```bash
   # Build MTA archive
   mbt build
   
   # Deploy to Cloud Foundry
   cf deploy mta_archives/jit-risk-app.mtar
   ```

4. **Post-deployment verification:**
   - Health endpoint check
   - Test single PO risk assessment
   - Verify agent produces valid proposals
   - Confirm approval workflow triggers

---

## Scenario B: End-to-End Data Integration from SAP S/4HANA

### Use Case B: Live ERP Data for Accurate Predictions

**Business situation:** The workshop used static CSV files exported from S/4HANA. In production, predictions must use current data—vendor performance changes, new POs arrive continuously, and historical patterns evolve.

**Why this use case matters:**
- Stale data leads to inaccurate predictions
- Manual data exports don't scale
- Enterprise teams need governed integration

**What the business needs:**
- Automatic data pipeline from S/4HANA to prediction service
- Near real-time updates for critical attributes
- Clear data freshness SLAs

### B.1 Data Sources in S/4HANA

| Data Element | S/4HANA Source | Update Frequency |
|--------------|----------------|------------------|
| Purchase Orders | EKKO/EKPO (OData: `A_PurchaseOrder`) | Real-time |
| Vendor Master | LFA1 (OData: `A_Supplier`) | Daily |
| Material Master | MARA (OData: `A_Product`) | Weekly |
| Delivery History | LIKP/LIPS (OData: `A_InbDeliveryHeader`) | Real-time |
| Quality Notifications | QMEL (OData: `A_DefectRecord`) | Daily |

### B.2 Integration Patterns

**Pattern 1: Scheduled Batch Synchronization**
- Best for: Historical context, vendor profiles, material master
- Frequency: Daily or weekly
- Implementation: Integration Suite iFlow → HANA Cloud or Object Store

```text
S/4HANA (OData)
   │ (scheduled extraction)
   v
Integration Suite iFlow
   │ (transform + validate)
   v
HANA Cloud / Object Store
   │
   v
Prediction Service reads from cache
```

**Pattern 2: Event-Driven Real-Time Updates**
- Best for: New PO creation, delivery confirmations
- Frequency: Near real-time (seconds to minutes)
- Implementation: S/4HANA Business Events → Event Mesh → CAP Service

```text
S/4HANA
   │ (Business Event: PO Created)
   v
SAP Event Mesh
   │ (pub/sub)
   v
CAP Service Event Handler
   │ (trigger risk assessment)
   v
Risk Dashboard Updated
```

### B.3 Integration Flow Example (Illustrative)

```groovy
// Integration Suite iFlow script (illustrative only)
// Extracts vendor performance metrics from S/4HANA

def message = bodyAs(String.class)
def vendors = parseVendorData(message)

vendors.each { vendor ->
    // Calculate OTIF from delivery history
    def deliveries = getDeliveryHistory(vendor.vendorId)
    def otifPercent = calculateOTIF(deliveries)
    def avgDelay = calculateAvgDelay(deliveries)
    
    vendor.otifPercent = otifPercent
    vendor.avgPastDelay = avgDelay
    vendor.lastUpdated = now()
}

// Output to HANA Cloud or Object Store
message.setBody(toJSON(vendors))
message.setHeader('targetTable', 'VENDOR_PERFORMANCE')
return message
```

### B.4 Event-Driven Scoring Architecture

```text
┌─────────────────┐     ┌───────────────┐     ┌─────────────────┐
│   S/4HANA       │────>│  Event Mesh   │────>│   CAP Service   │
│  PO Created     │     │  (Topic)      │     │  Event Handler  │
└─────────────────┘     └───────────────┘     └────────┬────────┘
                                                       │
                                                       v
                                              ┌─────────────────┐
                                              │  Risk Scoring   │
                                              │  (SAP-RPT-1)    │
                                              └────────┬────────┘
                                                       │
                                                       v
                                              ┌─────────────────┐
                                              │  If Red + Crit  │
                                              │  → Agent Flow   │
                                              └────────┬────────┘
                                                       │
                                                       v
                                              ┌─────────────────┐
                                              │  Notification   │
                                              │  to SC Manager  │
                                              └─────────────────┘
```

### B.5 Trade-offs Discussion

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| **Data sync** | Batch (daily) | Event-driven | Event for POs; Batch for master data |
| **S/4 access** | Direct OData | Integration Suite | Integration Suite for production |
| **Context storage** | In-memory | HANA Cloud | HANA Cloud for persistence + analytics |
| **Scoring trigger** | Scheduled batch | On PO creation | Event-driven for critical; Batch for bulk |

---

## Scenario C: HANA Cloud as the Prediction Data Store

### Use Case C: Scalable Context Management

**Business situation:** SAP-RPT-1 performs better with rich historical context. As PO volume grows, managing context efficiently becomes critical.

**Why HANA Cloud:**
- Native integration with BTP services
- Vector capabilities for future ML enhancements
- Enterprise-grade performance and security
- Single source of truth for analytics + ML

### C.1 Data Model for Prediction Context

```sql
-- Historical PO Performance (for SAP-RPT-1 context)
CREATE TABLE PO_HISTORY (
    PO_ID NVARCHAR(20) PRIMARY KEY,
    VENDOR_ID NVARCHAR(10),
    MATERIAL_ID NVARCHAR(18),
    ORDER_QUANTITY INTEGER,
    ORDER_DATE DATE,
    PLANNED_DELIVERY_DATE DATE,
    ACTUAL_DELIVERY_DATE DATE,
    ACTUAL_DELAY_DAYS DECIMAL(5,1),
    CREATED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Vendor Performance Metrics (derived)
CREATE TABLE VENDOR_PERFORMANCE (
    VENDOR_ID NVARCHAR(10) PRIMARY KEY,
    VENDOR_COUNTRY NVARCHAR(50),
    OTIF_PERCENT DECIMAL(5,2),
    AVG_DELAY_DAYS DECIMAL(5,2),
    TOTAL_POS INTEGER,
    LAST_UPDATED TIMESTAMP
);

-- Risk Predictions (audit trail)
CREATE TABLE RISK_PREDICTIONS (
    PREDICTION_ID NVARCHAR(36) PRIMARY KEY,
    PO_ID NVARCHAR(20),
    PREDICTED_DELAY DECIMAL(5,1),
    RISK_TIER NVARCHAR(10),
    CONFIDENCE DECIMAL(3,2),
    MODEL_VERSION NVARCHAR(20),
    PREDICTED_AT TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Mitigation Proposals
CREATE TABLE MITIGATION_PROPOSALS (
    PROPOSAL_ID NVARCHAR(36) PRIMARY KEY,
    PO_ID NVARCHAR(20),
    CURRENT_VENDOR NVARCHAR(10),
    RECOMMENDED_VENDOR NVARCHAR(10),
    STATUS NVARCHAR(20), -- PENDING, APPROVED, REJECTED
    JUSTIFICATION NCLOB,
    CREATED_AT TIMESTAMP,
    DECIDED_AT TIMESTAMP,
    DECIDED_BY NVARCHAR(100)
);
```

### C.2 Context Query for SAP-RPT-1

```sql
-- Get historical context for a prediction
SELECT 
    PO_ID,
    VENDOR_ID,
    VP.VENDOR_COUNTRY,
    VP.OTIF_PERCENT AS VENDOR_OTIF_PERCENT,
    VP.AVG_DELAY_DAYS AS VENDOR_AVG_PAST_DELAY,
    MATERIAL_ID,
    ORDER_QUANTITY,
    EXTRACT(MONTH FROM ORDER_DATE) AS ORDER_MONTH,
    ACTUAL_DELAY_DAYS
FROM PO_HISTORY PH
JOIN VENDOR_PERFORMANCE VP ON PH.VENDOR_ID = VP.VENDOR_ID
WHERE PH.VENDOR_ID = :targetVendor
   OR PH.MATERIAL_ID = :targetMaterial
ORDER BY ORDER_DATE DESC
LIMIT 200;
```

### C.3 Analytics Dashboard Queries

```sql
-- Weekly risk distribution
SELECT 
    RISK_TIER,
    COUNT(*) AS PO_COUNT,
    AVG(PREDICTED_DELAY) AS AVG_PREDICTED_DELAY
FROM RISK_PREDICTIONS
WHERE PREDICTED_AT >= ADD_DAYS(CURRENT_DATE, -7)
GROUP BY RISK_TIER;

-- Vendor risk ranking
SELECT 
    VENDOR_ID,
    COUNT(CASE WHEN RISK_TIER = 'Red' THEN 1 END) AS RED_COUNT,
    COUNT(CASE WHEN RISK_TIER = 'Amber' THEN 1 END) AS AMBER_COUNT,
    COUNT(*) AS TOTAL_POS
FROM RISK_PREDICTIONS RP
JOIN PO_HISTORY PH ON RP.PO_ID = PH.PO_ID
WHERE RP.PREDICTED_AT >= ADD_DAYS(CURRENT_DATE, -30)
GROUP BY VENDOR_ID
ORDER BY RED_COUNT DESC;

-- Mitigation effectiveness
SELECT 
    MP.RECOMMENDED_VENDOR,
    COUNT(*) AS PROPOSALS,
    SUM(CASE WHEN MP.STATUS = 'APPROVED' THEN 1 ELSE 0 END) AS APPROVED,
    AVG(CASE WHEN MP.STATUS = 'APPROVED' 
        THEN DATEDIFF(DAY, PH.PLANNED_DELIVERY_DATE, PH.ACTUAL_DELIVERY_DATE)
        END) AS AVG_ACTUAL_DELAY_AFTER_SWITCH
FROM MITIGATION_PROPOSALS MP
JOIN PO_HISTORY PH ON MP.PO_ID = PH.PO_ID
GROUP BY MP.RECOMMENDED_VENDOR;
```

---

## Scenario D: Go-Live Checklist and Operational Excellence

### D.1 Security Checklist

| Area | Requirement | BTP Implementation |
|------|-------------|-------------------|
| **API Authentication** | Only authorized callers | XSUAA + OAuth2 |
| **Credential Storage** | No secrets in code | BTP Credential Store |
| **Data Privacy** | PII handling compliance | Data masking, audit logs |
| **Network Isolation** | Restrict external access | Private Link, Cloud Connector |
| **Role-Based Access** | Separate viewer/approver | XSUAA scopes and roles |

### D.2 Observability Checklist

| Concern | Implementation |
|---------|---------------|
| **Structured Logging** | JSON logs to SAP Cloud Logging |
| **Prediction Audit** | Persist every prediction with context |
| **Agent Traces** | Store full reasoning trace |
| **Latency Monitoring** | Track model inference times |
| **Alerting** | SAP Alert Notification for anomalies |

### D.3 Model Operations (MLOps)

| Aspect | Approach |
|--------|----------|
| **Model Versioning** | Track SAP-RPT-1 deployment versions |
| **Accuracy Monitoring** | Compare predictions to actual outcomes |
| **Drift Detection** | Alert when vendor performance patterns change |
| **Retraining Trigger** | When accuracy drops below threshold |
| **A/B Testing** | Compare model versions before promotion |

### D.4 Recommended Go-Live Sequence

```text
Week 1:  Deploy CAP application with mock predictions
          Configure XSUAA, test authentication
          
Week 2:  Connect to SAP AI Core (SAP-RPT-1)
          Run parallel: live predictions vs historical outcomes
          
Week 3:  Add S/4HANA integration (read-only)
          Validate data freshness SLAs
          
Week 4:  Enable agent for mitigation proposals
          Configure human-in-the-loop workflow
          
Week 5:  Production pilot with limited PO subset
          Monitor accuracy and latency
          
Week 6:  Full rollout with all POs
          Enable automated alerting
          Stakeholder training
```

### D.5 Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Prediction Accuracy** | >80% within ±1 day | Compare predictions to actuals |
| **Risk Detection Rate** | >90% Red-tier caught | True positives / actual delays |
| **Mitigation Adoption** | >60% proposals approved | Approved / Generated |
| **Time to Detection** | <4 hours from PO creation | Event timestamp to alert |
| **Line-Down Avoidance** | Track avoided incidents | Mitigated POs that would have delayed |

---

## BTP Services Summary

| Service | Role in This Solution |
|---------|----------------------|
| **SAP AI Core** | Hosts SAP-RPT-1 model and LLM deployments |
| **SAP-RPT-1** | Tabular prediction for delay risk scoring |
| **Gen AI Hub** | LLM orchestration for agent reasoning |
| **CAP** | Application framework for APIs and UIs |
| **HANA Cloud** | Persistence for context, predictions, proposals |
| **Integration Suite** | Data pipeline from S/4HANA |
| **Event Mesh** | Real-time PO event distribution |
| **XSUAA** | Authentication and authorization |
| **Destination Service** | Secure connectivity to S/4HANA |
| **Build Process Automation** | Approval workflows |
| **Cloud Logging** | Centralized observability |
| **Alert Notification** | Proactive alerting |

---

## Final Guidance

### Start Simple, Add Complexity Incrementally

1. **Phase 1**: Batch scoring on historical data (prove model accuracy)
2. **Phase 2**: Event-driven scoring on new POs (prove operational value)
3. **Phase 3**: Agent-based mitigation (prove business impact)
4. **Phase 4**: Full automation with governance (scale)

### Key Architecture Principles

1. **AI is a capability, not the architecture** — Embed AI into existing CAP patterns
2. **Observability by design** — Log every prediction and agent step
3. **Human-in-the-loop for action** — Recommend, don't execute autonomously
4. **Data freshness matters** — Stale data → inaccurate predictions
5. **Start with the business outcome** — "$X avoided downtime" beats "N predictions made"

---

*This architecture guide is designed to support a 45-minute presenter-led discussion. Use the diagrams and tables as conversation anchors, and adapt the depth based on audience questions.*
