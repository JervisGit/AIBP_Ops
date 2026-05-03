# AIBP Operations Blueprint
## Step 6: Operational Feedback & Automated Self-Reflection

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Closing the loop — systematically capturing signals from production operations (officer corrections, HITL decisions, resolution outcomes, evaluation scores) and feeding them back into the continuous improvement of agent quality, AIGP policy thresholds, and operational runbooks.

**Why this step exists**: An agentic system that does not learn from its own operational experience will drift over time. The world it was designed for changes: taxpayer query patterns shift, tax policy is updated, new SOP categories are introduced, and HITL correction patterns accumulate. Without a structured feedback loop, none of this learning reaches the agents — the App & Data team is flying blind on where the platform is degrading, and officers are correcting the same class of mistakes repeatedly without any mechanism to propagate that correction into the system.

This step is not about AI model fine-tuning (that is an App & Data concern). It is about Ops owning the data pipelines and feedback infrastructure that make continuous model and policy improvement possible.

---

## 6.1 Structured Feedback Loop

### Implementation Overview

The structured feedback loop collects decision-quality signals from every point where a human interacts with or corrects the platform, and routes them to the right owner with the right level of detail.

**Signal sources** (four):
1. **HITL officer decisions** — when an officer approves or rejects an agent's requested action, the decision itself is a quality signal. A high rejection rate for a specific action type indicates the agent is over-proposing that action.
2. **Officer audit corrections** (Step 1.2 stratified audit) — when a tax officer relabels a triage decision in the SWEE audit tool, that correction is a direct accuracy signal for the triage model.
3. **Human Officer Queue cases** (fallback from Step 4.1) — emails that the platform failed to process automatically. These represent a ground-truth record of cases the agents could not handle; the resolution the officer reaches becomes a labelled training example.
4. **AO agent rejection / re-route events** — cases where the AO agent explicitly rejected the SWEE-routed SOP and requested re-classification; these are signals that SWEE's triage made a semantically wrong choice.

---

### Option 1 (Recommended): Event-Driven Feedback Pipeline → Langfuse Evaluation Dataset

**Technology Stack**: Azure Event Grid (feedback event routing), Azure Container Apps Job (feedback processor), Langfuse (self-hosted, dataset annotation), Azure Service Bus (feedback event queue), Azure Logic Apps (feedback summary reporting), Azure Monitor

---

#### Feedback Collection Architecture

All four signal sources emit structured feedback events. Each event is published to an **Azure Service Bus topic** (`ops-feedback-events`) via the respective system:

| Signal source | Event emitter | Event type | Key fields |
|---|---|---|---|
| HITL decision (Step 3.1) | HITL review app → Azure Function | `hitl.decision.submitted` | `messageId`, `taskId`, `decision` (APPROVE/REJECT), `correctionType`, `officerNotes`, `officerId` |
| Officer audit correction (Step 1.2) | SWEE audit tool | `triage.audit.correction` | `messageId`, `sweeLabel`, `officerLabel`, `confidenceBand`, `officerId` |
| Human Officer Queue resolution | Review app (manual process tab) | `manual.resolution.completed` | `messageId`, `sopId`, `officerResolutionSummary` (free text, anonymised), `officerId` |
| AO agent re-route request | AO agent via App Insights custom event | `ao.reroute.requested` | `messageId`, `originalSopId`, `requestedSopId` (if known), `agentName`, `rerouteReason` |

**Feedback processor** (Azure Container Apps Job, `feedback-processor`, runs every 15 minutes):

1. Dequeues all pending feedback events from `ops-feedback-events`
2. For each event, retrieves the corresponding trace from Langfuse using the `messageId`
3. Annotates the Langfuse trace with a structured score:

| Feedback type | Langfuse score name | Score value | Score comment |
|---|---|---|---|
| HITL APPROVE | `hitl_quality` | 1.0 | Officer approved the agent's proposed action |
| HITL REJECT | `hitl_quality` | 0.0 | Officer rejected; `correctionType` recorded in comment |
| Triage correction (different label) | `triage_accuracy` | 0.0 | `officerLabel` recorded; misroute confirmed |
| Triage correction (same label) | `triage_accuracy` | 1.0 | SWEE triage confirmed correct by officer |
| Manual resolution (fallback queue) | `agent_capability` | 0.0 | Agent failed to process; `sopId` and `officerResolutionSummary` recorded |
| AO re-route | `triage_accuracy` | 0.0 | AO explicitly rejected SWEE's SOP; `requestedSopId` recorded |

4. For feedback events where `score = 0.0` (failures), the processor adds the trace to the appropriate **Langfuse evaluation dataset**:
   - `triage.audit.correction` → added to `swee-triage-failures` dataset (used by App & Data / SWEE team for model improvement)
   - `hitl.decision.submitted` (REJECT) → added to `ao-hitl-rejections` dataset (used by AO team for agent prompt improvement)
   - `manual.resolution.completed` → added to `ao-capability-gaps` dataset (used by AO team to identify SOP categories the agent cannot handle)

