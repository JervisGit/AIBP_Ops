# AI-Based Processing Platform (AIBP)
## Operations Blueprint — Master Index

| Field | Value |
|---|---|
| **Version** | 0.1 (Draft — In Progress) |
| **Status** | Work in Progress |
| **Owner** | Operations Team |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

> This file is the master index. Each section of the blueprint lives in its own document listed below. Refer to `aibp-ops-preamble.md` for platform architecture, operational principles, team ownership, and compliance framework alignment.

---

## Document Map

| Document | Status | Coverage |
|---|---|---|
| `aibp-ops-preamble.md` | Draft | Platform overview, operational principles, team ownership, compliance alignment, reading guide |
| `aibp-ops-step1.md` | Draft | Step 1 — Ingestion & Triage Operations (SWEE layer) |
| `aibp-ops-step2.md` | Draft | Step 2 — Agentic Orchestration Operations (AO layer) |
| `aibp-ops-step3.md` | Draft | Step 3 — Governance & Control Operations (AIGP layer) |
| `aibp-ops-step4.md` | Draft | Step 4 — Platform Reliability Operations |
| `aibp-ops-step5.md` | Draft | Step 5 — FinOps: Token & Cost Management |
| `aibp-ops-step6.md` | Draft | Step 6 — Operational Feedback & Automated Self-Reflection |
| `aibp-ops-step7.md` | Draft | Step 7 — People, Process & Governance Operations |

---

## High-Level Blueprint Summary

The AIBP Operations Blueprint follows the email journey through the platform, applying operational design at each layer:

```
Email → SWEE → AO → AIGP → Internal DB/Microservices
         │       │     │
      Step 1  Step 2  Step 3
                │
          Steps 4–7 (cross-cutting operational concerns)
```

### Step 1 — Ingestion & Triage Operations (SWEE)
- **1.1** Email queue management: Azure Service Bus Premium with session ordering, DLQ automation
- **1.2** EvalOps Tier 0: Real-time triage confidence monitoring + embedding drift detection + weekly stratified officer audit

### Step 2 — Agentic Orchestration Operations (AO)
- **2.1** Agent Fleet & Version Registry: SemVer, signed container images, PostgreSQL registry DB
- **2.2** Deployment Lifecycle: DEV (local) → SIT → UAT → ORT → PROD via Azure DevOps multi-stage pipelines; AIGP MCP server tool call validation in ORT
- **2.3** Testing Strategy: Blue-Green + Canary via ACA traffic weight splitting
- **2.4** EvalOps: Azure AI Evaluation SDK + Langfuse (self-hosted) + LLM-as-a-Judge
- **2.5** Observability & Distributed Tracing: OpenTelemetry + OpenInference → Azure Monitor (App Insights, 90-day raw) → masking pipeline → Non-PII LAW → Langfuse adapter
- **2.6** Behavioral Auditing & Traceability: Append-only CosmosDB audit ledger; SOC integration via Non-PII LAW (replacing Event Hubs)
- **2.7** Behavioral Anomaly Detection: Embedding-based behavioral baseline comparison

### Step 3 — Governance & Control Operations (AIGP)
- **3.1** HITL Management: Risk-based triggers, HITL queue, officer review workflow
- **3.2** Incident Response & Emergency Stop: Kill switch hierarchy, automated runbook
- **3.3** RiskOps: Policy-as-code (OPA), Azure Defender, PDPA/IM8/ISO 27001 posture

### Step 4 — Platform Reliability Operations
- **4.1** ResiliencyOps: Circuit breakers, DLQ fallback, Azure OpenAI PTU sizing (frontier model)
- **4.2** SLA Management: KPI definitions, SLA framework alignment (Google SRE / ISO 42001 / NIST AI RMF / IMDA), Azure Monitor Workbooks
- **4.3** Model Lifecycle Management: Deprecation tracking, model migration pipeline, PTU contract review, fallback model configuration
- **4.4** External Dependency Health Monitoring: Azure Service Health alerts, quota headroom monitoring, AIGP API synthetic probe

