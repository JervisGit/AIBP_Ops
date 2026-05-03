# AIBP Operations Blueprint
## Step 7: People, Process & Governance Operations

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Defining who does what — the roles, responsibilities, access boundaries, day-to-day operating model, and the runbooks and escalation playbooks that govern how teams act during normal operations, incidents, and planned changes.

**Why people and process matter for agentic AI operations**: The operational model for an AI agent platform is fundamentally different from traditional application operations. The platform is probabilistic, not deterministic — it cannot be fully tested, and unexpected behaviours in production are expected, not exceptional. The operating model must therefore create clear accountability for quality (not just availability), clear channels for cross-team escalation, and disciplined runbooks that prevent uncoordinated responses when something goes wrong.

---

## 7.1 Roles & Responsibilities

### Core Teams and Operational Interfaces

The AIBP platform spans five teams. Ops must interact with all of them. The following RACI framework defines who is Responsible, Accountable, Consulted, and Informed for key operational events.

---

#### Team Descriptions (Operations Perspective)

**Operations Team (this team)**
- Own: Platform health monitoring, SLA reporting, FinOps, HITL queue management, Emergency Stop activation, feedback pipeline, post-mortem management, compliance evidence repository
- Day-to-day: Monitor dashboards, respond to alerts, manage HITL SLA, coordinate cross-team incident response, generate monthly SLA and FinOps reports
- Access: Read access to all monitoring systems (Azure Monitor, Langfuse, Cosmos DB audit ledger, Service Bus metrics). Write access to: App Configuration (kill switches), Automation Runbooks, incident management, compliance evidence store. No access to agent source code, AIGP policy code, or internal microservices.

**App & Data — AO Team**
- Own: Agent source code, `agent-manifest.json`, CI/CD pipeline, agent evaluation logic, LangGraph graph design
- Day-to-day: Develop and deploy agent updates, respond to evaluation failures in CI, review feedback datasets (from Step 6.1), release patches for quality regressions
- Access: ACR push, ACA deployment (via Azure DevOps CI/CD service principal), Langfuse (eval dataset management), Registry DB (read + insert via pipeline service principal). No production ACA admin access outside the CI/CD pipeline.

**App & Data — SWEE Team**
- Own: SWEE application code, SOP vector index in Azure AI Search, email ingestion pipeline, triage model
- Day-to-day: Monitor SWEE triage accuracy metrics (Step 1.2), respond to embedding drift alerts, review `swee-triage-failures` Langfuse dataset weekly
- Access: Azure AI Search (SOP index management), SWEE App Service / ACA deployment. No access to AO or AIGP systems.

**AIGP Team**
- Own: AIGP API codebase, OPA policy definitions (Rego), risk scoring configuration, HITL trigger thresholds, ForgeRock/Entra ID integration configuration
- Day-to-day: Review OPA policy effectiveness (Step 3.3), manage HITL threshold calibration, respond to AIGP policy-triggered Emergency Stop events, review `ao-hitl-rejections` dataset biweekly
- Access: AIGP API deployment, OPA policy repository, ForgeRock admin console (AuthZ policy). No access to AO agent source code or SWEE.

**Tax Officers (Business)**
- Own: HITL review decisions, SWEE audit corrections, Human Officer Queue manual resolutions
- Day-to-day: Process HITL tasks (within SLA tiers from Step 3.1), review weekly SWEE audit sample (within Ops-managed audit capacity), resolve emails in the Human Officer Queue
- Access: HITL review app (read/write HITL tasks), SWEE audit tool (read email excerpts, write officer label), Human Officer Queue tab (read/write). Read-only access to the MAS internal case management system via deep links from the review app. No direct access to any AIBP infrastructure systems.

**Internal SOC and GovTech SOC**
- Own: Security monitoring of AIBP infrastructure (not application logic)
- Day-to-day: Monitor MDC alerts, review Event Hubs anonymised application telemetry stream (Step 2.6), respond to infrastructure-level threats
- Access: Event Hubs capture storage (read only, anonymised telemetry), MDC alert feed (read). No access to Langfuse, Cosmos DB audit ledger, or any system containing agent reasoning data or email content.

---

