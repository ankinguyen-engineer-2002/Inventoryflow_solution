# InventoryFlow Solution Briefing — Master Reference

> **PURPOSE**: Single-file comprehensive reference for the entire solution architecture. Designed for both human reading (skim → deep dive) and AI consumption (structured, chunked, scannable).
>
> **WHEN TO USE**: When you need fast answers about *why this solution*, *why this setup*, *how it handles problem X*, or *what trade-offs were considered*. Each section is self-contained — an AI can answer from a single section without reading the rest.
>
> **CONVENTIONS**:
> - Claims tagged `[verified]` `[likely]` `[need-verify]` `[speculation]` per §0 zero-hallucination rule
> - Tables preferred over paragraphs
> - Decision trees rendered as plain text
> - Anchors for cross-reference: `→ §3.A.4` means Section 3, Solution A, point 4
> - Status: ✅ Implemented · 🧪 Demo for submission · 📐 Production target — deferred
>
> **AUTHOR'S VOICE**: First-person ("I think", "my experience") because this is a senior consultant deliverable, not a corporate doc.

---

## TABLE OF CONTENTS

```
§1   CONTEXT
     1.1 The brief
     1.2 What the brief is really testing
     1.3 My constraints (CV, role, hardware)

§2   DECISION FRAMEWORK
     2.1 Four solutions considered
     2.2 Six quantified A→B migration triggers
     2.3 Decision tree by volume + maturity

§3   SOLUTION A — Postgres + JSONB + TS/Node (recommended)
     3.A.1  TL;DR
     3.A.2  Stack 1:1 mapping to JD
     3.A.3  Architecture — five planes
     3.A.4  Data model — three shapes
     3.A.5  LLM strategy — self-host + cache
     3.A.6  Image OCR — vision pipeline
     3.A.7  Multi-tenancy + RLS
     3.A.8  Cost model
     3.A.9  Why this and not B/C/D
     3.A.10 What ships vs deferred
     3.A.11 Trade-offs I accept
     3.A.12 Setup runbook (concrete)

§4   SOLUTION B — Iceberg + Trino + Dagster + dbt + Polars
     3.B.1  TL;DR
     3.B.2  Stack
     3.B.3  Architecture
     3.B.4  Why this beats A above triggers
     3.B.5  Cost model
     3.B.6  Why I didn't ship this first
     3.B.7  Setup runbook

§5   SOLUTION C — Microsoft Fabric
     3.C.1  TL;DR
     3.C.2  Stack mapping
     3.C.3  When Fabric wins
     3.C.4  When Fabric loses
     3.C.5  Cost gating issue
     3.C.6  Hire-ability signal

§6   SOLUTION D — AWS big-data stack
     3.D.1  TL;DR
     3.D.2  Stack
     3.D.3  When AWS wins
     3.D.4  Cost
     3.D.5  Operational complexity

§7   LLM DEEP DIVE
     7.1  The four genuine alternatives
     7.2  Self-host on personal hardware (Apple Silicon)
     7.3  Self-host on cloud GPU
     7.4  Paid API (frontier vision models)
     7.5  Hybrid (cache + API fallback)
     7.6  Cost matrix by volume
     7.7  Real timing data — 1573 images
     7.8  GPU watchdog quirk on M1 Max
     7.9  max_tokens tuning
     7.10 Resize cap (1024px) — why
     7.11 Classical OCR preprocessing — why I skipped
     7.12 Cost discipline — the architectural lever ($2.50 vs $2,500)
     7.13 Fine-tuning vs prompt engineering vs cache
     7.14 Pure OCR alternative — why it loses for this workload
     7.15 Lessons from local self-host (5 production implications)
     7.16 My single most important paragraph (the policy decision)
     7.17 Provider abstraction (the interface)

§8   ACCURACY VERIFICATION (AI Engineer perspective)
     8.1  Five-layer accuracy framework
     8.2  Layer 3 — cross-row consistency
     8.3  Layer 4 — cross-source agreement (ensemble)
     8.4  Coverage rate vs parts table (ground truth signal)
     8.5  Disagreement detection
     8.6  Confidence tiering
     8.7  Manual sampling cadence
     8.8  The senior policy decision

§9   IMPLEMENTATION DETAILS
     9.1  Schema (12 tables, 3 shapes)
     9.2  Idempotency mechanisms
     9.3  Migration journal
     9.4  CI/CD pipeline
     9.5  Docker setup
     9.6  Observability instrumentation

§10  SECURITY ARCHITECTURE
     10.1 Five layers (IAM, tenant, secret, supply chain, audit)
     10.2 Threat model (STRIDE-derived)
     10.3 Shipped vs Demo vs Deferred

§11  OPERATIONS & SLO
     11.1 SLI definitions
     11.2 Error taxonomy
     11.3 Runbook entries
     11.4 DR / BCP

§12  COST CLAIMS — TAGGED AUDIT
     12.1 All $ figures with verification status

§13  PANEL ANSWERS (anticipated questions)
     13.1 "Why JSONB not normalized?"
     13.2 "Why not Iceberg?"
     13.3 "Why local 7B not GPT-4o?"
     13.4 "How do you know it's accurate?"
     13.5 "Where's the threat model?"
     13.6 "What if dealer count grows 100x?"
     13.7 How to pitch the results (clean framing + vocabulary)

§14  APPENDICES
     14.1 File map (solution repo)
     14.2 Cross-link to impl repo
     14.3 Reading order by panel role
```

---

## §1 CONTEXT

### 1.1 The brief

```yaml
TASK: InventoryFlow take-home test — Talemy × InventoryFlow Senior Engineer role
INPUT: 241 MB Kayo ATV xlsx catalog
  - 110 sheets (one per ATV model)
  - 1,586 embedded schematic images
  - Bilingual: English + Chinese part names
  - Numbered callouts on each schematic linking part-list to image position
OUTPUT_TARGET:
  - Clean part_number, name_en, name_cn
  - JSONB fitment column (denormalized for marketplace serving)
  - Schematic images in object storage with per-callout linkage
  - AI tooling layer (translation audit, callout extraction, defect detection)
STACK_HINT_FROM_JD: TypeScript/Node 22 + Fastify + PostgreSQL + Drizzle + Redis + BullMQ + Cloudflare R2
```

### 1.2 What the brief is really testing

Not just "can you parse a messy xlsx". My read:

```
1. Stack discipline       → Use the stack the JD names, defend it under pressure
2. Data modeling judgment → JSONB vs normalized — defend the choice
3. AI tooling discretion  → Use AI where it helps, NOT for hype
4. Cost awareness         → Solo founder context, ~$30/dealer not $300
5. Honest about limits    → What you skipped and why
6. Production thinking    → Idempotency, audit trail, RLS, observability
7. Migration path         → When does this stop scaling? Show triggers.
```

### 1.3 My constraints

```yaml
ROLE: Senior Data Engineer / Solution Architect (transitioning from BI/ETL)
LOCATION: Vietnam (Talemy is VN recruiter, role likely cross-border)
HARDWARE_FOR_LOCAL_TESTING: MacBook Pro M1 Max, 64 GB RAM
COST_POLICY: Zero paid API key for submission [memory: feedback_no_api_cost_strategy]
TIMELINE: Take-home spans multiple days — submission must be self-contained
EXPERIENCE_ANCHOR_POINTS:
  - Ashley Furniture: preflight DQ gates, severity tiers
  - Ecentric: multi-checkpoint DQ framework, critical-halt patterns
  - SOPA: data contracts, schema registry
  - ADP: large-scale tenant isolation
```

---

## §2 DECISION FRAMEWORK

### 2.1 Four solutions considered

| ID | Name | Best for | Cost at 1 dealer | Cost at 1000 dealers |
|---|---|---|---|---|
| **A** | Postgres + JSONB + TS/Node | 0–500 dealers, JD-native, ship in days | ~$76/mo `[likely]` | ~$200/mo `[speculation]` |
| **B** | Iceberg + Trino + Dagster + dbt | 500–5000 dealers, schema churn, $ governance | ~$25/mo `[likely]` | ~$500/mo `[speculation]` |
| **C** | Microsoft Fabric | Enterprise with M365, capacity-budget allows | Floor F2 ~$262/mo `[likely]` | F8 ~$1050/mo `[likely]` |
| **D** | AWS big-data | >50 TB, MSK/Kinesis streams, regulated | ~$2900/mo at 1000 dealers `[need-verify]` | scales linearly |

### 2.2 Six quantified A→B migration triggers

`Solution A is enough until one of these fires` — quantified, not vibes:

```
TRIGGER_1: dealer_count > 500
  REASON: Single-instance Postgres + 5 BullMQ workers comfortably handles 50–100 files/day.
  EVIDENCE: Bench p99 fitment query 0.99ms on M1 Max [verified from bench-results.json]

TRIGGER_2: historical_volume > 50 TB
  REASON: Postgres on managed tier costs 5× per TB vs S3 + Iceberg at this size.

TRIGGER_3: LLM_cost_share > 30% of cloud bill
  REASON: Cache-hit rate stabilizes ~99% in steady state. If LLM bill dominates, the
          architectural lever (cache discipline) is exhausted; need self-host or batch API.

TRIGGER_4: OLAP queries contend with OLTP for >10% wall time
  REASON: Marketplace catalog reads compete with dealer writes; need read replica
          or analytics-shape split (the canonical/analytics shape of solution A).

TRIGGER_5: schema churn ≥1 dealer/week with divergent xlsx layouts
  REASON: MDCP runtime dispatcher must take over; section_detect heuristics break
          when ≥3 OEMs ship divergent file shapes.

TRIGGER_6: required RTO < 1 hour
  REASON: Managed-snapshot recovery is ~2-4h. Iceberg VERSION AS OF sub-15min RTO
          (or hot standby with logical replication) needed.
```

### 2.3 Decision tree by volume + maturity

```
START
├── Today, 1 dealer, take-home: ───────────────────→ Solution A
├── 0–500 dealers, single-OEM stack: ───────────────→ Solution A
├── 500–1,500 dealers, schema churn:
│   ├── Enterprise customer requires Microsoft? ───→ Solution C
│   └── Otherwise: ──────────────────────────────→ Solution B
├── 1,500–5,000 dealers, streaming ingest critical:
│   ├── Already on AWS / customers regulated? ───→ Solution D
│   └── Cloud-agnostic preferred: ──────────────→ Solution B (Trino federate)
└── 5,000+ dealers, regulated, multi-region: ────→ Solution D (or Snowflake)
```

---

## §3 SOLUTION A — Postgres + JSONB + TS/Node (RECOMMENDED)

### 3.A.1 TL;DR

```yaml
WHAT: Single-region Fly.io deployment with Postgres 16 + JSONB fitment +
      BullMQ workers + R2 storage + self-hosted vision LLM
WHY: Matches JD stack 1:1, costs ~$30/dealer/month amortised, ships in days,
     defendable migration path to B/C/D at quantified triggers
SHIP_TIME: 3-5 days for an experienced TypeScript backend engineer
COST_AT_SCALE:
  - 1 dealer (today):    ~$76/mo  [likely]
  - 100 dealers:         ~$30/dealer/mo amortised  [speculation - architectural estimate]
  - 500 dealers:         migration trigger fires → Solution B
```

### 3.A.2 Stack 1:1 mapping to JD

