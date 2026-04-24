# Technical Round Playbook — Senior AI Hyper Automation Developer

Session-level guide for the technical round itself: what it'll look like, what they'll probe, and how to execute each section.

**Complements — do not duplicate — the existing artifacts:**
- `Senior_AI_Hyper_Automation_Interview_Prep.md` — positioning and training
- `Senior_AI_Hyper_Automation_Interview_Prep_QuestionBank.md` — 120 Qs for drill
- `Lead_Routing_System_Technical_Guide.md` — your headline-story defense
- `Power_Platform_Logic_Apps_Ramp_Plan.md` — hands-on build plan

This one is about the round itself: format, minute-by-minute flow, execution moves.

---

## 1. What the Round Will Actually Look Like

**Interviewers.** 1–3 engineers from the HM's team. Likely mix:
- A senior Power Platform developer (primary technical evaluator)
- An integration or Azure specialist (Logic Apps / enterprise integration)
- Possibly an AI/automation-adjacent person if the loop has more than one slot

**Format — most probable shapes:**
- **Option A (most common for this role type):** single 60-min session, 1 interviewer, conversational. Resume walkthrough + technical questions + design exercise. No coding.
- **Option B:** two back-to-back 45–60 min sessions, different interviewers, each focused (one Power Platform, one integration/AI).
- **Option C:** 60–90 min panel with 2–3 interviewers sharing questions.
- **Option D (less common):** 30-min quick screen first, then a deeper 60-min round.

**Coding component likelihood:** ~30–40%. Practical (Python script, API call, data transformation) — not LeetCode.

**Whiteboarding / live design:** ~60% likely in some form, usually verbal, occasionally a shared doc or Miro/Figma board.

**Total duration:** 45–90 minutes, typically 60.

**Ask the recruiter when they call:** *"What does the interview loop look like — will there be a coding component or a design exercise?"* Honest answer removes guesswork.

---

## 2. Content Coverage — What They'll Actually Ask

Ranked by probability based on the JD and role shape:

### Highly likely (80%+)
- **Walk me through a flow / system you built.** Lead Routing story is the headliner.
- **Power Automate questions** — connectors, flow types, triggers, actions, expressions, error handling.
- **Error handling and reliability** — retries, idempotency, failure modes.
- **Integration design** — REST, OAuth, JSON, rate limiting, pagination.
- **AI in workflows** — LLM integration, structured output, validation, guardrails.

### Likely (50–80%)
- **Design exercise** — "design an automation for X."
- **Platform tradeoffs** — Power Automate vs Logic Apps, when to use each.
- **Prompt engineering** — structured output, Pydantic, JSON schema, evals.
- **Solutions / ALM** — environments, connection references, promotion pipelines.
- **Copilot Studio familiarity** — named in JD.

### Possible (30–50%)
- **Logic Apps specifics** — stateful vs stateless, Consumption vs Standard.
- **Coding problem** — 30–45 min practical Python.
- **Dataverse basics** — tables, relationships, vs SharePoint for state.
- **RPA / Power Automate Desktop** — JD lists it as desirable.
- **Debugging scenario** — "your flow is failing intermittently — diagnose."

### Less likely (<30%)
- **MuleSoft** — desirable, not required. Don't fake.
- **Deep Azure services** (Functions, Service Bus internals) — unless role creep.
- **Algorithmic coding** — very low for this role.
- **Distributed systems at scale** — unusual for this level.

---

## 3. The Likely Interview Flow (60-min version)

**0:00 – 0:03 · Intro**
Interviewer introduces themselves. Brief TMAY if asked. If they skip it, don't force it.

**0:03 – 0:15 · Resume deep-dive**
*"Walk me through the most complex / most recent thing you've built."*
→ Lead Routing story. 30-sec verbal, then ready to go 5 minutes deep on any piece.
→ Expect 3–5 follow-up questions on specific technical decisions.

**0:15 – 0:35 · Technical questions**
Rapid-fire or focused on 2–4 topics from §2.
→ Conversational. They ask, you answer 60–90 sec, they follow up.
→ At least one "how would you handle X" scenario question.

**0:35 – 0:50 · Design exercise (if present)**
*"We need to automate [scenario]. How would you approach it?"*
→ Verbal or shared board.
→ Use §6 template — scope first, then design.

**0:50 – 0:55 · Your questions**
2–3 ready. Different from HM questions. See §8.

**0:55 – 0:60 · Close**
Next steps, timeline, thank them.

**If coding is present:** replaces or extends the design block — 30–40 min carved out of the middle.

---

## 4. How to Execute Each Section

### Resume deep-dive (minutes 3–15)

Most important 12 minutes of the round. One headline story: **Lead Routing Engine.** Rehearse the 30-sec verbal until automatic. Then be ready to go deep on:

- Decision/execution separation — why, how
- Round-robin algorithm — last-assignment storage, read/write pattern
- Idempotency — key, store, retry behavior
- Dual-entity writes — consistency, failure handling
- Data model — HubSpot custom objects, associations, why that structure
- Power Automate angle — what was ported, why n8n stayed primary

**Answer pattern (three beats):** one-sentence summary → concrete detail → tradeoff or alternative. Don't stop at summary — that's the junior version.

### Technical questions (minutes 15–35)

**Scenario questions — STAR-lite:**
- **Setup** (1 sentence) — the context
- **Approach** — what you'd do, 2–3 specific technical choices
- **Tradeoff** — what you considered but rejected
- **Result** — how you'd know it worked

Target 60–90 sec per answer. Pause for follow-ups rather than burning all the air.

**Open technical questions ("how does idempotency work?"):** pattern → one concrete implementation → failure mode it protects against. Three beats.

### Design exercise (minutes 35–50)