5. The processor emits a daily `ops.feedback.summary` metric to Azure Monitor:
   - Total feedback events processed by type
   - Running 7-day rejection rate by signal source (which agent / which SOP is producing the most corrections?)
   - New entries added to each evaluation dataset

---

#### Feedback Routing to Owning Team

Langfuse evaluation datasets are accessible to the owning teams as read-only shared views:

| Dataset | Owning team | Usage cadence | Action expected |
|---|---|---|---|
| `swee-triage-failures` | App & Data / SWEE squad | Weekly review | Investigate systematic misroute patterns; update SOP embedding corpus; retrain or tune triage model if patterns persist > 2 weeks |
| `ao-hitl-rejections` | AO team | Biweekly sprint review | Review rejection patterns; update agent prompts or AIGP risk thresholds if over-conservative; file a `PATCH` release for targeted prompt fixes |
| `ao-capability-gaps` | AO team | Monthly review | Identify SOP subcategories where the agent consistently fails; escalate to App & Data for SOP coverage expansion or new agent development |

Ops sends a weekly **Feedback Summary Report** (Azure Logic Apps, every Monday 09:00) to the relevant teams, containing:
- Count of new entries per dataset for the past week
- Top 3 most common `correctionType` or `rerouteReason` values
- Week-over-week trend (increasing or decreasing failure rates)

The Feedback Summary Report is a lightweight mechanism to ensure owning teams do not need to monitor Langfuse themselves — Ops acts as the centralised quality signal aggregator and distributor.

---

#### Feedback Quality Controls

Not all officer corrections are high-quality signals. Two quality controls are applied:

1. **Officer agreement rate**: If two or more officers independently label the same case differently, the trace is flagged as `ambiguous` and excluded from the evaluation dataset until a supervisor resolves the disagreement (this applies primarily to the SWEE audit, where multiple officers may independently review the same sampled case).

2. **30-day review window**: New failure traces are held in a quarantine section of the Langfuse dataset for 30 days before being incorporated into the CI evaluation baseline. This allows the AO team to formally review each failure trace and either (a) confirm it as a legitimate failure case, (b) mark it as an officer error (removed from dataset), or (c) flag it as an edge case requiring special handling. Without this gate, a confused officer's incorrect correction could corrupt the evaluation baseline.

**Pros**:
- Event-driven pipeline means feedback is captured within 15 minutes of the officer action — not in a weekly batch
- Langfuse evaluation dataset accumulation creates a compounding quality asset: the longer the platform runs, the richer the failure case library, and the more targeted and precise future agent updates become
- Feedback routing via weekly report keeps owning teams informed without requiring them to proactively monitor a dashboard
- 30-day quarantine gate prevents officer errors from polluting the CI evaluation baseline

**Cons**:
- Feedback processor job adds an additional ACA Job to maintain
- The 30-day quarantine means newly discovered failure patterns take up to 4 weeks to reach the CI gate; for critical regressions, the AO team should manually fast-track the review
- Officer notes in feedback events contain free text, which may inadvertently include PII if officers describe taxpayer characteristics. The feedback processor must apply the same Azure AI Language PII-stripping step (Step 1.2) to all free-text fields before they enter Langfuse.

---

### Option 2: Weekly Manual Feedback Collation by Ops

**Implementation**: Ops team manually queries Azure Monitor and Langfuse each week to find HITL rejections, audit corrections, and fallback queue cases, and compiles a summary report for the AO and SWEE teams.

**Pros**: No pipeline infrastructure required; follows established manual reporting workflows

**Cons**:
- Weekly cadence means the AO team receives failure signals up to 7 days late; a systematic agent error can generate 70–100 affected emails in a week before the team is notified
- Manual collation is error-prone; patterns that span multiple signal types (HITL rejections and AO re-routes on the same SOP category) may be missed if signals are reviewed independently
- Does not produce structured Langfuse dataset annotations, which means the weekly report cannot be directly consumed by the CI evaluation gate

---

### Recommendation Justification

**Option 1** is recommended. The feedback pipeline's primary value is not the report — it is the Langfuse dataset annotations it accumulates. Over time, these annotations transform the evaluation dataset from a static synthetic collection into a living record of real-world failure patterns, making the CI evaluation gate progressively more sensitive to exactly the kinds of failures the platform has historically experienced. This compounding quality asset is only achievable through an automated, real-time pipeline; a weekly manual process cannot build it.

---

## 6.2 Automated Post-Mortem Pipeline

### Implementation Overview

