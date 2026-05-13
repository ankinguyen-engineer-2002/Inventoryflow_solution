# SLI / SLO / Observability — measurable production discipline

> The senior critique I anticipate: *"you mentioned Pino + OpenTelemetry but where are the SLI/SLO definitions, the error taxonomy, the replay semantics?"* This document is that answer.

I separate observability into four operational disciplines. Each ships fundamentals in Solution A; senior-grade extensions are documented with explicit triggers.

---

## SLI — Service Level Indicators

The eight indicators I'd instrument from day one:

| # | SLI | Definition | Where measured |
|---|---|---|---|
| 1 | **Ingest success rate** | `successful_runs / total_runs` over a rolling window | `ingest_runs.status` |
| 2 | **Ingest latency** | wall-clock from `started_at` to `completed_at` per run | `ingest_runs.duration_ms` |
| 3 | **Parse error rate** | rows rejected by Zod / domain validators / 100 attempted | `ingest_runs.rows_failed / rows_attempted` |
| 4 | **R2 upload success rate** | successful PUTs / attempted PUTs | Pino logs + R2 response codes |
| 5 | **LLM cache hit rate** | `cache_hit=true / total LLM calls` | `ingest_audit.cache_hit` |
| 6 | **Catalog API latency** | p50 / p95 / p99 for `GET /api/products`, `GET /api/fitment` | Fastify request middleware + OpenTelemetry |
| 7 | **Catalog API error rate** | 5xx / total responses | Same |
| 8 | **LLM disagreement rate** | `agreement='disagree' / total audit rows` | `ingest_audit.agreement` |

Each SLI has a query you can run against the live system, not a vague description.

---

## SLO — Service Level Objectives

The bar I'd commit to publicly. Below the bar is a tracked incident.

| SLO | Target | Window | Rationale |
|---|---|---|---|
| Ingest success rate | ≥ 99.0% | 30-day rolling | One failed run per ~30 days is acceptable; OEM files have schema drift |
| Ingest latency p95 | ≤ 90 seconds for files < 250 MB | 30-day rolling | Measured baseline; some headroom for noisy parses |
| Parse error rate | ≤ 5% of rows | per run | Anything higher → schema drift or new signature needed |
| R2 upload success | ≥ 99.9% | 7-day rolling | Idempotent; transient failures retried |
| LLM cache hit rate | ≥ 95% at steady state | 7-day rolling | Below this = cost runaway, alert |
| Catalog API p95 | ≤ 100 ms | 5-min rolling | Marketplace SLA requirement |
| Catalog API p99 | ≤ 250 ms | 5-min rolling | Outliers acceptable; chronic = problem |
| Catalog API error rate | < 0.1% | 30-day rolling | Standard 99.9% availability bar |
| LLM disagreement rate | 10–25% (a *range*, not a max) | per dealer per ingest | Below 10% = LLM rubber-stamping; above 25% = systemic translation issue |

### Why some SLOs are ranges, not maximums

The LLM disagreement rate is the most interesting one. If it's **too low** (< 10%), the LLM is agreeing with the dealer regardless of correctness — useless. If it's **too high** (> 25%), the dealer's translation discipline is broken or our prompt is. **Both edges are alarms.** Most observability tooling only handles "above max" — I'd extend the alerting to handle the "below min" edge.

### Error budget

Each SLO implies an error budget. For 99.0% ingest success → 1% error budget → 7.2 hours of degraded service per 30-day window. When budget is consumed:

- Halt non-critical feature work
- Focus on reliability
- Post-mortem the failures
- Tighten the bar OR widen the budget after evidence

---

## Alerting — the rules I'd actually wire up

I split alerts into three severities. Every alert has an explicit "who responds, how soon, runbook URL".

### Severity 1 — Page on-call immediately