See §6. The scope-first move is the senior signal.

### Your questions (minutes 50–55)

See §8.

---

## 5. Headline Stories — Which to Use When

Three headline stories, mapped to likely prompts:

**Lead Routing Engine** — your deepest and most impressive.
- Use for: "walk me through a complex project," "event-driven work," "orchestration," "reliability"
- Verbal: 30 sec. Deep-dive: 5 min ready.
- Covers: triggers, decision logic, round-robin, idempotency, dual-entity writes, data modeling.

**AI QA Automation**
- Use for: "AI in workflows," "reliability with non-deterministic steps," "Pydantic / structured output," "observability for AI."
- Verbal: 30 sec. Deep-dive: 3 min.
- Covers: LLM tool calls, retry/checkpoint, structured output validation, structured logging.

**Power Platform Alerting Subsystem** (Power Apps + Logic Apps + polling flow)
- Use for: "Microsoft stack experience," "monitoring / observability," "multi-component system," "cross-platform work."
- Verbal: 30 sec. Deep-dive: 3 min.
- Covers: Power Apps dashboard, Logic Apps dispatcher, 5-min polling, support team integration.

If you have latitude, **lead with Lead Routing** — deepest, most impressive, most rehearsed.

---

## 6. Design Exercise — 7-Step Template

When asked *"design an automation for X"*:

### Step 1: Scope (do not skip — this is the senior signal)

*"Before I design, a few things I'd want to understand:"* Pick 2–3 from:
- Volume and latency
- Consumers (who uses the output, how)
- Failure tolerance (can some fail silently, or is everything load-bearing)
- Compliance context (GxP, PHI, audit requirements — this matters at Marken)
- Source and destination systems

### Step 2: Name the pattern

One sentence. *"This is a [content-based routing / scatter-gather / claim-check / event-driven pipeline] pattern."*

### Step 3: Sketch the flow

4–7 steps, in Microsoft vocabulary:
- **Trigger** — event source, why that trigger
- **Parse / validate** — schema validation, fail-fast
- **Enrich** — pull additional data if needed
- **Decision / routing** — Switch/Condition, or AI classification
- **Execute** — idempotent write to system of record
- **Audit** — log outcome
- **Alert** — notify on failure

### Step 4: Call out the non-obvious pieces

These are the things most candidates miss — say them explicitly:
- **Idempotency keys** on every write
- **Retry policy** (exponential, specific status codes, max attempts)
- **Rate-limit awareness** on source/destination APIs
- **Human-in-the-loop** on low-confidence AI outputs
- **Audit trail** for compliance

### Step 5: Name the platform choice

*"I'd implement this on [Power Automate / Logic Apps Standard / Consumption] because [VNet / cost model at volume / IT ownership / source-control requirement]."* Not "because it's easier."

### Step 6: Call out what could go wrong

*"The failure modes I'd watch for are [X] and [Y]. I'd handle them with [specific mechanism]."*

### Step 7: Invite critique

*"Does that match what you had in mind, or is there a constraint I missed?"*

Invites dialogue, avoids monologue, surfaces anything you missed without penalty.

---

## 7. If There's a Coding Component

### Expected shape

- **Language:** Python (you selected it on the application)
- **Task:** practical, 30–45 min — integration / automation flavor, not LeetCode. Common shapes:
  - Parse a JSON response, filter, transform, output
  - Call a REST API with auth, handle pagination or retries
  - Dedupe, group, or aggregate a list of records
  - Validate input against a schema (Pydantic or manual)
  - Wrap an LLM call with retry + validation
- **Format:** shared editor (CoderPad, HackerRank, or screen-share VS Code)

### Execution moves

1. **Read the problem twice before typing.**
2. **Ask 1–2 clarifying questions** — *"does order matter?"* / *"what should happen on a 5xx?"* / *"are duplicates possible?"* / *"what's the expected input size?"*
3. **Narrate your approach before coding** — 30 sec. *"I'll do X, then Y, then Z. Using `requests` for HTTP, dict keyed on ID for dedup."*
4. **Code in visible chunks.** Talk through each section as you write it.
5. **Test with 1–2 inputs** if time allows, even trivially.
6. **Name tradeoffs explicitly.** *"Sync implementation here; for higher volume I'd move to async."*

### Quick refresh (30–60 min, the morning of)

Each topic is paired with the practice problem that drills it:

- `requests` — GET/POST, headers, JSON body, `raise_for_status()`, timeout → **Q2, Q3, Q4**
- JSON — `json.loads`, nested access, `dict.get(key, default)` → **Q1, Q6**
- Error handling — `try/except` with specific exceptions, retry with exponential backoff → **Q3, Q9**
- List/dict comprehensions → **Q1**
- `collections.defaultdict`, `Counter` → **Q7**
- Pydantic — model definition, validation, `.model_validate()` → **Q8, Q9**
- Recursion over nested structures → **Q6**
- Time-windowed state (deque, `time.monotonic`) → **Q10**

### Practice problems with solutions

Ten problems, calibrated to the shape of tasks that actually come up for integration / automation roles. Work through them top-to-bottom — difficulty ramps up.

---

#### Q1 · Filter JSON records *(Basic)*

**Problem.** You're given a list of tickets. Return only those where `status == "open"` and `priority >= 3`.

```python
tickets = [
    {"id": 1, "status": "open", "priority": 5, "assignee": "alice"},
    {"id": 2, "status": "closed", "priority": 4, "assignee": "bob"},
    {"id": 3, "status": "open", "priority": 2, "assignee": "alice"},
    {"id": 4, "status": "open", "priority": 4, "assignee": "charlie"},
]
```

**Approach.** Single-pass filter; use `.get()` with defaults in case fields are missing.