| JD asks for | Solution A delivers |
|---|---|
| TypeScript / Node 22 | Node 22 LTS, strict TS, Zod runtime validation |
| Fastify | Fastify 5.x, plugin-based, OpenTelemetry instrumented |
| PostgreSQL | Postgres 16 with JSONB GIN index, RLS, generated columns |
| Drizzle ORM | Drizzle for typed query layer + migrations |
| Redis | Upstash pay-as-you-go for BullMQ |
| BullMQ | 5 workers parallel, idempotent on `source_file_sha256` |
| Cloudflare R2 | SHA-256-keyed images, zero egress fees |
| AI tooling | Provider-abstraction with cache + audit + ensemble (designed) |

**Implication**: nothing in the JD is unused; nothing is stretched. A panel auditing stack discipline finds 1:1 match.

### 3.A.3 Architecture — five planes

```
PLANE_1_INGRESS:    Fastify HTTP API + signed-upload endpoints
                    │
PLANE_2_CONTROL:    BullMQ job orchestration, ingest_runs registry, MDCP bindings
                    │
PLANE_3_DATA:       Worker pool — streaming xlsx → section detect → row normalize → DB upsert
                    │
PLANE_4_INTELLIGENCE: LLM provider abstraction (6 impls), cache decorator, audit mode
                    │
PLANE_5_STORAGE:    Postgres (canonical + serving) + R2 (binary objects) + Redis (queue)
                    │
PLANE_6_OBSERVE:    Pino structured logs + OpenTelemetry + Prometheus metrics + ingest_audit
```

Each plane has a single responsibility — diagnosability comes from clean boundaries.

### 3.A.4 Data model — three shapes

**The most important architectural decision in Solution A.** I run the SAME canonical data through three serving shapes:

```yaml
SHAPE_1_SERVING_JSONB:
  PURPOSE: Hot path for marketplace "find products fitting vehicle X"
  IMPLEMENTATION: products.fitment JSONB + GIN jsonb_path_ops index
  TRADE_OFF: Denormalized; UPDATE of a single fitment row touches whole JSONB doc
  WHEN_USED: GET /api/products?vehicle=AT125-B → <2ms p99 [verified, bench-results.json]

SHAPE_2_CANONICAL_NORMALIZED:
  PURPOSE: Governance, GDPR right-to-delete, audit, joins for analytics
  IMPLEMENTATION: fitment_canonical table normalized: (product_id, year, make, model, model_code, variant)
  TRADE_OFF: Slower for marketplace lookup; faster for "find all products affected by recall on model X"
  WHEN_USED: dbt model materializes from products.fitment; ops queries hit this shape

SHAPE_3_ANALYTICS_WIDE:
  PURPOSE: BI / Power BI / dbt models; aggregations across dealers
  IMPLEMENTATION: vw_product_fitment_wide materialized view (or external dbt to OLAP)
  TRADE_OFF: Eventual consistency vs serving; refresh cadence ~hourly
  WHEN_USED: Cross-dealer reporting, vendor scorecards, fitment coverage analysis
```

**Why three shapes**: a junior engineer picks ONE shape and defends it; a senior recognizes that different consumers need different shapes and **the same canonical data is materialized into each**.

This is the senior architectural reply to "why JSONB and not normalized?": *"I chose JSONB for serving. The canonical shape is normalized. The analytics shape is wide. They are not in conflict — they are different views of the same source of truth."*

### 3.A.5 LLM strategy — self-host + cache

```yaml
DECISION_TREE:
  - 1-shot test submission, $0 API budget:
      └── Self-host with Ollama or MLX on Apple Silicon
  - Production with <10k calls/month:
      └── Self-host with cache decorator default-on
  - Production 10k-100k/month:
      └── Self-host 7B model on GPU box ($200/mo) + paid API for misses
  - Production >100k/month:
      └── Cluster deployment + light fine-tuning + ensemble agreement layer

PROVIDER_ABSTRACTION:
  - mock                          # tests, deterministic safety fallback ($0)
  - cached(provider)              # DEFAULT decorator — wraps any upstream ($0 on hit)
  - claude-code-handoff           # dev cache seeding via Claude Max subscription ($0)
  - ollama                        # production self-host ($0 + hardware)
  - anthropic-batch               # cloud production via batch API (~$0.0005/call) [verified rate]
  - gemini                        # alternative cloud provider (rate similar)

CACHE_AS_ARCHITECTURAL_LEVER:
  CLAIM: 10-100× cost reduction vs sloppy implementation
  MECHANISM: Cache decorator is DEFAULT. You have to actively turn it off.
  EVIDENCE: 99% hit rate steady-state on dictionary-style translation workload
            (same Chinese term repeats across products/dealers) [speculation - typical]
```

### 3.A.6 Image OCR — vision pipeline

```yaml
MODEL: Qwen2.5-VL-7B-Instruct-8bit (MLX framework, Apple Silicon native)
WHY_THIS:
  - 7B sweet spot for M1 Max 64GB (8.3 GB weights, 8.8 GB peak per worker)
  - JSON adherence ~95% (vs 87% on 2B-4bit — empirically measured)
  - Vision capability matches the task: extract numbered callouts from schematic
  - Local self-host = $0 marginal cost per inference [memory: feedback_no_api_cost_strategy]

ACTUAL_TIMING_ON_M1_MAX_64GB:
  WORKERS: 3 parallel × 7B-8bit (RAM-bound — 4+ workers hits 32 GB wired ceiling)
  AVG_LATENCY: 25-35s per image with RESIZE_LONGEST_EDGE=1024 cap
  THROUGHPUT: ~5-6 images/min total
  RUN_TIME_FOR_1573_IMAGES: ~4-5 hours total (with occasional GPU-watchdog restart)
  KNOWN_QUIRK: 1 in 3 workers dies with kIOGPUCommandBufferCallbackErrorTimeout
              every 30-60 min when worker hits a "heavy" image (busy schematic)

PHASE_PIPELINE (all 5 stages shipped + verified live in DB, measured numbers below):
  PHASE_1_OCR:           7B on all 1573 images → 1463 OK (93.0%), 110 fail
  PHASE_2_REFINEMENT:    Anti-loop retry on 110 fails → 39 recovered (35.5%)
  PHASE_3A_LAYER3:       Internal consistency check (duplicate_n, pos hallucination)
                         Demoted 264 records from naive "Phase 1 OK"
  PHASE_4_LAYER4:        Ground-truth cross-reference vs parts_table xlsx
                         Per-image precision + per-sheet union coverage
                         Demoted 359 more records (the value Layer 4 adds)
  PHASE_5_DB_INTEGRATE:  1573 rows upserted into image_callouts via psycopg
                         Live verified: HIGH 42.9% / MEDIUM 29.7% / LOW 27.4%
```

### 3.A.7 Multi-tenancy + RLS

```yaml
PRIMARY_MECHANISM: PostgreSQL Row Level Security
SESSION_CONTEXT: SET LOCAL app.current_dealer_id = '<from JWT claim>'
DEFAULT_DENY: Postgres rejects query if context not set — bug-tolerant safety

TABLES_WITH_RLS:
  - products              (source_dealer_id NOT NULL after migration 0006)
  - product_images
  - ingest_audit          (added migration 0006; tenant-scoped via ingest_runs join)
  - ingest_runs
  - stream_events
  - dealer_pattern_bindings

POLICY_SHAPE:
  CREATE POLICY products_tenant_scope ON products
    USING (source_dealer_id = current_setting('app.current_dealer_id')::uuid
        OR current_setting('app.current_dealer_id') = 'all');

WHY_DAY_ONE:
  Even at 1 dealer, RLS is enabled. Retrofitting RLS to multi-tenant is a 2-week
  project with downtime risk. Enabling on day one is a config flag.
```

### 3.A.8 Cost model

`Source pricing pages, late 2025 / early 2026. Tagged per claim.`

| Component | Tier | Cost/mo | Tag |
|---|---|---|---|
| Fly.io 2× shared-cpu-2x · 4 GB | Production | $30 | `[likely]` |
| Neon Pro Postgres 1 CU + 10 GB | Serverless | $40 | `[likely]` |
| Upstash Redis pay-as-you-go | Free tier + bursts | $5-10 | `[speculation]` |
| Cloudflare R2 50 GB + 100k ops | $0.015/GB + $4.50/M | $5 | `[verified rate]` |
| Axiom logs + Sentry errors | Both free tier | $0-25 | `[likely]` |
| Cloudflare TLS/DNS/WAF | Free plan | $0 | `[verified]` |
| PagerDuty (1 user) | Free, +$21/user | $0-21 | `[likely]` |
| **TOTAL (1 dealer)** | | **~$76/mo** | architectural estimate |

**Defensible**: "These are order-of-magnitude estimates from published baseline pricing. For a real customer engagement I'd run a 30-day billing simulation with synthetic load. The line items I can defend on rate × quantity are R2 storage, S3 egress, and per-API-call costs."

### 3.A.9 Why this and not B/C/D

| Question | Solution A's answer |
|---|---|
| Why not Iceberg + Trino (B)? | At 1 dealer, B costs the same operationally but takes 3× the engineering time. Postgres + JSONB ships faster, runs simpler. B is the *target* when triggers fire. |
| Why not Microsoft Fabric (C)? | F2 capacity floor is $262/mo `[likely]` — 3.5× A at 1 dealer. Lock-in to M365 ecosystem. Designed for enterprises that already pay Microsoft. |
| Why not AWS big-data (D)? | At 1k dealers, D is ~5× A's cost. Operational complexity (Glue, MSK, EMR) requires a 3-person data team to maintain. Justified only at >5k dealers or regulated industries. |
| Why TypeScript not Python? | JD-native stack. The JD names Node + Fastify + Drizzle explicitly. A pragmatic senior engineer reads the JD and aligns. |

### 3.A.10 What ships vs deferred

**Implemented (✅)**:
- 12-entity schema with RLS, generated columns, GIN index, idempotency keys
- 5-worker BullMQ pipeline with `source_file_sha256` idempotency
- SHA-256 content-addressed R2 upload with HEAD-before-PUT
- 6-implementation LLM provider abstraction + cache decorator default
- ingest_audit with `dealer_id` + `agreement` columns (migration 0005)
- CI workflow: SHA-pinned actions, pnpm audit, pip-audit, Trivy scan
- Pino structured logs with run_id correlation

**Demo for submission (🧪)**:
- Auth via `x-dealer-id` header (production = JWT verify, same plugin file)
- R2 returns public URLs (production = signed URLs via S3 presign)
- MinIO `mc anonymous set download` for local dev (production = credentials required)
- R2 per-dealer prefix isolation (current = sha256-only key)

**Production target — deferred (📐)**:
- Full RBAC role middleware (currently single-role)
- OTLP exporter wired to concrete backend (Tempo/Honeycomb/Datadog/SigNoz)
- Grafana / Datadog operational dashboard
- Severity-routed alerting (PagerDuty for sev-1, Slack for sev-2)
- Outbox publisher draining stream_outbox to Redpanda/Kafka
- MDCP runtime dispatcher reading dealer_pattern_bindings
- Ensemble agreement layer (run 2 LLM providers, flag disagreements)
- Marketplace feedback loop (listing rejection → cache invalidation)

`See impl repo's STATUS.md for the row-by-row truth table.`

### 3.A.11 Trade-offs I accept

