# Lead Routing System — Project Walkthrough Script for the Technical Round

**Purpose:** the single rehearsable script for "walk me through the most complex thing you've built." The technical round will almost certainly open with this question or one like it. The next 5–7 minutes after that question are the highest-leverage minutes of the entire interview. This document gives you tiered versions, exact phrasing for AI labeling, and recovery moves for predictable interruptions.

**Companion to:** `Senior_AI_Hyper_Automation_Interview_Prep.md`, `Lead_Routing_System_Technical_Guide.md`, `Senior_AI_Hyper_Automation_Interview_Prep_QuestionBank.md`.

---

## 0. The shape of a senior project walkthrough

A senior walkthrough is **layered, not linear**. You start at the top and go deeper only when invited. Three layers:

1. **Layer 1 (60–90 seconds)** — the executive summary. Problem, architecture shape, AI placement, outcome. Then *pause* and ask "do you want me to go deeper on any part?"
2. **Layer 2 (3–4 minutes)** — the structured design walkthrough. Triggered by "tell me more" or "walk me through the design."
3. **Layer 3 (deep dive on demand)** — section-by-section detail. Triggered by specific questions.

Most candidates dump Layer 3 immediately. The interviewer either zones out or interrupts. **Senior pacing is: short, then deeper on invitation.**

---

## 1. Layer 1 — the 60–90 second version (rehearse this verbatim)

Say this out loud. Set a timer. Aim for 75 seconds.

> "I built an event-driven lead routing system at my real-estate client on top of HubSpot. New leads come in via webhook; the system classifies priority based on the call-to-action — schedule tour, third-party registration, interest list — assigns the lead to an eligible online sales counselor using round-robin with an availability tiebreak, and applies the assignment idempotently to HubSpot's contact and custom-object records.
>
> The architecture deliberately separates decision logic from execution. The decision layer is a pure function we test with fixtures; the execution layer is what mutates state. That split made the system testable, replayable, and auditable.
>
> AI is used in two specific places: a fallback classifier for call-to-action strings the rules don't recognize, and intent extraction from free-text notes when they're present. Everything else — priority assignment, round-robin, idempotency, audit logging — is rule-based, because the inputs are structured and the decisions are verifiable.
>
> The system saved roughly 30 hours per cycle of manual triage and reduced average deal-close time from 41 to 36 days, observed over a 90-day window.
>
> Want me to go deeper on any part?"

That last sentence — *"Want me to go deeper on any part?"* — is critical. It hands control back to them. Senior candidates do this; juniors monologue.

### What this script does for you

- Names the **business problem** in one breath (lead routing).
- Names the **architecture pattern** (event-driven, decision/execution split).
- Pre-emptively answers the AI question (where, why, where not).
- Quantifies the **outcome** with caveat language ("observed over a 90-day window").
- Ends with a question — invites direction.

### What it deliberately leaves out

- Round-robin mechanics (drill-down on demand)
- Idempotency strategy (drill-down on demand)
- Specific platforms — n8n vs Power Automate (drill-down on demand; positioning intentional)
- AI specifics (drill-down on demand; you set the hook in the summary)

---

## 2. Layer 2 — the 3–4 minute structured walkthrough

When they say "tell me more" or "walk me through the design," deliver this. Roughly 4 minutes spoken.

### Beat 1 — the problem (30 seconds)

> "The client is a real-estate sales operation. Leads come from multiple channels — web forms, third-party platforms like BDX and Zonda, partner referrals, agent outreach. A coordinator was manually reading each lead, deciding which online sales counselor to assign based on the lead's neighborhood, the call-to-action, and rep availability, and updating HubSpot. That's about 30 hours a week of work, plus uneven distribution — top reps got overloaded, quieter reps sat idle — plus SLA slippage because nobody was actively chasing stuck deals."

### Beat 2 — the architecture (90 seconds)