### Step 5 — FinOps
- **5.1** Token & cost monitoring per agent and per email
- **5.2** Cost comparison model: agent-resolved vs. human-officer-resolved

### Step 6 — Operational Feedback & Self-Reflection
- **6.1** Structured feedback loop: HITL officer corrections → Langfuse evaluation dataset
- **6.2** Automated post-mortem pipeline: event-driven trace analysis after HITL interventions

### Step 7 — People, Process & Governance Operations
- **7.1** Roles & responsibilities across teams
- **7.2** Runbooks & escalation playbooks

---

*The content below this line is the original draft from Google AI and is superseded by the individual step documents above. It is retained here for reference during the drafting phase and will be removed in a later version.*

---

## Original Draft (Reference Only)

### Purpose & Scope

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

### Platform Architecture Overview

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
│  • Classify to SOP                  │
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

> **Note on LangGraph**: The AO layer is expected to use LangGraph as the agent orchestration framework. This is the current recommendation by the App & Data team and is subject to architecture review. Sections of this blueprint that reference LangGraph-specific instrumentation (e.g., OTel instrumentation, trace spans) will require review if an alternative framework is selected.

> **Note on SWEE**: SWEE is currently hosted on Azure App Service. A migration to Azure Container Apps is under consideration. If this occurs, the Service Bus integration design in Step 1.1 remains unchanged; only the compute hosting changes.

---

### Operational Principles

The following principles govern all design decisions in this blueprint:

1. **Observability-first**: No agent may be deployed to production without emitting structured telemetry. If it cannot be observed, it cannot be operated.

2. **Fail safe, not silent**: A failed or uncertain agent action must escalate to a human. An agent that silently succeeds on the wrong action is worse than one that fails loudly.

3. **Data minimisation in ops tooling**: Operational tooling (dashboards, logs, traces) must handle only anonymised or metadata-level data. PII must not appear in any ops pipeline, log stream, or dashboard. This is both a PDPA obligation and a SOC data-sharing constraint.

4. **Policy as code**: Operational guardrails (rate limits, kill switches, risk thresholds) are defined as code and version-controlled. Manual configuration drift is a reliability and audit risk.

5. **Separation of concerns across teams**: The AO team owns agent logic; the AIGP team owns governance enforcement; the Ops team owns production health. No single team has unilateral access to all layers in production.

6. **Cost accountability per resolution**: Every email processed must carry a measurable cost. Token spend, compute time, and human HITL time are attributed to individual email threads.

---

### Team Ownership Map

| Layer | Owning Team | Ops Interface |
|---|---|---|
| SWEE (ingestion + triage) | App & Data (SWEE squad) | Ops monitors queue depth, triage accuracy, DLQ |
| AO (agent execution) | App & Data / AO team | Ops monitors agent health, version registry, eval scores |
| AIGP (governance + policy) | AIGP team | Ops monitors HITL queue, kill switch status, policy violations |
| Internal Microservices | Platform & Infra | Ops receives error signals via Kafka events |
| Security monitoring | Internal SOC + GovTech SOC | Ops streams anonymised application logs (no PII, no email content) |
| Tax Officers | Business / Operations | Perform HITL reviews; provide accuracy feedback |

> **SOC Log-Sharing Constraint**: Logs forwarded to Internal SOC and GovTech SOC contain application telemetry only — request metadata, error codes, latency metrics, and correlation IDs. Email content, taxpayer identifiers, and agent reasoning traces are **never** included in SOC-bound log streams.

---

### Compliance Framework Alignment

This blueprint is designed to comply with the following frameworks. Relevant compliance callouts are tagged throughout each section.

| Framework | Relevance |
|---|---|
| **PDPA (Singapore)** | Data minimisation in all ops tooling; retention periods; PII handling in logs |
| **IM8 (Singapore Government ICT&SS Management)** | Data classification, audit log retention (minimum 5 years for government systems), incident response timelines |
| **ISO 27001** | Change management, access control for ops tooling, incident management, business continuity |

