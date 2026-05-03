# AI-Based Processing Platform (AIBP)
## Operations Blueprint — Preamble

| Field | Value |
|---|---|
| **Version** | 0.1 (Draft — In Progress) |
| **Status** | Work in Progress |
| **Owner** | Operations Team |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

## Purpose & Scope

This blueprint defines the operational design of the AI-Based Processing (AIBP) platform — a greenfield agentic system that processes taxpayer emails end-to-end using AI agents. It is the authoritative reference for how the operations function will instrument, monitor, govern, and sustain the platform in production.

This document covers the full email lifecycle from ingestion at SWEE through to agent execution and resolution. It is scoped strictly to the **Operations** domain. Adjacent domains — App & Data, Platform & Infrastructure, Security & Identity, Governance & Risk, and People & Process — are referenced where necessary but not owned by this document.

**What this document covers:**

- Operational instrumentation at each platform layer
- Deployment and version management practices
- Evaluation, quality assurance, and continuous feedback loops
- Human-in-the-Loop (HITL) workflows and incident response
- Cost governance (FinOps)
- Service Level Agreements (SLAs)
- Operating model: team roles, runbooks, and escalation playbooks

**What this document does not cover:**

- Application architecture design (owned by App & Data)
- Network topology and infrastructure provisioning (owned by Platform & Infra)
- Identity federation and access policy definitions (owned by Security & Identity)
- Data schema design and LLM prompt engineering (owned by App & Data)

---

## Platform Architecture Overview

The AIBP platform processes taxpayer emails through four sequential layers. Operations touches every layer.

```
Taxpayer Email
      │
      ▼
┌─────────────────────────────────────┐
│  SWEE                               │
│  (Azure App Service, GCC 2.0)       │
│  • Receive inbound email            │
│  • Anonymise / PII-strip            │
│  • Classify to SOP via vector search│
│  • Enqueue to Service Bus           │
└───────────────────┬─────────────────┘
                    │  Azure Service Bus (Premium)
                    ▼
┌─────────────────────────────────────┐
│  Agentic Orchestration (AO)         │
│  (Azure Container Apps, LangGraph)  │
│  • Dequeue and parse intent         │
│  • Execute agent workflow           │
│  • Call tools via AIGP              │
│  One container app per agent        │
└───────────────────┬─────────────────┘
                    │  AIGP API (ForgeRock AuthZ + Entra ID AuthN)
                    ▼
┌─────────────────────────────────────┐
│  AI Governance Platform (AIGP)      │
│  • Policy enforcement (OPA)         │
│  • HITL gating for high-risk actions│
│  • Audit trail emission             │
└───────────────────┬─────────────────┘
                    │  Kafka (AKS-hosted)
                    ▼
┌─────────────────────────────────────┐
│  Internal Microservices / DB        │
│  (AKS, Kafka, ForgeRock-protected)  │
└─────────────────────────────────────┘
```

> **Note on LangGraph**: The AO layer is expected to use LangGraph as the agent orchestration framework. This is the current recommendation by the App & Data team and is subject to architecture review. Sections of this blueprint that reference LangGraph-specific instrumentation (e.g., OTel trace spans, graph-node-level metrics) will require review if an alternative framework is selected.

> **Note on SWEE Hosting**: SWEE is currently hosted on Azure App Service. A migration to Azure Container Apps is under consideration. If this occurs, the Service Bus integration design in Step 1 remains unchanged; only the compute hosting changes.

---

## Operational Principles

The following principles govern all design decisions in this blueprint:

1. **Observability-first**: No agent may be deployed to production without emitting structured telemetry. If it cannot be observed, it cannot be operated.

2. **Fail safe, not silent**: A failed or uncertain agent action must escalate to a human. An agent that silently succeeds on the wrong action is worse than one that fails loudly.

3. **Data minimisation in ops tooling**: Operational tooling (dashboards, logs, traces) must handle only anonymised or metadata-level data. PII must not appear in any ops pipeline, log stream, or dashboard. This is both a PDPA obligation and a SOC data-sharing constraint.