```yaml
TRADE_1: JSONB updates are doc-level, not row-level
  ACCEPTED_BECAUSE: 99% of mutations are dealer-batched (new ingest run replaces);
                    individual fitment edits are rare and acceptable as full doc rewrite.

TRADE_2: Cross-dealer global rows are not supported
  ACCEPTED_BECAUSE: source_dealer_id NOT NULL (migration 0006) closes the NULL-leak.
                    Global rows would need a separate is_global flag + explicit policy.

TRADE_3: Single-region deployment, no multi-region failover
  ACCEPTED_BECAUSE: Phase 1 (0-500 dealers) doesn't justify cross-region cost.
                    Trigger 6 (RTO < 1h) is the gate to multi-region or Solution B.

TRADE_4: 2-3% Phase 1 OCR records require Phase 2 retry
  ACCEPTED_BECAUSE: 7B vision model occasionally hallucination-loops on busy schematics.
                    max_tokens=1024 caps wasted GPU time; Phase 2 handles remainders.

TRADE_5: Local LLM costs 4-5 hours wall time per 1573-image batch
  ACCEPTED_BECAUSE: $0 marginal cost vs $30-60 for API frontier. For take-home + early
                    production, the wall time is acceptable. Production scale → API.
```

### 3.A.12 Setup flow (concise)

```
clone → docker-compose up → pnpm install → migrate (6 migrations) → seed MDCP
      → pnpm ingest:full <xlsx> → pnpm query → pnpm bench → (optional) MLX vision OCR
```

Full commands in impl repo README + ADR-002. The flow is intentionally boring — no exotic deployment tooling, just JD-stack defaults.

---

## §4 SOLUTION B — Iceberg + Trino + Dagster + dbt + Polars

### 3.B.1 TL;DR

```yaml
WHAT: OSS lakehouse stack on Hetzner — Iceberg storage, Trino federated query,
      Dagster orchestration, dbt transforms, Polars for in-memory processing
WHY: When dealer count > 500 OR historical volume > 50 TB OR LLM cost share > 30%
COST: ~$25/mo at 1 dealer (Hetzner CX41 + CX21 PG); ~$0.50/dealer at 100 dealers `[likely]`
WHEN: After A's quantified triggers fire — typically year 2 for InventoryFlow
```

### 3.B.2 Stack

| Layer | Component | Why this |
|---|---|---|
| Storage | Apache Iceberg on S3/MinIO | ACID, time-travel, schema evolution, $0.023/GB-mo `[verified]` |
| Compute | Trino | Federated query, no vendor lock-in, OSS |
| Orchestration | Dagster | Asset-based DSL, Python-native, multi-track ergonomics |
| Transform | dbt + dbt-trino | Industry-standard, lineage, tests |
| In-memory | Polars | Lazy evaluation, multi-core, 10-100× faster than pandas |
| Streaming | Redpanda (Kafka-compatible) | Lighter than Kafka, single-binary, OSS |
| Object store | Cloudflare R2 OR MinIO self-host | Zero egress fees for R2 |

### 3.B.3 Architecture

```
INGEST → Redpanda topic → Dagster asset (bronze) → dbt model (silver) → dbt model (gold)
                              │
                              ▼
                       Iceberg table at S3/MinIO
                              │
                              ▼
                    Trino query layer (federate across bronze/silver/gold)
                              │
                              ▼
                    Fastify thin API (read-through Trino)
```

### 3.B.4 Why this beats A above triggers

- **schema evolution**: Iceberg handles add/drop column without table rewrite; Postgres alter table on 50M rows is hours of downtime
- **time travel**: VERSION AS OF for parser-bug recovery; A would need full re-ingest
- **OLAP + OLTP split**: Trino is read-only, no contention with serving; A's analytics view contends with serving
- **multi-engine**: Spark, Flink, Trino can all read the same Iceberg table; A is Postgres-only
- **storage cost**: 50 TB on Iceberg/S3 = $1,150/mo; Postgres-managed at same size ~$3000/mo `[need-verify]`

### 3.B.5 Cost model

| Tier | Hetzner | Postgres | Storage | Total |
|---|---|---|---|---|
| 1 dealer | CX21 €4.51/mo | CX21 €4.51/mo | MinIO local | ~€10 (~$11) `[likely]` |
| 100 dealers | CX41 €15/mo | CX41 separate €15/mo | R2 50 GB $5 | ~€35 (~$40) `[likely]` |
| 1000 dealers | CCX13 €40/mo | CCX13 €40/mo + replica | R2 5 TB $75 | ~€155 (~$180) `[speculation]` |

`OSS stack on Hetzner is ~5-10× cheaper than equivalent AWS managed services at this scale.`

### 3.B.6 Why I didn't ship this first

```
1. Engineering time:    3× longer to ship (3 weeks vs 1 week)
2. JD stack mismatch:   JD names TypeScript; this is mostly Python + dbt
3. 1-dealer overhead:   Trino/Dagster bring complexity that's not justified
4. Migration is cheaper than rewrite: A→B is a real path, not a redo
```

**Parity verified empirically (2026-05-14)**: I ran Track B's Python parser
on the SAME example.xlsx Track A ingested. Wrote Track B output to CSV in
Track A's schema. Diffed against Track A's committed reference.

```json
{
  "track_a_rows": 3938,
  "track_b_rows": 3937,
  "common_part_numbers": 3937,
  "name_en_mismatches": 0,
  "name_cn_mismatches": 10,
  "retail_price_mismatches": 0,
  "fitment_model_match": 3743,
  "fitment_model_mismatch": 0,
  "fitment_year_mismatch": 0,
  "parity_pct": 99.97
}
```

**99.97% parity**. But the 0.03% delta is the interesting bit. I investigated each disagreement:

**Category 1 — 1 header artefact ("U8 Code")**:

Track A's looser parser treated a row containing the literal header tokens (`U8 Code | Model | EN name | Specification`) as a product. Track B's stricter parser skipped it. **Track B is correct.**

**Category 2 — 4 encoding corruption rows**:

```
PN 802007-0371:
  Track A: 前减护板贴花 2�� 513胶      ← U+FFFD replacement char (broken UTF-8 read)
  Track B: 前减护板贴花 2号 513胶      ← correct

PN 405009-0009 / 0020 / 0024:
  Track A: 后刹泵��架                  ← character corruption
  Track B: 后刹泵支架                  ← correct
```

Track A has 4 rows with `��` (U+FFFD = the canonical "broken-character" replacement). Track B reads UTF-8 correctly. **Track B is correct.**

**Category 3 — 6 rows ambiguous (Track A has name_cn, Track B empty)**:

These need the source xlsx open in Excel to adjudicate. Either Track A is over-eager (reading from merged cells / adjacent overflow) or Track B is dropping legitimate data. Without ground truth I won't claim either way.

**Verdict**: of the 11 disagreements, Track B is verifiably correct on **5** (1 header + 4 encoding), Track A is verifiably correct on **0**, and **6 are ambiguous**. The "0.03% noise" is not noise — it's **Track B catching real bugs in Track A** that the parity test surfaced.

**This is the senior architectural argument for having two tracks**: the second implementation isn't redundant work, it's a *verification mechanism*. Two independent parsers on the same input is exactly the cross-validation pattern from Layer 4 of the accuracy framework, applied at the system level.

**Implication for ADR-009 (when to migrate to Track B)**: the migration is not just fidelity-preserving — at the limit of what parity tests can prove, it's **fidelity-improving**. The TypeScript parser has small bugs the Python parser doesn't. A → B is a re-platform with strict semantic-equivalence at 99.97%, and the gap is in favour of B.

Committed evidence in impl repo: `sample-output/track-b/parity-report.json` + `sample-output/track-b/data/products-full.csv` for line-by-line diff against Track A's `sample-output/data/products-full.csv`.

### 3.B.7 Setup flow (concise)

```
Hetzner CX41 → docker-compose: postgres + minio + trino + dagster + caddy(TLS)
            → dbt models → iceberg catalog (Postgres-backed) → Trino query
```

Full compose file in `inventoryflow-catalog-ingest/track-b-data-engineering/`.

---

## §5 SOLUTION C — Microsoft Fabric

### 3.C.1 TL;DR

```yaml
WHAT: Microsoft Fabric integrated platform — Lakehouse + Direct Lake + Eventhouse +
      Notebook + Spark + Pipelines, all under one capacity unit (CU)
WHY: When enterprise customer mandates M365 / Power BI integration, OR when
     "single-platform" governance is a procurement requirement
COST_FLOOR: F2 capacity ~$262/mo `[likely]` (3.5× A at 1 dealer)
WHEN: Enterprise contracts with Microsoft estate; never for early-stage solo founders
```

### 3.C.2 Stack mapping

| Component | Fabric service |
|---|---|
| Storage | OneLake (Delta Parquet at the storage layer) |
| Compute (Spark) | Fabric Notebooks + Pipelines |
| Streaming | Eventhouse (KQL real-time intelligence) |
| Serving (BI) | Direct Lake semantic model + Power BI |
| Governance | Purview integration, sensitivity labels |
| Orchestration | Data Factory + Pipelines + Dataflow Gen2 |

### 3.C.3 When Fabric wins

```yaml
WIN_1: Customer already pays for M365 — capacity is bundled or discounted
WIN_2: Power BI is the primary consumer; Direct Lake is genuinely fast
WIN_3: Microsoft Purview / compliance / sensitivity labels matter
WIN_4: Enterprise IT requires "single-vendor integrated platform"
WIN_5: KQL real-time intelligence is genuinely useful for high-cardinality time-series
```

### 3.C.4 When Fabric loses

```yaml
LOSE_1: Capacity-unit pricing is fixed-cost (you pay even if idle)
LOSE_2: F2 floor ~$262/mo blocks early-stage startups
LOSE_3: Direct Lake limitations on row count and complex transforms force fallbacks
LOSE_4: OneLake is genuinely useful only if your Spark workload is in Fabric
LOSE_5: Lock-in to Microsoft tenancy
```

### 3.C.5 Cost gating issue

Fabric is sold by *capacity*, not query. Smallest reserved capacity is F2 — ~$262/mo `[likely]` in early 2026.

My realistic minimum for InventoryFlow production (Direct Lake + Eventhouse + Spark notebooks) is **F8 around $1,050/mo `[likely]`**.

Compared to Solution A at 1 dealer (~$76/mo), Fabric F8 is **14× the cost floor**. The economics flip when:
- Microsoft licensing already paid → marginal cost is the capacity itself
- Customer requires M365 integration as procurement gate
- Or volume reaches 1k+ dealers where Fabric's capacity becomes a fixed cost per unit-volume

### 3.C.6 Hire-ability signal

Solution C is the one I'd propose to an enterprise customer with Microsoft estate. My CV signals 5+ years of Fabric / Power BI / Spark / dbt — relevant hire-ability evidence.

---

## §6 SOLUTION D — AWS big-data stack

### 3.D.1 TL;DR

```yaml
WHAT: AWS-managed stack — S3 + Glue + Kinesis + MSK + Flink + Lambda +
      DynamoDB + Athena + Redshift Serverless
WHY: When dealer count > 5000, regulated industry, or AWS-mandate procurement
COST_AT_1000_DEALERS: ~$2,900/mo (5× Solution B at same scale) `[need-verify]`
WHEN: Enterprise / regulated, customer already on AWS, scale justifies operational complexity
```

### 3.D.2 Stack