**Solution:**

```python
def filter_tickets(tickets: list[dict]) -> list[dict]:
    return [
        t for t in tickets
        if t.get("status") == "open" and t.get("priority", 0) >= 3
    ]
```

**Follow-up.** *"What if there are 100k tickets and you need it fast?"* → Already O(n); can't beat linear. If the filter runs repeatedly, pre-index by status.

---

#### Q2 · REST API call with Bearer auth *(Basic)*

**Problem.** Call a GET endpoint with a Bearer token. Return parsed JSON. Raise a clear exception on 4xx/5xx.

**Approach.** `requests.get`, auth header, `raise_for_status()`, explicit timeout.

**Solution:**

```python
import requests

def fetch_json(url: str, token: str, timeout: float = 10.0) -> dict:
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json",
    }
    response = requests.get(url, headers=headers, timeout=timeout)
    response.raise_for_status()
    return response.json()
```

**Notes.** The timeout is critical — the default is infinite, which will hang forever. `raise_for_status()` is the idiomatic way to turn a bad status into an exception.

**Follow-up.** *"What if the server sometimes returns 429?"* → Pair with retry logic (Q3).

---

#### Q3 · Retry with exponential backoff *(Medium)*

**Problem.** Wrap an API call with retries. Retry on 429 and 5xx. Exponential backoff with jitter. Max 5 attempts. Raise if all attempts fail.

**Approach.** Loop up to N times. On retryable status (429, 5xx) or timeout, sleep `2**attempt + jitter`. On terminal status (4xx non-429), raise immediately.

**Solution:**

```python
import time
import random
import requests

RETRYABLE_STATUSES = {429, 500, 502, 503, 504}

def fetch_with_retry(url: str, token: str, max_attempts: int = 5) -> dict:
    headers = {"Authorization": f"Bearer {token}"}
    last_error = None
    for attempt in range(max_attempts):
        try:
            response = requests.get(url, headers=headers, timeout=10)
            if response.status_code in RETRYABLE_STATUSES:
                last_error = requests.HTTPError(f"Retryable status {response.status_code}")
                if attempt < max_attempts - 1:
                    delay = (2 ** attempt) + random.uniform(0, 1)
                    time.sleep(delay)
                    continue
            response.raise_for_status()
            return response.json()
        except requests.Timeout as e:
            last_error = e
            if attempt == max_attempts - 1:
                raise
            time.sleep((2 ** attempt) + random.uniform(0, 1))
    raise RuntimeError(f"Failed after {max_attempts} attempts: {last_error}")
```

**Notes.** Only retry transient errors. Don't retry 401 or 404 — those won't recover. Jitter prevents thundering-herd behavior when many clients retry at once.

**Follow-up.** *"What about 503 with a `Retry-After` header?"* → Honor the header if present, overriding the exponential calc.

---

#### Q4 · Pagination *(Medium)*

**Problem.** An API returns `{"data": [...], "next_cursor": "..."}` (or `"next_cursor": null` when done). Fetch all pages.

**Approach.** Loop until the cursor is null. Accumulate results. For huge datasets, yield instead.

**Solution:**

```python
import requests

def fetch_all_pages(base_url: str, token: str) -> list[dict]:
    headers = {"Authorization": f"Bearer {token}"}
    results = []
    cursor = None
    while True:
        url = base_url if cursor is None else f"{base_url}?cursor={cursor}"
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()
        payload = response.json()
        results.extend(payload.get("data", []))
        cursor = payload.get("next_cursor")
        if cursor is None:
            break
    return results
```

**Follow-up.** *"What if there are 10 million records?"* → Stream as a generator instead of accumulating:

```python
def stream_all_pages(base_url: str, token: str):
    headers = {"Authorization": f"Bearer {token}"}
    cursor = None
    while True:
        url = base_url if cursor is None else f"{base_url}?cursor={cursor}"
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()
        payload = response.json()
        for item in payload.get("data", []):
            yield item
        cursor = payload.get("next_cursor")
        if cursor is None:
            return
```

---

#### Q5 · Dedupe keeping the latest *(Medium)*

**Problem.** Given a list of events with `event_id` and `timestamp`, return only the latest version per `event_id`.

```python
events = [
    {"event_id": "A", "timestamp": 1000, "value": "old"},
    {"event_id": "B", "timestamp": 2000, "value": "x"},
    {"event_id": "A", "timestamp": 3000, "value": "new"},
]
# Expected: [{"event_id": "A", ..., "value": "new"}, {"event_id": "B", ...}]
```

**Approach.** Dict keyed on `event_id`; keep the entry with max timestamp. Single pass, O(n).

**Solution:**

```python
def dedupe_by_latest(events: list[dict]) -> list[dict]:
    latest = {}
    for event in events:
        key = event["event_id"]
        if key not in latest or event["timestamp"] > latest[key]["timestamp"]:
            latest[key] = event
    return list(latest.values())
```

**Follow-up.** *"What if events arrive out of order in a stream?"* → Same logic applies incrementally — maintain the dict as events arrive.

---

#### Q6 · Flatten nested dict *(Medium)*

**Problem.** Flatten a nested dict into dotted-key format.

```python
# Input:
{"user": {"name": "Alice", "address": {"city": "Chicago"}}, "active": True}
# Output:
{"user.name": "Alice", "user.address.city": "Chicago", "active": True}
```

**Approach.** Recursion. For each key, if the value is a dict, recurse with an extended prefix.

**Solution:**

```python
def flatten(d: dict, prefix: str = "") -> dict:
    result = {}
    for key, value in d.items():
        new_key = f"{prefix}.{key}" if prefix else key
        if isinstance(value, dict):
            result.update(flatten(value, new_key))
        else:
            result[new_key] = value
    return result
```

