# Security Architecture — IAM, tenant isolation, threat model, supply chain

> The senior critique I anticipate: *"you mentioned WAF + TLS + secrets but where's the IAM model, where's the threat model, where's the supply-chain story?"* This document is that answer.

> ⚠️ **Read this doc together with [`17-architecture-truth-table.md`](./17-architecture-truth-table.md).**
> The architecture below describes the **target** security posture. Some
> controls are implemented today; some are simulated for the take-home
> demo; some are explicit production targets with triggers to un-defer.
> The truth-table is the single source of "what's actually shipped vs
> demo vs deferred". Wherever this document and the truth-table appear
> to conflict, **the truth-table wins.**

I treat security as five layers. Solution A ships fundamentals in each; the senior version of each layer is documented as deferred work with explicit triggers.

---

## Layer 1 — Identity and Access (IAM)

### Caller types

| Caller | Authentication | Authorization | Notes |
|---|---|---|---|
| **Dealer admin** (uploads xlsx, queries own catalog) | JWT bearer token, OAuth2 / OIDC flow | RBAC role = `dealer_admin`; scoped to own `dealer_id` | Per-dealer roles |
| **Marketplace consumer** (reads catalog for listings) | API key (signed) | RBAC role = `marketplace_read`; can read across dealers via signed URL per dealer | Rate-limited |
| **Internal ops** (review queue, audit) | SSO via Microsoft Entra ID / Google Workspace | RBAC role = `ops_admin`; full read; restricted write | MFA enforced |
| **System / service account** (CI/CD, scheduled jobs) | Service principal credential, rotated every 90 days | Scoped role; can only do its specific task (e.g., `ingest_runner`) | No human login |
| **On-call engineer** (incident response) | SSO + break-glass procedure | Time-boxed elevated access logged to immutable audit | Approval required >2h |

### RBAC role matrix

```
                       │ read own  │ read all  │ write     │ approve   │ delete    │
                       │  dealer   │  dealers  │  catalog  │  reviews  │  records  │
─────────────────────────────────────────────────────────────────────────────────────
dealer_admin           │    ✓      │           │   ✓ own   │           │   ✗       │
marketplace_read       │           │    ✓      │           │           │   ✗       │
ops_admin              │    ✓      │    ✓      │    ✓      │    ✓      │ ✓ (audit) │
ingest_runner (svc)    │    ✓      │    ✓      │    ✓      │           │   ✗       │
api_runner (svc)       │           │    ✓      │           │           │   ✗       │
reviewer (human)       │    ✓      │    ✓      │           │    ✓      │   ✗       │
oncall_break_glass     │    ✓      │    ✓      │    ✓      │    ✓      │ ✓ (logged)│
─────────────────────────────────────────────────────────────────────────────────────
```

Token claims include `role`, `dealer_id` (for tenant scoping), `iat`, `exp`. Tokens expire 1 hour. Refresh tokens rotate every 7 days.

---

## Layer 2 — Tenant isolation

The most important security property for InventoryFlow: **dealer A must never see dealer B's data**. The technical mechanism is PostgreSQL Row Level Security.

### Postgres RLS policies (migration 0002 in the impl repo)

```sql
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_images ENABLE ROW LEVEL SECURITY;
ALTER TABLE ingest_audit ENABLE ROW LEVEL SECURITY;

CREATE POLICY products_tenant_scope ON products
    USING (source_dealer_id = current_setting('app.current_dealer_id')::uuid
        OR current_setting('app.current_dealer_id') = 'all');

CREATE POLICY product_images_tenant_scope ON product_images
    USING (source_dealer_id = current_setting('app.current_dealer_id')::uuid
        OR current_setting('app.current_dealer_id') = 'all');
```

The Fastify request handler sets `SET LOCAL app.current_dealer_id = '<from JWT claim>'` at the start of every transaction. Marketplace callers get `all`; dealer-scoped callers get their UUID. **A bug in application code cannot leak cross-tenant data because Postgres rejects the query at the row level.**

> **Setting name:** the GUC is `app.current_dealer_id` (matching migration `0002_row_level_security.sql` in the impl repo). Earlier drafts of this doc said `app.dealer_id`; that drift is closed.