#### RACI Matrix — Key Operational Events

| Event | Ops | AO Team | SWEE Team | AIGP Team | Tax Officers |
|---|---|---|---|---|---|
| New agent version deployed to production | I | R/A | I | C | — |
| HITL task created (agent action escalated) | I | I | — | I | R (review) |
| HITL task SLA breached | A | I | — | I | R (escalate) |
| Emergency Stop activated (automated) | R (monitor + resume procedure) | C (root cause) | — | C (if AIGP-triggered) | — |
| Emergency Stop activated (manual) | R/A | C | — | C | — |
| Platform resume after Emergency Stop | **A (double approval)** | R (confirm fix) | — | R (if AIGP policy involved) | — |
| Triage accuracy drop alert | A (report + coordinate) | — | R (investigate) | — | C (audit data) |
| Behavioral anomaly detection alert | A (monitor + escalate) | R (investigate) | — | C | — |
| Agent version evaluation failure (CI) | I | R/A | — | C (if tool permission issue) | — |
| DLQ spike | R/A (runbook) | C | C | — | I (new tasks in manual queue) |
| Monthly SLA report | R/A | I | I | I | I |
| Monthly FinOps report | R/A | I | — | — | — |
| Compliance audit preparation | R/A | C | C | C | C |
| Penetration test (IM8 annual) | C | C | C | C | — |
| OPA policy change | I | — | — | R/A | — |
| Agent capability scope expansion (MAJOR version) | C (DPIA trigger check) | R/A | — | A (AIGP tool registration) | — |

---

#### Ops Team Structure & Staffing

For a platform processing <1,000 emails/day in a steady-state regime, the Ops function does not warrant a dedicated 24/7 NOC. The recommended model is:

**Day operations (business hours, 08:00–18:00 SGT, Mon–Fri)**:
- 1 × **Ops Lead** — owns the operational runbooks, manages team, chairs weekly platform health review, signs off platform resumes after Emergency Stop
- 2 × **Ops Engineers** — monitor dashboards, respond to alerts, manage HITL SLA, generate reports, maintain compliance evidence repository

**After-hours on-call**:
- On-call rotation shared between Ops Engineers (1 per week)
- After-hours scope: P1 alerts only (platform availability, Emergency Stop). P2/P3 alerts are queued for business-hours response.
- On-call engineer has access to: Azure Monitor, App Configuration (kill switch activation), Azure Automation Runbooks. Azure DevOps pipeline approvals (for emergency resume) must involve the Ops Lead; the on-call engineer escalates immediately for Emergency Stop resume decisions.

**Tax Officer HITL capacity planning**:
- At <1,000 emails/day and a projected 10–15% HITL rate, peak HITL task load is approximately 100–150 tasks/day
- At an estimated 5–10 minutes per HITL review, this represents 8–25 officer-hours/day of HITL work
- This load must be factored into the business owner's resourcing plan; HITL is not a part-time add-on to full casework — it requires planned capacity
- The `hitl.officer.taskLoad` metric (Step 3.1) monitors per-officer load; if it consistently exceeds 15 tasks/officer/day, the Ops team escalates to the business owner for staffing review

---

### Access Control Policy Summary

The following access control principles govern all team access to AIBP systems:

| Principle | Implementation |
|---|---|
| **Managed Identities only** — no service account passwords or API keys in application code | Enforced via Azure Policy (Step 3.3) |
| **Least privilege RBAC** — each team has the minimum Azure role assignments needed for their function | Reviewed quarterly (access review report in compliance evidence repository) |
| **No shared accounts** — every human access is tied to a named individual's Entra ID UPN | Enforced via Entra ID group membership; service accounts are non-human named identities |
| **Production access is pipeline-mediated** — no direct production ACA or Service Bus write access for humans outside the Emergency Stop / runbook path | ACA production environment has no human write roles except via Azure DevOps pipeline service principal |
| **Quarterly RBAC access reviews** — all role assignments reviewed and recertified quarterly | Azure Logic Apps auto-exports role assignments; Ops Lead recertifies |