**Follow-up.** *"What about lists of dicts?"* → Index into the list: `"items.0.name"`. Add an `isinstance(value, list)` branch that recurses with `f"{new_key}.{i}"`.

---

#### Q7 · Group and aggregate *(Medium)*

**Problem.** Given a list of orders, return total revenue per customer.

```python
orders = [
    {"customer_id": 1, "amount": 50.0},
    {"customer_id": 2, "amount": 30.0},
    {"customer_id": 1, "amount": 20.0},
    {"customer_id": 3, "amount": 100.0},
]
# Expected: {1: 70.0, 2: 30.0, 3: 100.0}
```

**Approach.** `defaultdict(float)`, iterate and accumulate.

**Solution:**

```python
from collections import defaultdict

def total_per_customer(orders: list[dict]) -> dict[int, float]:
    totals = defaultdict(float)
    for order in orders:
        totals[order["customer_id"]] += order["amount"]
    return dict(totals)
```

**Notes.** `defaultdict` removes the check-then-set boilerplate. Return a plain `dict` to avoid surprising callers who don't expect a `defaultdict` back.

**Follow-up.** *"What if orders include refunds (negative amounts)?"* → Same logic naturally handles them. Filter first if you specifically want gross vs. net.

---

#### Q8 · Validate a webhook payload with Pydantic *(Medium)*

**Problem.** An incoming webhook has this shape. Validate it — required fields, correct types, enum constraint on `event_type`.

```python
{
    "event_type": "lead_created",  # one of: lead_created, lead_updated, lead_deleted
    "timestamp": "2026-04-22T14:30:00Z",
    "payload": {
        "lead_id": 12345,
        "email": "alice@example.com",
        "tags": ["hot", "referral"]
    }
}
```

**Approach.** Two Pydantic models. `Literal` for the closed enum; `datetime` auto-coerces from ISO-8601.

**Solution:**

```python
from datetime import datetime
from typing import Literal
from pydantic import BaseModel, Field

class Payload(BaseModel):
    lead_id: int
    email: str
    tags: list[str] = Field(default_factory=list)

class Webhook(BaseModel):
    event_type: Literal["lead_created", "lead_updated", "lead_deleted"]
    timestamp: datetime
    payload: Payload

# Usage:
# webhook = Webhook.model_validate(incoming_json)
# webhook.payload.lead_id  # typed int
```

**Follow-up.** *"What if email needs real format validation?"* → Use `EmailStr` from Pydantic. *"What if tags should be unique?"* → Add a `@field_validator` that converts to a set and back.

---

#### Q9 · LLM output with retry-on-invalid *(Harder)*

**Problem.** Call an LLM, expect JSON output matching a Pydantic schema. If the output is invalid, retry with a prompt that includes the error. Max 3 attempts.

**Approach.** Call LLM → parse + validate → on failure, feed the error back into the next prompt. This is the standard reliability pattern for non-deterministic LLM output.

**Solution:**

```python
import json
from pydantic import BaseModel, ValidationError

class Classification(BaseModel):
    category: str
    confidence: float

def call_llm(prompt: str) -> str:
    # Stub — replace with your LLM client call
    ...

def classify_with_retry(text: str, max_attempts: int = 3) -> Classification:
    base_prompt = (
        f"Classify this text. Return JSON matching this schema: "
        f"{Classification.model_json_schema()}\n\nText: {text}"
    )
    prompt = base_prompt
    last_error = None
    for attempt in range(max_attempts):
        raw = call_llm(prompt)
        try:
            return Classification.model_validate(json.loads(raw))
        except (json.JSONDecodeError, ValidationError) as e:
            last_error = e
            prompt = (
                f"{base_prompt}\n\nYour previous output was:\n{raw}\n\n"
                f"It failed validation with: {e}\n"
                f"Return valid JSON only, matching the schema exactly."
            )
    raise ValueError(
        f"LLM failed to produce valid output after {max_attempts} attempts: {last_error}"
    )
```

**Notes.** The feed-the-error-back pattern works well for simple validation failures. Production version: log every attempt (prompt, response, error), track cost per attempt, cap total budget.

**Follow-up.** *"What if the LLM is expensive and you can't afford retries?"* → Use native structured-output APIs (OpenAI tool calling, Anthropic tool use) — the model guarantees schema at decode time. Retries become a fallback, not the norm.

---

#### Q10 · Simple rate limiter *(Harder)*

**Problem.** Implement a rate limiter — max N requests per M seconds. Block the caller (sleep) if over the limit, or optionally return False.

**Approach.** Sliding window with a deque of timestamps. On each acquire: drop expired timestamps, check count, either admit or wait.

**Solution:**

```python
import time
from collections import deque

class RateLimiter:
    def __init__(self, max_requests: int, per_seconds: float):
        self.max_requests = max_requests
        self.window = per_seconds
        self.timestamps: deque[float] = deque()

    def acquire(self, block: bool = True) -> bool:
        now = time.monotonic()
        # Drop timestamps outside the window
        while self.timestamps and self.timestamps[0] <= now - self.window:
            self.timestamps.popleft()

        if len(self.timestamps) >= self.max_requests:
            if not block:
                return False
            sleep_for = self.timestamps[0] + self.window - now
            time.sleep(max(sleep_for, 0))
            return self.acquire(block=True)

        self.timestamps.append(now)
        return True

# Usage:
# limiter = RateLimiter(max_requests=100, per_seconds=60)
# for item in items:
#     limiter.acquire()
#     call_api(item)
```

**Notes.** `time.monotonic()` is correct for elapsed-time measurements — `time.time()` is wall-clock and can jump backward on NTP adjustments.

