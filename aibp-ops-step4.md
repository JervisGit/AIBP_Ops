# AIBP Operations Blueprint
## Step 4: Platform Reliability Operations

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Ensuring the AIBP platform remains available, performant, and within agreed service levels — accounting for the reliability characteristics unique to LLM-based agentic systems, particularly their dependency on external AI model endpoints and their non-deterministic execution profiles.

**Reliability context for agentic systems**: Traditional service reliability concerns (uptime, latency, throughput) apply here, but agentic platforms introduce two additional reliability dimensions:

1. **Semantic reliability**: The platform can be technically "up" (agents are running, APIs are responding) but delivering incorrect or harmful outputs. Semantic failures are harder to detect and cannot be resolved by a simple restart. EvalOps (Step 2.4) and HITL (Step 3.1) are the primary mechanisms for semantic reliability; this step focuses on the infrastructure and operational reliability layer underneath them.

2. **Dependency depth**: A single email processing event may involve 6–10 sequential external API calls (Azure OpenAI, AIGP tools, internal microservices via Kafka). Any one of these can fail, introducing partial failure scenarios that must be handled gracefully rather than causing entire email threads to be lost.

---

## 4.1 ResiliencyOps

### Implementation Overview

ResiliencyOps defines how the platform behaves under stress, partial failure, and external dependency outages. It encompasses three concerns:

1. **Circuit breaking**: Preventing cascading failures when a dependency (Azure OpenAI, internal microservices, AIGP API) is degraded
2. **Fallback routing**: Ensuring emails are never silently lost when the automated processing path is unavailable
3. **Capacity management**: Ensuring the Azure OpenAI tier is appropriate for the platform's throughput and latency requirements

---

### Option 1 (Recommended): Tenacity Circuit Breakers + Service Bus DLQ Fallback + Azure OpenAI Provisioned Throughput Units (PTU)

**Technology Stack**: Tenacity (Python retry/circuit-breaker library), Azure Service Bus (DLQ, Dead Letter Queue), Azure OpenAI (Provisioned Throughput Units), Azure Monitor (circuit state metrics), Azure Logic Apps (fallback notification)

---

#### Circuit Breaker Pattern (Tenacity)

Each external call in the AO agent's tool execution path is wrapped with a **Tenacity** circuit breaker. Tenacity is a Python retry library that provides configurable retry-with-backoff and circuit breaker state machine logic.

**Circuit breaker states**:
- **Closed** (healthy): Calls pass through normally
- **Open** (degraded): After `n` consecutive failures, all calls to that dependency fail immediately without attempting the call. This prevents latency compounding (where slow failing calls stack up and consume all available agent processing slots)
- **Half-Open** (recovery probe): After a cooling period, one test call is allowed through. If it succeeds, the circuit closes (healthy). If it fails, it reopens immediately.

**Circuit breaker configuration per dependency**:

| Dependency | Failure threshold (opens circuit) | Reset timeout | Retry before open | Backoff strategy |
|---|---|---|---|---|
| Azure OpenAI (frontier model endpoint) | 3 consecutive failures | 60 seconds | 2 retries with exponential backoff (2s, 4s) | Exponential + jitter |
| AIGP API | 3 consecutive failures | 30 seconds | 2 retries, 1s fixed | Fixed |
| Internal microservices (via AIGP) | 5 consecutive failures | 120 seconds | 3 retries exponential (1s, 2s, 4s) | Exponential |
| Langfuse trace export | 5 consecutive failures | 60 seconds | 3 retries | Exponential (non-blocking — trace export is async; failures do not fail the email processing) |

**Implementation in LangGraph agents**:

Each tool call node in the LangGraph graph wraps its external call with a Tenacity `@retry` decorator and a shared circuit breaker state object. Circuit breaker state is stored in an **in-process Python object** (not shared across containers) — each ACA container maintains its own circuit state. This is intentional: if one container is experiencing failures, that container's circuit opens and it stops queuing up failing calls, but other healthy containers continue processing. Cross-container circuit coordination would add latency for a marginal benefit at the platform's <1,000 emails/day scale.