4. **Policy as code**: Operational guardrails (rate limits, kill switches, risk thresholds) are defined as code and version-controlled. Manual configuration drift is a reliability and audit risk.

5. **Separation of concerns across teams**: The AO team owns agent logic; the AIGP team owns governance enforcement; the Ops team owns production health. No single team has unilateral access to all layers in production.

6. **Cost accountability per resolution**: Every email processed must carry a measurable cost. Token spend, compute time, and human HITL time are attributed to individual email threads.

---

## Team Ownership Map

| Layer | Owning Team | Ops Interface |
|---|---|---|
| SWEE (ingestion + triage) | App & Data (SWEE squad) | Ops monitors queue depth, triage accuracy metrics, DLQ |
| AO (agent execution) | App & Data / AO team | Ops monitors agent health, version registry, eval scores |
| AIGP (governance + policy) | AIGP team | Ops monitors HITL queue depth, kill switch status, policy violations |
| Internal Microservices | Platform & Infra | Ops receives error signals via Kafka consumer group metrics |
| Security monitoring | Internal SOC + GovTech SOC | Ops streams anonymised application logs (no PII, no email content) |
| Tax Officers | Business / Operations | Perform HITL reviews; provide accuracy ground-truth feedback |

> **SOC Log-Sharing Constraint**: Logs forwarded to Internal SOC and GovTech SOC contain application telemetry only — request metadata, error codes, latency metrics, and correlation IDs. Email content, taxpayer identifiers, and agent reasoning traces are **never** included in SOC-bound log streams. This constraint applies to all steps in this blueprint without exception.

---

## Compliance Framework Alignment

This blueprint is designed to comply with the following frameworks. Relevant compliance callouts are tagged throughout each step document.

| Framework | Relevance to Operations |
|---|---|
| **PDPA (Singapore)** | Data minimisation in all ops tooling; retention periods; PII handling in logs and traces |
| **IM8 (Singapore Government ICT&SS Management)** | Data classification requirements, audit log retention (minimum 5 years), incident response timelines, penetration testing obligations |
| **ISO 27001** | Change management procedures, access control for ops tooling, incident management lifecycle, business continuity controls |

---

## How to Read This Document

This blueprint is structured as a series of step documents, each corresponding to a layer or operational domain in the platform processing flow. The steps follow the sequence of the email journey through the platform.

| Step Document | Coverage |
|---|---|
| `aibp-ops-step1.md` | Ingestion & Triage Operations (SWEE layer) |
| `aibp-ops-step2.md` | Agentic Orchestration Operations (AO layer) |
| `aibp-ops-step3.md` | Governance & Control Operations (AIGP layer) |
| `aibp-ops-step4.md` | Platform Reliability Operations |
| `aibp-ops-step5.md` | FinOps — Token & Cost Management |
| `aibp-ops-step6.md` | Operational Feedback & Automated Self-Reflection |
| `aibp-ops-step7.md` | People, Process & Governance Operations |

This document serves a mixed audience. Use the following guide:

| Reader | Recommended reading |
|---|---|
| **Operations engineers** | All step documents; focus on implementation detail and technology choices |
| **Technical architects** | Preamble (this document), all "Option" sub-sections, and compliance callouts |
| **Management / Executives** | Preamble, section headers, recommendation summaries, SLA (Step 4), FinOps (Step 5) |
| **Auditors / Governance reviewers** | Preamble compliance table, Step 2 (Behavioral Auditing), Step 3 (AIGP Ops, RiskOps), Step 7 (People & Process) |

Each sub-step within the step documents follows a consistent structure:
1. **Implementation Overview** — what this component does and why it matters operationally
2. **Option N (Recommended / Alternative)** — specific technology stack, implementation detail, pros, cons
3. **Recommendation Justification** — rationale for the chosen option in the context of this platform
4. **Compliance Notes** — PDPA, IM8, ISO 27001 callouts where applicable
