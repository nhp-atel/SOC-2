# SOC 2 for Engineers — A Practical, Interview-Ready Guide

A hands-on guide for backend/full-stack engineers working with APIs, databases, dashboards, CI/CD, and cloud deployments. Focused on real engineering artifacts, not compliance theory.

---

## Table of Contents

1. [Foundations](#1-foundations-the-short-version)
2. [Mapping Each TSC to Engineering](#2-mapping-each-tsc-to-engineering)
3. [Controls Deep Dive](#3-controls-deep-dive)
4. [Evidence & Audits](#4-evidence--audits-the-part-most-engineers-never-learn)
5. [Hands-on: Making a Real System SOC 2 Aligned](#5-hands-on-making-a-real-system-soc-2-aligned)
6. [Interview Readiness](#6-interview-readiness)
7. [Mini Project / Practice Checklist](#7-mini-project--practice-checklist)

---

# 1. Foundations (the short version)

**What SOC 2 is:** An audit report produced by a licensed CPA firm (AICPA standard) that tells your customers: *"We inspected this company's systems and processes, and they actually do what they claim around security."* It's not a law. It's not a certification. It's an **opinion letter** from an auditor.

**Why it exists:** Before SOC 2, every enterprise buyer sent their own 400-question security questionnaire to every vendor. SOC 2 standardized that. Now a SaaS vendor gets audited once a year, hands the report to 500 customers, and unlocks enterprise sales. In practice: *no SOC 2 report = no Fortune 500 deals.*

**Type 1 vs Type 2:**
- **Type 1:** "On December 1, these controls existed and were designed correctly." (Snapshot)
- **Type 2:** "Over the last 6–12 months, these controls existed AND operated effectively." (What customers actually want.)

**The 5 Trust Service Criteria (TSC):**

| TSC | Plain-English question |
|---|---|
| Security (a.k.a. Common Criteria) | Can attackers get in or mess with your systems? |
| Availability | Is your service up when customers need it? |
| Processing Integrity | Does your system process data correctly and completely? |
| Confidentiality | Do you protect data marked confidential (source code, business data)? |
| Privacy | Do you handle PII the way you told users you would? |

**Key fact:** Only **Security** is mandatory. The other four are opt-in based on what you promise customers. Most SaaS companies pick **Security + Availability + Confidentiality**. Privacy is rarer (GDPR/CCPA often cover it separately). Processing Integrity matters for fintech/data pipelines.

---

# 2. Mapping Each TSC to Engineering

### 2.1 Security (Common Criteria — CC1 through CC9)

**What it means in real systems:** Unauthorized people can't access your systems, data, or config. Covers identity, network, endpoints, code, and incident response.

**What engineers actually build:**
- SSO + MFA enforcement (Okta, Google Workspace, Entra ID)
- RBAC in the app (role claims in JWT, middleware that checks them)
- AWS IAM with least-privilege policies + conditions (`aws:SourceIp`, `aws:MultiFactorAuthPresent`)
- VPC with private subnets, security groups, WAF in front of public endpoints
- Secrets Manager / Vault (never `.env` in git)
- Vulnerability scanning: Snyk, Dependabot, Trivy on container images
- SIEM pipeline: app logs + CloudTrail + VPC Flow Logs → Datadog/Splunk
- Incident response runbook with defined severities and on-call

**Real example:** Engineer adds a new admin endpoint. Security controls that apply: must be behind `requireRole('admin')` middleware, logged with `user_id` + `request_id` to CloudWatch, rate-limited, accessible only over TLS 1.2+, and the change must go through a PR with one approver + passing CI.

### 2.2 Availability

**What it means:** You meet whatever uptime promise you made (SLA in your MSA).

**What engineers build:**
- SLOs/SLIs tracked (e.g., 99.9% API availability, p99 < 500ms)
- Alerting: Datadog monitors → PagerDuty → on-call rotation
- Multi-AZ RDS, auto-scaling groups, load balancers
- Daily automated backups + **quarterly restore drills** (auditors ask for this specifically)
- DR plan with defined RTO (recovery time) and RPO (max data loss)
- Status page (statuspage.io) for customer-facing incident comms

**Real example:** Postgres RDS with `multi_az = true`, PITR enabled (35-day window), `aws backup` plan pushing snapshots to a separate account. A quarterly Jira ticket: "DR drill Q2 2026 — restored prod snapshot to staging, validated app connects, documented in runbook."

### 2.3 Processing Integrity

**What it means:** Data going through your system comes out complete, accurate, and on time. Most relevant for payment systems, data pipelines, billing.

**What engineers build:**
- Input validation at the API boundary (Zod, Pydantic, JSON Schema)
- **Idempotency keys** on mutating endpoints (avoid duplicate charges)
- Database constraints (NOT NULL, UNIQUE, CHECK, FK)
- Reconciliation jobs: nightly cron that compares `stripe_charges` vs `our_invoices` and alerts on drift
- Data quality tests in pipelines (dbt tests, Great Expectations)
- Queue semantics documented: at-least-once vs exactly-once, DLQs

**Real example:** Stripe webhook handler uses the `Stripe-Signature` header to verify authenticity, extracts the `event.id`, upserts into `processed_events` with `id` as PK. Duplicate events no-op. Failures go to a DLQ that pages on-call.

### 2.4 Confidentiality

**What it means:** Data *you classified as confidential* (customer data, source code, internal financials) is protected from unauthorized disclosure.

**What engineers build:**
- Encryption at rest (KMS-backed — RDS, S3, EBS all `encrypted: true`)
- Encryption in transit (TLS 1.2+ everywhere, including internal service-to-service)
- Data classification tags on resources (`data_classification: confidential`)
- Access limited to need-to-know (no `*:*` IAM policies)
- DLP for dev laptops (optional but common for higher tiers)

**Real example:** An engineer tags an S3 bucket with `Classification=Confidential`. A Config rule auto-checks that all such buckets have `ServerSideEncryption=aws:kms`, `BlockPublicAccess=true`, and access logging enabled. Non-compliant buckets trigger a Slack alert.

### 2.5 Privacy

**What it means:** You handle PII (names, emails, SSNs) the way your privacy policy claims you do.

**What engineers build:**
- Consent capture + storage (timestamp, version of ToS accepted)
- DSAR endpoints (Data Subject Access Request): `GET /me/data-export`, `DELETE /me`
- Data retention jobs: `DELETE FROM users WHERE deleted_at < now() - interval '30 days'`
- PII redaction in logs (never log passwords, tokens, SSNs)
- Data maps: "PII lives in `users`, `sessions`, `audit_log.actor_email`"

**Real example:** GDPR deletion request comes in. Engineer runs a script that: cascades user delete across 8 tables, purges from ElasticSearch, revokes all active sessions, removes from email list in Customer.io, confirms via email. Entire flow logged with ticket ID as evidence.

---

# 3. Controls Deep Dive

### 3.1 Access Control

**What it is:** Only the right humans and services can access the right resources at the right time.

**How to implement it:**

```ts
// API middleware (Express)
function requireRole(role: Role) {
  return (req, res, next) => {
    const user = req.user; // decoded JWT
    if (!user) return res.status(401).send();
    if (!user.roles.includes(role)) {
      audit.log('access_denied', { userId: user.id, required: role, path: req.path });
      return res.status(403).send();
    }
    next();
  };
}

app.delete('/api/tenants/:id', requireRole('admin'), deleteTenant);
```

Operationally:
- SSO (Okta/Google) for all human access. No local passwords for internal tools.
- MFA enforced via IdP, not per-app.
- SCIM auto-provisioning/deprovisioning — terminated employee loses access within minutes, not days.
- **Quarterly access review:** export users per system (AWS, GitHub, prod DB, Datadog), manager certifies each line.
- Break-glass account: one emergency root account, credentials split (Shamir-style) in a safe, alerting on any use.

**What goes wrong if missing:**
- Ex-employee still has GitHub push access 3 months after leaving (classic finding).
- Developer has `AdministratorAccess` in prod "just in case."
- Service account with hardcoded password shared across teams.
- Nobody knows who has prod DB access.

### 3.2 Audit Logging

**What it is:** A tamper-evident record of who did what, when, where, and to what.

**How to implement it:**

```ts
// Structured audit event
logger.info({
  event: 'user.role_granted',
  actor_id: req.user.id,
  actor_email: req.user.email,
  target_user_id: targetId,
  role: 'admin',
  request_id: req.id,
  ip: req.ip,
  user_agent: req.headers['user-agent'],
  tenant_id: req.tenantId,
  timestamp: new Date().toISOString(),
});
```

Pipeline:
- App → stdout (JSON) → Fluent Bit → Datadog (hot, 30 days) + S3 (cold, 1+ year with Object Lock in Compliance mode so nobody — including the root account — can delete).
- CloudTrail + VPC Flow Logs + RDS audit logs all go to the same S3.
- Correlation ID (`request_id`) propagated across services via `X-Request-ID` header.

**Must log (minimum for SOC 2):**
- Authentication (success, failure, MFA challenges)
- Authorization decisions (especially denials)
- Privileged actions (role grants, data exports, prod deploys)
- Data access to sensitive tables (via DB audit or app-level)
- Configuration changes (feature flags, infra)

**What goes wrong if missing:**
- Breach happens, you can't answer "whose credentials were used?"
- Logs are kept 7 days — audit incident window is gone.
- Logs live in a mutable S3 bucket — attacker wipes their trail.
- No correlation ID, so you can't trace a request across services.

### 3.3 Data Protection

**What it is:** Data is encrypted, keys are managed, secrets are not leaked.

**How to implement it:**
- **At rest:**
  - RDS: `storage_encrypted = true`, `kms_key_id = aws_kms_key.db.arn`
  - S3: default encryption with KMS, `BucketKeyEnabled = true`
  - EBS: default encryption on at account level
- **In transit:**
  - ALB/CloudFront TLS 1.2+ only, HSTS headers
  - Internal: service mesh (Istio/App Mesh) with mTLS, or at minimum TLS on internal ALBs
- **Secrets:**
  - AWS Secrets Manager or HashiCorp Vault
  - Rotated every 90 days (automated rotation lambda for RDS)
  - Injected at runtime (ECS task definition pulls from Secrets Manager) — never baked into images
  - Pre-commit hook: `gitleaks` or `trufflehog` to catch accidental commits
- **Key management:**
  - Customer-managed KMS keys (CMK) for sensitive workloads
  - Key rotation enabled
  - Key policy restricts who can `Decrypt`

**What goes wrong if missing:**
- `.env` file pushed to public GitHub, AWS keys compromised in minutes.
- Unencrypted RDS snapshot shared accidentally with another account.
- TLS terminated at the load balancer but backend uses plain HTTP inside VPC — insider threat reads traffic.
- DB password last rotated 4 years ago, known by 12 ex-employees.

### 3.4 Change Management

**What it is:** Changes to production are reviewed, tested, tracked, and reversible.

**How to implement it:**
- **Branch protection** on `main`:
  - Require PR
  - Require 1+ approving reviews (CODEOWNERS for sensitive paths)
  - Require passing CI (tests, lint, SAST, IaC scan)
  - No direct pushes, no force-pushes
- **PR template** includes: description, risk assessment, rollback plan, test evidence.
- **CI/CD pipeline** (GitHub Actions example):

  ```yaml
  jobs:
    test:     # unit + integration tests
    sast:     # Semgrep/CodeQL
    scan:     # Snyk on dependencies, Trivy on Docker image
    iac:      # tfsec or Checkov on Terraform
    deploy:   # only on merge to main, uses OIDC to assume AWS role
  ```

- **Separation of duties:** Developer writes code. Different person approves PR. CI/CD deploys — humans don't SSH into prod to run commands.
- **Immutable deploys:** Container images tagged by git SHA. Rolling back = redeploy an old SHA.
- **Every prod change traceable:** PR → Jira ticket → deploy log (commit SHA, timestamp, deployer).

**What goes wrong if missing:**
- Developer pushes a "quick fix" straight to main at 2am, takes down prod.
- No way to know who deployed what when.
- A secret ends up in a commit that's in 50 engineers' local clones forever.
- Rolling back means guessing which commit was "good."

### 3.5 Availability / Monitoring

**What it is:** You know when things break, and you can recover.

**How to implement it:**
- **SLOs** defined: `availability >= 99.9%` measured as `successful_requests / total_requests` over 30d.
- **Monitors** at multiple layers:
  - Infrastructure (CPU, memory, disk — Datadog agent)
  - Application (error rate, latency p99 — APM)
  - Business (checkout success rate, signup rate)
  - Synthetic (external probes every 1m from multiple regions)
- **Alerting** hierarchy: info → Slack, warning → Slack + email, critical → PagerDuty (24/7 on-call).
- **Runbooks** per alert linked in Slack. "CPU > 90%" → here are the 3 things to check.
- **Backups:** automated, encrypted, stored in a separate AWS account (blast radius), tested via quarterly restore drill.
- **DR plan:** documented RTO/RPO, tested annually.

**What goes wrong if missing:**
- Service down for 4 hours before a customer tweets about it.
- Backups "succeeded" nightly for 2 years, but the restore fails when you try.
- On-call engineer paged at 3am with no runbook — flails for an hour.

---

# 4. Evidence & Audits (the part most engineers never learn)

### How an audit actually runs

1. **Readiness / gap assessment (2–4 weeks):** Auditor walks the company through the Trust Services Criteria, identifies missing controls. You get a punch list.
2. **Remediation (1–6 months):** Engineers implement missing controls (MFA enforcement, log retention, etc.).
3. **Audit window begins** (Type 2 only): Typically 3–12 months of *observation*. Everything from here on must work continuously. *This is why auditors sample dates across the window.*
4. **Fieldwork (2–6 weeks):** Auditor requests evidence, interviews engineers, samples tickets.
5. **Report issued:** SOC 2 report (50–100 pages). You send it to customers under NDA.

### What "evidence" actually means

Evidence = anything that proves a control is operating. Auditors prefer evidence in this order:

1. **System-generated** (best) — CloudTrail export, GitHub audit log, Okta admin report, Terraform state file.
2. **Ticket/workflow** — Jira ticket with approvals, PR with reviewer, Zendesk offboarding ticket.
3. **Screenshot** (acceptable, annoying) — AWS Console showing encryption enabled, with URL + timestamp visible.
4. **Policy document** (baseline only — "having a policy" doesn't mean you follow it; auditor tests operation separately).

### Example auditor requests (real ones)

| Request | What you hand over |
|---|---|
| "Provide a list of all users with production database access as of March 31, 2026." | Query output: `SELECT usename FROM pg_user;` + IAM role memberships, with timestamp |
| "For this sample of 25 terminated employees, show evidence access was removed within 24 hours." | Zendesk/Jira offboarding tickets + Okta deactivation log + GitHub audit log entries |
| "Provide evidence of the Q2 2025 access review." | Spreadsheet/Vanta export showing reviewer, date, each user reviewed, action taken |
| "Show the change management ticket for this deploy on 2026-02-14." | PR link with approvals, CI run ID, Jira ticket, deploy log line |
| "Provide evidence that database backups are tested." | Jira ticket from quarterly DR drill: snapshot ID restored, validation checklist, sign-off |
| "Show encryption at rest configuration for customer data stores." | Terraform code + `aws rds describe-db-instances` output showing `StorageEncrypted: true` |
| "Provide the last 12 months of security incidents." | Incident tracker export with severity, detection time, resolution time, postmortem links |
| "Show evidence of vulnerability scanning." | Snyk dashboard export, Dependabot PRs merged, last penetration test report |

### How engineers prepare

- **Automate evidence collection.** Use Vanta, Drata, or Secureframe — they connect to AWS, GitHub, Okta, etc. and continuously pull evidence. Saves 100+ engineering hours per audit.
- **Tag everything.** Every AWS resource: `Environment`, `Owner`, `DataClassification`, `ManagedBy=Terraform`.
- **Structured tickets.** A standard PR template + deploy ticket format makes sampling trivial.
- **Log retention >= 13 months** (covers a full Type 2 window + buffer).
- **A single source of truth** for access (Okta → provisions to AWS/GitHub/Datadog via SCIM). Never manually created accounts.
- **Named control owners.** Each control has an accountable engineer. During fieldwork, auditor says "walk me through how you deploy to prod" — that person demos.

---

# 5. Hands-on: Making a Real System SOC 2 Aligned

### The sample system

- **API:** Node.js/Express or Python/FastAPI, running on AWS ECS Fargate
- **DBs:** Postgres (primary transactional) + DynamoDB (session/cache) + S3 (user uploads)
- **Dashboard:** React app on CloudFront + S3
- **CI/CD:** GitHub Actions → AWS via OIDC
- **Auth:** Okta SSO for employees; Auth0 or Cognito for end users

### The architecture with controls mapped

```
                          ┌─────────────────────┐
 End Users ──HTTPS────────▶│  CloudFront + WAF  │  [Security: WAF, DDoS / Availability]
                          └──────────┬──────────┘
                                     │ TLS 1.2+
                          ┌──────────▼──────────┐
                          │   ALB (public SG)   │  [Confidentiality: TLS]
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │  ECS Fargate API    │  ◀── Secrets Manager (runtime inject)
                          │  - JWT auth         │      [Security: secrets mgmt]
                          │  - RBAC middleware  │
                          │  - Input validation │
                          │  - Rate limiting    │      Audit logs ──▶ CloudWatch ──▶ S3 (Object Lock, 13mo)
                          │  - Request IDs      │      [Security + Privacy: logging]
                          └────┬─────────┬──────┘
                               │         │
                     (private) │         │ (private)
                          ┌────▼───┐ ┌───▼────────┐
                          │  RDS   │ │ DynamoDB   │  [Confidentiality: KMS encryption]
                          │Postgres│ │ (sessions) │  [Availability: multi-AZ, PITR]
                          │Multi-AZ│ └────────────┘
                          └────────┘
                               │
                               ▼ daily automated
                          ┌────────────┐
                          │ Backups    │  (separate AWS account, encrypted)
                          │ S3+Glacier │  [Availability: tested quarterly]
                          └────────────┘

 Employees ──Okta SSO + MFA──▶ AWS Console / GitHub / Datadog / Prod DB (via bastion)
                                [Security: SSO/MFA, RBAC, access reviews]

 Developers ──PR──▶ GitHub ──CI──▶ OIDC ──▶ AWS Deploy
                    │
                    ├─ Branch protection (2 approvers, CODEOWNERS)
                    ├─ CI: tests, Semgrep SAST, Snyk deps, Trivy image, Checkov IaC
                    └─ No long-lived AWS keys [Security: change mgmt]
```

### Concrete changes to make, grouped by control

**Access Control**
- [ ] All employee apps behind Okta with MFA enforced in the IdP policy.
- [ ] GitHub: org-level MFA required, SCIM from Okta, CODEOWNERS file for `/infra/**` and `/services/auth/**`.
- [ ] AWS: one IAM role per job function (`developer`, `oncall`, `admin`), assumed via Okta federation, max session 4h.
- [ ] Prod DB: no shared account. Engineers connect via IAM DB auth through a bastion logged by Session Manager.
- [ ] Quarterly access review automated: script pulls all users from each system, posts to a Notion/Vanta review queue.

**Audit Logging**
- [ ] App emits structured JSON logs with `request_id`, `user_id`, `tenant_id`, `event`, `ip`.
- [ ] CloudTrail multi-region on, integrated with CloudWatch Events for anomaly alerts (root login, IAM policy change).
- [ ] VPC Flow Logs + ALB access logs on.
- [ ] All logs shipped to S3 with Object Lock (Compliance mode, 400 days).
- [ ] PII never in logs — central logger scrubs `password`, `ssn`, `token` keys.

**Data Protection**
- [ ] `aws_rds_cluster.storage_encrypted = true` + customer-managed KMS key.
- [ ] S3 bucket policy denies non-TLS requests (`aws:SecureTransport = false`).
- [ ] All Secrets Manager secrets have rotation enabled.
- [ ] Pre-commit: `gitleaks` in `.pre-commit-config.yaml`; GitHub secret scanning on.
- [ ] KMS key rotation enabled; key policy restricts `Decrypt` to the app role only.

**Change Management**
- [ ] Branch protection: require PR, 1+ reviews, CODEOWNERS for sensitive dirs, status checks must pass.
- [ ] PR template: description, risk, rollback plan, test evidence, linked ticket.
- [ ] CI runs: unit, integration, Semgrep, Snyk, Trivy, Checkov. All must pass.
- [ ] Deploys triggered only from `main` via GitHub Actions using OIDC (no static AWS keys).
- [ ] Every deploy tagged with git SHA; rollback = redeploy old task definition.

**Availability / Monitoring**
- [ ] RDS: Multi-AZ, `backup_retention_period = 35`, `performance_insights_enabled = true`.
- [ ] Datadog monitors for p99 latency, error rate, DB connection count, disk, cert expiry.
- [ ] PagerDuty service with 24/7 rotation; runbooks linked in each monitor.
- [ ] Synthetic check every 1m against `/healthz` from 3 regions.
- [ ] Quarterly DR drill: restore latest snapshot to staging, run smoke tests, sign-off ticket.
- [ ] Documented SLOs in `/docs/slo.md`; error budget policy.

**Privacy (if in scope)**
- [ ] `/me/data-export` and `DELETE /me` endpoints implemented, tested.
- [ ] Retention cron: hard-delete users 30 days after soft-delete.
- [ ] Data map in `/docs/data-inventory.md`: what PII lives where, which systems process it.

---

# 6. Interview Readiness

### Talking about your experience authentically

The trap: sounding like you memorized a compliance doc. The fix: **talk about what you built or fixed**, then let the framework language come after.

Wrong: *"I implemented CC6.1 logical access controls per the TSC."*

Right: *"We had a finding where three offboarded contractors still had GitHub access. I wrote a script that pulls Okta terminations nightly and reconciles against GitHub, AWS, and our prod DB. It now opens a Jira ticket for anything out of sync. That's the kind of thing auditors want evidence of — it became one of our primary access-review artifacts."*

### Common interview questions + strong answers

**Q: What is SOC 2 and what are the Trust Services Criteria?**

*"SOC 2 is an audit report that says an independent CPA inspected our security controls and they work. It's built on five criteria — Security is mandatory, then Availability, Confidentiality, Processing Integrity, and Privacy are opt-in based on what we promise customers. At [company] we were scoped for Security, Availability, and Confidentiality because we're a B2B SaaS handling customer business data but not regulated PII."*

**Q: Difference between Type 1 and Type 2?**

*"Type 1 is a snapshot — controls exist and are designed well on a specific date. Type 2 is operational — the same controls worked continuously over a 6- or 12-month window. Customers almost always want Type 2; Type 1 is a stepping stone for first-time companies."*

**Q: How do you implement access control for SOC 2?**

*"Three layers. First, identity: SSO via Okta with MFA enforced, SCIM provisioning so terminations propagate in minutes. Second, authorization: RBAC in the app via role claims in JWTs, middleware that logs every denial. Third, review: quarterly access review where I export users from Okta, AWS, GitHub, and prod DB, and managers certify each line. We automated evidence collection with Vanta so the auditor could self-serve."*

**Q: How do you log for audit purposes?**

*"Structured JSON logs with actor, action, target, and request ID — correlated across services. Everything lands in CloudWatch for 30 days and S3 with Object Lock for 13 months so logs can't be tampered with, even by admins. We log auth events, privileged actions, data exports, and config changes. PII is scrubbed at the logger level to avoid leaking data into the logs themselves."*

**Q: How do you handle secrets?**

*"AWS Secrets Manager with automatic rotation on RDS credentials. Secrets are pulled at container start, never baked into images. We have pre-commit hooks with gitleaks and GitHub push protection on. When a secret does leak — which happened once — we rotate immediately, audit CloudTrail for usage, and write a postmortem."*

**Q: Walk me through your change management process.**

*"Every change goes through a PR. Branch protection requires one reviewer and CODEOWNERS approval for sensitive areas. CI runs unit, integration, Semgrep for SAST, Snyk for dependencies, Checkov on Terraform. On merge, GitHub Actions assumes an AWS role via OIDC — no long-lived keys — and deploys the new container image tagged by git SHA. Rollback is redeploying the previous SHA. Every deploy is traceable from Jira ticket to PR to CI run to CloudTrail event."*

**Q: An auditor asks you to prove that only authorized people have production access. What do you show them?**

*"Three artifacts. First, the access control policy doc that defines who's authorized. Second, the current state: Okta group membership for the `prod-access` group plus AWS IAM role assumptions from CloudTrail. Third, the Q-over-Q access reviews showing that membership was reviewed and approved. For sampled terminations, the Zendesk offboarding ticket plus the Okta deactivation timestamp."*

**Q: How would you prepare a system that has no SOC 2 controls today?**

*"I'd work backwards from the five criteria. Week one: baseline hygiene — MFA everywhere, branch protection, CloudTrail on, secrets out of git. Week two to four: logging pipeline with retention, RBAC audit, encryption verification, backup testing. Then policies to match — access control, change management, incident response. Then a readiness assessment with the auditor to find gaps. Type 1 in month three or four, then a Type 2 window begins. The goal is to build controls that survive engineers moving fast, not to bolt on checklists."*

**Q: What's a SOC 2 control that's easy to pass on paper but hard to do well?**

*"Access reviews. It's easy to run them quarterly and check a box. It's hard to make them meaningful — most managers rubber-stamp the list. What works is showing deltas: who's new, who changed roles, who hasn't logged in in 60 days. Forces a real decision. Same with incident response — having a runbook is easy; running a tabletop exercise every six months is what makes it real."*

---

# 7. Mini Project / Practice Checklist

Build or harden a small service and apply the controls. Pick one:

**Option A — Harden an existing side project**
Take any backend + DB + frontend you've built. Apply the checklist below.

**Option B — Build a minimal "audit-ready" CRUD API**
Node/Python API + Postgres + React dashboard, deployed to AWS (or fly.io / Render if cheaper).

### The checklist

**Identity & Access**
- [ ] SSO via Auth0/Cognito/Clerk (free tier is fine)
- [ ] MFA required for admin role
- [ ] RBAC in the app: at least `user`, `admin` roles
- [ ] Middleware logs every authorization denial
- [ ] Document who has access to what in `ACCESS.md`

**Logging**
- [ ] Structured JSON logging with `request_id`, `user_id`, `event`
- [ ] Correlation ID propagated through all services
- [ ] Audit log for: login, logout, role changes, data exports
- [ ] Logs retained for 30+ days (CloudWatch or Loki)
- [ ] No PII/secrets in logs (test with a grep)

**Data Protection**
- [ ] TLS 1.2+ enforced (HTTPS redirect + HSTS)
- [ ] DB encryption at rest enabled
- [ ] Secrets in a secret manager (not `.env` committed)
- [ ] `gitleaks` pre-commit hook
- [ ] A `data-classification.md` listing what data is where

**Change Management**
- [ ] Branch protection on `main` (require PR, 1 approver, passing CI)
- [ ] CI runs tests + a SAST tool (Semgrep free) + a dependency scanner (Dependabot/Snyk)
- [ ] PR template with rollback plan
- [ ] Deploys automated from CI; no manual `ssh` to prod
- [ ] Every deploy tagged with git SHA

**Availability**
- [ ] Healthcheck endpoint + uptime monitor (UptimeRobot free)
- [ ] Automated DB backups
- [ ] **Actually restore a backup once** and document it in `DR.md`
- [ ] Alerting that pages you (email/SMS/Pushover) on downtime

**Documentation (the "policy" side)**
- [ ] `SECURITY.md` — how to report vulnerabilities
- [ ] `INCIDENT_RESPONSE.md` — severity levels, who does what
- [ ] `ACCESS.md` — who has access and why
- [ ] `DR.md` — RTO/RPO, backup/restore procedure
- [ ] `CHANGE_MGMT.md` — how changes get to prod

**Stretch goals**
- [ ] Automate an "evidence bundle": a script that outputs current IAM, user list, last 10 PRs, last 10 deploys, most recent backup restore log — the auditor's dream.
- [ ] Write a postmortem for a real or simulated incident.
- [ ] Do a tabletop: "Someone leaked an AWS key on GitHub. Walk through the next 30 minutes." Write what you'd do.

### How to talk about the project in interviews

After you do this, you have a legitimate story: *"I built a service with SOC 2 controls end-to-end — SSO+MFA, RBAC, immutable audit logs, encrypted storage, CI-enforced change management, tested backups. Here's the repo. Here's the evidence bundle script. Here's the runbook."*

That beats memorized criteria every time.