**Follow-up.** *"What if you have multiple processes?"* → Move state to Redis (`INCR` + `EXPIRE` on a per-window key) or a shared service. In-memory per-process limiters each enforce their own quota.

---

### Stretch problems

Higher difficulty. Attempt only after Q1–Q10 are solid. Q11 especially is the kind of problem that pops up in integration-focused rounds.

---

#### Q11 · Webhook signature verification *(Harder)*

**Problem.** An incoming webhook arrives with an `X-Hub-Signature-256` header — an HMAC-SHA256 of the raw body, signed with a shared secret. Verify the signature before processing.

```
POST /webhook
X-Hub-Signature-256: sha256=abc123...
Content-Type: application/json

{"event": "contact.created", "contact_id": 12345}
```

**Approach.** Recompute HMAC-SHA256 over the raw body. Compare with `hmac.compare_digest` for constant-time comparison (prevents timing attacks).

**Solution:**

```python
import hmac
import hashlib

def verify_webhook_signature(raw_body: bytes, signature_header: str, secret: str) -> bool:
    if not signature_header.startswith("sha256="):
        return False
    provided = signature_header.removeprefix("sha256=")
    expected = hmac.new(
        key=secret.encode("utf-8"),
        msg=raw_body,
        digestmod=hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(provided, expected)

# Usage:
# if not verify_webhook_signature(request.body, request.headers["X-Hub-Signature-256"], WEBHOOK_SECRET):
#     return 401
```

**Notes.** Three subtleties: (1) verify against **raw bytes**, not parsed-then-reserialized JSON (byte-level differences change the HMAC); (2) use `hmac.compare_digest`, not `==` (constant-time, guards against timing attacks); (3) strip the literal `sha256=` prefix.

**Follow-up.** *"What if the secret is rotated?"* → Maintain a list of current-plus-previous secrets; accept the signature if any of them match.

---

#### Q12 · Async API calls with `asyncio.gather` *(Harder)*

**Problem.** Fetch 100 URLs concurrently. Return a list of results. Per-URL failures shouldn't fail the whole batch.

**Approach.** `aiohttp` session + `asyncio.gather(..., return_exceptions=True)` so failures come back as return values, not raised exceptions.

**Solution:**

```python
import asyncio
import aiohttp

async def fetch_one(session: aiohttp.ClientSession, url: str) -> dict | Exception:
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as resp:
            resp.raise_for_status()
            return await resp.json()
    except Exception as e:
        return e

async def fetch_many(urls: list[str]) -> list:
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_one(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# Usage:
# results = asyncio.run(fetch_many(["https://api.example.com/1", ...]))
# for r in results:
#     if isinstance(r, Exception):
#         log_failure(r)
#     else:
#         process(r)
```

**Notes.** Returning exceptions (rather than raising) preserves partial success — you process what succeeded, log what didn't. One session per batch amortizes TCP handshakes. For very large batches, add `asyncio.Semaphore(20)` to cap concurrency.

**Follow-up.** *"What if the server's rate limit is 10 req/sec?"* → Wrap each task with a semaphore-backed rate limiter, or combine with Q10's sliding-window approach adapted for async.

---

#### Q13 · Simple state machine *(Medium-Hard)*

**Problem.** Model a ticket lifecycle with states: `open → in_progress → resolved → closed`. Also allow `open → closed` (rejection) and `resolved → in_progress` (reopen). Reject invalid transitions with a clear error.

**Approach.** Dict of allowed transitions per state; validate on every transition; track history.

**Solution:**

```python
VALID_TRANSITIONS: dict[str, set[str]] = {
    "open": {"in_progress", "closed"},
    "in_progress": {"resolved"},
    "resolved": {"closed", "in_progress"},
    "closed": set(),  # terminal
}

class Ticket:
    def __init__(self, ticket_id: str):
        self.ticket_id = ticket_id
        self.state = "open"
        self.history: list[tuple[str, str]] = []

    def transition(self, new_state: str) -> None:
        allowed = VALID_TRANSITIONS.get(self.state, set())
        if new_state not in allowed:
            raise ValueError(
                f"Invalid transition {self.state} → {new_state} "
                f"for ticket {self.ticket_id}. Allowed: {sorted(allowed)}"
            )
        self.history.append((self.state, new_state))
        self.state = new_state
```

**Notes.** Dict-of-sets is the simplest correct implementation. History tracking is cheap and invaluable for audit. For complex state machines with side-effects-per-transition, move to a library (`transitions`, `statemachine`).

**Follow-up.** *"What if two processes try to transition the same ticket concurrently?"* → Optimistic locking: include the current state in the UPDATE's WHERE clause (`WHERE state = 'open'`). If no rows update, someone else won the race.

---

### HubSpot integration problems

Calibrated to the HubSpot depth on your resume. Given your work history, expect 1–2 of these in the round if the interviewers skim your experience first.

---

#### Q14 · HubSpot contact upsert by email *(Medium)*

**Problem.** Given a contact dict, create it in HubSpot. If a contact with that email already exists (409 Conflict), update the existing one instead.

**Approach.** POST to `/crm/v3/objects/contacts`. On 409, extract the existing ID from the error body, then PATCH.

**Solution:**

```python
import requests

HUBSPOT_BASE = "https://api.hubapi.com"

def upsert_contact(contact: dict, token: str) -> dict:
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    resp = requests.post(
        f"{HUBSPOT_BASE}/crm/v3/objects/contacts",
        headers=headers,
        json={"properties": contact},
        timeout=10,
    )
    if resp.status_code == 409:
        existing_id = resp.json().get("message", "").split("Existing ID: ")[-1].strip()
        resp = requests.patch(
            f"{HUBSPOT_BASE}/crm/v3/objects/contacts/{existing_id}",
            headers=headers,
            json={"properties": contact},
            timeout=10,
        )
    resp.raise_for_status()
    return resp.json()
```

