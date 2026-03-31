# Predictive JIT Supply Chain Risk with SAP BTP

A hands-on workshop demonstrating how to predict supplier delivery delays using SAP AI Core (SAP-RPT-1) and build agentic mitigation workflows with SAP Gen AI Hub.

## Workshop Overview

**Duration:** 5 hours (including breaks)

| Part | Duration | Description |
|------|----------|-------------|
| Presentation | 1 hour | Use case, SAP-RPT-1 model, code-based agents |
| Part 1 Hands-On | ~1 hour | Predictive delay scoring with SAP-RPT-1 |
| Part 2 Hands-On | ~1 hour | Agentic mitigation with ReAct pattern |
| Architecture Discussion | 45 min | Production deployment on SAP BTP |
| Survey & Wrap-up | 15 min | Feedback and next steps |

## Business Scenario

**BestRun Technologies** operates a Just-In-Time (JIT) manufacturing model. A delayed shipment of a critical component creates a "line-down" situation costing $15,000+ per hour.

This workshop shows how to:
1. **Predict** which purchase orders are at risk of delay
2. **Classify** risk into actionable tiers (Green/Amber/Red)
3. **Automate** mitigation proposals using AI agents
4. **Govern** decisions with human-in-the-loop approval

## Repository Contents

```
├── part1_jit_prediction.ipynb    # Hands-on: SAP-RPT-1 prediction
├── part2_jit_agent.ipynb         # Hands-on: Agentic mitigation
├── architecture_discussion.md    # Production architecture guide
└── data/
    ├── historical_po_data.csv    # 600 historical POs with outcomes
    ├── new_po_prediction.csv     # Sample PO for prediction
    └── alt_supplier_table.csv    # Alternative supplier options
```

## Prerequisites

- SAP BTP subaccount with SAP AI Core
- SAP AI Launchpad access
- Python 3.9+
- Jupyter Notebook environment (e.g., SAP Business Application Studio)

## Setup

1. Clone this repository
2. Create a `.env` file with your SAP AI Core credentials:

```env
AICORE_AUTH_URL=https://...
AICORE_CLIENT_ID=...
AICORE_CLIENT_SECRET=...
AICORE_BASE_URL=https://...
AICORE_RESOURCE_GROUP=...
RPT1_DEPLOYMENT_URL=https://...      # Part 1
ORCH_DEPLOYMENT_URL=https://...      # Part 2
```

3. Open the notebooks and follow the step-by-step instructions

## SAP BTP Services Used

| Service | Purpose |
|---------|---------|
| SAP AI Core | Model hosting and inference runtime |
| SAP-RPT-1 | Tabular prediction for delay risk |
| Gen AI Hub | LLM orchestration (Claude 4.5 Sonnet) |
| Integration Suite | S/4HANA data connectivity (architecture) |

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.
