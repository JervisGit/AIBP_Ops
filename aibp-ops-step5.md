# AIBP Operations Blueprint
## Step 5: FinOps — Token & Cost Management

| Field | Value |
|---|---|
| **Parent document** | `aibp-ops-preamble.md` |
| **Version** | 0.1 (Draft — In Progress) |
| **Date** | 3 May 2026 |
| **Classification** | Internal — Restricted |

---

**Operational Focus**: Controlling, attributing, and reporting the cost of operating the AIBP platform — ensuring every dollar of AI spend is visible, attributable to a specific agent and email, benchmarked against the cost of human alternatives, and protected by automated guardrails against runaway costs.

**Why FinOps is a first-class operational concern for agentic AI**: Unlike traditional software, where compute cost is relatively predictable and scales linearly with usage, LLM-based agents have inherently variable token consumption. A single email that causes an agent to enter an iterative reasoning loop can consume 50× the tokens of a routine case. Without per-session cost attribution and guardrails, a small number of "stuck" agents can quietly exhaust the monthly Azure OpenAI budget while processing a tiny fraction of total emails.

---

## 5.1 Token & Cost Monitoring Per Agent and Per Email

### Implementation Overview

Every email processing event must produce a cost record attributing the Azure OpenAI token spend and ACA compute cost to that specific email, agent, and SOP category. This enables:

- Detecting stuck or looping agents (emails consuming disproportionate token spend)
- Identifying which SOP categories are the most expensive to process
- Building the cost comparison model (Step 5.2)
- Triggering cost guardrails before budget thresholds are breached

---

### Option 1 (Recommended): Per-Session Cost Attribution via OTel Custom Metrics + Azure Cost Management + Automated Guardrails

**Technology Stack**: Azure Monitor / Application Insights (OTel custom metrics from Step 2.5), Azure Cost Management, Log Analytics (KQL cost queries), Azure Monitor alert rules, Azure App Configuration (guardrail thresholds), Azure Container Apps Jobs (cost aggregation and anomaly detection)

---

#### Per-Session Cost Attribution

**Token cost per email** is captured as part of the OTel instrumentation already described in Step 2.5. The relevant custom metrics per email processing event are:

| Metric | Already captured in Step 2.5? | Additional detail for FinOps |
|---|---|---|
| `ao.email.tokens.prompt` | Yes | Must include tokens from ALL LLM calls in the full agent run (not just the last call) — the OTel span hierarchy aggregates this |
| `ao.email.tokens.completion` | Yes | Same as above |
| `ao.email.tool_calls.count` | Yes | Proxy for AIGP API call cost (each tool call has a fixed overhead) |
| `ao.email.latency_ms` | Yes | Used to estimate ACA compute cost (duration × per-minute compute rate) |

**Derived cost metrics** (computed by a daily Azure Container Apps Job, `finops-cost-aggregator`):

| Derived metric | Formula | Written to |
|---|---|---|
| `cost.tokens.usd` per email | `(prompt_tokens × prompt_token_price) + (completion_tokens × completion_token_price)` | Log Analytics custom table `AIBPEmailCosts` |
| `cost.compute.usd` per email | `(latency_ms / 60,000) × aca_per_minute_cost` | Log Analytics custom table `AIBPEmailCosts` |
| `cost.total.usd` per email | `cost.tokens.usd + cost.compute.usd` | Log Analytics custom table `AIBPEmailCosts` |
| `cost.total.sgd` per email | `cost.total.usd × fx_rate` (FX rate refreshed daily from Azure Cost Management) | Log Analytics custom table `AIBPEmailCosts` |

**Azure OpenAI token pricing** (frontier model — confirm GCC 2.0 rates at time of procurement):

| Component | Rate |
|---|---|
| Frontier model — prompt tokens (e.g., GPT-5 or next-generation equivalent) | TBC: confirm with Azure/Microsoft account team; indicatively 3–5× higher than GPT-4o rates |
| Frontier model — completion tokens | TBC: confirm with Azure account team; indicatively 3–5× higher than GPT-4o completion rates |
| text-embedding-3-small (Step 2.7 anomaly detection) | USD 0.00002 / 1,000 tokens |
| Frontier model — small/reasoning variant (Tier 0 EvalOps, Step 1.2 — if available; otherwise a lower-cost variant) | TBC: confirm with Azure |