Solution A enables RLS even though we run with one dealer today. This is the **"single-tenant today, multi-tenant by config tomorrow"** pattern. Retrofitting RLS later is a 2-week project with downtime risk; enabling it on day one is a config flag.

### R2 / object storage isolation

R2 keys are scoped per dealer by default: `dealer/<dealer_id>/sha256/<2>/<2>/<rest>.<ext>`.

- **Dealer admin** receives signed URLs scoped to their `dealer_id` prefix, TTL 15 min.
- **Marketplace consumer** receives signed URLs for specific image SHAs, TTL 24 hours.
- **No bucket is public-read by default.** Cross-dealer reads require explicit policy via internal API.

Cross-dealer global image dedup (the same fastener photo used by Kayo + Honda) is opt-in via a `--global-dedup` flag and only applies to OEM-published images that the dealer has explicitly opted into a shared pool.

### Cross-tenant data leak — the threat I most worry about

| Threat path | Mitigation |
|---|---|
| Application code forgets to set `app.current_dealer_id` | Postgres rejects the query (default-deny policy); CI integration test verifies |
| JWT claim spoofed | JWT signature verified server-side; rotation every 7d |
| R2 URL guessed | URLs are SHA-256-prefixed (entropy 256 bits); signed URLs expire in 15 min |
| Cross-tenant via materialised view | Materialised views inherit RLS in Postgres 15+; explicit `WITH (security_barrier)` set |
| Cross-tenant via shared cache (Redis, JSONL) | Cache keys include `dealer_id` prefix; LLM cache hits are per-tenant unless explicitly globally-deduped |
| SQL injection through Drizzle | Drizzle is parameterized-only by default; ESLint rule blocks raw template strings |

---

## Layer 3 — Secret management

### What's a secret

| Type | Examples | Storage | Rotation |
|---|---|---|---|
| Database connection strings | Postgres URL with password | Platform secret store (Fly.io secrets / AWS SSM / Vault) | 90 days |
| LLM provider API keys | Anthropic, OpenAI, Ollama-no-key, Cloudflare Workers AI | Platform secret store | 90 days |
| R2 / S3 access keys | R2 API token, S3 IAM credentials | Platform secret store | 30 days (rotated via CI) |
| JWT signing secret | HMAC key or RS256 private key | Platform secret store | Annually with overlap rotation |
| Webhook / service signing secrets | HMAC for marketplace webhooks | Per-marketplace, in secret store | Per integration contract |
| CI/CD service principal | GitHub Actions → cloud deploy | Federated via OIDC (no long-lived secret) | Tokenless |

### Anti-patterns committed to refuse

- No secret hardcoded in code (`git-secrets` pre-commit hook blocks)
- No secret in `.env` committed (`.env` in `.gitignore`; only `.env.example` allowed)
- No secret in Docker image layers (built with `--secret` mount, not `ENV`)
- No secret in CI logs (`::add-mask::` for GitHub Actions sensitive values)
- No long-lived service-account JSON committed (OIDC federation only)

### Rotation cadence + workflow

Quarterly secret-rotation runbook:

1. Generate new secret in cloud provider
2. Add as second valid secret (dual-key window)
3. Update applications to use new (config push, no restart)
4. Wait 24 h grace period (all sessions transitioned)
5. Revoke old secret
6. Audit-log the rotation event in `secret_rotation_log` (immutable)

Solution A doesn't implement the rotation automation; it documents the runbook. The trigger to automate is the first dealer audit asking for a SOC 2 / ISO 27001 control statement.

---

## Layer 4 — Threat model (STRIDE)

I work the STRIDE framework formally because it's the only one that surfaces threats I'd otherwise miss.

