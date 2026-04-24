# Azure OpenAI — Getting Started, End-to-End

**Audience:** you, prepping for the Senior AI Hyper Automation technical round at UPS Healthcare / Marken.
**Time budget:** 3–5 hours of hands-on work total, done as a single sitting.
**Outcome:** you will have provisioned Azure OpenAI, deployed a model, called it from Python / Logic Apps / Power Automate with Managed Identity, handled structured output, and understand private endpoints / cost levers / observability well enough to discuss them in an enterprise interview.

**Companion to:** `Senior_AI_Hyper_Automation_Interview_Prep_QuestionBank.md`, `Power_Platform_Logic_Apps_Ramp_Plan.md`, `Lead_Routing_System_Technical_Guide.md`.

---

## 0. Before you start

### 0.1 Prerequisites
- **Azure subscription.** If you don't have one, create a free account (https://azure.microsoft.com/free — $200 credit). A personal subscription is fine for this. Do **not** use your UPS corporate subscription for practice.
- **Credit card on file** (even for free tier — Azure requires it). You won't be charged if you stay within free credits and clean up after.
- **Access to Azure OpenAI.** As of late 2025 / early 2026, Azure OpenAI is generally available without the access-request gate that used to be required. If you hit a "request access" screen, fill out the short form; approvals typically come same-day. Some premium models (o1, specific preview models) still have per-model gating.