> "The shape is event-driven and split into three logical layers.
>
> **Ingress** — HubSpot webhook fires on new lead creation, with the lead envelope including the sales lead, contact, neighborhood, and any related sales leads.
>
> **Decision layer** — a pure function. Takes the lead context, the eligible rep pool, and the routing rules; outputs an assignment event. Pure means it doesn't write to HubSpot, doesn't call external services, has no side effects. We can test it with fixtures, replay it, and unit-test every branch. Inside the decision layer is where priority classification happens — and this is where the AI/no-AI boundary lives, which I'll come back to.
>
> **Execution layer** — consumes the decision event and mutates state. Updates the contact's owner in HubSpot, creates an `assignment` custom-object record, updates the chosen rep's `last_assigned_at` timestamp, mirrors to a SharePoint list for non-licensed teams. Every step is idempotent — designed assuming retries.
>
> Around all of that: an Azure Logic Apps alerting layer that watches for SLA breaches and execution failures, and a Power Apps dashboard for ops to see workflow health and replay DLQ items."

### Beat 3 — the AI placement (60 seconds, this is the most important beat)

> "On AI specifically — there are two places it earns its slot.
>
> First, a fallback classifier for call-to-action strings the rules don't recognize. The deterministic rule covers schedule-tour, interest-list, onsite registration, third-party platforms — that's 95% of leads. When marketing adds a new CTA value the rule hasn't seen, the system doesn't drop the lead — it sends the projected, PII-stripped metadata to Azure OpenAI, gets back a best-guess priority with a confidence score, and routes low-confidence cases to human review. The AI also suggests rule additions, so over time the rule grows and AI usage shrinks. AI as discovery tool.
>
> Second, intent extraction from free-text notes — when an OSC writes a comment like 'family of four, looking for 4-bedroom in Hudson, urgent — closing on current home next month,' the AI extracts structured intent. Free text is the canonical AI use case because no rule scales over open vocabulary.
>
> Everything else is rule-based, deliberately. Priority for known CTAs is a string match — running an LLM on it would add latency, cost, a hallucination surface, and PII exposure for no benefit. Round-robin is a SQL-style query against a `rep_capacity` custom object. Idempotency is a key check before each write. Senior judgment here is recognizing where AI earns its slot and where rules already do the job."

### Beat 4 — outcomes (30 seconds)

> "Operationally, the coordinator's 30 hours a week of manual routing went to near zero — the role was reassigned to higher-value work. Average deal-close time moved from 41 days to 36 days over a 90-day post-launch window. SLA breach rate — deals sitting more than three days without a touch — dropped from about 18% to under 2%.
>
> Honest caveat on the close-time number: I can't fully isolate it from confounders like seasonality and other initiatives running in parallel. I'd characterize it as 'associated with' the routing automation, not strictly 'caused by.'"

### Beat 5 — pause and redirect (the move that distinguishes you)

> "That's the shape. Where would you like to go deeper?"

Don't keep talking. Wait. Let them pick the direction. **The candidate who stops talking signals confidence; the candidate who keeps filling silence signals anxiety.**

---

## 3. Layer 3 — deep-dive triggers and what to say

Each of these is what you say when the interviewer drills into a specific area. Each is 60–90 seconds, then pause again.

### 3.1 If they ask about round-robin

> "The state lives in a HubSpot custom object called `rep_capacity` — one record per rep, with multi-select properties for region, lead-type, and priority eligibility, plus a `last_assigned_at` timestamp and a `current_open_count`.
>
> The decision flow queries that object filtered by eligibility, sorted ascending on `last_assigned_at`, takes the top result. If the top two are within an hour of each other, we tie-break on `current_open_count` to avoid piling onto someone who's already behind.
>
> Race conditions matter — two leads arriving simultaneously could both pick the same rep. We solved that with single-concurrency on the decision step. At higher volume the path is to put the decision output on a Service Bus queue with a single consumer, decoupling decision throughput from trigger arrival. We didn't hit that scale; the design accommodates it.
>
> No AI in this path. The decision is structured query plus a deterministic tie-break. If anything went wrong here it would be a database-level concern, not an AI-quality concern."

### 3.2 If they ask about idempotency

