# AIBP Operations Blueprint
## Step 3: Governance & Control Operations (AIGP Layer)

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Managing the safety valve between the Agentic Orchestration (AO) layer and the internal microservices — ensuring that every agent action is authorised, gated for human review when warranted, permanently auditable, and stoppable within seconds if the platform behaves dangerously.

**AIGP Layer Context**: The AI Governance Platform (AIGP) is the mandatory intermediary through which every agent tool call must pass before touching internal databases or microservices. No agent may directly call internal APIs. All calls go through AIGP, which:

1. Authenticates the agent's identity (via Entra ID — the agent presents a Managed Identity token)
2. Authorises the requested action against the agent's declared capability manifest (via ForgeRock, which federates to Entra ID for AuthN while owning AuthZ policy)
3. Evaluates the risk score of the proposed action
4. Either permits the action, escalates it to a human (HITL), or blocks it
5. Emits an immutable audit event for every decision

**Team boundary**: The AIGP layer is owned and operated by the AIGP team, not the AO team. The Ops function is responsible for monitoring the health and operational metrics of the AIGP layer, managing the HITL workflow that bridges AIGP to tax officers, and operating the Emergency Stop mechanisms that the AIGP layer exposes.

**AO / AIGP interface**: AO agents call AIGP tools via a REST API protected by APIM. The AIGP API is the gatekeeper; it receives the agent's intended action and its parameters, evaluates it, and either executes it (returning the result to the agent), holds it pending human review (returning a `PENDING_HITL` status to the agent), or rejects it.

---

## 3.1 Human-in-the-Loop (HITL) Management

### Implementation Overview

HITL is the mechanism by which the platform defers certain agent actions to a human tax officer before execution. It is not a failure mode — it is a designed safety feature for actions that carry risk too high for fully automated execution.

HITL is triggered by the AIGP layer when an agent requests an action that meets defined risk criteria. The agent's workflow pauses at the point of the AIGP API call, awaiting a human decision. The tax officer's decision is then returned to the AIGP API, which completes the action (or rejects it) and notifies the waiting agent.

**The HITL system has three operational dimensions**:

1. **Trigger design**: What criteria cause the AIGP to escalate an action for HITL review?
2. **Review workflow**: How does the tax officer receive, review, and decide on the escalation?
3. **Timeout and escalation**: What happens if a HITL review is not completed within the SLA window?

**Tax officer context**: Tax officers who perform HITL reviews are the same officers who handle normal tax casework. HITL review is an additional duty, not a full-time role. The review interface must be efficient (minimal context-switching), clear, and auditable. Officer load must be monitored to prevent HITL backlog from becoming a bottleneck.

---

### Option 1 (Recommended): Risk-Score-Based AIGP Triggers + Azure Service Bus HITL Queue + Custom Review App on ACA

**Technology Stack**: AIGP risk scoring engine (AIGP team), Azure Service Bus (Premium), Azure Container Apps (HITL review web app), Azure Table Storage (HITL decision log), Azure Monitor, Azure Logic Apps (timeout escalation)

---

#### HITL Trigger Design (Risk-Based Escalation)

The AIGP layer computes a **risk score** (0.00–1.00) for every agent tool call based on the following factors:

| Risk factor | Weight | Description |
|---|---|---|
| **Action type** | High | Write/execute actions score higher than read actions. Irreversible actions (e.g., bank account changes, refund disbursements) score highest. |
| **Amount / impact magnitude** | High | Actions involving amounts above defined thresholds score higher. Thresholds are defined per SOP category by the AIGP team. |
| **Taxpayer sensitivity flag** | Medium | Taxpayers flagged as disputed, under audit, or high-complexity score higher. |
| **Agent confidence proxy** | Medium | If the AO agent's own reasoning included a low-confidence qualifier (e.g., "I am uncertain whether this qualifies as..."), the AIGP receives this flag and applies a higher risk multiplier. |
| **Action frequency anomaly** | Low | If the same action is being requested for the same taxpayer more than once within a short window, the risk score is elevated. |
| **Novel action pattern** | Low | If the combination of tool name + input parameters does not match any pattern in the agent's historical baseline (from Step 2.7), the risk score is elevated slightly. |

**Escalation thresholds**:

| Risk score range | AIGP decision | Agent receives |
|---|---|---|
| 0.00–0.40 | AUTO-PERMIT | Action executed immediately; result returned to agent |
| 0.41–0.69 | PERMIT WITH AUDIT | Action executed; additional audit record written; no human required |
| 0.70–0.89 | HITL REQUIRED | Action held; HITL task created; agent receives `PENDING_HITL` with a task ID |
| 0.90–1.00 | AUTO-BLOCK | Action rejected; agent receives `BLOCKED` with reason code; email re-routed to human officer queue |

> **These thresholds are initial values and must be calibrated during the greenfield baseline period (first 90 days of production).** The AIGP team, in consultation with Ops and the business owner, should review threshold settings monthly during the first 6 months and quarterly thereafter.

**HITL-triggering action categories** (examples — full list maintained by AIGP team):

