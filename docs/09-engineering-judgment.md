# Engineering Judgment — the part you can't get from a tech stack

> *"In the AI era, tech stack and coding are commodities. What's left is judgment under constraint, taste in trade-offs, and the discipline to say no to elegance when shipping matters."*

This is the closing argument of the submission. The other documents prove I can build the system. This one explains why I'd build *this* version of it.

---

## The wrong question is "which stack"

Every architect interview I've sat in eventually drifts toward "would you use Kafka or Redpanda?" or "Snowflake versus Databricks?". Those are real questions; they're also the ones AI assistants and Stack Overflow have already answered to within a margin of error that doesn't matter for most companies.

The question I find myself asking instead, over and over, is some version of:

- **What's the actual bottleneck?** Almost never the runtime. Usually the schema you committed to in week one.
- **Who maintains this in 18 months?** A Lakehouse + dbt + Dagster stack assumes you can hire data engineers. In Bình Dương or Ho Chi Minh City, that hiring pool is small; in San Francisco it's expensive. Match the stack to the hiring market, not to the conference talks.
- **What's the cost of being wrong?** If the wrong data hits a marketplace listing, what's the downstream blast radius? That number determines how much validation infrastructure is justified — long before it determines compute choice.
- **What can I delete?** Senior engineers earn their pay by removing pieces of the system, not adding them. The temptation, especially with AI tooling pumping out scaffolding for free, is to ship a 12-service compose file when 4 services would do.

I have built systems where I got the answer wrong. The 60-minute mart cycle I cut to 20 minutes at Ashley wasn't a tech choice — it was the result of finally drawing the DAG and noticing that 17 of the 20 marts could run in parallel. The Azure → Fabric migration at Ecentric wasn't about Direct Lake being faster (it is); it was about removing the import-refresh bottleneck that no one had explicitly chosen and everyone had quietly accepted. **The win was deletion, not addition.**

---

## What I drew on, deciding for InventoryFlow

A few specific things from prior work, applied to this brief:

### From Ashley Furniture — *metadata-driven control plane* (Jan 2026 – Present)

At Ashley, the global supply-chain analytics platform runs on Microsoft Fabric across 5,000+ enterprise tables and 30 years of history. The team's original pattern was per-table stored procedures: every new dataset meant editing T-SQL.

The refactor was a metadata-driven control plane: a registry of datasets, a generic load runner routing 8 patterns (overwrite, incremental, upsert, SCD2, CDC, date-key, date-range, identity), a lineage builder, a DQ engine, an audit ledger. Onboarding a new dataset is now a registry insert. **Zero per-asset code.**