> "Three layers, because retry safety is the core requirement — HubSpot rate-limits, webhooks redeliver, and you never want a deal to flip owners twice in a rep's activity feed.
>
> Every decision generates an `attemptId` UUID at decision time. Before any write, the execution layer checks for an existing applied assignment with that ID — duplicate delivery is a no-op. Second, the owner update is conditional: if the current owner already matches the target, we don't issue the write at all. Third, the assignment record itself is append-only — we never overwrite history.
>
> Combined, the same decision event can be delivered ten times and nothing double-applies. Audit trail stays clean. No AI in this path either — it's deterministic state-machine logic with explicit invariants."

### 3.3 If they ask about the AI fallback specifically

> "The fallback runs when the deterministic rule returns 'unknown' for a call-to-action. The lead is projected first — only metadata fields go to the model: CTA string, neighborhood name, lead source, broker flag. Names, emails, phones, broker contact info are never in the prompt. The model literally cannot leak what it never received.
>
> The system prompt restates the same priority rules the deterministic classifier uses, and asks for a JSON response with priority, confidence, reasoning, and a suggested rule addition. Output schema is enforced via Pydantic so downstream code is type-safe.
>
> If confidence is below 0.7, the lead routes to a human review queue rather than auto-applying. If the AI's suggested-rule-addition field is populated, it surfaces in the dashboard as a candidate rule for ops to promote.
>
> Eval-wise: a golden set of edge-case CTAs runs on every prompt or model change, and we sample production AI calls for human review to catch drift. Quality drift is measurable, not faith-based."

### 3.4 If they ask about PII handling

> "Layered, because relying on prompt-level cleverness alone isn't enough.
>
> First, data minimization. The lead envelope from HubSpot has names, emails, phones, broker contacts, internal owner records. Classification only needs the CTA string and a small amount of neighborhood metadata. So before any prompt is built, the lead goes through a projection function that returns only non-PII fields. Strongest possible PII control: the model never sees what isn't in the input.
>
> Second, redaction for free-text fields when present. If an OSC's notes contain names, addresses, or phone numbers, Presidio runs entity-level redaction with reversible token mapping kept on our side. The model sees `<PERSON_1>` not 'John Smith.'
>
> Third, defense in depth at the platform: Azure OpenAI Private Endpoint for regulated workloads, Microsoft-side content logging disabled via the abuse-monitoring opt-out, Managed Identity auth instead of API keys. On our side we log token counts, latency, model version, and decision outcomes — but never raw prompt content. Audit trail captures decisions with prompt and model versions for reproducibility."

### 3.5 If they ask why HubSpot custom objects instead of contact properties

> "Because an assignment is a fact that happened at a point in time, not a current state. Properties get overwritten; custom-object records are an append-only log. That distinction matters when commissions are audited — the question 'who owned this deal on March 3rd' is a real business question, and you can only answer it if assignment history is first-class data.
>
> Custom objects also model many-to-many cleanly via association labels. One contact has many assignments over time; one assignment relates to a contact and a deal and a rep. Properties can't express that."

### 3.6 If they ask about Power Automate vs n8n

> "The implementation history is honest: n8n was the original platform when I built this — the patterns I'm describing are platform-independent and the n8n version was the production system. I've since architected the equivalent design for Power Automate to align with where the enterprise stack is going.
>
> The patterns transfer cleanly. n8n's workflow-with-error-trigger is Power Automate's Scope-with-runAfter-Failed pattern. n8n's HTTP node with retry is Power Automate's HTTP action with a configured retry policy. Custom objects and HubSpot's API don't change. The decision/execution split, idempotency strategy, audit-trail design — all platform-independent.
>
> Where Power Automate adds value over n8n is the ALM tooling, the connector ecosystem, and the enterprise governance posture — solutions, environment variables, connection references, DLP policies, CoE tooling. For a Marken-scale deployment those matter."

### 3.7 If they ask about scale / volume