> **Note on frontier model pricing**: As of mid-2026, frontier models on Azure above GPT-4o (such as GPT-5 or equivalent) do not have stable, publicly listed PTU pricing. Actual rates must be confirmed with the Microsoft/Azure account team under the organisation's GCC 2.0 commercial agreement. The cost model variables (`FRONTIER_MODEL_PROMPT_PRICE_PER_1K`, `FRONTIER_MODEL_COMPLETION_PRICE_PER_1K`) in the `finops-cost-aggregator` job should be stored in Azure App Configuration and updated when pricing is confirmed, rather than being hardcoded. While frontier models carry higher per-token costs, stronger reasoning capability typically results in fewer tool-call loops and hallucination retries per email — partially offsetting the cost increase. The actual net per-email cost must be baselined empirically from SIT and the first 30 days of production data.

**Cost record schema** (`AIBPEmailCosts` Log Analytics custom table):

| Column | Description |
|---|---|
| `TimeGenerated` | Azure Monitor standard timestamp |
| `MessageId` | Root correlation ID of the email event |
| `AgentName` | e.g., `refund-agent` |
| `AgentSemver` | e.g., `1.3.0` |
| `SopId` | e.g., `SOP-REFUND-001` |
| `ResolutionType` | `automated` / `hitl` / `failed` |
| `PromptTokens` | Total prompt tokens across all LLM calls |
| `CompletionTokens` | Total completion tokens across all LLM calls |
| `TotalLLMCostUSD` | Computed token cost in USD |
| `ComputeCostUSD` | ACA compute cost in USD |
| `TotalCostUSD` | Sum of LLM + compute |
| `TotalCostSGD` | SGD equivalent |
| `IsAnomalousCost` | Boolean flag: true if `TotalCostSGD` > per-session guardrail threshold |

---

#### Per-Session Cost Guardrails

The per-session guardrail is the critical mechanism against runaway costs. An agent that enters an infinite reasoning loop or repeatedly retries failing tool calls will accumulate token cost linearly until something stops it.

**Guardrail design**:

A **token budget** is enforced at the agent level using an LangGraph graph node check. Before each LLM call, the agent checks its cumulative token spend for the current email session against the per-session budget stored in Azure App Configuration:

```
Azure App Configuration key: agents:{agentName}:costGuardrail.maxTokensPerSession
Default value: 8,000 tokens (configurable per agent)
```

If the cumulative token count exceeds the budget:
1. The agent stops making further LLM calls
2. The current session is flagged as `BUDGET_EXCEEDED` in the OTel trace
3. The email is routed to the Human Officer Queue (Step 4.1 fallback chain)
4. A `ao.email.budget.exceeded` metric event is emitted to Azure Monitor

**Budget threshold rationale**: At ~3,000 tokens/email average, a 8,000-token per-session budget allows 2.7× the average — sufficient for genuinely complex cases — while capping the worst-case expenditure per email at approximately SGD 0.25 (well above the target of SGD 0.20/resolution but providing headroom for legitimate complexity). The threshold is configurable per agent and per SOP category via App Configuration.

**Daily and monthly budget alerts**:

Azure Cost Management budget alerts are configured at three levels:

| Alert type | Threshold | Action |
|---|---|---|
| Azure OpenAI daily spend alert | 120% of expected daily budget (based on monthly budget ÷ 30) | Notify Ops + AIGP team lead |
| Azure OpenAI monthly budget alert (forecast) | Forecasted monthly spend exceeds 90% of monthly budget | Notify Ops + finance sponsor |
| Azure OpenAI monthly budget alert (actual) | Actual spend exceeds 100% of monthly budget | Emergency notify: Ops lead + CIO-level sponsor; consider reducing per-session token budget |

**Estimated monthly Azure OpenAI cost** (indicative — pending confirmed frontier model pricing):

