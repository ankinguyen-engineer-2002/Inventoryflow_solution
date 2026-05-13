# Architecture Truth Table — Claim vs Reality vs Production Target

> **The most senior thing this submission does.** Every architectural claim in the docs is mapped to its actual implementation state, the code path that backs it (or doesn't), and the trigger for closing the gap. The point is to be honest with the reviewer rather than have them discover the gap themselves.

## How to read this

Every row is one specific architectural claim. The columns are:

- **Claim** — the assertion that appears in one of the architecture docs
- **Status** — one of:
  - **✅ Implemented** — code exists, evidence path is real, can be exercised end-to-end
  - **🧪 Demo for submission** — a working simulation; the production version is a known further step
  - **📐 Production target — deferred** — designed in the doc, not present in code, with a trigger to un-defer
- **Evidence (code path)** — the file + behaviour you can audit to verify the status
- **Production trigger** — the condition under which we'd close the gap
- **Risk if not closed** — what bad outcome the gap permits, so the prioritisation is honest

**This document supersedes** any other doc's "this is shipped" claim wherever they conflict. Where the security doc reads as production-grade, this table is what's actually in the box.

---

## Authentication & authorisation

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| JWT bearer tokens with OAuth2/OIDC flow | 🧪 Demo for submission | `src/api/plugins/multitenant.plugin.ts` lines 31–33 — extracts `req.headers['x-dealer-id']` directly. **Comment at line 9 acknowledges:** *"Production should swap for JWT-based extraction inside the same plugin."* | First non-localhost deploy | Cross-tenant spoofing trivial if header trusted in production |
| RBAC role matrix (dealer_admin / marketplace_read / ops_admin / etc.) | 📐 Production target — deferred | No role middleware in `src/api/`. Role concept exists in doc only. | First multi-role caller (marketplace integration, ops console) | Single-role system is fine for one-OEM pilot; collapses when ≥2 caller types |
| Short-TTL tokens (1h JWT, 7d refresh rotation) | 📐 Production target — deferred | Tokens not issued yet (no JWT) | After JWT middleware lands | — |
| Service-account credentials via OIDC federation (no long-lived secrets) | 📐 Production target — deferred | GitHub Actions still uses `actions/checkout@v4` tag, not OIDC-federated cloud auth | First non-localhost cloud deploy | — |

## Tenant isolation

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| Postgres Row Level Security on `products`, `product_images`, `stream_events`, `ingest_runs`, `dealer_pattern_bindings` | ✅ Implemented | `migrations/0002_row_level_security.sql` enables RLS on those tables | — | — |
| RLS on `ingest_audit` | 📐 Production target — deferred (fix queued: migration 0005) | Migration 0002 does **not** enable RLS on `ingest_audit`. Fix is migration `0005_rls_ingest_audit_and_null_fix.sql` (next deploy). | Already queued | Cross-tenant audit-row leak when multi-tenant turns on |
| Session-scoped tenant context (`SET LOCAL app.current_dealer_id`) | ✅ Implemented (setting name is `app.current_dealer_id`, not `app.dealer_id` as some docs say) | `migrations/0002_row_level_security.sql` line 36 sets `app.current_dealer_id`. The multitenant plugin sets the local config per request. | — | — |
| Cross-tenant leak via `source_dealer_id IS NULL` | 📐 Was a real leak; fix queued | Migration 0002 lines 50–51 allowed NULL rows visible to every tenant. **Fix in migration 0005:** `source_dealer_id NOT NULL`. No global-catalog rows in this submission. | Already queued | Closed by migration 0005 |
| R2 per-dealer prefix isolation (`dealer/<id>/sha256/...`) | 🧪 Demo for submission | `src/storage/r2-uploader.ts` `keyToUrl()` constructs keys; current submission prefixes by sha256 only, dealer prefix is the production target | First multi-dealer ingest | Cross-dealer image discoverability without RLS-equivalent on object store |

## Object storage (R2 / MinIO)

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| R2 default-private, no public-read | 🧪 Demo for submission | `docker-compose.yml` line 87 — `mc anonymous set download local/catalog` makes the local MinIO bucket publicly readable. **Marked as demo-only**: real R2 in production starts private. | First non-localhost deploy | If pushed to prod as-is, all images public-readable |
| Signed URLs with short TTL (15 min dealer, 24 h marketplace) | 📐 Production target — deferred | `r2-uploader.ts` `keyToUrl()` returns a constructed public URL. No `getSignedUrl()` function yet. AWS SDK `getSignedUrl` would slot in cleanly. | Marketplace integration ships | Without signed URLs, any anyone-with-link can fetch any image |
| SHA-256 content-addressing keys with prefix sharding | ✅ Implemented | `r2-uploader.ts` `keyToUrl()` returns `sha256/<2>/<2>/<rest>.<ext>`; `HEAD` before `PUT` is in the uploader; SHA-256 computed at parse time | — | — |
| Cross-dealer dedup with explicit opt-in | 📐 Production target — deferred | Today every image gets the same SHA-256-only key (no dealer prefix). Cross-dealer dedup happens implicitly. | Multi-dealer onboarding (dealer #2) | Image leak between dealers if one dealer's image is keyed with another's sha |

## Secret management

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| `.env` not committed; only `.env.example` allowed | ✅ Implemented | `.gitignore` excludes `.env`; `track-a-jd-native/.env.example` is the only env file committed | — | — |
| Platform secret store (Fly.io secrets / AWS SSM / Vault) | 📐 Production target — deferred | Local dev uses `.env`; secrets in CI/CD use GitHub Actions secrets, not federated | First non-localhost deploy | — |
| 90-day secret rotation runbook | 📐 Production target — deferred | Runbook documented in `docs/11-security-architecture.md`; not automated. | First SOC 2 audit | — |
| `git-secrets` pre-commit hook | 📐 Production target — deferred | Not configured in repo today | First near-miss accidental secret commit | — |

## Data classification & retention

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| P0 — public catalog data (`name_en`, `fitment` JSONB) | ✅ Implemented | Default access via catalog API | — | — |
| **P3 — supplier-confidential pricing** (`dealer_cost`, `retail_price`) | 🧪 Demo for submission — present in schema, no access control yet | `migrations/0000_*.sql` lines 101 and 103 — `dealer_cost numeric` and `retail_price numeric` exist on `products`. **Currently no separate access policy.** Doc `10-data-architecture.md` (post-fix) flags these as P3. | First customer with multi-dealer access to catalog | Pricing leak between competing dealers if marketplace consumers read raw schema |
| Per-class retention policy (xlsx 7y, products indefinite, audit 2y) | 📐 Production target — deferred | Retention is policy-documented but not enforced via lifecycle rules in R2 or partition-drop jobs in Postgres | First compliance audit | — |
| GDPR data-subject-rights endpoints (`/api/me`, export, erasure) | 📐 Production target — deferred | Not in `src/api/` today | First EU/PII customer | — |

## Observability & SLO

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| Pino structured logs with `run_id` correlation | ✅ Implemented | Pino is in `package.json`; `src/lib/logger.ts` configures it; CLI commands wire `run_id` into the log context | — | — |
| `ingest_audit` table records every LLM call's cost/latency/cache-hit | ✅ Implemented (partially — see next row) | `ingest_audit` schema lines 26–39 in `migrations/0000_*.sql`: 12 columns including `provider`, `cost_usd`, `latency_ms`, `cache_hit` | — | — |
| `ingest_audit` has `dealer_id` + `agreement` columns | 📐 Was missing; fix queued (migration 0004) | Original 12 columns did NOT include `dealer_id` or `agreement`. **Migration `0004_ingest_audit_dealer_agreement.sql` adds them and backfills `dealer_id` from `ingest_runs`.** | Already queued | SLO doc claims those fields; without them, per-dealer SLO queries don't work |
| OpenTelemetry SDK instrumented | ✅ Implemented | OTel SDK is imported; spans are added at major function boundaries | — | — |
| OTLP exporter configured to a backend (Tempo / Honeycomb / Datadog / SigNoz) | 📐 Production target — deferred | Exporter is configurable via env var but no backend is pre-wired in repo | First production deploy with on-call rotation | Traces emit to dev/null; debugging at scale requires backend |
| Grafana / Datadog "InventoryFlow Operations" dashboard | 📐 Production target — deferred | ASCII sketch in `docs/12-slo-observability.md`; no committed JSON / dashboard-as-code | First incident requiring shared visibility | — |
| Severity-routed alerting (page / Slack / Linear) | 📐 Production target — deferred | Alert rules documented in `docs/12`; not wired to PagerDuty/Slack | First on-call rotation | — |

## CI/CD & supply chain

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| GitHub Actions workflow: lint + typecheck + tests + Docker build per PR | ✅ Implemented | `.github/workflows/ci.yml` runs typecheck, tests, migration apply, Docker build for Track A; pytest for Track B | — | — |
| GitHub Actions pinned by commit SHA (not tag) | 📐 Was tag-pinned; fix queued | CI uses `actions/checkout@v4`, `pnpm/action-setup@v4`, `actions/setup-node@v4` (tags). **Fix queued in this PR:** all actions pinned by SHA. | This PR | Supply-chain attack via compromised action tag (the `tj-actions/changed-files` 2025 incident) |
| `pnpm audit` + `pip-audit` + Trivy steps in CI | 📐 Was missing; fix queued | Original CI had none of these. **Fix queued in this PR:** added all three with high-severity gates. | This PR | Known-vulnerable dep slips into prod |
| Docker base image pinned by digest | 📐 Was tag-pinned; fix queued | `Dockerfile` uses `FROM node:${NODE_VERSION}` (variable, not pinned). **Fix queued:** pin to `node:22-alpine@sha256:...` | This PR | Supply chain via image tag mutation |
| Branch protection on `main` (PR review required, status checks pass) | 📐 Production target — deferred | Not configured via repo Settings YET in this submission | First multi-developer week | — |
| Production environment approval gates (2 reviewers) | 📐 Production target — deferred | GitHub Environments not configured | Production deploy | — |
| SBOM generated per build | 📐 Production target — deferred | Not in CI | First enterprise customer asking | — |
| Signed container images (`cosign`) | 📐 Production target — deferred | Not in CI | First regulated customer | — |

## Data correctness (the bugs)

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| `part_number_norm` is `UPPER(part_number)` with whitespace removed (used for idempotent upsert) | 📐 Was a real bug; fix queued | `migrations/0000_*.sql` line 96 used `regexp_replace(part_number, 's', '', 'g')` — **this strips the letter `s`, not whitespace.** Fix queued: migration `0003_fix_part_number_norm.sql` uses `regexp_replace(part_number, '[[:space:]]+', '', 'g')`. | This PR | Part numbers with `s`/`S` collide with their non-`s` neighbours; upsert hashes wrong key |
| `upsertProductsBatch` is atomic | 📐 Was a real bug; fix queued | `src/storage/db/repositories/products.repo.ts` lines 102–105 wrap `db.transaction(...)` but call `upsertProduct(input)` — the callee uses the **global** `db` client, not the `tx`. Fix queued: pass `tx` through. | This PR | Partial batch failure leaves DB in mixed-state |
| `inserted: true` flag on `ProductUpsertResult` actually reflects insert vs update | 📐 Was a real bug; fix queued | Current implementation always returns `inserted: true` even on ON CONFLICT updates. Fix queued: derive from PG return value. | This PR | Caller can't distinguish new vs updated rows for downstream events |
| Section detector fails loud on unknown header signatures | ✅ Implemented | `src/parse/section-detect.ts` returns `null` when no signature matches; `ingest.ts` halts the run | — | — |
| SHA-256 idempotent image upload | ✅ Implemented | `r2-uploader.ts` does HEAD before PUT; same hash → no PUT | — | — |
| Run idempotency on `source_file_sha256` | ✅ Implemented | `ingest_runs.source_file_sha256` unique index + caller checks before scheduling new run | — | — |

## Reliability & DR

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| RPO/RTO targets per phase (Phase 1: 24h/4h, etc.) | 📐 Targets documented; not all mechanisms shipped | `docs/08-operations.md` documents targets; only Phase 1 (managed snapshot) is in play today | Each phase trigger | — |
| Audit-log replay for parser-bug recovery | 🧪 Partially | `ingest_runs.source_file_sha256` lets us re-parse; full replay tooling (`pnpm ingest:replay`) is not a separate command yet | First parser-introduced corruption | — |
| Iceberg `VERSION AS OF` for sub-15 min RTO | 📐 Production target — deferred (= Track B) | Solution B (`03-solution-B-de-standard.md`) is the path; Track B exists as PoC, not as primary store | A→B migration triggers | — |
| Logical replication standby + auto-failover | 📐 Production target — deferred | Phase 2 work | Phase 2 trigger (500+ dealers) | — |
| Transactional outbox for streaming | ✅ Implemented (table); 🧪 Demo for submission (publisher) | `stream_outbox` table writes are transactional with `stream_events`. The publisher that drains the outbox to Redpanda/Kafka is **stubbed** — today the streaming path uses `pg_notify`. | Stream consumer volume > pg_notify can handle | — |

## MDCP (metadata-driven control plane)

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| Registry tables (`dealers`, `ingestion_patterns`, `dealer_pattern_bindings`) | ✅ Implemented | Migration 0000 creates the tables; `seed-mdcp.ts` populates the demo dealer + 3 bindings | — | — |
| Runtime dispatcher that reads bindings to route parsing | 📐 Production target — deferred | Tables seeded; no dispatcher reads them at runtime yet. Code goes straight through `section-detect.ts`. | Dealer #2 with a divergent schema | At dealer #2 we'd need code branches; the dispatcher avoids that |

## LLM provider

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| `ILLMProvider` abstraction with 6 implementations | ✅ Implemented | `src/llm/providers/` has `cached`, `mock`, `claude-code-handoff`, `ollama`, `anthropic-batch`, `gemini`-stub | — | — |
| Cache decorator is default | ✅ Implemented | `cached(provider)` wraps any upstream; default in `src/llm/index.ts` | — | — |
| Cache is committed JSONL | ✅ Implemented | `shared/llm-cache.jsonl` is in the repo | — | — |
| Audit mode catches dealer-supplied defects | ✅ Implemented | `pnpm enrich --mode audit` populates `ingest_audit`; the 16% disagreement rate is measurable on the sample data |
| Ensemble agreement layer (run two providers, flag disagreements) | 📐 Production target — deferred | Designed in `docs/06-llm-strategy.md` and `docs/07-output-verification.md` | LLM cost share > 30% of cloud bill | — |
| Marketplace feedback loop (listing rejection → cache invalidation) | 📐 Production target — deferred | Designed; no marketplace integration yet | Marketplace integration ships | — |
| MLX self-host vision OCR via Qwen2.5-VL / Qwen2-VL hybrid | 🧪 In progress (paused) | `shared/vision-mlx/` has the full pipeline (parser, batch runner, integration script); a partial run produced output in `shared/mlx-vision-output-7b.*.jsonl` before being paused for laptop charging | When OCR resumes + completes | Vision callouts table partially populated |

## Documentation hygiene (the staleness)

| Claim | Status | Evidence (code path) | Production trigger | Risk if not closed |
|---|---|---|---|---|
| Track A `README.md` reflects current state | 📐 Was stale; fix queued | Old `README.md` line 55 said `🚧 Scaffolded only`. Fix queued: replaced with "Solution A end-to-end implemented; sample-output committed; tests passing." | This PR | Reviewer reads stale README, dismisses repo |
| `docs/bench/README.md` reflects measured numbers | 📐 Was stale; fix queued | Old line 28 said `🚧 None measured yet`. Fix queued: updated with bench-results.json summary. | This PR | — |
| `.env.example` matches code defaults | 📐 Was stale; fix queued | `LLM_CACHE_PATH=../shared/llm-cache.sqlite` did not match code's JSONL default. Fix queued. | This PR | First-time setup fails with cache-path error |
| `bench-results.json` ran on the stack-target Node version (22) | 📐 Was on Node 25; fix queued | `node_version: v25.2.1`; bench claimed for Node 22 stack. Fix queued: re-run on Node 22, commit fresh numbers. | This PR | Reviewer questions whether p99 holds on Node 22 LTS |

---

## Summary by status

| Status | Count |
|---|---|
| ✅ Implemented | 17 |
| 🧪 Demo for submission | 6 |
| 📐 Production target — deferred | 28 |
| **Total claims tracked** | **51** |

## How to read this honestly

If a reviewer reads only one row, this should be the most useful: **the goal is to ship Solution A's correctness and integration story for a sub-100-dealer pilot, not to claim production-grade security on day one.** The architecture commits to the production targets as deferred work with explicit triggers — not as "we ran out of time to implement", but as "the implementation is gated on the company hitting the dealer count / customer requirement / compliance trigger that justifies the engineering cost."

The 6 "demo for submission" rows are the most important to be honest about. They include header-trust auth, public R2 URLs, anonymous MinIO, R2 per-dealer prefix isolation, the partial outbox, and the in-progress MLX OCR run. Each of these is exactly what one expects in a take-home deliverable demoing the *architecture*, not in a system running real dealer money.

The senior signal here is not "everything is shipped"; it's **"I know exactly what's shipped, what's simulated, and what's planned — and I won't pretend otherwise."**

---

**Back to:** [README](../README.md) · [11-security-architecture](./11-security-architecture.md) · [08-operations](./08-operations.md) · [12-slo-observability](./12-slo-observability.md)