> "The system runs at roughly 400 leads per week peak. At that volume, single-concurrency on the decision step is fine — throughput isn't the binding constraint.
>
> If volume scaled 10× or 100×, the design accommodates it: move the decision output to a Service Bus queue with a single consumer to serialize the critical section while letting the trigger ingest in parallel; partition the queue by neighborhood for higher parallelism if neighborhood independence holds; move state from custom-object queries to a faster cache for the hot path. The architecture isn't scale-bound; it's correctness-first, and correctness scales by adding queueing and partitioning rather than rewriting the logic."

### 3.8 If they ask about failure handling

> "Three layers, mapped to failure class.
>
> Transient failures — HubSpot 429s, network timeouts — get exponential-backoff retries with jitter, capped at five attempts. We honor the `Retry-After` header on 429 responses.
>
> Persistent failures — bad data, schema mismatches, downstream service down for an extended period — go to a dead-letter queue with the original payload, error trace, and retry history. Ops reviews DLQ daily through the Power Apps dashboard, with one-click replay.
>
> Systemic failures — error rate above 40% over a 30-request window — trip a circuit breaker that pauses the decision flow via a shared control flag. Prevents us from burning quota when the downstream is genuinely unhealthy.
>
> Alerting on all of this routes through Azure Logic Apps — different severity levels go to different channels: DLQ depth growth to a Teams channel, circuit trips and SLA breaches to PagerDuty for on-call."

### 3.9 If they ask about how you measured the savings

> "Two parts to the measurement, and I want to be honest about the rigor on each.
>
> The 30-hours-per-cycle number is from the coordinator's self-reported timesheet pre-rollout. I cross-checked it with volume — about 400 leads a week times an estimated 4 minutes per lead of coordinator time triangulates to roughly 27 hours, which lines up with the self-report. Post-rollout, the coordinator was reassigned and automation handles routing end-to-end.
>
> The 41-to-36-days close-time improvement is from the same HubSpot deal query pre and post, averaged over 90-day windows. I can't fully isolate it from confounders — there were other initiatives running in parallel, including lead qualification automation, plus seasonal effects. So I'd frame it as 'associated with' the routing improvements, not strictly 'caused by.' If someone pushed on that I wouldn't claim a controlled-experiment causal link, and I think being calibrated on what we can actually demonstrate matters more than overclaiming."

This last paragraph is **gold for senior signaling**. Hiring managers know nobody A/B-tests internal automation. The candidate who admits that and calibrates is the candidate who'll be honest in a production incident.

---

## 4. Where AI is and isn't — verbal labeling cheat sheet

When walking through any component, **explicitly label** rule vs AI vs human. Don't make them guess.

| Component | Label | Verbal phrase to use |
|---|---|---|
| Ingress webhook | **No AI** | "Ingress is just a webhook from HubSpot — code, no AI." |
| PII projection | **No AI** | "Code-level projection — strips fields the downstream doesn't need." |
| Priority classification (known CTA) | **No AI, deliberately** | "Rule-based — known CTAs map to priorities. AI here would add cost and risk for no benefit." |
| Priority classification (unknown CTA) | **AI step #1** | "Fallback classifier — AI handles the CTAs the rule hasn't seen." |
| Eligibility filter | **No AI** | "Structured query against the rep_capacity custom object." |
| Round-robin pick | **No AI** | "Deterministic — oldest last_assigned, tiebreak on open count." |
| Idempotency check | **No AI** | "AttemptId lookup — pure code." |
| Conditional update | **No AI** | "If current owner matches target, no write — guards against retry doubling." |
| Notes intent extraction | **AI step #2 (optional)** | "Free-text fields, when present, get AI extraction with a Pydantic schema." |
| Confidence routing | **No AI (consumer of AI output)** | "Below 0.7, the lead goes to human review — that's a code branch on AI's output." |
| HubSpot writes | **No AI** | "Standard API calls, retry-safe, idempotent." |
| Audit log | **No AI** | "Append-only record of every decision with model and prompt versions." |
| SLA watcher | **No AI** | "Logic Apps schedule + HubSpot query — code." |
| Power Apps dashboard | **No AI** | "View layer over Dataverse — code." |