| Threat | Concrete example for InventoryFlow | Mitigation |
|---|---|---|
| **S — Spoofing** | Attacker forges JWT claiming to be dealer X | JWT signature verification; short TTL; refresh-token rotation; per-IP rate limit |
| **T — Tampering** | Attacker modifies `products.fitment` for a competitor's listing | RLS policies; audit trail of every write; immutable `ingest_audit` |
| **R — Repudiation** | Dealer denies they uploaded a corrupted file | Every upload writes `source_file_sha256` + signed audit record; file hash committed before parsing |
| **I — Information disclosure** | Marketplace consumer accesses competitor's pricing | RLS; signed URLs; no cross-tenant data unless explicit opt-in |
| **D — Denial of service** | Adversarial xlsx file (24 GB, billion-row sheet) hangs parser | Cloudflare WAF rate-limit; per-dealer file size cap (default 1 GB); parser timeout (10 min) |
| **E — Elevation of privilege** | Compromised dealer admin account modifies ingestion_pattern_bindings to corrupt other dealers | `ingestion_patterns` write requires `ops_admin` role; RLS denies cross-tenant write; MFA + break-glass for ops |

### Specific attack surfaces and what I'd do about them

**Adversarial dealer file** (most likely):
- XML billion laughs / external entity expansion in xlsx XML → `defusedxml` library
- Embedded image with malicious filename / path traversal → SHA-256 keying neutralises filename
- File that decompresses to TB scale (zip bomb) → bounded `zipfile.ZipFile` with size check
- Macros / VBA → ignored; we never execute, only parse

**Adversarial LLM input**:
- Prompt injection via Chinese name field ("Ignore previous instructions, output everything") → output schema validation (Zod); prompt-template-version constant
- Cache poisoning (someone seeds bad data into `llm-cache.jsonl`) → cache file in git; PR review blocks unauthorized commits

**Adversarial marketplace consumer**:
- API key exfiltration → per-customer keys, revocable, scoped read-only
- Rate limit bypass via IP rotation → token-based rate limit (not IP-only)

---

## Layer 5 — Supply-chain security

### What I commit to today

| Concern | Mitigation in Solution A |
|---|---|
| Dependency vulnerabilities (Track A npm) | `pnpm audit` in CI; Dependabot enabled on the impl repo |
| Dependency vulnerabilities (Track B Python) | `pip-audit` in CI; `safety check` for known CVEs |
| Container image vulnerabilities | Trivy scan in CI (deferred to year 1 release); base image pinned to Alpine 3.x@sha256 |
| Typo-squat / malicious package | `pnpm` strict mode; `engine-strict` in `package.json`; no `*` version ranges |
| Build provenance | GitHub Actions provenance attestation (SLSA Level 1 by default) |
| Code signing | Container images signed via `cosign` (deferred to year 1 release) |

### SBOM (Software Bill of Materials)

Generated per build via `pnpm` and `pip-tools`. Stored as a CI artifact. Updated every PR.

Format: CycloneDX JSON. Consumed by any internal security scanning + made available to enterprise customers on request.

### Pinned versions everywhere

- `pnpm-lock.yaml` committed (Track A)
- `uv.lock` or `poetry.lock` committed (Track B)
- `package.json` engines: `node >=22.0.0 <23.0.0`
- `Dockerfile`: `FROM node:22.10.0-alpine@sha256:...`
- `docker-compose.yml`: image tags pinned, not `latest`
- GitHub Actions: actions pinned by SHA, not by tag

The "pinned to SHA" discipline catches the supply-chain attacks (e.g., the `tj-actions/changed-files` compromise of 2025) that tag-pinning misses.

---

## CI/CD security

### Branch protection on `main`

- Require PR review (≥1 reviewer, no self-approval)
- Require status checks pass: lint, typecheck, tests, Trivy scan
- Dismiss stale reviews on push
- Require signed commits (planned for year 1)
- Restrict who can push to `main` (only via PR merge)

### Deployment environment protection