### 0.2 Cost expectations for this walkthrough
- `gpt-4o-mini` is the cheapest capable model. ~$0.15 per 1M input tokens, ~$0.60 per 1M output tokens (pricing changes; check Azure's pricing page).
- The full hands-on in this guide should cost under $1 if you don't loop-call it.
- **Set a budget alert** (Section 12) before you start.

### 0.3 Tools you'll need locally
- **Azure CLI**: `brew install azure-cli` on Mac.
- **Python 3.10+** with `pip`.
- **VS Code** with the Azure Account and Azure Resources extensions.
- A terminal.

### 0.4 Sign in once
```bash
az login
az account show   # verify subscription
az account set --subscription "<subscription-name-or-id>"   # if you have multiple
```

---

## 1. Create an Azure OpenAI resource

### 1.1 Portal walkthrough
1. Go to https://portal.azure.com
2. Top search bar → type **"Azure OpenAI"** → click the service.
3. Click **+ Create**.
4. Fill the form:
   - **Subscription**: your subscription.
   - **Resource group**: click **Create new**, name it `rg-aoai-lab`. Resource groups are logical containers; deleting the group deletes everything inside, which is how you'll clean up later.
   - **Region**: **East US** or **East US 2** — these have the widest model availability, including `gpt-4o-mini` and `gpt-4o`. Avoid regions with limited model lists for a lab.
   - **Name**: `aoai-<yourname>-lab` (must be globally unique). Example: `aoai-nimesh-lab`.
   - **Pricing tier**: **Standard S0**. (There's no free tier for Azure OpenAI; S0 is pay-per-use.)
5. Click **Next: Network**.
   - For this lab: **All networks** (public access). In production / regulated scenarios you'd choose **Selected networks** with Private Endpoint — we'll cover the concept in Section 10.
6. Click **Next: Tags** (skip), then **Next: Review + submit**.
7. Click **Create**. Deployment takes 1–2 minutes.

### 1.2 Verify via CLI (optional, good practice)
```bash
az cognitiveservices account show \
  --name aoai-nimesh-lab \
  --resource-group rg-aoai-lab \
  --query "{name:name, location:location, endpoint:properties.endpoint}"
```

You should see JSON with your endpoint URL (something like `https://aoai-nimesh-lab.openai.azure.com/`). **Copy this endpoint** — you'll use it for every API call.

---

## 2. Deploy a model

You have a resource, but no model yet. The resource is the billing/governance container; a **model deployment** is the actual callable endpoint.

### 2.1 Open Azure AI Foundry (formerly AI Studio)
1. In the portal, go to your Azure OpenAI resource.
2. In the left pane, click **Go to Azure AI Foundry portal** (or navigate to https://ai.azure.com and pick your resource).
3. In Foundry: left pane → **Deployments** (under "Shared resources").

### 2.2 Deploy gpt-4o-mini
1. Click **+ Deploy model** → **Deploy base model**.
2. Select **gpt-4o-mini**. Click **Confirm**.
3. Fill the deployment form:
   - **Deployment name**: `gpt-4o-mini-lab`. **This name matters** — it's what you pass in API calls, not the model name itself. Common production convention: `<model>-<env>`, e.g., `gpt-4o-mini-prod`.
   - **Deployment type**: **Standard** (pay-per-token). (Other option is **Provisioned** — reserved throughput / PTU — for predictable latency at scale. Too expensive for a lab.)
   - **Tokens per Minute Rate Limit (TPM)**: leave default (usually 30K–50K for a new deployment). This is your quota; you can increase it later via the portal's **Quotas** blade.
   - **Content filter**: leave as default (this is the Azure Content Safety filter — hate, sexual, violence, self-harm, jailbreak). Production regulated workloads sometimes use custom filters.
4. Click **Deploy**. Takes ~30 seconds.

### 2.3 Mental model to lock in
- **Resource** = the Azure OpenAI instance (billing, networking, auth boundary).
- **Deployment** = a specific model running under the resource, addressed by your chosen deployment name.
- **API call URL** = `{endpoint}/openai/deployments/{deployment-name}/chat/completions?api-version=<version>`

This is the #1 confusion point for people coming from the public OpenAI API where the model name is the model. In Azure OpenAI, **you call the deployment, not the model.**

---

## 3. Sanity-check in the Playground

Before writing code, verify the deployment works.

1. In Foundry: left pane → **Playgrounds** → **Chat**.
2. Top right: select your deployment `gpt-4o-mini-lab`.
3. **System message**: paste this:
   > You are a clinical-logistics operations assistant. Respond tersely and only with verifiable facts; if you don't know, say so.
4. **User message**:
   > Explain idempotency in two sentences, in the context of a workflow that updates a shipment record on retry.
5. Click **Send**.

You should get a coherent 2-sentence response. If you do: deployment is live.

**Optional**: click **View code** in the top right of the playground. Foundry shows you the Python / JavaScript / curl equivalents of what you just ran — including your endpoint, API version, and deployment name. Keep this tab open; you'll copy-paste from it.

---

## 4. Grab your credentials

You need three values for any programmatic call:

1. **Endpoint** — `https://aoai-nimesh-lab.openai.azure.com/` (from Section 1).
2. **API key** — in the Azure portal, go to your Azure OpenAI resource → left pane → **Keys and Endpoint** → copy **KEY 1**.
3. **Deployment name** — `gpt-4o-mini-lab` (from Section 2).
4. **API version** — `2024-10-21` is a stable recent version at time of writing (check Microsoft Learn for latest GA). Preview versions like `2024-12-01-preview` have newer features.

**Do not paste the API key into code.** Store it via the environment, or — better — Key Vault (Section 5).

Create a local `.env` file (add to `.gitignore` immediately):
```
AZURE_OPENAI_ENDPOINT=https://aoai-nimesh-lab.openai.azure.com/
AZURE_OPENAI_API_KEY=<paste-your-key>
AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini-lab
AZURE_OPENAI_API_VERSION=2024-10-21
```

---

## 5. Store the key in Key Vault (the enterprise pattern)

This is the pattern Logic Apps and Power Automate will use later. Do it now.

### 5.1 Create a Key Vault
```bash
# Pick a globally unique name, 3-24 chars, lowercase + hyphens
az keyvault create \
  --name kv-aoai-nimesh \
  --resource-group rg-aoai-lab \
  --location eastus
```

### 5.2 Store the API key as a secret
```bash
az keyvault secret set \
  --vault-name kv-aoai-nimesh \
  --name aoai-api-key \
  --value "<paste-your-key>"
```

### 5.3 Grant yourself permission to read it

New Key Vaults default to RBAC-based access. Give yourself the data-plane role:
```bash
# Get your user object id
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Get the vault resource id
VAULT_ID=$(az keyvault show --name kv-aoai-nimesh --query id -o tsv)

# Grant the Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $USER_ID \
  --scope $VAULT_ID
```

### 5.4 Verify
```bash
az keyvault secret show \
  --vault-name kv-aoai-nimesh \
  --name aoai-api-key \
  --query value -o tsv
```

Should print your key. If you see a 403, wait 30 seconds for the role assignment to propagate and retry.

---

## 6. Call Azure OpenAI from Python

This is your strongest language — start here.

### 6.1 Install the SDK
```bash
mkdir aoai-lab && cd aoai-lab
python -m venv .venv
source .venv/bin/activate
pip install openai python-dotenv pydantic
```

### 6.2 Minimal chat completion

Create `hello.py`:
```python
import os
from dotenv import load_dotenv
from openai import AzureOpenAI

load_dotenv()

client = AzureOpenAI(
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
)

response = client.chat.completions.create(
    model=os.environ["AZURE_OPENAI_DEPLOYMENT"],   # deployment name, not model name
    messages=[
        {"role": "system", "content": "You respond in one short sentence."},
        {"role": "user", "content": "What is idempotency?"},
    ],
)

print(response.choices[0].message.content)
print(f"tokens: {response.usage.total_tokens}")
```

Run:
```bash
python hello.py
```

You should see a one-sentence answer and a token count. **If this works, you have production-shaped code.** The same `AzureOpenAI` client, same method signatures, work anywhere — Function Apps, Python-based APIs, CLI tools.

### 6.3 Structured output with Pydantic (this is your bullet)

Create `structured.py`:
```python
import os
import json
from dotenv import load_dotenv
from openai import AzureOpenAI
from pydantic import BaseModel, Field

load_dotenv()

client = AzureOpenAI(
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
)

class ShipmentExtraction(BaseModel):
    shipment_id: str = Field(description="The unique shipment identifier")
    origin_country: str
    destination_country: str
    temperature_required_c: float | None = Field(
        default=None,
        description="Required storage temperature in Celsius, if specified"
    )
    priority: str = Field(description="One of: standard, expedited, critical")

response = client.beta.chat.completions.parse(
    model=os.environ["AZURE_OPENAI_DEPLOYMENT"],
    messages=[
        {"role": "system", "content": "Extract shipment details from the user's message. If a field is not present, leave it null."},
        {"role": "user", "content": "Shipment SH-2029-A from Germany to the US. Needs to stay between 2 and 8 Celsius. Critical priority."},
    ],
    response_format=ShipmentExtraction,
)

extraction: ShipmentExtraction = response.choices[0].message.parsed
print(extraction.model_dump_json(indent=2))
```

Run it. You should see a validated JSON object. **This is the exact pattern you already claim on your resume** — structured output enforced via Pydantic. Now you have the Azure OpenAI version of that pattern, live.

### 6.4 What you can now honestly claim
> "I've deployed Azure OpenAI and built Python integrations using the AzureOpenAI client, including structured-output extraction with Pydantic response models. The client behavior and parse API mirror what I've used in model-agnostic contexts — the Azure-specific parts are resource/deployment naming and API version handling."

---

## 7. Call Azure OpenAI from Logic Apps (Consumption — simplest path)

This is the integration pattern you'll describe in the interview for "how do I put AI into a Logic Apps workflow."

### 7.1 Create a Consumption Logic App
1. Portal → search **Logic App** → click **Create**.
2. Plan type: **Consumption**. (Simpler than Standard for a lab; same API-call pattern.)
3. Resource group: `rg-aoai-lab`. Name: `la-aoai-lab`. Region: same as your OpenAI resource.
4. Review + create.

### 7.2 Build the workflow
1. Open the Logic App → **Logic app designer**.
2. **Trigger**: **When a HTTP request is received** → click **Use sample payload to generate schema**, paste:
   ```json
   { "user_message": "Extract the shipment details..." }
   ```
   Save. Azure gives you a callback URL.

3. **Add action** → **HTTP**.
   - **Method**: `POST`
   - **URI**: `https://aoai-nimesh-lab.openai.azure.com/openai/deployments/gpt-4o-mini-lab/chat/completions?api-version=2024-10-21`
   - **Headers**:
     - `Content-Type` → `application/json`
     - `api-key` → (we'll wire this to Key Vault in Section 9 via Managed Identity; for now, paste the key to verify the flow works)
   - **Body**:
     ```json
     {
       "messages": [
         { "role": "system", "content": "You extract fields from shipment descriptions. Respond with valid JSON only." },
         { "role": "user", "content": "@{triggerBody()?['user_message']}" }
       ],
       "temperature": 0,
       "response_format": { "type": "json_object" }
     }
     ```

4. **Add action** → **Parse JSON** → feed it the HTTP response body; use this schema:
   ```json
   {
     "type": "object",
     "properties": {
       "choices": {
         "type": "array",
         "items": {
           "type": "object",
           "properties": {
             "message": {
               "type": "object",
               "properties": { "content": { "type": "string" } }
             }
           }
         }
       }
     }
   }
   ```

5. **Add action** → **Response** → body: `@{body('Parse_JSON')?['choices']?[0]?['message']?['content']}`.

Save the Logic App.

### 7.3 Test it
From your terminal:
```bash
curl -X POST "<paste-the-http-trigger-url>" \
  -H "Content-Type: application/json" \
  -d '{"user_message":"Shipment SH-88 from India to UK, 2-8C, expedited."}'
```

You should see the model's JSON response come back. That's your first AI-in-Logic-Apps workflow. Screenshot it.

---

## 8. Call Azure OpenAI from Power Automate

### 8.1 Create the flow
1. https://make.powerautomate.com → **+ Create** → **Instant cloud flow**.
2. Trigger: **Manually trigger a flow**. Add an input called `user_message` (text).
3. **+ New step** → **HTTP** (premium connector).
   - **Method**: `POST`
   - **URI**: `https://aoai-nimesh-lab.openai.azure.com/openai/deployments/gpt-4o-mini-lab/chat/completions?api-version=2024-10-21`
   - **Headers**:
     - `Content-Type`: `application/json`
     - `api-key`: `<your-key>` (for lab; in prod use Key Vault connector — see 8.3)
   - **Body**:
     ```json
     {
       "messages": [
         { "role": "system", "content": "You extract shipment fields. Respond with JSON only." },
         { "role": "user", "content": "@{triggerBody()['text']}" }
       ],
       "temperature": 0,
       "response_format": { "type": "json_object" }
     }
     ```
4. **+ New step** → **Parse JSON** (schema same as Section 7.2 step 4).
5. **+ New step** → **Compose** → value: `@{body('Parse_JSON')?['choices']?[0]?['message']?['content']}`.

Save. Click **Test** → **Manually** → pass a shipment description → run. Verify you get structured JSON.

### 8.2 Alternative: Key Vault connector instead of hardcoded key
Replace the `api-key` header value with an expression that pulls from Key Vault via the **Azure Key Vault** connector (premium). The pattern:
- **Get secret** action (from Key Vault connector, authenticated with your Azure AD account).
- Feed the secret's value into the HTTP action's `api-key` header.

### 8.3 When to use AI Builder instead
Power Automate also has **AI Builder** — a no-code wrapper around specific AI models. Use it when business users own the flow and want drag-and-drop. Use HTTP + Azure OpenAI directly (what you just did) when you need full prompt control, specific models, or structured-output enforcement. Know both exist; the HTTP pattern is what seniors pick for custom work.

---

## 9. Managed Identity authentication (the enterprise way — eliminate the API key)

This is what you'll describe in the interview when asked "how would you secure the Azure OpenAI call in production?"

### 9.1 Enable system-assigned Managed Identity on the Logic App
1. Portal → your Logic App (`la-aoai-lab`) → **Identity** → **System assigned** → toggle **On** → **Save**.
2. Copy the **Object (principal) ID** that appears.

### 9.2 Grant the Logic App's identity access to Azure OpenAI
```bash
# Get the Logic App's principal id
LA_PRINCIPAL=$(az logic workflow show \
  --name la-aoai-lab \
  --resource-group rg-aoai-lab \
  --query identity.principalId -o tsv)

# Get the OpenAI resource id
AOAI_ID=$(az cognitiveservices account show \
  --name aoai-nimesh-lab \
  --resource-group rg-aoai-lab \
  --query id -o tsv)

# Grant the Cognitive Services OpenAI User role
az role assignment create \
  --role "Cognitive Services OpenAI User" \
  --assignee $LA_PRINCIPAL \
  --scope $AOAI_ID
```

### 9.3 Change the Logic App HTTP action to use Managed Identity
1. Open the Logic App designer → open the **HTTP** action.
2. **Authentication** → **Managed Identity** → **System-assigned managed identity**.
3. **Audience**: `https://cognitiveservices.azure.com`.
4. **Remove** the `api-key` header (no longer needed).
5. Save. Re-test.

If it still works, you've eliminated the API key from the flow entirely. The Logic App authenticates to Azure OpenAI via its Managed Identity and Azure AD — no secret in the flow, no secret in Key Vault to rotate, nothing to leak.

**This is the interview gold standard answer.** "Managed Identity with the Cognitive Services OpenAI User role scoped to the AOAI resource — no secrets in the workflow."

---

## 10. Private Endpoint (for PHI / regulated workloads)

You probably won't fully provision this in the lab (it requires a VNet, which adds cost), but you need to understand it.

### 10.1 What it is
- By default, your Azure OpenAI endpoint is reachable from the public internet (auth-gated, but reachable).
- A **Private Endpoint** puts the OpenAI resource on a private IP inside *your* VNet. Public access can be fully disabled.
- Traffic from your Logic App / Function / App Service (also VNet-integrated) to Azure OpenAI flows over Microsoft's backbone — never the public internet.

### 10.2 Why it matters at Marken
- PHI, clinical trial data, export-controlled logistics data.
- Compliance requirements often mandate that sensitive data never traverse the public internet, even if encrypted.
- It's also a common risk-management control: defense in depth. Even if auth credentials leaked, without VNet access the endpoint is unreachable.

### 10.3 What the architecture looks like
```
[VNet]
  ├── Logic App Standard (VNet-integrated App Service Plan)
  │       │ calls via private IP
  │       ▼
  ├── Private Endpoint for Azure OpenAI (private IP)
  │       │
  │       └─► [Azure OpenAI resource — public access disabled]
  └── Private DNS zone for *.openai.azure.com (maps to private IP)
```

### 10.4 Interview framing
> "For regulated workloads I'd disable public network access on the Azure OpenAI resource and expose it only via a Private Endpoint in the same VNet as the consuming Logic App Standard or Function App. Calls stay on the Microsoft backbone. Paired with Managed Identity auth and disabled prompt/response logging, that's the enterprise pattern for anything PHI-adjacent."

---

## 11. Observability — Application Insights

### 11.1 Why this matters
In production you need to know: what prompts ran, how long they took, what they cost, what the content filter did. Your existing "structured logging at decision points" bullet lives here in the Microsoft stack.

### 11.2 Enable diagnostic logging on Azure OpenAI
1. Portal → your OpenAI resource → **Monitoring** → **Diagnostic settings** → **+ Add diagnostic setting**.
2. Select all log categories: `Audit`, `RequestResponse`, `Trace`.
3. Destination: **Send to Log Analytics workspace**. Create a workspace in the same resource group if you don't have one.
4. Save.

### 11.3 What you get
- Every API call logged with timestamp, deployment, token counts, latency, status.
- Queryable in Log Analytics with KQL. Example:
  ```kql
  AzureDiagnostics
  | where Category == "RequestResponse"
  | where TimeGenerated > ago(1h)
  | project TimeGenerated, OperationName, DurationMs, ResultDescription
  | order by TimeGenerated desc
  ```

### 11.4 When NOT to log
- For PHI / regulated: you can (and should) disable prompt/response content logging — the Azure support form for "abuse monitoring modification" lets you opt out of Microsoft's content logging. You keep operational metrics (latency, tokens, errors) but the actual prompt text isn't persisted on Microsoft's side.
- On *your* side: don't log raw prompt text into Application Insights for regulated workloads. Log a hash + pointer to a secure store where the content lives under your own access controls.

### 11.5 Interview framing
> "Operational telemetry goes to Log Analytics — latency, tokens, error codes. Content logging is deliberate: on by default in dev, off for PHI workflows, with prompt text replaced by hashes plus pointers to an access-controlled store. Keeps evaluation-ready data without creating a compliance surface."

---

## 12. Cost monitoring and controls

### 12.1 Set a budget alert now
```bash
# Get subscription id
SUB_ID=$(az account show --query id -o tsv)

# Via portal: Cost Management + Billing → Budgets → + Add
# CLI budget creation is clunky; do it in the portal.
```

Portal steps:
1. Portal → search **Cost Management** → **Budgets** → **+ Add**.
2. Scope: your subscription.
3. Amount: $10 (for a lab). Period: monthly.
4. Alert thresholds: 50%, 80%, 100% — email to your address.

This prevents bill shock if you forget to clean up.

### 12.2 Standard vs Provisioned — know the names
- **Standard (PAYG)**: pay per token. Elastic. No reservation. Default for lab, pilot, unpredictable workloads.
- **Provisioned Throughput Units (PTU)**: reserved capacity, paid hourly. Predictable latency, required for tight SLAs. Expensive — only makes sense at sustained volume.
- **Batch API**: asynchronous, cheaper per token, for non-real-time workloads. Good for overnight extraction jobs.

### 12.3 Cost levers for when you scale
1. **Prompt caching** — some models cache repeated prompt prefixes. Structure prompts so stable content (system prompt, few-shot examples) comes first.
2. **Smaller models for classification** — `gpt-4o-mini` does 80%+ of extraction/classification tasks and is 5–10× cheaper than `gpt-4o`.
3. **Batch for non-urgent** — nightly document processing doesn't need real-time.
4. **Per-tenant metering** — tag every call with tenant/use-case, aggregate, identify cost whales.

---

## 13. Clean up

**Do this at the end of your practice session so you don't accrue charges.**

### 13.1 Nuclear option (simplest)
```bash
az group delete --name rg-aoai-lab --yes --no-wait
```

This deletes the resource group and everything in it — Azure OpenAI, Logic App, Key Vault, Log Analytics, budgets. Wait 5 minutes and verify in the portal.

### 13.2 Soft-delete caveat for Key Vault
- Deleted Key Vaults enter a soft-delete state for 90 days by default (you can't reuse the name).
- For a lab, this is fine — use a different vault name next time.
- In production, purge protection is usually ON (a good thing).

### 13.3 Keep the code
Don't delete the `aoai-lab` directory with your Python scripts — that's your interview reference. Move it somewhere you won't lose it.

---

## 14. What you can honestly claim after completing this

Write these down. Rehearse them out loud.

> "I've provisioned Azure OpenAI end-to-end: resource creation, model deployment (gpt-4o-mini with a named deployment), and credential management via Key Vault. I've called it from Python using the AzureOpenAI SDK including structured-output parsing with Pydantic, from Logic Apps Consumption using HTTP with Managed Identity and the Cognitive Services OpenAI User role, and from Power Automate with both hardcoded keys for dev and the Key Vault connector for staged environments. For observability I've wired diagnostic logs to Log Analytics and can query them with KQL. I understand the Private Endpoint pattern for regulated workloads — VNet-integrated Logic App Standard, private DNS, public network access disabled — though I haven't stood up the full networking topology yet."

This is a specific, verifiable claim. Anyone drilling on it will find real understanding underneath.

---

## 15. Quick-reference cheat sheet

| Concept | Answer |
|---|---|
| Endpoint pattern | `https://<resource>.openai.azure.com/openai/deployments/<deployment>/chat/completions?api-version=<version>` |
| Resource vs deployment | Resource = billing/auth container. Deployment = callable model instance inside it. |
| Auth for prod | Managed Identity + role `Cognitive Services OpenAI User`, scoped to the AOAI resource. |
| Auth for dev | API key in Key Vault, referenced via Key Vault connector / Key Vault reference. |
| Private Endpoint | Disables public access; AOAI reachable only from the VNet via private IP. Required pattern for PHI. |
| Content filter | On by default. Configurable. Azure AI Content Safety for more aggressive filters. |
| Prompt/response logging | Microsoft-side can be disabled via support request for regulated workloads. Your-side logging: hash + pointer, not raw content. |
| Standard vs Provisioned | Standard = pay per token (default). Provisioned (PTU) = reserved hourly (latency-guaranteed, expensive). |
| Cheapest capable model | `gpt-4o-mini` for extraction/classification. |
| Structured output | `response_format={"type":"json_object"}` or `type:"json_schema"` with a schema. In Python SDK: `client.beta.chat.completions.parse(..., response_format=PydanticModel)`. |
| Python client | `from openai import AzureOpenAI`. Not the plain `OpenAI` client. |
| API version | Pass as query param. `2024-10-21` is a recent stable GA. Preview versions have newer features. |
| Clean up | `az group delete --name rg-aoai-lab --yes` |

---

## 16. If you get stuck

| Symptom | Likely cause | Fix |
|---|---|---|
| 401 Unauthorized from API | Wrong key, wrong endpoint, or wrong resource region | Recopy from Keys and Endpoint blade; verify `api-version`. |
| 404 from API | Wrong deployment name OR `api-version` too old for this model | Deployment name is case-sensitive. Try `2024-10-21` or newer. |
| 429 Rate limit | TPM quota exhausted | Increase TPM in Quotas blade, or add client-side backoff. |
| "Resource not found" creating AOAI | Azure OpenAI not yet approved for your subscription | Submit the access form — usually same-day. |
| Playground works, API doesn't | Auth header or URL wrong | Copy exactly from the Playground's **View code** output. |
| Managed Identity 403 | Role not propagated yet, or wrong audience | Wait 2 min; verify audience is `https://cognitiveservices.azure.com`. |
| Key Vault 403 | RBAC role not assigned or propagated | Wait 30 sec; re-run `az role assignment create`. |

---

## 17. Where this fits in the bigger prep picture

- **Before this**: you know AI-in-workflow patterns from your Gen AI Engineer work.
- **After this**: you know them *with concrete Azure-specific proof points* — the one thing that separates a "generic AI engineer" from an "AI engineer who can ship at Marken."
- **Pair with**: `Power_Platform_Logic_Apps_Ramp_Plan.md` Project C (Copilot Studio), which uses Azure OpenAI underneath. Doing AOAI hands-on first makes Copilot Studio click faster.
- **Don't waste further time on**: fine-tuning in Azure OpenAI, Prompt Flow, Azure ML. Not relevant for this role.

**Budget: 3–5 hours in one sitting. Clean up at the end. Rehearse Section 14 until it's reflexive.**