**Circuit state metrics published to Azure Monitor**:

Each circuit transition (Closed → Open, Open → Half-Open, Half-Open → Closed) emits a custom metric (`ao.circuit.{dependency}.state_change`) to Azure Monitor. An alert fires when any circuit enters Open state, notifying the AO team and Ops.

---

#### Fallback Routing: Service Bus DLQ → Human Officer Queue

When a circuit is open (dependency unavailable) and the maximum retry window has been exhausted, the agent's LangGraph graph must route the email gracefully rather than hanging or dropping it.

**Fallback chain** (in priority order):

```
Azure OpenAI unavailable
    → Retry with exponential backoff (max 3 retries over ~30 seconds)
    → Circuit opens: abandon in-progress processing
    → Release Service Bus message (with delivery count increment)
    → If delivery count < max (3): message returns to queue; another container will retry
    → If delivery count = max (3): message moves to Service Bus DLQ
    → DLQ runbook (Step 1.1) detects new DLQ message
    → DLQ runbook classifies failure as 'AO_UNAVAILABLE'
    → DLQ runbook routes email to Human Officer Queue (a dedicated Service Bus queue 'human-officer-queue')
    → Tax officer receives notification via Teams (Azure Logic Apps workflow)
    → Tax officer processes the email manually in the existing case management system
```

**Human Officer Queue**: A dedicated Service Bus queue (`human-officer-queue-prod`) that serves as the fallback for emails the platform cannot process automatically. Tax officers subscribe to this queue via the same review app used for HITL (a dedicated tab is added to the existing review app: "Manual Process Queue"). A Teams channel notification is triggered when a new message enters this queue.

The `human-officer-queue` also receives `AUTO-BLOCK` emails from the AIGP layer (Step 3.1) and emails from timed-out HITL tasks — it is the unified human escalation queue for all categories of emails that the platform cannot or must not process automatically.

**Inter-dependency failure isolation**: If the AIGP API specifically is unavailable (but Azure OpenAI is available), the agent can still complete any reasoning steps that do not require tool calls. However, in practice, most AOagent workflows require at least one AIGP tool call to retrieve taxpayer data. In these cases, the agent follows the same fallback chain above: AIGP circuit opens → message re-queued or routed to human-officer-queue.

---

#### Azure OpenAI Provisioned Throughput Units (PTU)

Azure OpenAI offers two deployment modes:
- **Pay-as-you-go (PAYG)**: Token-based pricing; throughput is shared across all tenants and subject to rate limits (TPM — tokens per minute)
- **Provisioned Throughput Units (PTU)**: A reserved capacity allocation providing guaranteed TPM throughput and lower, more consistent latency

**PTU sizing for AIBP**:

At <1,000 emails/day, the peak processing demand is estimated using the following sizing methodology. The token counts below are model-agnostic; PTU allocation must be re-derived based on the confirmed frontier model's TPM-per-PTU ratio, which differs from GPT-4o and must be confirmed with the Microsoft/Azure account team before procurement.

| Parameter | Value | Basis |
|---|---|---|
| Emails per day | 1,000 | Platform specification |
| Average emails per hour (business hours) | 1,000 / 9 ≈ 112 emails/hour | Assumes 9-hour business day with concentrated traffic |
| Average tokens per email (prompt + completion) | ~2,500–3,500 tokens | Estimated from SOP complexity (5–7 reasoning steps); frontier models may require fewer tokens per step due to stronger reasoning |
| Token demand per hour | 112 × 3,000 = 336,000 tokens/hour | ≈ 5,600 TPM (midpoint estimate) |
| Peak burst factor (morning rush) | 2× average | Estimated |
| Peak TPM required | 5,600 × 2 = **~11,200 TPM** | This is the minimum PTU throughput target |

PTU allocation in units is then: `ceil(Peak TPM required ÷ TPM-per-PTU ratio for the frontier model)`. This ratio must be obtained from Azure at procurement time, as frontier models above GPT-4o have different PTU denominations and minimum commitments.

