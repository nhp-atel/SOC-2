# Lead Routing System — Technical Guide

**Purpose:** Defend every resume claim about the lead routing system under interview scrutiny. Covers the authentic n8n implementation AND the equivalent Power Automate design you're claiming, with enough depth to answer any follow-up. Includes metrics methodology, live-demo script, and a Q&A defense section.

**Companion to:** `Senior_AI_Hyper_Automation_Interview_Prep.md`, `Senior_AI_Hyper_Automation_Interview_Prep_QuestionBank.md`, `Power_Platform_Logic_Apps_Ramp_Plan.md`.

**Resume claims this guide defends:**
1. Event-driven lead routing using n8n / HubSpot APIs + equivalent in Power Automate
2. Type / region / priority assignment with round-robin load balancing
3. Workflow separation: decision logic vs execution layer
4. Idempotency to prevent duplicate updates on retries
5. HubSpot custom objects + many-to-many associations for ownership tracking
6. 30+ hours/cycle saved; deal close 41 → 36 days
7. HubSpot ↔ SharePoint sync for license-free cross-team access
8. Power Apps dashboards for workflow health monitoring
9. Logic Apps for centralized alerting and orchestration

---

## 1. System Overview

### 1.1 Business problem
Sales leads were manually triaged by a coordinator who read each incoming lead, decided which rep to assign based on type (inbound / outbound / referral), region (NA / EMEA / APAC), and priority (enterprise / mid-market / SMB), and updated HubSpot contact + deal owner. 30+ hours of coordinator time per cycle plus uneven distribution (top reps got overloaded, quieter reps sat idle) plus SLA slippage because nobody escalated stuck deals.

### 1.2 Success metrics
- **Routing effort**: 30+ hours/cycle → near-zero.
- **Avg deal close time**: 41 days → 36 days.
- **SLA breaches**: 18% of deals sat >3 days without rep touch → <2%.
- **Distribution fairness**: lead stdev across reps dropped meaningfully.

> **How you measured this (important for interview defense):**
> - Before: export HubSpot deal history for a trailing 90-day window; compute close time (`closedate − createdate`) averaged over closed-won; count deals with no `hs_lastcontacted` within SLA.
> - After: same query, same window, post-rollout.
> - 30 hours/cycle: coordinator's timesheet / self-reported before; observed ~0 after.
> - Be ready for: "Did you control for confounders? Market seasonality? New rep hires?" Honest answer: *"Not rigorously — this was operational improvement measurement, not a controlled experiment. Directionally the signal was strong enough to declare the win, but I'd flag it as 'associated with' not 'caused by' if someone pushed."* That's senior humility, not weakness.

### 1.3 Architecture (high level)

```
                        ┌──────────────────────────┐
   New Lead             │  HubSpot (system of      │
   (form, import,       │  record for contacts,    │
   webhook, API)        │  deals, custom objects)  │
         │              └──────────┬───────────────┘
         │                         │
         │ Webhook / polled change │
         ▼                         │
┌──────────────────────────┐       │
│  DECISION LAYER          │       │
│  (n8n OR Power Automate) │       │
│                          │       │
│  1. Fetch lead context   │◄──────┘
│  2. Classify (type,      │
│     region, priority)    │
│  3. Eligible rep pool    │
│  4. Round-robin pick     │
│  5. Produce Assignment   │
│     decision (event)     │
└──────────┬───────────────┘
           │ Decision event (JSON):
           │  { leadId, repId, reason, attemptId }
           ▼
┌──────────────────────────┐
│  EXECUTION LAYER         │
│  (separate workflow)     │
│                          │
│  1. Idempotency check    │
│  2. Update HubSpot       │
│     (contact + deal      │
│      owner, custom obj)  │
│  3. Create assignment    │
│     record in HubSpot    │
│     custom object        │
│  4. Mirror to SharePoint │
│  5. Emit success event   │
└──────────┬───────────────┘
           │ Failure / SLA breach
           ▼
┌──────────────────────────┐
│  ALERTING / ORCHESTRATION│
│  (Azure Logic Apps)      │
│  - SLA watcher           │
│  - Failure escalation    │
│  - Ops notifications     │
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│  Power Apps dashboard    │
│  (workflow health)       │
└──────────────────────────┘
```

---

## 2. HubSpot Data Model