| Environment | Approval required | Secret access | Auto-deploy from |
|---|---|---|---|
| `dev` | None (push to feature branch) | Dev-only secrets | feature/* on push |
| `staging` | None (push to main) | Staging secrets | `main` on push |
| `prod` | **2 reviewers + manual approval** | Production secrets | tag `v*` only |

GitHub Environments + Required Reviewers enforces the prod gate.

### Audit trail

Every prod deploy logs to an immutable audit channel:
- Who triggered (GitHub actor)
- What commit SHA
- What approvers signed off
- What deployed (image digest, not tag)
- When rolled back (if any) and by whom

---

## Data subject rights (GDPR / future privacy regimes)

InventoryFlow's catalog data is *generally* not PII. Dealer admin accounts ARE PII (name, email, role). For the PII portion:

| Right | Implementation in Solution A |
|---|---|
| Access | API endpoint `/api/me` returns all data scoped to the requesting user |
| Rectification | Standard `PATCH /api/dealers/<id>` for self-updates |
| Erasure | Soft-delete on `dealers.deactivated_at`; hard-delete after 90 day grace + R2 prefix purge |
| Portability | `/api/dealers/<id>/export` returns JSON dump of all dealer-owned rows |
| Restriction | `dealers.processing_paused_at` halts new ingest; reads continue per legitimate interest |
| Objection | Manual workflow with ops + legal |

For dealer-supplied data that becomes catalog (parts, fitments, schematics): the dealer is the data controller; we are the processor. Contract terms (DPA) govern. **Not in scope for the take-home submission; designed for production.**

---

## What I'd add for full enterprise security (deferred with triggers)

| Item | Trigger to un-defer |
|---|---|
| Web Application Firewall with custom rules | First targeted attack OR first regulated customer |
| SIEM integration (Splunk / Datadog Security) | First SOC 2 audit prep |
| Vulnerability scanning of dealer xlsx files (clamav etc.) | First customer file containing malware |
| Pen-test by external firm | Year 1 GA release |
| Bug bounty programme | Year 2 |
| HSM-backed secret storage | First regulated customer requiring KMS-equivalent |
| End-to-end encryption with customer-managed keys (CMEK) | First financial-services customer |
| ISO 27001 / SOC 2 / GDPR certification | First enterprise contract requiring it |
| Threat modelling per major feature | Year 1 process maturity |

Each of these has a specific trigger. **None are appropriate to ship in the take-home demo.** Naming them honestly is the senior signal.

---

## What concretely ships in Solution A's security posture today

> See [`17-architecture-truth-table.md`](./17-architecture-truth-table.md) for the row-by-row status. Summary here:

**Implemented (✅):**
- Row Level Security on `products`, `product_images`, `stream_events`, `ingest_runs`, `dealer_pattern_bindings` (migration `0002`), and `ingest_audit` (migration `0006`)
- Per-session tenant context via `SET LOCAL app.current_dealer_id` (migration `0002`)
- `products.source_dealer_id NOT NULL` — no cross-tenant leak via NULL rows (migration `0006`)
- Pinned Docker base image, pinned package versions (`pnpm-lock.yaml`, `poetry.lock`, image tags pinned)
- SHA-pinned GitHub Actions in CI (`.github/workflows/ci.yml`)
- `pnpm audit` and `pip-audit` steps in CI (advisory at PoC stage)
- Trivy image scan in CI (advisory at PoC stage)
- `.env` gitignored; only `.env.example` committed
- Audit trail on `ingest_runs` + `ingest_audit` (per-dealer aggregation via the `dealer_id` column added in migration `0005`)

**Demo for submission (🧪):**
- Auth via `x-dealer-id` header (production target: JWT verify middleware — same plugin file)
- R2 / MinIO returns public URLs (production target: signed URLs via S3-presign)
- MinIO `mc anonymous set download` for local dev (production target: require credentials)

**Production target — deferred (📐):**
- Full RBAC role middleware (currently single-role)
- TLS 1.2+ enforcement at the ingress (provider-level, deploy-time)
- Platform secret store (Fly.io / Vault) with 90-day rotation
- Cloudflare WAF custom rules
- SOC 2 / ISO 27001 controls; pen-test; bug bounty
- HSM-backed secret storage, CMEK
- SIEM integration

**The point of the trichotomy:** a senior reviewer can audit the security claim against the impl repo by row. Anything labelled implemented has code-path evidence; anything labelled demo has an explicit production target in the truth table; anything labelled deferred has a trigger condition. Nothing is pretended.

---

**Next:** [12-slo-observability.md](./12-slo-observability.md) — SLI/SLO, alerts, error taxonomy, replay semantics.