---

### How to Read This Document

This document serves a mixed audience. Use the following guide:

| Reader | Recommended sections |
|---|---|
| **Operations engineers** | All sections; focus on implementation detail and technology choices |
| **Technical architects** | Preamble, all "Option" sub-sections, and compliance callouts |
| **Management / Executives** | Preamble, section headers, recommendation summaries, SLA (Step 4.2), FinOps (Step 5) |
| **Auditors / Governance reviewers** | Preamble compliance table, Step 2.6 (Audit Trail), Step 3 (AIGP Ops), Step 3.3 (RiskOps), Step 7 (People & Process) |

Each sub-step follows a consistent structure:
1. **Implementation Overview** — what this component does and why it matters
2. **Option N (Recommended / Alternative)** — specific technology, implementation detail, pros, cons
3. **Recommendation Justification** — rationale for the chosen option
4. **Compliance Notes** — where applicable

---

## Step 1: Ingestion & Triage Operations (SWEE Layer)

**Operational Focus**: Ensuring the platform entry point is resilient, observable, and that every email is correctly classified before committing compute resources downstream.

**SWEE Context**: SWEE is a greenfield AI-powered email triage system. It is the sole entry point for taxpayer emails into the AIBP platform. Its primary operational responsibilities are:
1. Receiving and authenticating inbound emails
2. Applying PII anonymisation before any AI processing
3. Classifying the email to the appropriate SOP
4. Enqueuing the classified payload reliably for the AO layer

Because this is a fully greenfield system, there is no legacy baseline for triage accuracy. All monitoring frameworks must be designed to establish baselines organically from Day 1 of production operation.

---

### 1.1 Email Ingestion & Queue Management

#### Implementation Overview

Before SWEE routes an email to the AO layer, it must deposit the message into an intermediary queue. This queue serves four operational purposes:

1. **Durability**: If the AO layer is temporarily unavailable (e.g., during a deployment, throttling event, or outage), emails are not lost.
2. **Backpressure**: The queue absorbs burst traffic and prevents the AO layer from being overwhelmed, protecting Azure OpenAI quota.
3. **Replayability**: Failed processings can be retried up to a configured maximum. Messages that exhaust retries are moved to a Dead Letter Queue (DLQ) for investigation and manual recovery.
4. **Auditability**: Each message in the queue carries a unique `MessageId` assigned by SWEE that becomes the root correlation ID for all downstream distributed tracing.

The queue layer sits between SWEE and AO. SWEE is the producer; AO container apps are consumers subscribed to their respective SOP topics.

---

#### Option 1 (Recommended): Azure Service Bus Premium + APIM Rate Limiting + Automated DLQ Runbook

**Technology Stack**: Azure Service Bus (Premium tier), Azure API Management, Azure Monitor, Azure Automation

**Implementation Detail**:

**Azure Service Bus — Premium Tier**

Azure Service Bus Premium is selected over Standard for four specific requirements:

- **Private Endpoint support**: The Standard tier does not support VNet integration or private endpoints. The Premium tier can be exposed exclusively over a private endpoint within the GCC 2.0 Virtual Network, ensuring email metadata never traverses the public internet between SWEE and the queue.
- **Message Sessions**: Enables FIFO ordering within a session boundary. SWEE sets `SessionId` = SHA-256 hash of the taxpayer's email address. This guarantees that multiple emails from the same taxpayer are processed sequentially, preventing race conditions when agents attempt to update the same taxpayer record.
- **Geo-Disaster Recovery**: Premium supports namespace pairing to a secondary Azure region. In the event of a regional outage, the secondary namespace is activated with an RPO of approximately 0 (no message loss) and an RTO of under 10 minutes.
- **4 GB per messaging unit (scalable)**: At <1,000 emails/day, a single messaging unit is sufficient. Capacity can be scaled in-place without re-provisioning.

**Queue and Topic Design**:

| Parameter | Value | Rationale |
|---|---|---|
| Entity type | Topic + Subscriptions | Allows fan-out to multiple SOP-specific consumers |
| Topic name | `email-triage-{env}` (e.g., `email-triage-prod`) | Environment-isolated |
| Subscriptions | One per SOP category (e.g., `sop-refund`, `sop-query`, `sop-dispute`) | Each AO agent container subscribes to one SOP |
| Message TTL | 7 days | Messages not consumed within 7 days are dead-lettered |
| Max delivery count | 3 | After 3 failed deliveries, message moves to DLQ |
| Session-based ordering | Enabled | `SessionId` = SHA-256(taxpayer email address) |
| Message lock duration | 5 minutes | Sufficient for AO agent processing; extendable via lock renewal |

**Message Payload Schema**:

```json
{
  "messageId": "uuid-v4",
  "correlationId": "uuid-v4",
  "emailBlobRef": "https://<storage>.blob.core.windows.net/<container>/<encrypted-blob-id>",
  "sopId": "SOP-REFUND-001",
  "triageCategory": "refund",
  "triageConfidenceScore": 0.94,
  "sweeVersion": "1.2.0",
  "timestamp": "2026-05-03T08:00:00Z",
  "sessionId": "sha256-hash-of-taxpayer-email"
}
```

> **PII Control**: The email body and taxpayer identifiers are **not** stored in the message payload. SWEE stores the anonymised email as an encrypted blob in Azure Blob Storage (customer-managed key via Azure Key Vault). The message payload carries only a blob reference. AO fetches the encrypted blob during processing using its managed identity. Raw email content never traverses Service Bus.

**Azure API Management (APIM) — Rate Limiting**:

SWEE triggers AO processing by publishing to Service Bus. To protect the AO layer from Azure OpenAI quota exhaustion during email bursts, SWEE places its Service Bus publish calls behind an APIM policy layer. The APIM policy enforces:

- **Rate limiting**: `rate-limit-by-key` with key = subscription ID, limit = 100 messages/minute. When the limit is exceeded, SWEE holds messages in the queue (natural backpressure) rather than dropping them.
- **JWT validation**: All calls from SWEE to downstream APIs are validated against an Entra ID-issued JWT token. SWEE uses its Azure Managed Identity to obtain the token; no credentials are stored in application configuration.
- **Retry policy header propagation**: APIM injects `x-correlation-id` headers from the Service Bus `MessageId` to ensure end-to-end trace continuity.

**Dead Letter Queue (DLQ) Automation**:

DLQ messages represent emails that the platform has failed to process. Each DLQ message must be investigated and either reprocessed or escalated to a human. The following automation handles this:

1. Azure Monitor **metric alert** fires when DLQ message count > 5 (configurable threshold).
2. Alert triggers an **Azure Automation Runbook** (`ops-dlq-recovery.ps1`) that:
   - Reads up to 50 DLQ messages per execution
   - Classifies each message by failure reason (using `DeadLetterReason` and `DeadLetterErrorDescription` properties)
   - For retryable failures (e.g., transient AO timeout): re-enqueues the message to the main topic
   - For non-retryable failures (e.g., schema validation error, AO rejection): writes the message metadata to an Azure Table Storage `ops-dlq-log` table and posts an alert to the Ops team via Azure Monitor Action Group (email + webhook)
3. DLQ messages older than 24 hours that remain unprocessed trigger a **P2 incident ticket**.

**Pros**:
- GCC 2.0-compliant: fully accessible over private endpoints, no public SaaS dependency
- Session-based FIFO ordering prevents taxpayer record corruption from concurrent agent writes
- Automated DLQ recovery reduces operational toil for <1,000 emails/day volumes
- APIM provides a single, auditable governance choke point for all AO access

**Cons**:
- Premium tier costs approximately SGD 680/month per namespace (vs SGD 10 for Standard); justified by Private Endpoint and session requirements
- APIM adds approximately 5–10 ms latency per downstream call
- DLQ runbook requires ongoing maintenance as failure patterns evolve