| Tool / action | Default risk score floor | Rationale |
|---|---|---|
| `aigp.update_bank_details` | 0.90 | Irreversible; fraud vector |
| `aigp.submit_refund_request` (amount > SGD 10,000) | 0.85 | High-value irreversible financial action |
| `aigp.submit_refund_request` (amount ≤ SGD 10,000) | 0.45 | Lower value; auto-permit with audit |
| `aigp.close_case` | 0.72 | Consequential status change |
| `aigp.get_taxpayer_record` | 0.15 | Read-only; low risk |
| `aigp.send_email_reply` | 0.20 | Low-risk communication action |

---

#### HITL Queue and Task Lifecycle

When the AIGP layer determines that an action requires HITL review, the following sequence occurs:

1. **AIGP creates a HITL task**: A JSON task document is inserted into Azure Table Storage (`hitl-tasks`):

```json
{
  "taskId": "uuid-v4",
  "messageId": "uuid (root correlation ID)",
  "agentName": "refund-agent",
  "agentSemver": "1.3.0",
  "sopId": "SOP-REFUND-001",
  "toolName": "aigp.submit_refund_request",
  "actionSummary": "Agent requests submission of refund for taxpayer [ref: sha256:...]. Amount category: HIGH. Status: PENDING_APPROVAL in system.",
  "riskScore": 0.84,
  "riskFactors": ["HIGH_AMOUNT", "TAXPAYER_DISPUTED_FLAG"],
  "createdAt": "2026-05-03T09:14:22Z",
  "slaDeadline": "2026-05-03T13:14:22Z",
  "status": "PENDING",
  "assignedOfficerId": null,
  "decision": null,
  "decisionAt": null,
  "decisionNotes": null
}
```

> **`actionSummary` content control**: The action summary surfaced to the officer **must not** contain raw taxpayer personal data. Taxpayer identity is surfaced in the review app by the officer looking up the case in the internal case management system using the taxpayer reference hash — the review app provides a deep link to the case management system for this purpose. The HITL task itself carries no PII.

2. **AIGP publishes a HITL event** to an **Azure Service Bus topic** (`hitl-tasks-{env}`). The message payload includes the `taskId` and `slaDeadline` only (not the full task document). The review app subscribes to this topic (`sub-officer-review`).

3. **The AO agent enters a polling loop**, calling the AIGP `GET /hitl/tasks/{taskId}/status` endpoint every 60 seconds. The AIGP API returns the current task status. The agent's LangGraph graph is paused at a `wait_for_hitl` node. The Service Bus message lock on the email processing message is extended (lock renewal) to prevent the original email from being dead-lettered while awaiting HITL.

   > **Lock renewal constraint**: Service Bus message lock renewal has a maximum duration enforced by the message TTL setting (7 days). At <4 hours HITL SLA (see below), this is not a concern in normal operation. For HITL tasks that breach SLA (escalated cases), the email must be manually re-enqueued after the HITL decision is made.

4. **Tax officer reviews and decides** via the HITL review app (see below).

5. **AIGP receives the decision**: The review app calls AIGP `POST /hitl/tasks/{taskId}/decision` with `{ "decision": "APPROVE" | "REJECT", "officerNotes": "..." }`. AIGP validates the officer's identity (Entra ID token), records the decision in the HITL task, and either executes the action (APPROVE) or rejects it (REJECT, returning a rejection result to the agent).

6. **Agent resumes**: The agent's polling loop detects the non-PENDING status, reads the outcome, and continues its LangGraph workflow accordingly. For REJECT: the agent generates a "case referred to officer" response to the taxpayer.

---

#### HITL Review Application

The review app is the tax officer's interface for HITL tasks.

**Option 1A (Recommended for HITL app): Custom React webapp on Azure Static Web Apps + Azure Functions API**

A purpose-built lightweight web application provides a focused HITL review experience:

- **Task inbox**: Lists all PENDING HITL tasks assigned to the officer's team, sorted by SLA deadline (most urgent first). Each row shows: SOP category, action type, risk score, time created, time remaining to SLA deadline.
- **Task detail view**: Full `actionSummary`, risk factor badges, and a deep link to the internal case management system (opens in a separate tab for the officer to look up the full taxpayer case before deciding).
- **Decision panel**: Two buttons — **Approve** / **Reject** — with a mandatory free-text notes field (minimum 20 characters; officers must articulate their reasoning). Notes are stored in the HITL task record.
- **Claim and release**: Officers "claim" a task before deciding, preventing two officers from simultaneously reviewing the same task. Claimed tasks auto-release after 30 minutes of inactivity.
- **SLA countdown**: Visual countdown timer per task, turning amber at < 1 hour and red at < 15 minutes remaining.