**PTU benefits** (model-independent):
- Eliminates PAYG rate-limit throttling (HTTP 429 errors) as a failure mode — the primary cause of AO agent circuit-break events in the PAYG model
- Consistent latency: PTU deployments provide predictable P95 latency; PAYG latency varies with shared tenant load
- Cost efficiency: At sustained throughput above approximately 50% utilisation of reserved capacity, PTU is cheaper per token than PAYG in Azure's GCC pricing

**PTU in the fallback design**: With PTU eliminating throttling as a failure mode, the circuit breaker configuration for Azure OpenAI above targets genuine service outages and model endpoint errors rather than throttle responses. This simplifies the circuit-open scenarios significantly.

> **Recommendation**: Confirm the frontier model's PTU minimum commitment and TPM-per-PTU ratio with the Azure account team before finalising the budget. Derive the initial PTU allocation from the sizing methodology above. After ORT load testing (Step 2.2, Stage 4), validate that the allocated PTU is sufficient at 2× daily volume. Review PTU utilisation via Azure Monitor monthly and adjust if utilisation consistently exceeds 60% (risk of approaching PTU ceiling) or drops below 20% (over-provisioned; consider reverting to PAYG or reducing PTU allocation).

> **Note on model deprecation**: PTU is a term commitment. If the frontier model is deprecated by Microsoft mid-term, the PTU commitment may need to be renegotiated. Model lifecycle monitoring is covered in Step 4.3.

**Pros (Option 1 overall)**:
- Tenacity circuit breakers are a battle-tested Python library; no additional infrastructure required beyond configuration
- The DLQ → Human Officer Queue fallback ensures no email is ever silently lost, regardless of what fails in the automated processing path
- PTU eliminates the most common failure mode (PAYG throttling)  for a platform running sustained batch-like AI workloads
- Circuit state metrics in Azure Monitor provide real-time visibility into dependency health without requiring manual inspection

**Cons**:
- Tenacity circuit state is per-container and not shared across ACA replicas — if all replicas simultaneously open their circuits (a common failure mode during a broad Azure OpenAI outage), the DLQ backlog may grow quickly. The DLQ runbook (Step 1.1) is designed to manage this scenario, but Ops must monitor DLQ depth during outage recovery.
- PTU requires an upfront commitment (monthly billing commitment); if the platform is de-scoped before contract end, the cost is not recoverable. For a government project, this commitment should align with the budget cycle.

---

### Option 2: Azure OpenAI PAYG + Static Retry Logic (No Circuit Breaker)

**Implementation**: Agents use Azure OpenAI PAYG model. Failed calls are retried a fixed number of times with a fixed sleep delay between retries. No circuit breaker state machine; no PTU commitment.

**Pros**: No upfront PTU commitment; no Tenacity library dependency; simpler implementation

**Cons**:
- During Azure OpenAI throttling events (HTTP 429), agents with static retry logic will all back off for the same duration and then simultaneously retry, potentially causing retry storms that compound the throttling effect
- No circuit breaker means a container processing a failing email will retry repeatedly across multiple LLM calls, blocking that container slot for an extended period and reducing overall platform throughput during degraded conditions
- PAYG latency variance means SLA breach risk during peak GCC 2.0-wide Azure OpenAI demand

---

### Option 3: Active-Passive AO Cluster (Secondary ACA Environment)

**Implementation**: A secondary, full-replica ACA environment is maintained in a second Azure region (Singapore secondary region). If the primary ACA environment becomes unavailable, SWEE's Service Bus namespace geo-pair (established in Step 1.1) fails over, and the secondary ACA environment begins consuming messages.

**Pros**: Maximum resiliency against a full primary-region ACA environment failure

**Cons**:
- Doubles the infrastructure cost of the AO layer for a failure scenario (primary ACA environment outage) that has very low probability given Azure's 99.95% ACA SLA
- At <1,000 emails/day, the business impact of a 30-minute ACA outage is approximately 40 emails re-queued for processing after recovery — the cost of maintaining a secondary cluster far exceeds the business impact mitigation value at this scale
- Not recommended at launch; should be re-evaluated if email volume scales to >10,000/day or if the platform becomes operationally critical with no maintenance window tolerance

---

### Recommendation Justification

