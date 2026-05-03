# AIBP Operations Blueprint
## Step 1: Ingestion & Triage Operations (SWEE Layer)

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Ensuring the platform entry point is resilient, observable, and that every email is correctly classified before committing compute resources downstream.

**SWEE Context**: SWEE is a greenfield AI-powered email triage system and the sole entry point for taxpayer emails into the AIBP platform. It uses semantic vector search against a large SOP corpus to classify each inbound email to the closest matching SOP before handing off to the Agentic Orchestration (AO) layer. Its primary operational responsibilities are:

1. Receiving and authenticating inbound emails
2. Applying PII anonymisation before any AI processing
3. Classifying the email to the appropriate SOP via vector similarity search
4. Enqueuing the classified payload reliably for the AO layer

Because this is a fully greenfield system with no pre-existing triage baseline, all monitoring frameworks described in this step are designed to establish operational baselines organically from Day 1 of production.

---

## 1.1 Email Ingestion & Queue Management

### Implementation Overview

Before SWEE routes an email to the AO layer, it deposits the message into an intermediary queue. This queue serves four operational purposes:

1. **Durability**: If the AO layer is temporarily unavailable (e.g., during deployment, throttling, or outage), emails are not lost.
2. **Backpressure**: The queue absorbs burst traffic and prevents the AO layer from being overwhelmed, protecting Azure OpenAI quota consumption.
3. **Replayability**: Failed processings can be retried up to a configured maximum. Messages that exhaust retries are moved to a Dead Letter Queue (DLQ) for investigation and manual recovery.
4. **Auditability**: Each message carries a unique `MessageId` assigned by SWEE that becomes the root correlation ID for all downstream distributed tracing.

SWEE is the producer; AO container apps are consumers subscribed to their respective SOP-specific topics.

---

### Option 1 (Recommended): Azure Service Bus Premium + APIM Rate Limiting + Automated DLQ Runbook

**Technology Stack**: Azure Service Bus (Premium tier), Azure API Management, Azure Monitor, Azure Automation

---

#### Azure Service Bus — Premium Tier

Azure Service Bus Premium is selected over Standard for four specific requirements:

- **Private Endpoint support**: The Standard tier does not support VNet integration or private endpoints. The Premium tier is exposed exclusively over a private endpoint within the GCC 2.0 Virtual Network, ensuring email metadata never traverses the public internet between SWEE and the queue.
- **Message Sessions**: Enables FIFO ordering within a session boundary. SWEE sets `SessionId` = SHA-256 hash of the taxpayer's email address. This guarantees that multiple emails from the same taxpayer are processed sequentially, preventing race conditions when agents attempt concurrent writes to the same taxpayer record.
- **Geo-Disaster Recovery**: Premium supports namespace pairing to a secondary Azure region. In the event of a regional outage, the secondary namespace activates with an RPO of approximately 0 (no message loss) and an RTO of under 10 minutes.
- **Scalable messaging units**: At <1,000 emails/day, a single messaging unit (1 MU) is sufficient. Capacity scales in-place without re-provisioning.

**Queue and Topic Design**:

| Parameter | Value | Rationale |
|---|---|---|
| Entity type | Topic + Subscriptions | Allows fan-out to multiple SOP-specific consumers |
| Topic name | `email-triage-{env}` (e.g., `email-triage-prod`) | Environment-isolated |
| Subscriptions | One per SOP category (e.g., `sop-refund`, `sop-query`, `sop-dispute`) | Each AO agent container subscribes to its SOP |
| Message TTL | 7 days | Messages not consumed within 7 days are dead-lettered |
| Max delivery count | 3 | After 3 failed deliveries, message moves to DLQ |
| Session-based ordering | Enabled | `SessionId` = SHA-256(taxpayer email address) |
| Message lock duration | 5 minutes | Sufficient for AO processing; extendable via lock renewal |

**Message Payload Schema**:

```json
{
  "messageId": "uuid-v4",
  "correlationId": "uuid-v4",
  "emailBlobRef": "https://<storage>.blob.core.windows.net/<container>/<encrypted-blob-id>",
  "sopId": "SOP-REFUND-001",
  "triageCategory": "refund",
  "triageConfidenceScore": 0.94,
  "topKCandidates": [
    { "sopId": "SOP-REFUND-001", "score": 0.94 },
    { "sopId": "SOP-REFUND-002", "score": 0.71 },
    { "sopId": "SOP-QUERY-015", "score": 0.62 }
  ],
  "sweeVersion": "1.2.0",
  "timestamp": "2026-05-03T08:00:00Z",
  "sessionId": "sha256-hash-of-taxpayer-email"
}
```