A post-mortem is a structured analysis of a specific failure event — an Emergency Stop activation, a significant HITL SLA breach, a sudden drop in accuracy rate, or a behavioural anomaly detection event. Traditional post-mortems are written manually by the on-call engineer after an incident. For an agentic AI platform, a significant portion of the post-mortem data (traces, metrics, event sequences) can be assembled automatically, reducing the time from incident to root cause analysis from hours to minutes.

The automated post-mortem pipeline assembles the factual record of an incident automatically; the human analyst completes the causal analysis (the "why") and writes the remediation plan.

---

### Option 1 (Recommended): Event-Driven Trace Analysis → Automated Post-Mortem Draft

**Technology Stack**: Azure Event Grid (incident trigger), Azure Container Apps Job (post-mortem assembler), Azure OpenAI (GPT-4o, summarisation of trace patterns), Langfuse (trace retrieval), Azure Blob Storage (post-mortem report store), Azure Logic Apps (report distribution)

---

#### Post-Mortem Trigger Events

The post-mortem pipeline is triggered by the following incident signal types:

| Trigger type | Source | Severity |
|---|---|---|
| Emergency Stop activation | Emergency Stop Cosmos DB event (Step 3.2) | P1 / P2 |
| Behavioral anomaly — severity HIGH (>50% anomaly rate) | `ops-anomaly-events` Service Bus topic (Step 2.7) | P2 |
| HITL SLA breach — sustained (>10% breach rate over 24h) | `hitl.sla.breachRate` Azure Monitor alert (Step 3.1) | P2 |
| Accuracy rate drop >5% (7-day rolling) | `ao.accuracy.degradation` Azure Monitor alert (Step 2.4) | P2 |
| DLQ spike (>3% DLQ rate over 1 hour) | `servicebus.dlq.messageCount` alert (Step 4.2) | P3 |

Each trigger emits an **Azure Event Grid** event to the `ops-incident-events` topic.

---

#### Post-Mortem Assembler (ACA Job)

The `postmortem-assembler` ACA Job is triggered on events from `ops-incident-events`. For each incident event, it produces an automated post-mortem draft:

**Step 1 — Timeline reconstruction**:

The assembler queries Langfuse and Azure Monitor to build a chronological timeline of events in the 2 hours preceding the incident trigger:

```
[Automated Timeline — refund-agent 1.3.0 — Emergency Stop 2026-05-03 09:42]
09:10  Agent deployed to canary (10% traffic)
09:14  First production trace begins
09:22  Behavioral anomaly rate: 12% (below 20% alert threshold)
09:28  Behavioral anomaly rate: 31% (below auto-kill threshold)
09:36  Behavioral anomaly rate: 54% (exceeds 50% auto-kill threshold)
09:36  Emergency Stop triggered automatically (source: anomaly_detector)
09:36  Kill switch activated: agents:refund-agent:1.3.0:enabled = false
09:37  7 in-flight messages returned to Service Bus queue
09:38  All ACA containers for refund-agent 1.3.0 drained
```

**Step 2 — Affected trace retrieval**:

The assembler retrieves the 20 most anomalous traces from Langfuse for the incident time window (using the behavioral anomaly score recorded in Step 2.7). For each trace, it extracts:
- Tool call sequence (as a clean text representation)
- Token count (prompt + completion)
- LLM reasoning step summaries (from `inputSummary` / `outputSummary` in the Cosmos DB audit ledger)
- Whether the trace resulted in an action on taxpayer data

**Step 3 — Pattern summary (GPT-4o)**:

The assembler sends the 20 anomalous trace summaries to GPT-4o with a structured analysis prompt:

```
System: You are an AI operations analyst reviewing agent execution traces
        after an incident. Identify patterns in the anomalous traces
        that distinguish them from normal behaviour. Be specific about
        tool call sequences, token utilisation, and reasoning patterns.
        Do not include any taxpayer personal data.

User: [20 anonymised trace summaries]

Provide:
1. The most common anomalous pattern (2–3 sentences)
2. Hypothesised root causes (ranked by likelihood)
3. Which traces appear to have resulted in an action on taxpayer data
   (flag for human review)
4. Recommended immediate investigation focus
```

GPT-4o's structured response becomes the "Automated Analysis" section of the post-mortem draft.

**Step 4 — Post-Mortem Draft Assembly**:

The assembler generates a Markdown post-mortem document and writes it to Azure Blob Storage (`ops-compliance-evidence/incident-reports/`):