**Two AI steps. Both bounded. Both with reasons.** When you can name every box and what's in it without hesitation, you've crossed the senior threshold.

---

## 5. Common interruptions and recovery moves

### "Why didn't you use AI for [X]?"

Default answer pattern:
> "Because [X] has structured input and a deterministic decision rule. AI would add latency, cost, a hallucination surface, and a governance scope — none of which buy anything when an `if`-statement already gets the right answer at zero error rate. AI's value is in handling fuzziness; this part wasn't fuzzy."

### "Could you have used AI for [X]?"

> "Technically yes; senior judgment is whether you should. For [X], the answer was no because [the four-question test fails on input shape / decision space / failure cost / rule sufficiency]. We reserved AI for the parts where it actually changes outcomes."

### "What if the LLM is down?"

> "The two AI steps are non-blocking. If the AI fallback classifier is unavailable, unknown CTAs route to human review — same as if confidence is low. If the notes-extraction step is unavailable, the lead routes anyway with notes flagged for human read-through. AI is a quality enhancer in this system, not a critical path. The deterministic rule covers 95% of leads regardless."

### "How would you know if the AI is silently wrong?"

> "Three signals. Golden-set regression tests run on every prompt or model change. Production sampling — a small percentage of AI classifications get human-reviewed and labeled. Downstream feedback — when a human in the dashboard corrects an AI-suggested priority, that correction is the highest-signal data and feeds back into the eval set. None of it is faith-based."

### "What's the cost of running this?"

> "AI cost is modest because AI runs on the long tail. Rules handle 95% of leads at zero AI cost. Of the remaining 5%, gpt-4o-mini at the projected token volume is on the order of a few dollars a month. The cost wasn't the binding constraint; the constraint was making sure AI was used where it earned its slot rather than reflexively."

### "What would you do differently if you rebuilt it?"

> "Three things. Put the decision event on a durable queue from day one rather than direct flow-to-flow calls — Service Bus gives you native retry, DLQ, and replay for free. Build the monitoring dashboard before the main feature so we wouldn't have spent the first six weeks debugging through run logs. And tighten governance on the `rep_capacity` records earlier — we had a drift incident where a manager edited eligibility flags without review, and it cost us two weeks of fairness in the distribution. Routing state is configuration, and configuration needs governance, not just code does."

### "Tell me about something that broke in production."

Use the rep-capacity-drift incident from the technical guide. It's specific, real-feeling, and teaches a senior lesson about treating routing state as governed configuration.

---

## 6. The whiteboard version (if they ask you to draw it)

If the interviewer asks "can you draw that out," do this **exact** sequence on the board. Narrate as you draw.

```
1. Draw: [HubSpot lead created] → arrow → [Webhook receiver]
   Say: "Ingress. Webhook from HubSpot fires on lead creation."

2. Draw: arrow → [Project & redact PII]
   Say: "First step is a code-level projection — only non-PII fields go forward."

3. Draw: arrow → [Priority classifier]
   Inside that box, draw two paths:
     [Rule match? YES] → priority assigned
     [Rule match? NO]  → [AI fallback (Azure OpenAI)] → priority + confidence
   Say: "Classifier is two-tier — rules first, AI fallback for unknowns. Label these clearly: rule path is deterministic, AI path is the only AI step in the routing decision."

4. Draw: arrow → [Eligibility filter (rep_capacity custom object)]
   Say: "Eligibility is a query on a HubSpot custom object — region, type, priority match."

5. Draw: arrow → [Round-robin pick (oldest last_assigned, tiebreak on open count)]
   Say: "Round-robin with availability tiebreak. State lives on the rep custom object."

6. Draw: arrow → [Decision event] (highlight this box)
   Say: "This is the boundary between layers. Decision is a pure function up to here. From here on is execution."

7. Draw: arrow → [Idempotency check (attemptId lookup)] → [Conditional HubSpot update]
                                                         → [Append assignment record]
                                                         → [Update rep last_assigned]
                                                         → [Mirror to SharePoint]
   Say: "Execution layer. Every step is idempotent — retry-safe."

8. Draw to the side:
   [Logic Apps: SLA watcher + failure escalation]
   [Power Apps: workflow health dashboard + DLQ replay]
   Say: "Around it: alerting via Logic Apps, ops dashboard via Power Apps."

9. Step back. Mark the AI step #1 (fallback classifier) and AI step #2 (notes extraction, if drawn) with a different color or asterisk.
   Say: "Two AI steps. Everything else is rule-based."
```