**Option 1** is recommended. At <1,000 emails/day, the two highest-impact failure modes are Azure OpenAI quota saturation (mitigated by PTU) and cascading retry storms during dependency degradation (mitigated by circuit breakers). Option 3's active-passive cluster is over-engineered for the current scale and compliance overhead does not justify. The DLQ → Human Officer Queue fallback ensures 100% email durability — no email is lost, even if the entire automated processing path is unavailable for an extended period.

> **Compliance Note (IM8)**: IM8 requires a documented Business Continuity Plan (BCP) for government systems. The fallback chain described in this section (DLQ → Human Officer Queue) constitutes the BCP for the AO layer — manual processing capacity must be verified as sufficient to handle the full email volume during an extended automated processing outage. BCP verification should be included in the quarterly drills referenced in Step 7.2.

---

## 4.2 SLA Management

### Implementation Overview

Service Level Agreements define the performance and quality commitments made by the AIBP platform to the business (tax officers, management, and ultimately taxpayers). For an agentic AI platform, SLAs must capture both **operational** dimensions (speed, availability) and **quality** dimensions (accuracy, escalation rate) — traditional IT SLAs covering only uptime and latency are insufficient.

Because SWEE is a greenfield system, the SLA targets in this section are proposed baselines. They must be validated against actual production data during the first 90 days of operation and formalised in a Service Level Agreement document between the Ops team and business stakeholders.

---

### Option 1 (Recommended): Multi-Dimensional SLA Framework with Azure Monitor Workbook Dashboards

**Technology Stack**: Azure Monitor (custom metrics, alert rules), Azure Monitor Workbooks (SLA dashboard), Langfuse (quality metric aggregation), Azure Log Analytics (SLA reporting queries)

---

#### SLA Framework Alignment

No single published standard defines SLAs specifically for agentic AI applications as of mid-2026; the field is evolving faster than standards bodies. The AIBP SLA framework is designed to be compatible with the following established frameworks, drawing from each where relevant:

| Framework | Relevance to AIBP SLA design |
|---|---|
| **Google SRE — SLI/SLO/SLA + Error Budgets** | The operational layer: SLIs (Service Level Indicators) map to the KPIs in the table below; SLOs are the proposed targets; error budgets provide a principled mechanism for balancing reliability investment against deployment velocity. The canary promotion criteria (Step 2.3) are operationally equivalent to burn-rate alerts on an error budget. |
| **ISO/IEC 42001:2023** (AI Management Systems) | Section 9 (Performance Evaluation) mandates ongoing monitoring of AI system performance against defined criteria and corrective action. The multi-dimensional KPI framework below — spanning Speed, Quality, Availability, and Cost — satisfies the §9.1 monitoring requirement. |
| **NIST AI RMF 1.0 — MANAGE function** | MG-2 (ongoing risk monitoring) and MG-4 (performance measurement) are implemented through the continuous EvalOps pipeline (Step 2.4), behavioral anomaly detection (Step 2.7), and the SLA dashboard below. |
| **Singapore IMDA AI Governance Framework 2nd Ed / AI Verify** | Operational transparency and accountability requirements: the SLA dashboard (shared with management stakeholders) and the monthly SLA report (stored in the compliance evidence repository) provide the documented evidence trail required for AI Verify's Transparency and Accountability principles. |
| **Azure Well-Architected Framework — AI Workloads** | Microsoft's reliability and performance guidance for Azure-hosted AI workloads; the KPI targets below are calibrated against the WAF's recommended baselines for Azure Container Apps and Azure OpenAI workloads. |
| **EU AI Act (Article 9 + Annex IV)** | Not binding in Singapore, but establishes the emerging international benchmark for high-risk AI ongoing monitoring obligations. AIBP's continuous eval pipeline and HITL accuracy tracking align with Annex IV's post-market monitoring system requirement. |

> **On agentic-specific SLAs**: Traditional IT SLAs covering only uptime and latency are insufficient for agentic systems, which can be technically available but semantically degraded — agents processing emails but producing incorrect or low-quality outputs. The **Quality** dimension (Dimension 2 below) is AIBP's primary operational differentiator from conventional SLA frameworks, and is the dimension most closely monitored by management stakeholders.