| Trigger | Threshold | Runbook |
|---|---|---|
| Catalog API p99 > 1 second | 5-min window | [Runbook 3](./08-operations.md#runbook-3--fitment-query-p95--50-ms-regression-r-04) |
| Catalog API error rate > 1% | 5-min window | Halt deploys; rollback last release if recent |
| Ingest run fails | Immediate | [Runbook 2](./08-operations.md#runbook-2--section-detector-fails-on-new-oem-file-r-01) |
| Postgres unreachable | 30-sec window | DBA on-call; failover to replica |
| Data corruption signal (audit row count deviation > 5%) | Per run | [Production data corruption response](./08-operations.md#how-id-respond-to-a-production-data-corruption-incident) |

### Severity 2 — Slack channel + Linear ticket within 1 business day

| Trigger | Threshold | Action |
|---|---|---|
| LLM cost share > 30% of cloud bill | Daily rollup | [Runbook 1](./08-operations.md#runbook-1--llm-cost-share--30-of-cloud-bill-r-02) |
| LLM disagreement rate > 25% | Per dealer per run | Cohort investigation; potentially prompt-template version bump |
| LLM disagreement rate < 10% | Per dealer per run | Provider review (is LLM rubber-stamping?) |
| Cache hit rate < 95% | 24-hour rolling | Check prompt-template-version changes; rebuild cache offline |
| R2 storage growth > 20% week-over-week | Weekly | Investigate; potentially per-dealer dedup |
| Test failures on `main` branch | Immediate | Revert + investigate |

### Severity 3 — Daily digest email

| Trigger | Action |
|---|---|
| Top 10 dealers by cost | Operations review |
| Top 10 failed runs | DE review |
| Schema-drift candidate signals | DE review weekly |
| Slow query log entries | DBA review weekly |

---

## Dashboard layout

A single Grafana / Datadog dashboard called **"InventoryFlow Operations"** with the following panels. ASCII sketch:

```
┌─────────────────────────────────────────────────────────────────────┐
│  InventoryFlow Operations · live · last 24h                         │
├──────────────────┬──────────────────┬───────────────────────────────┤
│  INGEST          │  CATALOG API     │  LLM                          │
│                  │                  │                               │
│  Success rate    │  p50 / p95 / p99 │  Cache hit rate              │
│  ████ 99.4%      │  60ms 87ms 142ms │  ████ 98.7%                  │
│                  │                  │                               │
│  Runs today: 47  │  Reqs/sec: 23.1  │  Calls: 142 / day            │
│  Failed: 3       │  Errors: 0.02%   │  Cost: $0.31 / day            │
│                  │                  │  Disagreement: 18%            │
├──────────────────┼──────────────────┼───────────────────────────────┤
│                  │                  │                               │
│  Ingest latency  │  API p95 trend   │  LLM cost trend (7d)         │
│  (last 24h)      │  (last 7d)       │                               │
│  ▁▃▅▇▅▃▁ 48s avg│  ▅▆▅▇▇▅▆ ~92ms  │  ▁▂▂▃▄▃▂ $2.10 / week         │
│                  │                  │                               │
├──────────────────┴──────────────────┴───────────────────────────────┤
│  R2 storage          │  Postgres connections │  Per-dealer break-out│
│  Used: 47 GB / 1 TB  │  Active: 12 / 100     │  table (top 5)       │
│  Cost: $4.70/mo      │  Idle: 88             │                       │
├──────────────────────┴───────────────────────┴───────────────────────┤
│  Recent incidents · 30-day rolling                                   │
│  ─ 2026-05-08: Section detector fail on dealer-13 (45m, runbook-2)  │
│  ─ 2026-04-22: Postgres connection exhaustion (12m, scaled pool)    │
└──────────────────────────────────────────────────────────────────────┘
```

Solution A ships the data sources for this (Pino logs, `ingest_audit`, `ingest_runs`). The dashboard itself is environment-specific — deferred to the point at which an on-call rotation exists.

---

## Error taxonomy

I classify every error into one of four categories. Each gets a different handling discipline.

| Category | Example | Handling | Auto-retry? |
|---|---|---|---|
| **User-data error** | Dealer xlsx has unknown header signature | Reject the run with specific error; surface to dealer's review queue | No — user must fix |
| **System error** | Postgres connection pool exhausted | Retry with exponential backoff; alert on persistent | Yes (capped at 3 attempts) |
| **Provider error** | Anthropic API returns 503 | Failover to backup provider (Ollama / cached); alert if persistent | Yes (failover) |
| **Logic error** | Our code's assertion fails | Fail the run; immediate alert; do not retry | No — needs human |

The taxonomy lives in code as an `IngestError` discriminated union:

```typescript
type IngestError =
  | { type: 'user_data'; sub: 'unknown_signature' | 'malformed_row' | 'unsupported_format'; ... }
  | { type: 'system';    sub: 'pg_unavailable' | 'r2_unavailable' | 'redis_unavailable'; ... }
  | { type: 'provider';  sub: 'llm_rate_limit' | 'llm_5xx' | 'llm_timeout'; ... }
  | { type: 'logic';     sub: 'invariant_violation' | 'assertion_failed'; ... }
```

The `type` field drives:
- Whether to retry
- Whether to alert
- Which audit category in `ingest_audit`
- Which dashboard panel shows it

**This is the discipline I find missing in most architecture docs.** Treating "an error happened" as one undifferentiated thing leads to retry storms on user errors and silent retries on logic errors.

---

## Replay and retry semantics

A senior architect's commit: **say explicitly what guarantee each pipeline gives**.

| Pipeline stage | Guarantee | Mechanism |
|---|---|---|
| **xlsx upload** → `ingest_runs` insert | Exactly-once | `source_file_sha256` UNIQUE on `ingest_runs`; conflict = no new run |
| **Row parse** → `products` upsert | Idempotent (re-run safe) | `ON CONFLICT (source_dealer_id, part_number) NULLS NOT DISTINCT DO UPDATE` |
| **Image upload** → R2 | Idempotent | SHA-256 content addressing; HEAD before PUT |
| **LLM call** → `ingest_audit` | At-least-once | Cache hit returns synchronously; miss retries 3× with exp backoff; failed calls record `error` row but don't kill the pipeline |
| **Stream event** → `stream_outbox` | Transactional with the row write | Single transaction insert into `stream_events` + `stream_outbox` |
| **Outbox** → Redpanda / Kafka | At-least-once | Publisher reads `stream_outbox`, sends, marks sent; failure leaves it for retry. Consumer dedupes on `event_id` |
| **Marketplace sync** | At-least-once | Same outbox pattern downstream |

### What I won't claim

- **Exactly-once across the full pipeline** — that requires distributed transactions, which I don't ship. The honest guarantee is **at-least-once with idempotent consumers**.
- **Total ordering across dealers** — the streaming plane orders within a dealer only.
- **Real-time guarantees < 100 ms** — sub-second is realistic; sub-100ms requires a different architecture.

### The replay command

Every run can be replayed:

```bash
# replay run abc-123 (re-parse the original xlsx, re-upsert products)
pnpm ingest:replay --run-id abc-123

# replay with current parser logic against the source file
pnpm ingest:replay --source-sha256 def456...
```

The mechanism: `ingest_runs.source_file_sha256` plus an R2 archive of every uploaded file means **any historical state is reproducible from the source**. This is the "audit log replay" path I describe in [`07-output-verification.md`](./07-output-verification.md). Solution B's Iceberg `VERSION AS OF` is a different and faster replay path for the same fundamental requirement.

---

## Tracing — distributed tracing with OpenTelemetry

Every request gets a `trace_id`. Every span gets a `span_id` + `parent_span_id`. The spans I instrument:

```
trace: ingest-run-abc-123
├── span: cli.ingest                          (parent)
│   ├── span: parse.read-xlsx                  (1.2 s)
│   ├── span: parse.detect-sections            (0.4 s)
│   │   ├── span: parse.section[chassis-v1]    (0.2 s)
│   │   ├── span: parse.section[engine-v1]     (0.1 s)
│   │   └── span: parse.section[u8-v1]         (0.1 s)
│   ├── span: image.extract                    (0.3 s)
│   │   ├── span: image.r2.put × 382           (0.2 s parallel)
│   │   └── span: image.r2.head × 1586         (0.1 s parallel)
│   ├── span: db.insert.products × 3938        (0.5 s parallel)
│   ├── span: llm.enrich                       (0.8 s)
│   │   ├── span: llm.cache-check × 68         (0 s — committed cache)
│   │   └── span: llm.call × 0                 (cache fully hit)
│   └── span: db.populate.vehicle_models       (0.1 s)
```

The full trace answers "where did the 60 seconds of wall-time go?" with a flamegraph. **Most ingest performance bugs are visible only at the trace level, not at the metric level.**

Solution A instruments these spans via the `@opentelemetry/api` package; the OTLP exporter target (Tempo, Honeycomb, SigNoz, Datadog) is environment-specific and configured via env var.

---

## Per-tenant SLO tracking (deferred — multi-tenant year-2)

When InventoryFlow becomes multi-dealer at scale, each dealer needs its own SLO scorecard:

| Dealer | Ingest success | API p95 | LLM disagreement | Status |
|---|---|---|---|---|
| Kayo | 99.4% | 87 ms | 14% | ✅ |
| Dealer X | 87.3% | 92 ms | 32% | 🚨 schema drift |
| Dealer Y | 99.8% | 145 ms | 9% | ⚠️ LLM rubber-stamping |

This requires per-dealer aggregation in the SLO calculations. Postgres `ingest_audit` already has `dealer_id`, so it's a SQL aggregation, not a new data path. Deferred to the point at which the dealer count > 10.

---

## What I'd add for full enterprise observability (deferred with triggers)

| Item | Trigger to un-defer |
|---|---|
| Full APM (Datadog APM / New Relic / Honeycomb) | Year 1 production release |
| Synthetic monitoring (browser-based catalog API canary) | First marketplace customer in production |
| User Experience Monitoring (RUM) for dealer admin UI | When dealer admin UI is built |
| Custom Grafana dashboards per persona (ops / DE / DBA / cost) | Team size > 5 |
| Runbook automation (auto-execute Runbook 1 on cost alert) | Recurring incidents (3+ in 30 days) |
| Chaos engineering (regular fault injection) | Year 2; only after SRE on staff |
| Service-level error budget burn-down dashboards | First quarterly SLO review |

Each has a specific trigger. **None ship in the take-home demo.** Naming them honestly is the senior signal.

---

## What concretely ships in Solution A's observability posture today

- ✅ Pino structured JSON logs with `run_id` correlation across workers
- ✅ `ingest_audit` table — per-LLM-call cost / latency / disagreement
- ✅ `ingest_runs` registry — run lifecycle state
- ✅ OpenTelemetry SDK instrumented for the major spans
- ✅ Prometheus `/metrics` endpoint stub (env var to enable)
- ✅ Bench gates in CI on fitment-query latency
- ✅ Sample-output committed for reviewer verification without running anything
- ✅ Error classification in code (the 4-category taxonomy)
- ✅ Idempotency tested (`pnpm test:idempotency`)

The aspirations above are real, scoped, and triggered. The shipped controls are visible in the impl repo.

---

**Back to:** [README](../README.md) · [01-problem-framing](./01-problem-framing.md) · [02-solution-A-recommended](./02-solution-A-recommended.md) · [09-engineering-judgment](./09-engineering-judgment.md)
