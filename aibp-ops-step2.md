# AIBP Operations Blueprint
## Step 2: Agentic Orchestration Operations (AO Layer)

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Managing the agent fleet — versioning, deployment, quality evaluation, observability, behavioral auditability, and anomaly detection — across the full lifecycle of agents running inside Azure Container Apps (ACA).

**AO Layer Context**: The Agentic Orchestration layer is the cognitive core of the platform. Each agent is a LangGraph-based application running in its own Azure Container App, consuming emails from a Service Bus subscription scoped to its SOP category. An agent's lifecycle spans: development → evaluation → canary deployment → full production → deprecation. Operations must instrument every stage and maintain continuous visibility into agent health, cost, and behavioral integrity in production.

**A note on one-agent-per-container architecture**: Deploying one agent per ACA container app (rather than a shared multi-agent runtime) has direct operational implications. It means each agent can be individually versioned, individually scaled, individually killed, and individually drained without affecting other agents. This architecture is strongly aligned with the operational requirements in this step and is assumed throughout.

---

## 2.1 Agent Fleet & Version Registry

### Implementation Overview

As the platform matures, agents will be updated frequently — prompt tuning, tool additions, bug fixes, behavioral corrections based on HITL feedback. Without a disciplined versioning and registry system, Ops cannot:

- Know which version of which agent is running in production at any given moment
- Roll back a specific agent to a previous version without redeploying the entire platform
- Audit which agent version produced a specific action on a taxpayer record
- Gate promotion based on evaluation scores from the CI/CD pipeline

The Agent Fleet & Version Registry is the operational ledger for agent identity. Every agent version that has ever been deployed — in any environment — has a permanent record in this registry.

---

### Option 1 (Recommended): Semantic Versioning + ACR Signed Images + PostgreSQL Registry DB

**Technology Stack**: Azure Container Registry (ACR), Notation (CNCF image signing standard), Azure Database for PostgreSQL Flexible Server, Azure DevOps, Azure Key Vault

---

#### Versioning Convention

Every agent follows Semantic Versioning (`MAJOR.MINOR.PATCH`):

| Increment | Trigger | Example |
|---|---|---|
| `MAJOR` | Breaking change in the agent's tool-call contract or SOP coverage scope (e.g., a new AIGP-registered tool added or removed; an SOP category boundary changed) | `1.0.0 → 2.0.0` |
| `MINOR` | New capability added without breaking existing contract (e.g., a new reasoning path added, new SOP sub-type handled) | `1.2.0 → 1.3.0` |
| `PATCH` | Prompt tuning, bug fix, latency optimisation, hallucination mitigation with no contract change | `1.2.0 → 1.2.1` |

A version bump must be explicitly declared in the agent repository's `agent-manifest.json` before the CI/CD pipeline will accept a build (see Section 2.2).

#### Container Image Signing (Notation / CNCF)

Every agent container image pushed to Azure Container Registry (ACR) must be signed using **Notation** (the CNCF standard for OCI image signing, successor to Docker Content Trust), backed by a signing key stored in **Azure Key Vault**.

**Signing workflow**:

1. CI pipeline builds the container image and pushes to ACR.
2. CI pipeline signs the image digest using the `notation sign` command with the agent team's signing certificate (stored in Azure Key Vault, accessed via Managed Identity).
3. The image signature is attached to the ACR repository as an OCI referrer artifact.
4. ACA's deployment step verifies the signature using `notation verify` before pulling the image. If verification fails, the deployment is blocked.

This ensures that **no unsigned or tampered image can be deployed to any environment**, providing a cryptographic guarantee that what was tested in CI is what runs in production.

#### Registry Database Schema

The registry is a PostgreSQL Flexible Server database (`aibp-agent-registry`) with the following core schema:

```sql
-- Agent version ledger (append-only; no updates or deletes permitted)
CREATE TABLE agent_versions (
    version_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_name        VARCHAR(100)  NOT NULL,   -- e.g., 'refund-agent'
    semver            VARCHAR(20)   NOT NULL,   -- e.g., '1.2.0'
    image_digest      VARCHAR(255)  NOT NULL,   -- SHA-256 of the container image in ACR
    image_ref         VARCHAR(500)  NOT NULL,   -- Full ACR reference including digest
    capability_manifest JSONB       NOT NULL,   -- Tools and permissions this agent version declares
    eval_score        DECIMAL(5,4),             -- Composite eval score from CI (NULL if not yet evaluated)
    eval_pass         BOOLEAN,                  -- Whether this version passed the eval gate
    deployment_status VARCHAR(20)   NOT NULL    -- 'candidate' | 'canary' | 'active' | 'deprecated' | 'retired'
                      CHECK (deployment_status IN ('candidate','canary','active','deprecated','retired')),
    aca_revision_name VARCHAR(200),             -- ACA revision label (e.g., 'refund-agent--abc123')
    deployed_env      VARCHAR(20),             -- 'dev' | 'test' | 'preprod' | 'prod'
    deployed_at       TIMESTAMPTZ,
    deployed_by       VARCHAR(200),            -- Azure DevOps pipeline run ID or operator UPN
    deprecated_at     TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (agent_name, semver)
);

-- Tool capability declarations per version
-- Must match the tools registered in AIGP
CREATE TABLE agent_tool_capabilities (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id        UUID NOT NULL REFERENCES agent_versions(version_id),
    tool_name         VARCHAR(100) NOT NULL,   -- Must match AIGP-registered tool name
    tool_semver       VARCHAR(20),             -- Version of the tool contract being used
    permission_scope  VARCHAR(20)  NOT NULL    -- 'read' | 'write' | 'execute'
                      CHECK (permission_scope IN ('read','write','execute'))
);

-- Environment promotion log
CREATE TABLE agent_promotions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id        UUID NOT NULL REFERENCES agent_versions(version_id),
    from_env          VARCHAR(20) NOT NULL,
    to_env            VARCHAR(20) NOT NULL,
    promoted_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    promoted_by       VARCHAR(200) NOT NULL,  -- Azure DevOps pipeline run ID or operator UPN
    approval_record   TEXT,                   -- Azure DevOps approval ID or manual approval note
    canary_traffic_pct SMALLINT               -- Traffic % if this promotion was a canary start
);
```

> **Append-only enforcement**: A PostgreSQL row-level security policy and a database role restriction ensure that no application account can execute `UPDATE` or `DELETE` on `agent_versions` or `agent_promotions`. The CI/CD service principal has `INSERT` + `SELECT` only. Ops read accounts have `SELECT` only. This provides a tamper-evident audit ledger aligned with ISO 27001 change management controls.

#### Agent Manifest File (`agent-manifest.json`)

Each agent repository must contain an `agent-manifest.json` at its root. This file is the authoritative source of truth for what the agent is and what it can do. The CI/CD pipeline reads this file to populate the registry DB.