> **Compliance Note (ISO 27001)**: ISO 27001 control A.9 (Access Control) requires formal access control policies, user access provisioning and de-provisioning procedures, and periodic access reviews. The quarterly RBAC review and Entra ID-mediated access in this section satisfy these requirements.

---

## 7.2 Runbooks & Escalation Playbooks

Runbooks are step-by-step operational procedures for common and critical operational tasks. They ensure that any trained Ops engineer can perform the task consistently, without relying on institutional memory.

All runbooks are stored in the Ops team's internal wiki (Azure DevOps Wiki or equivalent). Each runbook has:
- **Trigger**: The condition that initiates the runbook
- **Prerequisites**: Systems and access required
- **Steps**: Numbered, specific, and executable
- **Verification**: How to confirm the task completed successfully
- **Rollback / undo**: How to reverse the action if something went wrong
- **Escalation**: Who to call if the runbook cannot be completed

Runbooks are reviewed and updated:
- After every incident where the runbook was used (within 5 business days of the incident post-mortem)
- On a quarterly schedule (regardless of incidents)
- When the underlying system changes in a way that affects the procedure

---

### RB-01: New Agent Version Production Deployment

**Trigger**: AO team requests promotion of a pre-prod-tested agent version to production canary  
**Prerequisite**: Stage 3 pre-prod approval from AO team lead already completed in Azure DevOps

| Step | Action | Verification |
|---|---|---|
| 1 | Confirm with AO team lead that Stage 3 evaluation passed and the version is ready for canary | AO team lead approval in Azure DevOps pipeline |
| 2 | Confirm the registry DB record for the version shows `eval_pass = true` and `deployment_status = 'candidate'` | Query: `SELECT semver, eval_pass, deployment_status FROM agent_versions WHERE agent_name = '{name}'` |
| 3 | Initiate Stage 4 production canary deployment via Azure DevOps pipeline (manual trigger) | — |
| 4 | Confirm canary ACA revision is live with 10% traffic weight | Azure portal → Container Apps → Revisions → verify new revision shows 10% weight |
| 5 | Open the AO Operations Dashboard (Azure Monitor Workbook) and verify the canary revision metrics panel is showing data | Dashboard must show both blue and canary revision side-by-side |
| 6 | Set a calendar reminder for canary review at T+24h and T+48h | — |
| 7 | At T+24h: review canary vs. blue metrics against promotion criteria (Step 2.3) | Log assessment in Azure DevOps pipeline comment |
| 8 | If criteria pass: approve full promotion (Azure DevOps pipeline gate, double-approval: Ops + AO team lead) | Pipeline promotes canary to 100%; blue retired |
| 9 | If criteria fail or ambiguous: extend canary window to T+48h and repeat assessment | Log in pipeline; notify AO team |
| 10 | Update registry DB `deployment_status` to `'active'` for new version; `'deprecated'` for previous version | Done by CI/CD pipeline on promotion; verify in registry DB |

**Escalation**: If promotion criteria fail at T+48h, invoke RB-02 (Rollback).

---

### RB-02: Agent Version Rollback

**Trigger**: Canary metrics fail promotion criteria; or P2 alert during active canary window; or Ops team decision following behavioral anomaly  
**Prerequisite**: Identify the target rollback version (the current `active` or `deprecated` version to restore)

| Step | Action | Verification |
|---|---|---|
| 1 | Identify the rollback target version (prior `active` version) | Query registry DB: `SELECT semver, aca_revision_name FROM agent_versions WHERE agent_name = '{name}' AND deployment_status = 'active' ORDER BY deployed_at DESC` |
| 2 | In Azure DevOps, trigger rollback pipeline for the agent (pre-defined pipeline: `rollback-{agentName}`) | Specify target `semver` as pipeline parameter |
| 3 | Pipeline sets canary revision traffic weight to 0% and failed revision to `deprecated` | Verify in ACA Revisions panel: canary at 0%, prior version at 100% |
| 4 | Verify that the recovery version is processing messages correctly (spot-check 5 recent trace IDs in Langfuse) | Traces should show normal tool call sequences |
| 5 | Update registry DB `deployment_status` for the failed version to `'deprecated'`; log rollback reason | Done by pipeline; verify |
| 6 | Notify AO team with: failed version, rollback target version, the specific metrics that failed, and the Langfuse trace IDs of the most anomalous cases | Teams message in `#ao-ops-alerts` channel |
| 7 | Create a post-mortem draft in Azure DevOps (link to automated post-mortem from Step 6.2 if available) | Assign to on-call engineer for completion within 5 business days |