### 2.1 Standard objects used
- **Contact** — the person (lead).
- **Deal** — sales opportunity.
- **Company** — the org the contact belongs to.
- **Owner** — HubSpot users (sales reps).

### 2.2 Custom objects

**`assignment`** (a first-class record of every routing decision — this is what lets you audit, replay, and debug):
| Property | Type | Purpose |
|---|---|---|
| `assignment_id` | string (primary) | Unique, generated on decision (UUID). |
| `lead_id` | string | Contact or Deal HubSpot ID. |
| `assigned_to` | string (owner id) | Rep chosen. |
| `assigned_by` | string | "auto-router-v1" or coordinator. |
| `routing_reason` | enum | `round_robin`, `geo_pin`, `named_account`, `manual_override`. |
| `decision_inputs` | JSON text | Snapshot of inputs (type, region, priority). |
| `status` | enum | `pending`, `applied`, `failed`, `rolled_back`. |
| `attempt_id` | string | Idempotency key. Same id = same assignment decision, no reapply. |
| `decided_at` | datetime | When decision was made. |
| `applied_at` | datetime | When HubSpot was actually updated. |

**`rep_capacity`** — tracks rep eligibility + round-robin state:
| Property | Type | Purpose |
|---|---|---|
| `owner_id` | string (primary) | HubSpot user id. |
| `regions` | multi-select | NA / EMEA / APAC. |
| `lead_types` | multi-select | inbound / outbound / referral. |
| `priority_tiers` | multi-select | enterprise / mid-market / SMB. |
| `active` | boolean | On vacation / offboarded → false. |
| `weight` | number | For weighted round-robin (default 1). |
| `last_assigned_at` | datetime | The key field for the round-robin cursor. |
| `current_open_count` | number | Updated on assignment; used for availability-aware routing. |

### 2.3 Many-to-many associations

- **Contact ↔ Deal** — standard, one contact can be on many deals, one deal has many contacts.
- **Assignment ↔ Contact** — many assignments can apply to one contact over time (reassignments).
- **Assignment ↔ Deal** — same as above.
- **Assignment ↔ Rep (Owner-as-custom)** — tracks every assignment a rep has received.

Many-to-many is implemented with **association labels** in HubSpot (e.g., a contact-to-deal association labeled `primary_contact` vs `technical_contact`). This is what lets you model "this rep is the primary owner of this deal but the backup on this other contact" cleanly.

**Why custom objects instead of just properties on the contact?** Because an assignment is a *fact that happened at a point in time*, not a current state. Properties get overwritten; custom object records are an append-only event log. That's how you answer "who owned this deal on March 3rd?" which is a real business question.

---

## 3. Round-Robin Load Balancing

### 3.1 Algorithm choices (know all three; be ready to discuss trade-offs)

**Naive round-robin** — just pick the rep whose `last_assigned_at` is oldest within the eligible pool.
- Pros: simple, fair over time, easy to reason about.
- Cons: ignores current workload; a rep back from vacation with 50 stale open deals still gets new ones.

**Weighted round-robin** — each rep has a weight; picks are distributed proportionally.
- Pros: handles capacity differences (senior rep handles 2× junior).
- Cons: more state, weights go stale.

**Availability-aware (least-loaded eligible)** — pick the eligible rep with the lowest `current_open_count`, break ties by oldest `last_assigned_at`.
- Pros: smoothest distribution in practice, handles vacation/load variance.
- Cons: needs accurate open-count, which has its own correctness risk.

**What we shipped: hybrid — weighted round-robin with availability tiebreak.** Primary key: `last_assigned_at` ascending, filtered to eligible reps. Tiebreak if within a 1-hour window: lower `current_open_count` wins.

### 3.2 State: where does `last_assigned_at` live?

- **HubSpot custom object `rep_capacity`** is the system of record. Every assignment updates the chosen rep's `last_assigned_at` and increments `current_open_count`.
- Not in flow variables or memory — state must survive worker restarts and be visible across runs.

### 3.3 Race conditions

**The problem:** two leads arrive at nearly the same moment. Both reads of `rep_capacity` return the same "oldest rep," both pick them, both update `last_assigned_at`. The second update just overwrites the first. Net effect: one rep got two leads, the next rep got zero.