```json
{
  "agentName": "refund-agent",
  "semver": "1.3.0",
  "sopScope": ["SOP-REFUND-*"],
  "tools": [
    { "name": "aigp.get_taxpayer_record",   "semver": "2.0.0", "scope": "read"    },
    { "name": "aigp.get_refund_history",    "semver": "1.1.0", "scope": "read"    },
    { "name": "aigp.submit_refund_request", "semver": "1.0.0", "scope": "write"   },
    { "name": "aigp.send_email_reply",      "semver": "3.2.0", "scope": "execute" }
  ],
  "evalThresholds": {
    "minCompositeScore": 0.85,
    "minToolCallAccuracy": 0.95,
    "maxHallucinationRate": 0.03
  },
  "changelogEntry": "Added handling for partial refund scenarios (SOP-REFUND-014). Prompt tuned to reduce over-escalation to HITL on ambiguous amounts."
}
```

**Pros**:
- Cryptographic image signing means exactly what was evaluated in CI is what runs in production — no configuration drift
- Append-only registry DB provides a permanent, auditable history of every agent version that has ever run in any environment
- `agent-manifest.json` in the repository creates a review artefact — capability changes are code-reviewed like any other change
- SemVer enables targeted rollback (PATCH rollback without losing MINOR capabilities)
- AIGP cross-validation: the `tools` list in the manifest can be automatically compared against AIGP's registered tool list during the CI gate, blocking deployment of agents that declare tools not yet provisioned in AIGP

**Cons**:
- PostgreSQL registry adds an infrastructure dependency (managed by Ops, not App & Data)
- Notation image signing requires PKI certificate management via Azure Key Vault; signing certificate rotation must be planned for
- If SWEE's dynamic SOP-routing requires frequent agent additions, the one-subscription-per-SOP model in Service Bus may require periodic topic subscription management

---

### Option 2: Git SHA Versioning + ACR Tag Only

**Implementation**: Agents are identified solely by their Git commit SHA, stored as the ACR image tag. Deployment metadata is tracked in a flat JSON file in the repository (`deployed.json`).

**Pros**: Zero additional infrastructure; version history is implicit in Git

**Cons**:
- Git SHA is opaque to operational tooling — an on-call engineer cannot quickly determine what changed between the two "versions" running in canary without spelunking in Git
- No eval score association; no tool-capability manifest; no deployment status tracking
- Cannot satisfy ISO 27001 change management audit requirements without supplementary documentation

---

### Option 3: Monolithic Release (all agents versioned together)

**Implementation**: All agents share a single version number. Any change to any agent triggers a platform-wide release.

**Pros**: Simpler release coordination between teams

**Cons**:
- A prompt fix to one agent forces redeployment of all agents, multiplying blast radius
- A regression in one agent causes all agents to roll back, interrupting processing for all SOP categories
- Not viable for independent team ownership of individual agents

---

### Recommendation Justification

**Option 1** is recommended. The append-only registry DB and signed image artefacts together satisfy the ISO 27001 change management control (CI-04: Software Configuration Management) and provide the operational forensics capability needed in incident response — within minutes of an agent producing an unexpected output, Ops can query the registry to identify the exact image digest, `agent-manifest.json` hash, eval score, and promotion approver for the version that caused the incident.

---

## 2.2 Deployment Lifecycle — Dev → Test → Pre-Prod → Prod

### Implementation Overview

An agent's path from a developer's workstation to production must be scripted, gated, and auditable. This section defines the environments, the gates between them, and the automated checks that must pass before any promotion.

---

### Option 1 (Recommended): Azure DevOps Multi-Stage Pipeline with Manual Approval Gates

**Technology Stack**: Azure DevOps Pipelines, Azure Container Apps (per-environment Container App Environments), Azure Container Registry, Azure AI Evaluation SDK, Langfuse (self-hosted), Azure Key Vault

---

#### Environment Design