> **PII Control**: The email body and taxpayer identifiers are **not** stored in the message payload. SWEE stores the anonymised email as an encrypted blob in Azure Blob Storage (customer-managed key via Azure Key Vault). The message payload carries only a blob reference. AO fetches the encrypted blob during processing via its Managed Identity. Raw email content never traverses Service Bus.

> **Note on `topKCandidates`**: The top-3 candidates and their similarity scores are embedded in the message payload at no additional cost (they are a direct output of SWEE's vector search). They serve two downstream purposes: (a) confidence monitoring in Step 1.2, and (b) contrastive context for the AO agent if it needs to infer the closest SOP match in edge cases.

---

#### Azure API Management (APIM) — Rate Limiting

SWEE's Service Bus publish calls pass through an APIM policy layer that enforces:

- **Rate limiting**: `rate-limit-by-key` policy with key = SWEE application identity, limit = 100 messages/minute. When the limit is exceeded, SWEE holds messages in-queue (natural backpressure) rather than dropping them.
- **JWT validation**: All calls from SWEE use its Azure Managed Identity to obtain an Entra ID JWT. No credentials are stored in application configuration.
- **Correlation header propagation**: APIM injects `x-correlation-id` from the Service Bus `MessageId` into all downstream request headers, ensuring end-to-end trace continuity through the OTel pipeline (see Step 2.5).

---

#### Dead Letter Queue (DLQ) Automation

DLQ messages represent emails the platform has failed to process. Each DLQ message must be investigated and either reprocessed or escalated to a human.

1. An Azure Monitor **metric alert** fires when DLQ message count > 5 (configurable).
2. The alert triggers an **Azure Automation Runbook** (`ops-dlq-recovery.ps1`) that:
   - Reads up to 50 DLQ messages per execution
   - Classifies each by failure reason (using the Service Bus `DeadLetterReason` and `DeadLetterErrorDescription` properties)
   - For **retryable failures** (transient AO timeout, lock expiry): re-enqueues to the main topic
   - For **non-retryable failures** (schema parse error, AO hard rejection): writes message metadata to an Azure Table Storage `ops-dlq-log` table and posts an alert to the Ops team via Azure Monitor Action Group (email + Teams webhook)
3. DLQ messages older than 24 hours that remain unprocessed automatically generate a **P2 incident ticket**.

**Pros**:
- GCC 2.0-compliant: fully accessible over private endpoints, no public SaaS dependency
- Session-based FIFO ordering prevents taxpayer record corruption from concurrent agent writes
- Automated DLQ recovery reduces operational toil at <1,000 emails/day scale
- APIM provides a single, auditable, policy-controlled choke point for all downstream access

**Cons**:
- Premium tier costs approximately SGD 680/month per namespace (vs SGD 10 for Standard); justified by private endpoint and session requirements
- APIM adds approximately 5–10 ms latency per downstream API call
- DLQ runbook requires ongoing maintenance as new failure patterns emerge

---

### Option 2: Azure Queue Storage + Static Application-Level Throttle

**Technology Stack**: Azure Storage Queues, SWEE application-level semaphore

**Implementation**: SWEE writes email metadata directly to Azure Queue Storage. A static concurrency semaphore in the SWEE application code limits throughput to 50 emails/minute. AO polls the queue on a fixed timer.

**Pros**:
- Significantly cheaper: ~SGD 0.0005 per 10,000 operations
- Simpler to set up; no additional Azure resources required beyond Storage Account

**Cons**:
- No Private Endpoint on Standard tier — fails GCC 2.0 network isolation requirements
- Message size limit of 64 KB; message payload with blob references and topK candidates may approach this
- No message session support — email thread ordering not guaranteed; taxpayer record race conditions are possible
- No native geo-replication; messages are at risk during regional failure
- Static throttle requires application redeployment to adjust; cannot adapt to AO quota availability dynamically

---

### Option 3: Azure Event Hubs + Consumer Groups per SOP

**Technology Stack**: Azure Event Hubs (Premium), LangGraph consumer clients in ACA

**Implementation**: SWEE publishes email-triage events to an Event Hubs namespace. AO agents consume from partition-assigned consumer groups, one per SOP. Event Hub's configurable retention enables event replay.

**Pros**:
- High-throughput ceiling — significant headroom if email volumes scale dramatically
- Native event replay via consumer offset management
- Private Endpoint supported on Premium tier
- Good integration with Azure Stream Analytics for real-time analytics

**Cons**:
- Event Hubs is a streaming model, not a task queue — no native concept of message completion, no built-in DLQ semantics, no delivery count tracking. Failed processing requires custom dead-letter logic
- Consumer offset management across SOP partitions adds operational complexity disproportionate to <1,000 emails/day
- No message session support for FIFO ordering within a taxpayer thread

---

### Recommendation Justification

**Option 1** is recommended. At <1,000 emails/day, throughput is not the constraint — correctness, durability, and compliance are. Azure Service Bus Premium is the only option that satisfies all three: private endpoint for GCC 2.0 network isolation, message sessions for taxpayer FIFO ordering, and native DLQ semantics for failed-message recovery without custom code. The cost premium over Option 2 is justified by the elimination of two critical operational risks: silent email loss during AO outages and taxpayer record corruption from out-of-order processing.

> **Compliance Note (IM8)**: All messages in the Service Bus queue must be tagged with the appropriate IM8 data classification label at the namespace level. Access to the queue must be controlled exclusively via Azure RBAC with Managed Identities. Shared Access Signature (SAS) keys must not be used in application or pipeline code.

---

## 1.2 Intent Triage Accuracy Monitoring (EvalOps — Tier 0)

### Implementation Overview

SWEE's SOP classification is the first and most consequential AI decision in the pipeline. A misclassified email routed to the wrong SOP causes the AO agent to execute an incorrect workflow, potentially performing wrong actions on taxpayer records. The downstream cost is high: agent tokens are consumed on an irrelevant workflow, HITL officer time is consumed to correct the error, and the email must be fully reprocessed from the correct SOP.

**The nature of SWEE's triage** must be understood before designing an accuracy monitoring strategy. SWEE is not selecting from a small, enumerable list of categories — it performs vector similarity search against a corpus of potentially thousands of SOPs. The output is not a classification label chosen from a dropdown; it is the nearest-neighbour document(s) in an embedding space.

This has a fundamental implication for monitoring: **any monitoring approach that attempts to independently re-classify the email faces the same information problem as SWEE** — it would need to perform the same retrieval against the same SOP corpus. A shadow LLM model that re-triages every email is not truly independent validation; it is just running SWEE's logic twice under a different name, at additional cost.

The correct operational strategy is therefore to **monitor the signals that SWEE itself already produces** (confidence/similarity scores and top-K candidate gaps), supplement with **downstream correction signals** (AO rejections, HITL corrections), and anchor accuracy measurement to **human ground truth** obtained through structured officer audits.

**Key Metrics**:

| Metric | Definition | Target (Proposed) |
|---|---|---|
| Top-1 Similarity Score (mean) | Mean cosine similarity score between the email embedding and the top-ranked SOP | Establish baseline in first 30 days; alert on > 5% drop |
| Top-1 vs Top-2 Gap | Difference in similarity score between the top-ranked and second-ranked SOP candidate | Alert when mean gap < 0.10 (indicates systematic ambiguity) |
| Low-Confidence Rate | % emails where top-1 similarity score < configured threshold (e.g., 0.75) | < 15% |
| Officer-Verified Accuracy Rate | % of audited emails confirmed as correctly triaged by tax officers | ≥ 95% (target, to be baselined in first 90 days) |
| AO Rejection Rate | % emails where the AO agent rejects the routed SOP and escalates for re-classification | < 5% |
| Misroute Correction Rate | % emails re-classified after HITL correction | Track; no hard target until baseline established |

---

### Option 1 (Recommended): Confidence Signal Monitoring + Embedding Drift Detection + Weekly Stratified Officer Audit

**Technology Stack**: Azure Monitor / Application Insights (custom metrics), Azure AI Search, Azure Container Apps Job (weekly sampler), Azure Table Storage or Azure SQL Database (audit review store), self-hosted Langfuse on ACA (evaluation dataset management)

---

#### Part A: Real-Time Confidence Signal Monitoring

SWEE's vector similarity search already produces the most reliable real-time proxy for triage quality — the similarity scores themselves. No additional model inference is required.

**Implementation**:

1. SWEE emits the following as **Azure Monitor custom metrics** after each triage decision, using the Application Insights SDK (or OTel custom metric exporter):

   | Metric name | Value | Dimensions |
   |---|---|---|
   | `swee.triage.similarity.top1` | Cosine similarity score of top-ranked SOP | `sopId`, `triageCategory`, `sweeVersion` |
   | `swee.triage.similarity.gap` | top-1 score minus top-2 score | `triageCategory`, `sweeVersion` |
   | `swee.triage.confidence.band` | Bucketed score: `high` (≥0.85), `medium` (0.70–0.85), `low` (<0.70) | `triageCategory` |

2. **Azure Monitor alert rules**:

   | Alert | Condition | Severity | Action |
   |---|---|---|---|
   | Similarity score degradation | Rolling 1-hour mean of `top1` drops > 5% vs. 7-day baseline | P2 | Notify AO team + Ops |
   | Systematic ambiguity | Rolling 1-hour mean `gap` drops below 0.10 | P2 | Notify AO team |
   | Low-confidence surge | % of `low` band emails exceeds 15% in any 1-hour window | P3 | Notify Ops |
   | Confidence threshold breach | Any 5 consecutive low-confidence emails (<0.70) within 15 minutes | P3 | Log to review queue |

3. These metrics are visualised in an **Azure Monitor Workbook** dashboard ("SWEE Triage Health"). The dashboard shows rolling 24-hour and 7-day trend lines for each metric, enabling Ops to distinguish short-term anomalies from sustained drift.

**Why similarity score monitoring is the right real-time signal**:

The similarity score from SWEE's vector search directly encodes how confidently the model believes the email belongs to the top-ranked SOP. When the score drops or when the top-1 vs top-2 gap narrows, it means SWEE is genuinely uncertain between two SOPs — this is precisely the condition where misrouting is most likely. Monitoring this score distribution continuously provides a real-time signal of triage quality without any additional model inference or cost.

---

#### Part B: Embedding Drift Detection

Over time, taxpayer email language may shift (e.g., new terminology for tax issues, policy changes that introduce new query types that don't map cleanly to existing SOPs). When this happens, the email embedding space drifts relative to the SOP embedding space, and retrieval quality degrades — even if SWEE's internal model weights are unchanged.

**Implementation**:

1. A weekly **Azure Container Apps Job** (`swee-drift-monitor`) executes the following:
   - Retrieves the last 7 days of email embeddings from SWEE's vector store (the vectors, not the raw emails — no PII)
   - Computes the centroid of the weekly email embedding distribution
   - Computes the cosine distance between this centroid and the centroid of the full SOP embedding corpus in Azure AI Search
   - Compares against the centroid distance from the previous 4 weeks (rolling baseline)

2. If the centroid distance increases by more than a configurable threshold (e.g., 0.05) week-over-week, a **drift alert** is published to Azure Monitor and sent to the App & Data / SWEE team as a signal that the SOP corpus or the email embedding model may need to be updated.

3. The drift metric (`swee.embedding.centroid_distance`) is logged to Azure Monitor and included in the SWEE Triage Health Workbook.

---

#### Part C: Weekly Stratified Officer Audit

The officer audit provides the only true ground-truth accuracy measurement in this system. Similarity scores are a proxy; only a human expert can confirm whether a triage decision was operationally correct.

**Stratified sampling design**:

The audit samples emails across three confidence bands to maximise the information value per officer-hour spent:

| Band | Similarity threshold | Sample percentage | Rationale |
|---|---|---|---|
| High confidence | ≥ 0.85 | 2% | Validate that high-confidence outputs are indeed correct; detect systematic errors in confident predictions |
| Medium confidence | 0.70–0.85 | 10% | The most informative band — SWEE is uncertain; officer review reveals whether uncertainty led to misrouting |
| Low confidence | < 0.70 | 25% | Highest misroute risk; needs closer scrutiny |

At <1,000 emails/day (~7,000/week), this sampling yields approximately:
- ~140 high-confidence cases × 2% = ~3 samples
- ~4,200 medium-confidence cases × 10% = ~420 samples
- ~700 low-confidence cases × 25% = ~175 samples
- **Total: ~600 cases/week** for officer review

This load is distributed across the tax officer pool and can be weighted by officer workload. Reviews are expected to take 1–2 minutes per case at the triage level (the officer reads the email excerpt and confirms or corrects the SOP label — they are not solving the case).

**Implementation**:

1. A weekly **Azure Container Apps Job** (`swee-audit-sampler`) executes on Monday 08:00:

   - Queries Langfuse (self-hosted) for the past week's triage decision traces, stratified by confidence band
   - Selects samples according to the percentages above using random sampling
   - For each selected case, retrieves the anonymised email excerpt from Azure Blob Storage (using the `emailBlobRef` in the trace)
   - Writes the sample set to an **Azure Table Storage** table `swee-audit-queue`, with fields:
     - `messageId` (correlation ID)
     - `emailExcerpt` (anonymised, PII-stripped by SWEE's original processing)
     - `sweeLabel` (SWEE's triage category)
     - `similarityScore`
     - `topKCandidates` (the shortlisted SOP IDs and scores)
     - `weekBatch` (ISO week number)
     - `reviewStatus` (pending / reviewed)
     - `officerLabel` (populated by officer)
     - `reviewedBy` (officer ID)
     - `reviewedAt`

2. Tax officers access the review queue via a lightweight **Power Apps canvas app** ("SWEE Audit Tool") that presents email excerpts and the top-K SOP candidates as a multiple-choice selection. Officers confirm or correct the triage label. The tool writes the `officerLabel` and `reviewedBy` back to Table Storage.

   > **Alternative if Power Apps has licensing constraints in GCC 2.0**: A simple custom React webapp hosted as an Azure Static Web App, backed by an Azure Function API that reads/writes to Table Storage. This is a lighter-weight alternative with no Power Platform licensing requirement.

3. A weekly **Azure Logic Apps** workflow runs on Friday 17:00 to:
   - Aggregate the week's completed audit records
   - Compute Officer-Verified Accuracy Rate by band and overall
   - Write a summary record to Langfuse as a `dataset` score annotation on the corresponding traces
   - Post the weekly accuracy summary to the Ops team Teams channel via webhook

4. Officer-labelled ground-truth records accumulate in Langfuse as a time-series evaluation dataset. This dataset is used to:
   - Track accuracy trend week-over-week
   - Identify specific SOP categories with systematically low accuracy (requiring SWEE prompt or embedding refresh)
   - Provide the ground-truth validation set for SWEE model updates in CI/CD (see Step 2.2)

**Greenfield bootstrapping (First 30 Days)**:

Because no historical baseline exists at launch, the sampling percentages above are increased for the first 30 days:

| Band | Launch sampling rate | Steady-state rate (after 30 days) |
|---|---|---|
| High | 5% | 2% |
| Medium | 20% | 10% |
| Low | 50% | 25% |

This doubles the ground-truth collection rate during the calibration period, establishing the Officer-Verified Accuracy Rate baseline earlier. The launch sampling rates are also used to calibrate the relationship between similarity score bands and actual accuracy — i.e., to confirm that the threshold values (0.70, 0.85) chosen above are appropriate for this specific SOP corpus.

**Pros (Option 1 overall)**:
- No additional LLM inference cost for triage monitoring — similarity scores are a free by-product of SWEE's vector search
- Confidence signal monitoring provides near-real-time (per-email) quality proxy
- Weekly officer audit provides true human ground-truth; officers' domain expertise cannot be replicated by any automated critic
- Embedding drift detection catches a failure mode that similarity score monitoring alone misses: gradual corpus drift
- Langfuse accumulates a growing evaluation dataset that improves CI/CD eval precision over time
- Fully GCC 2.0-compliant: Azure Monitor, Azure AI Search, Langfuse self-hosted, Power Apps (or Azure Static Web App), Table Storage — no external SaaS

**Cons**:
- Similarity scores are a proxy, not a direct accuracy measurement; they can be high even when SWEE makes a semantically wrong but metrically close match
- Officer audit introduces recurring operational burden (~600 reviews/week at launch); workload must be planned into officer rosters
- Embedding drift detection requires access to embedding vectors, which means SWEE's vector store must expose query capability to the drift monitor job (read-only managed identity access)

---

### Option 2: Contrastive Top-K Critic (Low-Confidence Subset Only)

**Technology Stack**: Azure OpenAI (GPT-4o-mini), Azure Container Apps Job, Azure AI Language (PII detection), Langfuse (self-hosted)

**Implementation**: For emails where SWEE's top-1 similarity score falls below a threshold (e.g., 0.75), an asynchronous **Critic ACA Job** is triggered. The critic does not attempt to independently re-triage the email against the full SOP corpus. Instead, it receives only the **already-retrieved top-3 candidate SOP names and brief descriptions** (not the full SOP text) alongside an anonymised email excerpt, and is asked to validate SWEE's ranking — i.e., does the top-ranked candidate seem more appropriate than the second or third?

```
System: You are an expert at classifying government tax authority emails.
        You are presented with an email and the top 3 SOP categories
        already shortlisted by a retrieval system.

User:   Email excerpt (PII removed): "{anonymisedExcerpt}"

        Shortlisted SOP candidates:
        1. {sop1_name}: {sop1_description} (similarity: {score1})
        2. {sop2_name}: {sop2_description} (similarity: {score2})
        3. {sop3_name}: {sop3_description} (similarity: {score3})

        Does the top-ranked candidate (#1) appear to be the most appropriate?
        Respond: { "topRankCorrect": true/false, "preferredRank": 1/2/3, "reasoning": "..." }
```

**Pros**:
- The critic validates a ranking decision (among 3 candidates) rather than re-performing retrieval — this is a fundamentally tractable expert task, unlike classifying against thousands of SOPs
- Targets only the genuinely ambiguous cases where additional validation is most valuable
- GPT-4o-mini is cost-effective; at <150 low-confidence emails/day, cost is approximately SGD 0.002/email for the critic call

**Cons**:
- The critic's validation is bounded by SWEE's retrieval quality — if the correct SOP was not in the top 3, the critic cannot identify it. The critic can at best re-rank what SWEE already found.
- Adds ACA Job infrastructure, Service Bus secondary subscription, and Langfuse trace overhead for a subset signal already partially covered by the officer audit
- Introduces a second LLM in the monitoring layer that itself can hallucinate or misrank, requiring its own quality monitoring over time
- Does not add value when the correct SOP is the top-1 result delivered at low confidence (SWEE was correct but uncertain — a common occurrence for novel email phrasings)

---

### Option 3 (Not Recommended): Full Independent Re-Triage Shadow Critic

**Technology Stack**: Azure OpenAI, Azure AI Search, Azure Container Apps Job

**Implementation**: An LLM-backed shadow critic independently performs the full triage process on each email in parallel to SWEE, using its own vector retrieval against the SOP corpus. The critic's output is compared to SWEE's label; disagreements are flagged.

**Why this does not work**:

This approach sounds intuitively appealing — have a second model independently verify the first. In practice, it has two fatal flaws for this specific platform:

1. **The critic faces the same retrieval problem as SWEE.** With thousands of SOPs, no LLM can hold all SOP summaries in its context window. The critic must also perform vector similarity retrieval against the same corpus. If the critic uses the same embedding model and the same index, it will produce the same results as SWEE (the critic is just SWEE running twice). If it uses a different model or index, any disagreement is ambiguous — it is not clear whether SWEE or the critic is correct.

2. **Agreement proves nothing; disagreement proves nothing.** If the two systems agree, it could mean both are correct, or both have the same systematic bias. If they disagree, it could mean SWEE is wrong, or the critic is wrong. Without human ground truth to arbitrate, the disagreement signal is not actionable.

**Pros**:
- None that are not better served by Option 1 or Option 2

**Cons**:
- Doubles the per-email triage cost (tokens + compute)
- Produces an ambiguous signal that cannot be acted upon without human verification
- Adds significant infrastructure complexity for no demonstrable accuracy gain

---

### Recommendation Justification

**Option 1** is recommended. The fundamental insight is that SWEE's own output — the similarity scores and top-K candidate gaps — is the most informative and cost-free real-time signal available. Monitoring this distribution continuously provides a near-real-time proxy for accuracy degradation without any additional model inference.

**Option 2** (contrastive critic) is a viable supplementary mechanism for teams that want an additional automated signal on low-confidence cases, but is not necessary to implement at launch. It should be considered as a Phase 2 enhancement once the Option 1 baseline has been established and the value of additional automation on the low-confidence subset is quantified.

**Option 3** is explicitly not recommended and should not be proposed or implemented.

The officer audit in Option 1 is the non-negotiable component: it is the only mechanism that provides true accuracy ground truth. It also builds officer familiarity with the platform's edge cases and failure modes — a benefit that extends beyond quality monitoring to better HITL decision-making in Step 3.

> **Compliance Note (PDPA)**: All email excerpts presented to officers in the audit tool must have passed through SWEE's original PII anonymisation step. The audit tool must not display raw email content. Officer review actions (label, timestamp, officer ID) constitute a processing record under PDPA and must be retained.

> **Compliance Note (IM8)**: Audit logs of all triage decisions — SWEE's classification label, similarity score, officer-verified label (where available), and any automated flags — must be retained for a minimum of 5 years. Langfuse's PostgreSQL backend requires daily automated backups with point-in-time restore, and backup retention set to at least 35 days with archival offload to Azure Blob Storage (cool tier) for long-term retention compliance.