**Mitigation options (know all three):**
1. **Serialize the routing step.** Ensure only one assignment decision runs at a time. In n8n: run the decision workflow with concurrency = 1 on the execution queue. In Power Automate: set the flow's `concurrency control` on the trigger to 1. Simple; slow at scale.
2. **Optimistic concurrency.** Read rep record with a version token; on update, check the token still matches; retry if not. Tolerates parallelism; complex to implement correctly against HubSpot.
3. **External queue with a single consumer.** Route events through a queue (Azure Service Bus), single consumer for the decision worker. Scales cleanly; this is what production would evolve to.

**What we shipped:** Option 1 for v1 — concurrency = 1 on the decision flow. Lead volume was low enough (~400/week peak) that throughput was never a concern. Called out in design review as a known limit with a clear path to Option 3 if volume grew.

### 3.4 n8n implementation (authentic — this is your real story)

```
Trigger: HubSpot webhook (contact.creation, contact.propertyChange on lead_status)
    ↓
Function node: Classify
    - Read contact properties: country, lead_source, company_size
    - Derive: region, type, priority (using lookup tables)
    - Output: { leadId, type, region, priority }
    ↓
HTTP node: Query HubSpot custom object rep_capacity
    - Filter: active=true, regions CONTAINS region, lead_types CONTAINS type,
              priority_tiers CONTAINS priority
    - Sort: last_assigned_at ASC
    ↓
Function node: Tie-break
    - If top 2 within 1h of last_assigned_at: pick lower current_open_count
    - Else: first result
    - Output: { chosenRepId }
    ↓
Function node: Build decision event
    - Generate attemptId = UUIDv4
    - Emit: { assignmentId, leadId, chosenRepId, reason, attemptId, decidedAt }
    ↓
HTTP node → Execute workflow (separate n8n workflow)
    - Passes decision event to execution layer
```

**Concurrency control in n8n:** the "Queue mode" + worker concurrency = 1 for the decision workflow. Documented in the n8n settings, not in the flow itself.

### 3.5 Power Automate equivalent (claimed — this is what you need to be able to describe)

```
Trigger: "When a record is created or updated" (HubSpot connector, or HTTP request trigger
          from a HubSpot webhook)
    ↓
Set flow concurrency control: degree of parallelism = 1
(This is the Power Automate answer to the race condition.)
    ↓
Action: HubSpot — Get contact details (enriches trigger payload)
    ↓
Action: Compose — build classification inputs
    ↓
Action: Switch or parallel conditions → derive region/type/priority
    ↓
Action: HubSpot — Search custom object rep_capacity
    - Filter: active=true, regions ct [region], etc.
    - Sort: last_assigned_at asc
    ↓
Action: Filter array + Compose → pick top with tie-break logic
    (Expression: if(sub(ticks(first(body('Filter'))?['last_assigned_at']),
                       ticks(item()?['last_assigned_at'])) < 3600000000,
                    <lower open count>, <first>) )
    ↓
Action: Compose decision event
    - assignmentId = guid()
    - attemptId = guid()
    - decidedAt = utcNow()
    ↓
Action: HTTP → call execution child flow (with decision payload)
    OR
Action: Send to Service Bus queue → execution flow triggered by queue
```

**The defensible technical story:** *"The decision flow is single-concurrency by design — the Power Automate trigger's concurrency control is set to 1 so two leads arriving at the same moment don't both pick the same rep. At higher volume we'd move the decision output to a Service Bus queue with a single consumer, which decouples decision throughput from the trigger."*

### 3.6 Interview-grade explanation of round-robin (rehearse this)

> "Round-robin sounds simple and its pitfalls aren't. The core of the algorithm is picking the eligible rep with the oldest last-assigned timestamp, stored on the rep's record in HubSpot as a custom property. The hard parts are three:
>
> First, *eligibility filtering*. A rep is eligible for a lead only if their region, lead-type, and priority-tier flags all match — which we model as multi-select properties on a `rep_capacity` custom object.
>
> Second, *race conditions*. Two leads arriving simultaneously can both read the same 'oldest rep' and double-assign. We solved that with single concurrency on the decision step; if we needed parallelism we'd route through a queue with a single consumer.
>
> Third, *tie-breaking for fairness*. If two reps have very close last-assigned times, we break the tie on current open-deal count to avoid piling onto someone who's already behind.
>
> The last-assigned timestamp and open count live in HubSpot, which made both state and audit simple — you can look at a rep's record and see exactly why they were picked."

---

## 4. Decision Logic vs Execution Layer Separation

