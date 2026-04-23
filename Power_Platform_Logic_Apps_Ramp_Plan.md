# Power Automate / Power Apps / Logic Apps — 2–3 Week Interview Ramp Plan

**Target role:** Senior AI Hyper Automation Developer — UPS Healthcare / Marken
**Companion to:** `Senior_AI_Hyper_Automation_Interview_Prep.md` (Parts I–III) and `Senior_AI_Hyper_Automation_Interview_Prep_QuestionBank.md` (Part IV)
**Goal:** Interview-ready, not certification-ready. Tuned for your background (n8n, HubSpot, C#/.NET, prototype-level Logic Apps).

---

## 0. Setup (1–2 hours — do this first)

Everything below is free.

1. **Microsoft 365 Developer Program** — https://developer.microsoft.com/microsoft-365/dev-program. Free, renewable M365 tenant with 25 user licenses, SharePoint, Dataverse, Teams, and Power Platform. This is your sandbox.
2. **Power Platform environments** — from the Power Platform admin center, create a *dev* environment with Dataverse enabled. Do NOT work in the default environment — you want to rehearse ALM patterns from day one.
3. **Azure free account** — https://azure.microsoft.com/free. $200 credit + always-free services. Enough for Logic Apps Consumption + Standard experimentation.
4. **VS Code + extensions:**
   - Azure Logic Apps (Standard)
   - Bicep
   - Power Platform Tools
   - Azurite (local storage emulator — Logic Apps Standard needs it)
   - Azure Functions Core Tools (`func` CLI)

Skipping this setup is the #1 reason people regret the ramp later. Two hours now saves a day of friction.

---

## 1. Translate, don't relearn (Day 1 — 1 hour)

Most of the "learning" is vocabulary. Your mental model from n8n/HubSpot transfers cleanly; the words are different.

| You know (n8n / HubSpot) | Microsoft equivalent |
|---|---|
| n8n workflow | Power Automate flow / Logic App workflow |
| n8n trigger node | Trigger (Dataverse row added, HTTP request, Service Bus message, schedule) |
| n8n HTTP Request node | HTTP action / custom connector |
| n8n Function node (JS) | Expression language / Power Fx / inline JavaScript (Logic Apps) / Office Script |
| n8n credentials | Connection / Connection reference / Managed Identity |
| n8n Error Trigger | Scope with `runAfter: [Failed]` |
| HubSpot workflow | Automated cloud flow |
| HubSpot custom object | Dataverse table |
| HubSpot association | Dataverse relationship (1:N, N:N) |
| HubSpot list | Dataverse view |
| Environment variable (n8n) | Environment variable (Power Platform) or app setting (Logic Apps) |
| Secret in n8n creds | Azure Key Vault reference |

Do this table on paper. Fastest concept transfer possible.

---

## 2. Three hands-on projects (where real skill comes from)

Skip long tutorials. Build three things that hit every interview-relevant surface.

### Project A — Power Automate + Power Apps (4–6 hours)

**Build: a "Shipment Exception Triage" mini-app.**

**Dataverse table:** `Shipment`
- Id (auto)
- Carrier (choice: DHL, FedEx, UPS, Marken)
- Status (choice: Created, In Transit, At Risk, Delivered, Exception)
- TemperatureC (decimal)
- DueBy (datetime)
- AssignedTo (lookup → User)
- Notes (multiline text)

**Power App (canvas):**
- Gallery listing shipments, filter by status.
- Detail form that lets you edit, Patch back to Dataverse.
- A "Mark as Exception" button that calls a flow.

**Automated flow #1** — trigger: Dataverse row added or modified.
- Condition: `TemperatureC > 8` OR `DueBy < utcNow()`.
- If true: post adaptive card to Teams, send email, set `Status = 'Exception'`.

**Automated flow #2** — trigger: manual (from the app's "Mark as Exception" button).
- Approval action (Teams adaptive card) to supervisor.
- Branch on approval outcome — update record accordingly.

**Scheduled flow** — trigger: recurrence (daily 7am).
- Query all `Status = Exception`. Compose an HTML digest. Email to a manager distribution list.

**Child flow** — extract a reusable "Notify Ops" block and call it from both automated flows.

**Senior-level signals to practice:**
- Everything goes in a **solution** (not the default environment). Practice: export unmanaged to dev backup, managed to "test" environment.
- Use **connection references** — never bind flows to specific connections directly.
- Use **environment variables** for the temperature threshold, distribution list addresses, etc.
- Wrap risky actions in a **Scope** with Try / Catch / Finally pattern:
  - `Scope_Try` — the real work.
  - `Scope_Catch` with `runAfter: [Failed, TimedOut]` — log, notify, escalate.
  - `Scope_Finally` with `runAfter: [Succeeded, Failed, TimedOut, Skipped]` — cleanup, emit metric.
- Test **delegation** in the canvas app: filter a big dataset and watch for the delegation warning squiggle.
- Rehearse the expression language cold: `coalesce()`, `if()`, `formatDateTime()`, `addDays()`, `parseJSON()`, `items('Apply_to_each')`, `outputs('...')`, `body('...')`.

**After this project you can speak to:** connectors, triggers, expressions, Dataverse, solutions, connection references, environment variables, error handling, child flows, approvals, Power Fx, delegation.

---

### Project B — Logic Apps Standard (4–6 hours)

**Build: a shipment-event ingestion workflow.**

**Trigger:** Service Bus queue (create a namespace + queue in Azure). Post test messages via Service Bus Explorer in the portal.

**Workflow steps:**
1. Receive message (peek-lock).
2. `Parse JSON` with an explicit schema — validate shape.
3. HTTP call to an external API (use `httpbin.org` or an Azure Function you write — doesn't matter, just needs to be a realistic HTTP call).
4. On success:
   - Complete the Service Bus message.
   - Write a row to Azure Table Storage / Cosmos / SQL.
5. On failure:
   - Retry policy (exponential backoff, 3 attempts) on the HTTP action.
   - After retries exhausted: explicit dead-letter of the Service Bus message with reason.

**Enterprise patterns to implement:**
- **Managed Identity** for the Service Bus connection and Key Vault access. No connection strings in app settings.
- Secrets (API keys) via **Key Vault reference** (`@Microsoft.KeyVault(SecretUri=...)`) in app settings.
- Deploy the whole thing with **Bicep** — parameterized for Service Bus namespace, API endpoint, Key Vault name. One template, three environments.

**Things to deliberately try:**
- Build **both stateful and stateless** versions of the same workflow. Feel the difference: stateful has run history and is pricier; stateless is faster and blind.
- Local dev: `func start` and run the workflow locally in VS Code before deploying to Azure.
- Inspect `workflow.json` — this is the source-control-friendly definition. Commit it.
- Run a small load test (100–500 messages through Service Bus Explorer) and watch scaling.
- Intentionally break the downstream API (make it return 503) — watch retry kick in, then DLQ.

**Senior-level signals to practice:**
- Consumption vs Standard — articulate the decision signals (volume, networking, DevOps maturity).
- Why Managed Identity > connection strings (no secret rotation, no leaks, Azure-native auth).
- How run history + Application Insights integrate; how you'd query across runs.
- Bicep parameter files per environment — this is how you avoid the "worked in dev broke in prod" class of bug.

**After this project you can speak to:** Consumption vs Standard, Managed Identity, Key Vault references, retry policies, DLQ patterns, Bicep deployment, stateful vs stateless, peek-lock semantics, local Logic Apps development.

---

### Project C — Copilot Studio + AI-in-workflow (3–4 hours)

**Build: an ops assistant.**

- **Copilot Studio agent** grounded on a small set of PDFs (sample SOPs — any docs will do for practice).
- Add a **topic** triggered on phrases like "where is shipment X?" or "shipment status for 12345". The topic calls a Power Automate flow via the Copilot Studio connector; the flow reads the Dataverse table you built in Project A and returns status.
- Add a **generative AI fallback** for questions not matched by a topic.
- Publish to Teams. Test as a user.

**Senior-level signals to practice:**
- The boundary between **topics (deterministic)** and **generative fallback (probabilistic)** — this IS the core design decision in Copilot Studio.
- Grounding: upload documents, ask a grounded question, observe citation behavior.
- Auth passthrough: the user's identity flowing into the called flow.
- When you'd choose **Azure OpenAI directly** instead (prompt control, model choice, eval rigor, regulated data residency — see QuestionBank Q13 and Part III of the main prep doc).

**After this project you can speak to:** Copilot Studio architecture, topics vs generative fallback, grounding, hybrid bots (Copilot Studio front + Azure OpenAI back), Teams publishing.

---

## 3. Concepts to drill beyond the projects

Interview-likely, not naturally covered by building one flow.

### Power Automate specific
- **Premium vs standard connectors** + licensing implications. Commonly asked.
- **Concurrency control** on `Apply to each` — default sequential vs parallel, and the throughput/throttling tradeoff.
- **Trigger conditions** vs conditional steps inside the flow — trigger conditions don't consume a run when false. Real cost/perf impact.
- **Environment strategy** at enterprise scale: default (locked down), personal productivity, team, enterprise-prod.
- **DLP policies** at the environment level — which connectors can combine with which.
- **Solution Checker** — static analysis for flows/apps before promotion.

### Power Apps specific
- **Delegation** — what delegates to Dataverse/SQL vs evaluates client-side, and the 500/2000 row limit. Classic interview bait.
- **Canvas vs model-driven** — decision signals (UI control vs form-over-data, citizen vs pro dev, licensing).
- **Power Fx** basics: `Filter`, `LookUp`, `Patch`, `Collect`, `With`, `ForAll`.
- **Offline scenarios** — rare but occasionally asked.

### Logic Apps specific
- **ISE (Integration Service Environment) is deprecated** — know it so you don't get caught recommending it; the current answer is Logic Apps Standard with VNet integration.
- **Integration Account** (maps, schemas, B2B / EDI) — know what it's for; don't need deep skill.
- **Parallel branches, aggregation, concurrency control** on `Until` loops.
- **Run history retention limits** and why you ship diagnostics to **Log Analytics / Application Insights**.
- **Connector throttling limits** — what happens when you blow them, and how to design around.

### Cross-cutting
- **ALM pipelines**: Power Platform Pipelines vs ALM Accelerator vs Azure DevOps with the Power Platform Build Tools. Pick one, know the tradeoffs, be able to whiteboard the flow.
- **Monitoring**:
  - Logic Apps Standard → Application Insights (native).
  - Power Platform → CoE Starter Kit + Power Platform Admin Center.
  - Alerts on failure rate, SLA breach, quota consumption.
- **Governance**: DLP policies, environment strategy, CoE roles, connector allowlists, makers vs admins.

---

## 4. Resources (use sparingly — don't binge)

Use these as lookups *while building*. Courses are breadth; interview prep is depth.

### Primary
- **Microsoft Learn — Power Automate**: do only the "build automated cloud flows" and "ALM for Power Platform" paths.
- **Microsoft Learn — Logic Apps Standard**: "Build workflows with Azure Logic Apps Standard" path.
- **Microsoft Learn — Copilot Studio**: one short path, enough for concepts.
- **Microsoft Learn — Dataverse fundamentals**: useful for schema design.

### Senior-level blogs and channels
- **Kent Weare's blog** — enterprise Power Platform patterns.
- **John Liu's blog** — practical senior-level techniques.
- **Serverless360** — Logic Apps in enterprise contexts.
- **YouTube: April Dunnam** — Power Platform practical walkthroughs.
- **YouTube: Microsoft Reactor** — Logic Apps and Azure walkthroughs.
- **Microsoft Architecture Center**: reference architectures for integration scenarios.

### Avoid
- Long Pluralsight / Udemy courses. They optimize for breadth and cert exam coverage; you need targeted depth on a narrow interview surface.

---

## 5. Interview-readiness checklist (Week 3)

If you can do these cold — from a blank whiteboard, no notes — you're ready.

- [ ] Whiteboard end-to-end automation: trigger → orchestration → error handling → retry → DLQ → observability. Name specific Microsoft components at every step.
- [ ] Explain Consumption vs Standard Logic Apps with a specific decision signal (volume, networking, DevOps, cost curve) — not vague "Standard is more powerful."
- [ ] Explain delegation in Power Apps and what happens when it fails.
- [ ] Walk through a solution-based ALM pipeline end-to-end without notes (dev → test → prod, managed vs unmanaged, connection references, environment variables).
- [ ] Describe how you'd secure a flow that calls an internal OAuth-protected API: Managed Identity + Key Vault + connection reference + no secrets in flow definitions.
- [ ] Explain when you'd pick Copilot Studio vs a custom Azure OpenAI bot (see Part IV Q13 of the QuestionBank).
- [ ] Show a live flow or app you built — some interviews ask for screenshots or screen share.
- [ ] Translate your n8n / HubSpot stories into Microsoft vocabulary on the fly. Practice this out loud.
- [ ] Explain the try/catch/finally Scope pattern in Power Automate with an example.
- [ ] Draw a Logic Apps Standard architecture with VNet integration, Managed Identity, Key Vault, and Service Bus.

---

## 6. Time-boxed versions

**3 weeks (recommended):**
- Week 1: Setup + translation table + Project A (full depth).
- Week 2: Project B (full depth, including Bicep deploy).
- Week 3: Project C + Section 3 concept drill + Section 5 checklist.

**1 week (if that's all you have):**
- Day 1: Setup + translation table.
- Days 2–4: Project A in depth.
- Days 5–6: Project B (skim — just build once, skip Bicep if time-pressed).
- Day 7: Section 5 checklist + rehearse positioning.
- Skip Copilot Studio hands-on; hold conceptual framing only.

**3 days (emergency mode):**
- Day 1: Setup + translation table + Project A.
- Day 2: Project B at a conceptual level (build a "hello world" flow + read the architecture docs).
- Day 3: Section 5 checklist + rehearse "platform ramp, not domain ramp" positioning.
- Honest framing will beat shallow demos.

---

## 7. The meta-point

Your interview edge isn't catching a Power Platform specialist on platform depth in three weeks — you won't, and you don't need to.

Your edge is demonstrating that you understand **production-grade automation patterns** — idempotency, retry, DLQ, observability, ALM, governance, security — and that you can apply them with Microsoft tools in your hands. These three projects let you show that on demand.

The gap between you and "needs 6 months of runway" is already smaller than the job description implies. These projects prove it.

---

## 8. What to flag honestly in the interview

- "I've built three hands-on projects in the Microsoft stack to validate the platform ramp: a Power Platform exception-triage app with full ALM, a Logic Apps Standard ingestion workflow deployed via Bicep with Managed Identity and Key Vault, and a Copilot Studio assistant integrated with Power Automate."
- "My production automation depth comes from n8n and HubSpot at scale; the patterns transfer directly. What I've been sharpening is Microsoft-specific vocabulary and platform conventions."
- "I'm not going to tell you I have 3 years of production Power Automate — I don't. What I'll tell you is I know what good looks like on any workflow platform, and I've verified on this one."

That's the senior framing. Honest, specific, forward-moving.