---

#### SLA Metrics Framework

**Dimension 1 — Speed**

| KPI | Definition | Measurement method | Proposed target |
|---|---|---|---|
| **Time to Resolution (TTR) — Automated** | Time from Service Bus message enqueue by SWEE to email reply sent / case actioned by the agent (automated path only) | `ao.email.latency_ms` metric (OTel, Step 2.5) | P95 ≤ 5 minutes during business hours |
| **Time to Resolution — HITL** | Time from HITL task creation to officer decision completion | `hitl.review.durationMean` + HITL task lifecycle record | P95 ≤ 4 business hours (Tier 1); ≤ 1 business hour (Tier 2) |
| **Queue Drain Time** | Time to clear the full Service Bus queue backlog after a recovery from an outage or bulk arrival | Measured via Service Bus queue depth metric during recovery events | ≤ 2 hours for backlogs up to 2,000 messages (2× daily volume) |

**Dimension 2 — Quality**

| KPI | Definition | Measurement method | Proposed target |
|---|---|---|---|
| **Automated Resolution Accuracy Rate** | % of automated resolutions subsequently confirmed correct (no re-work required) — validated from HITL corrections and officer audit | `hitl.correctionRate` (HITL corrections / total automated resolutions) | ≥ 95% |
| **Hallucination Rate** | % of agent responses containing at least one unsupported factual claim, as detected by the EvalOps LLM-as-a-Judge (Step 2.4) | Production eval composite score, `faithfulness` sub-metric | ≤ 3% |
| **Misroute Rate** | % of emails re-classified after AO agent rejection or HITL correction (Step 1.2) | `ao.email.rerouteRate` (AO re-routes / total emails) | ≤ 5% |
| **HITL Escalation Rate** | % of emails that required HITL intervention (excluding AUTO-BLOCK which is expected) | `ao.email.hitl_escalated` metric | Track; no initial target; expected to decrease over time as the system matures |

**Dimension 3 — Availability & Reliability**

| KPI | Definition | Measurement method | Proposed target |
|---|---|---|---|
| **Platform Availability** | % of business hours during which the automated processing path is operational (not in Emergency Stop, not experiencing ACA/Service Bus outage) | Azure Monitor availability metric (heartbeat from Service Bus consumer) | ≥ 99.5% during business hours |
| **DLQ Rate** | % of emails that enter the Dead Letter Queue (requiring manual intervention) | `servicebus.dlq.messageCount` / total messages enqueued | ≤ 1% |
| **HITL Queue SLA Breach Rate** | % of HITL tasks that breach the HITL SLA tiers defined in Step 3.1 | `hitl.sla.breachRate` metric | ≤ 5% |
| **Emergency Stop Events per Month** | Count of Emergency Stop activations per month | Emergency Stop Cosmos DB audit records | ≤ 2 P2 events/month; 0 P1 (platform-wide) events/month |

**Dimension 4 — Cost Efficiency** *(detailed in Step 5, monitored here as an SLA dimension)*

| KPI | Definition | Measurement method | Proposed target |
|---|---|---|---|
| **Cost per Automated Resolution** | Total Azure token + compute cost attributed to one email resolved by the agent | FinOps cost allocation (Step 5) | ≤ SGD 0.20/resolution (proposed; to be baselined) |
| **Automation Rate** | % of total emails resolved by the agent without HITL or manual fallback | `resolutionType` metric distribution | ≥ 80% within 90 days; ≥ 90% at 1 year |

---

#### SLA Dashboard (Azure Monitor Workbook)

The **Platform SLA Dashboard** (`aibp-platform-sla`) is an Azure Monitor Workbook published to the Ops team's Azure portal bookmark and shared with management and business stakeholders. It contains:

**Tab 1 — Current Period (Today / This Week)**:
- Speed KPIs: P50/P95 TTR (automated), HITL queue depth, HITL mean review time — rolling 24h and 7-day views
- Quality KPIs: Accuracy rate, misroute rate — from Langfuse metric export
- Availability KPI: Uptime heatmap (green/amber/red by 30-minute windows)