**Notes.** HubSpot also supports a native upsert via `idProperty=email` on create — mention this if the interviewer hasn't. The error-parsing fallback shown here is what you reach for when the API doesn't support native upsert.

**Follow-up.** *"What about a race where two upserts fire for the same email at once?"* → Both hit the create path; one succeeds, the loser gets 409 and falls through to PATCH. Idempotent as long as the PATCH is deterministic.

---

#### Q15 · HubSpot batch create with chunking *(Medium)*

**Problem.** HubSpot's batch endpoint accepts up to 100 records per call. You have 500 contacts to create. Do it efficiently.

**Approach.** Chunk into groups of 100. POST each chunk. Collect results.

**Solution:**

```python
import requests

def chunked(items: list, size: int):
    for i in range(0, len(items), size):
        yield items[i:i + size]

def batch_create_contacts(contacts: list[dict], token: str) -> list[dict]:
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    results = []
    for chunk in chunked(contacts, 100):
        resp = requests.post(
            f"{HUBSPOT_BASE}/crm/v3/objects/contacts/batch/create",
            headers=headers,
            json={"inputs": [{"properties": c} for c in chunk]},
            timeout=30,
        )
        resp.raise_for_status()
        results.extend(resp.json().get("results", []))
    return results
```

**Notes.** Timeout bumped to 30s — batch calls take longer than singles. The `chunked` helper is a pattern worth memorizing; you'll reuse it for any batch API. For 10k+ records, add the Q3 retry wrapper per chunk.

**Follow-up.** *"What if one chunk fails — do you roll back the others?"* → HubSpot batch operations are **not transactional**. You get partial success. Either surface a list of failed records for retry, or add compensation logic if rollback is actually required.

---

#### Q16 · Create HubSpot associations with retries *(Medium)*

**Problem.** Given a list of `(contact_id, deal_id)` pairs, create associations in HubSpot. Retry individual failures on transient errors. Return a summary of successes and failures.

**Approach.** Loop per pair. Wrap each call with the Q3 retry logic. Collect outcomes.

**Solution:**

```python
import requests

def associate_contact_to_deal(
    contact_id: str,
    deal_id: str,
    token: str,
) -> bool:
    headers = {"Authorization": f"Bearer {token}"}
    url = (
        f"{HUBSPOT_BASE}/crm/v4/objects/contact/{contact_id}"
        f"/associations/default/deal/{deal_id}"
    )
    resp = requests.put(url, headers=headers, timeout=10)
    resp.raise_for_status()
    return True

def associate_batch(pairs: list[tuple[str, str]], token: str) -> dict:
    successes, failures = [], []
    for contact_id, deal_id in pairs:
        try:
            associate_contact_to_deal(contact_id, deal_id, token)
            successes.append((contact_id, deal_id))
        except Exception as e:
            failures.append({"pair": (contact_id, deal_id), "error": str(e)})
    return {"successes": successes, "failures": failures}
```

**Notes.** PUT is used here, not POST — HubSpot's association endpoint is PUT because the operation is idempotent. For higher volumes, use the batch-associate endpoint (`/crm/v4/associations/contact/deal/batch/create`).

**Follow-up.** *"What if the same pair is already associated?"* → Usually a 2xx (idempotent by design) or 409 (treat as success). That's a feature, not a bug — you can re-run safely.

---

#### Q17 · Pull HubSpot contacts modified in the last 24h *(Medium-Hard)*

**Problem.** Use HubSpot's search API to find all contacts where `hs_lastmodifieddate` is within the last 24 hours. Handle pagination. Return all matching contacts.

**Approach.** POST to `/crm/v3/objects/contacts/search` with a filter on `hs_lastmodifieddate`. HubSpot uses cursor-based pagination via `after`.

**Solution:**

```python
import requests
from datetime import datetime, timedelta, timezone

def fetch_recently_modified_contacts(token: str) -> list[dict]:
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    }
    cutoff_ms = int((datetime.now(timezone.utc) - timedelta(hours=24)).timestamp() * 1000)
    results = []
    after = None

    while True:
        body = {
            "filterGroups": [{
                "filters": [{
                    "propertyName": "hs_lastmodifieddate",
                    "operator": "GTE",
                    "value": cutoff_ms,
                }]
            }],
            "sorts": [{"propertyName": "hs_lastmodifieddate", "direction": "ASCENDING"}],
            "limit": 100,
        }
        if after:
            body["after"] = after

        resp = requests.post(
            f"{HUBSPOT_BASE}/crm/v3/objects/contacts/search",
            headers=headers,
            json=body,
            timeout=15,
        )
        resp.raise_for_status()
        payload = resp.json()
        results.extend(payload.get("results", []))
        after = payload.get("paging", {}).get("next", {}).get("after")
        if not after:
            break

    return results
```

**Notes.** Three gotchas interviewers love to probe: (1) HubSpot uses **millisecond** timestamps, not seconds; (2) sorting by the filter field is the safe pagination strategy — otherwise new records inserted mid-pagination shift the window; (3) the search API caps at **10,000 results** — know this limit.

**Follow-up.** *"What if there are more than 10,000 modified contacts?"* → Partition the time window. Run the same search for 6-hour windows, merge the results.

---

#### Q18 · HubSpot webhook handler with signature + routing *(Harder)*

**Problem.** Write a webhook handler that:
1. Verifies the HubSpot signature.
2. Parses and validates the payload.
3. Routes to specific handlers based on `subscriptionType`.
4. Returns 401 on bad signature, 400 on unknown event, 200 on success.