| Component | Monthly estimate |
|---|---|
| Frontier model (production, <1,000 emails/day × estimated ~3,000 tokens avg × 31 days) | TBC: token volume similar to GPT-4o estimate; per-token cost 3–5× higher pending PTU pricing confirmation. Empirical baselining in SIT required before production budget is set. |
| Frontier model — small variant (Tier 0 critic, low-confidence subset) | TBC: confirm with Azure |
| text-embedding-3-small (anomaly detection) | ~USD 2/month |
| Frontier model — judge eval (CI pipeline eval runs, per-deploy) | ~USD 3–10 per deploy depending on model pricing; ~USD 30–100/month estimated |
| **Total estimated Azure OpenAI monthly cost** | **TBC: pending confirmed frontier model pricing. For planning purposes, assume 3–5× the equivalent GPT-4o estimate (~USD 1,400/month). Validate against actual SIT token consumption data before committing to a production budget.** |

> These are indicative estimates. The FinOps job should be used to establish actual per-email costs within the first 30 days of production (and during SIT) and to validate or revise the budget.

---

### Option 2: Azure Cost Management Aggregate Monitoring Only (No Per-Session Attribution)

**Implementation**: Monitor total Azure OpenAI spend via Azure Cost Management cost analysis. No per-email attribution; alerts fire when the monthly total approaches budget.

**Pros**: Zero implementation effort; Azure Cost Management is already available on the subscription

**Cons**:
- Cost visibility is at the subscription or resource level, not at the email/agent level. You cannot identify which specific agent or SOP category is the expensive one.
- No per-session guardrail; a single stuck agent running an infinite loop is invisible until the monthly aggregate shows an anomaly
- Provides no input to the cost comparison model (Step 5.2) — you cannot compute cost-per-resolution without per-email attribution

---

### Option 3: OpenCost on AKS (Kubernetes-Level Cost Attribution)

**Implementation**: Deploy OpenCost (open-source Kubernetes cost attribution tool) on the AKS cluster that hosts the Kafka microservices and AIGP components. Extend cost attribution to include AKS pod compute costs.

**Pros**: Provides Kubernetes-level compute cost attribution for the AKS components (AIGP, Kafka consumers)

**Cons**:
- OpenCost attributes Kubernetes compute cost but does not natively integrate with Azure OpenAI token usage — the core cost driver for AIBP. A hybrid implementation (OpenCost for AKS compute + OTel metrics for tokens) would still be required.
- The AKS components (AIGP, Kafka) are not the primary cost driver for AIBP; Azure OpenAI token spend dominates. OpenCost adds infrastructure complexity for a secondary cost category.
- OpenCost on AKS is owned and operated by the Platform & Infra team; the Ops function's primary cost concern is Azure OpenAI, which is better served by the OTel-based attribution in Option 1.

---

### Recommendation Justification

**Option 1** is recommended. Per-email cost attribution via OTel custom metrics is the foundation of FinOps for this platform because token spend varies so dramatically per email — aggregate monitoring (Option 2) is blind to the outliers that cause the most financial risk. The per-session token budget guardrail is the single most important cost control in an agentic platform: it converts the abstract budget risk of "what if an agent loops?" into a concrete, automatically enforced ceiling.

---

## 5.2 Cost Comparison Model — Agent vs. Human Officer

### Implementation Overview

The AIBP platform must justify its operational cost relative to the alternative: dedicated tax officers processing emails manually. The cost comparison model quantifies the economic case for the platform and provides management with the break-even analysis needed to evaluate the platform's Return on Investment (ROI) and to determine the optimal Automation Rate (the proportion of emails resolved by agents without human intervention).

This model is not purely operational — it requires input from HR (officer salary data), finance (overhead rates), and management (target automation rate). Ops owns the model structure and the agent-side cost data; the business owner provides the human-side inputs.

---

### Option 1 (Recommended): Azure Monitor Workbook Cost Comparison Dashboard + Monthly Logic Apps Report

**Technology Stack**: Log Analytics (`AIBPEmailCosts` custom table from Step 5.1), Azure Monitor Workbook, Azure Logic Apps (monthly report generation), Azure Blob Storage (report archive)

---

#### Cost Model Framework

**Human Officer Cost Per Email**:

| Component | Calculation | Notes |
|---|---|---|
| Officer annual salary | SGD A | To be provided by HR |
| Employer overhead (CPF, benefits, office space) | SGD A × overhead_rate (estimated 30–40% for government) | |
| Total annual officer cost | SGD A × (1 + overhead_rate) = SGD B | |
| Officer working hours/year | 52 weeks × 5 days × 7.5 hours − leave = ~1,700 hrs | Adjust for actual leave/training entitlement |
| Emails processed per officer per day (manual) | Estimate: 60–80 emails/day for routine cases; 20–30 for complex | To be validated with business owner |
| Officer emails per year | 70 emails/day × 250 working days = **17,500 emails/year** (illustrative) | |
| **Human cost per email** | SGD B ÷ 17,500 | |

Using illustrative values (SGD 80,000 salary + 35% overhead = SGD 108,000/year; 17,500 emails/year):
> **Human officer cost per email ≈ SGD 108,000 ÷ 17,500 = SGD 6.17/email**

**Agent Cost Per Email** (from Step 5.1 FinOps attribution):

| Component | Value |
|---|---|
| Frontier model token cost per email | TBC: indicatively 3–5× higher than GPT-4o equivalent (~SGD 0.21–0.70/email before efficiency gains); to be baselined from first 30 days of production data |
| ACA compute cost per email | ~SGD 0.01–0.02 |
| Allocated infrastructure cost per email | ~SGD 0.02 (Service Bus, Cosmos DB, Langfuse, support services, amortised monthly) |
| **Agent total cost per automated resolution** | **TBC: estimated SGD 0.25–0.75/email at frontier model pricing; validate from production data. Even at the upper estimate, the agent-vs-human cost advantage remains very large (see Step 5.2).** |

> These are illustrative estimates. The FinOps job in Step 5.1 will produce the actual agent cost per email from the first 30 days of production data. The model below uses the actual figure once available; `cost.total.sgd` from the `AIBPEmailCosts` table is the source.

**HITL-Assisted Resolution Cost**:

When an email requires HITL, the cost is a blend of agent cost and a fraction of officer time:

| Component | Calculation |
|---|---|
| Agent cost incurred before HITL trigger | ~SGD 0.07 (agent ran partial workflow before escalating) |
| HITL officer review time | Mean HITL review: 5–10 minutes |
| Officer HITL time cost | (8 minutes ÷ 450 minutes/working day) × SGD 108,000/year ÷ 250 days = ~SGD 0.77 per HITL review |
| **HITL-assisted resolution total** | **~SGD 0.07 + SGD 0.77 = SGD 0.84/email** |

**Break-Even Automation Rate**:

Given the agent cost, human cost, and expected HITL rate, the **break-even automation rate** is the proportion of emails that must be resolved by agents (without HITL) for the platform to cost less than full manual processing:

Let:
- `R` = automation rate (fraction 0–1)
- `H` = HITL rate (fraction of total emails that require HITL)
- `C_agent` = agent cost per automated resolution (SGD 0.15 illustrative)
- `C_hitl` = HITL-assisted resolution cost (SGD 0.84 illustrative)
- `C_manual` = full manual resolution cost (SGD 6.17 illustrative)
- `F` = fallback to manual queue rate (fully manual resolutions still required)

Platform cost per email = `R × C_agent + H × C_hitl + F × C_manual`

Break-even condition: Platform cost per email < `C_manual` for all emails:
```
R × 0.15 + H × 0.84 + F × 6.17 < 6.17
```

At an assumed HITL rate of 10% and fallback rate of 2%:
```
R × 0.15 + 0.10 × 0.84 + 0.02 × 6.17 < 6.17
R × 0.15 + 0.084 + 0.123 < 6.17
R × 0.15 < 5.963
R > 0  (the platform is profitable at any non-zero automation rate at these cost levels)
```

The agent cost advantage is so large (SGD 0.15 vs SGD 6.17, a 41× cost ratio) that the platform achieves positive ROI at almost any realistic automation rate. The material risk is not automation rate — it is **if per-session costs spiral** (looping agents or unexpectedly complex cases) or if the **HITL rate is far higher than projected** (e.g., 70% HITL rate due to over-conservative AIGP thresholds or poor agent quality).

**Revised cost model under adverse scenarios**:

| Scenario | Automation rate | HITL rate | Platform cost/email | Savings vs. manual |
|---|---|---|---|---|
| Optimistic | 90% | 8% | SGD 0.27 | 96% cost saving |
| Baseline | 80% | 15% | SGD 0.37 | 94% cost saving |
| Conservative | 60% | 30% | SGD 0.55 | 91% cost saving |
| Adverse | 40% | 50% | SGD 0.96 | 84% cost saving |
| Break-even | ~1% | 0% | SGD 0.15 | Any positive rate achieves savings |

Even under the most adverse scenario modelled, the platform generates approx. 84% cost savings per email processed automatically. The economic case is robust.

**Annual cost savings estimate** (<1,000 emails/day, baseline scenario):

| Metric | Value |
|---|---|
| Emails per year | 1,000/day × 250 working days = 250,000 |
| Full manual processing cost | 250,000 × SGD 6.17 = **SGD 1.54M/year** |
| Platform cost (baseline scenario, SGD 0.37/email) | 250,000 × SGD 0.37 = **SGD 93K/year** |
| **Annual savings** | **SGD 1.54M − SGD 0.093M ≈ SGD 1.45M/year** |
| Platform annual operating cost (infra + ops headcount) | ~SGD 200K/year (estimated; validate with finance) |
| **Net annual savings** | **~SGD 1.25M/year** |

> These figures are illustrative and dependent on the officer cost inputs provided by HR/finance. The Azure Monitor Workbook dashboard allows the human-side inputs (officer salary, overhead rate, emails/officer/day) to be entered as Workbook parameters, enabling management to run their own scenarios.

---

#### Azure Monitor Workbook — Cost Comparison Dashboard

The **FinOps Cost Dashboard** (`aibp-finops`) Azure Monitor Workbook contains:

**Tab 1 — Daily Cost Tracker**:
- Daily token spend vs. budget allocation (bar chart)
- Daily per-email cost (P50, P95, max — to detect outliers)
- Daily count of emails that breached the per-session token budget guardrail
- Top 5 most expensive emails of the day (by `TotalCostSGD`, no PII — identified only by `MessageId` and `SopId`)

**Tab 2 — Agent Cost Breakdown**:
- Cost per agent (bar chart, MTD): which agents are the most expensive to run?
- Cost per SOP category: which SOP categories are the most expensive to resolve?
- Cost per resolution type: `automated` vs. `hitl` vs. `failed`
- Token spend breakdown: prompt vs. completion (high completion ratios may indicate verbose or looping reasoning)

**Tab 3 — ROI Calculator** (interactive):
- Text input tiles: `Officer annual salary (SGD)`, `Overhead rate (%)`, `Emails per officer per day`
- Computed outputs: Human cost/email, Agent cost/email (live from Log Analytics), Savings per email, Break-even automation rate, Annualised savings at current automation rate
- This tab is designed to be screenshotted for management reporting

**Tab 4 — FinOps Anomaly Tracker**:
- Emails with `IsAnomalousCost = true` — near the per-session guardrail
- Daily count of budget-exceeded events
- Trend in mean per-email cost vs. rolling 30-day baseline (detect systematic cost increases indicating agent behaviour changes)

---

### Option 2: Monthly Spreadsheet Cost Report

**Implementation**: Ops team exports token usage data from Azure OpenAI usage reports and the Log Analytics `AIBPEmailCosts` table monthly, populates a spreadsheet with the cost model, and distributes to management.

**Pros**: No Workbook development effort; familiar format for management

**Cons**:
- No real-time visibility — cost anomalies (looping agents, budget exceedance) detected at month-end
- Manual export is error-prone and burdensome; the cost model has many inputs that change weekly
- Does not support the interactive ROI calculator needed for management scenario exploration

---

### Recommendation Justification

**Option 1** is recommended. The Azure Monitor Workbook FinOps dashboard provides real-time cost visibility rooted in the same Log Analytics data store that feeds all other operational dashboards — no additional data pipeline is required. The interactive ROI calculator is a specific management communication requirement: budget decision-makers need to be able to explore different automation rate scenarios themselves, which a Workbook parameter panel delivers without requiring a custom application.

> **Compliance Note (IM8)**: Cost records for government IT systems must be maintained as part of the system's financial governance records. The `AIBPEmailCosts` Log Analytics table and the monthly cost report archived to Blob Storage together constitute this record.