---

#### Option 2: Azure Queue Storage + Static Application-Level Throttle

**Technology Stack**: Azure Storage Queues, SWEE application-level semaphore

**Implementation**: SWEE writes email metadata directly to Azure Queue Storage. A static concurrency semaphore in the SWEE application code limits throughput to 50 emails/minute. AO polls the queue on a timer.

**Pros**:
- Significantly cheaper: ~SGD 0.0005 per 10,000 operations
- Simpler to set up; no additional Azure resources required

**Cons**:
- No Private Endpoint support on Standard tier — fails GCC 2.0 network isolation requirements without significant workarounds
- Message size limit of 64 KB; email metadata with blob references may approach this limit
- No message session support — email thread ordering not guaranteed; taxpayer record race conditions are possible
- No native geo-replication; messages are at risk during regional failures
- Static throttle requires application redeployment to adjust limits; cannot adapt dynamically to AO quota availability

---

#### Option 3: Azure Event Hubs + Event-Driven Consumer Groups

**Technology Stack**: Azure Event Hubs (Standard or Premium), LangGraph consumer clients in ACA

**Implementation**: SWEE publishes email-triage events to an Event Hubs namespace. AO agents consume from partition-assigned consumer groups (one per SOP). Event Hub's 7-day retention enables event replay.

**Pros**:
- High-throughput ceiling (millions of events/day) — headroom if email volumes scale significantly
- Native event replay via offset management
- Private Endpoint supported on Premium tier
- Excellent integration with Azure Stream Analytics for real-time anomaly detection

**Cons**:
- Event Hubs uses a streaming model, not a task queue — there is no concept of "completing" a message, no native DLQ semantics, and no delivery count tracking. Failed processings require custom dead-letter logic to be built from scratch.
- Consumer offset management per SOP partition adds operational complexity disproportionate to <1,000 emails/day volumes
- No message session support for FIFO ordering within a taxpayer thread

---

#### Recommendation Justification

**Option 1** is recommended. At <1,000 emails/day volume, throughput is not the primary concern — correctness, durability, and compliance are. Azure Service Bus Premium is the only option that satisfies all three: private endpoint for GCC 2.0 network isolation, message sessions for taxpayer FIFO ordering, and native DLQ semantics for failed-message recovery. The cost premium is justified by the elimination of two critical operational risks: email loss during AO outages and taxpayer record corruption from out-of-order processing.

> **Compliance Note (IM8)**: All messages in the Service Bus queue must be treated as government data. The queue namespace must be tagged with the appropriate IM8 data classification label. Access to the queue must be controlled via Azure RBAC (Managed Identities only; no shared access signature keys in application code).

---

### 1.2 Intent Triage Accuracy Monitoring (EvalOps — Tier 0)

#### Implementation Overview

SWEE's SOP classification is the first — and most consequential — AI decision in the pipeline. A misclassified email routed to the wrong SOP will cause the AO agent to execute an incorrect workflow, potentially performing wrong or harmful actions on taxpayer records. Downstream cost is high: a misrouted email consumes agent tokens, occupies HITL review time, and must be reprocessed from scratch.

Because SWEE is greenfield, there is no historical accuracy baseline. Tier 0 EvalOps must simultaneously (a) measure accuracy in real time and (b) construct the ground-truth dataset that future evaluation will be benchmarked against.

**Key Metrics**:

| Metric | Definition | Target (Proposed) |
|---|---|---|
| Triage Accuracy Rate | % emails sent to the correct SOP (HITL-verified ground truth) | ≥ 95% within 90 days |
| Low-Confidence Rate | % emails with SWEE confidence score < 0.80 | < 10% |
| Critic-SWEE Agreement Rate | % cases where shadow critic agrees with SWEE's label | ≥ 90% |
| Misroute Rate | % emails re-classified after AO rejection or HITL correction | < 5% |

---