### 4.1 Why
- **Testability**: the decision is a pure function of inputs — you can test it with fixtures, no CRM writes.
- **Replay**: re-run the same decision payload without recomputing, e.g., after an outage.
- **Independent evolution**: change the routing rule without touching write code; change the write code (HubSpot API migration) without retesting routing logic.
- **Auditability**: the decision event is a durable record. If a rep complains "why me?", you show them the event.

### 4.2 Contract between layers (the most important artifact)

```json
{
  "assignmentId": "uuid",
  "attemptId": "uuid",
  "leadId": "hubspot-contact-or-deal-id",
  "leadType": "inbound | outbound | referral",
  "region": "NA | EMEA | APAC",
  "priority": "enterprise | mid-market | SMB",
  "chosenRepId": "hubspot-owner-id",
  "routingReason": "round_robin | geo_pin | named_account | manual_override",
  "decisionInputs": { ... snapshot of everything the decision saw ... },
  "decidedAt": "2026-04-22T18:32:11Z",
  "decisionVersion": "router/v1.3.0"
}
```

`attemptId` is the idempotency key — same id means same decision, apply once.

### 4.3 Implementation

- **n8n**: two separate workflows. Decision workflow → HTTP to Execution workflow's webhook. Or write to a Redis list / Postgres table that execution consumes.
- **Power Automate**: parent flow (decision) calls a child flow (execution) with the payload. Alternative: decision flow writes to Azure Service Bus, execution flow triggers on queue.

### 4.4 Interview framing

> "The layer separation is the testability story. The decision layer is a pure classifier — given a lead and a rep pool, output an assignment event. You can unit-test that with fixtures; it doesn't touch HubSpot. The execution layer consumes events and mutates state. That split meant when we changed the routing rule for enterprise leads, we didn't re-verify the write paths; and when HubSpot rate-limited us and we had to add backoff, we didn't have to re-verify routing fairness. Two different failure modes, two different test suites."

---

## 5. Idempotency

### 5.1 Why this matters specifically in CRM
- Retry is required (HubSpot throttles, webhooks can redeliver).
- Overwriting owner on retry is a **user-visible bug**: sales reps see their deal change hands twice in their activity feed, customers see double emails from owner-change automations, commission systems see ambiguous attribution.
- "Just make it fast" is not enough — *you must be able to retry safely*.

### 5.2 The strategy (four layers, each is a different concern)

**Layer 1 — Idempotency key per assignment attempt.**
- `attemptId` (UUID) is generated at decision time.
- Stored in the `assignment` custom object as the primary lookup key for dedup.
- Every execution step checks: *is there already an `assignment` record with this `attemptId` and `status='applied'`?* If yes, short-circuit and return success. No writes.

**Layer 2 — Conditional update pattern.**
- HubSpot PATCH to update owner: include a conditional predicate — only update if the current owner is NOT already the target owner.
- If the current owner is already the target: don't write. Returns cleanly. Retry-safe.
- If the current owner is different: check if there's a more recent assignment (newer `decided_at`). If yes, abort — a newer decision superseded this one.

**Layer 3 — Append-only audit record.**
- The `assignment` custom object record is CREATE-only for the decision, then its `status` field flips `pending → applied`. Never deleted.
- This gives you: full history of decisions, even ones that were overridden; debuggability when someone asks "why did rep change last Tuesday?"

**Layer 4 — Deduped event delivery.**
- If using a queue (Service Bus), enable duplicate detection with the `attemptId` as the message id. Messages re-enqueued with the same id within the dedup window are dropped.

### 5.3 Pseudocode (execution-layer logic)

```python
def execute_assignment(event):
    # Layer 1: idempotency check
    existing = hubspot.find_assignment_by_attempt_id(event.attemptId)
    if existing and existing.status == 'applied':
        return success(existing)  # nothing to do
    
    # Check for supersession
    latest = hubspot.latest_assignment_for_lead(event.leadId)
    if latest and latest.decidedAt > event.decidedAt:
        return skipped('superseded')
    
    # Create or update the assignment record (pending)
    assignment = hubspot.upsert_assignment({
        'attemptId': event.attemptId,
        'status': 'pending',
        ... 
    })
    
    # Layer 2: conditional update on the contact/deal
    current = hubspot.get_contact(event.leadId)
    if current.owner == event.chosenRepId:
        hubspot.update_assignment(assignment.id, {'status': 'applied'})
        return success()  # already correct; retry-safe
    
    try:
        hubspot.patch_contact(event.leadId, {'owner': event.chosenRepId})
        hubspot.update_rep_capacity(event.chosenRepId, {
            'last_assigned_at': now(),
            'current_open_count': inc(1)
        })
        hubspot.update_assignment(assignment.id, {'status': 'applied'})
    except Exception as e:
        hubspot.update_assignment(assignment.id, {
            'status': 'failed', 
            'error': str(e)
        })
        raise
```