| Layer | AWS service | Why |
|---|---|---|
| Storage | S3 Iceberg | $0.023/GB-mo `[verified]`, transactional |
| Catalog | Glue Catalog | $1/M requests, negligible |
| Batch ETL | Glue ETL (Spark) | Serverless DPUs |
| Streaming ingest | Kinesis Data Streams | Per-shard pricing |
| Streaming process | MSK + Managed Flink | Heavier than Redpanda but AWS-managed |
| Triggers | Lambda + Step Functions | Event-driven file landing |
| Ad-hoc | Athena | Per-TB scan pricing |
| Analytics | Redshift Serverless | RPU-hour pricing |
| Operational | DynamoDB | On-demand pricing |
| API | API Gateway + Lambda | Standard pattern |
| Monitoring | CloudWatch + X-Ray | Logs + traces |

### 3.D.3 When AWS wins

- Customer is already AWS-native and procurement requires AWS
- Volume > 50 TB historical OR > 10M events/day streaming
- Regulated industry (HIPAA, PCI, SOC 2 audit prep)
- Multi-region failover is non-negotiable
- Team has 3+ AWS-certified data engineers

### 3.D.4 Cost (at 1000 dealers)

| Service | $/mo | Note |
|---|---|---|
| S3 storage (50 TB Iceberg) | $1,150 | `[verified - rate × quantity]` |
| Glue ETL (Spark) | $400 | `[speculation - DPU-hour estimate]` |
| MSK 3-broker m5.large | $400 | `[speculation]` |
| Managed Flink (small KPU) | $300 | `[speculation]` |
| Redshift Serverless | $400 | `[speculation - RPU-hour estimate]` |
| Kinesis 10 shards | $50 | `[likely]` |
| Lambda invocations | $20 | `[likely]` |
| Step Functions | $30 | `[likely]` |
| Athena (10 TB scan) | $5 | `[verified rate × estimated volume]` |
| DynamoDB on-demand | $50 | `[speculation]` |
| API Gateway 1M req | $30 | `[verified rate]` |
| CloudWatch + X-Ray | $80 | `[likely]` |
| **TOTAL** | **~$2,900/mo** | **~$2.90/dealer at 1000 dealers** |

Compare to Solution B at same scale: $0.50/dealer. AWS is ~5× more expensive — justified only by procurement / regulation / scale.

### 3.D.5 Operational complexity

- Glue ETL job authoring is Python+Spark; debuggability is poor
- MSK requires Kafka admin skills (rebalance, IPC, ACL)
- Iceberg compaction on S3 needs explicit management
- Cost-allocation across 12 services requires CloudWatch dashboards
- IAM policies for cross-service access become a senior-engineer-day per service

---

## §7 LLM DEEP DIVE

### 7.1 The four genuine alternatives

| ID | Approach | $/call | Quality | Latency | Setup complexity |
|---|---|---|---|---|---|
| 1 | Paid API (cloud LLM) | $0.0005-0.05 | Frontier | <500ms | Low (HTTP) |
| 2 | Self-hosted LLM | $0 + hardware | High but lags ~12mo | 1-30s | High (MLX/Ollama) |
| 3 | Free API tier | $0 w/ rate limits | Medium | <500ms | Low |
| 4 | Pure OCR + rules | $0 (OSS) | Brittle on multilingual | <1s | Medium |

### 7.2 Self-host on personal hardware (Apple Silicon)

```yaml
HARDWARE: MacBook Pro M1 Max, 64 GB unified memory
FRAMEWORK: MLX (Apple-native, replaces CUDA)
MODELS_TESTED:
  - Qwen2-VL-2B-Instruct-4bit:     1.4 GB, 5 workers, ~10s/img, 87% JSON OK
  - Qwen2.5-VL-7B-Instruct-8bit:   8.3 GB, 3 workers, ~25s/img, 95% JSON OK [CHOSEN]
  - Qwen2.5-VL-32B-Instruct-4bit:  ~16 GB, 2 workers, ~60s/img — RAM tight
  - Qwen2.5-VL-72B-Instruct-4bit:  ~36 GB, 1 worker, ~10min/img — impractical

GPU_LIMIT: M1 Max ~10 TFLOPS FP16. Compared to H100 (670 TFLOPS) — 67× slower.
SWEET_SPOT: 7B model = best quality / latency / RAM trade-off on this hardware

KNOWN_BEHAVIORS:
  - kIOGPUCommandBufferCallbackErrorTimeout if a single image takes > ~5s of GPU work
  - 3-worker parallel = borderline; occasional 1-worker death every 30-60 min
  - 2-worker parallel = proven safe
  - Apple memory compression kicks in around 60 GB used
```

### 7.3 Self-host on cloud GPU

| GPU | Provider | $/hr | Models that fit | Note |
|---|---|---|---|---|
| H100 80GB | RunPod / Vast.ai | $2-3 | 72B-4bit + cache | `[likely]` Cost-effective for one-shot batch |
| A100 40GB | RunPod | $1-2 | 32B-4bit, 7B-8bit comfortable | `[likely]` |
| L40 / RTX 6000 | Vast.ai | $0.5-1 | 7B-8bit comfortably | `[likely]` |

For 1573 images: 1× H100 with 72B-4bit ≈ 30-45 min, **~$1.50-2.00** `[need-verify - depends on actual throughput]`.

Setup overhead is the gating factor: pull model (~36 GB download), pull images, install MLX/vllm, configure firewall. Worth it for repeated runs or larger volumes.

### 7.4 Paid API (frontier vision models)

| Provider | Model | $/img (vision OCR) | 1573 imgs total | Quality | Latency |
|---|---|---|---|---|---|
| Anthropic | Claude Sonnet 4.6 | ~$0.015-0.020 | ~$25-32 | Excellent JSON | 2-5s |
| Anthropic | Claude Opus 4.7 | ~$0.075-0.100 | ~$120-160 | Best frontier | 2-5s |
| OpenAI | GPT-5 vision | ~$0.025-0.035 | ~$40-55 | Frontier | 2-5s |
| Google | Gemini 2.0 Pro vision | ~$0.015-0.020 | ~$25-32 | Strong | 2-5s |
| OCR-only | Mistral OCR | ~$0.001-0.005 | $1.5-8 | OK text, weak schematic | <1s |

`Per-image cost = (image tokens × input rate) + (output tokens × output rate). Vision input is typically ~1500 tokens; output for callouts ~1000 tokens.`

**Parallelization**: 10-way parallel via batch API → 5-15 min for 1573 images. Frontier models complete fastest.

### 7.5 Hybrid (cache + API fallback)

```
PRIMARY:    cache.lookup(input)
            └── HIT: return cached output ($0)
            └── MISS:
                  ├── PRIMARY: self-host call
                  │     └── On success: cache + return
                  └── FALLBACK: API call (rate limited)
                        └── On success: cache + return

CACHE_DESIGN:
  - JSONL append-only (committed to repo for dev seeding)
  - Key: SHA-256(model_id + prompt + input_hash)
  - Value: { output, cost_usd, latency_ms, provider, timestamp }
  - Hit rate at steady state: 99% for translation, 80-90% for vision
```

### 7.6 Cost matrix by volume

| Volume / month | Best strategy | $/month |
|---|---|---|
| <1k calls | Local 7B self-host, cache off | $0 (just electricity) |
| 1k-10k | Local 7B + cache-default-on | $0-1 |
| 10k-100k | Local 7B + cache + paid API for misses | $5-30 |
| 100k-1M | Self-host 7B on GPU box ($200/mo) | $200-400 |
| >1M | Cluster + light fine-tuning + ensemble | $500-2000 |

### 7.7 Real timing data — 1573 images (MEASURED, complete 5-stage run)

```yaml
ACTUAL_RUN_2026_05_14:
  HARDWARE:        MacBook Pro M1 Max 64 GB
  MODEL:           Qwen2.5-VL-7B-Instruct-8bit (MLX)
  CONFIG:          max_pixels=602112, resize_longest_edge=1024px,
                   kv_bits=4, kv_group_size=64

PHASE_1_TIMINGS (3 workers parallel):
  W0 (slice 0/3): 525 imgs, wall 102.5 min, avg 24.3 s/img → 235 OK + 18 fail
  W1 (slice 1/3): 304 imgs, wall 125.8 min, avg 24.8 s/img → 282 OK + 22 fail
  W2 (slice 2/3): 250 imgs (alone after 2 deaths), avg 25.0 s/img → 236 OK + 14 fail
  Total Phase 1:  1573/1573 (100%), 1463 OK (93.0%), 110 fail
  Wall (overlap): ~4-5 hours

PHASE_2_TIMINGS (1 worker, anti-loop config):
  max_tokens=512, temperature=0.3, stricter prompt
  Processed 110 Phase 1 fails, avg 14.1 s/img
  Recovery: 39 / 110 (35.5%)
  Wall: 25.9 min

PHASE_3A_TIMINGS (Layer 3 consistency, pure Python):
  Detected 264 duplicate_n, 51 pos_hallucination, 39 invalid_pos, 34 empty_list
  Wall: <1 minute on 1573 records

PHASE_4_TIMINGS (Layer 4 ground-truth cross-reference):
  Read parts_table from xlsx (242 MB, ~30s)
  Per-image precision + per-sheet union coverage computed
  Wall: ~1 minute on 1573 records + 110 sheets

DB_INTEGRATION_TIMINGS (psycopg upsert):
  1573 rows into image_callouts table via ON CONFLICT
  Wall: <30 seconds

FINAL_CONFIDENCE_DISTRIBUTION (in image_callouts table, verified live):
  HIGH:    675 / 1573 (42.9%)  — Phase 1 OK + Layer 3 clean + precision ≥90%
  MEDIUM:  467 / 1573 (29.7%)  — Phase 2 recovered OR precision 70-90%
  LOW:     431 / 1573 (27.4%)  — precision <70% + 71 DEAD-mapped-to-LOW
  TOTAL:   1573 / 1573 (100%)

PHASE_4_PRECISION_DISTRIBUTION (per-image, vs parts_table ground truth):
  ≥90%:    981 / 1573 (62.4%)  — almost all callouts real
  70-90%:  171 / 1573 (10.9%)  — minor hallucination
  <70%:    313 / 1573 (19.9%)  — significant hallucination
  0%:        3 / 1573 (0.2%)   — all callouts hallucinated
  no_truth: 105 / 1573 (6.7%)  — text-only sheets (TABLE OF CONTENTS etc.)

PHASE_4_PER_SHEET_UNION_COVERAGE:
  100% coverage:  69 / 107 sheets (64.5%)
  ≥70% coverage:  91 / 107 sheets (85.0%)
  0% coverage:    5 sheets (text-only specs without schematic diagrams)

LAYER_3_FINDINGS (Phase 3a):
  264 images had duplicate_n (content hallucination, JSON valid but lying)
  51  images had pos_hallucination (≥90% callouts assigned same position)
  39  images had invalid_pos (non-enum pos value)
  34  images had empty_list

LAYER_4_FINDINGS (Phase 4):
  359 more records demoted HIGH → MEDIUM/LOW after cross-reference
  HIGH confidence dropped from 65.7% (Phase 3a) → 42.9% (Phase 4)
  This 22-percentage-point swing is the value Layer 4 adds

THE_SENIOR_INSIGHT:
  Phase 1 OK rate (93%) measures JSON validity. Phase 3a (65.7% HIGH)
  adds Layer 3 consistency. Phase 4 (42.9% HIGH) adds ground-truth
  cross-reference. Each layer catches what the prior layer missed.
  Without Layer 4, we'd over-state quality by 22 percentage points.

COST:
  Marginal: $0 (electricity ~$1 over ~5h)
  Comparison: Same task via Claude Sonnet 4.6 vision → ~$25-32, ~30min-2h
```

### 7.8 GPU watchdog quirk on M1 Max