**Approach.** Compose Q11 (signature verification) + Q8 (Pydantic validation) + dict-based dispatch.

**Solution:**

```python
import hmac
import hashlib
import json
from typing import Callable, Literal
from pydantic import BaseModel

HUBSPOT_SECRET = "your-secret"

class HubSpotEvent(BaseModel):
    subscriptionType: Literal[
        "contact.creation",
        "contact.propertyChange",
        "deal.creation",
    ]
    objectId: int
    propertyName: str | None = None
    propertyValue: str | None = None

def verify_hubspot_signature(raw_body: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        key=secret.encode(),
        msg=raw_body,
        digestmod=hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(signature, expected)

def handle_contact_creation(event: HubSpotEvent) -> None: ...
def handle_contact_property_change(event: HubSpotEvent) -> None: ...
def handle_deal_creation(event: HubSpotEvent) -> None: ...

HANDLERS: dict[str, Callable[[HubSpotEvent], None]] = {
    "contact.creation": handle_contact_creation,
    "contact.propertyChange": handle_contact_property_change,
    "deal.creation": handle_deal_creation,
}

def process_webhook(raw_body: bytes, signature: str) -> tuple[int, str]:
    if not verify_hubspot_signature(raw_body, signature, HUBSPOT_SECRET):
        return 401, "invalid signature"

    try:
        events = [HubSpotEvent.model_validate(e) for e in json.loads(raw_body)]
    except Exception as e:
        return 400, f"invalid payload: {e}"

    for event in events:
        handler = HANDLERS.get(event.subscriptionType)
        if handler is None:
            return 400, f"unknown event: {event.subscriptionType}"
        handler(event)

    return 200, "ok"
```

**Notes.** HubSpot sends webhooks as **arrays**, not single events — easy to miss. Dict-based dispatch is cleaner than `if/elif` chains; adding a new event type is "add handler + register it," not "find the right place in a chain."

**Follow-up.** *"What if one handler fails — do the others still run?"* → Depends on your retry guarantees. If HubSpot retries the whole batch on a non-2xx response, fail fast. If you need at-least-once per event, wrap each handler in try/except and queue failures for retry independently.

---

### Narration patterns while coding

Interviewers grade thinking as much as output. These scripted phrases give you something to say when your brain is working on the code.

#### Opening narration (first 30 seconds, before typing)

**The template:**
> *"Okay, so I'm reading the problem. [Restate in your own words]. I want to clarify — [ask 1–2 things]. My approach is going to be [high-level approach] using [data structure / library]. Time complexity will be [O(n) / O(n log n) / whatever]. Let me code it up."*

**Concrete examples by problem type:**

- **Filter/transform (Q1):** *"This is a simple filter. List comprehension; two conditions become the if clause. Using `.get()` for safety in case any ticket is missing fields."*
- **API call (Q2):** *"Standard `requests` call. Bearer in the Authorization header, explicit timeout so it can't hang, `raise_for_status` to convert 4xx/5xx to exceptions."*
- **Retry (Q3):** *"Loop up to N attempts. Only retry on transient — 429 and 5xx. Other 4xx is terminal, raise immediately. Exponential backoff with jitter to avoid thundering herd."*
- **Pagination (Q4):** *"Loop until cursor is null. Accumulate into a list for now; if you mention huge datasets I'll switch to a generator."*
- **Dedup/group (Q5, Q7):** *"Single-pass dict approach keyed on whatever identifies the record. O(n), no sort needed."*
- **Recursion (Q6):** *"Recursive with a prefix accumulator. Base case is when the value isn't a dict."*
- **Pydantic (Q8):** *"Two models since it's nested. `Literal` for the closed enum. Pydantic coerces the ISO-8601 string into `datetime` automatically."*
- **LLM + retry (Q9):** *"Call → parse-and-validate → on failure feed the error into the next prompt. The pattern is that the system around the LLM is deterministic even if the LLM isn't."*
- **Rate limiter (Q10):** *"Sliding-window with a deque of timestamps. `time.monotonic` for elapsed time."*
- **Signature (Q11):** *"Recompute HMAC over the raw bytes, compare with `compare_digest` for constant-time. Three places to be careful — raw bytes not parsed, constant-time compare, strip the literal prefix."*
- **Async (Q12):** *"`aiohttp` session, `gather` with `return_exceptions=True` so failures come back instead of blowing up the batch."*

#### Mid-code narration (while typing)

One short phrase per chunk:

- *"Using `.get()` here because this field might be missing."*
- *"Explicit timeout — default is infinite, which would hang."*
- *"Handling the empty-input edge case."*
- *"`defaultdict` so I don't have to check-then-set."*
- *"`compare_digest` rather than `==` — constant-time, guards against timing attacks."*
- *"Yielding instead of accumulating so this doesn't blow memory on large inputs."*

#### Clarifying question phrasing

Short. Don't stall for five questions — pick one or two that genuinely shape the solution:

- *"Just to confirm — is the input always valid JSON, or should I handle malformed input?"*
- *"Should I assume order matters in the output?"*
- *"Expected input size — hundreds, millions?"*
- *"On a 5xx, do we retry or fail fast?"*
- *"Are duplicates possible, and if so, which one wins?"*

#### Tradeoff narration (before moving on)

Name what you're not doing and why:

- *"A simpler version would be [X], but that doesn't handle [Y]."*
- *"For higher volume, I'd move this to async."*
- *"This is O(n log n) because of the sort; if I didn't need sorted output I could do O(n) with a set."*
- *"Sync here; in production I'd wrap this in a queue."*

#### Testing narration

Even a trivial trace is worth it:

- *"Let me trace through with a simple case — three tickets, one open, one closed."*
- *"Quick sanity check — with zero inputs, what should we return? Empty list, let me make sure that works."*

#### Stuck recovery

When you blank or hit a bug:

- *"Let me step back and reconsider."*
- *"I'll come back to this — first let me get the happy path working, then handle this edge case."*
- *"I'm going to try a different approach — the one I started is getting complicated."*
- *"Syntax issue, let me check the docs on that quickly."* — asking for a quick check is fine; pretending you know and silently floundering isn't.

#### Ending narration

When you think you're done:

- *"Okay, I think that's the core logic. Let me quickly trace through once to make sure."*
- *"Things I'd add for production: retry logging, metrics, a test suite."*
- *"Happy path is solid; the edge case I'd want to think more about is [X]."*

Owning what's incomplete is better than pretending it's complete.

---

### How to practice these

- **First pass:** write each solution without looking. Time yourself — 15 min per Q1–Q10, 25 min per Q11–Q18.
- **Second pass:** read the solutions here, diff against your version, note what you missed.
- **Third pass (day before):** rewrite the 3–4 you found hardest from scratch.
- **If you get all 18 solid:** you're well above the bar for this round.

Don't memorize — understand the patterns. Interviewers reshape these into slightly different problems; recognizing the underlying shape is what matters.

### What to avoid

- Silent 30-sec pauses — think out loud
- Over-engineering — simplest correct first, iterate
- Importing unfamiliar libraries to show off
- Getting stuck debugging syntax when the algorithm matters — ask for a hint
- Writing code before narrating the approach — interviewers judge thinking as much as output

---

## 8. Questions to Ask the Technical Panel

Different from HM questions. Technical panel wants evidence you think like a builder, not a passer.

**Strong options (pick 2–3):**

- *"What's the hardest technical problem on the team's plate right now?"*
- *"How is the Power Platform deployment set up — solutions, environment strategy, pipelines? What's working, what's painful?"*
- *"How are you handling AI governance right now — is there a responsible-AI framework in place?"*
- *"What's the testing story for automation flows — unit, integration, eval sets for AI steps?"*
- *"What does on-call / production support look like for the flows the team owns?"*
- *"Is there a piece of the stack the team considers their biggest tech debt?"*

**The "hardest problem" and "tech debt" questions are the strongest** — they position you as someone thinking about working there, not just passing the round.

**Questions to skip:**
- Comp, benefits, PTO (recruiter scope)
- Anything the HM already answered
- Generic "what's the culture like" (shallow)

---

## 9. Red Flags — Ranked by Severity

### High severity
- **Overclaiming Power Automate depth.** If pressed on a feature you don't know: *"I haven't hit that specific case — I'd check the docs before committing."* Better than bluffing.
- **Contradicting your resume or the HM's framing.** Know what your resume says. Know what the HM heard.
- **Dismissing n8n in favor of PA.** Your production work is n8n. The cross-platform framing is your strength — respect both tools.
- **Skipping scope on design exercises.** Jumping to implementation without asking about volume, latency, or constraints is a junior tell.

### Medium severity
- **Too many "I think" / "I guess" / "probably."** Take a beat, commit.
- **Missing idempotency / retries / error handling** when they'd obviously apply. Mid-to-senior table stakes.
- **Features vs patterns.** "There's a connector for that" vs "this is a claim-check pattern because the payload is too big for the queue."
- **Monologue answers > 2 minutes.** Pause, invite follow-up.

### Low severity but annoying
- **Buzzwords without substance.** "Event-driven" is fine; "event-driven orchestration at scale" without backing it is fluff.
- **Skipping your own questions at the end.** Even if out of time — ask one.

---

## 10. Mid-Round Recovery Moves

If you realize you're losing the round, these moves work:

- **Froze on a question** → *"Let me take that from a different angle."* Reset with a simpler framing. Interviewers respect the reset more than a half-answer.
- **Realized you gave a wrong answer** → name it. *"Actually, let me correct something I said earlier — X should be Y, because..."* Shows self-awareness.
- **Question is out of your depth** → honest line + related thing you do know. *"I haven't worked with [X] directly; the closest is [Y], and the transferable pattern is [Z]."*
- **Running out of time** → *"I want to make sure we hit what you wanted to cover — is there anything you still need from me?"*

Recovering from a bad answer is a senior signal. Pretending a bad answer was good is a junior one.

---

## 11. Day-Of Checklist

**One hour before:**
- [ ] Re-read the 30-sec Lead Routing verbal
- [ ] Skim Lead Routing Guide Section 11 (the 12 Q&As)
- [ ] Tech check: camera, mic, internet, backup (phone tethering)
- [ ] Water, notepad, pen
- [ ] Close Slack / email notifications

**Five minutes before:**
- [ ] 30 seconds of slow breathing
- [ ] Open your resume in a side window (reference, not read-from)
- [ ] Have your "questions to ask" list visible

**Right after:**
- [ ] 10 min writing down what they asked, what you answered, what you struggled with — useful for the next round and the follow-up email
- [ ] Thank-you email to the interviewers (if you got names). Short, specific: *"Appreciated the question on [topic]."*

---

## 12. The Bottom Line

Unfair advantages going in:
- UPS internal history
- HM-advocated referral
- Real production automation experience (n8n, but patterns are universal)
- Strong API / backend foundation
- Recent Python / AI work

The technical round is yours to lose. Walk in calm, not swaggering. The questions that could trip you are almost all on Power Automate specifics — the Ramp Plan and Lead Routing Guide cover those. Drill those two artifacts, walk in confident, ask good questions back.

Final reminder: the interviewer is evaluating whether they want to work with you. Be the person they'd want next to them at 4pm on a Friday when a flow breaks in prod.