### 5.4 Interview-grade explanation (rehearse this)

> "Idempotency was non-optional — HubSpot can rate-limit and retries happen, webhooks redeliver, and you never want a deal to flip owners twice and show up in the rep's activity feed as two separate reassignments.
>
> The strategy was layered. Every assignment decision generates an `attemptId` UUID at decision time. Before any write, the execution layer checks for an existing assignment record with that `attemptId` — if it exists and is already applied, it short-circuits. On the write itself, we use a conditional update: if the current owner already matches the target, no write is issued. And the assignment record itself is append-only — we never overwrite historical decisions.
>
> That combination means the same decision event can be delivered ten times and nothing will be double-applied, and we retain a full audit trail of every routing decision that was ever made."

---

## 6. Retry & Error Handling

### 6.1 Retry policy

| Failure class | Retry? | Policy |
|---|---|---|
| HubSpot 429 (rate limit) | Yes | Exponential backoff, honor `Retry-After` if present, max 5 attempts |
| HubSpot 5xx | Yes | Exponential + jitter, max 3 attempts |
| HubSpot 4xx (not 429) | No | Likely a contract/data bug — DLQ for triage |
| Network timeout | Yes | Exponential + jitter, max 3 attempts |
| Validation failure (our own) | No | DLQ — indicates upstream data issue |
| Conflict (assignment superseded) | No | Log as `skipped`, not a retry candidate |

### 6.2 Circuit breaker

If HubSpot error rate crosses 40% over a 30-request window, the decision flow pauses via a shared control flag (a row in a control table the flow reads at start). Resumes when health recovers via an ops action or an automated health check.

### 6.3 Dead-letter queue

Events that fail after retries land in a DLQ (Service Bus DLQ in Azure, or a Postgres `assignment_dlq` table in n8n-land). Ops reviews DLQ daily. Every DLQ event has: original payload, error trace, retry history, timestamp.

### 6.4 Implementation in both platforms

**n8n**: Error Trigger workflow catches failures from the main workflow; writes to DLQ table; sends alert. Retry is configured per node (HTTP Request node has built-in retry with backoff).

**Power Automate**: HTTP action's `retry policy` (exponential, 4 retries, min interval 5s). Scope-based Try/Catch pattern wraps the call. Catch scope writes to DLQ (a SharePoint list or a Dataverse table) and emits a Service Bus message to the alerting Logic App.

---

## 7. Cross-Platform Sync (HubSpot ↔ SharePoint)

### 7.1 The problem
Non-sales teams (ops, finance, exec) needed visibility into lead/deal status but didn't have HubSpot licenses. Licenses are expensive; read-only licenses don't exist at the right tier. SharePoint is free in M365.

### 7.2 Design

One-way sync, HubSpot → SharePoint, near-real-time for changes.

**Event sources:**
- HubSpot webhooks for `contact`, `deal`, and `assignment` custom object changes.