```yaml
ERROR: libc++abi: terminating due to uncaught exception of type std::runtime_error:
       [METAL] Command buffer execution failed:
       (a) Impacting Interactivity (parallelism too high)
       (b) GPU Timeout (single image too heavy)

ROOT_CAUSE:
  macOS kernel has a watchdog that kills GPU command buffers exceeding ~5s
  to preserve UI interactivity. Vision-LLM inference on 7B model with high-resolution
  input can hit this when:
  - Image > 2048px on longest edge (prefill takes 30+ seconds)
  - 3+ workers compete for GPU bandwidth simultaneously

MITIGATION_APPLIED:
  - RESIZE_LONGEST_EDGE=1024 in parser.py (caps prefill at ~1300 tokens)
  - max_tokens=1024 (caps output, prevents hallucination loop wasting GPU time)
  - 2-3 workers parallel max (3 is borderline)
  - Append-mode + --resume → no data loss on worker death

NON_MITIGATION_OPTIONS:
  - Move to cloud GPU (Nvidia, no Metal watchdog)
  - Move to API (no GPU concerns)
```

### 7.9 max_tokens tuning

| max_tokens | Behavior | Trade-off |
|---|---|---|
| 512 | Truncates ~10% of dense schematics | Fast, low GPU; bad for >20 callout images |
| 1024 | Covers 3-30 callout images (95% of population) | Sweet spot |
| 2048 | Covers any reasonable schematic | Allows hallucination loop to waste 2 min/img before GPU watchdog kills |
| 4096 | Theoretical headroom | Worse — model burns more compute on garbage when it loops |

**Decision**: max_tokens=1024 in `parser.py`. Records that truncate (rare, dense schematics) become Phase 2 retry candidates.

### 7.10 Resize cap (1024px) — why

```yaml
WITHOUT_CAP (RESIZE=0):
  - Vision encoder receives raw image (could be 3000-4000px on longest edge)
  - Produces 8000+ prefill tokens
  - Single command buffer > 5 seconds → GPU watchdog kill
  - Workers die every 30 minutes

WITH_CAP (RESIZE=1024):
  - Pillow resize: 50-100ms overhead per image
  - Vision encoder receives 1024×N image
  - Prefill tokens: ~1300 max
  - GPU command buffer well under watchdog threshold
  - Workers stable for hours

QUALITY_IMPACT:
  Callout numbers in schematic are typically 12-24px tall at native resolution.
  At 1024px cap, they're still 8-15px — plenty for OCR. The model relies on
  context (line strokes, shading) which is preserved.
```

### 7.11 Classical OCR preprocessing — why I skipped

`[Tag: this is a section that anticipates a panel question]`

The Tesseract-era preprocessing pipeline (denoise → threshold → morphological ops → sharpen) is **the wrong tool for Qwen2.5-VL**:

```yaml
WHY_SKIP:
  1. INPUT_QUALITY:    Kayo schematic images are vector-derived PNGs from xlsx —
                       already clean, no scan artifacts to denoise.
  2. MODEL_MISMATCH:   Qwen2.5-VL is trained on raw natural images at internet
                       scale. Classical binary-OCR preprocessing strips shading,
                       colour, and gradient context the model uses internally.
                       Binarize/threshold throws away signal the model expects.
  3. RIGHT_TOOL_ALREADY_APPLIED: The only preprocessing that helped was adaptive
                       resize (1024px cap) — and that's a token-budget concern,
                       not a CV-quality concern.

CONSIDERED_FOR_FUTURE:
  - Document layout detection (DocLayout / LayoutLM) to crop non-schematic regions
    before VLM sees image. Different from classical preprocessing; addresses cost
    via fewer vision tokens. Deferred until > 10k images/day.

SENIOR_SIGNAL:
  Knowing the classical-CV toolkit exists and choosing NOT to apply it because
  the upstream model already does that work — adding preprocessing layers for
  "AI sophistication" would actively hurt quality.
```

### 7.12 Cost discipline — the architectural lever

`The single most important LLM-cost argument I make`:

> *"For early-stage data startups, LLM is the most over-spent line item I see. The bill, on workloads I've measured personally, can be in the $300-3,000/month range when it should be in the $0-30 range. That's 10-100× overspend. The discipline isn't the model — it's the cache. A well-designed system costs ~$2.50/month for this workload. The same system with sloppy implementation costs $2,500/month. Hardware doesn't change; the model doesn't change; the prompt doesn't change. What changes is whether anyone bothered to write `cached(provider).translate(...)` instead of `provider.translate(...)` everywhere it counted."* `[claim: speculation - illustrative, not empirically sourced]`

**The architectural lever**: cache decorator is **default-on** in Solution A. You have to actively turn it off to skip the cache. That single design choice is the difference between $2.50 and $2,500.

```yaml
COST_PATHS_I'VE_SEEN_OVERSPEND:
  - No caching: every call hits paid API
  - Cache without provider abstraction: can't swap to cheaper provider
  - No batch API: paying real-time rates for batch workload
  - No deduplication: same Chinese term translated N times across products
  - Audit mode running on every record (instead of sampled): 2× the API cost
  - "Quality assurance" calls that the cache already handled

COST_PATHS_DISCIPLINED_IMPLEMENTATIONS_USE:
  - Cache decorator default-on → 99% hit rate steady state
  - Global deduplication on Chinese term → 10-100× fewer unique calls
  - Provider abstraction: dev=mock, staging=cached+mock, prod=cached+anthropic-batch
  - Batch API where latency tolerates it (50% off Anthropic) [verified rate]
  - Audit mode sampled at 1-5%, not 100%
```

### 7.13 Fine-tuning vs prompt engineering vs cache

A separate question I get in interviews:

| Lever | When to apply | $/run | Time-to-effect |
|---|---|---|---|
| **Prompt engineering** | Always — first move | $0 | Hours |
| **Cache (this submission's lever)** | High repetition of identical inputs | $0 | Hours |
| **Fine-tuning** | After cache + prompts maxed out, AND you have ground-truth labels | $50-500 one-shot | Days-weeks |
| **Switch model class** | Quality ceiling reached on current model | Depends | Hours |
| **Ensemble (2 providers)** | Critical-quality workload where disagreement = defect | 2× per call | Hours |
| **Self-host fine-tune** | >100k/month + domain divergence from internet text | GPU box $200/mo | Weeks |

**My order of operations for InventoryFlow**:
1. Prompt engineering first (zero-cost, sets the floor)
2. Cache default-on (1-2 days, 10-100× cost lever)
3. Audit mode for quality measurement (1 week)
4. Ensemble if disagreement-rate matters (1-2 weeks)
5. Fine-tuning only if all above maxed out (1+ months, requires labeled corpus)

`Fine-tuning is the wrong first move 90% of the time. Most "we need to fine-tune" conversations resolve when someone fixes the prompt.`

### 7.14 Pure OCR alternative (option 4 from my matrix)

**Why pure OCR is tempting**: $0 marginal cost. OSS tooling (Tesseract, PaddleOCR, EasyOCR). No LLM hallucination concerns. Deterministic.

**Why I think it usually loses for this workload**:

```yaml
TESSERACT:
  PROS:  $0, 30 years of engineering, deterministic
  CONS:  Brittle on bilingual content; requires extensive preprocessing
         (threshold/morpho/denoise pipeline I rejected in §7.11);
         poor at unstructured text (callout labels embedded in line art);
         no semantic understanding (can't tell "1" from "I" without context)

PADDLEOCR:
  PROS:  Strong on Chinese; modern (CNN+CTC); $0; ONNX-exportable
  CONS:  Same preprocessing requirements as Tesseract;
         struggles with callout-label discrimination from drawing lines;
         no "what does this part name mean" semantic layer

EASYOCR:
  PROS:  Best out-of-box multilingual; pip install and go
  CONS:  Slower than PaddleOCR; same line-art confusion;
         output quality varies widely with image preprocessing

MISTRAL_OCR (paid managed):
  PROS:  Modern transformer OCR; handles layout reasonably
  CONS:  ~$0.001/page, breaks the $0-marginal-cost goal for self-host;
         still no semantic understanding of part naming
```

**What I'd actually use** (if forced to OCR-only):
- **PaddleOCR for Chinese text in parts table** (paid xlsx already extracts this, but if PDF input: PaddleOCR > Tesseract)
- **Qwen2.5-VL for schematic callout extraction** (VLM beats classical OCR on numbered diagrams)
- **Hybrid**: PaddleOCR for parts-table column rows, Qwen-VL for schematic spatial OCR

The reason Qwen-VL wins for schematic OCR specifically: it has **vision-language joint training**. It doesn't just read "1" — it knows "1" is in the top-left near the cylinder, which is the structural signal a downstream API needs.

### 7.15 Lessons from local self-host (production design implications)

The MLX run produced **empirical evidence** for design choices I'd otherwise have to defend with vibes:

**Lesson 1 — Model composition beats single-model picks**

Initially tested 2B-4bit alone. It was fast (10s/img) but **undercounted callouts** by 37% on busy schematics (verified against 7B output). The fix wasn't tuning the prompt — it was **switching model class**. 7B-8bit's quality improvement justified its 3× latency.

> *"For high-recall tasks (catalog OCR), small models that look fast in benchmarks fail silently in production. Recall is harder to measure than throughput; senior engineers measure it anyway."*

**Lesson 2 — GPU contention is the real ceiling on Apple Silicon, not RAM**

RAM stayed at ~26 GB total for 3 workers (out of 64). Plenty of headroom. But **GPU command-buffer time** is the bottleneck: 3 workers + heavy image → 1 worker dies with `kIOGPUCommandBufferCallbackErrorTimeout`. Pattern: 1 in 3 workers dies every 30-60 min.

> *"For Apple-Silicon self-host, the cost model is GPU-time per image, not RAM per worker. Plan for occasional restarts via `--resume` idempotent script."*

**Lesson 3 — Prompt engineering for small models (the 2B story)**

The 2B model had **56% JSON parse failure** on initial run because my prompt used `|` separator ("one of: a | b | c"). The 2B model treated `|` as literal text and included it in output, breaking JSON. Fixed by replacing with enumeration ("one of: a, b, c") + concrete example. Failure rate dropped to 13%.

> *"Small models need explicit, concrete prompt structure. The same prompt that works on 7B (which infers structure) fails on 2B (which copies literally). Test prompts on the smallest model in your fallback chain."*

**Lesson 4 — Cost economics is the architectural lever, not the model choice**

Running 7B locally for 1573 images: **$1 of electricity, 4-5 hours wall time**.
Same task via Claude Sonnet 4.6 vision: **$25-32, 30 min**.

Both are correct answers. The decision is which axis to optimize: cash burn (early-stage) vs wall time (real-time SLA).

> *"Don't optimize on quality alone. The architectural choice is which constraint binds: $$, time, or quality. Solution A picks cash. A production rollout with marketplace SLA would pick time and pay the API."*

**Lesson 5 — Production hardening I'd add beyond the take-home**

```yaml
LESSON_5_PRODUCTION_TODO:
  - OTLP exporter to Tempo/Honeycomb (not just SDK instrumented)
  - Severity-routed alerts (sev-1 PagerDuty, sev-2 Slack, sev-3 Linear)
  - Synthetic checks on fitment-lookup p99 latency every 60s
  - Chaos test: kill 1 worker mid-batch, verify --resume picks up cleanly
  - Cost-budget alarm: 24h LLM spend > 3× rolling-7d → page
  - Drift detection: re-run golden sample monthly, alert on >5% delta
  - Cohort review tooling: `pnpm audit --cohort <term>` for systemic disagreements
```

### 7.16 My single most important paragraph

`The policy decision that distinguishes a senior data engineer from a junior calling APIs and hoping`:

> *"The LLM is a defect detector, not the translator of record. That's the policy decision that matters. The dealer's translation goes to `name_en`. The LLM's goes to `name_en_llm` with a `data_quality` score. Disagreements get an `audit_status` flag. Marketplace-bound rows escalate to human review. Cohort patterns trigger a single prompt/rule fix instead of 100 individual reviews.*
>
> *That's the discipline. It's not 'my model is 95% accurate.' It's 'I have a 5-layer accuracy framework, an escalation path for failures, drift detection over time, and the LLM is in the supporting role.'*
>
> *Cache discipline is the cost lever. Audit discipline is the quality lever. The LLM provider is the **commodity** — swap one for another in 20 lines of code via the abstraction. The discipline is what compounds."*

### 7.17 Provider abstraction (the interface that makes the above possible)

```typescript
interface ILLMProvider {
  translate(input: TranslateInput): Promise<TranslateOutput>
  describe(input: ImageInput): Promise<DescribeOutput>
  // Each implementation: mock, cached, ollama, anthropic-batch, gemini, claude-code-handoff
}

const provider = cached(anthropicBatch(opts))  // composable decorator pattern
```

The **same interface** allows:
- Unit tests with `mock`
- Local dev with `cached(claudeCodeHandoff)` (zero API cost via Claude Max subscription)
- Production with `cached(anthropicBatch)` (50% discount via batch API)
- Audit mode with `ensemble(cached(anthropicBatch), cached(gemini))` (disagreement detection)

> *"Swapping providers is 1 line of config. Adding the cache is 1 line of code. Adding ensemble is 5 lines. The interface is what compounds — once you have it, the entire LLM strategy is composable. Without it, every LLM call is a hardcoded pet."*

---

## §8 ACCURACY VERIFICATION (AI Engineer perspective)

### 8.1 Five-layer accuracy framework

This is the production discipline I apply at Ashley Furniture (preflight DQ gates with severity tiers) and at Ecentric (multi-checkpoint DQ framework with automated halt on critical failures). Adapted to InventoryFlow:

| Layer | What I catch | Implementation in Solution A |
|---|---|---|
| 1. Schema validation | Wrong types, missing required fields, format violations | Zod runtime + Postgres NOT NULL constraints |
| 2. Domain rules | Year out of range, fitment incoherent, part_number malformed | `validators/` module + DB CHECK constraints |
| 3. Cross-row consistency | Same Chinese name translated differently across rows | Audit query in `ingest_audit` |
| 4. Cross-source agreement | LLM disagrees with dealer-supplied English | Audit mode; `agreement` column |
| 5. Downstream feedback | Marketplace listing rejected; SLA breach | Designed; deferred to marketplace integration |

### 8.2 Layer 3 — cross-row consistency

```sql
-- Same Chinese term must translate to same English. If not, one is wrong.
SELECT
  name_cn,
  array_agg(DISTINCT name_en) as distinct_translations,
  count(*) as row_count
FROM products
GROUP BY name_cn
HAVING count(DISTINCT name_en) > 1
ORDER BY row_count DESC;
```

Each disagreement is a candidate defect. Audit table records it. API decides:
- Surface dealer-supplied translation (default)
- Surface most common translation
- Flag for marketplace review

### 8.3 Layer 4 — cross-source agreement (ensemble)

**The policy decision that matters**: *the LLM is a defect detector, not the translator of record.*

```
DEALER_INPUT: name_en = "fuel filter"
LLM_OUTPUT:   name_en_llm = "fuel pump"

→ agreement: 'disagree'
→ audit_status: 'flagged'
→ ROUTING:
    PATH_1_AUTO_CORRECT: If LLM confidence = high AND Layer 3 also disagrees with
                         dealer, surface LLM translation, mark `auto_corrected`.
    PATH_2_MARKETPLACE_REVIEW: For marketplace-bound rows, any disagreement → human queue
    PATH_3_COHORT_REVIEW: Pattern of disagreements on same term → single fix in prompt/rules
```

### 8.4 Coverage rate vs parts table (ground truth signal)

**Most important verification for OCR specifically.** Ground truth EXISTS in the data:

```
Schematic image S → OCR returns callouts {n1, n2, n3, ...}
                       │
                       ▼ (Layer 3)
                       │
Parts table same sheet → rows numbered 1..N (ground truth from xlsx)
                       │
                       ▼
            coverage_rate = |OCR ∩ parts_table| / |parts_table|
```

| coverage_rate | Tier | Action |
|---|---|---|
| 100% | HIGH | Ship straight |
| 70-99% | MEDIUM | Ship + flag missing callouts in audit |
| < 70% | LOW | Halt + manual review queue (Path 2) |
| 0% / parse fail | DEAD | Phase 2 retry or human reads image |

### 8.5 Disagreement detection

```yaml
SOURCES_OF_DISAGREEMENT:
  - Dealer vs LLM (translation audit)
  - Cross-row inconsistency (same input → different outputs)
  - 2B vs 7B (different model classes give different counts)
  - 7B vs API frontier (when ensemble is wired)

WHERE_RECORDED:
  - ingest_audit.agreement: 'agree' | 'partial' | 'disagree'
  - ingest_audit.disagreement_severity: 'critical' | 'warning' | 'info'

WHAT_HAPPENS:
  - 'critical' + count > 5% of batch → halt run, alert ops
  - 'warning' → log + continue + dashboard count
  - 'info' → metric only
```

### 8.6 Confidence tiering (with measured 1573-image distribution, post-Layer 4)

| Tier | Trigger | Downstream behavior | Measured % |
|---|---|---|---|
| HIGH | Phase 1 OK + Layer 3 clean + precision ≥90% vs parts_table | Default API projection | **42.9% (675)** |
| MEDIUM | Phase 2 recovered OR precision 70-90% (demoted from HIGH) | Surface with audit flag | **29.7% (467)** |
| LOW | Layer 3 violations OR precision <70% (hallucinated callouts) | Hide from marketplace; ops review queue | **27.4% (431, includes 71 DEAD-mapped-to-LOW)** |

**The honest measurement story**:

| Step | "HIGH confidence" claim | Reality at that step |
|---|---|---|
| After Phase 1 (JSON parse OK only) | 93.0% (1463/1573) | But 264 had `duplicate_n` content hallucination |
| After Phase 3a (Layer 3 consistency) | 65.7% (1034/1573) | But 359 had hallucinated callout NUMBERS vs ground truth |
| After Phase 4 (Layer 4 ground truth) | **42.9% (675/1573)** | This is the defensible number |

Each layer catches what the prior layer missed:
- Phase 1 measures syntactic validity (JSON parses)
- Layer 3 measures internal consistency (no duplicate_n, no pos hallucination)
- Layer 4 measures content correctness vs external ground truth (parts_table)

The 22-percentage-point drop from 65.7% to 42.9% after Layer 4 is the **value Layer 4 adds**. Without it, we'd over-state quality by 22 percentage points. The 5-layer framework from `07-output-verification.md` is now empirically validated by this end-to-end run.

**Per-sheet UNION coverage** (different metric — across all images per sheet, are all parts callout-mapped?):
- 100% coverage: 69 / 107 sheets (64.5%)
- ≥70% coverage: 91 / 107 sheets (85.0%)
- 0% coverage: 5 sheets (TABLE OF CONTENTS, Carburetor Jets, etc. — text-only sheets without schematic diagrams, expected)

### 8.7 Manual sampling cadence

```yaml
DAILY:  Sample 10 records at random from yesterday's ingest. Manual reviewer
        eyeballs the schematic vs the callouts. 5 min/day = 50 records/week.

WEEKLY: Cohort review of disagreement patterns. If "carburetor jet" has
        3+ disagreements, fix the prompt or validator rule (not individual rows).

QUARTERLY: Re-run benchmark on golden sample. Drift detection: if accuracy on
        the SAME images drops month-over-month, investigate.
```

### 8.8 The senior policy decision

**You will never have 100% accuracy. You need to be measurable.**

Junior engineer: "the model is 95% accurate, so we're fine"
Senior engineer: "I measured X% high-confidence, Y% medium, Z% low — and here's
                  the escalation path for the Z%. Here's how I detect drift."

The policy decision matters more than the model choice:
- **Detect, don't pretend**: ingest_audit captures every disagreement
- **Defer, don't auto-trust**: marketplace-bound rows go through human queue
- **Cohort-fix, don't case-by-case**: pattern of N disagreements = single prompt fix
- **Measure drift over time**: golden samples re-evaluated quarterly

---

## §9 IMPLEMENTATION DETAILS

### 9.1 Schema (12 tables, 3 shapes)

```
SERVING_TABLES (denormalized, marketplace hot path):
  products              (main entity, fitment JSONB with GIN index)
  product_images        (R2 key references, callout count)
  image_callouts        (per-image OCR output, confidence tier)

CANONICAL_TABLES (normalized, governance):
  fitment_canonical     (year, make, model, model_code, variant per product)
  parts_metadata        (Chinese/English/Korean names, category)
  product_aliases       (part_number_norm mapping)

MDCP_TABLES (metadata-driven control plane):
  dealers
  ingestion_patterns
  dealer_pattern_bindings

OPERATIONAL:
  ingest_runs           (run lifecycle, source_file_sha256 unique)
  ingest_audit          (LLM calls, cost, latency, agreement [migration 0005])
  stream_events         (outbox pattern)
  stream_outbox         (transactional outbox)
```

### 9.2 Idempotency mechanisms

```yaml
LAYER_1_FILE: ingest_runs.source_file_sha256 UNIQUE
              → same xlsx file ingested twice = no-op

LAYER_2_PRODUCT: products.part_number_norm UPSERT KEY
                 → generated column: upper(regexp_replace(part_number, '[[:space:]]+', '', 'g'))
                 → "AB S-123" → "ABS-123" → matches "ABS-123" → upsert not insert
                 → migration 0004 fixed bug: was regexp_replace(part_number, 's', '', 'g')

LAYER_3_IMAGE: SHA-256 of image bytes → R2 key
               → HEAD before PUT → already exists = skip
               → same image used by N dealers = uploaded once

LAYER_4_LLM: cache key = SHA-256(model_id + prompt + input_hash)
             → identical translation/OCR call = cache hit, $0 marginal
```

### 9.3 Migration journal

```yaml
0000_cuddly_tiger_shark.sql:
  - 12 tables + indexes + generated columns
  - part_number_norm regex (bug fixed in 0004)

0001_add_*.sql:
  - Initial enums, types

0002_row_level_security.sql:
  - RLS on products, product_images, ingest_runs, stream_events, dealer_pattern_bindings
  - app.current_dealer_id session setting

0003_*.sql:
  - Additional column additions

0004_fix_part_number_norm.sql:
  - Drop broken generated column (was stripping letter 's', not whitespace)
  - Recreate with [[:space:]]+

0005_ingest_audit_dealer_agreement.sql:
  - Add dealer_id + agreement columns to ingest_audit
  - Backfill dealer_id from ingest_runs
  - CHECK constraint: agreement IN ('agree','partial','disagree')

0006_rls_ingest_audit_and_null_fix.sql:
  - Enable RLS on ingest_audit
  - source_dealer_id NOT NULL (closes NULL leak)
  - Tenant-scoped policy joining ingest_runs.dealer_id
```

### 9.4 CI/CD flow (concise)

```
PR → lint+typecheck → tests (with postgres+redis) → bench (gate: p99<5ms)
   → security (pnpm audit, pip-audit, trivy — advisory) → docker build (digest-pin)
   → merge to main
```

All GitHub Actions SHA-pinned per `tj-actions/changed-files` 2025 incident lesson. Audit/Trivy advisory until triage rotation exists, then fail-on-high.

### 9.5 Docker (concise)

- Base: `node:22-alpine@sha256:<digest>` via `NODE_DIGEST` build arg
- Compose: postgres+redis+minio+mc all pinned by digest (no `:latest`)

### 9.6 Observability (concise)

- **Logs**: Pino structured JSON, `run_id` correlation
- **Traces**: OpenTelemetry SDK at major span boundaries
- **Metrics**: Prometheus `/metrics` (env-flag enabled)
- **Audit**: `ingest_audit` row per LLM call (cost, latency, cache_hit, agreement)
- **Runs**: `ingest_runs` lifecycle (pending → running → succeeded/failed)
- **Errors**: 4 categories matching accuracy layers (Schema / Domain / Cross-row / LLM-disagree)

---

## §10 SECURITY ARCHITECTURE

### 10.1 Five layers

| Layer | What it covers | Solution A status |
|---|---|---|
| IAM | Caller identity, RBAC | Header-trust 🧪 (target: JWT/OIDC 📐) |
| Tenant isolation | Cross-dealer leak prevention | RLS on all data tables ✅ |
| Secret management | Credential storage | .env gitignore ✅, platform store 📐 |
| Supply chain | Dependency CVE, image integrity | SHA-pinned actions ✅, Trivy scan ✅ |
| Audit + observability | What happened, when, by whom | ingest_audit + Pino ✅ |

### 10.2 Threat model (STRIDE-derived)

```yaml
SPOOFING:    JWT signature verification (deferred), per-tenant tokens
TAMPERING:   Postgres ACID + SHA-256 content addressing
REPUDIATION: ingest_audit captures every operation with run_id correlation
INFORMATION_DISCLOSURE:
  - Cross-tenant via Postgres bug → RLS default-deny mitigates
  - JWT spoofing → server-side verify + 7d rotation
  - R2 URL guessed → SHA-256 prefix (256 bits entropy), signed URLs in production
  - Cross-tenant via materialised view → inherits RLS in Postgres 15+
  - Cross-tenant via shared cache → cache keys include dealer_id prefix
  - SQL injection → Drizzle parameterized-only by default
DOS:         BullMQ rate limits, Postgres connection pooler, Cloudflare WAF
ELEVATION:   RBAC role matrix (deferred); break-glass with time-boxed audit
```

### 10.3 Shipped vs Demo vs Deferred

See impl repo `STATUS.md` for the 55-row truth table. Summary:

```
IMPLEMENTED (27):    RLS, idempotency, SHA-pinned CI, audit/Trivy steps,
                     ingest_audit + dealer_id, .env safety, audit trail

DEMO_FOR_SUBMISSION (6):
                     Header-trust auth, public R2 URLs, anonymous MinIO local dev,
                     R2 per-dealer prefix isolation, partial outbox publisher,
                     in-progress MLX OCR

PRODUCTION_TARGET_DEFERRED (22):
                     Full JWT verify, RBAC role middleware, TLS at ingress,
                     platform secret store, WAF custom rules, SOC 2 / ISO 27001,
                     HSM-backed secrets, CMEK, SIEM, Grafana dashboard, OTLP
                     backend wiring, severity-routed alerting, chaos engineering,
                     ensemble agreement layer, marketplace feedback loop,
                     MDCP runtime dispatcher, signed images, SBOM, etc.
```

---

## §11 OPERATIONS & SLO

### 11.1 SLI definitions

| SLI | Target | Alerting |
|---|---|---|
| Ingest success rate | ≥ 99% | <95% sustained → page |
| Parse error rate | < 1% | >3% per dealer → warn |
| R2 upload success rate | ≥ 99.9% | <99% → page |
| LLM cache hit rate | ≥ 90% | <70% → warn (cost risk) |
| API p99 latency | < 200ms | >500ms → page |
| LLM disagreement rate | < 30% | >50% → cohort review |

### 11.2 Error taxonomy

4 categories matching the accuracy framework:

```
1. SCHEMA: row didn't parse (wrong type, missing required field)
   → reject row, log, continue
   → halt run if >5% of batch

2. DOMAIN: parsed but value impossible (year 1850, invalid model_code prefix)
   → reject row, log, continue
   → halt run if >2% of batch

3. CONSISTENCY: parsed and valid but inconsistent (same Chinese → different English)
   → accept row, flag in audit
   → cohort-review weekly

4. AGREEMENT: LLM disagrees with dealer-supplied English
   → accept row, mark `audit_status: flagged`
   → marketplace queue if marketplace-bound
```

### 11.3 Runbook entries

```yaml
RUNBOOK_R1_INGEST_FAILED:
  Trigger: ingest_run status = 'failed'
  Steps:
    1. Read ingest_audit for run_id
    2. Identify category (schema/domain/consistency/agreement)
    3. Schema/domain → fix validator, re-run with --replay
    4. Consistency/agreement → review audit table for pattern

RUNBOOK_R2_LLM_COST_SPIKE:
  Trigger: 24h LLM cost > 3× baseline
  Steps:
    1. Check cache hit rate (should be ~99%)
    2. If <70% → cache invalidation bug, investigate
    3. If new dealer onboarding → expected, will stabilize

RUNBOOK_R3_GPU_WATCHDOG_KILL (local dev only):
  Trigger: MLX worker exits with code 134
  Steps:
    1. Verify worker died: `ps aux | grep batch_vision_ocr`
    2. Check log: kIOGPUCommandBufferCallbackErrorTimeout or Impacting Interactivity
    3. Relaunch with --resume flag (idempotent, skips done SHAs)
    4. If persistent: reduce workers from 3 → 2, OR lower max_tokens
```

### 11.4 DR / BCP

```yaml
RPO_TARGETS:
  Phase 1 (0-500 dealers):   24 hour (managed snapshot)
  Phase 2 (500-1500):        4 hour (logical replication + 4h snapshot)
  Phase 3 (1500-5000):       15 min (sync replication + Iceberg time-travel)

RTO_TARGETS:
  Phase 1: 4 hour (managed snapshot restore)
  Phase 2: 1 hour (failover to read replica + promote)
  Phase 3: 15 min (Iceberg VERSION AS OF rollback)
```

---

## §12 COST CLAIMS — TAGGED AUDIT

`Per §0 zero-hallucination rule. Tagged classifications:`

### 12.1 All $ figures with verification status

| Claim | Value | Tag | Source |
|---|---|---|---|
| R2 storage rate | $0.015/GB-mo | `[verified]` | Cloudflare R2 published |
| S3 storage rate | $0.023/GB-mo | `[verified]` | AWS S3 published |
| EKS control plane | $73/mo | `[verified]` | $0.10/hr × 730h |
| Anthropic Sonnet 4.6 input | $3/M tokens | `[verified]` | Anthropic published |
| Anthropic Sonnet 4.6 output | $15/M tokens | `[verified]` | Anthropic published |
| Anthropic batch discount | 50% off | `[verified]` | Anthropic published |
| Solution A monthly (1 dealer) | ~$76 | `[likely]` | Sum of likely line items |
| Fly.io 2x shared-cpu-2x | $30/mo | `[likely]` | Fly.io pricing, may be outdated 2026 |
| Neon Pro 1 CU + 10 GB | $40/mo | `[likely]` | Neon pricing tier |
| Upstash Redis pay-as-you-go | $5-10/mo | `[speculation]` | Depends on actual usage |
| Hetzner CX41 | €15/mo | `[likely]` | Hetzner pricing |
| Hetzner CX21 | €4.51/mo | `[likely]` | Hetzner pricing |
| Microsoft Fabric F2 | $262/mo | `[likely]` | Microsoft published late-2025 |
| Microsoft Fabric F8 | $1,050/mo | `[likely]` | Microsoft published late-2025 |
| Vietnam senior salary | $3.5k-5k NET | `[likely]` | VietnamWorks/LinkedIn surveys |
| Solution D total at 1000 dealers | ~$2,900/mo | `[need-verify]` | Architecture estimate, not billing sim |
| Phase 2 projection | ~$200/mo | `[speculation]` | Linear projection from $76 |
| Phase 3 projection | ~$1,500/mo | `[speculation]` | Linear projection |
| "Teams paying $300-3000/mo" | qualitative | `[speculation]` | NOT a real conversation — illustrative |
| Steady-state LLM ~$2.50/mo | qualitative | `[speculation]` | Cache-hit assumption |
| $30/dealer/mo amortised | qualitative | `[speculation]` | Architectural division of $76/100 |

`Defense for the panel: "These are order-of-magnitude estimates assembled from published baseline pricing. The line items I can defend on rate × quantity are R2 storage, S3 storage, EKS control plane, Anthropic per-call. Aggregate monthly totals are architectural estimates, not 30-day billing simulations."`

---

## §13 PANEL ANSWERS (anticipated questions)

### 13.1 "Why JSONB not normalized?"

`Senior answer`:

> *"I chose JSONB for the **serving** shape because marketplace queries are 'find products fitting vehicle X' — that's exactly the access pattern JSONB + GIN excels at, sub-millisecond p99.*
>
> *I did **not** abandon normalization. The same canonical data is materialized into 3 shapes — JSONB for serving, normalized for governance/joins, wide for analytics. The canonical shape lives in `fitment_canonical` with normalized columns. Cross-dealer reporting hits the analytics shape via materialized view.*
>
> *So the choice isn't JSONB vs normalized. It's recognizing that different consumers need different shapes, and the same source of truth is materialized into each."*

### 13.2 "Why not Iceberg?"

`Senior answer`:

> *"Iceberg is Solution B in my deck. I chose A first because at 1 dealer it ships 3× faster, costs the same operationally, and uses the JD's exact stack.*
>
> *I would migrate to Iceberg when one of six triggers fires — most likely dealer_count > 500 or historical_volume > 50 TB. The migration path is real: existing Postgres data exports to Parquet, dbt models port to Trino. Iceberg gets schema evolution + time-travel that Postgres doesn't.*
>
> *The trigger language is the senior signal: I'm not saying A is forever. I'm saying A is right NOW and here's exactly when it stops being right."*

### 13.3 "Why local 7B not GPT-4o?"

`Senior answer`:

> *"Three reasons.*
>
> *First, **cost discipline**: $0 marginal cost for self-host vs $25-32 per 1573-image batch for Claude Sonnet 4.6 or GPT-5. For an early-stage company doing this hundreds of times during dev, that adds up to thousands.*
>
> *Second, **architectural signal**: knowing the API exists and choosing not to depend on it shows I understand the engineering trade-offs. A senior engineer can reason about when API economics flip — typically when volume > 100k/month, when team has SLA contracts, or when latency requirements drop below the model's local inference time.*
>
> *Third, **the production path**: solution A's LLM strategy doc describes the hybrid (cache + API fallback). The cache decorator is default-on, so the first time we onboard a new term, it might call API at $0.0005. Every subsequent call is $0. At 99% cache hit rate steady-state, the monthly bill is single-digit dollars even at thousands of calls."*

### 13.4 "How do you know it's accurate?"

`Senior answer`:

> *"I don't claim 100% accuracy. The senior signal isn't 'my model is perfect' — it's 'I have a 5-layer framework for measuring it and an escalation path for failures.'*
>
> *For OCR specifically, I have ground truth: the parts table xlsx has 1 row per callout. So I compute `coverage_rate = OCR_callouts ∩ parts_table_rows / parts_table_rows`, tier it (HIGH/MEDIUM/LOW), and route low-confidence to manual review.*
>
> *For LLM translation, I have cross-row consistency (same Chinese → same English) and cross-source agreement (LLM vs dealer-supplied). Disagreements get recorded in `ingest_audit.agreement` and routed to ops review or marketplace-listing-review depending on context.*
>
> *The policy decision is documented: the LLM is a defect detector, not the translator of record. Dealer English goes to `name_en`. LLM goes to `name_en_llm` with a quality score. Disagreements get a flag. That's the discipline that distinguishes a senior data engineer from someone calling APIs and hoping."*

### 13.5 "Where's the threat model?"

`Senior answer`:

> *"It's in `docs/11-security-architecture.md` and `docs/12-slo-observability.md`. STRIDE-derived. Five layers: IAM, tenant isolation, secret management, supply chain, audit.*
>
> *The most critical threat I track is **cross-tenant data leak** — dealer A seeing dealer B's data. I mitigate at three levels: Postgres RLS (default-deny), JWT signature verification (deferred to production), and R2 key entropy (SHA-256 256-bit prefix, signed URLs in production).*
>
> *For supply chain: GitHub Actions SHA-pinned (not tag-pinned, after the 2025 `tj-actions/changed-files` incident), pnpm audit + pip-audit + Trivy in CI as advisory gates, Docker base images digest-pinnable via NODE_DIGEST build arg. SOC 2 / ISO 27001 / bug bounty / pen-test are deferred with triggers.*
>
> *Each deferred item has a trigger condition documented. That's the difference between 'shipped checklist of fancy security features' and 'I know what I shipped, what's demo, what's deferred, and why.'"*

### 13.6 "What if dealer count grows 100×?"

`Senior answer`:

> *"100× is dealer_count = 10,000. That's past Trigger 1 (>500) and well into Solution B or C territory.*
>
> *Trigger 1 (>500 dealers): migrate to Iceberg + Trino. Storage cost flattens (no per-dealer DB). Compute scales horizontally with Trino workers.*
>
> *Trigger 5 (schema churn): MDCP runtime dispatcher comes online. Different OEM schemas get different parser plugins, registered in `dealer_pattern_bindings`. No hardcoded branching in Solution A.*
>
> *Trigger 6 (RTO < 1h): if marketplace SLA requires sub-15-min RTO, Iceberg VERSION AS OF rollback wins.*
>
> *Migration is incremental: data exports from Postgres → Iceberg via dbt seed. dbt models port. The Fastify API becomes a thin proxy to Trino for read paths. The migration risk is the **streaming pipeline** (BullMQ → Redpanda) — that's where I'd spend the most time on testing."*

### 13.7 How to read these results — the pitch framing

> The temptation when presenting measured results is to lead with "5-layer accuracy framework empirically validated, 22-percentage-point swing between Phase 3a and Phase 4 demonstrates cross-source agreement layer value". That language is academic. It's also wrong for a panel pitch — they want story and judgment, not jargon.
>
> Here is the cleaner framing.

**Opening — 20 seconds, one paragraph**:

> *"I parsed 3,938 products from the xlsx. I OCR'd 1,573 schematic images locally on my Mac, ~5 hours wall time, near-zero marginal cost. Then I verified the OCR output against the parts list in the same xlsx — using the data itself as ground truth, not self-evaluation. 43% of images came out high-confidence. The rest are tiered into review and reject queues, each with an explicit downstream path. No image is 'lost' — every one of the 1,573 has a route."*

**When asked "Why only 43% high-confidence?"** —

> *"43% is the defensible number. The model produces valid JSON 93% of the time — but JSON validity isn't content correctness. When I cross-reference against the parts list, 50 percentage points get demoted. That's the discipline of running all five accuracy layers, not just the first one. If I claimed 95% and a reviewer asked 'how do you know', I couldn't defend it. I'd rather claim 43% I can defend than 95% I can't."*

**When asked "Why two tracks?"** —

> *"I built the same pipeline twice. Track A on the JD's stack (TypeScript / Postgres / Drizzle) to ship fast. Track B on the production-target stack (Python / Iceberg / Dagster / dbt) to scale. The two implementations agree on 99.97% of products. More importantly, the 0.03% disagreement isn't random noise — Track B's stricter parser catches 4 encoding bugs and 1 header artefact in Track A. Two parsers on the same input is a cross-validation mechanism at the system level. Migration from A to B isn't just safe — it's quality-improving."*

**When asked about cost** —

> *"Local self-host: $0 marginal cost, 5 hours wall time. Same task through Claude Sonnet vision API: $25–32, 30 minutes. Both are correct answers. The decision is which constraint binds — cash burn or wall time. For a take-home and early-stage production, I picked cash. For a marketplace with real-time SLA, I'd pick the API. The trigger for switching is in the doc — I'm not hand-waving the trade-off."*

**Closing — 10 seconds**:

> *"I can tell you what percentage of my system is correct, what percentage is wrong, where it's wrong, and what the routing is for each error class. That's what I'm pitching — not the accuracy number, the accountability of the system."*

---

**Vocabulary to avoid in pitch (the academic-sounding phrases)**:

| Don't say | Say instead |
|---|---|
| "5-layer accuracy framework empirically validated" | "I checked the output 5 different ways" |
| "Phase 4 Layer 4 cross-source agreement" | "Cross-checked the model against the parts list" |
| "Confidence distribution: HIGH 42.9%, MEDIUM 29.7%, LOW 27.4%" | "43% clean, 30% flagged, 27% review" |
| "0.03% parity delta" | "Match 99.97% — the gap caught real bugs" |
| "Layer 3 detected 264 duplicate_n hallucinations" | "Caught 264 cases where the model repeated callout numbers — 30-line consistency check" |
| "kIOGPUCommandBufferCallbackErrorTimeout" | "Apple's GPU watchdog killed it when I pushed too many parallel workers" |
| "Architecturally restrained from over-engineering" | "I deliberately didn't ship X — here's the trigger when I would" |

The pattern: **numbers as anchors, judgment as content, story as throughline**. The panel will remember the story. The numbers earn the right to tell it.

---

## §14 APPENDICES

### 14.1 File map (solution repo)

```
Inventoryflow_solution/
├── README.md                          ← Landing page, full Solution A overview
├── BRIEFING.md                        ← THIS FILE (master reference)
├── LICENSE
├── assets/
│   ├── CV_Aric_Nguyen.pdf
│   ├── cv-page-1.png
│   └── cv-page-2.png
└── docs/
    ├── 00-tldr.md                     ← 3-minute decision summary
    ├── 01-problem-framing.md          ← My read of the brief
    ├── 02-solution-A-recommended.md   ← Detail of Solution A
    ├── 03-solution-B-de-standard.md   ← Detail of Solution B (Iceberg)
    ├── 04-solution-C-fabric-brief.md  ← Microsoft Fabric brief
    ├── 05-solution-D-aws-brief.md     ← AWS big-data brief
    ├── 06-llm-strategy.md             ← LLM strategy + preprocessing decision
    ├── 07-output-verification.md      ← 5-layer accuracy framework
    ├── 08-operations.md               ← CI/CD, runbooks, risk register
    ├── 09-engineering-judgment.md     ← Closing argument
    ├── 10-data-architecture.md        ← Canonical/serving/analytics shapes
    ├── 11-security-architecture.md    ← IAM, tenant, threat model
    ├── 12-slo-observability.md        ← SLI/SLO/alerts/error taxonomy
    └── 13-integration-contracts.md    ← API contracts, schema registry
```

### 14.2 Cross-link to impl repo

```
github.com/ankinguyen-engineer-2002/inventoryflow-catalog-ingest

├── STATUS.md                          ← Implementation truth table (55 claims)
├── track-a-jd-native/                 ← Solution A code (TypeScript)
│   ├── src/
│   ├── migrations/
│   │   ├── 0000_cuddly_tiger_shark.sql
│   │   ├── 0002_row_level_security.sql
│   │   ├── 0004_fix_part_number_norm.sql
│   │   ├── 0005_ingest_audit_dealer_agreement.sql
│   │   └── 0006_rls_ingest_audit_and_null_fix.sql
│   ├── package.json
│   ├── docker-compose.yml
│   └── Dockerfile
├── track-b-data-engineering/          ← Solution B PoC (Python + dbt + Trino)
├── shared/
│   ├── llm-cache.jsonl                ← Committed cache
│   ├── vision-mlx/                    ← MLX OCR pipeline
│   │   ├── parser.py                  ← VisionParser (Qwen 7B)
│   │   ├── batch_vision_ocr.py        ← Parallel runner
│   │   ├── phase2_refine.py           ← Phase 2 retry
│   │   └── integrate_into_track_a.py  ← Phase 3 DB upsert
│   └── handoff/                       ← Claude Code seed artifacts
├── docs/
│   ├── decisions/                     ← 15 ADRs
│   │   ├── ADR-001 through ADR-015
│   └── bench/
│       └── bench-results.json
└── .github/workflows/ci.yml           ← SHA-pinned actions + audit + Trivy
```

### 14.3 Reading order by panel role

```yaml
SENIOR_ENGINEER (60 min):
  1. BRIEFING.md §1, §2, §3.A.1-9   (40 min)
  2. impl-repo/STATUS.md             (15 min)
  3. impl-repo/docs/decisions/       (skim, 5 min)

SOLUTION_ARCHITECT (90 min):
  1. BRIEFING.md §1, §2, §3.A.3-12  (40 min)
  2. BRIEFING.md §3.B-D              (30 min)
  3. BRIEFING.md §13                  (20 min)

DATA_ARCHITECT (60 min):
  1. BRIEFING.md §3.A.4, §3.A.7      (15 min)
  2. BRIEFING.md §8 (verification)   (20 min)
  3. docs/10-data-architecture.md    (20 min)
  4. BRIEFING.md §3.B.4              (5 min)

SECURITY_ENGINEER (30 min):
  1. BRIEFING.md §10                  (15 min)
  2. docs/11-security-architecture.md (10 min)
  3. impl-repo/STATUS.md security rows (5 min)

PRODUCT / EM (20 min):
  1. BRIEFING.md §1.2, §3.A.1, §2.2  (10 min)
  2. BRIEFING.md §13.1-13.4           (10 min)
```

---

## END OF BRIEFING

```yaml
TOTAL_LINES: ~1500
TOTAL_SOLUTIONS: 4 (A, B, C, D — full coverage)
TOTAL_PHASES_OF_IMPLEMENTATION: 3 (OCR / Refinement / Integration)
TOTAL_ACCURACY_LAYERS: 5
TOTAL_SECURITY_LAYERS: 5
TOTAL_PANEL_QUESTIONS_ANSWERED: 6
TOTAL_COST_CLAIMS_TAGGED: 22

PRIMARY_OUTPUT_FORMAT: AI-friendly structured tables + YAML-like blocks
SECONDARY_OUTPUT_FORMAT: Human reading via TOC + anchors

LAST_UPDATED: 2026-05-14
AUTHOR: Aric Nguyen
ROLE: Senior Data Engineer / Solution Architect applicant for InventoryFlow × Talemy
```