#### Option 1 (Recommended): Asynchronous Shadow Scoring via GPT-4o-mini Critic + Self-Hosted Langfuse

**Technology Stack**: Azure OpenAI (GPT-4o-mini), Azure Container Apps Job, Langfuse (self-hosted on ACA), Azure AI Language (PII detection), Azure Monitor

**Implementation Detail**:

**Critic Pipeline**:

The shadow critic operates asynchronously — it does not block or delay the main email processing path.

1. SWEE publishes the triage decision to Service Bus (as per Step 1.1). The message payload includes the `sopId`, `triageCategory`, and `triageConfidenceScore`.
2. A separate **Critic ACA Job** (scheduled every 5 minutes, runs as a container job rather than a persistent app) reads messages from a **secondary Service Bus subscription** (`sub-critic`) that receives all messages on the same topic in parallel to the main AO subscriptions.
3. Before invoking the critic model, the Critic Job calls **Azure AI Language** (PII entity recognition) on the email subject line and a truncated 500-character excerpt of the email body. Named entities (names, NRIC, phone numbers, email addresses) are replaced with `[REDACTED]`.
4. The anonymised excerpt is submitted to **Azure OpenAI (GPT-4o-mini)** with the following prompt structure:

```
System: You are an expert email classifier for a government tax authority.
        You classify taxpayer emails into the following categories:
        [LIST OF SOP CATEGORIES WITH BRIEF DESCRIPTIONS]

User: The following email has been classified as: "{triageCategory}" with confidence {triageConfidenceScore}.
      Email excerpt (PII removed): "{anonymisedExcerpt}"

      Do you agree with this classification?
      Respond in JSON: { "agreement": true/false, "suggestedCategory": "...", "confidence": 0.00-1.00, "reasoning": "..." }
```