Backend: Azure Functions (Python) for HITL task read/write to Table Storage and AIGP API calls. Deployed on Azure Container Apps (consistent with the platform's ACA-first approach).

**Pros**: Purpose-built UX for the specific HITL workflow; no Power Platform licensing concerns; fully within GCC 2.0 as an ACA-hosted web app; officer experience is focused and fast (designed to complete a HITL review in < 2 minutes per task).

**Cons**: Requires design and build effort (estimated 2–3 weeks for an MVP by a frontend engineer). Not as quickly configurable as Power Apps.

**Option 1B: Power Apps Canvas App**

A Power Apps canvas app with a SharePoint list as the task store, sending notifications via Power Automate.

**Pros**: Faster initial build (Power Apps low-code); Power Automate provides push notifications to the officer's Teams channel when a new HITL task is assigned.

**Cons**: Power Apps / Power Automate licencing in GCC 2.0 must be confirmed (Power Platform Government Community Cloud has specific service availability). SharePoint as a task store does not natively integrate with Service Bus, requiring a Power Automate middleware layer. Less control over UX for a workflow where speed matters.

**Recommendation for review app**: **Option 1A** is recommended if GCC 2.0 Power Platform availability cannot be confirmed within the project timeline. Teams notification integration (a strong UX feature of Option 1B) can be replicated in Option 1A via an Azure Logic Apps workflow that posts an adaptive card to the officer's Teams channel on new HITL task creation.

---

#### HITL SLA & Timeout Escalation

**Proposed HITL SLA**: HITL tasks for the AIBP platform are categorised into two SLA tiers:

| Tier | Trigger criteria | Officer response SLA | Breach action |
|---|---|---|---|
| **Tier 1 — Standard** | Risk score 0.70–0.84; non-irreversible action | 4 business hours | Auto-escalate to supervisor; notify Ops |
| **Tier 2 — Urgent** | Risk score 0.85–0.89; or any action involving taxpayer-disputed accounts | 1 business hour | Auto-escalate to supervisor + branch manager; notify Ops + AIGP team |
| **Auto-Block** | Risk score ≥ 0.90 | No human review — immediately rejected and routed to human officer queue | N/A |

**Timeout enforcement**: An **Azure Logic Apps** recurrence workflow runs every 15 minutes and queries the `hitl-tasks` Table Storage for tasks approaching or past their `slaDeadline`:

- **At SLA − 30 minutes**: Posts a Teams adaptive card reminder to the officer's channel
- **At SLA breach**: Updates the task `status` to `SLA_BREACHED`; posts a Teams alert to the supervisor; writes a metric event to Azure Monitor (`hitl.sla.breached`, with `tier` and `agentName` dimensions)
- **At SLA + 2 hours**: If still unresolved — escalates to the branch manager; creates a P2 incident ticket in the ops incident management flow; the AIGP API begins returning `HITL_TIMEOUT` to the waiting agent, which triggers the agent to send a "case under review" interim response to the taxpayer

**HITL Capacity Monitoring**:

Azure Monitor tracks the following HITL operational metrics, published to the **HITL Operations Dashboard** (Azure Monitor Workbook):

| Metric | Alert threshold |
|---|---|
| `hitl.queue.depth` — pending HITL task count | > 20 pending at any time → P3 alert to Ops |
| `hitl.sla.breachRate` — % of tasks breaching SLA (rolling 7 days) | > 10% → Ops + AIGP team notified |
| `hitl.review.durationMean` — mean officer review time in minutes | Tracked; no hard alert (informational for capacity planning) |
| `hitl.escalation.rate` — % of HITL tasks re-escalated to supervisor | > 20% rolling 7 days → Ops + business owner review |
| `hitl.officer.taskLoad` — tasks per officer per day | > 15 tasks/officer/day → P3 alert to Ops (capacity risk) |

**Pros**:
- Risk-based escalation targets HITL effort at genuinely high-risk actions; low-risk read actions never enter the queue
- Auto-Block for the highest-risk category prevents dangerous actions without requiring human availability at all times
- SLA enforcement automation prevents HITL tasks from silently ageing without resolution
- Capacity monitoring enables proactive staffing decisions before HITL becomes a throughput bottleneck

**Cons**:
- Risk threshold calibration is critical; if set too loosely, HITL volume overwhelms officers; if too tightly, genuinely risky actions pass auto-permitted thresholds
- The polling loop in the AO agent (every 60 seconds) means the agent occupies a Service Bus message lock and an ACA container slot while waiting; at <1,000 email/day this is manageable, but must be monitored
- AO agent timeout handling must be robust — if the ACA container is recycled (e.g., during a deployment) while an agent is in a HITL wait state, the in-progress state must survive the restart

---

### Option 2: Mandatory Review for All Write Actions

**Implementation**: Every tool call that involves a write or execute permission (regardless of risk score or amount) is held for HITL review before execution.

**Pros**: Maximum safety guarantee; simplest risk threshold to define and audit

**Cons**:
- At <1,000 emails/day, if even 30% of emails involve a write action, this creates ~300 HITL tasks/day requiring officer attention — at 2 minutes/review, that is ~10 hours of officer HITL work daily, effectively eliminating the automation benefit of the platform
- Contradicts the core value proposition of the system: automated resolution of routine tax queries. Taxpayers would experience multi-hour delays for even simple refund status updates if a write action is always gated
- Officers reviewing high volumes of mostly-routine actions will experience alert fatigue, degrading the quality of review for genuinely risky escalations

---

### Option 3: Periodic Batch Review (Deferred HITL)

**Implementation**: Agents execute all actions immediately (no real-time hold). A nightly batch job collects all write actions from the audit log and presents them to officers the next morning for retrospective review, flagging anomalies for follow-up.

**Pros**: No latency impact on the taxpayer experience; zero pause in the agent workflow

**Cons**:
- Actions are executed before human review — a wrong or fraudulent action cannot be prevented, only identified after the fact. For irreversible actions (bank detail changes, disbursements), this is operationally and legally unacceptable
- Does not constitute a genuine HITL control; would not satisfy any definition of "human oversight of AI-driven decisions" under governance frameworks

---

### Recommendation Justification

**Option 1** is recommended. Risk-based triggering is the only approach that balances automation efficiency with genuine human oversight. It concentrates officer attention on the actions where human judgement is materially important — irreversible, high-value, or anomalous actions — while allowing the platform to deliver its automation benefit on the majority of routine, low-risk interactions.

> **Compliance Note (ISO 27001)**: HITL records (task ID, action summary, risk score, officer decision, officer ID, timestamp, notes) constitute the human oversight audit trail for AI-driven decisions on taxpayer records. This record must be retained for the same period as the Cosmos DB audit ledger (Step 2.6) — minimum 5 years per IM8.

> **Compliance Note (PDPA)**: The HITL task document must not contain personal data. The `actionSummary` field is the highest-risk field; it must be reviewed by the App & Data and AIGP teams to ensure AIGP generates summaries that describe the action category, tool name, and risk factors without including taxpayer personal data (name, NRIC, address, financial account details). Taxpayer context must be accessed only by the officer from the internal case management system using the taxpayer reference, not from the HITL task document itself.

---

## 3.2 Incident Response & Emergency Stop

### Implementation Overview

Despite all preventive controls — evaluation gates, canary deployments, HITL, behavioral anomaly detection — production incidents will occur. The Emergency Stop system is the last line of defence: the ability to halt agent activity instantly, at any granularity, without requiring a code deployment or manual infrastructure intervention.

**Emergency Stop must satisfy the following requirements**:

1. **Instant effect**: Traffic to a specific agent (or all agents) must stop within **≤ 30 seconds** of the stop command being issued. This is a hard operational requirement.
2. **Granular control**: Stops must be applicable at three levels: (a) a specific agent version, (b) a specific agent (all versions), (c) all agents platform-wide.
3. **No deployment required**: The stop mechanism must be activatable by an authorised Ops or AIGP team member without triggering a CI/CD pipeline run or requiring code changes.
4. **Automatically triggered**: The stop mechanism must also be triggerable automatically by the behavioral anomaly detector (Step 2.7) and the AIGP policy engine — human availability cannot be assumed at the moment of a critical incident.
5. **Auditable**: Every stop event (who triggered it, when, at what scope, and why) must be recorded.
6. **Safe resume**: After an emergency stop, resuming agent traffic must require an explicit positive action and must not occur automatically.

---

### Option 1 (Recommended): Azure App Configuration Feature Flag Kill Switch + Azure Automation Runbook + Service Bus Drain

**Technology Stack**: Azure App Configuration (feature flags), Azure Automation (runbook), Azure Service Bus (topic subscription management), Azure Monitor (alert-to-runbook trigger), Azure Key Vault (runbook credential management)

---

#### Kill Switch Architecture

**Kill switch hierarchy** — three levels, independently controllable:

| Level | Flag key in Azure App Configuration | Scope | Example use |
|---|---|---|---|
| **Agent-version kill** | `agents:{agentName}:{semver}:enabled` | Single agent version | Roll back a specific canary revision that is producing anomalies |
| **Agent kill** | `agents:{agentName}:enabled` | All versions of a specific agent | Stop all processing for a specific SOP category (e.g., refund agent has a critical fault) |
| **Platform kill** | `agents:platform:enabled` | All agents, all SOP categories | Catastrophic failure across the platform; full halt |

Azure App Configuration is selected for kill switches because:
- It supports **feature flag change propagation in < 5 seconds** to all connected clients (using Azure App Configuration's event-driven push model with Azure Event Grid)
- It is fully within the GCC 2.0 boundary and accessible over private endpoints
- It is independent of the ACA deployment pipeline — no deployment is required to change a flag value
- Access to modify flag values is controlled by Azure RBAC (Ops and AIGP team leads only, via Entra ID-authenticated Azure App Configuration data owner role)

**Kill switch enforcement in AO agents**:

Each LangGraph agent checks the relevant kill switch flags at two points:
1. **On message dequeue**: Before beginning email processing, the agent checks `agents:platform:enabled` and `agents:{agentName}:enabled`. If either is `false`, the message is immediately released back to the Service Bus queue (with a delivery count increment) and the container app scales to zero replicas (via ACA scale rule on the App Configuration flag event).
2. **Before each AIGP tool call**: The agent checks `agents:{agentName}:{semver}:enabled`. If `false`, the in-progress workflow is halted, the tool call is not made, and the email is re-queued for processing by a different version (if available) or routed to the human officer queue.

> **App Configuration SDK polling vs. push**: The Azure App Configuration SDK supports both polling (checking flags on a schedule) and event-driven refresh. For kill switch latency requirements (≤ 30 seconds), the AO agent must use the **event-driven refresh** mode, where the SDK subscribes to App Configuration change events via Azure Event Grid and updates the in-process flag value without waiting for a poll cycle. This must be specified in the AO team's agent implementation requirements.

---

#### Automated Kill Switch Trigger (Runbook)

Emergency stops can be triggered in three ways:

**Trigger 1 — Manual (Ops/AIGP team operator)**:
- Operator authenticates to the Azure portal or uses the Azure CLI with their Entra ID credentials
- Operator sets the relevant flag to `false` in Azure App Configuration
- Effect propagates to all running agent clients within ≤ 30 seconds via Event Grid push

**Trigger 2 — Automated (from anomaly detector — Step 2.7)**:
- The `ops-anomaly-events` Service Bus topic (from the anomaly detection job) is monitored by an **Azure Monitor alert rule** on the custom metric `ao.anomaly.rate`
- When the anomaly rate for a specific agent exceeds 50% over any 30-minute window, the alert fires an **Azure Automation Runbook** (`ops-emergency-stop.ps1`):

```powershell
# ops-emergency-stop.ps1 (simplified)
param(
    [string]$AgentName,
    [string]$Semver,
    [string]$TriggerReason,
    [string]$TriggerSource  # 'anomaly_detector' | 'aigp_policy' | 'manual'
)

# Authenticate using Automation Account Managed Identity
Connect-AzAccount -Identity

# Set the kill switch flag
$appConfigEndpoint = "https://aibp-appconfig-prod.azconfig.io"
Set-AzAppConfigurationKeyValue `
    -Endpoint $appConfigEndpoint `
    -Key "agents:$AgentName`:$Semver`:enabled" `
    -Value "false" `
    -Label "prod"

# Write stop event to Cosmos DB audit ledger
$stopEvent = @{
    id          = [System.Guid]::NewGuid().ToString()
    eventType   = "emergency_stop"
    agentName   = $AgentName
    semver      = $Semver
    scope       = "agent_version"
    triggeredBy = $TriggerSource
    reason      = $TriggerReason
    stoppedAt   = (Get-Date -Format "o")
    status      = "active"
}
# [Cosmos DB write via REST — implementation detail omitted]

# Notify Ops team via Azure Monitor Action Group (Teams + email)
Send-AzMonitorActionGroupNotification `
    -ActionGroupId "/subscriptions/.../actionGroups/ops-critical-alerts" `
    -AlertText "EMERGENCY STOP activated for $AgentName $Semver. Reason: $TriggerReason. Source: $TriggerSource."

Write-Output "Emergency stop activated for $AgentName $Semver at $(Get-Date -Format o)"
```

**Trigger 3 — Automated (from AIGP policy engine)**:
- The AIGP layer detects a policy-violating pattern (e.g., an agent attempting to call a tool outside its declared capability manifest, or a jailbreak attempt detected in tool call parameters)
- AIGP calls the AIGP-to-Ops integration endpoint (`POST /ops/emergency-stop`) authenticated with its Managed Identity
- This endpoint (an Azure Function) triggers the same Automation Runbook as Trigger 2, with `TriggerSource = 'aigp_policy'`

---

#### Service Bus Drain

Activating a kill switch stops new email processing but does not affect the ~20 emails already in-flight (agent containers currently processing a message and holding a lock). For a platform kill:

1. The runbook additionally calls the **Azure Service Bus Management API** to set the topic to **Disabled** mode — this stops new messages from being enqueued by SWEE and stops AO consumers from dequeuing new messages
2. In-flight messages with active locks will either: (a) complete processing in the next ≤ 5 minutes (normal processing time), or (b) be abandoned by the agent on its next App Configuration flag check, returning them to the queue
3. After all in-flight locks expire (within 5 minutes), the queue is effectively drained — no new processing occurs

**Resume procedure** (explicit positive action required):

A platform resume must follow a defined checklist:

1. Ops team documents the incident root cause in the incident management system
2. AO team or AIGP team (depending on root cause) confirms the issue is resolved
3. Ops team lead and AIGP team lead both approve the resume in Azure DevOps (parallel approval task)
4. Ops operator re-enables the flag in Azure App Configuration
5. Service Bus topic is re-enabled
6. Ops team monitors the AO Operations Dashboard for 30 minutes post-resume

The resume is logged in the emergency stop Cosmos DB record (`status` updated to `"resumed"`, `resumedAt` and `resumedBy` fields populated).

---

#### Kill Switch Event Audit Record (Cosmos DB, `ops-emergency-events` container)

| Field | Description |
|---|---|
| `id` | UUID |
| `eventType` | `emergency_stop` or `emergency_resume` |
| `agentName` | Agent name (or `"PLATFORM"` for platform kill) |
| `semver` | Agent version (or `"ALL"`) |
| `scope` | `agent_version` / `agent` / `platform` |
| `triggeredBy` | `anomaly_detector` / `aigp_policy` / `manual` |
| `triggeredByIdentity` | Entra ID UPN (for manual) or service principal name (for automated) |
| `reason` | Free-text or structured reason code |
| `stoppedAt` | ISO 8601 timestamp |
| `resumedAt` | ISO 8601 timestamp (populated on resume) |
| `resumedBy` | Entra ID UPN of approving Ops team lead |
| `messagesInFlightAtStop` | Count of messages held by active locks at stop time |
| `affectedEmailCount` | Count of emails that were re-queued or routed to human queue as a result |

**Pros**:
- Azure App Configuration flag propagation in ≤ 5 seconds meets the 30-second kill latency requirement with significant margin
- Three independently controllable levels prevents over-stopping (stopping the whole platform when only one agent has an issue)
- Automated runbook means the kill switch triggers even if no Ops engineer is online at the moment of the anomaly
- Full audit record in Cosmos DB satisfies ISO 27001 incident management record requirements
- Explicit double-approval resume procedure prevents premature restart

**Cons**:
- App Configuration event-driven refresh requires specific SDK configuration in agent code; must be validated during the CI pipeline (a test that verifies the agent responds to a flag change within 30 seconds in the test environment)
- Service Bus Disabled mode during a platform kill may affect SWEE's ability to queue emails; SWEE must handle Service Bus publish failures gracefully (dead-letter SWEE-side, or queue in memory briefly) — this must be specified to the SWEE team
- Automation Runbook must be tested periodically (at minimum quarterly in the test environment) to confirm that the kill switch end-to-end path is functional

---

### Option 2: Azure API Management Policy Kill Switch

**Implementation**: APIM inbound policies for the AIGP API are updated to return HTTP 503 for all agent requests when a kill switch is needed. APIM policy changes are applied via the Azure REST API.

**Pros**: Centralised at the APIM layer; does not require any agent-side SDK changes

**Cons**:
- APIM policy propagation is not instant — policy changes can take 30–60 seconds to propagate across APIM gateway nodes, which may not meet the ≤ 30-second requirement in all scenarios
- APIM kill stops agents at the AIGP API call layer only — agents that have already dequeued a message and are performing LLM reasoning steps (before reaching a tool call) will continue running until they attempt a tool call, which could be several seconds to minutes later
- Does not address the SWEE-to-Service-Bus ingestion path; new emails continue to be enqueued and dequeued by agents even if AIGP calls are blocked — this creates a processing pile-up where agents loop retrying AIGP calls

---

### Option 3: ACA Scale-to-Zero Force

**Implementation**: Force all ACA Container Apps to scale to zero replicas immediately by updating the scale rule via the Azure ARM API.

**Pros**: Definitive — agents cannot run if containers are stopped

**Cons**:
- ACA scale-to-zero is not instant; it takes 1–3 minutes for running containers to complete their current work and drain before scaling down
- Does not constitute a true "emergency stop" for in-flight transactions already in progress
- When the kill is lifted, containers take additional startup time before resuming — the restart can take 30–60 seconds per container, delaying full resume
- Service Bus messages remain locked to the containers that were running; message locks expire naturally (in ≤ 5 minutes), but this creates a window of uncertainty about message state

---

### Recommendation Justification

**Option 1** is recommended as the primary kill switch, with **Option 3 (ACA scale-to-zero) as a secondary action** invoked 5 minutes after Option 1 to ensure that all agent containers are definitively stopped even if a container fails to respond to the App Configuration flag change. The two-layer approach combines the speed of App Configuration flag propagation (< 5 seconds for the intent to stop) with the finality of container termination (for persistent agents that may have a bug in their flag-check implementation).

> **Compliance Note (ISO 27001)**: ISO 27001 control A.16 (Information Security Incident Management) requires documented procedures for incident response, including records of incidents, remediation actions, and lessons learned. The emergency stop Cosmos DB record, combined with the Azure DevOps double-approval resume trail, constitutes the incident management record for Emergency Stop events.

> **Compliance Note (IM8)**: IM8 requires that government systems maintain the ability to immediately isolate and contain a compromised or malfunctioning service. The Emergency Stop system fulfils this requirement for the AI agent layer. The Ops team must maintain a documented Emergency Stop runbook (see Step 7.2) and conduct at least one Emergency Stop drill per quarter in the test environment.

---

## 3.3 RiskOps

### Implementation Overview

RiskOps is the ongoing risk governance function for the AIBP platform. It is distinct from the reactive operations described in Steps 3.1 and 3.2 — RiskOps is proactive and continuous: monitoring the platform's risk posture, enforcing policies as code, and maintaining compliance evidence for regulatory and internal audit purposes.

RiskOps has three components:
1. **Policy governance**: Defining and enforcing what agents are and are not permitted to do, as machine-readable policy code
2. **Security posture management**: Continuous monitoring of the platform's security health against known threat patterns
3. **Compliance evidence management**: Maintaining the records and controls required by PDPA, IM8, and ISO 27001

---

### Option 1 (Recommended): Open Policy Agent (OPA) + Azure Policy + Microsoft Defender for Cloud

**Technology Stack**: Open Policy Agent (OPA, deployed in AIGP layer on AKS), Azure Policy, Microsoft Defender for Cloud (CSPM), Azure Security Center, Azure Policy (guest configuration), Azure Monitor

---

#### Open Policy Agent (OPA) — Agent Action Policy Enforcement

OPA is an open-source (CNCF graduated), general-purpose policy engine. In the AIBP context, OPA is deployed as a sidecar or standalone microservice within the AIGP layer, responsible for evaluating whether a specific agent action is permitted under the platform's policy rules.

Every AIGP tool call evaluation is routed through OPA before execution:

```
Agent tool call request → AIGP API → OPA policy evaluation → [PERMIT / DENY / HITL] → Execute or hold
```

**Why OPA over custom AIGP rule logic**:
- OPA policies are written in **Rego** (a purpose-built policy language), which is declarative, version-controlled, and testable. Policies live in the agent repository (or a shared policy repository) alongside the agent code — changes go through the same CI/CD review process as code changes.
- OPA policies are evaluated in microseconds (typically < 1 ms for a policy decision). They do not add meaningful latency to the AIGP approval path.
- OPA produces a **decision log** (structured JSON) for every evaluation — this log includes the input (agent identity, tool name, parameters, risk score) and the output (permit/deny/hitl, matched rule, rule rationale). This decision log is the audit evidence trail for OPA.

**Sample OPA policies (Rego)**:

```rego
package aibp.aigp.policy

# Default deny
default allow = false
default require_hitl = false

# Permit read-only actions for any active agent
allow {
    input.permission_scope == "read"
    agent_is_active(input.agent_name, input.agent_semver)
}

# Permit write actions below risk threshold for declared agents
allow {
    input.permission_scope == "write"
    input.risk_score < 0.70
    agent_has_tool_permission(input.agent_name, input.agent_semver, input.tool_name)
    not input.taxpayer_disputed_flag
}

# Require HITL for elevated risk write actions
require_hitl {
    input.permission_scope == "write"
    input.risk_score >= 0.70
    input.risk_score < 0.90
    agent_has_tool_permission(input.agent_name, input.agent_semver, input.tool_name)
}

# Always deny bank detail modifications regardless of risk score
# (HITL is insufficient for this action category — requires offline process)
deny_always {
    input.tool_name == "aigp.update_bank_details"
    not input.offline_approval_ref  # must be pre-approved via offline process
}

# Deny if agent is calling a tool not in its declared manifest
deny_tool_not_in_manifest {
    not agent_has_tool_permission(input.agent_name, input.agent_semver, input.tool_name)
}

# Helper: check agent is in registry with active status
agent_is_active(agent_name, semver) {
    data.agent_registry[agent_name][semver].status == "active"
}

# Helper: check tool is in agent's declared capability manifest
agent_has_tool_permission(agent_name, semver, tool_name) {
    data.agent_registry[agent_name][semver].tools[_].name == tool_name
}
```

**OPA data synchronisation**: The `data.agent_registry` used in OPA policies is populated by syncing the PostgreSQL registry DB (Step 2.1) to OPA's data API at every agent deployment and on a 5-minute refresh schedule. This ensures OPA has current knowledge of which agents are active and what tools they are permitted to call.

**OPA decision log → Cosmos DB**: OPA's decision log is written to the same Cosmos DB audit ledger (`agent-audit-log`, Step 2.6) as a `policy_decision` record type, keyed to the same `messageId` correlation ID. This creates a complete action-level audit chain: every tool call in the Cosmos DB audit document has a corresponding OPA policy decision record.

---

#### Azure Policy — Infrastructure-Level Guardrails

Azure Policy enforces guardrails on the Azure infrastructure resources that constitute the AIBP platform. These are preventive controls that ensure infrastructure configuration cannot drift into a non-compliant state.

**Key policy assignments** for the AIBP resource group:

| Policy | Effect | Rationale |
|---|---|---|
| Require Private Endpoints for Azure Service Bus namespaces | Deny | GCC 2.0 network isolation |
| Require Private Endpoints for Azure Cosmos DB accounts | Deny | GCC 2.0 network isolation |
| Require Customer-Managed Keys for Azure Cosmos DB | Deny | IM8 encryption requirement |
| Deny public network access to Azure Container Registry | Deny | Prevent image pull from outside GCC 2.0 VNet |
| Require Managed Identity for ACA Container Apps | Audit + Deny | No service principal credentials in app configuration |
| Require approved container registry sources for ACA | Deny | ACA can only pull images from the designated internal ACR |
| Require diagnostic settings enabled for Service Bus, Cosmos DB | Deny | Ensure all resources emit logs to Log Analytics |
| Allowed Azure regions: {GCC 2.0 allowed regions only} | Deny | Prevent data residency violations |

Azure Policy initiative assignments are managed via Terraform in the platform's IaC repository (owned by Platform & Infra team). RiskOps reviews the policy compliance report in the Azure Policy compliance dashboard monthly.

---

#### Microsoft Defender for Cloud — Security Posture Management

Microsoft Defender for Cloud (MDC) provides continuous Cloud Security Posture Management (CSPM) for the AIBP Azure resources. MDC is enabled on the GCC 2.0 subscription with the **Defender for Containers** and **Defender for Servers** plans.

**AIBP-specific MDC configuration**:

- **Defender for Containers**: Scans ACA container images in ACR for known CVEs before deployment. The CI pipeline is configured to fail if MDC returns any HIGH or CRITICAL severity vulnerabilities on the agent container image.
- **Defender for App Service / ACA**: Runtime threat detection for the ACA environment — alerts on unusual process activity, suspicious outbound connections, and container escape attempts.
- **Defender CSPM**: Generates a **Secure Score** for the subscription with remediation recommendations. RiskOps reviews the Secure Score dashboard weekly and tracks open recommendations in the risk register.

**MDC alerts forwarded to SOC**: MDC security alerts (not AIBP application events) are forwarded to the Internal SOC and GovTech SOC via the SOC's connected workspace. These are infrastructure-level security alerts (threat detection, vulnerability findings) and do not contain taxpayer data — they are distinct from the application audit events in the Event Hubs SOC stream (Step 2.6).

---

#### Compliance Evidence Management

RiskOps maintains a **Compliance Evidence Repository** — a structured Azure Blob Storage container (`ops-compliance-evidence`) that aggregates the artefacts required by PDPA, IM8, and ISO 27001 assessors. The evidence is organised as:

```
ops-compliance-evidence/
├── pdpa/
│   ├── dpia/                  # Data Protection Impact Assessments (per MAJOR agent version change)
│   ├── data-flow-diagrams/    # Current and versioned data flow diagrams
│   └── consent-records/       # Processing basis documentation
├── im8/
│   ├── audit-log-retention/   # Evidence of 5-year retention policy (screenshot + policy config)
│   ├── incident-reports/      # Post-incident reports from Emergency Stop events (Step 7.2)
│   ├── penetration-test/      # Annual penetration test reports
│   └── backups/               # Backup completion reports (monthly)
└── iso27001/
    ├── change-records/        # Azure DevOps pipeline approval exports (monthly batch)
    ├── access-reviews/        # Quarterly Azure RBAC access review reports
    ├── risk-register/         # Current risk register (reviewed quarterly)
    └── supplier-assessments/  # Azure / GovTech supplier risk assessments
```

A monthly **Azure Logic Apps** workflow generates and deposits the following evidence artefacts automatically:
- Azure Policy compliance report export (PDF)
- Azure RBAC role assignment export (CSV) for quarterly access review
- Azure DevOps pipeline approval logs for the month (production deployments and Emergency Stop resumes)
- Cosmos DB audit log row count and retention verification report

**Pros**:
- OPA provides a machine-readable, version-controlled policy contract that is auditable, testable, and transparent — superior to ad-hoc rule logic embedded in the AIGP application code
- Azure Policy drift prevention ensures that infrastructure changes (even well-intentioned ones) cannot inadvertently weaken security controls
- MDC Defender for Containers provides pre-deployment CVE scanning natively within the ACA/ACR pipeline, satisfying IM8 vulnerability management requirements
- Compliance evidence repository automation reduces the manual effort of collating audit evidence at assessment time from days to hours

**Cons**:
- OPA Rego requires developer familiarity; the AIGP team must nominate a Rego policy owner who is responsible for maintaining policy accuracy as agents and tools evolve
- OPA data synchronisation with the PostgreSQL registry introduces a 5-minute potential lag between an agent version being deprecated in the registry and OPA's data store reflecting this — an agent version that is deprecated in the registry could make a tool call within the 5-minute window before OPA detects the deprecation. Mitigation: on agent deprecation, the App Configuration kill switch (Step 3.2) is activated simultaneously with the registry update, which prevents the agent from processing any new messages before OPA catches up.
- Azure Policy Deny effects on existing non-compliant resources trigger remediation tasks; these must be tracked and resolved by the Platform & Infra team before production deployment

---

### Option 2: Custom AIGP Application-Level Rule Engine

**Implementation**: The AIGP team builds a custom rule engine (e.g., a Python-based rule evaluation module embedded in the AIGP API service) that evaluates agent action requests against a set of rules defined in a configuration file or database table.

**Pros**: No OPA dependency; rules are in Python, which the AIGP team is familiar with; no new technology to learn

**Cons**:
- Rules embedded in application code are significantly harder to audit than Rego policies — they require reading Python source code to understand what is and is not permitted. OPA policies are a self-documenting, declarative format designed for exactly this purpose.
- Custom rule engines are not tested in isolation — a bug in the Python rule logic is not caught until integration testing. OPA ships with a testing framework (`opa test`) that allows policies to be unit-tested independently of the application.
- Does not produce a structured decision log natively; the AIGP team must implement custom logging equivalent to OPA's built-in decision log
- Cannot easily be version-controlled with the same CI/CD gate as agent code; rules embedded in application code require a full application deployment to update

---

### Recommendation Justification

**Option 1** is recommended. The combination of OPA (agent action governance), Azure Policy (infrastructure governance), and Defender for Cloud (security posture) provides defence-in-depth at three distinct layers: what agents are permitted to do (OPA), how the platform is configured (Azure Policy), and whether the platform is being attacked or misconfigured (MDC). These are complementary, not redundant. OPA in particular operationalises the principle that governance rules are code: version-controlled, reviewable, testable, and auditable by assessors without requiring them to read application source code.

> **Compliance Note (PDPA)**: The DPIA process must be triggered for any OPA policy change that alters the data processing permissions of agents (e.g., a new tool permission that enables agents to access a new data category). DPIAs are stored in the compliance evidence repository under `/pdpa/dpia/`.

> **Compliance Note (IM8)**: IM8 Clause 5 (System Security Management) requires that all IT systems undergo vulnerability assessment before major deployments. MDC Defender for Containers automated CVE scanning satisfies this requirement for container images. The CI pipeline must be configured to also run `az acr check-health` and review the MDC scan report before promoting from test to pre-prod.

> **Compliance Note (ISO 27001)**: OPA policy files and their change history (Git blame, pull request reviews) constitute the documentation required for ISO 27001 control A.18.1.1 (Identification of Applicable Legislation and Contractual Requirements) and A.18.2 (Information Security Reviews).