**Tab 2 — SLA Compliance (Month to Date)**:
- Each KPI with current MTD value vs. target
- RAG (Red/Amber/Green) status per KPI:
  - Green: Within target
  - Amber: Within 10% of target threshold (at risk)
  - Red: Breaching target
- Trend sparklines (rolling 4-week trend)

**Tab 3 — Cost Efficiency**:
- Cost per resolution trend (links to FinOps Step 5 detail)
- Automation rate trend
- Monthly cost vs. budget

**Alerting**:

| Alert | Condition | Priority | Notified parties |
|---|---|---|---|
| P95 TTR breach | P95 automated TTR > 8 minutes (60% above target) for any 1-hour rolling window | P3 | Ops team |
| Accuracy rate degradation | 7-day rolling accuracy rate drops below 92% | P2 | Ops + AO team + business owner |
| DLQ rate spike | DLQ rate > 3% over any 1-hour window | P2 | Ops + AO team |
| Platform availability breach | Heartbeat missing for > 5 minutes during business hours | P1 | Ops + on-call engineer |
| Automation rate decline | 7-day rolling automation rate drops more than 10 percentage points from the prior 7-day period | P3 | Ops + AO team |

---

#### Monthly SLA Report

A monthly SLA report is generated automatically by an **Azure Logic Apps** workflow on the first business day of each month. The report:
- Pulls the prior month's SLA metric values from Log Analytics (KQL queries)
- Computes compliance percentage per KPI (days/hours within target vs. total business hours)
- Flags any KPIs that breached target for ≥ 3 consecutive days (sustained degradation requiring root cause analysis)
- Outputs a PDF report to the compliance evidence repository (`ops-compliance-evidence/sla-reports/`) and posts a summary card to the management Teams channel

---

### Option 2: Manual Monthly SLA Report (Spreadsheet)

**Implementation**: Ops team manually extracts metrics from Azure Monitor and Langfuse monthly and populates a spreadsheet SLA report for management review.

**Pros**: No dashboard infrastructure to build or maintain

**Cons**:
- No real-time visibility — SLA breaches are discovered at month-end, not when they occur
- Manual data extraction is error-prone and time-consuming; at the number of KPIs defined above, this would require 4–8 hours of analyst effort per month
- Cannot support alert-driven escalation for in-flight SLA breaches (the most operationally important capability)

---

### Option 3: Power BI Service Dashboard

**Implementation**: Azure Monitor metrics are fed into Power BI Service via the Azure Monitor connector. SLA dashboards are published in Power BI.

**Pros**: Rich visualisation capability; widely used in government organisations; familiar to management stakeholders

**Cons**:
- Power BI Service in GCC 2.0 — availability on the Government Community Cloud must be confirmed. Power BI embedded within Azure Monitor Workbooks (Option 1) avoids this dependency.
- Power BI dashboards are typically refreshed on a schedule (15-minute minimum for streaming datasets), not in real time; Azure Monitor Workbooks can query Application Insights in real time
- An additional licensing and platform dependency for a function (operations dashboard) that Azure Monitor Workbooks can serve natively

---

### Recommendation Justification

**Option 1** is recommended. Azure Monitor Workbooks are natively integrated with Application Insights and Log Analytics — the two stores already receiving all AIBP operational metrics. Building the SLA dashboard in Workbooks avoids introducing an additional platform (Power BI Service) for a function that Azure Monitor already supports. The automated monthly report via Logic Apps ensures management visibility without manual extraction effort.

> **Compliance Note (IM8)**: IM8 requires that government IT systems maintain records of system performance and service delivery. The monthly SLA report, stored in the compliance evidence repository, satisfies this requirement. The SLA targets agreed between Ops and the business owner should be documented in a formal Service Level Agreement document, reviewed annually.

---

## 4.3 Model Lifecycle Management

### Implementation Overview

