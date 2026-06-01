# 365 HR Copilot

A **Microsoft Copilot Studio agent for enterprise HR** (`365 HR Copilot`) that answers HR policy questions from a grounded knowledge base and automates leave requests with multi-stage **AI + human-in-the-loop** approvals — delivered inside Microsoft Teams.

## What this agent does

| Capability | Description |
|------------|-------------|
| 🔎 Answer HR questions | Answers HR policy questions **grounded** in the company knowledge base via **semantic / vector RAG**, with citations — no external or general knowledge. |
| 🏖️ Request leave | Collects start date, end date and reason in a guided card, validates the dates, computes **working days** (excluding weekends and public holidays), and saves the request. |
| ✅ Multi-stage approval (HITL) | Scores each request on duration and reason clarity, then auto-approves or escalates to the right manager — resolved live from the directory. |

Scope is limited to **HR policy** and **leave management**; off-topic requests are politely declined.

### How it works
1. The user signs in (**Entra ID**, integrated auth) and the orchestrator routes each turn by intent.
2. **HR policy question →** semantic RAG retrieves the most relevant passages and the agent answers **only** from them.
3. **Leave request →** a guided card collects the fields → dates are validated → working days are computed → the request is written to the database and routed for approval.

### Approval routing (human-in-the-loop)

| Duration / signal | Decision |
|-------------------|----------|
| Under 3 working days, clear reason | **AI auto-approves** |
| 3–15 working days, or ambiguous reason, or reason that doesn't match the duration | Escalate to **N+1 manager** |
| Over 15 working days | Escalate to **N+2 senior approver** |

Approvers are never hard-coded — they are resolved dynamically from the **Entra ID** directory at request time.

## Architecture

```
User → Copilot Studio (365 HR Copilot)
        ├─ Orchestration (routes by intent)
        ├─ HR policy questions (RAG) ──→ Azure AI Search  ←── Azure Blob (HR docs → AI Foundry embeddings)
        └─ Leave request ──→ Power Automate Agent Flow ──→ Azure SQL (leave + status)
                                                       ├─→ Public Holidays REST API (working-days calc)
                                                       └─→ Entra ID (approvers & RBAC)
```

See [`ressources/agent-schema.md`](ressources/agent-schema.md) for the full user-journey, agent-flow, technical-architecture, and approval-flow diagrams.

## Prerequisites

- **Microsoft Power Platform / Power Apps** → https://make.powerapps.com
- **Microsoft Copilot Studio** → https://copilotstudio.microsoft.com
- **Azure subscription** for the RAG pipeline and data:
  - **Azure Blob Storage** — source HR policy documents
  - **Azure AI Search** — semantic / vector index
  - **Azure AI Foundry** — embedding model
  - **Azure SQL Database** — leave requests and their status
- **Microsoft Entra ID** — authentication, RBAC, and approver lookup

## Build order

The agent, the Power Automate flows and the connection references are packaged together as a **solution** (note the `lkar_` publisher prefix), so it can be promoted across environments via a **Power Platform pipeline**. Build in this order:

1. **Power Apps** (https://make.powerapps.com) → develop in the **Dev** environment *(Dataverse-backed).*
2. Provision the Azure resources (Blob, AI Search, AI Foundry, Azure SQL) and index the HR documents.
3. Create a **solution** in Dev, then build the **agent** (+ leave flows and knowledge source) inside it.
4. Use a **pipeline** to deploy the solution from Dev → UAT → Prod.

### Environment topology

| Environment | Type |
|-------------|------|
| Host (pipeline host) | Production (Managed) |
| Dev | Sandbox (Managed) |
| UAT | Sandbox (Managed) |
| Prod | Production (Managed) |

## Knowledge & RAG setup

HR policy documents in **Azure Blob Storage** are chunked, embedded with an **Azure AI Foundry** model, and indexed for **semantic / vector** retrieval in **Azure AI Search**. Model knowledge is disabled, so the agent answers strictly from the indexed documents.

For the full step-by-step index configuration, see:

📄 [`ressources/Azure AI Search Complete Setup Guide.pdf`](ressources/Azure%20AI%20Search%20Complete%20Setup%20Guide.pdf)

## Telemetry & analytics

The agent emits **Application Insights** events to measure self-service rate, AI auto-approval rate, and approval turnaround — surfaced in a **Power BI** KPI dashboard (HR hours saved, deflection rate).

## Security

- **Entra ID** sign-in with **RBAC** enforced at the agent level — authorized Microsoft 365 employees only.
- Each employee sees and manages only their own leave.
- Answers are traceable to an approved source; no open-web search, no model guessing.