That pattern appears explicitly in this submission's Solution A as `dealer_pattern_bindings` and `ingestion_patterns`. The current implementation seeds the tables but doesn't run the dispatcher (deferred to dealer #2 — one of the migration triggers in [`08-operations.md`](./08-operations.md)). It's there because the JD says *"hundreds of dealerships, efficient onboarding"*, and I've seen what happens when that requirement is met with per-dealer code branches: every new dealer is a week of work and a hidden technical debt. The bindings table costs nothing to seed today and pays off the first time a dealer asks for a schema variant.

### From Ecentric — *Direct Lake semantic boundary* (Mar 2023 – Jan 2026)

At Ecentric, I led the phased Azure → Fabric migration for a hybrid Lakehouse + Warehouse platform serving 4 departments and 15+ Power BI reports. The non-obvious lesson: Direct Lake quietly falls back to DirectQuery if the underlying table is wrong shape (calculated columns, certain data types, view-over-view). When that happens, Power BI users see queries 10–100× slower with no error message — they just file tickets.

The fix is to make the Direct Lake boundary an explicit architectural object: physical Gold tables (no views) with disciplined column types, per the Microsoft official guidance. Semantic-model refresh orchestrated post-load via Fabric REST API, so the latest data shows up minutes after ETL completes — not after the user complains.

The InventoryFlow analogue: marketplace-facing data is the equivalent of a "Gold serving table". You don't want LLM translations leaking into production through a view layer that someone "temporarily" created. So in Solution A, `products` is the serving table; `ingest_audit` is the staging-quality table; the boundary is enforced by where you write, not by where you query. **A senior architect makes that boundary explicit in week one; it's almost impossible to retrofit in year two without a migration.**

### From SOPA / TheSocietyPass — *streaming-capable but boring* (Mar 2023 – Jan 2024)

At SOPA the platform was multi-pattern ingestion: Airbyte for batch, Debezium for CDC, WebSocket for real-time. All converged into Redpanda with a schema registry, then Flink SQL for stream processing, dbt + Trino for batch on Iceberg, dual-write to ClickHouse (hot) and Iceberg/MinIO (cold).

That stack is correct **for SOPA's scale and access pattern** (multi-marketplace e-commerce, sub-second OLAP requirements, billions of events). It is wildly incorrect for a company that is going to ingest its first OEM catalog this quarter.

The InventoryFlow Solution B (Polars + Iceberg + Dagster + dbt + Redpanda + RisingWave) is essentially the SOPA stack adapted for parts catalog data. It's the architecture I would *recommend at year two*. It is not the architecture I would *ship today*. **The difference between a senior architect and a senior architect interviewing for a senior architect role is the willingness to recommend the boring thing.**

### From ADP — *governance comes from naming, not from tools* (Jan 2022 – Nov 2022)

At ADP I deployed OpenMetadata. It is good. So is DataHub, so is Unity Catalog, so is Microsoft Purview. The trap is thinking the tool *is* the governance. The naming conventions, the data-dictionary discipline, and the source-of-truth documentation we wrote were what actually moved the needle on data quality. The tool just gave them a UI.

In this submission, `docs/decisions/` (in the impl repo) is the data dictionary. ADRs are governance. The tool stack — Pino logs, OpenTelemetry, Prometheus stubs, structured `run_id` correlation — is the floor, not the ceiling. **If you take this codebase and rename half the columns, no observability stack will save you.**

---

## What I would never do on this brief

A senior consultant signals taste partly through what they refuse. So:

1. **I would not use Prisma here.** Drizzle's JSONB type inference is concrete; Prisma's `Json` type collapses to `unknown` and forces runtime casts. The fitment column is the hot path of the test specification. Type safety on the hot path is non-negotiable.

2. **I would not deploy Kafka or Kinesis for streaming under 1,000 events/sec.** Postgres `LISTEN/NOTIFY` and the transactional outbox pattern give sub-500ms p95 latency with a single dependency that already runs everywhere. The day you need Redpanda, you swap one publisher implementation. Until then, you don't.

3. **I would not call OpenAI or Anthropic on every row.** The LLM provider abstraction in this submission has six implementations specifically because the API cost question is the most over-paid line item in early-stage data startups. A JSONL cache committed to git, plus an Ollama or Claude-handoff fallback for development, plus a paid API for production with a 99%+ cache hit rate at steady state, costs roughly $0.25/month at 1,000 dealers. Most teams I've talked to are paying $300–$3,000 for the same workload.

4. **I would not use Gemini for OEM data.** Their Terms of Service permit training on customer data by default. Even with the opt-out, the legal hold is shakier than Anthropic or self-hosted. For a dealer-supplied catalog, the data-residency conversation is short: don't put it on a provider whose default is "we keep it".

5. **I would not pretend to ship Solution C (Microsoft Fabric) without a capacity commitment.** Fabric is excellent for the company I work at now (Ashley Furniture has US HQ infrastructure agreements). For an early-stage AI company without that commitment, the floor cost of Fabric F2 — let alone F8 — is a six-figure annual line item that wasn't in the original problem.

6. **I would not skip the audit table.** `ingest_audit` records every LLM call's cost, latency, cache-hit, and disagreement with the dealer's translation. That table is 80% of what makes the AI tooling section credible. Without it, the LLM is a black box; with it, the LLM is a measurable subsystem. **You cannot run AI in production without an audit trail. Anyone who tells you otherwise has not yet been on the receiving end of a "why did the part number list change overnight?" support ticket.**

---

## What I'd build differently if this were real production

The 12-hour AI-assisted submission has three deliberate corners cut. A live walkthrough should hit each:

1. **Outbox publisher is stubbed.** The `stream_outbox` table exists and is written transactionally; a separate publisher draining to Redpanda or Kafka is not yet implemented. Today the streaming path uses `pg_notify` (which works at this scale). The outbox is there so when streaming demand crosses the threshold, you swap publishers without touching the write path.

2. **MDCP dispatcher is metadata-only.** Three registry tables seeded; runtime dispatch deferred to dealer #2. The brief asks for "efficient onboarding of hundreds of dealerships". The current shape demonstrates the design; the runtime is a follow-on week.

3. **OpenTelemetry exporter is configured but not pointed at a backend.** Pino logs are structured with `run_id` correlation, Prometheus `/metrics` endpoint is a placeholder. Pointing OTLP at an actual collector (Tempo, SigNoz, Honeycomb) is environment-specific. Decided to ship the instrumentation, defer the backend choice to the company's observability standard.

These are not bugs. They are explicit deferrals, recorded in ADRs, with the trigger conditions for un-deferring documented in [`08-operations.md`](./08-operations.md).

---

## The brief in one sentence — restated

The job description says "*not over-engineered, enterprise-heavy boilerplates*" in one paragraph and "*pragmatism and speed*" in the next. It would be possible to read those phrases as permission to skip rigor. I read them the other way: they are permission to **omit specific complexity, not to omit specific discipline**. The discipline is in the validation, the audit table, the ADRs, the migration triggers, the cost economics. The complexity that's omitted is the lakehouse, the streaming bus, the Microsoft capacity contract. **Knowing which to omit and which to keep is the job.**

---

## What's left after the AI does its part

I think a lot about this. Junior engineers used to spend their early years learning syntax, debugging deployment scripts, copy-pasting from Stack Overflow, mastering one ORM. AI assistants compress that learning into months. The career advice "learn TypeScript / Python / Rust" matters less every year.

What still seems to compound is:

- **Taste in trade-offs**, calibrated by watching production systems fail in specific ways
- **The willingness to say "I don't know"** in a design review, then follow up the next morning with a concrete experiment
- **The discipline to keep saying "no" to complexity** when AI is making complexity nearly free to generate
- **The instinct to draw boundaries** — between dev and prod, between hot path and cold path, between LLM-validated and human-validated — and to make those boundaries explicit objects in the architecture
- **The ability to translate** between business stakeholders (who want certainty and ROI) and engineers (who want elegance and degrees of freedom), and to refuse to ship either group's worst impulses

None of those are downloadable from npm. None of them are obvious from a CV bullet point. **Four years of production data platforms is, mostly, four years of being wrong slightly less often.**

If this submission is judged purely on whether the code runs, Solution A passes. If it's judged on whether the author can think — that is the conversation I would like to have.

---

**— Aric**

[github.com/ankinguyen-engineer-2002](https://github.com/ankinguyen-engineer-2002) · [aricnguyen.analytics2002@gmail.com](mailto:aricnguyen.analytics2002@gmail.com)