Frontier LLM models on Azure have defined lifecycle dates — a deprecation notice period followed by a retirement date after which the model endpoint is no longer available. For AIBP, an unplanned model endpoint retirement would be an operational P1 event: agents would cease processing entirely until a replacement model is provisioned, configured, and promoted through the full SIT → UAT → ORT → PROD pipeline. Model lifecycle management is the proactive operational practice of tracking deprecation notices and executing model transitions in a controlled, tested manner before the retirement date.

This concern is distinct from App & Data's responsibility for prompt engineering and model selection — App & Data determines *what* replacement model to use and *how* to configure it; Ops owns the *deployment pipeline* for a model version change and the *operational readiness* of the platform on the new model.

---

### Model Lifecycle Operational Process

**Monitoring deprecation notices**: Azure OpenAI publishes model retirement dates in the Azure OpenAI service documentation and via Azure Service Health notifications. The Ops team subscribes to Azure Service Health Health Advisory alerts scoped to the Azure OpenAI resource group (see also Step 4.4). Upon receiving a model deprecation notice, the Ops team initiates the following process:

**Step 1 — Assessment** (within 5 business days of notice):
- Confirm the retirement date and recommended replacement model from Azure documentation
- Notify the AO team, AIGP team, and business owner of the retirement timeline
- Create a Model Migration work item in Azure DevOps, assigned to AO team lead, with the retirement date as the deadline and a **target completion date at least 30 days before retirement**

**Step 2 — Model Update Development** (AO team, led by App & Data):
- AO team updates model configuration in the agent codebase (`LLM_MODEL_NAME`, PTU endpoint references)
- AO team updates `agent-manifest.json` evaluation thresholds if the replacement model's baseline scores are expected to differ
- AO team re-runs the synthetic evaluation suite against the replacement model to confirm eval score compatibility
- This constitutes a `PATCH` or `MINOR` version bump (no SOP scope change, no tool-contract change)

**Step 3 — Full Pipeline Promotion** (SIT → UAT → ORT → PROD):
- The model update is promoted through the full SIT → UAT → ORT → PROD pipeline (Step 2.2)
- **ORT stage receives additional emphasis**: the ORT load test (Stage 4) must be run against the replacement model to confirm that the PTU allocation for the new model is sufficient and that latency and token spend remain within SLA thresholds (see Step 4.1 PTU sizing)
- Target: complete full pipeline promotion at least **30 days before the retirement date**, providing a rollback buffer if the replacement model introduces regressions

**Step 4 — PTU Contract Review**:
- If the replacement model has a different PTU denomination or minimum commitment, the Azure account team must be contacted to renegotiate the PTU contract before the current model's retirement date
- The Ops team coordinates this with the finance sponsor; a PTU contract change is a financial commitment requiring management approval

**Step 5 — Post-Migration Monitoring**:
- After full production promotion, run an extended **7-day canary monitoring window** (vs. the standard 24–48 hours) for model replacement deployments, due to the broader distribution shift introduced by a different model
- Flag the deployment in Azure DevOps and Langfuse with a `model-replacement` tag for traceability
- Compare production token spend per email during the monitoring window against the ORT load test baseline to detect unexpected cost escalation on the new model

---

### Model Fallback Configuration

If the primary frontier model endpoint becomes unavailable (service outage, not retirement), the circuit breaker (Step 4.1) opens after 3 consecutive failures. To prevent all email processing from falling through to the DLQ during a short outage, Ops may optionally configure a **fallback model endpoint**:

- A secondary model (e.g., a lower-tier model still within its lifecycle) is configured in Azure App Configuration: `agents:llm:fallbackEndpoint`
- If the primary endpoint circuit is open, the agent retries against the fallback endpoint for a configurable grace period (default: 30 minutes)
- After the grace period, if the primary endpoint has not recovered, the agent routes to the DLQ → Human Officer Queue fallback chain (Step 4.1)

> **Fallback model quality note**: A fallback model is by definition lower-capability than the primary frontier model. Emails processed via the fallback should be flagged with `processedOnFallbackModel: true` in their OTel trace, and considered for routing to HITL review if the quality gap between models is material.

> **Compliance Note (IM8 / ISO 27001 Change Management)**: Model retirement-driven deployments are system changes that must be documented in the change record. The `agent_promotions` registry DB table records the promotion of the updated agent version; the model name change in `agent-manifest.json` provides the traceability link between the deployment event and the model lifecycle trigger.