Four environments are provisioned as separate **ACA Environments** (ACA's logical isolation boundary), each in the same GCC 2.0 subscription but with separate VNet integration, separate Service Bus namespaces, and separate Azure OpenAI quota pools:

| Environment | Purpose | Azure OpenAI quota | Who deploys | Manual gate required |
|---|---|---|---|---|
| `dev` | Local iteration; developer-controlled | Shared, limited (PTU-free) | Developer (feature branch push) | No |
| `test` | Automated CI evaluation against synthetic emails | Dedicated test quota | CI pipeline (PR merge to `main`) | No |
| `preprod` | Full integration test; replica of production topology | Shared preprod quota | CD pipeline (manual trigger) | Yes — AO team lead |
| `prod` | Live taxpayer email processing | PTU (see Step 4.1) | CD pipeline (manual trigger) | Yes — AO team lead + Ops |

> **SOC Note**: The `dev` environment is explicitly excluded from SOC log streaming. Only `preprod` and `prod` emit logs to the SOC pipeline. The `test` environment emits logs to Azure Monitor only.

---

#### Pipeline Stages

**Stage 1 — Build & Sign** (triggered on PR merge to `main`)

1. Read `agent-manifest.json`; validate schema and SemVer against the registry DB (reject if version already exists)
2. Cross-validate `tools` declarations against AIGP's registered tool list via AIGP API (fail if tool not found or scope mismatch)
3. Build Docker image; push to ACR with `{agentName}:{semver}` tag
4. Sign image using Notation; verify signature before proceeding
5. Register the new version in the registry DB with `deployment_status = 'candidate'`

**Stage 2 — Test Environment Deployment & Evaluation** (automatic, post-Stage 1)

1. Deploy candidate image to `test` ACA environment
2. Run **Synthetic Email Evaluation Suite** (100 pre-labelled synthetic emails per SOP category, stored in Langfuse as a versioned dataset):
   - **Structural assertions** (pytest): Did the agent call the correct tools? In the correct order? Did it avoid calling unauthorised tools?
   - **Semantic reasoning evaluation** (Azure AI Evaluation SDK + GPT-4o as judge): Faithfulness score, answer relevance score, task completion score, hallucination rate
   - **Regression check**: Compare composite eval score against the current `active` version's score in the registry DB. Block promotion if the new version's composite score is more than 5% lower than the current active version.
3. Write eval scores back to `agent_versions` row in registry DB (`eval_score`, `eval_pass`)
4. If any assertion fails or eval scores are below the thresholds declared in `agent-manifest.json → evalThresholds`: pipeline fails; AO team is notified; version remains `candidate`

**Stage 3 — Pre-Prod Deployment** (manual trigger by AO team lead)

1. **Manual approval gate** in Azure DevOps: AO team lead reviews evaluation report before approving
2. Deploy to `preprod` ACA environment with **100% traffic weight** (blue-green: new revision receives all traffic)
3. Run **integration tests**: real inbound email flow with redacted real-pattern emails (no live taxpayer data; uses email patterns derived from historical cases with PII removed)
4. Run **load test** (Azure Load Testing): simulate 2× expected daily volume over 1 hour; verify P95 latency and token spend remain within thresholds
5. Ops team reviews preprod Azure Monitor Workbook for 24 hours post-deployment

**Stage 4 — Production Canary Deployment** (manual trigger by AO team lead + Ops sign-off)

1. **Double manual approval gate**: AO team lead + Ops team lead must both approve (Azure DevOps two-approver policy)
2. Deploy new revision to `prod` ACA environment; assign **10% traffic weight** (canary)
3. Agent begins receiving 10% of live taxpayer emails
4. Monitor canary metrics for **24–48 hours** (see Step 2.3 for canary promotion criteria)
5. On criteria pass: promote to 100% traffic weight; retire old revision
6. On criteria fail: revert to 0% canary traffic; incident review; return to Stage 2

**Promotion records** are written to `agent_promotions` in the registry DB at every stage transition, with the Azure DevOps pipeline run ID and approver UPN.

**Pros**:
- All gates and approvals are recorded in Azure DevOps and the registry DB — satisfies ISO 27001 change audit trail requirements
- Synthetic email dataset in Langfuse provides a version-controlled, reproducible evaluation baseline
- Double approval for production prevents unilateral deployment
- Environment isolation prevents test/preprod load from consuming production Azure OpenAI quota

**Cons**:
- Multi-stage pipeline adds elapsed time to the deployment lifecycle (a patch can take 24–72 hours from merge to full production, depending on approval turnaround)
- Synthetic evaluation dataset requires ongoing curation — as officer audit data accumulates (Step 1.2), it should be used to refresh and extend the synthetic dataset
- Pre-Prod integration tests require a continuously maintained set of redacted real-pattern emails

---

### Option 2: GitHub Actions + Environment Protection Rules

**Technology Stack**: GitHub Actions, GitHub Environments with protection rules

**Implementation**: Essentially identical pipeline logic to Option 1 but hosted in GitHub Actions. Environment protection rules provide mandatory review gates equivalent to Azure DevOps approval gates.

**Pros**: Better developer experience; tight pull request integration; native integration with GitHub Container Registry

**Cons**:
- GitHub Actions runners must be self-hosted within GCC 2.0 (Microsoft-hosted runners are not GCC 2.0-compliant for processing government workloads). Standing up and maintaining self-hosted runner infrastructure is an added operational overhead.
- The organisation likely has existing Azure DevOps infrastructure for other government projects; GitHub Actions introduces a second CI/CD platform to govern and secure

---

### Option 3: Manual Deployment with Runbook

**Implementation**: AO team builds the container image locally, pushes to ACR manually, and follows a written runbook to deploy each environment.

**Pros**: No CI/CD infrastructure to maintain

**Cons**:
- No automated eval gate — agent quality is entirely dependent on developer diligence
- No audit trail beyond Git commit history — fails ISO 27001 change management
- Error-prone; manual container signing is often skipped under time pressure

---

### Recommendation Justification

**Option 1** is recommended. Government ICT deployments require a fully auditable change trail; Azure DevOps provides this natively within the existing GCC 2.0 governance boundary. The multi-stage pipeline encodes the operational quality contract (evaluation thresholds in `agent-manifest.json`) as an automated enforcement mechanism rather than a policy document that humans must remember to follow.

> **Compliance Note (ISO 27001)**: The double-approval gate for production deployments satisfies ISO 27001 control A.12.1.2 (Change Management) — specifically the requirement for formal approval prior to changes to production systems. The Azure DevOps approval audit log, combined with the `agent_promotions` registry DB table, provides the required documentary evidence.

> **Compliance Note (IM8)**: Any changes to agent logic that affect how taxpayer data is processed must be accompanied by a Data Protection Impact Assessment (DPIA) update if the processing purpose or data flows change. The `agent-manifest.json` tool-capability change in a `MAJOR` version bump is a trigger for DPIA review.

---

## 2.3 Testing Strategy: Blue-Green + Canary

### Implementation Overview

LLMs are non-deterministic. An agent that passes all synthetic evaluation tests can still exhibit unexpected behaviour on the diverse, unpredictable distribution of real taxpayer emails. The testing strategy for production must expose new agent versions to real traffic at low volume before full promotion — while retaining the ability to instantly revert if the canary metrics degrade.

Azure Container Apps natively supports **revision-based traffic splitting**, making blue-green and canary deployments a first-class operational capability at no additional infrastructure cost.

---

### Option 1 (Recommended): Blue-Green Foundation with Canary Promotion

**Technology Stack**: Azure Container Apps (revision traffic weights), Azure Monitor (canary health metrics), Azure DevOps (automated promotion gate)

---

#### Blue-Green Foundation

"Blue" is the current active revision. "Green" is the new candidate revision. Both revisions run simultaneously in the same ACA Container App, receiving traffic according to their assigned weights.

- **At canary start**: Blue = 90%, Green = 10%
- **At canary midpoint** (if 24h metrics pass): Blue = 50%, Green = 50%
- **At full promotion**: Blue = 0%, Green = 100%
- **At rollback**: immediately set Blue = 100%, Green = 0% — takes effect within seconds

ACA applies traffic splitting at the HTTP level (for synchronous trigger scenarios) and at the Service Bus consumer level — the 10% canary weight means the Green revision's Service Bus consumer group processes approximately 10% of incoming messages. This is achieved by setting `maxConcurrentCalls` on the Service Bus trigger binding for each revision proportionally.

> **Important**: ACA traffic splitting for Service Bus consumers is controlled via scale rules and `maxConcurrentCalls` on the Service Bus trigger, not via HTTP routing weights. The Ops team must verify that the AO team's ACA deployment configuration correctly implements proportional consumption, not just HTTP traffic splitting. This should be validated in the pre-prod load test (Step 2.2, Stage 3).

---

#### Canary Promotion Criteria

These criteria are evaluated automatically by an **Azure DevOps canary gate task** (using the Azure Monitor query task plugin) that polls metrics every 4 hours during the canary window:

| Metric | Canary pass threshold | Source |
|---|---|---|
| Agent error rate (unhandled exceptions) | < 2% of canary-processed emails | Azure Monitor / App Insights |
| P95 end-to-end latency (Service Bus receipt → resolution) | Within 20% of blue revision baseline | Azure Monitor |
| Token spend per email | Within 20% of blue revision baseline | Azure Monitor custom metric |
| HITL escalation rate | Within 30% of blue revision baseline (new version should not escalate significantly more) | Azure Monitor custom metric |
| Eval score on canary traces (LLM-as-a-Judge, 4-hour batch) | ≥ declared `minCompositeScore` from `agent-manifest.json` | Langfuse → Azure Monitor metric export |
| AO rejection / re-route rate | < 5% of canary-processed emails | Azure Monitor |

**Canary window**: Minimum 24 hours; maximum 72 hours. If criteria are not met within 72 hours, the deployment is automatically failed and the pipeline posts an alert.

**Rollback trigger**: If, at any point during the canary window, **any two metrics** breach their thresholds simultaneously, an **automatic rollback** is triggered by the Azure DevOps pipeline (no manual intervention required). The rollback sets the Blue revision to 100% traffic weight and logs a rollback event to the registry DB.

---

#### Handling Non-Determinism in Canary Evaluation

LLMs produce different outputs for the same input on different runs. This means that some canary metric fluctuation is expected and is not a signal of regression. Two design choices mitigate false rollbacks:

1. **Evaluation is batched**: LLM-as-a-Judge evaluation runs in 4-hour batches on accumulated canary traces, not per-message. This smooths out per-message variance.
2. **Two-metric simultaneous threshold**: A single metric breaching its threshold does not trigger rollback; two simultaneous breaches are required. This prevents a single statistical outlier from causing an unnecessary rollback event.

**Pros**:
- ACA native revision weighting means no additional load balancer or proxy infrastructure
- Automatic rollback removes the risk of a canary failure going unnoticed during non-business hours
- Canary exposure limits the number of taxpayer emails affected by a regression before rollback (at 10% weight and <1,000 emails/day, a regression is limited to ~100 emails/day maximum before the 4-hour metric check)

**Cons**:
- Service Bus consumer-proportional traffic splitting requires careful ACA trigger configuration; must be validated in pre-prod
- Canary window (24–72 hours) means a hotfix patch takes at minimum 24 hours to reach 100% production, even in an emergency. The Emergency Stop mechanism (Step 3.2) should be used for critical safety issues that cannot wait for canary completion.
- Two-metric trigger for rollback means a single severe regression in one metric does not auto-rollback; Ops team must monitor the canary dashboard

---

### Option 2: Feature-Flag-Based Canary (Azure App Configuration)

**Technology Stack**: Azure App Configuration (feature flags), custom routing logic in SWEE/AO

**Implementation**: Azure App Configuration feature flags control which emails are routed to the new agent version. A feature flag `agent-refund-v2` at 10% targeting routes that percentage of emails to the v2 agent container.

**Pros**: More fine-grained targeting available (e.g., route canary only to a specific taxpayer segment or SOP subcategory)

**Cons**:
- Requires custom routing logic in SWEE or AO to read the feature flag and fork the email to the appropriate agent version — additional code to test and maintain
- ACA revision weighting is simpler and requires no application code changes; feature flags are more appropriate for UI feature gating than infrastructure-level agent traffic routing

---

### Option 3: Blue-Green Full Cutover (No Canary)

**Implementation**: All or nothing. New revision deployed directly to 100% traffic weight; old revision immediately retired.

**Pros**: Simpler mental model; no canary monitoring required

**Cons**:
- If the new version has a regression that was not caught in evaluation (possible with LLMs on real-world email distribution), 100% of taxpayer emails are affected before the problem is detected
- Rollback is possible but requires a new deployment cycle: updating traffic weight to the old revision, which still requires a few minutes
- Not acceptable for a production LLM system with variable, unpredictable outputs

---

### Recommendation Justification

**Option 1** is recommended. The nature of LLM-based agents makes blue-green full cutover (Option 3) unacceptably risky — synthetic evaluation cannot fully represent real-world email diversity. The canary approach provides a controlled exposure window at minimal infrastructure overhead, given ACA's native revision traffic splitting. Option 2's feature flag approach introduces application-layer complexity that is unnecessary when the infrastructure already provides the capability.

---

## 2.4 EvalOps — Continuous Evaluation Pipeline

### Implementation Overview

EvalOps is the systematic, ongoing measurement of agent reasoning quality. Unlike traditional software, where functional tests provide a binary pass/fail, LLM agents must be evaluated on dimensions that are inherently probabilistic: Does the reasoning chain make sense? Is the tool-call sequence appropriate for the SOP? Is the response faithful to the taxpayer's query? Is the agent hallucinating facts about the taxpayer record?

EvalOps runs at two points:
1. **Pre-deployment** (CI gate in Stage 2 of the deployment pipeline): evaluates a candidate version against the synthetic email dataset before any environment promotion
2. **In-production (continuous)**: evaluates completed production traces in batches to detect post-deployment quality drift

---

### Option 1 (Recommended): Three-Layer Evaluation — Structural Assertions + LLM-as-a-Judge + HITL Feedback Integration

**Technology Stack**: Azure AI Evaluation SDK, pytest + pytest-asyncio, Azure OpenAI (GPT-4o as judge), Langfuse (self-hosted, dataset and scoring management), Azure Container Apps Job (evaluation batch runner)

---

#### Layer 1: Structural Assertions (Deterministic)

These checks verify the procedural correctness of agent behaviour — whether the agent called the right tools, in the right order, without calling forbidden tools. They run synchronously in the CI pipeline and provide instant binary pass/fail feedback.

**Implementation**: pytest test suite (`tests/eval/`) committed to the agent repository. Each test case is a golden email input + expected tool-call sequence. Test cases cover:

- **Happy path**: Standard email of each SOP type → verify the canonical tool-call sequence
- **Boundary cases**: Edge-case email phrasings → verify the agent reaches the correct resolution path
- **Negative cases**: Malformed email, missing taxpayer reference → verify the agent gracefully escalates to HITL rather than hallucinating a taxpayer identity
- **Tool-call prohibition**: Emails that should never trigger a `write` tool call (read-only queries) → verify no unauthorised write tool was called

**Tool-call trace extraction**: LangGraph's OTel instrumentation (Step 2.5) emits a structured tool-call sequence as part of each trace. The pytest tests query Langfuse's API to retrieve the trace and assert against the tool-call sequence.

**Threshold**: 100% of structural assertion tests must pass for the pipeline to proceed to Layer 2.

---

#### Layer 2: LLM-as-a-Judge — Semantic Quality Evaluation

These checks evaluate the semantic quality of agent reasoning — dimensions that cannot be captured by structural assertions alone.

**Implementation**: Azure AI Evaluation SDK (`azure-ai-evaluation`) is used to compute the following metrics over the synthetic email evaluation dataset (stored in Langfuse):

| Metric | Definition | Tools | Threshold |
|---|---|---|---|
| **Task Completion** | Did the agent successfully resolve the taxpayer's query (as intended by the SOP)? | GPT-4o judge | ≥ 0.85 |
| **Faithfulness** | Are the agent's statements grounded in the information retrieved from taxpayer records (no hallucination of facts)? | GPT-4o judge | ≥ 0.90 |
| **Tool-Call Relevance** | Were the tools called relevant to the task, or did the agent call unnecessary tools (wasted API calls / cost)? | Rule-based + GPT-4o | ≥ 0.88 |
| **Response Appropriateness** | Is the final email response appropriate for a government tax authority communication (tone, accuracy, no unsolicited advice)? | GPT-4o judge | ≥ 0.85 |
| **Hallucination Rate** | % of agent outputs that contain unsupported factual claims about the taxpayer | GPT-4o judge (binary per output) | ≤ 0.03 |

**Composite score**: `0.25 × TaskCompletion + 0.30 × Faithfulness + 0.20 × ToolCallRelevance + 0.15 × ResponseAppropriateness + 0.10 × (1 - HallucinationRate)`. The composite score must meet the `minCompositeScore` declared in the agent's `agent-manifest.json`.

**Eval dataset versioning**: The synthetic email evaluation dataset is stored in Langfuse as a named, versioned dataset (e.g., `refund-agent-eval-v3`). Updates to the dataset (e.g., adding new edge cases from officer audit findings) increment the dataset version. The CI pipeline always evaluates against the latest dataset version; regression comparisons use scores from the same dataset version to ensure apples-to-apples comparison.

---

#### Layer 3: HITL Feedback Integration (In-Production Continuous Evaluation)

As agents process real emails in production, tax officers occasionally correct agent actions via the HITL workflow (Step 3.1). Each HITL correction is a ground-truth signal that the agent made a suboptimal or incorrect decision. This signal is fed back into the evaluation pipeline.

**Implementation**:

1. When a tax officer submits a HITL correction (Step 3.1), the correction event includes:
   - `messageId` (correlation ID)
   - `correctionType` (`escalation_override` | `action_correction` | `response_revision`)
   - `officerNotes` (optional free-text)

2. An **Azure Event Grid** event is emitted on HITL correction submission.

3. An **Azure Container Apps Job** (`eval-hitl-feedback-processor`) subscribes to this event and:
   - Retrieves the full agent trace from Langfuse using the `messageId`
   - Annotates the trace in Langfuse with a `score` of 0 (failed), a `label` of the `correctionType`, and the officer's notes
   - If the correction is a `action_correction` (agent took the wrong action): the trace is added to the active evaluation dataset in Langfuse as a new negative example (with a 30-day review period before it is incorporated into the CI eval baseline, to allow for officer error rate correction)

4. Langfuse's **production eval metrics** dashboard tracks:
   - HITL correction rate by agent and by SOP category
   - Trend in correction type distribution (escalation vs. action correction vs. response revision)
   - Production composite score (computed on a 7-day rolling window of HITL-annotated traces)

5. If the production composite score on HITL-annotated data drops below 15% of the CI eval score baseline, an **alert is sent to the AO team** to investigate and prepare a patch release.

**Pros (Option 1 overall)**:
- Three-layer evaluation catches three distinct failure modes: procedural errors (Layer 1), reasoning quality regressions (Layer 2), and production-specific edge cases invisible to synthetic data (Layer 3)
- Azure AI Evaluation SDK is Azure-native, GCC 2.0-compatible, and maintained by Microsoft; no external dependency on LangSmith, RAGAS open-source, or other tools
- HITL feedback integration creates a self-improving evaluation dataset: the longer the platform runs, the more ground-truth examples are available, and the more precise the evaluation becomes
- Langfuse self-hosted provides dataset versioning, score aggregation, and trace annotation in a single tool

**Cons**:
- GPT-4o-as-judge adds evaluation token cost; at 100 synthetic cases per eval run and ~500 tokens per evaluation call, this is approximately 50,000 tokens per CI eval run (~SGD 0.10 per run — negligible)
- LLM judges have known biases (preference for longer responses, position bias in ranked comparisons). These biases are mitigated by the faithfulness and hallucination rate metrics, which use constrained binary evaluation prompts rather than open-ended ranking

---

### Option 2: Azure Prompt Flow Evaluation

**Technology Stack**: Azure AI Studio / Prompt Flow (evaluation flows)

**Implementation**: Azure Prompt Flow provides UI-driven evaluation flow definitions with built-in metrics (groundedness, relevance, coherence). Evaluation flows are run in Azure AI Studio against uploaded test datasets.

**Pros**: Lower code overhead to set up; built-in metric definitions; UI for non-technical reviewers to inspect results

**Cons**:
- Azure Prompt Flow evaluation is tightly coupled to Azure AI Studio, which has varying GCC 2.0 availability status — confirm compatibility before adoption
- Does not integrate natively with the ACA-based deployment pipeline; requires a separate step to trigger evaluation and retrieve scores
- Limited extensibility for custom structural assertion logic (Layer 1 equivalent); Python tests are more flexible

---

### Recommendation Justification

**Option 1** is recommended. The three-layer architecture is justified because each layer catches a category of failure that the others cannot. No single evaluation approach can cover all failure modes of an LLM agent: Layer 1 catches procedural errors instantly (before spending GPT-4o tokens on evaluation), Layer 2 catches reasoning quality regressions against a controlled benchmark, and Layer 3 grounds evaluation in ground truth from real production behaviour.

> **Compliance Note (IM8)**: Evaluation results (scores, pass/fail, eval dataset versions) must be retained as part of the change record for each agent version deployment. Langfuse trace and score data constitute part of this record.

---

## 2.5 Observability & Distributed Tracing

### Implementation Overview

An agent that cannot be observed cannot be operated. Observability in the AO layer serves three distinct audiences with different needs:

- **Ops team**: Real-time health signals — is the agent healthy, within latency and cost budgets, processing at expected throughput?
- **AO team**: Debugging — what exactly did the agent do for a specific email that produced an unexpected outcome?
- **Auditors / AIGP**: Compliance — was every tool call authorised, within declared capabilities, and recorded?

The observability stack must therefore capture both **aggregate metrics** (for operations) and **per-trace detail** (for investigation and audit).

---

### Option 1 (Recommended): OpenTelemetry with OpenInference Conventions → Azure Monitor + Langfuse

**Technology Stack**: OpenTelemetry (OTel) SDK for Python, OpenInference semantic conventions (CNCF), Azure Monitor OpenTelemetry Distro, Langfuse (self-hosted, OTel trace sink), Azure Monitor Application Insights, Azure Monitor Workbooks

---

#### OpenInference Semantic Conventions

[OpenInference](https://github.com/Arize-ai/openinference) is the CNCF working group standard for LLM observability trace semantics. It defines a set of span attribute names that are consistent across different LLM frameworks — `llm.input_messages`, `llm.output_messages`, `llm.token_count.prompt`, `llm.token_count.completion`, `tool.name`, `tool.parameters`, `tool.output`, etc.

By using OpenInference conventions, the OTel traces emitted by the LangGraph agent are interpretable by any OpenInference-compatible backend (Langfuse, Arize Phoenix self-hosted, future tooling) without requiring schema changes.

#### Instrumentation Architecture

LangGraph is instrumented using the **`openinference-instrumentation-langchain`** library (which covers LangChain-based frameworks including LangGraph), which automatically instruments:

- Each LangGraph graph node as an OTel span
- Each LLM call (`ChatOpenAI.invoke`) as a child span with token counts and model name
- Each tool call as a child span with tool name, input parameters, output, and latency

The OTel SDK is configured with two exporters running in parallel:

1. **Azure Monitor OTel Distro exporter**: Exports spans and metrics to Azure Application Insights. This is used for aggregate operational metrics, SLA dashboards, and alert rules.

2. **Langfuse OTel exporter** (`langfuse.otel.LangfuseSpanExporter`): Exports the full trace (all spans) to Langfuse for detailed trace inspection, eval scoring, and HITL annotation.

The two-exporter pattern means: aggregate signals go to Azure Monitor (operational view), detailed trace data goes to Langfuse (debugging and evaluation view). This separation avoids bloating Application Insights with high-cardinality trace data while still retaining full trace fidelity in Langfuse.

#### Span Hierarchy for a Single Email Processing Event

```
[Root Span] email.processing
│  Attributes: messageId, correlationId, sopId, agentName, agentVersion
│  Duration: end-to-end wall clock for full resolution
│
├── [Span] ao.agent.{agentName}
│   Attributes: agentName, agentSemver, sopId
│   │
│   ├── [Span] llm.invoke  (first reasoning step — planning)
│   │   Attributes: llm.model_name, llm.token_count.prompt, llm.token_count.completion,
│   │               llm.latency_ms, llm.input_messages[], llm.output_messages[]
│   │
│   ├── [Span] tool.call: aigp.get_taxpayer_record
│   │   Attributes: tool.name, tool.parameters (taxpayer_ref_hash only — no PII),
│   │               tool.output_summary (status code + record_found: bool),
│   │               tool.latency_ms
│   │
│   ├── [Span] llm.invoke  (second reasoning step — decision)
│   │   ...
│   │
│   ├── [Span] tool.call: aigp.submit_refund_request
│   │   Attributes: tool.name, tool.parameters (hash refs only),
│   │               tool.output_summary (request_id, status),
│   │               tool.latency_ms, aigp.risk_score (from AIGP response)
│   │
│   └── [Span] llm.invoke  (final response generation)
│       ...
│
└── [Span] ao.resolution
    Attributes: resolution_type (automated|hitl_escalated|failed),
                total_llm_tokens, total_tool_calls, total_latency_ms,
                hitl_escalated (bool), failure_reason (if applicable)
```

> **PII in traces**: Span attributes must never contain raw taxpayer data. Tool call parameters that would normally include taxpayer identifiers must use the hash reference (`taxpayer_ref_hash`) from the Service Bus message. Tool outputs must be summarised (status codes, boolean flags, aggregate counts) — never the raw record content. This is enforced by code review gate in the CI pipeline and audited by the AIGP team periodically.

#### Custom Metrics (Azure Monitor)

In addition to spans, the OTel SDK emits the following custom metrics to Azure Monitor on each email resolution:

| Metric name | Value | Dimensions |
|---|---|---|
| `ao.email.tokens.prompt` | Total prompt tokens consumed | `agentName`, `agentSemver`, `sopId`, `env` |
| `ao.email.tokens.completion` | Total completion tokens consumed | `agentName`, `agentSemver`, `sopId`, `env` |
| `ao.email.latency_ms` | End-to-end resolution latency | `agentName`, `agentSemver`, `sopId`, `resolutionType` |
| `ao.email.tool_calls.count` | Number of tool calls made | `agentName`, `toolName` |
| `ao.email.hitl_escalated` | 1 if escalated to HITL, 0 if not | `agentName`, `escalationReason` |
| `ao.email.resolution_type` | Categorical: `automated` / `hitl` / `failed` | `agentName`, `sopId` |

#### Azure Monitor Workbook — Agent Operations Dashboard

An Azure Monitor Workbook (`AO-Operations-Dashboard`) provides the operational health view for the AO layer. Key panels:

- **Agent Fleet Status**: Table showing all active agents, their current version, traffic weight (blue/canary), error rate, and P95 latency — refreshed every 5 minutes
- **Token Spend Trend**: Daily token spend per agent vs. budget threshold (linked to FinOps thresholds in Step 5)
- **Resolution Mix**: % automated vs. HITL escalated vs. failed, per agent, rolling 7 days
- **Canary Comparison View**: Side-by-side metric comparison between blue and canary revision during active canary windows

**Pros**:
- OpenInference conventions ensure the trace format is portable — if Langfuse is replaced in future, traces remain interpretable by any compatible backend
- Two-exporter pattern provides both aggregate (Azure Monitor) and per-trace (Langfuse) views without duplication
- LangGraph auto-instrumentation means the AO team does not need to write manual OTel span code for standard graph nodes — only custom tool calls require explicit span attributes
- Fully GCC 2.0-compliant: Azure Monitor is Azure-native; Langfuse is self-hosted within the boundary

**Cons**:
- Two exporters mean trace data exists in two stores (Azure Monitor and Langfuse); query routing (which tool for which question) must be documented for Ops
- Langfuse's OTel exporter is relatively new; verify stability and version pinning in production
- High trace volume (though at <1,000 emails/day, this is modest) requires Langfuse PostgreSQL storage capacity planning over time

---

### Option 2: Azure Monitor Application Insights Only (No Langfuse)

**Implementation**: OTel traces exported to Application Insights only. Detailed trace data is queried via Kusto (KQL) in Log Analytics.

**Pros**: Single observability store; no Langfuse infrastructure to maintain

**Cons**:
- Application Insights is not designed for LLM trace semantics — there is no first-class concept of "LLM reasoning step" or "tool call sequence." All of this must be stored as custom dimensions on Application Insights events, making investigation queries complex
- No native eval scoring or dataset management — the EvalOps (Step 2.4) integration requires Langfuse or an equivalent store
- Querying a multi-step agent reasoning trace in KQL is significantly more cumbersome than Langfuse's purpose-built trace UI

---

### Recommendation Justification

**Option 1** is recommended. The combination of Azure Monitor (operational metrics) and Langfuse (trace and evaluation data) cleanly separates concerns: Azure Monitor for real-time operations and alerting, Langfuse for investigation, evaluation, and feedback. Attempting to consolidate everything into Application Insights (Option 2) results in a store that serves neither function well.

> **Compliance Note (PDPA / IM8)**: All trace data entering Langfuse must have had PII removed (see PII control note above). Langfuse's PostgreSQL store is subject to the same IM8 5-year retention and backup requirements as the evaluation dataset. A data classification label must be applied to the Langfuse PostgreSQL database resource reflecting the sensitivity of the operational trace data it holds (even without PII, traces may reveal internal SOP logic and process flows that warrant RESTRICTED classification).

---

## 2.6 Behavioral Auditing & Traceability

### Implementation Overview

Observability (Step 2.5) answers "what is happening right now." Auditing answers "what happened, exactly, and why, and who authorised it." The audit trail is the evidentiary record for compliance reviews, incident post-mortems, and taxpayer dispute resolution.

The audit trail must answer, for any historical email:
- Which agent version processed it, and when?
- What reasoning steps did the agent take?
- What actions did the agent take on taxpayer records (read/write)?
- Was a human officer involved, and what did they decide?
- Was the agent's action within its declared capability manifest?

This record must be **tamper-evident** — no one (including Ops or the AO team) should be able to delete or modify an audit trace post-hoc.

---

### Option 1 (Recommended): Append-Only Azure Cosmos DB Audit Ledger + Event Hubs for SOC Streaming

**Technology Stack**: Azure Cosmos DB for NoSQL (append-only enforcement), Azure Event Hubs, Azure Key Vault (encryption), Azure Blob Storage (long-term archive)

---

#### Cosmos DB Audit Ledger

Each email processing event produces one audit document written to a Cosmos DB container (`agent-audit-log`) at resolution time. The document captures the complete reasoning and action trace in a purpose-built structure:

```json
{
  "id": "uuid-v4",
  "messageId": "uuid (Service Bus MessageId — root correlation ID)",
  "agentName": "refund-agent",
  "agentSemver": "1.3.0",
  "imageDigest": "sha256:abc123...",
  "sopId": "SOP-REFUND-001",
  "sessionId": "sha256-hash (taxpayer thread, no PII)",
  "processedAt": "2026-05-03T09:14:22Z",
  "resolutionType": "automated",
  "steps": [
    {
      "stepId": 1,
      "type": "llm_reasoning",
      "modelName": "gpt-4o",
      "inputSummary": "Taxpayer query classified as refund status enquiry. SOP: REFUND-001.",
      "outputSummary": "Plan: retrieve record, check status, respond if pending, escalate if error.",
      "tokenCount": { "prompt": 512, "completion": 148 },
      "latencyMs": 1340
    },
    {
      "stepId": 2,
      "type": "tool_call",
      "toolName": "aigp.get_taxpayer_record",
      "toolSemver": "2.0.0",
      "inputRef": { "taxpayer_ref_hash": "sha256:def456..." },
      "outputSummary": "Record found. Refund status: PENDING_APPROVAL. Amount: [REDACTED].",
      "aigpPolicyOutcome": "PERMITTED",
      "aigpRiskScore": 0.12,
      "latencyMs": 287
    },
    {
      "stepId": 3,
      "type": "tool_call",
      "toolName": "aigp.send_email_reply",
      "toolSemver": "3.2.0",
      "inputRef": { "responseTemplateId": "REFUND-STATUS-PENDING-001" },
      "outputSummary": "Email reply queued for dispatch. MessageId: xyz789.",
      "aigpPolicyOutcome": "PERMITTED",
      "aigpRiskScore": 0.04,
      "latencyMs": 110
    }
  ],
  "hitlInvolved": false,
  "evalScoreAtRuntime": null,
  "schemaVersion": "1.0"
}
```

> **Data minimisation in audit documents**: Audit documents do not contain raw email body text, taxpayer names, NRIC numbers, or dollar amounts. All taxpayer references use hash identifiers. Tool call outputs are summarised (status labels, boolean flags) rather than recorded verbatim. The `inputSummary` and `outputSummary` in LLM reasoning steps are narrative summaries abstracted from the chain-of-thought, not verbatim LLM outputs. Verbatim LLM outputs are retained in Langfuse traces (operational tool) but not in the compliance audit ledger, since they may contain inferred taxpayer details.

**Cosmos DB Configuration for Tamper-Evidency**:

| Setting | Value | Rationale |
|---|---|---|
| Container partition key | `/agentName` | Balanced distribution; efficient queries by agent during incident investigations |
| Analytical store | Enabled | Enables Azure Synapse Link for long-term audit analysis without impacting operational RU/s |
| RBAC — write | Agent Managed Identity: `Cosmos DB Built-in Data Contributor` (limited to INSERT, no DELETE/REPLACE) | Enforced via custom Cosmos DB role with restricted action set |
| RBAC — read | Ops team Managed Identity: `Cosmos DB Built-in Data Reader` | Read-only for investigation; no write access |
| Resource lock | `CanNotDelete` at container level | Prevents accidental container deletion |
| Customer-managed encryption key | Azure Key Vault CMK | IM8 encryption at rest requirement |
| TTL | Disabled | Records must be retained indefinitely (archive policy applies) |
| Archive to cool storage | Azure Synapse Link → Azure Blob Storage (cool tier) after 2 years | IM8 5-year minimum retention satisfied by combination of hot Cosmos DB (0–2 years) + cool archive (2–7+ years) |

> **Note on "tamper-evident" vs "immutable"**: While Cosmos DB does not provide blockchain-level immutability, the combination of `CanNotDelete` resource lock, RBAC with no UPDATE/DELETE permissions for any service identity, and the append-only insert pattern creates a strong operational control. For the highest-assurance environments, consider enabling [Azure Cosmos DB Ledger](https://learn.microsoft.com/en-us/azure/cosmos-db/ledger/ledger-overview) (currently in preview for NoSQL API), which provides cryptographic verification of row integrity.

---

#### Event Hubs for SOC Streaming

A parallel, operationally informative (not audit-grade) event stream is published to **Azure Event Hubs** for consumption by the Internal SOC and GovTech SOC:

**Events published** (anonymised application telemetry — no taxpayer content):

| Event type | Fields included | Fields explicitly excluded |
|---|---|---|
| `agent.processing.started` | `messageId`, `agentName`, `agentSemver`, `sopId`, `timestamp` | Email content, taxpayer identifiers |
| `agent.tool.called` | `messageId`, `toolName`, `latencyMs`, `aigpPolicyOutcome`, `aigpRiskScore` | Tool input parameters, tool output content |
| `agent.resolution.completed` | `messageId`, `agentName`, `resolutionType`, `totalTokens`, `totalLatencyMs` | Reasoning summaries, response content |
| `agent.hitl.escalated` | `messageId`, `agentName`, `escalationReason` (category only), `timestamp` | Officer identity, taxpayer details |
| `agent.error` | `messageId`, `agentName`, `errorCode`, `errorCategory`, `timestamp` | Stack traces with application internals (sanitised before publishing) |

**Event Hubs capture**: Azure Event Hubs Capture is enabled to write event batches to Azure Blob Storage (SOC-owned storage account) at 5-minute intervals. SOC teams consume from this storage rather than needing direct Event Hubs connectivity.

**Pros**:
- Append-only Cosmos DB with RBAC-enforced no-delete provides a tamper-evident record that can withstand audit scrutiny
- Analytical store + Synapse Link enables complex audit queries over years of data without impacting operational performance
- Event Hubs SOC stream satisfies SOC monitoring requirements without exposing sensitive data — the schema explicitly enumerates every excluded field, providing a clear data-sharing agreement

**Cons**:
- Cosmos DB costs are usage-based (RU/s + storage); at <1,000 emails/day with ~5 steps/email, this is approximately 5,000 documents/day — modest cost (approximately SGD 30–50/month at autoscale settings)
- Event Hubs adds another service to manage; at low email volumes, a Log Analytics workspace export to SOC could suffice, but Event Hubs provides a cleaner data-sharing boundary
- RBAC-enforced append-only on Cosmos DB requires custom Cosmos DB role definitions (not available through the portal; must be created via ARM template or Terraform)

---

### Option 2: Append Blobs in Azure Blob Storage

**Implementation**: Audit records are written as individual JSON blobs to Azure Blob Storage with immutability policies enabled (WORM — Write Once Read Many).

**Pros**: Even lower cost than Cosmos DB; WORM policy provides stronger immutability than Cosmos DB RBAC controls; straightforward to archive

**Cons**:
- No native query capability — investigating a specific email's trace requires downloading and parsing individual blobs or setting up a separate query layer (Azure Data Explorer, Log Analytics)
- WORM lock precludes any modification including metadata updates; very rigid for a system still evolving its audit schema
- No partition/indexing structure; queries over large time ranges scan the entire blob container

---

### Recommendation Justification

**Option 1** is recommended. The Cosmos DB append-only ledger provides the query performance needed for incident investigation (find all actions by agent version X in the last 6 hours) while the RBAC controls provide adequate tamper-evidency for compliance. The Event Hubs SOC stream cleanly separates compliance audit data from security monitoring data — SOC receives real-time operational events without receiving audit ledger content that would pose data governance risks.

> **Compliance Note (PDPA)**: The audit ledger is a processing record under PDPA. The data minimisation design (hash references, summarised outputs) must be validated with the organisation's Data Protection Officer (DPO) to confirm that the record does not constitute a personal data store under PDPA definitions. If the hash-to-taxpayer mapping is held by internal microservices and the audit ledger holds only hashes, the ledger may qualify as pseudonymised data rather than personal data — a material distinction for PDPA obligations.

> **Compliance Note (IM8)**: Audit records must be retained for a minimum of 5 years. The Cosmos DB → Blob Storage archive pipeline must be tested and verified at least annually.

---

## 2.7 Behavioral Anomaly Detection

### Implementation Overview

Standard observability (Step 2.5) monitors whether agents are operating within quantitative thresholds (latency, token spend, error rate). Behavioral anomaly detection addresses a different failure mode: **the agent is operating within normal metrics but its reasoning or tool-call pattern has changed in a way that indicates unexpected or unsafe behaviour**.

Examples:
- An agent that normally makes 2–3 tool calls per resolution starts consistently making 6–8 tool calls (possible agentic loop or confusion state)
- An agent starts calling a tool it rarely used before (possible prompt injection causing abnormal tool selection)
- An agent's reasoning step count per resolution is abnormally high for a simple SOP (possible over-analysis or hallucination spiral)

These patterns may not immediately surface as errors (the agent may still resolve successfully), but they represent behavioral drift that often precedes visible failures.

---

### Option 1 (Recommended): Embedding-Based Behavioral Baseline Comparison via Azure AI Search

**Technology Stack**: Azure AI Search (vector index), Azure Container Apps Job (baseline and comparison compute), Azure OpenAI (text-embedding-3-small for behavioral vector encoding), Azure Monitor (anomaly metric publishing)

---

#### Behavioral Feature Vector Design

For each completed email processing event, a **behavioral feature vector** is computed from the OTel trace:

| Feature | Encoding |
|---|---|
| Tool call sequence | Ordered list of tool names encoded as a sequence embedding (using `text-embedding-3-small` on the concatenated tool name sequence string, e.g., `"get_taxpayer_record → get_refund_history → send_email_reply"`) |
| Reasoning step count | Scalar (number of LLM invoke spans) |
| Tool call count | Scalar (total tool call spans) |
| Token utilisation ratio | Scalar (completion tokens / prompt tokens) — high ratio may indicate verbose/confused reasoning |
| Resolution type | One-hot encoded categorical: `automated` / `hitl` / `failed` |
| P95 latency bucket | Ordinal bucket: `fast` (<5s) / `normal` (5–30s) / `slow` (>30s) |

The tool-call sequence embedding is the most information-rich feature. It encodes the semantic meaning of the agent's action sequence, meaning that two agents that called different tools in different orders but with semantically similar intent will produce similar vectors — while an agent that called an unusual combination of tools will produce a dissimilar vector.

#### Baseline Generation

At every production deployment (during Stage 4 of the deployment pipeline), a **baseline behavioral profile** is generated using the first 200 processed emails in the canary phase:

1. Compute behavioral feature vectors for all 200 canary traces
2. Compute the centroid of the 200 vectors (mean vector in embedding space)
3. Compute the P95 radius (the distance from the centroid within which 95% of vectors fall)
4. Store the centroid vector and P95 radius in the **Azure AI Search behavioral baseline index** under a document keyed by `{agentName}-{semver}`:

```json
{
  "id": "refund-agent-1.3.0",
  "agentName": "refund-agent",
  "semver": "1.3.0",
  "baselineVector": [...],
  "p95Radius": 0.18,
  "sampleCount": 200,
  "computedAt": "2026-05-05T12:00:00Z"
}
```

#### Runtime Anomaly Detection

An **Azure Container Apps Job** (`ao-anomaly-detector`) runs every 30 minutes:

1. Retrieves the last 30 minutes of completed email traces from Langfuse for each active agent
2. Computes behavioral feature vectors for each trace
3. Queries the Azure AI Search behavioral baseline index for the agent's current-version baseline
4. Computes the cosine distance between each trace vector and the baseline centroid
5. Flags a trace as **anomalous** if its distance exceeds the baseline P95 radius
6. If the **anomaly rate** (% of traces flagged) in the last 30-minute window exceeds 20% for the same agent, publishes an anomaly event to Azure Monitor and to a Service Bus topic (`ops-anomaly-events`)

The anomaly event triggers:
- A P2 alert to the AO team (investigate whether the agent is in an unexpected state)
- If anomaly rate exceeds 50%: automatic escalation to the Emergency Stop workflow (Step 3.2)

> **This mechanism is a supplement, not a replacement, for standard threshold monitoring.** An agent could exhibit behavioral drift (unusual tool-call sequences) while remaining within latency and token budgets. Behavioral anomaly detection catches this class of failure that metrics-only monitoring misses.

**Pros**:
- Embedding similarity is a semantically meaningful distance — it catches changes in agent "intent" (which tools it chooses and in what order), not just volumetric changes
- Self-calibrating: baseline is auto-generated at deployment; no manual threshold tuning required beyond the P95 radius
- Azure AI Search's vector search capability makes the baseline comparison computationally trivial (single vector lookup per trace batch)
- Integrated with Emergency Stop workflow (Step 3.2) for automatic escalation on severe behavioral drift

**Cons**:
- Baseline generated from 200 canary traces may not cover the full distribution of email types, especially for agents handling diverse SOP subcategories; baseline quality improves over time as more emails are processed
- `text-embedding-3-small` adds a per-trace encoding cost (minimal — approximately 20 tokens per tool-call sequence)
- 30-minute detection cadence means behavioral drift can persist for up to 30 minutes before detection; for a <1,000 emails/day platform, this translates to approximately 20 affected emails maximum before detection

---

### Option 2: Statistical Threshold Monitoring on OTel Metrics

**Implementation**: Set fixed Azure Monitor alert thresholds on `ao.email.tool_calls.count` (e.g., alert if rolling average > 5 calls/email for an agent that typically makes 3) and `ao.email.tokens.completion` deviation.

**Pros**: Zero additional infrastructure; these metrics are already flowing to Azure Monitor (Step 2.5)

**Cons**:
- Fixed thresholds require manual calibration per agent and do not self-adjust when the agent's behaviour legitimately changes (e.g., a new SOP subcategory that requires 5 tool calls becomes "normal" for that agent after a MINOR version update)
- Cannot detect qualitative changes in tool-call pattern (calling unusual tools) — only quantitative changes (tool call count)
- Produces high false-positive alert rates during canary periods when behavior is intentionally changing

---

### Recommendation Justification

**Option 1** is recommended as the primary behavioral anomaly detection mechanism. **Option 2 is deployed in parallel** as a "belt and braces" first-pass threshold check, given that the threshold metrics (tool call count, token spend) are already flowing to Azure Monitor at zero additional cost. The two mechanisms catch complementary classes of anomalies: Option 2 catches gross quantitative outliers immediately; Option 1 catches subtle but systematic behavioral pattern changes over a 30-minute window.

> **Integration with Step 3.2 (Emergency Stop)**: The `ops-anomaly-events` Service Bus topic is the shared integration point between the anomaly detection job and the Emergency Stop runbook. The runbook subscribes to this topic and applies automatic intervention logic based on the anomaly severity level encoded in the event payload.