5. The critic response is written to **Langfuse** (self-hosted on ACA) as a trace with the following attributes:
   - `messageId` (correlation ID from Service Bus)
   - `swee_label` (SWEE's triage category)
   - `critic_label` (critic's suggested category)
   - `agreement` (boolean)
   - `critic_confidence` (float)
   - `swee_confidence` (float from SWEE)
   - `sweeVersion`, `criticModelVersion`

6. Langfuse aggregates these traces into daily accuracy reports. The Langfuse → OTel exporter emits custom metrics to **Azure Monitor / Application Insights**. Alert rules fire when:
   - Rolling 1-hour critic-SWEE agreement rate drops below 85%
   - 5 consecutive low-confidence triage decisions (SWEE confidence < 0.70) are detected within any 30-minute window
   - Daily Triage Accuracy Rate (from HITL-verified cases) drops below 93%

**Greenfield Baseline Protocol (First 30 Days)**:

Because no historical baseline exists at launch, the following supplementary protocol runs for the first 30 days of production:

- A **10% random sample** of all triage decisions — regardless of confidence score — is flagged in the HITL queue (see Step 3.1) for tax officer ground-truth labelling.
- Officer-labelled cases are written back to Langfuse as `score` annotations on the trace, creating the first ground-truth evaluation dataset.
- After 30 days, this dataset is used to:
  1. Calibrate critic-agreement vs. human-accuracy correlation
  2. Identify systematic misclassification patterns (e.g., a specific SOP that SWEE consistently confuses with another)
  3. Establish the operational baselines used in the SLA metrics (Step 4.2)
- The 10% HITL sampling rate is then reduced to 3% as confidence in the system grows.

**Langfuse Deployment Architecture**:

Langfuse is deployed as a self-hosted instance on Azure Container Apps (two container apps: `langfuse-web` and `langfuse-worker`) backed by Azure Database for PostgreSQL Flexible Server. This deployment is fully within GCC 2.0 and accesses no external SaaS endpoints. All trace data remains within the government cloud boundary.

**Pros**:
- Catches misrouting before AO expends tokens on the wrong workflow, reducing cost and latency impact
- Fully asynchronous — zero added latency to email processing
- Self-hosted Langfuse satisfies GCC 2.0 data residency and no-SaaS-exfiltration requirements
- Greenfield baseline protocol builds ground truth organically from real production traffic
- GPT-4o-mini is significantly cheaper than GPT-4o for a binary classification critic task

**Cons**:
- Adds approximately SGD 0.001–0.002 per email in critic model token cost
- Requires Langfuse ACA deployment and PostgreSQL instance (~SGD 150/month ongoing)
- The critic model has its own error rate — it is a probabilistic validation layer, not a deterministic oracle. Human ground-truth seeding in the first 30 days mitigates this.
- Secondary Service Bus subscription adds a small per-message fee

---

#### Option 2: Weekly Batch Ground-Truth Audit by Tax Officers

**Technology Stack**: Azure SQL Database (or SharePoint list), Azure Logic Apps for sampling and export, manual officer review

**Implementation**: A weekly sampling job (Azure Logic Apps) exports 5% of that week's triage decisions (anonymised) to a review list. Tax officers label each sample with the correct SOP. Results are compared against SWEE's labels in a weekly accuracy report.

**Pros**:
- No additional AI infrastructure required
- Officers build familiarity with edge cases and SWEE failure modes through the review process
- Audit-friendly: human-verified ground truth with named officer attribution

**Cons**:
- Reactive — accuracy degradation can compound for up to 7 days before detection
- Officer workload is non-trivial, especially during launch; conflicts with primary casework duties
- No alerting on real-time accuracy drops; trends only emerge in the weekly report
- Does not scale if email volume increases beyond <1,000/day

---

#### Option 3: Deterministic Keyword Assertion Layer (Supplementary)

**Technology Stack**: Azure Logic Apps or SWEE application middleware

**Implementation**: A rule engine executes keyword/regex checks on the email subject and first paragraph immediately after SWEE's triage decision. If keyword signals strongly associated with a different SOP category are detected, the triage is flagged as a potential misroute.

**Pros**:
- Effectively zero cost; runs in-process within SWEE
- Deterministic and interpretable — clear audit trail for why a flag was raised
- Zero latency impact (synchronous, lightweight)
- Catches the most obvious misclassifications instantly (e.g., the word "refund" in an email classified under `sop-query`)

**Cons**:
- Cannot detect semantic misclassifications (e.g., an email about "account closure" that is contextually a refund request written in indirect language)
- Rule maintenance becomes a growing burden as SOPs evolve or taxpayer language patterns shift
- High false positive rate for ambiguous emails, which can desensitise operators to flags over time
- Does not generate accuracy metrics or evaluate the overall triage quality

---

#### Recommendation Justification

**Option 1** is recommended as the primary continuous monitoring mechanism. **Option 3 is also deployed as a complementary first-pass layer** — not as a standalone option, but as a zero-cost safety net that catches the most obvious misclassifications immediately without waiting for the asynchronous critic.

In a greenfield agentic system, accuracy degradation is invisible without continuous instrumentation. A weekly audit (Option 2) is operationally insufficient because a misclassification that repeats 50 times in a week has downstream consequences across 50 email threads before anyone notices. The shadow critic provides a near-real-time signal that something has changed in SWEE's classification behaviour — allowing the AO team to roll back a model update or SOP change within hours, not days.

> **Compliance Note (PDPA)**: The PII-stripping step using Azure AI Language before any email content reaches the critic model is a PDPA data minimisation obligation. The legal basis for processing taxpayer email content within the government cloud boundary must be established; no email content (even anonymised) should leave the GCC 2.0 boundary. Langfuse is therefore deployed self-hosted; no LangSmith, Helicone, or other external SaaS evaluation platform is to be used.

> **Compliance Note (IM8)**: Audit logs of all triage decisions — including SWEE's classification label, confidence score, and critic agreement — must be retained for a minimum of 5 years per IM8 data management requirements. Langfuse's PostgreSQL backend must be backed up daily with point-in-time restore enabled, and backup retention set to at least 35 days with cold archive offload for long-term retention.