**Escalation**: If rollback pipeline fails (e.g., prior version image no longer in ACR), contact AO team immediately to rebuild and re-push the target version image.

---

### RB-03: Emergency Stop Activation (Manual)

**Trigger**: Ops or AIGP team decision to halt agent traffic due to observed anomalous behaviour, security concern, or management direction  
**Prerequisite**: Authorisation — Ops Lead (for agent-level stop) or Ops Lead + AIGP Team Lead (for platform-level stop)

| Step | Action | Level |
|---|---|---|
| 1 | Determine the stop scope: **agent-version**, **agent-all-versions**, or **platform** | Scope decision |
| 2 | For **automated-triggered** stop: confirm the automated runbook already activated the kill switch (check Azure App Configuration for the flag value) | If already activated, skip to Step 6 |
| 3 | For **manual** stop: authenticate to Azure portal with Ops Lead Entra ID credentials | — |
| 4 | Navigate to Azure App Configuration → Configuration Explorer → set the relevant flag to `false`: `agents:{agentName}:{semver}:enabled` OR `agents:{agentName}:enabled` OR `agents:platform:enabled` | Flag set |
| 5 | Confirm propagation: within 90 seconds, the AO Operations Dashboard should show zero new processing events for the stopped agent(s) | Dashboard verification |
| 6 | For in-flight messages: wait 5 minutes for message locks to expire and messages to return to queue | Service Bus queue depth for the stopped agent's subscription should stabilise |
| 7 | For platform kill: additionally disable Service Bus topic to prevent SWEE enqueueing new messages (Azure CLI: `az servicebus topic update --status Disabled`) | Verify topic status = Disabled in Azure portal |
| 8 | Write the stop event to the Emergency Stop Cosmos DB container (automated runbook does this; for manual stops, complete the record manually): `eventType`, `agentName`, `triggeredBy = 'manual'`, `triggeredByIdentity`, `reason` | Record created |
| 9 | Notify all teams via Teams: `#ops-critical-alerts` channel; include stop scope, reason, and `stoppedAt` timestamp | Notification sent |
| 10 | Confirm with AIGP team whether any in-flight AIGP tool calls were in progress at stop time; if so, create HITL tasks for review (Step 3.1 AIGP risk flow) | HITL tasks created if needed |

**Escalation**: If App Configuration flag-setting fails, fall back to Option 3 from Step 3.2: force ACA scale-to-zero via Azure CLI (`az containerapp update --min-replicas 0 --max-replicas 0`).

---

### RB-04: Emergency Stop Resume

**Trigger**: AO or AIGP team confirms root cause is resolved; Ops Lead approves resume  
**Prerequisite**: Completed post-mortem draft with confirmed root cause; both Ops Lead and AIGP Team Lead available to approve

| Step | Action | Verification |
|---|---|---|
| 1 | Confirm the root cause has been resolved: either a rollback (RB-02) was performed, or the AIGP team has confirmed the triggering condition is no longer present | Written confirmation in the Azure DevOps post-mortem task |
| 2 | Confirm no open HITL tasks from the incident period remain unreviewed | Query `hitl-tasks` Table Storage: `WHERE status = 'PENDING' AND createdAt >= stoppedAt` |
| 3 | Double-approval in Azure DevOps (parallel approval task: Ops Lead + AIGP Team Lead must both approve) | Both approvals recorded in Azure DevOps before proceeding |
| 4 | For platform kill: re-enable Service Bus topic (`az servicebus topic update --status Active`) | Topic status = Active |
| 5 | Restore App Configuration flag(s) to `true` | Flags set |
| 6 | Monitor AO Operations Dashboard for 30 minutes post-resume: confirm normal processing metrics, no new anomaly alerts | Dashboard shows normal operations |
| 7 | Update Emergency Stop Cosmos DB record: `status = 'resumed'`, `resumedAt`, `resumedBy` | Record updated |
| 8 | Post resume notification to `#ops-critical-alerts` Teams channel | Notification sent |
| 9 | Schedule follow-up post-mortem completion review for T+5 business days | Calendar invite sent to all RACI parties |