```markdown
# Post-Mortem: Emergency Stop — refund-agent v1.3.0
**Incident ID**: INC-2026-0503-001
**Status**: DRAFT — Automated (pending human completion)
**Date/Time**: 2026-05-03 09:36 SGT
**Trigger**: Behavioral anomaly rate exceeded 50% threshold (auto-kill)
**Severity**: P2
**Duration**: 6 minutes from first anomaly to stop activation

## Automated Timeline
[assembled timeline from Step 1]

## Impact
- Emails affected (in-flight at stop): 7
- Emails re-queued: 7 (all recovered to Service Bus, no data loss)
- Emails that reached a completed tool action before stop: 2 (flagged for
  manual review — see HITL task IDs below)
- Email processing downtime: 18 minutes until canary traffic reverted
  to blue revision

## Automated Analysis (GPT-4o)
[GPT-4o pattern summary from Step 3]

## Human Completion Required
- [ ] Root cause confirmed by: ________ (AO team)
- [ ] HITL review completed for 2 flagged actions: Task IDs HT-001, HT-002
- [ ] Remediation plan documented below
- [ ] Lessons learned added to runbook

## Remediation Plan
[Human-authored]

## Lessons Learned
[Human-authored]

## Actions
| Action | Owner | Due date | Status |
|--------|-------|----------|--------|
| | | | |
```

**Step 5 — Distribution**:

An **Azure Logic Apps** workflow distributes the post-mortem draft:
- Posts a Teams card to the `#ops-incidents` channel with a link to the draft in Blob Storage
- Assigns a review task in Azure DevOps to the on-call Ops engineer
- Tags the relevant Langfuse traces with the incident ID for cross-referencing

**Human completion SLA**: The post-mortem draft must be completed (root cause, remediation plan, lessons learned) within:
- P1 incidents: 24 hours
- P2 incidents: 5 business days
- P3 incidents: 10 business days

Uncompleted post-mortems generate automated reminders via the Logic Apps workflow at 50% of the SLA window and a final escalation at the SLA deadline.

**Pros**:
- Timeline reconstruction (the most time-consuming part of manual post-mortems) is fully automated — the on-call engineer receives a factual record of exactly what happened without needing to piece it together from multiple dashboards
- GPT-4o pattern analysis on anomalous traces can identify root cause hypotheses that the engineer may not immediately recognise, particularly for subtle LLM reasoning pattern changes
- Automated identification of traces where an action reached internal systems (before the stop) ensures no taxpayer impact goes unreviewed — HITL tasks are created for these cases automatically
- Post-mortem completion tracking in Azure DevOps ensures lessons learned are not lost

**Cons**:
- GPT-4o pattern analysis may suggest incorrect or superficial root cause hypotheses; the human analyst must treat it as a starting point, not a conclusion
- PII control is critical in Step 3: the 20 trace summaries fed to GPT-4o must pass through the same PII-stripping step as all content entering an LLM (Azure AI Language PII detection before GPT-4o call)
- Post-mortem drafts in Blob Storage must be access-controlled (Entra ID RBAC, Ops team read/write only) — even anonymised incident reports contain internal system behaviour details that warrant RESTRICTED classification

---

### Option 2: Automated Alert with Manual Post-Mortem

**Implementation**: Azure Monitor alerts notify the on-call engineer on incident trigger. The engineer manually queries Azure Monitor, Langfuse, and Cosmos DB to assemble the timeline and writes the post-mortem from scratch.

**Pros**: No post-mortem assembler infrastructure; familiar process for operations teams used to manual incident management

**Cons**:
- Manual timeline assembly from distributed data sources (Langfuse, Azure Monitor, Cosmos DB, Service Bus logs, App Configuration audit) typically takes 1–2 hours for a complex incident. This reduces time available for root cause analysis and remediation.
- Systematic patterns across multiple traces are hard to identify manually (comparing 20 trace summaries visually is error-prone). The GPT-4o pattern analysis step provides genuine value here.
- With manual process only, post-mortem completion discipline degrades over time — engineers resolve the incident, move on, and the post-mortem draft is never completed. Automated reminders and Azure DevOps assignment counteract this.

---

### Recommendation Justification

**Option 1** is recommended. The automated timeline reconstruction and GPT-4o pattern analysis are the two components that provide material time savings over pure manual process. The post-mortem draft is never a complete substitute for human judgment — but by handling the factual assembly automatically, it ensures the engineer's limited incident-response time is spent on causal reasoning and remediation design, not data retrieval.

> **Compliance Note (IM8)**: IM8 requires formal incident reports for all significant security or service disruptions. The completed post-mortem document stored in the compliance evidence repository (`ops-compliance-evidence/incident-reports/`) constitutes this record. IM8 specifies reporting timelines to the Government Cybersecurity Operations Centre (GCOC) for incidents involving cyber threats or data breaches — the Ops team must be aware of these timelines and the GCOC notification interface, which is a separate process from this internal post-mortem.

> **Compliance Note (ISO 27001)**: ISO 27001 control A.16.1.6 (Learning from Information Security Incidents) requires that lessons learned from incidents are used to update controls and prevent recurrence. The "Lessons Learned" section of the post-mortem, combined with the Langfuse dataset annotation of anomalous traces (Step 6.1), satisfies this control.