---

## 4.4 External Dependency Health Monitoring

### Implementation Overview

The AIBP platform's reliability depends on external services that Ops does not provision: Azure OpenAI endpoints, Azure Container Apps, Azure Service Bus, Azure Container Registry, and the AIGP API. The circuit breaker pattern (Step 4.1) handles runtime failures reactively. This section covers **proactive monitoring** — detecting planned maintenance, quota ceilings, and impending lifecycle events before they cause circuit-break events or deployment failures.

---

### Azure Service Health Alerts

Azure Service Health provides three notification categories relevant to AIBP:

| Alert type | What it covers | AIBP relevance |
|---|---|---|
| **Service Issues** | Ongoing Azure outages affecting specific services and regions | Azure OpenAI outage → all agent circuit breakers open; ACA issue → agent fleet unavailable; Service Bus issue → email queue disrupted |
| **Planned Maintenance** | Microsoft-initiated maintenance windows that may temporarily reduce performance | PTU throughput may be reduced; plan no production deployments during announced windows |
| **Health Advisories** | Service lifecycle notifications — deprecation, retirements, API version changes | Model retirement notices (feed into Step 4.3); ACA or Service Bus API deprecation notices |

**Configuration**:
1. In Azure Service Health → Health Alerts, create alert rules scoped to: `Azure OpenAI`, `Azure Container Apps`, `Azure Service Bus`, `Azure Container Registry`
2. Action group: notify Ops team via email + Teams webhook to `#ops-platform-alerts`
3. **Service Issues**: P2 priority; notify immediately
4. **Planned Maintenance**: P3 priority; Ops team schedules a review to decide whether to pause deployments during the announced window
5. **Health Advisories**: P3 priority; triage within 2 business days; model retirement advisories automatically trigger the Step 4.3 lifecycle process

---

### Quota & Capacity Headroom Monitoring

Azure subscriptions have resource quotas. Running close to a quota ceiling without advance notice causes deployment failures. Ops monitors headroom for the following critical quotas:

| Resource | Metric | Alert threshold | Action on alert |
|---|---|---|---|
| Azure OpenAI PTU utilisation | `PTUUtilizationPercentage` (Azure Monitor metric) | > 70% sustained over 1 hour | Review token consumption trends; initiate PTU expansion request to Azure account team |
| Azure OpenAI PAYG TPM (fallback quota) | PAYG consumption during circuit-open periods | > 50% of PAYG quota in any hour | Log as operational event; review if fallback model usage is higher than expected |
| ACA replica count | Active replicas per agent container app vs. configured `maxReplicas` | > 80% of `maxReplicas` | Review scale-out configuration; raise `maxReplicas` if email volume growth warrants it |
| ACR storage | Registry storage utilisation | > 70% of quota | Run ACR image purge for deprecated agent versions (`az acr run ... --cmd 'acr purge'`) |

**Quota review cadence**: A quarterly capacity review is conducted by the Ops team. Ops queries current quota limits and utilisation for each resource above, documents the headroom, and flags any resources projected to reach the alert threshold before the next quarterly review based on observed growth trends.

---

### AIGP API Health Monitoring

The AIGP API is an internal dependency, not an Azure service, so Azure Service Health does not cover it. Ops monitors AIGP API health via:

1. **Azure Monitor availability test**: A synthetic probe pings the AIGP API health endpoint (`/health`) every 5 minutes. If 3 consecutive probes fail, a P2 alert fires to Ops and the AIGP team.
2. **Circuit breaker state metric**: The `ao.circuit.aigp_api.state_change` custom metric (Step 4.1) is the reactive signal when the AIGP circuit opens. The synthetic probe is the proactive signal. Both should be active; neither replaces the other.
3. **AIGP team maintenance coordination**: When the AIGP team plans a maintenance window, they notify Ops at least 5 business days in advance. Ops posts a maintenance notice to `#ops-platform-alerts` and ensures no production canary deployments (Stage 5, Step 2.2) are scheduled during the window.