---

### RB-05: DLQ Recovery

**Trigger**: DLQ count > 5 (automated runbook `ops-dlq-recovery.ps1` activates first); OR Azure Monitor alert to Ops team for manual review  
**Prerequisite**: Access to Azure Service Bus, Azure Table Storage (`ops-dlq-log`), Human Officer Queue

| Step | Action | Verification |
|---|---|---|
| 1 | Review the `ops-dlq-log` Azure Table Storage for the new DLQ entries; classify by `DeadLetterReason` | Categories: `MaxDeliveryCountExceeded`, `TimeToLiveExpired`, `SchemaValidationError`, `AO_UNAVAILABLE`, `AIGP_REJECTED` |
| 2 | For `AO_UNAVAILABLE` (transient): confirm the AO layer is currently healthy (AO Ops Dashboard). If healthy, re-enqueue using the DLQ runbook's re-enqueue function | Messages re-appear in main topic within 60 seconds |
| 3 | For `MaxDeliveryCountExceeded` with no AO outage: the agent rejected the message consistently. Route to Human Officer Queue. Log the `sopId` and failure pattern for AO team review in the weekly Feedback Summary (Step 6.1) | Messages in `human-officer-queue` |
| 4 | For `SchemaValidationError`: investigate SWEE message schema against the current Service Bus message schema contract. This may indicate a SWEE deployment produced a schema-breaking change. Escalate to SWEE team. | Escalation to SWEE team in `#swee-ops-alerts` channel |
| 5 | For `AIGP_REJECTED`: AIGP rejected a tool call attempt. Review the OPA decision log for the `messageId`. Escalate to AIGP team for policy review if a legitimate action type is being blocked. | Escalation to AIGP team |
| 6 | Confirm Human Officer Queue tasks are assigned (tax officers notified via Teams) | Task count in `human-officer-queue` monitored |

---

### RB-06: HITL Queue Backlog Management

**Trigger**: `hitl.queue.depth` > 20 OR HITL SLA breach rate > 10% (rolling 7 days)  
**Prerequisite**: Access to HITL task store, officer roster, Teams

| Step | Action |
|---|---|
| 1 | Query `hitl-tasks` Table Storage for tasks by `status`, `slaDeadline`, `assignedOfficerId` |
| 2 | Identify: (a) unassigned tasks, (b) tasks assigned to officers who are OOO, (c) tasks approaching SLA breach |
| 3 | Contact the business owner / supervisor to activate additional HITL capacity (if queue depth suggests a staffing shortfall) |
| 4 | For breached tasks: confirm supervisor-level escalation has occurred (automated by Logic Apps, Step 3.1). If not, manually post the Teams escalation card. |
| 5 | Review if the backlog is driven by a sudden spike in a specific SOP category's HITL rate — this may indicate an AIGP risk threshold is set too conservatively for that category. Escalate to AIGP team if 3+ consecutive days show a spike in the same category. |
| 6 | Document the backlog event in the weekly SLA report as a `Capacity Issue` note |

---

### RB-07: Quarterly SOC Log-Sharing Review

**Trigger**: Quarterly calendar reminder  
**Prerequisite**: Access to Event Hubs capture storage schema, Internal SOC and GovTech SOC contacts

| Step | Action |
|---|---|
| 1 | Review the current Event Hubs event schema against the approved SOC data-sharing agreement (the excluded fields list in Step 2.6) |
| 2 | Confirm no new event types have been added to the `ops-feedback-events` or SOC Event Hubs stream that were not part of the original agreement |
| 3 | Review the SOC's acknowledgement that no data from the Event Hubs stream has been forwarded outside the agreed boundary |
| 4 | Confirm PDPA data minimisation compliance: verify PII-stripping step is active in the feedback processor (Step 6.1) and that no officer free-text notes containing PII have surfaced in the Langfuse store (spot-check 10 recent feedback records) |
| 5 | Update the compliance evidence repository `pdpa/` section with the quarterly review record |