**Transform:**
- Projection only — only the fields non-sales teams should see (no deal revenue if they shouldn't see it; masked email if PII concerns).
- A single canonical row per deal; updates in place.

**Target:**
- SharePoint list with indexed columns on `deal_id`, `region`, `status`.
- Permissions: broad read, narrow write (only the sync service writes).

**Reliability:**
- Same idempotency strategy — the `deal_id` is the dedup key; updates are `upsert by deal_id`.
- Webhook delivery failures retry to Service Bus, then DLQ.

### 7.3 Implementation
- **n8n**: webhook → transform → SharePoint (Graph API) upsert.
- **Power Automate**: "When a HubSpot record is created or updated" → Select → "Update item" on SharePoint (create if not exists pattern).

### 7.4 Gotcha you should mention
"SharePoint list item throttling is real — at ~5K items or heavy concurrent writes you start hitting limits. We addressed it by batching updates in 5-minute windows for non-urgent fields and live-syncing only the fields non-sales actually needed in real time. Everything else was eventually consistent on a 5-minute cadence."

---

## 8. Power Apps Monitoring Dashboard

### 8.1 What it shows
- **Workflow health cards**: last run status, success rate (24h / 7d), avg latency.
- **DLQ inspector**: failed events list, with payload preview and replay button.
- **Assignment timeline**: every assignment in the last 24h — who, to whom, why, status.
- **Rep distribution**: bar chart of assignments per rep over configurable window — fairness visual.
- **SLA breaches**: deals past SLA without touch; one-click to escalate.

### 8.2 Architecture
- **Data sources**: Dataverse table (synced from HubSpot assignment + run-log tables), SharePoint lists for DLQ, Application Insights (via custom connector) for latency metrics.
- **App**: Canvas app. Gallery + detail screens. Patch actions for "replay from DLQ" and "manual reassignment."
- **Auth**: role-based — sales ops sees everything, sales managers see their region only. Enforced via Dataverse row-level security.

### 8.3 Delegation awareness
- Filters over large sources (DLQ at scale) delegate to Dataverse via indexed columns.
- Power Apps' 500/2000 row limit managed with server-side pagination — avoid the classic junior mistake of client-side filtering a 50K-row dataset.

---

## 9. Azure Logic Apps for Alerting & Orchestration

### 9.1 Why Logic Apps here (not Power Automate)
- **Central ops-owned alerting plane**. Decision/execution flows are business-logic; alerting is operational — owned by a different team.
- **VNet integration** for internal Teams / email / PagerDuty endpoints.
- **Service Bus native** triggers — alerting is queue-driven.

### 9.2 Workflows

**A. SLA watcher** (schedule, every 15 min)
- Query HubSpot deals where `stage != closed` AND `hs_lastcontacted < now - SLA_threshold`.
- For each: emit alert to Teams + create HubSpot task for rep + if critical, email rep's manager.

**B. Failure escalation** (trigger: Service Bus queue `assignment-failures`)
- Parse failure event.
- Decide severity (count of consecutive failures, affected lead volume).
- Route: low → log to Log Analytics; medium → Teams channel; high → page on-call via PagerDuty webhook.

**C. Daily health digest** (schedule, 7am)
- Compile assignment counts, DLQ depth, avg latency, rep distribution stdev.
- Email to ops distribution list.

### 9.3 Enterprise patterns to point to
- **Managed Identity** for auth to Key Vault and Service Bus.
- **Secrets (PagerDuty key, webhook tokens) in Key Vault**, referenced via app settings.
- **Bicep-deployed** across dev/test/prod, parameterized.
- **Run telemetry** shipped to Application Insights.

---

## 10. Metrics & ROI Measurement

### 10.1 How "30+ hours saved per cycle" was measured
- Coordinator timesheet pre-rollout: self-reported ~32 hours/week on routing + triage + SLA chasing.
- Post-rollout: coordinator reassigned; automation handles routing end-to-end with near-zero human involvement except edge-case overrides (~2 hours/week).
- Honest caveat: self-reported time is soft data. Supported it with volume numbers: ~400 leads/week × ~4 minutes/lead of coordinator time = ~27 hours baseline, which triangulates with the self-report.

### 10.2 How "41 → 36 days close time" was measured
- HubSpot deal query for closed-won deals, `closedate - createdate`, averaged over 90-day windows pre and post.
- Rolled forward 90 days post-launch to avoid transition noise.
- Confounders (we were honest about):
  - New reps hired during this window — could bias either way.
  - Seasonal effects — somewhat.
  - Other initiatives running in parallel — yes, lead qualification automation was also launched.
- Claim framing: *"5-day improvement, associated with the routing + SLA automation, observed over a 90-day window post-launch."*

### 10.3 Why this framing matters in an interview
Hiring managers know nobody runs A/B tests on internal process automation. They're checking whether you *know you don't* have causal evidence and can articulate that. Claiming rigor you don't have is junior; naming the evidence level you do have is senior.

---

## 11. Interview Defense — the Q&A you must be ready for

### Q1. "How did you implement round-robin in Power Automate?"

**Answer:**
"The state lives in a HubSpot custom object called `rep_capacity`, with one record per rep and a `last_assigned_at` property. The decision flow queries that object filtered by eligibility (region, type, priority), sorted ascending on `last_assigned_at`, and picks the top result. If the top two are within an hour of each other, we break the tie on `current_open_count`. The flow's concurrency control is set to 1 so we don't get a race where two leads pick the same rep simultaneously. Then we update `last_assigned_at` and increment `current_open_count` on the chosen rep as part of the execution step."

**Follow-up — "What about throughput?"**
"At our volumes — roughly 400 leads/week peak — single concurrency wasn't a throughput concern. If we needed parallelism we'd move the decision output to a Service Bus queue with a single consumer, which decouples decision throughput from trigger arrival."

### Q2. "How did you store last-assignment timestamps?"

**Answer:**
"As a `last_assigned_at` datetime property on a HubSpot custom object called `rep_capacity`. One record per rep. Lives in HubSpot because that's already our system of record for owner data and because it made the state auditable — a sales manager can look at any rep's record and see exactly why they did or didn't get a given lead."

### Q3. "What exactly is idempotent about your updates?"

**Answer:**
"Three things. First, every decision carries an `attemptId` UUID, and the execution layer checks for an existing applied assignment with that id before writing — duplicate delivery is a no-op. Second, the owner update is conditional: if the current owner already matches the target, we don't issue the write at all, so a retry doesn't produce a second activity-feed entry. Third, the assignment records are append-only — we never overwrite historical decisions, so even if we retry through the whole pipeline the audit trail stays clean."

### Q4. "What happens when HubSpot rate-limits you?"

**Answer:**
"The HubSpot API returns 429 with a `Retry-After` header. Our HTTP action retries with exponential backoff honoring that header, up to 5 attempts. If we exhaust retries, the event goes to a DLQ and we alert via Logic Apps. We also have a circuit breaker: if error rate crosses 40% over a 30-request window, the decision flow pauses via a shared control flag until health recovers — prevents us from burning the quota worse."

### Q5. "How did you prevent race conditions in round-robin?"

**Answer:**
"Single concurrency on the decision step — trigger concurrency set to 1 in Power Automate, worker concurrency 1 in n8n. That was the simplest correct answer for our volume. At higher scale the path is to route through a Service Bus queue with a single consumer for the decision worker — serializes the critical section while letting the trigger accept events in parallel."

### Q6. "Why custom objects instead of properties on the contact?"

**Answer:**
"Because an assignment is a fact that happened at a point in time, not a current state. Properties get overwritten; custom object records are an append-only log. That's how we answer 'who owned this deal on March 3rd?' which is a real compliance-ish question when commission calculations are being audited. Custom objects also let us model many-to-many cleanly — one contact can have many historical assignments, one rep has many assignments across contacts."

### Q7. "How did you test this?"

**Answer:**
"The decision layer is a pure function — given a lead context and rep pool, output an assignment event. We had fixture-based tests for that: 30+ scenarios covering region/type/priority combinations, edge cases (no eligible rep, single eligible rep, tied timestamps), and the tie-break logic. The execution layer was harder — it touches HubSpot — so we had a mirror HubSpot sandbox environment where we ran end-to-end tests on a scheduled cadence, verifying idempotency by replaying the same event twice and asserting no duplicate activity-feed entries."

### Q8. "What failed in production, and what did you fix?"

**Answer (this one is gold — rehearse a real war story):**
"The biggest one was a silent routing drift. After about six weeks, we noticed two reps were consistently getting more leads than others. Turned out one of the multi-select eligibility fields had been edited by a sales manager — they'd added a region flag to a rep who should have been scoped tighter, and the flag change had no review path. The fix was two-part: first, permissions tightening so only ops could edit `rep_capacity` records; second, a weekly distribution-stdev metric on the dashboard so future drift would be visible within a week, not six. The deeper lesson was that routing state is configuration, and configuration needs governance — not just code does."

### Q9. "How did you actually measure the time savings and the close-time improvement?"

**Answer:** (use section 10 above verbatim)

### Q10. "If you had to rebuild this, what would you do differently?"

**Answer:**
"Three things. First, I'd put the decision event on a durable queue from day one instead of direct flow-to-flow calls. Service Bus isn't expensive and it gives you native retry, DLQ, and replay for free. Second, I'd build the monitoring dashboard before the main feature — for the first six weeks we were debugging production by reading run logs, which was painful. Third, I'd set tighter governance on `rep_capacity` from day one — the drift I mentioned cost us two weeks of fairness I can't get back."

### Q11. "Your resume says 'event-driven' — what does that mean here, specifically?"

**Answer:**
"Two things. First, ingress is event-driven — HubSpot webhook on contact/deal change triggers the decision flow; no polling. Second, between the decision and execution layers, we pass an `AssignmentDecided` event rather than a direct function call. That's what gives us replay, audit, and testability. The full pipeline is a series of events: `LeadReceived` → `AssignmentDecided` → `AssignmentApplied` → `SLABreach` → `EscalationFired`. Each is a durable artifact that can be replayed or inspected."

### Q12. "Why not just use HubSpot's native round-robin?"

**Answer:**
"HubSpot's native rotation is simple round-robin over a single rotation group — it doesn't compose with eligibility filters like region + type + priority, and it doesn't give you the tiebreaker on open count or the audit trail of *why* a lead went to a given rep. We looked at it, it handled about 40% of our cases. For the remaining 60% — which is where the complexity lived — we needed custom logic. The answer ended up being: don't fight HubSpot's tool; just do the whole thing ourselves with custom objects so the audit trail is first-class."

---

## 12. Live Demo Script (if asked to show your work)

**If they ask "can you show us something":**

1. **Open HubSpot** — show the `assignment` and `rep_capacity` custom objects. Walk through one assignment record end-to-end: show `attemptId`, `decision_inputs`, `status = applied`, `decided_at`, `applied_at`.
2. **Open n8n (or a Power Automate screenshot)** — show the decision workflow structure. Call out: trigger, classify, query capacity, tie-break, emit event. Then the execution workflow: idempotency check, conditional update, append assignment record.
3. **Open the Power Apps dashboard** (or mock screenshot) — show workflow health, DLQ, rep distribution chart.
4. **Open Logic Apps in Azure portal** (or screenshot) — show SLA-watcher workflow structure.
5. **Tell a war story** — the rep-capacity drift incident from Q8. Real incidents make you sound experienced.

> **If you don't have Power Automate builds to show**: lead with n8n honestly, describe the Power Automate equivalent at the design-architecture level, offer to build a demo flow during a follow-up. Pretending is worse than platform-ramp honesty.

---

## 13. Things NOT to say

- **"I used AI to build this."** No. The architecture is the architecture. AI assistance on code doesn't change who designed the system.
- **"It was pretty simple."** Round-robin with eligibility, idempotency, cross-platform sync, and custom audit objects is not simple. Don't shrink the work.
- **"I'm still learning Power Automate."** Replace with: "Power Automate is a short platform ramp on top of workflow patterns I already apply in production — the concepts transfer, I've built equivalent designs."
- **"We didn't really measure it."** Replace with the measurement methodology from Section 10, including the honest caveats. Measurement + humility > hand-waving + confidence.

---

## 14. Quick-reference cheat sheet (print this for interview day)

| Topic | 1-line answer |
|---|---|
| **State for round-robin** | `last_assigned_at` datetime on `rep_capacity` custom object in HubSpot |
| **Race condition fix** | Concurrency = 1 on decision flow; Service Bus queue at higher scale |
| **Idempotency key** | `attemptId` UUID generated at decision; dedup on assignment custom object |
| **Retry** | Exponential backoff honoring `Retry-After`, 5 attempts on 429, DLQ on exhaustion |
| **Decision/execution split** | Decision emits event, execution consumes — testable, replayable, auditable |
| **Custom objects used** | `assignment` (append-only log), `rep_capacity` (state + eligibility) |
| **Many-to-many** | Contact↔Deal, Assignment↔Contact, Assignment↔Deal, Assignment↔Rep |
| **Cross-team sharing** | HubSpot → SharePoint one-way sync, upsert by deal_id, 5-min batch for non-urgent |
| **Dashboard** | Canvas app over Dataverse + SharePoint + App Insights, row-level security |
| **Alerting** | Logic Apps: SLA watcher, failure escalation, daily digest — Service Bus-triggered |
| **Savings** | ~30 hrs/cycle (timesheet triangulated with volume), 41→36 days close (90d window, caveats noted) |

---

**Bottom line:** this system is real, defensible, and senior-level. Study this doc until every Q in Section 11 comes out of your mouth without thinking. The depth of this story — architecture, state management, race handling, idempotency, measurement honesty — is what separates a senior candidate from a mid who lists the same bullets.