That's a clean, explainable whiteboard in 4–5 minutes of drawing while talking. Practice once before the interview.

---

## 7. Bridging to your other work

When relevant, briefly tie this project to your wider experience without derailing the walkthrough.

### When to reference Gen AI Engineer work
If they ask about AI patterns specifically:
> "The Pydantic-enforced structured-output pattern — same thing I rely on in my current Gen AI Engineer role for an unrelated QA-automation system. There the failure mode was malformed LLM responses breaking downstream parsing; the fix was the same structured-output contract. The pattern generalizes: AI is a useful component when its output shape is rigorously constrained."

### When to reference UPS BaSE work
If they ask about reliability or scale:
> "I work on the UPS side as well, supporting logistics systems running 2–3 million packages a day. The discipline of running production at that scale — change control, on-call rotation, post-mortems — shapes how I design even smaller automations. The idempotency-and-DLQ-and-circuit-breaker discipline you're seeing in this project comes from that environment."

### When to reference n8n vs Power Automate
Use the framing from Section 3.6 above. Short, honest, forward.

**Don't do**: long tangents that derail the lead routing walkthrough. Reference, then return.

---

## 8. Things to NOT say

- **"It was pretty simple."** Round-robin with eligibility, idempotency, two-tier classification, audit trail, and PII handling is not simple. Don't shrink the work.
- **"I used AI to build this."** This question is about *what you built*, not what tools you used to write code.
- **"I'm still learning Power Automate."** Replace with: "Power Automate is a short platform ramp on top of patterns I already apply in production."
- **"We didn't really test it / measure it."** Replace with the measurement methodology + honest caveats from Section 3.9.
- **"AI is the future / AI is everywhere."** Editorializing. Stay specific to the decisions you made.
- **"Probably / maybe / I think / kind of."** Hedge words appearing every sentence read junior. Use sparingly and only where genuinely uncertain.
- **"To be honest..."** (implies the rest of what you said wasn't honest). Just be honest.

---

## 9. Practice protocol

The script is only useful if it's reflexive on the day. Three rounds of practice:

**Round 1 — read aloud, no timer.** Read each layer aloud. Listen for awkward phrases or words you stumble on. Edit until everything flows.

**Round 2 — timed, with the script in front of you.** Layer 1 should land in 75 seconds, Layer 2 in 4 minutes. If you're over, cut. If you're under, you're rushing.

**Round 3 — mirror or recording, no script.** Speak it from memory while watching yourself or recording. Listen for: filler words, monotone delivery, looking down (you're imagining the script). Re-record until it sounds conversational, not recited.

**Stop practicing 24 hours before the interview.** Last-minute drilling produces a recited delivery; a day off produces a relaxed, conversational one. The difference is visible.

---

## 10. The single most important thing on the whiteboard

When you label the diagram, **physically distinguish AI boxes from rule boxes** — different color, asterisk, dashed border, anything. The visual reinforces the verbal: "two AI steps, deliberately bounded, everything else rule-based."

The diagram alone communicates judgment. The narration confirms it. Together they're decisive.

---

## 11. Final reminder — the meta-move

The interviewer is not grading you on the *project*. They're grading you on whether you can:

1. Explain a complex system at multiple levels of detail without getting lost.
2. Defend technical decisions in terms of cost, value, and tradeoffs.
3. Draw the line between AI and rules in a way that signals deliberate choice, not default.
4. Be calibrated about what you know, what you measured, and what you can claim.

The lead routing system is the artifact. The walkthrough is the actual interview. Treat the script accordingly.