---

### RB-08: Quarterly RBAC Access Review

**Trigger**: Quarterly calendar reminder (aligned to financial quarter)  
**Prerequisite**: Access to compliance evidence repository, Azure RBAC export

| Step | Action |
|---|---|
| 1 | Retrieve the monthly Azure Logic Apps RBAC export (CSV) from `ops-compliance-evidence/iso27001/access-reviews/` |
| 2 | Review all role assignments for: (a) former staff who should have been de-provisioned, (b) role creep (individuals with more permissions than their current role requires), (c) service accounts with roles that could be replaced by more restrictive Managed Identity scoped roles |
| 3 | Raise de-provisioning requests for any identified anomalies to the Security & Identity team via the standard IAM process |
| 4 | Certify the review: Ops Lead signs the access review form and deposits to `ops-compliance-evidence/iso27001/access-reviews/` with the review date |
| 5 | Communicate the review results to the ISO 27001 internal auditor if the quarterly review falls within an audit window |

---

### Escalation Contact Directory

The following escalation contacts are maintained separately as a classified internal document (not in this blueprint). The Ops team must maintain a current version of this contact directory:

| Role | Escalation trigger |
|---|---|
| AO Team Lead | Agent quality regression, CI failure, deployment decision |
| SWEE Team Lead | Triage accuracy degradation, Service Bus schema issues |
| AIGP Team Lead | OPA policy violation, AIGP API outage, Emergency Stop (AIGP-triggered) |
| Platform & Infra Lead | AKS/Kafka/ACA infrastructure failure, network connectivity issues |
| Ops Lead (after hours) | P1 incidents, Emergency Stop platform-level, resume approvals |
| Business Owner | HITL capacity crisis, SLA breach > 2 consecutive days, taxpayer complaint escalation |
| Internal SOC | Security incident, MDC threat alert, suspected data breach |
| GovTech SOC | Persistent network threat, incident requiring government-level response |
| DPO (Data Protection Officer) | Any suspected PDPA breach, new data processing activity requiring DPIA |

> The contact directory must be reviewed and updated at every quarterly RBAC review (RB-08) and whenever there is a personnel change in any of the listed roles.

---

## 7.3 Platform Health Review Cadence

The following regular reviews ensure the platform's operational health is continuously monitored and that cross-team issues are surfaced and addressed before they become incidents.

| Review | Frequency | Chair | Attendees | Input artefacts | Output |
|---|---|---|---|---|---|
| **Daily Ops Stand-up** | Daily (Mon–Fri, 09:30 SGT) | Ops Engineer (rotating) | Ops team | AO Ops Dashboard, HITL queue depth, overnight alerts | Daily status update to `#ops-daily` Teams channel |
| **Weekly Platform Health Review** | Weekly (Monday, 13:00 SGT) | Ops Lead | Ops, AO team lead, AIGP team lead | SLA Workbook MTD, Feedback Summary report, open post-mortems | Actions list; Feedback datasets distributed to owning teams |
| **Biweekly Ops Review** | Fortnightly (Wednesday, 14:00 SGT) | Ops Lead | All team leads + Business Owner | SLA report, FinOps report, HITL capacity, open incidents | Decisions on threshold adjustments, staffing changes, budget updates |
| **Monthly SLA and FinOps Review** | Monthly (first Tuesday) | Ops Lead | Management + Business Owner | Monthly SLA PDF, FinOps Cost Dashboard, ROI summary | Management sign-off on SLA targets; budget reconfirmation |
| **Quarterly Governance Review** | Quarterly | Ops Lead | All team leads + DPO + Internal Auditor | Compliance evidence pack, access review, OPA policy review, risk register update | Risk register updated; compliance evidence pack submitted |

---

> **Final note on this document**: This Step 7 completes the initial draft of the AIBP Operations Blueprint. All seven steps are authored as draft documents requiring review by the relevant team leads and the business owner before the blueprint is formalised. A formal review meeting should be scheduled once all team leads have had 5 business days to review their respective sections. The review output should update the document version from `0.1 (Draft)` to `1.0 (Approved)`.
